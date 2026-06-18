# Human Task Lifecycle Alignment — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add FAULTED and OBSOLETE statuses to WorkItemStatus, simple progress reporting, align WorkEventType, fix hardcoded status enumeration across the codebase, and file cross-repo issues for engine/parent/qhorus changes.

**Architecture:** Two new terminal statuses (FAULTED, OBSOLETE) with corresponding service methods, REST endpoints, and persistence changes. All hardcoded status lists replaced with `isTerminal()`/`isActive()` calls. Cross-repo work (casehub-engine adapter changes, casehub-parent LIFECYCLE.md, casehub-qhorus javadoc) filed as separate issues — not implemented in this branch.

**Tech Stack:** Java 21, Quarkus 3.32, JPA/Hibernate, MongoDB Panache, JUnit 5/AssertJ/Mockito

**Spec:** `specs/2026-06-18-lifecycle-alignment-design.md`

---

## Scope

This plan covers **casehub-work changes only**. Cross-repo deliverables (casehub-engine, casehub-parent, casehub-qhorus) are captured as GitHub issues in Task 13.

---

### Task 1: WorkItemStatus — add FAULTED and OBSOLETE

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/model/WorkItemStatusTest.java`

- [ ] **Step 1: Write failing tests for FAULTED and OBSOLETE**

Add to `WorkItemStatusTest.java`:

```java
@Test
void faulted_isTerminal() {
    assertThat(WorkItemStatus.FAULTED.isTerminal()).isTrue();
}

@Test
void faulted_isNotActive() {
    assertThat(WorkItemStatus.FAULTED.isActive()).isFalse();
}

@Test
void obsolete_isTerminal() {
    assertThat(WorkItemStatus.OBSOLETE.isTerminal()).isTrue();
}

