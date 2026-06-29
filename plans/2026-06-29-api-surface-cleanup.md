# API Surface Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Clean up the casehub-work API surface — align the event hierarchy (#278), migrate SPIs to api/spi/ (#276), and consolidate template creation (#277).

**Architecture:** Three sequential refactorings on one branch. #278 reshapes the event model (WorkItemEvent becomes the primary typed interface, WorkLifecycleEvent is deleted). #276 moves 14 SPI interfaces to api/spi/. #277 unifies template creation through createFromTemplate().

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JPA/Hibernate, IntelliJ MCP for all moves/renames

**Spec:** `specs/issue-276-spi-migration-and-api-cleanup/2026-06-27-api-surface-cleanup-design.md`

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>` — never full build
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all moves, renames, and code navigation — never bash grep/find for classes
- After IntelliJ moves, run `mcp__intellij-index__ide_sync_files` before building
- Every commit references issues: `Refs casehubio/work#276, Refs casehubio/work#277, Refs casehubio/work#278`
- Breaking cross-repo callers is expected — engine#585 tracks the engine migration

---

### Task 1: #278 — Event Hierarchy Alignment

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/WorkItemEvent.java`
- Modify: `api/src/test/java/io/casehub/work/api/WorkItemEventTest.java`
- Modify: `api/src/test/java/io/casehub/work/api/WorkEventTypeTest.java`
- Modify: `api/src/main/java/io/casehub/work/api/NotificationPayload.java`
- Delete: `api/src/main/java/io/casehub/work/api/WorkLifecycleEvent.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEvent.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/event/WorkItemLifecycleEventTest.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/filter/FilterAction.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/action/ApplyLabelAction.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/action/SetPriorityAction.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/action/OverrideCandidateGroupsAction.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/action/ApplyLabelActionTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/action/SetPriorityActionTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/action/OverrideCandidateGroupsActionTest.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/filter/FilterRegistryEngine.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/filter/FilterRegistryEngineTest.java`
- Modify: `ai/src/main/java/io/casehub/work/ai/escalation/EscalationSummaryObserver.java`
- Modify: `notifications/src/main/java/io/casehub/work/notifications/service/NotificationDispatcher.java`
- Modify: `notifications/src/main/java/io/casehub/work/notifications/channel/SlackNotificationChannel.java`
- Modify: `notifications/src/main/java/io/casehub/work/notifications/channel/TeamsNotificationChannel.java`
- Modify: `notifications/src/main/java/io/casehub/work/notifications/channel/HttpWebhookChannel.java`

**Interfaces:**
- Produces: `WorkItemEvent` with `eventType()`, `occurredAt()`, `actor()`, `detail()` abstract methods
- Produces: `WorkItemLifecycleEvent.workItem(): WorkItem` (replaces `source(): Object`)
- Produces: `FilterAction.apply(WorkItem, Map<String, Object>)` (replaces `Object` parameter)
- Produces: `NotificationPayload(WorkItemEvent event, ...)` (replaces `WorkLifecycleEvent`)

#### Phase A — Enrich WorkItemEvent + fix api/ tests

- [ ] **Step 1: Add abstract methods to WorkItemEvent**

Add 4 new abstract methods to `api/src/main/java/io/casehub/work/api/WorkItemEvent.java`:

```java
package io.casehub.work.api;

import java.time.Instant;
import java.util.UUID;

public interface WorkItemEvent {

    WorkItemRef ref();

    WorkEventType eventType();

    Instant occurredAt();

    String actor();

    String detail();

    default UUID workItemId() { return ref().id(); }
    default WorkItemStatus status() { return ref().status(); }
    default String callerRef() { return ref().callerRef(); }
    default String assigneeId() { return ref().assigneeId(); }
    default String resolution() { return ref().resolution(); }
    default String candidateGroups() { return ref().candidateGroups(); }
    default String outcome() { return ref().outcome(); }
    default String tenancyId() { return ref().tenancyId(); }
}
```

- [ ] **Step 2: Fix WorkItemEventTest — lambda → anonymous impl**

The test used `WorkItemEvent` as a functional interface via `() -> ref`. With 5 abstract methods it's no longer functional. Rewrite both tests in `api/src/test/java/io/casehub/work/api/WorkItemEventTest.java`:

```java
package io.casehub.work.api;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.UUID;

