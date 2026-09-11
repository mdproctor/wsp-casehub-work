# Engine Repo Briefing — #399 TenantContextExecutor Removal

**From:** casehub-work branch `issue-399-remove-tenant-context-executor-spi`
**Requires:** work-api `0.2-SNAPSHOT` with this branch merged

## What Changed in work-api

1. `TenantContextExecutor` interface **deleted** from `io.casehub.work.api.spi`
2. `WorkItemOperations` gained a new `default` method:
   ```java
   default WorkItem createInTenantContext(String tenancyId, WorkItemCreateRequest request)
   ```
   `WorkItemService` implements it by delegating to `TenantContextRunner` + `create()`.

## Required Engine Changes

### `InboundWorkItemBridge` (`casehub-engine-inbound`)

**Current code** (after engine#974):
```java
@Inject WorkItemOperations workItemOperations;
@Inject TenantContextExecutor tenantContextExecutor;

// In onMessage():
tenantContextExecutor.runInTenantContext(
    event.tenancyId(), () -> workItemOperations.create(stamp(request)));
```

**Updated code:**
```java
@Inject WorkItemOperations workItemOperations;
// TenantContextExecutor injection removed

// In onMessage():
workItemOperations.createInTenantContext(event.tenancyId(), stamp(request));
```

### Files to modify

| File | Change |
|------|--------|
| `casehub-engine-inbound/src/main/java/.../InboundWorkItemBridge.java` | Drop `TenantContextExecutor` import + injection; replace `tenantContextExecutor.runInTenantContext(...)` with `workItemOperations.createInTenantContext(...)` |
| `casehub-engine-inbound/src/test/java/.../InboundWorkItemBridgeTest.java` | Remove `RecordingTenantContextRunner` inner class; mock `workItemOperations.createInTenantContext()` instead |
| `casehub-engine-inbound/src/test/java/.../InboundWorkItemBridgeGuardTest.java` | Remove `TenantContextExecutor` mock |
| `casehub-engine-inbound/pom.xml` | Verify `casehub-work-api` still resolves (no other changes needed — `TenantContextExecutor` was the only type removed) |

### Javadoc updates

`InboundWorkItemBridge` Javadoc references `TenantContextExecutor` in exception handling and request context paragraphs. Update to reference `WorkItemOperations.createInTenantContext()`.
