# CloudEvent WorkItem Bridge — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make casehub-work WorkItems addressable via CloudEvents — external systems fire a `requested` CloudEvent to create a WorkItem, and lifecycle events flow back as CloudEvents for correlation.

**Architecture:** Type constants in `casehub-work-api` (pure Java). Inbound adapter in `runtime` alongside the existing outbound `WorkCloudEventAdapter`. Template-only creation path — domain data flows through opaquely. Partial unique index on `(caller_ref, tenancy_id)` for CloudEvent idempotency in clustered deployments.

**Tech Stack:** Java 21, Quarkus 3.32, CNCF CloudEvents Java SDK (`io.cloudevents`), CDI async events, Flyway, H2 (test)

## Global Constraints

- Java source/target: 21 (running on Java 26 JVM)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use scripts/ helpers for timeouts (see `scripts/README.md`)
- Flyway domain range: V1–V999 (current max: V39). Next: V40
- `casehub-work-api` is pure Java — no Quarkus, no Jackson, no CDI runtime deps
- All `@ObservesAsync` handlers must use `TenantContextRunner.runInTenantContext()`
- Outbound CloudEvent types use prefix `io.casehub.work.workitem.` + `WorkEventType.name().toLowerCase()`
- Group CloudEvent types use prefix `io.casehub.work.group.` + `GroupStatus.name().toLowerCase()`
- Extension attribute names: `tenancyid`, `templateid` (lowercase, no separator — CloudEvents spec convention)

---

