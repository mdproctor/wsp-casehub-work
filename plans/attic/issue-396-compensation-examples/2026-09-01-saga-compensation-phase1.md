# Saga Compensation Phase 1 — casehub-work Foundation

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #238 — saga compensation support across the casehub platform
**Issue group:** #238

**Goal:** Add compensation domain model, service methods, and REST API to casehub-work so that WorkItems can be compensated via a separate compensating WorkItem entity.

**Architecture:** A completed WorkItem can be compensated by creating a new compensating WorkItem linked via `compensatesWorkItemId`. The original gets a denormalized `compensationStatus` field (NONE → COMPENSATING → COMPENSATED). The terminal-state invariant is preserved — the original stays COMPLETED. A compensating WorkItem cannot itself be compensated (D14).

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate/JPA, Flyway, MongoDB (optional persistence), REST/Jackson

## Global Constraints

- Java 21 source level on Java 26 JVM
- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` for all builds
- Build with `mvn` not `./mvnw`
- Always use `-pl <module>` — never full-project builds
- Use `ide_insert_member` / `ide_replace_member` / `ide_edit_member` for code edits
- Use `ide_refactor_rename` for renames — never sed/grep
- All commits reference `Refs #238`

---

## Batch 1: Domain Model + Persistence

### Task 1: CompensationStatus Enum + WorkItem Record + WorkItemEntity Fields

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/CompensationStatus.java`
- Modify: `api/src/main/java/io/casehub/work/api/WorkItem.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemEntity.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemEntityMapper.java`
- Test: `api/src/test/java/io/casehub/work/api/CompensationStatusTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/WorkItemEntityMapperTest.java`

**Interfaces:**
- Produces: `CompensationStatus` enum (NONE, COMPENSATING, COMPENSATED) — used by Task 2, 3, 4, 5
- Produces: `WorkItem.compensationStatus()` accessor — used by Task 3, 4, 5
- Produces: `WorkItem.compensatesWorkItemId()` accessor — used by Task 3, 4, 5
- Produces: `WorkItemEntity.compensationStatus` field — used by Task 2, 3
- Produces: `WorkItemEntity.compensatesWorkItemId` field — used by Task 2, 3

- [ ] **Step 1: Write CompensationStatus enum test**

```java
// api/src/test/java/io/casehub/work/api/CompensationStatusTest.java
package io.casehub.work.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CompensationStatusTest {

    @Test
    void enumValues() {
        assertEquals(3, CompensationStatus.values().length);
        assertNotNull(CompensationStatus.NONE);
        assertNotNull(CompensationStatus.COMPENSATING);
        assertNotNull(CompensationStatus.COMPENSATED);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CompensationStatusTest -pl api`
Expected: FAIL — `CompensationStatus` does not exist

- [ ] **Step 3: Create CompensationStatus enum**

```java
// api/src/main/java/io/casehub/work/api/CompensationStatus.java
package io.casehub.work.api;

public enum CompensationStatus {
    NONE,
    COMPENSATING,
    COMPENSATED
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CompensationStatusTest -pl api`
Expected: PASS

- [ ] **Step 5: Add compensation fields to WorkItem record**

Use `ide_edit_member` on `WorkItem` record to add two new fields after `escalationGenerateSummary`:

```java
CompensationStatus compensationStatus,
UUID compensatesWorkItemId
```

Update `Builder` class: add two new fields with builders and include in `build()` method. Update `toBuilder()` to include the new fields.

- [ ] **Step 6: Add compensation fields to WorkItemEntity**

Use `ide_insert_member` on `WorkItemEntity` to add after `escalationGenerateSummary`:

```java
@Enumerated(EnumType.STRING)
@Column(name = "compensation_status")
public CompensationStatus compensationStatus = CompensationStatus.NONE;

@Column(name = "compensates_work_item_id")
public UUID compensatesWorkItemId;
```

- [ ] **Step 7: Update WorkItemEntityMapper**

Use `ide_edit_member` on `WorkItemEntityMapper` to map the two new fields in both `toWorkItem()` (entity → domain) and `toEntity()` (domain → entity) methods:

```java
// in toWorkItem():
.compensationStatus(entity.compensationStatus != null ? entity.compensationStatus : CompensationStatus.NONE)
.compensatesWorkItemId(entity.compensatesWorkItemId)

// in toEntity():
entity.compensationStatus = workItem.compensationStatus() != null ? workItem.compensationStatus() : CompensationStatus.NONE;
entity.compensatesWorkItemId = workItem.compensatesWorkItemId();
```

- [ ] **Step 8: Update WorkItemEntityMapper test**

Add a test case in `WorkItemEntityMapperTest` that verifies compensation fields round-trip through the mapper.

- [ ] **Step 9: Run tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=CompensationStatusTest && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemEntityMapperTest`
Expected: PASS

- [ ] **Step 10: Verify with ide_diagnostics**

Run `ide_diagnostics` on `WorkItemEntity.java`, `WorkItem.java`, and `WorkItemEntityMapper.java` to catch any compilation errors.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add api/src runtime/src
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#238): add CompensationStatus enum and entity fields Refs #238"
```

### Task 2: Flyway Migration + MongoWorkItemDocument

**Files:**
- Create: `runtime/src/main/resources/db/work/migration/V43__compensation_fields.sql`
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java`
- Test: `integration-tests` (existing IT suite validates schema)

**Interfaces:**
- Consumes: `CompensationStatus` enum from Task 1
- Consumes: `WorkItemEntity.compensationStatus` and `.compensatesWorkItemId` fields from Task 1
- Produces: `compensation_status` and `compensates_work_item_id` columns in `work_item` table
- Produces: `compensationStatus` and `compensatesWorkItemId` fields on `MongoWorkItemDocument`

- [ ] **Step 1: Create Flyway V43 migration**

```sql
-- V43__compensation_fields.sql
ALTER TABLE work_item ADD COLUMN compensation_status VARCHAR(20) DEFAULT 'NONE';
ALTER TABLE work_item ADD COLUMN compensates_work_item_id UUID;
ALTER TABLE work_item ADD CONSTRAINT fk_compensates_work_item
    FOREIGN KEY (compensates_work_item_id) REFERENCES work_item(id);
CREATE INDEX idx_work_item_compensates ON work_item(compensates_work_item_id)
    WHERE compensates_work_item_id IS NOT NULL;
```

- [ ] **Step 2: Add fields to MongoWorkItemDocument**

Use `ide_insert_member` to add after the last field:

```java
public CompensationStatus compensationStatus;
public UUID compensatesWorkItemId;
```

Update `from(WorkItem)` method to map:
```java
doc.compensationStatus = wi.compensationStatus();
doc.compensatesWorkItemId = wi.compensatesWorkItemId();
```

Update `toDomain()` method to map:
```java
.compensationStatus(compensationStatus != null ? compensationStatus : CompensationStatus.NONE)
.compensatesWorkItemId(compensatesWorkItemId)
```

- [ ] **Step 3: Verify with ide_diagnostics**

Run `ide_diagnostics` on `MongoWorkItemDocument.java`.

- [ ] **Step 4: Run runtime module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS (Flyway migration applies during test startup)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add runtime/src persistence-mongodb/src
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#238): add V43 compensation migration and MongoDB document fields Refs #238"
```

---

## Batch 2: Service Layer + Events

### Task 3: WorkItemService.compensate() and markCompensated()

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java`

**Interfaces:**
- Consumes: `CompensationStatus` enum (Task 1)
- Consumes: `WorkItemEntity.compensationStatus` and `.compensatesWorkItemId` (Task 1)
- Produces: `WorkItemService.compensate(UUID originalId, WorkItemCreateRequest request, String triggeredBy, String reason)` → returns new compensating `WorkItem`
- Produces: `WorkItemService.markCompensated(UUID originalId)` → returns updated original `WorkItem`

- [ ] **Step 1: Write test for compensate() — happy path**

```java
// In WorkItemServiceTest.java — add test method
@Test
void compensate_completedWorkItem_createsCompensatingWorkItem() {
    // Given: a COMPLETED WorkItem
    WorkItem original = createAndComplete("Original task");

    // When: compensate is called
    WorkItemCreateRequest request = WorkItemCreateRequest.builder()
            .title("Reverse: Original task")
            .candidateGroups("reviewers")
            .createdBy("operator-1")
            .build();
    WorkItem compensating = workItemService.compensate(
            original.id(), request, "operator-1", "Clinical trial withdrawn");

    // Then: compensating WorkItem is created with link
    assertNotNull(compensating);
    assertEquals(original.id(), compensating.compensatesWorkItemId());
    assertEquals(WorkItemStatus.PENDING, compensating.status());

    // And: original is marked COMPENSATING
    WorkItem updated = workItemService.findById(original.id()).orElseThrow();
    assertEquals(CompensationStatus.COMPENSATING, updated.compensationStatus());
}
```

- [ ] **Step 2: Write test for compensate() — rejects non-COMPLETED**

```java
@Test
void compensate_pendingWorkItem_throws() {
    WorkItem pending = createWorkItem("Pending task");
    WorkItemCreateRequest request = WorkItemCreateRequest.builder()
            .title("Compensate").candidateGroups("g").createdBy("op").build();

    assertThrows(IllegalStateException.class, () ->
            workItemService.compensate(pending.id(), request, "op", "reason"));
}
```

- [ ] **Step 3: Write test for compensate() — rejects already-compensating**

```java
@Test
void compensate_alreadyCompensating_throws() {
    WorkItem original = createAndComplete("Original");
    WorkItemCreateRequest request = WorkItemCreateRequest.builder()
            .title("Compensate").candidateGroups("g").createdBy("op").build();
    workItemService.compensate(original.id(), request, "op", "reason");

    // Second compensation attempt
    assertThrows(IllegalStateException.class, () ->
            workItemService.compensate(original.id(), request, "op", "again"));
}
```

- [ ] **Step 4: Write test for compensate() — rejects compensating a compensating WorkItem (D14)**

```java
@Test
void compensate_compensatingWorkItem_throws() {
    WorkItem original = createAndComplete("Original");
    WorkItemCreateRequest request = WorkItemCreateRequest.builder()
            .title("Compensate").candidateGroups("g").createdBy("op").build();
    WorkItem compensating = workItemService.compensate(original.id(), request, "op", "reason");

    // Complete the compensating WorkItem, then try to compensate IT
    workItemService.claim(compensating.id(), "worker-1");
    workItemService.start(compensating.id(), "worker-1");
    workItemService.complete(compensating.id(), "worker-1", "done", null);

    assertThrows(IllegalStateException.class, () ->
            workItemService.compensate(compensating.id(), request, "op", "meta"));
}
```

- [ ] **Step 5: Write test for markCompensated()**

```java
@Test
void markCompensated_setsStatusToCompensated() {
    WorkItem original = createAndComplete("Original");
    WorkItemCreateRequest request = WorkItemCreateRequest.builder()
            .title("Compensate").candidateGroups("g").createdBy("op").build();
    workItemService.compensate(original.id(), request, "op", "reason");

    workItemService.markCompensated(original.id());

    WorkItem updated = workItemService.findById(original.id()).orElseThrow();
    assertEquals(CompensationStatus.COMPENSATED, updated.compensationStatus());
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest#compensate* -pl runtime`
Expected: FAIL — methods do not exist

- [ ] **Step 7: Implement compensate() method**

Use `ide_insert_member` on `WorkItemService` to add after `cancelFromSystem()`:

```java
public WorkItem compensate(UUID originalId, WorkItemCreateRequest request,
                           String triggeredBy, String reason) {
    WorkItemEntity original = requireWorkItem(originalId);

    if (original.status != WorkItemStatus.COMPLETED) {
        throw new IllegalStateException("Only COMPLETED WorkItems can be compensated; current status: " + original.status);
    }
    if (original.compensationStatus != CompensationStatus.NONE) {
        throw new IllegalStateException("WorkItem already has compensation activity: " + original.compensationStatus);
    }
    if (original.compensatesWorkItemId != null) {
        throw new IllegalStateException("Compensating WorkItems cannot themselves be compensated");
    }

    original.compensationStatus = CompensationStatus.COMPENSATING;
    workItemStore.put(original);
    audit(original.id, "COMPENSATION_STARTED", triggeredBy, reason);

    WorkItem compensatingItem = create(request.toBuilder()
            .compensatesWorkItemId(originalId)
            .build());

    lifecycleEmitter.fire(WorkEventType.COMPENSATION_STARTED, original, triggeredBy, reason);

    return compensatingItem;
}

public WorkItem markCompensated(UUID originalId) {
    WorkItemEntity original = requireWorkItem(originalId);
    original.compensationStatus = CompensationStatus.COMPENSATED;
    workItemStore.put(original);
    audit(original.id, "COMPENSATION_COMPLETED", "system", "Compensating WorkItem completed");
    lifecycleEmitter.fire(WorkEventType.COMPENSATION_COMPLETED, original, "system", null);
    return WorkItemEntityMapper.toWorkItem(original);
}
```

Note: `WorkItemCreateRequest` needs a `compensatesWorkItemId` field and `toBuilder()` method — add via `ide_insert_member` if not already present.

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest -pl runtime`
Expected: PASS

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on `WorkItemService.java`.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add runtime/src
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#238): add compensate() and markCompensated() to WorkItemService Refs #238"
```

### Task 4: CompensationLifecycleObserver + WorkEventType Additions

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/WorkEventType.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/event/CompensationLifecycleObserver.java`
- Test: `api/src/test/java/io/casehub/work/api/WorkEventTypeTest.java` (existing — verify new values)
- Test: `runtime/src/test/java/io/casehub/work/runtime/event/CompensationLifecycleObserverTest.java`

**Interfaces:**
- Consumes: `WorkItemService.markCompensated(UUID)` (Task 3)
- Consumes: `WorkEventType` enum (existing)
- Produces: `WorkEventType.COMPENSATION_STARTED` — fired when compensation begins
- Produces: `WorkEventType.COMPENSATION_COMPLETED` — fired when compensation finishes
- Produces: `CompensationLifecycleObserver` — auto-marks original as COMPENSATED when compensating WorkItem completes

- [ ] **Step 1: Add COMPENSATION_STARTED and COMPENSATION_COMPLETED to WorkEventType**

Use `ide_edit_member` on `WorkEventType` to add two new values before the closing brace:

```java
COMPENSATION_STARTED,
COMPENSATION_COMPLETED
```

- [ ] **Step 2: Verify existing WorkEventTypeTest still passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkEventTypeTest -pl api`
Expected: PASS (or update count assertion if test checks enum size)

- [ ] **Step 3: Write CompensationLifecycleObserver test**

```java
// runtime/src/test/java/io/casehub/work/runtime/event/CompensationLifecycleObserverTest.java
package io.casehub.work.runtime.event;

import io.casehub.work.api.*;
import io.casehub.work.runtime.service.WorkItemService;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.UUID;

import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CompensationLifecycleObserverTest {

    @Mock WorkItemService workItemService;
    @InjectMocks CompensationLifecycleObserver observer;

    @Test
    void completedCompensatingWorkItem_marksOriginalCompensated() {
        UUID originalId = UUID.randomUUID();
        UUID compensatingId = UUID.randomUUID();
        WorkItemStatusEvent event = new WorkItemStatusEvent(
                WorkEventType.COMPLETED, compensatingId, WorkItemStatus.COMPLETED,
                "worker-1", "done", null, null, null, null, "default", Instant.now());

        WorkItem compensatingItem = WorkItem.builder()
                .id(compensatingId).status(WorkItemStatus.COMPLETED)
                .compensatesWorkItemId(originalId).build();
        when(workItemService.findById(compensatingId))
                .thenReturn(java.util.Optional.of(compensatingItem));

        observer.onWorkItemCompleted(event);

        verify(workItemService).markCompensated(originalId);
    }

    @Test
    void completedRegularWorkItem_doesNothing() {
        UUID regularId = UUID.randomUUID();
        WorkItemStatusEvent event = new WorkItemStatusEvent(
                WorkEventType.COMPLETED, regularId, WorkItemStatus.COMPLETED,
                "worker-1", "done", null, null, null, null, "default", Instant.now());

        WorkItem regularItem = WorkItem.builder()
                .id(regularId).status(WorkItemStatus.COMPLETED).build();
        when(workItemService.findById(regularId))
                .thenReturn(java.util.Optional.of(regularItem));

        observer.onWorkItemCompleted(event);

        verify(workItemService, never()).markCompensated(any());
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CompensationLifecycleObserverTest -pl runtime`
Expected: FAIL — class does not exist

- [ ] **Step 5: Implement CompensationLifecycleObserver**

```java
// runtime/src/main/java/io/casehub/work/runtime/event/CompensationLifecycleObserver.java
package io.casehub.work.runtime.event;

import io.casehub.work.api.WorkEventType;
import io.casehub.work.api.WorkItem;
import io.casehub.work.api.WorkItemStatusEvent;
import io.casehub.work.runtime.service.WorkItemService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

@ApplicationScoped
public class CompensationLifecycleObserver {

    @Inject
    WorkItemService workItemService;

    void onWorkItemCompleted(@Observes WorkItemStatusEvent event) {
        if (event.eventType() != WorkEventType.COMPLETED) {
            return;
        }
        workItemService.findById(event.workItemId()).ifPresent(item -> {
            if (item.compensatesWorkItemId() != null) {
                workItemService.markCompensated(item.compensatesWorkItemId());
            }
        });
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CompensationLifecycleObserverTest -pl runtime`
Expected: PASS

- [ ] **Step 7: Verify with ide_diagnostics**

Run `ide_diagnostics` on `CompensationLifecycleObserver.java` and `WorkEventType.java`.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add api/src runtime/src
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#238): add CompensationLifecycleObserver and compensation event types Refs #238"
```

---

## Batch 3: REST API + DTOs

### Task 5: REST Endpoints + Response DTOs + Context Builder

**Files:**
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemResource.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemResponse.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemWithAuditResponse.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemMapper.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java`
- Test: `rest/src/test/java/io/casehub/work/rest/WorkItemResourceTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/event/WorkItemContextBuilderTest.java`

**Interfaces:**
- Consumes: `WorkItemService.compensate()` (Task 3)
- Consumes: `CompensationStatus` (Task 1)
- Produces: `POST /workitems/{id}/compensate` REST endpoint
- Produces: `compensationStatus` and `compensatesWorkItemId` fields on WorkItemResponse

- [ ] **Step 1: Add compensation fields to WorkItemResponse**

Use `ide_edit_member` to add two new fields to the `WorkItemResponse` record:

```java
CompensationStatus compensationStatus,
UUID compensatesWorkItemId
```

- [ ] **Step 2: Add compensation fields to WorkItemWithAuditResponse**

Same pattern — add the two fields.

- [ ] **Step 3: Update WorkItemMapper**

Use `ide_edit_member` on `WorkItemMapper.toResponse()` and `toWithAudit()` to map the new fields:

```java
item.compensationStatus(),
item.compensatesWorkItemId()
```

- [ ] **Step 4: Add compensate endpoint to WorkItemResource**

Use `ide_insert_member` on `WorkItemResource` to add:

```java
@POST
@Path("/{id}/compensate")
public Response compensate(@PathParam("id") UUID id, CompensateRequest request) {
    WorkItem compensating = workItemService.compensate(
            id,
            WorkItemCreateRequest.builder()
                    .title(request.title())
                    .candidateGroups(request.candidateGroups())
                    .createdBy(request.actor())
                    .build(),
            request.actor(),
            request.reason());
    return Response.status(Response.Status.CREATED)
            .entity(WorkItemMapper.toResponse(compensating))
            .build();
}

public record CompensateRequest(String title, String candidateGroups, String actor, String reason) {}
```

- [ ] **Step 5: Update WorkItemContextBuilder**

Use `ide_edit_member` on `WorkItemContextBuilder.toMap()` to add:

```java
map.put("compensationStatus", workItem.compensationStatus() != null
        ? workItem.compensationStatus().name() : "NONE");
map.put("compensatesWorkItemId", workItem.compensatesWorkItemId());
```

- [ ] **Step 6: Write REST endpoint test**

Add to `WorkItemResourceTest`:

```java
@Test
void compensate_completedWorkItem_returns201() {
    // Given: a completed WorkItem (use existing test helper)
    UUID id = createAndCompleteWorkItem();

    // When: POST /workitems/{id}/compensate
    given()
        .contentType(ContentType.JSON)
        .body("""
            {"title": "Reverse action", "candidateGroups": "reviewers",
             "actor": "operator-1", "reason": "trial withdrawn"}
            """)
        .when().post("/workitems/" + id + "/compensate")
        .then()
        .statusCode(201)
        .body("compensatesWorkItemId", is(id.toString()))
        .body("status", is("PENDING"));

    // And: original shows COMPENSATING
    given()
        .when().get("/workitems/" + id)
        .then()
        .body("compensationStatus", is("COMPENSATING"));
}

@Test
void compensate_pendingWorkItem_returns400or500() {
    UUID id = createWorkItem();

    given()
        .contentType(ContentType.JSON)
        .body("""
            {"title": "Compensate", "candidateGroups": "g",
             "actor": "op", "reason": "reason"}
            """)
        .when().post("/workitems/" + id + "/compensate")
        .then()
        .statusCode(anyOf(is(400), is(500)));
}
```

- [ ] **Step 7: Update WorkItemContextBuilderTest**

Add test verifying the new fields appear in the context map.

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemContextBuilderTest -pl runtime`
Expected: PASS

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on all modified files.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add rest/src runtime/src
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#238): add compensation REST endpoint and response DTO fields Refs #238"
```

---

## References

- [2026-09-01-saga-compensation-design.md] — design spec this plan implements (§6: casehub-work)
- [decisions.md] — D2 (separate entity), D13 (idempotency), D14 (no compensation chains), D15 (skip already-compensated)
- `api/src/main/java/io/casehub/work/api/WorkItemStatus.java` — existing 12-value status enum
- `api/src/main/java/io/casehub/work/api/WorkItem.java` — domain record with Builder
- `api/src/main/java/io/casehub/work/api/WorkEventType.java` — 25-value event vocabulary
- `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemEntity.java` — JPA entity
- `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java:617-767` — existing terminal transition methods (fault, obsolete, cancel patterns)
- `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java` — context map builder
- `rest/src/main/java/io/casehub/work/rest/WorkItemResource.java` — REST endpoints
- `rest/src/main/java/io/casehub/work/rest/WorkItemResponse.java` — response DTO
- `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java` — MongoDB document
- `runtime/src/main/resources/db/work/migration/V42__escalation_fields.sql` — latest migration
- GitHub #238 — saga compensation support
