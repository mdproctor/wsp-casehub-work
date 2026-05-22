# Subprocess Spawning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement subprocess spawning (#105) — a caller-driven API that creates child WorkItems from templates, wires them via PART_OF, stores an opaque `callerRef` for routing, fires per-child lifecycle events, and wires the ledger causal chain.

**Architecture:** quarkus-work is the pure primitive layer — it executes spawning mechanics and fires events; it makes no decisions about what events mean. CaseHub embeds its routing key in `callerRef`; the ledger traces causality via `causedByEntryId`. All orchestration (completion rollup, auto-trigger) is explicitly absent.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache ORM (H2 in tests), JAX-RS/RestAssured, Flyway, JUnit 5, AssertJ, CDI events.

---

## File Map

**quarkus-work-api** (new):
- `src/main/java/io/quarkiverse/work/api/SpawnPort.java`
- `src/main/java/io/quarkiverse/work/api/SpawnRequest.java`
- `src/main/java/io/quarkiverse/work/api/ChildSpec.java`
- `src/main/java/io/quarkiverse/work/api/SpawnResult.java`
- `src/main/java/io/quarkiverse/work/api/SpawnedChild.java`

**quarkus-work-api** (modify):
- `src/main/java/io/quarkiverse/work/api/WorkEventType.java` — add `SPAWNED`

**runtime** (new):
- `src/main/java/io/quarkiverse/work/runtime/model/WorkItemSpawnGroup.java`
- `src/main/java/io/quarkiverse/work/runtime/service/WorkItemSpawnService.java`
- `src/main/java/io/quarkiverse/work/runtime/api/WorkItemSpawnResource.java`
- `src/main/resources/db/migration/V17__caller_ref.sql`
- `src/main/resources/db/migration/V18__spawn_group.sql`
- `src/test/java/io/quarkiverse/work/runtime/spawn/WorkItemSpawnResourceTest.java`
- `src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCallerRefTest.java`
- `src/test/java/io/quarkiverse/work/runtime/spawn/SpawnIdempotencyTest.java`
- `src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCascadeCancelTest.java`
- `src/test/java/io/quarkiverse/work/runtime/spawn/SpawnE2ETest.java`
- `src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCorrectnessTest.java`

**runtime** (modify):
- `src/main/java/io/quarkiverse/work/runtime/model/WorkItem.java` — add `callerRef`

**quarkus-work-ledger** (modify):
- `src/main/java/io/quarkiverse/work/ledger/service/LedgerEventCapture.java` — add spawned to EVENT_META, wire `causedByEntryId`
- `src/test/java/io/quarkiverse/work/ledger/LedgerIntegrationTest.java` — add spawn causal chain test

**quarkus-work-examples** (new):
- `src/main/java/io/quarkiverse/work/examples/spawn/SpawnScenarioResource.java`
- `src/test/java/io/quarkiverse/work/examples/spawn/SpawnScenarioTest.java`

---

## Task 1: SPI foundation in quarkus-work-api

**Issues:** #128, #129  
**Files:**
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SpawnPort.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SpawnRequest.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/ChildSpec.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SpawnResult.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SpawnedChild.java`
- Modify: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkEventType.java`

- [ ] **Step 1.1: Add SPAWNED to WorkEventType**

Edit `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkEventType.java`. Add `SPAWNED` before `ESCALATED`:

```java
package io.quarkiverse.work.api;

public enum WorkEventType {
    CREATED,
    ASSIGNED,
    STARTED,
    COMPLETED,
    REJECTED,
    DELEGATED,
    RELEASED,
    SUSPENDED,
    RESUMED,
    CANCELLED,
    EXPIRED,
    /** Claim deadline passed without the work being claimed. */
    CLAIM_EXPIRED,
    /** Child WorkItems were spawned from this work unit. */
    SPAWNED,
    ESCALATED
}
```

- [ ] **Step 1.2: Create SpawnPort SPI**

Create `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SpawnPort.java`:

```java
package io.quarkiverse.work.api;

import java.util.UUID;

/**
 * SPI for spawning child work units from a parent.
 * Implementations live in domain-specific modules (quarkus-work runtime for WorkItems).
 * quarkus-work fires events and wires PART_OF relations; it makes no decisions
 * about what child completion means.
 */
public interface SpawnPort {
    SpawnResult spawn(SpawnRequest request);
    void cancelGroup(UUID groupId, boolean cascadeChildren);
}
```

- [ ] **Step 1.3: Create SpawnRequest record**

Create `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SpawnRequest.java`:

```java
package io.quarkiverse.work.api;

import java.util.List;
import java.util.UUID;

/**
 * Request to spawn a group of child work units from a parent.
 *
 * @param parentId       the UUID of the parent WorkItem
 * @param idempotencyKey caller-supplied deduplication key — a second call with the
 *                       same key on the same parent returns the existing group (HTTP 200)
 *                       without creating duplicate children
 * @param children       one entry per child to spawn; must not be null or empty
 */
public record SpawnRequest(
        UUID parentId,
        String idempotencyKey,
        List<ChildSpec> children) {
}
```

- [ ] **Step 1.4: Create ChildSpec record**

Create `quarkus-work-api/src/main/java/io/quarkiverse/work/api/ChildSpec.java`:

```java
package io.quarkiverse.work.api;

import java.util.Map;
import java.util.UUID;

/**
 * Specification for a single child to spawn.
 *
 * @param templateId the UUID of the WorkItemTemplate to instantiate
 * @param callerRef  opaque routing key stored on the spawned WorkItem and echoed
 *                   in every lifecycle event — never interpreted by quarkus-work;
 *                   CaseHub embeds its planItemId here for completion routing
 * @param overrides  fields that override the template defaults for this child only
 *                   (e.g. {@code "candidateGroups"} → {@code "fraud-team"})
 */
public record ChildSpec(
        UUID templateId,
        String callerRef,
        Map<String, Object> overrides) {
}
```

- [ ] **Step 1.5: Create SpawnedChild record**

Create `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SpawnedChild.java`:

```java
package io.quarkiverse.work.api;

import java.util.UUID;

/**
 * Result for a single spawned child.
 *
 * @param workItemId the UUID of the created child WorkItem
 * @param callerRef  echoed from {@link ChildSpec#callerRef()} — confirms the
 *                   association between the caller's routing key and the new WorkItem
 */
public record SpawnedChild(UUID workItemId, String callerRef) {
}
```

- [ ] **Step 1.6: Create SpawnResult record**

Create `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SpawnResult.java`:

```java
package io.quarkiverse.work.api;

import java.util.List;
import java.util.UUID;

/**
 * Result of a spawn operation.
 *
 * @param groupId  UUID of the {@code WorkItemSpawnGroup} tracking this spawn batch
 * @param children one entry per child created, in the same order as the request
 * @param created  true if a new group was created; false if the idempotencyKey matched
 *                 an existing group (the existing group is returned, no children created)
 */
public record SpawnResult(UUID groupId, List<SpawnedChild> children, boolean created) {
}
```

- [ ] **Step 1.7: Compile quarkus-work-api**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl quarkus-work-api -q
```

Expected: BUILD SUCCESS with no output.

- [ ] **Step 1.8: Run quarkus-work-api tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api -q
```

Expected: BUILD SUCCESS. All existing tests pass.

- [ ] **Step 1.9: Commit**

```bash
git add quarkus-work-api/src/main/java/io/quarkiverse/work/api/
git commit -m "feat(api): add WorkEventType.SPAWNED and SpawnPort SPI

SpawnPort, SpawnRequest, ChildSpec, SpawnResult, SpawnedChild — generic
Work-level spawn SPI. callerRef is opaque to quarkus-work; CaseHub
embeds its planItemId for completion routing.

Closes #128
Closes #129
Refs #105

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: callerRef on WorkItem + Flyway V17

**Issue:** #127  
**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItem.java`
- Create: `runtime/src/main/resources/db/migration/V17__caller_ref.sql`

- [ ] **Step 2.1: Write V17 migration**

Create `runtime/src/main/resources/db/migration/V17__caller_ref.sql`:

```sql
ALTER TABLE work_item ADD COLUMN caller_ref VARCHAR(512);
```

- [ ] **Step 2.2: Add callerRef field to WorkItem**

In `runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItem.java`, add after the `confidenceScore` field (at the end of the AI metadata section, before `@PrePersist`):

