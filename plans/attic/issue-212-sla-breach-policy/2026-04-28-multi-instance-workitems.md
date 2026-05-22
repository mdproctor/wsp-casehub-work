# Multi-Instance WorkItems Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add M-of-N parallel WorkItem instance semantics — a template with `instanceCount=5, requiredCount=3` auto-spawns 5 children; the parent completes when 3 approve; the inbox shows the parent as a thread root with aggregate progress.

**Architecture:** `WorkItemTemplateService.instantiate()` detects `instanceCount` on the template and delegates to `MultiInstanceSpawnService` which creates parent + N children in one transaction. `MultiInstanceCoordinator` (`@ObservesAsync`) reacts to child terminal events, updates `WorkItemSpawnGroup` counters via OCC (`@Version`), and fires `WorkItemGroupLifecycleEvent`. The inbox always returns roots (`parentId IS NULL`) with aggregate stats via a recursive CTE native query.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM / Panache, H2 (tests), PostgreSQL (prod dialect test), JUnit 5, AssertJ, REST-assured.

**Issue:** All commits reference `#106` (`Refs #106`).

**Build commands:**
```bash
scripts/mvn-test quarkus-work-api     # API module tests
scripts/mvn-install quarkus-work-api  # publish before runtime
scripts/mvn-test runtime              # runtime module tests
```

---

## File Map

### New — `quarkus-work-api`
| File | Responsibility |
|---|---|
| `quarkus-work-api/src/main/java/io/quarkiverse/work/api/ParentRole.java` | Enum: COORDINATOR \| PARTICIPANT |
| `quarkus-work-api/src/main/java/io/quarkiverse/work/api/OnThresholdReached.java` | Enum: CANCEL \| LEAVE |
| `quarkus-work-api/src/main/java/io/quarkiverse/work/api/GroupStatus.java` | Enum: IN_PROGRESS \| COMPLETED \| REJECTED |
| `quarkus-work-api/src/main/java/io/quarkiverse/work/api/MultiInstanceConfig.java` | Value object: instanceCount, requiredCount, parentRole, assignmentStrategyName, onThresholdReached, allowSameAssignee, explicitAssignees |
| `quarkus-work-api/src/main/java/io/quarkiverse/work/api/InstanceAssignmentStrategy.java` | SPI interface |
| `quarkus-work-api/src/main/java/io/quarkiverse/work/api/MultiInstanceContext.java` | Context passed to InstanceAssignmentStrategy |
| `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkItemGroupLifecycleEvent.java` | CDI event: group aggregate state changes |
| `quarkus-work-api/src/test/java/io/quarkiverse/work/api/MultiInstanceConfigTest.java` | Validation logic unit tests |

### New — `runtime/src/main/resources/db/migration`
| File | Responsibility |
|---|---|
| `V20__add_parent_id_to_work_item.sql` | `parent_id` column + index on `work_item` |
| `V21__multi_instance_support.sql` | Multi-instance columns on `work_item_spawn_group` + `work_item_template` |

### New — `runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance`
| File | Responsibility |
|---|---|
| `PoolAssignmentStrategy.java` | Default: copies candidateGroups/Users to all children |
| `RoundRobinAssignmentStrategy.java` | Uses WorkerSelectionStrategy per child with exclusion |
| `ExplicitListAssignmentStrategy.java` | Maps each child to a specific assignee from MultiInstanceConfig.explicitAssignees |
| `CompositeInstanceAssignmentStrategy.java` | Applies multiple strategies in sequence |
| `MultiInstanceGroupPolicy.java` | @ApplicationScoped @Transactional — OCC counter update + threshold evaluation |
| `MultiInstanceCoordinator.java` | @ApplicationScoped @ObservesAsync — delegates to MultiInstanceGroupPolicy, retries on OCC |
| `MultiInstanceSpawnService.java` | @ApplicationScoped @Transactional — creates parent + spawn group + N children + applies strategy |

### New — `runtime/src/main/java/io/quarkiverse/work/runtime/api`
| File | Responsibility |
|---|---|
| `WorkItemInstancesResource.java` | GET /workitems/{id}/instances |

### Modified — `runtime`
| File | Change |
|---|---|
| `model/WorkItem.java` | Add `parentId UUID` field |
| `model/WorkItemSpawnGroup.java` | Add OCC fields + `findMultiInstanceByParentId()` |
| `model/WorkItemTemplate.java` | Add multi-instance fields |
| `model/WorkItemRootView.java` | New: record — WorkItem + aggregate stats for inbox |
| `service/WorkItemService.java` | Add claim guard; add `complete()` / `reject()` overload callable by coordinator |
| `service/WorkItemTemplateService.java` | Detect instanceCount and delegate to MultiInstanceSpawnService |
| `service/WorkItemTemplateValidationService.java` | New: static validation for MultiInstanceConfig; called from template create/update |
| `repository/WorkItemStore.java` | Add default `scanRoots()` method |
| `repository/jpa/JpaWorkItemStore.java` | Override `scanRoots()` with recursive CTE native query |
| `api/WorkItemResource.java` | Update `/inbox` to use `scanRoots()` returning `WorkItemRootView` |
| `api/WorkItemResponse.java` | Add nullable aggregate fields: childCount, completedCount, requiredCount, groupStatus |
| `event/WorkItemContextBuilder.java` | Add parentId to JEXL context map |

### New — `runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance`
| File | Covers |
|---|---|
| `InstanceAssignmentStrategyTest.java` | All four strategies — pure Java unit tests |
| `MultiInstanceConfigValidationTest.java` | All 422 validation rules — pure Java unit tests |
| `MultiInstanceCreateTest.java` | @QuarkusTest: template with instanceCount creates parent + N children |
| `MultiInstanceCoordinatorTest.java` | @QuarkusTest: coordinator happy path, failure, CANCEL/LEAVE, idempotency |
| `MultiInstanceClaimGuardTest.java` | @QuarkusTest: allowSameAssignee guard |
| `MultiInstanceInboxTest.java` | @QuarkusTest: threaded inbox roots, aggregate stats, ancestor visibility |
| `WorkItemGroupLifecycleEventTest.java` | @QuarkusTest: correct event firing per child terminal |

### New — `runtime/src/test/java/io/quarkiverse/work/runtime/api`
| File | Covers |
|---|---|
| `WorkItemInstancesResourceTest.java` | @QuarkusTest: GET /workitems/{id}/instances |

---

## Task 1 — API enums

**Files:**
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/ParentRole.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/OnThresholdReached.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/GroupStatus.java`

- [ ] **Create the three enums**

```java
// ParentRole.java
package io.quarkiverse.work.api;
public enum ParentRole { COORDINATOR, PARTICIPANT }

// OnThresholdReached.java
package io.quarkiverse.work.api;
public enum OnThresholdReached { CANCEL, LEAVE }

// GroupStatus.java
package io.quarkiverse.work.api;
public enum GroupStatus { IN_PROGRESS, COMPLETED, REJECTED }
```

- [ ] **Compile**

```bash
scripts/mvn-compile quarkus-work-api
```
Expected: BUILD SUCCESS

- [ ] **Commit**

```bash
git add quarkus-work-api/src/main/java/io/quarkiverse/work/api/ParentRole.java \
        quarkus-work-api/src/main/java/io/quarkiverse/work/api/OnThresholdReached.java \
        quarkus-work-api/src/main/java/io/quarkiverse/work/api/GroupStatus.java
git commit -m "feat: add ParentRole, OnThresholdReached, GroupStatus enums to quarkus-work-api

Refs #106"
```

---

## Task 2 — MultiInstanceConfig + InstanceAssignmentStrategy SPI

**Files:**
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/MultiInstanceConfig.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/MultiInstanceContext.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/InstanceAssignmentStrategy.java`
- Create: `quarkus-work-api/src/test/java/io/quarkiverse/work/api/MultiInstanceConfigTest.java`

- [ ] **Write the failing tests**

```java
// MultiInstanceConfigTest.java
package io.quarkiverse.work.api;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

class MultiInstanceConfigTest {

    @Test
    void defaultParentRoleIsCoordinator() {
        var config = new MultiInstanceConfig(5, 3, null, null, null, false, null);
        assertThat(config.effectiveParentRole()).isEqualTo(ParentRole.COORDINATOR);
    }

    @Test
    void defaultOnThresholdReachedIsCancel() {
        var config = new MultiInstanceConfig(5, 3, null, null, null, false, null);
        assertThat(config.effectiveOnThresholdReached()).isEqualTo(OnThresholdReached.CANCEL);
    }

    @Test
    void defaultAssignmentStrategyIsPool() {
        var config = new MultiInstanceConfig(5, 3, null, null, null, false, null);
        assertThat(config.effectiveAssignmentStrategyName()).isEqualTo("pool");
    }

    @Test
    void explicitAssigneesRequiredWhenStrategyIsExplicit() {
        var config = new MultiInstanceConfig(2, 1, null, "explicit", null, false, null);
        assertThat(config.validate()).contains("explicitAssignees required when strategy is 'explicit'");
    }

    @Test
    void explicitAssigneesSizeMustMatchInstanceCount() {
        var config = new MultiInstanceConfig(3, 2, null, "explicit", null, false, List.of("alice", "bob"));
        assertThat(config.validate()).contains("explicitAssignees size (2) must match instanceCount (3)");
    }

    @Test
    void validConfigReturnsNoErrors() {
        var config = new MultiInstanceConfig(5, 3, ParentRole.COORDINATOR, "pool", OnThresholdReached.CANCEL, false, null);
        assertThat(config.validate()).isEmpty();
    }
}
```

- [ ] **Run test to verify it fails**

```bash
scripts/mvn-test quarkus-work-api
```
Expected: FAIL — MultiInstanceConfig not found

- [ ] **Create MultiInstanceConfig**

```java
// MultiInstanceConfig.java
package io.quarkiverse.work.api;

import java.util.ArrayList;
import java.util.List;

public record MultiInstanceConfig(
        int instanceCount,
        int requiredCount,
        ParentRole parentRole,
        String assignmentStrategyName,
        OnThresholdReached onThresholdReached,
        boolean allowSameAssignee,
        List<String> explicitAssignees) {

    public ParentRole effectiveParentRole() {
        return parentRole != null ? parentRole : ParentRole.COORDINATOR;
    }

    public OnThresholdReached effectiveOnThresholdReached() {
        return onThresholdReached != null ? onThresholdReached : OnThresholdReached.CANCEL;
    }

    public String effectiveAssignmentStrategyName() {
        return assignmentStrategyName != null ? assignmentStrategyName : "pool";
    }

    /** Returns a list of validation error messages; empty = valid. */
    public List<String> validate() {
        final List<String> errors = new ArrayList<>();
        if (instanceCount < 1) {
            errors.add("instanceCount must be at least 1");
        }
        if (requiredCount < 1) {
            errors.add("requiredCount must be at least 1");
        }
        if (instanceCount >= 1 && requiredCount >= 1 && requiredCount > instanceCount) {
            errors.add("requiredCount (" + requiredCount + ") cannot exceed instanceCount (" + instanceCount + ")");
        }
        if ("explicit".equals(effectiveAssignmentStrategyName())) {
            if (explicitAssignees == null || explicitAssignees.isEmpty()) {
                errors.add("explicitAssignees required when strategy is 'explicit'");
            } else if (explicitAssignees.size() != instanceCount) {
                errors.add("explicitAssignees size (" + explicitAssignees.size()
                        + ") must match instanceCount (" + instanceCount + ")");
            }
        }
        return errors;
    }
}
```