@Test
void obsolete_isNotActive() {
    assertThat(WorkItemStatus.OBSOLETE.isActive()).isFalse();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemStatusTest -pl runtime -q`
Expected: Compilation error — `FAULTED` and `OBSOLETE` do not exist yet.

- [ ] **Step 3: Add FAULTED and OBSOLETE to the enum**

In `WorkItemStatus.java`, add `FAULTED` and `OBSOLETE` to the enum values and update `isTerminal()` and `isActive()`:

```java
public enum WorkItemStatus {

    PENDING,
    ASSIGNED,
    IN_PROGRESS,
    COMPLETED,
    REJECTED,
    FAULTED,
    DELEGATED,
    SUSPENDED,
    CANCELLED,
    EXPIRED,
    ESCALATED,
    OBSOLETE;

    public boolean isTerminal() {
        return switch (this) {
            case COMPLETED, REJECTED, FAULTED, CANCELLED, OBSOLETE, EXPIRED, ESCALATED -> true;
            default -> false;
        };
    }

    public boolean isActive() {
        return switch (this) {
            case PENDING, ASSIGNED, IN_PROGRESS, SUSPENDED, DELEGATED -> true;
            default -> false;
        };
    }
}
```

Update javadoc on `isTerminal()` to list all 7 terminal states. Update javadoc on `isActive()` to list all 5 active states.

Add javadoc for `FAULTED`:
```java
/** System or infrastructure failure — distinct from {@link #REJECTED} (actor's deliberate decision).
 *  FAULTED means the system hosting or processing this WorkItem failed.
 *  PENDING→FAULTED means infrastructure failed before anyone could act. */
FAULTED,
```

Add javadoc for `OBSOLETE`:
```java
/** WorkItem superseded by context change — the case context changed, making this work irrelevant.
 *  Distinct from {@link #CANCELLED} (deliberate stop by a human or system).
 *  Typically triggered by the engine or an orchestrator, not by the actor. */
OBSOLETE;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemStatusTest -pl runtime -q`
Expected: All tests pass, including the existing `terminalStatusesAreNeverActive` exhaustive check.

- [ ] **Step 5: Commit**

```
feat(#240): add FAULTED and OBSOLETE to WorkItemStatus

FAULTED: system failure, distinct from REJECTED (actor refusal).
OBSOLETE: context superseded, distinct from CANCELLED (deliberate stop).
Both terminal. Aligns with WS-HumanTask 1.1 (ERROR→FAULTED) and
OpenHumanTask (obsolete→OBSOLETE).

Refs #240
```

---

### Task 2: WorkEventType — add missing values + fix eventType() fallback

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/WorkEventType.java`
- Modify: `api/src/test/java/io/casehub/work/api/WorkEventTypeTest.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEvent.java`

- [ ] **Step 1: Write failing test for new event types**

Add to `WorkEventTypeTest.java`:

```java
@Test
void faulted_exists() {
    assertThat(WorkEventType.valueOf("FAULTED")).isEqualTo(WorkEventType.FAULTED);
}

@Test
void obsolete_exists() {
    assertThat(WorkEventType.valueOf("OBSOLETE")).isEqualTo(WorkEventType.OBSOLETE);
}

@Test
void delegationAccepted_exists() {
    assertThat(WorkEventType.valueOf("DELEGATION_ACCEPTED")).isEqualTo(WorkEventType.DELEGATION_ACCEPTED);
}

@Test
void delegationDeclined_exists() {
    assertThat(WorkEventType.valueOf("DELEGATION_DECLINED")).isEqualTo(WorkEventType.DELEGATION_DECLINED);
}

@Test
void slaReassigned_exists() {
    assertThat(WorkEventType.valueOf("SLA_REASSIGNED")).isEqualTo(WorkEventType.SLA_REASSIGNED);
}

@Test
void slaExtended_exists() {
    assertThat(WorkEventType.valueOf("SLA_EXTENDED")).isEqualTo(WorkEventType.SLA_EXTENDED);
}

@Test
void progressUpdate_exists() {
    assertThat(WorkEventType.valueOf("PROGRESS_UPDATE")).isEqualTo(WorkEventType.PROGRESS_UPDATE);
}

@Test
void labelAdded_exists() {
    assertThat(WorkEventType.valueOf("LABEL_ADDED")).isEqualTo(WorkEventType.LABEL_ADDED);
}

@Test
void labelRemoved_exists() {
    assertThat(WorkEventType.valueOf("LABEL_REMOVED")).isEqualTo(WorkEventType.LABEL_REMOVED);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkEventTypeTest -pl api -q`
Expected: Compilation errors — new enum values don't exist yet.

- [ ] **Step 3: Add missing values to WorkEventType**

Replace the enum body in `WorkEventType.java`:

```java
public enum WorkEventType {
    CREATED,
    ASSIGNED,
    STARTED,
    COMPLETED,
    REJECTED,
    FAULTED,
    DELEGATED,
    DELEGATION_ACCEPTED,
    DELEGATION_DECLINED,
    RELEASED,
    SUSPENDED,
    RESUMED,
    CANCELLED,
    OBSOLETE,
    EXPIRED,
    CLAIM_EXPIRED,
    SPAWNED,
    ESCALATED,
    DEADLINE_EXTENDED,
    SLA_REASSIGNED,
    SLA_EXTENDED,
    SIGNAL_RECEIVED,
    PROGRESS_UPDATE,
    LABEL_ADDED,
    LABEL_REMOVED
}
```

- [ ] **Step 4: Fix eventType() fallback in WorkItemLifecycleEvent**

In `WorkItemLifecycleEvent.java`, replace the `eventType()` method:

```java
@JsonIgnore
@Override
public WorkEventType eventType() {
    final String name = type.substring(type.lastIndexOf('.') + 1).toUpperCase();
    return WorkEventType.valueOf(name);
}
```

Remove the try/catch that silently fell back to `CREATED`. `valueOf` now throws `IllegalArgumentException` if an unknown event name is used — this is a bug that tests must catch.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkEventTypeTest -pl api -q`
Expected: All pass.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemLifecycleEventTest -pl runtime -q`
Expected: All pass — existing tests should not rely on the CREATED fallback.

- [ ] **Step 6: Commit**

```
feat(#240): align WorkEventType to 25 values, fail-fast on unknown events

Add FAULTED, OBSOLETE, DELEGATION_ACCEPTED, DELEGATION_DECLINED,
SLA_REASSIGNED, SLA_EXTENDED, PROGRESS_UPDATE, LABEL_ADDED, LABEL_REMOVED.
eventType() now throws on unknown event names instead of silently
falling back to CREATED.

Refs #240
```

---

### Task 3: WorkItem entity — new fields + Flyway V37

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`
- Create: `runtime/src/main/resources/db/work/migration/V37__progress_fields.sql`

- [ ] **Step 1: Add fields to WorkItem entity**

Add after the `scope` field in `WorkItem.java`:

```java
@Column(name = "percent_complete")
public Integer percentComplete;

@Column(name = "status_note", columnDefinition = "TEXT")
public String statusNote;
```

- [ ] **Step 2: Create Flyway migration**

Create `runtime/src/main/resources/db/work/migration/V37__progress_fields.sql`:

```sql
ALTER TABLE work_item ADD COLUMN percent_complete SMALLINT;
ALTER TABLE work_item ADD COLUMN status_note TEXT;
```

- [ ] **Step 3: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q`
Expected: Compiles successfully.

- [ ] **Step 4: Commit**

```
feat(#240): add percentComplete and statusNote fields to WorkItem

V37 migration adds percent_complete SMALLINT and status_note TEXT
columns. Both nullable — existing rows naturally have NULL.

Refs #240
```

---

### Task 4: WorkItemService — fault() and faultFromSystem()

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`

- [ ] **Step 1: Write failing tests for fault()**

Add to `WorkItemServiceTest.java`:

```java
// -------------------------------------------------------------------------
// Happy paths — fault
// -------------------------------------------------------------------------

@Test
void fault_fromPending_transitionsToFaulted() {
    WorkItem wi = service.create(basicRequest());
    wi = service.fault(wi.id, "system", "infrastructure failure");
    assertThat(wi.status).isEqualTo(WorkItemStatus.FAULTED);
}

@Test
void fault_fromAssigned_transitionsToFaulted() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    wi = service.fault(wi.id, "system", "agent timeout");
    assertThat(wi.status).isEqualTo(WorkItemStatus.FAULTED);
}

@Test
void fault_fromInProgress_transitionsToFaulted() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    wi = service.fault(wi.id, "system", "context overflow");
    assertThat(wi.status).isEqualTo(WorkItemStatus.FAULTED);
}

