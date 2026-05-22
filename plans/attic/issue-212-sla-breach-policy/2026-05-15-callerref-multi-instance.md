# callerRef Propagation for Multi-Instance Groups — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Propagate `callerRef` from `WorkItemTemplateService.instantiate()` through to the parent `WorkItem` when instantiating multi-instance templates, so `WorkItemGroupLifecycleEvent` carries the correct routing signal for the casehub-engine.

**Architecture:** Add a `callerRef` parameter to `MultiInstanceSpawnService.createGroup()` and set it on the parent `WorkItem`; remove the warn-and-ignore guard in `WorkItemTemplateService.instantiate()` and pass the value through. Children remain callerRef-null — the coordinator's `WorkItemGroupLifecycleEvent` already reads `parent.callerRef` and echoes it correctly.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, JUnit 5 + AssertJ + Awaitility, `@QuarkusTest`

---

## File Map

| File | Action | What changes |
|------|--------|--------------|
| `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java` | Modify | Add `callerRef` param to `createGroup()` and `buildParentRequest()`; set it on parent `WorkItemCreateRequest` |
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java` | Modify | Remove warn-and-ignore block; pass `callerRef` to `createGroup()` |
| `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateInstantiateTest.java` | Modify | Fix assertion on `instantiate_multiInstanceTemplate_ignoresCallerRef` → expect callerRef on parent |
| `runtime/src/test/java/io/casehub/work/runtime/multiinstance/MultiInstanceCoordinatorTest.java` | Modify | Add test: callerRef set on parent only, null on all children |
| `runtime/src/test/java/io/casehub/work/runtime/multiinstance/WorkItemGroupLifecycleEventTest.java` | Modify | Add test: group COMPLETED event carries callerRef |

---

## Task 1: Write the failing tests (RED)

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateInstantiateTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/multiinstance/MultiInstanceCoordinatorTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/multiinstance/WorkItemGroupLifecycleEventTest.java`

- [ ] **Step 1.1: Fix the inverted assertion in `WorkItemTemplateInstantiateTest`**

The existing test `instantiate_multiInstanceTemplate_ignoresCallerRef` currently asserts
`parent.callerRef` is null — it documents broken behaviour. Rename it and invert the assertion:

In `WorkItemTemplateInstantiateTest.java`, replace the test at lines 46–55:

```java
@Test
void instantiate_multiInstanceTemplate_setsCallerRefOnParent() {
    final WorkItemTemplate template = persistedTemplate("Parallel Review", 3);
    final String callerRef = "case:550e8400-e29b-41d4-a716-446655440000/pi:review-gate";

    final WorkItem parent =
        templateService.instantiate(template, null, null, "system:engine", callerRef);

    assertThat(parent).isNotNull();
    assertThat(parent.callerRef).isEqualTo(callerRef);
}
```

- [ ] **Step 1.2: Add `createGroup_withCallerRef_setsOnParentOnly` to `MultiInstanceCoordinatorTest`**

Add this test and the helper method immediately before the `inTx` helpers at the bottom of the class. You will also need to add `import io.casehub.work.runtime.multiinstance.MultiInstanceSpawnService;` at the top and inject the service.

Add to the field declarations (`MultiInstanceSpawnService` is in the same package — no import needed):

```java
@Inject
MultiInstanceSpawnService spawnService;
```

Add the test:

```java
@Test
void createGroup_withCallerRef_setsOnParentOnly() {
    final String callerRef = "case:550e8400-e29b-41d4-a716-446655440000/pi:plan-gate";

    final UUID parentId = inTx(() -> {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "CallerRefTest";
        t.candidateGroups = "testers";
        t.createdBy = "test";
        t.instanceCount = 3;
        t.requiredCount = 2;
        t.persist();
        return spawnService.createGroup(t, null, "test", callerRef).id;
    });

    final WorkItem parent = inTx(() -> WorkItem.findById(parentId));
    assertThat(parent.callerRef).isEqualTo(callerRef);

    final List<WorkItem> children = inTx(() ->
        WorkItem.<WorkItem>list("parentId", parentId));
    assertThat(children).hasSize(3);
    assertThat(children).allSatisfy(child ->
        assertThat(child.callerRef).isNull());
}
```

- [ ] **Step 1.3: Add `completedGroupEvent_carriesCallerRef` to `WorkItemGroupLifecycleEventTest`**

`WorkItemGroupLifecycleEventTest` already has an `EventCapture` bean with `byStatus()`. Add the new
test before the closing `}` of the test class (before the `EventCapture` inner class):

First add import `import java.util.List;` if not already present (check line 1–25).

Then add the test:

```java
@Test
void completedGroupEvent_carriesCallerRef() {
    final String callerRef = "case:550e8400-e29b-41d4-a716-446655440000/pi:ethics-gate";

    final UUID parentId = inTx(() -> {
        WorkItemTemplate t = new WorkItemTemplate();
        t.name = "CallerRefEventTest";
        t.candidateGroups = "g";
        t.createdBy = "test";
        t.instanceCount = 2;
        t.requiredCount = 2;
        t.persist();
        return templateService.instantiate(t, null, null, "test", callerRef).id;
    });

    final List<UUID> childIds = inTx(() ->
        WorkItem.<WorkItem>list("parentId", parentId).stream()
            .map(w -> w.id).toList());

    inTx(() -> workItemService.claim(childIds.get(0), "alice"));
    inTx(() -> workItemService.start(childIds.get(0), "alice"));
    inTx(() -> workItemService.complete(childIds.get(0), "alice", "approved"));

    inTx(() -> workItemService.claim(childIds.get(1), "bob"));
    inTx(() -> workItemService.start(childIds.get(1), "bob"));
    inTx(() -> workItemService.complete(childIds.get(1), "bob", "approved"));

    Awaitility.await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
        final List<WorkItemGroupLifecycleEvent> completed =
            capture.byStatus(parentId, GroupStatus.COMPLETED);
        assertThat(completed).hasSize(1);
        assertThat(completed.get(0).callerRef()).isEqualTo(callerRef);
    });
}
```