- [ ] **Create MultiInstanceContext and InstanceAssignmentStrategy**

```java
// MultiInstanceContext.java
package io.quarkiverse.work.api;

public record MultiInstanceContext(Object parent, MultiInstanceConfig config) {}

// InstanceAssignmentStrategy.java
package io.quarkiverse.work.api;

import java.util.List;

public interface InstanceAssignmentStrategy {
    /**
     * Assign candidate users/groups to each instance.
     * Implementations mutate instance fields (candidateGroups, candidateUsers, assigneeId).
     *
     * @param instances ordered list of child WorkItems, not yet persisted by this call
     * @param context   parent WorkItem and resolved MultiInstanceConfig
     */
    void assign(List<Object> instances, MultiInstanceContext context);
}
```

- [ ] **Run tests to verify they pass**

```bash
scripts/mvn-test quarkus-work-api
```
Expected: PASS

- [ ] **Commit**

```bash
git add quarkus-work-api/src/main/java/io/quarkiverse/work/api/MultiInstanceConfig.java \
        quarkus-work-api/src/main/java/io/quarkiverse/work/api/MultiInstanceContext.java \
        quarkus-work-api/src/main/java/io/quarkiverse/work/api/InstanceAssignmentStrategy.java \
        quarkus-work-api/src/test/java/io/quarkiverse/work/api/MultiInstanceConfigTest.java
git commit -m "feat: add MultiInstanceConfig, InstanceAssignmentStrategy SPI, MultiInstanceContext to quarkus-work-api

Refs #106"
```

---

## Task 3 — WorkItemGroupLifecycleEvent

**Files:**
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkItemGroupLifecycleEvent.java`

- [ ] **Create the event class**

```java
package io.quarkiverse.work.api;

import java.time.Instant;
import java.util.UUID;

public final class WorkItemGroupLifecycleEvent {

    private final UUID parentId;
    private final UUID groupId;
    private final int instanceCount;
    private final int requiredCount;
    private final int completedCount;
    private final int rejectedCount;
    private final GroupStatus groupStatus;
    private final String callerRef;
    private final Instant occurredAt;

    private WorkItemGroupLifecycleEvent(final UUID parentId, final UUID groupId,
            final int instanceCount, final int requiredCount,
            final int completedCount, final int rejectedCount,
            final GroupStatus groupStatus, final String callerRef) {
        this.parentId = parentId;
        this.groupId = groupId;
        this.instanceCount = instanceCount;
        this.requiredCount = requiredCount;
        this.completedCount = completedCount;
        this.rejectedCount = rejectedCount;
        this.groupStatus = groupStatus;
        this.callerRef = callerRef;
        this.occurredAt = Instant.now();
    }

    public static WorkItemGroupLifecycleEvent of(final UUID parentId, final UUID groupId,
            final int instanceCount, final int requiredCount,
            final int completedCount, final int rejectedCount,
            final GroupStatus status, final String callerRef) {
        return new WorkItemGroupLifecycleEvent(parentId, groupId, instanceCount, requiredCount,
                completedCount, rejectedCount, status, callerRef);
    }

    public UUID parentId()       { return parentId; }
    public UUID groupId()        { return groupId; }
    public int instanceCount()   { return instanceCount; }
    public int requiredCount()   { return requiredCount; }
    public int completedCount()  { return completedCount; }
    public int rejectedCount()   { return rejectedCount; }
    public GroupStatus groupStatus() { return groupStatus; }
    public String callerRef()    { return callerRef; }
    public Instant occurredAt()  { return occurredAt; }
}
```

- [ ] **Compile and test**

```bash
scripts/mvn-test quarkus-work-api
```
Expected: PASS (all existing + new compile)

- [ ] **Install API module** (runtime depends on it)

```bash
scripts/mvn-install quarkus-work-api
```

- [ ] **Commit**

```bash
git add quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkItemGroupLifecycleEvent.java
git commit -m "feat: add WorkItemGroupLifecycleEvent to quarkus-work-api

Fires on every child terminal event (IN_PROGRESS) and on group
resolution (COMPLETED/REJECTED). Carries callerRef for CaseHub routing.

Refs #106"
```

---

## Task 4 — Flyway V20: parentId on work_item

**Files:**
- Create: `runtime/src/main/resources/db/migration/V20__add_parent_id_to_work_item.sql`

- [ ] **Write migration**

```sql
-- V20__add_parent_id_to_work_item.sql
ALTER TABLE work_item ADD COLUMN parent_id UUID REFERENCES work_item(id);
CREATE INDEX idx_work_item_parent_id ON work_item(parent_id);
```

- [ ] **Run runtime tests to verify migration applies cleanly**

```bash
scripts/mvn-test runtime -Dtest=WorkItemSmokeTest
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/resources/db/migration/V20__add_parent_id_to_work_item.sql
git commit -m "feat: V20 add parent_id to work_item for threaded inbox

Enables root detection (parentId IS NULL) and efficient child
queries without NOT EXISTS subqueries on work_item_relation.

Refs #106"
```

---

## Task 5 — Flyway V21: multi-instance columns on spawn group + template

**Files:**
- Create: `runtime/src/main/resources/db/migration/V21__multi_instance_support.sql`

- [ ] **Write migration**

```sql
-- V21__multi_instance_support.sql

-- WorkItemSpawnGroup: OCC + policy tracking
ALTER TABLE work_item_spawn_group
    ADD COLUMN instance_count       INTEGER,
    ADD COLUMN required_count       INTEGER,
    ADD COLUMN on_threshold_reached VARCHAR(10),
    ADD COLUMN allow_same_assignee  BOOLEAN     NOT NULL DEFAULT FALSE,
    ADD COLUMN parent_role          VARCHAR(15),
    ADD COLUMN completed_count      INTEGER     NOT NULL DEFAULT 0,
    ADD COLUMN rejected_count       INTEGER     NOT NULL DEFAULT 0,
    ADD COLUMN policy_triggered     BOOLEAN     NOT NULL DEFAULT FALSE,
    ADD COLUMN version              BIGINT      NOT NULL DEFAULT 0;

-- WorkItemTemplate: multi-instance config
ALTER TABLE work_item_template
    ADD COLUMN instance_count          INTEGER,
    ADD COLUMN required_count          INTEGER,
    ADD COLUMN parent_role             VARCHAR(15),
    ADD COLUMN assignment_strategy     VARCHAR(255),
    ADD COLUMN on_threshold_reached    VARCHAR(10),
    ADD COLUMN allow_same_assignee     BOOLEAN;
```

- [ ] **Run smoke test**

```bash
scripts/mvn-test runtime -Dtest=WorkItemSmokeTest
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/resources/db/migration/V21__multi_instance_support.sql
git commit -m "feat: V21 multi-instance columns on work_item_spawn_group and work_item_template

Adds OCC (@Version), counter columns, and policy config to spawn group.
Adds instanceCount/requiredCount/strategy config to template.

Refs #106"
```

---

## Task 6 — WorkItem entity: add parentId

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItem.java`
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/event/WorkItemContextBuilder.java`

- [ ] **Add parentId field to WorkItem** (after the `callerRef` field)

```java
/**
 * UUID of the parent WorkItem for multi-instance groups.
 * Null for standalone WorkItems. Non-null means this item is a child instance;
 * the parent is the group coordinator or participant root.
 */
@Column(name = "parent_id")
public UUID parentId;
```

- [ ] **Add parentId to WorkItemContextBuilder.toMap()**

Find the `toMap(WorkItem)` method in `WorkItemContextBuilder` and add:
```java
map.put("parentId", workItem.parentId != null ? workItem.parentId.toString() : null);
```

- [ ] **Run the context map drift-protection test** (it will fail if a field is missing)

```bash
scripts/mvn-test runtime -Dtest=JexlConditionEvaluatorTest
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItem.java \
        runtime/src/main/java/io/quarkiverse/work/runtime/event/WorkItemContextBuilder.java
git commit -m "feat: add parentId field to WorkItem entity and JEXL context map

parentId IS NULL identifies root WorkItems for threaded inbox.
parentId non-null identifies child instances in a multi-instance group.

Refs #106"
```

---

## Task 7 — WorkItemSpawnGroup entity: OCC fields + findMultiInstanceByParentId

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemSpawnGroup.java`

- [ ] **Add new fields and query method to WorkItemSpawnGroup**

Add these fields after the existing `createdAt` field:

```java
/** OCC version — incremented on every successful counter update. */
@Version
@Column(nullable = false)
public Long version = 0L;

/** instanceCount from MultiInstanceConfig; null for non-multi-instance groups. */
@Column(name = "instance_count")
public Integer instanceCount;

/** requiredCount from MultiInstanceConfig; null for non-multi-instance groups. */
@Column(name = "required_count")
public Integer requiredCount;

/** What to do with remaining non-terminal children when threshold is met. */
@Column(name = "on_threshold_reached", length = 10)
public String onThresholdReached;

/** When false, a claimant already holding another instance in this group is rejected. */
@Column(name = "allow_same_assignee", nullable = false)
public boolean allowSameAssignee = false;

/** COORDINATOR or PARTICIPANT. */
@Column(name = "parent_role", length = 15)
public String parentRole;

/** Number of child instances that have reached COMPLETED status. */
@Column(name = "completed_count", nullable = false)
public int completedCount = 0;

/** Number of child instances that have reached a non-COMPLETED terminal status. */
@Column(name = "rejected_count", nullable = false)
public int rejectedCount = 0;

/** True once the group outcome has been determined and the parent transitioned. */
@Column(name = "policy_triggered", nullable = false)
public boolean policyTriggered = false;
```

Add this static query method after the existing `findByParentId`:

```java
/**
 * Find the multi-instance spawn group for a parent — the group where
 * {@code requiredCount} is set. Returns null if no multi-instance group exists.
 */
public static WorkItemSpawnGroup findMultiInstanceByParentId(final UUID parentId) {
    return find("parentId = ?1 AND requiredCount IS NOT NULL", parentId).firstResult();
}
```

- [ ] **Compile**

```bash
scripts/mvn-compile runtime
```
Expected: BUILD SUCCESS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemSpawnGroup.java
git commit -m "feat: add OCC fields and findMultiInstanceByParentId to WorkItemSpawnGroup

Adds @Version for OCC, completedCount/rejectedCount/policyTriggered
for atomic M-of-N tracking, and policy config fields copied from template
at spawn time.

