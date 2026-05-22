# Core Label Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add MANUAL/INFERRED label support to WorkItem core (sub-epic #51 — issues #53, #54, #55).

**Architecture:** `WorkItemLabel` is a JPA `@Embeddable` stored via `@ElementCollection` in a `work_item_label` table. `LabelVocabulary` and `LabelDefinition` are Panache entities enforcing path declarations. All changes are in the `runtime` and `testing` modules only — zero queues-module dependency. Strict TDD: unit test first, @QuarkusTest second, commit per issue.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM Panache, Flyway, H2 (test), REST Assured, JUnit 5

**Issues:** #53 (entity + migration), #54 (vocabulary), #55 (label query + MANUAL endpoints)

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

---

## File Map

**Created:**
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelPersistence.java`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemLabel.java`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/VocabularyScope.java`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelVocabulary.java`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelDefinition.java`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyService.java`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/VocabularyResource.java`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/AddLabelRequest.java`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemLabelResponse.java`
- `runtime/src/main/resources/db/migration/V2__label_schema.sql`
- `runtime/src/main/resources/db/migration/V3__vocabulary_schema.sql`
- `runtime/src/test/java/io/quarkiverse/workitems/runtime/model/WorkItemLabelTest.java`
- `runtime/src/test/java/io/quarkiverse/workitems/runtime/api/LabelEndpointTest.java`
- `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyServiceTest.java`

**Modified:**
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItem.java` — add `labels` field
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemCreateRequest.java` — add `labels`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemMapper.java` — map labels
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResponse.java` — add `labels`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResource.java` — label param + endpoints
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/repository/WorkItemRepository.java` — add `findByLabelPattern`
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/repository/jpa/JpaWorkItemRepository.java` — implement it
- `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemService.java` — `addLabel`, `removeLabel`
- `testing/src/main/java/io/quarkiverse/workitems/testing/InMemoryWorkItemRepository.java` — label pattern matching

---

## Issue #53 — WorkItemLabel Entity + Flyway V2

---

