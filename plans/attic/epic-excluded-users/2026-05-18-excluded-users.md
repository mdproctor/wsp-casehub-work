# Excluded Users Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `excludedUsers` conflict-of-interest enforcement to WorkItem and WorkItemTemplate via an `ExclusionPolicy` SPI, enforcing exclusion at all five assignment paths: auto-assignment candidate filtering, manual claim, direct assignee at creation, delegation, and `SelectionContext` for external strategies.

**Architecture:** `ExclusionPolicy` SPI lives in `casehub-work-api` (Tier 1, pure Java). `CommaSeparatedExclusionPolicy` is the `@DefaultBean` in runtime. `excludedUsers` is a TEXT comma-separated field on both `WorkItemTemplate` (declared at template level) and `WorkItem` (snapshotted at instantiation). `SelectionContext` gains `excludedUsers` so external `WorkerSelectionStrategy` implementations can apply exclusion logic. All five enforcement points use the injected `ExclusionPolicy`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM/Panache, Flyway, REST Assured + JUnit 5.

---

## Cross-Repo Note

`SelectionContext` is also constructed in `casehub-engine` (`WorkOrchestrator.java`, `CaseContextChangedEventHandler.java`). Adding the `excludedUsers` field there is tracked in issue #187 and is NOT in scope for this plan. Engine compilation of those files will break when `SelectionContext` gains the 8th field — address in a separate session.

---

## Files to Create

| File | Purpose |
|------|---------|
| `api/src/main/java/io/casehub/work/api/ExclusionPolicy.java` | SPI interface |
| `runtime/src/main/java/io/casehub/work/runtime/service/CommaSeparatedExclusionPolicy.java` | `@DefaultBean` implementation |
| `runtime/src/main/resources/db/migration/V23__workitem_template_excluded_users.sql` | Add `excluded_users` to `work_item_template` |
| `runtime/src/main/resources/db/migration/V24__workitem_excluded_users.sql` | Add `excluded_users` to `work_item` |
| `api/src/test/java/io/casehub/work/api/ExclusionPolicyTest.java` | SPI contract test |
| `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemExcludedUsersTest.java` | Integration tests (all 5 enforcement points) |

## Files to Modify

| File | Change |
|------|--------|
| `api/src/main/java/io/casehub/work/api/SelectionContext.java` | Add `excludedUsers` (8th field) |
| `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemTemplate.java` | Add `excludedUsers` TEXT field |
| `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java` | Add `excludedUsers` TEXT field |
| `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemCreateRequest.java` | Add `excludedUsers` String (22nd param, after `permittedOutcomes`) |
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java` | Inject `ExclusionPolicy`; enforce in `claim()`, `create()`, `delegate()` |
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java` | Pass `template.excludedUsers` in `toCreateRequest()` |
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java` | Inject `ExclusionPolicy`; filter in `resolveCandidates()`; add to `SelectionContext` |
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemSpawnService.java` | Pass `template.excludedUsers` in `WorkItemCreateRequest` |
| `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java` | Pass `template.excludedUsers` in both request builders; update `SelectionContext` in `RoundRobinAssignmentStrategy.java` |
| `runtime/src/main/java/io/casehub/work/runtime/multiinstance/RoundRobinAssignmentStrategy.java` | Add `excludedUsers` to `SelectionContext` construction |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResource.java` | Change `delegate()` return type to `Response`; add 400 handler; 400 handler for `create()` already exists |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemTemplateResource.java` | Add `excludedUsers` to `CreateTemplateRequest` + `toResponse()` |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResponse.java` | Add `excludedUsers` String |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemWithAuditResponse.java` | Add `excludedUsers` String |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemMapper.java` | Map `excludedUsers` in `toResponse()` and `toWithAudit()` |
| `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java` | Add `excludedUsers` to JEXL map |
| All test files constructing `WorkItemCreateRequest` | Append `, null` for new `excludedUsers` param |

---

## Task 1: Write failing tests — ExclusionPolicy SPI contract

**Files:**
- Create: `api/src/test/java/io/casehub/work/api/ExclusionPolicyTest.java`

- [ ] **Step 1.1: Write the test**

```java
package io.casehub.work.api;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