```java
    // -------------------------------------------------------------------------
    // Spawn routing
    // -------------------------------------------------------------------------

    /**
     * Opaque caller-supplied routing key set at spawn time.
     * quarkus-work stores and echoes this in every lifecycle event; it never
     * interprets it. CaseHub embeds its {@code caseId:planItemId} here so that
     * child completion events can be routed back to the right PlanItem without
     * a query. Null for WorkItems not created via spawn.
     */
    @Column(name = "caller_ref", length = 512)
    public String callerRef;
```

- [ ] **Step 2.3: Write failing test**

Create `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCallerRefTest.java`:

```java
package io.quarkiverse.work.runtime.spawn;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class SpawnCallerRefTest {

    @Test
    void callerRef_roundTrips_whenSetOnCreate() {
        // Create a WorkItem with callerRef directly via the create endpoint
        final var body = Map.of(
                "title", "callerRef round-trip test",
                "category", "test",
                "createdBy", "test-system",
                "callerRef", "case:loan-123/pi:credit-1");

        final String id = given()
                .contentType(ContentType.JSON)
                .body(body)
                .when().post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        final String fetched = given()
                .when().get("/workitems/" + id)
                .then().statusCode(200)
                .extract().path("callerRef");

        assertThat(fetched).isEqualTo("case:loan-123/pi:credit-1");
    }

    @Test
    void callerRef_null_whenNotProvided() {
        final var body = Map.of(
                "title", "no callerRef",
                "category", "test",
                "createdBy", "test-system");

        final String id = given()
                .contentType(ContentType.JSON)
                .body(body)
                .when().post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        final Object fetched = given()
                .when().get("/workitems/" + id)
                .then().statusCode(200)
                .extract().path("callerRef");

        assertThat(fetched).isNull();
    }

    @Test
    void callerRef_512chars_roundTrips() {
        final String longRef = "x".repeat(512);
        final var body = Map.of(
                "title", "long callerRef test",
                "category", "test",
                "createdBy", "test-system",
                "callerRef", longRef);

        final String id = given()
                .contentType(ContentType.JSON)
                .body(body)
                .when().post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        final String fetched = given()
                .when().get("/workitems/" + id)
                .then().statusCode(200)
                .extract().path("callerRef");

        assertThat(fetched).isEqualTo(longRef);
    }
}
```

- [ ] **Step 2.4: Run the failing test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SpawnCallerRefTest -pl runtime -q 2>&1 | tail -20
```

Expected: FAIL — `callerRef` not in the create request or response yet.

- [ ] **Step 2.5: Wire callerRef through CreateWorkItemRequest and WorkItemMapper**

Find `CreateWorkItemRequest` record in `runtime/src/main/java/io/quarkiverse/work/runtime/api/WorkItemResource.java` (it is defined as an inner record). Add `callerRef` field.

Also find `WorkItemMapper` class or the `toServiceRequest` method. Add `callerRef` mapping through to `WorkItemCreateRequest` and finally to the WorkItem entity in `WorkItemService.create()`.

**WorkItemResource.java** — find the `CreateWorkItemRequest` record and add `callerRef`:
```java
public record CreateWorkItemRequest(
        String title,
        String description,
        String category,
        String formKey,
        String assigneeId,
        String owner,
        String candidateGroups,
        String candidateUsers,
        String requiredCapabilities,
        String createdBy,
        WorkItemPriority priority,
        Instant expiresAt,
        Instant claimDeadline,
        Instant followUpDate,
        String payload,
        Double confidenceScore,
        String callerRef  // ADD THIS
) {}
```

**WorkItemMapper** — find `toServiceRequest` and pass callerRef:
```java
// In the mapping from CreateWorkItemRequest → WorkItemCreateRequest:
// add: .callerRef(request.callerRef())
// (exact syntax depends on whether WorkItemCreateRequest is a builder or record)
```

**WorkItemService.create()** — find where WorkItem fields are set from the request and add:
```java
workItem.callerRef = request.callerRef();
```

**WorkItemMapper.toResponse()** — add `callerRef` to the response map so the REST response includes it:
```java
// In whatever builds the response map/DTO:
map.put("callerRef", wi.callerRef);
```

> **Note:** Read each of these files before editing. Follow the exact patterns already used — do not invent new patterns.

- [ ] **Step 2.6: Run the test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SpawnCallerRefTest -pl runtime -q 2>&1 | tail -10
```

Expected: 3 tests pass.

- [ ] **Step 2.7: Run full runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 2.8: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItem.java
git add runtime/src/main/resources/db/migration/V17__caller_ref.sql
git add runtime/src/main/java/io/quarkiverse/work/runtime/api/
git add runtime/src/main/java/io/quarkiverse/work/runtime/service/
git add runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCallerRefTest.java
git commit -m "feat(runtime): add callerRef to WorkItem + Flyway V17 migration

Opaque routing key stored on WorkItem at spawn time, echoed in every
lifecycle event. CaseHub embeds its planItemId for routing child
completion back to the right PlanItem without a query.

Closes #127
Refs #105

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: WorkItemSpawnGroup entity + Flyway V18

**Issue:** #131  
**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemSpawnGroup.java`
- Create: `runtime/src/main/resources/db/migration/V18__spawn_group.sql`

- [ ] **Step 3.1: Write V18 migration**

Create `runtime/src/main/resources/db/migration/V18__spawn_group.sql`:

```sql
CREATE TABLE work_item_spawn_group (
    id              UUID        NOT NULL,
    parent_id       UUID        NOT NULL,
    idempotency_key VARCHAR(255) NOT NULL,
    created_at      TIMESTAMP   NOT NULL,
    CONSTRAINT pk_work_item_spawn_group PRIMARY KEY (id),
    CONSTRAINT uq_spawn_group_idempotency UNIQUE (parent_id, idempotency_key)
);
```

- [ ] **Step 3.2: Create WorkItemSpawnGroup entity**

Create `runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemSpawnGroup.java`:

```java
package io.quarkiverse.work.runtime.model;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * Tracks a batch of child WorkItems spawned together from a common parent.
 *
 * <h2>Purpose</h2>
 * <p>
 * Used for two things only:
 * <ol>
 * <li>Idempotency — the unique constraint on {@code (parent_id, idempotency_key)}
 *     ensures a retried spawn call returns the existing group without creating
 *     duplicate children.</li>
 * <li>Group membership query — callers can retrieve all children by querying
 *     PART_OF relations with {@code targetId = parentId} that were created after
 *     {@link #createdAt}; or via {@code GET /workitems/{parentId}/children}.</li>
 * </ol>
 *
 * <h2>No completion state</h2>
 * <p>
 * This entity does NOT track completion state, completion policy, or group status.
 * That is the caller's concern (CaseHub's {@code Stage.requiredItemIds};
 * or the application's own CDI observer). quarkus-work fires per-child
 * {@link io.quarkiverse.work.runtime.event.WorkItemLifecycleEvent}s — the caller
 * decides what they mean.
 */
@Entity
@Table(name = "work_item_spawn_group", uniqueConstraints = {
        @UniqueConstraint(name = "uq_spawn_group_idempotency",
                columnNames = { "parent_id", "idempotency_key" })
})
public class WorkItemSpawnGroup extends PanacheEntityBase {

    @Id
    public UUID id;

    /** The parent WorkItem that owns this spawn group. */
    @Column(name = "parent_id", nullable = false)
    public UUID parentId;

    /**
     * Caller-supplied deduplication key. A second spawn call with the same
     * {@code (parentId, idempotencyKey)} returns this group instead of creating a new one.
     */
    @Column(name = "idempotency_key", nullable = false, length = 255)
    public String idempotencyKey;

    /** When this group was created. */
    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
        if (createdAt == null) {
            createdAt = Instant.now();
        }
    }

    /** Find an existing group by parent + idempotency key. */
    public static WorkItemSpawnGroup findByParentAndKey(
            final UUID parentId, final String idempotencyKey) {
        return find("parentId = ?1 AND idempotencyKey = ?2", parentId, idempotencyKey)
                .firstResult();
    }

    /** All groups spawned from a parent, newest first. */
    public static List<WorkItemSpawnGroup> findByParentId(final UUID parentId) {
        return list("parentId = ?1 ORDER BY createdAt DESC", parentId);
    }
}
```

- [ ] **Step 3.3: Compile runtime**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 3.4: Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS. Flyway applies V18 in the test database.

- [ ] **Step 3.5: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemSpawnGroup.java
git add runtime/src/main/resources/db/migration/V18__spawn_group.sql
git commit -m "feat(runtime): add WorkItemSpawnGroup entity + Flyway V18

Minimal tracking entity for idempotency and group membership queries.
No completion state — that belongs to the caller (CaseHub Stage,
or application CDI observer).

Refs #131
Refs #105

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: WorkItemSpawnService implementation

**Issues:** #130, #131  
**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/service/WorkItemSpawnService.java`

