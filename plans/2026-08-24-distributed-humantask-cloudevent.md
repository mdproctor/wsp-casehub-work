# Distributed HumanTask CloudEvent Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #972 — feat: distributed HumanTask — CloudEvent emitter and lifecycle consumer for cross-service WorkItem bridge
**Issue group:** #972

**Goal:** Enable the engine to dispatch HumanTask and ActionGate WorkItems via CloudEvents when the work service is remote (not co-located), and consume lifecycle CloudEvents to complete PlanItems.

**Architecture:** New optional engine module `work-cloudevent/` implements `HumanTaskScheduler` and `ActionGateScheduler` SPIs via CloudEvent emission. Shared `PlanItemCompletionApplier` and `GateCompletionApplier` services in engine-runtime handle inbound lifecycle events. The inbound CloudEvent consumer is a pure transport adapter that delegates to these shared services.

**Tech Stack:** CloudEvents SDK (`io.cloudevents:cloudevents-core`), CDI async events for transport-agnostic emission/consumption, Quarkus ARC for bean discovery.

## Global Constraints

- All `@ConsumeEvent` handlers use `@RunOnVirtualThread` + `void` (protocol PP-20260723-c4c1cf)
- SPIs follow the `@DefaultBean @ApplicationScoped` no-op pattern (protocol PP-20260514-engine-spi-noops-defaultbean)
- Plan-definition types in `engine-api`; execution types in `engine-common` (protocol PP-20260727-5267d2)
- No `casehub-work-api` dependency in `engine-common` or `engine-runtime` — work types stay in the CloudEvent module
- `TaskStatus` (engine-native) is the applier input type — callers map from external types
- Tests named `*Test.java` (surefire), never `*IT.java`

---

## Batch 1: Foundation — ActionGateScheduler SPI + CallerRefParser

### Task 1: ActionGateScheduler SPI, request type, handler refactoring

Create the `ActionGateScheduler` SPI (symmetric with `HumanTaskScheduler`), rename `ActionGateScheduleRequest` → `ActionGateScheduleRequest`, refactor the single production call site, and add the `@DefaultBean` no-op.

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/spi/ActionGateScheduler.java`
- Rename: `ActionGateScheduleRequest` → `ActionGateScheduleRequest` (use `ide_refactor_rename`)
- Move: `ActionGateScheduleRequest` from `common/internal/event/` → `common/spi/` (use `ide_move_file`)
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpActionGateScheduler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java` — `handleGate()` uses `Instance<ActionGateScheduler>` instead of event bus publish
- Test: `runtime/src/test/java/io/casehub/engine/internal/worker/NoOpActionGateSchedulerTest.java`
- Test: existing `ActionGateResolutionTest.java` and `ActionGateResolutionTypeTest.java` must still pass after refactoring

**Interfaces:**
- Produces: `ActionGateScheduler.schedule(ActionGateScheduleRequest)`, `ActionGateScheduleRequest(UUID caseId, String tenancyId, long gateId, PlannedAction plannedAction, RiskDecision.GateRequired gateRequired, Set<String> resolvedCandidateGroups, @Nullable String resolutionTypeName)`

- [ ] **Step 1: Write test for NoOpActionGateScheduler**

```java
// runtime/src/test/java/io/casehub/engine/internal/worker/NoOpActionGateSchedulerTest.java
package io.casehub.engine.internal.worker;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.engine.common.spi.ActionGateScheduler;
import org.junit.jupiter.api.Test;

class NoOpActionGateSchedulerTest {

  @Test
  void schedule_is_noop() {
    ActionGateScheduler scheduler = new NoOpActionGateScheduler();
    // Should not throw — silent no-op
    scheduler.schedule(null);
  }

  @Test
  void implements_interface() {
    assertThat(new NoOpActionGateScheduler()).isInstanceOf(ActionGateScheduler.class);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=NoOpActionGateSchedulerTest -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: compilation failure — `ActionGateScheduler` does not exist yet

- [ ] **Step 3: Create ActionGateScheduler interface**

```java
// common/src/main/java/io/casehub/engine/common/spi/ActionGateScheduler.java
package io.casehub.engine.common.spi;

