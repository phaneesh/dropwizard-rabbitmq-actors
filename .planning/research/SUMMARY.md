# Project Research Summary

**Project:** dropwizard-rabbitmq-actors (retry-engine migration)
**Domain:** Retry-engine replacement in a Java/Dropwizard RabbitMQ actor library (guava-retrying → failsafe.dev)
**Researched:** 2025-09-19
**Confidence:** HIGH

## Executive Summary

This is a like-for-like retry-engine swap inside a self-contained subsystem: replace the unmaintained `com.github.rholder:guava-retrying:2.0.0` with the actively maintained, zero-dependency `dev.failsafe:failsafe:3.3.2`. The migration scope is narrow — one abstract base class (`RetryStrategy`), seven strategy impls in `retry/impl/`, and the `pom.xml`. The public API (`execute(Callable<Boolean>)`), the `RetryType` enum, all JSON config subtypes, and the sole call site (`Handler.handleDelivery`) are untouched. failsafe is a near-perfect drop-in: synchronous `.get()` blocks the calling thread exactly like guava's `BlockStrategies.threadSleepStrategy()`, and `retryIfException(predicate)` maps cleanly to `handleIf(predicate)`.

The recommended approach is a three-phase atomic migration: (1) add failsafe to the pom while keeping guava-retrying, (2) rewrite the base class field type (`Retryer<Boolean>` → `RetryPolicy<Boolean>`) and all seven impl builder chains in one compile-coherent commit, (3) remove guava-retrying and run parity tests. Phase 2 must be atomic because the base constructor signature change breaks every impl at once — there is no intermediate-compiling state.

The dominant risk is **silent behavioral drift**, not compile errors. The two libraries index exponents differently (exponential backoff curve is halved if mapped 1:1), invert default exception handling (guava retries nothing by default; failsafe retries everything up to 3 attempts by default), and split attempt-counting across two APIs (`withMaxAttempts` vs `withMaxRetries`, off-by-one). These regressions pass type-checking and most "did a retry happen?" tests. Mitigation is timing-parity tests asserting exact sleep sequences and exact attempt counts, plus a grep gate forbidding `withMaxRetries` and `*Async` in the new code.

## Key Findings

### Recommended Stack

failsafe 3.3.2 is the clear choice: actively maintained (failsafe-lib/failsafe on GitHub), zero transitive dependencies (no Guava, no Netty), Apache 2.0, Java 8+ bytecode that runs on the project's Java 17 target. Its fluent `RetryPolicy` API covers every strategy variant in the codebase. Alternatives (Spring Retry, Resilience4j) are heavier and a worse fit for a programmatic per-strategy Dropwizard library; `net.jodah:failsafe` is the deprecated predecessor and must not be used.

**Core technologies:**

- `dev.failsafe:failsafe:3.3.2` — resilience/retry policy engine replacing guava-retrying; zero deps, fluent `RetryPolicy`, sync `.get()` matches the blocking contract.
- Maven (existing) — add failsafe, remove guava-retrying + its version property.

### Expected Features

The migration's "feature" is **behavioral parity**. Every existing consumer depends on the current retry semantics; the MVP is preserving them exactly.

**Must have (table stakes):**

- Stop-after-attempt parity (3 count-limited strategies) — `withMaxAttempts(n)`, never `withMaxRetries`.
- Stop-after-delay parity (3 time-limited strategies) — `withMaxDuration` + `withMaxRetries(-1)` to disable failsafe's default 3-attempt cap.
- Exponential-wait formula parity (2 strategies) — `delay = 2 * multiplier` to correct the halved curve.
- Fixed-wait parity (2 strategies) — trivial `withDelay`.
- Incremental-wait parity (2 strategies) — `withDelayFn` computing `initial + (attempt-1)*increment`.
- No-retry parity (1 strategy) — `withMaxAttempts(1)`; must not rely on failsafe's retry-by-default.
- Exception-predicate parity — `handleIf(isRetriable)`; empty set = retry-all preserved.
- Synchronous blocking execution — `.get()` only, never `*Async`.
- `Boolean` return as success signal — no result-based retry (`false` = reject, not retry).
- Exception propagation on exhaustion — last cause escapes to `Handler`'s `catch (Throwable)`.
- `RetryStrategy.execute(Callable<Boolean>)` public signature unchanged.

**Should have (competitive):** None — migration scope only.

**Defer (v2+):**

