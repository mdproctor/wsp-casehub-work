# Implement InboundWorkItemScheduler and Relocate WorkActorStateContributor

**Issues:** casehubio/work#397, casehubio/work#398
**Date:** 2026-09-06
**Status:** Draft

## Problem

Engine#974 (circular dependency fix) extracted two SPIs from the engine to break
engine→work compile dependencies:

1. **`InboundWorkItemScheduler`** (engine-common) — the `InboundWorkItemBridge` in
   engine-inbound previously injected `WorkItemService` + `TenantContextRunner`
   directly. Now it injects this SPI. A `NoOpInboundWorkItemScheduler` (`@DefaultBean`)
   logs a warning when the work module is absent. The work repo must provide the
   real implementation.

2. **`ActorStateContributor`** (platform-api) — `WorkActorStateContributor` was
   deleted from `casehub-engine-actor-state`. It queries `WorkItemStore` for active
   items assigned to an actor and contributes them to the actor state view. This is
   a work concern that belongs in the work repo.

Both implementations belong in `casehub-work-engine-adapter` (package
`io.casehub.work.engine`), which already bridges engine and work concepts.

## Approach

### InboundWorkItemScheduler Implementation

**Class:** `InboundWorkItemSchedulerImpl` in `io.casehub.work.engine`

**CDI wiring:** `@ApplicationScoped` — overrides the engine's
`NoOpInboundWorkItemScheduler` (`@DefaultBean`) when engine-adapter is on the
classpath. No `@Alternative` or `@Priority` needed — `@DefaultBean` yields to
any non-default bean automatically.

**Dependencies (all injectable via existing POM):**
- `WorkItemCreator` (work-api) — creates the WorkItem
- `TenantContextExecutor` (work-api) — establishes tenant context

**Flow:**

```
InboundWorkItemBridge.onMessage()                     [engine-inbound]
  └─ scheduler.schedule(stampedRequest)
       └─ InboundWorkItemSchedulerImpl.schedule()     [work-engine-adapter]
            ├─ Convert InboundWorkItemRequest → WorkItemCreateRequest
            ├─ tenantContextExecutor.runInTenantContext(request.tenancyId(), () ->
            │    workItemCreator.create(createRequest))
            └─ (void — fire-and-forget from bridge's perspective)
```

**Field mapping (InboundWorkItemRequest → WorkItemCreateRequest):**

| InboundWorkItemRequest | WorkItemCreateRequest | Notes |
|----------------------|----------------------|-------|
| `title()` | `.title()` | direct |
| `description()` | `.description()` | direct |
| `candidateGroups()` | `.candidateGroups()` | direct |
| `candidateUsers()` | `.candidateUsers()` | direct |
| `callerRef()` | `.callerRef()` | direct |
| `scope()` | `.scope()` | direct |
| `payload()` | `.payload()` | direct |
| `tenancyId()` | `.tenancyId()` | direct |
| `createdBy()` | `.createdBy()` | direct |
| `priority()` (String) | `.priority()` (WorkItemPriority) | `WorkItemPriority.valueOf()`, null-safe |
| `types()` | `.types()` | direct |
| `expiresAt()` | `.expiresAt()` | direct |

Fields not on `InboundWorkItemRequest` (formKey, assigneeId, requiredCapabilities,
labels, confidenceScore, templateId, etc.) default to null on `WorkItemCreateRequest`
— the service applies its own defaults (e.g., priority → MEDIUM, expiresAt → config
default).

**Priority mapping:** `request.priority()` is `@Nullable String`.
- null → null on `WorkItemCreateRequest` → defaults to MEDIUM in `WorkItemService.create()`
- non-null → `WorkItemPriority.valueOf(request.priority())` — `IllegalArgumentException`
  propagates on invalid values (fail-fast, consistent with request validation)