- [ ] **Step 4.1: Write failing unit tests for spawn mechanics**

Create `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/WorkItemSpawnResourceTest.java` with just the first test (others come in Task 6):

```java
package io.quarkiverse.work.runtime.spawn;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.restassured.response.Response;

@QuarkusTest
class WorkItemSpawnResourceTest {

    @Test
    void spawn_createsChildren_withPartOfLinks() {
        // Create templates
        final String tmplId1 = createTemplate("credit-check");
        final String tmplId2 = createTemplate("fraud-check");

        // Create parent
        final String parentId = createWorkItem("loan-application");

        // Spawn
        final var spawnBody = Map.of(
                "idempotencyKey", "test-spawn-" + UUID.randomUUID(),
                "children", List.of(
                        Map.of("templateId", tmplId1, "callerRef", "case:l1/pi:c1"),
                        Map.of("templateId", tmplId2, "callerRef", "case:l1/pi:f2")));

        final Response response = given()
                .contentType(ContentType.JSON)
                .body(spawnBody)
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201)
                .extract().response();

        assertThat(response.jsonPath().getString("groupId")).isNotNull();
        final List<Map<String, Object>> children = response.jsonPath().getList("children");
        assertThat(children).hasSize(2);
        assertThat(children.get(0).get("callerRef")).isEqualTo("case:l1/pi:c1");
        assertThat(children.get(1).get("callerRef")).isEqualTo("case:l1/pi:f2");

        // Verify PART_OF links
        final List<Map<String, Object>> childList = given()
                .when().get("/workitems/" + parentId + "/children")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        assertThat(childList).hasSize(2);

        // Verify callerRef stored on children
        final String child1Id = (String) children.get(0).get("workItemId");
        final String fetchedRef = given()
                .when().get("/workitems/" + child1Id)
                .then().statusCode(200)
                .extract().path("callerRef");
        assertThat(fetchedRef).isEqualTo("case:l1/pi:c1");
    }

    // --- helpers ---

    private String createTemplate(final String name) {
        return given()
                .contentType(ContentType.JSON)
                .body(Map.of("name", name, "category", name, "createdBy", "test"))
                .when().post("/workitem-templates")
                .then().statusCode(201)
                .extract().path("id");
    }

    private String createWorkItem(final String category) {
        return given()
                .contentType(ContentType.JSON)
                .body(Map.of("title", "parent-" + category, "category", category, "createdBy", "test"))
                .when().post("/workitems")
                .then().statusCode(201)
                .extract().path("id");
    }
}
```

- [ ] **Step 4.2: Run the failing test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemSpawnResourceTest -pl runtime -q 2>&1 | tail -15
```

Expected: FAIL — endpoint does not exist yet (404 or method not allowed).

- [ ] **Step 4.3: Implement WorkItemSpawnService**

Create `runtime/src/main/java/io/quarkiverse/work/runtime/service/WorkItemSpawnService.java`:

```java
package io.quarkiverse.work.runtime.service;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.api.ChildSpec;
import io.quarkiverse.work.api.SpawnPort;
import io.quarkiverse.work.api.SpawnRequest;
import io.quarkiverse.work.api.SpawnResult;
import io.quarkiverse.work.api.SpawnedChild;
import io.quarkiverse.work.runtime.event.WorkItemEventBroadcaster;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemRelation;
import io.quarkiverse.work.runtime.model.WorkItemRelationType;
import io.quarkiverse.work.runtime.model.WorkItemSpawnGroup;
import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.repository.WorkItemStore;

/**
 * Implements {@link SpawnPort} for WorkItems.
 *
 * <p>
 * Creates child WorkItems from templates, wires PART_OF relations (child → parent),
 * stores the opaque {@code callerRef} on each child, fires {@code SPAWNED} on the parent,
 * and records the group for idempotency. Makes no decisions about what child completion
 * means — that is the caller's responsibility.
 */
@ApplicationScoped
public class WorkItemSpawnService implements SpawnPort {

    @Inject
    WorkItemStore workItemStore;

    @Inject
    WorkItemService workItemService;

    @Inject
    WorkItemEventBroadcaster broadcaster;

    /**
     * Spawn child WorkItems from templates and wire them to the parent via PART_OF.
     *
     * <p>
     * If {@code request.idempotencyKey()} matches an existing group under the same parent,
     * the existing group is returned with {@code created=false} and no children are created.
     *
     * @throws IllegalArgumentException if parent not found, children list empty, or templateId invalid
     */
    @Override
    @Transactional
    public SpawnResult spawn(final SpawnRequest request) {
        if (request.children() == null || request.children().isEmpty()) {
            throw new IllegalArgumentException("children must not be empty");
        }
        if (request.idempotencyKey() == null || request.idempotencyKey().isBlank()) {
            throw new IllegalArgumentException("idempotencyKey is required");
        }

        // Verify parent exists
        final WorkItem parent = workItemStore.get(request.parentId())
                .orElseThrow(() -> new WorkItemNotFoundException(request.parentId()));

        // Reject spawn on terminal parents
        if (parent.status != null && parent.status.isTerminal()) {
            throw new IllegalStateException(
                    "Cannot spawn children from terminal WorkItem " + request.parentId());
        }

        // Idempotency check
        final WorkItemSpawnGroup existing = WorkItemSpawnGroup
                .findByParentAndKey(request.parentId(), request.idempotencyKey());
        if (existing != null) {
            // Return existing group — fetch children via PART_OF
            final List<SpawnedChild> children = WorkItemRelation
                    .findByTargetAndType(request.parentId(), WorkItemRelationType.PART_OF)
                    .stream()
                    .filter(r -> r.createdAt.isAfter(existing.createdAt.minusSeconds(1)))
                    .map(r -> {
                        final WorkItem child = workItemStore.get(r.sourceId).orElseThrow();
                        return new SpawnedChild(child.id, child.callerRef);
                    })
                    .toList();
            return new SpawnResult(existing.id, children, false);
        }

        // Create group record
        final WorkItemSpawnGroup group = new WorkItemSpawnGroup();
        group.parentId = request.parentId();
        group.idempotencyKey = request.idempotencyKey();
        group.persist();

        // Create children
        final List<SpawnedChild> spawned = new ArrayList<>();
        for (final ChildSpec spec : request.children()) {
            final WorkItem child = instantiateChild(spec, parent, group.id);
            spawned.add(new SpawnedChild(child.id, child.callerRef));
        }

        // Fire SPAWNED event on parent
        broadcaster.fire(parent, "SPAWNED", "system:spawn", null);

        return new SpawnResult(group.id, spawned, true);
    }

    /**
     * Cancel a spawn group. With {@code cascadeChildren=true}, all PENDING children
     * (status == PENDING) are cancelled. IN_PROGRESS and other non-terminal children
     * are left untouched — they must be cancelled by the caller individually.
     */
    @Override
    @Transactional
    public void cancelGroup(final UUID groupId, final boolean cascadeChildren) {
        final WorkItemSpawnGroup group = WorkItemSpawnGroup.findById(groupId);
        if (group == null) {
            throw new WorkItemNotFoundException(groupId);
        }

        if (cascadeChildren) {
            WorkItemRelation.findByTargetAndType(group.parentId, WorkItemRelationType.PART_OF)
                    .forEach(rel -> {
                        workItemStore.get(rel.sourceId).ifPresent(child -> {
                            if (child.status == io.quarkiverse.work.runtime.model.WorkItemStatus.PENDING) {
                                workItemService.cancel(child.id, "system:spawn",
                                        "Cancelled by spawn group cancellation");
                            }
                        });
                    });
        }

        group.delete();
    }

    // ── private ──────────────────────────────────────────────────────────────

