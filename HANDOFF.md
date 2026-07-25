# Handoff — 2026-07-25

## What's Done

**#319 — Retired reactive tier in casehub-work** (this session). Scope turned out to be 4 files in `engine-adapter/` — work was already blocking-first (`quarkus-hibernate-orm-panache`, zero reactive source files). Changed 3 handlers from `blocking=true` to `@RunOnVirtualThread`, switched `WorkItemLifecycleAdapter` from `ReactiveCrossTenantCaseInstanceRepository` to blocking `CrossTenantCaseInstanceRepository`.

Garden entry `GE-20260724-04bc63` — IntelliJ MCP `ide_search_text` returns cross-project results when multiple projects share an IDE instance.

## Known Issues

- `api/DeclineTarget.java` — pre-existing build error: platform SNAPSHOT added `toSerializedValue()` to `Preference` interface. Not related to #319.
- `engine-adapter` tests — pre-existing 21 CDI deployment failures (unsatisfied `WorkItemCreator` dependency).

## What's Left

- #317 — doc sync: update ARC42STORIES.MD and api-reference.md for LabelRule types · M · Low
- #316 — label-triggered CDI observer pattern for non-label side effects · S · Low
- Platform#185 (proactive membership cleanup on view deletion) — adopt when it lands · S · Low
- Fix `DeclineTarget.toSerializedValue()` build error · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Fix DeclineTarget build error (platform SNAPSHOT drift) | XS | Low | Blocking full build |
| #317 | Docs: update ARC42 + api-reference for LabelRule | M | Low | |
| #309 | Progress model: visualisation modes | M | Med | |
| #308 | Progress model: rollback control mechanism | S | Med | |
| #307 | Progress model: arbitrary JSON schema shape type | S | Low | |
