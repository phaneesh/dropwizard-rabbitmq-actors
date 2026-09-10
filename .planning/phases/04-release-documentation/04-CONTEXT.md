# Phase 4: Release documentation - Context

**Gathered:** 2026-09-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Document the behavioral and transitive-API changes from the guava-retrying → failsafe.dev migration for downstream consumers. The exhaustion exception type changes from guava's checked `RetryException` to failsafe's unchecked `FailsafeException` (or the raw original throwable) — a compile break for any consumer catching `RetryException` by type. This phase writes a changelog entry and a migration note. No source code changes; no compatibility shim.

</domain>

<decisions>
## Implementation Decisions

### Changelog entry (DOC-01)

- **D-01:** Add a new `## 5.0.2-1` (or next) section at the top of `CHANGELOG.md` documenting the retry-engine swap. The section names the `RetryException` → `FailsafeException`/raw-throwable change and flags it as a transitive compile break for consumers catching `RetryException` by type.
- **D-02:** The changelog entry states the verified facts: guava-retrying (`com.github.rholder:guava-retrying:2.0.0`) removed; failsafe (`dev.failsafe:failsafe:3.3.2`) added; all retry semantics (attempt counts, wait sequences, exception filtering, synchronous blocking) behaviorally identical — proven by 15 parity tests on the failsafe-only tree.
- **D-03:** The entry explicitly calls out that `FailsafeException` is unchecked (extends `RuntimeException`), whereas guava's `RetryException` was checked — so consumers who had `catch (RetryException e)` will get a compile error (type no longer exists) and must switch to catching the broader `Exception`/`Throwable` they already handle via `execute()`'s `throws Exception`.

### Migration note (DOC-02)

- **D-04:** Write a migration note (location decided below) stating that `RetryStrategy.execute()` retains its `throws Exception` declaration and that synchronous blocking behavior is unchanged — consumers' existing `try/catch` around `execute()` continues to work; only a typed `catch (RetryException)` needs updating.
- **D-05:** The migration note references the verified failsafe version (3.3.2) and the removed guava-retrying coordinate (`com.github.rholder:guava-retrying:2.0.0`) so consumers can reproduce the swap in their own downstream builds if needed.

### Documentation location

- **D-06:** Put the changelog entry in `CHANGELOG.md` (existing file, append a new top section). Put the migration note in a new `MIGRATION.md` at repo root — a dedicated, discoverable file is better than burying the exception-type change inside the changelog, and `README.md` is too thin (4 lines) to host it. Link to `MIGRATION.md` from the changelog entry.

### Tone and depth

- **D-07:** Keep both docs concise and factual — this is a like-for-like engine swap, not a feature release. Changelog entry: 4-6 bullets. Migration note: short sections (what changed, what to do, version refs). No marketing language.

### the agent's Discretion

- Exact heading style / bullet formatting in the changelog (match the existing loose style in `CHANGELOG.md`).
- Whether to add a one-line "retry engine" note to `README.md` pointing at `MIGRATION.md`. Optional; planner decides.
- Whether the migration note includes a tiny before/after code snippet for the `catch (RetryException)` → `catch (Exception)` change. Recommended but not required.

</decisions>

<canonical_refs>

## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project requirements

- `.planning/REQUIREMENTS.md` §DOC-01, §DOC-02 - Documentation requirements (changelog exception-type change, migration note preserves throws Exception)
- `.planning/PROJECT.md` §Constraints - Public API unchanged; behavioral parity; Java 17+
- `.planning/ROADMAP.md` §Phase 4 - Goal, rationale, success criteria, exit gate

### Verified behavior (source of truth for what the docs describe)

- `.planning/phases/03-remove-guava-retrying-verify-parity/03-VERIFICATION.md` - Phase 3 verification: all parity tests pass on failsafe-only tree (49 tests), exception-type change confirmed
- `.planning/phases/02-rewrite-retry-engine/02-CONTEXT.md` §D-17, §D-18 - Exhaustion exception propagation decisions: `FailsafeException` (unchecked) or raw throwable; `execute()` does not catch/wrap it; `throws Exception` covers propagation

### Source (the code being documented)

- `src/main/java/io/appform/dropwizard/actors/retry/RetryStrategy.java` - `execute(Callable<Boolean>) throws Exception` calls `Failsafe.with(retryPolicy).get(...)`; signature unchanged from pre-migration
- `src/main/java/io/appform/dropwizard/actors/base/Handler.java` §handleDelivery (lines ~100-117) - Sole call site of `execute()`; catches `Throwable` on exception (acks/rejects) — unaffected by exception-type change
- `pom.xml` - `dev.failsafe:failsafe:3.3.2` present; `com.github.rholder:guava-retrying` removed; `guava-retrying.version` property removed

### Existing docs (files being edited)

- `CHANGELOG.md` - Existing changelog; new section appended at top
- `README.md` - 4-line readme; optional link target for MIGRATION.md

No external ADRs or design docs exist for this project. Requirements are fully captured in the decisions above.

</canonical_refs>

<code_context>

## Existing Code Insights

### Reusable Assets

- `CHANGELOG.md` - Existing changelog with a loose bullet style; new section follows the same pattern (no Keep-a-Changelog strictness enforced in the file).
- `README.md` - Minimal; safe to add a one-line migration pointer if desired.

### Established Patterns

- **Changelog style:** Top-of-file newest-first, `## <version>` headings, free-form bullets. No "Added/Changed/Removed" sub-grouping in recent entries — match the loose style.
- **No existing MIGRATION.md:** This is a new file. The package/group-id change note in `README.md` ("from 1.3.12-1") is the only prior migration-style note in the repo — it's inline in README, but a dedicated file is warranted here because the exception-type change is a compile break.

### Integration Points

- The changelog entry must be consistent with the verified Phase 3 behavior (49 tests green, exception type = `FailsafeException`/raw throwable). Do not describe behavior that wasn't proven.
- `Handler.handleDelivery()` catches `Throwable` — the docs should reassure consumers that internal call-site behavior is unchanged; only *their own* typed `catch (RetryException)` breaks.

</code_context>

<specifics>
## Specific Ideas

- The exception-type change is the single most important thing to document: it's a transitive compile break for consumers catching `RetryException` by type, and there is no compatibility shim (explicitly out of scope per PROJECT.md).
- The docs must reference the verified failsafe version (3.3.2) and the removed guava-retrying coordinate so the swap is reproducible.
- This is the final phase of a 4-phase migration; the docs close out the milestone. Keep it factual — the parity is proven, not claimed.

</specifics>

<deferred>
## Deferred Ideas

None - discussion stayed within phase scope. Observability hooks (onFailure listeners), jitter, and async execution remain v2 requirements tracked in REQUIREMENTS.md, not this phase.

</deferred>

---

*Phase: 04-release-documentation*
*Context gathered: 2026-09-10*
