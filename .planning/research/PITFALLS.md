# Pitfalls Research

**Domain:** Retry-engine migration (guava-retrying `com.github.rholder:guava-retrying:2.0.0` → failsafe.dev `dev.failsafe:failsafe:3.3.2`) inside a Dropwizard RabbitMQ actors library
**Researched:** 2025-09-19
**Confidence:** HIGH (behavioral claims verified against decompiled/source jars of both libraries in `~/.m2`, not from memory)

## How to read this document

Every behavioral claim below was verified by reading the actual source of both libraries:

- guava-retrying 2.0.0 sources: `Retryer.call()` (`Retryer.java:155`), `RetryerBuilder` (`RetryerBuilder.java:130-183`), `ExponentialWaitStrategy` (`WaitStrategies.java`), `RetryException.java`.
- failsafe 3.3.2 sources: `SyncExecutionImpl.executeSync()` (`SyncExecutionImpl.java:186`), `FailurePolicy.isFailure()` (`spi/FailurePolicy.java`), `FailurePolicyBuilder` javadoc (`FailurePolicyBuilder.java:29-33`), `RetryPolicyBuilder.withMaxAttempts/withMaxRetries/withBackoff` (`RetryPolicyBuilder.java:447-492, 250-300`), `RetryPolicyExecutor.adjustForBackoff` (`internal/RetryPolicyExecutor.java`), `FailsafeException.java`.

The codebase facts were verified by reading `RetryStrategy.java`, all 7 `retry/impl/*` strategies, `RetryConfig.java`, `CountLimitedExponentialWaitRetryConfig.java`, `CommonUtils.isRetriable`, and the sole call site `Handler.handleDelivery` (`base/Handler.java:98-118`).

## Critical Pitfalls

### Pitfall 1: Exponential backoff curve is silently halved (off-by-one in the exponent)

**What goes wrong:**
The exponential wait strategy produces a different delay sequence after migration even when the config values are mapped 1:1. Messages retry on a slower schedule than before, changing effective throughput and time-to-dead-letter.

**Why it happens:**
The two libraries index the exponent differently.

guava-retrying `WaitStrategies.exponentialWait(multiplier, maxTime, unit)` — `ExponentialWaitStrategy.computeSleepTime`:

```
sleep = multiplier * 2^(attemptNumber)        // attemptNumber is 1-based: the FAILED attempt
```

After attempt 1 fails → `multiplier * 2^1 = 2 * multiplier`. After attempt 2 → `4 * multiplier`. After attempt 3 → `8 * multiplier`.

failsafe `withBackoff(delay, maxDelay, factor)` — `RetryPolicyExecutor.adjustForBackoff`:

```
if attemptCount != 1:  delay = delay * delayFactor   (capped at maxDelay)
```

So the base `delay` is used as-is for the FIRST retry, then multiplied. With `factor=2` and `delay=multiplier`:
After attempt 1 fails → `1 * multiplier`. After attempt 2 → `2 * multiplier`. After attempt 3 → `4 * multiplier`.

**The failsafe curve is exactly half the guava curve at every step** when `delay = multiplier` and `factor = 2`.

**How to avoid:**
To preserve the exact guava schedule, set the failsafe base delay to `2 * multiplier`:

```java
// guava:  exponentialWait(multiplier, maxTime, MILLISECONDS)  → 2m, 4m, 8m...
// failsafe: withBackoff(2 * multiplier, maxTime, ChronoUnit.MILLIS, 2.0)  → 2m, 4m, 8m...
```

Note failsafe's `withBackoff(delay, maxDelay, factor)` requires `delay > 0`, `delay < maxDelay`, and `delayFactor > 1` — all satisfied here. The config field is named `multipier` (sic, a pre-existing typo in `CountLimitedExponentialWaitRetryConfig`); do not "fix" the typo during migration (config-compatibility constraint).

If exact parity is not required, document the deliberate drift with a `ponytail:` comment naming the ceiling.

**Warning signs:**

