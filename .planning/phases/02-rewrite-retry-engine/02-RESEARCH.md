# Phase 2: Rewrite retry engine (base + all seven impls, atomic) - Research

**Researched:** 2026-09-10
**Domain:** Java retry-engine migration (guava-retrying → failsafe 3.3.2)
**Confidence:** HIGH

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Change the field type from `Retryer<Boolean> retryer` to `RetryPolicy<Boolean> retryPolicy`. Keep the field `private final`. (ENG-01)
- **D-02:** `execute(Callable<Boolean>)` body changes from `retryer.call(callable)` to `Failsafe.with(retryPolicy).get(callable)`. This is the synchronous blocking call — `.get()` blocks the calling thread exactly like guava's `retryer.call()`. Never use `*Async` APIs. (ENG-02, PAR-08)
- **D-03:** The `execute(Callable<Boolean>) throws Exception` public signature is byte-identical — no parameter, return type, or throws-clause change. (ENG-03)
- **D-04:** Constructor stays `protected RetryStrategy(RetryPolicy<Boolean> retryPolicy)`. The parameter type change is internal (impls pass the policy via `super(...)`); no external caller constructs `RetryStrategy` directly except the 7 impls and the factory.
- **D-05:** Use `withBackoff(long baseDelay, long factor)` where `baseDelay = 2 * config.getMultipier()` (in ms) and `factor = 2.0`. This corrects the halved-curve pitfall: guava computes `multiplier * 2^attempt`; failsafe computes `baseDelay * factor^(attempt-1)`. Setting `baseDelay = 2 * multiplier` makes attempt-1 sleep = `2 * multiplier`, matching guava. (PAR-03)
- **D-06:** Apply `withMaxDelay(config.getMaxTimeBetweenRetries().toMilliseconds())` to cap each sleep at the configured maximum, matching guava's `exponentialWait(multiplier, maxTimeBetweenRetries, MS)` second parameter. (PAR-03)
- **D-07:** Preserve the `getMultipier()` typo — read it as "exponential base / initial delay". Do not rename the config field (config-compatibility constraint). (carried from PROJECT.md constraints)
- **D-08:** Use `withDelayFn(attempt -> initial + (attempt - 1) * increment)` where `initial = config.getInitialWaitTime().toMilliseconds()` and `increment = config.getWaitIncrement().toMilliseconds()`. The `attempt` parameter in failsafe's delay function is 1-based (attempt 1 is the first execution). This produces: attempt 1 → 0 delay (no delay before first try), attempt 2 → `initial`, attempt 3 → `initial + increment`, etc. — matching guava's `incrementingWait(initial, increment)` sequence. (PAR-04)
- **D-09:** **Research flag (must confirm during planning):** `ExecutionContext.getAttemptCount()` basis (0 vs 1) must be empirically confirmed against failsafe 3.3.2 source. The delay function receives an `ExecutionContext`; if `getAttemptCount()` is 0-based at the first retry point, the formula adjusts to `initial + attempt * increment`. Planner must verify against the failsafe jar in `~/.m2` before finalizing. (PAR-04, open risk from research SUMMARY.md)
- **D-10:** Use `withDelay(config.getWaitTime().toMilliseconds())` — trivial 1:1 mapping from guava's `fixedWait(ms, MS)`. (PAR-05)
- **D-11:** Use `withMaxAttempts(config.getMaxAttempts())` — never `withMaxRetries`. guava's `stopAfterAttempt(n)` ≡ failsafe's `withMaxAttempts(n)` (both = n total attempts). `withMaxRetries(n)` = n+1 attempts (off-by-one pitfall). (PAR-01)
- **D-12:** Use `withMaxDuration(config.getMaxTime().toMilliseconds(), TimeUnit.MILLISECONDS)` AND `withMaxRetries(-1)` to disable failsafe's default 3-attempt cap. Without `withMaxRetries(-1)`, failsafe silently stops after 3 attempts even if the duration hasn't elapsed. (PAR-02)
- **D-13:** Use `withMaxAttempts(1)` explicitly. Do NOT rely on a bare `RetryPolicy.builder().build()` — failsafe's default is retry-up-to-3-times-on-any-exception, which would silently break NoRetryStrategy. No exception predicate needed (NoRetryStrategy has none in guava). (PAR-06)
- **D-14:** Map guava's `retryIfException(predicate)` to failsafe's `handleIf(predicate)`, where the predicate is `exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception)`. The `CommonUtils.isRetriable` logic is unchanged: empty/null set = retry-all (returns true), non-empty = retry only if exception class simple name is in the set. (PAR-07)
- **D-15:** `handleIf` replaces `retryIfException` — same semantics (predicate decides retry on exception). NoRetryStrategy gets no `handleIf` (matches guava: no predicate). (PAR-07)
- **D-16:** The `Callable<Boolean>` return value is a success signal, not a retry trigger. `true` = success (ack), `false` = reject (no retry). failsafe does NOT retry on `false` return by default — only on exceptions. This matches guava's behavior (guava's `Retryer.call()` does not retry on return value, only on exception predicate). No `handleResult(false)` needed. (PAR-09)
- **D-17:** On exhaustion, failsafe throws `FailsafeException` (unchecked, wrapping the last cause) or the raw original throwable. `execute()` declares `throws Exception`, so the raw throwable propagates to `Handler.handleDelivery()`'s `catch (Throwable)`. The codebase has zero `RetryException` references internally (verified in research), so no internal catch-site breaks. The exception-type change is documented in Phase 4, not shimmed here. (out of scope to shim; DOC-01 covers it)
- **D-18:** Do not catch/wrap `FailsafeException` inside `execute()` — let it propagate raw. Adding a try/catch to re-wrap as a checked exception would change the exception contract and add unrequested complexity. The `throws Exception` declaration already covers checked propagation; unchecked `FailsafeException` propagates naturally.

