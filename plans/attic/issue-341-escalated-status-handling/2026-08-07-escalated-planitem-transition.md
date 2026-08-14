# ESCALATED PlanItem Transition — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #341 — PlanItemCompletionApplier does not handle ESCALATED WorkItem status
**Issue group:** #341

**Goal:** Route ESCALATED WorkItem events through the normal terminal path so PlanItems transition to FAULTED, the escalation signal is written, and gates fire expiry events.

**Architecture:** Remove the ESCALATED intercept from `WorkItemLifecycleAdapter` (lines 77-79) and delete `handleEscalation()`. Add `case ESCALATED` to `PlanItemCompletionApplier.applyStatus()` and `ActionGateCompletionApplier.apply()`. Add resolution validation bypass and escalation signal writing to the applier.

**Tech Stack:** Java 21, Quarkus 3.32, JUnit 5, AssertJ, Awaitility

## Global Constraints

- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Test with `scripts/mvn-test casehub-work-engine-adapter`
- All commits reference `Refs #341`
- Use `mcp__intellij-index__*` tools for code navigation and editing

---

### Task 1: Add ESCALATED to PlanItemCompletionApplier

**Files:**
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java`
- Create: `engine-adapter/src/test/java/io/casehub/work/engine/PlanItemCompletionApplierTest.java`

**Interfaces:**
- Consumes: `PlanItemCompletionApplier.apply(UUID, String, WorkItemStatus, WorkItemRef)` — existing public API
- Produces: `writeEscalationSignal(CaseInstance, PlanItem, WorkItemRef)` — new private method; writes `workItemEscalated` signal to CaseContext

- [ ] **Step 1: Write the failing test — ESCALATED transitions PlanItem to FAULTED**

Create `PlanItemCompletionApplierTest.java`:

```java
package io.casehub.work.engine;

