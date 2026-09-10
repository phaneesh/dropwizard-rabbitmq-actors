# Architecture Research

**Domain:** Retry engine integration (guava-retrying → failsafe.dev migration)
**Researched:** 2025-09-19
**Confidence:** HIGH

## Standard Architecture

### System Overview

The retry engine is a self-contained subsystem inside the actor framework. It is
NOT the system architecture — it is the integration surface for the retry library
swap. The boundary is narrow: one abstract base class, one factory, seven impls,
and a single call site.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Actor Layer (unchanged)                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────────┐       │
│  │ Actor    │  │ Unmanaged│  │ Handler.handleDelivery()  │       │
│  │          │  │ BaseActor │  │  ← SOLE caller of execute │       │
│  └────┬─────┘  └────┬─────┘  └────────────┬─────────────┘       │
│       │             │                     │                      │
├───────┴─────────────┴─────────────────────┴──────────────────────┤
│                   Retry Subsystem (MIGRATION SCOPE)              │
│  ┌────────────────────┐  ┌──────────────────────────────────┐   │
│  │ RetryStrategyFactory│→ │ RetryStrategy (abstract base)     │   │
│  │  switch(RetryType)  │  │  holds Retryer<Boolean>  ──┐    │   │
│  └──────────┬─────────┘  │  execute(Callable<Boolean>) │    │   │
│             │            └────────────────────────────┼────┘   │
│  ┌──────────┴─────────────────────────────────────────┴────┐   │
│  │  7 strategy impls (retry/impl/)                          │   │
│  │  each builds Retryer via RetryerBuilder in constructor   │   │
│  └──────────────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────────────┤
│                    Config Layer (unchanged)                       │
│  ┌──────────────┐  ┌──────────────────────────────────────┐     │
│  │ RetryType    │  │ RetryConfig (abstract, @JsonSubTypes) │     │
│  │ enum (7 vals)│  │  + retriableExceptions: Set<String>    │     │
│  └──────────────┘  └──────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Migration Status |
| --------- | -------------- | ---------------- |
| `RetryStrategy` (abstract base) | Holds retry engine instance; `execute(Callable<Boolean>)` delegates to engine | **CHANGES** — field type + execute body |
| `RetryStrategyFactory` | `create(RetryConfig)` → switch on `RetryType` → instantiate impl | **UNCHANGED** — no engine refs |
| 7 `retry/impl/*Strategy` | Build retry engine from config in constructor, pass to `super(...)` | **CHANGES** — builder calls rewritten |
| `RetryType` enum | Dispatch key (7 values) | **UNCHANGED** |
| `RetryConfig` + 7 subconfigs | Polymorphic JSON config, `retriableExceptions` set | **UNCHANGED** |
| `Handler.handleDelivery()` | Sole caller: `retryStrategy.execute(handleCallable)` → ack/reject | **UNCHANGED** |
| `CommonUtils.isRetriable()` | Exception predicate: empty set = retry all; else match simple class name | **UNCHANGED** (reused by impls) |

## Recommended Project Structure

```
src/main/java/io/appform/dropwizard/actors/
├── retry/
│   ├── RetryStrategy.java          # CHANGES: Retryer → RetryPolicy field
│   ├── RetryStrategyFactory.java   # unchanged
│   ├── RetryType.java              # unchanged
│   ├── config/                     # all unchanged
│   └── impl/                       # 7 files CHANGES: RetryerBuilder → RetryPolicy.builder
└── base/
    └── Handler.java                # unchanged (sole execute() caller)
```

### Structure Rationale

