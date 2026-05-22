# casehub-work — Session Handover
**Date:** 2026-05-22 (fifth session)

## What Was Done This Session

Closed out the EscalationPolicy era: removed the interface, three impls, producer, qualifiers, config keys, and all test files (work#215). Added `AssignmentTrigger.SLA_ESCALATED` and wired `WorkItemAssignmentService.assign()` into `executeEscalateTo()` before `put()` so strategies pre-assign after breach escalation (work#216). Extended `writeAudit()` with a nullable detail field; `executeFail()` now records `fail.reason()` in the audit entry — preserved from `AutoRejectEscalationPolicy`. Synced parent docs for SlaBreachPolicy, scope field, scanRoots, and casehub-platform-api dep (parent#43). Closed work#205 (scanRoots 3-param was already in 66c4541). 738 runtime tests, 0 failures.

## Current State

- Both repos on `main`, origin current. `origin/issue-212-sla-breach-policy` remote branch is stale (merged) — can be deleted.
- `casehub-platform` local clone has `Path.root()` committed (`d8d8461`) but not pushed to `casehubio/platform` — blocks upstreaming to `casehubio/work`.
- `origin/issue-204-xs-s-backlog-cleanup` has 5 unmerged commits, 19h old — active in-progress work.

## Immediate Next Step

Push `Path.root()` from platform commit `d8d8461` to `casehubio/platform` and publish a SNAPSHOT. Then open PR from `origin/main` → `casehubio/work`.

## Cross-Module

**Blocking:**
- `casehub-engine` — needs `WorkItem.scope` field (V31) for `HumanTaskTarget.scope` propagation · S · Low · engine#330

**Blocked by:**
- `casehub-platform` — `Path.root()` must be published before `casehubio/work` PR can ship · XS · Low

## What's Left

- engine#330 — `HumanTaskTarget.scope` field propagation to `WorkItem.scope` · S · Low
- work#216 follow-up — no test for `SLA_ESCALATED` in `checkClaimDeadlines` path (shared code, low risk) · XS · Low
- Delete stale remote `origin/issue-212-sla-breach-policy` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Open PR origin → casehubio/work | XS | Low | After platform publishes `Path.root()` |
| — | casehub-clinical Epic 4 — adverse event escalation | L | Med | First consumer of SlaBreachPolicy |
| — | Resume `issue-204-xs-s-backlog-cleanup` | S | Low | 5 commits ready, 19h old |

## Key References

- Blog: `_notes/2026-05-22-mdp08-what-a-void-spi-costs-you.md` (mdproctor.github.io)
- Garden: GE-20260522-ac6b1d (ConfigMapping sweep — grep integration-tests/ too)
- Platform commit: `d8d8461` (Path.root() — local only, not pushed to casehubio/platform)
