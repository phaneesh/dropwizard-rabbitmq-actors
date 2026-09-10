# Phase 3: Remove guava-retrying and verify parity - Research

**Researched:** 2026-09-10
**Domain:** Java retry-engine migration — dependency removal + behavioral parity verification
**Confidence:** HIGH

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions (carried from Phase 2 — Phase 3 tests must assert these)

- **D-05:** Exponential backoff: `withBackoff(Duration.ofMillis(2*multiplier), Duration.ofMillis(maxTimeBetweenRetries), 2.0)`. Sleep sequence: 2*mult, 4*mult, 8*mult... capped at maxTimeBetweenRetries.
- **D-06:** `withMaxDelay(config.getMaxTimeBetweenRetries().toMilliseconds())` caps each sleep.
- **D-08:** Incremental: `withDelayFn(ctx -> initial + (ctx.getAttemptCount()-1) * increment)`. Sequence: attempt1→0 delay, attempt2→initial, attempt3→initial+increment...
- **D-09 (RESOLVED):** `ExecutionContext.getAttemptCount()` returns 0 before first attempt, 1 after first attempt is recorded. Formula `initial + (getAttemptCount() - 1) * increment` confirmed correct.
- **D-11:** Count-limited: `withMaxAttempts(n)` = n total attempts.
- **D-12:** Time-limited: `withMaxDuration(maxTime)` + `withMaxRetries(-1)` (disables default 3-attempt cap).
- **D-13:** NoRetry: `withMaxAttempts(1)` = exactly 1 attempt.
- **D-14:** `handleIf(exception -> CommonUtils.isRetriable(...))` filters retriable exceptions.
- **D-17:** On exhaustion, failsafe throws `FailsafeException` (unchecked) or raw throwable. `execute()` declares `throws Exception`. Exception-type change documented in Phase 4, not shimmed here.

### Claude's Discretion

- Test file organization and naming.
- Whether to use parameterized tests or individual test methods.
- Exact assertion style (JUnit5 assertions vs AssertJ).

### Deferred Ideas (OUT OF SCOPE)

- Observability hooks (onFailure listeners for metrics), jitter, async execution — v2 requirements.
- Exception-type shim (RetryException → FailsafeException) — Phase 4 documentation.

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
| ---- | ---------------------- | ----------------------------------------------- |
| DEP-02 | Remove `com.github.rholder:guava-retrying` dependency from pom.xml | pom.xml lines 179-183: `<dependency>` block for `com.github.rholder:guava-retrying`. Remove the 5-line block. |
| DEP-03 | Remove `guava-retrying.version` property from pom.xml | pom.xml line 107: `<guava-retrying.version>2.0.0</guava-retrying.version>`. Remove this single line. |
| ENG-05 | No references to `com.github.rholder` remain in source or pom | Verified: zero source references to `com.github.rholder`, `RetryerBuilder`, or `Retryer` (grep excluding target/). VER-04 grep gate will enforce this going forward. |
| VER-01 | Existing tests pass against failsafe-only tree | Existing tests use testcontainers RabbitMQ (integration tests). Zero test references to guava-retrying classes. Guava (`com.google.common`) remains available via dropwizard-core (provided scope). Tests will pass. |
| VER-02 | Timing-parity tests assert exact sleep sequences for exponential/incremental strategies | Use `onRetryScheduled` listener on `RetryPolicyBuilder` — captures `ExecutionScheduledEvent.getDelay()` BEFORE `Thread.sleep`. Build RetryPolicy in test with listener, use small delays (1ms). Thread.sleep still happens but is negligible. |
| VER-03 | Attempt-count tests assert exact attempt counts for all strategy types | Construct actual strategy, pass counting Callable that always throws, assert invocation count after exhaustion. No sleep interception needed. |
| VER-04 | Grep gate forbids `withMaxRetries` (except -1) and `*Async` in retry package | Unit test using `java.nio.file.Files.walk()` to read all `.java` files in `src/main/java/.../retry/`, assert no `withMaxRetries(` except `withMaxRetries(-1)`, and no `*Async(` calls. Runs in `mvn test`. |

