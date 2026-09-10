# Project State

**Last updated:** 2025-09-19
**Phase:** Roadmap created — ready for Phase 1 planning

## Current Phase

**Phase 1: Add failsafe dependency** — not started.

## Phase Status

| Phase | Name | Status |
| ----- | ---- | ------ |
| Phase 1 | Add failsafe dependency | Not started |
| Phase 2 | Rewrite retry engine (base + all seven impls, atomic) | Blocked by Phase 1 |
| Phase 3 | Remove guava-retrying and verify parity | Blocked by Phase 2 |
| Phase 4 | Release documentation | Blocked by Phase 3 |

## Active Decisions

- **failsafe.dev as replacement** — actively maintained, fluent RetryPolicy API, drop-in for guava-retrying's Retryer pattern. (Pending verification in Phase 2.)
- **Preserve RetryStrategy base class shape** — consumers depend on `execute(Callable<Boolean>)` contract. (Enforced in Phase 2.)
- **Map RetryerBuilder → RetryPolicy per strategy** — 1:1 strategy migration minimizes behavioral drift. (Implemented in Phase 2.)

## Open Risks

- **Silent behavioral drift** (HIGH): exponential backoff curve halved if mapped 1:1; default exception handling inverted; `withMaxRetries` vs `withMaxAttempts` off-by-one. Mitigated by Phase 2 parity construction + Phase 3 parity tests + grep gate.
- **Incremental-wait attempt-count basis** (MEDIUM): `ExecutionContext.getAttemptCount()` semantics (0 vs 1 based) must be confirmed against existing tests during Phase 2 planning.
- **Downstream `RetryException` consumers** (MEDIUM): unknown count of external consumers catching `RetryException` by type. Handled via Phase 4 changelog documentation, not a compatibility shim (out of scope).

## Blockers

None. Roadmap is created; Phase 1 is ready to plan.

## Next Action

Plan Phase 1: add `dev.failsafe:failsafe:3.3.2` to `pom.xml`. This is a single-dependency add with no source changes — standard Maven territory, no research phase needed.

## Notes

- Repo has `commit.gpgsign=true` but no GPG key in this environment. Do not attempt git commits; the orchestrator commits.
- `getMultipier()` typo in config must be preserved (config-compatibility constraint) — do not "fix" it during migration.
