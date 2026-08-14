# WorkItemStore SPI Extraction + Progress API Docs

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #337 — refactor: extract WorkItemStore SPI + WorkItem POJO into casehub-work-api
**Issue group:** #333, #337, #340

**Goal:** Extract WorkItemStore SPI and WorkItem domain type from runtime/ to api/, enabling consumers like persistence-memory to depend on api/ alone without JPA. Document the progress REST API.

**Architecture:** JPA entity renamed to WorkItemEntity first (eliminates name collision). Then WorkItem becomes an immutable record with builder in api/. Standalone WorkItemEntityMapper handles bidirectional conversion. InMemoryWorkItemStore dependency shifts from runtime → api/. CrossTenantWorkItemStore also extracted. LabelPatternMatcher extracted from LabelVocabularyService for persistence-memory label filtering.

**Task ordering rationale:** Entity rename (Task 2) runs before record creation (Task 3) to avoid two types named `WorkItemEntity` coexisting. Every intermediate commit compiles.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Maven multi-module

## Global Constraints

- `api/` module: no JPA, no CDI, no Quarkus dependencies — pure Java only
- All SPI boundary types in api/ are immutable (records or final classes with builders)
- version field (OCC) excluded from SPI types — stays on entity only
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl <module>`
- Test timeouts: see `scripts/README.md`
- Follow existing `ProgressInstanceMapper` pattern for mapper placement and style
- Type hierarchy matching: use `Path.parse()` + `isAncestorOf()` (available in api/ via casehub-platform-api dependency) — do NOT switch to string prefix matching

---

### Task 1: Close #340 — resolvedScope fix already shipped

**Files:**
- None — GitHub issue only

- [ ] **Step 1: Verify fix is in place**

Use `ide_find_definition` or `ide_read_file` on `HumanTaskScheduleHandler` to confirm `event.resolvedScope()` is used at lines 132-135 and 235-243. Do not close without verification.

- [ ] **Step 2: Close the issue**

```bash
gh issue close 340 --repo casehubio/work --comment "Already fixed — HumanTaskScheduleHandler uses event.resolvedScope() / event.resolvedTitle() with fallback to target.scope() / target.title() (lines 132-135, 235-243). Same pattern applied for both createInline() and handleTemplateMode()."
```

---

### Task 2: Rename entities in runtime/ (eliminates name collision)

**Files:**
- Rename: `WorkItemEntity` → `WorkItemEntity` in `runtime/src/main/java/io/casehub/work/runtime/model/` (use `ide_refactor_rename`)
- Rename: `WorkItemLabelEntity` → `WorkItemLabelEntity` in `runtime/src/main/java/io/casehub/work/runtime/model/` (use `ide_refactor_rename`)

**Interfaces:**
- Consumes: nothing new
- Produces: `WorkItemEntity` (renamed from `WorkItemEntity`), `WorkItemLabelEntity` (renamed from `WorkItemLabelEntity`)

- [ ] **Step 1: Rename WorkItem to WorkItemEntity using ide_refactor_rename**

Rename the class at `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`. IntelliJ updates all references across the project — entity references, store interfaces, test references, JPA queries.

Verify `@Table(name = "work_item")` is unchanged (table name stays the same).

- [ ] **Step 2: Rename WorkItemLabel to WorkItemLabelEntity using ide_refactor_rename**

Rename at `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemLabel.java`.

- [ ] **Step 3: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime`
Expected: BUILD SUCCESS — all references updated by IDE refactor

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "refactor(#337): rename WorkItem → WorkItemEntity, WorkItemLabel → WorkItemLabelEntity

Eliminates name collision before creating api/ WorkItem record.
Table names unchanged.

Refs #337"
```

---

### Task 3: Create api/ foundation types

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/WorkItemLabel.java`
- Create: `api/src/main/java/io/casehub/work/api/LabelPatternMatcher.java`
- Create: `api/src/main/java/io/casehub/work/api/WorkItem.java`
- Test: `api/src/test/java/io/casehub/work/api/WorkItemBuilderTest.java`
- Test: `api/src/test/java/io/casehub/work/api/LabelPatternMatcherTest.java`