- Per-attempt listener / metrics hooks (failsafe `onFailure`) — defer until observability need arises.
- Jitter on exponential backoff — defer until thundering-herd observed.
- New `RetryType` values / async execution model — explicitly out of scope (config-compat + ack/reject contract constraints).

### Architecture Approach

The retry engine is a narrow subsystem: one abstract base holds the engine instance and exposes `execute(Callable<Boolean>)`; a factory dispatches on the `RetryType` enum to instantiate one of seven impls, each building the engine from its typed config in its constructor and passing it to `super(...)`. The base owns execution; the impl owns construction. The sole call site is `Handler.handleDelivery()`, which acks on `true`, rejects on `false`, and catches `Throwable` on exception. The migration swaps the engine type held by the base (`Retryer<Boolean>` → `RetryPolicy<Boolean>`) and rewrites each impl's builder chain; everything outside `retry/` (and the pom) is untouched.

**Major components:**

1. `RetryStrategy` (abstract base) — CHANGES: field type + `execute()` body.
2. Seven `retry/impl/*Strategy` — CHANGES: builder chains rewritten from `RetryerBuilder` to `RetryPolicy.builder()`.
3. `RetryStrategyFactory`, `RetryType`, `RetryConfig` + subconfigs, `Handler`, `CommonUtils.isRetriable` — UNCHANGED (compatibility surface).

### Critical Pitfalls

1. **Exponential backoff curve is silently halved** — guava uses `multiplier * 2^attempt`; failsafe uses `delay * factor^(attempt-1)` with the base delay used as-is on the first retry. Mapping `delay = multiplier` halves every sleep. Fix: `delay = 2 * multiplier`, `factor = 2.0`. Verify with a timing-parity test asserting the exact sleep sequence.
2. **Default exception handling is inverted** — guava retries nothing by default; failsafe retries any exception up to 3 attempts by default. `NoRetryStrategy` (no predicate, `stopAfterAttempt(1)`) must map to `withMaxAttempts(1)` explicitly — a bare `RetryPolicy.builder().build()` silently becomes retry-3-times-on-anything.
3. **`withMaxRetries` vs `withMaxAttempts` off-by-one** — `withMaxRetries(n)` = n+1 total attempts; `withMaxAttempts(n)` = n total attempts. guava's `stopAfterAttempt(n)` ≡ `withMaxAttempts(n)`. Never use `withMaxRetries` for this migration.
4. **Time-limited strategies silently cap at 3 attempts** — failsafe's `withMaxDuration` does not disable the default `maxRetries=2`. The two `TimeLimited*` strategies must also set `withMaxRetries(-1)` or they stop early.
5. **Exhaustion exception type changes** — guava throws checked `RetryException`; failsafe throws `FailsafeException` (unchecked) or the raw original. The codebase catches `Throwable` (zero `RetryException` refs), so it's internally safe, but downstream consumers catching `RetryException` get a compile break — document in the changelog and keep `throws Exception` on `execute()`.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Dependency swap

**Rationale:** Adding failsafe first (while keeping guava-retrying) lets the tree compile at every subsequent step; removing guava-retrying first would break compilation until every impl is ported.
**Delivers:** failsafe on the classpath, nothing using it yet; `mvn compile` green.
**Addresses:** Stack readiness.
**Avoids:** Intermediate non-compiling state during Phase 2.

### Phase 2: Base + all seven impls (atomic)

**Rationale:** The base constructor parameter type changes from `Retryer<Boolean>` to `RetryPolicy<Boolean>`; every impl calls `super(...)` with the engine, so the base and all impls must change in one compile-coherent commit. There is no partial-migration state that compiles.
**Delivers:** All retry logic running on failsafe; guava-retrying unused but still on classpath; `mvn compile` green.
**Uses:** `dev.failsafe:failsafe:3.3.2` (`RetryPolicy.builder()`, `handleIf`, `withMaxAttempts`, `withMaxDuration`, `withDelay`, `withBackoff`, `withDelayFn`, `Failsafe.with(...).get()`).
**Implements:** `RetryStrategy` base + 7 strategy impls.
**Avoids:** Pitfalls 1–3, 5–7 (applied during the rewrite); Pitfall 8 (`withMaxRetries(-1)` on time-limited strategies).

### Phase 3: Cleanup + parity verification

