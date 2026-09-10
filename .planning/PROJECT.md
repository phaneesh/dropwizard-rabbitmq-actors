# Dropwizard RabbitMQ Actors

## What This Is

A Dropwizard bundle providing an actor abstraction over RabbitMQ for Dropwizard-based projects. It supplies message-handling actors with retry, dead-letter, and concurrency primitives so applications can consume and process RabbitMQ messages with backpressure, failure handling, and configurable retry semantics. Maintained by Appform (groupId `io.appform.dropwizard.actors`), version 5.0.x.

## Core Value

Reliable RabbitMQ message processing with configurable retry strategies that preserve existing behavior while modernizing the underlying retry engine.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. Inferred from existing codebase. -->

- ✓ Actor abstraction over RabbitMQ for Dropwizard - existing
- ✓ Configurable retry strategies (no-retry, count-limited, time-limited × exponential/incremental/fixed waits) - existing
- ✓ Retry strategy factory dispatching by RetryType - existing
- ✓ Exception-predicate-based retry decisions (retriable exception filtering) - existing
- ✓ Dead-letter and failure handling - existing
- ✓ Concurrency and backpressure controls - existing

### Active

<!-- Current scope. Building toward these. -->

- ✓ Replace `guava-retrying` (com.github.rholder:guava-retrying:2.0.0) with `failsafe.dev` library — engine swapped in Phase 2 (failsafe 3.3.2 on classpath, all retry logic runs on it)
- [ ] Remove all references to `guava-retrying` (imports, dependency, version property)
- ✓ Preserve existing retry semantics: stop-after-attempt, exponential/incremental/fixed waits, time-limited stops, exception-predicate filtering, blocking (thread-sleep) execution — validated in Phase 2 (parity construction: 2*multiplier backoff, withMaxAttempts/withMaxRetries(-1), handleIf, synchronous .get())

### Out of Scope

- Changing the public API of RetryStrategy / RetryStrategyFactory - consumers depend on it
- Adding new retry strategies - migration only, no new features
- Altering RetryType enum values - config compatibility
- Upgrading Dropwizard or other dependencies - isolate the retry library swap

## Context

- **Existing retry engine:** `com.github.rholder:guava-retrying:2.0.0` — an unmaintained library wrapping Guava's `Retryer`.
- **Usage surface:** `RetryStrategy` (base, holds `Retryer<Boolean>`, calls `retryer.call(callable)`) + 7 strategy impls in `retry/impl/`, each building a `Retryer` via `RetryerBuilder` with stop/wait/block strategies and an exception predicate.
- **Target engine:** `failsafe.dev` (dev.failsafe:failsafe) — actively maintained, fluent `RetryPolicy` API.
- **Build:** Maven (`pom.xml`), Java, Dropwizard 5.x.
- **GPG note:** repo has `commit.gpgsign=true` but no signing key in this environment; commits use `-c commit.gpgsign=false`.

## Constraints

- **Compatibility**: Public API of `RetryStrategy` and `RetryStrategyFactory` must remain unchanged - downstream consumers depend on it
- **Behavior**: Retry semantics (attempt limits, wait durations, exception filtering) must be behaviorally identical post-migration
- **Dependency**: Add `dev.failsafe:failsafe`, remove `com.github.rholder:guava-retrying`
- **Java version**: Must match existing project target (Dropwizard 5.x → Java 17+)

## Key Decisions

| Decision | Rationale | Outcome |
| -------- | --------- | ------- |
| Use failsafe.dev as replacement | Actively maintained, fluent RetryPolicy API, drop-in for guava-retrying's Retryer pattern | ✓ Validated Phase 2 — all retry logic runs on failsafe, tests green |
| Preserve RetryStrategy base class shape | Consumers depend on `execute(Callable<Boolean>)` contract | ✓ Validated Phase 2 — `execute(Callable<Boolean>) throws Exception` signature byte-identical |
| Map RetryerBuilder → RetryPolicy per strategy | 1:1 strategy migration minimizes behavioral drift | ✓ Validated Phase 2 — all 7 impls rewritten, parity construction (2*multiplier, withMaxAttempts, withMaxRetries(-1)) |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):

1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):

1. Full review of all sections
2. Core Value check - still the right priority?
3. Audit Out of Scope - reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-09-10 after Phase 2 completion*