**Interfaces:**
- Consumes: `WorkItemStatus`, `WorkItemPriority`, `DeclineTarget`, `LabelPersistence` (all already in api/)
- Produces: `WorkItemEntity` record (46 fields + builder + `toBuilder()`), `WorkItemLabelEntity` record, `LabelPatternMatcher.matchesPattern(String pattern, String path)`

- [ ] **Step 1: Write WorkItemLabel record**

```java
package io.casehub.work.api;

public record WorkItemLabel(String path, LabelPersistence persistence, String appliedBy) {}
```

- [ ] **Step 2: Write LabelPatternMatcher with failing test first**

Test:
```java
package io.casehub.work.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class LabelPatternMatcherTest {

    @Test
    void exactMatch() {
        assertTrue(LabelPatternMatcher.matchesPattern("legal", "legal"));
        assertFalse(LabelPatternMatcher.matchesPattern("legal", "finance"));
    }

    @Test
    void singleWildcard() {
        assertTrue(LabelPatternMatcher.matchesPattern("legal/*", "legal/contracts"));
        assertFalse(LabelPatternMatcher.matchesPattern("legal/*", "legal/contracts/nda"));
        assertFalse(LabelPatternMatcher.matchesPattern("legal/*", "finance/audit"));
    }

    @Test
    void multiWildcard() {
        assertTrue(LabelPatternMatcher.matchesPattern("legal/**", "legal/contracts"));
        assertTrue(LabelPatternMatcher.matchesPattern("legal/**", "legal/contracts/nda"));
        assertFalse(LabelPatternMatcher.matchesPattern("legal/**", "finance/audit"));
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LabelPatternMatcherTest -pl api`
Expected: FAIL — class does not exist

- [ ] **Step 4: Implement LabelPatternMatcher**

```java
package io.casehub.work.api;

public final class LabelPatternMatcher {

    private LabelPatternMatcher() {}

    public static boolean matchesPattern(final String pattern, final String path) {
        if (pattern.endsWith("/**")) {
            final String prefix = pattern.substring(0, pattern.length() - 3);
            return path.startsWith(prefix + "/");
        }
        if (pattern.endsWith("/*")) {
            final String prefix = pattern.substring(0, pattern.length() - 2);
            if (!path.startsWith(prefix + "/")) {
                return false;
            }
            final String remainder = path.substring(prefix.length() + 1);
            return !remainder.contains("/");
        }
        return pattern.equals(path);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LabelPatternMatcherTest -pl api`
Expected: PASS

- [ ] **Step 6: Write WorkItem record with builder — test first**