**Tenant context:** The `InboundWorkItemBridge` fires from a qhorus
`afterCompletion(STATUS_COMMITTED)` callback — outside the originating request scope.
The scheduler wraps `WorkItemCreator.create()` in
`tenantContextExecutor.runInTenantContext(request.tenancyId(), ...)` to establish
`CurrentPrincipal` for the store's Hibernate tenant filters.

**Deferred improvement:** #399 tracks making `WorkItemService` self-sufficient for
tenant context, eliminating the need for caller-side wrapping.

### WorkActorStateContributor

**Class:** `WorkActorStateContributor` in `io.casehub.work.engine`

**CDI wiring:** `@ApplicationScoped` — discovered via Jandex when engine-adapter
is on the classpath. The `ActorStateAggregator` (in engine-actor-state) injects
`Instance<ActorStateContributor>` and iterates all contributors.

**Dependencies (all injectable via existing POM):**
- `WorkItemStore` (work-api) — queries work items by assignee

**Flow:**

```
ActorStateAggregator.forActor(actorId)                [engine-actor-state]
  └─ contributor.contribute(actorId, accumulator)
       └─ WorkActorStateContributor.contribute()      [work-engine-adapter]
            ├─ WorkItemStore.scan(query) — ASSIGNED, IN_PROGRESS, SUSPENDED
            ├─ For each WorkItem:
            │    ├─ CallerRef.parse(wi.callerRef()) → caseId or null
            │    └─ accumulator.workItem(id, title, status, category, caseId)
            └─ (atomic: collect all items, then contribute)
```

**Query:** `WorkItemQuery.builder().assigneeId(actorId).statusIn(List.of(ASSIGNED, IN_PROGRESS, SUSPENDED)).build()`

- ASSIGNED — actor has claimed the item
- IN_PROGRESS — actor is actively working
- SUSPENDED — actor is still obligated (paused, not released)
- PENDING excluded — actor is a candidate but hasn't claimed

**CaseId extraction:** `CallerRef.parse(wi.callerRef())` returns:
- `PlanItemRef` (with `caseId()`) for `case:{uuid}/pi:{planItemId}` format
- `GateRef` (with `caseId()`) for `case:{uuid}/gate:{gateId}` format
- `null` for non-engine callerRefs or null input

This replaces the deleted `WorkItemCallerRef.parseCaseId()` that was in work-api.
`CallerRef` is already in the engine-adapter — same module, no new dependency.

**`sourceName()`:** returns `"work"` — matches the original contributor in
engine-actor-state and the actor-state-view design spec.

**Atomic contribution contract:** The contributor collects all items from
`WorkItemStore.scan()` (returns eager `List<WorkItem>`) before calling any
accumulator method. This satisfies the `ActorStateContributor` contract: no
partial data on exception.

## Dependencies

No POM changes needed. All required types are available through existing
engine-adapter dependencies:

| Type | Source | Available via |
|------|--------|--------------|
| `InboundWorkItemScheduler` | engine-common | `casehub-engine-planning` (transitive) |
| `InboundWorkItemRequest` | engine-common | `casehub-engine-planning` (transitive) |
| `WorkItemCreator` | work-api | `casehub-work-api` (direct) |
| `TenantContextExecutor` | work-api | `casehub-work-api` (direct) |
| `WorkItemStore` | work-api | `casehub-work-api` (direct) |
| `WorkItemQuery` | work-api | `casehub-work-api` (direct) |
| `WorkItemPriority` | work-api | `casehub-work-api` (direct) |
| `ActorStateContributor` | platform-api | `casehub-platform` (direct) |
| `ActorStateAccumulator` | platform-api | `casehub-platform` (direct) |
| `CallerRef` | engine-adapter | same module |

## Schema Impact

No database migrations. No new entities. No column changes. Both implementations
operate through existing service/store APIs and existing entity structures.

Schema coherence note: `WorkItem` entity fields (title, description, status,
candidateGroups, candidateUsers, callerRef, scope, payload, tenancyId, createdBy,
priority, types, expiresAt, category, assigneeId) are all established and stable.
The `InboundWorkItemRequest` field set is a strict subset — no schema drift risk.

## Testing