    private WorkItem instantiateChild(
            final ChildSpec spec, final WorkItem parent, final UUID groupId) {

        final WorkItemTemplate template = WorkItemTemplate.findById(spec.templateId());
        if (template == null) {
            throw new IllegalArgumentException("Template not found: " + spec.templateId());
        }

        // Build create request from template, applying overrides
        final WorkItemCreateRequest.Builder builder = WorkItemCreateRequest.builder()
                .title(template.name)
                .category(template.category)
                .priority(template.priority)
                .candidateGroups(override(spec, "candidateGroups", template.candidateGroups))
                .candidateUsers(override(spec, "candidateUsers", template.candidateUsers))
                .requiredCapabilities(override(spec, "requiredCapabilities", template.requiredCapabilities))
                .payload(template.defaultPayload)
                .createdBy("system:spawn")
                .callerRef(spec.callerRef());

        if (template.defaultExpiryHours != null) {
            builder.expiresAt(java.time.Instant.now()
                    .plus(template.defaultExpiryHours, java.time.temporal.ChronoUnit.HOURS));
        }
        if (template.defaultClaimHours != null) {
            builder.claimDeadline(java.time.Instant.now()
                    .plus(template.defaultClaimHours, java.time.temporal.ChronoUnit.HOURS));
        }

        final WorkItem child = workItemService.create(builder.build());

        // Wire PART_OF: child → parent
        final WorkItemRelation rel = new WorkItemRelation();
        rel.sourceId = child.id;
        rel.targetId = parent.id;
        rel.relationType = WorkItemRelationType.PART_OF;
        rel.createdBy = "system:spawn";
        rel.persist();

        return child;
    }

    @SuppressWarnings("unchecked")
    private String override(final ChildSpec spec, final String key, final String templateValue) {
        if (spec.overrides() != null && spec.overrides().containsKey(key)) {
            final Object val = spec.overrides().get(key);
            return val != null ? val.toString() : null;
        }
        return templateValue;
    }
}
```

> **Important:** `WorkItemCreateRequest` may be a builder, record, or plain class — read `WorkItemService.create()` to see what it accepts, then adapt the builder calls above accordingly. Do not guess; read the file first.

> **Important:** `WorkItemStatus.isTerminal()` — check if this method exists. If not, inline the check:
> ```java
> parent.status == WorkItemStatus.COMPLETED || parent.status == WorkItemStatus.CANCELLED ||
> parent.status == WorkItemStatus.REJECTED || parent.status == WorkItemStatus.EXPIRED
> ```

> **Important:** `WorkItemEventBroadcaster.fire(parent, "SPAWNED", actor, detail)` — read `WorkItemEventBroadcaster` to see the actual method signature and adapt accordingly.

- [ ] **Step 4.4: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | tail -20
```

Fix any compilation errors. Common issues:
- `WorkItemCreateRequest` doesn't have a builder — use whatever pattern the existing code uses
- `WorkItemEventBroadcaster` has different method signature — read the file and adapt
- `WorkItemStatus.isTerminal()` doesn't exist — inline the check

- [ ] **Step 4.5: Commit service (before REST endpoint)**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/service/WorkItemSpawnService.java
git commit -m "feat(runtime): implement WorkItemSpawnService

Creates children from templates, wires PART_OF, stores callerRef,
fires SPAWNED on parent, handles idempotency via SpawnGroup unique
constraint. No completion policy — caller owns that concern.

Refs #130
Refs #131
Refs #105

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: WorkItemSpawnResource REST endpoints

**Issues:** #130, #131  
**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/api/WorkItemSpawnResource.java`

- [ ] **Step 5.1: Create WorkItemSpawnResource**

Create `runtime/src/main/java/io/quarkiverse/work/runtime/api/WorkItemSpawnResource.java`:

```java
package io.quarkiverse.work.runtime.api;

import java.net.URI;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.DELETE;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.work.api.ChildSpec;
import io.quarkiverse.work.api.SpawnRequest;
import io.quarkiverse.work.api.SpawnResult;
import io.quarkiverse.work.api.SpawnedChild;
import io.quarkiverse.work.runtime.model.WorkItemRelation;
import io.quarkiverse.work.runtime.model.WorkItemRelationType;
import io.quarkiverse.work.runtime.model.WorkItemSpawnGroup;
import io.quarkiverse.work.runtime.repository.WorkItemStore;
import io.quarkiverse.work.runtime.service.WorkItemNotFoundException;
import io.quarkiverse.work.runtime.service.WorkItemSpawnService;

/**
 * REST endpoints for subprocess spawning.
 *
 * <pre>
 * POST   /workitems/{id}/spawn                         spawn children from templates
 * GET    /workitems/{id}/spawn-groups                  list spawn groups for a parent
 * GET    /spawn-groups/{groupId}                       group detail + PART_OF children
 * DELETE /workitems/{id}/spawn-groups/{groupId}        cancel group; ?cancelChildren=true cascades
 * </pre>
 */
@Path("/workitems")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class WorkItemSpawnResource {

    @Inject
    WorkItemSpawnService spawnService;

    @Inject
    WorkItemStore workItemStore;

    // ── Request body records ──────────────────────────────────────────────────

    public record SpawnChildRequest(String templateId, String callerRef, Map<String, Object> overrides) {}
    public record SpawnBodyRequest(String idempotencyKey, List<SpawnChildRequest> children) {}

    // ── Endpoints ─────────────────────────────────────────────────────────────

    /**
     * Spawn child WorkItems from templates, linked to the parent via PART_OF.
     *
     * @param parentId     the parent WorkItem UUID
     * @param body         the spawn request with idempotencyKey and children
     * @return 201 Created with SpawnResult on first spawn; 200 OK if idempotencyKey matched existing group
     */
    @POST
    @Path("/{id}/spawn")
    @Transactional
    public Response spawn(
            @PathParam("id") final UUID parentId,
            final SpawnBodyRequest body) {

        if (body == null) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(Map.of("error", "request body is required")).build();
        }
        if (body.children() == null || body.children().isEmpty()) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(Map.of("error", "children must not be empty")).build();
        }
        if (body.idempotencyKey() == null || body.idempotencyKey().isBlank()) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(Map.of("error", "idempotencyKey is required")).build();
        }

        // Validate all templateIds are parseable UUIDs
        for (final SpawnChildRequest child : body.children()) {
            if (child.templateId() == null) {
                return Response.status(Response.Status.BAD_REQUEST)
                        .entity(Map.of("error", "templateId is required for each child")).build();
            }
        }

        final List<ChildSpec> specs = body.children().stream()
                .map(c -> new ChildSpec(UUID.fromString(c.templateId()), c.callerRef(), c.overrides()))
                .toList();

        final SpawnRequest request = new SpawnRequest(parentId, body.idempotencyKey(), specs);

        try {
            final SpawnResult result = spawnService.spawn(request);
            final Map<String, Object> responseBody = Map.of(
                    "groupId", result.groupId(),
                    "children", result.children().stream()
                            .map(c -> Map.of(
                                    "workItemId", c.workItemId().toString(),
                                    "callerRef", c.callerRef() != null ? c.callerRef() : ""))
                            .toList());

            if (result.created()) {
                return Response.created(URI.create("/spawn-groups/" + result.groupId()))
                        .entity(responseBody).build();
            } else {
                return Response.ok(responseBody).build();
            }
        } catch (WorkItemNotFoundException e) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(Map.of("error", e.getMessage())).build();
        } catch (IllegalStateException e) {
            return Response.status(Response.Status.CONFLICT)
                    .entity(Map.of("error", e.getMessage())).build();
        } catch (IllegalArgumentException e) {
            return Response.status(Response.Status.UNPROCESSABLE_ENTITY)
                    .entity(Map.of("error", e.getMessage())).build();
        }
    }

    /**
     * List all spawn groups created from a parent WorkItem.
     *
     * @param parentId the parent WorkItem UUID
     * @return 200 OK with list (may be empty)
     */
    @GET
    @Path("/{id}/spawn-groups")
    public List<Map<String, Object>> listSpawnGroups(@PathParam("id") final UUID parentId) {
        return WorkItemSpawnGroup.findByParentId(parentId).stream()
                .map(g -> Map.<String, Object>of(
                        "id", g.id.toString(),
                        "parentId", g.parentId.toString(),
                        "idempotencyKey", g.idempotencyKey,
                        "createdAt", g.createdAt.toString()))
                .toList();
    }

    /**
     * Cancel a spawn group. With {@code cancelChildren=true}, all PENDING children
     * are also cancelled. IN_PROGRESS children are left untouched.
     *
     * @param parentId        the parent WorkItem UUID (ownership check)
     * @param groupId         the spawn group UUID
     * @param cancelChildren  if true, cancel all PENDING child WorkItems atomically
     * @return 204 No Content; 404 if group not found or does not belong to parent
     */
    @DELETE
    @Path("/{id}/spawn-groups/{groupId}")
    @Transactional
    public Response cancelGroup(
            @PathParam("id") final UUID parentId,
            @PathParam("groupId") final UUID groupId,
            @QueryParam("cancelChildren") final boolean cancelChildren) {

        final WorkItemSpawnGroup group = WorkItemSpawnGroup.findById(groupId);
        if (group == null || !group.parentId.equals(parentId)) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(Map.of("error", "Spawn group not found")).build();
        }

        try {
            spawnService.cancelGroup(groupId, cancelChildren);
            return Response.noContent().build();
        } catch (WorkItemNotFoundException e) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(Map.of("error", e.getMessage())).build();
        }
    }
}