Refs #106"
```

---

## Task 8 — WorkItemTemplate entity: multi-instance fields

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemTemplate.java`

- [ ] **Add multi-instance fields to WorkItemTemplate** (after `labelPaths`)

```java
/**
 * Number of parallel instances to spawn when this template is instantiated.
 * Null means standard (non-multi-instance) instantiation.
 */
@Column(name = "instance_count")
public Integer instanceCount;

/**
 * Minimum number of instances that must reach COMPLETED for the group to succeed.
 * Required when instanceCount is set.
 */
@Column(name = "required_count")
public Integer requiredCount;

/** COORDINATOR (default) or PARTICIPANT. Only meaningful when instanceCount is set. */
@Column(name = "parent_role", length = 15)
public String parentRole;

/** CDI bean name of the InstanceAssignmentStrategy; null defaults to "pool". */
@Column(name = "assignment_strategy", length = 255)
public String assignmentStrategy;

/** CANCEL (default) or LEAVE — what to do with remaining instances when threshold met. */
@Column(name = "on_threshold_reached", length = 10)
public String onThresholdReached;

/**
 * When false (default), a person already holding one instance cannot claim another
 * in the same group. When true, no such restriction applies.
 */
@Column(name = "allow_same_assignee")
public Boolean allowSameAssignee;
```

- [ ] **Compile**

```bash
scripts/mvn-compile runtime
```
Expected: BUILD SUCCESS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemTemplate.java
git commit -m "feat: add multi-instance config fields to WorkItemTemplate

instanceCount/requiredCount/parentRole/assignmentStrategy/
onThresholdReached/allowSameAssignee copied to WorkItemSpawnGroup
at instantiation time.

Refs #106"
```

---

## Task 9 — InstanceAssignmentStrategy implementations + tests

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/PoolAssignmentStrategy.java`
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/ExplicitListAssignmentStrategy.java`
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/RoundRobinAssignmentStrategy.java`
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/CompositeInstanceAssignmentStrategy.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/InstanceAssignmentStrategyTest.java`

- [ ] **Write failing tests**

```java
// InstanceAssignmentStrategyTest.java
package io.quarkiverse.work.runtime.multiinstance;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.quarkiverse.work.api.MultiInstanceConfig;
import io.quarkiverse.work.api.MultiInstanceContext;
import io.quarkiverse.work.runtime.model.WorkItem;

class InstanceAssignmentStrategyTest {

    private static WorkItem item() {
        WorkItem w = new WorkItem();
        return w;
    }

    private static WorkItem parent(String candidateGroups, String candidateUsers) {
        WorkItem p = new WorkItem();
        p.candidateGroups = candidateGroups;
        p.candidateUsers = candidateUsers;
        return p;
    }

    // --- PoolAssignmentStrategy ---

    @Test
    void pool_copiesCandidateGroupsToAllInstances() {
        var strategy = new PoolAssignmentStrategy();
        var parent = parent("reviewers,approvers", null);
        var instances = List.of(item(), item(), item());
        strategy.assign((List) instances, new MultiInstanceContext(parent,
                new MultiInstanceConfig(3, 2, null, "pool", null, false, null)));

        assertThat(instances).allMatch(i -> "reviewers,approvers".equals(i.candidateGroups));
    }

    @Test
    void pool_copiesCandidateUsersToAllInstances() {
        var strategy = new PoolAssignmentStrategy();
        var parent = parent(null, "alice,bob,carol");
        var instances = List.of(item(), item());
        strategy.assign((List) instances, new MultiInstanceContext(parent,
                new MultiInstanceConfig(2, 1, null, "pool", null, false, null)));

        assertThat(instances).allMatch(i -> "alice,bob,carol".equals(i.candidateUsers));
    }

    // --- ExplicitListAssignmentStrategy ---

    @Test
    void explicit_assignsEachInstanceToCorrespondingAssignee() {
        var strategy = new ExplicitListAssignmentStrategy();
        var instances = List.of(item(), item(), item());
        strategy.assign((List) instances, new MultiInstanceContext(new WorkItem(),
                new MultiInstanceConfig(3, 2, null, "explicit", null, false,
                        List.of("alice", "bob", "carol"))));

        assertThat(instances.get(0).assigneeId).isEqualTo("alice");
        assertThat(instances.get(1).assigneeId).isEqualTo("bob");
        assertThat(instances.get(2).assigneeId).isEqualTo("carol");
    }

    @Test
    void explicit_throwsWhenListSizeMismatch() {
        var strategy = new ExplicitListAssignmentStrategy();
        var instances = List.of(item(), item());
        assertThatThrownBy(() -> strategy.assign((List) instances,
                new MultiInstanceContext(new WorkItem(),
                        new MultiInstanceConfig(2, 1, null, "explicit", null, false,
                                List.of("alice")))))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("explicitAssignees");
    }

    // --- CompositeInstanceAssignmentStrategy ---

    @Test
    void composite_appliesStrategiesInOrder() {
        // Pool sets candidateGroups; Explicit then overrides assigneeId
        var pool = new PoolAssignmentStrategy();
        var explicit = new ExplicitListAssignmentStrategy();
        var composite = new CompositeInstanceAssignmentStrategy(List.of(pool, explicit));

        var parent = parent("reviewers", null);
        var instances = List.of(item(), item());
        composite.assign((List) instances, new MultiInstanceContext(parent,
                new MultiInstanceConfig(2, 1, null, "composite", null, false,
                        List.of("alice", "bob"))));

        // Pool ran first: candidateGroups set; Explicit ran second: assigneeId set
        assertThat(instances).allMatch(i -> "reviewers".equals(i.candidateGroups));
        assertThat(instances.get(0).assigneeId).isEqualTo("alice");
        assertThat(instances.get(1).assigneeId).isEqualTo("bob");
    }
}
```

- [ ] **Run tests to verify they fail**

```bash
scripts/mvn-test runtime -Dtest=InstanceAssignmentStrategyTest
```
Expected: FAIL — classes not found

- [ ] **Implement PoolAssignmentStrategy**

```java
package io.quarkiverse.work.runtime.multiinstance;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Named;

import io.quarkiverse.work.api.InstanceAssignmentStrategy;
import io.quarkiverse.work.api.MultiInstanceContext;
import io.quarkiverse.work.runtime.model.WorkItem;

@ApplicationScoped
@Named("pool")
public class PoolAssignmentStrategy implements InstanceAssignmentStrategy {

    @Override
    public void assign(final List<Object> instances, final MultiInstanceContext context) {
        final WorkItem parent = (WorkItem) context.parent();
        for (final Object obj : instances) {
            final WorkItem child = (WorkItem) obj;
            child.candidateGroups = parent.candidateGroups;
            child.candidateUsers = parent.candidateUsers;
        }
    }
}
```

- [ ] **Implement ExplicitListAssignmentStrategy**

```java
package io.quarkiverse.work.runtime.multiinstance;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Named;

import io.quarkiverse.work.api.InstanceAssignmentStrategy;
import io.quarkiverse.work.api.MultiInstanceContext;
import io.quarkiverse.work.runtime.model.WorkItem;

@ApplicationScoped
@Named("explicit")
public class ExplicitListAssignmentStrategy implements InstanceAssignmentStrategy {

    @Override
    public void assign(final List<Object> instances, final MultiInstanceContext context) {
        final List<String> assignees = context.config().explicitAssignees();
        if (assignees == null || assignees.size() != instances.size()) {
            throw new IllegalArgumentException(
                    "explicitAssignees size must match instanceCount; got "
                            + (assignees == null ? "null" : assignees.size())
                            + ", expected " + instances.size());
        }
        for (int i = 0; i < instances.size(); i++) {
            ((WorkItem) instances.get(i)).assigneeId = assignees.get(i);
        }
    }
}
```

- [ ] **Implement RoundRobinAssignmentStrategy**

```java
package io.quarkiverse.work.runtime.multiinstance;

import java.util.HashSet;
import java.util.List;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.inject.Named;

import io.quarkiverse.work.api.AssignmentDecision;
import io.quarkiverse.work.api.AssignmentTrigger;
import io.quarkiverse.work.api.InstanceAssignmentStrategy;
import io.quarkiverse.work.api.MultiInstanceContext;
import io.quarkiverse.work.api.SelectionContext;
import io.quarkiverse.work.api.WorkerSelectionStrategy;
import io.quarkiverse.work.runtime.model.WorkItem;

@ApplicationScoped
@Named("roundRobin")
public class RoundRobinAssignmentStrategy implements InstanceAssignmentStrategy {

    @Inject
    WorkerSelectionStrategy workerSelectionStrategy;

    @Override
    public void assign(final List<Object> instances, final MultiInstanceContext context) {
        final WorkItem parent = (WorkItem) context.parent();
        final Set<String> alreadyAssigned = new HashSet<>();

        for (final Object obj : instances) {
            final WorkItem child = (WorkItem) obj;
            // Build candidate pool excluding already-assigned workers
            final String filteredGroups = parent.candidateGroups;
            final String filteredUsers = filterOut(parent.candidateUsers, alreadyAssigned);

            final SelectionContext selCtx = new SelectionContext(
                    child.id, child.title, child.description, child.category,
                    child.requiredCapabilities, filteredUsers, filteredGroups);

            final AssignmentDecision decision = workerSelectionStrategy.select(selCtx);
            if (decision != null && decision.assigneeId() != null) {
                child.assigneeId = decision.assigneeId();
                alreadyAssigned.add(decision.assigneeId());
            } else {
                // fallback: copy pool candidates
                child.candidateGroups = parent.candidateGroups;
                child.candidateUsers = parent.candidateUsers;
            }
        }
    }

    private String filterOut(final String candidateUsers, final Set<String> excluded) {
        if (candidateUsers == null || candidateUsers.isBlank()) return candidateUsers;
        return java.util.Arrays.stream(candidateUsers.split(","))
                .map(String::trim)
                .filter(u -> !excluded.contains(u))
                .collect(java.util.stream.Collectors.joining(","));
    }
}
```

- [ ] **Implement CompositeInstanceAssignmentStrategy**

```java
package io.quarkiverse.work.runtime.multiinstance;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Named;

import io.quarkiverse.work.api.InstanceAssignmentStrategy;
import io.quarkiverse.work.api.MultiInstanceContext;

@ApplicationScoped
@Named("composite")
public class CompositeInstanceAssignmentStrategy implements InstanceAssignmentStrategy {

    private final List<InstanceAssignmentStrategy> delegates;

    public CompositeInstanceAssignmentStrategy(final List<InstanceAssignmentStrategy> delegates) {
        this.delegates = delegates;
    }

    @Override
    public void assign(final List<Object> instances, final MultiInstanceContext context) {
        for (final InstanceAssignmentStrategy strategy : delegates) {
            strategy.assign(instances, context);
        }
    }
}
```

- [ ] **Run tests to verify they pass**

```bash
scripts/mvn-test runtime -Dtest=InstanceAssignmentStrategyTest
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/ \
        runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/InstanceAssignmentStrategyTest.java
