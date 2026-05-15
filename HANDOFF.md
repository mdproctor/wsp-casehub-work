# casehub-work — Session Handover
**Date:** 2026-05-15

## What Was Done This Session

### casehubio/work#166 — callerRef propagation for multi-instance groups ✅ merged
- `MultiInstanceSpawnService.createGroup()` gains 4th `String callerRef` param; set on parent WorkItem only
- `WorkItemTemplateService.instantiate()` removes warn-and-ignore block; passes callerRef through
- Children remain null — `WorkItemGroupLifecycleEvent` already reads `parent.callerRef` (no change needed there)
- 3 new tests: `createGroup_withCallerRef_setsOnParentOnly`, `completedGroupEvent_carriesCallerRef` (with `during()` stability window), `instantiate_multiInstanceTemplate_setsCallerRefOnParent`
- 641 runtime tests, 0 failures. PR #173 merged, #166 closed.

### Stale ledger audit prompt (non-event)
- Prompt described 3 bugs in `casehub-work-ledger` — all fixed April 29. Verified and moved on.

## Immediate Next Step

**engine#255** — template mode in `HumanTaskScheduleHandler` + `WorkItemGroupLifecycleAdapter` (engine-side partner to #166). User said engine team will handle this.

Next casehub-work item: **#169–171** — OHT gaps (named outcomes, output schema, excluded owners). Needs design session first.

## Open / Next

| Priority | What |
|---|---|
| 1 | casehubio/work#169–171 — OHT gaps (named outcomes, output schema, excluded owners) |
| 2 | casehub-clinical — Epic 4 adverse event escalation |
| blocked | casehubio/work#97 — event mesh (blocked on qhorus#131, #132) |
| engine | casehubio/engine#255 — template mode + WorkItemGroupLifecycleAdapter (engine team) |

## Key References

- Blog: `blog/2026-05-15-mdp02-parent-gets-the-callerref.md`
- Spec: `specs/2026-05-15-callerref-multi-instance-design.md`
- Garden: `GE-20260515-ed10ee` — Awaitility `untilAsserted` vs `during` for exact event counts
- Stale squash plans (can delete): `docs/squash-plan-2026-05-07.md`, `docs/squash-plan-v3-2026-05-08.md`
- Clinical context: `~/claude/casehub/clinical/wksp/HANDOFF.md`
