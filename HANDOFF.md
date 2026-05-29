# Handover — 2026-05-29

## Last Session

Closed `issue-233-xs-s-batch-cleanup` — five XS/S issues: `SIGNAL_RECEIVED` enum (#221),
`ClaimFirstStrategy @Alternative @Priority(0)` CDI fix (#231), migration customizer
cosmetics (#232), capability matching contract javadoc (#220 partial), `CREATE_DENIED`
audit with pre-generated WorkItem ID + V32 FK drop (#192). Branch rebased onto main,
pushed to origin and upstream. All closed.

## Immediate Next Step

**`issue-223-provenance-supplement`** — branch exists locally, no PR filed.
Flagged in three consecutive handovers. Either raise the PR or close the branch.

## Cross-Module

*Unchanged — `git show HEAD~2:HANDOFF.md`*

## What's Left

- `issue-223-provenance-supplement` — open branch, no PR, deferred again · S · Low
- `#220` — short-term (shared constants) and long-term (CapabilityRegistry SPI) still open; only javadoc immediate step done · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| parent#66 | Apply CLAUDE.md size discipline to remaining casehubio repos | L | Low | casehub-work is reference impl |

## Key References

- Garden: GE-20260529-010101 (REQUIRES_NEW try-catch commit-time gotcha — new this session)
- Garden REVISE: GE-20260415-884e48 (@Priority alternative solution added — new this session)
- Blog: `2026-05-29-mdp06-audit-entry-that-couldnt-exist.md`