git commit -m "feat: add InstanceAssignmentStrategy implementations — Pool, Explicit, RoundRobin, Composite

Pool (default): copies candidateGroups/Users to all children.
Explicit: pre-assigns each child to a named person.
RoundRobin: uses WorkerSelectionStrategy per child with exclusion.
Composite: applies multiple strategies in sequence.

Refs #106"
```

---

## Task 10 — Template validation service

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/service/WorkItemTemplateValidationService.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceConfigValidationTest.java`

- [ ] **Write failing tests**

```java
// MultiInstanceConfigValidationTest.java
package io.quarkiverse.work.runtime.multiinstance;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThatCode;

import org.junit.jupiter.api.Test;

import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.service.WorkItemTemplateValidationService;

class MultiInstanceConfigValidationTest {

    private static WorkItemTemplate validMultiInstance() {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "Test";
        t.createdBy = "test";
        t.instanceCount = 5;
        t.requiredCount = 3;
        return t;
    }

    @Test
    void validMultiInstanceTemplatePassesValidation() {
        assertThatCode(() -> WorkItemTemplateValidationService.validate(validMultiInstance()))
                .doesNotThrowAnyException();
    }

    @Test
    void instanceCountZeroIsRejected() {
        WorkItemTemplate t = validMultiInstance();
        t.instanceCount = 0;
        assertThatThrownBy(() -> WorkItemTemplateValidationService.validate(t))
                .hasMessageContaining("instanceCount must be at least 1");
    }

    @Test
    void requiredCountZeroIsRejected() {
        WorkItemTemplate t = validMultiInstance();
        t.requiredCount = 0;
        assertThatThrownBy(() -> WorkItemTemplateValidationService.validate(t))
                .hasMessageContaining("requiredCount must be at least 1");
    }

    @Test
    void requiredCountExceedingInstanceCountIsRejected() {
        WorkItemTemplate t = validMultiInstance();
        t.instanceCount = 3;
        t.requiredCount = 5;
        assertThatThrownBy(() -> WorkItemTemplateValidationService.validate(t))
                .hasMessageContaining("requiredCount (5) cannot exceed instanceCount (3)");
    }

    @Test
    void coordinatorWithClaimDeadlineIsRejected() {
        WorkItemTemplate t = validMultiInstance();
        t.parentRole = "COORDINATOR";
        t.defaultClaimHours = 24;
        assertThatThrownBy(() -> WorkItemTemplateValidationService.validate(t))
                .hasMessageContaining("claimDeadline has no meaning in COORDINATOR mode");
    }

    @Test
    void coordinatorWithClaimBusinessHoursIsRejected() {
        WorkItemTemplate t = validMultiInstance();
        t.parentRole = "COORDINATOR";
        t.defaultClaimBusinessHours = 8;
        assertThatThrownBy(() -> WorkItemTemplateValidationService.validate(t))
                .hasMessageContaining("claimDeadline has no meaning in COORDINATOR mode");
    }

    @Test
    void oneOfOneIsValid() {
        WorkItemTemplate t = validMultiInstance();
        t.instanceCount = 1;
        t.requiredCount = 1;
        assertThatCode(() -> WorkItemTemplateValidationService.validate(t))
                .doesNotThrowAnyException();
    }

    @Test
    void nonMultiInstanceTemplateSkipsValidation() {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "Simple";
        t.createdBy = "test";
        // instanceCount null — not multi-instance
        assertThatCode(() -> WorkItemTemplateValidationService.validate(t))
                .doesNotThrowAnyException();
    }
}
```

- [ ] **Run tests to verify they fail**

```bash
scripts/mvn-test runtime -Dtest=MultiInstanceConfigValidationTest
```
Expected: FAIL — WorkItemTemplateValidationService not found

- [ ] **Implement WorkItemTemplateValidationService**

```java
package io.quarkiverse.work.runtime.service;

import java.util.ArrayList;
import java.util.List;

import io.quarkiverse.work.runtime.model.WorkItemTemplate;

public final class WorkItemTemplateValidationService {

    private WorkItemTemplateValidationService() {}

    /**
     * Validates a WorkItemTemplate for multi-instance consistency.
     * Throws {@link IllegalArgumentException} with a descriptive message if invalid.
     * No-op for templates without instanceCount (non-multi-instance).
     */
    public static void validate(final WorkItemTemplate template) {
        if (template.instanceCount == null) return;

        final List<String> errors = new ArrayList<>();

        if (template.instanceCount < 1) {
            errors.add("instanceCount must be at least 1");
        }
        if (template.requiredCount == null || template.requiredCount < 1) {
            errors.add("requiredCount must be at least 1");
        }
        if (template.instanceCount >= 1 && template.requiredCount != null
                && template.requiredCount > template.instanceCount) {
            errors.add("requiredCount (" + template.requiredCount
                    + ") cannot exceed instanceCount (" + template.instanceCount + ")");
        }
        final boolean isCoordinator = template.parentRole == null
                || "COORDINATOR".equals(template.parentRole);
        if (isCoordinator
                && (template.defaultClaimHours != null || template.defaultClaimBusinessHours != null)) {
            errors.add("claimDeadline has no meaning in COORDINATOR mode; "
                    + "remove defaultClaimHours and defaultClaimBusinessHours");
        }

        if (!errors.isEmpty()) {
            throw new IllegalArgumentException("Invalid multi-instance template: " + String.join("; ", errors));
        }
    }
}
```

- [ ] **Run tests to verify they pass**

```bash
scripts/mvn-test runtime -Dtest=MultiInstanceConfigValidationTest
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/service/WorkItemTemplateValidationService.java \
        runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceConfigValidationTest.java
git commit -m "feat: add WorkItemTemplateValidationService for multi-instance config rules

Validates instanceCount>=1, requiredCount>=1, requiredCount<=instanceCount,
COORDINATOR mode + claimDeadline conflict. Runs at template create time.

Refs #106"
```

---

## Task 11 — MultiInstanceSpawnService (create parent + N children)

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceSpawnService.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceCreateTest.java`

- [ ] **Write failing test**

```java
// MultiInstanceCreateTest.java
package io.quarkiverse.work.runtime.multiinstance;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemRelationType;
import io.quarkiverse.work.runtime.model.WorkItemSpawnGroup;
import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.service.WorkItemTemplateService;

@QuarkusTest
class MultiInstanceCreateTest {

    @Inject
    WorkItemTemplateService templateService;

    @Test
    @Transactional
    void instantiatingMultiInstanceTemplateCreatesParentAndNChildren() {
        WorkItemTemplate template = new WorkItemTemplate();
        template.name = "Approval";
        template.category = "approval";
        template.candidateGroups = "reviewers";
        template.createdBy = "test";
        template.instanceCount = 3;
        template.requiredCount = 2;
        template.persist();

        WorkItem parent = templateService.instantiate(template, null, null, "test");

        assertThat(parent.parentId).isNull(); // parent has no parent
        assertThat(parent.id).isNotNull();

        // Three children should exist
        List<WorkItem> children = WorkItem.list("parentId", parent.id);
        assertThat(children).hasSize(3);

        // All children have PART_OF relation to parent
        children.forEach(child -> {
            assertThat(child.parentId).isEqualTo(parent.id);
            long relations = WorkItemRelation.count(
                "sourceId = ?1 AND targetId = ?2 AND relationType = ?3",
                child.id, parent.id, WorkItemRelationType.PART_OF);
            assertThat(relations).isEqualTo(1);
        });

        // Spawn group created with policy
        WorkItemSpawnGroup group = WorkItemSpawnGroup.findMultiInstanceByParentId(parent.id);
        assertThat(group).isNotNull();
        assertThat(group.instanceCount).isEqualTo(3);
        assertThat(group.requiredCount).isEqualTo(2);
        assertThat(group.completedCount).isZero();
        assertThat(group.policyTriggered).isFalse();
    }

    @Test
    @Transactional
    void nonMultiInstanceTemplateCreatesOneWorkItem() {
        WorkItemTemplate template = new WorkItemTemplate();
        template.name = "Simple Task";
        template.createdBy = "test";
        template.persist();

        WorkItem item = templateService.instantiate(template, null, null, "test");

        assertThat(item.parentId).isNull();
        assertThat(WorkItem.count("parentId", item.id)).isZero();
    }
}
```

- [ ] **Run to verify it fails**

```bash
scripts/mvn-test runtime -Dtest=MultiInstanceCreateTest
```
Expected: FAIL

- [ ] **Implement MultiInstanceSpawnService**

```java
package io.quarkiverse.work.runtime.multiinstance;

import java.util.ArrayList;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.inject.Named;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.api.InstanceAssignmentStrategy;
import io.quarkiverse.work.api.MultiInstanceContext;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemCreateRequest;
import io.quarkiverse.work.runtime.model.WorkItemRelation;
import io.quarkiverse.work.runtime.model.WorkItemRelationType;
import io.quarkiverse.work.runtime.model.WorkItemSpawnGroup;
import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.service.WorkItemService;

@ApplicationScoped
public class MultiInstanceSpawnService {

    @Inject
    WorkItemService workItemService;

    @Inject
    @Named("pool")
    InstanceAssignmentStrategy defaultStrategy;

    @Inject
    @Any
    Instance<InstanceAssignmentStrategy> strategies;

    /**
     * Create a multi-instance group: parent WorkItem + N child instances.
     * All created in the caller's transaction.
     *
     * @return the parent WorkItem
     */
    @Transactional
    public WorkItem createGroup(final WorkItemTemplate template, final String titleOverride,
            final String createdBy) {
        // 1. Create parent
        final WorkItemCreateRequest parentReq = buildParentRequest(template, titleOverride, createdBy);
        final WorkItem parent = workItemService.create(parentReq);

        // 2. Create WorkItemSpawnGroup with policy
        final WorkItemSpawnGroup group = new WorkItemSpawnGroup();
        group.parentId = parent.id;
        group.idempotencyKey = "multi-instance:" + parent.id;
        group.instanceCount = template.instanceCount;
        group.requiredCount = template.requiredCount;
        group.onThresholdReached = template.onThresholdReached != null
                ? template.onThresholdReached : "CANCEL";
        group.allowSameAssignee = Boolean.TRUE.equals(template.allowSameAssignee);
        group.parentRole = template.parentRole != null ? template.parentRole : "COORDINATOR";
        group.persist();

        // 3. Create N child instances
        final List<WorkItem> children = new ArrayList<>();
        for (int i = 0; i < template.instanceCount; i++) {
            final WorkItemCreateRequest childReq = buildChildRequest(template, createdBy, i);
            final WorkItem child = workItemService.create(childReq);
            child.parentId = parent.id;

            // Wire PART_OF relation
            final WorkItemRelation rel = new WorkItemRelation();
            rel.sourceId = child.id;
            rel.targetId = parent.id;
            rel.relationType = WorkItemRelationType.PART_OF;
            rel.createdBy = "system:multi-instance:" + group.id;
            rel.persist();

            children.add(child);
        }

        // 4. Apply assignment strategy
        final InstanceAssignmentStrategy strategy = resolveStrategy(template.assignmentStrategy);
        final io.quarkiverse.work.api.MultiInstanceConfig config =
                new io.quarkiverse.work.api.MultiInstanceConfig(
                        template.instanceCount, template.requiredCount, null,
                        template.assignmentStrategy, null,
                        Boolean.TRUE.equals(template.allowSameAssignee), null);
        strategy.assign((List) children, new MultiInstanceContext(parent, config));

        return parent;
    }