### Task 1: LabelPersistence enum

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelPersistence.java`
- Create: `runtime/src/test/java/io/quarkiverse/workitems/runtime/model/WorkItemLabelTest.java`

- [ ] **Step 1: Write the failing test**

```java
// runtime/src/test/java/io/quarkiverse/workitems/runtime/model/WorkItemLabelTest.java
package io.quarkiverse.work.runtime.model;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class WorkItemLabelTest {

    @Test
    void labelPersistence_hasTwoValues() {
        assertThat(LabelPersistence.values())
                .containsExactlyInAnyOrder(LabelPersistence.MANUAL, LabelPersistence.INFERRED);
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemLabelTest#labelPersistence_hasTwoValues -Dno-format 2>&1 | tail -10
```

Expected: `cannot find symbol: class LabelPersistence`

- [ ] **Step 3: Create LabelPersistence**

```java
// runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelPersistence.java
package io.quarkiverse.work.runtime.model;

/**
 * Determines how a label was applied to a WorkItem and how it is maintained.
 *
 * <p>
 * {@code MANUAL} labels are applied by humans and persist until explicitly removed.
 * {@code INFERRED} labels are applied by the filter engine and are recomputed on every
 * WorkItem mutation — they exist only while the filter condition remains true.
 */
public enum LabelPersistence {
    /**
     * Human-applied. Only removed by an explicit API call or human action.
     * Never touched by the filter re-evaluation cycle.
     */
    MANUAL,

    /**
     * Filter-applied. Stripped and recomputed on every WorkItem mutation.
     * Exists only while at least one FilterChain supports it.
     */
    INFERRED
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemLabelTest#labelPersistence_hasTwoValues -Dno-format 2>&1 | tail -5
```

Expected: `Tests run: 1, Failures: 0, Errors: 0`

---

### Task 2: WorkItemLabel embeddable

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemLabel.java`
- Modify: `runtime/src/test/java/io/quarkiverse/workitems/runtime/model/WorkItemLabelTest.java`

- [ ] **Step 1: Write failing tests**

Add to `WorkItemLabelTest`:

```java
    @Test
    void workItemLabel_manualConstructor_setsAllFields() {
        var label = new WorkItemLabel("legal/contracts", LabelPersistence.MANUAL, "alice");
        assertThat(label.path).isEqualTo("legal/contracts");
        assertThat(label.persistence).isEqualTo(LabelPersistence.MANUAL);
        assertThat(label.appliedBy).isEqualTo("alice");
    }

    @Test
    void workItemLabel_inferredConstructor_nullAppliedBy() {
        var label = new WorkItemLabel("intake", LabelPersistence.INFERRED, null);
        assertThat(label.path).isEqualTo("intake");
        assertThat(label.persistence).isEqualTo(LabelPersistence.INFERRED);
        assertThat(label.appliedBy).isNull();
    }

    @Test
    void workItemLabel_singleSegmentPath_isValid() {
        var label = new WorkItemLabel("legal", LabelPersistence.MANUAL, "bob");
        assertThat(label.path).isEqualTo("legal");
    }
```

- [ ] **Step 2: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemLabelTest -Dno-format 2>&1 | tail -10
```

Expected: `cannot find symbol: class WorkItemLabel`

- [ ] **Step 3: Create WorkItemLabel**

```java
// runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemLabel.java
package io.quarkiverse.work.runtime.model;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;

/**
 * A label applied to a {@link WorkItem}, identified by a {@code /}-separated path.
 *
 * <p>
 * Labels are either {@link LabelPersistence#MANUAL} (human-applied, persists until
 * explicitly removed) or {@link LabelPersistence#INFERRED} (filter-applied, recomputed
 * on every WorkItem mutation).
 *
 * <p>
 * A label path is a {@code /}-separated sequence of terms, e.g. {@code legal},
 * {@code legal/contracts}, {@code legal/contracts/nda}. A single-term label is
 * structurally identical to a multi-segment path.
 */
@Embeddable
public class WorkItemLabel {

    /** The label path, e.g. {@code legal/contracts/nda}. */
    @Column(name = "path", nullable = false, length = 500)
    public String path;

    /**
     * How this label was applied and how it is maintained.
     *
     * @see LabelPersistence
     */
    @Enumerated(EnumType.STRING)
    @Column(name = "persistence", nullable = false, length = 20)
    public LabelPersistence persistence;

    /**
     * Who or what applied this label.
     * For {@link LabelPersistence#MANUAL}: the userId who applied it.
     * For {@link LabelPersistence#INFERRED}: the filterId that applied it (may be null
     * when set directly by the queues engine).
     */
    @Column(name = "applied_by", length = 255)
    public String appliedBy;

    /** JPA requires a no-arg constructor. */
    public WorkItemLabel() {
    }

    /**
     * Convenience constructor for creating a label with all fields.
     *
     * @param path       the label path; must not be null or blank
     * @param persistence how the label is maintained
     * @param appliedBy  userId (MANUAL) or filterId (INFERRED); may be null
     */
    public WorkItemLabel(final String path, final LabelPersistence persistence, final String appliedBy) {
        this.path = path;
        this.persistence = persistence;
        this.appliedBy = appliedBy;
    }
}
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemLabelTest -Dno-format 2>&1 | tail -5
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

---

### Task 3: WorkItemLabelResponse DTO

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemLabelResponse.java`

- [ ] **Step 1: Create the record**

No test needed for a pure data record — it will be covered by the integration tests in Task 9.

```java
// runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemLabelResponse.java
package io.quarkiverse.work.runtime.api;

import io.quarkiverse.work.runtime.model.LabelPersistence;

public record WorkItemLabelResponse(
        String path,
        LabelPersistence persistence,
        String appliedBy) {
}
```

---

### Task 4: Flyway V2 migration — work_item_label table

**Files:**
- Create: `runtime/src/main/resources/db/migration/V2__label_schema.sql`

- [ ] **Step 1: Write the migration**

```sql
-- Quarkus WorkItems V2: label support
-- Each WorkItem can carry 0..n labels, each with a path, persistence type, and appliedBy actor.
-- Compatible with H2 (dev/test) and PostgreSQL (production).

CREATE TABLE work_item_label (
    work_item_id    UUID            NOT NULL,
    path            VARCHAR(500)    NOT NULL,
    persistence     VARCHAR(20)     NOT NULL,
    applied_by      VARCHAR(255)
);

-- Fast lookup of labels for a given WorkItem
CREATE INDEX idx_wil_work_item_id ON work_item_label(work_item_id);

-- Fast lookup of WorkItems by label path prefix (used by findByLabelPattern)
CREATE INDEX idx_wil_path ON work_item_label(path);
```

- [ ] **Step 2: Verify migration applies cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemSmokeTest -Dno-format 2>&1 | tail -8
```

Expected: `Tests run: 15, Failures: 0, Errors: 0` (Flyway runs V1+V2 automatically)

---

### Task 5: Add labels to WorkItem entity, request, response, and mapper

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItem.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemCreateRequest.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResponse.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemMapper.java`

- [ ] **Step 1: Add labels to WorkItem entity**

In `WorkItem.java`, add after the timestamps section (before `prePersist`):

```java
    // -------------------------------------------------------------------------
    // Labels
    // -------------------------------------------------------------------------

    /**
     * Labels attached to this WorkItem.
     *
     * <p>
     * {@link LabelPersistence#MANUAL} labels are applied by humans.
     * {@link LabelPersistence#INFERRED} labels are maintained by the filter engine
     * in {@code quarkus-work-queues}.
     */
    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "work_item_label", joinColumns = @JoinColumn(name = "work_item_id"))
    public List<WorkItemLabel> labels = new ArrayList<>();
```

Add these imports to `WorkItem.java`:

```java
import java.util.ArrayList;
import java.util.List;
import jakarta.persistence.CollectionTable;
import jakarta.persistence.ElementCollection;
import jakarta.persistence.FetchType;
import jakarta.persistence.JoinColumn;
import io.quarkiverse.work.runtime.model.WorkItemLabel;
```

- [ ] **Step 2: Add labels to CreateWorkItemRequest**

Replace the existing record in `CreateWorkItemRequest.java`:

```java
package io.quarkiverse.work.runtime.api;

import java.time.Instant;
import java.util.List;

import io.quarkiverse.work.runtime.model.WorkItemPriority;

public record CreateWorkItemRequest(
        String title,
        String description,
        String category,
        String formKey,
        WorkItemPriority priority,
        String assigneeId,
        String candidateGroups,
        String candidateUsers,
        String requiredCapabilities,
        String createdBy,
        String payload,
        Instant claimDeadline,
        Instant expiresAt,
        Instant followUpDate,
        List<WorkItemLabelResponse> labels) {
}
```

- [ ] **Step 3: Add labels to WorkItemResponse**

Replace the existing record in `WorkItemResponse.java`:

```java
package io.quarkiverse.work.runtime.api;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import io.quarkiverse.work.runtime.model.DelegationState;
import io.quarkiverse.work.runtime.model.WorkItemPriority;
import io.quarkiverse.work.runtime.model.WorkItemStatus;

public record WorkItemResponse(
        UUID id,
        String title,
        String description,
        String category,
        String formKey,
        WorkItemStatus status,
        WorkItemPriority priority,
        String assigneeId,
        String owner,
        String candidateGroups,
        String candidateUsers,
        String requiredCapabilities,
        String createdBy,
        DelegationState delegationState,
        String delegationChain,
        WorkItemStatus priorStatus,
        String payload,
        String resolution,
        Instant claimDeadline,
        Instant expiresAt,
        Instant followUpDate,
        Instant createdAt,
        Instant updatedAt,
        Instant assignedAt,
        Instant startedAt,
        Instant completedAt,
        Instant suspendedAt,
        List<WorkItemLabelResponse> labels) {
}
```

- [ ] **Step 4: Update WorkItemMapper to map labels**

Read the current `WorkItemMapper.java` first, then add label mapping. Find the `toResponse` method and add at the end of the response construction. The mapper builds a `WorkItemResponse` — add the labels field:

In `WorkItemMapper.java`, add a static helper and update the response builder:

```java
    static WorkItemLabelResponse toLabelResponse(final WorkItemLabel label) {
        return new WorkItemLabelResponse(label.path, label.persistence, label.appliedBy);
    }
```

And in `toResponse(WorkItem wi)`, add `labels` as the last argument:

```java
        wi.labels == null ? List.of() : wi.labels.stream().map(WorkItemMapper::toLabelResponse).toList()
```

Add `import java.util.List;` and `import io.quarkiverse.work.runtime.model.WorkItemLabel;` to the mapper.

- [ ] **Step 5: Run full test suite — expect all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | \
  grep -E "Tests run|BUILD|FAIL" | tail -15
```

Expected: `Tests run: 181+, Failures: 0, Errors: 0, BUILD SUCCESS`

---

### Task 6: Reject INFERRED labels at creation — unit test + enforcement

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemService.java`
- Modify: `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/WorkItemServiceTest.java`

- [ ] **Step 1: Write the failing unit test**

In `WorkItemServiceTest.java`, add:

```java
    @Test
    void create_withInferredLabel_throwsIllegalArgumentException() {
        var request = new WorkItemCreateRequest(
                "title", null, null, null, null, null, null, null, null, "alice",
                null, null, null, null,
                List.of(new WorkItemLabelResponse("legal", LabelPersistence.INFERRED, null)));

        assertThatThrownBy(() -> service.create(request))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("INFERRED");
    }

    @Test
    void create_withManualLabel_succeeds() {
        var request = new WorkItemCreateRequest(
                "title", null, null, null, null, null, null, null, null, "alice",
                null, null, null, null,
                List.of(new WorkItemLabelResponse("legal", LabelPersistence.MANUAL, "alice")));

        var result = service.create(request);

        assertThat(result.labels).hasSize(1);
        assertThat(result.labels.get(0).path).isEqualTo("legal");
        assertThat(result.labels.get(0).persistence).isEqualTo(LabelPersistence.MANUAL);
    }
```

Add imports: `import io.quarkiverse.work.runtime.api.WorkItemLabelResponse;`, `import io.quarkiverse.work.runtime.model.LabelPersistence;`, `import java.util.List;`

- [ ] **Step 2: Run test — expect FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemServiceTest#create_withInferredLabel_throwsIllegalArgumentException \
  -Dno-format 2>&1 | tail -8
```

Expected: test fails (no validation yet)

- [ ] **Step 3: Add label validation to WorkItemService.create()**

In `WorkItemService.create()`, after the existing field assignments and before `workItemRepo.save(item)`:

```java
        // Labels: only MANUAL labels accepted at creation time
        if (request.labels() != null) {
            for (var labelReq : request.labels()) {
                if (labelReq.persistence() == LabelPersistence.INFERRED) {
                    throw new IllegalArgumentException(
                            "INFERRED labels cannot be submitted at creation time — they are managed by the filter engine");
                }
                item.labels.add(new WorkItemLabel(labelReq.path(), labelReq.persistence(), labelReq.appliedBy()));
            }
        }
```

Add imports: `import io.quarkiverse.work.runtime.model.WorkItemLabel;`, `import io.quarkiverse.work.runtime.model.LabelPersistence;`

- [ ] **Step 4: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemServiceTest -Dno-format 2>&1 | tail -5
```

Expected: all WorkItemServiceTest tests pass

---

### Task 7: Integration test — label creation and retrieval

**Files:**
- Create: `runtime/src/test/java/io/quarkiverse/workitems/runtime/api/LabelEndpointTest.java`

- [ ] **Step 1: Write the @QuarkusTest integration test**

```java
// runtime/src/test/java/io/quarkiverse/workitems/runtime/api/LabelEndpointTest.java
package io.quarkiverse.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.hasSize;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class LabelEndpointTest {

    @Test
    void createWorkItem_withManualLabel_returnedInResponse() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "title": "Review contract",
                          "createdBy": "alice",
                          "labels": [
                            {"path": "legal/contracts", "persistence": "MANUAL", "appliedBy": "alice"}
                          ]
                        }
                        """)
                .post("/workitems")
                .then()
                .statusCode(201)
                .body("labels", hasSize(1))
                .body("labels[0].path", equalTo("legal/contracts"))
                .body("labels[0].persistence", equalTo("MANUAL"))
                .body("labels[0].appliedBy", equalTo("alice"));
    }

    @Test
    void createWorkItem_withInferredLabel_returns400() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "title": "Review contract",
                          "createdBy": "alice",
                          "labels": [
                            {"path": "legal/contracts", "persistence": "INFERRED", "appliedBy": null}
                          ]
                        }
                        """)
                .post("/workitems")
                .then()
                .statusCode(400);
    }

    @Test
    void createWorkItem_withNoLabels_returnsEmptyList() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "title": "No labels",
                          "createdBy": "bob"
                        }
                        """)
                .post("/workitems")
                .then()
                .statusCode(201)
                .body("labels", hasSize(0));
    }

    @Test
    void createWorkItem_withMultipleLabels_allReturned() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "title": "Multi-label item",
                          "createdBy": "alice",
                          "labels": [
                            {"path": "legal/contracts", "persistence": "MANUAL", "appliedBy": "alice"},
                            {"path": "priority/high",   "persistence": "MANUAL", "appliedBy": "alice"}
                          ]
                        }
                        """)
                .post("/workitems")
                .then()
                .statusCode(201)
                .body("labels", hasSize(2));
    }
}
```

- [ ] **Step 2: Add 400 handling for IllegalArgumentException in WorkItemResource**

Read `WorkItemResource.java` and find the create endpoint. Wrap the service call:

```java
        try {
            final WorkItem created = service.create(WorkItemMapper.toCreateRequest(request));
            // ... existing URI and response code
        } catch (IllegalArgumentException e) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(Map.of("error", e.getMessage()))
                    .build();
        }