public interface ActionGateScheduler {
  void schedule(ActionGateScheduleRequest request);
}
```

- [ ] **Step 4: Rename ActionGateScheduleEvent → ActionGateScheduleRequest**

Use `ide_refactor_rename` on `ActionGateScheduleRequest` → `ActionGateScheduleRequest`.
Then use `ide_move_file` from `common/src/main/java/io/casehub/engine/common/internal/event/ActionGateScheduleRequest.java` to `common/src/main/java/io/casehub/engine/common/spi/ActionGateScheduleRequest.java`.

Verify all references updated (one production site in `WorkflowExecutionCompletedHandler`, two test sites in `ActionGateResolutionTest` and `ActionGateResolutionTypeTest`).

- [ ] **Step 5: Create NoOpActionGateScheduler**

```java
// runtime/src/main/java/io/casehub/engine/internal/worker/NoOpActionGateScheduler.java
package io.casehub.engine.internal.worker;

import io.casehub.engine.common.spi.ActionGateScheduleRequest;
import io.casehub.engine.common.spi.ActionGateScheduler;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpActionGateScheduler implements ActionGateScheduler {

  @Override
  public void schedule(ActionGateScheduleRequest request) {
    // intentional no-op — no work integration configured
  }
}
```

- [ ] **Step 6: Run NoOpActionGateScheduler test**

Run: `mvn install -DskipTests -q -pl common && mvn test -pl runtime -Dtest=NoOpActionGateSchedulerTest -q`
Expected: PASS

- [ ] **Step 7: Refactor WorkflowExecutionCompletedHandler.handleGate()**

Replace the event bus publish in `handleGate()` (line 727-736) with `Instance<ActionGateScheduler>` injection and direct SPI call. Add `Instance<ActionGateScheduler> actionGateScheduler` field with `@Inject`. Guard with `isResolvable()` — if false, log warning and return (same pattern as `publishHumanTaskSchedule`).

The `handleGate()` method should change from:
```java
eventBus.publish(
    EventBusAddresses.ACTION_GATE_SCHEDULE,
    new ActionGateScheduleEvent(...));
```
To:
```java
if (!actionGateScheduler.isResolvable()) {
  LOG.warnf("No ActionGateScheduler on classpath — skipping gate for caseId=%s", caseInstance.getUuid());
  return;
}
actionGateScheduler.get().schedule(
    new ActionGateScheduleRequest(...));
```

Note: the event bus import for `ACTION_GATE_SCHEDULE` can be removed if no other method uses it. Check with `ide_find_references` on `ACTION_GATE_SCHEDULE` first — it may still be used by `ActionGateCancelledHandler` for the cancellation path.

- [ ] **Step 8: Run existing ActionGate tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="ActionGateResolution*" -q`
Expected: PASS — the test `RecordingActionGateScheduler` (or equivalent test bean) needs to exist. If tests fail because no `ActionGateScheduler` bean is found in test context, create a `RecordingActionGateScheduler` test bean (same pattern as `RecordingHumanTaskScheduler`).

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on the `runtime` and `common` modules to confirm no compilation errors.

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#972): extract ActionGateScheduler SPI, rename ActionGateScheduleEvent → ActionGateScheduleRequest

Refs #972"
```

### Task 2: CallerRefParser utility

Engine-side callerRef encoding and parsing, symmetric with the work-engine-adapter's `CallerRef`/`PlanItemCallerRef`/`GateCallerRef`. Pure utility — no CDI, no dependencies beyond Java stdlib.

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/spi/CallerRefParser.java`
- Test: `common/src/test/java/io/casehub/engine/common/spi/CallerRefParserTest.java`

**Interfaces:**
- Produces: `CallerRefParser.encodePlanItem(UUID caseId, String planItemId) → String`, `CallerRefParser.encodeGate(UUID caseId, long gateId) → String`, `CallerRefParser.parse(String raw) → CallerRef` (sealed: `PlanItemRef(UUID caseId, String planItemId)` | `GateRef(UUID caseId, long gateId)`)

- [ ] **Step 1: Write tests for CallerRefParser**