- A migration PR that writes `withBackoff(config.getMultipier(), config.getMaxTimeBetweenRetries(), ChronoUnit.MILLIS, 2.0)` — this is the half-speed trap.
- Integration tests that only assert "a retry happened" but not the inter-attempt sleep durations.
- The `@Min(1)` / `@Max(Long.MAX_VALUE)` validation on `multipier` (default `1`) means the default first sleep drops from 2ms (guava) to 1ms (failsafe) — invisible without timing assertions.

**Phase to address:**
Implementation phase (per-strategy migration). Add a timing-parity test in the test phase that asserts the exact sleep sequence for `CountLimitedExponentialWaitRetryStrategy` and `TimeLimitedExponentialWaitRetryStrategy`.

---

### Pitfall 2: Default exception handling is INVERTED between the two libraries

**What goes wrong:**
A strategy that, in guava, retried on NO exceptions (because no `retryIfException` was set) becomes, in failsafe, a strategy that retries on EVERY exception (because no `handle` was set) — or vice versa. The most dangerous direction is the silent one: a strategy that previously did not retry on an exception now retries it, causing unexpected reprocessing and delayed failure surfacing.

**Why it happens:**

- guava-retrying: the default rejection predicate is `Predicates.alwaysFalse()` (`RetryerBuilder.java:39`). With NO `retryIfException`/`retryIfResult` call, **nothing is retried** — the first exception propagates immediately. `retryIfException(predicate)` ORs the predicate in, so retry happens ONLY when the predicate matches.
- failsafe: per the `FailurePolicyBuilder` javadoc (`FailurePolicyBuilder.java:29-33`) and `FailurePolicy.isFailure` (`spi/FailurePolicy.java`): with NO `handle`/`handleIf` condition configured, **any exception is a failure and triggers retry**. Once any `handle`/`handleIf` is added, `exceptionsChecked` becomes true and only matching exceptions are retried; non-matching exceptions are thrown immediately (not retried).

So the defaults are opposite. The codebase's 6 retrying strategies all call `retryIfException(predicate)` → these map cleanly to `handleIf(predicate)` and the per-exception behavior is preserved. **The trap is `NoRetryStrategy`**: it sets NO predicate and only `stopAfterAttempt(1)`. In guava that means "no retry predicate, stop after 1 attempt" — no retry. The failsafe equivalent must explicitly use `withMaxAttempts(1)`; if someone instead writes a bare `RetryPolicy.builder().build()`, failsafe's default of 3 attempts (2 retries) on ANY exception would silently turn no-retry into retry-3-times.

**How to avoid:**

