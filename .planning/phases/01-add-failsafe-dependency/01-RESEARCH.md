# Phase 1: Add failsafe dependency - Research

**Researched:** 2026-09-10
**Domain:** Maven dependency management / failsafe.dev retry library
**Confidence:** HIGH

## Summary

Phase 1 is a single mechanical pom.xml edit: add `dev.failsafe:failsafe:3.3.2` as a compile-scope dependency with a `<failsafe.version>` property, adjacent to the existing guava-retrying block. No source files change. The tree must still compile and run on guava-retrying; nothing references `dev.failsafe` in `.java` files after this phase.

failsafe 3.3.2 is the current latest release on Maven Central (published 2023-06-24, groupId `dev.failsafe`, artifactId `failsafe`). It is the successor to the deprecated `net.jodah:failsafe` and is the actively maintained line. Its published POM declares **zero compile/runtime dependencies** — only test-scoped deps (testng, mockito-core, concurrentunit) live in the parent POM. This means adding failsafe alongside guava-retrying introduces no transitive-dependency conflicts: guava-retrying pulls guava + jsr305; failsafe pulls nothing. The two coexist cleanly on the classpath for the duration of Phases 1–2.

The project targets Java 17 (`<source>17</source>`); failsafe 3.3.2 compiles to Java 8 bytecode, so it is fully compatible. Maven 3.9.16 is available in this environment. All CONTEXT.md decisions (D-01 through D-04) are directly achievable with standard Maven property + dependency declaration — no special handling required.

**Primary recommendation:** Add `<failsafe.version>3.3.2</failsafe.version>` to `<properties>` (after `guava-retrying.version`, line 107) and a compile-scope `<dependency>` block for `dev.failsafe:failsafe:${failsafe.version}` immediately after the guava-retrying block (line 182). Verify with `mvn compile` and a grep gate confirming no `dev.failsafe` references in `src/`.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Declare failsafe version as a property `<failsafe.version>3.3.2</failsafe.version>` in `<properties>`, and reference it as `${failsafe.version}` in the dependency block. This matches the existing pom convention where every dependency has a `*.version` property (guava-retrying.version, dropwizard.version, etc.).
- **D-02:** Use `compile` scope (no `<scope>` tag = default compile scope). This matches guava-retrying's declaration (no Scope tag) and ensures failsafe is available at runtime for Phase 2 source changes.
- **D-03:** Add the failsafe dependency block immediately after the existing guava-retrying dependency block (lines 178-182). Keeps retry-related dependencies grouped together, making the Phase 3 removal of guava-retrying a clean adjacent edit.
- **D-04:** Pin to exact version `3.3.2` (not a range or BOM-managed version). Matches the project's existing approach — no BOM manages failsafe, and REQUIREMENTS.md/ROADMAP.md both specify 3.3.2 explicitly.

### Claude's Discretion

- Whether to add a brief XML comment above the failsafe dependency block (e.g. `<!-- replaces guava-retrying in Phase 2 -->`). Not required; planner can decide.

### Deferred Ideas (OUT OF SCOPE)

None - discussion stayed within phase scope. Phase 1 is a single mechanical pom edit; no scope-creep opportunities arose.
</user_constraints>

<phase_requirements>

## Phase Requirements

| ID     | Description                                            | Research Support                                                                 |
| ------ | ------------------------------------------------------ | -------------------------------------------------------------------------------- |
| DEP-01 | Add `dev.failsafe:failsafe:3.3.2` dependency to pom.xml | Coordinates verified on Maven Central; zero-transitive-dep POM; compile-scope declaration pattern confirmed against existing pom conventions (D-01–D-04). |
</phase_requirements>

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
| ------- | ------- | ------- | ------------ |
| dev.failsafe:failsafe | 3.3.2 | Fault-tolerance / retry policy engine; replacement for guava-retrying | Actively maintained successor to net.jodah:failsafe; fluent RetryPolicy API; latest release on Maven Central |

### Supporting

| Library | Version | Purpose | When to Use |
| ------- | ------- | ------- | ----------- |
| (none) | — | failsafe 3.3.2 has zero runtime deps | No supporting libs needed for Phase 1 |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
| ---------- | --------- | -------- |
| dev.failsafe:failsafe 3.3.2 | net.jodah:failsafe (2.x) | net.jodah is the deprecated predecessor — explicitly out of scope per REQUIREMENTS.md. Do not use. |
| dev.failsafe:failsafe 3.3.2 | Resilience4j | Heavier, worse fit for programmatic per-strategy Dropwizard library — out of scope per REQUIREMENTS.md. |
| dev.failsafe:failsafe 3.3.2 | Spring Retry | Out of scope per REQUIREMENTS.md. |

