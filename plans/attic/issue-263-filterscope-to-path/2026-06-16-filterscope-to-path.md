# FilterScope → Path Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Delete the `FilterScope` enum (PERSONAL/TEAM/ORG) from the queues module and replace every usage with `Path` from `casehub-platform-api`, matching the pattern established by #236 (`VocabularyScope → Path`).

**Architecture:** `PathAttributeConverter` (already in `runtime/`) handles JPA mapping. `Path.root()` replaces `FilterScope.ORG` as the default. `WorkItemFilterBean.scope()` is deleted — it was dead code the engine never called. Scope on stored entities is management metadata with no current enforcement.

**Tech Stack:** Java 21, Quarkus 3, JPA/Hibernate, `io.casehub.platform.api.path.Path`

**Spec:** `specs/issue-263-filterscope-to-path/2026-06-16-filterscope-to-path-design.md`

---

## File Map

| File | Action |
|------|--------|
| `queues/src/main/resources/db/work/migration/V2000__queues_schema.sql` | Modify — widen scope column, drop owner_id |
| `queues/src/main/java/io/casehub/work/queues/model/FilterScope.java` | **Delete** |
| `queues/src/main/java/io/casehub/work/queues/model/WorkItemFilter.java` | Modify — scope field type, drop ownerId |
| `queues/src/main/java/io/casehub/work/queues/model/QueueView.java` | Modify — scope field type, drop ownerId |
| `queues/src/main/java/io/casehub/work/queues/service/WorkItemFilterBean.java` | Modify — delete scope() method |
| `queues/src/main/java/io/casehub/work/queues/api/FilterResource.java` | Modify — request record, create, list, update |
| `queues/src/main/java/io/casehub/work/queues/api/QueueResource.java` | Modify — request record, create, list |
| `queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaWorkItemFilterStoreTenancyTest.java` | Modify — Path.root() |
| `queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaQueueViewStoreTenancyTest.java` | Modify — Path.root() |
| `queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaFilterChainStoreTenancyTest.java` | Modify — Path.root() |
| `queues-dashboard/src/main/java/io/casehub/work/dashboard/SecurityWritersFilter.java` | Modify — remove scope() override |
| `queues-dashboard/src/main/java/io/casehub/work/dashboard/ReviewStepService.java` | Modify — Path.root() |
| `queues-examples/src/main/java/io/casehub/work/examples/queues/review/SecurityWritersFilter.java` | Modify — remove scope() override |
| `queues-examples/src/main/java/io/casehub/work/examples/queues/security/SecurityEscalationScenario.java` | Modify — Path.root() ×6 |
| `queues-examples/src/main/java/io/casehub/work/examples/queues/triage/SupportTriageScenario.java` | Modify — Path.root() ×7 |
| `queues-examples/src/main/java/io/casehub/work/examples/queues/finance/FinanceApprovalScenario.java` | Modify — Path.root() ×4 |
| `queues-examples/src/main/java/io/casehub/work/examples/queues/legal/LegalRoutingScenario.java` | Modify — Path.root() ×4 |
| `queues-examples/src/main/java/io/casehub/work/examples/queues/lifecycle/QueueLifecycleScenario.java` | Modify — Path.root() ×1 |
| `queues-examples/src/main/java/io/casehub/work/examples/queues/review/DocumentReviewScenario.java` | Modify — Path.root() ×4 |
| `examples/src/main/java/io/casehub/work/examples/queues/QueueModuleScenario.java` | Modify — Path.of("team"), drop ownerId |

---

## Task 1: Update Schema

**Files:**
- Modify: `queues/src/main/resources/db/work/migration/V2000__queues_schema.sql`

No compilation impact — schema change only. The column widens from 20 to 500 chars and `owner_id` is dropped from both tables.

- [ ] **Step 1: Update V2000 schema**

Replace both table definitions in `V2000__queues_schema.sql`. Change `scope VARCHAR(20)` to `scope VARCHAR(500)` and remove the `owner_id VARCHAR(255)` line from both `work_item_filter` and `queue_view`.

