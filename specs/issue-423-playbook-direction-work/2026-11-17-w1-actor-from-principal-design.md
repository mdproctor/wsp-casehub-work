# W1: Derive Actor/Claimant from CurrentPrincipal

**Issue:** #424 (child of #423 — Playbook language direction — work)
**Finding:** W1 from playbook-direction review
**Date:** 2026-11-17

## Problem

The `WorkItemLifecycleApi` SPI takes `String actor` / `String claimant` as method parameters. The platform annotation processor (`@McpDomain` / `@PlatformMutation`) maps these unannotated String params to `@QueryParam` in the generated REST resource and to GraphQL arguments in the generated resolver. Any HTTP caller can impersonate any actor by setting `?actor=alice` or `?claimant=bob`.

The generated resource already injects `CurrentPrincipal` and uses `currentPrincipal.tenancyId()` for `@ContextParam("tenancyId")` — the principal is available but only used for tenancy, not identity.

Three user-facing API surfaces are affected:

| SPI | Method(s) | Untrusted param |
|-----|-----------|-----------------|
| `WorkItemLifecycleApi` | 16 methods (claim, start, complete, reject, delegate, ...) | `String actor` or `String claimant` |
| `WorkItemQueueStateApi` | `pickup()` | `String claimant` |
| `WorkItemBulkApi` | `bulk()` via `BulkOperationRequest` | `actorId` field in request body |

## Design

### Layer separation

The fix applies to the **user-facing SPI layer** only. The internal `WorkItemOperations` interface keeps its `actorId`/`claimantId` parameters — these are used by system callers that legitimately specify an actor:

| Caller | Goes through | Actor source |
|--------|-------------|-------------|
| REST/GraphQL (human user) | `WorkItemLifecycleApi` → `DefaultWorkItemLifecycleApi` → `WorkItemOperations` | `CurrentPrincipal.actorId()` |
| Queue pickup (human user) | `WorkItemQueueStateApi` → `DefaultWorkItemQueueStateApi` → `WorkItemOperations` | `CurrentPrincipal.actorId()` |
| Bulk operations (human user) | `WorkItemBulkApi` → `DefaultWorkItemBulkApi` → `WorkItemOperations` | `CurrentPrincipal.actorId()` |
| Timer expiry | `WorkItemOperations` directly | `"system"` (via `@WorkSystem` principal) |
| Engine adapter | `WorkItemOperations` directly | `"system"` or engine-provided actor |
| Qhorus bridge | `WorkItemOperations.createInTenantContext()` | `"casehub-engine-inbound"` |
| Federation proxy | `WorkItemOperations` decorator | passes through from above layers |