import org.junit.jupiter.api.Test;

class WorkItemEventTest {

    private WorkItemEvent event(final WorkItemRef ref) {
        return new WorkItemEvent() {
            @Override public WorkItemRef ref() { return ref; }
            @Override public WorkEventType eventType() { return WorkEventType.CREATED; }
            @Override public Instant occurredAt() { return Instant.now(); }
            @Override public String actor() { return "test"; }
            @Override public String detail() { return null; }
        };
    }

    @Test
    void defaultMethods_delegateToRef() {
        final UUID id = UUID.randomUUID();
        final WorkItemRef ref = new WorkItemRef(id, WorkItemStatus.IN_PROGRESS, "caller-1",
                "bob", "{\"x\":1}", "team-b", "rejected", "tenant-2");

        final WorkItemEvent event = event(ref);

        assertThat(event.workItemId()).isEqualTo(id);
        assertThat(event.status()).isEqualTo(WorkItemStatus.IN_PROGRESS);
        assertThat(event.callerRef()).isEqualTo("caller-1");
        assertThat(event.assigneeId()).isEqualTo("bob");
        assertThat(event.resolution()).isEqualTo("{\"x\":1}");
        assertThat(event.candidateGroups()).isEqualTo("team-b");
        assertThat(event.outcome()).isEqualTo("rejected");
        assertThat(event.tenancyId()).isEqualTo("tenant-2");
    }

    @Test
    void defaultMethods_handleNullRefFields() {
        final WorkItemRef ref = new WorkItemRef(UUID.randomUUID(), WorkItemStatus.PENDING,
                null, null, null, null, null, null);

        final WorkItemEvent event = event(ref);

        assertThat(event.callerRef()).isNull();
        assertThat(event.assigneeId()).isNull();
        assertThat(event.resolution()).isNull();
    }
}
```

- [ ] **Step 3: Fix WorkEventTypeTest — WorkLifecycleEvent → WorkItemEvent**

In `api/src/test/java/io/casehub/work/api/WorkEventTypeTest.java`, replace the `concreteEvent_implementsAbstractMethods` test:

```java
@Test
void concreteEvent_implementsAbstractMethods() {
    final WorkItemRef ref = new WorkItemRef(UUID.randomUUID(), WorkItemStatus.PENDING,
            null, null, null, null, null, null);
    var event = new WorkItemEvent() {
        @Override public WorkItemRef ref() { return ref; }
        @Override public WorkEventType eventType() { return WorkEventType.CREATED; }
        @Override public java.time.Instant occurredAt() { return java.time.Instant.now(); }
        @Override public String actor() { return "test"; }
        @Override public String detail() { return null; }
    };
    assertThat(event.eventType()).isEqualTo(WorkEventType.CREATED);
    assertThat(event.ref()).isNotNull();
}
```

Remove the `WorkLifecycleEvent` import.

- [ ] **Step 4: Run api/ tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: all tests pass (WorkItemLifecycleEvent in runtime/ already has matching concrete methods for the new interface requirements)

#### Phase B — Type FilterAction + migrate observers

- [ ] **Step 5: Change FilterAction.apply() parameter from Object to WorkItem**

In `runtime/src/main/java/io/casehub/work/runtime/filter/FilterAction.java`:

```java
package io.casehub.work.runtime.filter;

import java.util.Map;

import io.casehub.work.runtime.model.WorkItem;

