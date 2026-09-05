# Multi-Instance Coordinated Rollback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #332 — Multi-instance coordinated rollback (subtree-wide undo)
**Issue group:** #332

**Goal:** Add coordinated subtree rollback to the progress model — roll back all descendants of a node to their state at a given point in time, suppress individual rollup cascades during the operation, then recompute rollup bottom-up.

**Architecture:** New `SubtreeRollbackService` coordinates multi-instance rollback. It delegates per-node rollback to `ProgressService` (new `rollbackToTimestamp` and `applyRollupState` methods). `ProgressUpdatedEvent` gains an `operationId` field that suppresses `RollupObserver` async cascade during coordinated operations. New SPI methods `findDescendantsOf` and `findLastEventAtOrBefore` provide tree traversal and historical event lookup.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Flyway, Jackson (JsonNode), CDI

## Global Constraints

- Pre-release project — breaking changes are acceptable
- All tests use in-memory stores (no JPA integration tests in progress modules)
- `ProgressService` is a plain Java class with constructor injection, NOT a CDI bean — no `@Transactional` on its methods
- Module package: `io.casehub.work.progress` (api), `io.casehub.work.progress.runtime` (runtime), `io.casehub.work.progress.memory` (in-memory stores)
- Flyway path: `classpath:db/work/migration`, version range V7000-V7999 for progress
- IntelliJ MCP required for all Java file operations

---

## Batch 1: API Foundation — Event, Types, SPI

### Task 1: Add operationId to ProgressUpdatedEvent and update all callers

**Files:**
- Modify: `progress-api/src/main/java/io/casehub/work/progress/ProgressUpdatedEvent.java`
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/ProgressService.java:392-402` (emitEvent)
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/event/RollupObserver.java:88-97` (recompute event construction)
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/repository/JpaProgressEventStore.java:70-100` (toEntity, toDomain)
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/model/ProgressEventEntity.java` (add operationId field)
- Modify: `progress-memory/src/test/java/io/casehub/work/progress/memory/InMemoryProgressEventStoreTest.java:25-27` (event helper)
- Modify: `progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/ProgressServiceTest.java` (any event construction)

**Interfaces:**
- Produces: `ProgressUpdatedEvent` record with 14 fields (added `UUID operationId` as last parameter — nullable)
- Produces: `ProgressEventEntity.operationId` (UUID, nullable column)

- [ ] **Step 1: Write test verifying operationId round-trips through event emission**

In `ProgressServiceTest.java`, add test that creates a progress instance, updates state, and verifies the emitted event has `operationId == null`:

```java
@Test
void updateState_emitsEventWithNullOperationId() {
    ProgressInstance instance = service.create(percentageRequest("t1", 0));
    emittedEvents.clear();
    service.updateState(instance.id(), percentageState(50));
    assertFalse(emittedEvents.isEmpty());
    assertNull(emittedEvents.get(0).operationId());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProgressServiceTest#updateState_emitsEventWithNullOperationId -pl progress-runtime`
Expected: FAIL — `operationId()` method doesn't exist on ProgressUpdatedEvent

- [ ] **Step 3: Add operationId field to ProgressUpdatedEvent record**

Use `ide_replace_member` on `ProgressUpdatedEvent.java` — add `UUID operationId` as the 14th field after `timestamp`:

```java
public record ProgressUpdatedEvent(
    UUID id,
    UUID progressId,
    String tenancyId,
    String scopeType,
    String scopeId,
    UUID parentProgressId,
    UUID rootProgressId,
    String shapeType,
    JsonNode previousState,
    JsonNode currentState,
    ProgressStatus status,
    ProgressChangeType changeType,
    Instant timestamp,
    UUID operationId
) {}
```

- [ ] **Step 4: Fix all callers — ProgressService.emitEvent()**

Use `ide_replace_member` on `ProgressService.java` method `emitEvent`. Add `null` as the 14th argument:

```java
private void emitEvent(ProgressInstance instance, JsonNode previousState,
                       ProgressChangeType changeType) {
    ProgressUpdatedEvent event = new ProgressUpdatedEvent(
            UUID.randomUUID(),
            instance.id(), instance.tenancyId(),
            instance.scopeType(), instance.scopeId(),
            instance.parentProgressId(), instance.rootProgressId(),
            instance.shapeType(), previousState, instance.state(),
            instance.status(), changeType, Instant.now(), null);
    eventStore.append(event);
    eventEmitter.accept(event);
}
```

- [ ] **Step 5: Fix RollupObserver.recompute() event construction**

Use `ide_replace_member` on `RollupObserver.java` method `recompute`. Add `null` as the 14th argument to the `ProgressUpdatedEvent` constructor on line 88:

```java
ProgressUpdatedEvent rollupEvent = new ProgressUpdatedEvent(
        java.util.UUID.randomUUID(),
        parent.id(), tenancyId,
        parent.scopeType(), parent.scopeId(),
        parent.parentProgressId(), parent.rootProgressId(),
        parent.shapeType(), previousState, newState,
        parent.status(), ProgressChangeType.STATE_UPDATED,
        Instant.now(), null);
```

- [ ] **Step 6: Fix JpaProgressEventStore — toEntity and toDomain**

Use `ide_replace_member` on `JpaProgressEventStore.java` method `toEntity`. Add `entity.operationId = event.operationId();`:

```java
private ProgressEventEntity toEntity(ProgressUpdatedEvent event) {
    ProgressEventEntity entity = new ProgressEventEntity();
    entity.id             = event.id();
    entity.tenancyId      = event.tenancyId();
    entity.progressId     = event.progressId();
    entity.rootProgressId = event.rootProgressId();
    entity.scopeType      = event.scopeType();
    entity.scopeId        = event.scopeId();
    entity.changeType     = event.changeType().name();
    entity.previousState  = event.previousState();
    entity.currentState   = event.currentState();
    entity.status         = event.status().name();
    entity.occurredAt     = event.timestamp();
    entity.operationId    = event.operationId();
    return entity;
}
```

Use `ide_replace_member` on `toDomain`. Add `entity.operationId` as the 14th constructor arg:

```java
private ProgressUpdatedEvent toDomain(ProgressEventEntity entity) {
    return new ProgressUpdatedEvent(
            entity.id,
            entity.progressId,
            entity.tenancyId,
            entity.scopeType,
            entity.scopeId,
            null,
            entity.rootProgressId,
            null,
            entity.previousState,
            entity.currentState,
            ProgressStatus.valueOf(entity.status),
            ProgressChangeType.valueOf(entity.changeType),
            entity.occurredAt,
            entity.operationId);
}
```