`work_item_filter` table becomes:
```sql
CREATE TABLE work_item_filter (
    id                   UUID            PRIMARY KEY,
    name                 VARCHAR(255)    NOT NULL,
    scope                VARCHAR(500)    NOT NULL,
    condition_language   VARCHAR(20)     NOT NULL,
    condition_expression VARCHAR(4000),
    actions              VARCHAR(4000),
    active               BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at           TIMESTAMP       NOT NULL
);
```

`queue_view` table becomes:
```sql
CREATE TABLE queue_view (
    id                    UUID            PRIMARY KEY,
    name                  VARCHAR(255)    NOT NULL,
    label_pattern         VARCHAR(500)    NOT NULL,
    scope                 VARCHAR(500)    NOT NULL,
    additional_conditions VARCHAR(2000),
    sort_field            VARCHAR(50)     NOT NULL DEFAULT 'createdAt',
    sort_direction        VARCHAR(4)      NOT NULL DEFAULT 'ASC',
    created_at            TIMESTAMP       NOT NULL
);
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add queues/src/main/resources/db/work/migration/V2000__queues_schema.sql
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#263): widen scope column to VARCHAR(500), drop owner_id — V2000

Refs #263"
```

---

## Task 2: Update Tests (TDD — these will fail to compile after Task 3)

**Files:**
- Modify: `queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaWorkItemFilterStoreTenancyTest.java`
- Modify: `queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaQueueViewStoreTenancyTest.java`
- Modify: `queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaFilterChainStoreTenancyTest.java`

Update the tests to use `Path.root()` before changing the entity. This makes the intent explicit and produces the compilation failure that drives Task 3.

- [ ] **Step 1: Update `JpaWorkItemFilterStoreTenancyTest`**

Replace the import and the `newFilter` helper method:

```java
// Remove:
import io.casehub.work.queues.model.FilterScope;

// Add:
import io.casehub.platform.api.path.Path;
```

```java
private WorkItemFilter newFilter(String name) {
    WorkItemFilter f = new WorkItemFilter();
    f.name = name;
    f.scope = Path.root();
    f.conditionLanguage = "jexl";
    f.conditionExpression = "true";
    f.active = true;
    return f;
}
```

- [ ] **Step 2: Update `JpaQueueViewStoreTenancyTest`**

Replace the import and the `newQueueView` helper method:

```java
// Remove:
import io.casehub.work.queues.model.FilterScope;

// Add:
import io.casehub.platform.api.path.Path;
```

```java
private QueueView newQueueView(String name) {
    QueueView qv = new QueueView();
    qv.name = name;
    qv.labelPattern = "test/" + name + "/**";
    qv.scope = Path.root();
    return qv;
}
```

- [ ] **Step 3: Update `JpaFilterChainStoreTenancyTest`**

Replace the import and the `createParentFilter` helper method:

```java
// Remove:
import io.casehub.work.queues.model.FilterScope;

// Add:
import io.casehub.platform.api.path.Path;
```

```java
private UUID createParentFilter(String name) {
    WorkItemFilter f = new WorkItemFilter();
    f.name = name;
    f.scope = Path.root();
    f.conditionLanguage = "jexl";
    f.conditionExpression = "true";
    f.active = true;
    filterStore.put(f);
    return f.id;
}
```

- [ ] **Step 4: Verify tests fail to compile (expected)**

```bash
scripts/mvn-compile casehub-work-queues
```

Expected: **FAIL** — `Path` cannot be assigned to field of type `FilterScope`. This is the red state that Task 3 resolves.

---

## Task 3: Core Model — Delete FilterScope, Update Entities and SPI

**Files:**
- Delete: `queues/src/main/java/io/casehub/work/queues/model/FilterScope.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/model/WorkItemFilter.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/model/QueueView.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/service/WorkItemFilterBean.java`

- [ ] **Step 1: Delete `FilterScope.java`**

Use `git rm` so the deletion is staged and included in the Task 5 commit:

```bash
git -C /Users/mdproctor/claude/casehub/work rm queues/src/main/java/io/casehub/work/queues/model/FilterScope.java
```

- [ ] **Step 2: Update `WorkItemFilter.java`**

Replace the `scope` field (and remove `ownerId`). The full updated field block:

```java
// Remove these imports:
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;

// Add these imports:
import jakarta.persistence.Convert;
import io.casehub.platform.api.path.Path;
import io.casehub.work.runtime.model.PathAttributeConverter;
```

