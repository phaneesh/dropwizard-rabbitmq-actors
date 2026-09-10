# Feature Research

**Domain:** Message-processing retry library (guava-retrying → failsafe.dev migration)
**Researched:** 2025-09-19
**Confidence:** HIGH

## Feature Landscape

### Table Stakes (Users Expect These)

Behaviors that existing consumers depend on. Breaking any of these silently changes
message-processing semantics (ack/reject/dead-letter routing). These MUST be
behaviorally identical post-migration.

| Feature | Why Expected | Complexity | Notes |
| --- | --- | --- | --- |
| Stop after N attempts (`stopAfterAttempt`) | Count-limited strategies cap retries; consumers tune `maxAttempts` per queue SLA | LOW | guava: `StopStrategies.stopAfterAttempt`. failsafe: `RetryPolicy.withMaxAttempts(n)`. **Off-by-one risk**: guava counts the *first* attempt as attempt 1; failsafe's `maxAttempts` is total attempts including the first. Verify N maps to N, not N+1. |
| Stop after max delay (`stopAfterDelay`) | Time-limited strategies bound total retry window | LOW | guava: `StopStrategies.stopAfterDelay(ms)`. failsafe: `RetryPolicy.withMaxDuration(Duration)`. Semantics: stop *before* next attempt if elapsed > max. Confirm boundary (>= vs >). |
| Exponential backoff wait | `exponentialWait(multiplier, maxTimeBetweenRetries)` | MEDIUM | guava: `multiplier * 2^(attempt-1)` ms, capped at `maxTimeBetweenRetries`. failsafe: `withBackoff(initialDelay, maxDelay, factor)`. **Formula parity**: guava's `multiplier` is the *initial* sleep in ms and grows `multiplier * 2^(n-1)`; failsafe's `factor` is a double multiplier. Must derive failsafe params so the sleep sequence matches exactly. |
| Fixed wait | `fixedWait(duration)` | LOW | guava: `WaitStrategies.fixedWait`. failsafe: `withDelay(Duration)`. Trivial 1:1. |
| Incremental wait | `incrementingWait(initial, increment)` | MEDIUM | guava: `initial + (attempt-1)*increment` ms. failsafe has no built-in incremental; must use `withDelay(Function<Integer,Duration>)` computing `initial + (n-1)*increment`. Verify attempt indexing. |
| Exception-predicate retry decision | `retryIfException` + `CommonUtils.isRetriable` | MEDIUM | guava: `retryIfException(predicate)`. failsafe: `handleIf(e -> predicate)`. Predicate logic: retry if `retriableExceptions` set is empty OR exception class simple-name is in the set. **Must preserve**: empty set = retry on ALL exceptions; null exception = no retry. |
| Blocking (thread-sleep) execution | `BlockStrategies.threadSleepStrategy()` | LOW | guava blocks the calling thread between attempts via `Thread.sleep`. failsafe's synchronous `run`/`get` also blocks the calling thread by default. **No async executor** — preserve synchronous blocking. |
| No-retry (single attempt) | `NoRetryStrategy` = `stopAfterAttempt(1)` | LOW | guava: stopAfterAttempt(1), no wait, no exception predicate. failsafe: `withMaxAttempts(1)`. Callable runs exactly once; any exception propagates immediately. |
| Callable returns Boolean (success signal) | `Handler.handleDelivery` acks on `true`, rejects on `false` | LOW | `retryer.call(callable)` returns the `Callable<Boolean>` result. failsafe: `Failsafe.with(policy).get(callable)` returns the result. **Return value is NOT a retry trigger** (see Anti-Features). |
| Exception propagation on exhaustion | When all attempts fail, an exception escapes `execute()` to `Handler`'s `catch (Throwable)` | MEDIUM | guava throws `RetryException` wrapping the last cause. failsafe throws the **last exception directly** (no wrapper). **Codebase does NOT reference `RetryException`** (grep confirmed zero hits) — consumers catch `Throwable`, so the wrapper-type change is safe. But the *cause* must still be the original processing exception, not a generic exhaustion marker. |

### Differentiators (Competitive Advantage)

Not required for parity, but failsafe.dev makes some available cheaply. Do NOT
add unless explicitly requested — migration scope only.

| Feature | Value Proposition | Complexity | Notes |
| --- | --- | --- | --- |
| Per-attempt listener / hooks | Observability into each retry attempt (logging, metrics) | LOW | failsafe `onFailure(e, ctx)`. guava had `RetryListener`. Not currently used. Defer. |
| Jitter on backoff | Reduce thundering-herd on shared downstream | LOW | failsafe `withJitter(factor)`. Not in current configs. Defer. |
| Max retries vs max attempts clarity | failsafe's API is less ambiguous than guava's | LOW | Cosmetic; no behavioral change. |

