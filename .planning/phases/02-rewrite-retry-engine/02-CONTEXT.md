# Phase 2: Rewrite retry engine (base + all seven impls, atomic) - Context

**Gathered:** 2026-09-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Swap the engine type held by `RetryStrategy` from `Retryer<Boolean>` (guava-retrying) to `RetryPolicy<Boolean>` (failsafe.dev) and rewrite all seven strategy impl builder chains from `RetryerBuilder` to `RetryPolicy.builder()`. After this phase, all retry logic runs on failsafe; guava-retrying remains on the classpath but no source references it. The base and all impls change in one compile-coherent commit — there is no partial-migration state that compiles. Behavioral parity is built in here, not deferred to Phase 3.

</domain>

<decisions>
## Implementation Decisions

### Base class (RetryStrategy)

- **D-01:** Change the field type from `Retryer<Boolean> retryer` to `RetryPolicy<Boolean> retryPolicy`. Keep the field `private final`. (ENG-01)
- **D-02:** `execute(Callable<Boolean>)` body changes from `retryer.call(callable)` to `Failsafe.with(retryPolicy).get(callable)`. This is the synchronous blocking call — `.get()` blocks the calling thread exactly like guava's `retryer.call()`. Never use `*Async` APIs. (ENG-02, PAR-08)
- **D-03:** The `execute(Callable<Boolean>) throws Exception` public signature is byte-identical — no parameter, return type, or throws-clause change. (ENG-03)
- **D-04:** Constructor stays `protected RetryStrategy(RetryPolicy<Boolean> retryPolicy)`. The parameter type change is internal (impls pass the policy via `super(...)`); no external caller constructs `RetryStrategy` directly except the 7 impls and the factory.

### Exponential backoff formula mapping (2 strategies: CountLimited + TimeLimited)

- **D-05:** Use `withBackoff(long baseDelay, double factor)` where `baseDelay = 2 * config.getMultipier()` (in ms) and `factor = 2.0`. This corrects the halved-curve pitfall: guava computes `multiplier * 2^attempt`; failsafe computes `baseDelay * factor^(attempt-1)`. Setting `baseDelay = 2 * multiplier` makes attempt-1 sleep = `2 * multiplier`, matching guava. (PAR-03)
- **D-06:** Apply `withMaxDelay(config.getMaxTimeBetweenRetries().toMilliseconds())` to cap each sleep at the configured maximum, matching guava's `exponentialWait(multiplier, maxTimeBetweenRetries, MS)` second parameter. (PAR-03)
- **D-07:** Preserve the `getMultipier()` typo — read it as "exponential base / initial delay". Do not rename the config field (config-compatibility constraint). (carried from PROJECT.md constraints)

### Incremental wait implementation (2 strategies: CountLimited + TimeLimited)

- **D-08:** Use `withDelayFn(attempt -> initial + (attempt - 1) * increment)` where `initial = config.getInitialWaitTime().toMilliseconds()` and `increment = config.getWaitIncrement().toMilliseconds()`. The `attempt` parameter in failsafe's delay function is 1-based (attempt 1 is the first execution). This produces: attempt 1 → 0 delay (no delay before first try), attempt 2 → `initial`, attempt 3 → `initial + increment`, etc. — matching guava's `incrementingWait(initial, increment)` sequence. (PAR-04)
- **D-09:** **Research flag (must confirm during planning):** `ExecutionContext.getAttemptCount()` basis (0 vs 1) must be empirically confirmed against failsafe 3.3.2 source. The delay function receives an `ExecutionContext`; if `getAttemptCount()` is 0-based at the first retry point, the formula adjusts to `initial + attempt * increment`. Planner must verify against the failsafe jar in `~/.m2` before finalizing. (PAR-04, open risk from research SUMMARY.md)

### Fixed wait implementation (2 strategies: CountLimited + TimeLimited)

- **D-10:** Use `withDelay(config.getWaitTime().toMilliseconds())` — trivial 1:1 mapping from guava's `fixedWait(ms, MS)`. (PAR-05)

### Count-limited stop strategy (3 strategies)