/**
 * Contract test for ExclusionPolicy SPI. Refs #171.
 * Using an anonymous implementation proves the interface contract exists independently of any concrete class.
 */
class ExclusionPolicyTest {

    // Anonymous implementation — compiler error here means isExcluded() is not yet default/abstract on the interface
    private final ExclusionPolicy policy = (userId, excludedUsers) -> {
        if (excludedUsers == null || excludedUsers.isBlank()) return false;
        for (final String id : excludedUsers.split(",")) {
            if (id.trim().equals(userId)) return true;
        }
        return false;
    };

    @Test
    void isExcluded_returnsTrue_whenUserIdInList() {
        assertThat(policy.isExcluded("alice", "alice,bob")).isTrue();
    }

    @Test
    void isExcluded_returnsFalse_whenUserIdNotInList() {
        assertThat(policy.isExcluded("carol", "alice,bob")).isFalse();
    }

    @Test
    void isExcluded_returnsFalse_whenExcludedUsersNull() {
        assertThat(policy.isExcluded("alice", null)).isFalse();
    }

    @Test
    void isExcluded_returnsFalse_whenExcludedUsersBlank() {
        assertThat(policy.isExcluded("alice", "")).isFalse();
    }

    @Test
    void isExcluded_trims_whitespace() {
        assertThat(policy.isExcluded("alice", " alice , bob ")).isTrue();
    }
}
```

- [ ] **Step 1.2: Verify RED**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl api 2>&1 | grep "ERROR" | head -5
```

Expected: `cannot find symbol — class ExclusionPolicy`

---

## Task 2: Write failing integration tests — all five enforcement points

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemExcludedUsersTest.java`

- [ ] **Step 2.1: Write the test**

```java
package io.casehub.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.notNullValue;
import static org.hamcrest.Matchers.nullValue;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

/**
 * Tests that excludedUsers is enforced at all five assignment paths. Refs #171.
 */
@QuarkusTest
class WorkItemExcludedUsersTest {

    // ── Template CRUD + snapshot ────────────────────────────────────────────