**Rationale:** Only after nothing imports `com.github.rholder` can the dependency be removed; parity tests must run against the failsafe-only tree to catch silent drift.
**Delivers:** guava-retrying + version property removed; `mvn test` green with timing-parity and attempt-count tests.
**Addresses:** All table-stakes parity features; Pitfalls 1, 3, 7, 8 (timing tests); Pitfalls 2, 6 (exception tests); Pitfall 5 (grep gate for `Async`).
**Avoids:** Shipping silent schedule drift or inverted default handling.

### Phase 4: Release documentation

**Rationale:** The exhaustion exception type change is a transitive API break for downstream consumers catching `RetryException`; it must be documented, not discovered at compile time by consumers.
**Delivers:** Changelog / migration note calling out the `RetryException` → `FailsafeException`/raw-throwable change and the `throws Exception` contract preservation.
**Addresses:** Pitfall 4 (external impact).

### Phase Ordering Rationale

- Phase 1 before 2: keeping guava-retrying on the classpath during the rewrite keeps the tree compiling at each step; removing it first makes Phase 2 non-compiling until complete.
- Phase 2 is atomic: the base constructor signature change propagates to all impls — no intermediate state compiles.
- Phase 3 after 2: parity tests require the failsafe-only tree; running them with both libs present can mask drift.
- Phase 4 last: documentation follows verified behavior, not assumptions.

### Research Flags

Phases likely needing deeper research during planning:

- **Phase 2 (incremental wait):** `ExecutionContext.getAttemptCount()` basis (0 vs 1) must be confirmed against existing `*IncrementalWaitRetryStrategy` tests — the one mapping that needs an empirical check, not just docs.
- **Phase 2 (exponential wait):** the `delay = 2 * multiplier` correction is math-verified but must be empirically confirmed with a per-attempt sleep-sequence diff.

Phases with standard patterns (skip research-phase):

- **Phase 1, 3, 4:** dependency add/remove and documentation — well-trodden Maven territory.

## Confidence Assessment

| Area         | Confidence | Notes    |
| ------------ | ---------- | -------- |
| Stack        | HIGH       | Version verified via Maven Central `maven-metadata.xml` + POMs; zero transitive deps confirmed; Java 17 compat confirmed. |
| Features     | HIGH       | Feature set derived directly from the 7 existing strategy impls and the sole call site; no speculation. |
| Architecture | HIGH       | Read directly from source: `RetryStrategy`, factory, 7 impls, config, `Handler` call site. |
| Pitfalls     | HIGH       | Every behavioral claim verified against decompiled/source jars of both libraries in `~/.m2`, not from memory. |

**Overall confidence:** HIGH

### Gaps to Address

- **Incremental-wait attempt-count basis:** confirm `ExecutionContext.getAttemptCount()` semantics (1-based completed attempts at first retry) against existing tests during Phase 2 planning; adjust the `-1` if tests show otherwise.
- **`getMultipier()` typo:** preserve the misspelled config field name (config-compatibility constraint); read it as "initial delay / exponential base". Do not "fix" it during migration.
- **Downstream `RetryException` consumers:** unknown how many external consumers catch `RetryException` by type; handle via changelog documentation, not a compatibility shim (out of scope per PROJECT.md).

## Sources

### Primary (HIGH confidence)

- Maven Central `maven-metadata.xml` + POMs for `dev.failsafe:failsafe:3.3.2` — version, zero deps, Java 8+, Apache 2.0.
- failsafe.dev docs (overview, retry, policies) + 3.3.2 javadoc (`RetryPolicyBuilder`, `DelayablePolicyBuilder`) — API signatures, blocking behavior, failure handling.
- guava-retrying 2.0.0 source jar (`~/.m2`) — `Retryer.call()`, `RetryerBuilder`, `WaitStrategies`, `RetryException`.
- failsafe 3.3.2 source jar (`~/.m2`) — `SyncExecutionImpl`, `FailurePolicy`, `FailurePolicyBuilder`, `RetryPolicyBuilder`, `RetryPolicyExecutor`, `FailsafeException`.
- Existing codebase — `RetryStrategy.java`, `retry/impl/*` (7 strategies), `retry/config/*`, `CommonUtils.java`, `base/Handler.java`, `pom.xml`.

### Secondary (MEDIUM confidence)

- failsafe-lib/failsafe GitHub — active maintenance status, CI green.

---
*Research completed: 2025-09-19*
*Ready for roadmap: yes*
