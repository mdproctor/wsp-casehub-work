# Design: Replace FilterScope with Path-based scope hierarchy

**Issue:** casehubio/work#263  
**Branch:** issue-263-filterscope-to-path  
**Date:** 2026-06-16

## Problem

`FilterScope` (PERSONAL/TEAM/ORG) is a hardcoded parallel scope type in the queues module,
violating the platform boundary rule: "Do not define parallel path, scope, preference, or
principal types." `casehub-platform-api` owns `Path` as the canonical scope type.
`#236` eliminated `VocabularyScope` from the runtime module using the same fix.

## Decision

Delete `FilterScope`. Replace every usage with `Path` from `casehub-platform-api`.
This is identical in pattern to #236 (`VocabularyScope → Path`), applied to the queues module.

## Data Model

### `WorkItemFilter` and `QueueView` (JPA entities)

- `scope: FilterScope` → `scope: Path` with `@Convert(converter = PathAttributeConverter.class)`
- `ownerId` field — **dropped**. Path encodes ownership; a personal filter for actor `alice`
  lives at `Path.of("alice")`, an org-wide filter at `Path.root()`. `ownerId` is not queried
  in any store SPI and is redundant once path carries the hierarchy.
- `PathAttributeConverter` is in `runtime/`, which queues already depends on — no module
  changes needed.

### `WorkItemFilterBean` (SPI interface)

- `FilterScope scope()` → `Path scope()`

### `FilterScope` enum

Deleted entirely.

## REST API

`CreateFilterRequest` and `CreateQueueRequest` replace `(FilterScope scope, String ownerId)`
with `String scope` (nullable path string).

- Null or absent → `Path.root()` (widest scope; equivalent to the old ORG default)
- Non-null → `Path.parse(req.scope())`

`list()` responses serialize `scope` via `path.value()` — same JSON field, now an open-ended
string instead of an enum constant.

## Schema

No Flyway migration: there are no installs. Modify `V2000__queues_schema.sql` directly:

- Both `work_item_filter` and `queue_view`: widen `scope VARCHAR(20)` → `scope VARCHAR(500)`
- Remove `owner_id VARCHAR(255)` column from both table definitions

## Call Sites

All touched in this PR — no callers outside the queues module tree.

### `WorkItemFilterBean` implementations

| File | Change |
|------|--------|
| `queues-dashboard/SecurityWritersFilter.java` | `return FilterScope.ORG;` → `return Path.root();` |
| `queues-examples/review/SecurityWritersFilter.java` | same |

### Examples and tests

All `FilterScope.ORG` → `Path.root()`.  
`FilterScope.TEAM` (QueueModuleScenario) → `Path.of("team")` (illustrative; callers choose their own paths).

## Platform Coherence

- Removes a parallel scope type — brings queues into alignment with the boundary rule
- No new types introduced; no new module deps
- `PathAttributeConverter` reused as-is from `runtime/`
- No consumers outside this repo depend on `FilterScope` (queues module is not published as a
  standalone artifact)

## Testing

- Existing tenancy tests (`JpaWorkItemFilterStoreTenancyTest`, `JpaQueueViewStoreTenancyTest`,
  `JpaFilterChainStoreTenancyTest`) updated: `f.scope = FilterScope.ORG` → `f.scope = Path.root()`
- `PathAttributeConverter` roundtrip already covered by `PathAttributeConverterTest` in runtime
- No new test types needed; the scope field change is mechanical
