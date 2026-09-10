# Stack Research

**Domain:** Retry-engine replacement for a Java/Dropwizard RabbitMQ actor library (guava-retrying → failsafe.dev)
**Researched:** 2025-09-19
**Confidence:** HIGH

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
| ---------- | ------- | ------- | --------------- |
| dev.failsafe:failsafe | 3.3.2 | Resilience / retry policy engine replacing `com.github.rholder:guava-retrying:2.0.0` | Actively maintained (failsafe-lib/failsafe on GitHub), fluent `RetryPolicy` API, zero transitive dependencies, Apache 2.0. Drop-in for the `Retryer.call(Callable)` pattern this codebase uses. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
| ------- | ------- | ------- | ------------ |
| (none) | — | — | failsafe 3.3.2 has zero runtime dependencies. No Guava, no Netty. This is the whole point of the swap: removes the unmaintained guava-retrying and its Guava coupling. |

### Development Tools

| Tool | Purpose | Notes |
| ---- | ------- | ----- |
| Maven | Build / dependency management | Existing. Add `dev.failsafe:failsafe:3.3.2`, remove `com.github.rholder:guava-retrying:2.0.0`. |

## Installation

```xml
<!-- pom.xml <properties> -->
<failsafe.version>3.3.2</failsafe.version>

<!-- Add -->
<dependency>
    <groupId>dev.failsafe</groupId>
    <artifactId>failsafe</artifactId>
    <version>${failsafe.version}</version>
</dependency>

<!-- Remove -->
<dependency>
    <groupId>com.github.rholder</groupId>
    <artifactId>guava-retrying</artifactId>
    <version>${guava-retrying.version}</version>
</dependency>
```

## Version & Compatibility Facts

| Fact | Value | Source | Confidence |
| ---- | ----- | ------ | ---------- |
| Latest Maven Central version | 3.3.2 | `maven-metadata.xml` on repo1.maven.org (`<latest>3.3.2</latest>`, `<release>3.3.2</release>`, lastUpdated 2023-06-24) | HIGH |
| Maven coordinates | `dev.failsafe:failsafe:3.3.2` | Maven Central POM | HIGH |
| Java requirement | Java 8+ (parent POM `maven.compiler.source/target = 1.8`) | failsafe-parent-3.3.2.pom | HIGH |
| Java 17 compatibility | YES — compiles to bytecode 1.8, runs on 17. Project targets `<release>17</release>`. | Parent POM + failsafe.dev ("Java 8+") | HIGH |
| Transitive dependencies | ZERO. `failsafe-3.3.2.pom` has no `<dependencies>` block (only build plugins). Parent POM deps are all `<scope>test</scope>`. | Maven Central POMs | HIGH |
| License | Apache 2.0 | failsafe-parent-3.3.2.pom | HIGH |
| Maintenance status | Active. failsafe-lib/failsafe on GitHub, CI green, docs at failsafe.dev. (guava-retrying is unmaintained — last release 2015.) | GitHub repo | HIGH |

> Note on "latest": Maven Central's latest is 3.3.2 (June 2023). The failsafe-lib/failsafe GitHub repo is the active home; the older `net.jodah:failsafe` coordinates are the predecessor and must NOT be used. Use `dev.failsafe:failsafe` only.

## Core API Surface (what the migration touches)

### Building a RetryPolicy

```java
RetryPolicy<Boolean> policy = RetryPolicy.<Boolean>builder()
    .handleIf(exception -> CommonUtils.isRetriable(retriableExceptions, exception))  // exception predicate
    .withMaxAttempts(n)          // OR .withMaxRetries(n)
    .withMaxDuration(Duration.ofMillis(maxMillis))
    .withDelay(Duration.ofMillis(fixedMillis))
    .withBackoff(Duration.ofMillis(initial), Duration.ofMillis(max), delayFactor)
    .withDelayFn(ctx -> computeIncrementalDelay(ctx))  // escape hatch for incremental waits
    .build();
```

### Executing a Callable<Boolean> (synchronous, blocking)

```java
// RetryStrategy.execute(Callable<Boolean>) replacement:
boolean result = Failsafe.with(policy).get(callable::call);
// throws the last exception if retries exhausted — same contract as Retryer.call()
```

- `Failsafe.with(policy).get(supplier)` — synchronous, **blocks the calling thread**. Returns the result or throws the last exception. This is the direct equivalent of `Retryer.call(callable)`.
- `Failsafe.with(policy).run(runnable)` — synchronous, blocking, no result.
- `Failsafe.with(policy).getAsync(supplier)` / `runAsync(...)` — asynchronous, returns `CompletableFuture`. NOT used by this migration (guava-retrying uses `BlockStrategies.threadSleepStrategy()`, i.e. blocking).

