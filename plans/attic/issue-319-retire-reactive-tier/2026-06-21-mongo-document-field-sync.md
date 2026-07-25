# MongoWorkItemDocument Field Sync + OCC Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 13 missing fields to MongoWorkItemDocument, implement optimistic concurrency control, fix the outcome query filter gap, and add 5 proper MongoDB query method overrides.

**Architecture:** The document class (`MongoWorkItemDocument`) gains 13 fields with mapping in `from()`/`toDomain()`. The store (`MongoWorkItemStore`) replaces `persistOrUpdate()` with version-checked `replaceOne` for OCC, adds an outcome clause to `buildFilter()`, and overrides 5 default query methods with proper MongoDB filters. All queries are tenant-scoped.

**Tech Stack:** Java 21, Quarkus MongoDB Panache, MongoDB BSON driver, AssertJ

---

### Task 1: MongoWorkItemDocument — add 13 fields + roundtrip test

**Files:**
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java`
- Modify: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java`

- [ ] **Step 1: Write the failing roundtrip test**

Add this test to `MongoWorkItemStoreTest.java` after the existing `candidateGroups_roundtrip_commaSeparatedPreserved` test:

```java
@Test
void put_and_get_roundtrip_allFields() {
    final WorkItem wi = pending("alice", "Full roundtrip");
    wi.description = "Full field coverage";
    wi.category = "review";
    wi.priority = WorkItemPriority.HIGH;
    wi.formKey = "review-form";
    wi.payload = "{\"ref\":\"PROJ-42\"}";
    wi.resolution = "done";
    wi.owner = "team-lead";
    wi.candidateGroups = "finance-team,hr-team";
    wi.candidateUsers = "bob,carol";
    wi.requiredCapabilities = "java,review";
    wi.delegationChain = "alice>bob";
    wi.delegationDeclineTarget = DeclineTarget.RETURN_TO_POOL;
    wi.priorStatus = WorkItemStatus.ASSIGNED;
    wi.claimDeadline = Instant.parse("2026-07-01T12:00:00Z");
    wi.expiresAt = Instant.parse("2026-07-15T12:00:00Z");
    wi.followUpDate = Instant.parse("2026-07-05T12:00:00Z");
    wi.assignedAt = Instant.parse("2026-06-20T10:00:00Z");
    wi.startedAt = Instant.parse("2026-06-20T11:00:00Z");
    wi.suspendedAt = Instant.parse("2026-06-20T15:00:00Z");

    // The 13 previously-missing fields
    wi.accumulatedUnclaimedSeconds = 3600L;
    wi.lastReturnedToPoolAt = Instant.parse("2026-06-20T09:00:00Z");
    wi.confidenceScore = 0.85;
    wi.callerRef = "case:abc-123/pi:step-7";
    wi.parentId = UUID.randomUUID();
    wi.scope = "casehubio/devtown/pr-review";
    wi.templateId = UUID.randomUUID();
    wi.permittedOutcomes = "[\"approved\",\"rejected\",\"needs-revision\"]";
    wi.excludedUsers = "dave,eve";
    wi.outcome = "approved";
    wi.inputDataSchema = "{\"type\":\"object\"}";
    wi.outputDataSchema = "{\"type\":\"string\"}";

    store.put(wi);
    final WorkItem loaded = store.get(wi.id).orElseThrow();

    // Original fields
    assertThat(loaded.description).isEqualTo("Full field coverage");
    assertThat(loaded.category).isEqualTo("review");
    assertThat(loaded.priority).isEqualTo(WorkItemPriority.HIGH);
    assertThat(loaded.formKey).isEqualTo("review-form");
    assertThat(loaded.payload).isEqualTo("{\"ref\":\"PROJ-42\"}");
    assertThat(loaded.resolution).isEqualTo("done");
    assertThat(loaded.owner).isEqualTo("team-lead");
    assertThat(loaded.candidateGroups).isEqualTo("finance-team,hr-team");
    assertThat(loaded.candidateUsers).isEqualTo("bob,carol");
    assertThat(loaded.delegationDeclineTarget).isEqualTo(DeclineTarget.RETURN_TO_POOL);
    assertThat(loaded.priorStatus).isEqualTo(WorkItemStatus.ASSIGNED);

    // The 13 previously-missing fields
    assertThat(loaded.accumulatedUnclaimedSeconds).isEqualTo(3600L);
    assertThat(loaded.lastReturnedToPoolAt).isEqualTo(Instant.parse("2026-06-20T09:00:00Z"));
    assertThat(loaded.confidenceScore).isEqualTo(0.85);
    assertThat(loaded.callerRef).isEqualTo("case:abc-123/pi:step-7");
    assertThat(loaded.parentId).isEqualTo(wi.parentId);
    assertThat(loaded.scope).isEqualTo("casehubio/devtown/pr-review");
    assertThat(loaded.templateId).isEqualTo(wi.templateId);
    assertThat(loaded.permittedOutcomes).isEqualTo("[\"approved\",\"rejected\",\"needs-revision\"]");
    assertThat(loaded.excludedUsers).isEqualTo("dave,eve");
    assertThat(loaded.outcome).isEqualTo("approved");
    assertThat(loaded.inputDataSchema).isEqualTo("{\"type\":\"object\"}");
    assertThat(loaded.outputDataSchema).isEqualTo("{\"type\":\"string\"}");
}
```