Replace the scope and ownerId fields:
```java
// Remove:
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 20)
public FilterScope scope;

@Column(name = "owner_id", length = 255)
public String ownerId;

// Add (scope only — ownerId is gone):
@Convert(converter = PathAttributeConverter.class)
@Column(nullable = false, length = 500)
public Path scope;
```

- [ ] **Step 3: Update `QueueView.java`**

Same changes as WorkItemFilter — replace scope field, drop ownerId, fix imports. The field comment must also be updated:

```java
// Remove these imports:
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;

// Add these imports:
import jakarta.persistence.Convert;
import io.casehub.platform.api.path.Path;
import io.casehub.work.runtime.model.PathAttributeConverter;
```

```java
// Remove:
/** Visibility scope of this queue view. */
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 20)
public FilterScope scope;

/** Owner identity for PERSONAL-scoped views; null for TEAM/ORG. */
@Column(name = "owner_id", length = 255)
public String ownerId;

// Add (scope only):
/** Visibility scope of this queue view — Path string, empty = root/widest. */
@Convert(converter = PathAttributeConverter.class)
@Column(nullable = false, length = 500)
public Path scope;
```

Also update the Javadoc on `CreateQueueRequest` in `QueueResource` (done in Task 5) — the `@param ownerId` line will be removed there.

- [ ] **Step 4: Update `WorkItemFilterBean.java`**

Delete the `scope()` method and its import entirely:

```java
// Remove:
import io.casehub.work.queues.model.FilterScope;

// Remove the method:
FilterScope scope();
```

The interface becomes:
```java
package io.casehub.work.queues.service;

import java.util.List;

import io.casehub.work.queues.model.FilterAction;
import io.casehub.work.runtime.model.WorkItem;

public interface WorkItemFilterBean {
    boolean matches(WorkItem workItem);

    List<FilterAction> actions();
}
```

- [ ] **Step 5: Verify compile — tests pass, other callers now broken**

```bash
scripts/mvn-compile casehub-work-queues
```

Expected: compile errors in `FilterResource` and `QueueResource` only — the test files now compile correctly. Tasks 4 and 5 fix the remaining errors.

---

## Task 4: Update `FilterResource`

**Files:**
- Modify: `queues/src/main/java/io/casehub/work/queues/api/FilterResource.java`

Three changes: request record drops `FilterScope scope, String ownerId` → `String scope`; `create()` parses scope with try/catch; `list()` calls `.value()`; `update()` explicitly ignores the scope field.

- [ ] **Step 1: Update imports**

```java
// Remove:
import io.casehub.work.queues.model.FilterScope;

// Add:
import io.casehub.platform.api.path.Path;
```

- [ ] **Step 2: Update `CreateFilterRequest` record**

```java
// Remove:
public record CreateFilterRequest(String name, FilterScope scope, String ownerId,
        String conditionLanguage, String conditionExpression, List<FilterAction> actions) {
}

// Add:
public record CreateFilterRequest(String name, String scope,
        String conditionLanguage, String conditionExpression, List<FilterAction> actions) {
}
```

- [ ] **Step 3: Update `list()` — serialize scope as string**

```java
// In list(), change:
"scope", f.scope,

// To:
"scope", f.scope.value(),
```

- [ ] **Step 4: Update `create()` — parse scope with error handling**

Replace the scope assignment in `create()`:

```java
// Remove:
f.scope = req.scope() != null ? req.scope() : FilterScope.ORG;
f.ownerId = req.ownerId();

// Add:
final Path scopePath;
try {
    scopePath = (req.scope() == null || req.scope().isBlank())
            ? Path.root()
            : Path.parse(req.scope());
} catch (IllegalArgumentException e) {
    return Response.status(400)
            .entity(Map.of("error", "invalid scope: " + e.getMessage())).build();
}
f.scope = scopePath;
```

- [ ] **Step 5: Update `update()` — scope is immutable, discard silently**

The `update()` method currently acts on `name`, `conditionExpression`, and `actions`. After the record change, `req.scope()` is available but must be ignored. No code change is needed beyond ensuring the method does not reference `req.scope()`. Verify the existing update body has no `scope` assignment — it does not — so no change is required here beyond removing any stale reference to `req.ownerId()` if it was assigned (it was not in the update path).