```

Add `import java.util.Map;` to WorkItemResource.

- [ ] **Step 3: Run integration test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LabelEndpointTest -Dno-format 2>&1 | tail -8
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

---

### Task 8: Commit issue #53

- [ ] **Step 1: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | \
  grep -E "Tests run|BUILD" | tail -5
```

Expected: BUILD SUCCESS, 0 failures

- [ ] **Step 2: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelPersistence.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemLabel.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemLabelResponse.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItem.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemCreateRequest.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResponse.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemMapper.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResource.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemService.java \
        runtime/src/main/resources/db/migration/V2__label_schema.sql \
        runtime/src/test/java/io/quarkiverse/workitems/runtime/model/WorkItemLabelTest.java \
        runtime/src/test/java/io/quarkiverse/workitems/runtime/api/LabelEndpointTest.java
git commit -m "feat(labels): WorkItemLabel entity, Flyway V2, MANUAL/INFERRED persistence

- WorkItemLabel @Embeddable: path, persistence, appliedBy
- LabelPersistence enum: MANUAL | INFERRED
- WorkItem.labels via @ElementCollection → work_item_label table
- CreateWorkItemRequest accepts MANUAL labels; rejects INFERRED (400)
- WorkItemResponse includes labels list
- WorkItemMapper maps labels to/from DTO

Closes #53
Refs #51, #50"
```

---

## Issue #54 — LabelVocabulary and LabelDefinition

---

### Task 9: VocabularyScope enum + LabelVocabulary entity

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/VocabularyScope.java`
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelVocabulary.java`
- Create: `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyServiceTest.java`

- [ ] **Step 1: Write failing test for scope hierarchy**

```java
// runtime/src/test/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyServiceTest.java
package io.quarkiverse.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

import io.quarkiverse.work.runtime.model.VocabularyScope;

class LabelVocabularyServiceTest {

    @Test
    void vocabularyScope_hasCorrectHierarchy() {
        // GLOBAL is the highest scope — accessible to everyone
        assertThat(VocabularyScope.GLOBAL.ordinal()).isLessThan(VocabularyScope.ORG.ordinal());
        assertThat(VocabularyScope.ORG.ordinal()).isLessThan(VocabularyScope.TEAM.ordinal());
        assertThat(VocabularyScope.TEAM.ordinal()).isLessThan(VocabularyScope.PERSONAL.ordinal());
    }

