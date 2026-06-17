# MongoDB Persistence Completion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement all 11 remaining MongoDB store SPIs so `persistence-mongodb` is a true drop-in replacement for the JPA stores.

**Architecture:** Each store follows the established pattern: a `@MongoEntity` Panache document class with `from()` / `toDomain()` conversion, and a `@ApplicationScoped @Alternative @Priority(1)` store class injecting `CurrentPrincipal` for tenant scoping. Two stores (WorkItemSchedule, WorkItemSpawnGroup) require OCC via raw `findOneAndUpdate` with version-check filter. RoutingCursor requires atomic `findOneAndUpdate` with `$inc`. A `@Startup` bean creates unique indexes at boot.

**Tech Stack:** Java 21, Quarkus 3.32.2, quarkus-mongodb-panache, JUnit 5 / @QuarkusTest, AssertJ

**Spec:** `specs/2026-06-17-mongodb-persistence-completion-design.md`

**Build:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl persistence-mongodb -Dquarkus.test.arg-line=--add-opens=java.base/java.lang=ALL-UNNAMED`

**Issue:** #253

---

## File Map

All files under `persistence-mongodb/src/`:

**Main sources** (`main/java/io/casehub/work/mongodb/`):
| File | Responsibility |
|------|---------------|
| `MongoWorkItemNoteDocument.java` | MongoDB document for WorkItemNote |
| `MongoWorkItemNoteStore.java` | WorkItemNoteStore implementation |
| `MongoWorkItemTemplateDocument.java` | MongoDB document for WorkItemTemplate |
| `MongoWorkItemTemplateStore.java` | WorkItemTemplateStore implementation |
| `MongoWorkItemScheduleDocument.java` | MongoDB document for WorkItemSchedule |
| `MongoWorkItemScheduleStore.java` | WorkItemScheduleStore implementation (OCC) |
| `MongoWorkItemLinkDocument.java` | MongoDB document for WorkItemLink |
| `MongoWorkItemLinkStore.java` | WorkItemLinkStore implementation |
| `MongoWorkItemSpawnGroupDocument.java` | MongoDB document for WorkItemSpawnGroup |
| `MongoWorkItemSpawnGroupStore.java` | WorkItemSpawnGroupStore implementation (OCC) |
| `MongoWorkItemRelationDocument.java` | MongoDB document for WorkItemRelation |
| `MongoWorkItemRelationStore.java` | WorkItemRelationStore implementation |
| `MongoLabelVocabularyDocument.java` | MongoDB document for LabelVocabulary |
| `MongoLabelVocabularyStore.java` | LabelVocabularyStore implementation |
| `MongoLabelDefinitionDocument.java` | MongoDB document for LabelDefinition |
| `MongoLabelDefinitionStore.java` | LabelDefinitionStore implementation |
| `MongoFilterRuleDocument.java` | MongoDB document for FilterRule |
| `MongoFilterRuleStore.java` | FilterRuleStore implementation |
| `MongoRoutingCursorDocument.java` | MongoDB document for RoutingCursor |
| `MongoRoutingCursorStore.java` | RoutingCursorStore implementation (atomic) |
| `MongoIssueLinkDocument.java` | MongoDB document for WorkItemIssueLink |
| `MongoIssueLinkStore.java` | IssueLinkStore implementation |
| `MongoIndexInitializer.java` | @Startup bean — creates unique indexes |

**Test sources** (`test/java/io/casehub/work/mongodb/`):
| File | Responsibility |
|------|---------------|
| `MutableCurrentPrincipal.java` | Test-only mutable principal for tenant switching |
| `MongoWorkItemNoteStoreTest.java` | Tests for note store |
| `MongoWorkItemTemplateStoreTest.java` | Tests for template store |
| `MongoWorkItemScheduleStoreTest.java` | Tests for schedule store (incl. OCC) |
| `MongoWorkItemLinkStoreTest.java` | Tests for link store |
| `MongoWorkItemSpawnGroupStoreTest.java` | Tests for spawn group store (incl. OCC) |
| `MongoWorkItemRelationStoreTest.java` | Tests for relation store |
| `MongoLabelVocabularyStoreTest.java` | Tests for vocabulary store (incl. findOrCreate) |
| `MongoLabelDefinitionStoreTest.java` | Tests for definition store |
| `MongoFilterRuleStoreTest.java` | Tests for filter rule store |
| `MongoRoutingCursorStoreTest.java` | Tests for routing cursor store |
| `MongoIssueLinkStoreTest.java` | Tests for issue link store |

**Modified files:**
| File | Change |
|------|--------|
| `persistence-mongodb/pom.xml` | Add `casehub-work-issue-tracker` dependency |
| `runtime/src/main/java/.../WorkItemNoteStore.java` | Update javadoc Tier 2 line |
| `runtime/src/main/java/.../WorkItemScheduleStore.java` | Update javadoc Tier 2 line |
| `runtime/src/main/java/.../WorkItemTemplateStore.java` | Update javadoc Tier 2 line |
| `runtime/src/main/java/.../WorkItemLinkStore.java` | Update javadoc Tier 2 line |
| `runtime/src/main/java/.../WorkItemSpawnGroupStore.java` | Update javadoc Tier 2 line |
| `runtime/src/main/java/.../WorkItemRelationStore.java` | Update javadoc Tier 2 line |
| `runtime/src/main/java/.../LabelVocabularyStore.java` | Update javadoc (already correct) |
| `runtime/src/main/java/.../LabelDefinitionStore.java` | Update javadoc (already correct) |
| `runtime/src/main/java/.../FilterRuleStore.java` | Update javadoc (already correct) |
| `core/src/main/java/.../RoutingCursorStore.java` | Update javadoc Tier 2 line |
| `issue-tracker/src/main/java/.../IssueLinkStore.java` | Update javadoc Tier 2 line |

---

### Task 1: Infrastructure — pom.xml + test principal

**Files:**
- Modify: `persistence-mongodb/pom.xml`
- Create: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MutableCurrentPrincipal.java`