    @Test
    void createTemplate_withExcludedUsers_storesAndReturnsIt() {
        given().contentType(ContentType.JSON)
                .body("{\"name\":\"Approval\",\"candidateGroups\":\"reviewers\"," +
                      "\"excludedUsers\":\"alice,bob\",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then()
                .statusCode(201)
                .body("excludedUsers", equalTo("alice,bob"));
    }

    @Test
    void instantiateTemplate_withExcludedUsers_snapshotsOntoWorkItem() {
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"Exclusion Template\",\"candidateGroups\":\"reviewers\"," +
                      "\"excludedUsers\":\"alice\",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(201)
                .body("excludedUsers", equalTo("alice"));
    }

    @Test
    void instantiateTemplate_withoutExcludedUsers_workItemExcludedUsersIsNull() {
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"No Exclusion\",\"candidateGroups\":\"ops\",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(201)
                .body("excludedUsers", nullValue());
    }

    // ── Claim enforcement ──────────────────────────────────────────────────

    @Test
    void claim_byExcludedUser_returns409() {
        final String id = workItemWithExcludedUser("alice");
        given().put("/workitems/" + id + "/claim?claimant=alice")
                .then().statusCode(409);
    }

    @Test
    void claim_byNonExcludedUser_returns200() {
        final String id = workItemWithExcludedUser("alice");
        given().put("/workitems/" + id + "/claim?claimant=bob")
                .then().statusCode(200);
    }

    // ── Direct assigneeId at creation enforcement ─────────────────────────

    @Test
    void createWorkItem_withExcludedAssigneeId_returns400() {
        given().contentType(ContentType.JSON)
                .body("{\"title\":\"Conflict task\",\"candidateGroups\":\"ops\"," +
                      "\"assigneeId\":\"alice\",\"excludedUsers\":\"alice\",\"createdBy\":\"system\"}")
                .post("/workitems")
                .then()
                .statusCode(400);
    }

    @Test
    void createWorkItem_withNonExcludedAssigneeId_returns201() {
        given().contentType(ContentType.JSON)
                .body("{\"title\":\"OK task\",\"candidateGroups\":\"ops\"," +
                      "\"assigneeId\":\"bob\",\"excludedUsers\":\"alice\",\"createdBy\":\"system\"}")
                .post("/workitems")
                .then()
                .statusCode(201);
    }

    // ── Delegation enforcement ─────────────────────────────────────────────

    @Test
    void delegate_toExcludedUser_returns400() {
        final String id = workItemWithExcludedUser("alice");
        given().put("/workitems/" + id + "/claim?claimant=bob").then().statusCode(200);
        given().contentType(ContentType.JSON)
                .body("{\"to\":\"alice\"}")
                .put("/workitems/" + id + "/delegate?actor=bob")
                .then()
                .statusCode(400);
    }

    @Test
    void delegate_toNonExcludedUser_returns200() {
        final String id = workItemWithExcludedUser("alice");
        given().put("/workitems/" + id + "/claim?claimant=bob").then().statusCode(200);
        given().contentType(ContentType.JSON)
                .body("{\"to\":\"carol\"}")
                .put("/workitems/" + id + "/delegate?actor=bob")
                .then()
                .statusCode(200);
    }

    // ── No exclusion when excludedUsers is null/blank ─────────────────────

    @Test
    void claim_noExclusion_whenExcludedUsersNull_returns200() {
        final String workItemId = given().contentType(ContentType.JSON)
                .body("{\"title\":\"Open task\",\"candidateGroups\":\"ops\",\"createdBy\":\"system\"}")
                .post("/workitems")
                .then().statusCode(201).extract().path("id");
        given().put("/workitems/" + workItemId + "/claim?claimant=alice")
                .then().statusCode(200);
    }

    // ── Helper ─────────────────────────────────────────────────────────────

    private String workItemWithExcludedUser(final String excludedUser) {
        return given().contentType(ContentType.JSON)
                .body("{\"title\":\"Exclusion test\",\"candidateGroups\":\"ops\"," +
                      "\"excludedUsers\":\"" + excludedUser + "\",\"createdBy\":\"system\"}")
                .post("/workitems")
                .then().statusCode(201).extract().path("id");
    }
}
```

- [ ] **Step 2.2: Verify RED**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=WorkItemExcludedUsersTest 2>&1 | tail -10
```

Expected: tests fail (fields/validation don't exist yet). Failures should be on assertions, not compile errors.

---

## Task 3: ExclusionPolicy SPI + CommaSeparatedExclusionPolicy

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/ExclusionPolicy.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/CommaSeparatedExclusionPolicy.java`

- [ ] **Step 3.1: Create `ExclusionPolicy.java`**

```java
package io.casehub.work.api;

/**
 * SPI for evaluating whether a user is excluded from acting on a WorkItem.
 *
 * <p>
 * The default implementation ({@code CommaSeparatedExclusionPolicy}) checks
 * whether {@code userId} appears in a comma-separated {@code excludedUsers} string.
 * Custom implementations can plug in LDAP group membership, role-based rules,
 * or time-window logic (see casehubio/work#185).
 */
public interface ExclusionPolicy {

    /**
     * Returns {@code true} if {@code userId} is excluded.
     *
     * @param userId the identity to check; must not be null
     * @param excludedUsers comma-separated user IDs; null or blank means no exclusion
     * @return {@code true} if the user is excluded, {@code false} otherwise
     */
    boolean isExcluded(String userId, String excludedUsers);
}
```

- [ ] **Step 3.2: Create `CommaSeparatedExclusionPolicy.java`**

```java
package io.casehub.work.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.work.api.ExclusionPolicy;
import io.quarkus.arc.DefaultBean;

/**
 * Default {@link ExclusionPolicy} — checks whether {@code userId} appears in a
 * comma-separated {@code excludedUsers} string. Case-sensitive, whitespace-trimmed
 * per token. Activated via {@code @DefaultBean} so CDI {@code @Alternative}
 * implementations take precedence when present.
 */
@ApplicationScoped
@DefaultBean
public class CommaSeparatedExclusionPolicy implements ExclusionPolicy {

    @Override
    public boolean isExcluded(final String userId, final String excludedUsers) {
        if (excludedUsers == null || excludedUsers.isBlank()) {
            return false;
        }
        for (final String id : excludedUsers.split(",")) {
            if (id.trim().equals(userId)) {
                return true;
            }
        }
        return false;
    }
}
```

- [ ] **Step 3.3: Run api tests to confirm ExclusionPolicyTest passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api 2>&1 | tail -5
```

Expected: all tests pass (37 + 5 new = 42).

---

## Task 4: SelectionContext — add excludedUsers

**File:** `api/src/main/java/io/casehub/work/api/SelectionContext.java`

- [ ] **Step 4.1: Add `excludedUsers` as the 8th field**

Replace the record with:

```java
package io.casehub.work.api;

/**
 * Minimal WorkItem context passed to {@link WorkerSelectionStrategy#select}.
 *
 * <p>
 * Decouples strategies from the WorkItem JPA entity.
 *
 * @param category WorkItem category (may be null)
 * @param priority WorkItemPriority name e.g. "HIGH" (may be null)
 * @param requiredCapabilities comma-separated capability tags (may be null)
 * @param candidateGroups comma-separated group names (may be null)
 * @param candidateUsers comma-separated user IDs (may be null)
 * @param title work item title — used by semantic matchers (may be null)
 * @param description work item description — used by semantic matchers (may be null)
 * @param excludedUsers comma-separated user IDs excluded from this WorkItem (may be null)
 */
public record SelectionContext(
        String category,
        String priority,
        String requiredCapabilities,
        String candidateGroups,
        String candidateUsers,
        String title,
        String description,
        String excludedUsers) {
}
```

- [ ] **Step 4.2: Install api so dependents resolve it**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -pl api 2>&1 | tail -3
```

Expected: BUILD SUCCESS.

---

## Task 5: Flyway migrations

**Files:**
- Create: `runtime/src/main/resources/db/migration/V23__workitem_template_excluded_users.sql`
- Create: `runtime/src/main/resources/db/migration/V24__workitem_excluded_users.sql`

- [ ] **Step 5.1: Create V23**

```sql
-- Refs #171: excluded users for conflict-of-interest enforcement on WorkItemTemplate
ALTER TABLE work_item_template ADD COLUMN excluded_users TEXT;
```

- [ ] **Step 5.2: Create V24**

```sql
-- Refs #171: excluded users snapshotted onto WorkItem at instantiation
ALTER TABLE work_item ADD COLUMN excluded_users TEXT;
```

---

## Task 6: Data model fields

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemTemplate.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`

- [ ] **Step 6.1: Add to `WorkItemTemplate.java`** after the `outcomes` field:

```java
/**
 * Comma-separated user IDs excluded from claiming WorkItems instantiated from this template.
 * Consistent with {@link #candidateUsers}. Snapshotted onto
 * {@link WorkItem#excludedUsers} at instantiation time.
 * See casehubio/work#171.
 */
@Column(name = "excluded_users", columnDefinition = "TEXT")
public String excludedUsers;
```

- [ ] **Step 6.2: Add to `WorkItem.java`** after the `outcome` field (within the Named outcomes section):

```java
/**
 * Comma-separated user IDs excluded from claiming or being delegated this WorkItem.
 * Snapshotted from {@link WorkItemTemplate#excludedUsers} at instantiation.
 * Null means no user is excluded. Validated by {@link ExclusionPolicy} at
 * {@code claim()}, {@code create()} (for assigneeId), and {@code delegate()}.
 */
@Column(name = "excluded_users", columnDefinition = "TEXT")
public String excludedUsers;
```

---

## Task 7: WorkItemCreateRequest — add excludedUsers (22nd param)

**File:** `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemCreateRequest.java`

- [ ] **Step 7.1: Add `excludedUsers` after `permittedOutcomes`**

The record currently ends with:
```java
UUID templateId,
List<String> permittedOutcomes)
```

Change to:
```java
UUID templateId,
List<String> permittedOutcomes,
/** Comma-separated user IDs excluded from claiming this WorkItem; null = no exclusion. */
String excludedUsers)
```

Also add `@param excludedUsers Comma-separated user IDs excluded from claiming this WorkItem; null means no exclusion.` to the Javadoc block.

- [ ] **Step 7.2: Fix all production call sites**

Run:
```bash
grep -rn "new WorkItemCreateRequest(" /Users/mdproctor/claude/casehub/work/runtime/src/main/java --include="*.java"
```

For each call site NOT in `WorkItemTemplateService.toCreateRequest()` or spawn services, append `, null` before the closing `)`:
- `WorkItemMapper.toServiceRequest()` — append `, null`
- `WorkItemService.cloneWorkItem()` — append `, null`

- [ ] **Step 7.3: Fix all test call sites**

```bash
grep -rn "new WorkItemCreateRequest(" /Users/mdproctor/claude/casehub/work/runtime/src/test --include="*.java" | grep -v "WorkItemExcludedUsersTest"
```

Use the Python paren-counting script:

```python
import re, os

def fix_create_request(content):
    result = []
    i = 0
    while i < len(content):
        idx = content.find('new WorkItemCreateRequest(', i)
        if idx == -1:
            result.append(content[i:])
            break
        result.append(content[i:idx])
        start = idx + len('new WorkItemCreateRequest(')
        depth = 1
        j = start
        while j < len(content) and depth > 0:
            c = content[j]
            if c == '(': depth += 1
            elif c == ')': depth -= 1
            j += 1
        inner = content[start:j-1]
        result.append('new WorkItemCreateRequest(' + inner + ', null)')
        i = j
    return ''.join(result)

base = '/Users/mdproctor/claude/casehub/work'
test_dirs = [
    'runtime/src/test',
    'postgres-broadcaster/src/test',
    'ledger/src/test',
    'flow/src/test',
    'ai/src/test',
    'examples/src',
    'queues-examples/src',
    'queues-dashboard/src',
]
for d in test_dirs:
    path = os.path.join(base, d)
    if not os.path.exists(path): continue
    for root, dirs, files in os.walk(path):
        for f in files:
            if not f.endswith('.java'): continue
            fp = os.path.join(root, f)
            with open(fp) as fh: content = fh.read()
            updated = fix_create_request(content)
            if updated != content:
                with open(fp, 'w') as fh: fh.write(updated)
                print(f'Fixed: {fp}')
```

Also fix production files in `flow/src/main` and `examples/src/main`:
```bash
# Apply the same script to main source dirs
```

- [ ] **Step 7.4: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | grep "ERROR" | head -10
```

Expected: errors related to SelectionContext (8-field constructor) in RoundRobinAssignmentStrategy and WorkItemAssignmentService. Fix those in the next task.

---

## Task 8: WorkItemAssignmentService + RoundRobinAssignmentStrategy

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/RoundRobinAssignmentStrategy.java`

- [ ] **Step 8.1: Inject `ExclusionPolicy` into `WorkItemAssignmentService`**

Add to the CDI constructor parameters and field:
```java
private final ExclusionPolicy exclusionPolicy;
```

CDI constructor — add `ExclusionPolicy exclusionPolicy` as a parameter. Assign `this.exclusionPolicy = exclusionPolicy;`.

Unit-test constructor — add `ExclusionPolicy exclusionPolicy` parameter. Assign similarly.

Add import: `import io.casehub.work.api.ExclusionPolicy;`

- [ ] **Step 8.2: Filter excluded users in `resolveCandidates()`**

After building the full `candidates` list (after both `candidateUsers` and `candidateGroups` blocks), add:

```java
// Filter excluded users — ExclusionPolicy enforces conflict-of-interest constraints
if (workItem.excludedUsers != null) {
    candidates.removeIf(c -> exclusionPolicy.isExcluded(c.id(), workItem.excludedUsers));
}
```

- [ ] **Step 8.3: Pass `excludedUsers` in `SelectionContext` construction in `assign()`**

Change the `SelectionContext` constructor call from 7 to 8 args:
```java
final SelectionContext context = new SelectionContext(
        workItem.category,
        workItem.priority != null ? workItem.priority.name() : null,
        workItem.requiredCapabilities,
        workItem.candidateGroups,
        workItem.candidateUsers,
        workItem.title,
        workItem.description,
        workItem.excludedUsers);   // ← new 8th arg
```

- [ ] **Step 8.4: Fix `RoundRobinAssignmentStrategy.java` SelectionContext construction**

Find the 7-arg `SelectionContext` constructor call (line ~84). Add `child.excludedUsers` as the 8th argument:

```java
final SelectionContext selCtx = new SelectionContext(
        child.category,
        child.priority != null ? child.priority.name() : null,
        child.requiredCapabilities,
        parent.candidateGroups,
        filteredUsers,
        child.title,
        child.description,
        child.excludedUsers);   // ← new 8th arg
```

---

## Task 9: WorkItemTemplateService + spawn services

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemSpawnService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java`

- [ ] **Step 9.1: `WorkItemTemplateService.toCreateRequest()` — pass `template.excludedUsers`**

The 6-arg `toCreateRequest()` return currently ends with:
```java
                template.id,
                parseOutcomeNames(template.outcomes));
```

Change to:
```java
                template.id,
                parseOutcomeNames(template.outcomes),
                template.excludedUsers);
