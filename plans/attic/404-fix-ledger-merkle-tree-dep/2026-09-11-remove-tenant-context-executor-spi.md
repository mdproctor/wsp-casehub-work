# Remove TenantContextExecutor SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #399 — Remove unused TenantContextExecutor SPI — callers inject TenantContextRunner directly
**Issue group:** #399

**Goal:** Remove the unused `TenantContextExecutor` SPI from `casehub-work-api`, absorbing its functionality into `WorkItemOperations.createInTenantContext()` for cross-repo consumers, and switching in-repo callers to `TenantContextRunner` directly.

**Architecture:** Add a `default` method `createInTenantContext(String, WorkItemCreateRequest)` to `WorkItemOperations` (work-api SPI). Implement in `WorkItemService` (work-runtime) by delegating to `TenantContextRunner` + `create()`. Update `InboundWorkItemSchedulerImpl` (engine-adapter) to inject `TenantContextRunner` directly. Then delete `TenantContextExecutor` interface and clean up `TenantContextRunner`'s `implements` clause.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JUnit 5, Mockito, AssertJ

## Global Constraints

- Java 21 source (on Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build commands must target specific modules: `mvn test -pl <module>`
- Use `ide_insert_member` / `ide_replace_member` for structural edits
- Use `ide_refactor_safe_delete` for file deletion
- Commit every task with `Refs #399`

---

## Batch 1: Add createInTenantContext + update InboundWorkItemSchedulerImpl

After this batch: `createInTenantContext` exists on `WorkItemOperations`, `InboundWorkItemSchedulerImpl` uses `TenantContextRunner` directly, `TenantContextExecutor` is still present (no breakage for engine repo). Safe wrap point.

### Task 1: Add `createInTenantContext` to WorkItemOperations and implement in WorkItemService

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java:13` — add default method
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java:46` — add `TenantContextRunner` field injection + override method
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java` — add test for `createInTenantContext`

**Interfaces:**
- Consumes: `TenantContextRunner.runInTenantContext(String, Runnable)` (runtime, existing), `WorkItemService.create(WorkItemCreateRequest)` (runtime, existing)
- Produces: `WorkItemOperations.createInTenantContext(String, WorkItemCreateRequest)` — used by engine repo's `InboundWorkItemBridge` (future), and available for any cross-repo consumer that needs tenant-aware work item creation

- [ ] **Step 1: Add default method to `WorkItemOperations`**

Use `ide_insert_member` to add after the `create` method at line 15:

```java
/**
 * Create a work item within an explicitly established tenant context.
 *
 * <p>Activates a request scope for the given {@code tenancyId}, creates the work item,
 * and tears down the context. Use this from async/background paths where no request
 * scope is active (e.g., qhorus afterCompletion callbacks, inbound bridges).
 *
 * @param tenancyId  the tenant identity to establish — must be derived from the
 *                   authenticated security context, never from user-supplied input
 * @param request    the work item to create
 * @return the created work item
 */
default WorkItem createInTenantContext(String tenancyId, WorkItemCreateRequest request) {
    throw new UnsupportedOperationException(
        "Tenant-context-aware creation not supported by this implementation");
}
```

- [ ] **Step 2: Verify API module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 3: Write the failing test in `WorkItemServiceTest`**

Check the test class structure first with `ide_file_structure` on `WorkItemServiceTest.java` to understand the existing test pattern (QuarkusTest vs plain JUnit).

Add a test method. Since `WorkItemService` is `@ApplicationScoped` and `create()` is `@Transactional`, the test for `createInTenantContext` needs a running CDI container. Add to `WorkItemServiceTest`:

```java
@Test
void createInTenantContext_establishes_context_and_delegates_to_create() {
    var request = WorkItemCreateRequest.builder()
            .title("Tenant-scoped item")
            .tenancyId("test-tenant-ctx")
            .build();

    var result = workItemService.createInTenantContext("test-tenant-ctx", request);

    assertThat(result).isNotNull();
    assertThat(result.getTenancyId()).isEqualTo("test-tenant-ctx");
    assertThat(result.getTitle()).isEqualTo("Tenant-scoped item");
    assertThat(result.getStatus()).isEqualTo(WorkItemStatus.PENDING);
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemServiceTest#createInTenantContext_establishes_context_and_delegates_to_create`
Expected: FAIL — `UnsupportedOperationException` from the default method (or compilation error if the test file uses the method before the override exists)

- [ ] **Step 5: Add `TenantContextRunner` field injection to `WorkItemService`**

Use `ide_insert_member` to add a field after the existing field injections (after line 81):

```java
@Inject
TenantContextRunner tenantContextRunner;
```

Add the import: `import io.casehub.work.runtime.service.TenantContextRunner;` — though this is same-package, so no import needed.

- [ ] **Step 6: Implement `createInTenantContext` in `WorkItemService`**

Use `ide_insert_member` to add the method:

```java
@Override
public io.casehub.work.api.WorkItem createInTenantContext(String tenancyId,
        WorkItemCreateRequest request) {
    var result = new io.casehub.work.api.WorkItem[1];
    tenantContextRunner.runInTenantContext(tenancyId, () -> result[0] = create(request));
    return result[0];
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemServiceTest#createInTenantContext_establishes_context_and_delegates_to_create`
Expected: PASS

- [ ] **Step 8: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: BUILD SUCCESS — no regressions

- [ ] **Step 9: Commit**

```bash
git -C "$PROJECT" add api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java
git -C "$PROJECT" commit -m "feat(#399): add createInTenantContext to WorkItemOperations

Adds a default method to WorkItemOperations SPI for cross-repo consumers
that need tenant-aware work item creation. WorkItemService implements by
delegating to TenantContextRunner + create().

Refs #399"
```

### Task 2: Update InboundWorkItemSchedulerImpl to inject TenantContextRunner directly

**Files:**
- Modify: `engine-adapter/pom.xml` — promote `casehub-work` from test to compile scope
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java` — change injection from `TenantContextExecutor` to `TenantContextRunner`
- Modify: `engine-adapter/src/test/java/io/casehub/work/engine/InboundWorkItemSchedulerImplTest.java` — update mock

**Interfaces:**
- Consumes: `TenantContextRunner.runInTenantContext(String, Runnable)` (runtime, existing)
- Produces: no interface changes — `InboundWorkItemSchedulerImpl` still implements `InboundWorkItemScheduler`

- [ ] **Step 1: Update engine-adapter POM**

In `engine-adapter/pom.xml`, change `casehub-work` from test scope to compile scope (default). Use Edit tool to remove the `<scope>test</scope>` line:

Before (lines 45-48):
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work</artifactId>
    <scope>test</scope>
</dependency>
```

After:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work</artifactId>
</dependency>
```

- [ ] **Step 2: Update test mock — change TenantContextExecutor to TenantContextRunner**

In `InboundWorkItemSchedulerImplTest.java`, replace `TenantContextExecutor` with `TenantContextRunner` throughout:

1. Change import: `io.casehub.work.api.spi.TenantContextExecutor` → `io.casehub.work.runtime.service.TenantContextRunner`
2. Change field declaration (line 28): `private TenantContextExecutor tenantContext;` → `private TenantContextRunner tenantContext;`
3. Change mock creation (line 34): `tenantContext = mock(TenantContextExecutor.class);` → `tenantContext = mock(TenantContextRunner.class);`
4. Constructor call (line 39): unchanged — `new InboundWorkItemSchedulerImpl(creator, tenantContext)` (constructor signature updates in Step 3)

- [ ] **Step 3: Update InboundWorkItemSchedulerImpl source**

In `InboundWorkItemSchedulerImpl.java`:

1. Change import: `io.casehub.work.api.spi.TenantContextExecutor` → `io.casehub.work.runtime.service.TenantContextRunner`
2. Change field injection (line 16): `@Inject TenantContextExecutor tenantContextExecutor;` → `@Inject TenantContextRunner tenantContextRunner;`
3. Update constructor parameter (line 22): `final TenantContextExecutor tenantContextExecutor` → `final TenantContextRunner tenantContextRunner`
4. Update constructor body (line 24): `this.tenantContextExecutor = tenantContextExecutor;` → `this.tenantContextRunner = tenantContextRunner;`
5. Update field declaration (no explicit line — implied by injection): field name changes to `tenantContextRunner`
6. Update usage in `schedule()` (line 45): `tenantContextExecutor.runInTenantContext(` → `tenantContextRunner.runInTenantContext(`

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -Dtest=InboundWorkItemSchedulerImplTest`
Expected: PASS — all 5 tests pass

- [ ] **Step 5: Run full engine-adapter test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git -C "$PROJECT" add engine-adapter/pom.xml engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java engine-adapter/src/test/java/io/casehub/work/engine/InboundWorkItemSchedulerImplTest.java
git -C "$PROJECT" commit -m "refactor(#399): InboundWorkItemSchedulerImpl injects TenantContextRunner directly

Promote casehub-work from test to compile scope in engine-adapter POM.
Runtime is always present when engine-adapter is on the classpath, so
the compile dependency is justified over the SPI indirection.

Refs #399"
```

## Batch 2: Delete TenantContextExecutor SPI + documentation

After this batch: `TenantContextExecutor` is deleted, `TenantContextRunner` no longer implements it, CLAUDE.md and ARC42STORIES.MD are updated. Engine repo briefing written. **Breaking for engine repo** — engine must update `InboundWorkItemBridge` against the new work-api SNAPSHOT.

### Task 3: Delete TenantContextExecutor and clean up TenantContextRunner

**Files:**
- Delete: `api/src/main/java/io/casehub/work/api/spi/TenantContextExecutor.java` (use `ide_refactor_safe_delete`)
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/TenantContextRunner.java:28` — remove `implements TenantContextExecutor`, remove `@Override`, remove import
- Modify: `CLAUDE.md` — update engine-adapter section
- Modify: `ARC42STORIES.MD` — update references

**Interfaces:**
- Consumes: nothing new
- Produces: `TenantContextRunner.runInTenantContext(String, Runnable)` remains as a concrete method (no longer an interface override)

- [ ] **Step 1: Remove `implements TenantContextExecutor` from `TenantContextRunner`**

In `TenantContextRunner.java` (line 28):

Before:
```java
public class TenantContextRunner implements TenantContextExecutor {
```

After:
```java
public class TenantContextRunner {
```

Remove import (line 3):
```java
import io.casehub.work.api.spi.TenantContextExecutor;
```

Remove `@Override` annotation from `runInTenantContext(String, Runnable)` method (line 36).

- [ ] **Step 2: Verify runtime compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime`
Expected: BUILD SUCCESS

- [ ] **Step 3: Delete `TenantContextExecutor.java`**

Use `ide_refactor_safe_delete` on `api/src/main/java/io/casehub/work/api/spi/TenantContextExecutor.java`.

If safe-delete finds usages (in specs, docs, blog, plans — non-code files), proceed anyway — those are documentation references, not code dependencies. All code references were removed in Tasks 1-2.

- [ ] **Step 4: Verify API module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 5: Run full test suite for affected modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,engine-adapter`
Expected: BUILD SUCCESS — all tests pass

- [ ] **Step 6: Update CLAUDE.md engine-adapter section**

In `CLAUDE.md`, update the `## engine-adapter Module` section. Find:
```
`TenantContextExecutor` (work-api SPI)
```
and similar references. Replace with `TenantContextRunner` (work-runtime).

Update the routing context threading paragraph to reflect that `InboundWorkItemSchedulerImpl` now uses `TenantContextRunner` directly.

Add a note about `WorkItemOperations.createInTenantContext()` for cross-repo consumers.

- [ ] **Step 7: Update ARC42STORIES.MD**

Search for `TenantContextExecutor` references in `ARC42STORIES.MD`. Update to reflect the SPI removal. Typical locations: §5 Building Block View, §9.4 Layer Entries.

- [ ] **Step 8: Write engine repo briefing**

Create a briefing file at `$WORKSPACE/HANDOFF-engine-399.md` documenting the required engine repo changes:
- `InboundWorkItemBridge` drops `TenantContextExecutor` injection
- Replace `tenantContextExecutor.runInTenantContext(tenancyId, () -> workItemOperations.create(request))` with `workItemOperations.createInTenantContext(tenancyId, request)`
- Update `InboundWorkItemBridgeTest` and `InboundWorkItemBridgeGuardTest`
- Requires work-api `0.2-SNAPSHOT` with `createInTenantContext`

- [ ] **Step 9: Commit**

```bash
git -C "$PROJECT" add api/src/main/java/io/casehub/work/api/spi/ runtime/src/main/java/io/casehub/work/runtime/service/TenantContextRunner.java CLAUDE.md ARC42STORIES.MD
git -C "$PROJECT" commit -m "refactor(#399): delete TenantContextExecutor SPI

All in-repo callers inject TenantContextRunner directly. Cross-repo
consumers (engine InboundWorkItemBridge) use the new
WorkItemOperations.createInTenantContext() method instead.

Closes #399"
```

Commit the engine briefing to workspace:
```bash
git -C "$WORKSPACE" add HANDOFF-engine-399.md
git -C "$WORKSPACE" commit -m "wip: engine repo briefing for #399 coordination Refs #399"
```

## References

- [specs/issue-399-remove-tenant-context-executor-spi/2026-09-11-remove-tenant-context-executor-spi-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/work/api/spi/TenantContextExecutor.java:25] — SPI being removed
- [api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java:13] — target SPI for new method
- [runtime/src/main/java/io/casehub/work/runtime/service/TenantContextRunner.java:28] — implementation losing `implements` clause
- [runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java:46] — `WorkItemOperations` implementor
- [engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java:16] — only work-repo consumer
- [engine-adapter/src/test/java/io/casehub/work/engine/InboundWorkItemSchedulerImplTest.java] — test to update
- [runtime/src/test/java/io/casehub/work/runtime/service/TenantContextRunnerTest.java] — existing tests (unchanged)
- [Protocol PP-20260609-fb6563] — async-event-tenant-context-propagation
- [Engine repo commit 066878ef] — engine#974 introduced engine dependency on TenantContextExecutor
- [GitHub #399] — focal issue
- [GitHub #397] — InboundWorkItemSchedulerImpl origin