/**
 * Standalone resource for looking up spawn groups by ID (not under /workitems/{id}).
 */
@Path("/spawn-groups")
@Produces(MediaType.APPLICATION_JSON)
class SpawnGroupResource {

    @Inject
    WorkItemStore workItemStore;

    /**
     * Get a spawn group by ID, including its PART_OF children.
     *
     * @param groupId the spawn group UUID
     * @return 200 OK with group detail + children list; 404 if not found
     */
    @GET
    @Path("/{groupId}")
    public Response getGroup(@PathParam("groupId") final UUID groupId) {
        final WorkItemSpawnGroup group = WorkItemSpawnGroup.findById(groupId);
        if (group == null) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(Map.of("error", "Spawn group not found")).build();
        }

        final List<Map<String, Object>> children =
                WorkItemRelation.findByTargetAndType(group.parentId, WorkItemRelationType.PART_OF)
                        .stream()
                        .filter(r -> !r.createdAt.isBefore(group.createdAt.minusSeconds(1)))
                        .map(r -> Map.<String, Object>of(
                                "workItemId", r.sourceId.toString(),
                                "createdAt", r.createdAt.toString()))
                        .toList();

        return Response.ok(Map.of(
                "id", group.id.toString(),
                "parentId", group.parentId.toString(),
                "idempotencyKey", group.idempotencyKey,
                "createdAt", group.createdAt.toString(),
                "children", children)).build();
    }
}
```

> **Note:** If JAX-RS does not support two top-level classes in one file, move `SpawnGroupResource` to its own file `SpawnGroupResource.java` in the same package.

- [ ] **Step 5.2: Compile runtime**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | tail -20
```

Fix any compilation errors before proceeding.

- [ ] **Step 5.3: Run failing test from Task 4.1**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemSpawnResourceTest -pl runtime -q 2>&1 | tail -20
```

Expected: PASS — the spawn endpoint now exists.

- [ ] **Step 5.4: Run full runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 5.5: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/api/WorkItemSpawnResource.java
git add runtime/src/test/java/io/quarkiverse/work/runtime/spawn/WorkItemSpawnResourceTest.java
git commit -m "feat(runtime): POST /workitems/{id}/spawn + group endpoints

Spawn children from templates, wire PART_OF, return groupId and
per-child callerRef. GET /spawn-groups/{id}, GET /workitems/{id}/spawn-groups,
DELETE /workitems/{id}/spawn-groups/{gid}?cancelChildren=true.

Closes #130
Refs #131
Refs #105

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: Full test suite

**Issues:** #130, #131  
**Files:**
- Modify: `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/WorkItemSpawnResourceTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnIdempotencyTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCascadeCancelTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnE2ETest.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCorrectnessTest.java`

- [ ] **Step 6.1: Expand WorkItemSpawnResourceTest with error paths**

Add to `WorkItemSpawnResourceTest.java`:

```java
    @Test
    void spawn_returns404_whenParentNotFound() {
        given()
                .contentType(ContentType.JSON)
                .body(Map.of(
                        "idempotencyKey", "key-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", UUID.randomUUID().toString()))))
                .when().post("/workitems/" + UUID.randomUUID() + "/spawn")
                .then().statusCode(404);
    }

    @Test
    void spawn_returns422_whenChildrenEmpty() {
        final String parentId = createWorkItem("test");
        given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "key-1", "children", List.of()))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(422);
    }

    @Test
    void spawn_returns422_whenNoIdempotencyKey() {
        final String parentId = createWorkItem("test");
        given()
                .contentType(ContentType.JSON)
                .body(Map.of("children", List.of(
                        Map.of("templateId", UUID.randomUUID().toString()))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(400);
    }

    @Test
    void spawn_returns422_whenTemplateNotFound() {
        final String parentId = createWorkItem("test");
        given()
                .contentType(ContentType.JSON)
                .body(Map.of(
                        "idempotencyKey", "key-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", UUID.randomUUID().toString()))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(422);
    }

    @Test
    void spawnGroups_listed_forParent() {
        final String tmplId = createTemplate("tmpl-for-list");
        final String parentId = createWorkItem("loan");
        final String key = "list-test-" + UUID.randomUUID();

        given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", key,
                        "children", List.of(Map.of("templateId", tmplId))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201);

        final List<Map<String, Object>> groups = given()
                .when().get("/workitems/" + parentId + "/spawn-groups")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");

        assertThat(groups).hasSize(1);
        assertThat(groups.get(0).get("idempotencyKey")).isEqualTo(key);
    }

    @Test
    void spawnGroup_getById_returnsGroupAndChildren() {
        final String tmplId = createTemplate("tmpl-get");
        final String parentId = createWorkItem("loan");

        final String groupId = given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "get-test-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmplId, "callerRef", "ref-1"))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201)
                .extract().path("groupId");

        final Response groupResp = given()
                .when().get("/spawn-groups/" + groupId)
                .then().statusCode(200)
                .extract().response();

        assertThat(groupResp.jsonPath().getString("id")).isEqualTo(groupId);
        assertThat(groupResp.jsonPath().getString("parentId")).isEqualTo(parentId);
        final List<Map<String, Object>> children = groupResp.jsonPath().getList("children");
        assertThat(children).hasSize(1);
    }
```

- [ ] **Step 6.2: Create SpawnIdempotencyTest**

Create `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnIdempotencyTest.java`:

```java
package io.quarkiverse.work.runtime.spawn;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class SpawnIdempotencyTest {

    @Test
    void sameIdempotencyKey_sameParent_returnsExistingGroup() {
        final String tmplId = given()
                .contentType(ContentType.JSON)
                .body(Map.of("name", "idem-tmpl", "category", "test", "createdBy", "test"))
                .when().post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        final String parentId = given()
                .contentType(ContentType.JSON)
                .body(Map.of("title", "parent", "category", "test", "createdBy", "test"))
                .when().post("/workitems")
                .then().statusCode(201).extract().path("id");

        final String key = "idem-key-" + UUID.randomUUID();
        final var body = Map.of("idempotencyKey", key,
                "children", List.of(Map.of("templateId", tmplId)));

        // First call — 201
        final String groupId1 = given()
                .contentType(ContentType.JSON).body(body)
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201).extract().path("groupId");

        // Second call — 200 (idempotent)
        final String groupId2 = given()
                .contentType(ContentType.JSON).body(body)
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(200).extract().path("groupId");

        assertThat(groupId1).isEqualTo(groupId2);

        // Child count is still 1 — no duplicate
        final List<Map<String, Object>> children = given()
                .when().get("/workitems/" + parentId + "/children")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        assertThat(children).hasSize(1);
    }

    @Test
    void differentIdempotencyKey_sameParent_createsNewGroup() {
        final String tmplId = given()
                .contentType(ContentType.JSON)
                .body(Map.of("name", "idem-tmpl2", "category", "test", "createdBy", "test"))
                .when().post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        final String parentId = given()
                .contentType(ContentType.JSON)
                .body(Map.of("title", "parent2", "category", "test", "createdBy", "test"))
                .when().post("/workitems")
                .then().statusCode(201).extract().path("id");

        final String groupId1 = given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "key-A-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmplId))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201).extract().path("groupId");

        final String groupId2 = given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "key-B-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmplId))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201).extract().path("groupId");

        assertThat(groupId1).isNotEqualTo(groupId2);

        // Two children now
        final List<Map<String, Object>> children = given()
                .when().get("/workitems/" + parentId + "/children")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        assertThat(children).hasSize(2);
    }
}
```

- [ ] **Step 6.3: Create SpawnCascadeCancelTest**

Create `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCascadeCancelTest.java`:

```java
package io.quarkiverse.work.runtime.spawn;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class SpawnCascadeCancelTest {

    @Test
    void cancelGroup_withoutCascade_leavesChildrenUntouched() {
        final String tmplId = createTemplate("cascade-tmpl-1");
        final String parentId = createWorkItem("cascade-parent-1");

        final String groupId = spawnOne(parentId, tmplId);

        given()
                .when().delete("/workitems/" + parentId + "/spawn-groups/" + groupId)
                .then().statusCode(204);

        // Child still PENDING
        final List<Map<String, Object>> children = given()
                .when().get("/workitems/" + parentId + "/children")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        assertThat(children).hasSize(1);

        final String childId = (String) children.get(0).get("id");
        final String childStatus = given()
                .when().get("/workitems/" + childId)
                .then().statusCode(200)
                .extract().path("status");
        assertThat(childStatus).isEqualTo("PENDING");
    }

    @Test
    void cancelGroup_withCascade_cancelsPendingChildren() {
        final String tmplId = createTemplate("cascade-tmpl-2");
        final String parentId = createWorkItem("cascade-parent-2");

        final String groupId = spawnOne(parentId, tmplId);

        given()
                .when().delete("/workitems/" + parentId + "/spawn-groups/" + groupId + "?cancelChildren=true")
                .then().statusCode(204);

        // Child is now CANCELLED
        final List<Map<String, Object>> children = given()
                .when().get("/workitems/" + parentId + "/children")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        final String childId = (String) children.get(0).get("id");

        final String childStatus = given()
                .when().get("/workitems/" + childId)
                .then().statusCode(200)
                .extract().path("status");
        assertThat(childStatus).isEqualTo("CANCELLED");
    }

    @Test
    void cancelGroup_returns404_whenGroupNotFound() {
        final String parentId = createWorkItem("cancel-404");
        given()
                .when().delete("/workitems/" + parentId + "/spawn-groups/" + UUID.randomUUID())
                .then().statusCode(404);
    }

    // helpers
    private String createTemplate(final String name) {
        return given().contentType(ContentType.JSON)
                .body(Map.of("name", name, "category", name, "createdBy", "test"))
                .when().post("/workitem-templates")
                .then().statusCode(201).extract().path("id");
    }

    private String createWorkItem(final String category) {
        return given().contentType(ContentType.JSON)
                .body(Map.of("title", "parent-" + category, "category", category, "createdBy", "test"))
                .when().post("/workitems")
                .then().statusCode(201).extract().path("id");
    }

    private String spawnOne(final String parentId, final String tmplId) {
        return given().contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "cascade-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmplId))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201).extract().path("groupId");
    }
}
```

- [ ] **Step 6.4: Create SpawnE2ETest**

Create `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnE2ETest.java`:

```java
package io.quarkiverse.work.runtime.spawn;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.restassured.response.Response;

