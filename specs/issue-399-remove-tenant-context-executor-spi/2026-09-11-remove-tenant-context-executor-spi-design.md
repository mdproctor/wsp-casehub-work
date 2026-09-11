# Remove TenantContextExecutor SPI

**Issue:** casehubio/work#399
**Date:** 2026-09-11
**Status:** Approved

## Problem

`TenantContextExecutor` exists as an SPI in `casehub-work-api` with a single method:
`void runInTenantContext(String tenancyId, Runnable work)`. Within the work repo, zero
production callers inject it — all 7 callers inject `TenantContextRunner` (the implementation
in `casehub-work-runtime`) directly. The SPI is unnecessary indirection within the work repo.

However, engine#974 moved the engine repo's `InboundWorkItemBridge` FROM `TenantContextRunner`
TO `TenantContextExecutor` to break the compile dependency on `casehub-work` runtime. The
engine repo is now an active consumer of this SPI. Deleting it requires a coordinated change.

## Approach

Absorb tenant context into `WorkItemOperations` — the SPI the engine bridge already injects.
Add a `default` method `createInTenantContext(String tenancyId, WorkItemCreateRequest request)`
that combines tenant context establishment with work item creation. The engine bridge drops
`TenantContextExecutor` and calls the combined method. The work repo deletes the SPI.

The `tenancyId` parameter is explicit and separate from request fields. This preserves the
compliance audit trail — every tenant boundary crossing remains visible in code, satisfying
the `CurrentPrincipal.tenancyId()` Javadoc contract: "must never be sourced from user-supplied
input — always derived from the authenticated security context."

## Work Repo Changes

### 1. Add `createInTenantContext` to `WorkItemOperations`

`api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java`:

```java
default WorkItem createInTenantContext(String tenancyId, WorkItemCreateRequest request) {
    throw new UnsupportedOperationException(
        "Tenant-context-aware creation not supported by this implementation");
}
```

`default` method with `UnsupportedOperationException` — follows the existing
`WorkItemCreator.createMultiInstance()` pattern. Non-breaking addition: existing
implementors (mocks, test doubles) compile without change.

### 2. Implement in `WorkItemService`

`WorkItemService` implements `WorkItemOperations`. Override:

```java
@Override
public WorkItem createInTenantContext(String tenancyId, WorkItemCreateRequest request) {
    var result = new WorkItem[1];
    tenantContextRunner.runInTenantContext(tenancyId, () -> result[0] = create(request));
    return result[0];
}
```

`WorkItemService` already injects `TenantContextRunner` — no new dependency needed.

### 3. Update `InboundWorkItemSchedulerImpl`

Switch from `TenantContextExecutor` (work-api SPI) to `TenantContextRunner` (work-runtime
implementation) directly. The engine-adapter is a bridge module where work-runtime is always
present on the classpath — the compile dependency is justified (per memory:
`feedback_compile_time_over_wrappers`).

Before:
```java
@Inject TenantContextExecutor tenantContextExecutor;
// ...
tenantContextExecutor.runInTenantContext(request.tenancyId(), () -> workItemCreator.create(createRequest));
```

After:
```java
@Inject TenantContextRunner tenantContextRunner;
// ...
tenantContextRunner.runInTenantContext(request.tenancyId(), () -> workItemCreator.create(createRequest));
```

### 4. POM change — engine-adapter

Promote `casehub-work` from `<scope>test</scope>` to compile scope (default):

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work</artifactId>
</dependency>
```

### 5. Remove `TenantContextExecutor`

- Delete `api/src/main/java/io/casehub/work/api/spi/TenantContextExecutor.java`
- Remove `implements TenantContextExecutor` from `TenantContextRunner` class declaration
- Remove the `@Override` annotation from `TenantContextRunner.runInTenantContext(String, Runnable)`
- Remove `import io.casehub.work.api.spi.TenantContextExecutor` from `TenantContextRunner`

## Engine Repo Changes (briefing — separate session)

`InboundWorkItemBridge` (`casehub-engine-inbound`):

1. Drop `@Inject TenantContextExecutor tenantContextExecutor;`
2. Replace:
   ```java
   tenantContextExecutor.runInTenantContext(
       event.tenancyId(), () -> workItemOperations.create(stamp(request)));
   ```
   With:
   ```java
   workItemOperations.createInTenantContext(event.tenancyId(), stamp(request));
   ```
3. Update `InboundWorkItemBridgeTest` and `InboundWorkItemBridgeGuardTest`:
   - Remove `TenantContextExecutor` mocks/implementations
   - Mock `workItemOperations.createInTenantContext()` instead
4. Remove `TenantContextExecutor` import from POM if it was the only consumer

## Sequencing

The `default` method on `WorkItemOperations` makes this non-breaking at the API level.
Both repos use `0.2-SNAPSHOT`. Deployment order:

1. Work repo publishes SNAPSHOT with `createInTenantContext` added + `TenantContextExecutor` deleted
2. Engine repo updates `InboundWorkItemBridge` against the new SNAPSHOT

Between steps 1 and 2, the engine compiles (the `default` method exists) but calling
`createInTenantContext` at runtime would hit `UnsupportedOperationException` if somehow
the work-runtime implementation isn't present. In practice this cannot happen — engine-inbound
always runs with work-runtime on the classpath.

## Testing

### Updated tests

| Test | Module | Change |
|------|--------|--------|
| `InboundWorkItemSchedulerImplTest` | engine-adapter | Mock `TenantContextRunner` instead of `TenantContextExecutor` |
| `TenantContextRunnerTest` | runtime | Remove any assertions about `TenantContextExecutor` interface conformance (if present) |

### New test

| Test | Module | Description |
|------|--------|-------------|
| `WorkItemService.createInTenantContext` | runtime (or integration-tests) | Verify tenant context wrapping + delegation to `create()` |

### Verification

- `mvn test -pl api` — compiles without `TenantContextExecutor`
- `mvn test -pl runtime` — `TenantContextRunner` compiles without `implements` clause
- `mvn test -pl engine-adapter` — `InboundWorkItemSchedulerImpl` compiles with `TenantContextRunner`
- `mvn verify -pl integration-tests` — full integration test pass

## Schema Impact

None. No database migrations. No new entities. No column changes.

## Documentation Updates

### CLAUDE.md

Update `## engine-adapter Module` section:
- Change "injects `TenantContextExecutor`" references to `TenantContextRunner`
- Document `createInTenantContext` on `WorkItemOperations`

### ARC42STORIES.MD

Update any references to `TenantContextExecutor` in layer entries and building block view.

## References

- `api/src/main/java/io/casehub/work/api/spi/TenantContextExecutor.java` — SPI being removed
- `api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java:13` — target SPI for new method
- `runtime/src/main/java/io/casehub/work/runtime/service/TenantContextRunner.java:28` — implementation, loses `implements` clause
- `engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java:16` — only work-repo consumer of `TenantContextExecutor`
- Engine repo commit `066878ef` — engine#974 introduced engine dependency on `TenantContextExecutor`
- Protocol PP-20260609-fb6563 — `@ObservesAsync` handlers must use `TenantContextRunner`
- `docs/specs/issue-397-inbound-scheduler-actor-state/2026-09-06-inbound-scheduler-actor-state-design.md` — #397 spec (InboundWorkItemSchedulerImpl origin)
- Issue #399 body — rejected alternative (auto-establish from request fields)