### Claude's Discretion

- Whether to add a brief javadoc note on `RetryStrategy` explaining the failsafe swap. Not required; planner can decide.
- Exact import ordering / static-import style for failsafe APIs. Follow existing codebase conventions.
- Whether to inline the delay function lambda or extract a static helper for incremental wait. Inline is simpler; planner decides if readability suffers.

### Deferred Ideas (OUT OF SCOPE)

None - discussion stayed within phase scope. The phase is a like-for-like engine swap with parity construction; no new capabilities were proposed. Observability hooks (onFailure listeners), jitter, and async execution are explicitly out of scope (v2 requirements, tracked in REQUIREMENTS.md).
</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
| -------- | ---------------------- | ----------------------------------------------- |
| ENG-01 | RetryStrategy base class holds `RetryPolicy<Boolean>` instead of `Retryer<Boolean>` | D-01 confirmed: `RetryPolicy<R>` is an interface in `dev.failsafe`, `RetryPolicyBuilder.build()` returns `RetryPolicy<R>`. Field type swap is a direct type change. |
| ENG-02 | RetryStrategy.execute runs via `Failsafe.with(policy).get()` (synchronous, blocks calling thread) | D-02 confirmed: `Failsafe.with(RetryPolicy)` returns `FailsafeExecutor<R>`, `.get(CheckedSupplier<T>)` is synchronous blocking. `Callable<Boolean>` → `CheckedSupplier<Boolean>` via lambda adapter `() -> callable.call()`. |
| ENG-03 | Public API of RetryStrategy (execute signature, throws Exception) unchanged | D-03 confirmed: `execute(Callable<Boolean>) throws Exception` signature is byte-identical. Only the body changes. |
| ENG-04 | All 7 strategy impls rewritten from RetryerBuilder to RetryPolicy.builder() | All 7 impls verified. Each constructor builds a `RetryerBuilder` chain → will build a `RetryPolicy.builder()` chain. Constructor signatures unchanged (each takes its typed config). |
| PAR-01 | Count-limited strategies stop after exactly maxAttempts (withMaxAttempts, never withMaxRetries) | D-11 confirmed: `withMaxAttempts(n)` sets `config.maxRetries = n - 1` (n total attempts). `withMaxRetries(n)` = n+1 attempts (off-by-one). guava's `stopAfterAttempt(n)` = n total attempts = `withMaxAttempts(n)`. |
| PAR-02 | Time-limited strategies stop after maxDuration AND disable failsafe's default 3-attempt cap (withMaxRetries(-1)) | D-12 confirmed: `withMaxDuration` does NOT disable default `maxRetries=2` (3 attempts). Must add `withMaxRetries(-1)` to disable the cap. Source: `RetryPolicyBuilder` constructor sets `DEFAULT_MAX_RETRIES = 2`, and `withMaxDuration` javadoc explicitly says "This setting will not disable max retries, which are still 2 by default." |
| PAR-03 | Exponential wait strategies produce identical sleep sequences (delay = 2 * multiplier correction) | D-05/D-06 confirmed via source analysis. Guava: `multiplier * 2^attemptNumber` (attemptNumber starts at 1). Failsafe: `baseDelay * factor^(attempt-1)` via `adjustForBackoff`. Setting `baseDelay = 2 * multiplier, factor = 2.0` produces: 2*mult, 4*mult, 8*mult — matching guava's 2*mult, 4*mult, 8*mult. |
| PAR-04 | Incremental wait strategies produce identical sleep sequences (initial + (attempt-1)*increment) | D-08/D-09 confirmed: `ExecutionContext.getAttemptCount()` returns 0 before first attempt, increments to 1 after first attempt is recorded. At first retry point (after attempt 1 fails), `getAttemptCount()=1`. Formula `initial + (getAttemptCount() - 1) * increment` = `initial + 0 = initial`. Matches guava's `initial + increment * (attemptNumber - 1)`. |
| PAR-05 | Fixed wait strategies produce identical delays | D-10 confirmed: `withDelay(Duration.ofMillis(config.getWaitTime().toMilliseconds()))` is a trivial 1:1 mapping. |
| PAR-06 | NoRetryStrategy executes exactly once (withMaxAttempts(1), no retry-by-default) | D-13 confirmed: `RetryPolicyBuilder` default is `maxRetries=2` (3 attempts). Must explicitly set `withMaxAttempts(1)`. No `handleIf` needed (matches guava: no predicate). |
| PAR-07 | Exception predicate (handleIf) filters retriable exceptions identically to retryIfException | D-14/D-15 confirmed: `handleIf(CheckedPredicate<Throwable>)` sets `exceptionsChecked=true`, adds predicate to `failureConditions`. `FailurePolicy.isFailure()` uses the predicate: if it returns `false`, the exception is NOT a failure → no retry. Matches guava's `retryIfException(predicate)`. |
| PAR-08 | Synchronous blocking execution preserved (never *Async APIs) | D-02 confirmed: `FailsafeExecutor.get(CheckedSupplier)` is synchronous. The `RetryPolicyExecutor.apply()` loop calls `Thread.sleep()` directly — blocks the calling thread. No `*Async` APIs used. |
| PAR-09 | Boolean return value treated as success signal (false = reject, not retry) | D-16 confirmed: failsafe only retries on exceptions (via `handleIf`), not on return values. No `handleResult(false)` or `handleResultIf` is used. `false` return = success (no retry). Matches guava behavior. |
</phase_requirements>