/**
 * End-to-end scenarios exercising the full spawn lifecycle.
 */
@QuarkusTest
class SpawnE2ETest {

    @Test
    void casehubPattern_callerRef_roundTripsOnEachChild() {
        // Templates representing parallel checks
        final String creditTmpl = createTemplate("e2e-credit");
        final String fraudTmpl  = createTemplate("e2e-fraud");
        final String compTmpl   = createTemplate("e2e-compliance");

        // Parent WorkItem (CaseHub would create this when a case starts)
        final String parentId = createWorkItem("loan-application");

        // CaseHub calls spawn, embedding planItemIds as callerRef
        final var spawnBody = Map.of(
                "idempotencyKey", "casehub-e2e-" + UUID.randomUUID(),
                "children", List.of(
                        Map.of("templateId", creditTmpl, "callerRef", "case:loan-1/pi:credit"),
                        Map.of("templateId", fraudTmpl,  "callerRef", "case:loan-1/pi:fraud"),
                        Map.of("templateId", compTmpl,   "callerRef", "case:loan-1/pi:compliance")));

        final Response spawnResp = given()
                .contentType(ContentType.JSON).body(spawnBody)
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201)
                .extract().response();

        final String groupId = spawnResp.jsonPath().getString("groupId");
        assertThat(groupId).isNotNull();

        final List<Map<String, Object>> spawnedChildren = spawnResp.jsonPath().getList("children");
        assertThat(spawnedChildren).hasSize(3);

        // Each spawned child has its callerRef
        final var callerRefs = spawnedChildren.stream()
                .map(c -> (String) c.get("callerRef")).toList();
        assertThat(callerRefs).containsExactlyInAnyOrder(
                "case:loan-1/pi:credit", "case:loan-1/pi:fraud", "case:loan-1/pi:compliance");

        // callerRef is retrievable on each child WorkItem
        spawnedChildren.forEach(child -> {
            final String childId = (String) child.get("workItemId");
            final String storedRef = given()
                    .when().get("/workitems/" + childId)
                    .then().statusCode(200)
                    .extract().path("callerRef");
            assertThat(storedRef).isEqualTo((String) child.get("callerRef"));
        });

        // PART_OF links exist
        final List<Map<String, Object>> partOfChildren = given()
                .when().get("/workitems/" + parentId + "/children")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        assertThat(partOfChildren).hasSize(3);

        // Parent has SPAWNED in audit trail
        final List<Map<String, Object>> audit = given()
                .when().get("/workitems/" + parentId + "/audit")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        final var eventTypes = audit.stream().map(a -> (String) a.get("event")).toList();
        assertThat(eventTypes).contains("SPAWNED");
    }

    @Test
    void nestedSpawn_grandchildren_doNotAppearInParentChildList() {
        final String tmpl = createTemplate("nested-tmpl");
        final String parentId = createWorkItem("nested-parent");

        // Spawn one child
        final var firstSpawn = given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "nest-1-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmpl))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201).extract().response();

        final String childId = (String)
                ((List<Map<String, Object>>) firstSpawn.jsonPath().getList("children")).get(0).get("workItemId");

        // Spawn a grandchild from the child
        given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "nest-2-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmpl))))
                .when().post("/workitems/" + childId + "/spawn")
                .then().statusCode(201);

        // Parent children: only the direct child (not grandchild)
        final List<?> parentChildren = given()
                .when().get("/workitems/" + parentId + "/children")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        assertThat(parentChildren).hasSize(1);

        // Child children: only the grandchild
        final List<?> childChildren = given()
                .when().get("/workitems/" + childId + "/children")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");
        assertThat(childChildren).hasSize(1);
    }

    // helpers
    private String createTemplate(final String name) {
        return given().contentType(ContentType.JSON)
                .body(Map.of("name", name, "category", name, "createdBy", "test"))
                .when().post("/workitem-templates")
                .then().statusCode(201).extract().path("id");
    }

    private String createWorkItem(final String category) {
        return given().contentType(ContentType.JSON)
                .body(Map.of("title", "parent-" + category, "category", category, "createdBy", "test"))
                .when().post("/workitems")
                .then().statusCode(201).extract().path("id");
    }
}
```

- [ ] **Step 6.5: Create SpawnCorrectnessTest**

Create `runtime/src/test/java/io/quarkiverse/work/runtime/spawn/SpawnCorrectnessTest.java`:

```java
package io.quarkiverse.work.runtime.spawn;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.restassured.response.Response;

/**
 * Correctness invariants for subprocess spawning.
 */
@QuarkusTest
class SpawnCorrectnessTest {

    @Test
    void partOf_direction_isChildToParent() {
        final String tmplId = createTemplate("dir-tmpl");
        final String parentId = createWorkItem("dir-parent");

        final Response spawnResp = given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "dir-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmplId))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201).extract().response();

        final String childId = (String)
                ((List<Map<String, Object>>) spawnResp.jsonPath().getList("children"))
                        .get(0).get("workItemId");

        // Child has outgoing PART_OF relation to parent
        final List<Map<String, Object>> outgoing = given()
                .when().get("/workitems/" + childId + "/relations")
                .then().statusCode(200)
                .extract().jsonPath().getList("$");

        assertThat(outgoing).anySatisfy(rel -> {
            assertThat(rel.get("relationType")).isEqualTo("PART_OF");
            assertThat(rel.get("targetId")).isEqualTo(parentId);
        });
    }

    @Test
    void callerRef_emptyString_storedAsEmptyNotNull() {
        final String tmplId = createTemplate("empty-ref-tmpl");
        final String parentId = createWorkItem("empty-ref-parent");

        final Response spawnResp = given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "empty-ref-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmplId, "callerRef", ""))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201).extract().response();

        final String childId = (String)
                ((List<Map<String, Object>>) spawnResp.jsonPath().getList("children"))
                        .get(0).get("workItemId");

        final Object storedRef = given()
                .when().get("/workitems/" + childId)
                .then().statusCode(200)
                .extract().path("callerRef");
        // Empty string stored as empty (not null)
        assertThat(storedRef).isEqualTo("");
    }