```java
// common/src/test/java/io/casehub/engine/common/spi/CallerRefParserTest.java
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;
import org.junit.jupiter.api.Test;

class CallerRefParserTest {

  private static final UUID CASE_ID = UUID.fromString("550e8400-e29b-41d4-a716-446655440000");

  @Test
  void encodePlanItem_produces_correct_format() {
    String ref = CallerRefParser.encodePlanItem(CASE_ID, "pi-001");
    assertThat(ref).isEqualTo("case:550e8400-e29b-41d4-a716-446655440000/pi:pi-001");
  }

  @Test
  void encodeGate_produces_correct_format() {
    String ref = CallerRefParser.encodeGate(CASE_ID, 42L);
    assertThat(ref).isEqualTo("case:550e8400-e29b-41d4-a716-446655440000/gate:42");
  }

  @Test
  void parse_planItem_ref() {
    var ref = CallerRefParser.parse("case:550e8400-e29b-41d4-a716-446655440000/pi:pi-001");
    assertThat(ref).isInstanceOf(CallerRefParser.PlanItemRef.class);
    var pi = (CallerRefParser.PlanItemRef) ref;
    assertThat(pi.caseId()).isEqualTo(CASE_ID);
    assertThat(pi.planItemId()).isEqualTo("pi-001");
  }

  @Test
  void parse_gate_ref() {
    var ref = CallerRefParser.parse("case:550e8400-e29b-41d4-a716-446655440000/gate:42");
    assertThat(ref).isInstanceOf(CallerRefParser.GateRef.class);
    var gate = (CallerRefParser.GateRef) ref;
    assertThat(gate.caseId()).isEqualTo(CASE_ID);
    assertThat(gate.gateId()).isEqualTo(42L);
  }

  @Test
  void parse_null_returns_null() {
    assertThat(CallerRefParser.parse(null)).isNull();
  }

  @Test
  void parse_invalid_format_returns_null() {
    assertThat(CallerRefParser.parse("not-a-caller-ref")).isNull();
    assertThat(CallerRefParser.parse("case:invalid-uuid/pi:001")).isNull();
    assertThat(CallerRefParser.parse("")).isNull();
  }

  @Test
  void roundtrip_planItem() {
    String encoded = CallerRefParser.encodePlanItem(CASE_ID, "pi-001");
    var parsed = (CallerRefParser.PlanItemRef) CallerRefParser.parse(encoded);
    assertThat(parsed.caseId()).isEqualTo(CASE_ID);
    assertThat(parsed.planItemId()).isEqualTo("pi-001");
  }

  @Test
  void roundtrip_gate() {
    String encoded = CallerRefParser.encodeGate(CASE_ID, 99L);
    var parsed = (CallerRefParser.GateRef) CallerRefParser.parse(encoded);
    assertThat(parsed.caseId()).isEqualTo(CASE_ID);
    assertThat(parsed.gateId()).isEqualTo(99L);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl common -Dtest=CallerRefParserTest -q`
Expected: compilation failure — `CallerRefParser` does not exist

- [ ] **Step 3: Implement CallerRefParser**

```java
// common/src/main/java/io/casehub/engine/common/spi/CallerRefParser.java
package io.casehub.engine.common.spi;

import java.util.UUID;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class CallerRefParser {

  private static final Pattern PI_PATTERN =
      Pattern.compile("^case:([0-9a-fA-F-]{36})/pi:(.+)$");
  private static final Pattern GATE_PATTERN =
      Pattern.compile("^case:([0-9a-fA-F-]{36})/gate:(\\d+)$");

  private CallerRefParser() {}

  public static String encodePlanItem(UUID caseId, String planItemId) {
    return "case:" + caseId + "/pi:" + planItemId;
  }

  public static String encodeGate(UUID caseId, long gateId) {
    return "case:" + caseId + "/gate:" + gateId;
  }

  public static CallerRef parse(String raw) {
    if (raw == null || raw.isEmpty()) {
      return null;
    }
    Matcher pi = PI_PATTERN.matcher(raw);
    if (pi.matches()) {
      try {
        return new PlanItemRef(UUID.fromString(pi.group(1)), pi.group(2));
      } catch (IllegalArgumentException e) {
        return null;
      }
    }
    Matcher gate = GATE_PATTERN.matcher(raw);
    if (gate.matches()) {
      try {
        return new GateRef(UUID.fromString(gate.group(1)), Long.parseLong(gate.group(2)));
      } catch (IllegalArgumentException e) {
        return null;
      }
    }
    return null;
  }

  public sealed interface CallerRef permits PlanItemRef, GateRef {
    UUID caseId();
  }

  public record PlanItemRef(UUID caseId, String planItemId) implements CallerRef {}

  public record GateRef(UUID caseId, long gateId) implements CallerRef {}
}
```

