---
phase: 03
slug: remove-guava-retrying-verify-parity
status: complete
nyquist_compliant: true
wave_0_complete: true
created: 2026-09-10
---

# Phase 3 - Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property               | Value                                               |
| ---------------------- | --------------------------------------------------- |
| **Framework**          | JUnit 5 (Jupiter) via Maven Surefire 3.5.4          |
| **Config file**        | pom.xml (surefire + jacoco plugins)                 |
| **Quick run command**  | `mvn test -Dtest=PomStructureTest,RetryTimingParityTest,RetryAttemptCountTest,RetryGrepGateTest -DfailIfNoTests=false` |
| **Full suite command** | `mvn clean test`                                    |
| **Estimated runtime**  | ~3 seconds (quick), ~15 seconds (full)              |

---

## Sampling Rate

- **After every task commit:** Run quick run command
- **After every plan wave:** Run `mvn clean test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID   | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
| --------- | ---- | ---- | ----------- | --------- | ----------------- | ----------- | ------ |
| 03-01-01  | 01   | 1    | DEP-02      | unit      | `mvn test -Dtest=PomStructureTest#noGuavaRetryingDependencyInPom` | ✅ | ✅ green |
| 03-01-01  | 01   | 1    | DEP-03      | unit      | `mvn test -Dtest=PomStructureTest#noGuavaRetryingVersionPropertyInPom` | ✅ | ✅ green |
| 03-01-01  | 01   | 1    | ENG-05      | unit      | `mvn test -Dtest=PomStructureTest#noGuavaRetryingRefsInPom` | ✅ | ✅ green |
| 03-01-02  | 01   | 1    | VER-01      | integration | `mvn clean test` | ✅ | ✅ green |
| 03-02-01  | 02   | 2    | VER-02      | unit      | `mvn test -Dtest=RetryTimingParityTest` | ✅ | ✅ green |
| 03-02-02  | 02   | 2    | VER-03      | unit      | `mvn test -Dtest=RetryAttemptCountTest` | ✅ | ✅ green |
| 03-02-03  | 02   | 2    | VER-04      | unit      | `mvn test -Dtest=RetryGrepGateTest` | ✅ | ✅ green |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No Wave 0 setup needed.

---

## Manual-Only Verifications

All phase behaviors have automated verification.

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 15s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-09-10

---

## Validation Audit 2026-09-10

| Metric     | Count |
| ---------- | ----- |
| Gaps found | 3     |
| Resolved   | 3     |
| Escalated  | 0     |

### Gap Details

| # | Requirement | Gap | Resolution | Test Method |
| --- | ------------- | ----- | ------------ | ------------- |
| 1 | DEP-02 | No automated test asserted guava-retrying dependency block absent from pom.xml | Added `noGuavaRetryingDependencyInPom()` to PomStructureTest.java | ✅ green |
| 2 | DEP-03 | No automated test asserted guava-retrying.version property absent from pom.xml | Added `noGuavaRetryingVersionPropertyInPom()` to PomStructureTest.java | ✅ green |
| 3 | ENG-05 | Grep gate covered source files but not pom.xml for com.github.rholder references | Added `noGuavaRetryingRefsInPom()` to PomStructureTest.java | ✅ green |

All 3 gaps resolved by the gsd-nyquist-auditor. Full suite: 57 tests pass, BUILD SUCCESS.
