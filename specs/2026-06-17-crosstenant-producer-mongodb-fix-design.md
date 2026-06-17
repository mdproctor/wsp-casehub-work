# CrossTenantProducer MongoDB Backend Fix

**Issue:** #267
**Date:** 2026-06-17
**Status:** Approved

## Problem

`CrossTenantProducer` directly injects three JPA concrete types:
- `JpaCrossTenantWorkItemStore`
- `JpaCrossTenantWorkItemScheduleStore`
- `JpaCrossTenantRoutingCursorStore`

When `persistence-mongodb` is on the classpath, tenant-scoped stores correctly
resolve to Mongo implementations via `@Alternative @Priority(1)`, but the
cross-tenant path still hardcodes JPA. Result: `RoutingCursorCleanupJob` deletes
from the empty JPA store while live cursors are in MongoDB (silent no-op), and
`TimerRecoveryStartup` scans the wrong backing store for active items.

## Approach

Change `CrossTenantProducer` to inject by interface instead of JPA concrete type.
Create MongoDB implementations of the three cross-tenant store interfaces with
`@Alternative @Priority(1)` so CDI resolves them when `persistence-mongodb` is
on the classpath — same mechanism as tenant-scoped stores.

The producer keeps its `validateSystemPrincipal()` security gate unchanged.

### Rejected alternative

Eliminating the producer entirely and moving `isCrossTenantAdmin()` validation
into each store implementation. Rejected because it scatters security validation
across 6 classes (3 JPA + 3 Mongo) instead of centralising it in one place.

## Changes

### 1. CrossTenantProducer (runtime)

Replace three `@Inject` fields from JPA concrete types to interfaces:

```java
// Before
@Inject JpaCrossTenantWorkItemStore crossTenantWorkItemStore;
@Inject JpaCrossTenantWorkItemScheduleStore crossTenantScheduleStore;
@Inject JpaCrossTenantRoutingCursorStore crossTenantCursorStore;

// After
@Inject CrossTenantWorkItemStore crossTenantWorkItemStore;
@Inject CrossTenantWorkItemScheduleStore crossTenantScheduleStore;
@Inject CrossTenantRoutingCursorStore crossTenantCursorStore;
```

Remove the 3 JPA concrete imports. The 3 interface imports are already present.
`@Produces` methods and `validateSystemPrincipal()` unchanged.

### 2. MongoDB cross-tenant stores (persistence-mongodb)

Three new classes in `io.casehub.work.mongodb`:

**MongoCrossTenantWorkItemStore** implements `CrossTenantWorkItemStore`
- `findActiveWithDeadlines()`: query `work_items` for non-terminal statuses where
  `expiresAt` or `claimDeadline` is non-null. No `tenancyId` filter.
- `@ApplicationScoped @Alternative @Priority(1)`

**MongoCrossTenantWorkItemScheduleStore** implements `CrossTenantWorkItemScheduleStore`
- `findActive()`: query `work_item_schedules` for `active: true`. No `tenancyId` filter.
- `@ApplicationScoped @Alternative @Priority(1)`

**MongoCrossTenantRoutingCursorStore** implements `CrossTenantRoutingCursorStore`
- `cleanupStale(Instant cutoff)`: delete from `routing_cursors` where
  `lastAccessed < cutoff`. No `tenancyId` filter. Return deleted count.
- `@ApplicationScoped @Alternative @Priority(1)`

Each injects `MongoClient` and accesses the database/collection following the
same pattern as existing tenant-scoped Mongo stores. Cross-tenant in MongoDB
means omitting the `tenancyId` predicate — there is no RLS equivalent to bypass.

### 3. Tests (persistence-mongodb)

One test class per new store, following existing Mongo store test patterns:
- Insert documents across multiple tenancies
- Verify cross-tenant methods return results from all tenancies
- For cursor cleanup: verify correct delete count and non-stale row survival

Existing `CrossTenantWorkItemStoreTest` in runtime (JPA-specific) unchanged.
No new integration test classes — existing `TimerRecoveryStartup` and
`RoutingCursorCleanupJob` exercise the cross-tenant path through the producer.

## Scope

- 1 file modified: `CrossTenantProducer.java` (import swap)
- 3 files created: Mongo cross-tenant store implementations
- 3 files created: Mongo cross-tenant store tests
- No new interfaces, no API changes, no migrations
