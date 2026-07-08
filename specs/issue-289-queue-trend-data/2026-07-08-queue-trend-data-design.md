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
**Foreign key:** `queue_view_id REFERENCES queue_view(id) ON DELETE CASCADE` — deleting a QueueView automatically removes its snapshot history.

### Flyway Migration (V2004)

Creates `queue_snapshot` table with the above schema. Lives in `queues/src/main/resources/db/work/migration/` (per repo-scoped migration path convention).

### QueueSnapshotStore

Internal store SPI in the queues module (not a cross-module SPI):

- `put(QueueSnapshot)` — persist a snapshot; stamps `tenancyId` on insert per `store-tenancy-stamping-on-insert` protocol
- `findByQueueAndPeriod(UUID queueViewId, Instant from, Instant to)` — returns snapshots ordered by `snapshotAt` ascending
- `findLatestSnapshotTimes(Collection<UUID> queueViewIds)` — returns `Map<UUID, Instant>` mapping each queue view ID to its most recent `snapshot_at`. Maps to `SELECT queue_view_id, MAX(snapshot_at) FROM queue_snapshot WHERE queue_view_id IN (?1) AND tenancy_id = ?2 GROUP BY queue_view_id`. Batch query eliminates per-queue N+1.
- `deleteOlderThan(Instant cutoff)` — bulk delete for retention cleanup (tenant-scoped via `TenantAwareStore`)

JPA implementation: `JpaQueueSnapshotStore` extending `TenantAwareStore`.

In-memory implementation: `InMemoryQueueSnapshotStore` in `persistence-memory` module. Requires adding `casehub-work-queues` as a dependency of `persistence-memory` — this maintains the existing pattern where `persistence-memory` provides in-memory alternatives for all store SPIs.

### CrossTenantQueueViewStore

Cross-tenant store interface in the queues module, following the `cross-tenant-store-minimal-surface` protocol (PP-20260609-2144e0). Single-method interface bounded to the snapshot job's use case:

- `List<String> findDistinctTenancyIds()` — returns all distinct `tenancy_id` values from `queue_view`. Maps to `SELECT DISTINCT tenancy_id FROM queue_view`.

JPA implementation: `JpaCrossTenantQueueViewStore` extending `TenantAwareStore`, using `withCrossTenantQuery()` and `@Transactional(REQUIRES_NEW)`. Lives in `queues/src/main/java/io/casehub/work/queues/repository/jpa/`.

CDI producer: `QueueCrossTenantProducer` in the queues module — validates `@WorkSystem CurrentPrincipal.isCrossTenantAdmin()` before producing the `@CrossTenant`-qualified instance, following the `CrossTenantProducer` pattern in the runtime module.

### QueueMembershipService

Shared service in the queues module that evaluates queue membership. Extracts the evaluation logic currently in `QueueResource.query()`:

```java
@ApplicationScoped
public class QueueMembershipService {
    @Inject WorkItemStore workItemStore;
    @Inject FilterEvaluatorRegistry evaluatorRegistry;

    /** Returns all matching WorkItems — used by QueueResource to build the response. */
    public List<WorkItem> evaluateMembers(QueueView queue) {
        var candidates = workItemStore.scan(
            WorkItemQuery.byLabelPattern(queue.labelPattern));
        if (queue.additionalConditions != null
                && !queue.additionalConditions.isBlank()) {
            var jexl = evaluatorRegistry.find("jexl");
            if (jexl != null) {
                candidates = candidates.stream()
                    .filter(wi -> jexl.evaluate(wi,
                        ExpressionDescriptor.of("jexl",
                            queue.additionalConditions)))
                    .toList();
            }
        }
        return candidates;
    }

    /** Returns the member count without hydrating entities — used by QueueSnapshotJob. */
    public int countMembers(QueueView queue) {
        if (queue.additionalConditions == null
                || queue.additionalConditions.isBlank()) {
            return (int) workItemStore.countByQuery(
                WorkItemQuery.byLabelPattern(queue.labelPattern));
        }
        return evaluateMembers(queue).size();
    }
}
```

`QueueResource.query()` calls `evaluateMembers()` (needs full entities for the response). `QueueSnapshotJob` calls `countMembers()` (needs only the integer count — avoids hydrating all matching WorkItem entities into memory).