- [ ] **Step 1: Add issue-tracker dependency to pom.xml**

In `persistence-mongodb/pom.xml`, add after the `casehub-work` dependency:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-work-issue-tracker</artifactId>
      <version>${project.version}</version>
    </dependency>
```

- [ ] **Step 2: Create MutableCurrentPrincipal for tenant isolation tests**

```java
package io.casehub.work.mongodb;

import java.util.Set;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;

@ApplicationScoped
@Alternative
@Priority(100)
public class MutableCurrentPrincipal implements CurrentPrincipal {

    private String actorId = "test-user";
    private String tenancyId = TenancyConstants.DEFAULT_TENANT_ID;

    @Override
    public String actorId() {
        return actorId;
    }

    @Override
    public Set<String> groups() {
        return Set.of();
    }

    @Override
    public String tenancyId() {
        return tenancyId;
    }

    @Override
    public boolean isCrossTenantAdmin() {
        return false;
    }

    public void setTenancyId(String tenancyId) {
        this.tenancyId = tenancyId;
    }

    public void reset() {
        this.actorId = "test-user";
        this.tenancyId = TenancyConstants.DEFAULT_TENANT_ID;
    }
}
```

- [ ] **Step 3: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile test-compile -pl persistence-mongodb`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add persistence-mongodb/pom.xml persistence-mongodb/src/test/java/io/casehub/work/mongodb/MutableCurrentPrincipal.java
git commit -m "chore(#253): add issue-tracker dep + MutableCurrentPrincipal for tenant tests"
```

---

### Task 2: MongoWorkItemNoteStore

**Files:**
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemNoteDocument.java`
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemNoteStore.java`
- Create: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemNoteStoreTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.work.mongodb;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.runtime.model.WorkItemNote;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MongoWorkItemNoteStoreTest {

    @Inject MongoWorkItemNoteStore store;
    @Inject MutableCurrentPrincipal principal;

    @BeforeEach
    void clear() {
        MongoWorkItemNoteDocument.deleteAll();
        principal.reset();
    }

    @Test
    void append_assignsIdAndTimestamps() {
        final WorkItemNote note = note(UUID.randomUUID(), "hello", "alice");
        assertThat(note.id).isNull();

        store.append(note);

        assertThat(note.id).isNotNull();
        assertThat(note.createdAt).isNotNull();
        assertThat(note.tenancyId).isNotNull();
    }

    @Test
    void append_and_findById_roundtrip() {
        final WorkItemNote note = note(UUID.randomUUID(), "content", "alice");
        store.append(note);

        final Optional<WorkItemNote> found = store.findById(note.id);
        assertThat(found).isPresent();
        assertThat(found.get().content).isEqualTo("content");
        assertThat(found.get().author).isEqualTo("alice");
    }

    @Test
    void findById_returnsEmpty_whenNotFound() {
        assertThat(store.findById(UUID.randomUUID())).isEmpty();
    }

    @Test
    void findByWorkItemId_orderedByCreatedAt() {
        final UUID wiId = UUID.randomUUID();
        final WorkItemNote n1 = note(wiId, "first", "alice");
        n1.createdAt = Instant.now().minusSeconds(10);
        store.append(n1);
        final WorkItemNote n2 = note(wiId, "second", "bob");
        n2.createdAt = Instant.now();
        store.append(n2);
        store.append(note(UUID.randomUUID(), "other wi", "carol"));

        final List<WorkItemNote> notes = store.findByWorkItemId(wiId);
        assertThat(notes).hasSize(2);
        assertThat(notes.get(0).content).isEqualTo("first");
        assertThat(notes.get(1).content).isEqualTo("second");
    }

    @Test
    void update_modifiesContent() {
        final WorkItemNote note = note(UUID.randomUUID(), "original", "alice");
        store.append(note);

        note.content = "edited";
        note.editedAt = Instant.now();
        store.update(note);

        final WorkItemNote loaded = store.findById(note.id).orElseThrow();
        assertThat(loaded.content).isEqualTo("edited");
        assertThat(loaded.editedAt).isNotNull();
    }

    @Test
    void delete_returnsTrue_whenExists() {
        final WorkItemNote note = note(UUID.randomUUID(), "to delete", "alice");
        store.append(note);

        assertThat(store.delete(note.id)).isTrue();
        assertThat(store.findById(note.id)).isEmpty();
    }

    @Test
    void delete_returnsFalse_whenNotFound() {
        assertThat(store.delete(UUID.randomUUID())).isFalse();
    }

    @Test
    void tenantIsolation_noteInvisibleToOtherTenant() {
        final WorkItemNote note = note(UUID.randomUUID(), "tenant-A", "alice");
        store.append(note);

        principal.setTenancyId("other-tenant");

        assertThat(store.findById(note.id)).isEmpty();
        assertThat(store.findByWorkItemId(note.workItemId)).isEmpty();
        assertThat(store.delete(note.id)).isFalse();
    }

    private WorkItemNote note(UUID workItemId, String content, String author) {
        final WorkItemNote n = new WorkItemNote();
        n.workItemId = workItemId;
        n.content = content;
        n.author = author;
        return n;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb -Dtest=MongoWorkItemNoteStoreTest`
Expected: FAIL — `MongoWorkItemNoteStore` and `MongoWorkItemNoteDocument` don't exist

- [ ] **Step 3: Write MongoWorkItemNoteDocument**