@Test
void fault_fromSuspended_transitionsToFaulted() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.suspend(wi.id, "alice", "paused");
    wi = service.fault(wi.id, "system", "host crashed");
    assertThat(wi.status).isEqualTo(WorkItemStatus.FAULTED);
}

@Test
void fault_fromDelegated_transitionsToFaulted() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", null);
    wi = service.fault(wi.id, "system", "delegation target unreachable");
    assertThat(wi.status).isEqualTo(WorkItemStatus.FAULTED);
}

@Test
void fault_setsCompletedAt() {
    WorkItem wi = service.create(basicRequest());
    wi = service.fault(wi.id, "system", "failure");
    assertThat(wi.completedAt).isNotNull();
}

@Test
void fault_setsResolution() {
    WorkItem wi = service.create(basicRequest());
    wi = service.fault(wi.id, "system", "API timeout");
    assertThat(wi.resolution).isEqualTo("API timeout");
}

// -------------------------------------------------------------------------
// Error cases — fault
// -------------------------------------------------------------------------

@Test
void fault_fromTerminal_throws() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    service.complete(wi.id, "alice", "done", null);
    final UUID id = wi.id;
    assertThatThrownBy(() -> service.fault(id, "system", "too late"))
            .isInstanceOf(IllegalStateException.class);
}

// -------------------------------------------------------------------------
// faultFromSystem — idempotent
// -------------------------------------------------------------------------

@Test
void faultFromSystem_fromPending_transitionsToFaulted() {
    WorkItem wi = service.create(basicRequest());
    wi = service.faultFromSystem(wi.id, "system", "infrastructure failure");
    assertThat(wi.status).isEqualTo(WorkItemStatus.FAULTED);
}

@Test
void faultFromSystem_fromTerminal_returnsSilently() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    service.complete(wi.id, "alice", "done", null);
    wi = service.faultFromSystem(wi.id, "system", "too late");
    assertThat(wi.status).isEqualTo(WorkItemStatus.COMPLETED);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest#fault* -pl runtime -q`
Expected: Compilation error — `fault()` and `faultFromSystem()` do not exist.

- [ ] **Step 3: Implement fault() and faultFromSystem()**

Add to `WorkItemService.java`:

```java
@Transactional
public WorkItem fault(final UUID id, final String systemActorId, final String errorDetail) {
    final WorkItem item = requireWorkItem(id);
    if (item.status.isTerminal()) {
        throw new IllegalStateException("Cannot fault WorkItem in status: " + item.status);
    }
    item.status = WorkItemStatus.FAULTED;
    item.completedAt = Instant.now();
    item.resolution = errorDetail;
    final WorkItem saved = workItemStore.put(item);
    timerService.cancelExpiry(saved.id);
    timerService.cancelClaimDeadline(saved.id);
    audit(saved.id, "FAULTED", systemActorId, errorDetail);
    if (lifecycleEvent != null) {
        final WorkItemLifecycleEvent evt = WorkItemLifecycleEvent.of("FAULTED", saved, systemActorId, errorDetail);
        lifecycleEvent.fire(evt);
        lifecycleEvent.fireAsync(evt);
    }
    return saved;
}

@Transactional
public WorkItem faultFromSystem(final UUID id, final String actorId, final String errorDetail) {
    final WorkItem item = requireWorkItem(id);
    if (item.status.isTerminal())
        return item;
    item.status = WorkItemStatus.FAULTED;
    item.completedAt = Instant.now();
    item.resolution = errorDetail;
    final WorkItem saved = workItemStore.put(item);
    timerService.cancelExpiry(saved.id);
    timerService.cancelClaimDeadline(saved.id);
    audit(saved.id, "FAULTED", actorId, errorDetail);
    if (lifecycleEvent != null) {
        final WorkItemLifecycleEvent evt = WorkItemLifecycleEvent.of("FAULTED", saved, actorId, errorDetail);
        lifecycleEvent.fire(evt);
        lifecycleEvent.fireAsync(evt);
    }
    return saved;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest -pl runtime -q`
Expected: All tests pass (including existing tests).

- [ ] **Step 5: Commit**