Test (covers builder, toBuilder, field access):
```java
package io.casehub.work.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class WorkItemBuilderTest {

    @Test
    void builderCreatesRecord() {
        UUID id = UUID.randomUUID();
        WorkItem item = WorkItem.builder()
                .id(id)
                .tenancyId("tenant-1")
                .title("Review contract")
                .status(WorkItemStatus.PENDING)
                .priority(WorkItemPriority.HIGH)
                .types(Set.of("legal", "compliance/audit"))
                .labels(List.of(new WorkItemLabel("urgent", LabelPersistence.MANUAL, "alice")))
                .build();

        assertEquals(id, item.id());
        assertEquals("tenant-1", item.tenancyId());
        assertEquals("Review contract", item.title());
        assertEquals(WorkItemStatus.PENDING, item.status());
        assertEquals(WorkItemPriority.HIGH, item.priority());
        assertEquals(Set.of("legal", "compliance/audit"), item.types());
        assertEquals(1, item.labels().size());
        assertEquals("urgent", item.labels().get(0).path());
    }

    @Test
    void toBuilderPreservesAllFields() {
        UUID id = UUID.randomUUID();
        Instant now = Instant.now();
        WorkItem original = WorkItem.builder()
                .id(id)
                .tenancyId("t1")
                .title("Original")
                .status(WorkItemStatus.PENDING)
                .priority(WorkItemPriority.MEDIUM)
                .createdAt(now)
                .build();

        WorkItem modified = original.toBuilder()
                .status(WorkItemStatus.ASSIGNED)
                .assigneeId("alice")
                .build();

        assertEquals(id, modified.id());
        assertEquals("t1", modified.tenancyId());
        assertEquals("Original", modified.title());
        assertEquals(WorkItemStatus.ASSIGNED, modified.status());
        assertEquals("alice", modified.assigneeId());
        assertEquals(now, modified.createdAt());
    }

    @Test
    void allFieldsRoundTrip() {
        UUID id = UUID.randomUUID();
        UUID parentId = UUID.randomUUID();
        UUID templateId = UUID.randomUUID();
        Instant now = Instant.now();
        WorkItem original = WorkItem.builder()
                .id(id).tenancyId("t1").title("T").description("D").formKey("fk")
                .status(WorkItemStatus.IN_PROGRESS).priority(WorkItemPriority.URGENT)
                .assigneeId("a").owner("o").candidateGroups("g").candidateUsers("u")
                .requiredCapabilities("rc").createdBy("cb").delegationChain("dc")
                .delegationDeclineTarget(DeclineTarget.POOL).priorStatus(WorkItemStatus.ASSIGNED)
                .payload("p").resolution("r").claimDeadline(now).expiresAt(now)
                .followUpDate(now).createdAt(now).updatedAt(now).assignedAt(now)
                .startedAt(now).completedAt(now).suspendedAt(now)
                .accumulatedUnclaimedSeconds(99L).lastReturnedToPoolAt(now)
                .labels(List.of(new WorkItemLabel("x", LabelPersistence.MANUAL, "y")))
                .types(Set.of("t")).confidenceScore(0.9).callerRef("cr")
                .parentId(parentId).scope("s").templateId(templateId)
                .templateVersion(2L).permittedOutcomes("po").excludedUsers("eu")
                .outcome("oc").inputDataSchema("ids").outputDataSchema("ods")
                .payloadTypeName("ptn").resolutionTypeName("rtn")
                .candidateScores("cs").routingExperiences("re")
                .build();

        WorkItem roundTripped = original.toBuilder().build();
        assertEquals(original, roundTripped);
    }

    @Test
    void defaultsForCollections() {
        WorkItem item = WorkItem.builder().build();
        assertNotNull(item.labels());
        assertTrue(item.labels().isEmpty());
        assertNotNull(item.types());
        assertTrue(item.types().isEmpty());
    }
}
```