- [ ] **Step 6: Compile check**

```bash
scripts/mvn-compile casehub-work-queues
```

Expected: `FilterResource` compiles. `QueueResource` still fails — fixed in Task 5.

---

## Task 5: Update `QueueResource`

**Files:**
- Modify: `queues/src/main/java/io/casehub/work/queues/api/QueueResource.java`

Same pattern as FilterResource: request record, create, list.

- [ ] **Step 1: Update imports**

```java
// Remove:
import io.casehub.work.queues.model.FilterScope;

// Add:
import io.casehub.platform.api.path.Path;
```

- [ ] **Step 2: Update `CreateQueueRequest` record and its Javadoc**

```java
// Remove:
/**
 * Request body for creating a new queue view.
 *
 * @param name display name
 * @param labelPattern label pattern used to populate the queue
 * @param scope visibility scope
 * @param ownerId owner identity for PERSONAL-scoped views
 * @param additionalConditions optional extra filter conditions
 * @param sortField field to sort results by
 * @param sortDirection sort direction (ASC or DESC)
 */
public record CreateQueueRequest(String name, String labelPattern, FilterScope scope,
        String ownerId, String additionalConditions, String sortField, String sortDirection) {
}

// Add:
/**
 * Request body for creating a new queue view.
 *
 * @param name display name
 * @param labelPattern label pattern used to populate the queue
 * @param scope visibility scope as a Path string (null/blank = root)
 * @param additionalConditions optional extra filter conditions
 * @param sortField field to sort results by
 * @param sortDirection sort direction (ASC or DESC)
 */
public record CreateQueueRequest(String name, String labelPattern, String scope,
        String additionalConditions, String sortField, String sortDirection) {
}
```

- [ ] **Step 3: Update `list()` — serialize scope as string**

```java
// In list(), change:
"scope", q.scope,

// To:
"scope", q.scope.value(),
```

- [ ] **Step 4: Update `create()` — parse scope with error handling**

Replace the scope and ownerId assignments:

```java
// Remove:
q.scope = req.scope() != null ? req.scope() : FilterScope.ORG;
q.ownerId = req.ownerId();

// Add:
final Path scopePath;
try {
    scopePath = (req.scope() == null || req.scope().isBlank())
            ? Path.root()
            : Path.parse(req.scope());
} catch (IllegalArgumentException e) {
    return Response.status(400)
            .entity(Map.of("error", "invalid scope: " + e.getMessage())).build();
}
q.scope = scopePath;
```

- [ ] **Step 5: Compile and test the queues module**

```bash
scripts/mvn-compile casehub-work-queues
```

Expected: **PASS** — all queues module files compile.

```bash
scripts/mvn-test casehub-work-queues
```

Expected: **PASS** — all three tenancy tests pass with `Path.root()`.

- [ ] **Step 6: Commit queues module changes**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  queues/src/main/java/io/casehub/work/queues/model/WorkItemFilter.java \
  queues/src/main/java/io/casehub/work/queues/model/QueueView.java \
  queues/src/main/java/io/casehub/work/queues/service/WorkItemFilterBean.java \
  queues/src/main/java/io/casehub/work/queues/api/FilterResource.java \
  queues/src/main/java/io/casehub/work/queues/api/QueueResource.java \
  queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaWorkItemFilterStoreTenancyTest.java \
  queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaQueueViewStoreTenancyTest.java \
  queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaFilterChainStoreTenancyTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#263): FilterScope → Path in entities, SPI, and REST resources

- WorkItemFilter.scope: FilterScope → Path + @Convert(PathAttributeConverter)
- QueueView.scope: same
- WorkItemFilter.ownerId, QueueView.ownerId: dropped (path encodes ownership)
- FilterScope enum: deleted
- WorkItemFilterBean.scope(): deleted (dead SPI method, engine never called it)
- FilterResource, QueueResource: String scope param, Path.parse() with 400 on error,
  .value() in list responses, scope immutable in PUT
- Tests updated to Path.root()