## Summary

This phase swaps the retry engine type held by `RetryStrategy` from guava-retrying's `Retryer<Boolean>` to failsafe's `RetryPolicy<Boolean>`, and rewrites all seven strategy impl builder chains from `RetryerBuilder` to `RetryPolicy.builder()`. The change is atomic — the base constructor parameter type changes, so every impl must change in the same commit. There is no partial-migration state that compiles.

All failsafe 3.3.2 API signatures have been verified against the extracted source jar at `/tmp/fsafe_src/`. The five critical pitfalls from the project research (exponential halved curve, inverted default handling, withMaxRetries off-by-one, time-limited 3-attempt cap, exhaustion exception type) are all addressed by the locked decisions D-01 through D-18. The one open research flag (D-09: `getAttemptCount()` basis) is **RESOLVED** — confirmed 1-based after first attempt via source analysis of `ExecutionImpl.recordAttempt()` and `RetryPolicyExecutor.onFailure()`.

**Primary recommendation:** Implement all 8 files (1 base + 7 impls) in a single compile-coherent commit. Use `java.time.Duration` for all failsafe API calls (the Duration-based overloads, not the `long + ChronoUnit` overloads). The `Callable<Boolean>` → `CheckedSupplier<Boolean>` adaptation is a one-line lambda: `() -> callable.call()`.

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
| ------- | ------- | ------- | ------------ |
| dev.failsafe:failsafe | 3.3.2 | Retry policy engine replacing guava-retrying | Already on classpath (Phase 1 complete). Zero transitive deps. Apache 2.0. Java 8+ bytecode. |

### Supporting

| Library | Version | Purpose | When to Use |
| ------- | ------- | ------- | ----------- |
| io.dropwizard:dropwizard-util | (existing) | `Duration` type for config values | All config classes use `io.dropwizard.util.Duration` — call `.toMilliseconds()` to extract long values for failsafe APIs |
| java.time.Duration | (JDK 17) | Failsafe API parameter type | All failsafe `withDelay`, `withBackoff`, `withMaxDuration` overloads take `java.time.Duration` |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
| ---------- | --------- | -------- |
| `java.time.Duration` overloads | `long + ChronoUnit` overloads | Duration overloads are cleaner and avoid TimeUnit import. Both work. Duration is preferred. |
| `withDelayFn(ContextualSupplier)` | `withDelayFnOn(ContextualSupplier, Class)` | `withDelayFn` applies to all exceptions; `withDelayFnOn` only for specific exception types. We want all-exceptions delay. |

**Installation:**

```bash
# Already installed in Phase 1 — no pom changes in this phase
# Verify:
mvn dependency:tree | grep failsafe
# Expected: dev.failsafe:failsafe:jar:3.3.2:compile
```

## Architecture Patterns

### Recommended Project Structure

```
src/main/java/io/appform/dropwizard/actors/retry/
├── RetryStrategy.java           # CHANGES: field type + execute() body
├── RetryStrategyFactory.java    # UNCHANGED
├── RetryType.java                # UNCHANGED
├── config/                       # UNCHANGED (all 7 config classes)
└── impl/                         # CHANGES: all 7 builder chains rewritten
    ├── CountLimitedExponentialWaitRetryStrategy.java
    ├── CountLimitedFixedWaitRetryStrategy.java
    ├── CountLimitedIncrementalWaitRetryStrategy.java
    ├── NoRetryStrategy.java
    ├── TimeLimitedExponentialWaitRetryStrategy.java
    ├── TimeLimitedFixedWaitRetryStrategy.java
    └── TimeLimitedIncrementalWaitRetryStrategy.java
```

