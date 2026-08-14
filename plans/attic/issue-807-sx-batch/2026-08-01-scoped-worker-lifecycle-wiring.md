# Scoped Worker Lifecycle Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #823 — ScopedWorkerRegistry production registration in dispatch/execution path
**Issue group:** #823, #824

**Goal:** Wire scoped worker infrastructure into production code paths — session registration, re-invocation routing, accumulated state threading, output application, and PERSISTENT thread lifecycle.

**Architecture:** Foundation types revised first (ScopedWorkerSession records, ExecutionMetadata, event types). Then dispatch gate opened for scoped re-dispatch. Then registration + routing wired in CaseContextChangedEventHandler. Then accumulated state threaded through the execution chain. Then output application via dedicated handler. Then PERSISTENT execution via new WorkerFunctionHandler + PersistentScope implementation. Finally schedule trigger integration.

**Tech Stack:** Quarkus CDI, Vert.x event bus, virtual threads, Quartz scheduler, JQ expressions

## Global Constraints

- All `@ConsumeEvent` handlers use `@RunOnVirtualThread` + `void` (protocol PP-20260723-c4c1cf)
- Plan-definition types in `engine-api`; execution types in `engine-common` (protocol PP-20260727-5267d2)
- No new types in `engine-api` or `casehubio/worker`
- Event bus publishes use Mutiny `EventBus` (not core Vert.x) for automatic codec handling
- `scheduler-quartz` depends on `engine-api` + `engine-common` only — no `runtime` dependency. CDI beans from `runtime` can be injected at deployment time but imports must not reference `runtime` packages.

---

### Task 1: Foundation Types

Revise `ScopedWorkerSession` records, expand `ExecutionMetadata`, add `ScopedWorkerOutputEvent` and `EventBusAddresses` constant, add execution lock infrastructure to `ScopedWorkerRegistry`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/scope/ScopedWorkerSession.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/executor/ExecutionMetadata.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/EventBusAddresses.java`
- Create: `common/src/main/java/io/casehub/engine/common/internal/event/ScopedWorkerOutputEvent.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/scope/ScopedWorkerRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/worker/scope/ScopedWorkerRegistryTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/worker/scope/ScopedWorkerRegistryTest.java`

**Interfaces:**
- Produces: `ScopedWorkerSession.Reinvoked(bindingName, caseId, executorName, scope, participation, accumulatedState, lastInputDataHash)`, `ScopedWorkerSession.Persistent(bindingName, caseId, executorName, scope, participation, mailbox)`, `ExecutionMetadata(workerName, inputDataHash, bindingName, executionMode)`, `ScopedWorkerOutputEvent(caseInstance, bindingName, output, executionMode)`, `EventBusAddresses.SCOPED_WORKER_OUTPUT`, `ScopedWorkerRegistry.executionLock(ScopeKey)`

- [ ] **Step 1: Write failing tests for ScopedWorkerSession revision**

Update `ScopedWorkerRegistryTest` to construct sessions with the new record shape — `executorName` replaces `planItemId`, `Reinvoked` gains `lastInputDataHash`, `Persistent` loses `workerThread`:

```java
@Test
void register_reinvoked_session_with_executorName() {
    var session = new ScopedWorkerSession.Reinvoked(
        "binding-a", caseId, "worker-1",
        LifecycleScope.COMPOUND, Participation.PARTICIPANT,
        new AtomicReference<>(Map.of()),
        new AtomicReference<>(null));
    registry.register(new ScopedWorkerRegistry.ScopeKey(caseId, "binding-a"), session);
    assertThat(registry.get(caseId, "binding-a")).isPresent();
    assertThat(registry.get(caseId, "binding-a").get().executorName()).isEqualTo("worker-1");
}
```

```java
@Test
void executionLock_returns_same_lock_for_same_key() {
    var key = new ScopedWorkerRegistry.ScopeKey(caseId, "b1");
    ReentrantLock lock1 = registry.executionLock(key);
    ReentrantLock lock2 = registry.executionLock(key);
    assertThat(lock1).isSameAs(lock2);
}

