# Handover — 2026-05-26

## Last Session

*(Previous session 2026-05-25: cleared S/XS trailing obligations, stamped 4 branches closed.)*

**Parent session (2026-05-26) made commits to this repo directly:**
- `fix(#224)` on main: SlaBreachEvent Javadoc corrected — `@Observes` not `@ObservesAsync` (event is synchronous `fire()`)
- `docs(#198)` on main: WorkItemLabelMapperTest — one-line comment noting MANUAL-only scope, INFERRED tested in WorkItemServiceTest
- `fix(#223)` on branch `issue-223-provenance-supplement`: LedgerEventCapture now attaches ProvenanceSupplement on the creation entry for case-orchestrated WorkItems. `callerRef` stored unchanged as `sourceEntityId` per PP-20260526-6d39e5. Placement before Merkle hash — critical. 27 tests green.

Closed this session: work#224, work#198, work#223, work#164 (folder names already correct).

## Immediate Next Step

**PR the open branch `issue-223-provenance-supplement`** — created by parent session, 27 tests passing.
Branch is on `issue-223-provenance-supplement` in the project repo.

## Cross-Module

**New issue filed by parent session:**
- `casehubio/work#229` — rename `db/migration/` → `db/work/migration/` across all submodules (runtime, ai, notifications, queues, ledger, issue-tracker). Protocol PP-20260525-607b33. Coordinated breaking change — consumer repos (aml, clinical, devtown) must update simultaneously.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| work#229 | Rename db/migration/ → db/work/migration/ (Flyway scoping) | M | Low | Coordinated with aml, clinical, devtown |
| engine#330 | WorkItem.scope V31 — HumanTaskTarget propagation | S | Low | Engine session, not here |
| parent#66 | Apply CLAUDE.md size discipline to remaining casehubio repos | L | Low | Checklist in issue; casehub-work is the reference impl |

## Key References

- Open branch: `issue-223-provenance-supplement` (project repo — needs PR)
- Protocols: PP-20260526-6d39e5 (opaque cross-module identifiers — new), PP-20260525-607b33 (Flyway repo-scoped paths — new), PP-20260525-8c361f, PP-20260525-5b1efa
- Garden: GE-20260525-58fcbf, GE-20260525-3fe619 (unchanged)
