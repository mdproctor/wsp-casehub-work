# Compensation Notifications Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #389 — connectors compensation notifications
**Issue group:** #238 (saga compensation epic)

**Goal:** Wire compensation events (work-level and case-level) through the platform subscription engine so external notification channels (Email/Slack/Teams/Webhook) deliver compensation alerts out-of-box.

**Architecture:** Two bridges feed the platform notification DataSource: (1) the existing `WorkItemSubscriptionBridge` in work/runtime — needs 2 event types added to its registration set, (2) a new `CaseCompensationNotifier` in engine-adapter — observes `CaseLifecycleEvent` CDI events, filters for compensation states, pushes a thin `CaseCompensationEvent` into the DataSource. A `CompensationSubscriptionBootstrap` registers 5 default system subscriptions at startup.

**Tech Stack:** Java 21, Quarkus 3.32, CDI events, platform subscription engine (`DataSourceRegistry`, `SubscriptionStore`, `EventTypeRegistry`)

## Global Constraints

- Package: `io.casehub.work.engine` (engine-adapter), `io.casehub.work.runtime.event` (work/runtime)
- All new files in engine-adapter follow existing pattern: constructor injection, `@ApplicationScoped`
- Tests use JUnit 5 + AssertJ + Mockito (no `@QuarkusTest` for unit tests)
- `casehub-connectors` requires zero changes
- Build with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter`
- Work/runtime build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

---

## Batch 1: Event Infrastructure

### Task 1: CaseCompensationEvent record

**Files:**
- Create: `engine-adapter/src/main/java/io/casehub/work/engine/CaseCompensationEvent.java`
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/CaseCompensationEventTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.subscription.SubscribableEvent` (platform-api)
- Produces: `CaseCompensationEvent(Kind, String tenancyId, UUID caseId, String caseDefinitionName, String caseStatus, String actorId)` — used by Task 2 (notifier) and Task 3 (bootstrap)

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.work.engine;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class CaseCompensationEventTest {

    @Test
    void type_returnsCorrectPrefix_forStarted() {
        var event = new CaseCompensationEvent(
                CaseCompensationEvent.Kind.STARTED, "tenant-1",
                UUID.randomUUID(), "ClinicalTrial", "COMPENSATING", "operator-1");
        assertThat(event.type()).isEqualTo("io.casehub.engine.case.compensation.started");
    }

    @Test
    void type_returnsCorrectPrefix_forCompleted() {
        var event = new CaseCompensationEvent(
                CaseCompensationEvent.Kind.COMPLETED, "tenant-1",
                UUID.randomUUID(), "ClinicalTrial", "COMPENSATED", "operator-1");
        assertThat(event.type()).isEqualTo("io.casehub.engine.case.compensation.completed");
    }

    @Test
    void type_returnsCorrectPrefix_forFaulted() {
        var event = new CaseCompensationEvent(
                CaseCompensationEvent.Kind.FAULTED, "tenant-1",
                UUID.randomUUID(), "ClinicalTrial", "COMPENSATION_FAULTED", "operator-1");
        assertThat(event.type()).isEqualTo("io.casehub.engine.case.compensation.faulted");
    }

    @Test
    void tenancyId_returnsConstructionValue() {
        var event = new CaseCompensationEvent(
                CaseCompensationEvent.Kind.STARTED, "my-tenant",
                UUID.randomUUID(), null, "COMPENSATING", null);
        assertThat(event.tenancyId()).isEqualTo("my-tenant");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=CaseCompensationEventTest`
Expected: FAIL — `CaseCompensationEvent` does not exist

- [ ] **Step 3: Write the implementation**

Use `ide_create_file` to create `engine-adapter/src/main/java/io/casehub/work/engine/CaseCompensationEvent.java`:

```java
package io.casehub.work.engine;

import io.casehub.platform.api.subscription.SubscribableEvent;
import java.util.Objects;
import java.util.UUID;

public record CaseCompensationEvent(
        Kind kind,
        String tenancyId,
        UUID caseId,
        String caseDefinitionName,
        String caseStatus,
        String actorId
) implements SubscribableEvent {

    public enum Kind { STARTED, COMPLETED, FAULTED }

    private static final String TYPE_PREFIX = "io.casehub.engine.case.compensation.";

    public CaseCompensationEvent {
        Objects.requireNonNull(kind, "kind");
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(caseId, "caseId");
    }

    @Override
    public String type() {
        return TYPE_PREFIX + kind.name().toLowerCase();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=CaseCompensationEventTest`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add engine-adapter/src/main/java/io/casehub/work/engine/CaseCompensationEvent.java \
       engine-adapter/src/test/java/io/casehub/work/engine/CaseCompensationEventTest.java
git commit -m "feat(#389): CaseCompensationEvent — SubscribableEvent for case compensation notifications

Refs #389 Refs #238"
```

---

### Task 2: CaseCompensationNotifier CDI observer

**Files:**
- Create: `engine-adapter/src/main/java/io/casehub/work/engine/CaseCompensationNotifier.java`
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/CaseCompensationNotifierTest.java`

**Interfaces:**
- Consumes: `CaseCompensationEvent` (Task 1), `CaseLifecycleEvent` (engine-common), `DataSourceRegistry` (platform-api)
- Produces: Pushes `CaseCompensationEvent` into the notification DataSource. No downstream code depends on this directly — the subscription engine picks events up automatically.

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.work.engine;

import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.platform.api.path.Path;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import java.util.ArrayList;
import java.util.Optional;
import java.util.UUID;

import static io.casehub.platform.api.identity.TenancyConstants.PLATFORM_TENANT_ID;
import static io.casehub.platform.api.subscription.SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class CaseCompensationNotifierTest {

    @ParameterizedTest
    @CsvSource({
        "CaseCompensating,    STARTED",
        "CaseCompensated,     COMPLETED",
        "CaseCompensationFaulted, FAULTED"
    })
    void firesCompensationEvent_forCompensationEventTypes(String eventType, String expectedKind) {
        var captured = new ArrayList<Object>();
        var notifier = buildNotifier(captured);

        notifier.onCaseLifecycle(caseEvent(eventType, "tenant-1", "ClinicalTrial"));

        assertThat(captured).hasSize(1);
        var fired = (CaseCompensationEvent) captured.get(0);
        assertThat(fired.kind()).isEqualTo(CaseCompensationEvent.Kind.valueOf(expectedKind));
        assertThat(fired.tenancyId()).isEqualTo("tenant-1");
        assertThat(fired.caseDefinitionName()).isEqualTo("ClinicalTrial");
    }

    @Test
    void ignoresNonCompensationEventTypes() {
        var captured = new ArrayList<Object>();
        var notifier = buildNotifier(captured);

        notifier.onCaseLifecycle(caseEvent("CaseCompleted", "tenant-1", "Trial"));
        notifier.onCaseLifecycle(caseEvent("CaseFaulted", "tenant-1", "Trial"));
        notifier.onCaseLifecycle(caseEvent("CaseCancelled", "tenant-1", "Trial"));

        assertThat(captured).isEmpty();
    }

    @Test
    void noOp_whenRegistryUnsatisfied() {
        var notifier = new CaseCompensationNotifier();
        setField(notifier, "dataSourceRegistryInstance", unsatisfiedInstance());

        notifier.onCaseLifecycle(caseEvent("CaseCompensating", "t", "T"));
    }

    @Test
    void noOp_whenDataSourceNotResolved() {
        var registry = mock(DataSourceRegistry.class);
        when(registry.resolveSource(any(Path.class), any(String.class)))
                .thenReturn(Optional.empty());
        var notifier = new CaseCompensationNotifier();
        setField(notifier, "dataSourceRegistryInstance", satisfiedInstance(registry));

        notifier.onCaseLifecycle(caseEvent("CaseCompensating", "t", "T"));
    }

    @Test
    void catchesExceptions_fromDataSource() {
        var registry = mock(DataSourceRegistry.class);
        when(registry.resolveSource(any(Path.class), any(String.class)))
                .thenThrow(new RuntimeException("DataSource unavailable"));
        var notifier = new CaseCompensationNotifier();
        setField(notifier, "dataSourceRegistryInstance", satisfiedInstance(registry));

        notifier.onCaseLifecycle(caseEvent("CaseCompensating", "t", "T"));
    }

    private CaseCompensationNotifier buildNotifier(ArrayList<Object> captured) {
        @SuppressWarnings("unchecked")
        DataSource<Object> ds = mock(DataSource.class);
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return null; })
                .when(ds).add(any());
        var registry = mock(DataSourceRegistry.class);
        when(registry.resolveSource(eq(NOTIFICATION_DATASOURCE_PATH), eq(PLATFORM_TENANT_ID)))
                .thenReturn(Optional.of(ds));
        var notifier = new CaseCompensationNotifier();
        setField(notifier, "dataSourceRegistryInstance", satisfiedInstance(registry));
        return notifier;
    }

    private CaseLifecycleEvent caseEvent(String eventType, String tenancyId, String defName) {
        return new CaseLifecycleEvent(
                UUID.randomUUID(), tenancyId, "command", eventType,
                "COMPENSATING", "operator-1", "System", null,
                defName, null, null, null, null);
    }

    @SuppressWarnings("unchecked")
    private <T> Instance<T> satisfiedInstance(T value) {
        var instance = mock(Instance.class);
        when(instance.isUnsatisfied()).thenReturn(false);
        when(instance.get()).thenReturn(value);
        return instance;
    }

    @SuppressWarnings("unchecked")
    private <T> Instance<T> unsatisfiedInstance() {
        var instance = mock(Instance.class);
        when(instance.isUnsatisfied()).thenReturn(true);
        return instance;
    }

    private void setField(Object target, String fieldName, Object value) {
        try {
            var field = target.getClass().getDeclaredField(fieldName);
            field.setAccessible(true);
            field.set(target, value);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=CaseCompensationNotifierTest`
Expected: FAIL — `CaseCompensationNotifier` does not exist

- [ ] **Step 3: Write the implementation**

Use `ide_create_file` to create `engine-adapter/src/main/java/io/casehub/work/engine/CaseCompensationNotifier.java`:

```java
package io.casehub.work.engine;

import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Optional;
import java.util.Set;

import static io.casehub.platform.api.identity.TenancyConstants.PLATFORM_TENANT_ID;
import static io.casehub.platform.api.subscription.SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH;

@ApplicationScoped
public class CaseCompensationNotifier {

    private static final Logger LOG = Logger.getLogger(CaseCompensationNotifier.class);

    private static final Set<String> COMPENSATION_EVENT_TYPES = Set.of(
            "CaseCompensating", "CaseCompensated", "CaseCompensationFaulted");

    @Inject
    Instance<DataSourceRegistry> dataSourceRegistryInstance;

    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (!COMPENSATION_EVENT_TYPES.contains(event.eventType())) {
            return;
        }
        if (dataSourceRegistryInstance.isUnsatisfied()) {
            return;
        }
        CaseCompensationEvent.Kind kind = switch (event.eventType()) {
            case "CaseCompensating" -> CaseCompensationEvent.Kind.STARTED;
            case "CaseCompensated" -> CaseCompensationEvent.Kind.COMPLETED;
            case "CaseCompensationFaulted" -> CaseCompensationEvent.Kind.FAULTED;
            default -> null;
        };
        if (kind == null) {
            return;
        }
        fire(new CaseCompensationEvent(
                kind, event.tenancyId(), event.caseId(),
                event.caseDefinitionName(), event.caseStatus(),
                event.actorId()));
    }

    private void fire(CaseCompensationEvent event) {
        try {
            Optional<DataSource<?>> ds = dataSourceRegistryInstance.get()
                    .resolveSource(NOTIFICATION_DATASOURCE_PATH, PLATFORM_TENANT_ID);
            if (ds.isEmpty()) {
                LOG.warnf("Notification DataSource not available — dropping %s event", event.kind());
                return;
            }
            @SuppressWarnings("unchecked")
            DataSource<Object> source = (DataSource<Object>) ds.get();
            source.add(event);
        } catch (Exception e) {
            LOG.warnf("Failed to fire compensation event %s: %s", event.kind(), e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=CaseCompensationNotifierTest`
Expected: PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git add engine-adapter/src/main/java/io/casehub/work/engine/CaseCompensationNotifier.java \
       engine-adapter/src/test/java/io/casehub/work/engine/CaseCompensationNotifierTest.java
git commit -m "feat(#389): CaseCompensationNotifier — bridges engine CaseLifecycleEvent to notification DataSource

Observes CaseLifecycleEvent via @ObservesAsync, filters for compensation
eventTypes, pushes CaseCompensationEvent into the platform notification
DataSource.

Refs #389 Refs #238"
```

---

## Batch 2: Subscription Registration

### Task 3: CompensationSubscriptionBootstrap + EventTypeDescriptor registration

**Files:**
- Create: `engine-adapter/src/main/java/io/casehub/work/engine/CompensationSubscriptionBootstrap.java`
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/CompensationSubscriptionBootstrapTest.java`

**Interfaces:**
- Consumes: `CaseCompensationEvent.Kind` (Task 1) for event type string constants, `SubscriptionStore` (platform-api), `EventTypeRegistry` (platform-api)
- Produces: 5 system subscriptions + 3 case event type descriptors registered at startup. No code depends on this — the subscription engine uses them for event matching.

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.work.engine;

import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.subscription.EventTypeDescriptor;
import io.casehub.platform.api.subscription.EventTypeRegistry;
import io.casehub.platform.api.subscription.Subscription;
import io.casehub.platform.api.subscription.SubscriptionInput;
import io.casehub.platform.api.subscription.SubscriptionStore;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.ArrayList;
import java.util.List;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class CompensationSubscriptionBootstrapTest {

    @Test
    void registersAllFiveSubscriptions() {
        var store = mock(SubscriptionStore.class);
        when(store.findAllEnabled()).thenReturn(Stream.empty());
        var eventRegistry = mock(EventTypeRegistry.class);
        var bootstrap = buildBootstrap(store, eventRegistry);

        bootstrap.onStartup(mock(StartupEvent.class));

        var captor = ArgumentCaptor.forClass(SubscriptionInput.class);
        verify(store, times(5)).store(captor.capture());
        var eventTypes = captor.getAllValues().stream()
                .map(SubscriptionInput::eventType).toList();
        assertThat(eventTypes).containsExactlyInAnyOrder(
                "io.casehub.work.workitem.compensation_started",
                "io.casehub.work.workitem.compensation_completed",
                "io.casehub.engine.case.compensation.started",
                "io.casehub.engine.case.compensation.completed",
                "io.casehub.engine.case.compensation.faulted");
    }

    @Test
    void skipsAlreadyRegisteredSubscriptions() {
        var existing = mock(Subscription.class);
        when(existing.eventType()).thenReturn("io.casehub.engine.case.compensation.started");
        var store = mock(SubscriptionStore.class);
        when(store.findAllEnabled()).thenReturn(Stream.of(existing));
        var eventRegistry = mock(EventTypeRegistry.class);
        var bootstrap = buildBootstrap(store, eventRegistry);

        bootstrap.onStartup(mock(StartupEvent.class));

        var captor = ArgumentCaptor.forClass(SubscriptionInput.class);
        verify(store, times(4)).store(captor.capture());
        var eventTypes = captor.getAllValues().stream()
                .map(SubscriptionInput::eventType).toList();
        assertThat(eventTypes).doesNotContain("io.casehub.engine.case.compensation.started");
    }

    @Test
    void registersThreeCaseEventTypeDescriptors() {
        var store = mock(SubscriptionStore.class);
        when(store.findAllEnabled()).thenReturn(Stream.empty());
        var eventRegistry = mock(EventTypeRegistry.class);
        var bootstrap = buildBootstrap(store, eventRegistry);

        bootstrap.onStartup(mock(StartupEvent.class));

        var captor = ArgumentCaptor.forClass(EventTypeDescriptor.class);
        verify(eventRegistry, times(3)).register(captor.capture());
        var eventTypes = captor.getAllValues().stream()
                .map(EventTypeDescriptor::eventType).toList();
        assertThat(eventTypes).containsExactlyInAnyOrder(
                "io.casehub.engine.case.compensation.started",
                "io.casehub.engine.case.compensation.completed",
                "io.casehub.engine.case.compensation.faulted");
    }

    @Test
    void caseCompensationStarted_hasUrgentSeverity() {
        var store = mock(SubscriptionStore.class);
        when(store.findAllEnabled()).thenReturn(Stream.empty());
        var eventRegistry = mock(EventTypeRegistry.class);
        var bootstrap = buildBootstrap(store, eventRegistry);

        bootstrap.onStartup(mock(StartupEvent.class));

        var captor = ArgumentCaptor.forClass(SubscriptionInput.class);
        verify(store, times(5)).store(captor.capture());
        var started = captor.getAllValues().stream()
                .filter(s -> s.eventType().equals("io.casehub.engine.case.compensation.started"))
                .findFirst().orElseThrow();
        assertThat(started.template().severity()).isEqualTo(NotificationSeverity.URGENT);
    }

    @Test
    void noOp_whenStoreUnsatisfied() {
        var bootstrap = new CompensationSubscriptionBootstrap();
        setField(bootstrap, "subscriptionStoreInstance", unsatisfiedInstance());
        setField(bootstrap, "eventTypeRegistryInstance", unsatisfiedInstance());

        bootstrap.onStartup(mock(StartupEvent.class));
    }

    @Test
    void handlesStoreFailureGracefully() {
        var store = mock(SubscriptionStore.class);
        when(store.findAllEnabled()).thenReturn(Stream.empty());
        when(store.store(any())).thenThrow(new RuntimeException("DB down"));
        var eventRegistry = mock(EventTypeRegistry.class);
        var bootstrap = buildBootstrap(store, eventRegistry);

        bootstrap.onStartup(mock(StartupEvent.class));
    }

    private CompensationSubscriptionBootstrap buildBootstrap(
            SubscriptionStore store, EventTypeRegistry eventRegistry) {
        var bootstrap = new CompensationSubscriptionBootstrap();
        setField(bootstrap, "subscriptionStoreInstance", satisfiedInstance(store));
        setField(bootstrap, "eventTypeRegistryInstance", satisfiedInstance(eventRegistry));
        return bootstrap;
    }

    @SuppressWarnings("unchecked")
    private <T> Instance<T> satisfiedInstance(T value) {
        var instance = mock(Instance.class);
        when(instance.isUnsatisfied()).thenReturn(false);
        when(instance.get()).thenReturn(value);
        return instance;
    }

    @SuppressWarnings("unchecked")
    private <T> Instance<T> unsatisfiedInstance() {
        var instance = mock(Instance.class);
        when(instance.isUnsatisfied()).thenReturn(true);
        return instance;
    }

    private void setField(Object target, String fieldName, Object value) {
        try {
            var field = target.getClass().getDeclaredField(fieldName);
            field.setAccessible(true);
            field.set(target, value);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=CompensationSubscriptionBootstrapTest`
Expected: FAIL — `CompensationSubscriptionBootstrap` does not exist

- [ ] **Step 3: Write the implementation**

Use `ide_create_file` to create `engine-adapter/src/main/java/io/casehub/work/engine/CompensationSubscriptionBootstrap.java`:

```java
package io.casehub.work.engine;

import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.subscription.EventFieldDescriptor;
import io.casehub.platform.api.subscription.EventTypeDescriptor;
import io.casehub.platform.api.subscription.EventTypeRegistry;
import io.casehub.platform.api.subscription.NotificationTarget;
import io.casehub.platform.api.subscription.NotificationTemplate;
import io.casehub.platform.api.subscription.Subscription;
import io.casehub.platform.api.subscription.SubscriptionInput;
import io.casehub.platform.api.subscription.SubscriptionScope;
import io.casehub.platform.api.subscription.SubscriptionStore;
import io.casehub.platform.api.subscription.TargetType;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

import static io.casehub.platform.api.identity.TenancyConstants.PLATFORM_TENANT_ID;

@ApplicationScoped
public class CompensationSubscriptionBootstrap {

    private static final Logger LOG = Logger.getLogger(CompensationSubscriptionBootstrap.class);
    private static final String OWNER_ID = "system:compensation";
    private static final String COMPENSATION_PREFIX = "io.casehub.engine.case.compensation.";
    private static final String WORK_COMPENSATION_PREFIX = "io.casehub.work.workitem.compensation_";

    @Inject
    Instance<SubscriptionStore> subscriptionStoreInstance;

    @Inject
    Instance<EventTypeRegistry> eventTypeRegistryInstance;

    void onStartup(@Observes StartupEvent event) {
        if (subscriptionStoreInstance.isUnsatisfied()) {
            return;
        }
        SubscriptionStore store = subscriptionStoreInstance.get();

        Set<String> existing = store.findAllEnabled()
                .filter(s -> s.eventType().startsWith(COMPENSATION_PREFIX)
                          || s.eventType().startsWith(WORK_COMPENSATION_PREFIX))
                .map(Subscription::eventType)
                .collect(Collectors.toSet());

        registerWorkSubscription(store, existing, "compensation_started",
                "assigneeId", "WorkItem compensation started",
                NotificationSeverity.WARNING);
        registerWorkSubscription(store, existing, "compensation_completed",
                "assigneeId", "WorkItem compensation completed",
                NotificationSeverity.INFO);

        registerCaseSubscription(store, existing, "started",
                "Case compensation started: {caseDefinitionName}",
                NotificationSeverity.URGENT);
        registerCaseSubscription(store, existing, "completed",
                "Case compensation completed: {caseDefinitionName}",
                NotificationSeverity.INFO);
        registerCaseSubscription(store, existing, "faulted",
                "Case compensation FAULTED: {caseDefinitionName}",
                NotificationSeverity.URGENT);

        if (!eventTypeRegistryInstance.isUnsatisfied()) {
            registerCaseEventTypes(eventTypeRegistryInstance.get());
        }
    }

    private void registerWorkSubscription(SubscriptionStore store, Set<String> existing,
                                          String suffix, String targetField,
                                          String titlePattern, NotificationSeverity severity) {
        String eventType = WORK_COMPENSATION_PREFIX + suffix;
        if (existing.contains(eventType)) {
            return;
        }
        try {
            store.store(new SubscriptionInput(
                    OWNER_ID, PLATFORM_TENANT_ID,
                    "work.compensation." + suffix, eventType,
                    List.of(),
                    List.of(new NotificationTarget(TargetType.EVENT_FIELD, targetField)),
                    false,
                    new NotificationTemplate(titlePattern, null, severity,
                            "work.compensation." + suffix, null,
                            "workitem", "workItemId", "actor"),
                    true, SubscriptionScope.SYSTEM));
            LOG.infof("Registered compensation subscription for %s", eventType);
        } catch (Exception e) {
            LOG.warnf("Failed to register compensation subscription for %s: %s",
                      eventType, e.getMessage());
        }
    }

    private void registerCaseSubscription(SubscriptionStore store, Set<String> existing,
                                          String suffix, String titlePattern,
                                          NotificationSeverity severity) {
        String eventType = COMPENSATION_PREFIX + suffix;
        if (existing.contains(eventType)) {
            return;
        }
        try {
            store.store(new SubscriptionInput(
                    OWNER_ID, PLATFORM_TENANT_ID,
                    "case.compensation." + suffix, eventType,
                    List.of(),
                    List.of(new NotificationTarget(TargetType.EVENT_FIELD, "actorId")),
                    false,
                    new NotificationTemplate(titlePattern, null, severity,
                            "case.compensation." + suffix, null,
                            "case", "caseId", "actorId"),
                    true, SubscriptionScope.SYSTEM));
            LOG.infof("Registered compensation subscription for %s", eventType);
        } catch (Exception e) {
            LOG.warnf("Failed to register compensation subscription for %s: %s",
                      eventType, e.getMessage());
        }
    }

    private void registerCaseEventTypes(EventTypeRegistry registry) {
        List<EventFieldDescriptor> fields = List.of(
                new EventFieldDescriptor("caseId", "Case ID", "string"),
                new EventFieldDescriptor("caseDefinitionName", "Case Definition", "string"),
                new EventFieldDescriptor("caseStatus", "Case Status", "string"),
                new EventFieldDescriptor("actorId", "Actor", "string"));
        for (CaseCompensationEvent.Kind kind : CaseCompensationEvent.Kind.values()) {
            String eventType = "io.casehub.engine.case.compensation." + kind.name().toLowerCase();
            String displayName = "Case compensation " + kind.name().toLowerCase();
            registry.register(new EventTypeDescriptor(eventType, displayName, null, fields));
        }
        LOG.info("Registered 3 case compensation event types with subscription engine");
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=CompensationSubscriptionBootstrapTest`
Expected: PASS (6 tests)

- [ ] **Step 5: Run all engine-adapter tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter`
Expected: All pass (except pre-existing `HumanTaskPlannerIntegrationTest` timeout — known issue from handover)

- [ ] **Step 6: Commit**

```bash
git add engine-adapter/src/main/java/io/casehub/work/engine/CompensationSubscriptionBootstrap.java \
       engine-adapter/src/test/java/io/casehub/work/engine/CompensationSubscriptionBootstrapTest.java
git commit -m "feat(#389): CompensationSubscriptionBootstrap — default subscriptions for compensation notifications

Registers 5 system subscriptions (2 work-level, 3 case-level) and 3 case
event type descriptors at startup. Follows QhorusSubscriptionBootstrap
pattern.

Refs #389 Refs #238"
```

---

### Task 4: WorkItemSubscriptionBridge event type registration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemSubscriptionBridge.java` — add 2 entries to `EMITTED_EVENT_TYPES`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/event/WorkItemSubscriptionBridgeTest.java` — add assertion for compensation types

**Interfaces:**
- Consumes: `WorkEventType.COMPENSATION_STARTED`, `WorkEventType.COMPENSATION_COMPLETED` (work-api, already exist)
- Produces: Event type registrations with `EventTypeRegistry` (used by subscription engine for matching)

- [ ] **Step 1: Write the failing test**

Add a new test method to `WorkItemSubscriptionBridgeTest.java`:

```java
@Test
void emittedEventTypes_includesCompensationTypes() {
    // EMITTED_EVENT_TYPES is private — verify via event type registration
    var eventRegistry = mock(EventTypeRegistry.class);
    var bridge = new WorkItemSubscriptionBridge();
    setField(bridge, "dataSourceRegistryInstance", unsatisfiedInstance());
    setField(bridge, "eventTypeRegistryInstance", satisfiedInstance(eventRegistry));

    bridge.onStartup(mock(io.quarkus.runtime.StartupEvent.class));

    var captor = ArgumentCaptor.forClass(
            io.casehub.platform.api.subscription.EventTypeDescriptor.class);
    verify(eventRegistry, atLeastOnce()).register(captor.capture());
    var registeredTypes = captor.getAllValues().stream()
            .map(io.casehub.platform.api.subscription.EventTypeDescriptor::eventType)
            .toList();
    assertThat(registeredTypes).contains(
            "io.casehub.work.workitem.compensation_started",
            "io.casehub.work.workitem.compensation_completed");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemSubscriptionBridgeTest#emittedEventTypes_includesCompensationTypes`
Expected: FAIL — compensation types not in `EMITTED_EVENT_TYPES`

- [ ] **Step 3: Add compensation types to EMITTED_EVENT_TYPES**

Use `ide_edit_member` on `WorkItemSubscriptionBridge`, member `EMITTED_EVENT_TYPES`, to add the two entries:

```java
private static final Set<WorkEventType> EMITTED_EVENT_TYPES = Set.of(
        WorkEventType.CREATED, WorkEventType.ASSIGNED, WorkEventType.STARTED,
        WorkEventType.COMPLETED, WorkEventType.REJECTED, WorkEventType.FAULTED,
        WorkEventType.DELEGATED, WorkEventType.DELEGATION_ACCEPTED,
        WorkEventType.DELEGATION_DECLINED, WorkEventType.RELEASED,
        WorkEventType.SUSPENDED, WorkEventType.RESUMED, WorkEventType.CANCELLED,
        WorkEventType.OBSOLETE, WorkEventType.EXPIRED, WorkEventType.CLAIM_EXPIRED,
        WorkEventType.SPAWNED, WorkEventType.ESCALATED,
        WorkEventType.DEADLINE_EXTENDED, WorkEventType.SLA_REASSIGNED,
        WorkEventType.SLA_EXTENDED, WorkEventType.SIGNAL_RECEIVED,
        WorkEventType.MANUALLY_ESCALATED, WorkEventType.PROGRESS_UPDATE,
        WorkEventType.COMPENSATION_STARTED, WorkEventType.COMPENSATION_COMPLETED
);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemSubscriptionBridgeTest`
Expected: PASS (all tests including new one)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/event/WorkItemSubscriptionBridge.java \
       runtime/src/test/java/io/casehub/work/runtime/event/WorkItemSubscriptionBridgeTest.java
git commit -m "feat(#389): register COMPENSATION_STARTED/COMPLETED in WorkItemSubscriptionBridge

Adds compensation event types to EMITTED_EVENT_TYPES so the subscription
engine recognizes them for matching. Events already flow through the
bridge — this gap was in EventTypeRegistry registration only.

Refs #389 Refs #238"
```

---

## References

- [2026-09-04-compensation-notifications-design.md] — design spec this plan implements
- [WorkItemSubscriptionBridge.java (work/runtime)] — existing event bridge, 2-line fix target
- [WorkItemSubscriptionBridgeTest.java (work/runtime)] — existing test pattern (Instance mocking, setField)
- [CommitmentEventNotifier.java (qhorus/notification-bridge)] — CDI observer → DataSource pattern
- [QhorusSubscriptionBootstrap.java (qhorus/notification-bridge)] — subscription registration pattern
- [CaseLifecycleEvent.java (engine-common)] — CDI event carrying case lifecycle transitions
- [CaseStatusChangedHandler.java (engine/runtime)] — compensation eventType string values
- [SubscriptionInput, NotificationTemplate, EventTypeDescriptor (platform-api)] — API signatures
- [notifications.md (parent/docs/platform)] — notification pipeline architecture
- [GitHub #389] — focal issue
- [GitHub #238] — parent epic