### Blocking behavior (confirmed)

`Failsafe.with(policy).get()` / `.run()` execute **synchronously on the calling thread** and sleep the calling thread between attempts. This is the exact equivalent of guava-retrying's `BlockStrategies.threadSleepStrategy()`. There is no `BlockStrategy` concept in failsafe — synchronous execution blocks by default; async execution uses a `ScheduledExecutorService`. (Source: failsafe.dev overview + async-execution docs — `run`/`get` are the sync variants, `runAsync`/`getAsync` the async ones.)

### Failure handling (exception predicates)

From `FailurePolicyBuilder` (which `RetryPolicyBuilder` extends via `DelayablePolicyBuilder`):

| Method | Purpose |
| ------ | ------- |
| `.handle(Class<? extends Throwable>...)` | Handle specific exception types |
| `.handleIf(CheckedPredicate<Throwable>)` | Handle exceptions matching a predicate ← **this is what `retryIfException(predicate)` maps to** |
| `.handleIf(CheckedBiPredicate<R, Throwable>)` | Predicate over result AND exception |
| `.handleResult(R)` / `.handleResultIf(...)` | Handle specific results (not needed here — this codebase retries on exceptions only) |

Default: a policy handles **any `Exception`** unless narrowed. `handleIf` on an exception predicate replaces the default. (Source: failsafe.dev/policies failure-handling section.)

## guava-retrying → failsafe API Mapping

This is the prescriptive mapping for the 7 strategy impls in `retry/impl/`. Every row verified against the failsafe 3.3.2 javadoc.

### Builder / execution

| guava-retrying | failsafe 3.3.2 | Notes |
| -------------- | --------------- | ----- |
| `RetryerBuilder.<Boolean>newBuilder()...build()` → `Retryer<Boolean>` | `RetryPolicy.<Boolean>builder()...build()` → `RetryPolicy<Boolean>` | Policy is built separately from execution. |
| `retryer.call(callable)` | `Failsafe.with(policy).get(callable::call)` | `.get()` blocks calling thread, throws last exception on exhaustion. `RetryStrategy.execute` wraps this. |
| `retryIfException(predicate)` | `.handleIf(predicate)` | `handleIf(CheckedPredicate<Throwable>)`. Predicate signature matches. |
| `withBlockStrategy(BlockStrategies.threadSleepStrategy())` | *(implicit — remove)* | Sync `.get()`/`.run()` blocks the calling thread by default. No equivalent needed. |

### Stop strategies

| guava-retrying | failsafe 3.3.2 | Notes |
| -------------- | --------------- | ----- |
| `StopStrategies.stopAfterAttempt(n)` | `.withMaxAttempts(n)` | **Semantics match exactly**: guava's `stopAfterAttempt(n)` = stop after n attempts; failsafe's `withMaxAttempts(n)` = max n execution attempts. Both count total attempts (not retries). HIGH confidence. |
| `StopStrategies.stopAfterDelay(millis, MILLISECONDS)` | `.withMaxDuration(Duration.ofMillis(millis))` | Stops retrying once total elapsed time exceeds the duration. HIGH confidence. |

### Wait strategies

| guava-retrying | failsafe 3.3.2 | Notes |
| -------------- | --------------- | ----- |
| `WaitStrategies.fixedWait(time, unit)` | `.withDelay(Duration.ofMillis(time))` | Fixed delay between attempts. Direct 1:1. HIGH confidence. |
| `WaitStrategies.exponentialWait(multiplier, maxTime, unit)` | `.withBackoff(Duration.ofMillis(initial), Duration.ofMillis(max), delayFactor)` | **See note below — parameter semantics differ.** MEDIUM-HIGH confidence. |
| `WaitStrategies.incrementingWait(initial, iUnit, increment, incUnit)` | `.withDelayFn(ctx -> Duration.ofMillis(initial + ctx.getAttemptCount() * increment))` | No built-in incremental backoff. Use `withDelayFn` with `ExecutionContext.getAttemptCount()`. MEDIUM confidence (see note). |

#### Exponential-wait parameter mapping (important)

guava-retrying `exponentialWait(multiplier, maxTime, unit)`:

- delay for attempt *n* (1-indexed) = `multiplier * 2^(n-1)`, capped at `maxTime`.
- The first argument `multiplier` is **both the initial delay and the base multiplier**.