### Pattern 1: Base class holds RetryPolicy, executes via Failsafe.with()

**What:** `RetryStrategy` holds a `RetryPolicy<Boolean>` field and executes via `Failsafe.with(retryPolicy).get(supplier)`.
**When to use:** Always — this is the only execution path.
**Example:**

```java
// Source: /tmp/fsafe_src/dev/failsafe/Failsafe.java + FailsafeExecutor.java
package io.appform.dropwizard.actors.retry;

import dev.failsafe.Failsafe;
import dev.failsafe.RetryPolicy;

import java.util.concurrent.Callable;

public abstract class RetryStrategy {
    private final RetryPolicy<Boolean> retryPolicy;

    protected RetryStrategy(RetryPolicy<Boolean> retryPolicy) {
        this.retryPolicy = retryPolicy;
    }

    public boolean execute(Callable<Boolean> callable) throws Exception {
        return Failsafe.with(retryPolicy).get(() -> callable.call());
    }
}
```

**Key detail:** `FailsafeExecutor.get(CheckedSupplier<T>)` takes a `CheckedSupplier<T>` whose `get()` throws `Throwable`. `Callable<Boolean>.call()` throws `Exception`. The lambda `() -> callable.call()` adapts `Callable<Boolean>` → `CheckedSupplier<Boolean>` because `Exception` is a subtype of `Throwable`.

### Pattern 2: Impl builds RetryPolicy in constructor, passes to super()

**What:** Each impl constructor builds a `RetryPolicy<Boolean>` via `RetryPolicy.<Boolean>builder().withXxx(...).build()` and passes it to `super(...)`.
**When to use:** All 7 impls.
**Example (CountLimitedFixedWaitRetryStrategy):**

```java
// Source: /tmp/fsafe_src/dev/failsafe/RetryPolicy.java + RetryPolicyBuilder.java
package io.appform.dropwizard.actors.retry.impl;

import dev.failsafe.RetryPolicy;
import io.appform.dropwizard.actors.retry.RetryStrategy;
import io.appform.dropwizard.actors.retry.config.CountLimitedFixedWaitRetryConfig;
import io.appform.dropwizard.actors.utils.CommonUtils;

import java.time.Duration;

public class CountLimitedFixedWaitRetryStrategy extends RetryStrategy {
    public CountLimitedFixedWaitRetryStrategy(CountLimitedFixedWaitRetryConfig config) {
        super(RetryPolicy.<Boolean>builder()
                .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
                .withMaxAttempts(config.getMaxAttempts())
                .withDelay(Duration.ofMillis(config.getWaitTime().toMilliseconds()))
                .build());
    }
}
```

### Pattern 3: Exponential backoff with 2*multiplier correction

**What:** Use `withBackoff(Duration, Duration, double)` with `baseDelay = 2 * multiplier`, `maxDelay = maxTimeBetweenRetries`, `factor = 2.0`.
**Example (CountLimitedExponentialWaitRetryStrategy):**

```java
// Source: /tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java withBackoff()
super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxAttempts(config.getMaxAttempts())
        .withBackoff(
            Duration.ofMillis(2 * config.getMultipier()),           // baseDelay = 2 * multiplier
            Duration.ofMillis(config.getMaxTimeBetweenRetries().toMilliseconds()),  // maxDelay
            2.0)                                                     // factor
        .build());
```

**Math proof:**

- Guava: `multiplier * 2^attemptNumber` where attemptNumber starts at 1 → sleeps: 2M, 4M, 8M, ...
- Failsafe: `baseDelay * factor^(attempt-1)` via `adjustForBackoff()` → sleeps: base, base*2, base*4, ...
- With `baseDelay = 2*M`: 2M, 4M, 8M, ... ✓ matches

### Pattern 4: Incremental wait with withDelayFn

**What:** Use `withDelayFn(ContextualSupplier)` computing `initial + (getAttemptCount() - 1) * increment`.
**Example (CountLimitedIncrementalWaitRetryStrategy):**

```java
// Source: /tmp/fsafe_src/dev/failsafe/DelayablePolicyBuilder.java withDelayFn()
// Source: /tmp/fsafe_src/dev/failsafe/ExecutionContext.java getAttemptCount()
long initial = config.getInitialWaitTime().toMilliseconds();
long increment = config.getWaitIncrement().toMilliseconds();

super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxAttempts(config.getMaxAttempts())
        .withDelayFn(ctx -> Duration.ofMillis(
            initial + (ctx.getAttemptCount() - 1) * increment))
        .build());
```

**D-09 RESOLUTION:** `ExecutionContext.getAttemptCount()` returns 0 before the first attempt. After `record(result)` is called (which calls `recordAttempt()` → `attempts.incrementAndGet()`), it becomes 1. The delay function is called from `onFailure()` which runs AFTER `record()`. So at the first retry point, `getAttemptCount()=1`. Formula `initial + (1-1)*increment = initial` ✓.

