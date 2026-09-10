---
phase: 02-rewrite-retry-engine
plan: 01
status: complete
started: 2026-09-10T14:15:00Z
completed: 2026-09-10T14:22:00Z
---

# Plan 02-01: Rewrite retry engine (base + all seven impls, atomic)

## Objective

Atomically swap the retry engine in `RetryStrategy` from guava-retrying's `Retryer<Boolean>` to failsafe's `RetryPolicy<Boolean>`, and rewrite all 7 strategy impl builder chains from `RetryerBuilder` to `RetryPolicy.builder()`.

## Tasks

| # | Task | Status | Commit |
| --- | --- | --- | --- |
| 1 | Rewrite RetryStrategy base class (Retryer → RetryPolicy) | ✓ | 4007dea |
| 2 | Rewrite CountLimitedExponentialWaitRetryStrategy | ✓ | 4007dea |
| 3 | Rewrite CountLimitedFixedWaitRetryStrategy | ✓ | 4007dea |
| 4 | Rewrite CountLimitedIncrementalWaitRetryStrategy | ✓ | 4007dea |
| 5 | Rewrite NoRetryStrategy | ✓ | 4007dea |
| 6 | Rewrite TimeLimitedExponentialWaitRetryStrategy | ✓ | 4007dea |
| 7 | Rewrite TimeLimitedFixedWaitRetryStrategy | ✓ | 4007dea |
| 8 | Rewrite TimeLimitedIncrementalWaitRetryStrategy | ✓ | 4007dea |
| 9 | Compile and test the atomic rewrite | ✓ | 4007dea |

All 8 files rewritten in a single atomic commit (no partial-migration state compiles).

## Key Decisions Applied

- **D-01..D-04**: Base class field `Retryer<Boolean> retryer` → `RetryPolicy<Boolean> retryPolicy`; `execute()` body → `Failsafe.with(retryPolicy).get(() -> callable.call())`; public API signature byte-identical.
- **D-05**: Exponential backoff `baseDelay = 2 * config.getMultipier()` corrects the halved-curve pitfall (guava: `multiplier * 2^attempt`, first sleep = 2*multiplier; failsafe: `baseDelay * factor^(attempt-1)`, first sleep = baseDelay).
- **D-08/D-09**: Incremental wait uses `withDelayFn(ctx -> initial + (ctx.getAttemptCount() - 1) * increment)`. At first retry point `getAttemptCount()=1` (called from `onFailure()` after `record()`), so delay = `initial + 0 = initial`, matching guava's sequence.
- **D-11**: Count-limited strategies use `withMaxAttempts(n)` (n total attempts), never `withMaxRetries(n)`.
- **D-12**: Time-limited strategies use `withMaxDuration(...)` AND `withMaxRetries(-1)` — critical, since `withMaxDuration` does NOT disable failsafe's default `maxRetries=2` (3-attempt cap).
- **D-13**: `NoRetryStrategy` uses `withMaxAttempts(1)` explicitly (not a bare builder, which would default to 3 attempts).
- **D-14/D-15**: `retryIfException(predicate)` → `handleIf(exception -> CommonUtils.isRetriable(...))`.
- **D-07**: `getMultipier()` typo preserved (config-compatibility constraint).
- **D-16/D-17/D-18**: No `handleResult(false)`; `FailsafeException`/throwable propagates raw; no try/catch wrapping in `execute()`.

## Verification Results

| Check | Result |
| --- | --- |
| `mvn compile` | ✓ exit 0 |
| `mvn test` | ✓ exit 0 (all existing tests pass) |
| `grep -rn 'com.github.rholder' src/main/java/` | ✓ zero matches |
| `grep -rn 'RetryerBuilder' src/main/java/` | ✓ zero matches |
| `grep -rn 'withMaxRetries' .../retry/` | ✓ only `withMaxRetries(-1)` (3 lines, time-limited only) |
| `grep -rn 'Async' .../retry/` | ✓ zero matches |
| `execute()` signature count | ✓ 1 |
| failsafe imports in retry package | ✓ 8 files |

## Key Files

### Modified

- `src/main/java/io/appform/dropwizard/actors/retry/RetryStrategy.java` — base class, RetryPolicy<Boolean> field + Failsafe.with().get()
- `src/main/java/io/appform/dropwizard/actors/retry/impl/CountLimitedExponentialWaitRetryStrategy.java` — withBackoff, 2*multiplier, withMaxAttempts
- `src/main/java/io/appform/dropwizard/actors/retry/impl/CountLimitedFixedWaitRetryStrategy.java` — withDelay, withMaxAttempts
- `src/main/java/io/appform/dropwizard/actors/retry/impl/CountLimitedIncrementalWaitRetryStrategy.java` — withDelayFn, getAttemptCount()-1, withMaxAttempts
- `src/main/java/io/appform/dropwizard/actors/retry/impl/NoRetryStrategy.java` — withMaxAttempts(1), no handleIf
- `src/main/java/io/appform/dropwizard/actors/retry/impl/TimeLimitedExponentialWaitRetryStrategy.java` — withMaxDuration + withMaxRetries(-1), withBackoff
- `src/main/java/io/appform/dropwizard/actors/retry/impl/TimeLimitedFixedWaitRetryStrategy.java` — withMaxDuration + withMaxRetries(-1), withDelay
- `src/main/java/io/appform/dropwizard/actors/retry/impl/TimeLimitedIncrementalWaitRetryStrategy.java` — withMaxDuration + withMaxRetries(-1), withDelayFn

## Deviations

None. All 8 files written exactly per plan specifications.

## Issues Encountered

None.

## Self-Check: PASSED

- `mvn compile` exit 0
- `mvn test` exit 0
- Zero `com.github.rholder` / `RetryerBuilder` source references
- `execute(Callable<Boolean>) throws Exception` signature unchanged
- All 8 files contain `import dev.failsafe`
