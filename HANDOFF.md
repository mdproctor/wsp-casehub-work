# Handoff — 2026-07-28

## What's Done

**#319 — Retired reactive tier in casehub-work** (closed). 5 production files in `engine-adapter/`: `WorkItemLifecycleAdapter`, `PlanItemCompletionApplier` switched to blocking `CrossTenantCaseInstanceRepository`; 3 handlers switched from `blocking=true` to `@RunOnVirtualThread`.

**#320 — Platform/engine SNAPSHOT sync** (closed). Multi-round CI fix session syncing with platform #384 (reactive retirement) and engine #381:
- `DeclineTarget.toSerializedValue()`, `SettingsScope` tenancyId parameter (3 production + 2 test sites)
- `GroupMembershipProvider.membersOf()` tenancyId parameter (production + 3 test files)
- Engine-adapter tests: removed reactive alternatives, switched to blocking repository APIs, removed `.await().atMost()` and `.toCompletableFuture().join()` patterns
- `QueueDashboardTest` flaky fix: Awaitility polling replacing fixed 200ms sleeps

CI green on `32ab607b`.

Garden entry `GE-20260724-04bc63` — IntelliJ MCP cross-project index bleed.

## What's Left

- #317 — doc sync: update ARC42STORIES.MD and api-reference.md for LabelRule types · M · Low
- #316 — label-triggered CDI observer pattern for non-label side effects · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #317 | Docs: update ARC42 + api-reference for LabelRule | M | Low | |
| #309 | Progress model: visualisation modes | M | Med | |
| #308 | Progress model: rollback control mechanism | S | Med | |
| #307 | Progress model: arbitrary JSON schema shape type | S | Low | |
