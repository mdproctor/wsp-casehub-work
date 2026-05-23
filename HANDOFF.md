# casehub-work — Session Handover
**Date:** 2026-05-23 (sixth session)

## What Was Done This Session

Pushed all work from the previous session to both origin and upstream. Then chased
casehubio/work CI from red to green across 11 builds — fixing 7 accumulated failures
(API drift, missing casehub-platform deps, wrong scope, 5-field cron, ledger test
arg order). CI is now green: all 21 modules pass (Build and Publish run 26314534946).

## Current State

- Both repos on `main`, fully synced to casehubio upstream.
- Issue #218 closed. CI green.
- `casehub-platform` dep now correctly scoped: `test` for library modules, `runtime`
  for application modules that run `quarkus:build`.
- 6 old epic branches exist locally without EPIC-CLOSED.md — code is in upstream,
  branches are undeleted (14-day retention policy applies).

## Immediate Next Step

engine#330 — add `WorkItem.scope` field (V31 migration) for `HumanTaskTarget.scope`
propagation in casehub-engine. Small, unblocked.

## Cross-Module

**We're blocking:**
- `casehub-engine` — needs `WorkItem.scope` (V31) for `HumanTaskTarget.scope` · S · Low · engine#330

**Blocked by:** nothing currently.

## What's Left

- engine#330 — `HumanTaskTarget.scope` propagation to `WorkItem.scope` · S · Low
- Delete stale remote `origin/issue-212-sla-breach-policy` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | engine#330 — WorkItem.scope V31 | S | Low | First unblocked item |
| — | casehub-clinical Epic 4 — adverse event escalation | L | Med | First consumer of SlaBreachPolicy |

## Key References

- Garden: GE-20260523-60365e (quarkus:build test-scope CDI), GE-20260523-51620a (SNAPSHOT API drift)