public interface FilterAction {
    String type();
    void apply(WorkItem workItem, Map<String, Object> params);
}
```

- [ ] **Step 6: Remove casts from all 3 FilterAction implementations**

`ApplyLabelAction.java` — change method signature to `apply(final WorkItem workItem, ...)`, remove `final WorkItem workItem = (WorkItem) workUnit;` line.

`SetPriorityAction.java` — same: change signature, remove cast line.

`OverrideCandidateGroupsAction.java` — same: change signature, remove cast line.

- [ ] **Step 7: Migrate FilterRegistryEngine — observer type + source() → workItem()**

In `runtime/src/main/java/io/casehub/work/runtime/filter/FilterRegistryEngine.java`:

1. Change import: `WorkLifecycleEvent` → `WorkItemLifecycleEvent`
2. `onLifecycleEvent(@Observes final WorkLifecycleEvent event)` → `onLifecycleEvent(@Observes final WorkItemLifecycleEvent event)`
3. `processEvent(final WorkLifecycleEvent event, ...)` → `processEvent(final WorkItemLifecycleEvent event, ...)`
4. `applyFilters(final WorkLifecycleEvent event, ...)` → `applyFilters(final WorkItemLifecycleEvent event, ...)`
5. In `applyFilters()`: `final Object workUnit = event.source();` → `final WorkItem workItem = event.workItem();`
6. In `applyFilters()`: `a.apply(workUnit, action.params())` → `a.apply(workItem, action.params())`

- [ ] **Step 8: Rewrite FilterRegistryEngineTest — WorkItem fixtures**

The test currently uses anonymous `WorkLifecycleEvent` with string sources. Rewrite with real `WorkItemLifecycleEvent.of()` requiring WorkItem entities:

```java
package io.casehub.work.runtime.filter;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.api.WorkEventType;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;

class FilterRegistryEngineTest {

    private FilterRegistryEngine engine;
    private List<WorkItem> capturedWorkItems;
    private FilterAction capturingAction;

    @BeforeEach
    void setUp() {
        capturedWorkItems = new ArrayList<>();
        capturingAction = new FilterAction() {
            @Override
            public String type() {
                return "CAPTURE";
            }

            @Override
            public void apply(final WorkItem workItem, final Map<String, Object> params) {
                capturedWorkItems.add(workItem);
            }
        };
        engine = new FilterRegistryEngine(new JexlConditionEvaluator(), List.of(capturingAction));
    }

    private WorkItemLifecycleEvent event(final WorkEventType type, final String category) {
        final WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.category = category;
        wi.status = io.casehub.work.api.WorkItemStatus.PENDING;
        return WorkItemLifecycleEvent.of(type.name(), wi, "test", null);
    }

