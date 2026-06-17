# #253 — Complete MongoDB Persistence Module

## Problem

`persistence-mongodb/` implements `WorkItemStore` and `AuditEntryStore` but not the remaining three SPIs. Consumers wanting MongoDB-only persistence still need a JPA datasource for:

- `WorkItemNoteStore` (runtime)
- `IssueLinkStore` (issue-tracker)
- `RoutingCursorStore` (core)

## Approach

Implement all three in `persistence-mongodb/` following the established pattern: Panache MongoDB document class + `@Alternative @Priority(1)` store class + tenant scoping via `CurrentPrincipal`.

Add `casehub-work-issue-tracker` as a compile dependency of `persistence-mongodb`. The module already has JPA on its classpath transitively via `casehub-work` (runtime), so the added coupling is minimal.

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

### MongoIssueLinkStore + MongoIssueLinkDocument

- Collection: `work_item_issue_links`
- Fields: `id`, `tenancyId`, `workItemId`, `trackerType`, `externalRef`, `title`, `url`, `status`, `linkedAt`, `linkedBy`
- `findById()`: filter on `_id + tenancyId`
- `findByWorkItemId()`: filter on `workItemId + tenancyId`, sort by `linkedAt ASC`
- `findByRef()`: compound filter on `workItemId + trackerType + externalRef + tenancyId`
- `findByTrackerRef()`: filter on `trackerType + externalRef + tenancyId`
- `save()`: assign UUID + linkedAt + tenancyId if absent, then `persistOrUpdate()`
- `delete()`: filter by `_id + tenancyId`, call `deleteOne()`

### MongoRoutingCursorStore + MongoRoutingCursorDocument

- Collection: `routing_cursors`
- Document ID: `poolHash + ":" + tenancyId` (composite string key)
- `acquireNext()`: use `MongoCollection.findOneAndUpdate()` with `$inc: {lastIndex: 1}` and `upsert: true`. The document stores a raw counter; `acquireNext()` returns `result.lastIndex % poolSize`. Atomic — no CAS loop needed.
- `lastAccessed` stamped via `$set` in the same update.
- New documents start at `lastIndex: 0` (the `$inc` on upsert sets `0 + 1 = 1`, but we handle the initial value by setting the default in `$setOnInsert`).

## CDI

All three stores: `@ApplicationScoped @Alternative @Priority(1)` — Tier 2 per `persistence-backend-cdi-priority.md`.

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

Update the three SPI interfaces to replace "No Tier 2 (MongoDB) exists yet" with the Tier 2 entry referencing `casehub-work-persistence-mongodb`.

## Out of Scope — Filed Separately

**CrossTenantProducer hardcodes JPA types.** `CrossTenantProducer` directly injects `JpaCrossTenantRoutingCursorStore` (and the other Jpa* cross-tenant stores). When MongoDB is active, tenant-scoped `RoutingCursorStore` routes to Mongo but `@CrossTenant CrossTenantRoutingCursorStore` still routes to JPA — cleanup job operates on the wrong backend. This is a design issue with the producer pattern, not with #253.

## Tests

Each store gets a `@QuarkusTest` following the existing `MongoWorkItemStoreTest` / `MongoAuditEntryStoreTest` pattern:
- `MongoWorkItemNoteStoreTest` — append/find/update/delete roundtrip, tenant isolation
- `MongoIssueLinkStoreTest` — save/find/findByRef/findByTrackerRef/delete, tenant isolation
- `MongoRoutingCursorStoreTest` — sequential acquireNext advances cursor, wraps at poolSize, upsert on first call