### Pattern 5: Time-limited strategies need withMaxRetries(-1)

**What:** Time-limited strategies must call both `withMaxDuration(...)` AND `withMaxRetries(-1)`.
**Example (TimeLimitedFixedWaitRetryStrategy):**

```java
super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxDuration(Duration.ofMillis(config.getMaxTime().toMilliseconds()))
        .withMaxRetries(-1)  // disable default 3-attempt cap
        .withDelay(Duration.ofMillis(config.getWaitTime().toMilliseconds()))
        .build());
```

### Pattern 6: NoRetryStrategy with withMaxAttempts(1)

**What:** NoRetryStrategy must explicitly set `withMaxAttempts(1)`. No `handleIf` predicate.
**Example:**

```java
super(RetryPolicy.<Boolean>builder()
        .withMaxAttempts(1)
        .build());
```

### Anti-Patterns to Avoid

- **Using `withMaxRetries(n)` instead of `withMaxAttempts(n)`:** Off-by-one. `withMaxRetries(n)` = n+1 total attempts. `withMaxAttempts(n)` = n total attempts. guava's `stopAfterAttempt(n)` ≡ `withMaxAttempts(n)`.
- **Bare `RetryPolicy.builder().build()`:** Defaults to 3 attempts on any exception. NoRetryStrategy would silently retry.
- **Forgetting `withMaxRetries(-1)` on time-limited strategies:** `withMaxDuration` does NOT disable the default `maxRetries=2`. Strategies silently stop after 3 attempts.
- **Using `delay = multiplier` for exponential backoff:** Halves every sleep. Must use `delay = 2 * multiplier`.
- **Adding `handleResult(false)`:** Would make failsafe retry on `false` return values. guava does NOT retry on return values. `false` = reject, not retry.
- **Catching `FailsafeException` inside `execute()`:** Changes the exception contract. Let it propagate raw to `Handler.handleDelivery()`'s `catch (Throwable)`.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
| ------- | ----------- | ----------- | --- |
| Callable → CheckedSupplier adaptation | Wrapper class or anonymous class | Lambda `() -> callable.call()` | One line. `CheckedSupplier.get()` throws `Throwable`, `Callable.call()` throws `Exception` — compatible. |
| Exponential backoff delay computation | Manual `Math.pow(2, attempt) * multiplier` | `withBackoff(Duration, Duration, double)` | Failsafe handles the backoff curve, maxDelay cap, and attempt tracking internally. |
| Incremental wait delay computation | Manual loop or stateful counter | `withDelayFn(ctx -> Duration.ofMillis(...))` | Failsafe provides `ExecutionContext.getAttemptCount()` — no need to track state. |
| Exception filtering | Custom try/catch in execute() | `handleIf(predicate)` | Failsafe's `FailurePolicy.isFailure()` handles the predicate evaluation. |
| Thread blocking | `Thread.sleep()` or `BlockStrategies` | `Failsafe.with(policy).get()` | `.get()` is synchronous by default — blocks the calling thread. |

**Key insight:** Every guava-retrying concept has a direct failsafe equivalent. No custom logic is needed.

## Common Pitfalls

### Pitfall 1: Exponential backoff curve silently halved

**What goes wrong:** Mapping `delay = multiplier` (1:1) halves every sleep because guava uses `multiplier * 2^attempt` (attempt starts at 1) while failsafe uses `delay * factor^(attempt-1)` (first sleep = delay, not delay*2).
**Why it happens:** The two libraries index exponents differently. guava's first sleep is `multiplier * 2^1 = 2*multiplier`. Failsafe's first sleep is `baseDelay * 2^0 = baseDelay`. Setting `baseDelay = multiplier` gives `multiplier` instead of `2*multiplier`.
**How to avoid:** Set `baseDelay = 2 * config.getMultipier()`. Verified: failsafe produces 2M, 4M, 8M — matching guava.
**Warning signs:** Sleep times are half of what they used to be. Tests pass because retries still happen, just faster.

### Pitfall 2: withMaxRetries vs withMaxAttempts off-by-one

**What goes wrong:** `withMaxRetries(n)` = n+1 total attempts. `withMaxAttempts(n)` = n total attempts. Using `withMaxRetries` when you mean `withMaxAttempts` gives one extra attempt.
**Why it happens:** Failsafe has two APIs for the same concept. `maxRetries` counts retries (attempts after the first). `maxAttempts` counts total attempts.
**How to avoid:** Always use `withMaxAttempts(config.getMaxAttempts())`. Never use `withMaxRetries` except for `-1` (disable cap).
**Warning signs:** One extra retry attempt compared to guava.

### Pitfall 3: Time-limited strategies silently cap at 3 attempts

