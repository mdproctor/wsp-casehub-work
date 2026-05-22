# casehub-work — Session Handover
**Date:** 2026-05-22 (fourth session)

## What Was Done This Session

Shipped `SlaBreachPolicy` SPI replacing `EscalationPolicy` (#212/#213): sealed `BreachDecision` types (Fail/EscalateTo/Extend/Chained), `ExpiryLifecycleService` rewritten policy-driven, `SlaBreachEvent` CDI event carrying the leaf decision, V31 scope field, `scanRoots` 3-param split (#205), queues cleanup (#208–#210). Three DevTown review rounds. Two critical fixes post-review: `EscalateTo.to()` validates non-empty groups at factory; executor converts bare empty `EscalateTo` to logged Fail (transaction boundary guard). `BreachedTask.taskId` changed from `String` to `UUID`. Fixed persistent blog misrouting: `write-blog` skill now reads `~/.claude/blog-routing.yaml` first — entries go directly to `mdproctor.github.io/_notes/`, no workspace staging. Branch closed, both repos on main.

## Current State

- Both repos on `main`, origin current. Upstream (casehubio/work) is 7 commits ahead of its last PR merge.
- `casehub-platform` local clone has `Path.root()` committed (`d8d8461`) but not pushed to `casehubio/platform` — blocks #212 from shipping to upstream until platform publishes.
- 746 runtime tests, 84 queues tests, 61 api tests — all green.

## Immediate Next Step

Coordinate with platform team to push and publish `Path.root()` from platform commit `d8d8461`. Once published, open PR from `origin/main` → `casehubio/work`.

## Cross-Module

**Blocking:**
- `casehub-engine` — needs `WorkItem.scope` field (V31) for `HumanTaskTarget.scope` propagation · S · Low · engine#330

**Blocked by:**
- `casehub-platform` — `Path.root()` must be published before #212 can ship to casehubio/work · XS · Low

## What's Left

- engine#330 — `HumanTaskTarget.scope` field propagation to `WorkItem.scope` · S · Low
- work#215 — remove deprecated `EscalationPolicy` impls + producer (after consuming apps migrate) · S · Low
- work#216 — WorkBroker auto-assignment after `EscalateTo` decision · S · Med
- parent#43 — sync `docs/repos/casehub-work.md` for SlaBreachPolicy, scope field, 3-param scanRoots · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Open PR origin → casehubio/work | XS | Low | After platform publishes Path.root() |
| — | casehub-clinical Epic 4 — adverse event escalation | L | Med | First consumer of SlaBreachPolicy |
| #205 | candidateUser + assignee inbox divergence | S | Med | Filed, unresolved |

## Key References

- Blog: `_notes/2026-05-22-mdp03-the-decision-the-policy-returns.md` (mdproctor.github.io)
- Garden: GE-20260522-44bbf3 (@Transactional batch rollback), GE-20260511-3e5a75 (REVISE — deprecated + SlaBreachPolicy migration)
- Spec: `docs/specs/issue-212-sla-breach-policy/2026-05-22-sla-breach-policy-design.md`
- Platform commit: `d8d8461` (Path.root() — local only, not pushed)