## Architecture Patterns

### Recommended Project Structure

```
pom.xml
├── <properties>            # add <failsafe.version>3.3.2</failsafe.version> after guava-retrying.version (line 107)
└── <dependencies>
    └── guava-retrying block (lines 178-182)
    └── failsafe block (NEW — immediately after, compile scope, no <scope> tag)
```

### Pattern 1: Version-property convention

**What:** Every dependency version is a `<*.version>` property in `<properties>`, referenced as `${name.version}` in the dependency block. No raw version literals in dependency blocks.
**When to use:** Always — this is the established project convention (D-01).
**Example:**

```xml
<!-- In <properties> -->
<failsafe.version>3.3.2</failsafe.version>

<!-- In <dependencies>, after guava-retrying block -->
<dependency>
    <groupId>dev.failsafe</groupId>
    <artifactId>failsafe</artifactId>
    <version>${failsafe.version}</version>
</dependency>
```

*Source: pom.xml lines 96-108 (properties), 178-182 (guava-retrying block) — existing convention.*

### Pattern 2: Compile-scope omission

**What:** Compile-scope dependencies omit the `<scope>` tag entirely (compile is the default). Only `provided`/`test` scopes declare `<scope>` explicitly.
**When to use:** For failsafe (D-02) — matches guava-retrying, commons-lang3, httpclient.
**Example:** See Pattern 1 — no `<scope>` tag present.

### Anti-Patterns to Avoid

- **Adding `<scope>compile</scope>` explicitly:** Breaks the project convention (guava-retrying omits it). Omit the tag.
- **Inlining the version literal `<version>3.3.2</version>`:** Breaks the `*.version` property convention (D-01). Always use `${failsafe.version}`.
- **Placing failsafe far from guava-retrying:** D-03 requires adjacency for clean Phase 3 removal. Keep them together.
- **Importing failsafe into source:** Exit gate forbids any `dev.failsafe` reference in `.java` files. Phase 1 is pom-only.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
| ------- | ----------- | ----------- | --- |
| Retry policy engine | Custom retry loop / keep guava-retrying | dev.failsafe:failsafe 3.3.2 | (Phase 2 concern, but dependency lands here) failsafe handles backoff, attempt caps, exception predicates, async — all edge cases already solved. |

**Key insight:** Phase 1 only lands the dependency; no hand-rolling is possible or needed. The "don't hand-roll" discipline applies starting Phase 2.

## Common Pitfalls

### Pitfall 1: Using the deprecated groupId

**What goes wrong:** Declaring `net.jodah:failsafe` instead of `dev.failsafe:failsafe` pulls the deprecated predecessor.
**Why it happens:** Old tutorials and Stack Overflow answers reference `net.jodah`. The project moved to `dev.failsafe` at the 3.0.0 release.
**How to avoid:** Use exactly `dev.failsafe:failsafe:3.3.2` as locked in REQUIREMENTS.md. Verified against Maven Central maven-metadata.xml (`<latest>3.3.2</latest>`, groupId `dev.failsafe`).
**Warning signs:** `mvn dependency:tree` showing `net.jodah` instead of `dev.failsafe`.

### Pitfall 2: Assuming transitive-dependency conflicts

**What goes wrong:** Worrying that failsafe and guava-retrying will clash over guava versions or other transitives.
**Why it happens:** guava-retrying depends on `com.google.guava:guava [10.+,)` and `findbugs:jsr305`.
**How to avoid:** Verified — failsafe 3.3.2's POM (and its parent `failsafe-parent:3.3.2`) declare **zero compile/runtime dependencies**. Only test-scoped deps (testng, mockito-core, concurrentunit) exist, and those do not propagate. No conflict is possible. The project already pins `guava 33.5.0-jre` via dependencyManagement, which satisfies guava-retrying's `[10.+,)` range.
**Warning signs:** None expected. If `mvn dependency:tree` shows unexpected failsafe transitives, the POM changed — re-verify.

### Pitfall 3: Touching source files

**What goes wrong:** Adding `import dev.failsafe.*` to any `.java` file during Phase 1.
**Why it happens:** Tempting to "try it out" while the dependency is present.
**How to avoid:** Exit gate criterion #2 explicitly forbids `dev.failsafe` in any `.java` file after Phase 1. Source changes belong to Phase 2.
**Warning signs:** `grep -rn "dev.failsafe" src/` returns any match.

### Pitfall 4: Forgetting the version property

**What goes wrong:** Inlining `<version>3.3.2</version>` in the dependency block.
**Why it happens:** Copy-paste from a Maven Central snippet.
**How to avoid:** D-01 mandates `<failsafe.version>3.3.2</failsafe.version>` in `<properties>` + `${failsafe.version}` reference. Matches every other dependency in this pom.