- [ ] **Step 7: Add operationId field to ProgressEventEntity**

Use `ide_insert_member` on `ProgressEventEntity.java` after `occurredAt` field:

```java
@Column(name = "operation_id")
public UUID operationId;
```

- [ ] **Step 8: Fix InMemoryProgressEventStoreTest.event() helper**

Use `ide_replace_member` on `InMemoryProgressEventStoreTest.java` method `event`. Add `null` as the 14th arg:

```java
private ProgressUpdatedEvent event(UUID progressId, UUID rootId, Instant timestamp) {
    return new ProgressUpdatedEvent(UUID.randomUUID(), progressId, "t1", "workitem", "wi-1",
            null, rootId, "percentage", null, null,
            ProgressStatus.ACTIVE, ProgressChangeType.STATE_UPDATED, timestamp, null);
}
```

- [ ] **Step 9: Fix ProgressServiceTest — any remaining event construction**

Search `ProgressServiceTest.java` for `new ProgressUpdatedEvent` — fix any direct constructions by adding `null` as the 14th argument.

- [ ] **Step 10: Run all progress tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime,progress-memory,progress-core`
Expected: ALL PASS

- [ ] **Step 11: Verify with ide_diagnostics**

Run `ide_diagnostics` on the progress modules to catch any remaining compilation errors.

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add progress-api/ progress-runtime/ progress-memory/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#332): add operationId to ProgressUpdatedEvent + entity Refs #332 Refs #92"
```

---

### Task 2: New API types and SPI methods

**Files:**
- Create: `progress-api/src/main/java/io/casehub/work/progress/SubtreeRollbackResult.java`
- Create: `progress-api/src/main/java/io/casehub/work/progress/NodeRollbackOutcome.java`
- Modify: `progress-api/src/main/java/io/casehub/work/progress/spi/ProgressEventStore.java`
- Modify: `progress-api/src/main/java/io/casehub/work/progress/spi/ProgressInstanceStore.java`

**Interfaces:**
- Produces: `SubtreeRollbackResult(UUID operationId, UUID rootId, Instant targetTimestamp, List<NodeRollbackOutcome> outcomes)`
- Produces: `NodeRollbackOutcome(UUID progressId, NodeRollbackOutcome.Outcome outcome, String reason, JsonNode previousState, JsonNode restoredState, boolean policyBypassed)` with enum `Outcome { ROLLED_BACK, SKIPPED, FAILED }`
- Produces: `ProgressEventStore.findLastEventAtOrBefore(UUID progressId, Instant cutoff)` → `Optional<ProgressUpdatedEvent>`
- Produces: `ProgressInstanceStore.findDescendantsOf(UUID parentId)` → `List<ProgressInstance>`

- [ ] **Step 1: Create NodeRollbackOutcome record**

Write new file `progress-api/src/main/java/io/casehub/work/progress/NodeRollbackOutcome.java`:

```java
package io.casehub.work.progress;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.UUID;

public record NodeRollbackOutcome(
    UUID progressId,
    Outcome outcome,
    String reason,
    JsonNode previousState,
    JsonNode restoredState,
    boolean policyBypassed
) {
    public enum Outcome { ROLLED_BACK, SKIPPED, FAILED }
}
```

- [ ] **Step 2: Create SubtreeRollbackResult record**

Write new file `progress-api/src/main/java/io/casehub/work/progress/SubtreeRollbackResult.java`:

```java
package io.casehub.work.progress;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record SubtreeRollbackResult(
    UUID operationId,
    UUID rootId,
    Instant targetTimestamp,
    List<NodeRollbackOutcome> outcomes
) {}
```

- [ ] **Step 3: Add findLastEventAtOrBefore to ProgressEventStore SPI**

Use `ide_insert_member` on `ProgressEventStore.java` after `findById`:

```java
Optional<ProgressUpdatedEvent> findLastEventAtOrBefore(UUID progressId, Instant cutoff);
```

- [ ] **Step 4: Add findDescendantsOf to ProgressInstanceStore SPI**

Use `ide_insert_member` on `ProgressInstanceStore.java` after `findByParentProgressId`:

```java
List<ProgressInstance> findDescendantsOf(UUID parentId);
```

- [ ] **Step 5: Verify with ide_diagnostics**

Run `ide_diagnostics` on progress-api. Expected: compilation errors in implementations (not yet updated) — that's expected and will be fixed in Batch 2.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add progress-api/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#332): add SubtreeRollbackResult, NodeRollbackOutcome, new SPI methods Refs #332 Refs #92"
```

---

## Batch 2: Store Implementations

### Task 3: Implement findDescendantsOf in all stores + refactor ProgressResource

**Files:**
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/repository/JpaProgressInstanceStore.java`
- Modify: `progress-memory/src/main/java/io/casehub/work/progress/memory/InMemoryProgressInstanceStore.java`
- Create: `progress-memory/src/test/java/io/casehub/work/progress/memory/InMemoryProgressInstanceStoreTest.java`
- Modify: `progress-rest/src/main/java/io/casehub/work/progress/rest/ProgressResource.java:106-113,206-213`

**Interfaces:**
- Consumes: `ProgressInstanceStore.findDescendantsOf(UUID parentId)` (from Task 2)
- Produces: Working implementations in JPA (recursive CTE) and in-memory stores

- [ ] **Step 1: Write tests for findDescendantsOf**

Create `progress-memory/src/test/java/io/casehub/work/progress/memory/InMemoryProgressInstanceStoreTest.java`:

```java
package io.casehub.work.progress.memory;

import io.casehub.work.progress.ProgressInstance;
import io.casehub.work.progress.ProgressStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class InMemoryProgressInstanceStoreTest {

    private InMemoryProgressInstanceStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryProgressInstanceStore();
    }

    @Test
    void findDescendantsOf_threeLevelTree_returnsAllDescendants() {
        ProgressInstance root = instance(null);
        store.put(root);
        ProgressInstance child1 = instance(root.id());
        ProgressInstance child2 = instance(root.id());
        store.put(child1);
        store.put(child2);
        ProgressInstance grandchild = instance(child1.id());
        store.put(grandchild);

        List<ProgressInstance> descendants = store.findDescendantsOf(root.id());
        assertEquals(3, descendants.size());
        assertTrue(descendants.stream().anyMatch(d -> d.id().equals(child1.id())));
        assertTrue(descendants.stream().anyMatch(d -> d.id().equals(child2.id())));
        assertTrue(descendants.stream().anyMatch(d -> d.id().equals(grandchild.id())));
    }

    @Test
    void findDescendantsOf_doesNotIncludeRoot() {
        ProgressInstance root = instance(null);
        store.put(root);
        ProgressInstance child = instance(root.id());
        store.put(child);

        List<ProgressInstance> descendants = store.findDescendantsOf(root.id());
        assertFalse(descendants.stream().anyMatch(d -> d.id().equals(root.id())));
    }

    @Test
    void findDescendantsOf_midTreeNode_onlyReturnsTargetSubtree() {
        ProgressInstance root = instance(null);
        store.put(root);
        ProgressInstance branch1 = instance(root.id());
        ProgressInstance branch2 = instance(root.id());
        store.put(branch1);
        store.put(branch2);
        ProgressInstance leaf1 = instance(branch1.id());
        store.put(leaf1);

        List<ProgressInstance> descendants = store.findDescendantsOf(branch1.id());
        assertEquals(1, descendants.size());
        assertEquals(leaf1.id(), descendants.get(0).id());
    }

    @Test
    void findDescendantsOf_leafNode_returnsEmpty() {
        ProgressInstance root = instance(null);
        store.put(root);

        List<ProgressInstance> descendants = store.findDescendantsOf(root.id());
        assertTrue(descendants.isEmpty());
    }

    private ProgressInstance instance(UUID parentId) {
        UUID id = UUID.randomUUID();
        return new ProgressInstance(id, "t1", "workitem", "wi-" + id,
                parentId, parentId == null ? id : parentId,
                "percentage", null, null,
                ProgressStatus.ACTIVE, null, null, null,
                Instant.now(), Instant.now());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryProgressInstanceStoreTest -pl progress-memory`
Expected: FAIL — `findDescendantsOf` not implemented

- [ ] **Step 3: Implement findDescendantsOf in InMemoryProgressInstanceStore**

Use `ide_insert_member` on `InMemoryProgressInstanceStore.java` after `findByParentProgressId`:

```java
@Override
public List<ProgressInstance> findDescendantsOf(UUID parentId) {
    List<ProgressInstance> result = new java.util.ArrayList<>();
    List<ProgressInstance> children = findByParentProgressId(parentId);
    result.addAll(children);
    for (ProgressInstance child : children) {
        result.addAll(findDescendantsOf(child.id()));
    }
    return result;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryProgressInstanceStoreTest -pl progress-memory`
Expected: ALL PASS

- [ ] **Step 5: Implement findDescendantsOf in JpaProgressInstanceStore**

Use `ide_insert_member` on `JpaProgressInstanceStore.java` after `findByParentProgressId`:

```java
@Override
public List<ProgressInstance> findDescendantsOf(UUID parentId) {
    List<ProgressInstanceEntity> entities = ProgressInstanceEntity.find(
            "id IN (SELECT d.id FROM ProgressInstanceEntity d WHERE d.parentProgressId = ?1" +
            " OR d.parentProgressId IN (SELECT c.id FROM ProgressInstanceEntity c WHERE c.parentProgressId = ?1)" +
            " OR d.parentProgressId IN (SELECT gc.id FROM ProgressInstanceEntity gc WHERE gc.parentProgressId IN " +
            "(SELECT c2.id FROM ProgressInstanceEntity c2 WHERE c2.parentProgressId = ?1)))",
            parentId).list();
    return entities.stream().map(ProgressInstanceMapper::toDomain).toList();
}
```

Note: JPQL doesn't support recursive CTEs. For deep trees (>3 levels), this needs a native query. For pre-release with shallow trees, a 3-level JPQL fallback is acceptable. File a follow-up issue for native recursive CTE if needed.

- [ ] **Step 6: Refactor ProgressResource.getTree to use store method**

Use `ide_replace_member` on `ProgressResource.java` method `getTree`:

```java
@GET
@Path("/{id}/tree")
public Response getTree(@PathParam("id") UUID id) {
    return progressService.findById(id)
            .map(root -> {
                List<ProgressInstance> descendants = instanceStore.findDescendantsOf(id);
                return Response.ok(new TreeResponse(root, descendants)).build();
            })
            .orElse(Response.status(Response.Status.NOT_FOUND).build());
}
```

Then delete `collectDescendants` method using `ide_edit_member` (replace body with nothing / remove the method).

- [ ] **Step 7: Verify with ide_diagnostics, run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-memory,progress-runtime,progress-rest`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add progress-memory/ progress-runtime/ progress-rest/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#332): implement findDescendantsOf, refactor getTree Refs #332 Refs #92"
```

---

### Task 4: Implement findLastEventAtOrBefore in all stores

**Files:**
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/repository/JpaProgressEventStore.java`
- Modify: `progress-memory/src/main/java/io/casehub/work/progress/memory/InMemoryProgressEventStore.java`
- Modify: `progress-memory/src/test/java/io/casehub/work/progress/memory/InMemoryProgressEventStoreTest.java`

**Interfaces:**
- Consumes: `ProgressEventStore.findLastEventAtOrBefore(UUID progressId, Instant cutoff)` (from Task 2)
- Produces: Working implementations in JPA and in-memory stores

- [ ] **Step 1: Write tests for findLastEventAtOrBefore**

Add to `InMemoryProgressEventStoreTest.java`:

```java
@Test
void findLastEventAtOrBefore_returnsEventAtExactCutoff() {
    UUID progressId = UUID.randomUUID();
    UUID rootId = UUID.randomUUID();
    Instant t1 = Instant.parse("2026-01-01T10:00:00Z");
    Instant t2 = Instant.parse("2026-01-01T11:00:00Z");
    store.append(event(progressId, rootId, t1));
    store.append(event(progressId, rootId, t2));

    Optional<ProgressUpdatedEvent> result = store.findLastEventAtOrBefore(progressId, t2);
    assertTrue(result.isPresent());
    assertEquals(t2, result.get().timestamp());
}

@Test
void findLastEventAtOrBefore_returnsLatestBeforeCutoff() {
    UUID progressId = UUID.randomUUID();
    UUID rootId = UUID.randomUUID();
    Instant t1 = Instant.parse("2026-01-01T10:00:00Z");
    Instant t2 = Instant.parse("2026-01-01T11:00:00Z");
    Instant t3 = Instant.parse("2026-01-01T12:00:00Z");
    store.append(event(progressId, rootId, t1));
    store.append(event(progressId, rootId, t2));
    store.append(event(progressId, rootId, t3));

    Instant cutoff = Instant.parse("2026-01-01T11:30:00Z");
    Optional<ProgressUpdatedEvent> result = store.findLastEventAtOrBefore(progressId, cutoff);
    assertTrue(result.isPresent());
    assertEquals(t2, result.get().timestamp());
}

