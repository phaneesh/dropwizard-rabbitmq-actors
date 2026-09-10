# Phase 2: Rewrite retry engine (base + all seven impls, atomic) - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md - this log preserves the alternatives considered.

**Date:** 2026-09-10
**Phase:** 02-rewrite-retry-engine
**Areas discussed:** Base class, Exponential backoff formula, Incremental wait, Fixed wait, Count-limited stop, Time-limited stop, No-retry, Exception predicate, Boolean semantics, Exhaustion exception

**Mode:** `--auto` — all gray areas auto-selected; each resolved with the recommended option (first/research-backed choice). No interactive AskUserQuestion calls.

---

## Base class (RetryStrategy)

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| Field `RetryPolicy<Boolean>`, execute via `Failsafe.with(policy).get()` | Swap field type; synchronous blocking call matches guava's `retryer.call()` | ✓ |
| Keep `Retryer<Boolean>`, wrap failsafe inside | Would not compile — guava removed in Phase 3; defeats the migration | |

**User's choice:** [auto] Field `RetryPolicy<Boolean>`, execute via `Failsafe.with(policy).get()` (recommended — research-verified drop-in)
**Notes:** Public signature `execute(Callable<Boolean>) throws Exception` byte-identical (ENG-03). Constructor param type changes internally; only 7 impls + factory construct strategies.

---

## Exponential backoff formula mapping

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| `withBackoff(2 * multiplier, 2.0)` + `withMaxDelay(maxTimeBetweenRetries)` | Corrects halved-curve pitfall; `baseDelay = 2 * multiplier` makes attempt-1 sleep match guava | ✓ |
| `withBackoff(multiplier, 2.0)` (naive 1:1) | Silently halves every sleep — the dominant drift risk | |

**User's choice:** [auto] `withBackoff(2 * multiplier, 2.0)` + `withMaxDelay` (recommended — math-verified in research SUMMARY.md §Critical Pitfalls #1)
**Notes:** `getMultipier()` typo preserved (config-compatibility constraint). Formula empirically confirmed via Phase 3 timing-parity test.

---

## Incremental wait implementation

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| `withDelayFn(ctx -> initial + (attempt-1) * increment)` | 1-based attempt; produces guava's `incrementingWait` sequence | ✓ |
| `withDelay(initial).withDelayFn(...)` (chained) | More complex, no benefit | |

**User's choice:** [auto] `withDelayFn` with 1-based attempt formula (recommended)
**Notes:** **Research flag** — `ExecutionContext.getAttemptCount()` basis (0 vs 1) MUST be empirically confirmed against failsafe 3.3.2 source jar during planning. If 0-based, formula adjusts to `initial + attempt * increment`. (research SUMMARY.md §Research Flags)

---

## Fixed wait implementation

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| `withDelay(waitTime.toMilliseconds())` | Trivial 1:1 mapping | ✓ |

**User's choice:** [auto] `withDelay(waitTime.toMilliseconds())` (recommended — trivial)
**Notes:** No drift risk.

---

## Count-limited stop strategy

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| `withMaxAttempts(maxAttempts)` | n total attempts; matches guava `stopAfterAttempt(n)` | ✓ |
| `withMaxRetries(maxAttempts)` | n+1 attempts — off-by-one pitfall | |

**User's choice:** [auto] `withMaxAttempts(maxAttempts)` (recommended — research SUMMARY.md §Critical Pitfalls #3)
**Notes:** Never use `withMaxRetries` for count-limited (off-by-one). Grep gate in Phase 3 enforces this.

---

## Time-limited stop strategy

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| `withMaxDuration(maxTime, MS)` + `withMaxRetries(-1)` | Disables failsafe's default 3-attempt cap | ✓ |
| `withMaxDuration(maxTime, MS)` only | Silently stops after 3 attempts — pitfall #4 | |

**User's choice:** [auto] `withMaxDuration` + `withMaxRetries(-1)` (recommended — research SUMMARY.md §Critical Pitfalls #4)
**Notes:** `withMaxRetries(-1)` is the only allowed use of `withMaxRetries` — Phase 3 grep gate allows the `-1` exception.

---

## No-retry strategy

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| `withMaxAttempts(1)` explicit | Executes exactly once; no retry-by-default | ✓ |
| Bare `RetryPolicy.builder().build()` | Failsafe default = retry-3-times-on-anything — silently breaks NoRetry | |

**User's choice:** [auto] `withMaxAttempts(1)` explicit (recommended — research SUMMARY.md §Critical Pitfalls #2)
**Notes:** No exception predicate (matches guava: NoRetryStrategy has none).

---

## Exception predicate mapping

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| `handleIf(isRetriable predicate)` | Same semantics as guava `retryIfException` | ✓ |
| `handleAll()` + filter post-hoc | Inverted default handling — pitfall #2 | |

**User's choice:** [auto] `handleIf(predicate)` (recommended)
**Notes:** `CommonUtils.isRetriable` unchanged: empty set = retry-all, non-empty = match by simple class name. NoRetryStrategy gets no `handleIf`.

---

## Boolean return value semantics

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| No `handleResult` — `false` = reject, not retry | Matches guava (retries on exception only) | ✓ |
| `handleResult(false)` — retry on false return | Would change semantics; guava doesn't do this | |

**User's choice:** [auto] No `handleResult` (recommended — matches guava behavior)
**Notes:** `true` = success (ack), `false` = reject. failsafe retries on exceptions only by default.

---

## Exhaustion exception propagation

| Option | Description | Selected |
| ---------- | ----------- | -------- |
| Let `FailsafeException`/raw throwable propagate | `throws Exception` covers it; `Handler` catches `Throwable` | ✓ |
| Catch + re-wrap as checked exception | Changes exception contract; unrequested complexity | |

**User's choice:** [auto] Let propagate raw (recommended)
**Notes:** Zero `RetryException` references internally (verified in research). Exception-type change documented in Phase 4 (DOC-01), not shimmed.

---

## the agent's Discretion

- Javadoc note on `RetryStrategy` explaining the swap (optional)
- Import ordering / static-import style (follow codebase conventions)
- Inline lambda vs static helper for incremental-wait delay function (inline simpler; planner decides)

## Deferred Ideas

None — discussion stayed within phase scope. Observability hooks, jitter, and async execution are v2 requirements tracked in REQUIREMENTS.md, not proposed during this discussion.
