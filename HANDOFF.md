# HANDOFF — 2026-07-13

## Last Session

Closed #287 — retrofitted 4 work SPIs (WorkerSelectionStrategy, InstanceAssignmentStrategy, ClaimSlaPolicy, SlaBreachPolicy) to extend platform `NamedStrategy` and resolve via `StrategyResolver` instead of 3 ad-hoc CDI mechanisms (Instance scanning + config switch, @Named iteration, @Alternative/@Priority). 13 implementations add `id()`, 5 resolution sites simplified. Design-reviewed adversarially (5 rounds, 15 issues, $17.04). 892+ unit tests, 38 integration tests green. Landed as `dcc0699` on main.

Also landed `4f7825f` (engine-adapter tenancyId in ActionGate completion events) — pre-existing commit from prior session.

Garden entry GE-20260713-cecee5: `@Singleton` StrategyResolver causes StackOverflowError when NamedStrategy bean injects StrategyResolver — fix: `Provider<StrategyResolver>`.

## Immediate Next Step

Engine-adapter has unstaged changes in the working tree (casehub-engine API update: `PlanItemStatus → TaskStatus`, `String → ExecutorRef`, `PlanItemSaveRequest` constructor change). These need a separate issue and branch to resolve.

## What's Left

- Engine-adapter unstaged changes — casehub-engine API update (PlanItemStatus → TaskStatus, ExecutorRef) · S · Med
- casehubio/garden#4 — create `definable-entity-types-labels.md` protocol file · S · Low
- casehubio/parent#354 — PLATFORM.md protocol cross-reference · XS · Low
- casehubio/engine#653 — engine work-adapter update for types (source-breaking) · S · Med
- 6 consumer apps need POM update: `casehub-engine-work-adapter` → `casehub-work-engine-adapter` · XS each · Low
- casehubio/work#300 — update #287 issue body to include SlaBreachPolicy scope · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #298 | Event contract refactoring — replace @ConsumeEvent with direct calls | M | Med | Deferred from #290 |
| #299 | CloudEvent bridge for cross-service HumanTask creation | M | Med | Deferred from #290 |
| #288 | Queue summary REST endpoint for dashboard cards | M | Med | Enhancement |
| #152 | Split examples into core and full variants | M | Low | |

## References

- Spec: `docs/specs/2026-07-12-retrofit-work-spis-namedstrategy-design.md` (project)
- Plan: `plans/2026-07-13-retrofit-work-spis-namedstrategy.md` (workspace)
- Design review: `~/adr/casehub-work/retrofit-work-spis-namedstrategy-20260713-000423/`
- Garden: GE-20260713-cecee5 (CDI circular dependency with @Singleton StrategyResolver)