- [ ] **Step 4: Run tests**

Run: `mvn test -pl common -Dtest=CallerRefParserTest -q`
Expected: PASS

- [ ] **Step 5: Verify with ide_diagnostics**

Run `ide_diagnostics` on the `common` module.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#972): add CallerRefParser utility for callerRef encoding/parsing

Refs #972"
```

---

## Batch 2: Shared Completion Services

### Task 3: PlanItemCompletionApplier

Shared service encapsulating the full PlanItem completion flow: lookup, resolution validation, status transition, JQ output mapping, conflict resolution, CDI event publishing, and CONTEXT_CHANGED. Uses `TaskStatus` (engine-native) — callers map from their external types.

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/completion/PlanItemCompletionApplier.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/completion/PlanItemCompletionApplierTest.java`

**Interfaces:**
- Consumes: `BlackboardRegistry.get(UUID)`, `CrossTenantCaseInstanceRepository.findByUuid(UUID)`, `BridgeResolver.resolveByTypeNameStrict(String)`, `JQEvaluator.eval(String, JsonNode)`, `CaseDefinitionRegistry.getCaseDefinition(CaseMetaModel)`, `EventBus.publish(String, Object)`, `PlanItem.markCompleted()` / `markRejected()` / `markFaulted()` / `markObsolete()` / `markCancelled()` / `markSuspended()` / `markResumed()`
- Produces: `PlanItemCompletionApplier.apply(UUID caseId, String planItemId, TaskStatus status, @Nullable String resolution, @Nullable String resolutionTypeName)`, `PlanItemCompletionApplier.applySuspend(UUID caseId, String planItemId)`, `PlanItemCompletionApplier.applyResume(UUID caseId, String planItemId)`

- [ ] **Step 1: Write tests for PlanItemCompletionApplier**

Create a unit test that verifies the key behaviors using mocks. Focus on:
1. COMPLETED status → `markCompleted()` called, CONTEXT_CHANGED published
2. REJECTED → `markRejected()`, `PlanItemStateChangedEvent` fired
3. FAULTED → `markFaulted()`, `PlanItemStateChangedEvent` fired
4. PlanItem not found → warning logged, no state change
5. CaseInstance not found → warning logged, no CONTEXT_CHANGED
6. PlanItem already terminal → idempotent skip (DEBUG logged)
7. Resolution validation failure → `workItemValidationFailed` signal written
8. Output mapping via JQ → context updated via ConflictResolver
9. SUSPENDED → `markSuspended()`
10. RESUMED from SUSPENDED → `markResumed()`

The test structure mirrors the decompiled `PlanItemCompletionApplier` from the work-engine-adapter but uses `TaskStatus` input. Use Mockito for `BlackboardRegistry`, `CrossTenantCaseInstanceRepository`, `BridgeResolver`, `JQEvaluator`, `EventBus`, `CaseDefinitionRegistry`, and CDI `Event` beans.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=PlanItemCompletionApplierTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement PlanItemCompletionApplier**

Create `@ApplicationScoped` class in `runtime/src/main/java/io/casehub/engine/internal/completion/PlanItemCompletionApplier.java`. Port the logic from the decompiled `io.casehub.work.engine.PlanItemCompletionApplier` but:
- Accept `TaskStatus` instead of `WorkItemStatus`
- Accept `@Nullable String resolution` and `@Nullable String resolutionTypeName` instead of `WorkItemRef`
- Extract resolution validation and output mapping into focused private methods
- Fire `PlanItemStateChangedEvent` and `PlanItemObsoleteEvent` via CDI async events
- Publish `CONTEXT_CHANGED` via event bus

Key mapping from TaskStatus to PlanItem transitions:
```
COMPLETED → markCompleted()
REJECTED → markRejected() + PlanItemStateChangedEvent(REJECTED)
FAULTED → markFaulted() + PlanItemStateChangedEvent(FAULTED)
OBSOLETE → markObsolete() + PlanItemObsoleteEvent
CANCELLED → markCancelled()
```

Output mapping: only for COMPLETED when the PlanItem's target is `HumanTaskTarget` with a non-null `outputMapping()`. Use `JQEvaluator.eval()` against the resolution JSON, apply via `ConflictResolver.resolve()` per key.

