---
phase: 03-remove-guava-retrying-verify-parity
plan: 02
status: complete
requirements: [VER-02, VER-03, VER-04]
---

# Plan 03-02 Summary: Write parity tests (timing, attempt counts, grep gate)

## What was built

Three test classes proving behavioral parity of the failsafe retry engine against the locked Phase 2 decisions (D-05 through D-14). All run against the failsafe-only tree (Plan 03-01 removed guava-retrying).

## Tasks completed

| Task | Description | Status |
| ---- | ----------- | ------ |
| 1 | RetryTimingParityTest — exact sleep sequence assertions (VER-02) | ✓ |
| 2 | RetryAttemptCountTest — exact attempt counts (VER-03) | ✓ |
| 3 | RetryGrepGateTest — source pattern enforcement (VER-04) | ✓ |

## Verification

- `mvn test -Dtest="RetryTimingParityTest,RetryAttemptCountTest,RetryGrepGateTest"` → 15 tests pass, BUILD SUCCESS
- `mvn clean test` (full suite) → 49 tests pass (34 existing + 15 new), BUILD SUCCESS

## Key files

- `src/test/java/io/appform/dropwizard/actors/retry/RetryTimingParityTest.java` — 5 tests, uses `onRetryScheduled` listener to capture `event.getDelay()` (avoids mockStatic(Thread.class) which Mockito blocks)
- `src/test/java/io/appform/dropwizard/actors/retry/RetryAttemptCountTest.java` — 7 tests, constructs actual strategy objects with counting Callable
- `src/test/java/io/appform/dropwizard/actors/retry/RetryGrepGateTest.java` — 3 tests, Files.walk source scan

## Decisions

- Added a private `read(Path)` helper in RetryGrepGateTest wrapping `Files.readString`'s checked IOException in UncheckedIOException — the `forEach` lambda (Consumer) can't propagate checked exceptions. Minimal fix, no behavior change.

## Self-Check: PASSED

All must_haves satisfied:

- Exponential backoff produces [2*mult, 4*mult, 8*mult] capped at maxTimeBetweenRetries ✓
- Incremental wait produces [initial, initial+increment, initial+2*increment] ✓
- Count-limited strategies stop after exactly maxAttempts ✓
- NoRetryStrategy executes exactly once ✓
- Time-limited strategies make >3 attempts (withMaxRetries(-1) disables default cap) ✓
- Non-matching exception not retried (attempt count = 1) ✓
- No withMaxRetries (except -1) in retry package source ✓
- No *Async API calls in retry package source ✓
