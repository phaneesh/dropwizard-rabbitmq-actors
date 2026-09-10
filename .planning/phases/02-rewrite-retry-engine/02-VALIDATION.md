---
phase: 02
slug: rewrite-retry-engine
status: complete
nyquist_compliant: true
wave_0_complete: true
created: 2026-09-10
---

# Phase 2 - Validation Strategy

> Per-phase validation contract for the retry engine rewrite (guava-retrying → failsafe).

---

## Test Infrastructure

| Property               | Value                                                        |
| ---------------------- | ----------------------------------------------------------- |
| **Framework**          | JUnit 5 (Jupiter) via maven-surefire-plugin                 |
| **Config file**        | `pom.xml` (surefire config)                                 |
| **Quick run command**  | `mvn test -Dtest="RetryAttemptCountTest,RetryGrepGateTest,RetryTimingParityTest,PomStructureTest"` |
| **Full suite command** | `mvn test`                                                   |
| **Estimated runtime**  | ~90 seconds (full suite, RabbitMQ integration tests included) |

---

## Sampling Rate

- **After every task commit:** Run quick run command (retry-package tests only, <2s)
- **After every plan wave:** Run `mvn test` (full suite)
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** ~90 seconds (full suite); <2s (quick)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
| ------- | ---- | ---- | ----------- | --------- | ----------------- | ----------- | ------ |
| 02-01-1 | 01   | 1    | ENG-01      | unit+grep | `RetryGrepGateTest` + `RetryAttemptCountTest` | ✅ | ✅ green |
| 02-01-2 | 01   | 1    | ENG-02      | unit      | `RetryAttemptCountTest` (calls execute()) | ✅ | ✅ green |
| 02-01-3 | 01   | 1    | ENG-03      | unit      | `RetryAttemptCountTest` (execute(Callable) throws Exception) | ✅ | ✅ green |
| 02-01-4 | 01   | 1    | ENG-04      | grep      | `RetryGrepGateTest.noGuavaRetryingImportsInRetryPackage` | ✅ | ✅ green |
| 02-01-2 | 01   | 1    | PAR-01      | unit+grep | `RetryGrepGateTest.noWithMaxRetriesExceptNegativeOne` + `RetryAttemptCountTest` (3 count-limited tests) | ✅ | ✅ green |
| 02-01-6 | 01   | 1    | PAR-02      | unit+grep | `RetryGrepGateTest` + `RetryAttemptCountTest.timeLimitedFixedWaitExceedsDefaultAttemptCap` + `RetryTimingParityTest` (2 time-limited tests) | ✅ | ✅ green |
| 02-01-2 | 01   | 1    | PAR-03      | unit      | `RetryTimingParityTest.exponentialBackoffProducesCorrectSleepSequence` + `exponentialBackoffCapsAtMaxTimeBetweenRetries` | ✅ | ✅ green |
| 02-01-4 | 01   | 1    | PAR-04      | unit      | `RetryTimingParityTest.incrementalWaitProducesCorrectSleepSequence` | ✅ | ✅ green |
| 02-01-3 | 01   | 1    | PAR-05      | unit      | `RetryAttemptCountTest.countLimitedFixedWaitStopsAfterMaxAttempts` | ✅ | ✅ green |
| 02-01-5 | 01   | 1    | PAR-06      | unit      | `RetryAttemptCountTest.noRetryStrategyExecutesExactlyOnce` | ✅ | ✅ green |
| 02-01-2 | 01   | 1    | PAR-07      | unit      | `RetryAttemptCountTest.nonMatchingExceptionNotRetried` + `matchingExceptionRetried` | ✅ | ✅ green |
| 02-01-9 | 01   | 1    | PAR-08      | grep      | `RetryGrepGateTest.noAsyncCallsInRetryPackage` | ✅ | ✅ green |
| 02-01-1 | 01   | 1    | PAR-09      | unit+grep | `RetryAttemptCountTest.falseReturnIsSuccessNotRetried` + `RetryGrepGateTest.noHandleResultInRetryPackage` | ✅ | ✅ green |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. JUnit 5 + maven-surefire was already configured; no framework install needed.

---

## Manual-Only Verifications

All phase behaviors have automated verification.

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 90s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-09-10

---

## Validation Audit 2026-09-10

| Metric     | Count |
| ---------- | ----- |
| Gaps found | 1     |
| Resolved   | 1     |
| Escalated  | 0     |

**Gap resolved:** PAR-09 (Boolean return = success signal, no `handleResult(false)`) had no automated test. Added:

- `RetryAttemptCountTest.falseReturnIsSuccessNotRetried` — behavioral test: Callable returning `false` is invoked exactly once (not retried).
- `RetryGrepGateTest.noHandleResultInRetryPackage` — grep gate: no `handleResult` call in any retry-package source file.

Full suite: 54 tests, 0 failures, 0 errors.
