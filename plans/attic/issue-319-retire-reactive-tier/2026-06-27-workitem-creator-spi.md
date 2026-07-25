# WorkItem Creator SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract a complete external control plane (create, find, cancel, complete) for WorkItems into `casehub-work-api`, enabling any module to manage WorkItems without depending on the runtime.

**Architecture:** Four value types move from runtime to api. Two SPI interfaces (`WorkItemCreator`, `WorkItemLifecycle`) are implemented by a hexagonal `WorkItemSpiAdapter` in runtime. `WorkItemEvent` interface in api eliminates entity downcasting for CDI event observers. Template creation is unified via `WorkItemTemplateService.createFromTemplate()`. The engine work-adapter's compile dependency on `casehub-work` is replaced by `casehub-work-api` (cross-repo, separate issue).

**Tech Stack:** Java 21, Quarkus 3.32, JPA/Panache, MongoDB, JUnit 5, AssertJ

## Global Constraints

- **Java:** `JAVA_HOME=$(/usr/libexec/java_home -v 26)` for all Maven commands
- **Build scripts:** Use `scripts/mvn-test`, `scripts/mvn-install`, `scripts/mvn-compile` — never bare `mvn`
- **Refactoring:** Use IntelliJ MCP (`mcp__intellij-index__ide_move_file`, `ide_refactor_rename`) for all moves and renames — never manual find-and-replace
- **Tests:** JUnit 5 + AssertJ. Api module tests are pure Java (no Quarkus). Runtime tests use `@QuarkusTest` where CDI is needed.
- **Flyway:** Runtime range V1–V999, currently at V37. Next available: V38.
- **Package:** Moved types → `io.casehub.work.api`. New SPIs → `io.casehub.work.api.spi`.
- **Spec:** `specs/2026-06-26-workitem-creator-spi-design.md` (revision 4)

---

### Task 1: Move value types and WorkItemStatus to casehub-work-api

**Files:**
- Move: `runtime/.../model/WorkItemPriority.java` → `api/.../api/WorkItemPriority.java`
- Move: `runtime/.../model/WorkItemLabelRequest.java` → `api/.../api/WorkItemLabelRequest.java`
- Move: `runtime/.../model/WorkItemStatus.java` → `api/.../api/WorkItemStatus.java`
- Move: `runtime/.../model/WorkItemCreateRequest.java` → `api/.../api/WorkItemCreateRequest.java`
- Move: `runtime/.../model/WorkItemCreateRequestTest.java` → `api/.../api/WorkItemCreateRequestTest.java`
- Move: `runtime/.../model/WorkItemStatusTest.java` → `api/.../api/WorkItemStatusTest.java`
- Modify: `api/.../api/WorkItemCreateRequest.java` — add `tenancyId` field + builder method

**Interfaces:**
- Consumes: nothing — this is the foundation task
- Produces: `WorkItemPriority`, `WorkItemLabelRequest`, `WorkItemStatus`, `WorkItemCreateRequest` in `io.casehub.work.api` — used by all subsequent tasks

- [ ] **Step 1: Move WorkItemPriority to api**

Use IntelliJ MCP:
```
ide_move_file(file="runtime/src/main/java/io/casehub/work/runtime/model/WorkItemPriority.java",
              destination="api/src/main/java/io/casehub/work/api")
```
This updates all import statements across the project automatically.

- [ ] **Step 2: Move WorkItemLabelRequest to api**

```
ide_move_file(file="runtime/src/main/java/io/casehub/work/runtime/model/WorkItemLabelRequest.java",
              destination="api/src/main/java/io/casehub/work/api")
```

- [ ] **Step 3: Move WorkItemStatus to api**

```
ide_move_file(file="runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java",
              destination="api/src/main/java/io/casehub/work/api")
```

- [ ] **Step 4: Move WorkItemCreateRequest to api**

```
ide_move_file(file="runtime/src/main/java/io/casehub/work/runtime/model/WorkItemCreateRequest.java",
              destination="api/src/main/java/io/casehub/work/api")
```

- [ ] **Step 5: Add tenancyId field to WorkItemCreateRequest**

In `api/src/main/java/io/casehub/work/api/WorkItemCreateRequest.java`, add after the `auditDetail` field:

```java
public final String tenancyId;
```

Add to the private constructor: `this.tenancyId = b.tenancyId;`

Add to Builder:
```java
private String tenancyId;
```
And in the copy constructor: `this.tenancyId = src.tenancyId;`

Add builder method:
```java
public Builder tenancyId(final String v) { this.tenancyId = v; return this; }
```

Add to `equals()`, `hashCode()`, and verify `builderHasSetterForEveryField` test will catch it.

- [ ] **Step 6: Move tests to api**

```
ide_move_file(file="runtime/src/test/java/io/casehub/work/runtime/model/WorkItemCreateRequestTest.java",
              destination="api/src/test/java/io/casehub/work/api")
ide_move_file(file="runtime/src/test/java/io/casehub/work/runtime/model/WorkItemStatusTest.java",
              destination="api/src/test/java/io/casehub/work/api")
```

- [ ] **Step 7: Update WorkItemCreateRequestTest for tenancyId**