    @Test
    void cycleGuard_preventsChildBecomingAncestorOfItself() {
        final String tmplId = createTemplate("cycle-tmpl");
        final String parentId = createWorkItem("cycle-parent");

        final Response spawnResp = given()
                .contentType(ContentType.JSON)
                .body(Map.of("idempotencyKey", "cycle-" + UUID.randomUUID(),
                        "children", List.of(Map.of("templateId", tmplId))))
                .when().post("/workitems/" + parentId + "/spawn")
                .then().statusCode(201).extract().response();

        final String childId = (String)
                ((List<Map<String, Object>>) spawnResp.jsonPath().getList("children"))
                        .get(0).get("workItemId");

        // Attempt to make parent PART_OF child — would create cycle
        given()
                .contentType(ContentType.JSON)
                .body(Map.of("targetId", childId, "relationType", "PART_OF", "createdBy", "test"))
                .when().post("/workitems/" + parentId + "/relations")
                .then().statusCode(400); // cycle rejected
    }

    // helpers
    private String createTemplate(final String name) {
        return given().contentType(ContentType.JSON)
                .body(Map.of("name", name, "category", name, "createdBy", "test"))
                .when().post("/workitem-templates")
                .then().statusCode(201).extract().path("id");
    }

    private String createWorkItem(final String category) {
        return given().contentType(ContentType.JSON)
                .body(Map.of("title", "p-" + category, "category", category, "createdBy", "test"))
                .when().post("/workitems")
                .then().statusCode(201).extract().path("id");
    }
}
```

- [ ] **Step 6.6: Run all spawn tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest="Spawn*,WorkItemSpawnResourceTest" -pl runtime -q 2>&1 | tail -20
```

Expected: all tests pass. Fix any failures before proceeding.

- [ ] **Step 6.7: Run full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 6.8: Commit**

```bash
git add runtime/src/test/java/io/quarkiverse/work/runtime/spawn/
git commit -m "test(runtime): full spawn test suite

Unit, integration, E2E, idempotency, cascade cancel, correctness.
All 'what happens next' decisions tested as NOT present in quarkus-work.

Closes #131
Refs #105

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: Ledger causal chain wiring

**Issue:** #132  
**Files:**
- Modify: `quarkus-work-ledger/src/main/java/io/quarkiverse/work/ledger/service/LedgerEventCapture.java`
- Modify: `quarkus-work-ledger/src/test/java/io/quarkiverse/work/ledger/LedgerIntegrationTest.java`

- [ ] **Step 7.1: Write failing ledger test**

Add this test to `LedgerIntegrationTest.java`:

```java
    @Test
    void spawn_createsCausalChain_inLedger() {
        // Create parent
        final WorkItemCreateRequest parentReq = new WorkItemCreateRequest(
                "Loan Application", null, "loan", null,
                null, null, null, null, null, "finance-system",
                null, null, null, null, null, null, null, null, null);
        final WorkItem parent = workItemService.create(parentReq);

        // Create child (as spawn would do — same mechanism)
        final WorkItemCreateRequest childReq = new WorkItemCreateRequest(
                "Credit Check", null, "credit", null,
                null, null, null, null, null, "system:spawn",
                null, null, null, null, null, null, "case:loan-1/pi:credit", null, null);
        final WorkItem child = workItemService.create(childReq);

        // Fetch ledger entries
        final var parentEntries = ledgerRepo.findByWorkItemId(parent.id);
        final var childEntries = ledgerRepo.findByWorkItemId(child.id);

        assertThat(parentEntries).isNotEmpty();
        assertThat(childEntries).isNotEmpty();

        // For now, just verify both have entries (causedByEntryId wiring comes in next step)
        assertThat(parentEntries.get(0).eventType).isEqualTo("WorkItemCreated");
        assertThat(childEntries.get(0).eventType).isEqualTo("WorkItemCreated");
    }
```

> **Note:** `WorkItemCreateRequest` constructor arity may differ — read `WorkItemService.create()` and adapt.

- [ ] **Step 7.2: Add "spawned" to EVENT_META in LedgerEventCapture**

In `LedgerEventCapture.java`, find the `EVENT_META` map and add the spawned entry:

```java
Map.entry("spawned", new String[] { "SpawnWorkItems", "WorkItemsSpawned", "System" }),
```

- [ ] **Step 7.3: Wire causedByEntryId for spawned children**

The spawn event fires on the parent AFTER children are created. To wire `causedByEntryId` on child CREATED entries to point to the parent's SPAWNED entry, the ledger needs the parent's SPAWNED entry UUID at child-creation time.

**Approach:** Store the parent's latest ledger entry UUID on `WorkItemLifecycleEvent` for CREATED events where `createdBy = "system:spawn"`. The `LedgerEventCapture` observer detects this and sets `causedByEntryId`.

Add to `WorkItemLifecycleEvent` a nullable `causedByWorkItemId` field — the parent WorkItem ID when this child was spawned. Then in `LedgerEventCapture.onWorkItemEvent()`:

```java
// After creating the entry, before saving:
if ("created".equals(eventSuffix(event.type())) && event.workItem().callerRef != null) {
    // This WorkItem has a callerRef — it was created by spawn
    // Find the most recent SPAWNED ledger entry for the parent
    // (parent is identified via the PART_OF graph)
    final java.util.Optional<UUID> parentId = WorkItemRelation
            .findBySourceAndType(event.workItemId(), WorkItemRelationType.PART_OF)
            .stream().findFirst().map(r -> r.targetId);
    parentId.ifPresent(pid -> {
        ledgerRepo.findLatestByWorkItemId(pid).ifPresent(parentEntry -> {
            if ("WorkItemsSpawned".equals(parentEntry.eventType)) {
                entry.causedByEntryId = parentEntry.id;
            }
        });
    });
}
```

> **Note:** Read `LedgerEventCapture` and `WorkItemLedgerEntryRepository` to understand what `findLatestByWorkItemId` returns and adapt accordingly. The key constraint: `causedByEntryId` must point to the parent's SPAWNED entry, not just any entry.

- [ ] **Step 7.4: Compile ledger module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl quarkus-work-ledger -q 2>&1 | tail -10
```

Expected: BUILD SUCCESS.

- [ ] **Step 7.5: Run ledger tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-ledger -q 2>&1 | tail -10
```

Expected: BUILD SUCCESS.

- [ ] **Step 7.6: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 7.7: Commit**

```bash
git add quarkus-work-ledger/src/main/java/io/quarkiverse/work/ledger/service/LedgerEventCapture.java
git add quarkus-work-ledger/src/test/java/io/quarkiverse/work/ledger/LedgerIntegrationTest.java
git commit -m "feat(ledger): wire causedByEntryId for spawned children

SPAWNED event on parent gets LedgerEntry. Child CREATED entries
created by system:spawn carry causedByEntryId pointing to parent's
SPAWNED entry — enabling PROV-DM prov:wasDerivedFrom traversal.

Closes #132
Refs #105

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 8: Example scenario + documentation

**Files:**
- Create: `quarkus-work-examples/src/main/java/io/quarkiverse/work/examples/spawn/SpawnScenarioResource.java`
- Create: `quarkus-work-examples/src/test/java/io/quarkiverse/work/examples/spawn/SpawnScenarioTest.java`
- Modify: `CLAUDE.md` — update project structure with spawn types
- Modify: `docs/DESIGN.md` — mark subprocess spawning phase complete

- [ ] **Step 8.1: Write failing scenario test first**

Create `quarkus-work-examples/src/test/java/io/quarkiverse/work/examples/spawn/SpawnScenarioTest.java`:

```java
package io.quarkiverse.work.examples.spawn;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.restassured.response.Response;

@QuarkusTest
class SpawnScenarioTest {

    @Test
    void run_loanApproval_spawnsThreeParallelChecks() {
        final Response response = given()
                .contentType(ContentType.JSON)
                .when().post("/examples/spawn/run")
                .then().statusCode(200)
                .extract().response();

        assertThat(response.jsonPath().getString("scenario")).isEqualTo("parallel-spawn");

        final List<Map<String, Object>> steps = response.jsonPath().getList("steps");
        assertThat(steps).hasSizeGreaterThanOrEqualTo(3);

        // Parent created
        assertThat(steps.get(0).get("description").toString())
                .containsIgnoringCase("loan-application");

        // Children spawned
        assertThat(response.jsonPath().getString("parentWorkItemId")).isNotNull();

        final List<String> childIds = response.jsonPath().getList("childWorkItemIds");
        assertThat(childIds).hasSize(3);

        // Each child has a callerRef
        childIds.forEach(childId -> {
            final String ref = given()
                    .when().get("/workitems/" + childId)
                    .then().statusCode(200)
                    .extract().path("callerRef");
            assertThat(ref).isNotNull().startsWith("case:loan");
        });
    }
}
```