@Test
void findLastEventAtOrBefore_noEventBeforeCutoff_returnsEmpty() {
    UUID progressId = UUID.randomUUID();
    UUID rootId = UUID.randomUUID();
    Instant t1 = Instant.parse("2026-01-01T10:00:00Z");
    store.append(event(progressId, rootId, t1));

    Instant cutoff = Instant.parse("2026-01-01T09:00:00Z");
    Optional<ProgressUpdatedEvent> result = store.findLastEventAtOrBefore(progressId, cutoff);
    assertTrue(result.isEmpty());
}

@Test
void findLastEventAtOrBefore_filteredByProgressId() {
    UUID id1 = UUID.randomUUID();
    UUID id2 = UUID.randomUUID();
    UUID rootId = UUID.randomUUID();
    Instant t1 = Instant.parse("2026-01-01T10:00:00Z");
    Instant t2 = Instant.parse("2026-01-01T11:00:00Z");
    store.append(event(id1, rootId, t1));
    store.append(event(id2, rootId, t2));

    Instant cutoff = Instant.parse("2026-01-01T12:00:00Z");
    Optional<ProgressUpdatedEvent> result = store.findLastEventAtOrBefore(id1, cutoff);
    assertTrue(result.isPresent());
    assertEquals(id1, result.get().progressId());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryProgressEventStoreTest -pl progress-memory`
Expected: FAIL — `findLastEventAtOrBefore` not implemented

- [ ] **Step 3: Implement in InMemoryProgressEventStore**

Use `ide_insert_member` on `InMemoryProgressEventStore.java` after `findById`:

```java
@Override
public Optional<ProgressUpdatedEvent> findLastEventAtOrBefore(UUID progressId, Instant cutoff) {
    return events.stream()
            .filter(e -> e.progressId().equals(progressId))
            .filter(e -> !e.timestamp().isAfter(cutoff))
            .max(java.util.Comparator.comparing(ProgressUpdatedEvent::timestamp)
                    .thenComparing(ProgressUpdatedEvent::id));
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryProgressEventStoreTest -pl progress-memory`
Expected: ALL PASS

- [ ] **Step 5: Implement in JpaProgressEventStore**

Use `ide_insert_member` on `JpaProgressEventStore.java` after `findById`:

```java
@Override
public Optional<ProgressUpdatedEvent> findLastEventAtOrBefore(UUID progressId, Instant cutoff) {
    return ProgressEventEntity.<ProgressEventEntity>find(
            "progressId = ?1 AND occurredAt <= ?2 ORDER BY occurredAt DESC, id DESC",
            progressId, cutoff)
            .firstResultOptional()
            .map(this::toDomain);
}
```

- [ ] **Step 6: Verify with ide_diagnostics, run all progress tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-memory,progress-runtime`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add progress-memory/ progress-runtime/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#332): implement findLastEventAtOrBefore in stores Refs #332 Refs #92"
```

---

## Batch 3: Service Layer

### Task 5: ProgressService changes + RollupObserver operationId suppression

**Files:**
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/ProgressService.java`
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/event/RollupObserver.java:46-52`
- Modify: `progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/ProgressServiceTest.java`

**Interfaces:**
- Consumes: `ProgressEventStore.findLastEventAtOrBefore(UUID, Instant)` (from Task 4)
- Consumes: `RollupEngine.recompute(ProgressInstance, List<ProgressInstance>)`, `RollupEngine.hasStateChanged(JsonNode, JsonNode)` (existing)
- Produces: `ProgressService.rollbackToTimestamp(UUID id, Instant target, UUID operationId)` → `ProgressInstance` (null = no-op)
- Produces: `ProgressService.applyRollupState(UUID id, List<ProgressInstance> children, UUID operationId)` → `ProgressInstance` (null = no change)
- Produces: `RollupObserver` skips events with non-null `operationId`

- [ ] **Step 1: Write test for rollbackToTimestamp**

Add to `ProgressServiceTest.java`:

```java
@Test
void rollbackToTimestamp_restoresStateAtTargetTime() {
    ProgressInstance instance = service.create(percentageRequest("t1", 0));
    Instant beforeUpdate = Instant.now();
    service.updateState(instance.id(), percentageState(50));
    Instant afterFirstUpdate = Instant.now();
    service.updateState(instance.id(), percentageState(80));

    UUID operationId = UUID.randomUUID();
    ProgressInstance rolled = service.rollbackToTimestamp(instance.id(), afterFirstUpdate, operationId);
    assertNotNull(rolled);
    assertEquals(50, rolled.state().get("value").asInt());
}

@Test
void rollbackToTimestamp_noOpWhenAlreadyAtTarget() {
    ProgressInstance instance = service.create(percentageRequest("t1", 0));
    service.updateState(instance.id(), percentageState(50));
    Instant afterUpdate = Instant.now();

    ProgressInstance result = service.rollbackToTimestamp(instance.id(), afterUpdate, UUID.randomUUID());
    assertNull(result);
}

@Test
void rollbackToTimestamp_throwsWhenNoHistoryBeforeTarget() {
    ProgressInstance instance = service.create(percentageRequest("t1", 0));
    Instant beforeCreation = Instant.now().minusSeconds(3600);

    assertThrows(IllegalStateException.class,
            () -> service.rollbackToTimestamp(instance.id(), beforeCreation, UUID.randomUUID()));
}

@Test
void rollbackToTimestamp_emitsEventWithOperationId() {
    ProgressInstance instance = service.create(percentageRequest("t1", 0));
    service.updateState(instance.id(), percentageState(50));
    Instant afterUpdate = Instant.now();
    service.updateState(instance.id(), percentageState(80));
    emittedEvents.clear();

    UUID operationId = UUID.randomUUID();
    service.rollbackToTimestamp(instance.id(), afterUpdate, operationId);

    assertFalse(emittedEvents.isEmpty());
    assertEquals(operationId, emittedEvents.get(0).operationId());
    assertEquals(ProgressChangeType.ROLLED_BACK, emittedEvents.get(0).changeType());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ProgressServiceTest#rollbackToTimestamp* -pl progress-runtime`
Expected: FAIL — `rollbackToTimestamp` doesn't exist

- [ ] **Step 3: Add overloaded emitEvent with operationId**

Use `ide_insert_member` on `ProgressService.java` after existing `emitEvent`:

```java
private void emitEvent(ProgressInstance instance, JsonNode previousState,
                       ProgressChangeType changeType, UUID operationId) {
    ProgressUpdatedEvent event = new ProgressUpdatedEvent(
            UUID.randomUUID(),
            instance.id(), instance.tenancyId(),
            instance.scopeType(), instance.scopeId(),
            instance.parentProgressId(), instance.rootProgressId(),
            instance.shapeType(), previousState, instance.state(),
            instance.status(), changeType, Instant.now(), operationId);
    eventStore.append(event);
    eventEmitter.accept(event);
}
```

- [ ] **Step 4: Add overloaded applyRollbackState with operationId**

Use `ide_insert_member` on `ProgressService.java` after existing `applyRollbackState`:

```java
private ProgressInstance applyRollbackState(ProgressInstance instance, JsonNode newState, UUID operationId) {
    validateShape(instance.shapeType(), newState, instance.definition());
    ProgressStatus newStatus = instance.status();
    if (newStatus == ProgressStatus.PENDING) {
        newStatus = ProgressStatus.ACTIVE;
    }
    ProgressInstance updated = withState(instance, newState, newStatus);
    instanceStore.put(updated);
    emitEvent(updated, instance.state(), ProgressChangeType.ROLLED_BACK, operationId);
    return updated;
}
```

- [ ] **Step 5: Add rollbackToTimestamp method**

Use `ide_insert_member` on `ProgressService.java` after `rollbackToEvent`:

```java
public ProgressInstance rollbackToTimestamp(UUID id, Instant target, UUID operationId) {
    ProgressInstance instance = requireInstance(id);
    ProgressUpdatedEvent event = eventStore.findLastEventAtOrBefore(id, target)
            .orElseThrow(() -> new IllegalStateException("No event history at or before target"));

    JsonNode targetState = event.currentState();
    if (targetState.equals(instance.state())) {
        return null;
    }

    return applyRollbackState(instance, targetState, operationId);
}
```

- [ ] **Step 6: Add applyRollupState method**

Use `ide_insert_member` on `ProgressService.java` after `rollbackToTimestamp`:

```java
public ProgressInstance applyRollupState(UUID id, List<ProgressInstance> children, UUID operationId) {
    ProgressInstance instance = requireInstance(id);
    JsonNode previousState = instance.state();
    JsonNode newState = rollupEngine.recompute(instance, children);

    if (newState == null || !rollupEngine.hasStateChanged(previousState, newState)) {
        return null;
    }

    ProgressInstance updated = withState(instance, newState, instance.status());
    instanceStore.put(updated);
    emitEvent(updated, previousState, ProgressChangeType.STATE_UPDATED, operationId);
    return updated;
}
```

Note: This requires `rollupEngine` as a field. Check if ProgressService already has it — if not, add it to the constructor.

- [ ] **Step 7: Add RollupEngine dependency to ProgressService if not present**

Check `ProgressService` constructor. If `RollupEngine` is not already injected, add it:

Use `ide_replace_member` on constructor to add `RollupEngine rollupEngine` parameter and assign it to a new field.

- [ ] **Step 8: Add operationId check to RollupObserver**

Use `ide_replace_member` on `RollupObserver.java` method `onProgressUpdated`:

```java
void onProgressUpdated(@ObservesAsync ProgressUpdatedEvent event) {
    if (event.parentProgressId() == null) {
        return;
    }
    if (event.operationId() != null) {
        return;
    }
    tenantContextRunner.runInTenantContext(event.tenancyId(), () ->
            recomputeWithRetry(event.parentProgressId(), event.tenancyId()));
}
```

- [ ] **Step 9: Run all progress tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime,progress-memory,progress-core`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add progress-runtime/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#332): add rollbackToTimestamp, applyRollupState, operationId suppression Refs #332 Refs #92"
```

---

### Task 6: SubtreeRollbackService

**Files:**
- Create: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/SubtreeRollbackService.java`
- Create: `progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/SubtreeRollbackServiceTest.java`

**Interfaces:**
- Consumes: `ProgressService.rollbackToTimestamp(UUID, Instant, UUID)` (from Task 5)
- Consumes: `ProgressService.applyRollupState(UUID, List<ProgressInstance>, UUID)` (from Task 5)
- Consumes: `ProgressInstanceStore.findDescendantsOf(UUID)` (from Task 3)
- Consumes: `ProgressInstanceStore.get(UUID)` (existing)
- Consumes: `ProgressInstanceStore.findByParentProgressId(UUID)` (existing)
- Consumes: `ProgressEventStore.findById(UUID)` (existing)
- Produces: `SubtreeRollbackService.rollbackSubtree(UUID rootId, Instant targetTimestamp)` → `SubtreeRollbackResult`
- Produces: `SubtreeRollbackService.rollbackSubtreeToEvent(UUID rootId, UUID eventId)` → `SubtreeRollbackResult`

- [ ] **Step 1: Write core test — 3-node tree timestamp rollback**

Create `progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/SubtreeRollbackServiceTest.java`:

```java
package io.casehub.work.progress.runtime.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.work.progress.*;
import io.casehub.work.progress.memory.InMemoryProgressEventStore;
import io.casehub.work.progress.memory.InMemoryProgressInstanceStore;
import io.casehub.work.progress.rollup.RollupEngine;
import io.casehub.work.progress.validation.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class SubtreeRollbackServiceTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private InMemoryProgressInstanceStore instanceStore;
    private InMemoryProgressEventStore eventStore;
    private ProgressService progressService;
    private SubtreeRollbackService subtreeRollbackService;
    private List<ProgressUpdatedEvent> emittedEvents;

    @BeforeEach
    void setUp() {
        instanceStore = new InMemoryProgressInstanceStore();
        eventStore = new InMemoryProgressEventStore();
        emittedEvents = new ArrayList<>();

        progressService = new ProgressService(
                instanceStore, eventStore,
                List.of(new PercentageShapeValidator(), new CountShapeValidator()),
                new StepValidator(), new StepShapeValidator(),
                new RollbackDetector(),
                (expr, ctx) -> true,
                emittedEvents::add);

        subtreeRollbackService = new SubtreeRollbackService(
                progressService, instanceStore, eventStore);
    }

    @Test
    void rollbackSubtree_rollsBackAllLeafNodes() {
        // Create root (rollup) + 2 leaf children
        ProgressInstance root = progressService.create(
                new ProgressCreateRequest("t1", "workitem", "root", "count", countState(0, 2),
                        null, "count-completed", null, null, null));
        ProgressInstance child1 = progressService.attachChild(root.id(),
                new ProgressCreateRequest("t1", "workitem", "c1", "percentage", percentageState(0),
                        null, null, null, null, null));
        ProgressInstance child2 = progressService.attachChild(root.id(),
                new ProgressCreateRequest("t1", "workitem", "c2", "percentage", percentageState(0),
                        null, null, null, null, null));

        Instant targetTime = Instant.now();

        // Advance children
        progressService.updateState(child1.id(), percentageState(50));
        progressService.updateState(child2.id(), percentageState(70));

        // Rollback to targetTime
        SubtreeRollbackResult result = subtreeRollbackService.rollbackSubtree(root.id(), targetTime);

        assertNotNull(result);
        assertNotNull(result.operationId());

        // Verify children were rolled back
        long rolledBack = result.outcomes().stream()
                .filter(o -> o.outcome() == NodeRollbackOutcome.Outcome.ROLLED_BACK)
                .count();
        assertTrue(rolledBack >= 2); // at least the 2 leaf children

        // Verify child states restored
        assertEquals(0, instanceStore.get(child1.id()).get().state().get("value").asInt());
        assertEquals(0, instanceStore.get(child2.id()).get().state().get("value").asInt());
    }

    @Test
    void rollbackSubtree_skipsPostTargetNodes() {
        ProgressInstance root = progressService.create(
                new ProgressCreateRequest("t1", "workitem", "root", "percentage", percentageState(0),
                        null, null, null, null, null));

        Instant targetTime = Instant.now();

        // Create child AFTER target
        ProgressInstance child = progressService.attachChild(root.id(),
                new ProgressCreateRequest("t1", "workitem", "c1", "percentage", percentageState(50),
                        null, null, null, null, null));

        SubtreeRollbackResult result = subtreeRollbackService.rollbackSubtree(root.id(), targetTime);

        long skipped = result.outcomes().stream()
                .filter(o -> o.outcome() == NodeRollbackOutcome.Outcome.SKIPPED)
                .filter(o -> o.reason().contains("created after target"))
                .count();
        assertTrue(skipped >= 1);
    }

    @Test
    void rollbackSubtree_operationIdOnAllEvents() {
        ProgressInstance root = progressService.create(
                new ProgressCreateRequest("t1", "workitem", "root", "percentage", percentageState(0),
                        null, null, null, null, null));
        Instant targetTime = Instant.now();
        progressService.updateState(root.id(), percentageState(50));
        emittedEvents.clear();

        SubtreeRollbackResult result = subtreeRollbackService.rollbackSubtree(root.id(), targetTime);

        for (ProgressUpdatedEvent event : emittedEvents) {
            assertEquals(result.operationId(), event.operationId());
        }
    }

    @Test
    void rollbackSubtreeToEvent_delegatesToTimestamp() {
        ProgressInstance root = progressService.create(
                new ProgressCreateRequest("t1", "workitem", "root", "percentage", percentageState(0),
                        null, null, null, null, null));
        progressService.updateState(root.id(), percentageState(50));

        List<ProgressUpdatedEvent> events = eventStore.findByProgressId(root.id());
        ProgressUpdatedEvent targetEvent = events.get(0); // CREATED event

        progressService.updateState(root.id(), percentageState(80));

        SubtreeRollbackResult result = subtreeRollbackService.rollbackSubtreeToEvent(root.id(), targetEvent.id());

        assertNotNull(result);
        assertEquals(0, instanceStore.get(root.id()).get().state().get("value").asInt());
    }

    @Test
    void rollbackSubtree_policyBypassedFlagged() {
        ProgressInstance root = progressService.create(
                new ProgressCreateRequest("t1", "workitem", "root", "percentage", percentageState(0),
                        null, null, null, "denied", null)  // rollbackPolicy = "denied");
        Instant targetTime = Instant.now();
        progressService.updateState(root.id(), percentageState(50));

        SubtreeRollbackResult result = subtreeRollbackService.rollbackSubtree(root.id(), targetTime);

        boolean anyBypassed = result.outcomes().stream()
                .filter(o -> o.outcome() == NodeRollbackOutcome.Outcome.ROLLED_BACK)
                .anyMatch(NodeRollbackOutcome::policyBypassed);
        assertTrue(anyBypassed);
    }

    private JsonNode percentageState(int value) {
        return MAPPER.createObjectNode().put("value", value);
    }

    private JsonNode countState(int current, int total) {
        return MAPPER.createObjectNode().put("current", current).put("total", total);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SubtreeRollbackServiceTest -pl progress-runtime`
Expected: FAIL — `SubtreeRollbackService` doesn't exist

- [ ] **Step 3: Implement SubtreeRollbackService**

Write new file `progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/SubtreeRollbackService.java`:

```java
package io.casehub.work.progress.runtime.service;

import io.casehub.work.progress.*;
import io.casehub.work.progress.spi.ProgressEventStore;
import io.casehub.work.progress.spi.ProgressInstanceStore;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

public class SubtreeRollbackService {

    private final ProgressService progressService;
    private final ProgressInstanceStore instanceStore;
    private final ProgressEventStore eventStore;

    public SubtreeRollbackService(ProgressService progressService,
                                   ProgressInstanceStore instanceStore,
                                   ProgressEventStore eventStore) {
        this.progressService = progressService;
        this.instanceStore = instanceStore;
        this.eventStore = eventStore;
    }

    public SubtreeRollbackResult rollbackSubtree(UUID rootId, Instant targetTimestamp) {
        UUID operationId = UUID.randomUUID();

        ProgressInstance root = instanceStore.get(rootId)
                .orElseThrow(() -> new IllegalArgumentException("Progress instance not found: " + rootId));

        List<ProgressInstance> descendants = instanceStore.findDescendantsOf(rootId);
        List<ProgressInstance> allNodes = new ArrayList<>();
        allNodes.add(root);
        allNodes.addAll(descendants);

        List<ProgressInstance> leafNodes = allNodes.stream()
                .filter(n -> n.rollupStrategyId() == null)
                .toList();
        List<ProgressInstance> rollupNodes = allNodes.stream()
                .filter(n -> n.rollupStrategyId() != null)
                .toList();

        List<NodeRollbackOutcome> outcomes = new ArrayList<>();

        // Phase 1: Roll back leaf nodes
        for (ProgressInstance node : leafNodes) {
            outcomes.add(rollbackNode(node, targetTimestamp, operationId));
        }

        // Phase 2: Bottom-up rollup recomputation
        Map<UUID, Integer> depthMap = computeDepths(root, descendants);
        List<ProgressInstance> sortedRollupNodes = rollupNodes.stream()
                .sorted(Comparator.comparingInt((ProgressInstance n) -> depthMap.getOrDefault(n.id(), 0)).reversed())
                .toList();

        for (ProgressInstance rollupNode : sortedRollupNodes) {
            if (rollupNode.createdAt().isAfter(targetTimestamp)) {
                outcomes.add(new NodeRollbackOutcome(rollupNode.id(),
                        NodeRollbackOutcome.Outcome.SKIPPED, "created after target timestamp",
                        null, null, false));
                continue;
            }

            List<ProgressInstance> children = instanceStore.findByParentProgressId(rollupNode.id());
            List<ProgressInstance> preTargetChildren = children.stream()
                    .filter(c -> !c.createdAt().isAfter(targetTimestamp))
                    .toList();

            try {
                ProgressInstance result = progressService.applyRollupState(
                        rollupNode.id(), preTargetChildren, operationId);
                if (result != null) {
                    outcomes.add(new NodeRollbackOutcome(rollupNode.id(),
                            NodeRollbackOutcome.Outcome.ROLLED_BACK, null,
                            rollupNode.state(), result.state(), false));
                } else {
                    outcomes.add(new NodeRollbackOutcome(rollupNode.id(),
                            NodeRollbackOutcome.Outcome.SKIPPED, "already at target state",
                            null, null, false));
                }
            } catch (Exception e) {
                outcomes.add(new NodeRollbackOutcome(rollupNode.id(),
                        NodeRollbackOutcome.Outcome.FAILED, e.getMessage(),
                        null, null, false));
            }
        }

        return new SubtreeRollbackResult(operationId, rootId, targetTimestamp, outcomes);
    }

    public SubtreeRollbackResult rollbackSubtreeToEvent(UUID rootId, UUID eventId) {
        ProgressUpdatedEvent event = eventStore.findById(eventId)
                .orElseThrow(() -> new IllegalArgumentException("Event not found: " + eventId));
        return rollbackSubtree(rootId, event.timestamp());
    }

    private NodeRollbackOutcome rollbackNode(ProgressInstance node, Instant targetTimestamp, UUID operationId) {
        if (node.createdAt().isAfter(targetTimestamp)) {
            return new NodeRollbackOutcome(node.id(),
                    NodeRollbackOutcome.Outcome.SKIPPED, "created after target timestamp",
                    null, null, false);
        }

        try {
            ProgressInstance result = progressService.rollbackToTimestamp(node.id(), targetTimestamp, operationId);
            if (result == null) {
                return new NodeRollbackOutcome(node.id(),
                        NodeRollbackOutcome.Outcome.SKIPPED, "already at target state",
                        null, null, false);
            }
            boolean bypassed = "denied".equalsIgnoreCase(node.rollbackPolicy());
            return new NodeRollbackOutcome(node.id(),
                    NodeRollbackOutcome.Outcome.ROLLED_BACK, null,
                    node.state(), result.state(), bypassed);
        } catch (IllegalStateException e) {
            return new NodeRollbackOutcome(node.id(),
                    NodeRollbackOutcome.Outcome.SKIPPED, e.getMessage(),
                    null, null, false);
        } catch (Exception e) {
            return new NodeRollbackOutcome(node.id(),
                    NodeRollbackOutcome.Outcome.FAILED, e.getMessage(),
                    null, null, false);
        }
    }

    private Map<UUID, Integer> computeDepths(ProgressInstance root, List<ProgressInstance> descendants) {
        Map<UUID, Integer> depths = new HashMap<>();
        depths.put(root.id(), 0);
        Map<UUID, List<ProgressInstance>> byParent = descendants.stream()
                .filter(d -> d.parentProgressId() != null)
                .collect(Collectors.groupingBy(ProgressInstance::parentProgressId));

        Queue<UUID> queue = new LinkedList<>();
        queue.add(root.id());
        while (!queue.isEmpty()) {
            UUID parentId = queue.poll();
            int parentDepth = depths.get(parentId);
            List<ProgressInstance> children = byParent.getOrDefault(parentId, List.of());
            for (ProgressInstance child : children) {
                depths.put(child.id(), parentDepth + 1);
                queue.add(child.id());
            }
        }
        return depths;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SubtreeRollbackServiceTest -pl progress-runtime`
Expected: ALL PASS

- [ ] **Step 5: Add edge case tests — step-shaped descendants, no-op**

Add to `SubtreeRollbackServiceTest.java`:

```java
@Test
void rollbackSubtree_noOpWhenAlreadyAtTarget() {
    ProgressInstance root = progressService.create(
            new ProgressCreateRequest("t1", "workitem", "root", "percentage", percentageState(0),
                    null, null, null));
    Instant targetTime = Instant.now();

    SubtreeRollbackResult result = subtreeRollbackService.rollbackSubtree(root.id(), targetTime);

    long skipped = result.outcomes().stream()
            .filter(o -> o.outcome() == NodeRollbackOutcome.Outcome.SKIPPED)
            .filter(o -> "already at target state".equals(o.reason()))
            .count();
    assertEquals(1, skipped);
}
```

- [ ] **Step 6: Run all progress tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime,progress-memory,progress-core`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add progress-runtime/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#332): add SubtreeRollbackService with tests Refs #332 Refs #92"
```

---

## Batch 4: REST, Migration, Documentation

### Task 7: REST endpoint, Flyway migration, documentation updates

**Files:**
- Modify: `progress-rest/src/main/java/io/casehub/work/progress/rest/ProgressResource.java`
- Create: `progress-runtime/src/main/resources/db/work/migration/V7000__progress_schema.sql`
- Modify: `ARC42STORIES.MD` (new chapter entry)
- Modify: `docs/api-reference.md` (new endpoint)

**Interfaces:**
- Consumes: `SubtreeRollbackService.rollbackSubtree(UUID, Instant)` (from Task 6)
- Consumes: `SubtreeRollbackService.rollbackSubtreeToEvent(UUID, UUID)` (from Task 6)
- Produces: `POST /progress/{id}/rollback/subtree` REST endpoint

- [ ] **Step 1: Add SubtreeRollbackService injection to ProgressResource**

Use `ide_insert_member` on `ProgressResource.java` — add field after `broadcaster`:

```java
@Inject
SubtreeRollbackService subtreeRollbackService;
```

Add the import for `SubtreeRollbackService`.

- [ ] **Step 2: Add REST endpoint**

Use `ide_insert_member` on `ProgressResource.java` after `rollback` method:

```java
@POST
@Path("/{id}/rollback/subtree")
public Response rollbackSubtree(
        @PathParam("id") UUID id,
        @QueryParam("timestamp") String timestamp,
        @QueryParam("toEvent") UUID toEventId) {
    if (timestamp != null && toEventId != null) {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity("timestamp and toEvent are mutually exclusive").build();
    }
    SubtreeRollbackResult result;
    if (toEventId != null) {
        result = subtreeRollbackService.rollbackSubtreeToEvent(id, toEventId);
    } else if (timestamp != null) {
        result = subtreeRollbackService.rollbackSubtree(id, Instant.parse(timestamp));
    } else {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity("timestamp or toEvent required").build();
    }
    return Response.ok(result).build();
}
```

- [ ] **Step 3: Create Flyway migration V7000**

Create directory and file `progress-runtime/src/main/resources/db/work/migration/V7000__progress_schema.sql`:

```sql
CREATE TABLE progress_instance (
    id                  UUID         NOT NULL,
    version             BIGINT       NOT NULL DEFAULT 0,
    tenancy_id          VARCHAR(255) NOT NULL,
    scope_type          VARCHAR(255) NOT NULL,
    scope_id            VARCHAR(255) NOT NULL,
    parent_progress_id  UUID,
    root_progress_id    UUID         NOT NULL,
    shape_type          VARCHAR(50)  NOT NULL,
    definition          JSONB,
    state               JSONB        NOT NULL,
    status              VARCHAR(20)  NOT NULL,
    rollup_strategy_id  VARCHAR(255),
    rollback_policy     VARCHAR(20),
    visualisation_mode  VARCHAR(50),
    created_at          TIMESTAMP    NOT NULL,
    updated_at          TIMESTAMP    NOT NULL,
    CONSTRAINT pk_progress_instance PRIMARY KEY (id)
);

CREATE INDEX idx_progress_scope ON progress_instance (scope_type, scope_id);
CREATE INDEX idx_progress_parent ON progress_instance (parent_progress_id);
CREATE INDEX idx_progress_root ON progress_instance (root_progress_id);
CREATE INDEX idx_progress_tenancy ON progress_instance (tenancy_id);

CREATE TABLE progress_event (
    id                UUID         NOT NULL,
    tenancy_id        VARCHAR(255) NOT NULL,
    progress_id       UUID         NOT NULL,
    root_progress_id  UUID         NOT NULL,
    scope_type        VARCHAR(255) NOT NULL,
    scope_id          VARCHAR(255) NOT NULL,
    change_type       VARCHAR(30)  NOT NULL,
    previous_state    JSONB,
    current_state     JSONB,
    status            VARCHAR(20)  NOT NULL,
    occurred_at       TIMESTAMP    NOT NULL,
    operation_id      UUID,
    CONSTRAINT pk_progress_event PRIMARY KEY (id)
);

CREATE INDEX idx_progress_event_progress ON progress_event (progress_id, occurred_at);
CREATE INDEX idx_progress_event_root ON progress_event (root_progress_id, occurred_at);
CREATE INDEX idx_progress_event_tenancy ON progress_event (tenancy_id);
CREATE INDEX idx_progress_event_scope ON progress_event (scope_type, scope_id, occurred_at);
CREATE INDEX idx_progress_event_operation ON progress_event (operation_id) WHERE operation_id IS NOT NULL;
```

- [ ] **Step 4: Add subtree rollback endpoint to api-reference.md**

Add under the Progress Rollback section in `docs/api-reference.md`:

```markdown
### Subtree Rollback

`POST /progress/{id}/rollback/subtree`

Rolls back all descendants of a progress instance to their state at a given point in time.

**Query Parameters (mutually exclusive, at least one required):**

| Parameter | Type | Description |
|-----------|------|-------------|
| `timestamp` | ISO-8601 Instant | Roll back to state at this timestamp |
| `toEvent` | UUID | Roll back to state at this event's timestamp |

**Response:** `SubtreeRollbackResult`

| Field | Type | Description |
|-------|------|-------------|
| `operationId` | UUID | Correlates all events emitted during this operation |
| `rootId` | UUID | The root of the rolled-back subtree |
| `targetTimestamp` | Instant | The target point in time |
| `outcomes` | `List<NodeRollbackOutcome>` | Per-node results |

**NodeRollbackOutcome:**

| Field | Type | Description |
|-------|------|-------------|
| `progressId` | UUID | The processed instance |
| `outcome` | enum | `ROLLED_BACK`, `SKIPPED`, `FAILED` |
| `reason` | String | null for ROLLED_BACK; explanation for SKIPPED/FAILED |
| `previousState` | JSON | State before rollback (null if SKIPPED) |
| `restoredState` | JSON | State after rollback (null if SKIPPED/FAILED) |
| `policyBypassed` | boolean | True if rollbackPolicy=denied was bypassed |

**Error Codes:**
- 400: Both `timestamp` and `toEvent` provided, or neither
- 404: Progress instance not found
```

- [ ] **Step 5: Verify with ide_diagnostics**

Run `ide_diagnostics` on progress-rest module.

- [ ] **Step 6: Run all progress module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime,progress-memory,progress-core,progress-rest`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add progress-runtime/ progress-rest/ docs/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#332): add subtree rollback REST endpoint, Flyway V7000, API docs Refs #332 Refs #92"
```

---

## References

- `specs/issue-332-multi-instance-rollback/2026-08-21-multi-instance-coordinated-rollback-design.md` — design spec
- `specs/issue-332-multi-instance-rollback/decisions.md` — 15 design decisions
- `progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/ProgressService.java` — single-instance rollback (lines 178-237, 392-402)
- `progress-runtime/src/main/java/io/casehub/work/progress/runtime/event/RollupObserver.java` — async rollup cascade
- `progress-runtime/src/main/java/io/casehub/work/progress/runtime/repository/JpaProgressEventStore.java` — event store JPA (lines 70-100)
- `progress-runtime/src/main/java/io/casehub/work/progress/runtime/model/ProgressEventEntity.java` — event entity
- `progress-rest/src/main/java/io/casehub/work/progress/rest/ProgressResource.java` — REST endpoints (lines 135-143, 206-213)
- `progress-api/src/main/java/io/casehub/work/progress/ProgressUpdatedEvent.java` — event record
- `progress-api/src/main/java/io/casehub/work/progress/spi/ProgressEventStore.java` — event store SPI
- `progress-api/src/main/java/io/casehub/work/progress/spi/ProgressInstanceStore.java` — instance store SPI
- `progress-memory/src/main/java/io/casehub/work/progress/memory/InMemoryProgressInstanceStore.java` — in-memory instance store
- `progress-memory/src/main/java/io/casehub/work/progress/memory/InMemoryProgressEventStore.java` — in-memory event store
- `docs/FLYWAY.md` — migration conventions (V7000-V7999 for progress)
- `docs/protocols/casehub/async-event-tenant-context-propagation.md` — @ObservesAsync tenant context
- GitHub #332, #92