- [ ] **Step 4: Run tests**

Run: `mvn install -DskipTests -q && mvn test -pl runtime -Dtest=PlanItemCompletionApplierTest -q`
Expected: PASS

- [ ] **Step 5: Verify with ide_diagnostics**

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#972): add PlanItemCompletionApplier shared completion service

Refs #972"
```

### Task 4: GateCompletionApplier

Shared service that routes ActionGate lifecycle events to the existing engine handlers via event bus. Maps `TaskStatus` → gate event bus addresses.

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/completion/GateCompletionApplier.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/completion/GateCompletionApplierTest.java`

**Interfaces:**
- Consumes: `EventBus.publish(String, Object)`, `EventBusAddresses.ACTION_GATE_APPROVED` / `_REJECTED` / `_EXPIRED`
- Produces: `GateCompletionApplier.apply(UUID caseId, String tenancyId, long gateId, TaskStatus status, @Nullable String resolution, @Nullable String actorId)`

- [ ] **Step 1: Write tests**

```java
// Test cases:
// 1. COMPLETED → publishes ActionGateApprovedEvent on ACTION_GATE_APPROVED
// 2. REJECTED → publishes ActionGateRejectedEvent on ACTION_GATE_REJECTED
// 3. CANCELLED → publishes ActionGateRejectedEvent on ACTION_GATE_REJECTED (same path)
// 4. FAULTED (EXPIRED mapping) → publishes ActionGateExpiredEvent on ACTION_GATE_EXPIRED
// 5. Unsupported status → warning logged, no event published
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement GateCompletionApplier**

```java
@ApplicationScoped
public class GateCompletionApplier {

  @Inject EventBus eventBus;

  public void apply(UUID caseId, String tenancyId, long gateId,
      TaskStatus status, @Nullable String resolution, @Nullable String actorId) {
    switch (status) {
      case COMPLETED -> eventBus.publish(
          EventBusAddresses.ACTION_GATE_APPROVED,
          new ActionGateApprovedEvent(caseId, tenancyId, gateId, resolution, actorId, null));
      case REJECTED, CANCELLED -> eventBus.publish(
          EventBusAddresses.ACTION_GATE_REJECTED,
          new ActionGateRejectedEvent(caseId, tenancyId, gateId, resolution, actorId));
      case FAULTED -> eventBus.publish(
          EventBusAddresses.ACTION_GATE_EXPIRED,
          new ActionGateExpiredEvent(caseId, tenancyId, gateId));
      default -> LOG.warnf("Unsupported gate status %s for caseId=%s gateId=%d", status, caseId, gateId);
    }
  }
}
```

- [ ] **Step 4: Run tests**

Run: `mvn test -pl runtime -Dtest=GateCompletionApplierTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#972): add GateCompletionApplier shared gate lifecycle service