The `countMembers` fast path (no `additionalConditions`) uses `WorkItemStore.countByQuery(WorkItemQuery)` — a new method on the store SPI that maps to SQL `SELECT COUNT(*) ... WHERE label_pattern = ?`. When JEXL filters are present, the full-entity path is unavoidable since the in-memory filter needs entity fields. The fast path covers the common case.

`countByQuery(WorkItemQuery)` is a natural addition to `WorkItemStore` alongside the existing `scan(WorkItemQuery)` — the count pattern already exists (`countByParentAndAssignee`). All three store implementations (JPA, MongoDB, in-memory) need the new method.

## Preferences

Two `PreferenceKey<DurationPreference>` constants in the queues module (`queues/src/main/java/io/casehub/work/queues/config/`), using the platform's existing `DurationPreference` record (which implements `MultiValuePreference extends Preference`). Follows the same pattern as `DeclineTarget.KEY`. These are queues-specific preferences consumed only by `QueueSnapshotJob` — they belong in the queues module (L4 Label System), not in `casehub-work-api` (L1 Domain Baseline). `PreferenceKey` and `DurationPreference` are transitively available via queues → runtime → api → casehub-platform-api.

### QueueSnapshotInterval

- **Namespace:** `casehub.work.queues`
- **Name:** `snapshot-interval`
- **Type:** `DurationPreference` (wraps `java.time.Duration`)
- **Default:** `new DurationPreference(Duration.ofHours(1))`
- **Parser:** `s -> new DurationPreference(Duration.parse(s))`

Controls how frequently queue membership is sampled. The Quartz heartbeat ticks every 5 minutes; the preference determines whether a snapshot is actually taken at each tick. Callers access the underlying duration via `preferenceProvider.get(QueueSnapshotInterval.KEY).duration()`.

### QueueTrendRetention

- **Namespace:** `casehub.work.queues`
- **Name:** `trend-retention`
- **Type:** `DurationPreference` (wraps `java.time.Duration`)
- **Default:** `new DurationPreference(Duration.ofDays(7))`
- **Parser:** `s -> new DurationPreference(Duration.parse(s))`

Controls how long snapshots are retained before pruning. At default (7 days, hourly snapshots), that's 168 rows per queue.

## Snapshot Job

### QueueSnapshotJob

`@ApplicationScoped` bean with `@Scheduled(identity = "queue-snapshot-heartbeat", every = "5m")` heartbeat.

#### Transaction boundaries

The outer `tick()` method is annotated `@Transactional(TxType.NOT_SUPPORTED)` — no encompassing transaction. This follows the `WorkItemScheduleService.processSchedules()` precedent. Each discrete unit of work runs in its own `REQUIRES_NEW` transaction:

- **Cross-tenant query:** `CrossTenantQueueViewStore.findDistinctTenancyIds()` runs in `REQUIRES_NEW` (on the JPA implementation method, following `JpaCrossTenantWorkItemStore.findActiveWithDeadlines()`)
- **Per-tenant preparation:** `prepareSnapshot(tenancyId)` runs in `REQUIRES_NEW` — reads preferences, scans queues, batch-checks snapshot times, and returns a `SnapshotPlan` listing which queues need snapshotting. Follows the `WorkItemScheduleService.findDueSchedules()` pattern — a short-lived read transaction separate from the write transactions.
- **Per-queue snapshot:** `snapshotQueue(tenancyId, queueViewId)` runs in `REQUIRES_NEW` — one failure does not roll back other snapshots
- **Per-tenant retention cleanup:** `cleanupRetention(tenancyId)` runs in `REQUIRES_NEW` — independent of snapshot transactions

#### Job flow

At each tick:

