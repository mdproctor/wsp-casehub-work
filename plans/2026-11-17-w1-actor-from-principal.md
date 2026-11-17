# W1: Actor/Claimant from CurrentPrincipal — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #424 — W1: Derive actor/claimant from CurrentPrincipal, not request params
**Issue group:** #424 (child of #423)

**Goal:** Remove untrusted actor/claimant parameters from user-facing SPIs and derive identity from `CurrentPrincipal.actorId()`.

**Architecture:** Three user-facing SPI interfaces (`WorkItemLifecycleApi`, `WorkItemQueueStateApi`, `WorkItemBulkApi`) currently accept actor/claimant as method params or request body fields. Remove these params and wire `CurrentPrincipal` injection in the three implementation classes. The internal `WorkItemOperations` interface keeps its actorId params — system callers (timers, engine-adapter) pass explicit IDs.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JAX-RS (generated from `@McpDomain`), JUnit 5 + RestAssured

## Global Constraints

- `WorkItemOperations` interface signatures MUST NOT change — engine-adapter, federation, and timers depend on them
- `*FromSystem` method variants MUST NOT be affected
- Use `MutableCurrentPrincipal` test doubles (already exist in rest, queues, runtime, mongodb test scopes) to set identity in tests
- Use `ide_insert_member` / `ide_replace_member` for structural editing; `ide_refactor_rename` for renames
- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn` and always use `-pl <module>`
- Run tests via `scripts/` helpers when available for timeout safety

---

## Batch 1: TDD — principal-based identity verification test

### Task 1: Write failing test proving actor comes from principal

**Files:**
- Create: `rest/src/test/java/io/casehub/work/rest/WorkItemPrincipalIdentityTest.java`

**Interfaces:**
- Consumes: `MutableCurrentPrincipal` (rest/src/test), REST endpoints `/api/work/lifecycle/*`
- Produces: test class that verifies actor/claimant derived from principal, not query params

- [ ] **Step 1: Write the failing test**

This test proves the new behavior: when no `?claimant=` query param is sent, the claim uses `CurrentPrincipal.actorId()`. It will fail on the current code because the generated endpoint requires the query param (or passes null).

```java
package io.casehub.work.rest;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

import io.casehub.work.rest.test.MutableCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class WorkItemPrincipalIdentityTest extends WorkItemTestBase {

    @Inject
    MutableCurrentPrincipal principal;

    // R1: claim derives claimant from CurrentPrincipal, not query param
    @Test
    void claim_R1_claimant_from_principal() {
        String id = createWorkItem();
        principal.setActorId("alice");

        given()
                .post("/api/work/lifecycle/claim/" + id)
                .then()
                .statusCode(200)
                .body("assigneeId", equalTo("alice"));
    }

    // R2: complete derives actor from CurrentPrincipal, not query param
    @Test
    void complete_R2_actor_from_principal() {
        String id = createWorkItem();
        principal.setActorId("alice");
        given().post("/api/work/lifecycle/claim/" + id).then().statusCode(200);
        given().post("/api/work/lifecycle/start/" + id).then().statusCode(200);

        given()
                .contentType("application/json")
                .body("{\"resolution\": \"done\"}")
                .post("/api/work/lifecycle/complete/" + id)
                .then()
                .statusCode(200)
                .body("status", equalTo("COMPLETED"));
    }

    // R3: query param actor/claimant is ignored (endpoint doesn't accept it)
    @Test
    void claim_R3_query_param_ignored() {
        String id = createWorkItem();
        principal.setActorId("alice");

        // Even if someone sends ?claimant=mallory, the claim should use principal (alice)
        given()
                .queryParam("claimant", "mallory")
                .post("/api/work/lifecycle/claim/" + id)
                .then()
                .statusCode(200)
                .body("assigneeId", equalTo("alice"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemPrincipalIdentityTest -pl rest`
Expected: FAIL — current endpoint passes null claimant when no query param is sent, causing `claimantId is required` exception.

- [ ] **Step 3: Commit failing test**

```bash
git add rest/src/test/java/io/casehub/work/rest/WorkItemPrincipalIdentityTest.java
git commit -m "test: add failing principal identity tests for W1 Refs #424"
```

---

## Batch 2: API surface changes — SPI interfaces and request records

### Task 2: Remove actor/claimant params from WorkItemLifecycleApi

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/spi/WorkItemLifecycleApi.java`

**Interfaces:**
- Produces: `WorkItemLifecycleApi` with actor/claimant params removed from all 16 methods that had them. Only `@PathParam` and `@ContextParam("tenancyId")` remain.

- [ ] **Step 1: Remove actor/claimant params from all methods**

Use `ide_replace_member` for each method. The pattern for every method: remove `String actor` or `String claimant` parameter, keep `@PathParam UUID workItemId` and `@ContextParam("tenancyId") String tenancyId`, keep any request body parameter.

New signatures (all 16 affected methods):

```java
WorkItemView claim(@PathParam UUID workItemId, @ContextParam("tenancyId") String tenancyId);
WorkItemView start(@PathParam UUID workItemId, @ContextParam("tenancyId") String tenancyId);
WorkItemView complete(@PathParam UUID workItemId, CompleteRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView reject(@PathParam UUID workItemId, RejectRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView delegate(@PathParam UUID workItemId, DelegateRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView acceptDelegation(@PathParam UUID workItemId, @ContextParam("tenancyId") String tenancyId);
WorkItemView declineDelegation(@PathParam UUID workItemId, @ContextParam("tenancyId") String tenancyId);
WorkItemView release(@PathParam UUID workItemId, @ContextParam("tenancyId") String tenancyId);
WorkItemView suspend(@PathParam UUID workItemId, SuspendRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView resume(@PathParam UUID workItemId, @ContextParam("tenancyId") String tenancyId);
WorkItemView cancel(@PathParam UUID workItemId, CancelRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView fault(@PathParam UUID workItemId, FaultRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView obsolete(@PathParam UUID workItemId, ObsoleteRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView escalate(@PathParam UUID workItemId, EscalateRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView extend(@PathParam UUID workItemId, ExtendRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView updateDeadline(@PathParam UUID workItemId, UpdateDeadlineRequest body, @ContextParam("tenancyId") String tenancyId);
WorkItemView compensate(@PathParam UUID workItemId, CompensateRequest body, @ContextParam("tenancyId") String tenancyId);
```

- [ ] **Step 2: Verify api module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: PASS (interface change alone compiles)

- [ ] **Step 3: Commit**

```bash
git add api/src/main/java/io/casehub/work/api/spi/WorkItemLifecycleApi.java
git commit -m "refactor: remove actor/claimant params from WorkItemLifecycleApi Refs #424"
```

### Task 3: Remove actor from request records and claimant from queue SPI

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/view/FaultRequest.java`
- Modify: `api/src/main/java/io/casehub/work/api/view/ObsoleteRequest.java`
- Modify: `api/src/main/java/io/casehub/work/api/view/CompensateRequest.java`
- Modify: `api/src/main/java/io/casehub/work/api/view/BulkOperationRequest.java`
- Modify: `api/src/main/java/io/casehub/work/api/spi/WorkItemQueueStateApi.java`

**Interfaces:**
- Produces: request records without actor fields; `WorkItemQueueStateApi.pickup()` without claimant param

- [ ] **Step 1: Update FaultRequest — remove actor field**

```java
public record FaultRequest(String errorDetail) {}
```

- [ ] **Step 2: Update ObsoleteRequest — remove actor field**

```java
public record ObsoleteRequest(String reason) {}
```

- [ ] **Step 3: Update CompensateRequest — remove actor field**

```java
public record CompensateRequest(String title, String candidateGroups, String reason) {}
```

- [ ] **Step 4: Update BulkOperationRequest — remove actorId field**

```java
public record BulkOperationRequest(String operation, List<String> workItemIds, String reason) {}
```

- [ ] **Step 5: Update WorkItemQueueStateApi.pickup() — remove claimant param**

```java
@PlatformMutation("Pick up a work item from a queue")
WorkItemView pickup(@PathParam UUID workItemId, @ContextParam("tenancyId") String tenancyId);
```

- [ ] **Step 6: Verify api module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/
git commit -m "refactor: remove actor fields from request records and queue SPI Refs #424"
```

---

## Batch 3: Wire CurrentPrincipal in implementations

### Task 4: Wire CurrentPrincipal in DefaultWorkItemLifecycleApi

**Files:**
- Modify: `rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemLifecycleApi.java`

**Interfaces:**
- Consumes: `CurrentPrincipal` (io.casehub.platform.api.identity.CurrentPrincipal), `WorkItemOperations`
- Produces: implementation that derives actor from `currentPrincipal.actorId()` for all lifecycle methods

- [ ] **Step 1: Add CurrentPrincipal injection**

Add field:
```java
@Inject
io.casehub.platform.api.identity.CurrentPrincipal currentPrincipal;
```

- [ ] **Step 2: Update all method implementations**

Each method: remove the `String actor`/`String claimant` parameter, use `currentPrincipal.actorId()` when calling `workItemOperations`.

```java
@Override
public WorkItemView claim(UUID workItemId, String tenancyId) {
    return ViewMapper.toView(workItemOperations.claim(workItemId, currentPrincipal.actorId()));
}

@Override
public WorkItemView start(UUID workItemId, String tenancyId) {
    return ViewMapper.toView(workItemOperations.start(workItemId, currentPrincipal.actorId()));
}

@Override
public WorkItemView complete(UUID workItemId, CompleteRequest body, String tenancyId) {
    String resolution = body != null ? body.resolution() : null;
    String outcome = body != null ? body.outcome() : null;
    return ViewMapper.toView(workItemOperations.complete(workItemId, currentPrincipal.actorId(), resolution, outcome));
}

@Override
public WorkItemView reject(UUID workItemId, RejectRequest body, String tenancyId) {
    String reason = body != null ? body.reason() : null;
    String outcome = body != null ? body.outcome() : null;
    return ViewMapper.toView(workItemOperations.reject(workItemId, currentPrincipal.actorId(), reason, outcome));
}

@Override
public WorkItemView delegate(UUID workItemId, DelegateRequest body, String tenancyId) {
    return ViewMapper.toView(
            workItemOperations.delegate(workItemId, currentPrincipal.actorId(), body.to(), body.declineTarget()));
}

@Override
public WorkItemView acceptDelegation(UUID workItemId, String tenancyId) {
    return ViewMapper.toView(workItemOperations.acceptDelegation(workItemId, currentPrincipal.actorId()));
}

@Override
public WorkItemView declineDelegation(UUID workItemId, String tenancyId) {
    return ViewMapper.toView(workItemOperations.declineDelegation(workItemId, currentPrincipal.actorId()));
}

@Override
public WorkItemView release(UUID workItemId, String tenancyId) {
    return ViewMapper.toView(workItemOperations.release(workItemId, currentPrincipal.actorId()));
}

@Override
public WorkItemView suspend(UUID workItemId, SuspendRequest body, String tenancyId) {
    String reason = body != null ? body.reason() : null;
    return ViewMapper.toView(workItemOperations.suspend(workItemId, currentPrincipal.actorId(), reason));
}

@Override
public WorkItemView resume(UUID workItemId, String tenancyId) {
    return ViewMapper.toView(workItemOperations.resume(workItemId, currentPrincipal.actorId()));
}

@Override
public WorkItemView cancel(UUID workItemId, CancelRequest body, String tenancyId) {
    String reason = body != null ? body.reason() : null;
    return ViewMapper.toView(workItemOperations.cancel(workItemId, currentPrincipal.actorId(), reason));
}

@Override
public WorkItemView fault(UUID workItemId, FaultRequest body, String tenancyId) {
    return ViewMapper.toView(
            workItemOperations.fault(workItemId, currentPrincipal.actorId(), body.errorDetail()));
}

@Override
public WorkItemView obsolete(UUID workItemId, ObsoleteRequest body, String tenancyId) {
    return ViewMapper.toView(
            workItemOperations.obsolete(workItemId, currentPrincipal.actorId(), body.reason()));
}

@Override
public WorkItemView escalate(UUID workItemId, EscalateRequest body, String tenancyId) {
    return ViewMapper.toView(
            workItemOperations.escalate(workItemId, currentPrincipal.actorId(), body.targetGroup(), body.reason()));
}

@Override
public WorkItemView extend(UUID workItemId, ExtendRequest body, String tenancyId) {
    return ViewMapper.toView(
            workItemOperations.extend(workItemId, body != null ? body.newExpiresAt() : null, currentPrincipal.actorId()));
}

@Override
public WorkItemView updateDeadline(UUID workItemId, UpdateDeadlineRequest body, String tenancyId) {
    return ViewMapper.toView(
            workItemOperations.updateDeadline(workItemId, body != null ? body.newDeadline() : null, currentPrincipal.actorId()));
}

@Override
public WorkItemView compensate(UUID workItemId, CompensateRequest body, String tenancyId) {
    WorkItemCreateRequest request = WorkItemCreateRequest.builder()
            .title(body.title())
            .candidateGroups(body.candidateGroups())
            .createdBy(currentPrincipal.actorId())
            .build();
    return ViewMapper.toView(
            workItemOperations.compensate(workItemId, request, currentPrincipal.actorId(), body.reason()));
}
```

- [ ] **Step 3: Verify rest module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl rest`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemLifecycleApi.java
git commit -m "feat: wire CurrentPrincipal in DefaultWorkItemLifecycleApi Refs #424"
```

### Task 5: Wire CurrentPrincipal in DefaultWorkItemQueueStateApi and DefaultWorkItemBulkApi

**Files:**
- Modify: `queues/src/main/java/io/casehub/work/queues/service/DefaultWorkItemQueueStateApi.java`
- Modify: `rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemBulkApi.java`

**Interfaces:**
- Consumes: `CurrentPrincipal`
- Produces: implementations using `currentPrincipal.actorId()` instead of request params

- [ ] **Step 1: Update DefaultWorkItemQueueStateApi**

Add `CurrentPrincipal` injection:
```java
@Inject
io.casehub.platform.api.identity.CurrentPrincipal currentPrincipal;
```

Update `pickup` signature and body:
```java
@Override
public WorkItemView pickup(UUID workItemId, String tenancyId) {
    final String claimant = currentPrincipal.actorId();
    final WorkItem wi = workItemStore.get(workItemId).orElse(null);
    // ... rest of method unchanged, using local `claimant` variable
}
```

- [ ] **Step 2: Update DefaultWorkItemBulkApi**

Add `CurrentPrincipal` injection:
```java
@Inject
io.casehub.platform.api.identity.CurrentPrincipal currentPrincipal;
```

Update `bulk` method — replace `request.actorId()` with `currentPrincipal.actorId()`:
```java
final String actor = currentPrincipal.actorId();
```

- [ ] **Step 3: Verify both modules compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl queues,rest`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add queues/src/main/java/io/casehub/work/queues/service/DefaultWorkItemQueueStateApi.java
git add rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemBulkApi.java
git commit -m "feat: wire CurrentPrincipal in queue and bulk API implementations Refs #424"
```

---

## Batch 4: Rebuild, regenerate, and update SPI test

### Task 6: Rebuild to regenerate REST/GraphQL and fix SpiInterfaceTest

**Files:**
- Modify: `api/src/test/java/io/casehub/work/api/spi/SpiInterfaceTest.java`

**Interfaces:**
- Consumes: regenerated `WorkLifecycleResource` (auto-generated from updated SPI)
- Produces: passing SpiInterfaceTest with correct method count

- [ ] **Step 1: Rebuild api module to trigger annotation processor**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`

- [ ] **Step 2: Rebuild rest module to regenerate WorkLifecycleResource**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl rest`

Verify: read the regenerated `rest/target/generated-sources/annotations/io/casehub/work/rest/spi/WorkLifecycleResource.java` and confirm `@QueryParam("actor")` and `@QueryParam("claimant")` are gone.

- [ ] **Step 3: Run SpiInterfaceTest — expect pass or update**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=SpiInterfaceTest -pl api`

The method count test (`workItemLifecycleApiMethodCount`) asserts 17 methods — this should still pass since we changed signatures, not method count. If any assertion fails, update the expected values.

- [ ] **Step 4: Commit**

```bash
git add api/src/test/java/io/casehub/work/api/spi/SpiInterfaceTest.java
git commit -m "test: update SpiInterfaceTest after SPI signature changes Refs #424"
```

---

## Batch 5: Test migration — update existing REST tests

### Task 7: Migrate rest module tests to use MutableCurrentPrincipal

**Files:**
- Modify: `rest/src/test/java/io/casehub/work/rest/WorkItemDelegationTest.java`
- Modify: `rest/src/test/java/io/casehub/work/rest/WorkItemExcludedUsersTest.java`
- Modify: `rest/src/test/java/io/casehub/work/rest/WorkItemOptimisticLockTest.java`
- Modify: `rest/src/test/java/io/casehub/work/rest/WorkItemBulkTest.java`
- Modify: `rest/src/test/java/io/casehub/work/rest/WorkItemCloneTest.java`
- Modify: `rest/src/test/java/io/casehub/work/rest/WorkItemSchemaValidationTest.java`
- Modify: `rest/src/test/java/io/casehub/work/rest/MultiTenancyIT.java`
- Modify: any other rest test files that use `?actor=` or `?claimant=` query params

**Interfaces:**
- Consumes: `MutableCurrentPrincipal` (rest/src/test)
- Produces: all rest tests passing with principal-based identity

- [ ] **Step 1: Identify all affected test files**

Use `ide_search_text` to find all `?actor=` and `?claimant=` and `queryParam("actor"` and `queryParam("claimant"` in test files.

- [ ] **Step 2: Pattern for each test file**

For each test class that uses query params for actor/claimant:

1. Add `@Inject MutableCurrentPrincipal principal;` field
2. Replace `given().queryParam("claimant", "alice").post(...)` with:
   ```java
   principal.setActorId("alice");
   given().post(...)
   ```
3. Replace `given().queryParam("actor", "bob").post(...)` with:
   ```java
   principal.setActorId("bob");
   given().post(...)
   ```
4. Remove `?claimant=alice` and `?actor=bob` from URL strings like `"/api/work/lifecycle/claim/" + id + "?claimant=alice"`

Where multiple users act in sequence within one test, set `principal.setActorId()` before each call.

- [ ] **Step 3: Update request body tests**

For tests sending `FaultRequest`, `ObsoleteRequest`, `CompensateRequest`, or `BulkOperationRequest` with actor fields in JSON — remove the actor field from the JSON body and ensure `principal.setActorId()` is set.

- [ ] **Step 4: Run all rest module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add rest/src/test/
git commit -m "test: migrate rest tests to principal-based identity Refs #424"
```

### Task 8: Migrate queues module tests

**Files:**
- Modify: `queues/src/test/java/io/casehub/work/queues/api/QueueStateResourceTest.java`
- Modify: `queues/src/test/java/io/casehub/work/queues/api/FullPipelineTest.java`
- Modify: `queues/src/test/java/io/casehub/work/queues/api/QueueResourceTest.java`
- Modify: `queues/src/test/java/io/casehub/work/queues/event/WorkItemQueueEventTest.java`

**Interfaces:**
- Consumes: `MutableCurrentPrincipal` (queues/src/test)
- Produces: all queues tests passing with principal-based identity

- [ ] **Step 1: Apply same migration pattern as Task 7**

Add `@Inject MutableCurrentPrincipal principal;` and replace query param patterns with `principal.setActorId(...)`.

- [ ] **Step 2: Run queues module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues`
Expected: ALL PASS

- [ ] **Step 3: Commit**

```bash
git add queues/src/test/
git commit -m "test: migrate queues tests to principal-based identity Refs #424"
```

### Task 9: Run full build and verify principal identity tests

**Files:**
- Test: `rest/src/test/java/io/casehub/work/rest/WorkItemPrincipalIdentityTest.java`

- [ ] **Step 1: Run the new principal identity tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemPrincipalIdentityTest -pl rest`
Expected: ALL PASS — actor comes from principal, query param ignored

- [ ] **Step 2: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl api,runtime-core,runtime,rest,queues,federation`
Expected: ALL PASS

- [ ] **Step 3: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests`
Expected: ALL PASS

- [ ] **Step 4: Final commit if any fixups needed**

```bash
git add -A
git commit -m "fix: resolve remaining test failures from W1 migration Refs #424"
```

---

## References

- [2026-11-17-w1-actor-from-principal-design.md] — design spec this plan implements
- [WorkItemLifecycleApi.java] — api/src/main/java/io/casehub/work/api/spi/WorkItemLifecycleApi.java
- [DefaultWorkItemLifecycleApi.java] — rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemLifecycleApi.java
- [WorkItemOperations.java] — api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java
- [WorkItemQueueStateApi.java] — api/src/main/java/io/casehub/work/api/spi/WorkItemQueueStateApi.java
- [DefaultWorkItemQueueStateApi.java] — queues/src/main/java/io/casehub/work/queues/service/DefaultWorkItemQueueStateApi.java
- [WorkItemBulkApi.java] — api/src/main/java/io/casehub/work/api/spi/WorkItemBulkApi.java
- [DefaultWorkItemBulkApi.java] — rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemBulkApi.java
- [SpiInterfaceTest.java] — api/src/test/java/io/casehub/work/api/spi/SpiInterfaceTest.java
- [MutableCurrentPrincipal] — rest/src/test, queues/src/test, runtime/src/test
- [playbook-direction.md] — W1 finding
- [GitHub #424] — focal issue
- [GitHub #423] — parent epic