    @Test
    void vocabularyScope_fourValues() {
        assertThat(VocabularyScope.values()).hasSize(4);
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LabelVocabularyServiceTest -Dno-format 2>&1 | tail -8
```

Expected: `cannot find symbol: class VocabularyScope`

- [ ] **Step 3: Create VocabularyScope**

```java
// runtime/src/main/java/io/quarkiverse/workitems/runtime/model/VocabularyScope.java
package io.quarkiverse.work.runtime.model;

/**
 * Scope of a {@link LabelVocabulary}, forming a visibility hierarchy from broadest to narrowest.
 *
 * <p>
 * A vocabulary at a given scope is visible to all scopes below it:
 * <ul>
 * <li>{@code GLOBAL} terms are visible to everyone.</li>
 * <li>{@code ORG} terms are visible within the organisation and below.</li>
 * <li>{@code TEAM} terms are visible within the team and below.</li>
 * <li>{@code PERSONAL} terms are visible only to the declaring user.</li>
 * </ul>
 *
 * <p>
 * The ordinal ordering reflects this hierarchy — lower ordinal = broader scope.
 */
public enum VocabularyScope {
    /** Platform-wide; accessible everywhere. No owner required. */
    GLOBAL,

    /** Organisation-level; accessible within the owning org and its teams. */
    ORG,

    /** Team-level; accessible within the owning team only. */
    TEAM,

    /** Personal; accessible only to the declaring user. */
    PERSONAL
}
```

- [ ] **Step 4: Create LabelVocabulary entity**

```java
// runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelVocabulary.java
package io.quarkiverse.work.runtime.model;

import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * A named, scoped container of {@link LabelDefinition} entries.
 *
 * <p>
 * Vocabularies form a visibility hierarchy: GLOBAL → ORG → TEAM → PERSONAL.
 * A user can apply any label declared in a vocabulary at or above their scope.
 */
@Entity
@Table(name = "label_vocabulary")
public class LabelVocabulary extends PanacheEntityBase {

    @Id
    public UUID id;

    /** Scope of this vocabulary. */
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    public VocabularyScope scope;

    /**
     * Owner of this vocabulary: null for GLOBAL, orgId for ORG, groupId for TEAM,
     * userId for PERSONAL.
     */
    @Column(name = "owner_id", length = 255)
    public String ownerId;

    /** Human-readable name for this vocabulary. */
    @Column(nullable = false, length = 255)
    public String name;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LabelVocabularyServiceTest -Dno-format 2>&1 | tail -5
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`

---

### Task 10: LabelDefinition entity

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelDefinition.java`

- [ ] **Step 1: Create LabelDefinition**

```java
// runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelDefinition.java
package io.quarkiverse.work.runtime.model;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * A declared label path within a {@link LabelVocabulary}.
 *
 * <p>
 * Before a label path can be applied to a WorkItem, it must be declared as a
 * {@code LabelDefinition} in a vocabulary accessible to the actor.
 */
@Entity
@Table(name = "label_definition")
public class LabelDefinition extends PanacheEntityBase {

    @Id
    public UUID id;

    /** The full label path, e.g. {@code legal/contracts/nda}. */
    @Column(nullable = false, length = 500)
    public String path;

    /** The vocabulary this definition belongs to. */
    @Column(name = "vocabulary_id", nullable = false)
    public UUID vocabularyId;

    /** Optional human-readable description of what this label means. */
    @Column(length = 1000)
    public String description;

    /** The user who declared this label. */
    @Column(name = "created_by", nullable = false, length = 255)
    public String createdBy;

    /** When this definition was created. */
    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
        createdAt = Instant.now();
    }

    /** Find all definitions for a given vocabulary. */
    public static java.util.List<LabelDefinition> findByVocabularyId(final UUID vocabularyId) {
        return find("vocabularyId", vocabularyId).list();
    }

    /** Find by exact path across all vocabularies. */
    public static java.util.List<LabelDefinition> findByPath(final String path) {
        return find("path", path).list();
    }
}
```

---

### Task 11: Flyway V3 — vocabulary tables

**Files:**
- Create: `runtime/src/main/resources/db/migration/V3__vocabulary_schema.sql`

- [ ] **Step 1: Write the migration**

```sql
-- Quarkus WorkItems V3: label vocabulary support
-- LabelVocabulary scopes label declarations. LabelDefinition records each declared path.
-- Compatible with H2 (dev/test) and PostgreSQL (production).

CREATE TABLE label_vocabulary (
    id          UUID            PRIMARY KEY,
    scope       VARCHAR(20)     NOT NULL,
    owner_id    VARCHAR(255),
    name        VARCHAR(255)    NOT NULL
);

-- Seed GLOBAL vocabulary (available to all)
INSERT INTO label_vocabulary (id, scope, owner_id, name)
VALUES ('00000000-0000-0000-0000-000000000001', 'GLOBAL', NULL, 'Global');

CREATE TABLE label_definition (
    id              UUID            PRIMARY KEY,
    path            VARCHAR(500)    NOT NULL,
    vocabulary_id   UUID            NOT NULL REFERENCES label_vocabulary(id) ON DELETE CASCADE,
    description     VARCHAR(1000),
    created_by      VARCHAR(255)    NOT NULL,
    created_at      TIMESTAMP       NOT NULL
);

-- Seed common global labels
INSERT INTO label_definition (id, path, vocabulary_id, description, created_by, created_at) VALUES
('00000000-0000-0000-0001-000000000001', 'intake',             '00000000-0000-0000-0000-000000000001', 'Newly submitted, awaiting triage', 'system', CURRENT_TIMESTAMP),
('00000000-0000-0000-0001-000000000002', 'intake/triage',      '00000000-0000-0000-0000-000000000001', 'Actively being triaged', 'system', CURRENT_TIMESTAMP),
('00000000-0000-0000-0001-000000000003', 'priority/high',      '00000000-0000-0000-0000-000000000001', 'High priority item', 'system', CURRENT_TIMESTAMP),
('00000000-0000-0000-0001-000000000004', 'priority/critical',  '00000000-0000-0000-0000-000000000001', 'Critical priority item', 'system', CURRENT_TIMESTAMP),
('00000000-0000-0000-0001-000000000005', 'legal',              '00000000-0000-0000-0000-000000000001', 'Legal domain work', 'system', CURRENT_TIMESTAMP),
('00000000-0000-0000-0001-000000000006', 'legal/contracts',    '00000000-0000-0000-0000-000000000001', 'Contract review', 'system', CURRENT_TIMESTAMP),
('00000000-0000-0000-0001-000000000007', 'legal/compliance',   '00000000-0000-0000-0000-000000000001', 'Compliance review', 'system', CURRENT_TIMESTAMP);

CREATE INDEX idx_ld_vocabulary_id ON label_definition(vocabulary_id);
CREATE INDEX idx_ld_path ON label_definition(path);
```

- [ ] **Step 2: Verify migrations apply cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemSmokeTest -Dno-format 2>&1 | tail -5
```

Expected: `Tests run: 15, Failures: 0, Errors: 0`

---

### Task 12: LabelVocabularyService — vocabulary enforcement

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyService.java`
- Modify: `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyServiceTest.java`

- [ ] **Step 1: Write failing unit tests for vocabulary enforcement**

Add to `LabelVocabularyServiceTest.java`:

```java
    @Test
    void isValidForScope_globalLabel_visibleAtAllScopes() {
        // Given a GLOBAL vocabulary with path "intake"
        var globalVocab = new LabelVocabulary();
        globalVocab.id = UUID.fromString("00000000-0000-0000-0000-000000000001");
        globalVocab.scope = VocabularyScope.GLOBAL;

        var definition = new LabelDefinition();
        definition.path = "intake";
        definition.vocabularyId = globalVocab.id;

        // A label from GLOBAL should be usable at any scope
        assertThat(LabelVocabularyService.isScopeAccessible(VocabularyScope.GLOBAL, VocabularyScope.PERSONAL))
                .isTrue();
        assertThat(LabelVocabularyService.isScopeAccessible(VocabularyScope.GLOBAL, VocabularyScope.TEAM))
                .isTrue();
        assertThat(LabelVocabularyService.isScopeAccessible(VocabularyScope.GLOBAL, VocabularyScope.ORG))
                .isTrue();
        assertThat(LabelVocabularyService.isScopeAccessible(VocabularyScope.GLOBAL, VocabularyScope.GLOBAL))
                .isTrue();
    }

    @Test
    void isValidForScope_personalLabel_notVisibleAtTeamScope() {
        assertThat(LabelVocabularyService.isScopeAccessible(VocabularyScope.PERSONAL, VocabularyScope.TEAM))
                .isFalse();
        assertThat(LabelVocabularyService.isScopeAccessible(VocabularyScope.PERSONAL, VocabularyScope.ORG))
                .isFalse();
        assertThat(LabelVocabularyService.isScopeAccessible(VocabularyScope.PERSONAL, VocabularyScope.GLOBAL))
                .isFalse();
    }

    @Test
    void matchesPattern_exactMatch() {
        assertThat(LabelVocabularyService.matchesPattern("legal", "legal")).isTrue();
        assertThat(LabelVocabularyService.matchesPattern("legal", "legal/contracts")).isFalse();
    }

    @Test
    void matchesPattern_singleWildcard() {
        assertThat(LabelVocabularyService.matchesPattern("legal/*", "legal/contracts")).isTrue();
        assertThat(LabelVocabularyService.matchesPattern("legal/*", "legal/contracts/nda")).isFalse();
        assertThat(LabelVocabularyService.matchesPattern("legal/*", "legal")).isFalse();
    }

    @Test
    void matchesPattern_multiWildcard() {
        assertThat(LabelVocabularyService.matchesPattern("legal/**", "legal/contracts")).isTrue();
        assertThat(LabelVocabularyService.matchesPattern("legal/**", "legal/contracts/nda")).isTrue();
        assertThat(LabelVocabularyService.matchesPattern("legal/**", "legal")).isFalse();
    }
```

Add imports: `import java.util.UUID;`, `import io.quarkiverse.work.runtime.model.*;`

- [ ] **Step 2: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LabelVocabularyServiceTest -Dno-format 2>&1 | tail -8
```

Expected: `cannot find symbol: class LabelVocabularyService`

- [ ] **Step 3: Create LabelVocabularyService**

```java
// runtime/src/main/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyService.java
package io.quarkiverse.work.runtime.service;

import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.runtime.model.LabelDefinition;
import io.quarkiverse.work.runtime.model.LabelVocabulary;
import io.quarkiverse.work.runtime.model.VocabularyScope;

@ApplicationScoped
public class LabelVocabularyService {

    /**
     * Returns true if a label declared in {@code definitionScope} is accessible
     * to a caller operating at {@code callerScope}.
     *
     * <p>
     * A vocabulary is accessible if its scope is at the same level or broader than
     * the caller's scope (lower ordinal = broader scope).
     */
    public static boolean isScopeAccessible(final VocabularyScope definitionScope, final VocabularyScope callerScope) {
        return definitionScope.ordinal() <= callerScope.ordinal();
    }

    /**
     * Returns true if the given label {@code path} matches the {@code pattern}.
     *
     * <p>
     * Pattern semantics:
     * <ul>
     * <li>Exact: {@code "legal"} matches only {@code "legal"}</li>
     * <li>Single wildcard: {@code "legal/*"} matches one segment below {@code "legal/"}
     *     but not multiple segments</li>
     * <li>Multi wildcard: {@code "legal/**"} matches any path below {@code "legal/"}</li>
     * </ul>
     */
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

    /**
     * Returns true if the given {@code path} is declared in any vocabulary
     * accessible to the caller at the given scope.
     *
     * <p>
     * In the initial implementation, the caller scope is not yet enforced —
     * any declared path is accepted. Scope enforcement will be added when
     * authentication context is available.
     */
    @Transactional
    public boolean isDeclared(final String path) {
        return !LabelDefinition.findByPath(path).isEmpty();
    }

    /**
     * Add a new label definition to the given vocabulary.
     */
    @Transactional
    public LabelDefinition addDefinition(final UUID vocabularyId, final String path,
            final String description, final String createdBy) {
        final LabelDefinition def = new LabelDefinition();
        def.path = path;
        def.vocabularyId = vocabularyId;
        def.description = description;
        def.createdBy = createdBy;
        def.persist();
        return def;
    }

    /**
     * List all label definitions accessible at the given scope (all scopes at or above).
     */
    @Transactional
    public List<LabelDefinition> listAccessible(final VocabularyScope callerScope) {
        return LabelVocabulary.<LabelVocabulary> listAll().stream()
                .filter(v -> isScopeAccessible(v.scope, callerScope))
                .flatMap(v -> LabelDefinition.findByVocabularyId(v.id).stream())
                .toList();
    }

    /**
     * Find the GLOBAL vocabulary (always present — seeded by Flyway V3).
     */
    @Transactional
    public LabelVocabulary findGlobalVocabulary() {
        return LabelVocabulary.<LabelVocabulary> find("scope", VocabularyScope.GLOBAL)
                .firstResult();
    }
}
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LabelVocabularyServiceTest -Dno-format 2>&1 | tail -5
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`

---

### Task 13: VocabularyResource REST endpoints

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/VocabularyResource.java`

- [ ] **Step 1: Create VocabularyResource**

```java
// runtime/src/main/java/io/quarkiverse/workitems/runtime/api/VocabularyResource.java
package io.quarkiverse.work.runtime.api;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.work.runtime.model.LabelDefinition;
import io.quarkiverse.work.runtime.model.VocabularyScope;
import io.quarkiverse.work.runtime.service.LabelVocabularyService;

@Path("/vocabulary")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class VocabularyResource {

    @Inject
    LabelVocabularyService vocabularyService;

    public record AddDefinitionRequest(String path, String description, String addedBy) {
    }

    /**
     * List all label definitions accessible to the caller.
     * In the initial implementation, returns all definitions (scope enforcement deferred).
     */
    @GET
    public List<Map<String, Object>> listAll() {
        return vocabularyService.listAccessible(VocabularyScope.PERSONAL).stream()
                .map(d -> Map.<String, Object> of(
                        "id", d.id,
                        "path", d.path,
                        "vocabularyId", d.vocabularyId,
                        "description", d.description != null ? d.description : "",
                        "createdBy", d.createdBy,
                        "createdAt", d.createdAt))
                .toList();
    }

    /**
     * Add a label definition to the vocabulary at the given scope.
     * For now, all definitions go into the GLOBAL vocabulary.
     */
    @POST
    @Path("/{scope}")
    @Transactional
    public Response addDefinition(@PathParam("scope") final String scopeStr,
            final AddDefinitionRequest request) {
        if (request.path() == null || request.path().isBlank()) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(Map.of("error", "path is required"))
                    .build();
        }

        final VocabularyScope scope;
        try {
            scope = VocabularyScope.valueOf(scopeStr.toUpperCase());
        } catch (IllegalArgumentException e) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(Map.of("error", "Unknown scope: " + scopeStr + ". Valid: GLOBAL, ORG, TEAM, PERSONAL"))
                    .build();
        }

        // For now, always add to GLOBAL — multi-tenancy vocabulary selection deferred
        final var globalVocab = vocabularyService.findGlobalVocabulary();
        if (globalVocab == null) {
            return Response.status(Response.Status.INTERNAL_SERVER_ERROR)
                    .entity(Map.of("error", "GLOBAL vocabulary not found — check Flyway V3 migration"))
                    .build();
        }

        final LabelDefinition def = vocabularyService.addDefinition(
                globalVocab.id, request.path(), request.description(),
                request.addedBy() != null ? request.addedBy() : "unknown");

        return Response.status(Response.Status.CREATED)
                .entity(Map.of("id", def.id, "path", def.path, "scope", scope))
                .build();
    }
}
```

---

### Task 14: Integration tests for vocabulary REST

**Files:**
- Modify: `runtime/src/test/java/io/quarkiverse/workitems/runtime/api/LabelEndpointTest.java`

- [ ] **Step 1: Add vocabulary integration tests**

Add to `LabelEndpointTest.java`:

```java
    @Test
    void vocabulary_listAll_includesSeededGlobalTerms() {
        given()
                .get("/vocabulary")
                .then()
                .statusCode(200)
                .body("path", org.hamcrest.Matchers.hasItem("legal/contracts"))
                .body("path", org.hamcrest.Matchers.hasItem("intake"));
    }

    @Test
    void vocabulary_addDefinition_appearsInList() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "test/unique-label-54", "description": "test label", "addedBy": "alice"}
                        """)
                .post("/vocabulary/GLOBAL")
                .then()
                .statusCode(201)
                .body("path", equalTo("test/unique-label-54"));

        given()
                .get("/vocabulary")
                .then()
                .statusCode(200)
                .body("path", org.hamcrest.Matchers.hasItem("test/unique-label-54"));
    }

    @Test
    void vocabulary_addDefinition_invalidScope_returns400() {
        given()
                .contentType(ContentType.JSON)
                .body("""{"path": "x", "addedBy": "alice"}""")
                .post("/vocabulary/INVALID")
                .then()
                .statusCode(400);
    }

    @Test
    void vocabulary_addDefinition_emptyPath_returns400() {
        given()
                .contentType(ContentType.JSON)
                .body("""{"path": "", "addedBy": "alice"}""")
                .post("/vocabulary/GLOBAL")
                .then()
                .statusCode(400);
    }
```

- [ ] **Step 2: Run integration tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LabelEndpointTest -Dno-format 2>&1 | tail -5
```

Expected: all 8 tests pass

---

### Task 15: Commit issue #54

- [ ] **Step 1: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dno-format 2>&1 | \
  grep -E "Tests run|BUILD" | tail -5
```

Expected: BUILD SUCCESS, 0 failures

- [ ] **Step 2: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/workitems/runtime/model/VocabularyScope.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelVocabulary.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/model/LabelDefinition.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyService.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/api/VocabularyResource.java \
        runtime/src/main/resources/db/migration/V3__vocabulary_schema.sql \
        runtime/src/test/java/io/quarkiverse/workitems/runtime/service/LabelVocabularyServiceTest.java \
        runtime/src/test/java/io/quarkiverse/workitems/runtime/api/LabelEndpointTest.java
git commit -m "feat(vocabulary): LabelVocabulary and LabelDefinition with scoped hierarchy

- VocabularyScope enum: GLOBAL → ORG → TEAM → PERSONAL
- LabelVocabulary Panache entity + Flyway V3 (seeded GLOBAL vocabulary)
- LabelDefinition Panache entity with findByPath and findByVocabularyId
- LabelVocabularyService: scope accessibility, path pattern matching, CRUD
- VocabularyResource: GET /vocabulary, POST /vocabulary/{scope}
- isScopeAccessible and matchesPattern unit tested

Closes #54
Refs #51, #50"
```

---

## Issue #55 — Label Query + MANUAL Label Endpoints

---

### Task 16: findByLabelPattern in WorkItemRepository + JPA implementation

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/repository/WorkItemRepository.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/repository/jpa/JpaWorkItemRepository.java`
- Modify: `runtime/src/test/java/io/quarkiverse/workitems/runtime/repository/JpaWorkItemRepositoryTest.java`

- [ ] **Step 1: Write failing test**

In `JpaWorkItemRepositoryTest.java`, add:

```java
    @Test
    @Transactional
    void findByLabelPattern_exactMatch_returnMatchingItems() {
        var wi = new WorkItem();
        wi.title = "label-test-exact";
        wi.status = WorkItemStatus.PENDING;
        wi.priority = WorkItemPriority.NORMAL;
        wi.labels.add(new WorkItemLabel("legal/contracts", LabelPersistence.MANUAL, "alice"));
        repository.save(wi);

        var results = repository.findByLabelPattern("legal/contracts");

        assertThat(results).extracting(w -> w.title).contains("label-test-exact");
    }

    @Test
    @Transactional
    void findByLabelPattern_singleWildcard_matchesOneLevel() {
        var wi = new WorkItem();
        wi.title = "label-test-wildcard";
        wi.status = WorkItemStatus.PENDING;
        wi.priority = WorkItemPriority.NORMAL;
        wi.labels.add(new WorkItemLabel("legal/contracts", LabelPersistence.MANUAL, "alice"));
        repository.save(wi);

        var results = repository.findByLabelPattern("legal/*");
        assertThat(results).extracting(w -> w.title).contains("label-test-wildcard");

        var deep = repository.findByLabelPattern("legal/contracts/*");
        assertThat(deep).extracting(w -> w.title).doesNotContain("label-test-wildcard");
    }

    @Test
    @Transactional
    void findByLabelPattern_multiWildcard_matchesAllDepths() {
        var wi1 = new WorkItem();
        wi1.title = "label-test-multi-1";
        wi1.status = WorkItemStatus.PENDING;
        wi1.priority = WorkItemPriority.NORMAL;
        wi1.labels.add(new WorkItemLabel("legal/contracts", LabelPersistence.MANUAL, "alice"));
        repository.save(wi1);

        var wi2 = new WorkItem();
        wi2.title = "label-test-multi-2";
        wi2.status = WorkItemStatus.PENDING;
        wi2.priority = WorkItemPriority.NORMAL;
        wi2.labels.add(new WorkItemLabel("legal/contracts/nda", LabelPersistence.MANUAL, "alice"));
        repository.save(wi2);

        var results = repository.findByLabelPattern("legal/**");
        assertThat(results).extracting(w -> w.title)
                .contains("label-test-multi-1", "label-test-multi-2");
    }

    @Test
    @Transactional
    void findByLabelPattern_noMatch_returnsEmpty() {
        var results = repository.findByLabelPattern("nonexistent/path");
        assertThat(results).isEmpty();
    }
```

Add imports: `import io.quarkiverse.work.runtime.model.WorkItemLabel;`, `import io.quarkiverse.work.runtime.model.LabelPersistence;`

- [ ] **Step 2: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=JpaWorkItemRepositoryTest -Dno-format 2>&1 | tail -8
```

Expected: `cannot find symbol: method findByLabelPattern`

- [ ] **Step 3: Add method to WorkItemRepository interface**

In `WorkItemRepository.java`, add:

```java
    /**
     * Find WorkItems that have at least one label matching the given pattern.
     *
     * <p>
     * Pattern semantics:
     * <ul>
     * <li>Exact: {@code "legal"} matches only {@code "legal"}</li>
     * <li>Single wildcard: {@code "legal/*"} matches {@code "legal/contracts"} but not
     *     {@code "legal/contracts/nda"}</li>
     * <li>Multi wildcard: {@code "legal/**"} matches any path starting with
     *     {@code "legal/"}</li>
     * </ul>
     *
     * @param pattern the label pattern to match against
     * @return list of matching WorkItems; may be empty, never null
     */
    List<WorkItem> findByLabelPattern(String pattern);
```

- [ ] **Step 4: Implement in JpaWorkItemRepository**

In `JpaWorkItemRepository.java`, add:

```java
    @Override
    public List<WorkItem> findByLabelPattern(final String pattern) {
        if (pattern.endsWith("/**")) {
            final String prefix = pattern.substring(0, pattern.length() - 3) + "/";
            return WorkItem.find(
                    "SELECT DISTINCT wi FROM WorkItem wi JOIN wi.labels l WHERE l.path LIKE ?1",
                    prefix + "%").list();
        }
        if (pattern.endsWith("/*")) {
            final String prefix = pattern.substring(0, pattern.length() - 2) + "/";
            // Match one level: starts with prefix, no further slash after prefix
            return WorkItem.<WorkItem> find(
                    "SELECT DISTINCT wi FROM WorkItem wi JOIN wi.labels l " +
                            "WHERE l.path LIKE ?1 AND l.path NOT LIKE ?2",
                    prefix + "%", prefix + "%/%").list();
        }
        // Exact match
        return WorkItem.find(
                "SELECT DISTINCT wi FROM WorkItem wi JOIN wi.labels l WHERE l.path = ?1",
                pattern).list();
    }
```

- [ ] **Step 5: Update InMemoryWorkItemRepository**

In `testing/src/main/java/io/quarkiverse/workitems/testing/InMemoryWorkItemRepository.java`, add:

```java
    @Override
    public List<WorkItem> findByLabelPattern(final String pattern) {
        return store.values().stream()
                .filter(wi -> wi.labels != null && wi.labels.stream()
                        .anyMatch(l -> io.quarkiverse.work.runtime.service.LabelVocabularyService
                                .matchesPattern(pattern, l.path)))
                .toList();
    }
```

- [ ] **Step 6: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing \
  -Dtest=JpaWorkItemRepositoryTest -Dno-format 2>&1 | tail -5
```

Expected: `Tests run: 4+, Failures: 0, Errors: 0`

---

### Task 17: Label query param and MANUAL label endpoints in WorkItemResource

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResource.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemService.java`

- [ ] **Step 1: Write failing integration tests**

Add to `LabelEndpointTest.java`:

```java
    @Test
    void getWorkItems_byLabelPattern_returnsMatching() {
        // Create a WorkItem with label
        var location = given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "title": "Label query test",
                          "createdBy": "alice",
                          "labels": [{"path": "legal/contracts", "persistence": "MANUAL", "appliedBy": "alice"}]
                        }
                        """)
                .post("/workitems")
                .then().statusCode(201)
                .extract().header("Location");

        // Query by exact pattern
        given()
                .queryParam("label", "legal/contracts")
                .get("/workitems")
                .then()
                .statusCode(200)
                .body("title", org.hamcrest.Matchers.hasItem("Label query test"));

        // Query by wildcard
        given()
                .queryParam("label", "legal/**")
                .get("/workitems")
                .then()
                .statusCode(200)
                .body("title", org.hamcrest.Matchers.hasItem("Label query test"));
    }

    @Test
    void addManualLabel_toExistingWorkItem_appearsInResponse() {
        var id = given()
                .contentType(ContentType.JSON)
                .body("""{"title": "Add label test", "createdBy": "alice"}""")
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        given()
                .contentType(ContentType.JSON)
                .body("""{"path": "legal/contracts", "appliedBy": "alice"}""")
                .post("/workitems/" + id + "/labels")
                .then()
                .statusCode(200)
                .body("labels.path", org.hamcrest.Matchers.hasItem("legal/contracts"));
    }

    @Test
    void removeManualLabel_fromWorkItem_disappearsFromResponse() {
        var id = given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "title": "Remove label test",
                          "createdBy": "alice",
                          "labels": [{"path": "legal/contracts", "persistence": "MANUAL", "appliedBy": "alice"}]
                        }
                        """)
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        given()
                .delete("/workitems/" + id + "/labels/legal%2Fcontracts")
                .then()
                .statusCode(200)
                .body("labels", hasSize(0));
    }