1. Query distinct `tenancyId` values from `CrossTenantQueueViewStore.findDistinctTenancyIds()` (`REQUIRES_NEW`)
2. For each tenant, wrap in `TenantContextRunner.runInTenantContext(tenancyId, ...)`:
   a. `prepareSnapshot(tenancyId)` (`REQUIRES_NEW`) — reads `QueueSnapshotInterval` preference, loads all `QueueView` definitions via `QueueViewStore.scanAll()`, batch-checks last snapshot times via `QueueSnapshotStore.findLatestSnapshotTimes(queueViewIds)`. Returns a `SnapshotPlan`: the list of queues due for snapshotting plus the tenant's retention duration.
   b. For each queue in the plan:
      - `snapshotQueue(queue)` (`REQUIRES_NEW`) — evaluates membership via `QueueMembershipService.countMembers(queue)`, persists `QueueSnapshot(queueViewId, count, Instant.now())`
   c. `cleanupRetention(retentionDuration)` (`REQUIRES_NEW`) — calls `deleteOlderThan(now - retention)`

Retention cleanup is per-tenant (inside the loop) so each tenant's own retention preference applies and `deleteOlderThan` is tenant-scoped via `TenantAwareStore`.

### Failure Handling

- If a single queue's snapshot fails, log the error and continue to the next queue (the `REQUIRES_NEW` transaction for that queue rolls back independently)
- Missing snapshots create gaps — the trend endpoint returns whatever data points exist
- No retry logic — the next heartbeat tick will attempt again

### Clustering

Quartz clustered mode (`quarkus.quartz.clustered=true`) with the explicit `identity = "queue-snapshot-heartbeat"` ensures only one node fires the trigger per interval. This is the standard `@Scheduled` dedup mechanism — every existing `@Scheduled` job specifies an identity (e.g., `WorkItemScheduleService`: `identity = "work-item-schedule-check"`).

The unique constraint on `(tenancy_id, queue_view_id, snapshot_at)` provides a secondary safety net — if two snapshots for the same queue arrive with the same timestamp, the constraint violation is caught and logged.

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

No server-side bucket aggregation. Issue #289 mentioned `bucket=1h` for server-side aggregation, but this is unnecessary: snapshots are already taken at a configured interval (default hourly), so data points naturally align with bucket boundaries. The client receives raw snapshot-granularity data and decides rendering (sparkline, bar chart, interpolation).

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
| `runtime` | `WorkItemStore.java` | Add `countByQuery(WorkItemQuery)` method to store SPI |
| `runtime` | `JpaWorkItemStore.java` | Implement `countByQuery` via Panache count query |
| `queues` | `QueueSnapshotInterval.java` | New preference constant (`PreferenceKey<DurationPreference>`) |
| `queues` | `QueueTrendRetention.java` | New preference constant (`PreferenceKey<DurationPreference>`) |
| `queues` | `QueueSnapshot.java` | New JPA entity |
| `queues` | `QueueSnapshotStore.java` | New store interface |
| `queues` | `JpaQueueSnapshotStore.java` | JPA store implementation |
| `queues` | `CrossTenantQueueViewStore.java` | New cross-tenant store interface (single method: `findDistinctTenancyIds()`) |
| `queues` | `JpaCrossTenantQueueViewStore.java` | JPA cross-tenant implementation using `withCrossTenantQuery()` |
| `queues` | `QueueCrossTenantProducer.java` | CDI producer for `@CrossTenant` queue stores |
| `queues` | `QueueMembershipService.java` | Extracted membership evaluation with `evaluateMembers()` and `countMembers()` |
| `queues` | `QueueSnapshotJob.java` | Scheduled snapshot + cleanup job |
| `queues` | `QueueResource.java` | Add `GET /queues/{id}/trend`; refactor `query()` to use `QueueMembershipService` |
| `queues` | `QueueTrendResponse.java` | Response DTO |
| `queues` | `V2004__queue_snapshot.sql` | Flyway migration (includes FK to `queue_view` with `ON DELETE CASCADE`) |
| `queues` | `QueueSnapshotJobTest.java` | Unit tests |
| `queues` | `QueueTrendIntegrationTest.java` | Integration tests |
| `persistence-memory` | `InMemoryQueueSnapshotStore.java` | In-memory store for testing |
| `persistence-memory` | `InMemoryWorkItemStore.java` | Implement `countByQuery` |
| `persistence-memory` | `pom.xml` | Add `casehub-work-queues` dependency |
| `persistence-mongodb` | `MongoWorkItemStore.java` | Implement `countByQuery` |
| `docs` | `api-reference.md` | Document new endpoint |