```java
package io.casehub.work.mongodb;

import java.time.Instant;
import java.util.UUID;

import org.bson.codecs.pojo.annotations.BsonId;

import io.casehub.work.runtime.model.WorkItemNote;
import io.quarkus.mongodb.panache.PanacheMongoEntityBase;
import io.quarkus.mongodb.panache.common.MongoEntity;

@MongoEntity(collection = "work_item_notes")
public class MongoWorkItemNoteDocument extends PanacheMongoEntityBase {

    @BsonId
    public String id;

    public String tenancyId;
    public String workItemId;
    public String content;
    public String author;
    public Instant createdAt;
    public Instant editedAt;

    public static MongoWorkItemNoteDocument from(final WorkItemNote note) {
        final MongoWorkItemNoteDocument doc = new MongoWorkItemNoteDocument();
        doc.id = note.id != null ? note.id.toString() : UUID.randomUUID().toString();
        doc.tenancyId = note.tenancyId;
        doc.workItemId = note.workItemId != null ? note.workItemId.toString() : null;
        doc.content = note.content;
        doc.author = note.author;
        doc.createdAt = note.createdAt;
        doc.editedAt = note.editedAt;
        return doc;
    }

    public WorkItemNote toDomain() {
        final WorkItemNote note = new WorkItemNote();
        note.id = UUID.fromString(id);
        note.tenancyId = tenancyId;
        note.workItemId = workItemId != null ? UUID.fromString(workItemId) : null;
        note.content = content;
        note.author = author;
        note.createdAt = createdAt;
        note.editedAt = editedAt;
        return note;
    }
}
```

- [ ] **Step 4: Write MongoWorkItemNoteStore**

```java
package io.casehub.work.mongodb;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import org.bson.Document;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.runtime.model.WorkItemNote;
import io.casehub.work.runtime.repository.WorkItemNoteStore;

@ApplicationScoped
@Alternative
@Priority(1)
public class MongoWorkItemNoteStore implements WorkItemNoteStore {

    @Inject
    CurrentPrincipal currentPrincipal;

    @Override
    public WorkItemNote append(final WorkItemNote note) {
        if (note.id == null) {
            note.id = UUID.randomUUID();
        }
        if (note.createdAt == null) {
            note.createdAt = Instant.now();
        }
        if (note.tenancyId == null) {
            note.tenancyId = currentPrincipal.tenancyId();
        }
        MongoWorkItemNoteDocument.from(note).persist();
        return note;
    }

    @Override
    public Optional<WorkItemNote> findById(final UUID noteId) {
        final Document filter = new Document("_id", noteId.toString())
                .append("tenancyId", currentPrincipal.tenancyId());
        final MongoWorkItemNoteDocument doc = MongoWorkItemNoteDocument.find(filter).firstResult();
        return Optional.ofNullable(doc).map(MongoWorkItemNoteDocument::toDomain);
    }

    @Override
    public List<WorkItemNote> findByWorkItemId(final UUID workItemId) {
        final Document filter = new Document("workItemId", workItemId.toString())
                .append("tenancyId", currentPrincipal.tenancyId());
        return MongoWorkItemNoteDocument.<MongoWorkItemNoteDocument>find(filter)
                .stream()
                .sorted((a, b) -> a.createdAt.compareTo(b.createdAt))
                .map(MongoWorkItemNoteDocument::toDomain)
                .toList();
    }

    @Override
    public WorkItemNote update(final WorkItemNote note) {
        if (note.tenancyId == null) {
            note.tenancyId = currentPrincipal.tenancyId();
        }
        MongoWorkItemNoteDocument.from(note).persistOrUpdate();
        return note;
    }

    @Override
    public boolean delete(final UUID noteId) {
        final Document filter = new Document("_id", noteId.toString())
                .append("tenancyId", currentPrincipal.tenancyId());
        return MongoWorkItemNoteDocument.delete(filter) > 0;
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb -Dtest=MongoWorkItemNoteStoreTest`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemNoteDocument.java persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemNoteStore.java persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemNoteStoreTest.java
git commit -m "feat(#253): MongoWorkItemNoteStore — document, store, tests

Refs #253"
```

---

### Task 3: MongoWorkItemTemplateStore

**Files:**
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemTemplateDocument.java`
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemTemplateStore.java`
- Create: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemTemplateStoreTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.work.mongodb;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MongoWorkItemTemplateStoreTest {

    @Inject MongoWorkItemTemplateStore store;
    @Inject MutableCurrentPrincipal principal;

    @BeforeEach
    void clear() {
        MongoWorkItemTemplateDocument.deleteAll();
        principal.reset();
    }

    @Test
    void put_assignsIdAndTimestamps() {
        final WorkItemTemplate t = template("Review Template");
        store.put(t);

        assertThat(t.id).isNotNull();
        assertThat(t.createdAt).isNotNull();
        assertThat(t.tenancyId).isNotNull();
    }

    @Test
    void put_and_get_roundtrip() {
        final WorkItemTemplate t = template("Loan Review");
        t.description = "For loan applications";
        t.category = "finance";
        t.priority = WorkItemPriority.HIGH;
        t.candidateGroups = "finance-team";
        t.defaultExpiryHours = 48;
        t.labelPaths = "[\"finance/review\"]";
        t.outcomes = "[{\"name\":\"approved\"},{\"name\":\"rejected\"}]";
        t.scope = "casehubio/finance";
        store.put(t);

        final WorkItemTemplate loaded = store.get(t.id).orElseThrow();
        assertThat(loaded.name).isEqualTo("Loan Review");
        assertThat(loaded.description).isEqualTo("For loan applications");
        assertThat(loaded.category).isEqualTo("finance");
        assertThat(loaded.priority).isEqualTo(WorkItemPriority.HIGH);
        assertThat(loaded.candidateGroups).isEqualTo("finance-team");
        assertThat(loaded.defaultExpiryHours).isEqualTo(48);
        assertThat(loaded.labelPaths).isEqualTo("[\"finance/review\"]");
        assertThat(loaded.outcomes).isEqualTo("[{\"name\":\"approved\"},{\"name\":\"rejected\"}]");
        assertThat(loaded.scope).isEqualTo("casehubio/finance");
    }

    @Test
    void getByName_findsExact() {
        store.put(template("Alpha"));
        store.put(template("Beta"));

        assertThat(store.getByName("Alpha")).isPresent();
        assertThat(store.getByName("Alpha").get().name).isEqualTo("Alpha");
        assertThat(store.getByName("Gamma")).isEmpty();
    }

    @Test
    void scanAll_orderedByName() {
        store.put(template("Zebra"));
        store.put(template("Apple"));

        final List<WorkItemTemplate> all = store.scanAll();
        assertThat(all).hasSize(2);
        assertThat(all.get(0).name).isEqualTo("Apple");
        assertThat(all.get(1).name).isEqualTo("Zebra");
    }

    @Test
    void delete_removesAndReturnsTrueOrFalse() {
        final WorkItemTemplate t = template("To Delete");
        store.put(t);
        assertThat(store.delete(t.id)).isTrue();
        assertThat(store.get(t.id)).isEmpty();
        assertThat(store.delete(UUID.randomUUID())).isFalse();
    }

    @Test
    void tenantIsolation() {
        final WorkItemTemplate t = template("Tenant-A template");
        store.put(t);

        principal.setTenancyId("other-tenant");

        assertThat(store.get(t.id)).isEmpty();
        assertThat(store.getByName("Tenant-A template")).isEmpty();
        assertThat(store.scanAll()).isEmpty();
        assertThat(store.delete(t.id)).isFalse();
    }

    private WorkItemTemplate template(String name) {
        final WorkItemTemplate t = new WorkItemTemplate();
        t.name = name;
        t.createdBy = "test-user";
        return t;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb -Dtest=MongoWorkItemTemplateStoreTest`