```

- [ ] **Step 9.2: `WorkItemSpawnService` — pass `template.excludedUsers`**

Find the `new WorkItemCreateRequest(` call. It ends with `WorkItemTemplateService.parseOutcomeNames(template.outcomes))`. Change to:
```java
                template.id,
                WorkItemTemplateService.parseOutcomeNames(template.outcomes),
                template.excludedUsers);
```

- [ ] **Step 9.3: `MultiInstanceSpawnService.buildParentRequest()` — pass `template.excludedUsers`**

Same change as Step 9.2 for `buildParentRequest()`.

- [ ] **Step 9.4: `MultiInstanceSpawnService.buildChildRequest()` — pass `template.excludedUsers`**

Same change for `buildChildRequest()`.

---

## Task 10: WorkItemService — inject ExclusionPolicy and enforce

**File:** `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`

- [ ] **Step 10.1: Inject `ExclusionPolicy`**

Add field injection (consistent with `schemaValidator`):
```java
@Inject
ExclusionPolicy exclusionPolicy;
```

Add import: `import io.casehub.work.api.ExclusionPolicy;`

- [ ] **Step 10.2: Set `excludedUsers` in `create()`**

After `item.outputDataSchema = request.outputDataSchema();` (if on output-schema branch) or after `item.permittedOutcomes = ...` (on main), add:
```java
item.excludedUsers = request.excludedUsers();
```

- [ ] **Step 10.3: Enforce in `create()` — check `assigneeId`**

After setting all item fields (including `item.excludedUsers`) and before `assignmentService.assign()`:
```java
if (request.assigneeId() != null && exclusionPolicy.isExcluded(request.assigneeId(), item.excludedUsers)) {
    throw new IllegalArgumentException(
            "assigneeId '" + request.assigneeId() + "' is excluded from this WorkItem");
}
```

- [ ] **Step 10.4: Enforce in `claim()`**

After the multi-instance guard block and before the status check (`if (item.status != WorkItemStatus.PENDING)`), add:
```java
if (exclusionPolicy.isExcluded(claimantId, item.excludedUsers)) {
    throw new IllegalStateException(
            "Claimant '" + claimantId + "' is excluded from this WorkItem");
}
```

- [ ] **Step 10.5: Enforce in `delegate()`**

Find `delegate()`. It currently resolves the toAssigneeId. After `final WorkItem item = requireWorkItem(id)` and before delegating, add:
```java
if (exclusionPolicy.isExcluded(toAssigneeId, item.excludedUsers)) {
    throw new IllegalArgumentException(
            "Cannot delegate to excluded user: '" + toAssigneeId + "'");
}
```

---

## Task 11: REST layer — WorkItemResource, WorkItemTemplateResource, response types, mapper, context

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResource.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemTemplateResource.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResponse.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemWithAuditResponse.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemMapper.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java`