Add `tenancyId` to the `builder_setsAllFields` test:
```java
.tenancyId("tenant-1")
```
And assertion: `assertEquals("tenant-1", req.tenancyId);`

Add to `builder_defaultsAllOptionalFieldsToNull`: `assertNull(req.tenancyId);`

Add to `toBuilder_copiesAllFields`:
```java
.tenancyId("tenant-1")
```
And assertion: `assertEquals("tenant-1", copy.tenancyId);`

- [ ] **Step 8: Sync files and verify api module compiles and tests pass**

```bash
ide_sync_files(paths=["api/src/main/java/io/casehub/work/api"])
scripts/mvn-test api
```
Expected: all api tests pass (including moved WorkItemCreateRequestTest, WorkItemStatusTest)

- [ ] **Step 9: Verify runtime module compiles**

```bash
scripts/mvn-compile runtime
```
Expected: compiles — all imports updated by IntelliJ move-refactor

- [ ] **Step 10: Install api to local Maven repo**

```bash
scripts/mvn-install api
```

- [ ] **Step 11: Verify persistence-memory and persistence-mongodb compile**

```bash
scripts/mvn-compile persistence-memory
scripts/mvn-compile persistence-mongodb
```

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "refactor(#275): move WorkItemPriority, WorkItemLabelRequest, WorkItemStatus, WorkItemCreateRequest to casehub-work-api

Four value types move from io.casehub.work.runtime.model to io.casehub.work.api.
All are pure Java with zero JPA/Quarkus deps. WorkItemCreateRequest gains
tenancyId field for multi-tenant SPI callers. Tests move with types.

Refs #275"
```

---

### Task 2: Create WorkItemRef, WorkItemEvent, and SPI interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/WorkItemRef.java`
- Create: `api/src/main/java/io/casehub/work/api/WorkItemEvent.java`
- Create: `api/src/main/java/io/casehub/work/api/spi/WorkItemCreator.java`
- Create: `api/src/main/java/io/casehub/work/api/spi/WorkItemLifecycle.java`
- Create: `api/src/test/java/io/casehub/work/api/WorkItemRefTest.java`
- Create: `api/src/test/java/io/casehub/work/api/WorkItemEventTest.java`

**Interfaces:**
- Consumes: `WorkItemStatus`, `WorkItemCreateRequest` from Task 1
- Produces: `WorkItemRef`, `WorkItemEvent`, `WorkItemCreator`, `WorkItemLifecycle` — used by Tasks 3-5 and the engine migration

- [ ] **Step 1: Write WorkItemRef test**

Create `api/src/test/java/io/casehub/work/api/WorkItemRefTest.java`:
```java
package io.casehub.work.api;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import org.junit.jupiter.api.Test;

class WorkItemRefTest {

    @Test
    void record_carriesAllFields() {
        final UUID id = UUID.randomUUID();
        final WorkItemRef ref = new WorkItemRef(id, WorkItemStatus.PENDING, "caller-1",
                "alice", "{}", "team-a", "approved", "tenant-1");

        assertThat(ref.id()).isEqualTo(id);
        assertThat(ref.status()).isEqualTo(WorkItemStatus.PENDING);
        assertThat(ref.callerRef()).isEqualTo("caller-1");
        assertThat(ref.assigneeId()).isEqualTo("alice");
        assertThat(ref.resolution()).isEqualTo("{}");
        assertThat(ref.candidateGroups()).isEqualTo("team-a");
        assertThat(ref.outcome()).isEqualTo("approved");
        assertThat(ref.tenancyId()).isEqualTo("tenant-1");
    }

    @Test
    void statusHelpers_delegateCorrectly() {
        final WorkItemRef active = new WorkItemRef(UUID.randomUUID(), WorkItemStatus.PENDING,
                null, null, null, null, null, null);
        final WorkItemRef terminal = new WorkItemRef(UUID.randomUUID(), WorkItemStatus.COMPLETED,
                null, null, null, null, null, null);

        assertThat(active.status().isActive()).isTrue();
        assertThat(active.status().isTerminal()).isFalse();
        assertThat(terminal.status().isTerminal()).isTrue();
        assertThat(terminal.status().isActive()).isFalse();
    }
}
```

