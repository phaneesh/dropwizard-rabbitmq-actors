# Roadmap: Dropwizard RabbitMQ Actors

**Created:** 2025-09-19
**Granularity:** coarse (4 phases)
**Core Value:** Reliable RabbitMQ message processing with configurable retry strategies that preserve existing behavior while modernizing the underlying retry engine.

## Overview

A like-for-like retry-engine swap: replace the unmaintained `com.github.rholder:guava-retrying:2.0.0` with the actively maintained `dev.failsafe:failsafe:3.3.2`. The migration is narrow in surface (one abstract base, seven strategy impls, the pom) but high in behavioral-drift risk. Phases are ordered to keep the tree compiling at every step and to verify parity only against the failsafe-only tree.

**Total v1 requirements:** 23
**Mapped to phases:** 23
**Unmapped:** 0

---

## Phase 1: Add failsafe dependency

**Goal:** Put failsafe on the classpath without touching any source. The tree still compiles and runs on guava-retrying; nothing uses failsafe yet.

**Rationale:** Adding failsafe first (while keeping guava-retrying) lets the tree compile at every subsequent step. Removing guava-retrying first would break compilation until every impl is ported. This phase delivers stack readiness only.

**Requirements covered:**

- DEP-01 — Add `dev.failsafe:failsafe:3.3.2` dependency to pom.xml

**Success criteria:**

1. `mvn compile` succeeds with `dev.failsafe:failsafe:3.3.2` resolvable on the classpath.
2. No source file references `dev.failsafe` yet (failsafe is present but unused — verified by grep).
3. All existing tests pass unchanged (guava-retrying still the active engine).

**Exit gate:** `mvn test` green; failsafe declared in pom; zero source references to `dev.failsafe`.

---

## Phase 2: Rewrite retry engine (base + all seven impls, atomic)

**Goal:** Swap the engine type held by `RetryStrategy` from `Retryer<Boolean>` to `RetryPolicy<Boolean>` and rewrite all seven strategy impl builder chains from `RetryerBuilder` to `RetryPolicy.builder()`. After this phase, all retry logic runs on failsafe; guava-retrying is unused but still on the classpath.

**Rationale:** The base constructor parameter type changes, and every impl calls `super(...)` with the engine — so the base and all impls must change in one compile-coherent commit. There is no partial-migration state that compiles. Behavioral parity is built in here, not deferred: each impl's `RetryPolicy` is constructed to reproduce the exact sleep sequence, attempt count, and exception filtering of its guava predecessor.

**Requirements covered:**

- ENG-01 — RetryStrategy base class holds a `RetryPolicy<Boolean>` instead of `Retryer<Boolean>`
- ENG-02 — RetryStrategy.execute(Callable<Boolean>) runs via `Failsafe.with(policy).get()` (synchronous, blocks calling thread)
- ENG-03 — Public API of RetryStrategy (execute signature, throws Exception) unchanged
- ENG-04 — All 7 strategy impls rewritten from RetryerBuilder to RetryPolicy.builder()
- PAR-01 — Count-limited strategies stop after exactly maxAttempts (withMaxAttempts, never withMaxRetries)
- PAR-02 — Time-limited strategies stop after maxDuration AND disable failsafe's default 3-attempt cap (withMaxRetries(-1))
- PAR-03 — Exponential wait strategies produce identical sleep sequences to guava-retrying (delay = 2 * multiplier correction)
- PAR-04 — Incremental wait strategies produce identical sleep sequences (initial + (attempt-1)*increment)
- PAR-05 — Fixed wait strategies produce identical delays
- PAR-06 — NoRetryStrategy executes exactly once (withMaxAttempts(1), no retry-by-default)
- PAR-07 — Exception predicate (handleIf) filters retriable exceptions identically to retryIfException
- PAR-08 — Synchronous blocking execution preserved (never *Async APIs)
- PAR-09 — Boolean return value treated as success signal (false = reject, not retry)

**Success criteria:**

1. `mvn compile` succeeds with `RetryStrategy` holding a `RetryPolicy<Boolean>` and all seven impls building via `RetryPolicy.builder()`.
2. `RetryStrategy.execute(Callable<Boolean>)` signature and `throws Exception` contract are byte-identical to the pre-migration API (no public-API change).
3. Existing tests pass against the failsafe-driven engine (guava-retrying still on classpath but no source references it).
4. A timing-parity check (assert-based or test) confirms the exponential-wait sleep sequence matches the `delay = 2 * multiplier` correction, and the incremental-wait sequence matches `initial + (attempt-1)*increment`.

**Exit gate:** `mvn test` green; zero source references to `com.github.rholder` or `RetryerBuilder`; `RetryStrategy` public API unchanged.

---

## Phase 3: Remove guava-retrying and verify parity