import io.casehub.api.model.HumanTaskTarget;
import io.casehub.api.model.TaskStatus;
import io.casehub.engine.planning.plan.PlanItem;
import io.casehub.engine.planning.registry.BlackboardRegistry;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.internal.context.CaseContextImpl;
import io.casehub.work.api.WorkItemRef;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.api.spi.WorkloadProvider;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class PlanItemCompletionApplierTest {

    @Alternative @Priority(1) @ApplicationScoped
    static class StubWorkloadProvider implements WorkloadProvider {
        @Override public int getActiveWorkCount(String workerId) { return 0; }
    }

    @Inject BlackboardRegistry registry;
    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject PlanItemCompletionApplier applier;

    private UUID caseId;
    private PlanItem planItem;
    private String planItemId;

    @BeforeEach
    void setUp() {
        caseId = UUID.randomUUID();
        planItem = PlanItem.create("escalation-binding",
                io.casehub.api.model.ExecutorRef.of("ht-worker"), 10,
                HumanTaskTarget.inline().title("Review").build());
        planItem.tryMarkDispatching();
        planItem.markDelegated();
        planItemId = planItem.getPlanItemId();
        registry.getOrCreate(caseId, "test-tenant").addPlanItem(planItem);

        CaseInstance instance = new CaseInstance();
        instance.setUuid(caseId);
        instance.setState(io.casehub.api.model.CaseStatus.RUNNING);
        instance.setCaseContext(new CaseContextImpl(Map.of("stage", "review")));
        caseInstanceRepository.save(instance, "test-tenant");
    }

    @AfterEach
    void tearDown() {
        registry.evict(caseId);
    }

    @Test
    void escalated_marksFaulted_writesSignal_skipsOutputMapping() {
        UUID workItemId = UUID.randomUUID();
        WorkItemRef ref = new WorkItemRef(
                workItemId, WorkItemStatus.ESCALATED,
                PlanItemCallerRef.encode(caseId, planItemId),
                null, null, "committee-a,committee-b",
                null, "test-tenant", null, null, null);

        applier.apply(caseId, planItemId, WorkItemStatus.ESCALATED, ref);

        assertThat(planItem.getStatus()).isEqualTo(TaskStatus.FAULTED);

        CaseInstance updated = caseInstanceRepository.findByUuid(caseId, "test-tenant");
        Object signal = updated.getCaseContext().get("workItemEscalated");
        assertThat(signal).isNotNull().isInstanceOf(Map.class);
        @SuppressWarnings("unchecked")
        Map<String, Object> signalMap = (Map<String, Object>) signal;
        assertThat(signalMap)
                .containsEntry("workItemId", workItemId.toString())
                .containsEntry("bindingName", "escalation-binding");
        assertThat(signalMap.get("lastCandidateGroups"))
                .asList()
                .containsExactlyInAnyOrder("committee-a", "committee-b");

        // Output mapping skipped — context unchanged except for signal
        assertThat(updated.getCaseContext().get("stage")).isEqualTo("review");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `scripts/mvn-test casehub-work-engine-adapter -Dtest=PlanItemCompletionApplierTest`
Expected: FAIL — `applyStatus()` hits default branch, returns false, signal not written

- [ ] **Step 3: Add `case ESCALATED` to `applyStatus()`**

In `PlanItemCompletionApplier.applyStatus()`, add the ESCALATED case to the switch:

```java
case COMPLETED -> item.markCompleted();
case REJECTED -> item.markRejected();
case FAULTED -> item.markFaulted();
case EXPIRED -> item.markFaulted();
case ESCALATED -> item.markFaulted();
case OBSOLETE -> item.markObsolete();
case CANCELLED -> item.markCancelled();
```

- [ ] **Step 4: Add resolution validation bypass for ESCALATED**

In `PlanItemCompletionApplier.apply()`, restructure the validation guard (lines 105-117) to bypass for ESCALATED:

```java
if (status != WorkItemStatus.ESCALATED
        && ref != null && ref.resolutionTypeName() != null && ref.resolution() != null) {
    try {
        var bridge = bridgeResolver.resolveByTypeNameStrict(ref.resolutionTypeName());
        bridge.deserialise(MAPPER.readTree(ref.resolution()));
    } catch (Exception e) {
        LOG.warnf(e,
                  "Resolution validation failed for PlanItem %s caseId=%s — "
                  + "resolution does not match resolutionType %s",
                  planItemId, caseId, ref.resolutionTypeName());
        writeValidationFailedSignal(instance, item, ref, e);
        return;
    }
}
```

- [ ] **Step 5: Add `writeEscalationSignal()` and call it from `apply()`**

Add the new private method to `PlanItemCompletionApplier`:

```java
private void writeEscalationSignal(CaseInstance instance, PlanItem item, WorkItemRef ref) {
    final List<String> lastGroups =
            ref.candidateGroups() != null
                    ? List.of(ref.candidateGroups().split("\\s*,\\s*"))
                    : List.of();
    instance
            .getCaseContext()
            .set(
                    "workItemEscalated",
                    Map.of(
                            "workItemId", ref.id().toString(),
                            "lastCandidateGroups", lastGroups,
                            "bindingName", item.getBindingName()));
}
```

In `apply()`, after `applyStatus()` returns true and before `applyOutputMapping()`:

```java
if (status == WorkItemStatus.ESCALATED) {
    writeEscalationSignal(instance, item, ref);
}
```

- [ ] **Step 6: Add ESCALATED to PlanItemStateChangedEvent firing**

In `apply()`, update the FAULTED event firing block:

```java
if (status == WorkItemStatus.FAULTED || status == WorkItemStatus.EXPIRED
        || status == WorkItemStatus.ESCALATED) {
    planItemStateChangedEvents.fireAsync(
            new PlanItemStateChangedEvent(caseId, planItemId, bindingName,
                                          previousStatus, TaskStatus.FAULTED, instance.tenancyId));
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `scripts/mvn-test casehub-work-engine-adapter -Dtest=PlanItemCompletionApplierTest`
Expected: PASS

- [ ] **Step 8: Write stale resolution bypass test**

Add to `PlanItemCompletionApplierTest`:

```java
@Test
void escalated_withStaleResolution_bypassesValidation_stillTransitions() {
    UUID workItemId = UUID.randomUUID();
    WorkItemRef ref = new WorkItemRef(
            workItemId, WorkItemStatus.ESCALATED,
            PlanItemCallerRef.encode(caseId, planItemId),
            null,
            "{\"partial\": \"data\"}",        // stale resolution
            "committee-a",
            null, "test-tenant", null, null,
            "io.casehub.SomeResolutionType");  // resolutionTypeName set

    applier.apply(caseId, planItemId, WorkItemStatus.ESCALATED, ref);

    assertThat(planItem.getStatus())
            .as("ESCALATED must bypass resolution validation")
            .isEqualTo(TaskStatus.FAULTED);

    CaseInstance updated = caseInstanceRepository.findByUuid(caseId, "test-tenant");
    Object signal = updated.getCaseContext().get("workItemEscalated");
    assertThat(signal).isNotNull();
}
```

- [ ] **Step 9: Run all applier tests**

Run: `scripts/mvn-test casehub-work-engine-adapter -Dtest=PlanItemCompletionApplierTest`
Expected: 2 tests PASS

- [ ] **Step 10: Commit**

```bash
git add engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java
git add engine-adapter/src/test/java/io/casehub/work/engine/PlanItemCompletionApplierTest.java
git commit -m "feat: handle ESCALATED in PlanItemCompletionApplier

Add case ESCALATED → markFaulted() to applyStatus().
Write workItemEscalated signal with lastCandidateGroups.
Bypass resolution validation for ESCALATED (not actor-submitted).
Fire PlanItemStateChangedEvent with TaskStatus.FAULTED.

Refs #341"
```

---

### Task 2: Add ESCALATED to ActionGateCompletionApplier

**Files:**
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/ActionGateCompletionApplier.java`
- Modify: `engine-adapter/src/test/java/io/casehub/work/engine/ActionGateHandlerTest.java`

**Interfaces:**
- Consumes: `ActionGateCompletionApplier.apply(GateCallerRef, WorkItemStatus, WorkItemRef, String)` — existing public API
- Produces: no new interfaces — ESCALATED routes to existing `handleExpired()`

- [ ] **Step 1: Write the failing test**

Add to `ActionGateHandlerTest`:

```java
@Test
void completionApplier_escalated_publishesExpiredEvent() {
    UUID caseId = UUID.randomUUID();
    GateCallerRef gateRef = new GateCallerRef(caseId, 42);
    WorkItemRef ref = new WorkItemRef(
            UUID.randomUUID(), WorkItemStatus.ESCALATED,
            GateCallerRef.encode(caseId, 42),
            null, null, "committee-a", null, TENANCY_ID, null, null, null);

    actionGateCompletionApplier.apply(gateRef, WorkItemStatus.ESCALATED, ref, TENANCY_ID);

    await().atMost(5, TimeUnit.SECONDS)
           .untilAsserted(() -> assertThat(GateEventRecorder.expiredEvents)
                   .anyMatch(e -> e.caseId().equals(caseId) && e.gateId() == 42));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `scripts/mvn-test casehub-work-engine-adapter -Dtest=ActionGateHandlerTest#completionApplier_escalated_publishesExpiredEvent`
Expected: FAIL — ESCALATED hits default branch, no event published

- [ ] **Step 3: Add ESCALATED to the failure case arm**

In `ActionGateCompletionApplier.apply()`, change line 66:

From:
```java
case EXPIRED, FAULTED -> handleExpired(gateRef, tenancyId);
```

To:
```java
case EXPIRED, FAULTED, ESCALATED -> handleExpired(gateRef, tenancyId);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `scripts/mvn-test casehub-work-engine-adapter -Dtest=ActionGateHandlerTest#completionApplier_escalated_publishesExpiredEvent`
Expected: PASS

- [ ] **Step 5: Run all gate tests to check for regressions**

Run: `scripts/mvn-test casehub-work-engine-adapter -Dtest=ActionGateHandlerTest`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add engine-adapter/src/main/java/io/casehub/work/engine/ActionGateCompletionApplier.java
git add engine-adapter/src/test/java/io/casehub/work/engine/ActionGateHandlerTest.java
git commit -m "feat: handle ESCALATED in ActionGateCompletionApplier

Add ESCALATED to the EXPIRED/FAULTED case arm so gate-backed
WorkItems that reach terminal ESCALATED fire ActionGateExpiredEvent
instead of being silently dropped.

Refs #341"
```

---

### Task 3: Remove ESCALATED intercept from WorkItemLifecycleAdapter

**Files:**
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java`
- Modify: `engine-adapter/src/test/java/io/casehub/work/engine/WorkItemLifecycleAdapterTest.java`

**Interfaces:**
- Consumes: `PlanItemCompletionApplier.apply()` with ESCALATED (from Task 1)
- Consumes: `ActionGateCompletionApplier.apply()` with ESCALATED (from Task 2)
- Produces: no new interfaces — removes the intercept so ESCALATED flows through existing paths

- [ ] **Step 1: Update the existing ESCALATED test**

In `WorkItemLifecycleAdapterTest`, replace the `workItemEscalated_writesContextSignal_planItemUnchanged` test (lines 193-227). The new test verifies ESCALATED routes through the applier:

```java
@Test
void workItemEscalated_routesToApplier_marksPlanItemFaulted() {
    PlanItem delegatedItem = PlanItem.create("escalation-ht",
            io.casehub.api.model.ExecutorRef.of("ht-worker"), 10);
    delegatedItem.tryMarkDispatching();
    delegatedItem.markDelegated();
    registry.getOrCreate(caseId, "test-tenant").addPlanItem(delegatedItem);
    String delegatedItemId = delegatedItem.getPlanItemId();

    WorkItem escalatedItem = new WorkItem();
    escalatedItem.id = UUID.randomUUID();
    escalatedItem.status = WorkItemStatus.ESCALATED;
    escalatedItem.candidateGroups = "committee-a,committee-b";
    escalatedItem.callerRef = PlanItemCallerRef.encode(caseId, delegatedItemId);

    lifecycleEvents.fireAsync(
            WorkItemLifecycleEvent.of("workitem.escalated", escalatedItem, "system", null));

    await()
            .atMost(5, TimeUnit.SECONDS)
            .untilAsserted(
                    () -> assertThat(delegatedItem.getStatus()).isEqualTo(TaskStatus.FAULTED));

    CaseInstance updated = caseInstanceRepository.findByUuid(caseId, "test-tenant");
    Object signal = updated.getCaseContext().get("workItemEscalated");
    assertThat(signal).isNotNull().isInstanceOf(Map.class);
    @SuppressWarnings("unchecked")
    Map<String, Object> signalMap = (Map<String, Object>) signal;
    assertThat(signalMap)
            .containsEntry("workItemId", escalatedItem.id.toString())
            .containsEntry("bindingName", "escalation-ht");
    assertThat(signalMap.get("lastCandidateGroups"))
            .asList()
            .containsExactlyInAnyOrder("committee-a", "committee-b");
}
```

- [ ] **Step 2: Run updated test to verify it fails (intercept still present)**

Run: `scripts/mvn-test casehub-work-engine-adapter -Dtest=WorkItemLifecycleAdapterTest#workItemEscalated_routesToApplier_marksPlanItemFaulted`
Expected: FAIL — handleEscalation() intercepts, PlanItem stays RUNNING/DELEGATED, signal uses old field name

- [ ] **Step 3: Remove the ESCALATED intercept and delete handleEscalation()**

In `WorkItemLifecycleAdapter.onWorkItemLifecycle()`, remove lines 77-79:

```java
// DELETE these three lines:
if (status == WorkItemStatus.ESCALATED) {
    handleEscalation(wie);
    return;
}
```

Delete the entire `handleEscalation()` method (lines 176-223).

- [ ] **Step 4: Update class-level Javadoc**

Replace the ESCALATED paragraph in the class Javadoc (lines 52-57):

From:
```
 * <p>ESCALATED is terminal — all SLA breach policy branches have been exhausted. The adapter writes
 * a {@code workItemEscalated} signal to the case context so the engine can react (e.g. notify
 * supervisor, create replacement task). The PlanItem stays DELEGATED. Note: SLA breach policies
 * that re-route the WorkItem to new groups (the {@code EscalateTo} decision) do not set ESCALATED —
 * the WorkItem stays PENDING with updated candidate groups, so the adapter's terminal filter skips
 * it entirely. Refs engine#338, engine#400.
```

To:
```
 * <p>ESCALATED is terminal — all SLA breach policy branches have been exhausted. The applier
 * transitions the PlanItem to FAULTED and writes a {@code workItemEscalated} signal to the case
 * context. SLA breach policies that re-route the WorkItem to new groups ({@code EscalateTo}) do
 * not set ESCALATED — the WorkItem stays PENDING with updated candidate groups. Refs #341.
```

- [ ] **Step 5: Run updated test to verify it passes**

Run: `scripts/mvn-test casehub-work-engine-adapter -Dtest=WorkItemLifecycleAdapterTest#workItemEscalated_routesToApplier_marksPlanItemFaulted`
Expected: PASS

- [ ] **Step 6: Run full engine-adapter test suite**

Run: `scripts/mvn-test casehub-work-engine-adapter`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git add engine-adapter/src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java
git add engine-adapter/src/test/java/io/casehub/work/engine/WorkItemLifecycleAdapterTest.java
git commit -m "fix: remove ESCALATED intercept from WorkItemLifecycleAdapter

ESCALATED now flows through the normal terminal path to
PlanItemCompletionApplier (PlanItem-backed) and
ActionGateCompletionApplier (gate-backed). Delete handleEscalation()
and its contradictory Javadoc. Update adapter test to verify
ESCALATED routes through to the applier.

Closes #341"
```