```
feat(#240): add fault() and faultFromSystem() to WorkItemService

fault() transitions any non-terminal → FAULTED, throws on terminal.
faultFromSystem() is the idempotent variant for infrastructure that
races with other terminal transitions.

Refs #240
```

---

### Task 5: WorkItemService — obsolete() and obsoleteFromSystem()

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`

- [ ] **Step 1: Write failing tests for obsolete()**

Add to `WorkItemServiceTest.java`:

```java
// -------------------------------------------------------------------------
// Happy paths — obsolete
// -------------------------------------------------------------------------

@Test
void obsolete_fromPending_transitionsToObsolete() {
    WorkItem wi = service.create(basicRequest());
    wi = service.obsolete(wi.id, "system", "case withdrawn");
    assertThat(wi.status).isEqualTo(WorkItemStatus.OBSOLETE);
}

@Test
void obsolete_fromInProgress_transitionsToObsolete() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    wi = service.obsolete(wi.id, "system", "context changed");
    assertThat(wi.status).isEqualTo(WorkItemStatus.OBSOLETE);
}

@Test
void obsolete_setsCompletedAt() {
    WorkItem wi = service.create(basicRequest());
    wi = service.obsolete(wi.id, "system", "superseded");
    assertThat(wi.completedAt).isNotNull();
}

@Test
void obsolete_setsResolution() {
    WorkItem wi = service.create(basicRequest());
    wi = service.obsolete(wi.id, "system", "trial withdrawn");
    assertThat(wi.resolution).isEqualTo("trial withdrawn");
}

// -------------------------------------------------------------------------
// Error cases — obsolete
// -------------------------------------------------------------------------

@Test
void obsolete_fromTerminal_throws() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    service.complete(wi.id, "alice", "done", null);
    final UUID id = wi.id;
    assertThatThrownBy(() -> service.obsolete(id, "system", "too late"))
            .isInstanceOf(IllegalStateException.class);
}

// -------------------------------------------------------------------------
// obsoleteFromSystem — idempotent
// -------------------------------------------------------------------------

@Test
void obsoleteFromSystem_fromPending_transitionsToObsolete() {
    WorkItem wi = service.create(basicRequest());
    wi = service.obsoleteFromSystem(wi.id, "system", "context changed");
    assertThat(wi.status).isEqualTo(WorkItemStatus.OBSOLETE);
}

@Test
void obsoleteFromSystem_fromTerminal_returnsSilently() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    service.complete(wi.id, "alice", "done", null);
    wi = service.obsoleteFromSystem(wi.id, "system", "too late");
    assertThat(wi.status).isEqualTo(WorkItemStatus.COMPLETED);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest#obsolete* -pl runtime -q`
Expected: Compilation error.

- [ ] **Step 3: Implement obsolete() and obsoleteFromSystem()**

Add to `WorkItemService.java`:

```java
@Transactional
public WorkItem obsolete(final UUID id, final String triggeredBy, final String reason) {
    final WorkItem item = requireWorkItem(id);
    if (item.status.isTerminal()) {
        throw new IllegalStateException("Cannot obsolete WorkItem in status: " + item.status);
    }
    item.status = WorkItemStatus.OBSOLETE;
    item.completedAt = Instant.now();
    item.resolution = reason;
    final WorkItem saved = workItemStore.put(item);
    timerService.cancelExpiry(saved.id);
    timerService.cancelClaimDeadline(saved.id);
    audit(saved.id, "OBSOLETE", triggeredBy, reason);
    if (lifecycleEvent != null) {
        final WorkItemLifecycleEvent evt = WorkItemLifecycleEvent.of("OBSOLETE", saved, triggeredBy, reason);
        lifecycleEvent.fire(evt);
        lifecycleEvent.fireAsync(evt);
    }
    return saved;
}

@Transactional
public WorkItem obsoleteFromSystem(final UUID id, final String triggeredBy, final String reason) {
    final WorkItem item = requireWorkItem(id);
    if (item.status.isTerminal())
        return item;
    item.status = WorkItemStatus.OBSOLETE;
    item.completedAt = Instant.now();
    item.resolution = reason;
    final WorkItem saved = workItemStore.put(item);
    timerService.cancelExpiry(saved.id);
    timerService.cancelClaimDeadline(saved.id);
    audit(saved.id, "OBSOLETE", triggeredBy, reason);
    if (lifecycleEvent != null) {
        final WorkItemLifecycleEvent evt = WorkItemLifecycleEvent.of("OBSOLETE", saved, triggeredBy, reason);
        lifecycleEvent.fire(evt);
        lifecycleEvent.fireAsync(evt);
    }
    return saved;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest -pl runtime -q`
Expected: All pass.

- [ ] **Step 5: Commit**

```
feat(#240): add obsolete() and obsoleteFromSystem() to WorkItemService

obsolete() transitions any non-terminal → OBSOLETE, throws on terminal.
OBSOLETE is context-driven termination — typically triggered by the
engine when the case context changes, not by the actor.

Refs #240
```

---

