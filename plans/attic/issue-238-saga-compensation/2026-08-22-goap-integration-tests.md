# Plan: GOAP/Adaptive Planning Strategy Integration Tests

**Issue:** casehubio/engine#924
**Module:** `planning/src/test/java/io/casehub/engine/planning/it/`
**Pattern:** Follow `SequentialStrategyIntegrationTest` — `@QuarkusTest`, inner `CaseHub` subclass, `startCase()`, `Awaitility.await()`, assert on `CaseInstanceCache`.

## Batch 1 — Core GOAP Dispatch (parallel)

### Task 1: Basic GOAP dispatch through real pipeline
**File:** `GoapStrategyIntegrationTest.java`
**Test:** `goapStrategy_firesBindingsInPlannedOrder`
- Define 3 workers (A, B, C) with GOAP preconditions/effects creating a dependency chain:
  - A: no preconditions → produces `analysisResult`
  - B: requires `analysisResult` → produces `clauseList`  
  - C: requires `analysisResult`, `clauseList` → produces `riskAssessment`
- Set `planningStrategy("goap")` on the definition
- Configure `goapActions` matching the workers with matching preconditions/effects
- Set `goalToEffectKeys` mapping a goal to `riskAssessment`
- Use `ContextChangeTrigger("true")` on all bindings (GOAP catch-all pattern)
- Record execution order in `CopyOnWriteArrayList`
- Start case, await COMPLETED
- Assert execution order is A → B → C (GOAP-planned, not declaration order)
- Assert final context contains `analysisResult`, `clauseList`, `riskAssessment`

**TDD:**
1. Write test with CaseHub subclass and assertions
2. Run — expect pass (production code already exists)
3. If fails, diagnose wiring issues

### Task 2: Empty plan — case waits for context
**File:** `GoapEmptyPlanIntegrationTest.java`
**Test:** `goapStrategy_emptyPlan_caseContinuesWhenContextChanges`
- Define 2 workers where both require a precondition not initially present
- Start case with empty context — GOAP finds no valid path
- Assert case is RUNNING (not faulted, not completed)
- Signal context with the missing precondition key
- Await case progression — GOAP now finds a valid path
- Assert workers execute and case completes

**TDD:**
1. Write test
2. Run — expect pass
3. May need to verify GOAP re-evaluates on CONTEXT_CHANGED

## Batch 2 — Adaptive and Guards (parallel)

### Task 3: Adaptive per-step replanning
**File:** `AdaptiveStrategyIntegrationTest.java`
**Test:** `adaptiveStrategy_replansAfterContextChange`
- Define 3 workers (A, B, C):
  - A: no preconditions → produces `step1Result`
  - B: requires `step1Result` → produces `step2Result` (cost 0.3)
  - C: requires `step1Result` → produces `step2Result` (cost 0.8, higher benefit)
- Use `planningStrategy("adaptive")`
- Initial plan would choose B (cheaper path to `step2Result`)
- After A completes, worker A's output also sets a key that changes the cost calculus
- Verify Adaptive replans and picks the optimal next step from current world state

**Test:** `adaptiveStrategy_excludesExecutedActions`
- Define 3 workers in a chain
- Verify that after step 1 completes, Adaptive does NOT re-dispatch step 1
- Assert execution has no duplicates

**TDD:**
1. Write both tests
2. Run — verify replan logic works end-to-end

### Task 4: Guard interaction with GOAP filtering
**File:** `GoapGuardInteractionIntegrationTest.java`
**Test:** `goapStrategy_respectsWhenGuards`
- Define 3 workers, all with GOAP actions
- Worker B has a `ContextChangeTrigger` guard that is initially false
- Start case — GOAP plans with A and C only (B ineligible)
- After A completes, signal context to make B's guard true
- Verify B is now included in the replan and executes

**TDD:**
1. Write test
2. Run — GOAP only receives eligible bindings from `PlanningStrategyLoopControl`, so guard filtering should work automatically

## Batch 3 — Failure and End-to-End

### Task 5: Failure reroute + adaptive replan
**File:** `GoapFailureReplanIntegrationTest.java`
**Test:** `adaptiveStrategy_replansAfterWorkerFailure`
- Define 3 workers: A (no preconditions), B and C both produce `result` (alternative paths)
- B has lower cost, C has higher cost
- GOAP initially plans A → B
- B returns `WorkerResult.failed("simulated failure")`
- Configure `OutcomePolicy.REROUTE` on B's binding
- After B fails and is excluded, Adaptive replans and finds A → C path
- Assert C executes and case completes
- Assert `_diagnostics.bindingB.status` is `FAILED`

**TDD:**
1. Write test
2. Run — failure cascade + replan is the complex interaction to verify

### Task 6: End-to-end annotated GOAP case
**File:** `GoapAnnotatedEndToEndTest.java` (in `examples/goap-case-annotated/src/test/`)
**Test:** `annotatedGoapCase_executesThroughFullPipeline`
- Use `GoapAnnotatedCase` classes from the existing example
- This is a `@QuarkusTest` (not `QuarkusUnitTest`) — requires full engine stack
- Start case with trigger input
- Provide mock worker functions (the annotated case has `WorkerFunction.NONE`)
- Await completion
- Assert workers executed in GOAP-planned dependency order
- Assert context contains all produced effects

**Decision:** If `goap-case-annotated` example workers use `WorkerFunction.NONE` (external workers), this test may need a companion `CaseHub` subclass that overrides with real functions. Evaluate at implementation time — may merge into Task 1's test class with annotation-style definition instead.

## Verification

After all tests pass:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl planning
```
Verify no regressions in existing planning tests. Then:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl planning,runtime
```
Cross-module verification — GOAP strategies are CDI beans discovered by both modules.