Refs #972"
```

---

## Batch 3: CloudEvent Module — Outbound Emitters

### Task 5: Module scaffold + CloudEventHumanTaskScheduler

Create the `work-cloudevent/` module with POM, directory structure, and the HumanTask CloudEvent emitter.

**Files:**
- Create: `work-cloudevent/pom.xml`
- Create: `work-cloudevent/src/main/java/io/casehub/engine/work/cloudevent/CloudEventHumanTaskScheduler.java`
- Create: `work-cloudevent/src/main/resources/application.properties` (empty or minimal)
- Test: `work-cloudevent/src/test/java/io/casehub/engine/work/cloudevent/CloudEventHumanTaskSchedulerTest.java`
- Modify: root `pom.xml` — add `<module>work-cloudevent</module>`

**Interfaces:**
- Consumes: `HumanTaskScheduler.schedule(HumanTaskScheduleRequest)`, `BlackboardRegistry.get(UUID)`, `CasePlanModel.getPlanItemByBindingName(String)`, `PlanItemStore.save(PlanItemSaveRequest, String)`, `CallerRefParser.encodePlanItem(UUID, String)`, `Event<CloudEvent>.fireAsync(CloudEvent)`
- Produces: `CloudEventHumanTaskScheduler` (CDI bean implementing `HumanTaskScheduler`), emits `io.casehub.work.workitem.create` CloudEvents

- [ ] **Step 1: Create module directory and POM**

Create `work-cloudevent/pom.xml` modeled on `a2a/pom.xml`. Key dependencies:
- `casehub-engine-common` (compile — SPIs, CallerRefParser)
- `casehub-engine` runtime (compile — PlanItemCompletionApplier, GateCompletionApplier)
- `casehub-engine-planning` (compile — CasePlanModel, PlanItem)
- `casehub-work-api` (compile — WorkCloudEventTypes, WorkItemStatus)
- `io.cloudevents:cloudevents-core` (compile)
- `quarkus-arc` (compile)
- `quarkus-vertx` (compile — for CDI Event<CloudEvent>)
- Test: `casehub-engine-persistence-memory`, `casehub-engine-scheduler-quartz`, `casehub-ledger-testing`, `casehub-engine-ledger`, `quarkus-jdbc-h2`, `quarkus-junit`, junit, assertj, mockito

Add `<module>work-cloudevent</module>` to root `pom.xml` (after `react`).
Add CloudEvents version property to root `pom.xml`: `<version.io.cloudevents>4.0.1</version.io.cloudevents>` and managed dependency for `io.cloudevents:cloudevents-core`.

Create `work-cloudevent/src/main/resources/META-INF/beans.xml` (empty marker for CDI).

- [ ] **Step 2: Write test for CloudEventHumanTaskScheduler**

Unit test with mocks. Verify:
1. `schedule()` with inline mode → CloudEvent fired with type `CREATE`, correct data fields (title, candidateGroups, payload, callerRef, scope, expiresAt), PlanItem marked DELEGATED
2. `schedule()` with template mode → CloudEvent data includes `templateId`
3. PlanItem not found → `revertDispatching()`, no CloudEvent fired
4. PlanItem not DISPATCHING → `revertDispatching()`, no CloudEvent fired
5. Field mapping completeness — candidateScores, routingExperiences, permittedOutcomes, payloadTypeName, resolutionTypeName all mapped

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn install -DskipTests -q && mvn test -pl work-cloudevent -Dtest=CloudEventHumanTaskSchedulerTest -q`

- [ ] **Step 4: Implement CloudEventHumanTaskScheduler**

`@ApplicationScoped` class implementing `HumanTaskScheduler`. On `schedule()`:
1. Look up PlanItem via `BlackboardRegistry.get(caseId)` → `plan.getPlanItemByBindingName(bindingName)`
2. Validate `item.getStatus() == TaskStatus.DISPATCHING`, revert if not
3. Build callerRef via `CallerRefParser.encodePlanItem(caseId, item.getPlanItemId())`
4. Build CloudEvent data JSON with all mapped fields (see spec §Part 1 field mapping table)
5. Build CloudEvent via `CloudEventBuilder`: type=`WorkCloudEventTypes.CREATE`, source=`"/engine/cases/" + caseId`, tenancyid extension
6. Fire `cloudEventEmitter.fireAsync(cloudEvent)`
7. Persist PlanItem as DELEGATED via `planItemStore.save(PlanItemSaveRequest.primitive(..., TaskStatus.DELEGATED, ...))`
8. `item.markDelegated()`

Handle template vs inline by checking `target.isTemplateMode()`.

- [ ] **Step 5: Run tests**

Expected: PASS

- [ ] **Step 6: Verify with ide_diagnostics**

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#972): add work-cloudevent module with CloudEventHumanTaskScheduler

Refs #972"
```

### Task 6: CloudEventActionGateScheduler

ActionGate CloudEvent emitter implementing the `ActionGateScheduler` SPI.

**Files:**
- Create: `work-cloudevent/src/main/java/io/casehub/engine/work/cloudevent/CloudEventActionGateScheduler.java`
- Test: `work-cloudevent/src/test/java/io/casehub/engine/work/cloudevent/CloudEventActionGateSchedulerTest.java`

**Interfaces:**
- Consumes: `ActionGateScheduler.schedule(ActionGateScheduleRequest)`, `CallerRefParser.encodeGate(UUID, long)`, `Event<CloudEvent>.fireAsync(CloudEvent)`
- Produces: `CloudEventActionGateScheduler` (CDI bean implementing `ActionGateScheduler`)

- [ ] **Step 1: Write tests**

Verify:
1. Single-approver gate → CloudEvent with type CREATE, gate callerRef, payload with description/actionType/reversible/context, candidateGroups, expiresAt
2. Quorum gate → warning logged, CloudEvent skipped (work-side multi-instance not yet supported)
3. Null candidateGroups → omitted from data

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement CloudEventActionGateScheduler**

`@ApplicationScoped` class implementing `ActionGateScheduler`. Simpler than the HumanTask emitter — no PlanItem management (PendingActionGate is already set by `WorkflowExecutionCompletedHandler.handleGate()` before calling the scheduler).

- [ ] **Step 4: Run tests**

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#972): add CloudEventActionGateScheduler for distributed gate dispatch

Refs #972"
```