@Test
void terminateByCase_cleans_up_execution_locks() {
    var key = new ScopedWorkerRegistry.ScopeKey(caseId, "b1");
    registry.executionLock(key);
    registry.register(key, createReinvokedSession(caseId, "b1"));
    registry.terminateByCase(caseId);
    // New lock after cleanup is a different instance
    ReentrantLock newLock = registry.executionLock(key);
    assertThat(newLock).isNotSameAs(registry.executionLock(new ScopedWorkerRegistry.ScopeKey(UUID.randomUUID(), "b1")));
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=ScopedWorkerRegistryTest -f pom.xml`
Expected: compilation errors — `planItemId` parameter doesn't match, `executorName()` method not found, `executionLock()` method not found.

- [ ] **Step 3: Revise ScopedWorkerSession records**

Replace `planItemId` with `executorName` on sealed interface and both records. Remove `workerThread` from `Persistent`. Add `lastInputDataHash` to `Reinvoked`:

```java
public sealed interface ScopedWorkerSession
    permits ScopedWorkerSession.Persistent, ScopedWorkerSession.Reinvoked {

    String bindingName();
    UUID caseId();
    String executorName();
    LifecycleScope scope();
    Participation participation();

    record Persistent(
        String bindingName, UUID caseId, String executorName,
        LifecycleScope scope, Participation participation,
        BlockingQueue<ContextEvent> mailbox
    ) implements ScopedWorkerSession {}

    record Reinvoked(
        String bindingName, UUID caseId, String executorName,
        LifecycleScope scope, Participation participation,
        AtomicReference<Map<String, Object>> accumulatedState,
        AtomicReference<String> lastInputDataHash
    ) implements ScopedWorkerSession {}
}
```

- [ ] **Step 4: Add executionLock to ScopedWorkerRegistry**

Add `executionLocks` map, `executionLock()` method, and cleanup in `terminateByCase` and `terminateByScope`:

```java
private final ConcurrentHashMap<ScopeKey, ReentrantLock> executionLocks = new ConcurrentHashMap<>();

public ReentrantLock executionLock(ScopeKey key) {
    return executionLocks.computeIfAbsent(key, k -> new ReentrantLock());
}
```

In `terminateByCase`: add `executionLocks.remove(new ScopeKey(caseId, bindingName))` after session removal.
In `terminateByScope`: add `executionLocks.remove(new ScopeKey(caseId, bindingName))` after each session removal.

- [ ] **Step 5: Expand ExecutionMetadata**

Add `bindingName` and `executionMode` fields with backward-compatible constructor:

```java
public record ExecutionMetadata(
    String workerName, String inputDataHash,
    String bindingName, ExecutionMode executionMode
) {
    public ExecutionMetadata(String workerName, String inputDataHash) {
        this(workerName, inputDataHash, null, null);
    }
}
```

- [ ] **Step 6: Add ScopedWorkerOutputEvent and EventBusAddresses constant**

Create `ScopedWorkerOutputEvent` in `common/internal/event/`:

```java
public record ScopedWorkerOutputEvent(
    CaseInstance caseInstance, String bindingName,
    Map<String, Object> output, ExecutionMode executionMode
) {}
```

Add to `EventBusAddresses`:

```java
public static final String SCOPED_WORKER_OUTPUT = "casehub.engine.scoped-worker-output";
```

- [ ] **Step 7: Fix existing tests that construct ScopedWorkerSession**

Update `ScopedWorkerRegistryTest` and `LifecycleScopeIntegrationTest` — replace `planItemId` with `executorName`, add `lastInputDataHash` parameter for Reinvoked, remove `workerThread` from Persistent.

- [ ] **Step 8: Run all tests — verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=ScopedWorkerRegistryTest`
Expected: all PASS.

- [ ] **Step 9: Commit**

```
feat(#823): revise ScopedWorkerSession records, expand ExecutionMetadata, add ScopedWorkerOutputEvent

Replace planItemId with executorName on session records (zero existing
references to planItemId). Remove workerThread from Persistent (termination
is mailbox-only). Add lastInputDataHash to Reinvoked for cycle detection.
Add per-binding execution locks to ScopedWorkerRegistry.

Refs #823, #824
```

---

### Task 2: Dispatch Gate + Protocol Fix

Open the dispatch gate for scoped bindings and fix the termination handler protocol violation.

**Files:**
- Modify: `planning/src/main/java/io/casehub/engine/planning/control/PlanningStrategyLoopControl.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/ScopedWorkerTerminationHandler.java`
- Test: `planning/src/test/java/io/casehub/engine/planning/control/PlanningStrategyLoopControlTest.java` (or relevant integration test)

**Interfaces:**
- Consumes: `Binding.lifecycleScope()`, `TaskStatus.RUNNING`, `LifecycleScope.BINDING`
- Produces: scoped bindings with RUNNING PlanItems now appear in the dispatched list

- [ ] **Step 1: Write failing test for scope-aware dispatch gate**

In the PlanningStrategyLoopControl test, create a binding with `lifecycleScope=COMPOUND` and a PlanItem already in RUNNING state. Assert the binding IS included in the dispatched list:

```java
@Test
void select_allows_redispatch_for_scoped_binding_with_running_planitem() {
    Binding scopedBinding = Binding.builder()
        .name("scoped-worker")
        .target(new CapabilityTarget(capability))
        .trigger(contextChangeTrigger)
        .lifecycleScope(LifecycleScope.COMPOUND)
        .executionMode(ExecutionMode.REINVOKED)
        .build();
    // Create PlanItem already RUNNING
    PlanItem pi = PlanItem.create("scoped-worker", ExecutorRef.of("worker-1"), capability.name());
    pi.tryMarkRunning(); // PENDING → RUNNING
    // ... wire into CasePlanModel ...
    
    List<Binding> dispatched = loopControl.select(ctx);
    assertThat(dispatched).contains(scopedBinding);
}
```

- [ ] **Step 2: Run test — verify it fails**

Expected: FAIL — the scoped binding is excluded because `tryMarkRunning()` returns false for RUNNING PlanItem.

- [ ] **Step 3: Implement scope-aware gate**

In `PlanningStrategyLoopControl.filterAndIndexForDispatch()`, before the `tryMarkRunning()` call:

```java
if (binding.target() instanceof CapabilityTarget) {
    if (binding.lifecycleScope() != LifecycleScope.BINDING
            && pi.getStatus() == TaskStatus.RUNNING) {
        dispatched.add(binding);
    } else if (pi.tryMarkRunning()) {
        registry.indexForCompletion(caseId, pi.executorName(), pi.getPlanItemId());
        dispatched.add(binding);
    }
}
```

- [ ] **Step 4: Fix ScopedWorkerTerminationHandler protocol violation**

Replace `@ConsumeEvent(value = EventBusAddresses.COMPOUND_COMPLETED, blocking = true)` with:

```java
@ConsumeEvent(EventBusAddresses.COMPOUND_COMPLETED)
@RunOnVirtualThread
public void onCompoundCompleted(CompoundCompletedEvent event) { ... }
```

Add import: `import io.quarkus.virtual.threads.VirtualThreads;` — wait, `@RunOnVirtualThread` is `io.smallrye.common.annotation.RunOnVirtualThread`.

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl planning && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=ScopedWorkerTerminationHandler*`
Expected: all PASS.

- [ ] **Step 6: Commit**

```
feat(#823): scope-aware dispatch gate + protocol compliance fix

Allow scoped bindings with RUNNING PlanItems to re-dispatch through
PlanningStrategyLoopControl — the registry check in publishWorkerSchedule
handles the actual routing decision.

Fix ScopedWorkerTerminationHandler: blocking=true → @RunOnVirtualThread
per protocol PP-20260723-c4c1cf.

Refs #823
```

---

### Task 3: Session Registration and Re-Dispatch Routing

Wire `registerScopedSession()` and `routeToExistingSession()` in `CaseContextChangedEventHandler`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Test: new test class or extend existing handler tests

**Interfaces:**
- Consumes: `ScopedWorkerRegistry.register()`, `ScopedWorkerRegistry.get()`, `ScopedWorkerSession.Reinvoked`, `ScopedWorkerSession.Persistent`, `ContextEvent`
- Produces: sessions registered in the registry on first activation; subsequent dispatches routed to existing sessions

- [ ] **Step 1: Write failing test — REINVOKED first dispatch registers session**

```java
@Test
void publishWorkerSchedule_registers_reinvoked_session_on_first_dispatch() {
    Binding binding = Binding.builder()
        .name("reinvoked-binding")
        .target(new CapabilityTarget(capability))
        .trigger(contextChangeTrigger)
        .lifecycleScope(LifecycleScope.COMPOUND)
        .executionMode(ExecutionMode.REINVOKED)
        .build();
    // ... trigger dispatch ...
    
    assertThat(scopedWorkerRegistry.get(caseId, "reinvoked-binding")).isPresent();
    assertThat(scopedWorkerRegistry.get(caseId, "reinvoked-binding").get())
        .isInstanceOf(ScopedWorkerSession.Reinvoked.class);
}
```

- [ ] **Step 2: Write failing test — REINVOKED second dispatch routes to session (no duplicate scheduling)**

```java
@Test
void publishWorkerSchedule_routes_reinvoked_to_existing_session() {
    // Register a Reinvoked session
    scopedWorkerRegistry.register(
        new ScopeKey(caseId, "reinvoked-binding"),
        new ScopedWorkerSession.Reinvoked("reinvoked-binding", caseId, "worker-1",
            LifecycleScope.COMPOUND, Participation.PARTICIPANT,
            new AtomicReference<>(Map.of("prior", "state")),
            new AtomicReference<>(null)));
    // ... trigger dispatch with changed context ...
    
    // Verify scheduleWorker was called with executorName from session, not from routing
    // Verify no new session was registered (still the original one)
}
```

- [ ] **Step 3: Write failing test — PERSISTENT first dispatch registers session with mailbox**

```java
@Test
void publishWorkerSchedule_registers_persistent_session_with_mailbox() {
    Binding binding = Binding.builder()
        .name("persistent-binding")
        .target(new CapabilityTarget(capability))
        .trigger(contextChangeTrigger)
        .lifecycleScope(LifecycleScope.CASE)
        .executionMode(ExecutionMode.PERSISTENT)
        .participation(Participation.COMPANION)
        .build();
    // ... trigger dispatch ...
    
    var session = scopedWorkerRegistry.get(caseId, "persistent-binding");
    assertThat(session).isPresent();
    assertThat(session.get()).isInstanceOf(ScopedWorkerSession.Persistent.class);
    assertThat(((ScopedWorkerSession.Persistent) session.get()).mailbox()).isNotNull();
}
```

- [ ] **Step 4: Write failing test — REINVOKED cycle detection suppresses unchanged input**

```java
@Test
void routeToExistingSession_skips_reinvoked_when_input_hash_unchanged() {
    var session = new ScopedWorkerSession.Reinvoked("binding", caseId, "worker-1",
        LifecycleScope.COMPOUND, Participation.PARTICIPANT,
        new AtomicReference<>(Map.of()),
        new AtomicReference<>("existing-hash"));
    scopedWorkerRegistry.register(new ScopeKey(caseId, "binding"), session);
    // Set up context such that computeInputHash returns "existing-hash"
    // ... trigger dispatch ...
    
    // Verify scheduleWorker was NOT called (re-invocation suppressed)
}
```

- [ ] **Step 5: Run tests — verify they fail**

Expected: compilation/assertion failures — methods don't exist yet.

- [ ] **Step 6: Implement registerScopedSession()**

Add `registerScopedSession(CaseInstance, Binding, String executorName)` to `CaseContextChangedEventHandler`:

```java
private void registerScopedSession(CaseInstance caseInstance, Binding binding,
        String executorName) {
    var key = new ScopedWorkerRegistry.ScopeKey(caseInstance.getUuid(), binding.getName());
    switch (binding.executionMode()) {
        case REINVOKED -> scopedWorkerRegistry.register(key,
            new ScopedWorkerSession.Reinvoked(
                binding.getName(), caseInstance.getUuid(), executorName,
                binding.lifecycleScope(), binding.participation(),
                new AtomicReference<>(Map.of()),
                new AtomicReference<>(null)));
        case PERSISTENT -> {
            var mailbox = new LinkedBlockingQueue<ContextEvent>();
            scopedWorkerRegistry.register(key,
                new ScopedWorkerSession.Persistent(
                    binding.getName(), caseInstance.getUuid(), executorName,
                    binding.lifecycleScope(), binding.participation(),
                    mailbox));
        }
        default -> {}
    }
}
```

- [ ] **Step 7: Implement routeToExistingSession() with cycle detection**

Replace existing `routeToExistingSession()` with the pattern-matching switch from the spec:

```java
private void routeToExistingSession(ScopedWorkerSession session,
        CaseInstance caseInstance, Binding binding, List<Worker> workers,
        Capability capability, UUID signalId, List<RetrievedExperience> experiences) {
    switch (session) {
        case ScopedWorkerSession.Persistent p -> {
            JsonNode snapshot = caseInstance.getCaseContext()
                .layer(ContextLayer.WORKING).asJsonNode();
            p.mailbox().offer(new ContextEvent(snapshot, Map.of()));
        }
        case ScopedWorkerSession.Reinvoked r -> {
            String projection = binding.getInputProjectionOverride();
            String currentHash = computeInputHash(caseInstance, projection);
            if (currentHash != null && currentHash.equals(r.lastInputDataHash().get())) {
                LOG.debugf("Skipping re-invocation for '%s' — projected input unchanged",
                    binding.getName());
                return;
            }
            r.lastInputDataHash().set(currentHash);
            scheduleWorker(caseInstance, workers, binding, capability,
                r.executorName(), signalId, experiences);
        }
    }
}
```

Add `computeInputHash()` helper using JQ evaluator + SHA-256.

- [ ] **Step 8: Wire registration into publishWorkerSchedule() after routing**

In the `RoutingResult.Selected` case, after `scheduleWorker`:

```java
case RoutingResult.Selected s -> {
    final var a = s.single();
    if (binding.lifecycleScope() != LifecycleScope.BINDING) {
        registerScopedSession(caseInstance, binding, a.executorId());
    }
    scheduleWorker(caseInstance, workers, binding, capability,
        a.executorId(), signalId, experiences);
}
```

Update `routeToExistingSession()` call to pass binding, workers, capability, signalId, experiences.

- [ ] **Step 9: Run tests — verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime`
Expected: all PASS.

- [ ] **Step 10: Commit**

```
feat(#823): wire session registration and re-dispatch routing

Register REINVOKED and PERSISTENT sessions in publishWorkerSchedule after
agent routing succeeds. Route subsequent dispatches to existing sessions:
PERSISTENT → mailbox, REINVOKED → re-schedule with session's executorName
(skipping agent routing). Input-hash cycle detection breaks feedback loops.

Refs #823, #824
```

---

### Task 4: Accumulated State Threading

Thread accumulated state through the execution chain: ExecutionMetadata → handler → WorkerRuntime.

**Files:**
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/SyncAgentWorkerFunctionHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java`
- Test: unit tests for handler and runtime

**Interfaces:**
- Consumes: `ExecutionMetadata.bindingName()`, `ExecutionMetadata.executionMode()`, `ScopedWorkerRegistry.get()`, `ScopedWorkerRegistry.executionLock()`
- Produces: `WorkerScope.accumulatedState()` returns prior invocation output for REINVOKED workers

- [ ] **Step 1: Write failing test — DefaultWorkerRuntime returns accumulated state**

```java
@Test
void accumulatedState_returns_provided_state() {
    Map<String, Object> state = Map.of("count", 3, "lastSeen", "item-7");
    var runtime = new DefaultWorkerRuntime(caseId, "task-1", context, state, ...);
    assertThat(runtime.accumulatedState()).isEqualTo(state);
}

@Test
void accumulatedState_returns_empty_when_not_provided() {
    var runtime = workerRuntimeFactory.create(caseId, "task-1", context);
    assertThat(runtime.accumulatedState()).isEmpty();
}
```

- [ ] **Step 2: Implement DefaultWorkerRuntime accumulated state**

Add `accumulatedState` field to `DefaultWorkerRuntime`, update constructor, override `accumulatedState()`. Add 4-arg `WorkerRuntimeFactory.create()` overload delegating with `Map.of()` from 3-arg.

- [ ] **Step 3: Write failing test — SyncAgentWorkerFunctionHandler reads/writes accumulated state for REINVOKED**

```java
@Test
void execute_reinvoked_reads_accumulated_state_from_registry() {
    // Register a Reinvoked session with prior output
    var session = new ScopedWorkerSession.Reinvoked("binding-a", caseId, "worker-1",
        LifecycleScope.COMPOUND, Participation.PARTICIPANT,
        new AtomicReference<>(Map.of("prior", "value")),
        new AtomicReference<>(null));
    scopedWorkerRegistry.register(new ScopeKey(caseId, "binding-a"), session);

    // Execute with REINVOKED metadata
    var metadata = new ExecutionMetadata("worker-1", "hash", "binding-a", ExecutionMode.REINVOKED);
    WorkerFunction.Sync<Map, Map> fn = new WorkerFunction.Sync<>(Map.class, Map.class,
        (input, scope) -> {
            assertThat(scope.accumulatedState()).containsEntry("prior", "value");
            return WorkerResult.of(Map.of("new", "output"));
        });
    handler.execute(fn, Map.of("input", "data"), context, 5000, metadata);

    // Verify accumulated state was updated
    var updated = ((ScopedWorkerSession.Reinvoked) scopedWorkerRegistry
        .get(caseId, "binding-a").get()).accumulatedState().get();
    assertThat(updated).containsEntry("new", "output");
}
```

- [ ] **Step 4: Implement accumulated state threading in SyncAgentWorkerFunctionHandler**

Inject `ScopedWorkerRegistry`. Before creating `WorkerRuntime`:
- If `metadata.executionMode() == REINVOKED`: acquire execution lock, read accumulated state
- Pass accumulated state to `workerRuntimeFactory.create(caseId, workerName, context, accState)`

After execution:
- If REINVOKED: write output as new accumulated state, release lock (in finally block)

- [ ] **Step 5: Populate ExecutionMetadata in QuartzWorkerExecutionJob**

Change `new ExecutionMetadata(workerId, inputDataHash)` to `new ExecutionMetadata(workerId, inputDataHash, bindingName, executionMode)`. Both values already read from EventLog metadata.

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl scheduler-quartz`
Expected: all PASS.

- [ ] **Step 7: Commit**

```
feat(#823): thread accumulated state through execution chain

REINVOKED workers access prior invocation output via
WorkerScope.accumulatedState(). Per-binding ReentrantLock serializes
the read-execute-write sequence. ExecutionMetadata carries bindingName
and executionMode from EventLog metadata.

Refs #823, #826
```

---

### Task 5: Scoped Worker Output Application

Replace the dead `scoped-worker-output` publish with proper event handling. Add REINVOKED idempotency bypass.

**Files:**
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/ScopedWorkerOutputHandler.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/WorkerScheduleEventHandler.java` (idempotency bypass)
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/ScopedWorkerOutputHandlerTest.java`

**Interfaces:**
- Consumes: `EventBusAddresses.SCOPED_WORKER_OUTPUT`, `ScopedWorkerOutputEvent`, `ConflictResolver.resolve(strategy, key, existing, incoming)`, `CaseDefinitionRegistry`, `ScopedWorkerRegistry`
- Produces: scoped output applied to case context, `CONTEXT_CHANGED` published

- [ ] **Step 1: Write failing test — ScopedWorkerOutputHandler applies output to context**

```java
@Test
void onScopedWorkerOutput_applies_output_with_conflict_resolution() {
    var event = new ScopedWorkerOutputEvent(caseInstance, "binding-a",
        Map.of("result", "processed"), ExecutionMode.REINVOKED);
    
    handler.onScopedWorkerOutput(event);
    
    assertThat(caseInstance.getCaseContext().get("result")).isEqualTo("processed");
}

@Test
void onScopedWorkerOutput_discards_output_when_session_terminated() {
    // No session in registry — scope ended while job was in-flight
    var event = new ScopedWorkerOutputEvent(caseInstance, "terminated-binding",
        Map.of("result", "late"), ExecutionMode.REINVOKED);
    
    handler.onScopedWorkerOutput(event);
    
    assertThat(caseInstance.getCaseContext().get("result")).isNull();
}
```

- [ ] **Step 2: Implement ScopedWorkerOutputHandler**

Create `ScopedWorkerOutputHandler.java` per spec §4 — `@ApplicationScoped`, `@ConsumeEvent(SCOPED_WORKER_OUTPUT)`, `@RunOnVirtualThread`. Guard on session existence. Apply output per-key via `ConflictResolver.resolve()`. Publish `CONTEXT_CHANGED`.

- [ ] **Step 3: Update QuartzWorkerExecutionJob.onSuccess()**

Replace:
```java
vertx.eventBus().publish("casehub.engine.scoped-worker-output", ...)
```
With:
```java
eventBus.publish(EventBusAddresses.SCOPED_WORKER_OUTPUT,
    new ScopedWorkerOutputEvent(instance, bindingName, output, executionMode));
```

- [ ] **Step 4: Implement REINVOKED idempotency bypass in WorkerScheduleEventHandler**

In `scheduleUnderLock()`, bypass `decideAction()` when `executionMode == REINVOKED`:

```java
ScheduleAction action;
if (executionMode == ExecutionMode.REINVOKED) {
    action = ScheduleAction.createNew();
} else {
    action = decideAction(existing, inputDataHash);
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=ScopedWorkerOutputHandlerTest && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl scheduler-quartz`
Expected: all PASS.

- [ ] **Step 6: Commit**

```
feat(#823): scoped worker output application handler

Replace dead scoped-worker-output publish with ScopedWorkerOutputEvent
consumed by ScopedWorkerOutputHandler. Applies output using
ConflictResolver per-key strategy. Guards on session existence —
discards output when scope ended during in-flight execution.
REINVOKED bypasses idempotency dedup in WorkerScheduleEventHandler.

Refs #823, #825
```

---

### Task 6: Persistent Execution — DefaultPersistentScope + PersistentWorkerFunctionHandler

Implement the PERSISTENT execution model: scope backed by mailbox + handler that spawns virtual thread.

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/scope/DefaultPersistentScope.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/executor/PersistentWorkerFunctionHandler.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/worker/scope/DefaultPersistentScopeTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/executor/PersistentWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `PersistentScope<T>`, `WorkerFunction.Persistent<T>`, `ScopeTerminatedException`, `ScopedWorkerRegistry`, `BridgeResolver`, `JQEvaluator`, `WorkerRuntimeFactory`, `EventBus`
- Produces: `PersistentWorkerFunctionHandler.supports(WorkerFunction.Persistent)`, `DefaultPersistentScope.nextEvent()`, `DefaultPersistentScope.emit()`

- [ ] **Step 1: Write failing test — DefaultPersistentScope.nextEvent() delivers typed events**

```java
@Test
void nextEvent_deserializes_context_snapshot_to_typed_input() throws Exception {
    var mailbox = new LinkedBlockingQueue<ContextEvent>();
    var scope = new DefaultPersistentScope<>(Map.class, mailbox, caseId, "task-1",
        context, eventBus, ".", null, bridgeResolver, jqEvaluator,
        expressionEngineRegistry, workerRuntimeFactory);
    
    JsonNode snapshot = MAPPER.valueToTree(Map.of("key", "value"));
    mailbox.offer(new ContextEvent(snapshot, Map.of()));
    
    Map result = scope.nextEvent();
    assertThat(result).containsEntry("key", "value");
}

@Test
void nextEvent_throws_ScopeTerminatedException_on_shutdown() {
    var mailbox = new LinkedBlockingQueue<ContextEvent>();
    var scope = new DefaultPersistentScope<>(...);
    
    mailbox.offer(ContextEvent.SHUTDOWN);
    
    assertThatThrownBy(scope::nextEvent)
        .isInstanceOf(ScopeTerminatedException.class);
}
```

- [ ] **Step 2: Write failing test — DefaultPersistentScope.emit() publishes ScopedWorkerOutputEvent**

```java
@Test
void emit_publishes_scoped_worker_output_event() {
    // ... setup scope with mock eventBus ...
    scope.emit(Map.of("output", "data"));
    
    // Verify eventBus.publish was called with SCOPED_WORKER_OUTPUT address
}
```

- [ ] **Step 3: Implement DefaultPersistentScope**

Create `DefaultPersistentScope<T>` implementing `PersistentScope<T>`:
- `nextEvent()`: `mailbox.take()`, check `isShutdown()`, apply input projection via JQ, deserialize via bridge
- `emit()`: apply output schema if present, publish `ScopedWorkerOutputEvent` via event bus
- `caseId()`, `taskId()`: return constructor args
- `execute()`: delegate to inner WorkerRuntime
- `accumulatedState()`: return `Map.of()`

- [ ] **Step 4: Write failing test — PersistentWorkerFunctionHandler starts thread and returns**

```java
@Test
void execute_persistent_starts_virtual_thread_and_returns_empty_success() {
    // Pre-register persistent session with mailbox
    var mailbox = new LinkedBlockingQueue<ContextEvent>();
    scopedWorkerRegistry.register(
        new ScopeKey(caseId, "persistent-binding"),
        new ScopedWorkerSession.Persistent("persistent-binding", caseId, "worker-1",
            LifecycleScope.CASE, Participation.COMPANION, mailbox));
    
    var latch = new CountDownLatch(1);
    var persistent = new WorkerFunction.Persistent<>(Map.class, scope -> {
        latch.countDown();
        try { scope.nextEvent(); } catch (ScopeTerminatedException e) { /* shutdown */ }
    });
    
    var metadata = new ExecutionMetadata("worker-1", "hash", "persistent-binding",
        ExecutionMode.PERSISTENT);
    var result = handler.execute(persistent, Map.of(), context, 5000, metadata);
    
    assertThat(result.output()).isEqualTo(Map.of());
    assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
}
```

- [ ] **Step 5: Implement PersistentWorkerFunctionHandler**

Create handler per spec §5 — retrieve pre-registered session, build `DefaultPersistentScope`, submit virtual thread, return `WorkerResult.of(Map.of())`.

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest="DefaultPersistentScopeTest,PersistentWorkerFunctionHandlerTest"`
Expected: all PASS.

- [ ] **Step 7: Commit**

```
feat(#824): PERSISTENT execution model — PersistentScope + handler

DefaultPersistentScope backs PersistentScope<T> with a BlockingQueue
mailbox. nextEvent() applies input projection and bridge deserialization.
emit() publishes ScopedWorkerOutputEvent with output schema projection.

PersistentWorkerFunctionHandler retrieves the pre-registered session,
builds the scope, spawns a virtual thread, and returns empty Success.
Thread faults publish WorkflowExecutionCompleted for PlanItem completion.

Refs #824
```

---

### Task 7: Schedule Trigger Integration

Wire scoped worker session checks into `ScheduledTriggerJob` and `ConditionalScheduledTriggerJob`.

**Files:**
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/ScheduledTriggerJob.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/ConditionalScheduledTriggerJob.java` (if it exists and publishes WorkerScheduleEvent)
- Test: extend existing schedule trigger tests

**Interfaces:**
- Consumes: `ScopedWorkerRegistry.get()`, `ScopedWorkerSession.Persistent`, `ScopedWorkerSession.Reinvoked`, `ContextEvent`
- Produces: schedule-triggered context changes routed to existing scoped sessions

- [ ] **Step 1: Write failing test — cron tick routes to existing REINVOKED session**

```java
@Test
void scheduledTrigger_routes_to_existing_reinvoked_session() {
    // Register existing session
    scopedWorkerRegistry.register(
        new ScopeKey(caseId, "cron-binding"),
        new ScopedWorkerSession.Reinvoked("cron-binding", caseId, "worker-1",
            LifecycleScope.COMPOUND, Participation.PARTICIPANT,
            new AtomicReference<>(Map.of()),
            new AtomicReference<>(null)));
    // ... fire scheduled trigger ...
    
    // Verify WorkerScheduleEvent published with session's executorName
    // Verify no new session registered
}
```

- [ ] **Step 2: Implement registry check in ScheduledTriggerJob**

Inject `ScopedWorkerRegistry`. Before publishing `WorkerScheduleEvent`, check registry for existing sessions. Use session's `executorName` for REINVOKED. Pass lifecycle fields on the event for first activation.

- [ ] **Step 3: Apply same changes to ConditionalScheduledTriggerJob**

Mirror the registry check pattern.

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl scheduler-quartz`
Expected: all PASS.

- [ ] **Step 5: Commit**

```
feat(#823): schedule trigger scoped session routing

ScheduledTriggerJob and ConditionalScheduledTriggerJob check
ScopedWorkerRegistry before dispatch. Subsequent cron/delay fires
route to existing sessions. REINVOKED uses session's executorName
to maintain agent routing first-activation-only invariant.

Refs #823
```

---

## Verification

After all tasks complete:

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl planning
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl scheduler-quartz
```

All three modules must be green. Then run the full build:

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test
```
