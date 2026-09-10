---
phase: 01
slug: add-failsafe-dependency
status: approved
nyquist_compliant: true
wave_0_complete: true
created: 2026-09-10
---

# Phase 1 - Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property               | Value                                   |
| ---------------------- | --------------------------------------- |
| **Framework**          | JUnit 5 (Jupiter) + JUnit Vintage engine |
| **Config file**        | pom.xml (surefire plugin)               |
| **Quick run command**  | `mvn test -Dtest=PomStructureTest`     |
| **Full suite command** | `mvn test`                              |
| **Estimated runtime**  | ~60 seconds                             |

---

## Sampling Rate

- **After every task commit:** Run `mvn test -Dtest=PomStructureTest`
- **After every plan wave:** Run `mvn test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** ~60 seconds

---

## Per-Task Verification Map

| Task ID   | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
| --------- | ---- | ---- | ----------- | --------- | ----------------- | ----------- | ------ |
| 01-01-T1  | 01   | 1    | DEP-01      | unit      | `mvn test -Dtest=PomStructureTest` | ✅ | ✅ green |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

### Requirement → Test Cross-Reference

| Requirement | Criterion | Test | Status |
| ------------- | ----------- | ------- | -------- |
| DEP-01 | failsafe 3.3.2 resolvable on classpath | `RetryTimingParityTest` (imports `dev.failsafe.Failsafe`; won't compile without dep) | COVERED |
| DEP-01 | `mvn test` passes (all existing green) | Full suite (52 tests, 0 failures) | COVERED |
| DEP-01 | `<failsafe.version>3.3.2</failsafe.version>` property present | `PomStructureTest#failsafeVersionPropertyPresent` | COVERED |
| DEP-01 | dep block uses `${failsafe.version}`, no `<scope>` | `PomStructureTest#failsafeDependencyUsesPropertyVersionAndNoScopeTag` | COVERED |
| DEP-01 | no `net.jodah` deprecated groupId | `PomStructureTest#noDeprecatedNetJodahGroupId` | COVERED |
| DEP-01 | no `dev.failsafe` in `src/` (Phase 1 exit gate) | — | N/A — intentionally superseded by Phase 2 (engine rewrite uses failsafe) |

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No Wave 0 stubs needed.

---

## Manual-Only Verifications

All phase behaviors have automated verification.

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 60s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-09-10

---

## Validation Audit 2026-09-10

| Metric     | Count |
| ---------- | ----- |
| Gaps found | 3     |
| Resolved   | 3     |
| Escalated  | 0     |

**Auditor:** gsd-nyquist-auditor (general-purpose, session model)
**Result:** ## GAPS FILLED — 3/3 resolved

### Tests Created

| # | File | Type | Command |
| --- | ---- | ---- | ------- |
| 1 | `src/test/java/io/appform/dropwizard/actors/retry/PomStructureTest.java` | unit | `mvn test -Dtest=PomStructureTest` |

### Verification

- `mvn test -Dtest=PomStructureTest` → exit 0 (Tests run: 3, Failures: 0, Errors: 0)
- `mvn test` (full suite) → exit 0 (Tests run: 52, Failures: 0, Errors: 0)