    @Test
    void matchingCondition_actionFired() {
        final var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("CAPTURE", Map.of())));
        final var evt = event(WorkEventType.CREATED, "finance");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkItems).hasSize(1);
        assertThat(capturedWorkItems.get(0).category).isEqualTo("finance");
    }

    @Test
    void nonMatchingCondition_actionNotFired() {
        final var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'legal'", Map.of(),
                List.of(ActionDescriptor.of("CAPTURE", Map.of())));
        final var evt = event(WorkEventType.CREATED, "finance");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkItems).isEmpty();
    }

    @Test
    void disabledDefinition_actionNotFired() {
        final var def = FilterDefinition.onAdd("test", "desc", false,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("CAPTURE", Map.of())));
        final var evt = event(WorkEventType.CREATED, "finance");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkItems).isEmpty();
    }

    @Test
    void wrongEventType_actionNotFired() {
        final var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("CAPTURE", Map.of())));
        final var evt = event(WorkEventType.ASSIGNED, "finance");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkItems).isEmpty();
    }

    @Test
    void unknownActionType_ignoredGracefully() {
        final var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("UNKNOWN_ACTION", Map.of())));
        final var evt = event(WorkEventType.CREATED, "finance");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkItems).isEmpty();
    }

    @Test
    void reentrancyGuard_preventsRecursiveProcessing() {
        final int[] callCount = { 0 };
        final FilterAction selfFiringAction = new FilterAction() {
            @Override
            public String type() {
                return "SELF_FIRE";
            }

            @Override
            public void apply(final WorkItem workItem, final Map<String, Object> params) {
                callCount[0]++;
            }
        };
        engine = new FilterRegistryEngine(new JexlConditionEvaluator(), List.of(selfFiringAction));
        final var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("SELF_FIRE", Map.of())));
        final var evt = event(WorkEventType.CREATED, "finance");
        engine.processEvent(evt, List.of(def));
        assertThat(callCount[0]).isEqualTo(1);
    }
}
```

- [ ] **Step 9: Migrate EscalationSummaryObserver**

In `ai/src/main/java/io/casehub/work/ai/escalation/EscalationSummaryObserver.java`:

1. Change import: `WorkLifecycleEvent` → `WorkItemLifecycleEvent` (from `io.casehub.work.runtime.event`)
2. Observer: `void onEscalation(@Observes final WorkLifecycleEvent event)` → `void onEscalation(@Observes final WorkItemLifecycleEvent event)`
3. Body: replace `event.source() instanceof io.casehub.work.runtime.model.WorkItem wi` with `final WorkItem wi = event.workItem(); if (wi != null)`

- [ ] **Step 10: Update WorkItemLifecycleEvent — remove extends, source() → workItem()**

In `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEvent.java`:

1. Remove `extends WorkLifecycleEvent` from class declaration
2. Remove `import io.casehub.work.api.WorkLifecycleEvent;`
3. Rename method `source()` → `workItem()`, change return type `Object` → `WorkItem`
4. Update `fromWire()` Javadoc: replace "Callers must not invoke source() or context()" with "Callers must not invoke workItem() or context()"
5. Remove `@Override` from `eventType()`, `context()` (no longer overriding abstract class methods; `eventType()` now implements `WorkItemEvent` interface method — keep `@Override` on it)

- [ ] **Step 11: Fix WorkItemLifecycleEventTest**

In `runtime/src/test/java/io/casehub/work/runtime/event/WorkItemLifecycleEventTest.java`:

Update the test `of_sourceReturnsWorkItemEntity` → rename to `of_workItemReturnsEntity`, change `event.source()` → `event.workItem()`.

- [ ] **Step 12: Run runtime/ tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all tests pass

#### Phase C — NotificationPayload + channels + delete WorkLifecycleEvent

- [ ] **Step 13: Update NotificationPayload — WorkLifecycleEvent → WorkItemEvent**

In `api/src/main/java/io/casehub/work/api/NotificationPayload.java`:

Change record parameter from `WorkLifecycleEvent event` to `WorkItemEvent event`. Remove `WorkLifecycleEvent` reference entirely.

- [ ] **Step 14: Update NotificationDispatcher — source() → workItem()**

In `notifications/src/main/java/io/casehub/work/notifications/service/NotificationDispatcher.java`:

Line 73: `final WorkItem wi = (WorkItem) event.source();` → `final WorkItem wi = event.workItem();`

(The event parameter is already typed `WorkItemLifecycleEvent`, so `workItem()` returns `WorkItem` directly — no cast needed.)

- [ ] **Step 15: Update notification channels — cast + workItem()**

In each of `SlackNotificationChannel.java`, `TeamsNotificationChannel.java`, `HttpWebhookChannel.java`:

Change: `final WorkItem wi = (WorkItem) payload.event().source();`
To: `final WorkItem wi = ((WorkItemLifecycleEvent) payload.event()).workItem();`

Add import: `import io.casehub.work.runtime.event.WorkItemLifecycleEvent;`

- [ ] **Step 16: Delete WorkLifecycleEvent**

Delete `api/src/main/java/io/casehub/work/api/WorkLifecycleEvent.java`.

- [ ] **Step 17: Build all affected modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,ai,notifications`
Expected: all tests pass

- [ ] **Step 18: Commit**

```
feat(#278): align event hierarchy — promote WorkItemEvent, delete WorkLifecycleEvent

- Enrich WorkItemEvent with eventType(), occurredAt(), actor(), detail()
- Delete WorkLifecycleEvent abstract class
- Replace source():Object with typed workItem():WorkItem on WorkItemLifecycleEvent
- Type FilterAction.apply() parameter from Object to WorkItem
- Migrate observers: FilterRegistryEngine, EscalationSummaryObserver
- Update NotificationPayload field type to WorkItemEvent
- Breaking: engine#585 must migrate WorkItemLifecycleAdapter observer type

Refs casehubio/work#278
```