failsafe `withBackoff(Duration delay, Duration maxDelay, double delayFactor)`:

- delay for attempt *n* = `delay * delayFactor^(n-1)`, capped at `maxDelay`.
- `delay` = initial delay; `delayFactor` = the multiplier (guava hardcodes factor 2; failsafe lets you set it).

**Mapping for this codebase:** `config.getMultipier()` is guava's `multiplier` (the initial delay AND base). To preserve behavior:

```java
.withBackoff(
    Duration.ofMillis(config.getMultipier()),        // initial delay = guava multiplier
    Duration.ofMillis(config.getMaxTimeBetweenRetries()),  // cap
    2.0)                                             // guava hardcodes factor 2
```

Caveat: guava's formula is `multiplier * 2^(n-1)`; failsafe's is `delay * delayFactor^(n-1)`. With `delay = multiplier` and `delayFactor = 2.0` these are **identical**. HIGH confidence on the math, but the config field name `getMultipier()` (sic — typo in existing code) should be read as "initial delay / base". Verify against existing tests.

#### Incremental-wait mapping (the one non-built-in)

guava-retrying `incrementingWait(initial, iUnit, increment, incUnit)`:

- attempt *n* delay = `initial + (n-1) * increment`.

failsafe has no `withIncrementingBackoff`. Use `withDelayFn`:

```java
.withDelayFn(ctx -> {
    long attempt = ctx.getAttemptCount();  // attempts so far; first retry is attempt 1
    return Duration.ofMillis(config.getInitialWaitTime().toMilliseconds()
                            + attempt * config.getWaitIncrement().toMilliseconds());
})
```

**Verify the off-by-one**: `ExecutionContext.getAttemptCount()` returns the number of attempts that have completed. For the delay *before* the 1st retry, 1 attempt has completed → delay = `initial + 1*increment`. guava's first wait (before 1st retry) = `initial`. **This is an off-by-one — the failsafe version adds the increment one attempt earlier.** To match guava exactly:

```java
.withDelayFn(ctx -> Duration.ofMillis(
    config.getInitialWaitTime().toMilliseconds()
    + (ctx.getAttemptCount() - 1) * config.getWaitIncrement().toMilliseconds()))
```

MEDIUM confidence — this is the one mapping that needs a test to lock down the off-by-one. The existing `CountLimitedIncrementalWaitRetryStrategy` and `TimeLimitedIncrementalWaitRetryStrategy` tests (if any) are the regression net.

## Strategy-by-Strategy Mapping (all 7 impls)

| Strategy class | guava config | failsafe RetryPolicy |
| -------------- | ------------ | -------------------- |
| `NoRetryStrategy` | `stopAfterAttempt(1)` | `.withMaxAttempts(1)` (no `handleIf`, no delay) |
| `CountLimitedFixedWaitRetryStrategy` | `stopAfterAttempt(max)` + `fixedWait(waitTime)` + `threadSleep` + `retryIfException` | `.handleIf(...).withMaxAttempts(max).withDelay(waitTime)` |
| `CountLimitedExponentialWaitRetryStrategy` | `stopAfterAttempt(max)` + `exponentialWait(multiplier, maxTime)` + `threadSleep` + `retryIfException` | `.handleIf(...).withMaxAttempts(max).withBackoff(multiplier, maxTime, 2.0)` |
| `CountLimitedIncrementalWaitRetryStrategy` | `stopAfterAttempt(max)` + `incrementingWait(initial, inc)` + `threadSleep` + `retryIfException` | `.handleIf(...).withMaxAttempts(max).withDelayFn(ctx -> initial + (attempt-1)*inc)` |
| `TimeLimitedFixedWaitRetryStrategy` | `stopAfterDelay(maxTime)` + `fixedWait(waitTime)` + `threadSleep` + `retryIfException` | `.handleIf(...).withMaxDuration(maxTime).withDelay(waitTime)` |
| `TimeLimitedExponentialWaitRetryStrategy` | `stopAfterDelay(maxTime)` + `exponentialWait(multiplier, maxTimeBetweenRetries)` + `threadSleep` + `retryIfException` | `.handleIf(...).withMaxDuration(maxTime).withBackoff(multiplier, maxTimeBetweenRetries, 2.0)` |
| `TimeLimitedIncrementalWaitRetryStrategy` | `stopAfterDelay(maxTime)` + `incrementingWait(initial, inc)` + `threadSleep` + `retryIfException` | `.handleIf(...).withMaxDuration(maxTime).withDelayFn(ctx -> initial + (attempt-1)*inc)` |

## RetryStrategy base class

