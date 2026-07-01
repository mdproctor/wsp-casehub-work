# GroupStatus Lifecycle Protocol Compliance

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring GroupStatus into lifecycle protocol compliance — add `isTerminal()`/`isActive()`, persist on WorkItemSpawnGroup, eliminate scattered derivation, register in LIFECYCLE.md.

**Architecture:** GroupStatus gains lifecycle methods matching the platform pattern (WorkItemStatus, PlanItemStatus, CommitmentState). A new `groupStatus` column on `work_item_spawn_group` is set authoritatively by `MultiInstanceGroupPolicy` and read directly by consumers, replacing 3 separate derivation sites that reconstruct status from `policyTriggered` + counts.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Flyway, CDI

## Global Constraints

- Java 21 source (on Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn` not `./mvnw`; always use `-pl <module>` — never full-project build
- Flyway: runtime module owns V1–V999; next available is V39
- Test timeouts: use `scripts/` helpers (see `scripts/README.md`)
- IntelliJ MCP for all rename/move/find-references — never bash grep for semantic operations
- All commits reference issue: `Refs #279`

---

### Task 1: Add `isTerminal()` and `isActive()` to GroupStatus

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/GroupStatus.java`
- Create: `api/src/test/java/io/casehub/work/api/GroupStatusTest.java`

**Interfaces:**
- Produces: `GroupStatus.isTerminal(): boolean`, `GroupStatus.isActive(): boolean`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.work.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class GroupStatusTest {

    @Test
    void inProgress_isActive_notTerminal() {
        assertTrue(GroupStatus.IN_PROGRESS.isActive());
        assertFalse(GroupStatus.IN_PROGRESS.isTerminal());
    }

    @Test
    void completed_isTerminal_notActive() {
        assertTrue(GroupStatus.COMPLETED.isTerminal());
        assertFalse(GroupStatus.COMPLETED.isActive());
    }

    @Test
    void rejected_isTerminal_notActive() {
        assertTrue(GroupStatus.REJECTED.isTerminal());
        assertFalse(GroupStatus.REJECTED.isActive());
    }

    @Test
    void isTerminal_and_isActive_are_mutually_exclusive_and_exhaustive() {
        for (final GroupStatus status : GroupStatus.values()) {
            assertNotEquals(status.isTerminal(), status.isActive(),
                    status + " must be exactly one of terminal or active");
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=GroupStatusTest -pl api`
Expected: FAIL — `isTerminal()` and `isActive()` do not exist

- [ ] **Step 3: Implement isTerminal() and isActive()**

```java
package io.casehub.work.api;

/**
 * Aggregate completion status of a multi-instance WorkItem group.
 */
public enum GroupStatus {
    /**
     * Group is still accepting completions — threshold not yet reached.
     */
    IN_PROGRESS,
    /**
     * Threshold reached with majority approval — group completed successfully.
     */
    COMPLETED,
    /**
     * Threshold reached but with majority rejection or escalation — group rejected.
     */
    REJECTED;

    public boolean isTerminal() {
        return this == COMPLETED || this == REJECTED;
    }

    public boolean isActive() {
        return this == IN_PROGRESS;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=GroupStatusTest -pl api`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```
feat(#279): add isTerminal() and isActive() to GroupStatus

Lifecycle protocol compliance — GroupStatus gains the same terminal/active
methods as WorkItemStatus, PlanItemStatus, and CommitmentState.

Refs #279
```

---

### Task 2: Persist groupStatus on WorkItemSpawnGroup + Flyway migration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemSpawnGroup.java`
- Create: `runtime/src/main/resources/db/work/migration/V39__spawn_group_status.sql`
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemSpawnGroupDocument.java`

**Interfaces:**
- Consumes: `GroupStatus.isTerminal()` from Task 1
- Produces: `WorkItemSpawnGroup.groupStatus` field (nullable `GroupStatus`)

- [ ] **Step 1: Add the groupStatus field to WorkItemSpawnGroup**

Add to `WorkItemSpawnGroup.java` after the `policyTriggered` field (line ~116):

```java
@Column(name = "group_status", length = 15)
@jakarta.persistence.Enumerated(jakarta.persistence.EnumType.STRING)
public GroupStatus groupStatus;
```

Add import: `import io.casehub.work.api.GroupStatus;`

- [ ] **Step 2: Create Flyway migration V39**

Create `runtime/src/main/resources/db/work/migration/V39__spawn_group_status.sql`:

```sql
ALTER TABLE work_item_spawn_group ADD COLUMN group_status VARCHAR(15);

UPDATE work_item_spawn_group SET group_status = CASE
    WHEN policy_triggered = true AND completed_count >= required_count THEN 'COMPLETED'
    WHEN policy_triggered = true AND completed_count < required_count THEN 'REJECTED'
    WHEN required_count IS NOT NULL THEN 'IN_PROGRESS'
    ELSE NULL
END;
```

- [ ] **Step 3: Update MongoWorkItemSpawnGroupDocument**

Add field to `MongoWorkItemSpawnGroupDocument.java` after `policyTriggered`:

```java
public String groupStatus;
```

In `from()`, add after `doc.policyTriggered = group.policyTriggered;`:

```java
doc.groupStatus = group.groupStatus != null ? group.groupStatus.name() : null;
```

In `toDomain()`, add after `group.policyTriggered = policyTriggered;`:

```java
group.groupStatus = groupStatus != null ? GroupStatus.valueOf(groupStatus) : null;
```

Add import: `import io.casehub.work.api.GroupStatus;`

- [ ] **Step 4: Run the runtime module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaWorkItemSpawnGroupStoreTenancyTest`
Expected: PASS — existing tests should still work (new nullable column, no constraint changes)

- [ ] **Step 5: Run the MongoDB store test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb -Dtest=MongoWorkItemSpawnGroupStoreTest`
Expected: PASS — document field added, round-trips null correctly

- [ ] **Step 6: Commit**

```
feat(#279): persist groupStatus on WorkItemSpawnGroup

Adds nullable group_status column (V39) with conditional backfill:
- Multi-instance groups with policy_triggered=true → COMPLETED or REJECTED
  based on count comparison
- Multi-instance groups still in progress → IN_PROGRESS
- Non-multi-instance groups → NULL (no lifecycle state machine)

MongoDB document updated with groupStatus field mapping.

Refs #279
```

---

### Task 3: Set groupStatus authoritatively in MultiInstanceGroupPolicy and MultiInstanceSpawnService

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceGroupPolicy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/multiinstance/WorkItemGroupLifecycleEventTest.java`

**Interfaces:**
- Consumes: `WorkItemSpawnGroup.groupStatus` field from Task 2

- [ ] **Step 1: Write a test asserting groupStatus is set on creation**

In `WorkItemGroupLifecycleEventTest.java`, add a test that verifies the spawn group has `groupStatus = IN_PROGRESS` after multi-instance group creation. Find the existing test setup — it already creates multi-instance groups via the spawn service.

Add test:

```java
@Test
void spawnGroup_hasInProgressStatus_afterCreation() {
    // Use existing test infrastructure to create a multi-instance group
    // then verify group.groupStatus == GroupStatus.IN_PROGRESS
    final WorkItemSpawnGroup group = spawnGroupStore.findMultiInstanceByParentId(parentId).orElseThrow();
    assertEquals(GroupStatus.IN_PROGRESS, group.groupStatus);
}
```

The exact test setup depends on the test class structure — adapt to use the existing parent/group creation helpers.

- [ ] **Step 2: Write a test asserting groupStatus is set on threshold**

Add test asserting that after enough children complete, the group's persisted `groupStatus` is `COMPLETED`:

```java
@Test
void groupStatus_isCompleted_afterThresholdMet() {
    // Complete enough children to trigger threshold
    // then reload the spawn group and verify:
    final WorkItemSpawnGroup group = spawnGroupStore.findMultiInstanceByParentId(parentId).orElseThrow();
    assertEquals(GroupStatus.COMPLETED, group.groupStatus);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemGroupLifecycleEventTest`
Expected: FAIL — groupStatus is not set yet

- [ ] **Step 4: Set groupStatus in MultiInstanceSpawnService**

In `MultiInstanceSpawnService.createGroup()`, add after `group.parentRole = ...` (line ~86):

```java
group.groupStatus = GroupStatus.IN_PROGRESS;
```

Add import: `import io.casehub.work.api.GroupStatus;`

- [ ] **Step 5: Set groupStatus in MultiInstanceGroupPolicy.resolve()**

In `MultiInstanceGroupPolicy.resolve()`, add after `group.policyTriggered = true;` (line ~96):

```java
group.groupStatus = outcome;
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemGroupLifecycleEventTest`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(#279): set groupStatus authoritatively in policy and spawn service

MultiInstanceSpawnService sets IN_PROGRESS at group creation.
MultiInstanceGroupPolicy.resolve() sets COMPLETED/REJECTED when the
M-of-N threshold is reached. groupStatus is now the single source of
truth — no derivation from policyTriggered + counts required.

Refs #279
```

---

### Task 4: Eliminate scattered GroupStatus derivation in consumers

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemInstancesResource.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemInstancesResourceTest.java` (verify existing tests still pass)

**Interfaces:**
- Consumes: `WorkItemSpawnGroup.groupStatus` from Task 2, set authoritatively by Task 3

- [ ] **Step 1: Replace derivation in WorkItemInstancesResource**

In `WorkItemInstancesResource.getInstances()`, replace lines 81–83:

```java
final GroupStatus status = group.policyTriggered
        ? (group.completedCount >= group.requiredCount ? GroupStatus.COMPLETED : GroupStatus.REJECTED)
        : GroupStatus.IN_PROGRESS;
```

With:

```java
final GroupStatus status = group.groupStatus != null ? group.groupStatus : GroupStatus.IN_PROGRESS;
```

- [ ] **Step 2: Replace derivation in JpaWorkItemStore.scanRoots()**

In `JpaWorkItemStore.scanRoots()`, replace lines 267–271:

```java
final GroupStatus status = group.policyTriggered
        ? (group.completedCount >= group.requiredCount
                ? GroupStatus.COMPLETED
                : GroupStatus.REJECTED)
        : GroupStatus.IN_PROGRESS;
```

With:

```java
final GroupStatus status = group.groupStatus != null ? group.groupStatus : GroupStatus.IN_PROGRESS;
```

- [ ] **Step 3: Run WorkItemInstancesResourceTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemInstancesResourceTest`
Expected: PASS — existing tests validate the same behaviour; the stored value matches what was previously derived

- [ ] **Step 4: Run MultiInstanceInboxTest (covers scanRoots)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MultiInstanceInboxTest`
Expected: PASS

- [ ] **Step 5: Commit**

```
refactor(#279): read persisted groupStatus instead of deriving from counts

WorkItemInstancesResource and JpaWorkItemStore.scanRoots() now read
group.groupStatus directly instead of reconstructing it from
policyTriggered + completedCount + requiredCount.

Refs #279
```

---

### Task 5: Register GroupStatus in LIFECYCLE.md and update PLATFORM.md

**Files:**
- Modify: `../parent/docs/LIFECYCLE.md` (in casehub-parent repo)
- Modify: `../parent/docs/PLATFORM.md` (in casehub-parent repo)

**Note:** Per `peer-session-cross-repo-commit-discipline` protocol, these changes are committed to the parent repo and recorded in the workspace HANDOFF.md. The parent repo lives at `~/claude/casehub/work/../parent/`.

**Interfaces:**
- Consumes: GroupStatus enum with `isTerminal()`, `isActive()` from Task 1

- [ ] **Step 1: Add GroupStatus to the LIFECYCLE.md registered state machines table**

Add a row after the `CommitmentState` row:

```
| `GroupStatus` | `casehub-work` | `IN_PROGRESS`, `COMPLETED`, `REJECTED` (3) | `isTerminal()`, `isActive()` |
```

- [ ] **Step 2: Update PLATFORM.md capability ownership**

Find the M-of-N / multi-instance row in PLATFORM.md's capability ownership table. Update it to mention GroupStatus as a registered lifecycle state machine with `isTerminal()` / `isActive()`.

- [ ] **Step 3: Commit to parent repo**

```
docs(casehub-work#279): register GroupStatus in LIFECYCLE.md

GroupStatus (casehub-work) — 3 states, isTerminal()/isActive().
Retroactive registration — in production since Chapter C23 (2026-04-28).
PLATFORM.md M-of-N row updated to reference registered state machine.

Refs casehub-work#279
```

- [ ] **Step 4: Record in workspace HANDOFF.md**

Note the parent repo commit in the workspace HANDOFF.md for cross-repo tracking.

---

### Task 6: File cross-repo engine issue and close #279

**Files:** None — GitHub operations only

- [ ] **Step 1: Verify engine#624 exists**

```bash
gh issue view 624 --repo casehubio/engine --json state,title --jq '"[\(.state)] \(.title)"'
```

If it doesn't exist, create it:

```bash
gh issue create --repo casehubio/engine \
  --title "refactor: use GroupStatus.isTerminal() in WorkItemLifecycleAdapter" \
  --body "WorkItemLifecycleAdapter.onWorkItemGroupLifecycle() currently checks:
\`status != GroupStatus.COMPLETED && status != GroupStatus.REJECTED\`

Replace with \`!status.isTerminal()\` — GroupStatus now has isTerminal()/isActive()
per lifecycle protocol (casehubio/work#279).

Refs casehubio/work#279" \
  --label "scale: XS" --label "complexity: Low"
```

- [ ] **Step 2: Close #279**

After all Tasks 1–5 are committed and tests pass:

```bash
gh issue close 279 --repo casehubio/work --comment "Closed by issue-279 branch.

Evaluation: WorkItemGroupLifecycleEvent confirmed standalone — no hierarchy integration.
GroupStatus lifecycle compliance: isTerminal()/isActive() added, persisted on WorkItemSpawnGroup,
scattered derivation eliminated. Registered in LIFECYCLE.md.

Engine consumer migration tracked in casehubio/engine#624."
```