    private WorkItemCreateRequest buildParentRequest(final WorkItemTemplate template,
            final String titleOverride, final String createdBy) {
        final String title = (titleOverride != null && !titleOverride.isBlank())
                ? titleOverride : template.name;
        final boolean isCoordinator = template.parentRole == null
                || "COORDINATOR".equals(template.parentRole);
        return new WorkItemCreateRequest(
                title, template.description, template.category, null, template.priority,
                null,
                isCoordinator ? null : template.candidateGroups,
                isCoordinator ? null : template.candidateUsers,
                template.requiredCapabilities, createdBy,
                template.defaultPayload, null, null, null, null, null, null,
                null,
                isCoordinator ? null : template.defaultExpiryBusinessHours);
    }

    private WorkItemCreateRequest buildChildRequest(final WorkItemTemplate template,
            final String createdBy, final int index) {
        return new WorkItemCreateRequest(
                template.name + " [" + (index + 1) + "/" + template.instanceCount + "]",
                template.description, template.category, null, template.priority,
                null, template.candidateGroups, template.candidateUsers,
                template.requiredCapabilities, createdBy,
                template.defaultPayload, null, null, null, null, null, null,
                template.defaultClaimBusinessHours, template.defaultExpiryBusinessHours);
    }

    private InstanceAssignmentStrategy resolveStrategy(final String name) {
        if (name == null || "pool".equals(name)) return defaultStrategy;
        return strategies.select(new Named() {
            public Class<? extends java.lang.annotation.Annotation> annotationType() { return Named.class; }
            public String value() { return name; }
        }).get();
    }
}
```

- [ ] **Wire into WorkItemTemplateService.instantiate()**

In `WorkItemTemplateService.java`, inject `MultiInstanceSpawnService` and add the branch:

```java
@Inject
jakarta.enterprise.inject.Instance<MultiInstanceSpawnService> multiInstanceSpawnService;

@Transactional
public WorkItem instantiate(final WorkItemTemplate template, final String titleOverride,
        final String assigneeIdOverride, final String createdBy) {

    // Multi-instance path
    if (template.instanceCount != null) {
        return multiInstanceSpawnService.get().createGroup(template, titleOverride, createdBy);
    }

    // Existing single-instance path (unchanged below)
    final WorkItemCreateRequest request = toCreateRequest(template, titleOverride, assigneeIdOverride, createdBy);
    WorkItem workItem = workItemService.create(request);
    final List<WorkItemLabel> labels = parseLabels(template);
    for (final WorkItemLabel label : labels) {
        workItem = workItemService.addLabel(workItem.id, label.path, label.appliedBy);
    }
    return workItem;
}
```

Also call validation at template creation time. Find the method where `WorkItemTemplate` is persisted (likely in a `WorkItemTemplateResource`) and add:

```java
WorkItemTemplateValidationService.validate(template);
```
before `template.persist()`.

- [ ] **Run tests**

```bash
scripts/mvn-test runtime -Dtest=MultiInstanceCreateTest
```
Expected: PASS

- [ ] **Run full runtime suite to check for regressions**

```bash
scripts/mvn-test runtime
```
Expected: PASS, no regressions

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceSpawnService.java \
        runtime/src/main/java/io/quarkiverse/work/runtime/service/WorkItemTemplateService.java \
        runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceCreateTest.java
git commit -m "feat: auto-spawn N child WorkItems when template has instanceCount set

MultiInstanceSpawnService creates parent + WorkItemSpawnGroup + N children
in one @Transactional. WorkItemTemplateService.instantiate() detects
instanceCount and delegates. Validation applied at template creation.

Refs #106"
```

---

## Task 12 — MultiInstanceGroupPolicy (OCC counter update)

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceGroupPolicy.java`

- [ ] **Implement MultiInstanceGroupPolicy**

This is the `@Transactional` bean the coordinator delegates to. It owns the OCC logic.

```java
package io.quarkiverse.work.runtime.multiinstance;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.api.GroupStatus;
import io.quarkiverse.work.api.OnThresholdReached;
import io.quarkiverse.work.api.WorkItemGroupLifecycleEvent;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemSpawnGroup;
import io.quarkiverse.work.runtime.model.WorkItemStatus;
import io.quarkiverse.work.runtime.service.WorkItemService;

@ApplicationScoped
public class MultiInstanceGroupPolicy {

    @Inject
    WorkItemService workItemService;

    @Inject
    Event<WorkItemGroupLifecycleEvent> groupEvent;

    /**
     * Update the spawn group counters and evaluate the M-of-N threshold.
     * Called from MultiInstanceCoordinator on @ObservesAsync.
     * Throws OptimisticLockException if concurrent update detected — caller retries.
     */
    @Transactional
    public void process(final WorkItem child) {
        final WorkItemSpawnGroup group = WorkItemSpawnGroup.findMultiInstanceByParentId(child.parentId);
        if (group == null) return;
        if (group.policyTriggered) return;

        if (child.status == WorkItemStatus.COMPLETED) {
            group.completedCount++;
        } else {
            group.rejectedCount++;
        }

        final int remaining = group.instanceCount - group.completedCount - group.rejectedCount;
        final int needed = group.requiredCount - group.completedCount;

        if (group.completedCount >= group.requiredCount) {
            resolve(group, GroupStatus.COMPLETED, child);
        } else if (remaining < needed) {
            resolve(group, GroupStatus.REJECTED, child);
        } else {
            fireGroupEvent(group, GroupStatus.IN_PROGRESS, child);
        }
    }

    private void resolve(final WorkItemSpawnGroup group, final GroupStatus outcome, final WorkItem child) {
        group.policyTriggered = true;

        final WorkItem parent = WorkItem.findById(group.parentId);
        if (outcome == GroupStatus.COMPLETED) {
            workItemService.complete(group.parentId, "system:multi-instance",
                    "threshold-met: " + group.completedCount + "/" + group.requiredCount);
        } else {
            workItemService.reject(group.parentId, "system:multi-instance",
                    "cannot-reach-threshold: " + group.rejectedCount + " rejections");
        }

        if (OnThresholdReached.CANCEL.name().equals(group.onThresholdReached)) {
            cancelRemainingChildren(group);
        }

        fireGroupEvent(group, outcome, child);
    }

    private void cancelRemainingChildren(final WorkItemSpawnGroup group) {
        WorkItem.<WorkItem>list("parentId = ?1 AND status NOT IN ?2",
                group.parentId,
                java.util.List.of(
                        WorkItemStatus.COMPLETED, WorkItemStatus.CANCELLED,
                        WorkItemStatus.REJECTED, WorkItemStatus.EXPIRED))
                .forEach(child -> workItemService.cancel(child.id, "system:multi-instance",
                        "threshold-met — cancelled by group policy"));
    }

    private void fireGroupEvent(final WorkItemSpawnGroup group, final GroupStatus status,
            final WorkItem child) {
        final WorkItem parent = WorkItem.findById(group.parentId);
        groupEvent.fire(WorkItemGroupLifecycleEvent.of(
                group.parentId, group.id,
                group.instanceCount, group.requiredCount,
                group.completedCount, group.rejectedCount,
                status,
                parent != null ? parent.callerRef : null));
    }
}
```

- [ ] **Compile**

```bash
scripts/mvn-compile runtime
```
Expected: BUILD SUCCESS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceGroupPolicy.java
git commit -m "feat: add MultiInstanceGroupPolicy — OCC counter update and threshold evaluation

@Transactional bean that increments completedCount/rejectedCount on
WorkItemSpawnGroup (@Version), evaluates M-of-N threshold, transitions
parent, and fires WorkItemGroupLifecycleEvent. Separated from the
@ObservesAsync coordinator so OptimisticLockException retries are clean.

Refs #106"
```

---

## Task 13 — MultiInstanceCoordinator + tests

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceCoordinator.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceCoordinatorTest.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/WorkItemGroupLifecycleEventTest.java`

- [ ] **Write failing tests**

```java
// MultiInstanceCoordinatorTest.java
package io.quarkiverse.work.runtime.multiinstance;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemSpawnGroup;
import io.quarkiverse.work.runtime.model.WorkItemStatus;
import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.service.WorkItemService;
import io.quarkiverse.work.runtime.service.WorkItemTemplateService;

import java.time.Duration;

@QuarkusTest
class MultiInstanceCoordinatorTest {

    @Inject
    WorkItemTemplateService templateService;

    @Inject
    WorkItemService workItemService;

    private WorkItem createGroup(int instanceCount, int requiredCount) {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "Test";
        t.candidateGroups = "testers";
        t.createdBy = "test";
        t.instanceCount = instanceCount;
        t.requiredCount = requiredCount;
        t.persist();
        return templateService.instantiate(t, null, null, "test");
    }

