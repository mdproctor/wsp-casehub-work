# Queue Summary REST Endpoint

**Issue:** casehubio/work#288
**Date:** 2026-07-14
**Status:** Approved

## Problem

The blocks-ui queue board component (casehubio/blocks-ui#7) computes queue summary
metrics client-side from the full item list returned by `GET /queues/{id}`. This
works for typical queue sizes but won't scale past ~500 items. A dedicated summary
endpoint returning pre-computed aggregates avoids sending the full list.

## Design

### 1. Rename InboxSummaryBuilder → WorkItemSummaryBuilder

`InboxSummaryBuilder` (in `runtime/service/`) is a pure static function on
`List<WorkItem>` with no inbox-specific logic. Its name reflects its first consumer,
not its actual scope. Rename:

- `InboxSummaryBuilder` → `WorkItemSummaryBuilder`
- `InboxSummary` → `WorkItemSummary`

Add `oldestCreatedAt` field — the `min(createdAt)` across non-terminal items.
Nullable (null when no non-terminal items exist). Follows the
`QueueHealthReport.oldestUnclaimedCreatedAt` convention of returning raw instants.

```java
public record WorkItemSummary(
        long total,
        Map<String, Long> byStatus,
        Map<String, Long> byPriority,
        long overdue,
        long claimDeadlineBreached,
        Instant oldestCreatedAt) {}
```

Use IDE rename to propagate to all callers:
- `WorkItemResource` (inbox summary endpoint in `rest/`)
- `InboxSummaryBuilderTest` → `WorkItemSummaryBuilderTest`
- `InboxSummaryTest` (REST integration test)

### 2. New endpoint: GET /queues/{id}/summary

Location: `QueueResource.java` in the `queues` module, alongside the existing
`query()`, `trend()`, and `streamQueueEvents()` methods.

Implementation:
1. Look up `QueueView` by id → 404 if not found
2. `QueueMembershipService.evaluateMembers(queue)` → `List<WorkItem>`
3. `WorkItemSummaryBuilder.build(items, Instant.now())` → `WorkItemSummary`
4. Return `WorkItemSummary` as 200 response body

Annotations: `@GET @Path("/{id}/summary") @Transactional`

Error cases: 404 (queue view not found) — same pattern as `query()` and `trend()`.

Response example:
```json
{
  "total": 42,
  "byStatus": {"PENDING": 20, "ASSIGNED": 15, "IN_PROGRESS": 7},
  "byPriority": {"LOW": 10, "MEDIUM": 20, "HIGH": 10, "URGENT": 2},
  "overdue": 3,
  "claimDeadlineBreached": 1,
  "oldestCreatedAt": "2026-07-01T10:30:00Z"
}
```

### 3. Testing

**Unit tests (runtime module):**
- Rename `InboxSummaryBuilderTest` → `WorkItemSummaryBuilderTest`
- New test: multiple items with different `createdAt` → oldest non-terminal selected
- New test: all items terminal → `oldestCreatedAt` is null
- New test: empty list → `oldestCreatedAt` is null

**REST test (queues module):**
- New `QueueSummaryResourceTest` (follows `QueueResourceTest` pattern)
- Happy path: queue with items at various statuses/priorities → verify all fields
- Empty queue: `total: 0`, empty maps, `oldestCreatedAt: null`
- Queue not found: 404
- Mixed terminal/non-terminal: `oldestCreatedAt` reflects only non-terminal

**Existing inbox test:**
- `InboxSummaryTest` — update for renamed types, verify `oldestCreatedAt` in response

### 4. API reference

Add `GET /queues/{id}/summary` to the Queues section of `docs/api-reference.md`.

## Out of Scope

- Database-level aggregation (push-down optimisation for large queues) — future work
- Caching or materialised views — premature until scale demands it
- No Flyway migration needed — this is a read-only aggregation over existing data