### Task 6: WorkItemService — cancelFromSystem()

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`

- [ ] **Step 1: Write failing tests**

```java
// -------------------------------------------------------------------------
// cancelFromSystem — idempotent
// -------------------------------------------------------------------------

@Test
void cancelFromSystem_fromPending_transitionsToCancelled() {
    WorkItem wi = service.create(basicRequest());
    wi = service.cancelFromSystem(wi.id, "system", "group policy");
    assertThat(wi.status).isEqualTo(WorkItemStatus.CANCELLED);
}

@Test
void cancelFromSystem_fromTerminal_returnsSilently() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    service.complete(wi.id, "alice", "done", null);
    wi = service.cancelFromSystem(wi.id, "system", "too late");
    assertThat(wi.status).isEqualTo(WorkItemStatus.COMPLETED);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest#cancelFromSystem* -pl runtime -q`
Expected: Compilation error.

- [ ] **Step 3: Implement cancelFromSystem()**

```java
@Transactional
public WorkItem cancelFromSystem(final UUID id, final String actorId, final String reason) {
    final WorkItem item = requireWorkItem(id);
    if (item.status.isTerminal())
        return item;
    item.status = WorkItemStatus.CANCELLED;
    item.completedAt = Instant.now();
    final WorkItem saved = workItemStore.put(item);
    timerService.cancelExpiry(saved.id);
    timerService.cancelClaimDeadline(saved.id);
    audit(saved.id, "CANCELLED", actorId, reason);
    if (lifecycleEvent != null) {
        final WorkItemLifecycleEvent evt = WorkItemLifecycleEvent.of("CANCELLED", saved, actorId, reason);
        lifecycleEvent.fire(evt);
        lifecycleEvent.fireAsync(evt);
    }
    return saved;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest -pl runtime -q`
Expected: All pass.

- [ ] **Step 5: Commit**

```
feat(#240): add cancelFromSystem() for idempotent cancel

Completes the *FromSystem() pattern across all terminal transitions.
Use case: group policy "cancel remaining children on first completion."

Refs #240
```

---

### Task 7: WorkItemService — progress()

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`

- [ ] **Step 1: Write failing tests**

```java
// -------------------------------------------------------------------------
// Happy paths — progress
// -------------------------------------------------------------------------

@Test
void progress_updatesPercentComplete() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    wi = service.progress(wi.id, "alice", 42, "reviewing clause 14");
    assertThat(wi.percentComplete).isEqualTo(42);
}

@Test
void progress_updatesStatusNote() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    wi = service.progress(wi.id, "alice", 80, "almost done");
    assertThat(wi.statusNote).isEqualTo("almost done");
}

@Test
void progress_doesNotChangeStatus() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    wi = service.progress(wi.id, "alice", 50, null);
    assertThat(wi.status).isEqualTo(WorkItemStatus.IN_PROGRESS);
}

@Test
void progress_allowsNullPercentComplete() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    wi = service.progress(wi.id, "alice", null, "status update only");
    assertThat(wi.percentComplete).isNull();
    assertThat(wi.statusNote).isEqualTo("status update only");
}

// -------------------------------------------------------------------------
// Error cases — progress
// -------------------------------------------------------------------------

@Test
void progress_fromPending_throws() {
    WorkItem wi = service.create(basicRequest());
    final UUID id = wi.id;
    assertThatThrownBy(() -> service.progress(id, "alice", 50, "nope"))
            .isInstanceOf(IllegalStateException.class);
}

@Test
void progress_fromAssigned_throws() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    final UUID id = wi.id;
    assertThatThrownBy(() -> service.progress(id, "alice", 50, "nope"))
            .isInstanceOf(IllegalStateException.class);
}

