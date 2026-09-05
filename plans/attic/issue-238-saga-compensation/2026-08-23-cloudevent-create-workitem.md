# CloudEvent CREATE — Full-Request WorkItem Creation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #299 — CloudEvent bridge for cross-service HumanTask creation
**Issue group:** #299

**Goal:** Add a `io.casehub.work.workitem.create` CloudEvent type that creates WorkItems with full `WorkItemCreateRequest` control, enabling distributed HumanTask patterns without engine co-location.

**Architecture:** Add `CREATE` constant to `WorkCloudEventTypes` (api module). Add a second `@ObservesAsync CloudEvent` method to the existing `WorkCloudEventInboundAdapter` (runtime module) that deserializes CloudEvent data via `JsonNode` → `WorkItemCreateRequest.Builder`, overrides `createdBy`/`callerRef`/`tenancyId`, and routes to template or direct creation.

**Tech Stack:** Java 21, Quarkus 3.32.2, CloudEvents SDK 4.x, Jackson, CDI

## Global Constraints

- Java 21 source (on Java 26 JVM)
- `WorkItemCreateRequest` has no Jackson annotations — use `JsonNode` → Builder, never direct deserialization
- `createdBy` always `"cloudevent:" + ce.getSource()` — required for partial unique index coverage
- `tenancyId` on request always set to `null` — tenant context via `TenantContextRunner` from extension
- Test with `scripts/mvn-test <module>` — enforces 90s timeout
- IntelliJ MCP mandatory for all `.java` edits — use `ide_insert_member`, `ide_replace_member`, `ide_edit_member`

---

## Batch 1: CREATE type constant and inbound consumer