### Anti-Features (Commonly Requested, Often Problematic)

Deliberately NOT added. Adding these changes the execution model and risks
breaking the synchronous ack/reject contract in `Handler`.

| Feature | Why Requested | Why Problematic | Alternative |
| --- | --- | --- | --- |
| Async / non-blocking execution | "failsafe supports async via CompletableFuture" | `Handler.handleDelivery` runs on the RabbitMQ consumer thread and synchronously acks/rejects based on the `Boolean` result. Async execution decouples the result from the ack, breaking delivery semantics and ordering. | Keep synchronous `Failsafe.with(policy).get(callable)`. |
| Retry on `false` return value (`retryIfResult`) | "Some handlers return false on transient failure" | Current code uses `retryIfException` only — `false` means *deliberately reject this message*, not "try again". Adding result-based retry would re-process messages that handlers intentionally rejected, causing duplicates. | Keep exception-only retry. If a handler wants retry, it must throw. |
| Circuit breaker | "failsafe bundles CircuitBreaker" | Out of scope; changes failure semantics from per-message retry to cross-message breaking. No current consumer expects it. | None. Migration only. |
| New retry strategies / RetryType values | "Add jittered-exponential" | Breaks `RetryType` enum and `RetryConfig` JSON subtypes — config compatibility constraint. | None. |
| Changing public API of `RetryStrategy` / `RetryStrategyFactory` | "Cleaner failsafe-native API" | Downstream consumers depend on `execute(Callable<Boolean>)` contract. | Keep `RetryStrategy` shape; swap only the internal `Retryer<Boolean>` for a failsafe `RetryPolicy<Boolean>`. |
| Wrapping exhaustion in a custom exception | "Consistent exception type" | Consumers catch `Throwable`; a wrapper adds no value and hides the real cause. | Let failsafe's native exception (the last cause) propagate. |

## Feature Dependencies

```
[Exception-predicate retry] ──requires──> [CommonUtils.isRetriable semantics preserved]
        │
        └──requires──> [Empty retriableExceptions set = retry-on-all]

[Count-limited stop] ──requires──> [Attempt-counting parity (off-by-one check)]
[Time-limited stop] ──requires──> [Delay-boundary parity (>= vs >)]

[Exponential wait] ──requires──> [Formula parity: multiplier semantics]
[Incremental wait] ──requires──> [Custom delay function in failsafe]

[Blocking execution] ──conflicts──> [Async execution model]
[Boolean return as success signal] ──conflicts──> [retryIfResult / result-based retry]
```

### Dependency Notes

- **Exception-predicate requires `isRetriable` semantics:** The predicate `CommonUtils.isRetriable(set, e)` returns true when the set is empty (retry all) OR the exception's class simple-name is in the set. This two-mode behavior must be preserved verbatim in the failsafe `handleIf` predicate.
- **Count-limited requires attempt-counting parity:** guava's `stopAfterAttempt(N)` stops when attempt number > N (i.e., N total attempts). failsafe's `withMaxAttempts(N)` is also total attempts including the first. Confirm by test, not by docs.
- **Exponential wait requires formula parity:** guava `exponentialWait(multiplier, max)` sleeps `min(multiplier * 2^(attempt-1), max)` ms. failsafe `withBackoff(initial, max, factor)` sleeps `min(initial * factor^(attempt-1), max)`. To match: `initial = multiplier`, `factor = 2.0`, `max = maxTimeBetweenRetries`. Verify the growth base.
- **Incremental wait requires a custom function:** failsafe has no built-in incremental strategy. Use `withDelay(ctx -> Duration.ofMillis(initial + ctx.getAttemptCount() * increment))` (or equivalent). Verify `getAttemptCount()` is 0-based or 1-based.
- **Blocking conflicts with async:** The synchronous ack/reject in `Handler.handleDelivery` depends on `execute()` blocking until the final result. Any async path breaks this.
- **Boolean-return-as-signal conflicts with result-retry:** `false` from the callable means "reject this message," not "retry." `retryIfResult` would convert deliberate rejects into retries.

## MVP Definition

### Launch With (v1)

The migration's MVP = behavioral parity. Every item below is essential because an
existing consumer depends on it.

