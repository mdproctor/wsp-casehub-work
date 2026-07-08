# Queue Trend Data — Design Spec

**Issue:** casehubio/work#289
**Date:** 2026-07-08
**Status:** Approved

## Problem

Queue dashboard cards need sparkline trend data (member count over time). No historical queue membership data exists — all queue queries evaluate membership live against the current WorkItem store.

## Solution

Periodic snapshots of queue membership counts, stored as a JPA entity, queryable via a REST endpoint.

## Data Model

### QueueSnapshot Entity

| Field | Type | Column | Constraint |
|-------|------|--------|------------|
| `id` | UUID | `id` | PK, auto-generated |
| `tenancyId` | String | `tenancy_id` | NOT NULL |
| `queueViewId` | UUID | `queue_view_id` | NOT NULL |
| `memberCount` | long | `member_count` | NOT NULL |
| `snapshotAt` | Instant | `snapshot_at` | NOT NULL |

**Index:** `(tenancy_id, queue_view_id, snapshot_at)` — covers the trend query.
**Unique constraint:** `(tenancy_id, queue_view_id, snapshot_at)` — prevents duplicate snapshots.

### Flyway Migration (V2004)

Creates `queue_snapshot` table with the above schema. Lives in `queues/src/main/resources/db/work/migration/` (per repo-scoped migration path convention).

### QueueSnapshotStore

Internal store SPI in the queues module (not a cross-module SPI):

- `put(QueueSnapshot)` — persist a snapshot; stamps `tenancyId` on insert per `store-tenancy-stamping-on-insert` protocol
- `findByQueueAndPeriod(UUID queueViewId, Instant from, Instant to)` — returns snapshots ordered by `snapshotAt` ascending
- `deleteOlderThan(Instant cutoff)` — bulk delete for retention cleanup

JPA implementation: `JpaQueueSnapshotStore` extending `TenantAwareStore`.

In-memory implementation: `InMemoryQueueSnapshotStore` in `persistence-memory` module for testing.

## Preferences

Two typed `PreferenceKey<T>` records in `casehub-work-api`, following the `DeclineTarget` precedent and the `typed-preference-keys` protocol.

### QueueSnapshotInterval

- **Namespace:** `casehub.work.queues`
- **Name:** `snapshot-interval`
- **Type:** `Duration`
- **Default:** `PT1H` (1 hour)
- **Parser:** `Duration.parse(s)`

Controls how frequently queue membership is sampled. The Quartz heartbeat ticks every 5 minutes; the preference determines whether a snapshot is actually taken at each tick.

### QueueTrendRetention

- **Namespace:** `casehub.work.queues`
- **Name:** `trend-retention`
- **Type:** `Duration`
- **Default:** `P7D` (7 days)
- **Parser:** `Duration.parse(s)`

Controls how long snapshots are retained before pruning. At default (7 days, hourly snapshots), that's 168 rows per queue.

## Snapshot Job

### QueueSnapshotJob

`@ApplicationScoped` bean with `@Scheduled(every = "5m")` heartbeat.

At each tick:

1. Query distinct `tenancyId` values from `QueueView` (single cross-tenant query)
2. For each tenant, wrap in `TenantContextRunner.runInTenantContext(tenancyId, ...)`:
   a. Read `QueueSnapshotInterval` preference via `PreferenceProvider`
   b. Load all `QueueView` definitions via `QueueViewStore.scanAll()`
   c. For each queue, check last snapshot time from `QueueSnapshotStore`
   d. Skip if last snapshot is within the configured interval
   e. Evaluate membership using the same path as `GET /queues/{id}`:
      - `workItemStore.scan(WorkItemQuery.byLabelPattern(q.labelPattern))`
      - Apply `additionalConditions` via `FilterEvaluatorRegistry` if set
   f. Persist `QueueSnapshot(queueViewId, count, Instant.now())`