**Goal:** Remove guava-retrying and its version property from the pom, and verify behavioral parity against the failsafe-only tree. Parity tests must run with guava-retrying gone — running them with both libs present can mask drift.

**Rationale:** Only after nothing imports `com.github.rholder` can the dependency be removed. Parity verification (timing sequences, attempt counts, exception handling, grep gate) belongs here, against the clean tree, to catch the silent-drift regressions that pass type-checking.

**Requirements covered:**

- DEP-02 — Remove `com.github.rholder:guava-retrying` dependency from pom.xml
- DEP-03 — Remove `guava-retrying.version` property from pom.xml
- ENG-05 — No references to `com.github.rholder` remain in source or pom
- VER-01 — Existing tests pass against failsafe-only tree
- VER-02 — Timing-parity tests assert exact sleep sequences for exponential/incremental strategies
- VER-03 — Attempt-count tests assert exact attempt counts for all strategy types
- VER-04 — Grep gate forbids `withMaxRetries` (except -1) and `*Async` in retry package

**Success criteria:**

1. `mvn test` passes on a tree with no `com.github.rholder` dependency, no `guava-retrying.version` property, and no source references to either.
2. Timing-parity tests assert exact per-attempt sleep sequences for exponential and incremental strategies and fail if the sequence drifts.
3. Attempt-count tests assert exact total attempt counts for count-limited, time-limited, and no-retry strategies.
4. A grep gate (test or CI check) fails if `withMaxRetries` (other than `-1`) or any `*Async` API appears in the retry package.

**Exit gate:** `mvn test` green on failsafe-only tree; guava-retrying fully removed; parity + grep-gate tests in place and passing.

---

## Phase 4: Release documentation

**Goal:** Document the behavioral and transitive-API changes for downstream consumers. The exhaustion exception type changes from guava's checked `RetryException` to failsafe's `FailsafeException` (unchecked) or the raw original throwable — a compile break for any consumer catching `RetryException` by type. This must be documented, not discovered at compile time by consumers.

**Rationale:** Documentation follows verified behavior, not assumptions. Only after Phase 3 proves parity on the failsafe-only tree can the changelog accurately describe what changed.

**Requirements covered:**

- DOC-01 — Changelog documents exhaustion exception type change (RetryException → FailsafeException/raw throwable)
- DOC-02 — Migration note preserves `throws Exception` contract on execute()

**Success criteria:**

1. A changelog entry names the `RetryException` → `FailsafeException`/raw-throwable change and flags it as a transitive compile break for consumers catching `RetryException` by type.
2. A migration note states that `RetryStrategy.execute()` retains its `throws Exception` declaration and that synchronous blocking behavior is unchanged.
3. The documentation references the verified failsafe version (3.3.2) and the removed guava-retrying coordinate, so consumers can reproduce the swap.

**Exit gate:** Changelog + migration note written and consistent with the verified Phase 3 behavior.

---

## Phase Ordering Rationale

- **Phase 1 before 2:** Keeping guava-retrying on the classpath during the rewrite keeps the tree compiling at each step; removing it first makes Phase 2 non-compiling until complete.
- **Phase 2 is atomic:** The base constructor signature change propagates to all impls — no intermediate state compiles.
- **Phase 3 after 2:** Parity tests require the failsafe-only tree; running them with both libs present can mask drift.
- **Phase 4 last:** Documentation follows verified behavior, not assumptions.

## Traceability

| Requirement | Phase | Status |
| ----------- | ----- | ------ |
| DEP-01 | Phase 1 | Pending |
| DEP-02 | Phase 3 | Pending |
| DEP-03 | Phase 3 | Pending |
| ENG-01 | Phase 2 | Pending |
| ENG-02 | Phase 2 | Pending |
| ENG-03 | Phase 2 | Pending |
| ENG-04 | Phase 2 | Pending |
| ENG-05 | Phase 3 | Pending |
| PAR-01 | Phase 2 | Pending |
| PAR-02 | Phase 2 | Pending |
| PAR-03 | Phase 2 | Pending |
| PAR-04 | Phase 2 | Pending |
| PAR-05 | Phase 2 | Pending |
| PAR-06 | Phase 2 | Pending |
| PAR-07 | Phase 2 | Pending |
| PAR-08 | Phase 2 | Pending |
| PAR-09 | Phase 2 | Pending |
| VER-01 | Phase 3 | Pending |
| VER-02 | Phase 3 | Pending |
| VER-03 | Phase 3 | Pending |
| VER-04 | Phase 3 | Pending |
| DOC-01 | Phase 4 | Pending |
| DOC-02 | Phase 4 | Pending |

**Coverage:**

- v1 requirements: 23 total
- Mapped to phases: 23
- Unmapped: 0 ✓

---
*Created: 2025-09-19*
*Derived from: REQUIREMENTS.md, research/SUMMARY.md*