- [ ] **Step 2: Run test — verify it fails (WorkItemRef doesn't exist)**

```bash
scripts/mvn-test api -Dtest=WorkItemRefTest
```
Expected: FAIL — `WorkItemRef` not found

- [ ] **Step 3: Create WorkItemRef**

Create `api/src/main/java/io/casehub/work/api/WorkItemRef.java`:
```java
package io.casehub.work.api;

import java.util.UUID;

public record WorkItemRef(
    UUID id,
    WorkItemStatus status,
    String callerRef,
    String assigneeId,
    String resolution,
    String candidateGroups,
    String outcome,
    String tenancyId
) {}
```

- [ ] **Step 4: Run test — verify it passes**

```bash
scripts/mvn-test api -Dtest=WorkItemRefTest
```
Expected: PASS

- [ ] **Step 5: Write WorkItemEvent test**

Create `api/src/test/java/io/casehub/work/api/WorkItemEventTest.java`:
```java
package io.casehub.work.api;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import org.junit.jupiter.api.Test;

class WorkItemEventTest {

    @Test
    void defaultMethods_delegateToRef() {
        final UUID id = UUID.randomUUID();
        final WorkItemRef ref = new WorkItemRef(id, WorkItemStatus.IN_PROGRESS, "caller-1",
                "bob", "{\"x\":1}", "team-b", "rejected", "tenant-2");

        final WorkItemEvent event = () -> ref;

        assertThat(event.workItemId()).isEqualTo(id);
        assertThat(event.status()).isEqualTo(WorkItemStatus.IN_PROGRESS);
        assertThat(event.callerRef()).isEqualTo("caller-1");
        assertThat(event.assigneeId()).isEqualTo("bob");
        assertThat(event.resolution()).isEqualTo("{\"x\":1}");
        assertThat(event.candidateGroups()).isEqualTo("team-b");
        assertThat(event.outcome()).isEqualTo("rejected");
        assertThat(event.tenancyId()).isEqualTo("tenant-2");
    }

    @Test
    void defaultMethods_handleNullRefFields() {
        final WorkItemRef ref = new WorkItemRef(UUID.randomUUID(), WorkItemStatus.PENDING,
                null, null, null, null, null, null);

        final WorkItemEvent event = () -> ref;

        assertThat(event.callerRef()).isNull();
        assertThat(event.assigneeId()).isNull();
        assertThat(event.resolution()).isNull();
    }
}
```

- [ ] **Step 6: Run test — verify it fails**

```bash
scripts/mvn-test api -Dtest=WorkItemEventTest
```
Expected: FAIL — `WorkItemEvent` not found

- [ ] **Step 7: Create WorkItemEvent**

Create `api/src/main/java/io/casehub/work/api/WorkItemEvent.java`:
```java
package io.casehub.work.api;

import java.util.UUID;

public interface WorkItemEvent {

    WorkItemRef ref();

    default UUID workItemId() { return ref().id(); }
    default WorkItemStatus status() { return ref().status(); }
    default String callerRef() { return ref().callerRef(); }
    default String assigneeId() { return ref().assigneeId(); }
    default String resolution() { return ref().resolution(); }
    default String candidateGroups() { return ref().candidateGroups(); }
    default String outcome() { return ref().outcome(); }
    default String tenancyId() { return ref().tenancyId(); }
}
```

- [ ] **Step 8: Run test — verify it passes**

```bash
scripts/mvn-test api -Dtest=WorkItemEventTest
```
Expected: PASS

- [ ] **Step 9: Create SPI interfaces**

Create directory: `api/src/main/java/io/casehub/work/api/spi/`

Create `api/src/main/java/io/casehub/work/api/spi/WorkItemCreator.java`:
```java
package io.casehub.work.api.spi;

import java.util.Optional;

import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemRef;

public interface WorkItemCreator {

    WorkItemRef create(WorkItemCreateRequest request);

    Optional<WorkItemRef> findByCallerRef(String callerRef);

    Optional<WorkItemRef> findActiveByCallerRef(String callerRef);
}
```

Create `api/src/main/java/io/casehub/work/api/spi/WorkItemLifecycle.java`:
```java
package io.casehub.work.api.spi;

import java.util.UUID;

public interface WorkItemLifecycle {

    void cancel(UUID id, String actorId, String reason);

    void complete(UUID id, String actorId, String resolution, String outcome,
                  String rationale, String planRef);

    default void complete(UUID id, String actorId, String resolution, String outcome) {
        complete(id, actorId, resolution, outcome, null, null);
    }
}
```

- [ ] **Step 10: Verify api compiles and all tests pass**

```bash
scripts/mvn-test api
```
Expected: all tests pass

- [ ] **Step 11: Install api**

```bash
scripts/mvn-install api
```

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "feat(#275): WorkItemRef, WorkItemEvent, WorkItemCreator, WorkItemLifecycle

WorkItemRef record — externally-visible WorkItem state snapshot.
WorkItemEvent interface — composes over WorkItemRef with default accessors.
WorkItemCreator SPI — create + findByCallerRef + findActiveByCallerRef.
WorkItemLifecycle SPI — cancel + complete (with rationale/planRef overload).
All in casehub-work-api, pure Java, Tier 1.

Refs #275"
```

---

### Task 3: Add findActiveByCallerRef to WorkItemStore + V38 index

**Files:**
- Modify: `runtime/.../repository/WorkItemStore.java` — add `findActiveByCallerRef` default method
- Modify: `runtime/.../repository/jpa/JpaWorkItemStore.java` — add indexed override
- Modify: `persistence-mongodb/.../MongoWorkItemStore.java` — add query override
- Create: `runtime/src/main/resources/db/work/migration/V38__caller_ref_index.sql`
- Modify: `runtime/.../service/WorkItemService.java` — add `tenancyId` mapping in `create()`, add `findActiveByCallerRef()` method

**Interfaces:**
- Consumes: `WorkItemStatus.TERMINAL_STATUSES` from Task 1
- Produces: `WorkItemStore.findActiveByCallerRef(String)`, `WorkItemService.findActiveByCallerRef(String)` — used by Task 4

- [ ] **Step 1: Add default method to WorkItemStore**

In `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemStore.java`, add after the `findByCallerRef` default method:

```java
    default Optional<WorkItem> findActiveByCallerRef(String callerRef) {
        return scanAll().stream()
                .filter(wi -> callerRef.equals(wi.callerRef))
                .filter(wi -> wi.status != null && wi.status.isActive())
                .findFirst();
    }
```

- [ ] **Step 2: Add indexed override in JpaWorkItemStore**

In `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java`, add after the `findByCallerRef` override:

```java
    @Override
    public Optional<WorkItem> findActiveByCallerRef(final String callerRef) {
        return withTenantQuery(() ->
                WorkItem.find("callerRef = ?1 AND status NOT IN (?2) AND tenancyId = ?3",
                        callerRef, WorkItemStatus.TERMINAL_STATUSES, currentPrincipal.tenancyId())
                        .firstResultOptional());
    }
```

- [ ] **Step 3: Add query override in MongoWorkItemStore**

In `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java`, add after the `findByCallerRef` override:

```java
    @Override
    public Optional<WorkItem> findActiveByCallerRef(final String callerRef) {
        final List<String> terminalNames = WorkItemStatus.TERMINAL_STATUSES.stream()
                .map(Enum::name).toList();
        final Document filter = new Document("callerRef", callerRef)
                .append("status", new Document("$nin", terminalNames))
                .append("tenancyId", currentPrincipal.tenancyId());
        final MongoWorkItemDocument doc = MongoWorkItemDocument.find(filter).firstResult();
        return Optional.ofNullable(doc).map(MongoWorkItemDocument::toDomain);
    }
```

- [ ] **Step 4: Create V38 migration**

Create `runtime/src/main/resources/db/work/migration/V38__caller_ref_index.sql`:
```sql
CREATE INDEX idx_work_item_caller_ref ON work_item (caller_ref);
```

- [ ] **Step 5: Add tenancyId mapping in WorkItemService.create()**

In `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`, in the `create()` method, add after `item.scope = request.scope;` (around line 121):

```java
        item.tenancyId = request.tenancyId;
```

- [ ] **Step 6: Add findActiveByCallerRef to WorkItemService**

In `WorkItemService.java`, add after the existing `findByCallerRef()` method:

```java
    public Optional<WorkItem> findActiveByCallerRef(final String callerRef) {
        return workItemStore.findActiveByCallerRef(callerRef);
    }
```

- [ ] **Step 7: Compile runtime**

```bash
scripts/mvn-compile runtime
```
Expected: compiles

- [ ] **Step 8: Run runtime tests**

```bash
scripts/mvn-test runtime
```
Expected: all pass (including Flyway picking up V38)

- [ ] **Step 9: Compile persistence modules**

```bash
scripts/mvn-compile persistence-memory
scripts/mvn-compile persistence-mongodb
```

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#275): findActiveByCallerRef + V38 caller_ref index

Add findActiveByCallerRef to WorkItemStore with linear-scan default.
JpaWorkItemStore uses indexed query (status NOT IN terminal).
MongoWorkItemStore uses $nin filter. V38 migration adds caller_ref index.
WorkItemService.create() now maps request.tenancyId to entity.
Supersedes #274 for callerRef-based lookups.

Refs #275"
```

---

### Task 4: WorkItemSpiAdapter + createFromTemplate + createGroup modification

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/spi/WorkItemSpiAdapter.java`
- Modify: `runtime/.../service/WorkItemTemplateService.java` — add `createFromTemplate(WorkItemCreateRequest)`
- Modify: `runtime/.../multiinstance/MultiInstanceSpawnService.java` — change `createGroup()` signature
- Create: `runtime/src/test/java/io/casehub/work/runtime/spi/WorkItemSpiAdapterTest.java`
- Create: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceCreateFromTemplateTest.java`

**Interfaces:**
- Consumes: `WorkItemCreator`, `WorkItemLifecycle`, `WorkItemRef` from Task 2; `WorkItemService.findByCallerRef()`, `WorkItemService.findActiveByCallerRef()` from Task 3
- Produces: `WorkItemSpiAdapter` CDI bean implementing `WorkItemCreator` + `WorkItemLifecycle` — injectable by external modules

- [ ] **Step 1: Write WorkItemSpiAdapter test**

Create `runtime/src/test/java/io/casehub/work/runtime/spi/WorkItemSpiAdapterTest.java`:
```java
package io.casehub.work.runtime.spi;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.util.Optional;
import java.util.UUID;

import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemRef;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.work.runtime.service.WorkItemTemplateService;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class WorkItemSpiAdapterTest {

    private WorkItemService workItemService;
    private WorkItemTemplateService workItemTemplateService;
    private WorkItemSpiAdapter adapter;

    @BeforeEach
    void setUp() {
        workItemService = mock(WorkItemService.class);
        workItemTemplateService = mock(WorkItemTemplateService.class);
        adapter = new WorkItemSpiAdapter(workItemService, workItemTemplateService);
    }

    @Test
    void create_noTemplateId_delegatesToWorkItemService() {
        final WorkItemCreateRequest request = WorkItemCreateRequest.builder()
                .title("test").createdBy("system").build();
        final WorkItem entity = new WorkItem();
        entity.id = UUID.randomUUID();
        entity.status = WorkItemStatus.PENDING;
        when(workItemService.create(request)).thenReturn(entity);

        final WorkItemRef ref = adapter.create(request);

        assertThat(ref.id()).isEqualTo(entity.id);
        assertThat(ref.status()).isEqualTo(WorkItemStatus.PENDING);
        verify(workItemTemplateService, never()).createFromTemplate(any());
    }

    @Test
    void create_withTemplateId_delegatesToTemplateService() {
        final UUID templateId = UUID.randomUUID();
        final WorkItemCreateRequest request = WorkItemCreateRequest.builder()
                .title("test").templateId(templateId).createdBy("system").build();
        final WorkItem entity = new WorkItem();
        entity.id = UUID.randomUUID();
        entity.status = WorkItemStatus.PENDING;
        when(workItemTemplateService.createFromTemplate(request)).thenReturn(entity);

        final WorkItemRef ref = adapter.create(request);

        assertThat(ref.id()).isEqualTo(entity.id);
        verify(workItemService, never()).create(any());
    }

    @Test
    void findByCallerRef_delegatesAndConverts() {
        final WorkItem entity = new WorkItem();
        entity.id = UUID.randomUUID();
        entity.status = WorkItemStatus.IN_PROGRESS;
        entity.callerRef = "case:123/pi:456";
        entity.assigneeId = "alice";
        when(workItemService.findByCallerRef("case:123/pi:456")).thenReturn(Optional.of(entity));

        final Optional<WorkItemRef> result = adapter.findByCallerRef("case:123/pi:456");

        assertThat(result).isPresent();
        assertThat(result.get().id()).isEqualTo(entity.id);
        assertThat(result.get().callerRef()).isEqualTo("case:123/pi:456");
        assertThat(result.get().assigneeId()).isEqualTo("alice");
    }

    @Test
    void findActiveByCallerRef_delegatesAndConverts() {
        final WorkItem entity = new WorkItem();
        entity.id = UUID.randomUUID();
        entity.status = WorkItemStatus.PENDING;
        when(workItemService.findActiveByCallerRef("ref-1")).thenReturn(Optional.of(entity));

        final Optional<WorkItemRef> result = adapter.findActiveByCallerRef("ref-1");

        assertThat(result).isPresent();
        assertThat(result.get().status()).isEqualTo(WorkItemStatus.PENDING);
    }

    @Test
    void cancel_delegatesToService() {
        final UUID id = UUID.randomUUID();
        final WorkItem entity = new WorkItem();
        entity.id = id;
        entity.status = WorkItemStatus.PENDING;
        when(workItemService.cancel(id, "system", "done")).thenReturn(entity);

        adapter.cancel(id, "system", "done");

        verify(workItemService).cancel(id, "system", "done");
    }

    @Test
    void cancel_idempotentOnTerminal() {
        final UUID id = UUID.randomUUID();
        final WorkItem entity = new WorkItem();
        entity.id = id;
        entity.status = WorkItemStatus.COMPLETED;
        when(workItemService.cancel(id, "system", "done"))
                .thenThrow(new IllegalStateException("Cannot cancel"));
        when(workItemService.findById(id)).thenReturn(Optional.of(entity));

        adapter.cancel(id, "system", "done");

        verify(workItemService).cancel(id, "system", "done");
    }

    @Test
    void cancel_rethrowsOnActiveNonCancellable() {
        final UUID id = UUID.randomUUID();
        final WorkItem entity = new WorkItem();
        entity.id = id;
        entity.status = WorkItemStatus.IN_PROGRESS;
        when(workItemService.cancel(id, "system", "done"))
                .thenThrow(new IllegalStateException("Cannot cancel"));
        when(workItemService.findById(id)).thenReturn(Optional.of(entity));

        assertThatThrownBy(() -> adapter.cancel(id, "system", "done"))
                .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void complete_idempotentOnTerminal() {
        final UUID id = UUID.randomUUID();
        final WorkItem entity = new WorkItem();
        entity.id = id;
        entity.status = WorkItemStatus.COMPLETED;
        when(workItemService.complete(eq(id), eq("alice"), eq("{}"), eq("approved"), any(), any()))
                .thenThrow(new IllegalStateException("Cannot complete"));
        when(workItemService.findById(id)).thenReturn(Optional.of(entity));

        adapter.complete(id, "alice", "{}", "approved", null, null);

        verify(workItemService).complete(id, "alice", "{}", "approved", null, null);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
scripts/mvn-test runtime -Dtest=WorkItemSpiAdapterTest
```
Expected: FAIL — `WorkItemSpiAdapter` not found

- [ ] **Step 3: Create WorkItemSpiAdapter**

Create `runtime/src/main/java/io/casehub/work/runtime/spi/WorkItemSpiAdapter.java`:
```java
package io.casehub.work.runtime.spi;

import java.util.Optional;
import java.util.UUID;

import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemRef;
import io.casehub.work.api.spi.WorkItemCreator;
import io.casehub.work.api.spi.WorkItemLifecycle;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.work.runtime.service.WorkItemTemplateService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class WorkItemSpiAdapter implements WorkItemCreator, WorkItemLifecycle {

    private final WorkItemService workItemService;
    private final WorkItemTemplateService workItemTemplateService;

    @Inject
    public WorkItemSpiAdapter(final WorkItemService workItemService,
                              final WorkItemTemplateService workItemTemplateService) {
        this.workItemService = workItemService;
        this.workItemTemplateService = workItemTemplateService;
    }

    @Override
    public WorkItemRef create(final WorkItemCreateRequest request) {
        final WorkItem item;
        if (request.templateId != null) {
            item = workItemTemplateService.createFromTemplate(request);
        } else {
            item = workItemService.create(request);
        }
        return toRef(item);
    }

    @Override
    public Optional<WorkItemRef> findByCallerRef(final String callerRef) {
        return workItemService.findByCallerRef(callerRef).map(WorkItemSpiAdapter::toRef);
    }

    @Override
    public Optional<WorkItemRef> findActiveByCallerRef(final String callerRef) {
        return workItemService.findActiveByCallerRef(callerRef).map(WorkItemSpiAdapter::toRef);
    }

    @Override
    public void cancel(final UUID id, final String actorId, final String reason) {
        try {
            workItemService.cancel(id, actorId, reason);
        } catch (final IllegalStateException e) {
            if (isTerminal(id)) return;
            throw e;
        }
    }

    @Override
    public void complete(final UUID id, final String actorId, final String resolution,
                         final String outcome, final String rationale, final String planRef) {
        try {
            workItemService.complete(id, actorId, resolution, outcome, rationale, planRef);
        } catch (final IllegalStateException e) {
            if (isTerminal(id)) return;
            throw e;
        }
    }

    private boolean isTerminal(final UUID id) {
        return workItemService.findById(id)
                .map(wi -> wi.status != null && wi.status.isTerminal())
                .orElse(false);
    }

    static WorkItemRef toRef(final WorkItem wi) {
        return new WorkItemRef(
                wi.id, wi.status, wi.callerRef, wi.assigneeId,
                wi.resolution, wi.candidateGroups, wi.outcome, wi.tenancyId);
    }
}
```

- [ ] **Step 4: Run adapter test — verify it passes**

```bash
scripts/mvn-test runtime -Dtest=WorkItemSpiAdapterTest
```
Expected: PASS

- [ ] **Step 5: Write createFromTemplate test**

Create `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceCreateFromTemplateTest.java` — a unit test verifying the merge logic. Test that request fields override template fields, that templateId resolution throws on not-found, and that the method delegates to `workItemService.create()` for simple templates.

The test needs Mockito since it mocks `WorkItemTemplateStore`, `WorkItemService`, `TemplateExpander`, and `MultiInstanceSpawnService`. Write tests for:
- Simple template: request title overrides template name
- Template not found: throws IllegalArgumentException
- Null request fields use template defaults
- Group expansion applies when template has excludedGroups

- [ ] **Step 6: Implement createFromTemplate on WorkItemTemplateService**

Add to `WorkItemTemplateService`:
```java
    @Transactional
    public WorkItem createFromTemplate(final WorkItemCreateRequest request) {
        final WorkItemTemplate template = templateStore.findById(request.templateId)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Template not found: " + request.templateId));

        final String expandedExcludedUsers = templateExpander.expandExcludedUsers(template);

        final WorkItemCreateRequest merged = mergeRequestWithTemplate(template, request, expandedExcludedUsers);

        if (template.instanceCount != null) {
            return multiInstanceSpawnService.get()
                    .createGroup(merged, template, expandedExcludedUsers);
        }

        final WorkItem workItem = workItemService.create(merged);

        final List<WorkItemLabel> labels = parseLabels(template);
        WorkItem result = workItem;
        for (final WorkItemLabel label : labels) {
            result = workItemService.addLabel(result.id, label.path, label.appliedBy);
        }
        return result;
    }

    static WorkItemCreateRequest mergeRequestWithTemplate(
            final WorkItemTemplate template, final WorkItemCreateRequest request,
            final String expandedExcludedUsers) {
        return WorkItemCreateRequest.builder()
                .title(request.title != null ? request.title : template.name)
                .description(request.description != null ? request.description : template.description)
                .category(request.category != null ? request.category : template.category)
                .formKey(request.formKey != null ? request.formKey : template.formKey)
                .priority(request.priority != null ? request.priority : template.priority)
                .assigneeId(request.assigneeId != null ? request.assigneeId : null)
                .candidateGroups(request.candidateGroups != null ? request.candidateGroups : template.candidateGroups)
                .candidateUsers(request.candidateUsers != null ? request.candidateUsers : template.candidateUsers)
                .requiredCapabilities(request.requiredCapabilities != null ? request.requiredCapabilities : template.requiredCapabilities)
                .createdBy(request.createdBy)
                .payload(request.payload != null ? request.payload : template.defaultPayload)
                .claimDeadline(request.claimDeadline)
                .expiresAt(request.expiresAt)
                .followUpDate(request.followUpDate)
                .confidenceScore(request.confidenceScore)
                .callerRef(request.callerRef)
                .claimDeadlineBusinessHours(request.claimDeadlineBusinessHours)
                .expiresAtBusinessHours(request.expiresAtBusinessHours)
                .templateId(request.templateId)
                .permittedOutcomes(request.permittedOutcomes != null
                        ? request.permittedOutcomes
                        : WorkItemTemplateService.decodeOutcomes(template.permittedOutcomes))
                .inputDataSchema(request.inputDataSchema != null ? request.inputDataSchema : template.inputDataSchema)
                .outputDataSchema(request.outputDataSchema != null ? request.outputDataSchema : template.outputDataSchema)
                .excludedUsers(expandedExcludedUsers != null ? expandedExcludedUsers : request.excludedUsers)
                .scope(request.scope)
                .tenancyId(request.tenancyId)
                .build();
    }
```

Note: the exact template field names must be verified against `WorkItemTemplate` entity fields and `toCreateRequest()`. The static `mergeRequestWithTemplate` is testable without CDI.

- [ ] **Step 7: Modify MultiInstanceSpawnService.createGroup signature**

Change `createGroup` from:
```java
public WorkItem createGroup(final WorkItemTemplate template, final String expandedExcludedUsers,
        final String titleOverride, final String createdBy, final String callerRef)
```
To:
```java
public WorkItem createGroup(final WorkItemCreateRequest mergedRequest,
        final WorkItemTemplate template, final String expandedExcludedUsers)
```

Update the method body: replace `buildParentRequest(template, ...)` call with using `mergedRequest` directly for the parent. Children still use `buildChildRequest(template, ...)` for per-instance variation, but inherit `scope`, `tenancyId`, and `permittedOutcomes` from `mergedRequest`.

Update the one call site in `WorkItemTemplateService.instantiate()` to pass the appropriate arguments.

- [ ] **Step 8: Run tests**

```bash
scripts/mvn-test runtime -Dtest=WorkItemSpiAdapterTest
scripts/mvn-test runtime -Dtest=WorkItemTemplateServiceCreateFromTemplateTest
scripts/mvn-test runtime
```
Expected: all pass

- [ ] **Step 9: Install runtime**

```bash
scripts/mvn-install runtime
```

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#275): WorkItemSpiAdapter + createFromTemplate + createGroup merge

WorkItemSpiAdapter implements WorkItemCreator + WorkItemLifecycle via hexagonal
adapter pattern. Routes template creation to WorkItemTemplateService.createFromTemplate()
which merges request overrides with template defaults before create(). Fixes
post-mutation corruption: events/assignment/timers now use correct values.
MultiInstanceSpawnService.createGroup() accepts merged request. Idempotent
cancel/complete via exception catch + terminal check.

Refs #275"
```

---

### Task 5: WorkItemLifecycleEvent implements WorkItemEvent + wire format

**Files:**
- Modify: `runtime/.../event/WorkItemLifecycleEvent.java` — implement `WorkItemEvent`, store fields independently, extend `fromWire()`
- Modify: `postgres-broadcaster/.../WorkItemEventPayload.java` — add 4 fields
- Modify: `runtime/src/test/.../event/WorkItemLifecycleEventTest.java` — add tests for `ref()` and wire roundtrip

**Interfaces:**
- Consumes: `WorkItemEvent`, `WorkItemRef` from Task 2
- Produces: `WorkItemLifecycleEvent implements WorkItemEvent` — engine adapter can observe `WorkItemEvent` (api) instead of `WorkItemLifecycleEvent` (runtime)

- [ ] **Step 1: Write tests for WorkItemEvent implementation**

Add to `runtime/src/test/java/io/casehub/work/runtime/event/WorkItemLifecycleEventTest.java`:

```java
@Test
void ref_returnsWorkItemRefFromEntity() {
    final WorkItem workItem = new WorkItem();
    workItem.id = UUID.randomUUID();
    workItem.status = WorkItemStatus.IN_PROGRESS;
    workItem.callerRef = "case:1/pi:2";
    workItem.assigneeId = "alice";
    workItem.resolution = "{}";
    workItem.candidateGroups = "team-a";
    workItem.outcome = "approved";
    workItem.tenancyId = "tenant-1";

    final WorkItemLifecycleEvent event = WorkItemLifecycleEvent.of("COMPLETED", workItem, "alice", null);
    final WorkItemRef ref = event.ref();

    assertThat(ref.id()).isEqualTo(workItem.id);
    assertThat(ref.callerRef()).isEqualTo("case:1/pi:2");
    assertThat(ref.assigneeId()).isEqualTo("alice");
    assertThat(ref.resolution()).isEqualTo("{}");
    assertThat(ref.candidateGroups()).isEqualTo("team-a");
    assertThat(ref.tenancyId()).isEqualTo("tenant-1");
}

@Test
void ref_fromWireEvent_returnsWorkItemRefFromStoredFields() {
    final UUID id = UUID.randomUUID();
    final WorkItemLifecycleEvent wireEvent = WorkItemLifecycleEvent.fromWire(
            "io.casehub.work.workitem.completed", "/workitems/" + id, id.toString(),
            id, WorkItemStatus.COMPLETED, Instant.now(), "alice", null, null, null,
            "approved", "tenant-1",
            "case:1/pi:2", "alice", "{}", "team-a");

    final WorkItemRef ref = wireEvent.ref();

    assertThat(ref.id()).isEqualTo(id);
    assertThat(ref.callerRef()).isEqualTo("case:1/pi:2");
    assertThat(ref.assigneeId()).isEqualTo("alice");
    assertThat(ref.resolution()).isEqualTo("{}");
    assertThat(ref.candidateGroups()).isEqualTo("team-a");
}
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
scripts/mvn-test runtime -Dtest=WorkItemLifecycleEventTest
```
Expected: FAIL — `ref()` method not found, `fromWire()` wrong arity

- [ ] **Step 3: Modify WorkItemLifecycleEvent**

In `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEvent.java`:

1. Add `implements WorkItemEvent` to the class declaration
2. Add 4 new stored fields: `callerRef`, `assigneeId`, `resolution`, `candidateGroups`
3. Update both constructors to accept and store these fields
4. For the `of()` factory methods (local events): read from the embedded `workItem`
5. Update `fromWire()` to accept 4 additional parameters and store them
6. Implement `ref()`:

```java
@JsonIgnore
@Override
public WorkItemRef ref() {
    if (workItem != null) {
        return new WorkItemRef(workItemId, status, workItem.callerRef, workItem.assigneeId,
                workItem.resolution, workItem.candidateGroups, outcome, tenancyId);
    }
    return new WorkItemRef(workItemId, status, callerRef, assigneeId,
            resolution, candidateGroups, outcome, tenancyId);
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
scripts/mvn-test runtime -Dtest=WorkItemLifecycleEventTest
```
Expected: PASS

- [ ] **Step 5: Update WorkItemEventPayload**

In `postgres-broadcaster/src/main/java/io/casehub/work/postgres/broadcaster/WorkItemEventPayload.java`:

Add 4 fields to the record:
```java
@JsonProperty("callerRef") String callerRef,
@JsonProperty("assigneeId") String assigneeId,
@JsonProperty("resolution") String resolution,
@JsonProperty("candidateGroups") String candidateGroups
```

Update `from()`:
```java
static WorkItemEventPayload from(final WorkItemLifecycleEvent event) {
    return new WorkItemEventPayload(
            event.type(), event.sourceUri(), event.subject(),
            event.workItemId(), event.status(), event.occurredAt(),
            event.actor(), event.detail(), event.rationale(), event.planRef(),
            event.outcome(), event.tenancyId(),
            event.callerRef(), event.assigneeId(), event.resolution(), event.candidateGroups());
}
```

Update `toEvent()`:
```java
WorkItemLifecycleEvent toEvent() {
    return WorkItemLifecycleEvent.fromWire(type, source, subject,
            workItemId, status, occurredAt, actor, detail, rationale, planRef, outcome, tenancyId,
            callerRef, assigneeId, resolution, candidateGroups);
}
```

- [ ] **Step 6: Compile postgres-broadcaster**

```bash
scripts/mvn-compile postgres-broadcaster
```

- [ ] **Step 7: Run full test suite for affected modules**

```bash
scripts/mvn-test runtime
scripts/mvn-test postgres-broadcaster
```
Expected: all pass

- [ ] **Step 8: Install all modules**

```bash
scripts/mvn-install api
scripts/mvn-install runtime
```

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#275): WorkItemLifecycleEvent implements WorkItemEvent + wire format

WorkItemLifecycleEvent implements WorkItemEvent via ref() composition.
Local events read from embedded entity; wire events read from stored fields.
fromWire() extended with callerRef, assigneeId, resolution, candidateGroups.
WorkItemEventPayload gains matching fields for distributed broadcasting.

Refs #275"
```

---

### Task 6: Engine work-adapter migration (cross-repo issue)

**This task is cross-repo work** — the engine is at `casehub-engine`, not `casehub-work`. Per session repo boundary rules, this task files a detailed issue on the engine repo rather than implementing in this session.

**Files:**
- None modified in this session

- [ ] **Step 1: File engine migration issue**

Create issue on `casehubio/engine` with the full migration spec from the design document's "Engine Work-Adapter Migration" section. Include:
- POM change: `casehub-work` → `casehub-work-api` (compile)
- Per-file migration table (HumanTaskScheduleHandler, ActionGateCancelledHandler, WorkItemLifecycleAdapter, ActionGateCompletionApplier, PlanItemCompletionApplier, HumanTaskRecoveryService)
- Import changes for all moved types
- Test scope: keeps `casehub-work` + `casehub-work-persistence-memory`

```bash
gh issue create --repo casehubio/engine \
  --title "refactor: migrate work-adapter from casehub-work to casehub-work-api (SPI)" \
  --body "Blocked by casehubio/work#275.

## Context
casehub-work#275 extracts WorkItemCreator + WorkItemLifecycle SPIs into casehub-work-api. The engine work-adapter's compile dependency on casehub-work (runtime) can now be replaced by casehub-work-api.

## Changes
[full migration table from spec]

## Dependency
Requires casehub-work 0.2-SNAPSHOT with #275 merged and published."
```

- [ ] **Step 2: Commit — no code changes, just the issue reference**

No commit needed — the issue is tracked externally.

---

## Cross-Repo Notes

After all casehub-work tasks are complete and merged:
1. Publish `casehub-work-api` and `casehub-work` 0.2-SNAPSHOT to GitHub Packages
2. Engine work-adapter migration (filed issue in Task 6) can proceed independently
3. casehub-desiredstate#43 is unblocked — can depend on `casehub-work-api` and inject `WorkItemCreator`