**What goes wrong:** `withMaxDuration(...)` does NOT disable the default `maxRetries=2` (3 total attempts). Time-limited strategies stop after 3 attempts even if the duration hasn't elapsed.
**Why it happens:** `withMaxDuration` and `withMaxRetries` are independent settings. The default `maxRetries=2` is set in the `RetryPolicyBuilder` constructor and is not cleared by `withMaxDuration`.
**How to avoid:** Always add `withMaxRetries(-1)` to time-limited strategies. Source: `RetryPolicyBuilder` javadoc for `withMaxDuration` explicitly says "This setting will not disable max retries, which are still 2 by default."
**Warning signs:** Time-limited strategies stop after 3 attempts regardless of configured duration.

### Pitfall 4: NoRetryStrategy silently retries 3 times

**What goes wrong:** A bare `RetryPolicy.builder().build()` defaults to `maxRetries=2` (3 attempts on any exception). NoRetryStrategy would silently retry.
**Why it happens:** Failsafe's default is retry-on-any-exception-3-times. guava's default (with `stopAfterAttempt(1)`) is no retry.
**How to avoid:** Explicitly set `withMaxAttempts(1)` on NoRetryStrategy.
**Warning signs:** NoRetryStrategy retries on exceptions.

### Pitfall 5: withDelay(Duration.ZERO) throws IllegalArgumentException

**What goes wrong:** `DelayablePolicyBuilder.withDelay(Duration)` asserts `delay.toNanos() > 0`. If `waitTime` is 0, this throws at construction time.
**Why it happens:** Failsafe validates delay > 0. Guava's `fixedWait(0, MS)` allows 0 (sleeps 0).
**How to avoid:** This is a config validation issue. The config has `@NotNull` on `waitTime` but no `@Min`. If `waitTime` can be 0, use `withDelayFn(ctx -> Duration.ofMillis(config.getWaitTime().toMilliseconds()))` instead, which allows 0. But the default is 500ms, so this is unlikely in practice.
**Warning signs:** `IllegalArgumentException: delay must be greater than 0` at strategy construction.

### Pitfall 6: withBackoff assertion delay < maxDuration

**What goes wrong:** `withBackoff` asserts `delay < maxDuration` (if maxDuration is set). For time-limited exponential, `delay = 2 * multiplier` must be < `maxTime`. If `maxTime` is very small, this throws.
**Why it happens:** Failsafe validates that the base delay is less than the max duration. Guava doesn't check this.
**How to avoid:** This is a config validation issue. The default `maxTime = 30s` and `multipier = 1` (so `delay = 2ms`) is fine. Only misconfigured small `maxTime` values would fail.
**Warning signs:** `IllegalStateException: delay must be < the maxDuration` at strategy construction.

### Pitfall 7: withBackoff assertion delay < maxDelay

**What goes wrong:** `withBackoff` asserts `delay < maxDelay`. If `2 * multiplier >= maxTimeBetweenRetries`, this throws.
**Why it happens:** Failsafe validates base delay < max delay. Guava also checks `multiplier < maximumWait`, so this constraint is preserved from the original config.
**How to avoid:** Config already has `@Min(1)` on `multipier` and `maxTimeBetweenRetries` defaults to 500ms. As long as `2 * multipier < maxTimeBetweenRetries.toMilliseconds()`, this is fine.
**Warning signs:** `IllegalArgumentException: delay must be < the maxDelay` at strategy construction.

## Code Examples

Verified patterns from failsafe 3.3.2 source at `/tmp/fsafe_src/`:

### Base class (RetryStrategy.java)

```java
// Source: /tmp/fsafe_src/dev/failsafe/Failsafe.java, FailsafeExecutor.java, RetryPolicy.java
package io.appform.dropwizard.actors.retry;

import dev.failsafe.Failsafe;
import dev.failsafe.RetryPolicy;

import java.util.concurrent.Callable;

public abstract class RetryStrategy {
    private final RetryPolicy<Boolean> retryPolicy;

    protected RetryStrategy(RetryPolicy<Boolean> retryPolicy) {
        this.retryPolicy = retryPolicy;
    }

    public boolean execute(Callable<Boolean> callable) throws Exception {
        return Failsafe.with(retryPolicy).get(() -> callable.call());
    }
}
```

### CountLimitedExponentialWaitRetryStrategy

```java
// Source: /tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java withBackoff(Duration, Duration, double)
// Guava: exponentialWait(multiplier, maxTimeBetweenRetries, MS) → multiplier * 2^attempt
// Failsafe: withBackoff(2*multiplier, maxTimeBetweenRetries, 2.0) → 2*mult * 2^(attempt-1)
super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxAttempts(config.getMaxAttempts())
        .withBackoff(
            Duration.ofMillis(2 * config.getMultipier()),
            Duration.ofMillis(config.getMaxTimeBetweenRetries().toMilliseconds()),
            2.0)
        .build());
```

### CountLimitedFixedWaitRetryStrategy

```java
// Source: /tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java withDelay(Duration)
super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxAttempts(config.getMaxAttempts())
        .withDelay(Duration.ofMillis(config.getWaitTime().toMilliseconds()))
        .build());
```