- **retry/** is the entire migration scope. Nothing outside it (except the pom) changes.
- **impl/** holds the 7 strategies — each is a 1:1 port from `RetryerBuilder` chain to `RetryPolicy.builder()` chain.
- **config/** and **RetryType** are config-compatibility surface: changing them breaks consumer YAML. Untouched.

## Architectural Patterns

### Pattern 1: Base-holds-engine, impl-builds-engine

**What:** `RetryStrategy` is an abstract base that holds the retry engine instance
(`Retryer<Boolean>`) and exposes `execute(Callable<Boolean>)`. Each impl's
constructor builds the engine from its config and passes it to `super(...)`.
The base owns execution; the impl owns construction.

**When to use:** When the execution contract is uniform but construction varies
per strategy. This is exactly the case here.

**Trade-offs:** Pro — single execute path, easy to swap engine in one place.
Con — the base field type is the engine type, so swapping engines means touching
the base. This is the one change that propagates.

**Current code:**

```java
public abstract class RetryStrategy {
    private final Retryer<Boolean> retryer;          // guava-retrying type
    protected RetryStrategy(Retryer<Boolean> retryer) { this.retryer = retryer; }
    public boolean execute(Callable<Boolean> callable) throws Exception {
        return retryer.call(callable);               // guava-retrying call
    }
}
```

### Pattern 2: Factory dispatches on enum, casts config

**What:** `RetryStrategyFactory.create(RetryConfig)` switches on
`config.getType()` (a `RetryType` enum), casts the abstract config to the
concrete subtype, and `new`s the matching impl. No engine types leak into the
factory — it only knows configs and impl classes.

**Trade-offs:** Pro — factory is engine-agnostic, survives the swap unchanged.
Con — `switch` without default + `return null` at the end (existing smell, not
ours to fix).

### Pattern 3: Polymorphic config via Jackson @JsonSubTypes

**What:** `RetryConfig` is abstract with `@JsonSubTypes` keyed on the `type`
property (a `RetryType` name string). Jackson deserializes YAML/JSON config
into the right concrete config class. The `type` field doubles as the factory
dispatch key.

**Trade-offs:** This binds config shape to `RetryType` enum values — changing
enum values breaks consumer configs. **Do not touch the enum.**

## Data Flow

### Retry Execution Flow

```
RabbitMQ message arrives
    ↓
Handler.handleDelivery()
    ↓ builds Callable<Boolean> handleCallable (returns true=success)
    ↓
retryStrategy.execute(handleCallable)        ← SOLE integration point
    ↓
RetryStrategy.execute()  →  retryer.call(callable)   [CURRENT]
                         →  Failsafe.with(policy).get(supplier)  [TARGET]
    ↓
    ├── callable returns true  → execute returns true  → Handler basicAck
    ├── callable returns false → execute returns false → Handler basicReject
    └── callable throws retriable exception → retry per policy
            └── exhausted → exception propagates → Handler catch (Throwable)
```

### Config → Strategy Construction Flow

```
RetryConfig (YAML, @JsonSubTypes deserialization)
    ↓ carries RetryType + retriableExceptions + strategy-specific fields
    ↓
RetryStrategyFactory.create(config)
    ↓ switch(config.getType())
    ↓
new XxxRetryStrategy((XxxConfig) config)
    ↓ constructor builds Retryer via RetryerBuilder  [CURRENT]
    ↓  → RetryPolicy.builder()                        [TARGET]
    ↓
super(retryer)  /  super(retryPolicy)
    ↓
RetryStrategy field set → ready for execute()
```

### Key Data Flows

1. **Execution:** `Handler.handleDelivery()` is the **only** caller of
   `execute()` in the entire codebase (grep-confirmed). `UnmanagedConsumer`
   holds a `RetryStrategy` field but routes execution through `Handler`. The
   integration surface is exactly one call site.
2. **Exception filtering:** Each impl (except `NoRetryStrategy`) calls
   `CommonUtils.isRetriable(config.getRetriableExceptions(), exception)` as
   the retry predicate. Empty set → retry on any exception. Non-empty → retry
   only if `exception.getClass().getSimpleName()` is in the set. This predicate
   is reused unchanged in the failsafe port.
3. **Result semantics:** `execute()` returns `boolean`. `true` → ack,
   `false` → reject. The callable's boolean return is the retry *success*
   signal: guava-retrying does NOT retry on `false` (only on exceptions), so
   `false` is a terminal failure. **This must be preserved** — see parity note.

## Proposed failsafe.dev Integration

### Base class change

```java
// TARGET RetryStrategy
import dev.failsafe.Failsafe;
import dev.failsafe.RetryPolicy;
import java.util.concurrent.Callable;

public abstract class RetryStrategy {
    private final RetryPolicy<Boolean> retryPolicy;

    protected RetryStrategy(RetryPolicy<Boolean> retryPolicy) {
        this.retryPolicy = retryPolicy;
    }

    public boolean execute(Callable<Boolean> callable) throws Exception {
        return Failsafe.with(retryPolicy).get(callable::call);
    }
}
```

- `Retryer<Boolean>` → `RetryPolicy<Boolean>`
- `retryer.call(callable)` → `Failsafe.with(retryPolicy).get(supplier)`
- `.get()` (not `.run()`) because we need the `Boolean` return value.
- `callable::call` adapts `Callable` to failsafe's `CheckedSupplier`.

### Strategy impl mapping (per builder call)

| guava-retrying | failsafe.dev | Parity concern |
| -------------- | ------------ | -------------- |
| `RetryerBuilder.<Boolean>newBuilder()` | `RetryPolicy.<Boolean>builder()` | none |
| `.retryIfException(e -> isRetriable(...))` | `.handleIf(e -> isRetriable(...))` | **verify**: failsafe `handleIf` retries on matching exceptions; non-matching propagate. Confirm non-retriable exceptions are NOT swallowed. |
| `StopStrategies.stopAfterAttempt(n)` | `.withMaxAttempts(n)` | **verify off-by-one**: both = n total attempts including first. guava "stop after attempt n" = n attempts. failsafe `maxAttempts` = n attempts. Confirm equal. |
| `StopStrategies.stopAfterDelay(ms, MS)` | `.withMaxDuration(Duration.ofMillis(ms))` | **verify**: guava stops when cumulative delay since start ≥ maxDelay; failsafe `maxDuration` is wall-clock from start. Semantically equivalent for blocking execution. |
| `WaitStrategies.fixedWait(t, MS)` | `.withDelay(Duration.ofMillis(t))` | none |
| `WaitStrategies.exponentialWait(mult, max, MS)` | `.withBackoff(initial, max, MS).withBackoffFactor(mult)` | **HIGH RISK** — see below |
| `WaitStrategies.incrementingWait(init, MS, incr, MS)` | `.withDelayFn(ctx -> init + (ctx.getAttemptCount()-1)*incr)` | **no built-in** — needs delay function |
| `BlockStrategies.threadSleepStrategy()` | (implicit — failsafe blocks synchronously) | none |

### Exponential wait parity (HIGH RISK)

guava-retrying `exponentialWait(multiplier, maxTimeBetweenRetries, unit)`:
wait time = `multiplier * 2^(attempt-1)` ms, capped at `maxTimeBetweenRetries`.
The first retry waits `multiplier * 1` ms.

failsafe `withBackoff(initialDelay, maxDelay, unit)` + `withBackoffFactor(factor)`:
wait = `initialDelay * factor^(attempt-1)`, capped at `maxDelay`.

To match: set `initialDelay = multiplier` (ms), `factor = 2.0`, `maxDelay = maxTimeBetweenRetries`.
**This must be verified empirically** — the exact wait sequence per attempt
must be compared before/after. Flag for a parity test.

### Incrementing wait parity

No failsafe built-in. Use `.withDelayFn(ctx -> Duration.ofMillis(
initialWaitMs + (ctx.getAttemptCount() - 1) * incrementMs))`. Verify
attempt-count base (1-based vs 0-based) against guava's incrementing wait.

### Exception propagation shape

guava-retrying wraps exhausted retries in `RetryException` (and non-retriable
exceptions may be wrapped). failsafe propagates the original exception (or
wraps in `CompletionException` for async). `Handler.handleDelivery()` catches
`Throwable`, so propagation shape does not break the caller — but any test
asserting on exception type would. Flag for test review.

## Component Boundaries: What Changes vs. What Stays

### CHANGES (migration scope)

| File | Change |
| ---- | ------ |
| `pom.xml` | Add `dev.failsafe:failsafe`, remove `com.github.rholder:guava-retrying` + version property |
| `retry/RetryStrategy.java` | Field `Retryer<Boolean>` → `RetryPolicy<Boolean>`; `execute()` body → `Failsafe.with(...).get(...)` |
| `retry/impl/NoRetryStrategy.java` | `RetryerBuilder` → `RetryPolicy.builder()` with `.withMaxAttempts(1)` |
| `retry/impl/CountLimitedExponentialWaitRetryStrategy.java` | Rewrite builder chain |
| `retry/impl/CountLimitedFixedWaitRetryStrategy.java` | Rewrite builder chain |
| `retry/impl/CountLimitedIncrementalWaitRetryStrategy.java` | Rewrite builder chain (delay function) |
| `retry/impl/TimeLimitedExponentialWaitRetryStrategy.java` | Rewrite builder chain |
| `retry/impl/TimeLimitedFixedWaitRetryStrategy.java` | Rewrite builder chain |
| `retry/impl/TimeLimitedIncrementalWaitRetryStrategy.java` | Rewrite builder chain (delay function) |

### UNCHANGED (compatibility surface)

| File | Why it stays |
| ---- | ------------ |
| `RetryStrategyFactory.java` | No engine types — only knows configs + impl classes |
| `RetryType.java` | Enum values are config keys; changing breaks consumer YAML |
| `RetryConfig.java` + 7 subconfigs | Config shape is consumer contract |
| `Handler.java` | Calls `execute(Callable<Boolean>)` — contract preserved |
| `UnmanagedConsumer.java` | Holds `RetryStrategy` field, routes through Handler |
| `CommonUtils.isRetriable()` | Reused as-is in impl predicates |
| All actor/base/connectivity code | No engine references |

## Suggested Build Order

The build order is dictated by compile dependencies: the base class field type
change forces all impls to change in the same compile unit. There is no way to
land a partial migration that compiles — `RetryStrategy`'s constructor signature
change breaks all 7 impls at once.

```
Phase 1: Dependency swap
  └── pom.xml: add dev.failsafe:failsafe (keep guava-retrying temporarily)
        → verify mvn compile (both libs present, nothing uses failsafe yet)

Phase 2: Base + all impls (atomic — must compile together)
  ├── RetryStrategy.java: Retryer → RetryPolicy, execute() → Failsafe.with().get()
  └── 7 impls: rewrite each constructor's builder chain
        → verify mvn compile (guava-retrying now unused but still on classpath)

Phase 3: Cleanup
  └── pom.xml: remove guava-retrying dep + version property
        → verify mvn compile + mvn test

Phase 4: Parity verification
  ├── exponential wait: compare wait sequence per attempt (guava vs failsafe)
  ├── incrementing wait: compare wait sequence per attempt
  ├── stopAfterAttempt vs withMaxAttempts: confirm attempt count parity
  └── exception propagation: confirm non-retriable exceptions still reach Handler
```

**Why atomic Phase 2:** `RetryStrategy`'s constructor parameter type changes
from `Retryer<Boolean>` to `RetryPolicy<Boolean>`. Every impl calls
`super(...)` with the engine instance. Changing the base without changing all
impls = compile failure. There is no intermediate-compiling state. Land Phase 2
as one commit.

**Why keep guava-retrying through Phase 2:** removing it first means Phase 2
won't compile until every impl is ported. Keeping it lets the tree compile at
each step. Remove only after nothing imports `com.github.rholder`.

## Scaling Considerations

Not applicable — this is a library migration, not a capacity problem. The retry
engine executes synchronously on the consumer thread (blocking sleep). That
does not change: failsafe in synchronous mode also blocks. No concurrency model
change.

## Anti-Patterns

### Anti-Pattern 1: Introducing a RetryPolicy abstraction layer

**What people do:** Add a `RetryPolicyProvider` interface or a
`RetryEngineAdapter` to "make future swaps easier."
**Why it's wrong:** This is a one-time migration, not a pluggable-engine
system. The existing base+factory+impl pattern already is the abstraction.
Adding another layer is speculative flexibility for a swap that won't recur.
**Do this instead:** Swap `Retryer` → `RetryPolicy` directly. The base class
IS the seam.

### Anti-Pattern 2: Changing RetryType enum or config shape

**What people do:** Rename enum values or restructure configs "while we're in
there."
**Why it's wrong:** `RetryType` values are Jackson `@JsonSubTypes` keys in
consumer YAML. Renaming silently breaks every downstream config at runtime.
**Do this instead:** Touch nothing in `RetryType`, `RetryConfig`, or any
subconfig. Migration is engine-only.

### Anti-Pattern 3: Assuming exponential wait formulas match

**What people do:** Map `exponentialWait(mult, max)` to `withBackoff(...)` 1:1
and ship.
**Why it's wrong:** guava-retrying and failsafe use different formulas and
different parameter meanings for exponential backoff. A 1:1 mapping can
produce different wait sequences — a behavioral regression invisible to
compile.
**Do this instead:** Write a parity check that prints the wait sequence for
attempts 1..N under both engines (before removing guava) and diff them.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
| ------- | ------------------- | ----- |
| Maven Central | `dev.failsafe:failsafe` dependency | Add to `pom.xml` `<dependencies>`; pick a version compatible with Java 17 / Dropwizard 5.x |
| Maven Central | `com.github.rholder:guava-retrying:2.0.0` | Remove after migration; also remove `<guava-retrying.version>` property |

### Internal Boundaries

| Boundary | Communication | Notes |
| -------- | ------------- | ----- |
| Handler ↔ RetryStrategy | `execute(Callable<Boolean>) throws Exception` | Contract preserved; only the engine behind it changes |
| Factory ↔ impls | `new XxxStrategy(config)` constructor | Unchanged |
| impls ↔ config | direct field access on typed config | Unchanged |
| impls ↔ CommonUtils | static `isRetriable(Set, Throwable)` | Reused as failsafe predicate |

## Sources

- `src/main/java/io/appform/dropwizard/actors/retry/RetryStrategy.java` — base class (current)
- `src/main/java/io/appform/dropwizard/actors/retry/RetryStrategyFactory.java` — factory
- `src/main/java/io/appform/dropwizard/actors/retry/RetryType.java` — enum
- `src/main/java/io/appform/dropwizard/actors/retry/impl/*.java` — 7 strategy impls
- `src/main/java/io/appform/dropwizard/actors/retry/config/RetryConfig.java` — polymorphic config base
- `src/main/java/io/appform/dropwizard/actors/base/Handler.java:101` — sole `execute()` call site
- `src/main/java/io/appform/dropwizard/actors/utils/CommonUtils.java` — `isRetriable` predicate
- `pom.xml:107,179-181` — guava-retrying dependency
- `.planning/PROJECT.md` — migration scope and constraints

---
*Architecture research for: retry engine integration (guava-retrying → failsafe.dev)*
*Researched: 2025-09-19*