Refs #263"
```

- [ ] **Step 7: Install queues module so dependents resolve it**

```bash
scripts/mvn-install casehub-work-queues
```

Expected: **PASS** — artifact published to local Maven repo.

---

## Task 6: Fix `queues-dashboard` Module

**Files:**
- Modify: `queues-dashboard/src/main/java/io/casehub/work/dashboard/SecurityWritersFilter.java`
- Modify: `queues-dashboard/src/main/java/io/casehub/work/dashboard/ReviewStepService.java`

- [ ] **Step 1: Update `queues-dashboard/SecurityWritersFilter.java`**

Remove the `scope()` override and its import. The file becomes:

```java
package io.casehub.work.dashboard;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.work.queues.model.FilterAction;
import io.casehub.work.queues.service.WorkItemFilterBean;
import io.casehub.work.runtime.model.WorkItem;

/**
 * Lambda filter: any document submitted by the security writing team is urgent,
 * regardless of the priority field on the WorkItem.
 *
 * <p>
 * This demonstrates the Lambda CDI filter pattern — business rules that require
 * Java logic rather than an expression string. A document flagged as security-related
 * ({@code candidateGroups} contains {@code "security-writers"}) must always enter the
 * urgent review queue to meet compliance publication timelines, even if submitted with
 * {@code MEDIUM} priority by mistake.
 *
 * <p>
 * Lambda filters are always active while deployed. They are discovered automatically
 * by {@code LambdaFilterRegistry} via CDI {@code Instance<WorkItemFilterBean>}.
 */
@ApplicationScoped
public class SecurityWritersFilter implements WorkItemFilterBean {

    @Override
    public boolean matches(final WorkItem workItem) {
        return workItem.candidateGroups != null
                && workItem.candidateGroups.contains("security-writers")
                && workItem.status != null
                && !workItem.status.isTerminal();
    }

    @Override
    public List<FilterAction> actions() {
        return List.of(FilterAction.applyLabel("review/urgent"));
    }
}
```

- [ ] **Step 2: Update `ReviewStepService.java` — replace `FilterScope.ORG` in `persist()`**

In the `persist()` helper method (around line 314):

```java
// Remove:
import io.casehub.work.queues.model.FilterScope;

// Add:
import io.casehub.platform.api.path.Path;
```

```java
// In persist():
// Remove:
f.scope = FilterScope.ORG;

// Add:
f.scope = Path.root();
```

- [ ] **Step 3: Compile check**

```bash
scripts/mvn-compile queues-dashboard
```

Expected: **PASS**.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  queues-dashboard/src/main/java/io/casehub/work/dashboard/SecurityWritersFilter.java \
  queues-dashboard/src/main/java/io/casehub/work/dashboard/ReviewStepService.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#263): fix queues-dashboard — remove scope() override, Path.root() in ReviewStepService

Refs #263"
```

---

## Task 7: Fix `queues-examples` Module

**Files:**
- Modify: `queues-examples/src/main/java/io/casehub/work/examples/queues/review/SecurityWritersFilter.java`
- Modify: `queues-examples/src/main/java/io/casehub/work/examples/queues/security/SecurityEscalationScenario.java`
- Modify: `queues-examples/src/main/java/io/casehub/work/examples/queues/triage/SupportTriageScenario.java`
- Modify: `queues-examples/src/main/java/io/casehub/work/examples/queues/finance/FinanceApprovalScenario.java`
- Modify: `queues-examples/src/main/java/io/casehub/work/examples/queues/legal/LegalRoutingScenario.java`
- Modify: `queues-examples/src/main/java/io/casehub/work/examples/queues/lifecycle/QueueLifecycleScenario.java`
- Modify: `queues-examples/src/main/java/io/casehub/work/examples/queues/review/DocumentReviewScenario.java`

In every file the pattern is the same: remove `import io.casehub.work.queues.model.FilterScope;`, add `import io.casehub.platform.api.path.Path;`, and replace every `FilterScope.ORG` with `Path.root()`.

- [ ] **Step 1: Update `queues-examples/review/SecurityWritersFilter.java`**

Remove the `scope()` override and its import — identical change to Task 6 Step 1 (same file, different package):

