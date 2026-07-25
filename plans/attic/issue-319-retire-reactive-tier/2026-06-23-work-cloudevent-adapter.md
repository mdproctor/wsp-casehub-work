# WorkCloudEventAdapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bridge WorkItem lifecycle events to CloudEvents via a dual-channel emitter and canonical adapter, fixing QueueDashboard's missing non-terminal events.

**Architecture:** Extract `WorkItemLifecycleEmitter` and `WorkItemGroupLifecycleEmitter` beans that structurally enforce dual-channel CDI firing (`fire()` + `fireAsync()`). Migrate all 5 event-producing classes to use the emitters. Add `WorkCloudEventAdapter` following the canonical 6-rule CloudEvent adapter pattern (GE-20260621-629712).

**Tech Stack:** Java 21, Quarkus 3.32, CDI `Event<T>`, CloudEvents Java SDK (`cloudevents-core` 4.0.1, already transitive), Jackson `ObjectMapper`

**Spec:** `specs/issue-273-work-cloudevent-adapter/2026-06-23-work-cloudevent-adapter-design.md`

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=<TestClass>`

**Full build (after all tasks):** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

---

### Task 1: WorkItemLifecycleEmitter — test + implementation

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/event/WorkItemLifecycleEmitterTest.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEmitter.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.work.runtime.event;

import static org.mockito.Mockito.*;

import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import jakarta.enterprise.event.Event;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;

@ExtendWith(MockitoExtension.class)
class WorkItemLifecycleEmitterTest {

    @Mock
    Event<WorkItemLifecycleEvent> delegate;

    private WorkItemLifecycleEmitter emitter;

    @BeforeEach
    void setUp() {
        emitter = new WorkItemLifecycleEmitter();
        emitter.delegate = delegate;
    }

    @Test
    void emit_callsBothFireAndFireAsync() {
        when(delegate.fireAsync(any())).thenReturn(CompletableFuture.completedFuture(null));
        final WorkItemLifecycleEvent event = sampleEvent("CREATED");

        emitter.emit(event);

        verify(delegate).fire(event);
        verify(delegate).fireAsync(event);
    }

    @Test
    void emit_fireAsyncFailure_loggedNotThrown() {
        final CompletableFuture<WorkItemLifecycleEvent> failed = new CompletableFuture<>();
        failed.completeExceptionally(new RuntimeException("observer failure"));
        when(delegate.fireAsync(any())).thenReturn(failed);

        final WorkItemLifecycleEvent event = sampleEvent("COMPLETED");

        emitter.emit(event);

        verify(delegate).fire(event);
        verify(delegate).fireAsync(event);
    }

    private WorkItemLifecycleEvent sampleEvent(final String name) {
        final WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.status = WorkItemStatus.PENDING;
        return WorkItemLifecycleEvent.of(name, wi, "test", null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemLifecycleEmitterTest`
Expected: FAIL — `WorkItemLifecycleEmitter` class does not exist

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.work.runtime.event;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

@ApplicationScoped
public class WorkItemLifecycleEmitter {

    private static final Logger LOG = Logger.getLogger(WorkItemLifecycleEmitter.class);

    @Inject
    Event<WorkItemLifecycleEvent> delegate;