### CountLimitedIncrementalWaitRetryStrategy

```java
// Source: /tmp/fsafe_src/dev/failsafe/DelayablePolicyBuilder.java withDelayFn(ContextualSupplier)
// Source: /tmp/fsafe_src/dev/failsafe/ExecutionContext.java getAttemptCount()
long initial = config.getInitialWaitTime().toMilliseconds();
long increment = config.getWaitIncrement().toMilliseconds();
super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxAttempts(config.getMaxAttempts())
        .withDelayFn(ctx -> Duration.ofMillis(
            initial + (ctx.getAttemptCount() - 1) * increment))
        .build());
```

### NoRetryStrategy

```java
// Source: /tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java withMaxAttempts(int)
super(RetryPolicy.<Boolean>builder()
        .withMaxAttempts(1)
        .build());
```

### TimeLimitedExponentialWaitRetryStrategy

```java
// Source: /tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java withMaxDuration(Duration), withMaxRetries(int)
super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxDuration(Duration.ofMillis(config.getMaxTime().toMilliseconds()))
        .withMaxRetries(-1)
        .withBackoff(
            Duration.ofMillis(2 * config.getMultipier()),
            Duration.ofMillis(config.getMaxTimeBetweenRetries().toMilliseconds()),
            2.0)
        .build());
```

### TimeLimitedFixedWaitRetryStrategy

```java
super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxDuration(Duration.ofMillis(config.getMaxTime().toMilliseconds()))
        .withMaxRetries(-1)
        .withDelay(Duration.ofMillis(config.getWaitTime().toMilliseconds()))
        .build());
```

### TimeLimitedIncrementalWaitRetryStrategy