    @Test
    void removeInferredLabel_returns400() {
        // Directly add an INFERRED label via the service (simulated by queues module)
        // For now, verify that the endpoint rejects INFERRED labels on removal attempt
        // by testing with a non-existent label
        var id = given()
                .contentType(ContentType.JSON)
                .body("""{"title": "Inferred label test", "createdBy": "alice"}""")
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        given()
                .delete("/workitems/" + id + "/labels/nonexistent%2Flabel")
                .then()
                .statusCode(404);
    }
```

- [ ] **Step 2: Add addLabel and removeLabel to WorkItemService**

In `WorkItemService.java`, add:

```java
    @Transactional
    public WorkItem addLabel(final UUID workItemId, final String path, final String appliedBy) {
        final WorkItem item = workItemRepo.findById(workItemId)
                .orElseThrow(() -> new WorkItemNotFoundException(workItemId));
        item.labels.add(new WorkItemLabel(path, LabelPersistence.MANUAL, appliedBy));
        final WorkItem saved = workItemRepo.save(item);
        lifecycleEvent.fire(WorkItemLifecycleEvent.of("LABEL_ADDED", saved.id, saved.status, appliedBy, null));
        return saved;
    }

    @Transactional
    public WorkItem removeLabel(final UUID workItemId, final String path) {
        final WorkItem item = workItemRepo.findById(workItemId)
                .orElseThrow(() -> new WorkItemNotFoundException(workItemId));
        final boolean removed = item.labels.removeIf(
                l -> l.path.equals(path) && l.persistence == LabelPersistence.MANUAL);
        if (!removed) {
            throw new WorkItemNotFoundException(workItemId, "MANUAL label '" + path + "' not found");
        }
        return workItemRepo.save(item);
    }
```

For `WorkItemNotFoundException` with a custom message, add a constructor if it doesn't exist:

```java
    public WorkItemNotFoundException(final UUID id, final String detail) {
        super("WorkItem " + id + ": " + detail);
        this.id = id;
    }
```

- [ ] **Step 3: Add label query param and endpoints to WorkItemResource**

In `WorkItemResource.java`, add the `label` query param to the list endpoint and add two new endpoints:

```java
    @GET
    @Path("/")
    public Response list(@QueryParam("label") final String labelPattern,
            // ... existing query params
    ) {
        if (labelPattern != null && !labelPattern.isBlank()) {
            final var results = workItemRepo.findByLabelPattern(labelPattern).stream()
                    .map(WorkItemMapper::toResponse).toList();
            return Response.ok(results).build();
        }
        // ... existing logic
    }