- **D-11:** Use `withMaxAttempts(config.getMaxAttempts())` — never `withMaxRetries`. guava's `stopAfterAttempt(n)` ≡ failsafe's `withMaxAttempts(n)` (both = n total attempts). `withMaxRetries(n)` = n+1 attempts (off-by-one pitfall). (PAR-01)

### Time-limited stop strategy (3 strategies)

- **D-12:** Use `withMaxDuration(config.getMaxTime().toMilliseconds(), TimeUnit.MILLISECONDS)` AND `withMaxRetries(-1)` to disable failsafe's default 3-attempt cap. Without `withMaxRetries(-1)`, failsafe silently stops after 3 attempts even if the duration hasn't elapsed. (PAR-02)

### No-retry strategy (1 strategy)

- **D-13:** Use `withMaxAttempts(1)` explicitly. Do NOT rely on a bare `RetryPolicy.builder().build()` — failsafe's default is retry-up-to-3-times-on-any-exception, which would silently break NoRetryStrategy. No exception predicate needed (NoRetryStrategy has none in guava). (PAR-06)

### Exception predicate mapping (6 strategies with predicates)

- **D-14:** Map guava's `retryIfException(predicate)` to failsafe's `handleIf(predicate)`, where the predicate is `exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception)`. The `CommonUtils.isRetriable` logic is unchanged: empty/null set = retry-all (returns true), non-empty = retry only if exception class simple name is in the set. (PAR-07)
- **D-15:** `handleIf` replaces `retryIfException` — same semantics (predicate decides retry on exception). NoRetryStrategy gets no `handleIf` (matches guava: no predicate). (PAR-07)

### Boolean return value semantics

- **D-16:** The `Callable<Boolean>` return value is a success signal, not a retry trigger. `true` = success (ack), `false` = reject (no retry). failsafe does NOT retry on `false` return by default — only on exceptions. This matches guava's behavior (guava's `Retryer.call()` does not retry on return value, only on exception predicate). No `handleResult(false)` needed. (PAR-09)

### Exhaustion exception propagation

- **D-17:** On exhaustion, failsafe throws `FailsafeException` (unchecked, wrapping the last cause) or the raw original throwable. `execute()` declares `throws Exception`, so the raw throwable propagates to `Handler.handleDelivery()`'s `catch (Throwable)`. The codebase has zero `RetryException` references internally (verified in research), so no internal catch-site breaks. The exception-type change is documented in Phase 4, not shimmed here. (out of scope to shim; DOC-01 covers it)
- **D-18:** Do not catch/wrap `FailsafeException` inside `execute()` — let it propagate raw. Adding a try/catch to re-wrap as a checked exception would change the exception contract and add unrequested complexity. The `throws Exception` declaration already covers checked propagation; unchecked `FailsafeException` propagates naturally.

### the agent's Discretion

- Whether to add a brief javadoc note on `RetryStrategy` explaining the failsafe swap. Not required; planner can decide.
- Exact import ordering / static-import style for failsafe APIs. Follow existing codebase conventions.
- Whether to inline the delay function lambda or extract a static helper for incremental wait. Inline is simpler; planner decides if readability suffers.

</decisions>

<canonical_refs>

## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project requirements

- `.planning/REQUIREMENTS.md` §ENG-01..ENG-05 - Engine rewrite requirements (base field type, execute body, public API unchanged, all 7 impls rewritten, no guava refs)
- `.planning/REQUIREMENTS.md` §PAR-01..PAR-09 - Behavioral parity requirements (attempt counts, sleep sequences, exception filtering, blocking, boolean semantics)
- `.planning/PROJECT.md` §Constraints - Public API unchanged; behavioral parity; add failsafe remove guava-retrying; Java 17+
- `.planning/ROADMAP.md` §Phase 2 - Goal, rationale, success criteria, exit gate

### Research (parity-critical)

- `.planning/research/SUMMARY.md` §Critical Pitfalls - All 5 pitfalls (exponential halved curve, inverted default handling, withMaxRetries off-by-one, time-limited 3-attempt cap, exhaustion exception type)
- `.planning/research/SUMMARY.md` §Research Flags - Incremental-wait `getAttemptCount()` basis must be empirically confirmed during planning

### Source (the code being rewritten)