- Map `retryIfException(predicate)` → `handleIf((result, ex) -> predicate.test(ex))` (note: failsafe's `handleIf(CheckedBiPredicate)` takes a `(result, exception)` bi-predicate; the single-arg `handleIf(CheckedPredicate<Throwable>)` also exists and is the closer match).
- Map `NoRetryStrategy`'s `StopStrategies.stopAfterAttempt(1)` → `RetryPolicy.builder().withMaxAttempts(1).build()`. Do NOT rely on "no config = no retry" — in failsafe, no config = retry-on-anything-up-to-3.
- After migration, grep the new code for `RetryPolicy.builder()` that has no `.handle`/`.handleIf`/`.withMaxAttempts(1)` — each one is a latent "retry on everything" bug.

**Warning signs:**

- A `RetryPolicy.builder().build()` with no failure conditions and no explicit attempt cap.
- Tests that pass because the happy path returns a result, but never exercise the exception path — the inverted default only shows up under failure.
- `NoRetryStrategy` suddenly taking 3x longer to fail in integration tests.

**Phase to address:**
Implementation phase (per-strategy). Verification phase must include a test that throws a non-retriable exception through `NoRetryStrategy` and asserts exactly ONE attempt, and a test that throws a retriable exception through a count-limited strategy and asserts the configured attempt count.

---

### Pitfall 3: Attempt-counting API has two names with off-by-one between them

**What goes wrong:**
The migrator picks `withMaxRetries(n)` thinking it equals guava's `stopAfterAttempt(n)`, but `withMaxRetries(n)` allows `n+1` total attempts. Messages get one extra retry attempt, changing failure semantics and dead-letter timing.

**Why it happens:**

- guava `StopStrategies.stopAfterAttempt(n)`: stop when `attemptNumber >= n`. So `n` = **total attempts** (attempt 1 is the initial call; attempts 2..n are retries).
- failsafe `withMaxAttempts(n)`: `n` = **total attempts** (`RetryPolicyBuilder.java:441-442`: "2 retries equal 3 attempts"). Internally stores `maxRetries = n - 1`.
- failsafe `withMaxRetries(n)`: `n` = **retries after the first attempt** = `n + 1` total attempts.

So `stopAfterAttempt(n)` ≡ `withMaxAttempts(n)` ≡ `withMaxRetries(n - 1)`. The trap is using `withMaxRetries(config.getMaxAttempts())`, which yields `maxAttempts + 1` total attempts.

**How to avoid:**
Always translate guava `stopAfterAttempt(maxAttempts)` → failsafe `withMaxAttempts(maxAttempts)`. Never use `withMaxRetries` for this migration. The config field is `maxAttempts` (`@Min(1)`, default `1`) — its name already says "attempts", so `withMaxAttempts` is the semantically matching call.

Edge case: `NoRetryStrategy` uses `stopAfterAttempt(1)` → `withMaxAttempts(1)`. `withMaxAttempts(1)` is legal (failsafe rejects 0 and values < -1, but 1 is fine) and means "one attempt, zero retries" — exact equivalent.

**Warning signs:**

- A diff containing `withMaxRetries(config.getMaxAttempts())`.
- Tests asserting attempt count that pass with either off-by-one direction (write the assertion against the exact configured number, not "more than 1").
- `maxAttempts = 1` (the `@Min` default) silently becoming 2 attempts via `withMaxRetries(1)`.

**Phase to address:**
Implementation phase. Verification phase: a count-limited strategy test that throws a retriable exception and asserts the callable was invoked exactly `maxAttempts` times, not `maxAttempts + 1`.

---

### Pitfall 4: Exhaustion exception type changes (RetryException → FailsafeException / raw RuntimeException)

**What goes wrong:**
When retries are exhausted on a checked exception, guava throws `RetryException` (a checked `Exception` whose `getCause()` is the original failure); failsafe wraps checked exceptions in `FailsafeException` (a `RuntimeException`) and rethrows unchecked exceptions and `Error`s directly. Downstream code that catches `RetryException` by type, or that branches on `instanceof RetryException`, silently stops catching the exhaustion case.

**Why it happens:**

- guava `Retryer.call()` (`Retryer.java:173-174`): on stop-with-pending-exception, `throw new RetryException(attemptNumber, attempt)`. `RetryException extends Exception` (checked). Its cause is the original throwable.
- failsafe `SyncExecutionImpl.executeSync()` (`SyncExecutionImpl.java:190-197`): `RuntimeException` → rethrown directly; `Error` → rethrown directly; any other (checked) `Throwable` → `throw new FailsafeException(exception)`. `FailsafeException extends RuntimeException`.

So the exhaustion exception's static type changes from checked to unchecked, and its class changes entirely.

**Codebase impact (verified):**

- `grep -r RetryException src/` → **zero matches**. This library does not catch `RetryException` anywhere.
- The sole call site, `Handler.handleDelivery` (`base/Handler.java:106`), catches `Throwable t` and routes it through `errorCheckFunction.apply(t)` (a caller-supplied `Function<Throwable, Boolean>`) and `exceptionHandler.handle()`. Neither branches on `RetryException` internally.
- `execute(Callable<Boolean>)` declares `throws Exception` (`RetryStrategy.java:33`); the public API contract is "throws Exception on exhaustion" — preserved by failsafe (it throws something).

So **this library is internally safe**, but downstream consumers of `dropwizard-actors` may catch `com.github.rholder.retry.RetryException` directly. Removing the guava-retrying dependency removes that class from the classpath, turning any such `catch (RetryException)` into a compile error in the consumer. That is a transitive API break.

**How to avoid:**

- Do NOT change the `throws Exception` signature of `RetryStrategy.execute` — it already abstracts the exhaustion type.
- In the migration notes / changelog, call out that the thrown exception type on exhaustion changes from `RetryException` to `FailsafeException` (checked exceptions) or the raw original `RuntimeException`/`Error`. Consumers catching `RetryException` must switch to catching `Throwable`/`Exception` or `FailsafeException`.
- If a consumer-facing compatibility shim is required (out of current scope per PROJECT.md), wrap failsafe's throw in a library-specific exception — but the project scope says "migration only, no new features", so prefer documenting the change.

**Warning signs:**

- A consumer's `catch (com.github.rholder.retry.RetryException)` failing to compile after the dependency is removed.
- Tests in this repo that assert on a specific exception class on exhaustion (none exist today — keep it that way; assert on cause or on attempt count, not on the wrapper type).
- `Handler.handleDelivery`'s `errorCheckFunction` (caller-supplied) suddenly classifying exhaustion differently because the throwable's class changed.

**Phase to address:**
Implementation phase (keep `throws Exception`). Release phase (document the transitive exception-type change in the changelog / migration notes).

---

### Pitfall 5: Accidentally using failsafe's async API breaks the synchronous (thread-sleep) contract

**What goes wrong:**
guava-retrying with `BlockStrategies.threadSleepStrategy()` blocks the calling thread synchronously — `retryer.call()` returns only after all retries and sleeps complete, on the same thread. failsafe offers both synchronous (`run`/`get`) and async (`runAsync`/`getAsync`/`getStageAsync`) entry points. Using an async entry point returns a `CompletableFuture` immediately, breaking the contract that `execute()` blocks until completion. The RabbitMQ `Handler.handleDelivery` would ack/reject based on a future that hasn't resolved, causing premature acks and lost messages.

**Why it happens:**
failsafe's `FailsafeExecutor` exposes `run(CheckedRunnable)`, `get(CheckedSupplier)`, `runAsync`, `getAsync`, `getStageAsync` side by side (`FailsafeExecutor.java:111-264`). The async variants return `CompletableFuture<T>` and execute on a `ForkJoinPool` by default. A migrator skimming the docs could grab `getAsync` thinking it's the "get" method.

The current contract: `RetryStrategy.execute` returns `boolean` synchronously and `Handler.handleDelivery` uses the return value immediately to decide `basicAck` vs `basicReject` (`Handler.java:101-105`). An async call would make `execute` return before the retry loop finishes.

**How to avoid:**

- Use `Failsafe.with(retryPolicy).get(supplier)` (synchronous) — never `getAsync`/`getStageAsync`/`runAsync`.
- `RetryStrategy.execute` must keep its `boolean execute(Callable<Boolean>) throws Exception` signature and internally call the synchronous `get`. The `Callable<Boolean>` maps to a `CheckedSupplier<Boolean>`.
- Add a code-review check: the only `Failsafe.with(...).` call in the new code must end in `.get(` or `.run(`, never `*Async`.

**Warning signs:**

- A diff where `execute()` returns a `CompletableFuture` or where `Failsafe.with(...).getAsync(` appears.
- `Handler.handleDelivery` acking before the message handler's retry loop has finished (messages vanishing).
- Tests that use `.toCompletableFuture().join()` to paper over async return types — that's a smell that the wrong API was chosen.

**Phase to address:**
Implementation phase. Code review must verify synchronous entry point.

---

### Pitfall 6: `isRetriable` empty-set semantics — "retry on everything" vs "retry on nothing"

**What goes wrong:**
`CommonUtils.isRetriable(set, exception)` returns `true` when the set is **empty/null** (`CommonUtils.java:42-46`): `isEmpty(retriableExceptions) || set.contains(simpleName)`. So an empty `retriableExceptions` set means "retry on ALL exceptions". Migrating this predicate to failsafe's `handleIf` preserves the predicate, but the interaction with failsafe's default-handling (Pitfall 2) must be re-checked: when the set is empty, the predicate returns `true` for every exception, so `handleIf` retries on every exception — matching guava. The trap is the opposite: a non-empty set that does NOT contain the thrown exception's simple name returns `false`, and in failsafe (with `exceptionsChecked=true`) that exception is NOT retried and is thrown immediately — which is also what guava does. So the predicate itself is faithful. The pitfall is **assuming** the empty-set case needs special handling when it does not.

**Why it happens:**
The empty-set-means-retry-all behavior is non-obvious and easy to "fix" during migration by adding a guard that breaks it.

**How to avoid:**

- Map the predicate verbatim: `handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))`. Do not add an empty-set short-circuit.
- Add a test: empty `retriableExceptions` + a thrown `IOException` → retries (asserts the empty-set-means-all contract survives migration).
- Add a test: `retriableExceptions = {"IOException"}` + a thrown `IllegalStateException` → does NOT retry, throws immediately.

**Warning signs:**

- A migration diff that "simplifies" `isRetriable` or adds `if (isEmpty(set)) return false`.
- Tests that only cover the non-empty-set case.

**Phase to address:**
Implementation + verification phases.

---

### Pitfall 7: Incremental wait strategy has no direct failsafe equivalent

**What goes wrong:**
guava's `WaitStrategies.incrementingWait(initial, unit, increment, unit)` produces `initial + (attemptNumber - 1) * increment` per sleep. failsafe has `withDelay` (fixed), `withBackoff` (exponential), and `withDelay(min, max, random)` — but **no built-in linear/incremental wait**. A migrator may reach for `withBackoff` with `factor=1`, but failsafe rejects `delayFactor <= 1` (`RetryPolicyBuilder.java:274-275`). The two `IncrementalWaitRetryStrategy` impls need a custom delay function.

**Why it happens:**
failsafe's delay model is fixed/random/exponential/computed. Linear increment is the one shape without a one-liner.

**How to avoid:**
Use failsafe's computed-delay API: `withDelayFn` / the `ContextualSupplier<R, Duration>` delay function on `RetryPolicyBuilder`, computing `initial + (attemptCount) * increment` from the `ExecutionContext`. Verify the attempt-count basis matches guava's `initial + (attemptNumber - 1) * increment` (guava's first sleep, after attempt 1 fails, = `initial + 0*increment = initial`).

Alternatively, keep it lazy: `withDelay(Duration.ofMillis(initial))` gives a fixed delay equal to the guava *first* sleep, but that drops the increment — not equivalent. Use the computed delay function.

**Warning signs:**

- A diff using `withBackoff(initial, max, 1.0)` — failsafe throws at build time (`delayFactor must be > 1`), so this fails loudly, not silently. Good.
- A diff using `withDelay(initial)` for the incremental strategies — silently drops the increment (messages retry at a flat rate, not increasing). This is the silent bug.
- `CountLimitedIncrementalWaitRetryStrategy` / `TimeLimitedIncrementalWaitRetryStrategy` tests not asserting the increasing sleep sequence.

**Phase to address:**
Implementation phase (the two incremental strategies). Verification phase: timing-parity test asserting `initial, initial+inc, initial+2*inc, ...`.

---

### Pitfall 8: `stopAfterDelay` (time-limited) semantics — wall-clock vs attempt-bounded

**What goes wrong:**
guava `StopStrategies.stopAfterDelay(maxTime, unit)` stops when `delaySinceFirstAttempt >= maxTime`. failsafe `withMaxDuration(maxDuration)` stops when elapsed time exceeds maxDuration. The semantics are equivalent (both wall-clock from first attempt), but failsafe's `withMaxDuration` does NOT disable `withMaxRetries` (default 2 retries / 3 attempts) — see `RetryPolicyBuilder.java:460-462`. So a time-limited strategy that previously could retry indefinitely within the time window now silently caps at 3 attempts unless `withMaxRetries(-1)` is also set.

**Why it happens:**
guava's `stopAfterDelay` is the ONLY stop condition (it replaces the default `neverStop`). failsafe composes `maxDuration` AND `maxRetries` — both apply. The default `maxRetries=2` is still active.

**How to avoid:**
For the two `TimeLimited*RetryStrategy` impls, set BOTH:

```java
RetryPolicy.builder()
    .withMaxDuration(Duration.ofMillis(config.getMaxTime().toMilliseconds()))
    .withMaxRetries(-1)   // disable the default 2-retry cap; let maxDuration govern
    ...
```

Verify the original guava behavior: `stopAfterDelay` alone, with no attempt cap, retries until time runs out. The failsafe equivalent must disable the attempt cap to match.

**Warning signs:**

- A time-limited strategy that stops after 3 attempts even when `maxTime` hasn't elapsed.
- Tests with a large `maxTime` and a fast-failing callable that stop at 3 attempts instead of running until timeout.

**Phase to address:**
Implementation phase (the two `TimeLimited*` strategies). Verification phase: test that a callable failing instantly with a 10s `maxTime` retries more than 3 times (asserts the retry cap is disabled).

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
| ---------- | ------------------- | ---------------- | ----------------- |
| Map exponential `multiplier` → failsafe `delay` 1:1 (Pitfall 1) | One-line per strategy | Retry schedule halved; throughput/time-to-dead-letter drifts | Never — either fix the factor or document with `ponytail:` and a timing test |
| Use `withMaxRetries` instead of `withMaxAttempts` (Pitfall 3) | Reads more naturally | +1 attempt per strategy | Never for this migration |
| Skip timing-parity tests | Faster migration | Silent schedule drift ships to production | Never — the whole point is behavioral parity |
| Keep `multipier` typo in config field | Config compatibility | Minor confusion | Always (config-compatibility constraint in PROJECT.md) |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
| ------------- | ---------------- | ------------------ |
| Dropwizard 5.x BOM | Adding failsafe without checking it conflicts with Dropwizard's managed deps | failsafe 3.3.2 has no transitive deps that overlap Dropwizard's BOM (it's self-contained) — but run `mvn dependency:tree` after adding to confirm no version overrides |
| `com.github.rholder:guava-retrying` removal | Leaving the property `<guava-retrying.version>` and import usages behind | Grep for `com.github.rholder` and `guava-retrying` after migration; remove the property and dependency |
| Guava itself | Assuming removing guava-retrying removes Guava | Guava is still a direct dependency (`<guava.version>`); guava-retrying wraps Guava's `Retryer` but this codebase uses guava-retrying, not Guava's `Retryer` directly. Keep Guava. |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Thread-sleep blocking under high concurrency | Same as before — failsafe sync `get` blocks the calling thread identically to guava's `threadSleepStrategy` | This is the intended behavior (parity); no change needed. Do NOT switch to async to "fix" perceived blocking. | N/A — blocking is the contract |
| Default `ForkJoinPool` if async API is accidentally used (Pitfall 5) | Messages processed out of order, premature acks | Use synchronous `get`/`run` only | Any load — the bug is immediate, not scale-dependent |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| (None specific to this migration) | — | The retry engine swap does not touch auth, input validation, or data boundaries. The `isRetriable` predicate reads exception class simple names from config — already the case pre-migration; no new attack surface. |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Changed retry schedule (Pitfall 1) | Downstream apps see slower retries / later dead-lettering with no config change | Preserve the exact schedule via `delay = 2 * multiplier`, or document the change explicitly in the changelog |
| Changed exhaustion exception type (Pitfall 4) | Downstream apps' `catch (RetryException)` stops compiling | Document in migration notes; keep `throws Exception` on `execute()` |

## "Looks Done But Isn't" Checklist

- [ ] **NoRetryStrategy:** Often mapped to `withMaxAttempts(1)` but missing the `handleIf`/no-predicate analysis — verify it does NOT retry on any exception (Pitfall 2). Check: throw an exception through `NoRetryStrategy`, assert exactly 1 attempt.
- [ ] **Count-limited strategies:** Often assert "retried N times" but not "N total attempts" — verify `withMaxAttempts` not `withMaxRetries` (Pitfall 3). Check: assert callable invoked exactly `maxAttempts` times.
- [ ] **Exponential strategies:** Often assert "delay > 0" but not the exact sequence — verify the curve isn't halved (Pitfall 1). Check: assert exact sleep durations `[2m, 4m, 8m]` for `maxAttempts=4`.
- [ ] **Time-limited strategies:** Often pass with a short `maxTime` that never hits the 3-attempt default cap — verify `withMaxRetries(-1)` is set (Pitfall 8). Check: use a long `maxTime` + instant-fail callable, assert >3 attempts.
- [ ] **Incremental strategies:** Often mapped to `withDelay(initial)` (fixed) by mistake — verify a computed delay function is used (Pitfall 7). Check: assert increasing sleep sequence.
- [ ] **Exception predicate:** Often only tested with a matching exception — verify non-matching exception is NOT retried (Pitfall 6). Check: throw `IllegalStateException` when set is `{"IOException"}`, assert 1 attempt.
- [ ] **Synchronous contract:** Often "works in tests" because the test thread blocks on the future — verify `Failsafe.with(...).get(` not `getAsync` (Pitfall 5). Check: grep for `Async` in the new retry code.
- [ ] **Dependency cleanup:** Often leaves `guava-retrying` property in pom — verify full removal. Check: `grep -r guava-retrying pom.xml` returns nothing.

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
| --------- | --------------- | ---------------- |
| Halved backoff curve (Pitfall 1) | LOW | Change `delay` to `2 * multiplier` in the two exponential strategies; add timing test |
| Inverted default handling (Pitfall 2) | MEDIUM | Audit each `RetryPolicy.builder()` for missing `handleIf`/`withMaxAttempts(1)`; add per-strategy exception tests |
| Off-by-one attempts (Pitfall 3) | LOW | Replace `withMaxRetries` with `withMaxAttempts`; fix the test assertion to exact count |
| Exception type change (Pitfall 4) | MEDIUM | Internal: none. External: publish migration note, bump minor version, keep `throws Exception` |
| Async API misuse (Pitfall 5) | HIGH | If shipped, messages were prematurely acked — replay from dead-letter/queue. Fix: switch to `get`/`run` |
| Incremental wait (Pitfall 7) | LOW | Replace `withDelay` with `withDelayFn` computing `initial + attempt * increment` |
| Time-limited cap (Pitfall 8) | LOW | Add `withMaxRetries(-1)` to the two `TimeLimited*` strategies |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
| --------- | ------------------ | -------------- |
| 1. Halved backoff curve | Implementation | Timing-parity test asserting exact sleep sequence |
| 2. Inverted default handling | Implementation | Exception-through-NoRetryStrategy test asserts 1 attempt; grep for bare `RetryPolicy.builder()` |
| 3. Off-by-one attempts | Implementation | Count-limited test asserts callable invoked exactly `maxAttempts` times |
| 4. Exception type change | Implementation (keep `throws Exception`) + Release (changelog) | `grep -r RetryException src/` returns 0; downstream migration note published |
| 5. Async API misuse | Implementation + code review | Grep new retry code for `Async` returns 0 |
| 6. Empty-set predicate semantics | Implementation | Empty-set test retries; non-matching-set test does not retry |
| 7. Incremental wait | Implementation | Incremental test asserts increasing sleep sequence |
| 8. Time-limited retry cap | Implementation | Time-limited test with long `maxTime` asserts >3 attempts |

## Sources

- guava-retrying 2.0.0 source jar (`~/.m2/repository/com/github/rholder/guava-retrying/2.0.0/guava-retrying-2.0.0-sources.jar`): `Retryer.java`, `RetryerBuilder.java`, `WaitStrategies.java` (`ExponentialWaitStrategy`), `RetryException.java`
- failsafe 3.3.2 source jar (`~/.m2/repository/dev/failsafe/failsafe/3.3.2/failsafe-3.3.2-sources.jar`): `SyncExecutionImpl.java`, `spi/FailurePolicy.java`, `FailurePolicyBuilder.java`, `RetryPolicyBuilder.java`, `internal/RetryPolicyExecutor.java`, `FailsafeException.java`, `FailsafeExecutor.java`
- failsafe.dev retry docs (<https://failsafe.dev/retry/>) — max attempts, max duration, delays, failure handling
- Codebase: `RetryStrategy.java`, `retry/impl/*` (7 strategies), `retry/config/RetryConfig.java`, `retry/config/CountLimitedExponentialWaitRetryConfig.java`, `utils/CommonUtils.java`, `base/Handler.java` (sole call site), `pom.xml`

---
*Pitfalls research for: guava-retrying → failsafe.dev retry-engine migration*
*Researched: 2025-09-19*