---

## Batch 4: CloudEvent Module — Inbound Consumer + Integration

### Task 7: WorkItemLifecycleCloudEventConsumer

Pure transport adapter — parses incoming lifecycle CloudEvents, maps types to `TaskStatus`, routes via `CallerRefParser`, delegates to `PlanItemCompletionApplier` or `GateCompletionApplier`.

**Files:**
- Create: `work-cloudevent/src/main/java/io/casehub/engine/work/cloudevent/WorkItemLifecycleCloudEventConsumer.java`
- Test: `work-cloudevent/src/test/java/io/casehub/engine/work/cloudevent/WorkItemLifecycleCloudEventConsumerTest.java`

**Interfaces:**
- Consumes: `PlanItemCompletionApplier.apply(UUID, String, TaskStatus, String, String)`, `PlanItemCompletionApplier.applySuspend(UUID, String)`, `PlanItemCompletionApplier.applyResume(UUID, String)`, `GateCompletionApplier.apply(UUID, String, long, TaskStatus, String, String)`, `CallerRefParser.parse(String)`, `WorkCloudEventTypes` constants

- [ ] **Step 1: Write tests**

Cover:
1. COMPLETED CloudEvent with PlanItem callerRef → delegates to `PlanItemCompletionApplier.apply(caseId, planItemId, COMPLETED, resolution, resolutionTypeName)`
2. REJECTED CloudEvent → delegates with REJECTED status
3. FAULTED/EXPIRED/ESCALATED → delegates with FAULTED
4. SUSPENDED → delegates to `applySuspend()`
5. RESUMED → delegates to `applyResume()`
6. Gate callerRef COMPLETED → delegates to `GateCompletionApplier.apply()` with COMPLETED
7. Gate REJECTED → REJECTED
8. Gate EXPIRED → FAULTED
9. Non-lifecycle CloudEvent type (CREATED, ASSIGNED) → ignored
10. Missing tenancyid extension → ERROR logged, no delegation
11. Invalid callerRef → WARN logged, no delegation
12. Null data → ERROR logged, no delegation

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement WorkItemLifecycleCloudEventConsumer**

```java
@ApplicationScoped
public class WorkItemLifecycleCloudEventConsumer {

  @Inject PlanItemCompletionApplier planItemApplier;
  @Inject GateCompletionApplier gateApplier;

  public void onLifecycleCloudEvent(@ObservesAsync CloudEvent ce) {
    TaskStatus status = mapCloudEventType(ce.getType());
    if (status == null) return; // not a lifecycle type we handle

    String tenancyId = (String) ce.getExtension("tenancyid");
    if (tenancyId == null) { LOG.errorf(...); return; }

    JsonNode data = parseData(ce);
    if (data == null) { LOG.errorf(...); return; }

    String callerRef = data.has("callerRef") ? data.get("callerRef").asText() : null;
    var ref = CallerRefParser.parse(callerRef);
    if (ref == null) { LOG.warnf(...); return; }

    switch (ref) {
      case CallerRefParser.PlanItemRef pi -> handlePlanItem(pi, status, data);
      case CallerRefParser.GateRef gate -> handleGate(gate, tenancyId, status, data);
    }
  }
}
```

The `mapCloudEventType()` method maps CloudEvent type strings to `TaskStatus`:
- `WorkCloudEventTypes.COMPLETED` → `TaskStatus.COMPLETED`
- `REJECTED` → `REJECTED`
- `FAULTED` → `FAULTED`
- `EXPIRED` → `FAULTED`
- `ESCALATED` → `FAULTED`
- `OBSOLETE` → `OBSOLETE`
- `CANCELLED` → `CANCELLED`
- `SUSPENDED` → `SUSPENDED`
- `RESUMED` → null (special-cased, calls `applyResume()`)
- All others → null (ignored)