The base class holds a `Retryer<Boolean>` and calls `retryer.call(callable)`. Minimal change:

```java
// Before
private final Retryer<Boolean> retryer;
protected RetryStrategy(Retryer<Boolean> retryer) { this.retryer = retryer; }
public boolean execute(Callable<Boolean> callable) throws Exception {
    return retryer.call(callable);
}

// After
private final RetryPolicy<Boolean> retryPolicy;
protected RetryStrategy(RetryPolicy<Boolean> retryPolicy) { this.retryPolicy = retryPolicy; }
public boolean execute(Callable<Boolean> callable) throws Exception {
    return Failsafe.with(retryPolicy).get(callable::call);
}
```

Public API (`execute(Callable<Boolean>)` signature) is unchanged. Constructor param type changes from `Retryer<Boolean>` to `RetryPolicy<Boolean>` — internal to the package, not part of the public API per PROJECT.md constraints.

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
| ----------- | ----------- | ----------------------- |
| dev.failsafe:failsafe 3.3.2 | net.jodah:failsafe (2.x) | Never — predecessor coordinates, superseded by `dev.failsafe`. |
| dev.failsafe:failsafe 3.3.2 | Spring Retry (`@Retryable`) | Only if the host app is already Spring-centric and wants annotation-driven retry. This library is Dropwizard (not Spring) and needs programmatic per-strategy policies — failsafe is the better fit. |
| dev.failsafe:failsafe 3.3.2 | Resilience4j retry | If the host app already uses Resilience4j for circuit breakers/rate limiting. Heavier dependency (Vavr transitive). failsafe's zero-dependency profile is preferable here. |
| dev.failsafe:failsafe 3.3.2 | Keep guava-retrying | Not viable — unmaintained (last release 2015), pulls Guava. This is the problem being solved. |

## What NOT to Use

| Avoid | Why | Use Instead |
| ----- | --- | ----------- |
| `net.jodah:failsafe` (2.4.x) | Old coordinates, predecessor to `dev.failsafe`. API differs (2.x used `withRetryOn`, 3.x uses `handleIf`). | `dev.failsafe:failsafe:3.3.2` |
| `com.github.rholder:guava-retrying` | Unmaintained since 2015, drags in Guava. | `dev.failsafe:failsafe:3.3.2` |
| failsafe async API (`getAsync`/`runAsync`) | This codebase uses blocking `Retryer.call()` via `BlockStrategies.threadSleepStrategy()`. Switching to async would change threading semantics. | `Failsafe.with(policy).get()` (synchronous, blocking) |
| `withMaxRetries(n)` | Semantically different from guava's `stopAfterAttempt(n)`. `withMaxRetries(n)` = n retries *after* the first attempt (n+1 total). `withMaxAttempts(n)` = n total attempts. guava's `stopAfterAttempt(n)` = n total attempts. | `.withMaxAttempts(n)` to preserve exact semantics. |

## Stack Patterns by Variant

**If the strategy is count-limited** (`CountLimited*`):

- Use `.withMaxAttempts(config.getMaxAttempts())`
- Because guava `stopAfterAttempt(maxAttempts)` = failsafe `withMaxAttempts(maxAttempts)` (both count total attempts).

**If the strategy is time-limited** (`TimeLimited*`):

- Use `.withMaxDuration(Duration.ofMillis(config.getMaxTime().toMilliseconds()))`
- Because guava `stopAfterDelay(maxTime)` = failsafe `withMaxDuration(maxTime)`.

**If the wait is fixed:**

- Use `.withDelay(Duration.ofMillis(waitTime))`

**If the wait is exponential:**

- Use `.withBackoff(Duration.ofMillis(multiplier), Duration.ofMillis(maxTimeBetweenRetries), 2.0)`
- Because guava's `exponentialWait(multiplier, max, unit)` = `multiplier * 2^(n-1)` capped at `max`, and failsafe's `withBackoff(delay, max, 2.0)` = `delay * 2^(n-1)` capped at `max`. Identical when `delay = multiplier`.

**If the wait is incremental:**

- Use `.withDelayFn(ctx -> Duration.ofMillis(initial + (ctx.getAttemptCount() - 1) * increment))`
- Because failsafe has no built-in incremental backoff; `withDelayFn` computes per-attempt delay. The `-1` corrects the off-by-one vs guava (guava's first wait = `initial`; failsafe's `getAttemptCount()` at first-retry = 1).

## Version Compatibility

