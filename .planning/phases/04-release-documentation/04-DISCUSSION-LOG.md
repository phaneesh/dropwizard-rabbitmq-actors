# Phase 4: Release documentation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md - this log preserves the alternatives considered.

**Date:** 2026-09-10
**Phase:** 04-release-documentation
**Areas discussed:** Changelog entry, Migration note, Documentation location, Tone and depth

> **Note:** This phase ran in `--auto` mode. All gray areas were auto-selected and the recommended option chosen for each question. The log below records the alternatives that were considered and the auto-selected choice.

---

## Changelog entry

| Option | Description | Selected |
| ---------- | ---------------------------------- | -------- |
| New top section in CHANGELOG.md | Append a `## <version>` section at top of existing changelog naming the RetryException → FailsafeException change | ✓ |
| Separate BREAKING.md file | Dedicated breaking-changes file | |

**Auto-selected:** New top section in CHANGELOG.md (recommended — matches existing changelog convention, single source of truth)
**Notes:** Entry must state verified facts (guava removed, failsafe 3.3.2 added, 15 parity tests green) and flag FailsafeException as unchecked vs guava's checked RetryException.

---

## Migration note

| Option | Description | Selected |
| ---------- | ---------------------------------- | -------- |
| New MIGRATION.md at repo root | Dedicated discoverable file for the exception-type compile break | ✓ |
| Inline in CHANGELOG.md | Bury the migration detail inside the changelog entry | |
| Inline in README.md | Add to the 4-line readme | |

**Auto-selected:** New MIGRATION.md at repo root (recommended — exception-type change is a compile break, deserves a discoverable file; README too thin, changelog buries it)
**Notes:** Migration note states `execute()` retains `throws Exception`, synchronous blocking unchanged, references failsafe 3.3.2 + removed guava-retrying coordinate. Link from changelog entry to MIGRATION.md.

---

## Tone and depth

| Option | Description | Selected |
| ---------- | ---------------------------------- | -------- |
| Concise and factual | 4-6 changelog bullets, short migration sections, no marketing language | ✓ |
| Detailed with code samples | Full before/after snippets, deep explanation | |

**Auto-selected:** Concise and factual (recommended — like-for-like engine swap, not a feature release)
**Notes:** A tiny before/after `catch (RetryException)` → `catch (Exception)` snippet in the migration note is recommended but left to planner discretion.

---

## the agent's Discretion

- Exact heading/bullet formatting in changelog (match existing loose style)
- Optional one-line retry-engine pointer in README.md → MIGRATION.md
- Whether migration note includes a before/after code snippet (recommended, not required)

## Deferred Ideas

None - discussion stayed within phase scope.
