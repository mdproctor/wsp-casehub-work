# #253 — Complete MongoDB Persistence Module

## Problem

`persistence-mongodb/` implements `WorkItemStore` and `AuditEntryStore` but not the remaining seven runtime/core SPIs. Consumers wanting MongoDB-only persistence still need a JPA datasource for all of these:

| SPI | Module | Complexity |
|-----|--------|-----------|
| WorkItemNoteStore | runtime | Simple CRUD |
| WorkItemScheduleStore | runtime | Simple CRUD + `findDue()` query |
| WorkItemLinkStore | runtime | Simple CRUD + `findByWorkItemIdAndType()` |
| LabelVocabularyStore | runtime | Simple CRUD |
| FilterRuleStore | runtime | Simple CRUD |
| RoutingCursorStore | core | Atomic increment (`acquireNext()`) |
| IssueLinkStore | issue-tracker | CRUD + compound-key lookups |

The original issue listed only three (WorkItemNoteStore, IssueLinkStore, RoutingCursorStore). The actual gap is seven.

## Approach

Implement all seven in `persistence-mongodb/` following the established pattern: Panache MongoDB document class + `@Alternative @Priority(1)` store class + tenant scoping via `CurrentPrincipal`.

Add `casehub-work-issue-tracker` as a compile dependency of `persistence-mongodb`. This follows the `persistence-memory` precedent — both persistence backend modules centralize all store implementations in one module. The alternative (each optional module contributing its own Mongo store) distributes MongoDB knowledge across the codebase rather than centralizing it. The centralized approach is better: one module owns all MongoDB persistence, one place to look, one dependency to add.

## Implementations

### MongoWorkItemNoteStore + MongoWorkItemNoteDocument

- Collection: `work_item_notes`
- Fields: `id` (BsonId, String), `tenancyId`, `workItemId`, `content`, `author`, `createdAt`, `editedAt`
- Standard `from(WorkItemNote)` / `toDomain()` conversion
- `append()`: assign UUID + createdAt + tenancyId if absent, then `persist()`
- `findById()`: filter on `_id + tenancyId`
- `findByWorkItemId()`: filter on `workItemId + tenancyId`, sort by `createdAt ASC`
- `update()`: stamp tenancyId if absent, then `persistOrUpdate()`
- `delete()`: `deleteOne()` with `_id + tenancyId` filter, return `deletedCount > 0`

**tenancyId stamping on update():** The "if absent" guard on `update()` follows the JPA and InMemory implementations. A note reaching `update()` without tenancyId is itself a caller bug (the insert path should have stamped it). The guard silently masks that. Fixing the contract to throw instead belongs in a cross-tier design change, not scoped to #253.

### MongoIssueLinkStore + MongoIssueLinkDocument

- Collection: `work_item_issue_links`
- Fields: `id`, `tenancyId`, `workItemId`, `trackerType`, `externalRef`, `title`, `url`, `status`, `linkedAt`, `linkedBy`
- `findById()`: filter on `_id + tenancyId`
- `findByWorkItemId()`: filter on `workItemId + tenancyId`, sort by `linkedAt ASC`
- `findByRef()`: compound filter on `workItemId + trackerType + externalRef + tenancyId`
- `findByTrackerRef()`: filter on `trackerType + externalRef + tenancyId`
- `save()`: assign UUID + linkedAt + tenancyId if absent, then `persistOrUpdate()`
- `delete(WorkItemIssueLink link)`: extract `link.id`, filter by `_id` + validate `currentPrincipal.tenancyId()` matches the stored document's tenancyId before deleting (matching InMemory behavior)

### MongoRoutingCursorStore + MongoRoutingCursorDocument