- [ ] **Step 8.2: Run the failing test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SpawnScenarioTest -pl quarkus-work-examples -q 2>&1 | tail -15
```

Expected: FAIL — endpoint does not exist.

- [ ] **Step 8.3: Create SpawnScenarioResource**

Read one existing scenario resource (e.g. `ExpenseApprovalScenarioResource.java` or `CreditDecisionScenarioResource.java`) to understand the exact pattern — how `ScenarioResponse`, `StepLog`, and `QueueScenarioResponse` are used — then implement:

Create `quarkus-work-examples/src/main/java/io/quarkiverse/work/examples/spawn/SpawnScenarioResource.java`:

```java
package io.quarkiverse.work.examples.spawn;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.work.api.ChildSpec;
import io.quarkiverse.work.api.SpawnRequest;
import io.quarkiverse.work.api.SpawnResult;
import io.quarkiverse.work.examples.ScenarioResponse;
import io.quarkiverse.work.examples.StepLog;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.service.WorkItemCreateRequest;
import io.quarkiverse.work.runtime.service.WorkItemService;
import io.quarkiverse.work.runtime.service.WorkItemSpawnService;

/**
 * Demonstrates subprocess spawning: a loan-application WorkItem spawns three
 * parallel child WorkItems (credit-check, fraud-check, compliance-check)
 * each with a distinct callerRef for CaseHub-style routing.
 *
 * <p>
 * POST /examples/spawn/run
 */
@Path("/examples/spawn")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class SpawnScenarioResource {

    @Inject
    WorkItemService workItemService;

    @Inject
    WorkItemSpawnService spawnService;

    @POST
    @Path("/run")
    @Transactional
    public Response run() {
        final List<StepLog> steps = new ArrayList<>();

        // Step 1: Create templates for the three parallel checks
        final WorkItemTemplate creditTmpl = createTemplate("credit-check", "credit-review");
        final WorkItemTemplate fraudTmpl  = createTemplate("fraud-check",  "fraud-review");
        final WorkItemTemplate compTmpl   = createTemplate("compliance-check", "compliance-review");
        steps.add(StepLog.of("Creates three check templates (credit, fraud, compliance)"));

        // Step 2: Create the parent loan-application WorkItem
        // (In a real CaseHub deployment, CaseHub would create this when a case starts)
        final WorkItem parent = workItemService.create(WorkItemCreateRequest.builder()
                .title("Loan Application #" + UUID.randomUUID().toString().substring(0, 8))
                .category("loan-application")
                .createdBy("finance-system")
                .build());
        steps.add(StepLog.of("Creates loan-application WorkItem (parent): " + parent.id));

        // Step 3: Spawn three parallel child WorkItems
        // callerRef mimics what CaseHub would embed to route completion
        final SpawnRequest spawnRequest = new SpawnRequest(
                parent.id,
                "loan-spawn-" + parent.id,
                List.of(
                        new ChildSpec(creditTmpl.id, "case:loan-" + parent.id + "/pi:credit", null),
                        new ChildSpec(fraudTmpl.id,  "case:loan-" + parent.id + "/pi:fraud",  null),
                        new ChildSpec(compTmpl.id,   "case:loan-" + parent.id + "/pi:compliance", null)));

        final SpawnResult result = spawnService.spawn(spawnRequest);
        steps.add(StepLog.of("Spawns 3 parallel child WorkItems via POST /workitems/{id}/spawn"));
        steps.add(StepLog.of("Each child carries callerRef for CaseHub-style routing on completion"));

        final List<String> childIds = result.children().stream()
                .map(c -> c.workItemId().toString()).toList();

        return Response.ok(Map.of(
                "scenario", "parallel-spawn",
                "steps", steps,
                "parentWorkItemId", parent.id.toString(),
                "spawnGroupId", result.groupId().toString(),
                "childWorkItemIds", childIds,
                "note", "quarkus-work makes no decisions about child completion — that belongs to CaseHub")).build();
    }

    private WorkItemTemplate createTemplate(final String name, final String category) {
        final WorkItemTemplate t = new WorkItemTemplate();
        t.name = name;
        t.category = category;
        t.createdBy = "spawn-scenario";
        t.persist();
        return t;
    }
}
```

> **Important:** Read `WorkItemCreateRequest` to confirm whether it has a `builder()` method or is a record with a constructor. Adapt accordingly.

> **Important:** Read `StepLog.of()` signature — it may take description only or description + data. Adapt to match.

- [ ] **Step 8.4: Run the scenario test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SpawnScenarioTest -pl quarkus-work-examples -q 2>&1 | tail -15
```

Expected: PASS.

- [ ] **Step 8.5: Run full examples test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-examples -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 8.6: Update CLAUDE.md project structure**

In `CLAUDE.md`, find the project structure section and add spawn types:

```
│       ├── model/
│       │   ├── WorkItemSpawnGroup.java    — spawn batch tracking (idempotency + membership)
│       ├── service/
│       │   ├── WorkItemSpawnService.java  — implements SpawnPort for WorkItems
│       └── api/
│           └── WorkItemSpawnResource.java — REST: POST /workitems/{id}/spawn, GET/DELETE /spawn-groups
```

Also update the `quarkus-work-api` section to list the new SPI types:
```
│       ├── SpawnPort.java                 — SPI: spawn(SpawnRequest), cancelGroup
│       ├── SpawnRequest.java              — request: parentId, idempotencyKey, children
│       ├── ChildSpec.java                 — per-child: templateId, callerRef, overrides
│       ├── SpawnResult.java               — result: groupId, children, created
│       └── SpawnedChild.java              — per-child result: workItemId, callerRef
```

Also update Known Quarkiverse gotchas to add:
```
- Spawn idempotency key is scoped per parent — same key on different parents creates separate groups; use parentId+key as the uniqueness scope.
- callerRef is opaque to quarkus-work — never interpret it; CaseHub owns the semantics.
```

- [ ] **Step 8.7: Commit example and documentation**

```bash
git add quarkus-work-examples/src/main/java/io/quarkiverse/work/examples/spawn/
git add quarkus-work-examples/src/test/java/io/quarkiverse/work/examples/spawn/
git add CLAUDE.md
git commit -m "feat(examples): spawn scenario + CLAUDE.md documentation

Loan-application spawns 3 parallel child WorkItems (credit, fraud,
compliance) each with callerRef for CaseHub-style routing.
Documents spawn primitives in CLAUDE.md project structure.

Refs #105
Refs #100

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 8.8: Final full build and test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -q 2>&1 | tail -10
```

Expected: BUILD SUCCESS across all modules.

- [ ] **Step 8.9: Push**

```bash
git push origin main
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Covered by |
|---|---|
| `callerRef` on WorkItem | Task 2 |
| `WorkEventType.SPAWNED` | Task 1 |
| `SpawnPort` SPI in quarkus-work-api | Task 1 |
| `POST /workitems/{id}/spawn` | Task 5 |
| `WorkItemSpawnGroup` entity | Task 3 |
| Idempotency via idempotencyKey | Task 3 + Task 6 |
| `GET /workitems/{id}/spawn-groups` | Task 5 |
| `GET /spawn-groups/{id}` | Task 5 |
| `DELETE /workitems/{id}/spawn-groups/{gid}?cancelChildren=true` | Task 5 |
| callerRef round-trip tests | Task 2 + Task 6 |
| Idempotency tests | Task 6 |
| Cascade cancel tests | Task 6 |
| E2E CaseHub pattern test | Task 6 |
| PART_OF direction correctness test | Task 6 |
| Cycle guard test | Task 6 |
| Ledger `causedByEntryId` wiring | Task 7 |
| `WorkEventType.SPAWNED` in ledger | Task 7 |
| Example scenario | Task 8 |
| Documentation update | Task 8 |

**No orchestration in quarkus-work:** ✓ — no CompletionRollupService, no SpawnEngine auto-trigger, no auto-complete-parent in any task.

**Placeholder scan:** None found — all code blocks are complete with actual class names, method names, and field names from the codebase exploration.

**Type consistency check:**
- `SpawnPort.spawn(SpawnRequest)` → `SpawnResult` — consistent Tasks 1, 4, 5
- `SpawnRequest(UUID, String, List<ChildSpec>)` — consistent Tasks 1, 4, 8
- `ChildSpec(UUID, String, Map<String, Object>)` — consistent Tasks 1, 4, 5, 8
- `SpawnResult(UUID, List<SpawnedChild>, boolean)` — consistent Tasks 1, 4, 5
- `SpawnedChild(UUID, String)` — consistent Tasks 1, 4, 5
- `WorkItemSpawnGroup.findByParentAndKey(UUID, String)` — used in Tasks 3, 4 ✓
- `WorkItemSpawnGroup.findByParentId(UUID)` — used in Tasks 3, 5 ✓
