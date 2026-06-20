# MongoWorkItemDocument Field Sync + OCC

**Issue:** casehubio/work#270
**Date:** 2026-06-20

## Problem

`MongoWorkItemDocument` is missing 13 fields present on the `WorkItem` JPA entity. Data written through the MongoDB store silently drops these fields on write and never restores them on read. This breaks:

- **Multi-instance groups:** `parentId` not persisted → `findByParentId()`, `findByParentIdExcludingStatuses()`, `findByParentIdWithStatuses()`, `countByParentAndAssignee()` all return empty/zero via default linear-scan implementations that read `toDomain()` output.
- **Engine spawn routing:** `callerRef` not persisted → `findByCallerRef()` returns empty. Engine-initiated WorkItems cannot be correlated back to their originating PlanItem.
- **Claim SLA tracking:** `accumulatedUnclaimedSeconds`, `lastReturnedToPoolAt` reset on every round-trip.
- **Named outcomes:** `templateId`, `permittedOutcomes`, `outcome`, `excludedUsers` lost.
- **Schema validation:** `inputDataSchema`, `outputDataSchema` lost.
- **AI metadata:** `confidenceScore` lost.
- **Scope:** `scope` lost.
- **Concurrency:** `version` not mapped. Additionally, `MongoWorkItemStore.put()` uses `persistOrUpdate()` (unconditional upsert) with no version check — two concurrent claims can both succeed. JPA prevents this via `@Version` → `OptimisticLockException` → HTTP 409.

## Design

### 1. MongoWorkItemDocument — 13 new fields

| Field | Doc type | Conversion | Notes |
|-------|----------|------------|-------|
| `version` | `Long` | Direct | OCC version, managed by store |
| `accumulatedUnclaimedSeconds` | `long` | Direct | Primitive, default 0 |
| `lastReturnedToPoolAt` | `Instant` | Direct | |
| `confidenceScore` | `Double` | Direct | |
| `callerRef` | `String` | Direct | |
| `parentId` | `String` | `UUID.toString()` / `UUID.fromString()` | Same as `id` |
| `scope` | `String` | Direct | |
| `templateId` | `String` | `UUID.toString()` / `UUID.fromString()` | Same as `id` |
| `permittedOutcomes` | `String` | Direct | JSON array string — not queried at store level |
| `excludedUsers` | `List<String>` | `splitCsv()` / `joinCsv()` | Consistent with candidateGroups/candidateUsers |
| `outcome` | `String` | Direct | |
| `inputDataSchema` | `String` | Direct | |
| `outputDataSchema` | `String` | Direct | |

**`excludedUsers` as `List<String>`:** Same CSV→array conversion as `candidateGroups`/`candidateUsers`. Enables `$in` queries and follows the established MongoDB-native representation for sets of user IDs.

**`permittedOutcomes` as `String`:** It's a JSON array, not CSV. Service layer owns parsing. No store-level query need.

All fields mapped in both `from()` and `toDomain()`.

### 2. MongoWorkItemStore.put() — optimistic concurrency control

Replace the current `persistOrUpdate()` upsert with version-checked writes.

**Insert path** (workItem.version == null):
1. Assign id, tenancyId, timestamps (unchanged)
2. Set `workItem.version = 0L`
3. `MongoWorkItemDocument.from(workItem).persist()`

**Update path** (workItem.version != null):
1. Update timestamps (unchanged)
2. Build doc via `MongoWorkItemDocument.from(workItem)`
3. Override `doc.version = workItem.version + 1`
4. `replaceOne(filter={_id: id, version: workItem.version}, doc)`
5. If `modifiedCount == 0` → throw `jakarta.persistence.OptimisticLockException`
6. Set `workItem.version = doc.version`

**Why `replaceOne` instead of `findOneAndUpdate`:** SpawnGroupStore/ScheduleStore use `findOneAndUpdate` with explicit `Updates.set()` per field (~12 fields each). WorkItem has 44 fields — enumerating them all is brittle. `replaceOne` reuses `from()` for the full document; future field additions to `from()` are automatically included.

**Insert vs update detection via `version == null`:** Avoids the SpawnGroupStore pattern of querying `find({_id: id}).firstResult() != null` (extra round trip). Mirrors JPA semantics: null version = transient, non-null = managed.

**Tenancy protocol compliance:** Unchanged — `tenancyId` stamped on insert when null, preserved on update.

### 3. MongoWorkItemStore — 5 query method overrides

All currently fall back to default `WorkItemStore` implementations that linear-scan `scanAll()`. With `callerRef` and `parentId` now persisted, these become proper indexed queries. All are tenant-scoped.

- **`findByCallerRef(String callerRef)`** — filter `{callerRef, tenancyId}`, return first or empty
- **`findByParentId(UUID parentId)`** — filter `{parentId, tenancyId}`, return list
- **`findByParentIdExcludingStatuses(UUID, List<WorkItemStatus>)`** — filter `{parentId, tenancyId, status: {$nin: ...}}`
- **`findByParentIdWithStatuses(UUID, List<WorkItemStatus>)`** — filter `{parentId, tenancyId, status: {$in: ...}}`
- **`countByParentAndAssignee(UUID, String, UUID)`** — filter `{parentId, assigneeId, tenancyId, _id: {$ne: excludeId}}`, return count

### 4. Test coverage

Extend `MongoWorkItemStoreTest`:

**Roundtrip:** Single test populating all 13 fields → `put()` → `get()` → assert every field. `excludedUsers` CSV `"alice,bob"` round-trips through `List<String>` storage.

**OCC:**
- `put_setsVersionToZero_onInsert`
- `put_incrementsVersion_onUpdate`
- `put_throwsOptimisticLockException_onStaleVersion` (mirrors SpawnGroupStoreTest/ScheduleStoreTest pattern)

**Query methods:**
- `findByCallerRef_returnsMatching` / `_returnsEmpty_whenNotFound`
- `findByParentId_returnsChildren`
- `findByParentIdExcludingStatuses_excludesTerminal`
- `findByParentIdWithStatuses_filtersToGivenStatuses`
- `countByParentAndAssignee_excludesSelf`

## Files changed

| File | Change |
|------|--------|
| `persistence-mongodb/.../MongoWorkItemDocument.java` | Add 13 fields, update `from()`, `toDomain()` |
| `persistence-mongodb/.../MongoWorkItemStore.java` | OCC in `put()`, 5 query overrides |
| `persistence-mongodb/.../MongoWorkItemStoreTest.java` | Roundtrip, OCC, query method tests |

## Protocol coherence

- **store-tenancy-stamping-on-insert:** `put()` stamps on insert, preserves on update — unchanged.
- **cross-tenant-store-minimal-surface:** `MongoCrossTenantWorkItemStore` uses `MongoWorkItemDocument.toDomain()` — benefits automatically from the field mapping fix. No changes needed there.