External callers (engine repo's `InboundWorkItemBridge`) only use `createInTenantContext()` — no lifecycle methods. Confirmed by grep across all casehub repos.

### Changes

#### 1. `WorkItemLifecycleApi` — remove actor/claimant params

Remove the `String actor` and `String claimant` parameters from all 16 methods. The `@ContextParam("tenancyId")` parameter stays.

Before:
```java
WorkItemView claim(@PathParam UUID workItemId, String claimant,
                   @ContextParam("tenancyId") String tenancyId);
```

After:
```java
WorkItemView claim(@PathParam UUID workItemId,
                   @ContextParam("tenancyId") String tenancyId);
```

This applies to: `claim`, `start`, `complete`, `reject`, `delegate`, `acceptDelegation`, `declineDelegation`, `release`, `suspend`, `resume`, `cancel`, `escalate`, `extend`, `updateDeadline`. Methods `fault`, `obsolete`, and `compensate` already take actor in their request body — these need the same treatment (actor from body ignored, principal used).

#### 2. `DefaultWorkItemLifecycleApi` — inject CurrentPrincipal

Inject unqualified `CurrentPrincipal` (request-scoped). Use `currentPrincipal.actorId()` when calling `WorkItemOperations`.

```java
@Inject
CurrentPrincipal currentPrincipal;

@Override
public WorkItemView claim(UUID workItemId, String tenancyId) {
    return ViewMapper.toView(
        workItemOperations.claim(workItemId, currentPrincipal.actorId()));
}
```

For methods where actor was embedded in the request body (`fault`, `obsolete`, `compensate`): ignore `body.actor()` and use `currentPrincipal.actorId()`.

#### 3. `WorkItemQueueStateApi` — remove claimant param from pickup

Before:
```java
WorkItemView pickup(@PathParam UUID workItemId, String claimant,
                    @ContextParam("tenancyId") String tenancyId);
```

After:
```java
WorkItemView pickup(@PathParam UUID workItemId,
                    @ContextParam("tenancyId") String tenancyId);
```

`DefaultWorkItemQueueStateApi` injects `CurrentPrincipal` and uses `actorId()`.

#### 4. `BulkOperationRequest` — remove actorId field

Remove the `actorId` field from the record. `DefaultWorkItemBulkApi` injects `CurrentPrincipal` and uses `actorId()`.

Before: `record BulkOperationRequest(String operation, List<String> workItemIds, String actorId, String reason)`
After: `record BulkOperationRequest(String operation, List<String> workItemIds, String reason)`

Pre-release stage — no backward compatibility concern.

#### 5. Generated code — automatic

The annotation processor generates `@QueryParam` for unannotated String params. Removing those params from the SPI means the generated REST resource and GraphQL resolver drop the query params automatically. No code generator changes needed.

#### 6. `FaultRequest`, `ObsoleteRequest`, `CompensateRequest` — remove actor field

These request bodies contain an `actor` field that should also be derived from principal. Remove the field from each record. The implementation uses `currentPrincipal.actorId()` instead.

### What does NOT change

- `WorkItemOperations` interface — all actorId/claimantId params stay
- `WorkItemService` — all method signatures stay (they receive actorId from the layer above)
- `FederationProxyService` — decorator on `WorkItemOperations`, passes through
- Engine adapter (`HumanTaskScheduleHandler`, `WorkItemLifecycleAdapter`) — calls `WorkItemOperations` directly
- Qhorus bridge — calls `createInTenantContext()` only
- `SystemCurrentPrincipal` — `@WorkSystem` qualified, unchanged
- `*FromSystem` method variants — unchanged, used by timers and internal callers

### Test impact

Tests that call `WorkItemLifecycleApi` or `WorkItemQueueStateApi` directly need signature updates (remove actor/claimant args). Tests that call `WorkItemOperations` or `WorkItemService` directly are unaffected — they still pass actorId explicitly.

`MutableCurrentPrincipal` test doubles exist in `rest`, `queues`, `runtime`, and `mongodb` test scopes — these will be used to set the authenticated identity in tests.

## References

- `WorkItemLifecycleApi` — api/src/main/java/io/casehub/work/api/spi/WorkItemLifecycleApi.java
- `DefaultWorkItemLifecycleApi` — rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemLifecycleApi.java
- `WorkLifecycleResource` (generated) — rest/target/generated-sources/annotations/io/casehub/work/rest/spi/WorkLifecycleResource.java
- `WorkItemOperations` — api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java
- `WorkItemService` — runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java
- `WorkItemQueueStateApi` — api/src/main/java/io/casehub/work/api/spi/WorkItemQueueStateApi.java
- `DefaultWorkItemQueueStateApi` — queues/src/main/java/io/casehub/work/queues/service/DefaultWorkItemQueueStateApi.java
- `WorkItemBulkApi` — api/src/main/java/io/casehub/work/api/spi/WorkItemBulkApi.java
- `DefaultWorkItemBulkApi` — rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemBulkApi.java
- `BulkOperationRequest` — api/src/main/java/io/casehub/work/api/view/BulkOperationRequest.java
- `FederationProxyService` — federation/src/main/java/io/casehub/work/federation/FederationProxyService.java
- `InboundWorkItemBridge` — engine/engine-support-core/src/main/java/io/casehub/engine/inbound/InboundWorkItemBridge.java
- `CurrentPrincipal` — platform-api (io.casehub.platform.api.identity.CurrentPrincipal)
- `SystemCurrentPrincipal` — runtime/src/main/java/io/casehub/work/runtime/service/SystemCurrentPrincipal.java
- `MutableCurrentPrincipal` — rest/src/test, queues/src/test, runtime/src/test, persistence-mongodb/src/test
- playbook-direction.md — W1 finding, TN-6 tension
- #423 — parent epic
