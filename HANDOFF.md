# HANDOFF — 2026-07-13

## Last Session

Closed #287 — retrofitted 4 work SPIs to extend NamedStrategy + StrategyResolver. Design-reviewed (5 rounds, $17.04). Landed as `dcc0699`.

Closed #303 — synced engine-adapter compilation against current engine SNAPSHOTs. Landed as `21f0436`.

Closed #304 — fixed engine-adapter Quarkus test failures. Root cause: ARC's `Instance<NamedStrategy>` doesn't resolve beans whose NamedStrategy relationship is transitive through sub-interfaces. Fix: `WorkStrategyContributor` in engine-adapter registers work SPI beans with `EngineStrategyResolver` at startup via per-SPI-type `Instance<T>` injection. 79 engine-adapter tests green.

Engine: `EngineStrategyResolver.registerEntry()` made public + catch-all `@Any Instance<NamedStrategy>` added (`814078ce`).

Garden: GE-20260713-cecee5 — @Singleton StrategyResolver StackOverflowError with Provider<> fix.

## Immediate Next Step

Pick from What's Next — all blockers from this session are resolved.

## What's Left

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

- Spec: `docs/specs/2026-07-12-retrofit-work-spis-namedstrategy-design.md`
- Design review: `~/adr/casehub-work/retrofit-work-spis-namedstrategy-20260713-000423/`
- Garden: GE-20260713-cecee5
- Engine: `814078ce` (EngineStrategyResolver catch-all + public registerEntry)
