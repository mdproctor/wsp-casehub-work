# InboundWorkItemScheduler + WorkActorStateContributor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #397 — Implement InboundWorkItemScheduler in work-engine-adapter
**Issue group:** #397, #398

**Goal:** Provide the real implementations of `InboundWorkItemScheduler` (engine#974)
and `WorkActorStateContributor` (relocated from engine-actor-state) in
`casehub-work-engine-adapter`.

**Architecture:** Both are CDI beans in `io.casehub.work.engine` that implement
engine/platform SPIs. The scheduler converts `InboundWorkItemRequest` to
`WorkItemCreateRequest` and creates via `WorkItemCreator` inside tenant context.
The contributor queries `WorkItemStore` for active items by assignee and feeds
them to `ActorStateAccumulator`.

**Tech Stack:** Java 21, Quarkus CDI, JUnit 5, Mockito

## Global Constraints

- No POM changes — all dependencies already satisfied
- No Flyway migrations — no schema changes
- Package: `io.casehub.work.engine` (same as existing adapter classes)
- Module: `engine-adapter`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter`
- Test timeout: 120s (per scripts/ convention)

---

## Batch 1: InboundWorkItemScheduler — inbound message → WorkItem creation

### Task 1: InboundWorkItemSchedulerImpl (TDD)

**Files:**
- Create: `engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java`
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/InboundWorkItemSchedulerImplTest.java`

**Interfaces:**
- Consumes: `InboundWorkItemScheduler` (engine-common SPI), `InboundWorkItemRequest` (engine-common record), `WorkItemCreator` (work-api SPI), `TenantContextExecutor` (work-api SPI), `WorkItemCreateRequest` (work-api), `WorkItemPriority` (work-api enum)
- Produces: `InboundWorkItemSchedulerImpl` — `@ApplicationScoped` bean overriding `NoOpInboundWorkItemScheduler` (`@DefaultBean`) when engine-adapter is on the classpath

- [ ] **Step 1: Write the test class with all test methods**

Create `engine-adapter/src/test/java/io/casehub/work/engine/InboundWorkItemSchedulerImplTest.java`:

```java
package io.casehub.work.engine;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.doAnswer;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;

import io.casehub.engine.common.spi.InboundWorkItemRequest;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.api.WorkItemRef;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.api.spi.TenantContextExecutor;
import io.casehub.work.api.spi.WorkItemCreator;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicReference;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

class InboundWorkItemSchedulerImplTest {

  private WorkItemCreator creator;
  private TenantContextExecutor tenantContext;
  private InboundWorkItemSchedulerImpl scheduler;

  @BeforeEach
  void setUp() {
    creator = mock(WorkItemCreator.class);
    tenantContext = mock(TenantContextExecutor.class);
    doAnswer(inv -> {
      inv.getArgument(1, Runnable.class).run();
      return null;
    }).when(tenantContext).runInTenantContext(any(), any());
    scheduler = new InboundWorkItemSchedulerImpl(creator, tenantContext);
  }

  @Test
  void schedule_mapsAllFieldsCorrectly() {
    var expires = Instant.now().plusSeconds(3600);
    var request = InboundWorkItemRequest.builder()
        .title("Review document")
        .description("Please review the attached document")
        .candidateGroups("reviewers")
        .candidateUsers("alice,bob")
        .callerRef("case:3fa85f64-5717-4562-b3fc-2c963f66afa6/pi:step-1")
        .scope("investigations")
        .payload("{\"docId\":\"doc-123\"}")
        .tenancyId("tenant-abc")
        .createdBy("casehub-engine-inbound")
        .priority("HIGH")
        .types(List.of("review", "document"))
        .expiresAt(expires)
        .build();

    var ref = new WorkItemRef(UUID.randomUUID(), WorkItemStatus.PENDING,
        null, null, null, null, null, "tenant-abc", null, null, null, null);
    org.mockito.Mockito.when(creator.create(any())).thenReturn(ref);

    scheduler.schedule(request);

    var captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(creator).create(captor.capture());
    var cr = captor.getValue();

    assertThat(cr.title).isEqualTo("Review document");
    assertThat(cr.description).isEqualTo("Please review the attached document");
    assertThat(cr.candidateGroups).isEqualTo("reviewers");
    assertThat(cr.candidateUsers).isEqualTo("alice,bob");
    assertThat(cr.callerRef).isEqualTo("case:3fa85f64-5717-4562-b3fc-2c963f66afa6/pi:step-1");
    assertThat(cr.scope).isEqualTo("investigations");
    assertThat(cr.payload).isEqualTo("{\"docId\":\"doc-123\"}");
    assertThat(cr.tenancyId).isEqualTo("tenant-abc");
    assertThat(cr.createdBy).isEqualTo("casehub-engine-inbound");
    assertThat(cr.priority).isEqualTo(WorkItemPriority.HIGH);
    assertThat(cr.types).containsExactly("review", "document");
    assertThat(cr.expiresAt).isEqualTo(expires);
  }

  @Test
  void schedule_nullPriority_passesNullToRequest() {
    var request = InboundWorkItemRequest.builder()
        .title("Simple task")
        .tenancyId("tenant-abc")
        .build();

    scheduler.schedule(request);

    var captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(creator).create(captor.capture());
    assertThat(captor.getValue().priority).isNull();
  }

  @Test
  void schedule_invalidPriority_throwsIllegalArgument() {
    var request = InboundWorkItemRequest.builder()
        .title("Bad priority")
        .tenancyId("tenant-abc")
        .priority("BOGUS")
        .build();

    assertThatThrownBy(() -> scheduler.schedule(request))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void schedule_nullOptionalFields_passesNullsThrough() {
    var request = InboundWorkItemRequest.builder()
        .title("Minimal")
        .tenancyId("tenant-abc")
        .build();

    scheduler.schedule(request);

    var captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(creator).create(captor.capture());
    var cr = captor.getValue();

    assertThat(cr.description).isNull();
    assertThat(cr.candidateGroups).isNull();
    assertThat(cr.candidateUsers).isNull();
    assertThat(cr.callerRef).isNull();
    assertThat(cr.scope).isNull();
    assertThat(cr.payload).isNull();
    assertThat(cr.createdBy).isNull();
    assertThat(cr.types).isNull();
    assertThat(cr.expiresAt).isNull();
  }

  @Test
  void schedule_executeInsideTenantContext() {
    var request = InboundWorkItemRequest.builder()
        .title("Tenant check")
        .tenancyId("tenant-xyz")
        .build();

    var capturedTenancyId = new AtomicReference<String>();
    doAnswer(inv -> {
      capturedTenancyId.set(inv.getArgument(0, String.class));
      inv.getArgument(1, Runnable.class).run();
      return null;
    }).when(tenantContext).runInTenantContext(any(), any());

    scheduler.schedule(request);

    assertThat(capturedTenancyId.get()).isEqualTo("tenant-xyz");
    verify(creator).create(any());
  }

  @Test
  void schedule_eachValidPriority_parsesCorrectly() {
    for (WorkItemPriority p : WorkItemPriority.values()) {
      var request = InboundWorkItemRequest.builder()
          .title("Priority " + p.name())
          .tenancyId("tenant-abc")
          .priority(p.name())
          .build();

      scheduler.schedule(request);
    }

    verify(creator, org.mockito.Mockito.times(WorkItemPriority.values().length))
        .create(any());
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=InboundWorkItemSchedulerImplTest`
Expected: FAIL — `InboundWorkItemSchedulerImpl` class not found

- [ ] **Step 3: Implement InboundWorkItemSchedulerImpl**

Create `engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java`:

```java
package io.casehub.work.engine;

import io.casehub.engine.common.spi.InboundWorkItemRequest;
import io.casehub.engine.common.spi.InboundWorkItemScheduler;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.api.spi.TenantContextExecutor;
import io.casehub.work.api.spi.WorkItemCreator;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class InboundWorkItemSchedulerImpl implements InboundWorkItemScheduler {

  @Inject WorkItemCreator workItemCreator;
  @Inject TenantContextExecutor tenantContextExecutor;

  InboundWorkItemSchedulerImpl() {}

  InboundWorkItemSchedulerImpl(
      final WorkItemCreator workItemCreator,
      final TenantContextExecutor tenantContextExecutor) {
    this.workItemCreator = workItemCreator;
    this.tenantContextExecutor = tenantContextExecutor;
  }

  @Override
  public void schedule(final InboundWorkItemRequest request) {
    final WorkItemCreateRequest createRequest = WorkItemCreateRequest.builder()
        .title(request.title())
        .description(request.description())
        .candidateGroups(request.candidateGroups())
        .candidateUsers(request.candidateUsers())
        .callerRef(request.callerRef())
        .scope(request.scope())
        .payload(request.payload())
        .tenancyId(request.tenancyId())
        .createdBy(request.createdBy())
        .priority(request.priority() != null
            ? WorkItemPriority.valueOf(request.priority()) : null)
        .types(request.types())
        .expiresAt(request.expiresAt())
        .build();

    tenantContextExecutor.runInTenantContext(
        request.tenancyId(), () -> workItemCreator.create(createRequest));
  }
}
```

- [ ] **Step 4: Run test to verify all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=InboundWorkItemSchedulerImplTest`
Expected: all 6 tests PASS

- [ ] **Step 5: Run full engine-adapter test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter`
Expected: all existing tests still pass (no regression)

- [ ] **Step 6: Verify with ide_diagnostics**

Run `ide_diagnostics` on `engine-adapter` to confirm no compilation errors.

- [ ] **Step 7: Commit**

```bash
git -C "$PROJECT" add engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java engine-adapter/src/test/java/io/casehub/work/engine/InboundWorkItemSchedulerImplTest.java
git -C "$PROJECT" commit -m "feat(#397): implement InboundWorkItemScheduler in engine-adapter

Converts InboundWorkItemRequest to WorkItemCreateRequest and creates
via WorkItemCreator inside tenant context. Overrides the engine's
NoOpInboundWorkItemScheduler (@DefaultBean) when engine-adapter is
on the classpath.

Refs #397 Refs engine#974"
```

---

## Batch 2: WorkActorStateContributor — actor state view work-item data

### Task 2: WorkActorStateContributor (TDD)

**Files:**
- Create: `engine-adapter/src/main/java/io/casehub/work/engine/WorkActorStateContributor.java`
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/WorkActorStateContributorTest.java`

**Interfaces:**
- Consumes: `ActorStateContributor` (platform-api SPI), `ActorStateAccumulator` (platform-api), `WorkItemStore` (work-api SPI), `WorkItemQuery` (work-api), `WorkItem` (work-api record), `WorkItemStatus` (work-api enum), `CallerRef` (engine-adapter — same module)
- Produces: `WorkActorStateContributor` — `@ApplicationScoped` bean discovered by `ActorStateAggregator` via `Instance<ActorStateContributor>`

- [ ] **Step 1: Write the test class with all test methods**

Create `engine-adapter/src/test/java/io/casehub/work/engine/WorkActorStateContributorTest.java`:

```java
package io.casehub.work.engine;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import io.casehub.platform.api.actor.ActorStateAccumulator;
import io.casehub.work.api.WorkItem;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.api.WorkItemQuery;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.api.spi.WorkItemStore;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

class WorkActorStateContributorTest {

  private WorkItemStore store;
  private ActorStateAccumulator accumulator;
  private WorkActorStateContributor contributor;

  @BeforeEach
  void setUp() {
    store = mock(WorkItemStore.class);
    accumulator = mock(ActorStateAccumulator.class);
    contributor = new WorkActorStateContributor(store);
  }

  @Test
  void sourceName_returnsWork() {
    assertThat(contributor.sourceName()).isEqualTo("work");
  }

  @Test
  void contribute_activeItems_callsAccumulatorForEach() {
    var id1 = UUID.randomUUID();
    var id2 = UUID.randomUUID();
    var caseId = UUID.fromString("3fa85f64-5717-4562-b3fc-2c963f66afa6");

    var item1 = workItem(id1, "Review SAR", WorkItemStatus.ASSIGNED,
        "case:" + caseId + "/pi:step-1");
    var item2 = workItem(id2, "Approve transfer", WorkItemStatus.IN_PROGRESS,
        "case:" + caseId + "/gate:42");

    when(store.scan(any(WorkItemQuery.class))).thenReturn(List.of(item1, item2));

    contributor.contribute("agent-x", accumulator);

    var idCaptor = ArgumentCaptor.forClass(UUID.class);
    var titleCaptor = ArgumentCaptor.forClass(String.class);
    var statusCaptor = ArgumentCaptor.forClass(String.class);
    var categoryCaptor = ArgumentCaptor.forClass(String.class);
    var caseIdCaptor = ArgumentCaptor.forClass(UUID.class);

    verify(accumulator, org.mockito.Mockito.times(2)).workItem(
        idCaptor.capture(), titleCaptor.capture(), statusCaptor.capture(),
        categoryCaptor.capture(), caseIdCaptor.capture());

    assertThat(idCaptor.getAllValues()).containsExactly(id1, id2);
    assertThat(titleCaptor.getAllValues()).containsExactly("Review SAR", "Approve transfer");
    assertThat(statusCaptor.getAllValues()).containsExactly("ASSIGNED", "IN_PROGRESS");
    assertThat(caseIdCaptor.getAllValues()).containsExactly(caseId, caseId);
  }

  @Test
  void contribute_planItemCallerRef_extractsCaseId() {
    var id = UUID.randomUUID();
    var caseId = UUID.fromString("11111111-1111-1111-1111-111111111111");
    var item = workItem(id, "Task", WorkItemStatus.ASSIGNED,
        "case:" + caseId + "/pi:plan-item-1");

    when(store.scan(any())).thenReturn(List.of(item));
    contributor.contribute("agent-x", accumulator);

    verify(accumulator).workItem(id, "Task", "ASSIGNED", null, caseId);
  }

  @Test
  void contribute_gateCallerRef_extractsCaseId() {
    var id = UUID.randomUUID();
    var caseId = UUID.fromString("22222222-2222-2222-2222-222222222222");
    var item = workItem(id, "Gate", WorkItemStatus.IN_PROGRESS,
        "case:" + caseId + "/gate:99");

    when(store.scan(any())).thenReturn(List.of(item));
    contributor.contribute("agent-x", accumulator);

    verify(accumulator).workItem(id, "Gate", "IN_PROGRESS", null, caseId);
  }

  @Test
  void contribute_nonEngineCallerRef_caseIdIsNull() {
    var id = UUID.randomUUID();
    var item = workItem(id, "Qhorus task", WorkItemStatus.ASSIGNED,
        "qhorus:channel-1/msg-1/corr-1");

    when(store.scan(any())).thenReturn(List.of(item));
    contributor.contribute("agent-x", accumulator);

    verify(accumulator).workItem(id, "Qhorus task", "ASSIGNED", null, null);
  }

  @Test
  void contribute_nullCallerRef_caseIdIsNull() {
    var id = UUID.randomUUID();
    var item = workItem(id, "Manual task", WorkItemStatus.SUSPENDED, null);

    when(store.scan(any())).thenReturn(List.of(item));
    contributor.contribute("agent-x", accumulator);

    verify(accumulator).workItem(id, "Manual task", "SUSPENDED", null, null);
  }

  @Test
  void contribute_noActiveItems_noAccumulatorCalls() {
    when(store.scan(any())).thenReturn(List.of());
    contributor.contribute("agent-x", accumulator);

    verify(accumulator, never()).workItem(any(), any(), any(), any(), any());
  }

  @Test
  void contribute_queryUsesCorrectStatusFilter() {
    when(store.scan(any())).thenReturn(List.of());

    contributor.contribute("agent-x", accumulator);

    var captor = ArgumentCaptor.forClass(WorkItemQuery.class);
    verify(store).scan(captor.capture());
    var query = captor.getValue();

    assertThat(query.assigneeId()).isEqualTo("agent-x");
    assertThat(query.statusIn()).containsExactlyInAnyOrder(
        WorkItemStatus.ASSIGNED,
        WorkItemStatus.IN_PROGRESS,
        WorkItemStatus.SUSPENDED);
  }

  @Test
  void contribute_nullTitle_passedThrough() {
    var id = UUID.randomUUID();
    var item = workItem(id, null, WorkItemStatus.ASSIGNED, null);

    when(store.scan(any())).thenReturn(List.of(item));
    contributor.contribute("agent-x", accumulator);

    verify(accumulator).workItem(id, null, "ASSIGNED", null, null);
  }

  private WorkItem workItem(UUID id, String title, WorkItemStatus status, String callerRef) {
    return WorkItem.builder()
        .id(id)
        .title(title)
        .status(status)
        .callerRef(callerRef)
        .tenancyId("tenant-abc")
        .build();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=WorkActorStateContributorTest`
Expected: FAIL — `WorkActorStateContributor` class not found

- [ ] **Step 3: Implement WorkActorStateContributor**

Create `engine-adapter/src/main/java/io/casehub/work/engine/WorkActorStateContributor.java`:

```java
package io.casehub.work.engine;

import io.casehub.platform.api.actor.ActorStateAccumulator;
import io.casehub.platform.api.actor.ActorStateContributor;
import io.casehub.work.api.WorkItem;
import io.casehub.work.api.WorkItemQuery;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.api.spi.WorkItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class WorkActorStateContributor implements ActorStateContributor {

  @Inject WorkItemStore workItemStore;

  WorkActorStateContributor() {}

  WorkActorStateContributor(final WorkItemStore workItemStore) {
    this.workItemStore = workItemStore;
  }

  @Override
  public String sourceName() {
    return "work";
  }

  @Override
  public void contribute(final String actorId, final ActorStateAccumulator acc) {
    final List<WorkItem> items = workItemStore.scan(
        WorkItemQuery.builder()
            .assigneeId(actorId)
            .statusIn(List.of(
                WorkItemStatus.ASSIGNED,
                WorkItemStatus.IN_PROGRESS,
                WorkItemStatus.SUSPENDED))
            .build());

    for (final WorkItem wi : items) {
      final CallerRef ref = CallerRef.parse(wi.callerRef());
      final UUID caseId = ref != null ? ref.caseId() : null;
      acc.workItem(
          wi.id(),
          wi.title(),
          wi.status() != null ? wi.status().name() : null,
          null,
          caseId);
    }
  }
}
```

- [ ] **Step 4: Run test to verify all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=WorkActorStateContributorTest`
Expected: all 8 tests PASS

- [ ] **Step 5: Run full engine-adapter test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter`
Expected: all tests pass (including Task 1's tests)

- [ ] **Step 6: Verify with ide_diagnostics**

Run `ide_diagnostics` on `engine-adapter` to confirm no compilation errors.

- [ ] **Step 7: Commit**

```bash
git -C "$PROJECT" add engine-adapter/src/main/java/io/casehub/work/engine/WorkActorStateContributor.java engine-adapter/src/test/java/io/casehub/work/engine/WorkActorStateContributorTest.java
git -C "$PROJECT" commit -m "feat(#398): relocate WorkActorStateContributor to engine-adapter

Implements ActorStateContributor (platform-api) — queries WorkItemStore
for active items (ASSIGNED, IN_PROGRESS, SUSPENDED) by assignee and
contributes them to the actor state view. Uses CallerRef.parse() for
caseId extraction. Relocated from casehub-engine-actor-state.

Refs #398 Refs engine#974"
```

---

### Task 3: Documentation updates

**Files:**
- Modify: `CLAUDE.md` — update `## engine-adapter Module` section
- Modify: `ARC42STORIES.MD` — add layer entries for both implementations

**Interfaces:**
- Consumes: Task 1 and Task 2 deliverables
- Produces: Updated project documentation

- [ ] **Step 1: Update CLAUDE.md engine-adapter section**

Add to the `## engine-adapter Module` section in CLAUDE.md, after the existing
`WorkItemLifecycleAdapter` documentation:

```markdown
**Inbound work item creation (engine#974):** `InboundWorkItemSchedulerImpl` implements
`InboundWorkItemScheduler` SPI (engine-common). Converts `InboundWorkItemRequest` to
`WorkItemCreateRequest`, wraps creation in `TenantContextExecutor.runInTenantContext()`
(request arrives outside request scope from qhorus afterCompletion callback). Overrides
engine's `NoOpInboundWorkItemScheduler` (`@DefaultBean`) when engine-adapter is on the
classpath. Priority mapping: String → `WorkItemPriority.valueOf()`, null-safe, fail-fast
on invalid values.

**Actor state contribution (engine#974):** `WorkActorStateContributor` implements
`ActorStateContributor` (platform-api). Queries `WorkItemStore` for active items
(ASSIGNED, IN_PROGRESS, SUSPENDED) by `assigneeId`. Extracts `caseId` from `callerRef`
via `CallerRef.parse()` (same module — handles both PlanItemRef and GateRef formats).
`sourceName()` returns `"work"`. Relocated from `casehub-engine-actor-state`.
```

- [ ] **Step 2: Update ARC42STORIES.MD**

Add a layer entry in §9.4 for the engine#974 work:

```markdown
### L[N]: Inbound Scheduler + Actor State Contributor (engine#974)

**Issue:** #397, #398 | **Date:** 2026-09-06

Two SPI implementations relocated/created in engine-adapter as part of
engine#974 (circular dependency fix):

- `InboundWorkItemSchedulerImpl` — bridges inbound qhorus messages to
  WorkItem creation via `WorkItemCreator` + `TenantContextExecutor`
- `WorkActorStateContributor` — contributes active work items to the
  actor state view via `WorkItemStore` + `CallerRef.parse()`

**Key files:**
- `engine-adapter/.../InboundWorkItemSchedulerImpl.java`
- `engine-adapter/.../WorkActorStateContributor.java`

**Wiring:** Both are `@ApplicationScoped` CDI beans. The scheduler overrides
`NoOpInboundWorkItemScheduler` (`@DefaultBean`). The contributor is discovered
by `ActorStateAggregator` via `Instance<ActorStateContributor>`.

**Gotchas:** The scheduler must wrap creation in `TenantContextExecutor` because
it runs from a qhorus afterCompletion callback (no request scope). See #399 for
the planned self-sufficient service refactoring.
```

- [ ] **Step 3: Commit documentation**

```bash
git -C "$PROJECT" add CLAUDE.md ARC42STORIES.MD
git -C "$PROJECT" commit -m "docs(#397): document InboundWorkItemScheduler + WorkActorStateContributor in engine-adapter

Refs #397 Refs #398 Refs engine#974"
```

## References

- [2026-09-06-inbound-scheduler-actor-state-design.md] — design spec this plan implements
- [engine-common InboundWorkItemScheduler.java] — SPI interface
- [engine-common InboundWorkItemRequest.java] — request record
- [engine-runtime NoOpInboundWorkItemScheduler.java] — @DefaultBean fallback
- [engine-inbound InboundWorkItemBridge.java] — caller (afterCompletion callback)
- [platform-api ActorStateContributor/ActorStateAccumulator] — contributor SPI
- [engine-adapter CallerRef.java] — caseId extraction
- [engine-adapter HumanTaskScheduleHandler.java] — pattern reference
- [Protocol PP-20260609-fb6563] — async-event-tenant-context-propagation
- [Protocol PP-20260609-bdac7e] — store-tenancy-stamping-on-insert
- [#399] — self-sufficient tenant context (deferred)