- [ ] **Step 1.4: Run the tests to confirm they FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="WorkItemTemplateInstantiateTest,MultiInstanceCoordinatorTest,WorkItemGroupLifecycleEventTest" 2>&1 | grep -E "FAIL|ERROR|Tests run:"
```

Expected: three test failures —
- `instantiate_multiInstanceTemplate_setsCallerRefOnParent`: `expected: "case:…/pi:review-gate" but was: null`
- `createGroup_withCallerRef_setsOnParentOnly`: compile error — `createGroup` has no 4-arg overload yet
- `completedGroupEvent_carriesCallerRef`: compile error or null callerRef assertion failure

---

## Task 2: Implement `MultiInstanceSpawnService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java`

- [ ] **Step 2.1: Add `callerRef` parameter to `createGroup()` and `buildParentRequest()`**

Replace the `createGroup` method signature and its call to `buildParentRequest`:

```java
@Transactional
public WorkItem createGroup(final WorkItemTemplate template, final String titleOverride,
        final String createdBy, final String callerRef) {
    final boolean isCoordinator = template.parentRole == null
            || ParentRole.COORDINATOR.name().equals(template.parentRole);

    final WorkItemCreateRequest parentReq = buildParentRequest(template, titleOverride, createdBy, isCoordinator, callerRef);
    final WorkItem parent = workItemService.create(parentReq);
    // ... rest of method unchanged
```

Replace the `buildParentRequest` method signature and the `null, // callerRef` line:

```java
private WorkItemCreateRequest buildParentRequest(final WorkItemTemplate template,
        final String titleOverride, final String createdBy, final boolean isCoordinator,
        final String callerRef) {
    final String title = (titleOverride != null && !titleOverride.isBlank())
            ? titleOverride
            : template.name;
    return new WorkItemCreateRequest(
            title,
            template.description,
            template.category,
            null, // formKey — not templated
            template.priority,
            null, // assigneeId — coordinator has none; participant uses candidateGroups routing
            isCoordinator ? null : template.candidateGroups,
            isCoordinator ? null : template.candidateUsers,
            template.requiredCapabilities,
            createdBy,
            template.defaultPayload,
            null, // claimDeadline
            null, // expiresAt
            null, // followUpDate
            null, // labels — applied separately if needed
            null, // confidenceScore
            callerRef,
            null, // defaultClaimBusinessHours — coordinator has no deadline
            isCoordinator ? null : template.defaultExpiryBusinessHours);
}
```

`buildChildRequest()` is unchanged — children remain callerRef-null.

- [ ] **Step 2.2: Run the target tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="WorkItemTemplateInstantiateTest,MultiInstanceCoordinatorTest,WorkItemGroupLifecycleEventTest" 2>&1 | grep -E "FAIL|ERROR|Tests run:"
```

Expected: `createGroup_withCallerRef_setsOnParentOnly` now PASSES. The other two still FAIL —
`WorkItemTemplateService` still drops the callerRef.

---

## Task 3: Implement `WorkItemTemplateService` — pass callerRef through

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`

- [ ] **Step 3.1: Remove the warn-and-ignore block and pass callerRef through**

In the 5-arg `instantiate()` method, replace the multi-instance guard block (currently lines 100–107):

```java
// BEFORE (remove this):
if (template.instanceCount != null) {
    if (callerRef != null) {
        LOG.warnf(
            "callerRef '%s' ignored for multi-instance template %s — not yet supported (see casehubio/work#166)",
            callerRef, template.id);
    }
    return multiInstanceSpawnService.get().createGroup(template, titleOverride, createdBy);
}
```

```java
// AFTER:
if (template.instanceCount != null) {
    return multiInstanceSpawnService.get().createGroup(template, titleOverride, createdBy, callerRef);
}
```

- [ ] **Step 3.2: Run all three target tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="WorkItemTemplateInstantiateTest,MultiInstanceCoordinatorTest,WorkItemGroupLifecycleEventTest" 2>&1 | grep -E "FAIL|ERROR|Tests run:|BUILD"
```

Expected: all three test classes pass, `BUILD SUCCESS`.

- [ ] **Step 3.3: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: `BUILD SUCCESS`. No regressions.

- [ ] **Step 3.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java \
  runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java \
  runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateInstantiateTest.java \
  runtime/src/test/java/io/casehub/work/runtime/multiinstance/MultiInstanceCoordinatorTest.java \
  runtime/src/test/java/io/casehub/work/runtime/multiinstance/WorkItemGroupLifecycleEventTest.java

git -C /Users/mdproctor/claude/casehub/work commit -m "$(cat <<'EOF'
feat(multiinstance): propagate callerRef to parent WorkItem in createGroup() (Closes #166)

WorkItemTemplateService.instantiate() now passes callerRef through to
MultiInstanceSpawnService.createGroup(). The parent WorkItem carries the
callerRef; children remain null. WorkItemGroupLifecycleEvent already reads
parent.callerRef, so the engine routing signal is correct once parent is set.

Refs #106
EOF
)"
```