---

### Task 2: #276 — SPI Migration to api/spi/

**Files:**
- Move 14 files from `api/src/main/java/io/casehub/work/api/` to `api/src/main/java/io/casehub/work/api/spi/`

**Interfaces:**
- Consumes: clean api/ module from Task 1
- Produces: all consumer-facing SPIs at `io.casehub.work.api.spi`

- [ ] **Step 1: Move all 14 SPI interfaces using IntelliJ**

Use `mcp__intellij-index__ide_move_file` for each interface. Move destination: `api/src/main/java/io/casehub/work/api/spi`. The 14 interfaces:

1. BusinessCalendar
2. CapabilityRegistry
3. ClaimSlaPolicy
4. ExclusionPolicy
5. HolidayCalendar
6. InstanceAssignmentStrategy
7. NotificationChannel
8. SkillMatcher
9. SkillProfileProvider
10. SlaBreachPolicy
11. SpawnPort
12. WorkerRegistry
13. WorkerSelectionStrategy
14. WorkloadProvider

IntelliJ updates all imports across all modules automatically.

- [ ] **Step 2: Sync IDE and verify**

Run `mcp__intellij-index__ide_sync_files` after all moves.

- [ ] **Step 3: Build all modules that import SPIs**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,core,runtime,ai,notifications,queues,examples,flow,persistence-memory,persistence-mongodb`
Expected: all tests pass — imports updated by IntelliJ

- [ ] **Step 4: Commit**

```
refactor(#276): migrate 14 SPI interfaces to api/spi/ subpackage

Move consumer-facing SPIs from io.casehub.work.api to io.casehub.work.api.spi
per consumer-spi-placement protocol. Value types, events, exceptions, and
utilities stay in io.casehub.work.api flat.

FilterAction intentionally stays in runtime/filter/ — runtime-internal SPI
with WorkItem parameter type.

Refs casehubio/work#276
```

---

### Task 3: #277 — Template Consolidation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemTemplateResource.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemScheduleService.java`
- Modify: `examples/src/main/java/io/casehub/work/examples/formschema/FormSchemaScenario.java`
- Modify: 8 test classes in `runtime/src/test/java/`

**Interfaces:**
- Consumes: `WorkItemTemplateService.createFromTemplate(WorkItemCreateRequest)` — existing method
- Produces: consolidated single template creation path; `instantiate()` and `toCreateRequest()` deleted

- [ ] **Step 1: Fix audit gap in createFromTemplate()**

In `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`, after line 205 (`mergeRequestWithTemplate`) and before line 207 (multi-instance branch), add the audit detail logic:

```java
final WorkItemCreateRequest merged = mergeRequestWithTemplate(template, request, expandedExcludedUsers);

// Audit trail for excludedGroups expansion — mirrors the instantiate() pattern
if (expandedExcludedUsers != null && !expandedExcludedUsers.equals(template.excludedUsers)) {
    final String auditDetail = buildExpansionAuditDetail(template, expandedExcludedUsers);
    merged = merged.toBuilder().auditDetail(auditDetail).build();
}
```

Note: `merged` must change from `final` to non-final for the rebuild.

- [ ] **Step 2: Write test for audit gap fix**

Add a test in `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceCreateFromTemplateTest.java` (or the appropriate existing test class) verifying that `createFromTemplate()` produces an audit detail when excludedGroups expand:

```java
@Test
void createFromTemplate_excludedGroups_producesAuditDetail() {
    // Create template with excludedGroups that expand to new excludedUsers
    // Call createFromTemplate() with a request containing templateId
    // Verify the created WorkItem's audit entry contains expansion detail
}
```

(Exact test body depends on the existing test fixtures — follow the pattern in `WorkItemTemplateServiceResolutionTest.instantiate_excludedGroups_expandedIntoExcludedUsers`.)

- [ ] **Step 3: Migrate REST endpoint**

In `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemTemplateResource.java`, the `instantiate()` method (line 558–580):

Replace `templateService.instantiate(t, request.title(), request.assigneeId(), request.createdBy())` with:

```java
final var createRequest = WorkItemCreateRequest.builder()
        .templateId(id)
        .title(request.title())
        .assigneeId(request.assigneeId())
        .createdBy(request.createdBy())
        .build();
final var wi = templateService.createFromTemplate(createRequest);
```

The template lookup (`templateService.findById(id)`) can be removed since `createFromTemplate()` looks up the template internally by `request.templateId`. Simplify the endpoint to:

```java
try {
    final var createRequest = WorkItemCreateRequest.builder()
            .templateId(id)
            .title(request.title())
            .assigneeId(request.assigneeId())
            .createdBy(request.createdBy())
            .build();
    final var wi = templateService.createFromTemplate(createRequest);
    return Response.status(Response.Status.CREATED)
            .entity(WorkItemMapper.toResponse(wi)).build();
} catch (IllegalArgumentException e) {
    return Response.status(e.getMessage().startsWith("Template not found")
            ? Response.Status.NOT_FOUND : Response.Status.BAD_REQUEST)
            .entity(Map.of("error", e.getMessage())).build();
}
```

Note: `createFromTemplate()` throws `IllegalArgumentException("Template not found: ...")` when the template doesn't exist, so the 404 case is now an exception rather than an Optional.empty().

- [ ] **Step 4: Migrate WorkItemScheduleService**

In `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemScheduleService.java`, line 134:

Replace:
```java
templateService.instantiate(template, null, null, "schedule:" + scheduleId);
```

With:
```java
final var request = WorkItemCreateRequest.builder()
        .templateId(template.id)
        .createdBy("schedule:" + scheduleId)
        .build();
templateService.createFromTemplate(request);
```

The local `template` lookup (line 127) can remain since it's also used for the null check and logging.

- [ ] **Step 5: Migrate FormSchemaScenario examples**

In `examples/src/main/java/io/casehub/work/examples/formschema/FormSchemaScenario.java`, replace both `templateService.instantiate(template, title, assignee, createdBy)` calls with `WorkItemCreateRequest` builder + `createFromTemplate()`.

- [ ] **Step 6: Migrate test classes**

Migrate all 8 test classes (~17 call sites) from `templateService.instantiate(t, null, null, "test")` to:

```java
final var request = WorkItemCreateRequest.builder()
        .templateId(t.id)
        .createdBy("test")
        .build();
templateService.createFromTemplate(request);
```

For tests that pass title/assignee overrides, add those to the builder.

Test classes: WorkItemInstancesResourceTest, MultiInstanceInboxTest, MultiInstanceCoordinatorTest, WorkItemGroupLifecycleEventTest, MultiInstanceCreateTest, WorkItemTemplateInstantiateTest, WorkItemTemplateServiceResolutionTest, MultiInstanceClaimGuardTest.

- [ ] **Step 7: Delete instantiate() and toCreateRequest() methods**

In `WorkItemTemplateService.java`, delete:
- `instantiate(WorkItemTemplate, String, String, String)` (4-arg)
- `instantiate(WorkItemTemplate, String, String, String, String)` (5-arg)
- `instantiate(WorkItemTemplate, String, String, String, String, String)` (6-arg)
- `toCreateRequest(WorkItemTemplate, String, String, String)` (4-arg)
- `toCreateRequest(WorkItemTemplate, String, String, String, String)` (5-arg)
- `toCreateRequest(WorkItemTemplate, String, String, String, String, String)` (6-arg)

- [ ] **Step 8: Build and test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,examples`
Expected: all tests pass

- [ ] **Step 9: Commit**

```
refactor(#277): consolidate template creation through createFromTemplate()

Delete 3 instantiate() overloads and 3 toCreateRequest() statics.
All callers migrated to createFromTemplate(WorkItemCreateRequest):
- REST endpoint, schedule service, examples, 8 test classes
Fix audit gap: createFromTemplate() now records excludedGroups expansion detail.

Refs casehubio/work#277
```

---

## Post-Implementation

- [ ] Run full integration tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests`
- [ ] Code review via `superpowers:requesting-code-review`
- [ ] Doc sync via `implementation-doc-sync`