Expected: FAIL

- [ ] **Step 3: Write MongoWorkItemTemplateDocument**

```java
package io.casehub.work.mongodb;

import java.time.Instant;
import java.util.UUID;

import org.bson.codecs.pojo.annotations.BsonId;

import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.quarkus.mongodb.panache.PanacheMongoEntityBase;
import io.quarkus.mongodb.panache.common.MongoEntity;

@MongoEntity(collection = "work_item_templates")
public class MongoWorkItemTemplateDocument extends PanacheMongoEntityBase {

    @BsonId
    public String id;

    public String tenancyId;
    public String name;
    public String description;
    public String category;
    public String priority;
    public String candidateGroups;
    public String candidateUsers;
    public String requiredCapabilities;
    public Integer defaultExpiryHours;
    public Integer defaultClaimHours;
    public Integer defaultExpiryBusinessHours;
    public Integer defaultClaimBusinessHours;
    public String defaultPayload;
    public String labelPaths;
    public String outcomes;
    public String excludedUsers;
    public String excludedGroups;
    public String scope;
    public String inputDataSchema;
    public String outputDataSchema;
    public Integer instanceCount;
    public Integer requiredCount;
    public String parentRole;
    public String assignmentStrategy;
    public String onThresholdReached;
    public Boolean allowSameAssignee;
    public String createdBy;
    public Instant createdAt;

    public static MongoWorkItemTemplateDocument from(final WorkItemTemplate t) {
        final MongoWorkItemTemplateDocument doc = new MongoWorkItemTemplateDocument();
        doc.id = t.id != null ? t.id.toString() : UUID.randomUUID().toString();
        doc.tenancyId = t.tenancyId;
        doc.name = t.name;
        doc.description = t.description;
        doc.category = t.category;
        doc.priority = t.priority != null ? t.priority.name() : null;
        doc.candidateGroups = t.candidateGroups;
        doc.candidateUsers = t.candidateUsers;
        doc.requiredCapabilities = t.requiredCapabilities;
        doc.defaultExpiryHours = t.defaultExpiryHours;
        doc.defaultClaimHours = t.defaultClaimHours;
        doc.defaultExpiryBusinessHours = t.defaultExpiryBusinessHours;
        doc.defaultClaimBusinessHours = t.defaultClaimBusinessHours;
        doc.defaultPayload = t.defaultPayload;
        doc.labelPaths = t.labelPaths;
        doc.outcomes = t.outcomes;
        doc.excludedUsers = t.excludedUsers;
        doc.excludedGroups = t.excludedGroups;
        doc.scope = t.scope;
        doc.inputDataSchema = t.inputDataSchema;
        doc.outputDataSchema = t.outputDataSchema;
        doc.instanceCount = t.instanceCount;
        doc.requiredCount = t.requiredCount;
        doc.parentRole = t.parentRole;
        doc.assignmentStrategy = t.assignmentStrategy;
        doc.onThresholdReached = t.onThresholdReached;
        doc.allowSameAssignee = t.allowSameAssignee;
        doc.createdBy = t.createdBy;
        doc.createdAt = t.createdAt;
        return doc;
    }

    public WorkItemTemplate toDomain() {
        final WorkItemTemplate t = new WorkItemTemplate();
        t.id = UUID.fromString(id);
        t.tenancyId = tenancyId;
        t.name = name;
        t.description = description;
        t.category = category;
        t.priority = priority != null ? WorkItemPriority.valueOf(priority) : null;
        t.candidateGroups = candidateGroups;
        t.candidateUsers = candidateUsers;
        t.requiredCapabilities = requiredCapabilities;
        t.defaultExpiryHours = defaultExpiryHours;
        t.defaultClaimHours = defaultClaimHours;
        t.defaultExpiryBusinessHours = defaultExpiryBusinessHours;
        t.defaultClaimBusinessHours = defaultClaimBusinessHours;
        t.defaultPayload = defaultPayload;
        t.labelPaths = labelPaths;
        t.outcomes = outcomes;
        t.excludedUsers = excludedUsers;
        t.excludedGroups = excludedGroups;
        t.scope = scope;
        t.inputDataSchema = inputDataSchema;
        t.outputDataSchema = outputDataSchema;
        t.instanceCount = instanceCount;
        t.requiredCount = requiredCount;
        t.parentRole = parentRole;
        t.assignmentStrategy = assignmentStrategy;
        t.onThresholdReached = onThresholdReached;
        t.allowSameAssignee = allowSameAssignee;
        t.createdBy = createdBy;
        t.createdAt = createdAt;
        return t;
    }
}
```

- [ ] **Step 4: Write MongoWorkItemTemplateStore**