- [ ] **Step 11.1: `WorkItemResource.delegate()` — change return type to `Response`**

Change signature from `public WorkItemResponse delegate(...)` to `public Response delegate(...)`.

Wrap the body:
```java
try {
    return Response.ok(WorkItemMapper.toResponse(
            workItemService.delegate(id, actor, body.to()))).build();
} catch (final IllegalArgumentException e) {
    return Response.status(Response.Status.BAD_REQUEST)
            .entity(Map.of("error", e.getMessage()))
            .build();
}
```

Add import: `import jakarta.ws.rs.core.Response;` if not already present.

- [ ] **Step 11.2: `WorkItemTemplateResource.CreateTemplateRequest` — add `excludedUsers`**

Add `String excludedUsers` after `outcomes` and before `createdBy`:
```java
/** Comma-separated user IDs excluded from claiming instances; null = no exclusion. */
String excludedUsers,
String createdBy)
```

In `createTemplate()`, after `t.outcomes = ...`, add:
```java
t.excludedUsers = request.excludedUsers();
```

In `toResponse()`, after `m.put("outcomes", ...)`, add:
```java
m.put("excludedUsers", t.excludedUsers);
```

- [ ] **Step 11.3: `WorkItemResponse` — add `excludedUsers`**

Add after `permittedOutcomes)` (or at the end if #170 not merged):
```java
/** Comma-separated user IDs excluded from this WorkItem; null = unconstrained. */
String excludedUsers)
```

- [ ] **Step 11.4: `WorkItemWithAuditResponse` — add `excludedUsers`**

Add after the last existing field:
```java
String excludedUsers)
```

- [ ] **Step 11.5: `WorkItemMapper` — map `excludedUsers`**

In `toResponse()`, after `wi.permittedOutcomes` (or at end), add `wi.excludedUsers`.

In `toWithAudit()`, same position.

- [ ] **Step 11.6: `WorkItemContextBuilder.toMap()` — add `excludedUsers`**

After `map.put("outcome", workItem.outcome);`, add:
```java
map.put("excludedUsers", workItem.excludedUsers);
```

---

## Task 12: Verify GREEN

- [ ] **Step 12.1: Compile runtime**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | grep "ERROR" | head -10
```

Expected: clean.

- [ ] **Step 12.2: Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -8
```

Expected: all pass including new `WorkItemExcludedUsersTest` (10 tests).

- [ ] **Step 12.3: Run api tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api 2>&1 | tail -5
```

Expected: 42 tests, 0 failures.

- [ ] **Step 12.4: Broader compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile \
  -pl runtime,examples,ledger,flow,queues,notifications,reports,ai 2>&1 \
  | grep "ERROR" | grep -v "^\[ERROR\] $" | head -10
```

Expected: clean. If any failures, fix them (most likely `new WorkItemCreateRequest(` call sites with the wrong arg count — apply the Python script).

---

## Task 13: Code review, commit, doc sync, epic close

- [ ] **Step 13.1: Invoke `superpowers:requesting-code-review`** before committing. Fix any Important findings; batch Minor nits into one GitHub issue.

- [ ] **Step 13.2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add -A
git -C /Users/mdproctor/claude/casehub/work commit -m "feat: excluded users — conflict-of-interest enforcement (Closes #171)

ExclusionPolicy SPI (casehub-work-api) + CommaSeparatedExclusionPolicy (@DefaultBean).
Enforced at all 5 assignment paths: auto-assignment filter, claim(), create()
assigneeId, delegate(), and SelectionContext for external strategies.
excludedUsers TEXT field on WorkItemTemplate + WorkItem (V23+V24 migrations).

Note: casehub-engine SelectionContext call sites tracked in #187.

Refs #77
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 13.3: Invoke `implementation-doc-sync`**

- [ ] **Step 13.4: Write journal entry** via `java-update-design`.

- [ ] **Step 13.5: Epic close** via `/epic`.