## Code Examples

Verified patterns from official sources:

### Failsafe dependency declaration (Maven Central coordinates)

```xml
<!-- Source: https://repo1.maven.org/maven2/dev/failsafe/failsafe/3.3.2/failsafe-3.3.2.pom -->
<!-- groupId: dev.failsafe, artifactId: failsafe, version: 3.3.2 -->
<!-- Zero compile/runtime dependencies in the POM -->

<!-- Adapted to this project's conventions (D-01, D-02, D-03): -->
<dependency>
    <groupId>dev.failsafe</groupId>
    <artifactId>failsafe</artifactId>
    <version>${failsafe.version}</version>
</dependency>
```

### Version property (matches existing convention)

```xml
<!-- Source: pom.xml lines 96-108 — every dep has a *.version property -->
<failsafe.version>3.3.2</failsafe.version>
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
| ------------ | ---------------- | ------------ | ------ |
| `net.jodah:failsafe` 2.x | `dev.failsafe:failsafe` 3.x | 3.0.0 release (2022) | groupId changed; net.jodah is deprecated. Phase 1 uses dev.failsafe. |
| guava-retrying (com.github.rholder) | dev.failsafe:failsafe | This migration (Phase 1–3) | guava-retrying unmaintained since 2015; failsafe actively maintained. |

**Deprecated/outdated:**

- `net.jodah:failsafe`: deprecated predecessor to `dev.failsafe:failsafe`. Out of scope per REQUIREMENTS.md — must not be used.
- `com.github.rholder:guava-retrying` 2.0.0: unmaintained; removed in Phase 3.

## Open Questions

1. **Is 3.3.2 the final failsafe version for this migration?**
   - What we know: 3.3.2 is the latest release on Maven Central (lastUpdated 2023-06-24). No newer version exists.
   - What's unclear: Whether a 3.3.3+ will release before Phase 2 completes.
   - Recommendation: Pin 3.3.2 as locked (D-04, REQUIREMENTS.md). Do not chase a newer version without an explicit requirements update. failsafe's release cadence is slow (3.3.2 is 3+ years old and still latest).

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
| ---------- | ----------- | --------- | ------- | -------- |
| Maven (mvn) | Build / `mvn compile` exit gate | ✓ | 3.9.16 | — |
| Java 17 | Project compile target | ✓ (implied by mvn) | 17 | — |
| Maven Central (network) | Resolving dev.failsafe:failsafe:3.3.2 | ✓ (verified via repo1.maven.org fetch) | — | — |

**Missing dependencies with no fallback:** None.
**Missing dependencies with fallback:** None.

## Sources

### Primary (HIGH confidence)

- <https://repo1.maven.org/maven2/dev/failsafe/failsafe/maven-metadata.xml> — confirms `<latest>3.3.2</latest>`, `<release>3.3.2</release>`, groupId `dev.failsafe`, artifactId `failsafe`. 3.3.2 is the newest published version.
- <https://repo1.maven.org/maven2/dev/failsafe/failsafe/3.3.2/failsafe-3.3.2.pom> — confirms the artifact POM has no `<dependencies>` section (zero compile/runtime deps).
- <https://repo1.maven.org/maven2/dev/failsafe/failsafe-parent/3.3.2/failsafe-parent-3.3.2.pom> — confirms parent POM declares only test-scoped deps (testng, mockito-core, concurrentunit); `<maven.compiler.source>1.8</maven.compiler.source>` (Java 8 bytecode, compatible with project's Java 17 target).
- <https://repo1.maven.org/maven2/com/github/rholder/guava-retrying/2.0.0/guava-retrying-2.0.0.pom> — confirms guava-retrying's transitive deps (guava `[10.+,)`, jsr305 2.0.2); no overlap with failsafe's (empty) deps.
- `pom.xml` (local) lines 96-108, 178-182 — confirms existing `*.version` property convention and guava-retrying declaration block shape.

### Secondary (MEDIUM confidence)

- (none needed — Maven Central is authoritative for coordinates and POM contents)

### Tertiary (LOW confidence)

- (none)

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH — coordinates and POM verified directly against Maven Central (authoritative source).
- Architecture: HIGH — pom.xml conventions read directly from the project's own pom.xml; no inference.
- Pitfalls: HIGH — transitive-dep claim verified by reading both failsafe's and guava-retrying's published POMs; deprecated-groupId claim verified via maven-metadata.xml.

**Research date:** 2026-09-10
**Valid until:** 2026-10-10 (30 days — failsafe release cadence is slow; 3.3.2 has been latest since 2023-06-24, low drift risk)
