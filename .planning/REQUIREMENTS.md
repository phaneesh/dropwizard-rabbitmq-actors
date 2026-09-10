# Requirements: Dropwizard RabbitMQ Actors

**Defined:** 2025-09-19
**Core Value:** Reliable RabbitMQ message processing with configurable retry strategies that preserve existing behavior while modernizing the underlying retry engine.

## v1 Requirements

Requirements for the guava-retrying → failsafe.dev migration. Each maps to roadmap phases.

### Dependency

- [ ] **DEP-01**: Add `dev.failsafe:failsafe:3.3.2` dependency to pom.xml
- [ ] **DEP-02**: Remove `com.github.rholder:guava-retrying` dependency from pom.xml
- [ ] **DEP-03**: Remove `guava-retrying.version` property from pom.xml

### Retry Engine

- [ ] **ENG-01**: RetryStrategy base class holds a `RetryPolicy<Boolean>` instead of `Retryer<Boolean>`
- [ ] **ENG-02**: RetryStrategy.execute(Callable<Boolean>) runs via `Failsafe.with(policy).get()` (synchronous, blocks calling thread)
- [ ] **ENG-03**: Public API of RetryStrategy (execute signature, throws Exception) unchanged
- [ ] **ENG-04**: All 7 strategy impls rewritten from RetryerBuilder to RetryPolicy.builder()
- [ ] **ENG-05**: No references to `com.github.rholder` remain in source or pom

### Behavioral Parity

- [ ] **PAR-01**: Count-limited strategies stop after exactly maxAttempts (withMaxAttempts, never withMaxRetries)
- [ ] **PAR-02**: Time-limited strategies stop after maxDuration AND disable failsafe's default 3-attempt cap (withMaxRetries(-1))
- [ ] **PAR-03**: Exponential wait strategies produce identical sleep sequences to guava-retrying (delay = 2 * multiplier correction)
- [ ] **PAR-04**: Incremental wait strategies produce identical sleep sequences (initial + (attempt-1)*increment)
- [ ] **PAR-05**: Fixed wait strategies produce identical delays
- [ ] **PAR-06**: NoRetryStrategy executes exactly once (withMaxAttempts(1), no retry-by-default)
- [ ] **PAR-07**: Exception predicate (handleIf) filters retriable exceptions identically to retryIfException
- [ ] **PAR-08**: Synchronous blocking execution preserved (never *Async APIs)
- [ ] **PAR-09**: Boolean return value treated as success signal (false = reject, not retry)

### Verification

- [ ] **VER-01**: Existing tests pass against failsafe-only tree
- [ ] **VER-02**: Timing-parity tests assert exact sleep sequences for exponential/incremental strategies
- [ ] **VER-03**: Attempt-count tests assert exact attempt counts for all strategy types
- [ ] **VER-04**: Grep gate forbids `withMaxRetries` (except -1) and `*Async` in retry package

### Documentation

- [ ] **DOC-01**: Changelog documents exhaustion exception type change (RetryException → FailsafeException/raw throwable)
- [ ] **DOC-02**: Migration note preserves `throws Exception` contract on execute()

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Observability

- **OBS-01**: Per-attempt listener / metrics hooks via failsafe onFailure
- **OBS-02**: Jitter on exponential backoff to prevent thundering herd

### New Retry Types

- **NEW-01**: New RetryType values / async execution model

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
| --------- | -------------- |
| Changing RetryStrategy/RetryStrategyFactory public API | Downstream consumers depend on it |
| Adding new retry strategies | Migration only, no new features |
| Altering RetryType enum values | Config compatibility |
| Upgrading Dropwizard or other dependencies | Isolate the retry library swap |
| Spring Retry / Resilience4j alternatives | Heavier, worse fit for programmatic per-strategy Dropwizard library |
| net.jodah:failsafe (deprecated predecessor) | Deprecated, must not be used |
| Async/stage execution | Breaks synchronous ack/reject contract |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
| ----------- | --------- | ------- |
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
*Requirements defined: 2025-09-19*
*Last updated: 2025-09-19 after initial definition*