Add this import at the top of the test file:
```java
import io.casehub.work.api.DeclineTarget;
import java.util.UUID;
```

- [ ] **Step 2: Run test to verify it fails**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#put_and_get_roundtrip_allFields`

Expected: FAIL — the 13 fields are not mapped, assertions on `accumulatedUnclaimedSeconds`, `callerRef`, etc. fail with values being null/0.

- [ ] **Step 3: Add 13 fields to MongoWorkItemDocument**

In `MongoWorkItemDocument.java`, add these fields after `public List<MongoLabel> labels = new ArrayList<>();` (line 67):

```java
public Long version;
public long accumulatedUnclaimedSeconds;
public Instant lastReturnedToPoolAt;
public Double confidenceScore;
public String callerRef;
public String parentId;
public String scope;
public String templateId;
public String permittedOutcomes;
public List<String> excludedUsers = new ArrayList<>();
public String outcome;
public String inputDataSchema;
public String outputDataSchema;
```

- [ ] **Step 4: Update from() to map all 13 fields**

In `MongoWorkItemDocument.from()`, add these lines before the `if (wi.labels != null)` block:

```java
doc.version = wi.version;
doc.accumulatedUnclaimedSeconds = wi.accumulatedUnclaimedSeconds;
doc.lastReturnedToPoolAt = wi.lastReturnedToPoolAt;
doc.confidenceScore = wi.confidenceScore;
doc.callerRef = wi.callerRef;
doc.parentId = wi.parentId != null ? wi.parentId.toString() : null;
doc.scope = wi.scope;
doc.templateId = wi.templateId != null ? wi.templateId.toString() : null;
doc.permittedOutcomes = wi.permittedOutcomes;
doc.excludedUsers = splitCsv(wi.excludedUsers);
doc.outcome = wi.outcome;
doc.inputDataSchema = wi.inputDataSchema;
doc.outputDataSchema = wi.outputDataSchema;
```

- [ ] **Step 5: Update toDomain() to map all 13 fields**

In `MongoWorkItemDocument.toDomain()`, add these lines before the `wi.labels = ...` block:

```java
wi.version = version;
wi.accumulatedUnclaimedSeconds = accumulatedUnclaimedSeconds;
wi.lastReturnedToPoolAt = lastReturnedToPoolAt;
wi.confidenceScore = confidenceScore;
wi.callerRef = callerRef;
wi.parentId = parentId != null ? UUID.fromString(parentId) : null;
wi.scope = scope;
wi.templateId = templateId != null ? UUID.fromString(templateId) : null;
wi.permittedOutcomes = permittedOutcomes;
wi.excludedUsers = joinCsv(excludedUsers);
wi.outcome = outcome;
wi.inputDataSchema = inputDataSchema;
wi.outputDataSchema = outputDataSchema;
```