- [ ] **Step 7: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemBuilderTest -pl api`
Expected: FAIL — WorkItem class does not exist

- [ ] **Step 8: Implement WorkItem record with builder**

Create `api/src/main/java/io/casehub/work/api/WorkItem.java` — immutable record with all 46 fields, static inner `Builder` class, `builder()` factory, and `toBuilder()` instance method.

The record has these fields (in order): `id` (UUID), `tenancyId` (String), `title` (String), `description` (String), `formKey` (String), `status` (WorkItemStatus), `priority` (WorkItemPriority), `assigneeId` (String), `owner` (String), `candidateGroups` (String), `candidateUsers` (String), `requiredCapabilities` (String), `createdBy` (String), `delegationChain` (String), `delegationDeclineTarget` (DeclineTarget), `priorStatus` (WorkItemStatus), `payload` (String), `resolution` (String), `claimDeadline` (Instant), `expiresAt` (Instant), `followUpDate` (Instant), `createdAt` (Instant), `updatedAt` (Instant), `assignedAt` (Instant), `startedAt` (Instant), `completedAt` (Instant), `suspendedAt` (Instant), `accumulatedUnclaimedSeconds` (long), `lastReturnedToPoolAt` (Instant), `labels` (List\<WorkItemLabel\>), `types` (Set\<String\>), `confidenceScore` (Double), `callerRef` (String), `parentId` (UUID), `scope` (String), `templateId` (UUID), `templateVersion` (Long), `permittedOutcomes` (String), `excludedUsers` (String), `outcome` (String), `inputDataSchema` (String), `outputDataSchema` (String), `payloadTypeName` (String), `resolutionTypeName` (String), `candidateScores` (String), `routingExperiences` (String).

The canonical constructor normalizes `labels` and `types` to unmodifiable collections (empty list/set if null). The `Builder` has a setter for each field; `labels` and `types` default to empty. `toBuilder()` returns a new `Builder` pre-populated from all fields.

- [ ] **Step 9: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemBuilderTest -pl api`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/work/api/WorkItem.java api/src/main/java/io/casehub/work/api/WorkItemLabel.java api/src/main/java/io/casehub/work/api/LabelPatternMatcher.java api/src/test/java/io/casehub/work/api/WorkItemBuilderTest.java api/src/test/java/io/casehub/work/api/LabelPatternMatcherTest.java
git commit -m "feat(#337): add WorkItem record, WorkItemLabel, LabelPatternMatcher to api/"
```

---

### Task 3: Move WorkItemQuery, WorkItemRootView, WorkItemSummaryBuilder to api/

**Files:**
- Move: `runtime/.../repository/WorkItemQuery.java` → `api/.../api/WorkItemQuery.java` (use `ide_move_file`)
- Move: `runtime/.../model/WorkItemRootView.java` → `api/.../api/WorkItemRootView.java` (use `ide_move_file`)
- Move: `runtime/.../service/WorkItemSummaryBuilder.java` → `api/.../api/WorkItemSummaryBuilder.java` (use `ide_move_file`)
- Modify: `api/.../api/WorkItemSummaryBuilder.java` — field access changes
- Modify: `api/.../api/WorkItemRootView.java` — import change for WorkItem

**Interfaces:**
- Consumes: `WorkItemEntity` record (Task 2), `WorkItemStatus`, `WorkItemPriority`, `WorkItemSummary` (all in api/)
- Produces: `WorkItemQuery` (unchanged API), `WorkItemRootView` (unchanged API), `WorkItemSummaryBuilder.build()` (unchanged API)

- [ ] **Step 1: Move WorkItemQuery using ide_move_file**

Move `../api/src/main/java/io/casehub/work/api/WorkItemQuery.java` to `api/src/main/java/io/casehub/work/api/WorkItemQuery.java`. IntelliJ updates all import references across the project automatically.

- [ ] **Step 2: Move WorkItemRootView using ide_move_file**

Move `../api/src/main/java/io/casehub/work/api/WorkItemRootView.java` to `api/src/main/java/io/casehub/work/api/WorkItemRootView.java`. The record's `WorkItemEntity` parameter now resolves to the api/ record (same name, new package). Verify the import is correct after move.

- [ ] **Step 3: Move WorkItemSummaryBuilder using ide_move_file**

Move `../api/src/main/java/io/casehub/work/api/WorkItemSummaryBuilder.java` to `api/src/main/java/io/casehub/work/api/WorkItemSummaryBuilder.java`.

- [ ] **Step 4: Update WorkItemSummaryBuilder field access**

The `build()` method accesses entity public fields. After move, `WorkItemEntity` is a record — change all field references to accessor methods:

| Before (entity field) | After (record accessor) |
|---|---|
| `wi.status` | `wi.status()` |
| `wi.priority` | `wi.priority()` |
| `wi.expiresAt` | `wi.expiresAt()` |
| `wi.claimDeadline` | `wi.claimDeadline()` |
| `wi.createdAt` | `wi.createdAt()` |

Use `ide_replace_member` to update the `build()` method body.

- [ ] **Step 5: Build api/ module to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "refactor(#337): move WorkItemQuery, WorkItemRootView, WorkItemSummaryBuilder to api/

Refs #337"
```

---

### Task 4: Move WorkItemStore + CrossTenantWorkItemStore to api/spi/

**Files:**
- Move: `runtime/.../repository/WorkItemStore.java` → `api/.../api/spi/WorkItemStore.java` (use `ide_move_file`)
- Move: `runtime/.../repository/CrossTenantWorkItemStore.java` → `api/.../api/spi/CrossTenantWorkItemStore.java` (use `ide_move_file`)
- Modify: `api/.../api/spi/WorkItemStore.java` — update default method field access

**Interfaces:**
- Consumes: `WorkItemEntity` record, `WorkItemQuery`, `WorkItemRootView`, `WorkItemSummaryBuilder` (all in api/)
- Produces: `WorkItemStore` interface (same contract, new package), `CrossTenantWorkItemStore` interface

- [ ] **Step 1: Move WorkItemStore using ide_move_file**

Move to `api/src/main/java/io/casehub/work/api/spi/WorkItemStore.java`. IntelliJ updates all 81+ import sites.

- [ ] **Step 2: Update WorkItemStore default method bodies**

