# Handoff — 2026-07-26

## What's Done

**#319 — Retired reactive tier in casehub-work.** Scope was 5 files in `engine-adapter/` — work was already blocking-first. Switched `WorkItemLifecycleAdapter` and `PlanItemCompletionApplier` from `ReactiveCrossTenantCaseInstanceRepository` to blocking `CrossTenantCaseInstanceRepository`. Converted 3 handlers from `blocking=true` to `@RunOnVirtualThread`.

**#320 — Platform SNAPSHOT sync.** `DeclineTarget.toSerializedValue()` (new `Preference` method), `SettingsScope` constructor gained `tenancyId` (3 call sites + 2 test fixtures). `QueueDashboardTest` flaky test fixed with Awaitility polling replacing fixed 200ms sleeps.

Garden entry `GE-20260724-04bc63` — IntelliJ MCP cross-project index bleed gotcha.

## Known Issues

- engine-adapter tests: pre-existing 21 CDI deployment failures (unsatisfied `WorkItemCreator`) + test compilation errors referencing engine `Reactive*` types (tracked by parent#381)
- `QueueDashboardTest` flaky fix pushed — CI result pending

## What's Left

- #317 — doc sync: update ARC42STORIES.MD and api-reference.md for LabelRule types · M · Low
- #316 — label-triggered CDI observer pattern for non-label side effects · S · Low
- Platform#185 (proactive membership cleanup on view deletion) — adopt when it lands · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #317 | Docs: update ARC42 + api-reference for LabelRule | M | Low | |
| #309 | Progress model: visualisation modes | M | Med | |
| #308 | Progress model: rollback control mechanism | S | Med | |
| #307 | Progress model: arbitrary JSON schema shape type | S | Low | |