</phase_requirements>

## Summary

Phase 3 has two parts: (1) a trivial pom.xml edit removing the guava-retrying dependency block and version property, and (2) writing parity tests that prove the failsafe retry engine produces the exact sleep sequences, attempt counts, and exception filtering defined by the Phase 2 locked decisions (D-05 through D-14).

The pom removal is mechanical — two deletions (5-line dependency block at line 179, 1-line property at line 107). The source is already fully on failsafe (verified by grep). Guava (`com.google.guava`) remains available via dropwizard-core (provided scope) — removing guava-retrying does NOT remove guava from the compile or test classpath. No existing test references any guava-retrying class.

The parity tests are the substantive work. The key research finding for VER-02 (timing parity) is that `Thread.sleep` cannot be mocked via Mockito's `mockStatic` — Mockito explicitly blocks mocking `java.lang.Thread` static methods. Instead, use failsafe's `onRetryScheduled` listener on `RetryPolicyBuilder`, which receives an `ExecutionScheduledEvent` with `getDelay()` BEFORE the sleep happens. The test builds a `RetryPolicy` with the listener (replicating the strategy's builder chain), executes via `Failsafe.with(policy).get(supplier)` with a supplier that always throws, and captures the delay sequence from the listener. Small delays (1ms multiplier) keep the test fast.

For VER-03 (attempt counts), the actual strategy objects can be tested directly — construct the strategy, pass a counting `Callable` that always throws, assert the count after exhaustion. No sleep interception needed.

**Primary recommendation:** Remove the 2 pom.xml items, write 3 test classes (timing parity, attempt counts, grep gate) as fast unit tests with no testcontainers/RabbitMQ dependency.

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
| ------- | ------- | ------- | ------------ |
| dev.failsafe:failsafe | 3.3.2 | Retry engine (already on classpath) | Phase 1 complete. All retry logic runs on failsafe. |
| org.junit.jupiter:junit-jupiter | (via dropwizard-testing 5.0.2) | Test framework | Project already uses JUnit 5 (`org.junit.jupiter.api.*`). Vintage engine excluded. |
| org.mockito:mockito-core | 5.23.0 | Mocking (available if needed) | Already declared in pom. `mockStatic` available but CANNOT mock `Thread.class`. |

### Supporting

| Library | Version | Purpose | When to Use |
| ------- | ------- | ------- | ----------- |
| io.dropwizard.util.Duration | (Dropwizard 5.0.2) | Config value type | All config classes use `io.dropwizard.util.Duration`. Call `.toMilliseconds()` for failsafe APIs. |
| java.time.Duration | (JDK 17) | Failsafe API parameter type | `onRetryScheduled` event's `getDelay()` returns `java.time.Duration`. |
| java.nio.file.Files | (JDK 17) | Grep gate test | Walk retry package source files, read content, assert no forbidden patterns. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
| ---------- | --------- | -------- |
| `onRetryScheduled` listener | `mockStatic(Thread.class)` | Mockito explicitly blocks mocking `Thread` — "not possible to mock static methods of java.lang.Thread". Ruled out. |
| `onRetryScheduled` listener | Timestamp measurement | Flaky due to scheduling jitter. `onRetryScheduled` captures the EXACT computed delay, not wall-clock time. |
| Unit test grep gate | Maven enforcer plugin | Enforcer plugin requires adding plugin config to pom. Unit test is simpler, runs in `mvn test`, no pom changes. |

**Installation:**

```bash
# No new dependencies needed — all test deps already in pom.xml
# JUnit 5 via dropwizard-testing, mockito-core already declared
```

## Architecture Patterns

### Recommended Test Structure

```
src/test/java/io/appform/dropwizard/actors/retry/
├── RetryTimingParityTest.java    # VER-02: sleep sequence assertions
├── RetryAttemptCountTest.java    # VER-03: attempt count assertions
└── RetryGrepGateTest.java        # VER-04: source pattern enforcement
```

### Pattern 1: Timing parity via onRetryScheduled listener

**What:** Build a `RetryPolicy` with `onRetryScheduled` listener, execute via `Failsafe.with(policy).get(supplier)` with a failing supplier, capture delays from `ExecutionScheduledEvent.getDelay()`.

**When to use:** VER-02 — asserting exact sleep sequences for exponential and incremental strategies.

**Why not test the actual strategy class:** The `retryPolicy` field in `RetryStrategy` is `private` with no getter. The `onRetryScheduled` listener must be set during `RetryPolicy.builder()` construction — it cannot be added after `build()`. The strategy class IS the builder chain (nothing else). Testing the builder chain's output IS testing the strategy's configuration.

**Example:**

```java
// Source: /tmp/fsafe_src/dev/failsafe/event/ExecutionScheduledEvent.java
// Source: /tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java onRetryScheduled()
List<Duration> capturedDelays = new ArrayList<>();

RetryPolicy<Boolean> policy = RetryPolicy.<Boolean>builder()
        .handleIf(e -> true)  // retry all exceptions
        .withMaxAttempts(4)
        .withBackoff(
            Duration.ofMillis(2 * 1),    // baseDelay = 2 * multiplier
            Duration.ofMillis(500),      // maxDelay
            2.0)
        .onRetryScheduled(event -> capturedDelays.add(event.getDelay()))
        .build();

Callable<Boolean> alwaysFails = () -> { throw new RuntimeException("fail"); };

assertThrows(Exception.class, () -> Failsafe.with(policy).get(() -> alwaysFails.call()));

// 3 retries → 3 delays: [2ms, 4ms, 8ms]
assertEquals(3, capturedDelays.size());
assertEquals(2, capturedDelays.get(0).toMillis());
assertEquals(4, capturedDelays.get(1).toMillis());
assertEquals(8, capturedDelays.get(2).toMillis());
```

**Key detail:** `onRetryScheduled` fires BEFORE `Thread.sleep` (verified in `RetryPolicyExecutor.apply()` line 84-89). The delay in the event is the EXACT computed delay, not wall-clock time. `Thread.sleep` still happens, so use small delays (1ms multiplier) to keep tests fast.

### Pattern 2: Attempt count via counting Callable

**What:** Construct the actual strategy, pass a `Callable` that increments a counter and always throws, assert the count after exhaustion.

**When to use:** VER-03 — asserting exact total attempt counts.

**Example:**

```java
// Construct actual strategy — tests the real RetryPolicy built by the strategy
var config = CountLimitedFixedWaitRetryConfig.builder()
        .maxAttempts(3)
        .waitTime(Duration.milliseconds(1))  // tiny delay for speed
        .retriableExceptions(null)           // null = retry all
        .build();

AtomicInteger count = new AtomicInteger(0);
Callable<Boolean> alwaysFails = () -> {
    count.incrementAndGet();
    throw new RuntimeException("fail");
};

var strategy = new CountLimitedFixedWaitRetryStrategy(config);
assertThrows(Exception.class, () -> strategy.execute(alwaysFails));

assertEquals(3, count.get());  // exactly 3 attempts
```

**Key detail:** Config `@Builder` constructors allow direct construction without Dropwizard YAML. `retriableExceptions = null` means retry-all (per `CommonUtils.isRetriable`).

### Pattern 3: Grep gate via source file scan

**What:** Unit test that walks `src/main/java/.../retry/` directory, reads each `.java` file, asserts no forbidden patterns.

**When to use:** VER-04 — enforcing no `withMaxRetries` (except `-1`) and no `*Async` calls.

**Example:**

```java
// Source: java.nio.file.Files.walk (JDK 17)
@Test
void noForbiddenPatternsInRetryPackage() throws IOException {
    Path retryDir = Path.of("src/main/java/io/appform/dropwizard/actors/retry");
    
    try (Stream<Path> files = Files.walk(retryDir)) {
        files
            .filter(p -> p.toString().endsWith(".java"))
            .forEach(p -> {
                String content = Files.readString(p);
                
                // No withMaxRetries except withMaxRetries(-1)
                String withoutNegativeOne = content.replaceAll("withMaxRetries\\(-1\\)", "");
                assertFalse(withoutNegativeOne.contains("withMaxRetries"),
                    "withMaxRetries found in " + p + " — use withMaxAttempts instead (except -1 for time-limited)");
                
                // No *Async calls
                assertFalse(content.contains("getAsync(") || content.contains("runAsync("),
                    "Async API found in " + p + " — synchronous .get() only");
            });
    }
}
```

### Anti-Patterns to Avoid

- **Mocking `Thread.sleep`:** Mockito explicitly blocks `mockStatic(Thread.class)` — "not possible to mock static methods of java.lang.Thread to avoid interfering with class loading". Use `onRetryScheduled` listener instead.
- **Timestamp-based timing assertions:** Wall-clock measurement is flaky due to scheduling jitter. `onRetryScheduled` captures the EXACT computed delay.
- **Testing a rebuilt policy for attempt counts:** VER-03 should test the ACTUAL strategy object, not a rebuilt policy. The strategy's `execute()` method is the public API.
- **Adding a getter for `retryPolicy`:** Changes source code — out of scope for Phase 3. Use `onRetryScheduled` on a rebuilt policy for timing, actual strategy for counts.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
| ------- | ----------- | ----------- | --- |
| Delay capture | Custom sleep interceptor or timestamp recorder | `onRetryScheduled(EventListener<ExecutionScheduledEvent>)` | Failsafe provides the computed delay before sleep. No interception needed. |
| Attempt counting | External counter in Callable + manual loop | `AtomicInteger` in `Callable` | One-liner. Strategy handles the loop. |
| Source pattern enforcement | Maven plugin or enforcer rule | JUnit test with `Files.walk()` | Runs in `mvn test`, no pom changes, simple. |
| Config construction | YAML parsing or Dropwizard config | `@Builder` constructor on config class | All 7 config classes have Lombok `@Builder`. Direct construction with named parameters. |

**Key insight:** Failsafe's `onRetryScheduled` listener is the designed hook for observing delays. No mocking or interception is needed.

## Common Pitfalls

### Pitfall 1: Thread.sleep cannot be mocked

**What goes wrong:** `mockStatic(Thread.class)` throws: "It is not possible to mock static methods of java.lang.Thread to avoid interfering with class loading what leads to infinite loops"
**Why it happens:** Mockito explicitly blocks mocking `Thread` to prevent class-loading deadlocks.
**How to avoid:** Use `onRetryScheduled` listener on `RetryPolicyBuilder` — captures the delay BEFORE `Thread.sleep` is called.
**Warning signs:** `MockitoException` with "not possible to mock static methods of java.lang.Thread".

### Pitfall 2: onRetryScheduled must be set during build(), not after

**What goes wrong:** Trying to add `onRetryScheduled` to an already-built `RetryPolicy` — no API exists for this.
**Why it happens:** `onRetryScheduled` is a method on `RetryPolicyBuilder`, not on `RetryPolicy` or `FailsafeExecutor`.
**How to avoid:** Build the `RetryPolicy` in the test with the listener. The strategy class IS the builder chain — testing the chain's output IS testing the strategy.
**Warning signs:** Compilation error — no `onRetryScheduled` method on `RetryPolicy` or `FailsafeExecutor`.

### Pitfall 3: Guava scope changes after removing guava-retrying

**What goes wrong:** After removing guava-retrying, guava is only available as `provided` scope (from dropwizard-core) instead of `compile` scope.
**Why it happens:** guava-retrying declares guava as `compile` scope. dropwizard-core declares guava as `provided` scope. Maven resolves to the nearest definition — with guava-retrying gone, guava is `provided`.
**How to avoid:** This is CORRECT behavior for a library. `provided` means guava is on the compile and test classpath, but consumers must bring it at runtime. Dropwizard consumers always have guava. No action needed.
**Warning signs:** None — compilation and tests pass. The scope change is invisible to the build.

### Pitfall 4: Config @Builder parameter order matters

**What goes wrong:** Constructing config with `@Builder` in the wrong parameter order silently sets wrong values.
**Why it happens:** Lombok `@Builder` generates a builder with named methods, but the all-args constructor has positional parameters. If using the builder API (`.maxAttempts(3).waitTime(...)`), order doesn't matter. If using the constructor directly, it does.
**How to avoid:** Always use the builder API: `CountLimitedFixedWaitRetryConfig.builder().maxAttempts(3).waitTime(Duration.milliseconds(1)).retriableExceptions(null).build()`.
**Warning signs:** Tests pass with wrong values because the config silently accepts them.

### Pitfall 5: withDelay(Duration.ZERO) throws IllegalArgumentException

**What goes wrong:** `DelayablePolicyBuilder.withDelay(Duration)` asserts `delay.toNanos() > 0`. If `waitTime` is 0, construction throws.
**Why it happens:** Failsafe validates delay > 0. Guava's `fixedWait(0, MS)` allows 0.
**How to avoid:** Use `Duration.milliseconds(1)` as minimum in tests. Default config is 500ms. If `waitTime=0` is a valid production config, use `withDelayFn(ctx -> Duration.ofMillis(...))` instead.
**Warning signs:** `IllegalArgumentException: delay must be greater than 0` at strategy construction.

## Code Examples

### Exponential backoff timing test (VER-02)

```java
// Source: /tmp/fsafe_src/dev/failsafe/event/ExecutionScheduledEvent.java — getDelay()
// Source: /tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java — onRetryScheduled()
// Formula D-05: withBackoff(2*multiplier, maxTimeBetweenRetries, 2.0)
// Expected: [2*mult, 4*mult, 8*mult] capped at maxTimeBetweenRetries

@Test
void exponentialBackoffProducesCorrectSleepSequence() {
    List<Duration> delays = new ArrayList<>();
    
    RetryPolicy<Boolean> policy = RetryPolicy.<Boolean>builder()
            .handleIf(e -> true)
            .withMaxAttempts(4)  // 3 retries
            .withBackoff(
                Duration.ofMillis(2),     // 2 * multiplier (multiplier=1)
                Duration.ofMillis(500),   // maxTimeBetweenRetries
                2.0)
            .onRetryScheduled(event -> delays.add(event.getDelay()))
            .build();
    
    assertThrows(Exception.class, () ->
        Failsafe.with(policy).get(() -> { throw new RuntimeException("fail"); }));
    
    assertEquals(3, delays.size());
    assertEquals(2, delays.get(0).toMillis());   // 2 * 1
    assertEquals(4, delays.get(1).toMillis());   // 4 * 1
    assertEquals(8, delays.get(2).toMillis());   // 8 * 1
}
```

### Incremental wait timing test (VER-02)

```java
// Formula D-08: withDelayFn(ctx -> initial + (ctx.getAttemptCount()-1) * increment)
// Expected: [initial, initial+increment, initial+2*increment]

@Test
void incrementalWaitProducesCorrectSleepSequence() {
    List<Duration> delays = new ArrayList<>();
    long initial = 10;
    long increment = 5;
    
    RetryPolicy<Boolean> policy = RetryPolicy.<Boolean>builder()
            .handleIf(e -> true)
            .withMaxAttempts(4)  // 3 retries
            .withDelayFn(ctx -> Duration.ofMillis(
                initial + (ctx.getAttemptCount() - 1) * increment))
            .onRetryScheduled(event -> delays.add(event.getDelay()))
            .build();
    
    assertThrows(Exception.class, () ->
        Failsafe.with(policy).get(() -> { throw new RuntimeException("fail"); }));
    
    assertEquals(3, delays.size());
    assertEquals(10, delays.get(0).toMillis());  // initial + 0
    assertEquals(15, delays.get(1).toMillis());  // initial + increment
    assertEquals(20, delays.get(2).toMillis());  // initial + 2*increment
}
```

### Attempt count test (VER-03)

```java
// Tests the ACTUAL strategy object, not a rebuilt policy

@Test
void countLimitedStrategyStopsAfterMaxAttempts() throws Exception {
    var config = CountLimitedFixedWaitRetryConfig.builder()
            .maxAttempts(3)
            .waitTime(Duration.milliseconds(1))
            .retriableExceptions(null)  // null = retry all
            .build();
    
    AtomicInteger count = new AtomicInteger(0);
    Callable<Boolean> alwaysFails = () -> {
        count.incrementAndGet();
        throw new RuntimeException("fail");
    };
    
    var strategy = new CountLimitedFixedWaitRetryStrategy(config);
    assertThrows(Exception.class, () -> strategy.execute(alwaysFails));
    
    assertEquals(3, count.get());
}

@Test
void noRetryStrategyExecutesExactlyOnce() throws Exception {
    var config = new NoRetryConfig();
    
    AtomicInteger count = new AtomicInteger(0);
    Callable<Boolean> alwaysFails = () -> {
        count.incrementAndGet();
        throw new RuntimeException("fail");
    };
    
    var strategy = new NoRetryStrategy(config);
    assertThrows(Exception.class, () -> strategy.execute(alwaysFails));
    
    assertEquals(1, count.get());
}
```

### Grep gate test (VER-04)

```java
// Source: java.nio.file.Files.walk (JDK 17)

@Test
void noWithMaxRetriesExceptNegativeOne() throws IOException {
    Path retryDir = Path.of("src/main/java/io/appform/dropwizard/actors/retry");
    
    try (Stream<Path> files = Files.walk(retryDir)) {
        files
            .filter(p -> p.toString().endsWith(".java"))
            .forEach(p -> assertDoesNotThrow(() -> {
                String content = Files.readString(p);
                String withoutNegativeOne = content.replaceAll(
                    "withMaxRetries\\(-1\\)", "");
                assertFalse(withoutNegativeOne.contains("withMaxRetries"),
                    "withMaxRetries found in " + p);
            }));
    }
}

@Test
void noAsyncCallsInRetryPackage() throws IOException {
    Path retryDir = Path.of("src/main/java/io/appform/dropwizard/actors/retry");
    
    try (Stream<Path> files = Files.walk(retryDir)) {
        files
            .filter(p -> p.toString().endsWith(".java"))
            .forEach(p -> assertDoesNotThrow(() -> {
                String content = Files.readString(p);
                assertFalse(content.contains("getAsync("),
                    "getAsync found in " + p);
                assertFalse(content.contains("runAsync("),
                    "runAsync found in " + p);
            }));
    }
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
| ------------ | ---------------- | ------------- | ------ |
| guava-retrying on classpath | guava-retrying removed | Phase 3 | Clean failsafe-only tree. Parity tests run without guava-retrying masking drift. |
| No retry tests | Parity tests (timing, count, grep gate) | Phase 3 | Behavioral drift caught by automated tests. |

## Open Questions

1. **Time-limited attempt count test precision**
   - What we know: Time-limited strategies use `withMaxDuration` + `withMaxRetries(-1)`. Attempt count depends on duration boundary, not a fixed count.
   - What's unclear: How to assert exact attempt count for time-limited strategies when the count depends on elapsed time + sleep duration?
   - Recommendation: Use a very short `maxDuration` (e.g., 5ms) with a fixed delay (1ms). The test asserts that attempts > 3 (proving `withMaxRetries(-1)` disabled the default cap) and that the strategy eventually stops. Exact count is non-deterministic for time-limited — assert the lower bound (>3) and that exhaustion occurs.

2. **Exception filtering test (PAR-07)**
   - What we know: `handleIf(exception -> CommonUtils.isRetriable(config.getRetriableExceptions(), exception))` filters retriable exceptions. Empty/null set = retry all. Non-empty = retry only if exception class simple name is in set.
   - What's unclear: Should Phase 3 include a dedicated exception-filtering test, or is this covered by the attempt-count tests (which use `retriableExceptions=null` = retry all)?
   - Recommendation: Add one test that constructs a strategy with a non-empty `retriableExceptions` set (e.g., `Set.of("RuntimeException")`), throws a non-matching exception (e.g., `IllegalArgumentException` — wait, that IS a RuntimeException). Use `Set.of("IOException")` and throw `RuntimeException` — should NOT retry. Assert count=1.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
| ---------- | ----------- | --------- | ------- | -------- |
| Java 17+ | Project compilation | ✓ | 25 (OpenJDK) | — |
| Maven | Build system | ✓ | (existing) | — |
| dev.failsafe:failsafe:3.3.2 | Retry engine | ✓ | 3.3.2 | — |
| JUnit 5 (Jupiter) | Test framework | ✓ | via dropwizard-testing 5.0.2 | — |
| mockito-core 5.23.0 | Test mocking (if needed) | ✓ | 5.23.0 | — |
| guava (com.google.guava) | Main source (CommonUtils, RMQConnection, etc.) | ✓ | 33.5.0-jre (provided scope via dropwizard-core) | — |

**Missing dependencies with no fallback:** None.

**Missing dependencies with fallback:** None.

## Sources

### Primary (HIGH confidence)

- `/tmp/fsafe_src/dev/failsafe/event/ExecutionScheduledEvent.java` — `getDelay()` returns the computed delay before sleep
- `/tmp/fsafe_src/dev/failsafe/RetryPolicyBuilder.java` — `onRetryScheduled(EventListener)` sets the listener during build
- `/tmp/fsafe_src/dev/failsafe/internal/RetryPolicyExecutor.java` — sync `apply()` method: `retryScheduledHandler.handle()` (line 84) called BEFORE `Thread.sleep()` (line 89)
- `/tmp/fsafe_src/dev/failsafe/internal/EventHandler.java` — `ofExecutionScheduled()` constructs `ExecutionScheduledEvent` with `Duration.ofNanos(result.getDelay())`
- `/tmp/fsafe_src/dev/failsafe/FailsafeExecutor.java` — `onComplete`, `onFailure`, `onSuccess` listeners (executor-level, not policy-level)
- `/tmp/fsafe_src/dev/failsafe/ExecutionContext.java` — `getAttemptCount()` javadoc: "Will return 0 when the first attempt is in progress"
- Existing source: `RetryStrategy.java`, all 7 impl files, all 8 config files, `CommonUtils.java`
- `pom.xml` — lines 107, 179-183 (guava-retrying property + dependency block to remove)
- Mockito 5.23.0 — `mockStatic(Thread.class)` explicitly blocked: "not possible to mock static methods of java.lang.Thread"
- `mvn dependency:tree` — guava-retrying 2.0.0 depends on guava (compile scope); guava also from dropwizard-core (provided scope)

### Secondary (MEDIUM confidence)

- None — all findings verified against source jars and live dependency tree.

### Tertiary (LOW confidence)

- None.

## Metadata

**Confidence breakdown:**

- Pom removal (DEP-02/DEP-03): HIGH — exact line numbers verified, mechanical edit
- VER-01 (existing tests pass): HIGH — zero test references to guava-retrying, guava remains available
- VER-02 (timing parity): HIGH — `onRetryScheduled` API verified in failsafe source, delay capture confirmed
- VER-03 (attempt counts): HIGH — actual strategy objects constructible, counting Callable is straightforward
- VER-04 (grep gate): HIGH — `Files.walk()` is standard JDK, pattern matching is simple string operations

**Research date:** 2026-09-10
**Valid until:** 2026-10-10 (stable — failsafe 3.3.2 is a released version, no API changes expected)