Six default methods access entity public fields — change to record accessors:

- `findByCallerRef`: `wi.callerRef` → `wi.callerRef()`, `wi.createdAt` → `wi.createdAt()`
- `findActiveByCallerRef`: same + `wi.status` → `wi.status()`
- `findByParentIdExcludingStatuses`: `wi.parentId` → `wi.parentId()`, `wi.status` → `wi.status()`
- `findByParentIdWithStatuses`: same
- `findByParentId`: `wi.parentId` → `wi.parentId()`

Use `ide_replace_member` for each default method.

- [ ] **Step 3: Move CrossTenantWorkItemStore using ide_move_file**

Move to `api/src/main/java/io/casehub/work/api/spi/CrossTenantWorkItemStore.java`. Return type `List<WorkItem>` now resolves to the api/ record.

- [ ] **Step 4: Build api/ to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "refactor(#337): move WorkItemStore + CrossTenantWorkItemStore to api/spi/

Refs #337"
```

---

### Task 6: Create WorkItemEntityMapper

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemEntityMapper.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/WorkItemEntityMapperTest.java`

**Interfaces:**
- Consumes: `WorkItemEntity` record (api/), `WorkItemLabelEntity` record (api/), `WorkItemEntity` (runtime/, renamed in Task 2)
- Produces: `WorkItemEntityMapper.toDomain(WorkItemEntity)` → `WorkItemEntity`, `WorkItemEntityMapper.toEntity(WorkItem)` → `WorkItemEntity`, `WorkItemEntityMapper.updateEntity(WorkItemEntity, WorkItem)` → void

- [ ] **Step 1: Write failing test for WorkItemEntityMapper**

```java
package io.casehub.work.runtime.repository;

import io.casehub.work.api.*;
import io.casehub.work.runtime.model.WorkItemEntity;
import io.casehub.work.runtime.model.WorkItemLabelEntity;
import io.casehub.work.runtime.model.WorkItemType;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class WorkItemEntityMapperTest {

    @Test
    void toDomainMapsAllFields() {
        WorkItemEntity entity = new WorkItemEntity();
        entity.id = UUID.randomUUID();
        entity.tenancyId = "t1";
        entity.title = "Test";
        entity.status = WorkItemStatus.PENDING;
        entity.priority = WorkItemPriority.HIGH;
        entity.labels.add(new WorkItemLabelEntity("legal", LabelPersistence.MANUAL, "alice"));
        entity.types.add(new WorkItemType("compliance"));
        entity.accumulatedUnclaimedSeconds = 120L;

        WorkItem domain = WorkItemEntityMapper.toDomain(entity);

        assertEquals(entity.id, domain.id());
        assertEquals("t1", domain.tenancyId());
        assertEquals("Test", domain.title());
        assertEquals(WorkItemStatus.PENDING, domain.status());
        assertEquals(WorkItemPriority.HIGH, domain.priority());
        assertEquals(1, domain.labels().size());
        assertEquals("legal", domain.labels().get(0).path());
        assertEquals(Set.of("compliance"), domain.types());
        assertEquals(120L, domain.accumulatedUnclaimedSeconds());
    }

    @Test
    void toEntityMapsAllFields() {
        WorkItem domain = WorkItem.builder()
                .id(UUID.randomUUID())
                .tenancyId("t1")
                .title("Test")
                .status(WorkItemStatus.ASSIGNED)
                .priority(WorkItemPriority.LOW)
                .labels(List.of(new WorkItemLabel("finance", LabelPersistence.INFERRED, "rule-1")))
                .types(Set.of("audit"))
                .accumulatedUnclaimedSeconds(60L)
                .build();

        WorkItemEntity entity = WorkItemEntityMapper.toEntity(domain);

        assertEquals(domain.id(), entity.id);
        assertEquals("t1", entity.tenancyId);
        assertEquals("Test", entity.title);
        assertEquals(WorkItemStatus.ASSIGNED, entity.status);
        assertEquals(1, entity.labels.size());
        assertEquals("finance", entity.labels.get(0).path);
        assertEquals(1, entity.types.size());
        assertEquals(60L, entity.accumulatedUnclaimedSeconds);
    }

    @Test
    void updateEntityPreservesVersion() {
        WorkItemEntity entity = new WorkItemEntity();
        entity.id = UUID.randomUUID();
        entity.version = 5L;
        entity.tenancyId = "t1";
        entity.title = "Old";
        entity.status = WorkItemStatus.PENDING;
        entity.followUpDate = Instant.parse("2026-01-01T00:00:00Z");

        WorkItem domain = WorkItem.builder()
                .id(entity.id)
                .tenancyId("t1")
                .title("New")
                .status(WorkItemStatus.ASSIGNED)
                .assigneeId("bob")
                .build();

        WorkItemEntityMapper.updateEntity(entity, domain);

        assertEquals(5L, entity.version);
        assertEquals("New", entity.title);
        assertEquals(WorkItemStatus.ASSIGNED, entity.status);
        assertEquals("bob", entity.assigneeId);
    }

    @Test
    void updateEntityOverwritesPopulatedFieldsWithNull() {
        WorkItemEntity entity = new WorkItemEntity();
        entity.id = UUID.randomUUID();
        entity.version = 3L;
        entity.tenancyId = "t1";
        entity.title = "Title";
        entity.followUpDate = Instant.parse("2026-06-01T00:00:00Z");
        entity.assigneeId = "alice";
        entity.scope = "org/team";

        WorkItem domain = WorkItem.builder()
                .id(entity.id)
                .tenancyId("t1")
                .title("Title")
                .build();

        WorkItemEntityMapper.updateEntity(entity, domain);

        assertNull(entity.followUpDate);
        assertNull(entity.assigneeId);
        assertNull(entity.scope);
        assertEquals(3L, entity.version);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemEntityMapperTest -pl runtime`
