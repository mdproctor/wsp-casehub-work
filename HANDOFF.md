# HANDOFF — 2026-07-13

## Last Session

Closed #287 — retrofitted 4 work SPIs to extend NamedStrategy + StrategyResolver. Design-reviewed (5 rounds, $17.04). 892+ unit tests, 38 ITs green. Landed as `dcc0699`.

Closed #303 — synced engine-adapter compilation against current engine SNAPSHOTs (PlanItemSaveRequest fields, TaskStatus split, ExecutorRef, HumanTaskScheduleEvent param reorder). Landed as `cadedfd`.

Engine: added catch-all `@Any Instance<NamedStrategy>` to `EngineStrategyResolver` (`a3f9a7fc` on engine main). Needed for cross-module NamedStrategy discovery.

Filed #304 — engine-adapter Quarkus test failures. ARC prunes ClaimSlaPolicy beans after #287 removed direct injection points. Investigated 5 approaches (catch-all, @Unremovable, UnremovableBeanBuildItem, AdditionalBeanBuildItem, index-dependency). Root cause: ARC bean discovery doesn't find core/ beans in engine-adapter test context. Try `quarkus.arc.remove-unused-beans=false` in test properties.

Garden: GE-20260713-cecee5 — @Singleton StrategyResolver StackOverflowError with Provider<> fix.

## Immediate Next Step

Fix #304 — engine-adapter test failures. Start with `quarkus.arc.remove-unused-beans=false` in engine-adapter test properties to confirm the diagnosis. If that works, find the minimal unremovable configuration.

## What's Left

- casehubio/work#304 — engine-adapter Quarkus test failures (ARC bean pruning) · S · Med
- casehubio/work#300 — update #287 issue body to include SlaBreachPolicy scope · XS · Low
- casehubio/garden#4 — create `definable-entity-types-labels.md` protocol file · S · Low
- casehubio/parent#354 — PLATFORM.md protocol cross-reference · XS · Low
- casehubio/engine#653 — engine work-adapter update for types (source-breaking) · S · Med
- 6 consumer apps need POM update: `casehub-engine-work-adapter` → `casehub-work-engine-adapter` · XS each · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #298 | Event contract refactoring — replace @ConsumeEvent with direct calls | M | Med | Deferred from #290 |
| #299 | CloudEvent bridge for cross-service HumanTask creation | M | Med | Deferred from #290 |
| #288 | Queue summary REST endpoint for dashboard cards | M | Med | Enhancement |
| #152 | Split examples into core and full variants | M | Low | |

## References

- Spec: `docs/specs/2026-07-12-retrofit-work-spis-namedstrategy-design.md` (project)
- Design review: `~/adr/casehub-work/retrofit-work-spis-namedstrategy-20260713-000423/`
- Garden: GE-20260713-cecee5 (CDI circular dependency with @Singleton StrategyResolver)
- Engine commit: `a3f9a7fc` (EngineStrategyResolver catch-all)