    @Test
    void parentCompletesWhenMChildrenComplete() {
        // @Transactional per step — async observer needs committed data
        UUID parentId = inTx(() -> createGroup(3, 2).id);

        List<WorkItem> children = inTx(() -> WorkItem.list("parentId", parentId));

        // Complete first child
        inTx(() -> workItemService.complete(children.get(0).id, "alice", "approved"));
        // Complete second child — should trigger parent completion
        inTx(() -> workItemService.complete(children.get(1).id, "bob", "approved"));

        // Wait for async coordinator
        Awaitility.await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            WorkItem parent = inTx(() -> WorkItem.findById(parentId));
            assertThat(parent.status).isEqualTo(WorkItemStatus.COMPLETED);
        });
    }

    @Test
    void parentRejectedWhenGroupCannotReachM() {
        UUID parentId = inTx(() -> createGroup(3, 2).id);
        List<WorkItem> children = inTx(() -> WorkItem.list("parentId", parentId));

        // Reject 2 — remaining (1) < needed (2), group fails
        inTx(() -> workItemService.reject(children.get(0).id, "alice", "denied"));
        inTx(() -> workItemService.reject(children.get(1).id, "bob", "denied"));

        Awaitility.await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            WorkItem parent = inTx(() -> WorkItem.findById(parentId));
            assertThat(parent.status).isEqualTo(WorkItemStatus.REJECTED);
        });
    }

    @Test
    void cancelPolicyRemovesRemainingChildrenAfterThreshold() {
        UUID parentId = inTx(() -> {
            WorkItemTemplate t = new WorkItemTemplate();
            t.name = "CancelTest";
            t.candidateGroups = "testers";
            t.createdBy = "test";
            t.instanceCount = 5;
            t.requiredCount = 2;
            t.onThresholdReached = "CANCEL";
            t.persist();
            return templateService.instantiate(t, null, null, "test").id;
        });

        List<UUID> childIds = inTx(() ->
                WorkItem.<WorkItem>list("parentId", parentId).stream().map(w -> w.id).toList());

        inTx(() -> workItemService.complete(childIds.get(0), "alice", "approved"));
        inTx(() -> workItemService.complete(childIds.get(1), "bob", "approved"));

        Awaitility.await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            long cancelled = inTx(() -> WorkItem.count(
                    "parentId = ?1 AND status = ?2", parentId, WorkItemStatus.CANCELLED));
            assertThat(cancelled).isEqualTo(3);
        });
    }

    @Test
    void leavePolicyDoesNotCancelRemainingChildren() {
        UUID parentId = inTx(() -> {
            WorkItemTemplate t = new WorkItemTemplate();
            t.name = "LeaveTest";
            t.candidateGroups = "testers";
            t.createdBy = "test";
            t.instanceCount = 3;
            t.requiredCount = 2;
            t.onThresholdReached = "LEAVE";
            t.persist();
            return templateService.instantiate(t, null, null, "test").id;
        });

        List<UUID> childIds = inTx(() ->
                WorkItem.<WorkItem>list("parentId", parentId).stream().map(w -> w.id).toList());

        inTx(() -> workItemService.complete(childIds.get(0), "alice", "approved"));
        inTx(() -> workItemService.complete(childIds.get(1), "bob", "approved"));

        Awaitility.await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            WorkItem parent = inTx(() -> WorkItem.findById(parentId));
            assertThat(parent.status).isEqualTo(WorkItemStatus.COMPLETED);
        });

        // Third child still PENDING
        WorkItem third = inTx(() -> WorkItem.findById(childIds.get(2)));
        assertThat(third.status).isEqualTo(WorkItemStatus.PENDING);
    }

    @Test
    void policyTriggeredIsIdempotent_additionalEventsIgnored() {
        UUID parentId = inTx(() -> createGroup(3, 2).id);
        List<UUID> childIds = inTx(() ->
                WorkItem.<WorkItem>list("parentId", parentId).stream().map(w -> w.id).toList());

        // Complete 3 children (exceeds M=2) — parent should complete once only
        inTx(() -> workItemService.complete(childIds.get(0), "alice", "approved"));
        inTx(() -> workItemService.complete(childIds.get(1), "bob", "approved"));
        inTx(() -> workItemService.complete(childIds.get(2), "carol", "approved"));

        Awaitility.await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            WorkItemSpawnGroup group = inTx(() ->
                    WorkItemSpawnGroup.findMultiInstanceByParentId(parentId));
            assertThat(group.policyTriggered).isTrue();
            assertThat(group.completedCount).isLessThanOrEqualTo(3);
        });

        WorkItem parent = inTx(() -> WorkItem.findById(parentId));
        assertThat(parent.status).isEqualTo(WorkItemStatus.COMPLETED);
    }

    // Helpers

    @FunctionalInterface
    interface TxSupplier<T> { T get(); }

    @Transactional
    <T> T inTx(TxSupplier<T> supplier) { return supplier.get(); }

    @Transactional
    void inTx(Runnable r) { r.run(); }
}
```

```java
// WorkItemGroupLifecycleEventTest.java
package io.quarkiverse.work.runtime.multiinstance;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import org.awaitility.Awaitility;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.api.GroupStatus;
import io.quarkiverse.work.api.WorkItemGroupLifecycleEvent;
import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.service.WorkItemService;
import io.quarkiverse.work.runtime.service.WorkItemTemplateService;

import java.time.Duration;

@QuarkusTest
class WorkItemGroupLifecycleEventTest {

    @Inject
    WorkItemTemplateService templateService;

    @Inject
    WorkItemService workItemService;

    @Inject
    EventCapture capture;

    @Test
    void inProgressEventFiresOnEachChildTerminalBeforeThreshold() {
        capture.clear();

        UUID parentId = inTx(() -> {
            WorkItemTemplate t = new WorkItemTemplate();
            t.name = "EventTest";
            t.candidateGroups = "g";
            t.createdBy = "test";
            t.instanceCount = 3;
            t.requiredCount = 2;
            t.persist();
            return templateService.instantiate(t, null, null, "test").id;
        });

        List<UUID> children = inTx(() ->
                io.quarkiverse.work.runtime.model.WorkItem.<io.quarkiverse.work.runtime.model.WorkItem>
                        list("parentId", parentId).stream().map(w -> w.id).toList());

        inTx(() -> workItemService.complete(children.get(0), "a", "ok"));

        Awaitility.await().atMost(Duration.ofSeconds(5))
                .until(() -> capture.hasStatus(GroupStatus.IN_PROGRESS));

        assertThat(capture.byStatus(GroupStatus.IN_PROGRESS)).hasSize(1);
        assertThat(capture.byStatus(GroupStatus.IN_PROGRESS).get(0).completedCount()).isEqualTo(1);
    }

    @Test
    void completedEventFiresExactlyOnceAtThreshold() {
        capture.clear();

        UUID parentId = inTx(() -> {
            WorkItemTemplate t = new WorkItemTemplate();
            t.name = "CompletedEventTest";
            t.candidateGroups = "g";
            t.createdBy = "test";
            t.instanceCount = 3;
            t.requiredCount = 2;
            t.persist();
            return templateService.instantiate(t, null, null, "test").id;
        });

        List<UUID> children = inTx(() ->
                io.quarkiverse.work.runtime.model.WorkItem.<io.quarkiverse.work.runtime.model.WorkItem>
                        list("parentId", parentId).stream().map(w -> w.id).toList());

        inTx(() -> workItemService.complete(children.get(0), "a", "ok"));
        inTx(() -> workItemService.complete(children.get(1), "b", "ok"));
        inTx(() -> workItemService.complete(children.get(2), "c", "ok")); // exceeds M

        Awaitility.await().atMost(Duration.ofSeconds(5))
                .until(() -> capture.hasStatus(GroupStatus.COMPLETED));

        assertThat(capture.byStatus(GroupStatus.COMPLETED)).hasSize(1);
    }

    @Transactional
    <T> T inTx(java.util.function.Supplier<T> s) { return s.get(); }

    @Transactional
    void inTx(Runnable r) { r.run(); }

    /** CDI observer that captures WorkItemGroupLifecycleEvents for test assertions. */
    @jakarta.enterprise.context.ApplicationScoped
    static class EventCapture {
        private final List<WorkItemGroupLifecycleEvent> events = new ArrayList<>();

        void onEvent(@ObservesAsync WorkItemGroupLifecycleEvent event) {
            synchronized (events) { events.add(event); }
        }

        void clear() { synchronized (events) { events.clear(); } }

        boolean hasStatus(GroupStatus s) {
            synchronized (events) { return events.stream().anyMatch(e -> e.groupStatus() == s); }
        }

        List<WorkItemGroupLifecycleEvent> byStatus(GroupStatus s) {
            synchronized (events) {
                return events.stream().filter(e -> e.groupStatus() == s).toList();
            }
        }
    }
}
```

- [ ] **Implement MultiInstanceCoordinator**

```java
package io.quarkiverse.work.runtime.multiinstance;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.persistence.OptimisticLockException;

import io.quarkiverse.work.runtime.event.WorkItemLifecycleEvent;
import io.quarkiverse.work.runtime.model.WorkItem;

@ApplicationScoped
public class MultiInstanceCoordinator {

    @Inject
    MultiInstanceGroupPolicy policy;

    void onChildTerminal(@ObservesAsync WorkItemLifecycleEvent event) {
        final WorkItem child = (WorkItem) event.source();
        if (child.parentId == null) return;
        if (!child.status.isTerminal()) return;

        int attempt = 0;
        while (attempt < 2) {
            try {
                policy.process(child);
                return;
            } catch (OptimisticLockException e) {
                attempt++;
                // On retry: policyTriggered=true means another invocation already resolved — return silently
            }
        }
    }
}
```

- [ ] **Run coordinator tests**

```bash
scripts/mvn-test runtime -Dtest=MultiInstanceCoordinatorTest,WorkItemGroupLifecycleEventTest
```
Expected: PASS

- [ ] **Run full runtime suite**

```bash
scripts/mvn-test runtime
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceCoordinator.java \
        runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceCoordinatorTest.java \
        runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/WorkItemGroupLifecycleEventTest.java
git commit -m "feat: add MultiInstanceCoordinator — async M-of-N group policy observer

@ObservesAsync child terminal events, delegates to MultiInstanceGroupPolicy
(@Transactional), retries once on OptimisticLockException. policyTriggered
flag ensures exactly-once parent transition.

Refs #106"
```

---

## Task 14 — Claim guard

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/service/WorkItemService.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceClaimGuardTest.java`

- [ ] **Write failing test**

```java
// MultiInstanceClaimGuardTest.java
package io.quarkiverse.work.runtime.multiinstance;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThatCode;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.service.WorkItemService;
import io.quarkiverse.work.runtime.service.WorkItemTemplateService;

@QuarkusTest
class MultiInstanceClaimGuardTest {

    @Inject WorkItemTemplateService templateService;
    @Inject WorkItemService workItemService;

    private UUID createGroupAndGetParentId(boolean allowSameAssignee) {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "Claim Guard Test";
        t.candidateGroups = "reviewers";
        t.createdBy = "test";
        t.instanceCount = 3;
        t.requiredCount = 2;
        t.allowSameAssignee = allowSameAssignee;
        t.persist();
        return templateService.instantiate(t, null, null, "test").id;
    }

    @Test
    @Transactional
    void guardEnforced_sameAssigneeCannotClaimTwoInstances() {
        UUID parentId = createGroupAndGetParentId(false);
        List<WorkItem> children = WorkItem.list("parentId", parentId);

        // Claim first instance successfully
        workItemService.claim(children.get(0).id, "alice");

        // Second claim by same person should fail
        assertThatThrownBy(() -> workItemService.claim(children.get(1).id, "alice"))
                .hasMessageContaining("already hold another instance");
    }

    @Test
    @Transactional
    void guardDisabled_sameAssigneeCanClaimMultipleInstances() {
        UUID parentId = createGroupAndGetParentId(true);
        List<WorkItem> children = WorkItem.list("parentId", parentId);

        workItemService.claim(children.get(0).id, "alice");
        assertThatCode(() -> workItemService.claim(children.get(1).id, "alice"))
                .doesNotThrowAnyException();
    }

    @Test
    @Transactional
    void differentAssigneesCanAlwaysClaim() {
        UUID parentId = createGroupAndGetParentId(false);
        List<WorkItem> children = WorkItem.list("parentId", parentId);

        workItemService.claim(children.get(0).id, "alice");
        assertThatCode(() -> workItemService.claim(children.get(1).id, "bob"))
                .doesNotThrowAnyException();
    }
}
```