@Test
void progress_fromTerminal_throws() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.start(wi.id, "alice");
    service.complete(wi.id, "alice", "done", null);
    final UUID id = wi.id;
    assertThatThrownBy(() -> service.progress(id, "alice", 50, "too late"))
            .isInstanceOf(IllegalStateException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest#progress* -pl runtime -q`
Expected: Compilation error.

- [ ] **Step 3: Implement progress()**

```java
@Transactional
public WorkItem progress(final UUID id, final String actorId, final Integer percentComplete,
        final String statusNote) {
    final WorkItem item = requireWorkItem(id);
    if (item.status != WorkItemStatus.IN_PROGRESS) {
        throw new IllegalStateException("Cannot report progress for WorkItem in status: " + item.status);
    }
    item.percentComplete = percentComplete;
    item.statusNote = statusNote;
    item.updatedAt = Instant.now();
    final WorkItem saved = workItemStore.put(item);
    audit(saved.id, "PROGRESS_UPDATE", actorId, statusNote);
    if (lifecycleEvent != null) {
        lifecycleEvent.fire(WorkItemLifecycleEvent.of("PROGRESS_UPDATE", saved, actorId, statusNote));
    }
    return saved;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest -pl runtime -q`
Expected: All pass.

- [ ] **Step 5: Commit**

```
feat(#240): add progress() to WorkItemService

Updates percentComplete and statusNote during IN_PROGRESS.
Fires PROGRESS_UPDATE lifecycle event (sync only — not terminal).
Fields persist through suspend/resume.

Refs #240
```

---

### Task 8: REST endpoints — fault, obsolete, progress

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResource.java`

- [ ] **Step 1: Add request records and endpoints**

Add request records:

```java
record FaultRequest(String actor, String errorDetail) {}
record ObsoleteRequest(String actor, String reason) {}
record ProgressRequest(Integer percentComplete, String statusNote) {}
```

Add endpoint methods following existing patterns (`delegate`, `cancel`):

```java
@PUT
@Path("{id}/fault")
public Response fault(@PathParam("id") final UUID id,
        final FaultRequest body) {
    final WorkItem saved = workItemService.fault(id, body.actor(), body.errorDetail());
    return Response.ok(WorkItemMapper.toResponse(saved)).build();
}

@PUT
@Path("{id}/obsolete")
public Response obsolete(@PathParam("id") final UUID id,
        final ObsoleteRequest body) {
    final WorkItem saved = workItemService.obsolete(id, body.actor(), body.reason());
    return Response.ok(WorkItemMapper.toResponse(saved)).build();
}

@PUT
@Path("{id}/progress")
public Response progress(@PathParam("id") final UUID id,
        @HeaderParam("X-Actor-Id") final String actor,
        final ProgressRequest body) {
    final WorkItem saved = workItemService.progress(id, actor, body.percentComplete(), body.statusNote());
    return Response.ok(WorkItemMapper.toResponse(saved)).build();
}
```

- [ ] **Step 2: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q`
Expected: Compiles.

- [ ] **Step 3: Commit**

```
feat(#240): add REST endpoints for fault, obsolete, progress

PUT /workitems/{id}/fault — system failure → FAULTED
PUT /workitems/{id}/obsolete — context superseded → OBSOLETE
PUT /workitems/{id}/progress — progress update (IN_PROGRESS only)

Refs #240
```

---

### Task 9: Persistence and DTO updates

**Files:**
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResponse.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemWithAuditResponse.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemMapper.java`

- [ ] **Step 1: Add fields to MongoWorkItemDocument**

Add fields:
```java
public Integer percentComplete;
public String statusNote;
```

Add to `from()`:
```java
doc.percentComplete = wi.percentComplete;
doc.statusNote = wi.statusNote;
```

Add to `toDomain()`:
```java
wi.percentComplete = percentComplete;
wi.statusNote = statusNote;
```

- [ ] **Step 2: Add fields to WorkItemContextBuilder.toMap()**

Add before the closing `return map;`:
```java
map.put("percentComplete", workItem.percentComplete);
map.put("statusNote", workItem.statusNote);
```

- [ ] **Step 3: Add fields to WorkItemResponse and WorkItemWithAuditResponse**

Add `Integer percentComplete` and `String statusNote` parameters to both record declarations. Update `WorkItemMapper.toResponse()` and `toWithAudit()` to map the new fields.

- [ ] **Step 4: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime,persistence-mongodb -q`
Expected: Compiles.

- [ ] **Step 5: Commit**

```
feat(#240): add percentComplete/statusNote to MongoDB, DTOs, context builder

MongoWorkItemDocument: from()/toDomain() mappings.
WorkItemResponse/WithAuditResponse: new fields in JSON.
WorkItemContextBuilder: new fields for filter rule evaluation.

Refs #240
```

---

### Task 10: Fix hardcoded status lists

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/filter/FilterRegistryEngine.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceGroupPolicy.java`
- Modify: `reports/src/main/java/io/casehub/work/reports/service/ReportService.java`

- [ ] **Step 1: Fix FilterRegistryEngine.toFilterEvent()**

Replace the REMOVE case:

```java
private FilterEvent toFilterEvent(final WorkEventType eventType) {
    return switch (eventType) {
        case CREATED -> FilterEvent.ADD;
        case COMPLETED, REJECTED, FAULTED, CANCELLED, OBSOLETE, EXPIRED, ESCALATED -> FilterEvent.REMOVE;
        default -> FilterEvent.UPDATE;
    };
}
```

- [ ] **Step 2: Fix MultiInstanceGroupPolicy.cancelRemainingChildren()**

Replace the method body:

```java
private void cancelRemainingChildren(final WorkItemSpawnGroup group) {
    workItemStore.findByParentIdExcludingStatuses(group.parentId,
            java.util.Arrays.stream(WorkItemStatus.values())
                    .filter(WorkItemStatus::isTerminal)
                    .toList())
            .forEach(child -> workItemService.cancelFromSystem(child.id, "system:multi-instance",
                    "threshold-met — cancelled by group policy"));
}
```

Key changes: (1) derive exclusion list from `isTerminal()`, (2) use `cancelFromSystem()` for race safety.

- [ ] **Step 3: Fix ReportService — all 4 status lists**

Replace all hardcoded status lists with derived versions:

```java
// Replace all activeStatuses definitions with:
final List<WorkItemStatus> activeStatuses = java.util.Arrays.stream(WorkItemStatus.values())
        .filter(WorkItemStatus::isActive).toList();

// Replace all terminalStatuses definitions with:
final List<WorkItemStatus> terminalStatuses = java.util.Arrays.stream(WorkItemStatus.values())
        .filter(WorkItemStatus::isTerminal).toList();
```

Apply at lines 62-63, 65, 220-222, and 370-371.

- [ ] **Step 4: Verify build compiles and existing tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,reports -q`
Expected: All pass.

- [ ] **Step 5: Commit**

```
fix(#240): replace hardcoded status lists with isTerminal()/isActive()

FilterRegistryEngine: FAULTED + OBSOLETE added to REMOVE case.
MultiInstanceGroupPolicy: isTerminal() filtering + cancelFromSystem().
ReportService: all 4 hardcoded lists now derived from enum methods.
Eliminates the fragility pattern that caused the original EXPIRED bug.

Refs #240
```

---

### Task 11: SLA_EXTENDED lifecycle event

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java`

- [ ] **Step 1: Fire lifecycle event in executeExtend()**

In `executeExtend()`, replace the comment `// No lifecycle event — deadline extension is not a status transition` with:

```java
fireLifecycleEvent("SLA_EXTENDED", item);
```

This resolves the inconsistency: actor-initiated `DEADLINE_EXTENDED` fires a lifecycle event but policy-driven `SLA_EXTENDED` did not.

- [ ] **Step 2: Verify tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=ExpiryLifecycleServiceTest -pl runtime -q`
Expected: All pass.

- [ ] **Step 3: Commit**

```
fix(#240): fire SLA_EXTENDED lifecycle event for policy-driven extensions

Resolves inconsistency: actor-initiated DEADLINE_EXTENDED fired a
lifecycle event but policy-driven SLA_EXTENDED was audit-only.
SSE subscribers now see all deadline changes.

Refs #240
```

---

### Task 12: Javadoc — DELEGATED cross-references

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java`

- [ ] **Step 1: Update DELEGATED javadoc**

Replace the DELEGATED javadoc:

```java
/** WorkItem has been forwarded to a named actor for acceptance — pre-acceptance hold.
 *  Non-terminal: the named actor must call {@code acceptDelegation()} or {@code declineDelegation()}.
 *  <p><strong>Cross-system semantics differ:</strong>
 *  {@code CommitmentState.DELEGATED} (casehub-qhorus) is <em>terminal</em> — obligation transferred
 *  and discharged. {@code PlanItemStatus.DELEGATED} (casehub-engine) means control passed to an
 *  external actor (broader). See {@code docs/LIFECYCLE.md} for cross-system DELEGATED semantics. */
DELEGATED,
```

- [ ] **Step 2: Commit**

```
docs(#240): add DELEGATED cross-system javadoc warning

WorkItemStatus.DELEGATED is non-terminal; CommitmentState.DELEGATED
(Qhorus) is terminal with opposite semantics. Cross-reference added
to prevent misapplication in integration code.

Refs #240
```

---

### Task 13: File cross-repo issues

- [ ] **Step 1: File casehub-engine issue**

```bash
gh issue create --repo casehubio/engine --title "fix: lifecycle alignment — adapter, applier, recovery, gate updates for FAULTED/OBSOLETE" --body "$(cat <<'BODY'
## Context

casehubio/work#240 adds FAULTED and OBSOLETE to WorkItemStatus. The engine work-adapter module needs matching updates.

## Changes Required

### WorkItemLifecycleAdapter.onWorkItemLifecycle() — use isTerminal()
Replace explicit status filter (lines 80-83) with `!status.isTerminal()`. The ESCALATED pre-check at line 75 already returns before this guard. Eliminates fragile status enumeration.

### PlanItemCompletionApplier.applyStatus() — two new cases + PlanItemFaultedEvent
- FAULTED → markFaulted() + fire PlanItemFaultedEvent
- OBSOLETE → markCancelled()
- EXPIRED → fire PlanItemFaultedEvent (pre-existing gap — parity with PlanItemRejectedEvent)

### HumanTaskRecoveryService — use isTerminal()
Replace `TERMINAL_STATUSES` EnumSet (line 53-58) with `workItem.status.isTerminal()`.

### ActionGateCompletionApplier — FAULTED/OBSOLETE mapping
- FAULTED → handleExpired() (system failure, no human decision)
- OBSOLETE → handleRejected() (context changed, action should not proceed)

### Javadoc fixes
- WorkItemLifecycleAdapter class javadoc: fix "ESCALATED is not terminal" (it IS terminal)
- PlanItemStatus.FAULTED: add "Includes timeout failures from WorkItem layer"
- PlanItemStatus.DELEGATED: add cross-system semantics warning

## Spec
See casehubio/work specs/2026-06-18-lifecycle-alignment-design.md Sections 9a-9f, 10, 11.

Refs casehubio/work#240
BODY
)"
```

- [ ] **Step 2: File casehub-parent issue**

```bash
gh issue create --repo casehubio/parent --title "docs: create LIFECYCLE.md — normative cross-system lifecycle reference" --body "$(cat <<'BODY'
## Context

casehubio/work#240 identifies the need for a single normative document covering all 6 lifecycle layers across the platform.

## Deliverables

### docs/LIFECYCLE.md (new)
1. Contract — normative; any repo adding lifecycle states must update it
2. Six lifecycle layers — state machines for WorkflowStatus, CaseStatus, PlanItemStatus, HumanTask, WorkItemStatus, CommitmentState
3. Cross-layer bridges — WorkItem→PlanItem mapping, Commitment→WorkItem parallels
4. Naming coherence — alignment and intentional divergences (DELEGATED 3 meanings)
5. Industry spec alignment — WS-HumanTask 1.1, OpenHumanTask, BPMN 2.0, CMMN 1.1
6. Known gaps — PlanItem SUSPENDED, COMPENSATING/COMPENSATED
7. Design rationale — FAULTED not ERROR, OBSOLETE at WorkItem only, EXPIRED→FAULTED

### PLATFORM.md updates
- Capability ownership table: 12 statuses (was 10)
- New lifecycle coherence protocol entry
- Audit mandate for isTerminal()/isActive() consumers

## Spec
See casehubio/work specs/2026-06-18-lifecycle-alignment-design.md Sections 12-13.

Refs casehubio/work#240
BODY
)"
```

- [ ] **Step 3: File casehub-qhorus issue**

```bash
gh issue create --repo casehubio/qhorus --title "docs: CommitmentState.DELEGATED cross-system javadoc warning" --body "$(cat <<'BODY'
CommitmentState.DELEGATED is terminal (obligation transferred). WorkItemStatus.DELEGATED (casehub-work) is non-terminal (pre-acceptance hold). Add cross-reference javadoc to prevent semantic confusion in integration code.

Refs casehubio/work#240
BODY
)"
```

- [ ] **Step 4: File casehub-engine PlanItemStatus.SUSPENDED issue**

```bash
gh issue create --repo casehubio/engine --title "gap: PlanItemStatus has no SUSPENDED — observability gap" --body "$(cat <<'BODY'
WorkItemStatus, CaseStatus, and WorkflowStatus all have SUSPENDED. PlanItemStatus does not. When a WorkItem is SUSPENDED, the PlanItem stays DELEGATED — the engine cannot distinguish active work from paused work.

Low priority — cosmetic for monitoring, does not affect execution correctness.

Refs casehubio/work#240
BODY
)"
```

- [ ] **Step 5: File MongoWorkItemDocument field sync issue**

```bash
gh issue create --repo casehubio/work --title "fix: MongoWorkItemDocument missing 13+ fields from WorkItem entity" --body "$(cat <<'BODY'
MongoWorkItemDocument is missing: version, accumulatedUnclaimedSeconds, lastReturnedToPoolAt, confidenceScore, callerRef, parentId, scope, templateId, permittedOutcomes, excludedUsers, outcome, inputDataSchema, outputDataSchema.

MongoDB-backed deployments silently lose this data on round-trip. Pre-existing debt — not introduced by #240.

Refs #240
BODY
)"
```

- [ ] **Step 6: Commit (no code changes — just record the issue numbers)**

Update the spec with filed issue numbers if needed.

---

### Task 14: Integration test verification

- [ ] **Step 1: Run full runtime module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`
Expected: All pass.

- [ ] **Step 2: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests -q`
Expected: All pass.

- [ ] **Step 3: Run reports module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl reports -q`
Expected: All pass.

- [ ] **Step 4: Run IntelliJ find-references audit on WorkItemStatus**

Use `mcp__intellij-index__ide_find_references` on `WorkItemStatus` to scan for any remaining hardcoded status lists not covered by this plan. If any found, fix them.

---

### Task 15: Update api-reference.md

**Files:**
- Modify: `docs/api-reference.md`

- [ ] **Step 1: Add new endpoints to API reference**

Add `PUT /workitems/{id}/fault`, `PUT /workitems/{id}/obsolete`, `PUT /workitems/{id}/progress` to the endpoint table with request/response documentation.

- [ ] **Step 2: Update WorkItemStatus documentation**

Add FAULTED and OBSOLETE to the status table with descriptions.

- [ ] **Step 3: Commit**

```
docs(#240): update api-reference.md with new lifecycle endpoints

fault, obsolete, progress endpoints. FAULTED and OBSOLETE status
descriptions. percentComplete and statusNote in response schema.

Refs #240
```
