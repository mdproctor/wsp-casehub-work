# casehub-work — Session Handover
**Date:** 2026-05-22

## What Was Done This Session

Cleared the backlog (#200 inbox filters, #202 cursor TTL, #203 round-robin config), ran work-end on epic-exclusion-audit and epic-output-schema, then implemented #207 (JqConditionEvaluator → JQEvaluator from casehub-platform-expression). Git workflow corrected: rebase upstream before start/end, push to origin/main, PRs to upstream separately on demand.

Key findings: five inbox filter params were declared but silently ignored; `candidateUsers` causes immediate auto-assignment (status PENDING → ASSIGNED); `@QuarkusComponentTest` silently auto-stubs external beans unless listed in `value[]` (undocumented — found via bytecode inspection, GE-20260522-2664b9 filed).

## Current State

- Both repos on `main`, origin/main pushed, 5 commits ahead of upstream
- Runtime: 722 tests. Queues: 83 tests. All green.
- V30 Flyway migration live (routing_cursor.last_accessed)
- `RoutingCursorCleanupJob` scheduled daily at 02:00 (configurable)

## Immediate Next Step

Open PR from origin/main → upstream (casehubio/work) when ready — covers #200, #202, #203, #207 and the doc/workflow fixes.

## What's Left

- `epic-excluded-users` — open branch, recent commit (31h ago), no EPIC-CLOSED.md · XS · Low
- parent#41 — add casehub-platform-expression to BOM (filed, waiting on parent session) · XS · Low
- #208 — replace deprecated `quarkus-junit5` in queues pom · XS · Low
- #209 — inbox outcome filter positive-path test · S · Low
- #210 — inject CDI ObjectMapper into JqConditionEvaluator · XS · Low
- #205 — candidateUser dropped when assignee also provided in inbox · S · Med
- engine#314, engine#320 — same JQ consolidation in engine repo · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Open PR origin → casehubio/work | XS | Low | When ready |
| — | work-end epic-excluded-users | XS | Low | Check if merged |
| — | casehub-clinical Epic 4 — adverse event escalation | L | Med | — |
| #205 | candidateUser + assignee inbox gap | S | Med | — |

## Key References

- Blog: `blog/2026-05-22-mdp01-params-that-did-nothing.md`
- Garden: GE-20260522-2664b9 (@QuarkusComponentTest external bean wiring)
- parent#41: BOM update for casehub-platform-expression