- [ ] **Run to verify it fails**

```bash
scripts/mvn-test runtime -Dtest=MultiInstanceClaimGuardTest
```
Expected: FAIL — guard not implemented

- [ ] **Add claim guard to WorkItemService.claim()**

Find the `claim(UUID workItemId, String claimantId)` method. Add this guard at the very start, before the existing claim logic:

```java
// Multi-instance claim guard
if (item.parentId != null) {
    final WorkItemSpawnGroup group = WorkItemSpawnGroup.findMultiInstanceByParentId(item.parentId);
    if (group != null && !group.allowSameAssignee) {
        final long alreadyHeld = workItemStore.countByParentAndAssignee(item.parentId, claimantId, workItemId);
        if (alreadyHeld > 0) {
            throw new IllegalStateException(
                    "Claimant '" + claimantId + "' already holds another instance in this group");
        }
    }
}
```

Add `countByParentAndAssignee` to `WorkItemStore` SPI as a `default` returning 0:

```java
// In WorkItemStore.java
default long countByParentAndAssignee(UUID parentId, String assigneeId, UUID excludeId) {
    return 0L;
}
```

Implement in `JpaWorkItemStore.java`:

```java
@Override
public long countByParentAndAssignee(final UUID parentId, final String assigneeId, final UUID excludeId) {
    return WorkItem.count("parentId = ?1 AND assigneeId = ?2 AND id != ?3",
            parentId, assigneeId, excludeId);
}
```

- [ ] **Run tests**

```bash
scripts/mvn-test runtime -Dtest=MultiInstanceClaimGuardTest
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/service/WorkItemService.java \
        runtime/src/main/java/io/quarkiverse/work/runtime/repository/WorkItemStore.java \
        runtime/src/main/java/io/quarkiverse/work/runtime/repository/jpa/JpaWorkItemStore.java \
        runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceClaimGuardTest.java
git commit -m "feat: add claim guard — allowSameAssignee=false prevents same person holding multiple instances

Checked at claim time via countByParentAndAssignee. Default=false (guard on).
Guard disabled per group when allowSameAssignee=true on the template.

Refs #106"
```

---

## Task 15 — Threaded inbox: WorkItemRootView + scanRoots

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemRootView.java`
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/repository/WorkItemStore.java`
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/repository/jpa/JpaWorkItemStore.java`
- Modify: `runtime/src/main/java/io/quarkiverse/work/runtime/api/WorkItemResource.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceInboxTest.java`

- [ ] **Create WorkItemRootView**

```java
package io.quarkiverse.work.runtime.model;

import java.util.UUID;

import io.quarkiverse.work.api.GroupStatus;

/**
 * Projection of a root WorkItem (parentId IS NULL) enriched with
 * aggregate stats for the threaded inbox view.
 */
public record WorkItemRootView(
        WorkItem workItem,
        int childCount,
        Integer completedCount,  // null for non-multi-instance roots
        Integer requiredCount,   // null for non-multi-instance roots
        GroupStatus groupStatus  // null for non-multi-instance roots
) {}
```

- [ ] **Add scanRoots default to WorkItemStore SPI**

```java
// In WorkItemStore.java — add after existing scan() method
/**
 * Return all root WorkItems (parentId IS NULL) visible to the given user,
 * enriched with aggregate stats. "Visible" means: directly assigned to the user,
 * in a candidate group/user list, OR the user has visibility into at least one
 * descendant WorkItem.
 *
 * Default implementation returns an empty list — override in JPA store.
 */
default List<WorkItemRootView> scanRoots(String userId, List<String> candidateGroups) {
    return java.util.Collections.emptyList();
}
```

- [ ] **Implement scanRoots in JpaWorkItemStore**

Depth-1 ancestor lookup (works in H2 `MODE=PostgreSQL` and PostgreSQL). For multi-instance, depth is always exactly 2 (parent + children), so this covers all current use cases without a recursive CTE.

```java
@Override
public List<WorkItemRootView> scanRoots(final String userId, final List<String> userGroups) {
    // Build visibility predicate: direct assignee, candidateUsers, or candidateGroups membership
    final StringBuilder pred = new StringBuilder();
    final List<Object> params = new ArrayList<>();
    int p = 1;

    if (userId != null && !userId.isBlank()) {
        pred.append("assigneeId = ?").append(p++);
        params.add(userId);
        pred.append(" OR candidateUsers LIKE ?").append(p++);
        params.add("%" + userId + "%");
    }
    if (userGroups != null) {
        for (final String group : userGroups) {
            if (!pred.isEmpty()) pred.append(" OR ");
            pred.append("candidateGroups LIKE ?").append(p++);
            params.add("%" + group + "%");
        }
    }
    if (pred.isEmpty()) {
        return java.util.Collections.emptyList();
    }

    final List<WorkItem> directlyVisible = WorkItem.list(pred.toString(), params.toArray());

    // Collect roots from directly visible
    final Set<UUID> rootIds = new LinkedHashSet<>();
    final Map<UUID, WorkItem> rootItems = new LinkedHashMap<>();

    for (final WorkItem item : directlyVisible) {
        if (item.parentId == null) {
            rootIds.add(item.id);
            rootItems.put(item.id, item);
        } else {
            // Add ancestor
            final WorkItem parent = WorkItem.findById(item.parentId);
            if (parent != null && parent.parentId == null) {
                rootIds.add(parent.id);
                rootItems.put(parent.id, parent);
            }
        }
    }

    return rootIds.stream().map(id -> {
        final WorkItem root = rootItems.get(id);
        final WorkItemSpawnGroup group = WorkItemSpawnGroup.findMultiInstanceByParentId(id);
        final int childCount = (int) WorkItem.count("parentId", id);
        if (group != null) {
            final GroupStatus status = group.policyTriggered
                    ? (group.completedCount >= group.requiredCount
                            ? GroupStatus.COMPLETED : GroupStatus.REJECTED)
                    : GroupStatus.IN_PROGRESS;
            return new WorkItemRootView(root, childCount,
                    group.completedCount, group.requiredCount, status);
        }
        return new WorkItemRootView(root, childCount, null, null, null);
    }).toList();
}
```

- [ ] **Write failing inbox test**

```java
// MultiInstanceInboxTest.java
package io.quarkiverse.work.runtime.multiinstance;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.api.GroupStatus;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemRootView;
import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.repository.WorkItemStore;
import io.quarkiverse.work.runtime.service.WorkItemTemplateService;

@QuarkusTest
class MultiInstanceInboxTest {

    @Inject WorkItemTemplateService templateService;
    @Inject WorkItemStore store;

    @Test
    @Transactional
    void standaloneWorkItemAppearsAsRootWithChildCountZero() {
        WorkItem item = new WorkItem();
        item.assigneeId = "alice";
        item.status = io.quarkiverse.work.runtime.model.WorkItemStatus.PENDING;
        item.title = "Standalone";
        item.createdBy = "test";
        item.persist();

        List<WorkItemRootView> roots = store.scanRoots("alice", List.of());
        assertThat(roots).anyMatch(r -> r.workItem().id.equals(item.id)
                && r.childCount() == 0
                && r.completedCount() == null
                && r.groupStatus() == null);
    }

    @Test
    @Transactional
    void multiInstanceParentAppearsAsRootWithAggregateStats() {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "Inbox Group";
        t.candidateGroups = "inbox-group";
        t.createdBy = "test";
        t.instanceCount = 3;
        t.requiredCount = 2;
        t.persist();

        WorkItem parent = templateService.instantiate(t, null, null, "test");
        List<WorkItemRootView> roots = store.scanRoots(null, List.of("inbox-group"));

        assertThat(roots).anyMatch(r ->
                r.workItem().id.equals(parent.id)
                        && r.childCount() == 3
                        && r.completedCount() == 0
                        && r.requiredCount() == 2
                        && r.groupStatus() == GroupStatus.IN_PROGRESS);
    }

    @Test
    @Transactional
    void coordinatorParentVisibleWhenUserAssignedToChild() {
        // Coordinator parent has no candidate groups/assignee — only children do
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "Coordinator Visibility";
        t.candidateGroups = "child-group";
        t.createdBy = "test";
        t.instanceCount = 2;
        t.requiredCount = 1;
        t.parentRole = "COORDINATOR";
        t.persist();

        WorkItem parent = templateService.instantiate(t, null, null, "test");

        // User is in child-group (has visibility into children, not parent directly)
        List<WorkItemRootView> roots = store.scanRoots(null, List.of("child-group"));

        assertThat(roots).anyMatch(r -> r.workItem().id.equals(parent.id));
    }

    @Test
    @Transactional
    void childrenDoNotAppearDirectlyInInbox() {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "No Children In Inbox";
        t.candidateGroups = "g";
        t.createdBy = "test";
        t.instanceCount = 2;
        t.requiredCount = 1;
        t.persist();

        templateService.instantiate(t, null, null, "test");

        List<WorkItemRootView> roots = store.scanRoots(null, List.of("g"));

        // All returned items must have parentId == null (roots only)
        assertThat(roots).allMatch(r -> r.workItem().parentId == null);
    }
}
```

- [ ] **Update WorkItemResource.inbox() to use scanRoots**

Replace the existing `inbox()` method body:

```java
@GET
@Path("/inbox")
public List<WorkItemRootResponse> inbox(
        @QueryParam("assignee") final String assignee,
        @QueryParam("candidateGroup") final List<String> candidateGroups) {

    final List<WorkItemRootView> roots = workItemStore.scanRoots(assignee, candidateGroups);
    return roots.stream().map(WorkItemRootResponse::from).toList();
}
```

Add `WorkItemRootResponse` as a record in the `api` package:

```java
public record WorkItemRootResponse(
        WorkItemResponse item,
        int childCount,
        Integer completedCount,
        Integer requiredCount,
        String groupStatus) {

    public static WorkItemRootResponse from(final WorkItemRootView view) {
        return new WorkItemRootResponse(
                WorkItemMapper.toResponse(view.workItem()),
                view.childCount(),
                view.completedCount(),
                view.requiredCount(),
                view.groupStatus() != null ? view.groupStatus().name() : null);
    }
}
```

- [ ] **Run inbox tests**

```bash
scripts/mvn-test runtime -Dtest=MultiInstanceInboxTest
```
Expected: PASS

- [ ] **Run full runtime suite**

```bash
scripts/mvn-test runtime
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/model/WorkItemRootView.java \
        runtime/src/main/java/io/quarkiverse/work/runtime/repository/WorkItemStore.java \
        runtime/src/main/java/io/quarkiverse/work/runtime/repository/jpa/JpaWorkItemStore.java \
        runtime/src/main/java/io/quarkiverse/work/runtime/api/WorkItemResource.java \
        runtime/src/test/java/io/quarkiverse/work/runtime/multiinstance/MultiInstanceInboxTest.java