### InboundWorkItemSchedulerImplTest (plain JUnit + Mockito)

Mock `WorkItemCreator` and `TenantContextExecutor`. Verify:

- **Happy path** — schedule() converts fields correctly, calls create() inside
  tenant context with request.tenancyId()
- **Null priority** — passes null to WorkItemCreateRequest, no exception
- **Valid priority** — "HIGH" → WorkItemPriority.HIGH on the create request
- **Invalid priority** — "BOGUS" → IllegalArgumentException propagates
- **Null optional fields** — description, candidateGroups, candidateUsers, callerRef,
  scope, payload, createdBy, types, expiresAt all null → create() called with nulls
- **Tenant context wrapping** — verify tenantContextExecutor.runInTenantContext()
  called with request.tenancyId() and the create() happens inside the Runnable

### WorkActorStateContributorTest (plain JUnit + Mockito)

Mock `WorkItemStore`. Verify:

- **Happy path** — actor has 2 active items → accumulator.workItem() called twice
  with correct fields
- **CaseId from PlanItemRef** — callerRef `case:{uuid}/pi:{planItemId}` → caseId
  extracted correctly
- **CaseId from GateRef** — callerRef `case:{uuid}/gate:{gateId}` → caseId extracted
- **Non-engine callerRef** — callerRef `qhorus:{channelId}/...` → caseId is null
- **Null callerRef** — → caseId is null, no exception
- **No active items** — scan returns empty → no accumulator calls
- **Status filter** — verify query uses ASSIGNED, IN_PROGRESS, SUSPENDED only
- **sourceName** — returns "work"
- **Null title/category** — passes through as null to accumulator (externally-created items)

## Documentation Updates

### CLAUDE.md

Update the `## engine-adapter Module` section to document:
- `InboundWorkItemSchedulerImpl` — implements `InboundWorkItemScheduler` (engine-common),
  converts `InboundWorkItemRequest` to `WorkItemCreateRequest`, wraps in tenant context
- `WorkActorStateContributor` — implements `ActorStateContributor` (platform-api),
  queries active work items by assignee, contributes to actor state view

### ARC42STORIES.MD

Add entries in §9.4 Layer Entries for both implementations if engine-adapter
doesn't already have a layer entry. Reference engine#974 as the driver.

## Files

### New (2 source + 2 test)

| File | Module | Description |
|------|--------|-------------|
| `engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java` | engine-adapter | SPI implementation |
| `engine-adapter/src/main/java/io/casehub/work/engine/WorkActorStateContributor.java` | engine-adapter | SPI implementation |
| `engine-adapter/src/test/java/io/casehub/work/engine/InboundWorkItemSchedulerImplTest.java` | engine-adapter | Unit test |
| `engine-adapter/src/test/java/io/casehub/work/engine/WorkActorStateContributorTest.java` | engine-adapter | Unit test |

### Modified (0)

No existing files modified. Both implementations are purely additive.

## References

- engine-common `InboundWorkItemScheduler.java` — SPI interface (engine#974)
- engine-common `InboundWorkItemRequest.java` — request record (engine#974)
- engine-runtime `NoOpInboundWorkItemScheduler.java` — `@DefaultBean` fallback
- engine-inbound `InboundWorkItemBridge.java` — caller context (afterCompletion callback)
- platform-api `ActorStateContributor` / `ActorStateAccumulator` — contributor SPI
- engine-actor-state `LedgerActorStateContributor.java` — pattern reference
- engine-actor-state `ActorStateAccumulatorImpl.java` — accumulator contract
- `docs/specs/2026-06-01-actor-state-view-design.md` — original WorkActorStateContributor spec
- `docs/specs/2026-07-08-relocate-work-adapter-design.md` — adapter relocation spec (#290)
- Protocol `async-event-tenant-context-propagation` (PP-20260609-fb6563)
- Protocol `store-tenancy-stamping-on-insert` (PP-20260609-bdac7e)
- #399 — self-sufficient tenant context (deferred)
