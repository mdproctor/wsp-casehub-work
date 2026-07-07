# WorkItem Types and Labels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #291 — feat: add types: Set<Path> to WorkItemTemplate and WorkItem
**Issue group:** #291

**Goal:** Adopt the platform types/labels convention on WorkItem and WorkItemTemplate — add `types: Set<Path>`, remove `category`, update all 49 callers across 12 modules.

**Architecture:** WorkItemTemplate stores types as `typePaths` (JSON TEXT, same pattern as `labelPaths`). WorkItem stores types as `@ElementCollection` join table `work_item_type` with `Set<WorkItemType>`. `category` is removed from both entities and all API types. Path validation via `Path.parse()` at both creation paths (template instantiation and direct creation).

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate/Panache, H2 (test), PostgreSQL (prod), MongoDB (optional persistence)

## Global Constraints

- **Flyway migration V42** in the runtime V1–V999 range at `runtime/src/main/resources/db/work/migration/`
- **Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>` — always use `-pl`
- **Use `scripts/` helper scripts** for test execution with timeouts
- **H2 test URL must include** `MODE=PostgreSQL;DB_CLOSE_DELAY=-1`
- **`io.casehub.platform.api.path.Path`** is already on the classpath from `casehub-platform-api`
- **`PathAttributeConverter`** already exists at `runtime/src/main/java/io/casehub/work/runtime/model/PathAttributeConverter.java`
- **No backward compatibility.** Removing `category` is a hard break — the compiler catches every caller.

---

### Task 1: Foundation — WorkItemType, Flyway V42, API types

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemType.java`
- Create: `runtime/src/main/resources/db/work/migration/V42__workitem_types.sql`
- Modify: `api/src/main/java/io/casehub/work/api/WorkItemCreateRequest.java`
- Modify: `api/src/main/java/io/casehub/work/api/SelectionContext.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/model/WorkItemTypeTest.java`
- Test: `api/src/test/java/io/casehub/work/api/WorkItemCreateRequestTest.java`

**Interfaces:**
- Produces: `WorkItemType` embeddable (used by Task 2), `WorkItemCreateRequest.types` field (used by Tasks 2, 4), `SelectionContext` with types (used by Tasks 2, 6)

- [ ] **Step 1: Write WorkItemType embeddable**

Create `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemType.java`:

```java
package io.casehub.work.runtime.model;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;

@Embeddable
public class WorkItemType {

    @Column(name = "path", nullable = false, length = 500)
    public String path;

    public WorkItemType() {}

    public WorkItemType(String path) {
        this.path = path;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof WorkItemType other)) return false;
        return java.util.Objects.equals(path, other.path);
    }

    @Override
    public int hashCode() {
        return java.util.Objects.hash(path);
    }
}
```

- [ ] **Step 2: Write WorkItemType unit test**

Create `runtime/src/test/java/io/casehub/work/runtime/model/WorkItemTypeTest.java`:

```java
package io.casehub.work.runtime.model;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class WorkItemTypeTest {

    @Test
    void equals_samePath() {
        assertThat(new WorkItemType("approval")).isEqualTo(new WorkItemType("approval"));
    }

    @Test
    void equals_differentPath() {
        assertThat(new WorkItemType("approval")).isNotEqualTo(new WorkItemType("review"));
    }

    @Test
    void hashCode_samePath() {
        assertThat(new WorkItemType("approval").hashCode())
                .isEqualTo(new WorkItemType("approval").hashCode());
    }

    @Test
    void noArgConstructor() {
        WorkItemType type = new WorkItemType();
        assertThat(type.path).isNull();
    }
}
```

- [ ] **Step 3: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemTypeTest -pl runtime`
Expected: PASS

- [ ] **Step 4: Write Flyway migration V42**

Create `runtime/src/main/resources/db/work/migration/V42__workitem_types.sql`:

```sql
-- WorkItemTemplate: add type_paths column
ALTER TABLE work_item_template ADD COLUMN type_paths TEXT;

-- WorkItemTemplate: drop category
ALTER TABLE work_item_template DROP COLUMN category;