- [ ] **Step 6: Run test to verify it passes**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#put_and_get_roundtrip_allFields`

Expected: PASS

- [ ] **Step 7: Run all existing tests to verify no regressions**

Run: `scripts/mvn-test persistence-mongodb`

Expected: All tests PASS. The new fields are additive — existing tests don't set them, so `from()` maps nulls/defaults and `toDomain()` returns nulls/defaults.

- [ ] **Step 8: Commit**

```bash
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java
git commit -m "feat(#270): add 13 missing fields to MongoWorkItemDocument

Map version, accumulatedUnclaimedSeconds, lastReturnedToPoolAt,
confidenceScore, callerRef, parentId, scope, templateId,
permittedOutcomes, excludedUsers, outcome, inputDataSchema,
outputDataSchema in both from() and toDomain().

excludedUsers stored as List<String> (consistent with
candidateGroups/candidateUsers).

Refs casehubio/work#270"
```

---

### Task 2: MongoWorkItemStore.put() — optimistic concurrency control

**Files:**
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java`
- Modify: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java`

- [ ] **Step 1: Write OCC tests**

Add these tests to `MongoWorkItemStoreTest.java`:

```java
// ── OCC (Optimistic Concurrency Control) ─────────────────────────────────

@Test
void put_setsVersionToZero_onInsert() {
    final WorkItem wi = pending("alice", "Version insert test");
    store.put(wi);

    assertThat(wi.version).isEqualTo(0L);

    final WorkItem loaded = store.get(wi.id).orElseThrow();
    assertThat(loaded.version).isEqualTo(0L);
}

@Test
void put_incrementsVersion_onUpdate() {
    final WorkItem wi = pending("alice", "Version update test");
    store.put(wi);
    assertThat(wi.version).isEqualTo(0L);

    wi.status = WorkItemStatus.ASSIGNED;
    wi.assigneeId = "bob";
    store.put(wi);
    assertThat(wi.version).isEqualTo(1L);

    final WorkItem loaded = store.get(wi.id).orElseThrow();
    assertThat(loaded.version).isEqualTo(1L);
    assertThat(loaded.status).isEqualTo(WorkItemStatus.ASSIGNED);
}

@Test
void put_throwsOptimisticLockException_onStaleVersion() {
    final WorkItem wi = pending("alice", "OCC test");
    store.put(wi);

    // Two readers get the same version
    final WorkItem reader1 = store.get(wi.id).orElseThrow();
    final WorkItem reader2 = store.get(wi.id).orElseThrow();

    assertThat(reader1.version).isEqualTo(0L);
    assertThat(reader2.version).isEqualTo(0L);

    // Reader 1 updates successfully
    reader1.status = WorkItemStatus.ASSIGNED;
    reader1.assigneeId = "bob";
    store.put(reader1);
    assertThat(reader1.version).isEqualTo(1L);

    // Reader 2 attempts to update with stale version — should fail
    reader2.status = WorkItemStatus.ASSIGNED;
    reader2.assigneeId = "carol";
    assertThatThrownBy(() -> store.put(reader2))
            .isInstanceOf(OptimisticLockException.class)
            .hasMessageContaining("Version conflict");
}
```

Add these imports at the top of the test file:
```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import jakarta.persistence.OptimisticLockException;
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#put_setsVersionToZero_onInsert+put_incrementsVersion_onUpdate+put_throwsOptimisticLockException_onStaleVersion`

Expected: FAIL — current `put()` uses `persistOrUpdate()` with no version management. `put_setsVersionToZero_onInsert` fails because version stays at its initial `0L` from the field initializer but isn't explicitly set by the store. `put_throwsOptimisticLockException_onStaleVersion` fails because no OCC check exists.

- [ ] **Step 3: Rewrite MongoWorkItemStore.put() with OCC**

Replace the entire `put()` method in `MongoWorkItemStore.java` with:

```java
@Override
public WorkItem put(final WorkItem workItem) {
    if (workItem.id == null) {
        workItem.id = UUID.randomUUID();
    }
    if (workItem.tenancyId == null) {
        workItem.tenancyId = currentPrincipal.tenancyId();
    }
    final Instant now = Instant.now();
    if (workItem.createdAt == null) {
        workItem.createdAt = now;
    }
    workItem.updatedAt = now;

    final String idStr = workItem.id.toString();

    final boolean exists = MongoWorkItemDocument.find(
            new Document("_id", idStr)).firstResult() != null;

    if (!exists) {
        workItem.version = 0L;
        MongoWorkItemDocument.from(workItem).persist();
    } else {
        final MongoWorkItemDocument doc = MongoWorkItemDocument.from(workItem);
        final long currentVersion = workItem.version != null ? workItem.version : 0L;
        doc.version = currentVersion + 1;

        final Document filter = new Document("_id", idStr)
                .append("version", workItem.version);

        final com.mongodb.client.result.UpdateResult result =
                MongoWorkItemDocument.mongoCollection().replaceOne(filter, doc);

        if (result.getModifiedCount() == 0) {
            throw new OptimisticLockException(
                    "Version conflict on WorkItem " + idStr);
        }

        workItem.version = doc.version;
    }
    return workItem;
}
```

Add these imports to `MongoWorkItemStore.java`:
```java
import jakarta.persistence.OptimisticLockException;
```

- [ ] **Step 4: Run OCC tests to verify they pass**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#put_setsVersionToZero_onInsert+put_incrementsVersion_onUpdate+put_throwsOptimisticLockException_onStaleVersion`

Expected: PASS

- [ ] **Step 5: Run all tests to verify no regressions**

Run: `scripts/mvn-test persistence-mongodb`

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java
git commit -m "feat(#270): implement OCC in MongoWorkItemStore.put()

Replace persistOrUpdate() upsert with version-checked replaceOne.
Insert path: version = 0, persist(). Update path: replaceOne with
filter {_id, version} — modifiedCount 0 throws OptimisticLockException.

Uses exists-check pattern (matching SpawnGroupStore/ScheduleStore).
Uses replaceOne instead of findOneAndUpdate — avoids enumerating
44 fields in Updates.set(); from() handles the full document.

Null-safe version arithmetic handles migration of existing documents
written before this change (version field absent → null).