| Package | Compatible With | Notes |
| ------- | --------------- | ----- |
| `dev.failsafe:failsafe:3.3.2` | Java 8+ (bytecode level 1.8) | Runs on Java 17. Project compiles with `<release>17</release>`. No conflict. |
| `dev.failsafe:failsafe:3.3.2` | Dropwizard 5.x | No relationship — failsafe is standalone. No Dropwizard-specific integration needed. |
| `dev.failsafe:failsafe:3.3.2` | Guava (existing transitive via Dropwizard) | No conflict. Failsafe has zero deps. Removing guava-retrying removes its Guava requirement, but Guava remains via Dropwizard — that's fine, nothing in this migration touches Guava directly. |

## Quality Gate

- [x] Version is current (verified via Maven Central `maven-metadata.xml`: latest = 3.3.2, 2023-06-24)
- [x] API mapping covers all 7 strategy variants in the codebase (see strategy-by-strategy table)
- [x] Blocking behavior confirmed (`Failsafe.with(policy).get()` blocks calling thread — equivalent to `BlockStrategies.threadSleepStrategy()`)
- [x] Transitive deps checked (zero — `failsafe-3.3.2.pom` has no `<dependencies>`; parent POM deps are test-scoped only; no Guava)

## Open Questions for Implementation Phase

1. **Incremental off-by-one**: confirm `ExecutionContext.getAttemptCount()` semantics against existing `*IncrementalWaitRetryStrategy` tests. The mapping above assumes `getAttemptCount()` = completed attempts (1 at first retry). If tests show otherwise, adjust the `-1`.
2. **`getMultipier()` typo**: the existing config field is misspelled (`Multipier` not `Multiplier`). Preserve the field name — it's config-bound (JSON deserialization). Read it as "initial delay / exponential base".
3. **Exception propagation**: `Failsafe.with(policy).get()` throws the last exception when retries are exhausted. `Retryer.call()` does the same. Confirm `RetryStrategy.execute` callers don't catch a specific exception type that failsafe wraps differently (failsafe does NOT wrap — it rethrows the original). HIGH confidence, verify with tests.

## Sources

- Maven Central `maven-metadata.xml` — <https://repo1.maven.org/maven2/dev/failsafe/failsafe/maven-metadata.xml> — latest/release version (3.3.2). Confidence: HIGH.
- Maven Central POM `failsafe-3.3.2.pom` — <https://repo1.maven.org/maven2/dev/failsafe/failsafe/3.3.2/failsafe-3.3.2.pom> — zero `<dependencies>`. Confidence: HIGH.
- Maven Central parent POM `failsafe-parent-3.3.2.pom` — <https://repo1.maven.org/maven2/dev/failsafe/failsafe-parent/3.3.2/failsafe-parent-3.3.2.pom> — `maven.compiler.source/target=1.8`, Apache 2.0, test-only deps. Confidence: HIGH.
- failsafe.dev overview — <https://failsafe.dev/> — "zero-dependency library for handling failures in Java 8+", sync `.run()`/`.get()` vs async `.runAsync()`/`.getAsync()`. Confidence: HIGH.
- failsafe.dev retry docs — <https://failsafe.dev/retry/> — `withMaxAttempts`, `withMaxRetries`, `withMaxDuration`, `withDelay`, `withBackoff`, `withJitter`. Confidence: HIGH.
- failsafe.dev policies docs — <https://failsafe.dev/policies/> — `handle`/`handleIf` failure handling, default handles any Exception. Confidence: HIGH.
- failsafe 3.3.2 javadoc `RetryPolicyBuilder` — <https://failsafe.dev/javadoc/core/dev/failsafe/RetryPolicyBuilder.html> — exact method signatures for `withBackoff(Duration,Duration,double)`, `withMaxAttempts(int)`, `withMaxDuration(Duration)`, `withDelay(Duration)`. Confidence: HIGH.
- failsafe 3.3.2 javadoc `DelayablePolicyBuilder` — <https://failsafe.dev/javadoc/core/dev/failsafe/DelayablePolicyBuilder.html> — `withDelayFn(ContextualSupplier<R,Duration>)` for incremental waits. Confidence: HIGH.
- failsafe-lib/failsafe GitHub — <https://github.com/failsafe-lib/failsafe> — active maintenance, CI. Confidence: HIGH.
- Existing codebase — `src/main/java/io/appform/dropwizard/actors/retry/RetryStrategy.java` + `retry/impl/*.java` (7 strategies) + `pom.xml` (`<release>17</release>`, guava-retrying 2.0.0). Confidence: HIGH (read directly).

---
*Stack research for: guava-retrying → failsafe.dev retry-engine migration*
*Researched: 2025-09-19*