```java
package io.casehub.work.examples.queues.review;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.work.queues.model.FilterAction;
import io.casehub.work.queues.service.WorkItemFilterBean;
import io.casehub.work.runtime.model.WorkItem;

/**
 * Lambda filter: any document submitted by the security writing team is urgent,
 * regardless of the priority field on the WorkItem.
 *
 * <p>
 * This demonstrates the Lambda CDI filter pattern — business rules that require
 * Java logic rather than an expression string. A document flagged as security-related
 * ({@code candidateGroups} contains {@code "security-writers"}) must always enter the
 * urgent review queue to meet compliance publication timelines, even if submitted with
 * {@code MEDIUM} priority by mistake.
 *
 * <p>
 * Lambda filters are always active while deployed. They are discovered automatically
 * by {@code LambdaFilterRegistry} via CDI {@code Instance<WorkItemFilterBean>}.
 */
@ApplicationScoped
public class SecurityWritersFilter implements WorkItemFilterBean {

    @Override
    public boolean matches(final WorkItem workItem) {
        return workItem.candidateGroups != null
                && workItem.candidateGroups.contains("security-writers")
                && workItem.status != null
                && !workItem.status.isTerminal();
    }

    @Override
    public List<FilterAction> actions() {
        return List.of(FilterAction.applyLabel("review/urgent"));
    }
}
```

- [ ] **Step 2: Update the six scenario files**

In each file listed below, apply the same two-step change:
1. Replace `import io.casehub.work.queues.model.FilterScope;` with `import io.casehub.platform.api.path.Path;`
2. Replace every occurrence of `FilterScope.ORG` with `Path.root()`

Files and occurrence counts (all are `FilterScope.ORG` — none use other enum values):

| File | Occurrences |
|------|-------------|
| `security/SecurityEscalationScenario.java` | 6 |
| `triage/SupportTriageScenario.java` | 7 |
| `finance/FinanceApprovalScenario.java` | 4 |
| `legal/LegalRoutingScenario.java` | 4 |
| `lifecycle/QueueLifecycleScenario.java` | 1 |
| `review/DocumentReviewScenario.java` | 4 |

Example — in `SupportTriageScenario.java`, the import block changes from:
```java
import io.casehub.work.queues.model.FilterScope;
```
to:
```java
import io.casehub.platform.api.path.Path;
```

And each assignment changes from:
```java
filterA.scope = FilterScope.ORG;
```
to:
```java
filterA.scope = Path.root();
```

Apply identically to all six files.

- [ ] **Step 3: Compile check**

```bash
scripts/mvn-compile queues-examples
```

Expected: **PASS**.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add queues-examples/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#263): fix queues-examples — FilterScope.ORG → Path.root(), remove scope() overrides

Refs #263"
```

---

## Task 8: Fix `examples` Module

**Files:**
- Modify: `examples/src/main/java/io/casehub/work/examples/queues/QueueModuleScenario.java`

This file uses `FilterScope.TEAM` (not ORG) and also sets `queueView.ownerId = null` — both must change.

- [ ] **Step 1: Update imports**

```java
// Remove:
import io.casehub.work.queues.model.FilterScope;

// Add:
import io.casehub.platform.api.path.Path;
```

- [ ] **Step 2: Update the queue view creation block (around line 160)**

```java
// Remove:
queueView.scope = FilterScope.TEAM;
queueView.ownerId = null;

// Add:
queueView.scope = Path.of("team");
```

- [ ] **Step 3: Compile check**

```bash
scripts/mvn-compile examples
```

Expected: **PASS**.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add examples/src/main/java/io/casehub/work/examples/queues/QueueModuleScenario.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#263): fix examples — FilterScope.TEAM → Path.of(\"team\"), drop ownerId

Refs #263"
```

---

## Task 9: Full Verification

- [ ] **Step 1: Run queues tests**

```bash
scripts/mvn-test casehub-work-queues
```

Expected: **PASS** — all tenancy tests green.

- [ ] **Step 2: Compile all affected modules**

```bash
scripts/mvn-compile queues-dashboard && scripts/mvn-compile queues-examples && scripts/mvn-compile examples
```

Expected: **PASS** for all three.

- [ ] **Step 3: Check for any remaining FilterScope references**

```bash
scripts/mvn-compile casehub-work-queues
```

If any `FilterScope` reference remains, IntelliJ will catch it — use `ide_find_symbol` to search for `FilterScope` and fix any remaining references.

- [ ] **Step 4: Push project branch**

```bash
git -C /Users/mdproctor/claude/casehub/work push --set-upstream origin issue-263-filterscope-to-path
```
