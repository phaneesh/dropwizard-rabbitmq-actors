# Phase 1: Add failsafe dependency - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md - this log preserves the alternatives considered.

**Date:** 2026-09-10
**Phase:** 1-add-failsafe-dependency
**Areas discussed:** Dependency declaration style, Placement in pom, Version pinning

---

## Dependency declaration style

| Option | Description | Selected |
| ---------- | ---------------------------------- | -------- |
| Version property + `${failsafe.version}` ref | Matches existing pom convention (every dep has `*.version` property) | ✓ |
| Inline version literal | `<version>3.3.2</version>` directly in dependency block | |
| BOM-managed version | Import a failsafe BOM in dependencyManagement | |

**User's choice:** [auto] Version property + `${failsafe.version}` ref (recommended default — matches existing pom convention)
**Notes:** Auto mode selected the recommended option. Every dependency in this pom uses a `*.version` property; inline literals would break the pattern.

---

## Dependency scope

| Option | Description | Selected |
| ---------- | ---------------------------------- | -------- |
| Compile (default, no `<scope>` tag) | Matches guava-retrying declaration; available at runtime for Phase 2 | ✓ |
| Provided | Would require failsafe at compile only — wrong, Phase 2 needs it at runtime | |
| Test | Wrong — failsafe is production retry engine, not test-only | |

**User's choice:** [auto] Compile (default, no `<scope>` tag) (recommended default — matches guava-retrying)
**Notes:** Auto mode selected the recommended option. guava-retrying (the library being replaced) is compile-scope with no `<scope>` tag; failsafe must match.

---

## Placement in pom

| Option | Description | Selected |
| ---------- | ---------------------------------- | -------- |
| Adjacent to guava-retrying block | Keeps retry-related deps grouped; clean Phase 3 removal | ✓ |
| At end of `<dependencies>` | Separates from retry context | |
| Alphabetical by groupId | No existing alphabetical ordering in this pom | |

**User's choice:** [auto] Adjacent to guava-retrying block (recommended default — groups retry deps)
**Notes:** Auto mode selected the recommended option. Placing failsafe right after guava-retrying (lines 178-182) makes the Phase 3 removal a clean adjacent edit.

---

## Version pinning

| Option | Description | Selected |
| ---------- | ---------------------------------- | -------- |
| Exact `3.3.2` | Fixed by REQUIREMENTS.md and ROADMAP.md | ✓ |
| Version range | Not used anywhere in this pom | |
| Latest/SNAPSHOT | Contradicts fixed requirement | |

**User's choice:** [auto] Exact `3.3.2` (recommended default — fixed by requirements)
**Notes:** Auto mode selected the recommended option. Both REQUIREMENTS.md (DEP-01) and ROADMAP.md specify 3.3.2 explicitly.

---

## the agent's Discretion

- Whether to add a brief XML comment above the failsafe dependency block. Not required; planner decides.

## Deferred Ideas

None — Phase 1 is a single mechanical pom edit; no scope-creep opportunities arose.