Expected: FAIL — WorkItemEntityMapper does not exist

- [ ] **Step 3: Implement WorkItemEntityMapper**

Create `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemEntityMapper.java`. Static utility class following `ProgressInstanceMapper` pattern.

`toDomain()`: reads all entity fields, converts `WorkItemLabelEntity` → `WorkItemLabelEntity` records, converts `Set<WorkItemType>` → `Set<String>` (extracting `.path`), builds immutable `WorkItemEntity` via builder.

`toEntity()`: creates new `WorkItemEntity`, copies all domain record fields to entity fields, converts `WorkItemLabelEntity` records → `WorkItemLabelEntity` instances, converts `Set<String>` → `Set<WorkItemType>`.

`updateEntity()`: copies ALL domain fields onto existing entity (preserving only `version` and `id`). Null domain fields overwrite entity fields — no conditional null-skipping. Replaces labels and types collections.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemEntityMapperTest -pl runtime`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#337): create WorkItemEntityMapper — entity↔domain bidirectional mapping

Refs #337"
```

---

### Task 7: Update JPA store implementations

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaCrossTenantWorkItemStore.java`

**Interfaces:**
- Consumes: `WorkItemEntityMapper` (Task 6), `WorkItemEntity` record (api/), `WorkItemEntity` (runtime/)
- Produces: JPA stores implementing api/ SPI interfaces with entity↔domain mapping

- [ ] **Step 1: Update JpaWorkItemStore**

The store implements `WorkItemStore` (now in api/spi/). Methods that return `WorkItemEntity` (the record) must map from entity:

- `put(WorkItem)`: load existing entity by ID (for OCC version), call `WorkItemEntityMapper.updateEntity()`, persist, return `WorkItemEntityMapper.toDomain(entity)`. For new items (no existing entity), use `WorkItemEntityMapper.toEntity()`.
- `get(UUID)`: find entity, return `WorkItemEntityMapper.toDomain(entity)`
- `scan(WorkItemQuery)`: query entities, map each via `WorkItemEntityMapper.toDomain()`
- `scanAll()`: same mapping pattern
- `scanRoots()`: constructs `WorkItemRootView` from JPA query results — must call `WorkItemEntityMapper.toDomain(entity)` to get the api/ `WorkItemEntity` record, then wrap in `WorkItemRootView(domain, childCount, completedCount, requiredCount, groupStatus)`

Use `ide_replace_member` for each method that needs updating.

- [ ] **Step 2: Update JpaCrossTenantWorkItemStore**

`findActiveWithDeadlines()` returns `List<WorkItem>` (the record). Map each entity result via `WorkItemEntityMapper.toDomain()`.

- [ ] **Step 3: Run runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS (all existing tests)

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "refactor(#337): update JPA stores to use WorkItemEntityMapper for domain↔entity conversion

Refs #337"
```

