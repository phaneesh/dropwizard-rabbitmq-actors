---
phase: 03-remove-guava-retrying-verify-parity
status: passed
verified: 2026-09-10
requirements: [DEP-02, DEP-03, ENG-05, VER-01, VER-02, VER-03, VER-04]
---

# Phase 3 Verification: Remove guava-retrying and verify parity

## Phase Goal

Remove guava-retrying and its version property from the pom, and verify behavioral parity against the failsafe-only tree. Parity tests must run with guava-retrying gone — running them with both libs present can mask drift.

## Verification Result: PASSED

All 7 requirements verified. All 8 must_have truths confirmed.

## Requirement traceability

| Requirement | Description | Status | Evidence |
| ----------- | ----------- | ------ | -------- |
| DEP-02 | Remove guava-retrying dependency from pom.xml | ✓ | `grep -c "guava-retrying" pom.xml` → 0 |
| DEP-03 | Remove guava-retrying.version property from pom.xml | ✓ | `grep -c "guava-retrying.version" pom.xml` → 0 |
| ENG-05 | No references to com.github.rholder in source or pom | ✓ | `grep -rn "com.github.rholder" src/main pom.xml` → 0 (2 hits in test file are the grep-gate assertion string itself) |
| VER-01 | Existing tests pass against failsafe-only tree | ✓ | `mvn clean test` → 34 existing tests pass, BUILD SUCCESS |
| VER-02 | Timing-parity tests assert exact sleep sequences | ✓ | RetryTimingParityTest: 5 tests pass, asserts [2,4,8]ms exponential, [10,15,20]ms incremental |
| VER-03 | Attempt-count tests assert exact attempt counts | ✓ | RetryAttemptCountTest: 7 tests pass, asserts exact counts for count/time/no-retry/exception-filtering |
| VER-04 | Grep gate forbids withMaxRetries (except -1) and *Async | ✓ | RetryGrepGateTest: 3 tests pass, scans src/main/java/.../retry/ via Files.walk |

## Must-have truths

| # | Truth | Verified |
| - | ----- | -------- |
| 1 | pom.xml contains no com.github.rholder dependency block | ✓ |
| 2 | pom.xml contains no guava-retrying.version property | ✓ |
| 3 | mvn test passes on the failsafe-only tree | ✓ (49 tests) |
| 4 | No source file references com.github.rholder | ✓ |
| 5 | Exponential backoff produces [2*mult, 4*mult, 8*mult] capped at maxTimeBetweenRetries | ✓ |
| 6 | Incremental wait produces [initial, initial+increment, initial+2*increment] | ✓ |
| 7 | Count-limited strategies stop after exactly maxAttempts | ✓ |
| 8 | NoRetryStrategy executes exactly once | ✓ |
| 9 | Time-limited strategies make >3 attempts (withMaxRetries(-1) disables default cap) | ✓ |
| 10 | Non-matching exception is not retried (attempt count = 1) | ✓ |
| 11 | No withMaxRetries (except -1) in retry package source | ✓ |
| 12 | No *Async API calls in retry package source | ✓ |

## Success criteria

1. ✓ `mvn test` passes on a tree with no `com.github.rholder` dependency, no `guava-retrying.version` property, and no source references.
2. ✓ Timing-parity tests assert exact per-attempt sleep sequences for exponential and incremental strategies and fail if the sequence drifts.
3. ✓ Attempt-count tests assert exact total attempt counts for count-limited, time-limited, and no-retry strategies.
4. ✓ A grep gate test fails if `withMaxRetries` (other than `-1`) or any `*Async` API appears in the retry package.

## Exit gate

`mvn test` green on failsafe-only tree; guava-retrying fully removed; parity + grep-gate tests in place and passing. ✓ MET

## Human verification

None required. All verification is automated.