- [ ] **Step 4: Run tests**

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#972): add WorkItemLifecycleCloudEventConsumer for distributed lifecycle events

Refs #972"
```

### Task 8: Startup conflict detector + round-trip integration test

Fail-fast startup check when both co-located and distributed modules are on the classpath. Round-trip integration test validating the full outbound → inbound cycle using CDI events.

**Files:**
- Create: `work-cloudevent/src/main/java/io/casehub/engine/work/cloudevent/WorkIntegrationConflictDetector.java`
- Create: `work-cloudevent/src/test/java/io/casehub/engine/work/cloudevent/DistributedHumanTaskRoundTripTest.java`
- Create: `work-cloudevent/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `Instance<HumanTaskScheduler>` (to detect multiple implementations)

- [ ] **Step 1: Write WorkIntegrationConflictDetector**

```java
@ApplicationScoped
public class WorkIntegrationConflictDetector {

  @Inject Instance<HumanTaskScheduler> humanTaskSchedulers;

  void onStartup(@Observes @Priority(1) StartupEvent event) {
    long count = humanTaskSchedulers.stream().count();
    if (count > 1) {
      throw new IllegalStateException(
          "Multiple HumanTaskScheduler implementations detected (" + count
          + "). casehub-work-engine-adapter and casehub-engine-work-cloudevent "
          + "are mutually exclusive — remove one from the classpath.");
    }
  }
}
```

- [ ] **Step 2: Write round-trip integration test**

`@QuarkusTest` that:
1. Defines a CaseHub subclass with a HumanTask binding
2. Starts a case, triggers the binding
3. Captures the emitted CloudEvent (via a recording CDI observer)
4. Asserts: CloudEvent type is `CREATE`, callerRef present, data fields mapped correctly, PlanItem is DELEGATED
5. Simulates work-side response: fires a `COMPLETED` CloudEvent with the same callerRef and resolution data
6. Asserts: PlanItem is COMPLETED, output mapping applied, CONTEXT_CHANGED fired

Test `application.properties` must include all the standard memory-mode settings plus the work-cloudevent module indexed.

- [ ] **Step 3: Run full module test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl work-cloudevent -q`
Expected: all tests PASS

- [ ] **Step 4: Run full project build**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common,runtime,work-cloudevent -q`
Expected: PASS — no regressions in common or runtime from the SPI changes

- [ ] **Step 5: Update CLAUDE.md**

Add `## casehub-engine-work-cloudevent Module` section documenting the new module.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#972): add startup conflict detector and round-trip integration test

Refs #972"
```

---

## References

- `specs/issue-972-distributed-humantask-cloudevent/2026-08-24-distributed-humantask-cloudevent-design.md` — design spec
- `common/src/main/java/io/casehub/engine/common/spi/HumanTaskScheduler.java:25` — existing SPI pattern
- `common/src/main/java/io/casehub/engine/common/spi/HumanTaskScheduleRequest.java:26` — request record pattern
- `common/src/main/java/io/casehub/engine/common/internal/event/ActionGateScheduleEvent.java:24` — type to rename/move
- `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java:681-751` — handleGate refactoring site
- `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java:685-815` — publishHumanTaskSchedule (emitter field mapping reference)
- `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerProvisioner.java:31-50` — @DefaultBean pattern
- `runtime/src/test/java/io/casehub/engine/RecordingHumanTaskScheduler.java:24-33` — test recording pattern
- `io.casehub.work.engine.HumanTaskScheduleHandler` (work-engine-adapter JAR) — co-located handler, field mapping reference
- `io.casehub.work.engine.PlanItemCompletionApplier` (work-engine-adapter JAR) — completion logic reference
- `io.casehub.work.api.WorkCloudEventTypes` (work-api JAR) — CloudEvent type constants
- `a2a/pom.xml` — module POM template
- PP-20260723-c4c1cf — virtual-thread handler convention
- GE-20260521-a0f5a6 — HumanTask PlanItem pre-marking gotcha
- GE-20260718-9eb2c0 — WorkItem creation failure silently swallowed
- casehubio/engine#974 — repo build order resolution
