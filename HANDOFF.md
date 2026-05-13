# casehub-work — Session Handover
**Date:** 2026-05-13

## What Was Done This Session

### casehub-work#165 — WorkItemTemplateService callerRef overload (complete ✅)

`WorkItemTemplateService.instantiate()` and `toCreateRequest()` both hardcoded `callerRef = null`. Added 5-arg overloads for both; existing 4-arg forms delegate with `null` — all callers unchanged. 4 new tests (2 unit, 2 `@QuarkusTest`). Installed to local Maven repo.

Commits: `73376c3`, `56312b1`, `16b0836` (CLAUDE.md gotcha)

### casehub-engine#245 — HumanTaskTarget binding design (spec complete, no code yet)

Full design spec written and committed to engine workspace. The inbound adapter (`WorkItemLifecycleAdapter`, 8 tests) was already done. The design covers:
- `BindingTarget` sealed interface + `ExtensionTarget` escape hatch in `casehub-engine-api`
- `HumanTaskTarget` (template + inline modes, inputMapping/outputMapping via `ExpressionEvaluator`)
- `PlanItem.target()` — stores `BindingTarget` so adapter can look up outputMapping at completion without a separate registry
- `HumanTaskScheduleHandler` in `casehub-engine-work-adapter` (outbound)
- `WorkItemLifecycleAdapter` extended for outputMapping (inbound)

Spec: `engine/wksp/specs/2026-05-12-human-task-binding-design.md`

### Issues closed/created

- casehubio/work#136 — closed (inbound adapter was already done)
- casehubio/work#165 — closed (this session)
- casehubio/work#166 — open: multi-instance template + callerRef (complex, needs design)
- casehubio/work#167 — open: 4 Minor code quality findings from review
- casehubio/engine#245 — open: implement the HumanTaskTarget binding spec

## Open / Next

| Priority | What |
|---|---|
| 1 | casehubio/engine#245 — implement HumanTaskTarget binding (engine claude, waiting) |
| 2 | casehubio/work#167 — fix Javadoc asymmetry + @Transactional annotation on 5-arg |
| 3 | casehubio/work#166 — multi-instance + callerRef (design session needed) |
| 4 | casehubio/clinical — Epic 4 adverse event escalation (plan ready, see clinical handover) |

## Key References

- Engine design spec: `engine/wksp/specs/2026-05-12-human-task-binding-design.md`
- callerRef overload: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`
- Stale squash plans (can delete): `docs/squash-plan-2026-05-07.md`, `docs/squash-plan-v3-2026-05-08.md`
- Clinical context: `~/claude/casehub/clinical/wksp/HANDOFF.md`