- Collection: `routing_cursors`
- Document ID: `poolHash + ":" + tenancyId` (composite string key)
- Fields: `lastIndex` (BSON int64 / NumberLong), `lastAccessed` (Instant)
- `acquireNext()`: obtain the raw `MongoCollection` via `MongoRoutingCursorDocument.mongoCollection()` (Panache escape hatch — the standard Panache CRUD doesn't support atomic `findOneAndUpdate`). Execute `findOneAndUpdate()` with:
  - Filter: `{_id: poolHash:tenancyId}`
  - Update: `{$inc: {lastIndex: 1}, $set: {lastAccessed: now}, $setOnInsert: {lastIndex: -1}}`
  - Options: `upsert: true, returnDocument: AFTER`
- First call (upsert): `$setOnInsert` sets `lastIndex: -1`, then `$inc` advances to 0. Returned document has `lastIndex: 0`. `acquireNext()` returns `Math.floorMod(0, poolSize) = 0`. Matches JPA/InMemory first-call behavior.
- `lastIndex` is stored as BSON int64 (NumberLong) to extend the overflow horizon beyond int32's ~2.1B limit.
- Java side uses `Math.floorMod()` instead of `%` to guarantee non-negative results even if the counter somehow wraps.

### MongoWorkItemScheduleStore + MongoWorkItemScheduleDocument

- Collection: `work_item_schedules`
- Fields: `id`, `tenancyId`, `name`, `templateId`, `cronExpression`, `active`, `createdBy`, `createdAt`, `lastFiredAt`, `nextFireAt`, `version` (Long)
- `put()`: assign UUID + createdAt + tenancyId if absent, `persistOrUpdate()`
- `get()`: filter on `_id + tenancyId`
- `scanAll()`: filter on `tenancyId`, sort by `name ASC`
- `delete()`: filter on `_id + tenancyId`, return `deletedCount > 0`
- `findDue()`: filter on `active: true AND nextFireAt != null AND nextFireAt <= now AND tenancyId`, sort by `nextFireAt ASC`
- **OCC note:** The JPA entity uses `@Version` for optimistic locking (prevents double-fire in clustered deployments). MongoDB's `findOneAndUpdate` with a version-check filter provides equivalent OCC semantics.

### MongoWorkItemLinkStore + MongoWorkItemLinkDocument

- Collection: `work_item_links`
- Fields: `id`, `tenancyId`, `workItemId`, `url`, `title`, `relationType`, `linkedBy`, `createdAt`
- `put()`: assign UUID + createdAt + tenancyId if absent, `persistOrUpdate()`
- `get()`: filter on `_id + tenancyId`
- `findByWorkItemId()`: filter on `workItemId + tenancyId`, sort by `createdAt ASC`
- `findByWorkItemIdAndType()`: filter on `workItemId + relationType + tenancyId`, sort by `createdAt ASC`
- `delete()`: filter on `_id + tenancyId`, return `deletedCount > 0`

### MongoLabelVocabularyStore + MongoLabelVocabularyDocument

- Collection: `label_vocabularies`
- Fields: `id`, `tenancyId`, `scope` (String — `Path.toString()` on write, `Path.of()` on read), `name`
- `put()`: assign UUID + tenancyId if absent, `persistOrUpdate()`
- `get()`: filter on `_id + tenancyId`
- `scanAll()`: filter on `tenancyId`
- `findByScope()`: filter on `scope + tenancyId`
- `findOrCreate()`: inherits default method from interface (check-then-act); production-safe override using `findOneAndUpdate` with `upsert: true` for atomicity
- `delete()`: filter on `_id + tenancyId`, return `deletedCount > 0`

### MongoFilterRuleStore + MongoFilterRuleDocument

- Collection: `filter_rules`
- Fields: `id`, `tenancyId`, `name`, `description`, `enabled`, `condition`, `events`, `actionsJson`, `createdAt`
- `put()`: assign UUID + createdAt + tenancyId if absent, `persistOrUpdate()`
- `get()`: filter on `_id + tenancyId`
- `allEnabled()`: filter on `enabled: true + tenancyId`, sort by `createdAt ASC`
- `scanAll()`: filter on `tenancyId`, sort by `createdAt ASC`
- `delete()`: filter on `_id + tenancyId`, return `deletedCount > 0`

## CDI

All seven stores: `@ApplicationScoped @Alternative @Priority(1)` — Tier 2 per `persistence-backend-cdi-priority.md` (located at `casehub/garden/docs/protocols/universal/persistence-backend-cdi-priority.md`).

## Dependencies

Add to `persistence-mongodb/pom.xml`:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-work-issue-tracker</artifactId>
  <version>${project.version}</version>
</dependency>
```

## SPI Javadoc

Update all seven SPI interfaces: replace any "No Tier 2 (MongoDB) exists yet" lines with the Tier 2 entry referencing `casehub-work-persistence-mongodb`.

## Out of Scope — Filed as #267

**CrossTenantProducer hardcodes JPA types — data integrity impact.** `CrossTenantProducer` directly injects `JpaCrossTenantRoutingCursorStore`, `JpaCrossTenantWorkItemStore`, `JpaCrossTenantWorkItemScheduleStore`. When MongoDB is active:

- **RoutingCursorCleanupJob** calls `cleanupStale()` against an empty JPA table while live cursor data sits in MongoDB. Stale cursors accumulate forever with no cleanup.
- **WorkItemScheduleService** (via `CrossTenantWorkItemScheduleStore.findActive()`) finds zero schedules in the empty JPA table while live schedules sit in MongoDB. Recurring WorkItem creation silently stops.

This is a design defect in the producer pattern, not in the individual stores. Filed as #267.

## Tests

Each store gets a `@QuarkusTest` following the existing `MongoWorkItemStoreTest` / `MongoAuditEntryStoreTest` pattern. Inline in a single test class per store (matching the MongoDB module's convention).

Tenant isolation tests are inline — each test creates data under two distinct tenancyIds and asserts queries only return data for the active tenant:

- `MongoWorkItemNoteStoreTest` — append/findById/findByWorkItemId/update/delete roundtrip; tenant isolation (note created by tenant-A invisible to tenant-B)
- `MongoIssueLinkStoreTest` — save/findById/findByWorkItemId/findByRef/findByTrackerRef/delete; delete extracts `link.id` and validates tenancyId; tenant isolation
- `MongoRoutingCursorStoreTest` — first acquireNext returns 0; sequential calls advance and wrap at poolSize; concurrent-safe (atomic); tenant isolation (cursor state per-tenant)
- `MongoWorkItemScheduleStoreTest` — put/get/scanAll/delete/findDue roundtrip; findDue only returns active schedules with nextFireAt <= now; tenant isolation
- `MongoWorkItemLinkStoreTest` — put/get/findByWorkItemId/findByWorkItemIdAndType/delete; tenant isolation
- `MongoLabelVocabularyStoreTest` — CRUD + findByPathPrefix; tenant isolation
- `MongoFilterRuleStoreTest` — CRUD + findActive; tenant isolation