- [x] Stop-after-attempt parity (count-limited: 3 strategies)
- [x] Stop-after-delay parity (time-limited: 3 strategies)
- [x] Exponential-wait formula parity (2 strategies)
- [x] Fixed-wait parity (2 strategies)
- [x] Incremental-wait parity via custom delay function (2 strategies)
- [x] No-retry single-attempt parity (1 strategy)
- [x] Exception-predicate parity (`isRetriable`: empty-set = retry-all, else simple-name match)
- [x] Synchronous blocking execution (no async)
- [x] `Boolean` return value propagated as success signal (no result-based retry)
- [x] Exception propagation on exhaustion (last cause escapes to `Handler`'s `catch (Throwable)`)
- [x] `RetryStrategy.execute(Callable<Boolean>)` public signature unchanged
- [x] All 7 `RetryType` enum values + JSON config subtypes unchanged

### Add After Validation (v1.x)

Only after parity is proven by tests.

- [ ] Per-attempt listener for metrics/logging (if observability need arises)
- [ ] Jitter option on exponential backoff (if thundering-herd observed)

### Future Consideration (v2+)

- [ ] New RetryType values (requires config-compat plan)
- [ ] Async execution model (requires rethinking Handler ack/reject contract)

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
| --- | --- | --- | --- |
| Stop-after-attempt parity | HIGH | LOW | P1 |
| Stop-after-delay parity | HIGH | LOW | P1 |
| Exponential-wait formula parity | HIGH | MEDIUM | P1 |
| Fixed-wait parity | HIGH | LOW | P1 |
| Incremental-wait parity | HIGH | MEDIUM | P1 |
| No-retry parity | HIGH | LOW | P1 |
| Exception-predicate parity | HIGH | MEDIUM | P1 |
| Blocking execution | HIGH | LOW | P1 |
| Boolean return as signal | HIGH | LOW | P1 |
| Exhaustion exception propagation | HIGH | LOW | P1 |
| Public API unchanged | HIGH | LOW | P1 |
| Per-attempt listener | LOW | LOW | P3 |
| Jitter | LOW | LOW | P3 |
| Async execution | LOW | HIGH | P3 (anti-feature) |
| Circuit breaker | LOW | MEDIUM | P3 (anti-feature) |
| retryIfResult | LOW | LOW | P3 (anti-feature) |

**Priority key:**

- P1: Must have for launch (parity)
- P2: Should have, add when possible
- P3: Nice to have / explicitly deferred (anti-features)

## Competitor Feature Analysis

| Feature | guava-retrying (current) | failsafe.dev (target) | Our Approach |
| --- | --- | --- | --- |
| Stop after attempts | `StopStrategies.stopAfterAttempt(n)` | `RetryPolicy.withMaxAttempts(n)` | Map directly; verify off-by-one via test |
| Stop after delay | `StopStrategies.stopAfterDelay(ms)` | `RetryPolicy.withMaxDuration(d)` | Map directly; verify boundary |
| Exponential wait | `WaitStrategies.exponentialWait(mult, max, unit)` | `withBackoff(initial, max, factor)` | `initial=mult`, `factor=2.0`, `max=maxTimeBetweenRetries`; verify formula |
| Fixed wait | `WaitStrategies.fixedWait(t, unit)` | `withDelay(d)` | Direct 1:1 |
| Incremental wait | `WaitStrategies.incrementingWait(init, inc)` | `withDelay(Function)` | Custom function: `init + n*inc` |
| Exception predicate | `retryIfException(pred)` | `handleIf(pred)` | Wrap `CommonUtils.isRetriable` |
| Blocking | `BlockStrategies.threadSleepStrategy()` | Synchronous `get()` blocks by default | Use synchronous API only |
| No retry | `stopAfterAttempt(1)` | `withMaxAttempts(1)` | Direct |
| Exhaustion exception | Throws `RetryException` wrapping cause | Throws last cause directly | Safe: consumers catch `Throwable`, no `RetryException` refs in codebase |
| Result-based retry | `retryIfResult` (available, **unused**) | `handleResultIf` (available) | **Do not use** — `false` = reject, not retry |
| Async | Not used | `getAsync()` / CompletableFuture | **Do not use** — breaks ack/reject contract |

## Sources

- `src/main/java/io/appform/dropwizard/actors/retry/RetryStrategy.java` — base class, `retryer.call(callable)`, `Boolean` return
- `src/main/java/io/appform/dropwizard/actors/retry/impl/*.java` — all 7 strategy impls (RetryerBuilder usage)
- `src/main/java/io/appform/dropwizard/actors/retry/config/*.java` — config subtypes + `retriableExceptions` set
- `src/main/java/io/appform/dropwizard/actors/utils/CommonUtils.java` — `isRetriable` two-mode predicate
- `src/main/java/io/appform/dropwizard/actors/base/Handler.java` — sole consumer: `execute()` → ack on true / reject on false / `catch (Throwable)` on exception
- Grep for `RetryException` / `retryIfResult` — zero hits in codebase (confirms neither is relied upon)
- `.planning/PROJECT.md` — migration scope, constraints, out-of-scope

---
*Feature research for: guava-retrying → failsafe.dev retry library migration*
*Researched: 2025-09-19*
