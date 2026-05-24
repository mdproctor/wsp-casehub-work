# casehub-work — Session Handover
**Date:** 2026-05-24 (seventh session)

## What Was Done This Session

Chased CI from red to green: ledger#88 moved `ActorType`/`ActorTypeResolver` to
`io.casehub.platform.api.identity` (8 imports in ledger, 2 in examples fixed);
ledger#95 moved base migrations to `db/ledger/migration` (Flyway locations fixed
in ledger + examples test configs). CI green, all 76 ledger tests pass locally.
Closed the issue-212 workspace branch (journal + spec merged to workspace main,
stale remote deleted). Updated casehub-work.md and PLATFORM.md in parent with
new modules, platform-api deps, scope rule, and Flyway range table.

## Current State

- Both repos on `main`, CI green.
- Issues #219 (CI fixes) and #222 (epic-212 workspace close) closed.
- `issue-215-escalation-removal-and-fixes` workspace remote branch missing
  EPIC-CLOSED.md — minor paperwork, not blocking.
- Epic branches scheduled for deletion 2026-06-05 (14-day retention).

## Immediate Next Step

engine#330 — add `WorkItem.scope` to `HumanTaskTarget` and propagate through
`HumanTaskScheduleHandler`. **This is engine-repo work — open a Claude session
in `~/claude/casehub/engine`, not here.**

## Cross-Module

**We're blocking:**
- `casehub-engine` — needs `WorkItem.scope` (V31) for `HumanTaskTarget.scope` · S · Low · engine#330

**Blocked by:** nothing.

## What's Left

- Add EPIC-CLOSED.md to workspace `issue-215-escalation-removal-and-fixes` branch · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | engine#330 — WorkItem.scope V31 | S | Low | Engine session, not here |
| — | casehub-clinical Epic 4 — adverse event escalation | L | Med | First SlaBreachPolicy consumer |

## Key References

- Garden: GE-20260524-1f0045 (git push from wrong branch silently pushes local branch)
- Protocols: PP-20260524-a8f597 (casehub-platform scope), PP-20260524-10efef (Flyway ledger locations)
- Blog: `2026-05-24-mdp01-snapshot-drift-ten-imports.md`
