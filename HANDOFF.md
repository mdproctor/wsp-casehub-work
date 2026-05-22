# casehub-work — Session Handover
**Date:** 2026-05-22 (third session)

## What Was Done This Session

Squashed 43 → 23 commits on main. Pre-push hook was silently broken (`git log --oneline` SHA prefix defeats `^`-anchored grep; fixed to `--format=%s`). Merge commits also defeat `git rebase -i drop` (silent exit 0 via BUG path; fixed with `--no-rebase-merges`). Closed `epic-excluded-users` — found and promoted an unpromoted spec. All work pushed to both origin and upstream (casehubio/work).

*Earlier this session: `git show HEAD~1:HANDOFF.md` — backlog items #200/#202/#203, #207, workflow fixes.*

## Current State

- Both repos on `main`, origin and upstream current
- 23 clean commits on main since last PR merge (8ffe54f); all epic branches closed
- Runtime: 722 tests, Queues: 83 tests — all green

## Immediate Next Step

Open PR from origin/main → upstream (casehubio/work) — covers #200, #202, #203, #207 and workflow fixes.

## What's Left

- parent#41 — casehub-platform-expression into BOM · XS · Low
- #208 — replace quarkus-junit5 with quarkus-junit in queues pom · XS · Low
- #209 — inbox outcome filter positive-path test · S · Low
- #210 — inject CDI ObjectMapper into JqConditionEvaluator · XS · Low
- #205 — candidateUser + assignee inbox divergence · S · Med
- engine#314, engine#320 — JQ consolidation in engine · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Open PR origin → casehubio/work | XS | Low | When ready |
| — | casehub-clinical Epic 4 — adverse event escalation | L | Med | — |
| #205 | candidateUser + assignee inbox divergence | S | Med | — |

## Key References

- Blog: `blog/2026-05-22-mdp02-hook-that-passed-everything.md`
- Garden: GE-20260522-7159b4 (git log --oneline hook), GE-20260522-5b1589 (rebase drop silent), GE-20260522-2664b9 (@QuarkusComponentTest)
- parent#41: BOM update for casehub-platform-expression