    public void emit(final WorkItemLifecycleEvent event) {
        delegate.fire(event);
        delegate.fireAsync(event)
                .exceptionally(ex -> {
                    LOG.warnf(ex, "Async lifecycle observer failure for %s", event.type());
                    return null;
                });
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemLifecycleEmitterTest`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEmitter.java runtime/src/test/java/io/casehub/work/runtime/event/WorkItemLifecycleEmitterTest.java
git commit -m "feat(#273): WorkItemLifecycleEmitter — structural dual-channel CDI firing

Refs #273"
```

---

### Task 2: WorkItemGroupLifecycleEmitter — test + implementation

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/event/WorkItemGroupLifecycleEmitterTest.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemGroupLifecycleEmitter.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.work.runtime.event;

import static org.mockito.Mockito.*;

import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import jakarta.enterprise.event.Event;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import io.casehub.work.api.GroupStatus;
import io.casehub.work.api.WorkItemGroupLifecycleEvent;

@ExtendWith(MockitoExtension.class)
class WorkItemGroupLifecycleEmitterTest {

    @Mock
    Event<WorkItemGroupLifecycleEvent> delegate;

    private WorkItemGroupLifecycleEmitter emitter;

    @BeforeEach
    void setUp() {
        emitter = new WorkItemGroupLifecycleEmitter();
        emitter.delegate = delegate;
    }

    @Test
    void emit_callsBothFireAndFireAsync() {
        when(delegate.fireAsync(any())).thenReturn(CompletableFuture.completedFuture(null));
        final WorkItemGroupLifecycleEvent event = sampleGroupEvent();

        emitter.emit(event);

        verify(delegate).fire(event);
        verify(delegate).fireAsync(event);
    }

    @Test
    void emit_fireAsyncFailure_loggedNotThrown() {
        final CompletableFuture<WorkItemGroupLifecycleEvent> failed = new CompletableFuture<>();
        failed.completeExceptionally(new RuntimeException("observer failure"));
        when(delegate.fireAsync(any())).thenReturn(failed);

        final WorkItemGroupLifecycleEvent event = sampleGroupEvent();

        emitter.emit(event);

        verify(delegate).fire(event);
        verify(delegate).fireAsync(event);
    }

    private WorkItemGroupLifecycleEvent sampleGroupEvent() {
        return WorkItemGroupLifecycleEvent.of(
                UUID.randomUUID(), UUID.randomUUID(),
                3, 2, 1, 0,
                GroupStatus.IN_PROGRESS, "caller-ref", "tenant-1");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemGroupLifecycleEmitterTest`
Expected: FAIL — `WorkItemGroupLifecycleEmitter` class does not exist

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.work.runtime.event;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.work.api.WorkItemGroupLifecycleEvent;

@ApplicationScoped
public class WorkItemGroupLifecycleEmitter {

    private static final Logger LOG = Logger.getLogger(WorkItemGroupLifecycleEmitter.class);

    @Inject
    Event<WorkItemGroupLifecycleEvent> delegate;

    public void emit(final WorkItemGroupLifecycleEvent event) {
        delegate.fire(event);
        delegate.fireAsync(event)
                .exceptionally(ex -> {
                    LOG.warnf(ex, "Async group lifecycle observer failure for group %s", event.groupId());
                    return null;
                });
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemGroupLifecycleEmitterTest`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/event/WorkItemGroupLifecycleEmitter.java runtime/src/test/java/io/casehub/work/runtime/event/WorkItemGroupLifecycleEmitterTest.java
git commit -m "feat(#273): WorkItemGroupLifecycleEmitter — structural dual-channel CDI firing

Refs #273"
```

---

### Task 3: Migrate WorkItemService to WorkItemLifecycleEmitter

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`

This is a mechanical refactor — change the injection and simplify all call sites. The class currently has `@Inject Event<WorkItemLifecycleEvent> lifecycleEvent` as a field (line 68). All call sites use either `lifecycleEvent.fire(evt)` alone (13 methods) or `lifecycleEvent.fire(evt); lifecycleEvent.fireAsync(evt);` (12 methods), all guarded by `if (lifecycleEvent != null)`.

- [ ] **Step 1: Change the injection**

Replace the field injection at line 68:

```java
// BEFORE:
@Inject
Event<WorkItemLifecycleEvent> lifecycleEvent;

// AFTER:
@Inject
WorkItemLifecycleEmitter lifecycleEmitter;
```

Add import:
```java
import io.casehub.work.runtime.event.WorkItemLifecycleEmitter;
```

Remove import (no longer directly used):
```java
import jakarta.enterprise.event.Event;
```

Note: `Event` may still be imported if other `Event<>` usages exist in the file. Check after removing — the compiler will tell you.

- [ ] **Step 2: Migrate fire()-only call sites (13 methods)**

Each of these follows the pattern:
```java
// BEFORE:
if (lifecycleEvent != null) {
    lifecycleEvent.fire(WorkItemLifecycleEvent.of("EVENT_NAME", saved, actorId, detail));
}

// AFTER:
lifecycleEmitter.emit(WorkItemLifecycleEvent.of("EVENT_NAME", saved, actorId, detail));
```

Methods to change: `create()`, `claim()`, `start()`, `delegate()`, `acceptDelegation()`, `declineDelegation()`, `release()`, `suspend()`, `resume()`, `progress()`, `extend()`, `addLabel()`, `removeLabel()`.

- [ ] **Step 3: Migrate dual-fire call sites (12 methods)**

Each follows the pattern:
```java
// BEFORE:
if (lifecycleEvent != null) {
    final WorkItemLifecycleEvent evt = WorkItemLifecycleEvent.of("EVENT_NAME", saved, actorId, detail);
    lifecycleEvent.fire(evt);
    lifecycleEvent.fireAsync(evt);
}

// AFTER:
lifecycleEmitter.emit(WorkItemLifecycleEvent.of("EVENT_NAME", saved, actorId, detail));
```

Methods to change: `complete()` (both overloads), `reject()` (both overloads), `completeFromSystem()`, `rejectFromSystem()`, `cancel()`, `cancelFromSystem()`, `fault()`, `faultFromSystem()`, `obsolete()`, `obsoleteFromSystem()`.

Note: the `final WorkItemLifecycleEvent evt` local variable is no longer needed — inline the `WorkItemLifecycleEvent.of(...)` into the `emit()` call.

- [ ] **Step 4: Run the existing test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All existing tests PASS. The emitter fires both channels; existing `@Observes` handlers continue to receive via `fire()`.

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java
git commit -m "refactor(#273): migrate WorkItemService to WorkItemLifecycleEmitter

25 call sites simplified to emit(). Null checks removed.
13 fire()-only sites now also fire async (fixes QueueDashboard gap).

Refs #273"
```

---

### Task 4: Migrate ExpiryLifecycleService, WorkItemSpawnService, QueueStateResource

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemSpawnService.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/api/QueueStateResource.java`

Same mechanical pattern as Task 3 for each class.

- [ ] **Step 1: Migrate ExpiryLifecycleService**

Change injection (line 64):
```java
// BEFORE:
@Inject
Event<WorkItemLifecycleEvent> lifecycleEvent;

// AFTER:
@Inject
WorkItemLifecycleEmitter lifecycleEmitter;
```

Add import: `import io.casehub.work.runtime.event.WorkItemLifecycleEmitter;`

Change `fireLifecycleEvent()` method (line 357):
```java
// BEFORE:
private void fireLifecycleEvent(final String event, final WorkItem item) {
    if (lifecycleEvent != null) {
        lifecycleEvent.fire(WorkItemLifecycleEvent.of(event, item, "system", null));
    }
}

// AFTER:
private void fireLifecycleEvent(final String event, final WorkItem item) {
    lifecycleEmitter.emit(WorkItemLifecycleEvent.of(event, item, "system", null));
}
```

- [ ] **Step 2: Migrate WorkItemSpawnService**

Change injection (line 59):
```java
// BEFORE:
@Inject
Event<WorkItemLifecycleEvent> lifecycleEvent;

// AFTER:
@Inject
WorkItemLifecycleEmitter lifecycleEmitter;
```

Add import: `import io.casehub.work.runtime.event.WorkItemLifecycleEmitter;`

Change spawn() call site (line 176):
```java
// BEFORE:
if (lifecycleEvent != null) {
    lifecycleEvent.fire(WorkItemLifecycleEvent.of("SPAWNED", parent, "system:spawn", spawnDetail));
}

// AFTER:
lifecycleEmitter.emit(WorkItemLifecycleEvent.of("SPAWNED", parent, "system:spawn", spawnDetail));
```

- [ ] **Step 3: Migrate QueueStateResource**

Change injection (line 49):
```java
// BEFORE:
@Inject
Event<WorkItemLifecycleEvent> lifecycleEvent;

// AFTER:
@Inject
WorkItemLifecycleEmitter lifecycleEmitter;
```

Add import: `import io.casehub.work.runtime.event.WorkItemLifecycleEmitter;`

Change pickup() call site (line 126):
```java
// BEFORE:
lifecycleEvent.fire(WorkItemLifecycleEvent.of("ASSIGNED", saved, claimant,
        "Queue pickup (relinquishable takeover)"));

// AFTER:
lifecycleEmitter.emit(WorkItemLifecycleEvent.of("ASSIGNED", saved, claimant,
        "Queue pickup (relinquishable takeover)"));
```

- [ ] **Step 4: Run tests for both modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,queues`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java runtime/src/main/java/io/casehub/work/runtime/service/WorkItemSpawnService.java queues/src/main/java/io/casehub/work/queues/api/QueueStateResource.java
git commit -m "refactor(#273): migrate ExpiryLifecycleService, WorkItemSpawnService, QueueStateResource to emitter

Refs #273"
```

---

### Task 5: Migrate MultiInstanceGroupPolicy to WorkItemGroupLifecycleEmitter

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceGroupPolicy.java`

- [ ] **Step 1: Change the injection**

Replace the field injection (line 33):
```java
// BEFORE:
@Inject
Event<WorkItemGroupLifecycleEvent> groupEvent;

// AFTER:
@Inject
WorkItemGroupLifecycleEmitter groupEmitter;
```

Add import: `import io.casehub.work.runtime.event.WorkItemGroupLifecycleEmitter;`

- [ ] **Step 2: Change the fireEvent() method**

```java
// BEFORE:
public void fireEvent(final WorkItemGroupLifecycleEvent event) {
    if (event != null) {
        groupEvent.fireAsync(event);
    }
}

// AFTER:
public void fireEvent(final WorkItemGroupLifecycleEvent event) {
    if (event != null) {
        groupEmitter.emit(event);
    }
}
```

The null check stays — `process()` returns null for idempotent retries (policyTriggered guard). The emitter handles the dual-channel firing.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MultiInstance*`
Expected: PASS

- [ ] **Step 4: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceGroupPolicy.java
git commit -m "refactor(#273): migrate MultiInstanceGroupPolicy to WorkItemGroupLifecycleEmitter

Group events now fire on both sync and async channels.

Refs #273"
```

---

### Task 6: WorkCloudEventAdapter — test + implementation

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/event/WorkCloudEventAdapterTest.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventAdapter.java`

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.work.runtime.event;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

import java.time.Instant;
import java.util.Locale;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import com.fasterxml.jackson.databind.ObjectMapper;

import jakarta.enterprise.event.Event;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import io.cloudevents.CloudEvent;

import io.casehub.work.api.GroupStatus;
import io.casehub.work.api.WorkItemGroupLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;

@ExtendWith(MockitoExtension.class)
class WorkCloudEventAdapterTest {

    @Mock
    Event<CloudEvent> cloudEventBus;

    private final ObjectMapper objectMapper = new ObjectMapper();
    private WorkCloudEventAdapter adapter;

    @BeforeEach
    void setUp() {
        adapter = new WorkCloudEventAdapter(cloudEventBus, objectMapper);
        when(cloudEventBus.fireAsync(any())).thenReturn(CompletableFuture.completedFuture(null));
    }

    @Test
    void workItemEvent_buildsCorrectCloudEventFields() {
        final UUID id = UUID.randomUUID();
        final WorkItem wi = workItem(id, WorkItemStatus.COMPLETED, "tenant-1");
        final WorkItemLifecycleEvent event = WorkItemLifecycleEvent.of("COMPLETED", wi, "alice", "done");

        adapter.onWorkItemLifecycle(event);

        final CloudEvent ce = captureCloudEvent();
        assertThat(ce.getSpecVersion().toString()).isEqualTo("1.0");
        assertThat(ce.getType()).isEqualTo("io.casehub.work.workitem.completed");
        assertThat(ce.getSource().toString()).isEqualTo("/workitems/" + id);
        assertThat(ce.getSubject()).isEqualTo(id.toString());
        assertThat(ce.getDataContentType()).isEqualTo("application/json");
        assertThat(ce.getData()).isNotNull();
        assertThat(ce.getExtension("tenancyid")).isEqualTo("tenant-1");
        assertThat(ce.getTime()).isNotNull();
    }

    @Test
    void workItemEvent_usesEventTimestampNotAdapterTime() {
        final WorkItem wi = workItem(UUID.randomUUID(), WorkItemStatus.CREATED, "t1");
        final WorkItemLifecycleEvent event = WorkItemLifecycleEvent.of("CREATED", wi, "system", null);
        final Instant eventTime = event.occurredAt();

        adapter.onWorkItemLifecycle(event);

        final CloudEvent ce = captureCloudEvent();
        assertThat(ce.getTime().toInstant()).isEqualTo(eventTime);
    }

    @Test
    void workItemEvent_nullTenancyId_extensionOmitted() {
        final WorkItem wi = workItem(UUID.randomUUID(), WorkItemStatus.PENDING, null);
        final WorkItemLifecycleEvent event = WorkItemLifecycleEvent.of("CREATED", wi, "system", null);

        adapter.onWorkItemLifecycle(event);

        final CloudEvent ce = captureCloudEvent();
        assertThat(ce.getExtension("tenancyid")).isNull();
    }

    @Test
    void workItemEvent_serialisationFailure_firesWithEmptyData() {
        final ObjectMapper brokenMapper = mock(ObjectMapper.class);
        final WorkCloudEventAdapter brokenAdapter = new WorkCloudEventAdapter(cloudEventBus, brokenMapper);
        try {
            when(brokenMapper.writeValueAsBytes(any()))
                    .thenThrow(new com.fasterxml.jackson.core.JsonProcessingException("boom") {});
        } catch (final com.fasterxml.jackson.core.JsonProcessingException e) {
            throw new RuntimeException(e);
        }

        final WorkItem wi = workItem(UUID.randomUUID(), WorkItemStatus.COMPLETED, "t1");
        final WorkItemLifecycleEvent event = WorkItemLifecycleEvent.of("COMPLETED", wi, "alice", null);

        brokenAdapter.onWorkItemLifecycle(event);

        final CloudEvent ce = captureCloudEvent();
        assertThat(ce.getData().toBytes()).isEmpty();
    }

    @Test
    void groupEvent_buildsCorrectCloudEventFields() {
        final UUID parentId = UUID.randomUUID();
        final UUID groupId = UUID.randomUUID();
        final WorkItemGroupLifecycleEvent event = WorkItemGroupLifecycleEvent.of(
                parentId, groupId, 3, 2, 2, 0, GroupStatus.COMPLETED, "caller-ref", "tenant-1");

        adapter.onGroupLifecycle(event);

        final CloudEvent ce = captureCloudEvent();
        assertThat(ce.getSpecVersion().toString()).isEqualTo("1.0");
        assertThat(ce.getType()).isEqualTo("io.casehub.work.group.completed");
        assertThat(ce.getSource().toString()).isEqualTo("/workitems/groups/" + groupId);
        assertThat(ce.getSubject()).isEqualTo(groupId.toString());
        assertThat(ce.getDataContentType()).isEqualTo("application/json");
        assertThat(ce.getData()).isNotNull();
        assertThat(ce.getExtension("tenancyid")).isEqualTo("tenant-1");
    }

    @Test
    void groupEvent_typeDerivesFromGroupStatus() {
        for (final GroupStatus status : GroupStatus.values()) {
            reset(cloudEventBus);
            when(cloudEventBus.fireAsync(any())).thenReturn(CompletableFuture.completedFuture(null));

            final WorkItemGroupLifecycleEvent event = WorkItemGroupLifecycleEvent.of(
                    UUID.randomUUID(), UUID.randomUUID(), 3, 2, 1, 0, status, null, null);

            adapter.onGroupLifecycle(event);

            final CloudEvent ce = captureCloudEvent();
            assertThat(ce.getType())
                    .isEqualTo("io.casehub.work.group." + status.name().toLowerCase(Locale.ROOT));
        }
    }

    @Test
    void groupEvent_nullTenancyId_extensionOmitted() {
        final WorkItemGroupLifecycleEvent event = WorkItemGroupLifecycleEvent.of(
                UUID.randomUUID(), UUID.randomUUID(), 3, 2, 1, 0,
                GroupStatus.IN_PROGRESS, null, null);

        adapter.onGroupLifecycle(event);

        final CloudEvent ce = captureCloudEvent();
        assertThat(ce.getExtension("tenancyid")).isNull();
    }

    private CloudEvent captureCloudEvent() {
        final ArgumentCaptor<CloudEvent> captor = ArgumentCaptor.forClass(CloudEvent.class);
        verify(cloudEventBus).fireAsync(captor.capture());
        return captor.getValue();
    }

    private WorkItem workItem(final UUID id, final WorkItemStatus status, final String tenancyId) {
        final WorkItem wi = new WorkItem();
        wi.id = id;
        wi.status = status;
        wi.tenancyId = tenancyId;
        return wi;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkCloudEventAdapterTest`
Expected: FAIL — `WorkCloudEventAdapter` class does not exist

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.work.runtime.event;

import java.net.URI;
import java.time.ZoneOffset;
import java.util.Locale;
import java.util.UUID;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.work.api.WorkItemGroupLifecycleEvent;

@ApplicationScoped
public class WorkCloudEventAdapter {

    private static final Logger LOG = Logger.getLogger(WorkCloudEventAdapter.class);

    private final Event<CloudEvent> cloudEventBus;
    private final ObjectMapper objectMapper;

    @Inject
    public WorkCloudEventAdapter(final Event<CloudEvent> cloudEventBus, final ObjectMapper objectMapper) {
        this.cloudEventBus = cloudEventBus;
        this.objectMapper = objectMapper;
    }

    public void onWorkItemLifecycle(@ObservesAsync final WorkItemLifecycleEvent event) {
        byte[] data;
        try {
            data = objectMapper.writeValueAsBytes(event);
        } catch (final JsonProcessingException e) {
            LOG.warnf("Serialisation failed for %s: %s", event.type(), e.getMessage());
            data = new byte[0];
        }

        CloudEventBuilder builder = CloudEventBuilder.v1()
                .withId(UUID.randomUUID().toString())
                .withType(event.type())
                .withSource(URI.create(event.sourceUri()))
                .withSubject(event.subject())
                .withTime(event.occurredAt().atOffset(ZoneOffset.UTC))
                .withDataContentType("application/json")
                .withData(data);

        if (event.tenancyId() != null) {
            builder = builder.withExtension("tenancyid", event.tenancyId());
        }

        cloudEventBus.fireAsync(builder.build())
                .exceptionally(ex -> {
                    LOG.warnf(ex, "CloudEvent dispatch failed for %s", event.type());
                    return null;
                });
    }

    public void onGroupLifecycle(@ObservesAsync final WorkItemGroupLifecycleEvent event) {
        final String type = "io.casehub.work.group."
                + event.groupStatus().name().toLowerCase(Locale.ROOT);

        byte[] data;
        try {
            data = objectMapper.writeValueAsBytes(event);
        } catch (final JsonProcessingException e) {
            LOG.warnf("Serialisation failed for group %s: %s", event.groupId(), e.getMessage());
            data = new byte[0];
        }

        CloudEventBuilder builder = CloudEventBuilder.v1()
                .withId(UUID.randomUUID().toString())
                .withType(type)
                .withSource(URI.create("/workitems/groups/" + event.groupId()))
                .withSubject(event.groupId().toString())
                .withTime(event.occurredAt().atOffset(ZoneOffset.UTC))
                .withDataContentType("application/json")
                .withData(data);

        if (event.tenancyId() != null) {
            builder = builder.withExtension("tenancyid", event.tenancyId());
        }

        cloudEventBus.fireAsync(builder.build())
                .exceptionally(ex -> {
                    LOG.warnf(ex, "CloudEvent dispatch failed for group %s", event.groupId());
                    return null;
                });
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkCloudEventAdapterTest`
Expected: PASS (7 tests)

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventAdapter.java runtime/src/test/java/io/casehub/work/runtime/event/WorkCloudEventAdapterTest.java
git commit -m "feat(#273): WorkCloudEventAdapter — bridge lifecycle events to CloudEvent

24 WorkItem + 3 group CloudEvent types. Canonical 6-rule pattern.
Refs #273"
```

---

### Task 7: Full build verification

**Files:** None — verification only.

- [ ] **Step 1: Run the full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests PASS.

- [ ] **Step 2: Run queues module tests** (QueueStateResource was modified)

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues`
Expected: All tests PASS.

- [ ] **Step 3: Run the full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS. All modules compile and all tests pass.

If any test fails due to the refactored injection (e.g. a test that directly injects `Event<WorkItemLifecycleEvent>` and expects null-check behaviour), update the test to inject `WorkItemLifecycleEmitter` instead.

- [ ] **Step 4: Commit any test fixes**

Only if Step 3 surfaced failures:
```
git add -A
git commit -m "test(#273): update tests for emitter injection

Refs #273"
```

---

### Task 8: Platform documentation and parent issue

**Files:**
- No project code changes — external updates.

- [ ] **Step 1: File the parent evaluation issue**

```bash
gh issue create --repo casehubio/parent --title "evaluate: dual-channel CDI event firing as platform standard" --label "enhancement" --body "$(cat <<'EOF'
casehub-work normalized to dual-channel (`fire()` + `fireAsync()`) in casehubio/work#273 via `WorkItemLifecycleEmitter` and `WorkItemGroupLifecycleEmitter` beans that structurally enforce both calls.

## Context

All other repos currently fire async-only — this works because they have no sync observers.

casehub-work needs both channels because it has sync observers with transactional consistency requirements:
- `FilterEvaluationObserver` (`@Observes`) — queue membership updates must be atomic with the WorkItem mutation
- `LocalWorkItemEventBroadcaster` (`@Observes`) — ordered SSE delivery within transaction boundary
- `WorkItemMetrics` (`@Observes`) — counters incremented within transaction boundary
- Four `@Observes(during = TransactionPhase.AFTER_SUCCESS)` observers — `NotificationDispatcher`, `IssueLinkService`, `PostgresWorkItemEventBroadcaster`, `PostgresWorkItemQueueEventBroadcaster`

## Evaluation question

For each repo: **does any current or foreseeable observer need to run inside the producer's transaction?**

Patterns to look for:
- Observers that update derived views (like queue membership)
- Observers that aggregate state (like metrics counters)
- Observers that need ordering guarantees (like SSE broadcasting)
- Observers that use `@Observes(during = TransactionPhase.AFTER_SUCCESS)`

## Cost

Firing both costs nothing when a channel has no observers — CDI no-ops the empty channel.

## Note

This is an evaluation, not an adoption mandate. The other repos may not need sync observers at all.

Refs casehubio/work#273
EOF
)"
```

Record the returned issue number.

- [ ] **Step 2: Update PLATFORM.md Capability Ownership table**

Add this row to the Capability Ownership table in `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`, after the IoT → CloudEvent adapter row:

```markdown
| WorkItem → CloudEvent adapter | `casehub-work` runtime | `WorkCloudEventAdapter` — `@ObservesAsync WorkItemLifecycleEvent` + `WorkItemGroupLifecycleEvent`, produces `CloudEvent` via `Event<CloudEvent>.fireAsync()`. Event types: `io.casehub.work.workitem.*` (24 types), `io.casehub.work.group.*` (3 types). Dual-channel emitter enforces `fire()` + `fireAsync()` at all sites. Ships work#273. |
```

Commit to parent repo:
```
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: add WorkItem → CloudEvent adapter to Capability Ownership table

Refs casehubio/work#273"
```

- [ ] **Step 3: Update ARC42STORIES.MD**

Add the adapter delivery as a layer entry or chapter reference in the project's ARC42STORIES.MD. The specific section depends on the current structure — read the file and find the appropriate insertion point.

Commit to project repo:
```
git -C /Users/mdproctor/claude/casehub/work add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/work commit -m "docs(#273): ARC42STORIES — WorkCloudEventAdapter layer entry

Refs #273"
```