- `src/main/java/io/appform/dropwizard/actors/retry/RetryStrategy.java` - Base class: field `Retryer<Boolean>`, `execute()` calls `retryer.call(callable)`
- `src/main/java/io/appform/dropwizard/actors/retry/impl/*.java` - All 7 strategy impls with `RetryerBuilder` chains
- `src/main/java/io/appform/dropwizard/actors/retry/config/*.java` - Config classes (note `getMultipier()` typo to preserve)
- `src/main/java/io/appform/dropwizard/actors/utils/CommonUtils.java` - `isRetriable()` predicate logic (unchanged)
- `src/main/java/io/appform/dropwizard/actors/retry/RetryStrategyFactory.java` - Factory dispatch (unchanged, no guava refs)

### Failsafe API (in local maven cache)

- `~/.m2/repository/dev/failsafe/failsafe/3.3.2/failsafe-3.3.2-sources.jar` - Source for `RetryPolicy`, `RetryPolicyBuilder`, `Failsafe`, `ExecutionContext`, `FailsafeException` — planner must verify `getAttemptCount()` basis and `withBackoff`/`withDelayFn` signatures against this

No external ADRs or design docs exist for this project. Requirements are fully captured in the decisions and research above.

</canonical_refs>

<code_context>

## Existing Code Insights

### Reusable Assets

- `CommonUtils.isRetriable(Set<String>, Throwable)` - The exception predicate, reused verbatim as the `handleIf` predicate. Empty set = retry-all; non-empty = match by simple class name. No change needed.
- `RetryStrategyFactory` - Dispatches on `RetryType` enum to construct impls. Unchanged — no guava imports, just `new XxxStrategy(config)`.
- All 7 config classes - Unchanged. Each provides the values (maxAttempts, maxTime, waitTime, multipier, initialWaitTime, waitIncrement, maxTimeBetweenRetries, retriableExceptions) that the rewritten builder chains consume.

### Established Patterns

- **Constructor-delegates-to-super pattern:** Each impl builds the engine in its constructor and passes to `super(...)`. This pattern is preserved — only the builder API changes (`RetryerBuilder` → `RetryPolicy.builder()`).
- **Duration.toMilliseconds():** All configs use Dropwizard `Duration` and call `.toMilliseconds()` for the guava API. failsafe APIs accept `long` ms or `(long, TimeUnit)` — same `.toMilliseconds()` values feed in.
- **BlockStrategies.threadSleepStrategy():** guava's blocking strategy. Replaced by failsafe's synchronous `.get()` which blocks the calling thread by default — no explicit block strategy needed.

### Integration Points

- `RetryStrategy.execute(Callable<Boolean>)` is called from `Handler.handleDelivery()` (the sole call site). The signature and `throws Exception` contract must not change. `Handler` acks on `true`, rejects on `false`, catches `Throwable` on exception.
- The 7 impl constructors are called only by `RetryStrategyFactory.create()`. The constructor signatures don't change (each takes its typed config); only the body (builder chain) changes.
- `pom.xml` already has `dev.failsafe:failsafe:3.3.2` (Phase 1 complete). No pom change in this phase.

</code_context>

<specifics>
## Specific Ideas

- The `delay = 2 * multiplier` correction is math-verified in research but MUST be empirically confirmed with a per-attempt sleep-sequence test (that test lands in Phase 3, but the formula is locked here).
- The incremental-wait `getAttemptCount()` basis is the one mapping that needs an empirical check during planning, not just docs — research flagged it explicitly.
- `getMultipier()` typo is a config-compatibility constraint — preserve it, do not "fix" it.
- This is the only phase where source references `dev.failsafe`. Phase 3 removes guava-retrying from the pom; Phase 4 documents the exception-type change.

</specifics>

<deferred>
## Deferred Ideas

None - discussion stayed within phase scope. The phase is a like-for-like engine swap with parity construction; no new capabilities were proposed. Observability hooks (onFailure listeners), jitter, and async execution are explicitly out of scope (v2 requirements, tracked in REQUIREMENTS.md).

</deferred>

---

*Phase: 02-rewrite-retry-engine*
*Context gathered: 2026-09-10*