---

### Task 8: Rewrite InMemoryWorkItemStore — dependency shift to api/

**Files:**
- Modify: `persistence-memory/pom.xml` — change dependency from runtime to api
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemStore.java`
- Modify: tests in `persistence-memory/src/test/`

**Interfaces:**
- Consumes: `WorkItemEntity` record, `WorkItemStore`, `WorkItemQuery`, `LabelPatternMatcher` (all in api/)
- Produces: `InMemoryWorkItemStore` implementing api/ `WorkItemStore`, storing `WorkItemEntity` records directly

- [ ] **Step 1: Update pom.xml dependency**

Change `casehub-work` (runtime) dependency to `casehub-work-api`. This is the key deliverable — persistence-memory depends only on api/.

- [ ] **Step 2: Rewrite InMemoryWorkItemStore**

Major changes:
- Field access: all `wi.field` → `wi.field()` (record accessors)
- Type matching: `wi.types` was `Set<WorkItemType>` with `.path` access → now `Set<String>` directly. Continue using `Path.parse()` + `isAncestorOf()` for hierarchy matching (api/ depends on casehub-platform-api, `Path` is available)
- Label matching: `LabelVocabularyService.matchesPattern()` → `LabelPatternMatcher.matchesPattern()` (api/ utility)
- Storage: stores `WorkItemEntity` records directly in `ConcurrentHashMap` — no entity, no mapper needed
- `put()`: uses `toBuilder()` for id/tenancyId assignment (immutable record cannot be mutated in place):
  ```java
  WorkItem stored = workItem;
  if (stored.id() == null) {
      stored = stored.toBuilder().id(UUID.randomUUID()).build();
  }
  if (stored.tenancyId() == null) {
      stored = stored.toBuilder().tenancyId(currentPrincipal.tenancyId()).build();
  }
  store.put(stored.id(), stored);
  return stored;
  ```
  Note: `put()` may return a different object than the one passed in. Callers must use the return value, not the original reference.

- [ ] **Step 3: Update tests**

Adjust test imports. Tests should continue to pass — the store's external contract is unchanged, only internal implementation differs.

- [ ] **Step 4: Build persistence-memory to verify api-only dependency**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: PASS — compiles and tests pass with no runtime dependency

Verify no runtime dependency:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:tree -pl persistence-memory | grep casehub-work
```
Expected: only `casehub-work-api` appears, not `casehub-work` (runtime)

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "refactor(#337): shift persistence-memory dependency from runtime to api/

InMemoryWorkItemStore now stores WorkItem records directly. Label
pattern matching uses LabelPatternMatcher from api/. Type matching
uses Set<String> directly. No JPA dependency.

Refs #337"
```

---

### Task 9: Update MongoDB store implementations

**Files:**
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java`
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemStore.java`
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemEntityMapper.java` (if needed)

**Interfaces:**
- Consumes: `WorkItemEntity` record (api/), `WorkItemStore` (api/spi/)
- Produces: MongoDB stores implementing api/ SPI with BSON↔domain mapping, OCC via version-checked replaceOne

- [ ] **Step 1: Update MongoWorkItemStore**

The store implements `WorkItemStore` (now in api/spi/). MongoDB documents are the internal representation — the store converts between BSON documents and `WorkItemEntity` records.

OCC mechanics: load document with version field, apply domain record fields, version-checked `replaceOne` (match on `_id` + `version`). The version lives in the BSON document, not the domain record.

- [ ] **Step 2: Update MongoCrossTenantWorkItemStore**

Same pattern — `findActiveWithDeadlines()` returns `List<WorkItem>` records.

- [ ] **Step 3: Run MongoDB tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "refactor(#337): update MongoDB stores for WorkItem record + OCC