### Task 1: `WorkCloudEventTypes` constants + drift prevention test

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/WorkCloudEventTypes.java`
- Create: `api/src/test/java/io/casehub/work/api/WorkCloudEventTypesTest.java`

**Interfaces:**
- Consumes: `WorkEventType` enum (same module)
- Produces: `WorkCloudEventTypes.PREFIX`, `WorkCloudEventTypes.REQUESTED`, `WorkCloudEventTypes.CREATED`, `WorkCloudEventTypes.COMPLETED`, ..., `WorkCloudEventTypes.GROUP_PREFIX`, `WorkCloudEventTypes.EXT_TENANCY_ID`, `WorkCloudEventTypes.EXT_TEMPLATE_ID`, `WorkCloudEventTypes.forEventType(WorkEventType): String`

- [ ] **Step 1: Write the drift prevention test**

```java
package io.casehub.work.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class WorkCloudEventTypesTest {

    @Test
    void everyWorkEventType_hasMappedConstant() {
        for (final WorkEventType type : WorkEventType.values()) {
            final String expected = WorkCloudEventTypes.PREFIX + type.name().toLowerCase();
            assertThat(WorkCloudEventTypes.forEventType(type))
                    .as("Missing constant for WorkEventType.%s", type.name())
                    .isEqualTo(expected);
        }
    }

    @Test
    void requested_isNotInWorkEventType() {
        assertThat(WorkCloudEventTypes.REQUESTED)
                .isEqualTo("io.casehub.work.workitem.requested");
        for (final WorkEventType type : WorkEventType.values()) {
            assertThat(type.name()).isNotEqualToIgnoringCase("requested");
        }
    }

    @Test
    void prefixFormat() {
        assertThat(WorkCloudEventTypes.PREFIX).isEqualTo("io.casehub.work.workitem.");
        assertThat(WorkCloudEventTypes.GROUP_PREFIX).isEqualTo("io.casehub.work.group.");
    }

    @Test
    void extensionAttributeNames() {
        assertThat(WorkCloudEventTypes.EXT_TENANCY_ID).isEqualTo("tenancyid");
        assertThat(WorkCloudEventTypes.EXT_TEMPLATE_ID).isEqualTo("templateid");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCloudEventTypesTest -pl api`
Expected: FAIL — `WorkCloudEventTypes` class does not exist

- [ ] **Step 3: Implement `WorkCloudEventTypes`**

```java
package io.casehub.work.api;

import java.util.Locale;

public final class WorkCloudEventTypes {

    public static final String PREFIX = "io.casehub.work.workitem.";
    public static final String GROUP_PREFIX = "io.casehub.work.group.";

    // Inbound — not a lifecycle event, not in WorkEventType
    public static final String REQUESTED = PREFIX + "requested";

    // Lifecycle — one per WorkEventType value
    public static final String CREATED = PREFIX + "created";
    public static final String ASSIGNED = PREFIX + "assigned";
    public static final String STARTED = PREFIX + "started";
    public static final String COMPLETED = PREFIX + "completed";
    public static final String REJECTED = PREFIX + "rejected";
    public static final String FAULTED = PREFIX + "faulted";
    public static final String DELEGATED = PREFIX + "delegated";
    public static final String DELEGATION_ACCEPTED = PREFIX + "delegation_accepted";
    public static final String DELEGATION_DECLINED = PREFIX + "delegation_declined";
    public static final String RELEASED = PREFIX + "released";
    public static final String SUSPENDED = PREFIX + "suspended";
    public static final String RESUMED = PREFIX + "resumed";
    public static final String CANCELLED = PREFIX + "cancelled";
    public static final String OBSOLETE = PREFIX + "obsolete";
    public static final String EXPIRED = PREFIX + "expired";
    public static final String CLAIM_EXPIRED = PREFIX + "claim_expired";
    public static final String SPAWNED = PREFIX + "spawned";
    public static final String ESCALATED = PREFIX + "escalated";
    public static final String DEADLINE_EXTENDED = PREFIX + "deadline_extended";
    public static final String SLA_REASSIGNED = PREFIX + "sla_reassigned";
    public static final String SLA_EXTENDED = PREFIX + "sla_extended";
    public static final String SIGNAL_RECEIVED = PREFIX + "signal_received";
    public static final String MANUALLY_ESCALATED = PREFIX + "manually_escalated";
    public static final String PROGRESS_UPDATE = PREFIX + "progress_update";
    public static final String LABEL_ADDED = PREFIX + "label_added";
    public static final String LABEL_REMOVED = PREFIX + "label_removed";

    // Extension attribute names
    public static final String EXT_TENANCY_ID = "tenancyid";
    public static final String EXT_TEMPLATE_ID = "templateid";

    public static String forEventType(final WorkEventType type) {
        return PREFIX + type.name().toLowerCase(Locale.ROOT);
    }

    private WorkCloudEventTypes() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCloudEventTypesTest -pl api`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```
feat(#172): add WorkCloudEventTypes constants to casehub-work-api

Pure-Java type constants for CloudEvent types, extension attributes,
and a drift-prevention test that asserts sync with WorkEventType enum.

Refs #172, Refs #92
```

---

### Task 2: Migrate outbound adapter to use constants

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventAdapter.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEvent.java` (line 97, 121 — the `of()` factory methods)
- Modify: `runtime/src/test/java/io/casehub/work/runtime/event/WorkCloudEventAdapterTest.java`

**Interfaces:**
- Consumes: `WorkCloudEventTypes.PREFIX`, `WorkCloudEventTypes.GROUP_PREFIX`, `WorkCloudEventTypes.EXT_TENANCY_ID` (from Task 1)
- Produces: no new public API — behaviour-preserving migration

- [ ] **Step 1: Write a characterisation test asserting existing type strings**

Before changing anything, add a test that verifies the current outbound CloudEvent type format. This locks the contract before migrating to constants.

```java
// In WorkCloudEventAdapterTest — add this test
@Test
void onWorkItemLifecycle_typeMatchesWorkCloudEventTypes() {
    // fire a CREATED event and verify the CloudEvent type matches the constant
    final WorkItem wi = TestWorkItemFactory.minimal();
    final WorkItemLifecycleEvent event = WorkItemLifecycleEvent.of("CREATED", wi, "actor", null);

    adapter.onWorkItemLifecycle(event);

    assertThat(capturedCloudEvent().getType()).isEqualTo(WorkCloudEventTypes.CREATED);
}
```

- [ ] **Step 2: Run to verify it passes (existing behaviour already matches)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCloudEventAdapterTest -pl runtime`
Expected: PASS

- [ ] **Step 3: Migrate `WorkItemLifecycleEvent.of()` to use `WorkCloudEventTypes.PREFIX`**

In `WorkItemLifecycleEvent.java` lines 97 and 121, replace:
```java
"io.casehub.work.workitem." + eventName.toLowerCase(),
```
with:
```java
WorkCloudEventTypes.PREFIX + eventName.toLowerCase(java.util.Locale.ROOT),
```

Add import: `import io.casehub.work.api.WorkCloudEventTypes;`

- [ ] **Step 4: Migrate `WorkCloudEventAdapter.onGroupLifecycle()` to use `WorkCloudEventTypes.GROUP_PREFIX` and `EXT_TENANCY_ID`**

In `WorkCloudEventAdapter.java` line 48, replace:
```java
buildAndFire("io.casehub.work.group." + event.groupStatus().name().toLowerCase(Locale.ROOT),
```
with:
```java
buildAndFire(WorkCloudEventTypes.GROUP_PREFIX + event.groupStatus().name().toLowerCase(Locale.ROOT),
```

In `buildAndFire()` line 76, replace:
```java
builder = builder.withExtension("tenancyid", tenancyId);
```
with:
```java
builder = builder.withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, tenancyId);
```

Add import: `import io.casehub.work.api.WorkCloudEventTypes;`

- [ ] **Step 5: Run all existing adapter tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCloudEventAdapterTest,WorkItemLifecycleEventTest -pl runtime`
Expected: all PASS (behaviour unchanged)

- [ ] **Step 6: Commit**

```
refactor(#172): migrate outbound CloudEvent adapter to WorkCloudEventTypes constants

Replaces hardcoded type strings in WorkItemLifecycleEvent.of() and
WorkCloudEventAdapter with WorkCloudEventTypes constants. No behaviour change.

Refs #172, Refs #92
```

---

### Task 3: Flyway migration — CloudEvent idempotency index (Java migration)

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/migration/V40__CloudEventIdempotencyIndex.java`
- Create: `runtime/src/test/java/io/casehub/work/runtime/migration/CloudEventIdempotencyIndexTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: partial unique index `uq_workitem_cloudevent_idempotency` on `work_item (caller_ref, tenancy_id) WHERE created_by LIKE 'cloudevent:%'` (PostgreSQL only; graceful skip on H2)

Note: H2 does NOT support partial indexes (`CREATE INDEX ... WHERE`). This migration uses a Java-based Flyway migration that checks the database type and applies the partial index only on PostgreSQL. On H2, it logs and skips gracefully. The application-level idempotency (`findByCallerRef`) works on both databases.

- [ ] **Step 1: Write the behavioural idempotency test**

The test verifies idempotency through behaviour, not index metadata — this works on both H2 and PostgreSQL.

```java
package io.casehub.work.runtime.migration;

import io.quarkus.test.junit.QuarkusTest;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.casehub.work.runtime.service.WorkItemService;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class CloudEventIdempotencyIndexTest {

    @Inject WorkItemService workItemService;
    @Inject WorkItemStore workItemStore;

    @Test
    void applicationLevelIdempotency_findByCallerRef_preventsDuplicate() {
        final String callerRef = "ce-" + UUID.randomUUID();
        final WorkItemCreateRequest request = WorkItemCreateRequest.builder()
                .title("Idempotency test")
                .callerRef(callerRef)
                .createdBy("cloudevent:/test")
                .build();

        workItemService.create(request);

        assertThat(workItemService.findByCallerRef(callerRef)).isPresent();

        final long countBefore = workItemStore.scanAll().stream()
                .filter(wi -> callerRef.equals(wi.callerRef))
                .count();
        assertThat(countBefore).isEqualTo(1);
    }
}
```

- [ ] **Step 2: Run to verify it passes (application-level idempotency already works)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CloudEventIdempotencyIndexTest -pl runtime`
Expected: PASS

- [ ] **Step 3: Create the Java Flyway migration**

```java
package io.casehub.work.runtime.migration;

import java.sql.Statement;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import org.jboss.logging.Logger;

public class V40__CloudEventIdempotencyIndex extends BaseJavaMigration {

    private static final Logger LOG = Logger.getLogger(V40__CloudEventIdempotencyIndex.class);

    @Override
    public void migrate(final Context context) throws Exception {
        final String dbProduct = context.getConnection().getMetaData().getDatabaseProductName();

        if ("PostgreSQL".equalsIgnoreCase(dbProduct)) {
            try (Statement stmt = context.getConnection().createStatement()) {
                stmt.execute(
                    "CREATE UNIQUE INDEX uq_workitem_cloudevent_idempotency "
                  + "ON work_item (caller_ref, tenancy_id) "
                  + "WHERE created_by LIKE 'cloudevent:%'");
            }
            LOG.info("V40: Created partial unique index uq_workitem_cloudevent_idempotency (PostgreSQL)");
        } else {
            LOG.infof("V40: Skipping partial unique index — %s does not support WHERE clause on indexes. "
                     + "Application-level idempotency via findByCallerRef is active on all databases.", dbProduct);
        }
    }
}
```

- [ ] **Step 4: Run to verify migration executes without error on H2**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CloudEventIdempotencyIndexTest -pl runtime`
Expected: PASS (migration logs skip message, test still passes)

- [ ] **Step 5: Commit**

```
feat(#172): add CloudEvent idempotency partial unique index (V40)

Java Flyway migration: partial unique index on (caller_ref, tenancy_id)
WHERE created_by LIKE 'cloudevent:%' on PostgreSQL. Graceful skip on H2.
Application-level findByCallerRef idempotency active on all databases.

Refs #172, Refs #92
```

---

### Task 4: Inbound adapter + round-trip integration test

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventInboundAdapter.java`
- Create: `runtime/src/test/java/io/casehub/work/runtime/event/WorkCloudEventInboundAdapterTest.java`
- Create: `runtime/src/test/java/io/casehub/work/runtime/event/WorkCloudEventRoundTripTest.java`

**Interfaces:**
- Consumes: `WorkCloudEventTypes.REQUESTED`, `WorkCloudEventTypes.EXT_TENANCY_ID`, `WorkCloudEventTypes.EXT_TEMPLATE_ID` (Task 1); `WorkItemTemplateService.findByRef(String)`, `WorkItemTemplateService.createFromTemplate(WorkItemCreateRequest)`, `TenantContextRunner.runInTenantContext(String, Runnable)`, `WorkItemService.findByCallerRef(String)`
- Produces: `WorkCloudEventInboundAdapter` CDI bean (`@ApplicationScoped`, `@ObservesAsync CloudEvent`)

- [ ] **Step 1: Write the unit test — happy path (template-based creation)**

```java
package io.casehub.work.runtime.event;

import io.casehub.work.api.WorkCloudEventTypes;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.casehub.work.runtime.service.TenantContextRunner;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.work.runtime.service.WorkItemTemplateService;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class WorkCloudEventInboundAdapterTest {

    private WorkItemTemplateService templateService;
    private WorkItemService workItemService;
    private TenantContextRunner tenantContextRunner;
    private WorkCloudEventInboundAdapter adapter;

    @BeforeEach
    void setUp() {
        templateService = mock(WorkItemTemplateService.class);
        workItemService = mock(WorkItemService.class);
        tenantContextRunner = mock(TenantContextRunner.class);
        doAnswer(inv -> { ((Runnable) inv.getArgument(1)).run(); return null; })
                .when(tenantContextRunner).runInTenantContext(any(), any(Runnable.class));
        adapter = new WorkCloudEventInboundAdapter(templateService, workItemService, tenantContextRunner);
    }

    @Test
    void onCloudEvent_templatePath_createsWorkItem() {
        final UUID templateId = UUID.randomUUID();
        final String tenancy = "tenant-1";
        final String ceId = UUID.randomUUID().toString();
        final String payload = "{\"amount\":500,\"currency\":\"USD\"}";

        final WorkItemTemplate template = new WorkItemTemplate();
        template.id = templateId;
        template.name = "invoice-review";
        when(templateService.findByRef(templateId.toString())).thenReturn(Optional.of(template));

        final WorkItem created = new WorkItem();
        created.id = UUID.randomUUID();
        when(templateService.createFromTemplate(any())).thenReturn(created);
        when(workItemService.findByCallerRef(ceId)).thenReturn(Optional.empty());

        final CloudEvent ce = CloudEventBuilder.v1()
                .withId(ceId)
                .withType(WorkCloudEventTypes.REQUESTED)
                .withSource(URI.create("/workflows/invoice-approval"))
                .withDataContentType("application/json")
                .withData(payload.getBytes(StandardCharsets.UTF_8))
                .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, tenancy)
                .withExtension(WorkCloudEventTypes.EXT_TEMPLATE_ID, templateId.toString())
                .build();

        adapter.onCloudEvent(ce);

        verify(tenantContextRunner).runInTenantContext(eq(tenancy), any(Runnable.class));

        final ArgumentCaptor<WorkItemCreateRequest> captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
        verify(templateService).createFromTemplate(captor.capture());

        final WorkItemCreateRequest req = captor.getValue();
        assertThat(req.templateId).isEqualTo(templateId);
        assertThat(req.payload).isEqualTo(payload);
        assertThat(req.callerRef).isEqualTo(ceId);
        assertThat(req.createdBy).isEqualTo("cloudevent:/workflows/invoice-approval");
    }

    @Test
    void onCloudEvent_ignoresNonRequestedType() {
        final CloudEvent ce = CloudEventBuilder.v1()
                .withId(UUID.randomUUID().toString())
                .withType(WorkCloudEventTypes.COMPLETED)
                .withSource(URI.create("/test"))
                .build();

        adapter.onCloudEvent(ce);

        verifyNoInteractions(templateService, workItemService);
    }

    @Test
    void onCloudEvent_missingTenancyId_rejects() {
        final CloudEvent ce = CloudEventBuilder.v1()
                .withId(UUID.randomUUID().toString())
                .withType(WorkCloudEventTypes.REQUESTED)
                .withSource(URI.create("/test"))
                .withExtension(WorkCloudEventTypes.EXT_TEMPLATE_ID, UUID.randomUUID().toString())
                .build();

        adapter.onCloudEvent(ce);

        verifyNoInteractions(workItemService, templateService);
    }

    @Test
    void onCloudEvent_missingTemplateId_rejects() {
        final CloudEvent ce = CloudEventBuilder.v1()
                .withId(UUID.randomUUID().toString())
                .withType(WorkCloudEventTypes.REQUESTED)
                .withSource(URI.create("/test"))
                .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, "tenant-1")
                .build();

        adapter.onCloudEvent(ce);

        verifyNoInteractions(workItemService, templateService);
    }

    @Test
    void onCloudEvent_duplicateCallerRef_skips() {
        final String ceId = UUID.randomUUID().toString();
        final WorkItem existing = new WorkItem();
        existing.id = UUID.randomUUID();
        when(workItemService.findByCallerRef(ceId)).thenReturn(Optional.of(existing));

        final CloudEvent ce = CloudEventBuilder.v1()
                .withId(ceId)
                .withType(WorkCloudEventTypes.REQUESTED)
                .withSource(URI.create("/test"))
                .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, "tenant-1")
                .withExtension(WorkCloudEventTypes.EXT_TEMPLATE_ID, UUID.randomUUID().toString())
                .build();

        adapter.onCloudEvent(ce);

        verify(templateService, never()).createFromTemplate(any());
    }

    @Test
    void onCloudEvent_templateNotFound_rejects() {
        final String ceId = UUID.randomUUID().toString();
        final String templateRef = UUID.randomUUID().toString();
        when(workItemService.findByCallerRef(ceId)).thenReturn(Optional.empty());
        when(templateService.findByRef(templateRef)).thenReturn(Optional.empty());

        final CloudEvent ce = CloudEventBuilder.v1()
                .withId(ceId)
                .withType(WorkCloudEventTypes.REQUESTED)
                .withSource(URI.create("/test"))
                .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, "tenant-1")
                .withExtension(WorkCloudEventTypes.EXT_TEMPLATE_ID, templateRef)
                .build();

        adapter.onCloudEvent(ce);

        verify(templateService, never()).createFromTemplate(any());
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCloudEventInboundAdapterTest -pl runtime`
Expected: FAIL — `WorkCloudEventInboundAdapter` does not exist

- [ ] **Step 3: Implement `WorkCloudEventInboundAdapter`**

```java
package io.casehub.work.runtime.event;

import java.nio.charset.StandardCharsets;

import io.casehub.work.api.WorkCloudEventTypes;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.casehub.work.runtime.service.TenantContextRunner;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.work.runtime.service.WorkItemTemplateService;
import io.cloudevents.CloudEvent;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

@ApplicationScoped
public class WorkCloudEventInboundAdapter {

    private static final Logger LOG = Logger.getLogger(WorkCloudEventInboundAdapter.class);

    private final WorkItemTemplateService templateService;
    private final WorkItemService workItemService;
    private final TenantContextRunner tenantContextRunner;

    @Inject
    public WorkCloudEventInboundAdapter(final WorkItemTemplateService templateService,
                                         final WorkItemService workItemService,
                                         final TenantContextRunner tenantContextRunner) {
        this.templateService = templateService;
        this.workItemService = workItemService;
        this.tenantContextRunner = tenantContextRunner;
    }

    public void onCloudEvent(@ObservesAsync final CloudEvent ce) {
        if (!WorkCloudEventTypes.REQUESTED.equals(ce.getType())) {
            return;
        }

        final Object tenancyIdExt = ce.getExtension(WorkCloudEventTypes.EXT_TENANCY_ID);
        final Object templateIdExt = ce.getExtension(WorkCloudEventTypes.EXT_TEMPLATE_ID);

        if (tenancyIdExt == null) {
            LOG.errorf("CloudEvent %s from %s rejected: missing tenancyid extension", ce.getId(), ce.getSource());
            return;
        }
        if (templateIdExt == null) {
            LOG.errorf("CloudEvent %s from %s rejected: missing templateid extension", ce.getId(), ce.getSource());
            return;
        }

        final String tenancyId = tenancyIdExt.toString();
        final String templateRef = templateIdExt.toString();

        tenantContextRunner.runInTenantContext(tenancyId, () -> processInTenantContext(ce, templateRef));
    }

    private void processInTenantContext(final CloudEvent ce, final String templateRef) {
        if (workItemService.findByCallerRef(ce.getId()).isPresent()) {
            LOG.debugf("CloudEvent %s already processed — skipping", ce.getId());
            return;
        }

        final WorkItemTemplate template = templateService.findByRef(templateRef).orElse(null);
        if (template == null) {
            LOG.errorf("CloudEvent %s from %s rejected: template '%s' not found",
                    ce.getId(), ce.getSource(), templateRef);
            return;
        }

        final String payload = ce.getData() != null
                ? new String(ce.getData().toBytes(), StandardCharsets.UTF_8)
                : null;

        final WorkItemCreateRequest request = WorkItemCreateRequest.builder()
                .templateId(template.id)
                .payload(payload)
                .callerRef(ce.getId())
                .createdBy("cloudevent:" + ce.getSource())
                .build();

        templateService.createFromTemplate(request);
    }
}
```

- [ ] **Step 4: Run unit tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCloudEventInboundAdapterTest -pl runtime`
Expected: all 6 tests PASS

- [ ] **Step 5: Write the round-trip integration test**

```java
package io.casehub.work.runtime.event;

import io.casehub.work.api.WorkCloudEventTypes;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.casehub.work.runtime.repository.WorkItemTemplateStore;
import io.casehub.work.runtime.service.WorkItemService;
import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@QuarkusTest
class WorkCloudEventRoundTripTest {

    @Inject Event<CloudEvent> cloudEventBus;
    @Inject WorkItemTemplateStore templateStore;
    @Inject WorkItemStore workItemStore;
    @Inject WorkItemService workItemService;
    @Inject TestCloudEventCapture cloudEventCapture;

    @Test
    void roundTrip_requestedCloudEvent_createsWorkItem_completionFiresCloudEvent() {
        // 1. Create a template
        final WorkItemTemplate template = new WorkItemTemplate();
        template.id = UUID.randomUUID();
        template.name = "round-trip-test-" + UUID.randomUUID().toString().substring(0, 8);
        template.tenancyId = "test-tenant";
        template.candidateGroups = "reviewers";
        template.priority = WorkItemPriority.HIGH;
        templateStore.put(template);

        // 2. Fire a REQUESTED CloudEvent
        final String ceId = UUID.randomUUID().toString();
        final String domainPayload = "{\"documentId\":\"doc-42\",\"urgent\":true}";
        final CloudEvent requestedCe = CloudEventBuilder.v1()
                .withId(ceId)
                .withType(WorkCloudEventTypes.REQUESTED)
                .withSource(URI.create("/test/round-trip"))
                .withDataContentType("application/json")
                .withData(domainPayload.getBytes(StandardCharsets.UTF_8))
                .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, "test-tenant")
                .withExtension(WorkCloudEventTypes.EXT_TEMPLATE_ID, template.id.toString())
                .build();

        cloudEventCapture.clear();
        cloudEventBus.fireAsync(requestedCe);

        // 3. Assert WorkItem created
        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            final var wi = workItemService.findByCallerRef(ceId);
            assertThat(wi).isPresent();
            assertThat(wi.get().payload).isEqualTo(domainPayload);
            assertThat(wi.get().callerRef).isEqualTo(ceId);
            assertThat(wi.get().createdBy).isEqualTo("cloudevent:/test/round-trip");
            assertThat(wi.get().candidateGroups).isEqualTo("reviewers");
            assertThat(wi.get().priority).isEqualTo(WorkItemPriority.HIGH);
        });

        // 4. Complete the WorkItem
        final WorkItem wi = workItemService.findByCallerRef(ceId).orElseThrow();
        workItemService.complete(wi.id, "human-actor", "{\"decision\":\"approved\"}", "approved");

        // 5. Assert COMPLETED CloudEvent fired with callerRef for correlation
        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            final var completedEvents = cloudEventCapture.ofType(WorkCloudEventTypes.COMPLETED);
            assertThat(completedEvents).isNotEmpty();
            // The callerRef is in the data payload for correlation
        });

        // 6. Idempotency — fire same CloudEvent again
        cloudEventBus.fireAsync(requestedCe);
        // wait briefly then verify no second WorkItem
        await().during(1, TimeUnit.SECONDS).atMost(3, TimeUnit.SECONDS).untilAsserted(() -> {
            final long count = workItemStore.scanAll().stream()
                    .filter(w -> ceId.equals(w.callerRef))
                    .count();
            assertThat(count).isEqualTo(1);
        });
    }
}
```

This test requires a `TestCloudEventCapture` helper CDI bean. Create it:

```java
package io.casehub.work.runtime.event;

import io.casehub.work.api.WorkCloudEventTypes;
import io.cloudevents.CloudEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

@ApplicationScoped
public class TestCloudEventCapture {

    private final List<CloudEvent> captured = new CopyOnWriteArrayList<>();

    void onCloudEvent(@ObservesAsync final CloudEvent ce) {
        if (ce.getType() != null && ce.getType().startsWith(WorkCloudEventTypes.PREFIX)) {
            captured.add(ce);
        }
    }

    public List<CloudEvent> ofType(final String type) {
        return captured.stream().filter(ce -> type.equals(ce.getType())).toList();
    }

    public void clear() {
        captured.clear();
    }
}
```

- [ ] **Step 6: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCloudEventRoundTripTest -pl runtime`
Expected: PASS

- [ ] **Step 7: Run the full runtime test suite to check for regressions**

Run: `scripts/test-runtime.sh` (or `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`)
Expected: all existing tests still PASS

- [ ] **Step 8: Commit**

```
feat(#172): add inbound CloudEvent adapter with idempotency and round-trip test

WorkCloudEventInboundAdapter observes CloudEvent type
io.casehub.work.workitem.requested, resolves template by ref,
and creates a WorkItem with opaque domain data passthrough.

Idempotency via findByCallerRef + partial unique index (V40).
Round-trip integration test verifies request→create→complete→correlate.

Refs #172, Refs #92
```

---

## Post-Implementation Checklist

- [ ] Run `scripts/test-runtime.sh` — full runtime module green
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api` — api module green
- [ ] Run integration tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests`
- [ ] `superpowers:requesting-code-review` — before committing final
- [ ] `implementation-doc-sync` — update PLATFORM.md capability ownership, ARC42STORIES.MD
- [ ] Update issue #172 with what shipped