```java
long initial = config.getInitialWaitTime().toMilliseconds();
long increment = config.getWaitIncrement().toMilliseconds();
super(RetryPolicy.<Boolean>builder()
        .handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))
        .withMaxDuration(Duration.ofMillis(config.getMaxTime().toMilliseconds()))
        .withMaxRetries(-1)
        .withDelayFn(ctx -> Duration.ofMillis(
            initial + (ctx.getAttemptCount() - 1) * increment))
        .build());
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
| ------------ | ---------------- | ------------- | ------ |
| `RetryerBuilder` fluent API | `RetryPolicy.builder()` fluent API | failsafe 3.x | Builder chain structure is similar but API names differ |
| `retryIfException(predicate)` | `handleIf(predicate)` | failsafe 3.x | Same semantics, different method name |
| `StopStrategies.stopAfterAttempt(n)` | `withMaxAttempts(n)` | failsafe 3.x | Same semantics (n total attempts) |
| `StopStrategies.stopAfterDelay(ms, MS)` | `withMaxDuration(Duration)` + `withMaxRetries(-1)` | failsafe 3.x | Must explicitly disable default 3-attempt cap |
| `WaitStrategies.exponentialWait(mult, max, MS)` | `withBackoff(2*mult, max, 2.0)` | failsafe 3.x | Must correct base delay (2*multiplier, not multiplier) |
| `WaitStrategies.fixedWait(ms, MS)` | `withDelay(Duration.ofMillis(ms))` | failsafe 3.x | Trivial 1:1 mapping |
| `WaitStrategies.incrementingWait(initial, MS, inc, MS)` | `withDelayFn(ctx -> ...)` | failsafe 3.x | Must use delay function with ExecutionContext |
| `BlockStrategies.threadSleepStrategy()` | (implicit in `.get()`) | failsafe 3.x | No explicit block strategy needed |
| `Retryer.call(callable)` | `Failsafe.with(policy).get(() -> callable.call())` | failsafe 3.x | Lambda adapts Callable → CheckedSupplier |
| `RetryException` (checked) | `FailsafeException` (unchecked) or raw throwable | failsafe 3.x | Exception type change, documented in Phase 4 |

**Deprecated/outdated:**

- `net.jodah:failsafe`: Deprecated predecessor of `dev.failsafe:failsafe`. Must not be used.
- `com.github.rholder:guava-retrying`: Unmaintained. Being removed in Phase 3.

## Open Questions

1. **withDelay(Duration.ZERO) for fixed wait with waitTime=0**
   - What we know: `DelayablePolicyBuilder.withDelay(Duration)` asserts `delay.toNanos() > 0`. Guava's `fixedWait(0, MS)` allows 0.
   - What's unclear: Can `waitTime` be 0 in practice? Config has `@NotNull` but no `@Min`.
   - Recommendation: Use `withDelay(Duration.ofMillis(config.getWaitTime().toMilliseconds()))`. If `waitTime=0` is a valid config, switch to `withDelayFn(ctx -> Duration.ofMillis(config.getWaitTime().toMilliseconds()))` which allows 0. Default is 500ms, so this is unlikely. Planner can decide — this is an edge case.

2. **withBackoff + withMaxDuration ordering for time-limited exponential**
   - What we know: `withBackoff` asserts `delay < maxDuration` (if set). `withMaxDuration` asserts `maxDuration > delay` (if set). Both must be satisfied.
   - What's unclear: Does the order of `.withBackoff(...).withMaxDuration(...)` vs `.withMaxDuration(...).withBackoff(...)` matter?
   - Recommendation: Order doesn't matter — both methods read `config.maxDuration` / `config.delay` which are set independently. Use any order. The assertions check the same condition from both sides.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
| ---------- | ----------- | --------- | ------- | -------- |
| Java 17+ | Project compilation | ✓ | 17+ | — |
| Maven | Build system | ✓ | (existing) | — |
| dev.failsafe:failsafe:3.3.2 | Retry engine | ✓ | 3.3.2 | — |
| guava-retrying:2.0.0 | Old retry engine (still on classpath) | ✓ | 2.0.0 | — |

**Missing dependencies with no fallback:** None.

**Missing dependencies with fallback:** None.

## Sources

### Primary (HIGH confidence)

- `/tmp/fsafe_src/dev/failsafe/RetryPolicy.java` — interface definition, `builder()` static method
- `/tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java` — all builder methods: `withBackoff`, `withDelay`, `withMaxAttempts`, `withMaxRetries`, `withMaxDuration`, `build()`
- `/tmp/fsafe_src/dev/failsafe/DelayablePolicyBuilder.java` — `withDelayFn(ContextualSupplier)`, `withDelay(Duration)`
- `/tmp/fsafe_src/dev/failsafe/FailurePolicyBuilder.java` — `handleIf(CheckedPredicate)`, `handleIf(CheckedBiPredicate)`
- `/tmp/fsafe_src/dev/failsafe/FailurePolicy.java` (spi) — `isFailure()` default method: predicate evaluation logic
- `/tmp/fsafe_src/dev/failsafe/Failsafe.java` — `with(Policy)` static method
- `/tmp/fsafe_src/dev/failsafe/FailsafeExecutor.java` — `get(CheckedSupplier)` synchronous execution
- `/tmp/fsafe_src/dev/failsafe/ExecutionContext.java` — `getAttemptCount()` javadoc: "Will return 0 when the first attempt is in progress"
- `/tmp/fsafe_src/dev/failsafe/ExecutionImpl.java` — `recordAttempt()` → `attempts.incrementAndGet()`, `getAttemptCount()` returns `attempts.get()`
- `/tmp/fsafe_src/dev/failsafe/internal/RetryPolicyExecutor.java` — `onFailure()`: delay computation, `adjustForBackoff()`, `adjustForMaxDuration()`
- `/tmp/fsafe_src/dev/failsafe/internal/RetryPolicyImpl.java` — `computeDelay()` delegation to `DelayablePolicy`
- `/tmp/fsafe_src/dev/failsafe/spi/DelayablePolicy.java` — `computeDelay()` calls `config.getDelayFn().get(context)`
- `/tmp/fsafe_src/dev/failsafe/FailsafeException.java` — `extends RuntimeException`, wraps cause
- `/tmp/fsafe_src/dev/failsafe/RetryPolicyConfig.java` — `maxRetries` default, `getMaxAttempts()` = `maxRetries + 1`
- `/tmp/fsafe_src/dev/failsafe/function/CheckedSupplier.java` — `get() throws Throwable`
- `/tmp/fsafe_src/dev/failsafe/function/CheckedPredicate.java` — `test(T) throws Throwable`
- `/tmp/fsafe_src/dev/failsafe/function/ContextualSupplier.java` — `get(ExecutionContext<R>) throws Throwable`
- `/tmp/guava_retry_src/com/github/rholder/retry/WaitStrategies.java` — `ExponentialWaitStrategy`, `IncrementingWaitStrategy`, `FixedWaitStrategy` source
- `/tmp/guava_retry_src/com/github/rholder/retry/Retryer.java` — `attemptNumber` starts at 1 (line 157)
- `/tmp/guava_retry_src/com/github/rholder/retry/Attempt.java` — `getAttemptNumber()` javadoc: "The number, starting from 1"
- Existing source: `RetryStrategy.java`, all 7 impl files, all config files, `CommonUtils.java`, `RetryStrategyFactory.java`, `Handler.java`

### Secondary (MEDIUM confidence)

- None — all findings verified against source jars.

### Tertiary (LOW confidence)

- None.

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH — failsafe 3.3.2 source verified directly from extracted jar
- Architecture: HIGH — all 8 files read, all API signatures confirmed against source
- Pitfalls: HIGH — every behavioral claim verified against source code of both libraries
- D-09 resolution: HIGH — `getAttemptCount()` basis confirmed via `ExecutionImpl.recordAttempt()` → `attempts.incrementAndGet()` called from `record()` which is called from `postExecute()` before `onFailure()` (where delay is computed)

**Research date:** 2026-09-10
**Valid until:** 2026-10-10 (stable — failsafe 3.3.2 is a released version, no API changes expected)