```java
package io.casehub.work.mongodb;

import java.time.Instant;
import java.util.Comparator;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import org.bson.Document;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.casehub.work.runtime.repository.WorkItemTemplateStore;

@ApplicationScoped
@Alternative
@Priority(1)
public class MongoWorkItemTemplateStore implements WorkItemTemplateStore {

    @Inject
    CurrentPrincipal currentPrincipal;

    @Override
    public WorkItemTemplate put(final WorkItemTemplate template) {
        if (template.id == null) {
            template.id = UUID.randomUUID();
        }
        if (template.createdAt == null) {
            template.createdAt = Instant.now();
        }
        if (template.tenancyId == null) {
            template.tenancyId = currentPrincipal.tenancyId();
        }
        MongoWorkItemTemplateDocument.from(template).persistOrUpdate();
        return template;
    }

    @Override
    public Optional<WorkItemTemplate> get(final UUID id) {
        final Document filter = new Document("_id", id.toString())
                .append("tenancyId", currentPrincipal.tenancyId());
        final MongoWorkItemTemplateDocument doc = MongoWorkItemTemplateDocument.find(filter).firstResult();
        return Optional.ofNullable(doc).map(MongoWorkItemTemplateDocument::toDomain);
    }

    @Override
    public Optional<WorkItemTemplate> getByName(final String name) {
        final Document filter = new Document("name", name)
                .append("tenancyId", currentPrincipal.tenancyId());
        final MongoWorkItemTemplateDocument doc = MongoWorkItemTemplateDocument.find(filter).firstResult();
        return Optional.ofNullable(doc).map(MongoWorkItemTemplateDocument::toDomain);
    }

    @Override
    public List<WorkItemTemplate> scanAll() {
        final Document filter = new Document("tenancyId", currentPrincipal.tenancyId());
        return MongoWorkItemTemplateDocument.<MongoWorkItemTemplateDocument>find(filter)
                .stream()
                .sorted(Comparator.comparing(d -> d.name))
                .map(MongoWorkItemTemplateDocument::toDomain)
                .toList();
    }

    @Override
    public boolean delete(final UUID id) {
        final Document filter = new Document("_id", id.toString())
                .append("tenancyId", currentPrincipal.tenancyId());
        return MongoWorkItemTemplateDocument.delete(filter) > 0;
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb -Dtest=MongoWorkItemTemplateStoreTest`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemTemplate*.java persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemTemplateStoreTest.java
git commit -m "feat(#253): MongoWorkItemTemplateStore — document, store, tests

Refs #253"
```

---

### Task 4: MongoWorkItemScheduleStore (OCC)

**Files:**
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemScheduleDocument.java`
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemScheduleStore.java`
- Create: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemScheduleStoreTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.work.mongodb;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.OptimisticLockException;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.runtime.model.WorkItemSchedule;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MongoWorkItemScheduleStoreTest {

    @Inject MongoWorkItemScheduleStore store;
    @Inject MutableCurrentPrincipal principal;

    @BeforeEach
    void clear() {
        MongoWorkItemScheduleDocument.deleteAll();
        principal.reset();
    }

    @Test
    void put_assignsIdAndTimestamps() {
        final WorkItemSchedule s = schedule("Daily", "0 0 9 * * ?");
        store.put(s);

        assertThat(s.id).isNotNull();
        assertThat(s.createdAt).isNotNull();
        assertThat(s.tenancyId).isNotNull();
    }

    @Test
    void put_and_get_roundtrip() {
        final WorkItemSchedule s = schedule("Hourly", "0 0 * * * ?");
        s.templateId = UUID.randomUUID();
        s.nextFireAt = Instant.now().plus(1, ChronoUnit.HOURS);
        store.put(s);

        final WorkItemSchedule loaded = store.get(s.id).orElseThrow();
        assertThat(loaded.name).isEqualTo("Hourly");
        assertThat(loaded.cronExpression).isEqualTo("0 0 * * * ?");
        assertThat(loaded.active).isTrue();
        assertThat(loaded.templateId).isEqualTo(s.templateId);
    }

    @Test
    void scanAll_orderedByName() {
        store.put(schedule("Zebra", "0 0 9 * * ?"));
        store.put(schedule("Alpha", "0 0 9 * * ?"));

        final List<WorkItemSchedule> all = store.scanAll();
        assertThat(all).hasSize(2);
        assertThat(all.get(0).name).isEqualTo("Alpha");
    }

    @Test
    void findDue_returnsOnlyActivePastSchedules() {
        final WorkItemSchedule due = schedule("Due", "0 0 9 * * ?");
        due.active = true;
        due.nextFireAt = Instant.now().minusSeconds(60);
        store.put(due);

        final WorkItemSchedule future = schedule("Future", "0 0 9 * * ?");
        future.active = true;
        future.nextFireAt = Instant.now().plusSeconds(3600);
        store.put(future);

        final WorkItemSchedule inactive = schedule("Inactive", "0 0 9 * * ?");
        inactive.active = false;
        inactive.nextFireAt = Instant.now().minusSeconds(60);
        store.put(inactive);

        final List<WorkItemSchedule> result = store.findDue(Instant.now());
        assertThat(result).hasSize(1);
        assertThat(result.get(0).name).isEqualTo("Due");
    }

    @Test
    void put_update_incrementsVersion() {
        final WorkItemSchedule s = schedule("Versioned", "0 0 9 * * ?");
        store.put(s);

        final WorkItemSchedule loaded = store.get(s.id).orElseThrow();
        assertThat(loaded.version).isEqualTo(0L);

        loaded.lastFiredAt = Instant.now();
        store.put(loaded);

        final WorkItemSchedule reloaded = store.get(s.id).orElseThrow();
        assertThat(reloaded.version).isEqualTo(1L);
    }

    @Test
    void put_throwsOptimisticLockException_onStaleVersion() {
        final WorkItemSchedule s = schedule("OCC Test", "0 0 9 * * ?");
        store.put(s);

        final WorkItemSchedule reader1 = store.get(s.id).orElseThrow();
        final WorkItemSchedule reader2 = store.get(s.id).orElseThrow();

        reader1.lastFiredAt = Instant.now();
        store.put(reader1);

        reader2.lastFiredAt = Instant.now().plusSeconds(10);
        assertThatThrownBy(() -> store.put(reader2))
                .isInstanceOf(OptimisticLockException.class);
    }

    @Test
    void delete_removesAndReturnsBoolean() {
        final WorkItemSchedule s = schedule("To Delete", "0 0 9 * * ?");
        store.put(s);
        assertThat(store.delete(s.id)).isTrue();
        assertThat(store.get(s.id)).isEmpty();
        assertThat(store.delete(UUID.randomUUID())).isFalse();
    }

    @Test
    void tenantIsolation() {
        final WorkItemSchedule s = schedule("Tenant-A", "0 0 9 * * ?");
        s.active = true;
        s.nextFireAt = Instant.now().minusSeconds(60);
        store.put(s);

        principal.setTenancyId("other-tenant");

        assertThat(store.get(s.id)).isEmpty();
        assertThat(store.scanAll()).isEmpty();
        assertThat(store.findDue(Instant.now())).isEmpty();
        assertThat(store.delete(s.id)).isFalse();
    }

    private WorkItemSchedule schedule(String name, String cron) {
        final WorkItemSchedule s = new WorkItemSchedule();
        s.name = name;
        s.cronExpression = cron;
        s.templateId = UUID.randomUUID();
        s.active = true;
        s.createdBy = "test-user";
        return s;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb -Dtest=MongoWorkItemScheduleStoreTest`