Refs casehubio/work#270"
```

---

### Task 3: MongoWorkItemStore.buildFilter() — outcome filter fix

**Files:**
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java`
- Modify: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java`

- [ ] **Step 1: Write outcome filter test**

Add this test to `MongoWorkItemStoreTest.java`:

```java
@Test
void scan_byOutcome_filtersCorrectly() {
    final WorkItem approved = pending("alice", "Approved item");
    approved.status = WorkItemStatus.COMPLETED;
    approved.outcome = "approved";
    store.put(approved);

    final WorkItem rejected = pending("alice", "Rejected item");
    rejected.status = WorkItemStatus.REJECTED;
    rejected.outcome = "rejected";
    store.put(rejected);

    final WorkItem noOutcome = pending("alice", "No outcome");
    store.put(noOutcome);

    final List<WorkItem> results = store.scan(
            WorkItemQuery.builder().outcome("approved").build());

    assertThat(results).hasSize(1);
    assertThat(results.get(0).title).isEqualTo("Approved item");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#scan_byOutcome_filtersCorrectly`

Expected: FAIL — `buildFilter()` ignores outcome, so all 3 items are returned.

- [ ] **Step 3: Add outcome clause to buildFilter()**

In `MongoWorkItemStore.buildFilter()`, add this block after the label pattern block and before the final `if (ands.isEmpty())` return:

```java
// Outcome
if (q.outcome() != null) {
    ands.add(new Document("outcome", q.outcome()));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#scan_byOutcome_filtersCorrectly`

Expected: PASS

- [ ] **Step 5: Run all tests to verify no regressions**

Run: `scripts/mvn-test persistence-mongodb`

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java
git commit -m "fix(#270): add missing outcome clause to MongoWorkItemStore.buildFilter()

buildFilter() handled 11 of 12 WorkItemQuery dimensions but silently
ignored outcome. REST API GET /workitems?outcome=approved returned all
WorkItems regardless of outcome. JPA and InMemory stores handle it.

Refs casehubio/work#270"
```

---

### Task 4: MongoWorkItemStore — 5 query method overrides

**Files:**
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java`
- Modify: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java`

- [ ] **Step 1: Write query method tests**

Add these tests to `MongoWorkItemStoreTest.java`:

```java
// ── Query method overrides ───────────────────────────────────────────────

@Test
void findByCallerRef_returnsMatching() {
    final WorkItem wi = pending("alice", "CallerRef match");
    wi.callerRef = "case:abc-123/pi:step-7";
    store.put(wi);

    store.put(pending("bob", "No callerRef"));

    final java.util.Optional<WorkItem> result = store.findByCallerRef("case:abc-123/pi:step-7");
    assertThat(result).isPresent();
    assertThat(result.get().title).isEqualTo("CallerRef match");
}

@Test
void findByCallerRef_returnsEmpty_whenNotFound() {
    final WorkItem wi = pending("alice", "CallerRef item");
    wi.callerRef = "case:abc-123/pi:step-7";
    store.put(wi);

    assertThat(store.findByCallerRef("case:nonexistent/pi:nope")).isEmpty();
}

@Test
void findByParentId_returnsChildren() {
    final UUID parentId = UUID.randomUUID();

    final WorkItem child1 = pending("alice", "Child 1");
    child1.parentId = parentId;
    store.put(child1);

    final WorkItem child2 = pending("alice", "Child 2");
    child2.parentId = parentId;
    store.put(child2);

    store.put(pending("alice", "Standalone"));

    final List<WorkItem> children = store.findByParentId(parentId);
    assertThat(children).hasSize(2)
            .extracting(w -> w.title)
            .containsExactlyInAnyOrder("Child 1", "Child 2");
}

@Test
void findByParentIdExcludingStatuses_excludesTerminal() {
    final UUID parentId = UUID.randomUUID();

    final WorkItem active = pending("alice", "Active child");
    active.parentId = parentId;
    store.put(active);

    final WorkItem completed = withStatus("alice", "Completed child", WorkItemStatus.COMPLETED);
    completed.parentId = parentId;
    store.put(completed);

    final WorkItem faulted = withStatus("alice", "Faulted child", WorkItemStatus.FAULTED);
    faulted.parentId = parentId;
    store.put(faulted);

    final List<WorkItem> results = store.findByParentIdExcludingStatuses(parentId,
            List.of(WorkItemStatus.COMPLETED, WorkItemStatus.FAULTED));
    assertThat(results).hasSize(1);
    assertThat(results.get(0).title).isEqualTo("Active child");
}

@Test
void findByParentIdWithStatuses_filtersToGivenStatuses() {
    final UUID parentId = UUID.randomUUID();

    final WorkItem pending = pending("alice", "Pending child");
    pending.parentId = parentId;
    store.put(pending);

    final WorkItem assigned = withStatus("alice", "Assigned child", WorkItemStatus.ASSIGNED);
    assigned.parentId = parentId;
    store.put(assigned);

    final WorkItem completed = withStatus("alice", "Completed child", WorkItemStatus.COMPLETED);
    completed.parentId = parentId;
    store.put(completed);

    final List<WorkItem> results = store.findByParentIdWithStatuses(parentId,
            List.of(WorkItemStatus.PENDING, WorkItemStatus.ASSIGNED));
    assertThat(results).hasSize(2)
            .extracting(w -> w.title)
            .containsExactlyInAnyOrder("Pending child", "Assigned child");
}

@Test
void countByParentAndAssignee_excludesSelf() {
    final UUID parentId = UUID.randomUUID();

    final WorkItem child1 = withStatus("alice", "Child 1", WorkItemStatus.ASSIGNED);
    child1.parentId = parentId;
    child1.assigneeId = "bob";
    store.put(child1);

    final WorkItem child2 = withStatus("alice", "Child 2", WorkItemStatus.IN_PROGRESS);
    child2.parentId = parentId;
    child2.assigneeId = "bob";
    store.put(child2);

    final WorkItem completedChild = withStatus("alice", "Completed", WorkItemStatus.COMPLETED);
    completedChild.parentId = parentId;
    completedChild.assigneeId = "bob";
    store.put(completedChild);

    // Count bob's non-terminal instances excluding child1
    final long count = store.countByParentAndAssignee(parentId, "bob", child1.id);
    assertThat(count).isEqualTo(1L); // only child2 — child1 excluded, completedChild is terminal
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#findByCallerRef_returnsMatching+findByCallerRef_returnsEmpty_whenNotFound+findByParentId_returnsChildren+findByParentIdExcludingStatuses_excludesTerminal+findByParentIdWithStatuses_filtersToGivenStatuses+countByParentAndAssignee_excludesSelf`

Expected: FAIL — the default implementations on WorkItemStore linear-scan `scanAll()`. Now that `toDomain()` maps the fields, `findByCallerRef` and `findByParentId` will actually work via the defaults. But `countByParentAndAssignee` will fail because the default returns `0L` (no override in the current store). The tests for `findByParentIdExcludingStatuses` and `findByParentIdWithStatuses` may pass via defaults — the point of overriding is server-side filtering, not correctness. Run and observe which fail.

- [ ] **Step 3: Add 5 query method overrides to MongoWorkItemStore**

Add these methods to `MongoWorkItemStore.java` after the `scan()` method:

```java
@Override
public Optional<WorkItem> findByCallerRef(final String callerRef) {
    final Document filter = new Document("callerRef", callerRef)
            .append("tenancyId", currentPrincipal.tenancyId());
    final MongoWorkItemDocument doc = MongoWorkItemDocument.find(filter).firstResult();
    return Optional.ofNullable(doc).map(MongoWorkItemDocument::toDomain);
}

@Override
public List<WorkItem> findByParentId(final UUID parentId) {
    final Document filter = new Document("parentId", parentId.toString())
            .append("tenancyId", currentPrincipal.tenancyId());
    return MongoWorkItemDocument.<MongoWorkItemDocument> find(filter).list()
            .stream().map(MongoWorkItemDocument::toDomain).toList();
}

@Override
public List<WorkItem> findByParentIdExcludingStatuses(final UUID parentId,
        final List<WorkItemStatus> excludeStatuses) {
    final Document filter = new Document("parentId", parentId.toString())
            .append("tenancyId", currentPrincipal.tenancyId())
            .append("status", new Document("$nin",
                    excludeStatuses.stream().map(Enum::name).toList()));
    return MongoWorkItemDocument.<MongoWorkItemDocument> find(filter).list()
            .stream().map(MongoWorkItemDocument::toDomain).toList();
}

@Override
public List<WorkItem> findByParentIdWithStatuses(final UUID parentId,
        final List<WorkItemStatus> statuses) {
    final Document filter = new Document("parentId", parentId.toString())
            .append("tenancyId", currentPrincipal.tenancyId())
            .append("status", new Document("$in",
                    statuses.stream().map(Enum::name).toList()));
    return MongoWorkItemDocument.<MongoWorkItemDocument> find(filter).list()
            .stream().map(MongoWorkItemDocument::toDomain).toList();
}

@Override
public long countByParentAndAssignee(final UUID parentId, final String assigneeId,
        final UUID excludeId) {
    final List<String> terminalStatuses = java.util.Arrays.stream(WorkItemStatus.values())
            .filter(WorkItemStatus::isTerminal)
            .map(Enum::name)
            .toList();
    final Document filter = new Document("parentId", parentId.toString())
            .append("assigneeId", assigneeId)
            .append("tenancyId", currentPrincipal.tenancyId())
            .append("_id", new Document("$ne", excludeId.toString()))
            .append("status", new Document("$nin", terminalStatuses));
    return MongoWorkItemDocument.count(filter);
}
```

Add this import to `MongoWorkItemStore.java`:
```java
import io.casehub.work.runtime.model.WorkItemStatus;
```

- [ ] **Step 4: Run query tests to verify they pass**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#findByCallerRef_returnsMatching+findByCallerRef_returnsEmpty_whenNotFound+findByParentId_returnsChildren+findByParentIdExcludingStatuses_excludesTerminal+findByParentIdWithStatuses_filtersToGivenStatuses+countByParentAndAssignee_excludesSelf`

Expected: PASS

- [ ] **Step 5: Run all tests to verify no regressions**

Run: `scripts/mvn-test persistence-mongodb`

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java
git commit -m "feat(#270): add 5 query method overrides to MongoWorkItemStore

Override findByCallerRef, findByParentId,
findByParentIdExcludingStatuses, findByParentIdWithStatuses,
countByParentAndAssignee with proper MongoDB server-side filtered
queries instead of default linear-scan implementations.

countByParentAndAssignee uses WorkItemStatus.isTerminal() to derive
the terminal exclusion set dynamically (all 7 terminal statuses).

All queries are tenant-scoped via CurrentPrincipal.tenancyId().

Refs casehubio/work#270"
```

---

### Task 5: Tenant isolation test for new query methods

**Files:**
- Modify: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java`

- [ ] **Step 1: Add MutableCurrentPrincipal injection to test class**

`MongoWorkItemStoreTest` currently does not inject `MutableCurrentPrincipal`. Add the field after the existing `store` injection:

```java
@Inject
MutableCurrentPrincipal principal;
```

Update the `@BeforeEach` method `clearAll()` to also reset the principal:

```java
@BeforeEach
void clearAll() {
    principal.reset();
    MongoWorkItemDocument.deleteAll();
}
```

- [ ] **Step 2: Write tenant isolation test**

Add this test to `MongoWorkItemStoreTest.java`:

```java
// ── Tenant isolation ─────────────────────────────────────────────────────

@Test
void queryMethods_tenantIsolation() {
    principal.setTenancyId("tenant-a");
    final UUID parentId = UUID.randomUUID();

    final WorkItem wiA = pending("alice", "Tenant A item");
    wiA.callerRef = "case:shared-ref/pi:step-1";
    wiA.parentId = parentId;
    wiA.assigneeId = "bob";
    store.put(wiA);

    // Switch to tenant B — should not see tenant A's data
    principal.setTenancyId("tenant-b");

    assertThat(store.findByCallerRef("case:shared-ref/pi:step-1")).isEmpty();
    assertThat(store.findByParentId(parentId)).isEmpty();
    assertThat(store.findByParentIdExcludingStatuses(parentId,
            List.of(WorkItemStatus.COMPLETED))).isEmpty();
    assertThat(store.findByParentIdWithStatuses(parentId,
            List.of(WorkItemStatus.PENDING))).isEmpty();
    assertThat(store.countByParentAndAssignee(parentId, "bob", UUID.randomUUID()))
            .isEqualTo(0L);
}
```

- [ ] **Step 3: Run test to verify it passes**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoWorkItemStoreTest#queryMethods_tenantIsolation`

Expected: PASS — all query overrides include `tenancyId` in their filters.

- [ ] **Step 4: Run all tests to verify no regressions**

Run: `scripts/mvn-test persistence-mongodb`

Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoWorkItemStoreTest.java
git commit -m "test(#270): add tenant isolation test for new query methods

Verify findByCallerRef, findByParentId, findByParentIdExcludingStatuses,
findByParentIdWithStatuses, and countByParentAndAssignee all enforce
tenant isolation — tenant B cannot see tenant A's data.

Refs casehubio/work#270"
```

---

### Task 6: Final verification

- [ ] **Step 1: Run the full persistence-mongodb test suite**

Run: `scripts/mvn-test persistence-mongodb`

Expected: All tests PASS (existing + new).

- [ ] **Step 2: Compile dependent modules to verify no breakage**

Run: `scripts/mvn-compile integration-tests`

Expected: Compiles successfully. The document and store changes are additive — no API changes to `WorkItemStore` interface (methods were already declared, just not overridden).