3. Run retention cleanup: read `QueueTrendRetention` preference, call `deleteOlderThan(now - retention)`

### Failure Handling

- If a single queue's evaluation fails, log the error and continue to the next queue
- Missing snapshots create gaps — the trend endpoint returns whatever data points exist
- No retry logic — the next heartbeat tick will attempt again

### Clustering

Quartz clustered mode (`quarkus.quartz.clustered=true`) prevents duplicate execution across nodes. This is the existing pattern used by `ExpiryTimerJob`.

## REST Endpoint

### GET /queues/{id}/trend

Added to existing `QueueResource`.

**Query parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `period` | Duration string | `24h` | How far back (`1h`, `24h`, `7d`, `30d`) |

**Response:** `200 OK`

```json
{
  "queueViewId": "550e8400-e29b-41d4-a716-446655440000",
  "queueName": "Legal Reviews",
  "period": "PT24H",
  "dataPoints": [
    { "snapshotAt": "2026-07-07T00:00:00Z", "memberCount": 42 },
    { "snapshotAt": "2026-07-07T01:00:00Z", "memberCount": 38 }
  ]
}
```

No server-side bucket aggregation — data points are returned at snapshot granularity. Client decides rendering (sparkline, bar chart, interpolation).

**Error cases:**
- `404` — queue not found
- `200` with empty `dataPoints` — queue exists but no snapshots yet

**Period parsing:** accepts ISO-8601 duration (`PT24H`, `P7D`) and shorthand (`24h`, `7d`, `30d`). Shorthand is mapped to ISO-8601 before querying.

## Multi-Tenancy

- `QueueSnapshot.tenancyId` — tenant isolation at the data layer
- `JpaQueueSnapshotStore` extends `TenantAwareStore` — all queries scoped automatically
- Snapshot job uses `TenantContextRunner` for each tenant — `CurrentPrincipal.tenancyId()` available throughout
- Trend endpoint inherits tenant context from the HTTP request (same as existing `GET /queues/{id}`)

## Testing

### Unit Tests (no CDI, no DB)

- **QueueSnapshotJobTest** — mock stores and preference provider:
  - Snapshot persisted when interval elapsed
  - Snapshot skipped when interval not elapsed
  - Retention cleanup called with correct cutoff
  - Failed queue evaluation continues to next queue
  - Multiple tenants processed independently

### Integration Tests (`@QuarkusTest` with H2)

- Create queue → create matching WorkItems → trigger snapshot → `GET /queues/{id}/trend?period=24h` → verify data points contain correct count
- Retention: insert old snapshots → trigger cleanup → verify pruned
- Empty trend: new queue → GET trend → 200 with empty dataPoints
- Deleted queue: delete queue → GET trend → 404

### In-Memory Store

`InMemoryQueueSnapshotStore` in `persistence-memory` module following `InMemoryWorkItemStore` pattern.

## Files Changed

| Module | File | Change |
|--------|------|--------|
| `api` | `QueueSnapshotInterval.java` | New preference record |
| `api` | `QueueTrendRetention.java` | New preference record |
| `queues` | `QueueSnapshot.java` | New JPA entity |
| `queues` | `QueueSnapshotStore.java` | New store interface |
| `queues` | `JpaQueueSnapshotStore.java` | JPA store implementation |
| `queues` | `QueueSnapshotJob.java` | Scheduled snapshot + cleanup job |
| `queues` | `QueueResource.java` | Add `GET /queues/{id}/trend` |
| `queues` | `QueueTrendResponse.java` | Response DTO |
| `queues` | `V2004__queue_snapshot.sql` | Flyway migration |
| `queues` | `QueueSnapshotJobTest.java` | Unit tests |
| `queues` | `QueueTrendIntegrationTest.java` | Integration tests |
| `persistence-memory` | `InMemoryQueueSnapshotStore.java` | In-memory store for testing |
| `docs` | `api-reference.md` | Document new endpoint |