Refs #337"
```

---

### Task 10: Import sweep across remaining modules

**Files:**
- Modify: imports in `rest/`, `queues/`, `queues-dashboard/`, `ai/`, `ledger/`, `issue-tracker/`, `engine-adapter/`, `qhorus/`, `flow/`, `examples/`, `flow-examples/`, `queues-examples/`, `integration-tests/`, `integration-tests-memory/`, `src/main/java/io/casehub/actorstate/`

**Interfaces:**
- Consumes: all types now in api/ (WorkItem, WorkItemStore, WorkItemQuery, etc.)
- Produces: all modules compile against api/ types

- [ ] **Step 1: Check for remaining compilation errors**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl rest,queues,queues-dashboard,ai,ledger,issue-tracker,engine-adapter,qhorus,flow,examples,flow-examples,queues-examples`
Review errors — should be import resolution failures only.

- [ ] **Step 2: Fix imports in each module**

The `ide_move_file` and `ide_refactor_rename` from Tasks 3-5 should have updated most imports. For any remaining failures:
- `io.casehub.work.runtime.repository.WorkItemStore` → `io.casehub.work.api.spi.WorkItemStore`
- `io.casehub.work.runtime.repository.WorkItemQuery` → `io.casehub.work.api.WorkItemQuery`
- `io.casehub.work.runtime.model.WorkItem` → `io.casehub.work.api.WorkItem`
- `io.casehub.work.runtime.model.WorkItemRootView` → `io.casehub.work.api.WorkItemRootView`
- `io.casehub.work.runtime.service.WorkItemSummaryBuilder` → `io.casehub.work.api.WorkItemSummaryBuilder`

Modules that access entity fields directly (e.g., example scenarios that create `WorkItemEntity` via `new WorkItem()`) must switch to `WorkItem.builder()...build()`.

- [ ] **Step 3: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 4: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests`
Expected: PASS

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests-memory`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "refactor(#337): complete import sweep — all modules compile against api/ types

Refs #337, Closes #337"
```

---

### Task 11: Progress API docs (#333)

**Files:**
- Modify: `docs/api-reference.md`

**Interfaces:**
- Consumes: `ProgressResource.java` (source of truth for endpoints)
- Produces: complete progress REST API section in api-reference.md

- [ ] **Step 1: Add progress module to the module table**

Add row to the module table at the top of `docs/api-reference.md`:

```markdown
| `casehub-work-progress` | `/progress` | Structured progress tracking with state machines, step sequences, rollback, and tree hierarchies |
```

- [ ] **Step 2: Write the Progress section**

Add `## Progress` section after the existing `## SSE Events` section (before `## AsyncAPI`). Document all 18 endpoints following the existing format with HTTP method, path, description, parameter tables, response codes, and curl examples. Group as:

- CRUD (create, getById, findByScope, getTree, attachChild)
- State transitions (updateState, complete, fail, reactivate)
- Rollback (rollback, snapshots)
- Step convenience (5 endpoints)
- Events and streaming (events, stream)

Include request DTOs (`CreateProgressRequest`, `UpdateStateRequest`, `UpdateStepDataRequest`), response types (`ProgressInstance`, `ProgressSnapshot`, `ProgressUpdatedEvent`, `TreeResponse`), and new #329 fields (`rollbackPolicy`, `visualisationMode`).

- [ ] **Step 3: Commit**

```bash
git add docs/api-reference.md
git commit -m "docs(#333): add progress REST API to api-reference.md

Documents all 18 ProgressResource endpoints: CRUD, state transitions,
rollback, step convenience, events/SSE. Includes new #329 fields
(rollbackPolicy, visualisationMode) and rollback/snapshot endpoints.

Closes #333"
```

---

### Task 12: Final verification

- [ ] **Step 1: Full clean build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 2: Verify persistence-memory is JPA-free**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:tree -pl persistence-memory | grep casehub-work
```
Expected: only `casehub-work-api`, no `casehub-work` runtime

- [ ] **Step 3: Verify no runtime model imports remain in non-runtime modules**

Use `ide_search_text` to search for `io.casehub.work.runtime.model.WorkItem` across the project (excluding `runtime/` itself). Should return zero hits.

Use `ide_search_text` to search for `io.casehub.work.runtime.repository.WorkItemStore` across the project. Should return zero hits.

- [ ] **Step 4: Verify #340 is closed**

```bash
gh issue view 340 --repo casehubio/work --json state --jq '.state'
```
Expected: `CLOSED`