Expected: FAIL

- [ ] **Step 3: Write MongoWorkItemScheduleDocument**

```java
package io.casehub.work.mongodb;

import java.time.Instant;
import java.util.UUID;

import org.bson.codecs.pojo.annotations.BsonId;

import io.casehub.work.runtime.model.WorkItemSchedule;
import io.quarkus.mongodb.panache.PanacheMongoEntityBase;
import io.quarkus.mongodb.panache.common.MongoEntity;

@MongoEntity(collection = "work_item_schedules")
public class MongoWorkItemScheduleDocument extends PanacheMongoEntityBase {

    @BsonId
    public String id;

    public String tenancyId;
    public String name;
    public String templateId;
    public String cronExpression;
    public boolean active;
    public String createdBy;
    public Instant createdAt;
    public Instant lastFiredAt;
    public Instant nextFireAt;
    public Long version;

    public static MongoWorkItemScheduleDocument from(final WorkItemSchedule s) {
        final MongoWorkItemScheduleDocument doc = new MongoWorkItemScheduleDocument();
        doc.id = s.id != null ? s.id.toString() : UUID.randomUUID().toString();
        doc.tenancyId = s.tenancyId;
        doc.name = s.name;
        doc.templateId = s.templateId != null ? s.templateId.toString() : null;
        doc.cronExpression = s.cronExpression;
        doc.active = s.active;
        doc.createdBy = s.createdBy;
        doc.createdAt = s.createdAt;
        doc.lastFiredAt = s.lastFiredAt;
        doc.nextFireAt = s.nextFireAt;
        doc.version = s.version;
        return doc;
    }

    public WorkItemSchedule toDomain() {
        final WorkItemSchedule s = new WorkItemSchedule();
        s.id = UUID.fromString(id);
        s.tenancyId = tenancyId;
        s.name = name;
        s.templateId = templateId != null ? UUID.fromString(templateId) : null;
        s.cronExpression = cronExpression;
        s.active = active;
        s.createdBy = createdBy;
        s.createdAt = createdAt;
        s.lastFiredAt = lastFiredAt;
        s.nextFireAt = nextFireAt;
        s.version = version;
        return s;
    }
}
```

- [ ] **Step 4: Write MongoWorkItemScheduleStore (with OCC)**

```java
package io.casehub.work.mongodb;

import java.time.Instant;
import java.util.Comparator;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import jakarta.persistence.OptimisticLockException;

import org.bson.Document;

import com.mongodb.client.model.FindOneAndUpdateOptions;
import com.mongodb.client.model.ReturnDocument;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.runtime.model.WorkItemSchedule;
import io.casehub.work.runtime.repository.WorkItemScheduleStore;

@ApplicationScoped
@Alternative
@Priority(1)
public class MongoWorkItemScheduleStore implements WorkItemScheduleStore {

    @Inject
    CurrentPrincipal currentPrincipal;

    @Override
    public WorkItemSchedule put(final WorkItemSchedule schedule) {
        if (schedule.id == null) {
            schedule.id = UUID.randomUUID();
        }
        if (schedule.createdAt == null) {
            schedule.createdAt = Instant.now();
        }
        if (schedule.tenancyId == null) {
            schedule.tenancyId = currentPrincipal.tenancyId();
        }

        final String idStr = schedule.id.toString();
        final Document existing = MongoWorkItemScheduleDocument.find(new Document("_id", idStr)).firstResult() != null
                ? new Document() : null;

        if (existing == null) {
            schedule.version = 0L;
            MongoWorkItemScheduleDocument.from(schedule).persist();
        } else {
            final Document filter = new Document("_id", idStr)
                    .append("version", schedule.version);
            final Document setFields = new Document()
                    .append("tenancyId", schedule.tenancyId)
                    .append("name", schedule.name)
                    .append("templateId", schedule.templateId != null ? schedule.templateId.toString() : null)
                    .append("cronExpression", schedule.cronExpression)
                    .append("active", schedule.active)
                    .append("createdBy", schedule.createdBy)
                    .append("createdAt", schedule.createdAt)
                    .append("lastFiredAt", schedule.lastFiredAt)
                    .append("nextFireAt", schedule.nextFireAt);
            final Document update = new Document("$set", setFields)
                    .append("$inc", new Document("version", 1L));

            final MongoWorkItemScheduleDocument result = MongoWorkItemScheduleDocument
                    .mongoCollection()
                    .findOneAndUpdate(filter, update,
                            new FindOneAndUpdateOptions().returnDocument(ReturnDocument.AFTER));
            if (result == null) {
                throw new OptimisticLockException(
                        "Version conflict on WorkItemSchedule " + idStr);
            }
            schedule.version = result.version;
        }
        return schedule;
    }

    @Override
    public Optional<WorkItemSchedule> get(final UUID id) {
        final Document filter = new Document("_id", id.toString())
                .append("tenancyId", currentPrincipal.tenancyId());
        final MongoWorkItemScheduleDocument doc = MongoWorkItemScheduleDocument.find(filter).firstResult();
        return Optional.ofNullable(doc).map(MongoWorkItemScheduleDocument::toDomain);
    }

    @Override
    public List<WorkItemSchedule> scanAll() {
        final Document filter = new Document("tenancyId", currentPrincipal.tenancyId());
        return MongoWorkItemScheduleDocument.<MongoWorkItemScheduleDocument>find(filter)
                .stream()
                .sorted(Comparator.comparing(d -> d.name))
                .map(MongoWorkItemScheduleDocument::toDomain)
                .toList();
    }

    @Override
    public boolean delete(final UUID id) {
        final Document filter = new Document("_id", id.toString())
                .append("tenancyId", currentPrincipal.tenancyId());
        return MongoWorkItemScheduleDocument.delete(filter) > 0;
    }

    @Override
    public List<WorkItemSchedule> findDue(final Instant now) {
        final Document filter = new Document("tenancyId", currentPrincipal.tenancyId())
                .append("active", true)
                .append("nextFireAt", new Document("$ne", null).append("$lte", now));
        return MongoWorkItemScheduleDocument.<MongoWorkItemScheduleDocument>find(filter)
                .stream()
                .sorted(Comparator.comparing(d -> d.nextFireAt))
                .map(MongoWorkItemScheduleDocument::toDomain)
                .toList();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb -Dtest=MongoWorkItemScheduleStoreTest`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemSchedule*.java persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemScheduleStoreTest.java