git commit -m "feat: threaded inbox — GET /workitems/inbox always returns roots with aggregate stats

Inbox returns parentId=null WorkItems only. Coordinator parents visible
via descendant assignment. Each root carries childCount/completedCount/
requiredCount/groupStatus. Children never appear directly in inbox.

Refs #106"
```

---

## Task 16 — GET /workitems/{id}/instances endpoint

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/work/runtime/api/WorkItemInstancesResource.java`
- Create: `runtime/src/test/java/io/quarkiverse/work/runtime/api/WorkItemInstancesResourceTest.java`

- [ ] **Write failing test**

```java
// WorkItemInstancesResourceTest.java
package io.quarkiverse.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.runtime.model.WorkItemTemplate;
import io.quarkiverse.work.runtime.service.WorkItemTemplateService;

@QuarkusTest
class WorkItemInstancesResourceTest {

    @Inject WorkItemTemplateService templateService;

    @Test
    @Transactional
    void getInstancesReturnsChildrenWithGroupSummary() {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "Instances Test";
        t.candidateGroups = "g";
        t.createdBy = "test";
        t.instanceCount = 3;
        t.requiredCount = 2;
        t.persist();

        String parentId = templateService.instantiate(t, null, null, "test").id.toString();

        given()
                .when().get("/workitems/" + parentId + "/instances")
                .then()
                .statusCode(200)
                .body("parentId", equalTo(parentId))
                .body("instanceCount", equalTo(3))
                .body("requiredCount", equalTo(2))
                .body("completedCount", equalTo(0))
                .body("groupStatus", equalTo("IN_PROGRESS"))
                .body("instances", hasSize(3));
    }

    @Test
    void getInstancesReturns404ForNonExistentParent() {
        given()
                .when().get("/workitems/" + UUID.randomUUID() + "/instances")
                .then()
                .statusCode(404);
    }

    @Test
    @Transactional
    void getInstancesReturns404ForNonMultiInstanceWorkItem() {
        io.quarkiverse.work.runtime.model.WorkItem item =
                new io.quarkiverse.work.runtime.model.WorkItem();
        item.title = "Regular";
        item.status = io.quarkiverse.work.runtime.model.WorkItemStatus.PENDING;
        item.createdBy = "test";
        item.persist();

        given()
                .when().get("/workitems/" + item.id + "/instances")
                .then()
                .statusCode(404);
    }
}
```

- [ ] **Implement WorkItemInstancesResource**

```java
package io.quarkiverse.work.runtime.api;

import java.util.List;
import java.util.UUID;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.work.api.GroupStatus;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemSpawnGroup;
import io.quarkiverse.work.runtime.model.WorkItemStatus;

@Path("/workitems")
@Produces(MediaType.APPLICATION_JSON)
public class WorkItemInstancesResource {

    public record InstancesResponse(
            UUID parentId,
            UUID groupId,
            int instanceCount,
            int requiredCount,
            int completedCount,
            int rejectedCount,
            String groupStatus,
            List<WorkItemResponse> instances) {}

    @GET
    @Path("/{id}/instances")
    public Response getInstances(@PathParam("id") final UUID parentId) {
        final WorkItem parent = WorkItem.findById(parentId);
        if (parent == null) return Response.status(404).build();

        final WorkItemSpawnGroup group = WorkItemSpawnGroup.findMultiInstanceByParentId(parentId);
        if (group == null) return Response.status(404).build();

        final List<WorkItem> children = WorkItem.list("parentId", parentId);
        final GroupStatus status = deriveStatus(group);

        return Response.ok(new InstancesResponse(
                parentId, group.id,
                group.instanceCount, group.requiredCount,
                group.completedCount, group.rejectedCount,
                status.name(),
                children.stream().map(WorkItemMapper::toResponse).toList())).build();
    }

    private GroupStatus deriveStatus(final WorkItemSpawnGroup group) {
        if (!group.policyTriggered) return GroupStatus.IN_PROGRESS;
        return group.completedCount >= group.requiredCount
                ? GroupStatus.COMPLETED : GroupStatus.REJECTED;
    }
}
```

- [ ] **Run tests**

```bash
scripts/mvn-test runtime -Dtest=WorkItemInstancesResourceTest
```
Expected: PASS

- [ ] **Commit**

```bash
git add runtime/src/main/java/io/quarkiverse/work/runtime/api/WorkItemInstancesResource.java \
        runtime/src/test/java/io/quarkiverse/work/runtime/api/WorkItemInstancesResourceTest.java
git commit -m "feat: add GET /workitems/{id}/instances endpoint

Returns child instances with group summary (instanceCount, requiredCount,
completedCount, groupStatus). 404 for non-existent or non-multi-instance parents.

Refs #106"
```

---

## Task 17 — Full runtime test suite + install

- [ ] **Run full runtime suite**

```bash
scripts/mvn-test runtime
```
Expected: PASS, all existing tests unbroken

- [ ] **Install runtime to local Maven repo**

```bash
scripts/mvn-install runtime
```

- [ ] **Compile dependents**

```bash
scripts/mvn-compile quarkus-work-reports
scripts/mvn-compile quarkus-work-notifications
```
Expected: BUILD SUCCESS (no broken dependents)

---

## Task 18 — Documentation: DESIGN.md + CLAUDE.md + platform docs

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md` (gotchas section)
- Modify: `~/claude/casehub-parent/docs/repos/quarkus-work.md`
- Modify: `~/claude/casehub-parent/docs/PLATFORM.md`

- [ ] **Update docs/DESIGN.md**

Find the "Module Structure" or "Current State" section and add multi-instance to the completed features list:

```markdown
### Multi-Instance WorkItems (Epic #106)

Template-driven M-of-N parallel instances. `instanceCount` on `WorkItemTemplate` triggers
auto-spawn of N child WorkItems in `WorkItemTemplateService.instantiate()`. `MultiInstanceCoordinator`
(`@ObservesAsync`) reacts to child terminal events and applies group policy via `MultiInstanceGroupPolicy`
(OCC `@Version` on `WorkItemSpawnGroup`). Inbox always returns roots with aggregate stats.

Key classes:
- `MultiInstanceSpawnService` — parent + group + N children in one `@Transactional`
- `MultiInstanceGroupPolicy` — OCC counter update and threshold evaluation
- `MultiInstanceCoordinator` — `@ObservesAsync` entry point with retry
- `MultiInstanceCoordinator` — `InstanceAssignmentStrategy` SPI: Pool (default), RoundRobin, Explicit, Composite
- `WorkItemGroupLifecycleEvent` — group-level CDI event (distinct from `WorkItemLifecycleEvent`)
- `JpaWorkItemStore.scanRoots()` — threaded inbox query with ancestor visibility
```

- [ ] **Update CLAUDE.md gotchas section**

Add the following entry to the "Known Quarkiverse gotchas" section:

```markdown
- `@ObservesAsync` CDI observers with `@Transactional` delegated to a separate bean — if you put the `@Transactional` logic directly in the observer method body and call it via self-reference, CDI proxies won't intercept the transaction boundary (self-invocation). Always delegate to an injected `@ApplicationScoped @Transactional` bean (`MultiInstanceGroupPolicy`) rather than using `@Transactional` on the observer method itself or trying to self-inject.
- `WorkItemSpawnGroup.findMultiInstanceByParentId()` — returns the multi-instance spawn group (where `requiredCount IS NOT NULL`). A parent can have multiple spawn groups (from repeated spawn calls); use this method to get the multi-instance one specifically.
- `scanRoots()` in `JpaWorkItemStore` uses a depth-1 ancestor lookup (not a full recursive CTE) for H2 compatibility. For production PostgreSQL with deeply nested hierarchies, replace with the recursive CTE variant in the spec.
```

- [ ] **Update casehub-parent/docs/repos/quarkus-work.md**

In the "REST API" section, add:
```markdown
- `GET /workitems/{id}/instances` — child instances with group summary (M-of-N progress)
```

In the "What This Repo Explicitly Does NOT Do" section, update the completion rollup bullet to reflect the refined boundary:
```markdown
- **Heterogeneous plan-item completion** — whether named plan items A, B, and C have all completed to advance a Stage; that is CaseHub (see LAYERING.md). Homogeneous M-of-N group completion IS quarkus-work (MultiInstanceCoordinator).
```

In "Key Abstractions", add MultiInstanceCoordinator and WorkItemGroupLifecycleEvent to the Core Services table.

- [ ] **Update casehub-parent/docs/PLATFORM.md**

In the "Capability Ownership" table, add:
```markdown
| M-of-N parallel WorkItem completion (group policy primitive) | `quarkus-work` | `MultiInstanceCoordinator`; `WorkItemGroupLifecycleEvent`; see LAYERING.md |
```

- [ ] **Drift check: scan for stale references**

Search for any documentation that refers to the old inbox behaviour or incorrect boundary rules:

```bash
grep -r "completion rollup" ~/claude/casehub-parent/docs/ ~/claude/quarkus-work/docs/
grep -r "Stage.requiredItemIds" ~/claude/casehub-parent/docs/ ~/claude/quarkus-work/docs/
grep -r "parent auto-completion" ~/claude/quarkus-work/docs/
```

For each hit: check if the surrounding text still accurately reflects the refined design. Update where stale.

- [ ] **Commit documentation**

```bash
cd /Users/mdproctor/claude/quarkus-work
git add docs/DESIGN.md CLAUDE.md
git commit -m "docs: update DESIGN.md and CLAUDE.md for multi-instance WorkItems

Documents MultiInstanceCoordinator, group policy primitives, threaded inbox,
and two new gotchas (self-invocation with @Transactional, findMultiInstanceByParentId).

Closes #106"

cd /Users/mdproctor/claude/casehub-parent
git add docs/repos/quarkus-work.md docs/PLATFORM.md
git commit -m "docs: update platform docs for multi-instance WorkItems and refined LAYERING boundary

Adds M-of-N to capability ownership table. Clarifies that homogeneous
group completion is quarkus-work; heterogeneous plan-item completion is CaseHub."
```

---

## Self-Review Checklist

After completing all tasks, verify:

- [ ] `scripts/mvn-test quarkus-work-api` — PASS
- [ ] `scripts/mvn-test runtime` — PASS, no regressions
- [ ] `scripts/mvn-compile quarkus-work-reports` — PASS
- [ ] `scripts/mvn-compile quarkus-work-notifications` — PASS
- [ ] All commits reference `#106`
- [ ] `GET /workitems/{id}/instances` returns 404 for standalone WorkItems
- [ ] `GET /workitems/inbox` returns only roots (parentId IS NULL)
- [ ] Coordinator parent with `COORDINATOR` role has no candidateGroups/candidateUsers
- [ ] policyTriggered=true prevents double-completion under all code paths
- [ ] `MultiInstanceConfigValidationTest` covers all 7 validation rules from spec