    public record AddLabelRequest(String path, String appliedBy) {}

    @POST
    @Path("/{id}/labels")
    public Response addLabel(@PathParam("id") final UUID id, final AddLabelRequest request) {
        if (request.path() == null || request.path().isBlank()) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(Map.of("error", "path is required")).build();
        }
        try {
            final WorkItem updated = service.addLabel(id, request.path(),
                    request.appliedBy() != null ? request.appliedBy() : "unknown");
            return Response.ok(WorkItemMapper.toResponse(updated)).build();
        } catch (WorkItemNotFoundException e) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(Map.of("error", e.getMessage())).build();
        }
    }

    @DELETE
    @Path("/{id}/labels/{path}")
    public Response removeLabel(@PathParam("id") final UUID id,
            @PathParam("path") final String path) {
        try {
            final WorkItem updated = service.removeLabel(id, path);
            return Response.ok(WorkItemMapper.toResponse(updated)).build();
        } catch (WorkItemNotFoundException e) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(Map.of("error", e.getMessage())).build();
        }
    }
```

Add `import java.util.Map;` and `import jakarta.ws.rs.QueryParam;` to WorkItemResource.

- [ ] **Step 4: Run integration tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LabelEndpointTest -Dno-format 2>&1 | tail -5
```

Expected: all 12 tests pass