git commit -m "feat(#253): MongoWorkItemScheduleStore — OCC via findOneAndUpdate

Refs #253"
```

---

### Task 5–12: Remaining 8 stores

> **Implementation note:** Tasks 5–12 follow the identical pattern established in Tasks 2–4. Each task creates a Document class, Store class, and Test class. The plan continues with the same structure — test first, then document, then store, run, commit.
>
> **For brevity in the plan document, the remaining 8 store implementations are specified by their distinctive elements only.** The pattern (CDI annotations, tenant filter, from/toDomain, test structure) is identical to Tasks 2–4. The executing agent should reference Task 2 (simple CRUD) or Task 4 (OCC) as the template.

#### Task 5: MongoWorkItemLinkStore

**Collection:** `work_item_links`
**Fields:** `id`, `tenancyId`, `workItemId`, `url`, `title`, `relationType`, `linkedBy`, `createdAt`
**SPI methods:** `put()`, `get(UUID)`, `findByWorkItemId(UUID)`, `findByWorkItemIdAndType(UUID, String)`, `delete(UUID) → boolean`
**Pattern:** Simple CRUD (same as Task 2)
**Test focus:** roundtrip, findByWorkItemIdAndType filters correctly, tenant isolation

#### Task 6: MongoFilterRuleStore

**Collection:** `filter_rules`
**Fields:** `id`, `tenancyId`, `name`, `description`, `enabled` (boolean), `condition`, `events`, `actionsJson`, `createdAt`
**SPI methods:** `put()`, `get(UUID)`, `allEnabled()`, `scanAll()`, `delete(UUID) → boolean`
**Pattern:** Simple CRUD (same as Task 2)
**Test focus:** roundtrip, `allEnabled()` returns only enabled rules, `scanAll()` ordered by createdAt ASC, tenant isolation

#### Task 7: MongoLabelVocabularyStore

**Collection:** `label_vocabularies`
**Fields:** `id`, `tenancyId`, `scope` (String — `Path.toString()` / `Path.of()`), `name`
**SPI methods:** `put()`, `get(UUID)`, `scanAll()`, `findByScope(Path)`, `findOrCreate(Path, String)`, `delete(UUID) → boolean`
**Pattern:** Simple CRUD + atomic `findOrCreate` via `mongoCollection().findOneAndUpdate()` with `upsert: true`
**Conversion note:** `scope` field uses `Path.toString()` on write and `Path.of()` on read
**Test focus:** roundtrip, `findByScope` exact match, `findOrCreate` creates when absent and returns existing when present, tenant isolation

#### Task 8: MongoLabelDefinitionStore

**Collection:** `label_definitions`
**Fields:** `id`, `tenancyId`, `path` (String — `Path.toString()` / `Path.of()`), `vocabularyId`, `description`, `createdBy`, `createdAt`
**SPI methods:** `put()`, `get(UUID)`, `findByVocabularyId(UUID)`, `findByPath(Path)`, `delete(UUID) → boolean`
**Pattern:** Simple CRUD (same as Task 2)
**Conversion note:** `path` field uses `Path.toString()` on write and `Path.of()` on read
**Test focus:** roundtrip, findByVocabularyId/findByPath, tenant isolation

#### Task 9: MongoWorkItemRelationStore

**Collection:** `work_item_relations`
**Fields:** `id`, `tenancyId`, `sourceId`, `targetId`, `relationType`, `createdBy`, `createdAt`
**SPI methods:** `put()`, `get(UUID)`, `findBySourceId(UUID)`, `findByTargetId(UUID)`, `findBySourceAndType(UUID, String)`, `findByTargetAndType(UUID, String)`, `findExisting(UUID, UUID, String)`, `delete(UUID) → boolean`
**Pattern:** Simple CRUD with multiple query methods (same CDI pattern as Task 2)
**Test focus:** roundtrip, directional queries (source vs target), type-filtered queries, findExisting, tenant isolation

#### Task 10: MongoWorkItemSpawnGroupStore (OCC)

**Collection:** `work_item_spawn_groups`
**Fields:** `id`, `tenancyId`, `parentId`, `idempotencyKey`, `createdAt`, `version` (Long), `instanceCount`, `requiredCount`, `onThresholdReached`, `allowSameAssignee` (boolean), `parentRole`, `completedCount` (int), `rejectedCount` (int), `policyTriggered` (boolean)
**SPI methods:** `put()`, `get(UUID)`, `findByParentId(UUID)`, `findByParentAndKey(UUID, String)`, `findMultiInstanceByParentId(UUID)`, `delete(UUID) → boolean`
**Pattern:** OCC (same as Task 4) — `put()` uses `findOneAndUpdate` with version-check filter for existing documents
**Test focus:** roundtrip, findByParentAndKey, findMultiInstanceByParentId (only returns groups where requiredCount != null), OCC throws on stale version, tenant isolation

#### Task 11: MongoRoutingCursorStore (atomic)

**Collection:** `routing_cursors`
**Document ID:** `poolHash + ":" + tenancyId`
**Fields:** `lastIndex` (long), `lastAccessed` (Instant)
**SPI method:** `acquireNext(String poolHash, int poolSize) → int`
**Pattern:** Atomic `findOneAndUpdate` via `MongoRoutingCursorDocument.mongoCollection()`:
- Filter: `{_id: poolHash + ":" + tenancyId}`
- Update: `{$inc: {lastIndex: 1L}, $set: {lastAccessed: now}, $setOnInsert: {lastIndex: -1L}}`
- Options: `upsert: true, returnDocument: AFTER`
- Return: `Math.floorMod(result.lastIndex, poolSize)`
**Test focus:** first call returns 0, sequential calls advance and wrap at poolSize, tenant isolation (separate cursor per tenant)

#### Task 12: MongoIssueLinkStore

**Collection:** `work_item_issue_links`
**Fields:** `id`, `tenancyId`, `workItemId`, `trackerType`, `externalRef`, `title`, `url`, `status`, `linkedAt`, `linkedBy`
**SPI methods:** `findById(UUID)`, `findByWorkItemId(UUID)`, `findByRef(UUID, String, String)`, `findByTrackerRef(String, String)`, `save(WorkItemIssueLink)`, `delete(WorkItemIssueLink)`
**Pattern:** Simple CRUD (same as Task 2)
**delete() note:** SPI signature is `void delete(WorkItemIssueLink link)` — extract `link.id`, verify `currentPrincipal.tenancyId()` matches before deleting
**Test focus:** roundtrip, findByRef compound lookup, findByTrackerRef cross-WorkItem lookup, delete validates tenancyId, tenant isolation

---

### Task 13: MongoIndexInitializer

**Files:**
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoIndexInitializer.java`

