# Phase 1: Add failsafe dependency - Context

**Gathered:** 2026-09-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Add `dev.failsafe:failsafe:3.3.2` to the Maven classpath without touching any source. The tree still compiles and runs on guava-retrying; nothing uses failsafe yet. This phase delivers stack readiness only — no source files reference `dev.failsafe` after this phase.

</domain>

<decisions>
## Implementation Decisions

### Dependency declaration style

- **D-01:** Declare failsafe version as a property `<failsafe.version>3.3.2</failsafe.version>` in `<properties>`, and reference it as `${failsafe.version}` in the dependency block. This matches the existing pom convention where every dependency has a `*.version` property (guava-retrying.version, dropwizard.version, etc.).
- **D-02:** Use `compile` scope (no `<scope>` tag = default compile scope). This matches guava-retrying's declaration (no scope tag) and ensures failsafe is available at runtime for Phase 2 source changes.

### Placement in pom

- **D-03:** Add the failsafe dependency block immediately after the existing guava-retrying dependency block (lines 178-182). Keeps retry-related dependencies grouped together, making the Phase 3 removal of guava-retrying a clean adjacent edit.

### Version pinning

- **D-04:** Pin to exact version `3.3.2` (not a range or BOM-managed version). Matches the project's existing approach — no BOM manages failsafe, and REQUIREMENTS.md/ROADMAP.md both specify 3.3.2 explicitly.

### the agent's Discretion

- Whether to add a brief XML comment above the failsafe dependency block (e.g. `<!-- replaces guava-retrying in Phase 2 -->`). Not required; planner can decide.

</decisions>

<canonical_refs>

## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project requirements

- `.planning/REQUIREMENTS.md` §DEP-01 - "Add `dev.failsafe:failsafe:3.3.2` dependency to pom.xml" — the single requirement this phase covers
- `.planning/PROJECT.md` §Constraints - "Add `dev.failsafe:failsafe`, remove `com.github.rholder:guava-retrying`"; Java 17+ target
- `.planning/ROADMAP.md` §Phase 1 - Goal, rationale, success criteria, exit gate

### Build configuration

- `pom.xml` §properties (lines 99-107) - existing `*.version` property convention to follow
- `pom.xml` §dependencies (lines 178-182) - existing guava-retrying declaration block; failsafe goes adjacent

No external specs or ADRs exist for this project. Requirements are fully captured in the decisions above.

</canonical_refs>

<code_context>

## Existing Code Insights

### Reusable Assets

- `pom.xml` `<properties>` block: every dependency has a `*.version` property — failsafe follows the same pattern with `<failsafe.version>3.3.2</failsafe.version>`.

### Established Patterns

- **Version property convention:** All dependency versions are declared as properties and referenced via `${name.version}`. No raw version literals in dependency blocks.
- **Scope convention:** Compile-scope dependencies (guava-retrying, commons-lang3, httpclient) omit the `<scope>` tag; provided/test scopes declare it explicitly. Failsafe is compile-scope → omit the tag.

### Integration Points

- New `<failsafe.version>` property goes in `<properties>` (after `guava-retrying.version` on line 107, or grouped with retry-related props).
- New `<dependency>` block goes in `<dependencies>` immediately after the guava-retrying block (line 182).
- No source files change. No imports added. `dev.failsafe` must NOT appear in any `.java` file after this phase (exit gate criterion #2).

</code_context>

<specifics>
## Specific Ideas

- The version `3.3.2` is fixed by REQUIREMENTS.md and ROADMAP.md — do not substitute a newer failsafe version without an explicit requirements update.
- This is the only phase where both `guava-retrying` and `failsafe` coexist on the classpath. Phase 2 rewrites source to use failsafe; Phase 3 removes guava-retrying.

</specifics>

<deferred>
## Deferred Ideas

None - discussion stayed within phase scope. Phase 1 is a single mechanical pom edit; no scope-creep opportunities arose.

</deferred>

---

*Phase: 01-add-failsafe-dependency*
*Context gathered: 2026-09-10*