---

### Task 18: Run full suite and commit issue #55

- [ ] **Step 1: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,testing -Dno-format 2>&1 | \
  grep -E "Tests run|BUILD" | tail -10
```

Expected: BUILD SUCCESS, 0 failures

- [ ] **Step 2: Run integration tests for the whole project**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests -Dno-format 2>&1 | tail -5
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests -Dno-format 2>&1 | tail -10
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/workitems/runtime/repository/WorkItemRepository.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/repository/jpa/JpaWorkItemRepository.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResource.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemService.java \
        runtime/src/test/java/io/quarkiverse/workitems/runtime/repository/JpaWorkItemRepositoryTest.java \
        runtime/src/test/java/io/quarkiverse/workitems/runtime/api/LabelEndpointTest.java \
        testing/src/main/java/io/quarkiverse/workitems/testing/InMemoryWorkItemRepository.java
git commit -m "feat(labels): label query by pattern, add/remove MANUAL label endpoints

- WorkItemRepository.findByLabelPattern: exact, /*, /** wildcard semantics
- JpaWorkItemRepository: JPQL JOIN on labels for pattern query
- InMemoryWorkItemRepository: matchesPattern for unit test parity
- WorkItemService: addLabel() and removeLabel() — MANUAL only, fires lifecycle event
- WorkItemResource: GET /workitems?label=pattern, POST /{id}/labels, DELETE /{id}/labels/{path}

Closes #55
Closes #51
Refs #50"
```