-- WorkItem: create types join table
CREATE TABLE work_item_type (
    work_item_id UUID NOT NULL REFERENCES work_item(id),
    path         VARCHAR(500) NOT NULL,
    CONSTRAINT uq_work_item_type UNIQUE (work_item_id, path)
);
CREATE INDEX idx_work_item_type_path ON work_item_type(path);

-- WorkItem: drop category
ALTER TABLE work_item DROP COLUMN category;
```

- [ ] **Step 5: Modify WorkItemCreateRequest — remove category, add types**

In `api/src/main/java/io/casehub/work/api/WorkItemCreateRequest.java`:

Remove the `category` field (line ~12), the constructor assignment (line ~55), the builder field, the builder copy-constructor line, the builder setter method `.category()`, and the `equals`/`hashCode`/`toString` references.

Add a new field:

```java
public final List<String> types;
```

Add to constructor: `this.types = b.types;`

Add builder field: `private List<String> types;`

Add builder setter: `public Builder types(final List<String> v) { this.types = v; return this; }`

Add builder copy: `this.types = src.types;`

Add to `equals`: `&& Objects.equals(types, r.types)`

Add to `hashCode`: include `types`

Update `toString` to replace `category` with `types`.

- [ ] **Step 6: Modify SelectionContext — replace category with types**

Read the current `SelectionContext` file via IntelliJ MCP. Replace the `String category` constructor parameter and field with `List<String> types`. Update all accessor methods accordingly. The record/class carries: `types, priority, capabilities, candidateGroups, candidateUsers, title, description, excludedUsers`.

- [ ] **Step 7: Update WorkItemCreateRequestTest**

Update `api/src/test/java/io/casehub/work/api/WorkItemCreateRequestTest.java` to use `.types(List.of("approval"))` instead of `.category("approval")`. Verify the builder round-trip includes types.

- [ ] **Step 8: Run API module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: PASS (some tests may fail if they reference category — fix them)

- [ ] **Step 9: Commit**

```
feat(#291): add WorkItemType embeddable, V42 migration, update API types

Refs #291
```

---

### Task 2: Domain model — WorkItem and WorkItemTemplate

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemTemplate.java`

**Interfaces:**
- Consumes: `WorkItemType` from Task 1
- Produces: `WorkItem.types: Set<WorkItemType>` (used by Tasks 3–7), `WorkItemTemplate.typePaths: String` (used by Task 3)

- [ ] **Step 1: Modify WorkItem — remove category, add types**

In `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`:

Remove `category` field (line 77) and its Javadoc.

Add imports:

```java
import java.util.LinkedHashSet;
import java.util.Set;
import jakarta.persistence.CollectionTable;
import jakarta.persistence.ElementCollection;
```

(Some may already be imported for labels — check and add only missing ones.)

Add types field after the labels section (after line ~241):

```java
@ElementCollection(fetch = FetchType.EAGER)
@CollectionTable(name = "work_item_type", joinColumns = @JoinColumn(name = "work_item_id"))
public Set<WorkItemType> types = new LinkedHashSet<>();
```

- [ ] **Step 2: Modify WorkItemTemplate — remove category, add typePaths**

In `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemTemplate.java`:

Remove `category` field (line 69) and its Javadoc.

Add `typePaths` field after `labelPaths` (after line ~131):

```java
/**
 * JSON array of type path strings applied at instantiation.
 * Example: {@code ["approval", "compliance/audit"]}
 * Stored as a JSON string; parsed and validated via Path.parse() at instantiation time.
 */
@Column(name = "type_paths", columnDefinition = "TEXT")
public String typePaths;
```

- [ ] **Step 3: Commit**

```
feat(#291): add types field to WorkItem and WorkItemTemplate, remove category

Refs #291
```

---

### Task 3: Service layer — WorkItemService and WorkItemTemplateService

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceCreateFromTemplateTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateInstantiateTest.java`

**Interfaces:**
- Consumes: `WorkItemType` from Task 1, `WorkItem.types` from Task 2, `WorkItemTemplate.typePaths` from Task 2, `WorkItemCreateRequest.types` from Task 1
- Produces: `WorkItemTemplateService.parseTypes()` (static, testable), types flow through create/clone/instantiate

- [ ] **Step 1: Write parseTypes() test**

In `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceTest.java`, add tests:

```java
@Test
void parseTypes_validJson() {
    WorkItemTemplate template = new WorkItemTemplate();
    template.typePaths = "[\"approval\", \"compliance/audit\"]";
    List<WorkItemType> types = WorkItemTemplateService.parseTypes(template);
    assertThat(types).hasSize(2);
    assertThat(types.get(0).path).isEqualTo("approval");
    assertThat(types.get(1).path).isEqualTo("compliance/audit");
}

@Test
void parseTypes_nullTypePaths() {
    WorkItemTemplate template = new WorkItemTemplate();
    template.typePaths = null;
    assertThat(WorkItemTemplateService.parseTypes(template)).isEmpty();
}

@Test
void parseTypes_blankTypePaths() {
    WorkItemTemplate template = new WorkItemTemplate();
    template.typePaths = "  ";
    assertThat(WorkItemTemplateService.parseTypes(template)).isEmpty();
}

@Test
void parseTypes_invalidJson() {
    WorkItemTemplate template = new WorkItemTemplate();
    template.typePaths = "not json";
    assertThat(WorkItemTemplateService.parseTypes(template)).isEmpty();
}

@Test
void parseTypes_invalidPath_rejects() {
    WorkItemTemplate template = new WorkItemTemplate();
    template.typePaths = "[\"/leading/slash\"]";
    assertThrows(IllegalArgumentException.class,
            () -> WorkItemTemplateService.parseTypes(template));
}
```

- [ ] **Step 2: Run tests — expect failure (parseTypes not yet implemented)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemTemplateServiceTest -pl runtime`
Expected: FAIL — `parseTypes` method does not exist

- [ ] **Step 3: Implement parseTypes()**

In `WorkItemTemplateService.java`, add after `parseLabels()` (around line 230):

```java
public static List<WorkItemType> parseTypes(final WorkItemTemplate template) {
    if (template.typePaths == null || template.typePaths.isBlank()) {
        return List.of();
    }
    try {
        final List<String> paths = MAPPER.readValue(template.typePaths, new TypeReference<>() {});
        final List<WorkItemType> result = new ArrayList<>();
        for (final String path : paths) {
            if (path != null && !path.isBlank()) {
                Path.parse(path);
                result.add(new WorkItemType(path));
            }
        }
        return result;
    } catch (IllegalArgumentException e) {
        throw e;
    } catch (Exception e) {
        return List.of();
    }
}
```

Add import: `import io.casehub.platform.api.path.Path;` and `import io.casehub.work.runtime.model.WorkItemType;`

- [ ] **Step 4: Run tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemTemplateServiceTest -pl runtime`
Expected: PASS

- [ ] **Step 5: Update mergeRequestWithTemplate()**

In `WorkItemTemplateService.mergeRequestWithTemplate()` (around line 120), replace:

```java
.category(request.category != null ? request.category : template.category)
```

With:

```java
.types(request.types != null ? request.types
        : parseTypes(template).stream().map(t -> t.path).toList())
```

- [ ] **Step 6: Update createFromTemplate() to apply types**

In `createFromTemplate()` (around line 90), after `final WorkItem workItem = workItemService.create(merged);`, add type application before the label loop. But since types are now part of the WorkItemCreateRequest (via the merge), and `WorkItemService.create()` will handle them (see next step), no additional code is needed here — types flow through the merged request.

- [ ] **Step 7: Update WorkItemService.create() — copy types with validation**

In `WorkItemService.create()` (around line 105), replace:

```java
item.category = request.category;
```

With:

```java
if (request.types != null) {
    for (final String typePath : request.types) {
        io.casehub.platform.api.path.Path.parse(typePath);
        item.types.add(new WorkItemType(typePath));
    }
}
```

- [ ] **Step 8: Update WorkItemService.clone()**

In `WorkItemService.clone()` (around line 817), replace:

```java
.category(source.category)
```

With:

```java
.types(source.types.stream().map(t -> t.path).toList())
```

- [ ] **Step 9: Update existing template/service tests that reference category**

Search for `category` in all test files under `runtime/src/test/` and update:
- `WorkItemTemplateServiceCreateFromTemplateTest` — use `typePaths` instead of `category` on templates, `types` on requests
- `WorkItemTemplateInstantiateTest` — same
- `WorkItemServiceTest` — update matchesFilters and any category-based assertions
- `WorkItemCreateRequestAuditDetailTest` — remove category from builders

- [ ] **Step 10: Run runtime module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: Compilation failures in files not yet updated (query, event, assignment). Fix compile errors in runtime module tests only — other modules are updated in later tasks.

- [ ] **Step 11: Commit**

```
feat(#291): implement parseTypes(), types flow through create/clone/instantiate

Refs #291
```

---

### Task 4: Query and event layer

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemQuery.java`
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEvent.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/RoundRobinAssignmentStrategy.java`
- Test: `persistence-memory/src/test/java/io/casehub/work/memory/InMemoryRepositoryTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/event/WorkItemLifecycleEventTest.java`

**Interfaces:**
- Consumes: `WorkItem.types` from Task 2, `WorkItemType` from Task 1
- Produces: `WorkItemQuery.type(String)` (used by Tasks 5, 6), `WorkItemLifecycleEvent.types()` (used by Tasks 5, 6), ancestor matching in InMemoryWorkItemStore (used by all store consumers)

- [ ] **Step 1: Write ancestor matching test**

In `persistence-memory/src/test/java/io/casehub/work/memory/InMemoryRepositoryTest.java`, replace `findInbox_categoryFilter` test (around line 177):

```java
@Test
void scan_byType_exactMatch() {
    WorkItem wi = createWorkItem();
    wi.types.add(new WorkItemType("compliance/audit"));
    store.put(wi);

    List<WorkItem> result = store.scan(WorkItemQuery.builder().type("compliance/audit").build());
    assertThat(result).hasSize(1);
}

@Test
void scan_byType_ancestorMatch() {
    WorkItem wi = createWorkItem();
    wi.types.add(new WorkItemType("compliance/audit"));
    store.put(wi);

    List<WorkItem> result = store.scan(WorkItemQuery.builder().type("compliance").build());
    assertThat(result).hasSize(1);
}

@Test
void scan_byType_noMatch() {
    WorkItem wi = createWorkItem();
    wi.types.add(new WorkItemType("approval"));
    store.put(wi);

    List<WorkItem> result = store.scan(WorkItemQuery.builder().type("compliance").build());
    assertThat(result).isEmpty();
}
```

- [ ] **Step 2: Run test — expect failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryRepositoryTest -pl persistence-memory`
Expected: FAIL — `type()` method does not exist on WorkItemQuery

- [ ] **Step 3: Update WorkItemQuery — replace category with type**

In `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemQuery.java`:

Replace all occurrences of `category` (field, constructor, accessor, builder method, toBuilder) with `type`. The field type stays `String`. Update the Javadoc to describe ancestor matching semantics.

- [ ] **Step 4: Update InMemoryWorkItemStore.matchesFilters()**

In `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemStore.java` (around line 170), replace:

```java
if (q.category() != null && !q.category().equals(wi.category)) {
    return false;
}
```

With:

```java
if (q.type() != null) {
    final io.casehub.platform.api.path.Path queryPath = io.casehub.platform.api.path.Path.parse(q.type());
    boolean matched = wi.types.stream().anyMatch(t -> {
        final io.casehub.platform.api.path.Path typePath = io.casehub.platform.api.path.Path.parse(t.path);
        return typePath.equals(queryPath) || queryPath.isAncestorOf(typePath);
    });
    if (!matched) return false;
}
```

- [ ] **Step 5: Run test — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryRepositoryTest -pl persistence-memory`
Expected: PASS

- [ ] **Step 6: Update WorkItemLifecycleEvent — add types field**

In `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEvent.java`:

Add field: `private final List<String> types;`

Add to constructor parameter list and assignment.

Add accessor:

```java
@JsonProperty("types")
public List<String> types() {
    return types;
}
```

Update both `of()` factory methods to extract types from workItem:

```java
workItem.types.stream().map(t -> t.path).toList()
```

Update `fromWire()` to accept and pass `List<String> types`.

- [ ] **Step 7: Update WorkItemContextBuilder.toMap()**

In `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java` (line 41), replace:

```java
map.put("category", workItem.category);
```

With:

```java
map.put("types", workItem.types.stream().map(t -> t.path).toList());
```

- [ ] **Step 8: Update WorkItemAssignmentService**

In `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java` (around line 132), replace:

```java
workItem.category,
```

With:

```java
workItem.types.stream().map(t -> t.path).toList(),
```

(This matches the new SelectionContext constructor from Task 1.)

- [ ] **Step 9: Update RoundRobinAssignmentStrategy**

In `runtime/src/main/java/io/casehub/work/runtime/multiinstance/RoundRobinAssignmentStrategy.java` (around line 98), replace:

```java
child.category,
```

With:

```java
child.types.stream().map(t -> t.path).toList(),
```

- [ ] **Step 10: Update event and runtime tests**

Fix all compile errors in runtime tests that reference `category` on WorkItem, WorkItemQuery, or WorkItemLifecycleEvent. Key files:
- `WorkItemLifecycleEventTest` — update event construction
- `WorkItemLifecycleEventOutcomeTest`
- `WorkCloudEventRoundTripTest` — replace `category = "test-category"` with types
- `FilterRegistryEngineTest` — replace `wi.category = "finance"` with `wi.types.add(new WorkItemType("finance"))`
- `WorkItemServiceTest` — update matchesFilters

- [ ] **Step 11: Run runtime + persistence-memory tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: PASS (some tests in other modules will still fail — those are fixed in later tasks)

- [ ] **Step 12: Commit**

```
feat(#291): replace category with type filter in WorkItemQuery, add types to lifecycle event

Refs #291
```

---

### Task 5: REST layer

**Files:**
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemResponse.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemWithAuditResponse.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemMapper.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemResource.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemTemplateResource.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/WorkItemLabelResponse.java` (if needed)
- Test: REST integration tests in `rest/src/test/java/io/casehub/work/rest/`

**Interfaces:**
- Consumes: `WorkItem.types`, `WorkItemType`, `WorkItemTemplate.typePaths` from Tasks 1–2

- [ ] **Step 1: Update WorkItemResponse — replace category with types**

In `rest/src/main/java/io/casehub/work/rest/WorkItemResponse.java`, replace `String category` parameter with `List<String> types`.

- [ ] **Step 2: Update WorkItemWithAuditResponse — same change**

Replace `String category` with `List<String> types`.

- [ ] **Step 3: Update WorkItemMapper.toResponse() and toWithAudit()**

In `rest/src/main/java/io/casehub/work/rest/WorkItemMapper.java`:

Replace `wi.category` with `wi.types.stream().map(t -> t.path).toList()` in both mapper methods.

In `toServiceRequest()` (around line 69), replace `.category(...)` with `.types(...)` mapping from the REST create request.

- [ ] **Step 4: Update WorkItemResource.inbox() — replace @QueryParam("category") with @QueryParam("type")**

In `rest/src/main/java/io/casehub/work/rest/WorkItemResource.java` (around line 180):

Replace parameter `@QueryParam("category") final String category` with `@QueryParam("type") final String type`.

Replace filter: `if (category != null) stream = stream.filter(v -> category.equals(v.workItem().category));`

With ancestor-matching filter:

```java
if (type != null) {
    final io.casehub.platform.api.path.Path queryPath = io.casehub.platform.api.path.Path.parse(type);
    stream = stream.filter(v -> v.workItem().types.stream().anyMatch(t -> {
        final io.casehub.platform.api.path.Path typePath = io.casehub.platform.api.path.Path.parse(t.path);
        return typePath.equals(queryPath) || queryPath.isAncestorOf(typePath);
    }));
}
```

- [ ] **Step 5: Update WorkItemTemplateResource — add typePaths to CRUD**

In `rest/src/main/java/io/casehub/work/rest/WorkItemTemplateResource.java`:

In the create endpoint: read `typePaths` from the JSON body and set it on the template entity (same pattern as `labelPaths`).

In the update/patch endpoints: handle `typePaths` alongside existing fields.

In the response mapping: include `typePaths` in the template response.

Remove `category` from all template request/response handling.

- [ ] **Step 6: Update REST integration tests**

Key test files to update:
- `WorkItemTemplateTest` — create templates with `typePaths`, verify in response
- `WorkItemTemplateOutcomeTest` — remove category references
- `WorkItemTemplatePatchTest` — patch typePaths
- `WorkItemTemplateSchemaTest` — schema validation with typePaths
- `WorkItemCapabilityIT` — update WorkItem assertions
- All other REST tests referencing `category`

Add new test:
```java
@Test
void inbox_filterByType_ancestorMatch() {
    // Create WorkItem with type "compliance/audit"
    // GET /workitems/inbox?type=compliance
    // Assert: returned
}
```

- [ ] **Step 7: Run REST module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest`
Expected: PASS

- [ ] **Step 8: Commit**

```
feat(#291): update REST layer — types in responses, type query param for inbox filtering

Refs #291
```

---

### Task 6: Remaining modules — notifications, AI, reports, issue-tracker, queues, MongoDB

**Files:**
- Modify: `notifications/src/main/java/io/casehub/work/notifications/service/NotificationDispatcher.java`
- Modify: `notifications/src/main/java/io/casehub/work/notifications/channel/SlackNotificationChannel.java`
- Modify: `notifications/src/main/java/io/casehub/work/notifications/channel/TeamsNotificationChannel.java`
- Modify: `notifications/src/main/java/io/casehub/work/notifications/channel/HttpWebhookChannel.java`
- Modify: `ai/src/main/java/io/casehub/work/ai/suggestion/ResolutionSuggestionService.java`
- Modify: `ai/src/main/java/io/casehub/work/ai/escalation/EscalationSummaryService.java`
- Modify: `ai/src/main/java/io/casehub/work/ai/skill/ResolutionHistorySkillProfileProvider.java`
- Modify: `reports/src/main/java/io/casehub/work/reports/service/ReportService.java`
- Modify: `issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubIssueTrackerProvider.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/service/JexlConditionEvaluator.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/service/JqConditionEvaluator.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/api/FilterResource.java`
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java`
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java`
- Tests in each module

**Interfaces:**
- Consumes: `WorkItem.types`, `WorkItemQuery.type()`, `WorkItemLifecycleEvent.types()` from Tasks 1–4

- [ ] **Step 1: Notifications module — replace category with types**

**NotificationDispatcher** (line 74): replace `final String category = wi.category;` with `final List<String> types = wi.types.stream().map(t -> t.path).toList();` — update downstream usage.

**SlackNotificationChannel** (line 42): replace `"Category: " + wi.category` with `"Types: " + String.join(", ", wi.types.stream().map(t -> t.path).toList())`.

**TeamsNotificationChannel** (line 42): same pattern — `"Types: " + String.join(", ", wi.types.stream().map(t -> t.path).toList())`.

**HttpWebhookChannel** (line 47): replace `wi.category` parameter in `buildPayloadJson()` with types. Update `buildPayloadJson` method signature to accept `List<String> types` instead of `String category`.

- [ ] **Step 2: AI module — replace category with types**

**ResolutionSuggestionService** (lines 107–135):

`findExamples()`: replace `workItem.category` checks with types — use first type for similarity matching:
```java
if (!workItem.types.isEmpty()) {
    final String primaryType = workItem.types.iterator().next().path;
    final List<WorkItem> byType = completedWithResolution(primaryType);
    if (!byType.isEmpty()) return byType;
}
```

`completedWithResolution()`: change parameter from `String category` to `String type`, use `.type(type)` on query builder.

`buildPrompt()`: replace `current.category` with types list.

**EscalationSummaryService** (lines 131–132): replace `wi.category` with types:
```java
if (!wi.types.isEmpty()) {
    sb.append("Types: ").append(wi.types.stream().map(t -> t.path).collect(Collectors.joining(", "))).append("\n");
}
```

**ResolutionHistorySkillProfileProvider** (lines 59–64): replace `wi.category != null` filter with `!wi.types.isEmpty()`, and `.groupingBy(wi -> wi.category` with grouping by primary type:
```java
.filter(wi -> !wi.types.isEmpty())
.collect(Collectors.groupingBy(wi -> wi.types.iterator().next().path, Collectors.counting()));
```

- [ ] **Step 3: Reports module — replace category with types**

**ReportService** (lines 87–95): replace `wi.category` in `SlaBreachItem` constructor with types list. Update `byCategory` map to `byType`. Replace `wi.category != null` guard with `!wi.types.isEmpty()` and iterate types.

- [ ] **Step 4: Issue-tracker module — replace category with types**

**GitHubIssueTrackerProvider** (lines 370–371): replace single `category:` label with multiple `type:` labels:
```java
for (final WorkItemType type : workItem.types) {
    labels.add("type:" + type.path.toLowerCase());
}
```

- [ ] **Step 5: Queues module — replace category with types**

**JexlConditionEvaluator** (line 48): replace `map.put("category", wi.category)` with `map.put("types", wi.types.stream().map(t -> t.path).toList())`.

**JqConditionEvaluator** (line 45): same change.

**FilterResource.evaluate()** (line 144): replace `wi.category = req.workItem().category()` with types population. Update the `FilterEvaluationRequest.WorkItemData` record to use `types` instead of `category`.

- [ ] **Step 6: MongoDB persistence — replace category with types**

**MongoWorkItemDocument**: replace `String category` field with `List<String> types`. In `from(WorkItem)`: map `wi.types.stream().map(t -> t.path).toList()`. In `toDomain()`: iterate and create `WorkItemType` entries.

**MongoWorkItemStore.buildFilter()**: replace category filter with type filter using `$regex` for ancestor matching:
```java
if (q.type() != null) {
    filters.add(Filters.regex("types", "^" + Pattern.quote(q.type()) + "(/|$)"));
}
```

- [ ] **Step 7: Update all tests in affected modules**

Fix compile errors in test files for: notifications, ai, reports, issue-tracker, queues, persistence-mongodb. Key files:
- `EscalationSummaryServiceTest` — replace `wi.category = "legal"` with types
- `ResolutionSuggestionServiceTest` — replace category with types
- `ResolutionHistorySkillProfileProviderTest` — replace category with types
- `JexlConditionEvaluatorTest` — replace `wi.category = "legal"` with types
- `MongoWorkItemStoreTest` — replace all `wi.category` with types, rename `scan_byCategory` to `scan_byType`
- `GitHubLabelBuilderTest` — replace category with types

- [ ] **Step 8: Run all affected modules**

Run each module individually:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notifications
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ai
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl reports
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl issue-tracker
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb
```
Expected: PASS for each

- [ ] **Step 9: Commit**

```
feat(#291): migrate all modules from category to types — notifications, AI, reports, queues, MongoDB

Refs #291
```

---

### Task 7: Integration tests, cross-repo issues, final verification

**Files:**
- Modify: integration test files under `integration-tests/`
- No new files — cross-repo issues filed via `gh`

**Interfaces:**
- Consumes: everything from Tasks 1–6

- [ ] **Step 1: Update integration tests**

Fix any `category` references in `integration-tests/` module. These are black-box REST tests that exercise the full stack.

- [ ] **Step 2: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests`
Expected: PASS

- [ ] **Step 3: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,rest,persistence-memory,persistence-mongodb,notifications,ai,reports,issue-tracker,queues`
Expected: PASS across all modules

- [ ] **Step 4: Verify no category references remain**

Use IntelliJ MCP `ide_search_text` to search for `category` across all `.java` files in the project. Any remaining references are either:
- False positives (variable names, comments about the migration)
- Missed callers that need updating

Fix any real references found.

- [ ] **Step 5: File cross-repo GitHub issues**

File issues for deferred cross-repo work:

```bash
gh issue create --repo casehubio/garden --title "protocol: add definable-entity-types-labels convention" --body "..."
gh issue create --repo casehubio/parent --title "docs: add types/labels protocol cross-reference to PLATFORM.md" --body "..."
```

Update engine#653 with a comment referencing this implementation.

- [ ] **Step 6: Commit integration test updates**

```
test(#291): update integration tests for types/labels convention

Refs #291
```

---

## Task Dependencies

```
Task 1 (foundation) → Task 2 (domain model) → Task 3 (service layer) → Task 4 (query/event)
                                                                              ↓
                                                                    ┌─────────┼─────────┐
                                                                    ↓         ↓         ↓
                                                              Task 5     Task 6     Task 7
                                                              (REST)   (remaining)  (integ)
```

Tasks 1–4 are strictly sequential. Tasks 5–6 can run in parallel after Task 4. Task 7 runs last.