### Task 1: Add CREATE constant and drift prevention test

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/WorkCloudEventTypes.java:11` — add constant
- Modify: `api/src/test/java/io/casehub/work/api/WorkCloudEventTypesTest.java` — add drift test

**Interfaces:**
- Produces: `WorkCloudEventTypes.CREATE` = `"io.casehub.work.workitem.create"` — used by Task 2 for type filtering

- [ ] **Step 1: Write the drift prevention test**

Add to `WorkCloudEventTypesTest`:

```java
@Test
void create_isNotInWorkEventType() {
    assertThat(WorkCloudEventTypes.CREATE)
            .isEqualTo("io.casehub.work.workitem.create");
    for (final WorkEventType type : WorkEventType.values()) {
        assertThat(type.name()).isNotEqualToIgnoringCase("create");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `scripts/mvn-test casehub-work-api -Dtest=WorkCloudEventTypesTest#create_isNotInWorkEventType`
Expected: FAIL — `WorkCloudEventTypes.CREATE` does not exist

- [ ] **Step 3: Add CREATE constant**

Add to `WorkCloudEventTypes.java` after the `REQUESTED` constant (line 11):

```java
public static final String CREATE = PREFIX + "create";
```

- [ ] **Step 4: Run full test class**

Run: `scripts/mvn-test casehub-work-api -Dtest=WorkCloudEventTypesTest`
Expected: ALL PASS (4 tests — existing 3 + new 1)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add api/src/main/java/io/casehub/work/api/WorkCloudEventTypes.java api/src/test/java/io/casehub/work/api/WorkCloudEventTypesTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(api): add CREATE CloudEvent type constant with drift prevention test Refs #299"
```

---

### Task 2: Add CREATE handler to WorkCloudEventInboundAdapter

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventInboundAdapter.java` — add ObjectMapper injection, add `onCreateCloudEvent` method, add `buildRequestFromJson` helper
- Modify: `runtime/src/test/java/io/casehub/work/runtime/event/WorkCloudEventInboundAdapterTest.java` — add CREATE path tests

**Interfaces:**
- Consumes: `WorkCloudEventTypes.CREATE` (from Task 1)
- Consumes: `WorkItemService.create(WorkItemCreateRequest)`, `WorkItemService.findByCallerRef(String)`, `WorkItemTemplateService.createFromTemplate(WorkItemCreateRequest)`, `TenantContextRunner.runInTenantContext(String, Runnable)` — all existing
- Produces: WorkItem creation via CloudEvent (no downstream consumers in this issue)

- [ ] **Step 1: Write failing test — happy path inline creation**

Add to `WorkCloudEventInboundAdapterTest`. First update `setUp()` to add ObjectMapper and update constructor call:

```java
private ObjectMapper objectMapper;

@BeforeEach
void setUp() {
    templateService = mock(WorkItemTemplateService.class);
    workItemService = mock(WorkItemService.class);
    tenantContextRunner = mock(TenantContextRunner.class);
    objectMapper = new ObjectMapper();
    doAnswer(inv -> { ((Runnable) inv.getArgument(1)).run(); return null; })
            .when(tenantContextRunner).runInTenantContext(any(), any(Runnable.class));
    adapter = new WorkCloudEventInboundAdapter(templateService, workItemService, tenantContextRunner, objectMapper);
}
```

Then add the test:

```java
@Test
void onCreateCloudEvent_inlinePath_createsWorkItem() {
    final String tenancy = "tenant-1";
    final String ceId = UUID.randomUUID().toString();
    final String callerRef = "case:abc-123/pi:def-456";
    final String data = """
            {"title":"Review document","candidateGroups":"legal,compliance",\
            "callerRef":"%s","payload":"{\\"docId\\":\\"d1\\"}",\
            "scope":"app/legal","candidateScores":"{\\"alice\\":0.95}"}""".formatted(callerRef);

    when(workItemService.findByCallerRef(callerRef)).thenReturn(Optional.empty());
    final WorkItem created = WorkItem.builder().id(UUID.randomUUID()).build();
    when(workItemService.create(any())).thenReturn(created);

    final CloudEvent ce = CloudEventBuilder.v1()
            .withId(ceId)
            .withType(WorkCloudEventTypes.CREATE)
            .withSource(URI.create("/engine/cases/abc-123"))
            .withDataContentType("application/json")
            .withData(data.getBytes(StandardCharsets.UTF_8))
            .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, tenancy)
            .build();

    adapter.onCreateCloudEvent(ce);

    verify(tenantContextRunner).runInTenantContext(eq(tenancy), any(Runnable.class));

    final ArgumentCaptor<WorkItemCreateRequest> captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(workItemService).create(captor.capture());

    final WorkItemCreateRequest req = captor.getValue();
    assertThat(req.title).isEqualTo("Review document");
    assertThat(req.candidateGroups).isEqualTo("legal,compliance");
    assertThat(req.callerRef).isEqualTo(callerRef);
    assertThat(req.createdBy).isEqualTo("cloudevent:/engine/cases/abc-123");
    assertThat(req.payload).isEqualTo("{\"docId\":\"d1\"}");
    assertThat(req.scope).isEqualTo("app/legal");
    assertThat(req.candidateScores).isEqualTo("{\"alice\":0.95}");
    assertThat(req.tenancyId).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `scripts/mvn-compile runtime`
Expected: FAIL — `onCreateCloudEvent` method does not exist, constructor signature mismatch

- [ ] **Step 3: Implement the handler**

Update `WorkCloudEventInboundAdapter`:

1. Add `ObjectMapper` field and update constructor to accept it as fourth parameter.

2. Add `onCreateCloudEvent(@ObservesAsync CloudEvent ce)` method:

```java
public void onCreateCloudEvent(@ObservesAsync final CloudEvent ce) {
    if (!WorkCloudEventTypes.CREATE.equals(ce.getType())) {
        return;
    }

    final Object tenancyIdExt = ce.getExtension(WorkCloudEventTypes.EXT_TENANCY_ID);
    if (tenancyIdExt == null) {
        LOG.errorf("CloudEvent %s from %s rejected: missing tenancyid extension", ce.getId(), ce.getSource());
        return;
    }

    final String tenancyId = tenancyIdExt.toString();
    tenantContextRunner.runInTenantContext(tenancyId, () -> processCreateInTenantContext(ce));
}
```

3. Add `processCreateInTenantContext(CloudEvent ce)` method:

```java
private void processCreateInTenantContext(final CloudEvent ce) {
    final JsonNode root;
    try {
        root = parseData(ce);
    } catch (final Exception e) {
        LOG.errorf("CloudEvent %s from %s rejected: malformed data — %s", ce.getId(), ce.getSource(), e.getMessage());
        return;
    }
    if (root == null) {
        LOG.errorf("CloudEvent %s from %s rejected: null data", ce.getId(), ce.getSource());
        return;
    }

    final String callerRef = root.has("callerRef") && !root.get("callerRef").isNull()
            ? root.get("callerRef").asText()
            : ce.getId();

    if (workItemService.findByCallerRef(callerRef).isPresent()) {
        LOG.debugf("CloudEvent %s already processed (callerRef=%s) — skipping", ce.getId(), callerRef);
        return;
    }

    final WorkItemCreateRequest request;
    try {
        request = buildRequestFromJson(root, ce, callerRef);
    } catch (final IllegalArgumentException e) {
        LOG.errorf("CloudEvent %s from %s rejected: invalid request — %s", ce.getId(), ce.getSource(), e.getMessage());
        return;
    }

    try {
        if (request.templateId != null) {
            templateService.createFromTemplate(request);
        } else {
            workItemService.create(request);
        }
    } catch (final jakarta.persistence.PersistenceException e) {
        if (isUniqueConstraintViolation(e)) {
            LOG.debugf("CloudEvent %s — concurrent duplicate caught by database constraint", ce.getId());
            return;
        }
        throw e;
    }
}
```

4. Add `parseData(CloudEvent ce)` helper:

```java
private JsonNode parseData(final CloudEvent ce) throws Exception {
    if (ce.getData() == null) return null;
    return objectMapper.readTree(ce.getData().toBytes());
}
```

5. Add `buildRequestFromJson(JsonNode root, CloudEvent ce, String callerRef)` helper:

```java
private WorkItemCreateRequest buildRequestFromJson(final JsonNode root, final CloudEvent ce, final String callerRef) {
    final WorkItemCreateRequest.Builder b = WorkItemCreateRequest.builder();

    ifText(root, "title", b::title);
    ifText(root, "description", b::description);
    ifText(root, "formKey", b::formKey);
    ifText(root, "assigneeId", b::assigneeId);
    ifText(root, "candidateGroups", b::candidateGroups);
    ifText(root, "candidateUsers", b::candidateUsers);
    ifText(root, "requiredCapabilities", b::requiredCapabilities);
    ifText(root, "payload", b::payload);
    ifText(root, "scope", b::scope);
    ifText(root, "payloadTypeName", b::payloadTypeName);
    ifText(root, "resolutionTypeName", b::resolutionTypeName);
    ifText(root, "candidateScores", b::candidateScores);
    ifText(root, "routingExperiences", b::routingExperiences);
    ifText(root, "routingStrategy", b::routingStrategy);
    ifText(root, "excludedUsers", b::excludedUsers);
    ifText(root, "inputDataSchema", b::inputDataSchema);
    ifText(root, "outputDataSchema", b::outputDataSchema);

    if (root.has("priority") && !root.get("priority").isNull()) {
        b.priority(WorkItemPriority.valueOf(root.get("priority").asText()));
    }
    if (root.has("templateId") && !root.get("templateId").isNull()) {
        b.templateId(UUID.fromString(root.get("templateId").asText()));
    }
    if (root.has("expiresAt") && !root.get("expiresAt").isNull()) {
        b.expiresAt(Instant.parse(root.get("expiresAt").asText()));
    }
    if (root.has("claimDeadline") && !root.get("claimDeadline").isNull()) {
        b.claimDeadline(Instant.parse(root.get("claimDeadline").asText()));
    }
    if (root.has("followUpDate") && !root.get("followUpDate").isNull()) {
        b.followUpDate(Instant.parse(root.get("followUpDate").asText()));
    }
    if (root.has("claimDeadlineBusinessHours") && !root.get("claimDeadlineBusinessHours").isNull()) {
        b.claimDeadlineBusinessHours(root.get("claimDeadlineBusinessHours").intValue());
    }
    if (root.has("expiresAtBusinessHours") && !root.get("expiresAtBusinessHours").isNull()) {
        b.expiresAtBusinessHours(root.get("expiresAtBusinessHours").intValue());
    }
    if (root.has("minimumScore") && !root.get("minimumScore").isNull()) {
        b.minimumScore(root.get("minimumScore").doubleValue());
    }
    if (root.has("confidenceScore") && !root.get("confidenceScore").isNull()) {
        b.confidenceScore(root.get("confidenceScore").doubleValue());
    }
    if (root.has("types") && root.get("types").isArray()) {
        final List<String> types = new ArrayList<>();
        root.get("types").forEach(n -> types.add(n.asText()));
        b.types(types);
    }
    if (root.has("permittedOutcomes") && root.get("permittedOutcomes").isArray()) {
        final List<Outcome> outcomes = new ArrayList<>();
        for (final JsonNode n : root.get("permittedOutcomes")) {
            outcomes.add(new Outcome(
                    n.has("name") ? n.get("name").asText() : null,
                    n.has("displayName") ? n.get("displayName").asText() : null,
                    n.has("condition") ? n.get("condition").asText() : null));
        }
        b.permittedOutcomes(outcomes);
    }
    if (root.has("labels") && root.get("labels").isArray()) {
        final List<WorkItemLabelRequest> labels = new ArrayList<>();
        for (final JsonNode n : root.get("labels")) {
            final LabelPersistence persistence = n.has("persistence") && !n.get("persistence").isNull()
                    ? LabelPersistence.valueOf(n.get("persistence").asText())
                    : LabelPersistence.MANUAL;
            labels.add(new WorkItemLabelRequest(
                    n.has("path") ? n.get("path").asText() : null,
                    persistence,
                    n.has("appliedBy") ? n.get("appliedBy").asText() : null));
        }
        b.labels(labels);
    }

    // Overrides — these always take precedence over data payload values
    b.createdBy("cloudevent:" + ce.getSource());
    b.callerRef(callerRef);
    b.tenancyId(null);

    return b.build();
}

private static void ifText(final JsonNode root, final String field, final java.util.function.Consumer<String> setter) {
    if (root.has(field) && !root.get(field).isNull()) {
        setter.accept(root.get(field).asText());
    }
}
```

- [ ] **Step 4: Run the inline creation test**

Run: `scripts/mvn-test runtime -Dtest=WorkCloudEventInboundAdapterTest#onCreateCloudEvent_inlinePath_createsWorkItem`
Expected: PASS

- [ ] **Step 5: Write remaining unit tests**

Add all remaining tests to `WorkCloudEventInboundAdapterTest`:

```java
@Test
void onCreateCloudEvent_templatePath_createsViaTemplate() {
    final UUID templateId = UUID.randomUUID();
    final String callerRef = "workflow:run-1/step-3";
    final String data = """
            {"templateId":"%s","payload":"{\\"doc\\":\\"d1\\"}","callerRef":"%s"}"""
            .formatted(templateId, callerRef);

    when(workItemService.findByCallerRef(callerRef)).thenReturn(Optional.empty());
    final WorkItemTemplate template = new WorkItemTemplate();
    template.id = templateId;
    when(templateService.findByRef(templateId.toString())).thenReturn(Optional.of(template));
    final WorkItem created = WorkItem.builder().id(UUID.randomUUID()).build();
    when(templateService.createFromTemplate(any())).thenReturn(created);

    adapter.onCreateCloudEvent(buildCreateEvent(data, "tenant-1"));

    final ArgumentCaptor<WorkItemCreateRequest> captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(templateService).createFromTemplate(captor.capture());
    assertThat(captor.getValue().templateId).isEqualTo(templateId);
    assertThat(captor.getValue().callerRef).isEqualTo(callerRef);
    verify(workItemService, never()).create(any());
}

@Test
void onCreateCloudEvent_missingTenancyId_rejects() {
    final CloudEvent ce = CloudEventBuilder.v1()
            .withId(UUID.randomUUID().toString())
            .withType(WorkCloudEventTypes.CREATE)
            .withSource(URI.create("/test"))
            .withData("application/json", "{\"title\":\"t\"}".getBytes(StandardCharsets.UTF_8))
            .build();

    adapter.onCreateCloudEvent(ce);

    verifyNoInteractions(workItemService, templateService);
}

@Test
void onCreateCloudEvent_callerRefFallsBackToCeId() {
    final String ceId = UUID.randomUUID().toString();
    final String data = "{\"title\":\"No callerRef\"}";

    when(workItemService.findByCallerRef(ceId)).thenReturn(Optional.empty());
    final WorkItem created = WorkItem.builder().id(UUID.randomUUID()).build();
    when(workItemService.create(any())).thenReturn(created);

    adapter.onCreateCloudEvent(buildCreateEvent(ceId, data, "tenant-1"));

    final ArgumentCaptor<WorkItemCreateRequest> captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(workItemService).create(captor.capture());
    assertThat(captor.getValue().callerRef).isEqualTo(ceId);
}

@Test
void onCreateCloudEvent_duplicateCallerRef_skips() {
    final String callerRef = "case:abc/pi:def";
    final String data = "{\"title\":\"t\",\"callerRef\":\"" + callerRef + "\"}";

    when(workItemService.findByCallerRef(callerRef))
            .thenReturn(Optional.of(WorkItem.builder().id(UUID.randomUUID()).build()));

    adapter.onCreateCloudEvent(buildCreateEvent(data, "tenant-1"));

    verify(workItemService, never()).create(any());
    verify(templateService, never()).createFromTemplate(any());
}

@Test
void onCreateCloudEvent_createdByOverridden() {
    final String data = "{\"title\":\"t\",\"createdBy\":\"malicious-source\"}";

    when(workItemService.findByCallerRef(any())).thenReturn(Optional.empty());
    final WorkItem created = WorkItem.builder().id(UUID.randomUUID()).build();
    when(workItemService.create(any())).thenReturn(created);

    adapter.onCreateCloudEvent(buildCreateEvent(data, "tenant-1"));

    final ArgumentCaptor<WorkItemCreateRequest> captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(workItemService).create(captor.capture());
    assertThat(captor.getValue().createdBy).startsWith("cloudevent:");
}

@Test
void onCreateCloudEvent_tenancyIdInDataIgnored() {
    final String data = "{\"title\":\"t\",\"tenancyId\":\"tenant-B\"}";

    when(workItemService.findByCallerRef(any())).thenReturn(Optional.empty());
    final WorkItem created = WorkItem.builder().id(UUID.randomUUID()).build();
    when(workItemService.create(any())).thenReturn(created);

    adapter.onCreateCloudEvent(buildCreateEvent(data, "tenant-A"));

    verify(tenantContextRunner).runInTenantContext(eq("tenant-A"), any(Runnable.class));

    final ArgumentCaptor<WorkItemCreateRequest> captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(workItemService).create(captor.capture());
    assertThat(captor.getValue().tenancyId).isNull();
}

@Test
void onCreateCloudEvent_nonCreateType_ignored() {
    final CloudEvent ce = CloudEventBuilder.v1()
            .withId(UUID.randomUUID().toString())
            .withType(WorkCloudEventTypes.COMPLETED)
            .withSource(URI.create("/test"))
            .build();

    adapter.onCreateCloudEvent(ce);

    verifyNoInteractions(workItemService, templateService);
}

@Test
void onCreateCloudEvent_malformedData_rejects() {
    final CloudEvent ce = CloudEventBuilder.v1()
            .withId(UUID.randomUUID().toString())
            .withType(WorkCloudEventTypes.CREATE)
            .withSource(URI.create("/test"))
            .withData("application/json", "not-json".getBytes(StandardCharsets.UTF_8))
            .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, "tenant-1")
            .build();

    adapter.onCreateCloudEvent(ce);

    verify(workItemService, never()).create(any());
}

@Test
void onCreateCloudEvent_nullData_rejects() {
    final CloudEvent ce = CloudEventBuilder.v1()
            .withId(UUID.randomUUID().toString())
            .withType(WorkCloudEventTypes.CREATE)
            .withSource(URI.create("/test"))
            .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, "tenant-1")
            .build();

    adapter.onCreateCloudEvent(ce);

    verify(workItemService, never()).create(any());
}

@Test
void onCreateCloudEvent_constraintViolation_idempotentSuccess() {
    final String data = "{\"title\":\"t\"}";

    when(workItemService.findByCallerRef(any())).thenReturn(Optional.empty());
    when(workItemService.create(any())).thenThrow(
            new jakarta.persistence.PersistenceException("dup",
                    new org.hibernate.exception.ConstraintViolationException("dup", null, "uq_workitem")));

    adapter.onCreateCloudEvent(buildCreateEvent(data, "tenant-1"));

    verify(workItemService).create(any());
    // No exception propagated — constraint violation treated as idempotent success
}

@Test
void onCreateCloudEvent_permittedOutcomesMapping() {
    final String data = """
            {"title":"t","permittedOutcomes":[{"name":"approve","displayName":"Approve It","condition":"workItem.priority == 'HIGH'"},{"name":"reject"}]}""";

    when(workItemService.findByCallerRef(any())).thenReturn(Optional.empty());
    final WorkItem created = WorkItem.builder().id(UUID.randomUUID()).build();
    when(workItemService.create(any())).thenReturn(created);

    adapter.onCreateCloudEvent(buildCreateEvent(data, "tenant-1"));

    final ArgumentCaptor<WorkItemCreateRequest> captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(workItemService).create(captor.capture());
    final var outcomes = captor.getValue().permittedOutcomes;
    assertThat(outcomes).hasSize(2);
    assertThat(outcomes.get(0).name()).isEqualTo("approve");
    assertThat(outcomes.get(0).displayName()).isEqualTo("Approve It");
    assertThat(outcomes.get(0).condition()).isEqualTo("workItem.priority == 'HIGH'");
    assertThat(outcomes.get(1).name()).isEqualTo("reject");
    assertThat(outcomes.get(1).displayName()).isNull();
}

@Test
void onCreateCloudEvent_labelsMapping() {
    final String data = """
            {"title":"t","labels":[{"path":"priority/high","persistence":"MANUAL","appliedBy":"system"},{"path":"auto/flagged"}]}""";

    when(workItemService.findByCallerRef(any())).thenReturn(Optional.empty());
    final WorkItem created = WorkItem.builder().id(UUID.randomUUID()).build();
    when(workItemService.create(any())).thenReturn(created);

    adapter.onCreateCloudEvent(buildCreateEvent(data, "tenant-1"));

    final ArgumentCaptor<WorkItemCreateRequest> captor = ArgumentCaptor.forClass(WorkItemCreateRequest.class);
    verify(workItemService).create(captor.capture());
    final var labels = captor.getValue().labels;
    assertThat(labels).hasSize(2);
    assertThat(labels.get(0).path()).isEqualTo("priority/high");
    assertThat(labels.get(0).persistence()).isEqualTo(LabelPersistence.MANUAL);
    assertThat(labels.get(0).appliedBy()).isEqualTo("system");
    assertThat(labels.get(1).path()).isEqualTo("auto/flagged");
    assertThat(labels.get(1).persistence()).isEqualTo(LabelPersistence.MANUAL); // default
}

// --- helper ---

private CloudEvent buildCreateEvent(final String data, final String tenancy) {
    return buildCreateEvent(UUID.randomUUID().toString(), data, tenancy);
}

private CloudEvent buildCreateEvent(final String ceId, final String data, final String tenancy) {
    return CloudEventBuilder.v1()
            .withId(ceId)
            .withType(WorkCloudEventTypes.CREATE)
            .withSource(URI.create("/engine/cases/abc-123"))
            .withDataContentType("application/json")
            .withData(data.getBytes(StandardCharsets.UTF_8))
            .withExtension(WorkCloudEventTypes.EXT_TENANCY_ID, tenancy)
            .build();
}
```

- [ ] **Step 6: Run all tests**

Run: `scripts/mvn-test runtime -Dtest=WorkCloudEventInboundAdapterTest`
Expected: ALL PASS (6 existing + 13 new = 19 tests)

- [ ] **Step 7: Run full runtime test suite for regressions**

Run: `scripts/mvn-test runtime`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventInboundAdapter.java runtime/src/test/java/io/casehub/work/runtime/event/WorkCloudEventInboundAdapterTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(runtime): add CREATE CloudEvent handler for full-request WorkItem creation Refs #299"
```

---

## References

- [2026-08-23-cloudevent-create-workitem-design.md] — design spec this plan implements
- [runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventInboundAdapter.java] — existing REQUESTED handler
- [runtime/src/test/java/io/casehub/work/runtime/event/WorkCloudEventInboundAdapterTest.java] — existing test patterns
- [api/src/main/java/io/casehub/work/api/WorkCloudEventTypes.java] — type constants
- [api/src/test/java/io/casehub/work/api/WorkCloudEventTypesTest.java] — drift prevention test pattern
- [api/src/main/java/io/casehub/work/api/WorkItemCreateRequest.java] — field mapping source
- [api/src/main/java/io/casehub/work/api/Outcome.java] — `Outcome(name, displayName, condition)`
- [api/src/main/java/io/casehub/work/api/WorkItemLabelRequest.java] — `WorkItemLabelRequest(path, persistence, appliedBy)`
- [async-event-tenant-context-propagation protocol] — tenant context in @ObservesAsync
- [GitHub #299] — focal issue