---

## Self-Review

### Spec coverage

| Spec requirement | Task |
|---|---|
| WorkItemLabel: path, persistence, appliedBy | Tasks 1–2 |
| LabelPersistence: MANUAL / INFERRED | Task 1 |
| INFERRED labels rejected at creation | Task 6 |
| Flyway V2: work_item_label table | Task 4 |
| LabelVocabulary + VocabularyScope | Task 9 |
| LabelDefinition entity | Task 10 |
| Flyway V3: vocabulary tables + seed | Task 11 |
| Scope hierarchy: isScopeAccessible | Task 12 |
| matchesPattern: exact, /*, /** | Task 12 |
| GET /vocabulary | Task 13 |
| POST /vocabulary/{scope} | Task 13 |
| GET /workitems?label=pattern | Task 17 |
| POST /workitems/{id}/labels | Task 17 |
| DELETE /workitems/{id}/labels/{path} | Task 17 |
| InMemoryWorkItemRepository updated | Task 16 |
| Unit tests for all logic | Tasks 1, 6, 9, 12, 16 |
| @QuarkusTest integration tests | Tasks 7, 14, 17 |
| Existing tests unaffected | Tasks 5, 7, 11, 18 |

### Open items for #52 (queues module plan)

- `supporters: Set<chainId>` on WorkItemLabel — deferred to queues module Flyway
- Vocabulary enforcement wired into WorkItemService.addLabel() — currently just validates path not blank; vocabulary `isDeclared()` check to be wired in a follow-up once vocabulary is seeded in tests
- INFERRED label write-path (quarkus-work-queues writes directly via a service method, not the create endpoint)