- [ ] **Step 1: Write MongoIndexInitializer**

```java
package io.casehub.work.mongodb;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;

import org.bson.Document;

import com.mongodb.client.model.IndexOptions;

import io.quarkus.runtime.StartupEvent;

@ApplicationScoped
public class MongoIndexInitializer {

    void onStart(@Observes StartupEvent event) {
        final IndexOptions unique = new IndexOptions().unique(true);

        MongoWorkItemTemplateDocument.mongoCollection().createIndex(
                new Document("name", 1).append("tenancyId", 1), unique);

        MongoWorkItemSpawnGroupDocument.mongoCollection().createIndex(
                new Document("parentId", 1).append("idempotencyKey", 1), unique);

        MongoWorkItemRelationDocument.mongoCollection().createIndex(
                new Document("sourceId", 1).append("targetId", 1).append("relationType", 1), unique);

        MongoIssueLinkDocument.mongoCollection().createIndex(
                new Document("workItemId", 1).append("trackerType", 1).append("externalRef", 1), unique);

        MongoLabelVocabularyDocument.mongoCollection().createIndex(
                new Document("scope", 1).append("tenancyId", 1), unique);
    }
}
```

- [ ] **Step 2: Run all tests to verify nothing breaks**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb`
Expected: All tests PASS

- [ ] **Step 3: Commit**

```
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoIndexInitializer.java
git commit -m "feat(#253): MongoIndexInitializer — unique indexes at startup

Refs #253"
```

---

### Task 14: SPI javadoc updates

**Files:**
- Modify: 5 SPI interfaces that still have "No Tier 2 (MongoDB) exists yet" lines (the other 6 already have correct Tier 2 javadoc)

- [ ] **Step 1: Update remaining SPI javadoc**

In each file, replace the "No Tier 2" line with the correct Tier 2 entry. The files to check and update:
- `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemNoteStore.java` — replace `No Tier 2 (MongoDB) exists yet (tracked as casehubio/work#253).` with `Tier 2: {@code @Alternative @Priority(1)} (MongoDB) — {@code casehub-work-persistence-mongodb}.`
- `core/src/main/java/io/casehub/work/core/strategy/RoutingCursorStore.java` — same replacement
- `issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/IssueLinkStore.java` — same replacement

The remaining 8 SPIs (WorkItemScheduleStore, WorkItemTemplateStore, WorkItemLinkStore, WorkItemSpawnGroupStore, WorkItemRelationStore, LabelVocabularyStore, LabelDefinitionStore, FilterRuleStore) already have the correct Tier 2 line in their javadoc — verify and skip.

- [ ] **Step 2: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemNoteStore.java core/src/main/java/io/casehub/work/core/strategy/RoutingCursorStore.java issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/IssueLinkStore.java
git commit -m "docs(#253): update SPI javadoc — Tier 2 MongoDB now exists

Refs #253"
```

---

### Task 15: Full build verification

- [ ] **Step 1: Run all persistence-mongodb tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl persistence-mongodb`
Expected: All tests PASS, BUILD SUCCESS

- [ ] **Step 2: Run full project build (compile only, no tests — verify no cross-module breaks)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl persistence-mongodb,runtime,core,issue-tracker`
Expected: BUILD SUCCESS

- [ ] **Step 3: Final commit if any adjustments were needed**

Only if Step 1 or 2 revealed issues that required fixes.
