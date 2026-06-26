# Design: Extract WorkItem SPI into casehub-work-api

**Issue:** casehubio/work#275
**Date:** 2026-06-26
**Status:** Approved

---

## Goal

Extract a complete external control plane for WorkItems into `casehub-work-api` (Tier 1, pure Java). Any module — casehub-desiredstate, a future adapter, any orchestrator — can create, look up, cancel, and complete WorkItems by depending on `casehub-work-api` alone, without pulling in JPA, Flyway, Hibernate, or the full runtime.

The engine work-adapter (`casehub-engine-work-adapter`) is the reference migration: its compile dependency on `casehub-work` (runtime) is fully replaced by `casehub-work-api`.

---

## Types Moving to `casehub-work-api`

Four types move from `io.casehub.work.runtime.model` to `io.casehub.work.api`:

| Type | Kind | Dependencies | Notes |
|------|------|-------------|-------|
| `WorkItemPriority` | enum | none | 4 values: LOW, MEDIUM, HIGH, URGENT |
| `WorkItemLabelRequest` | record | `LabelPersistence` (already in api) | |
| `WorkItemCreateRequest` | builder class | `WorkItemPriority`, `WorkItemLabelRequest`, `Outcome` (already in api) | ~220 lines. Add `tenancyId` (String) field to support multi-tenant callers outside a CDI tenant context |
| `WorkItemStatus` | enum | `java.util.Arrays`, `java.util.List` | 12 statuses, `isTerminal()`, `isActive()`, `TERMINAL_STATUSES` constant. Domain vocabulary — same category as `WorkEventType` already in api |

All four are pure Java with zero JPA/Quarkus dependencies.

`tenancyId` on `WorkItemCreateRequest`: the engine's template mode explicitly sets `tenancyId` on the entity because event-bus handlers run outside a CDI tenant context. Adding it to the builder makes multi-tenant creation available through the SPI. When null, the runtime resolves from the session tenant.

---

## New Types in `casehub-work-api`

### `WorkItemRef`

Record carrying the externally-visible state of a WorkItem. Returned by all SPI methods.

```java
public record WorkItemRef(
    UUID id,
    WorkItemStatus status,
    String callerRef,
    String assigneeId,
    String resolution,
    String candidateGroups,
    String outcome,
    String tenancyId
) {}
```

Every field is justified by a verified call site in the engine work-adapter:

| Field | Consumer | Purpose |
|-------|----------|---------|
| `id` | all | target for cancel/complete, correlation |
| `status` | `ActionGateCancelledHandler`, `HumanTaskRecoveryService` | `isTerminal()` check |
| `status` | `PlanItemCompletionApplier` | status switch for plan item transition |
| `callerRef` | `WorkItemLifecycleAdapter` | route gate vs plan-item events |
| `assigneeId` | `ActionGateCompletionApplier` | resolve approver/rejector identity |
| `resolution` | `PlanItemCompletionApplier` | JQ output mapping |
| `candidateGroups` | `WorkItemLifecycleAdapter` | escalation signal context |
| `outcome` | lifecycle event passthrough | completion classification |
| `tenancyId` | `WorkItemLifecycleAdapter` | tenant scoping |

### `WorkItemEvent`

API-level interface for CDI event observation. Eliminates entity downcasting.

```java
public interface WorkItemEvent {
    UUID workItemId();
    WorkItemStatus status();
    String callerRef();
    String assigneeId();
    String resolution();
    String candidateGroups();
    String outcome();
    String tenancyId();
}
```

The runtime `WorkItemLifecycleEvent` implements `WorkItemEvent`, reading each field from its embedded entity. The engine adapter changes from `@ObservesAsync WorkItemLifecycleEvent` (runtime) to `@ObservesAsync WorkItemEvent` (api). CDI delivers the same instances — the runtime type is still `WorkItemLifecycleEvent`, which is assignable to `WorkItemEvent`.

---

## SPI Interfaces

Two interfaces in `io.casehub.work.api.spi`, split by ISP.

### `WorkItemCreator`

Creation and lookup. For desiredstate, future adapters, any module that creates WorkItems programmatically.

```java
public interface WorkItemCreator {
    WorkItemRef create(WorkItemCreateRequest request);
    Optional<WorkItemRef> findByCallerRef(String callerRef);
    Optional<WorkItemRef> findActiveByCallerRef(String callerRef);
}
```

- `create` — when `request.templateId` is non-null, the runtime resolves the template and merges defaults with the request's non-null fields (request wins). Throws `IllegalArgumentException` if the template is not found. This unifies the inline and template creation paths.
- `findByCallerRef` — finds any WorkItem (any status) matching the callerRef. Indexed query.
- `findActiveByCallerRef` — finds only non-terminal WorkItems. For idempotent creation. Indexed query. Supersedes #274.

### `WorkItemLifecycle`

Lifecycle control. For orchestrators that need to cancel or complete WorkItems.

```java
public interface WorkItemLifecycle {
    void cancel(UUID id, String actorId, String reason);
    void complete(UUID id, String actorId, String resolution, String outcome);
}
```

Both return `void` — callers don't need post-mutation state. The lifecycle event fires internally and reaches observers through `WorkItemEvent`.

### Implementation

`WorkItemService implements WorkItemCreator, WorkItemLifecycle`. One class, two interfaces. No `@DefaultBean` — CDI enforces that a runtime is on the classpath (same pattern as `CaseHubRuntime` in engine-api).

### Consumer dependency matrix

| Consumer | Injects | Dep change |
|----------|---------|------------|
| casehub-desiredstate | `WorkItemCreator` | adds `casehub-work-api` (compile) |
| engine `HumanTaskScheduleHandler` | `WorkItemCreator` | `casehub-work` → `casehub-work-api` |
| engine `ActionGateCancelledHandler` | `WorkItemCreator` + `WorkItemLifecycle` | `casehub-work` → `casehub-work-api` |
| engine `WorkItemLifecycleAdapter` | observes `WorkItemEvent` | `casehub-work` → `casehub-work-api` |
| engine `HumanTaskRecoveryService` | `WorkItemCreator` | `casehub-work` → `casehub-work-api` |
| engine `PlanItemCompletionApplier` | takes `WorkItemRef` param | `casehub-work` → `casehub-work-api` |
| engine `ActionGateCompletionApplier` | takes `WorkItemRef` param | `casehub-work` → `casehub-work-api` |

---

## Template Creation Unification

Currently two divergent creation paths:

- **Inline** (`WorkItemService.create()`): full lifecycle — validates, assigns, audits, schedules timers, fires events.
- **Template** (`WorkItemTemplateService.instantiate()` + direct entity mutation + `persist()`): bypasses assignment, timer scheduling, audit, and events.

The template path in `HumanTaskScheduleHandler` reimplements creation outside the validated path. This is a design defect that becomes visible when extracting the SPI.

**Fix:** When `WorkItemCreateRequest.templateId` is non-null, `WorkItemService.create()` resolves the template internally:

1. Look up template by `templateId`
2. Start with template field values as defaults
3. For each non-null field on the request, override the template default
4. Proceed through the normal `create()` path (validation, assignment, audit, timers, events)

Merge rule: **request wins over template for any non-null field**. Same merge-patch semantics as `PATCH /workitem-templates/{id}`.

The engine's `HumanTaskScheduleHandler` collapses both modes into one call:

```java
WorkItemCreateRequest request = WorkItemCreateRequest.builder()
    .templateId(target.isTemplateMode() ? UUID.fromString(target.templateRef()) : null)
    .title(target.title())
    .candidateGroups(toCsv(resolvedGroups))
    .candidateUsers(toCsv(resolvedUsers))
    .createdBy("casehub-engine")
    .payload(payload)
    .expiresAt(effectiveDeadline)
    .claimDeadlineBusinessHours(target.claimDeadlineHours())
    .callerRef(callerRef)
    .scope(target.scope())
    .permittedOutcomes(toOutcomeList(target.outcomes()))
    .tenancyId(event.tenancyId())
    .build();
workItemCreator.create(request);
```

No `WorkItemTemplateService` injection. No entity manipulation. No `OutcomeCodecs`. No `persist()`.

---

## Engine Work-Adapter Migration

The compile dependency changes from `casehub-work` to `casehub-work-api`.

| File | Drops | Uses instead |
|------|-------|-------------|
| `HumanTaskScheduleHandler` | `WorkItemService`, `WorkItemTemplateService`, `WorkItemTemplate`, `WorkItem`, `OutcomeCodecs` | `WorkItemCreator` |
| `ActionGateCancelledHandler` | `WorkItemStore`, `WorkItemService`, `WorkItem` | `WorkItemCreator.findByCallerRef()` + `WorkItemLifecycle.cancel()` |
| `WorkItemLifecycleAdapter` | `WorkItemLifecycleEvent` (runtime), `WorkItem` | `@ObservesAsync WorkItemEvent` (api) |
| `ActionGateCompletionApplier` | `WorkItem` param | `WorkItemRef` param |
| `PlanItemCompletionApplier` | `WorkItem` param | `WorkItemRef` param |
| `HumanTaskRecoveryService` | `WorkItemService`, `WorkItem` | `WorkItemCreator.findByCallerRef()` |

POM:
```xml
<!-- compile: casehub-work → casehub-work-api -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-api</artifactId>
</dependency>

<!-- test scope: keeps casehub-work + persistence-memory for @QuarkusTest -->
```

---

## Indexed Queries

Add database index on `caller_ref`:

```sql
-- V32 migration
CREATE INDEX idx_work_item_caller_ref ON work_item (caller_ref);
```

`JpaWorkItemStore` overrides both methods with proper queries instead of linear scan:
- `findByCallerRef`: `WHERE caller_ref = :callerRef`
- `findActiveByCallerRef`: `WHERE caller_ref = :callerRef AND status NOT IN (:terminalStatuses)`

`InMemoryWorkItemStore` and `MongoWorkItemStore` implement both as collection filters with status filtering.

---

## Deferred Concerns (follow-up issues)

1. **Existing SPIs not in `api/spi/`** — current SPIs in `io.casehub.work.api` flat. New SPIs go in `io.casehub.work.api.spi` per protocol. Migrating existing SPIs is a separate task.

2. **`WorkLifecycleEvent` vs `WorkItemEvent` coexistence** — `WorkItemLifecycleEvent extends WorkLifecycleEvent implements WorkItemEvent`. The abstract class approach (`WorkLifecycleEvent`) and the interface approach (`WorkItemEvent`) coexist. Long-term alignment is a separate concern that affects the filter engine.

3. **`WorkItemTemplateService` consolidation** — template creation unifies into `create()`, but template CRUD and the REST instantiation path may also want to route through unified `create()`. Separate issue.

4. **Engine work-adapter test migration** — test classes reference runtime types directly. Tests keep `casehub-work` in test scope. `FailingWorkItemStore` test double implements `WorkItemStore` (runtime interface) — stays runtime-coupled in test scope.

---

## Lifecycle Coherence

`WorkItemStatus` moving to api makes it the platform-authoritative lifecycle enum for Layer 5 (WorkItem). `casehub-parent/docs/LIFECYCLE.md` must be updated to reflect the current 12-state enum before this ships (update provided separately to parent repo).

The six lifecycle layers (from the lifecycle alignment spec):

| Layer | State machine | States |
|-------|--------------|--------|
| 1 | `WorkflowStatus` (CNCF SDK) | 7 |
| 2 | `CaseStatus` (engine) | 7 |
| 3 | `PlanItemStatus` (engine blackboard) | 9 |
| 4 | HumanTask (no enum — expressed through PlanItem + WorkItem) | — |
| 5 | **`WorkItemStatus` (casehub-work-api)** | **12** |
| 6 | `CommitmentState` (qhorus) | 7 |

Moving `WorkItemStatus` to api is consistent with how other layers expose their lifecycle enums — `PlanItemStatus` is in `casehub-engine-common` (Tier 2), `CommitmentState` is in `casehub-qhorus-api` (Tier 1). WorkItem's lifecycle vocabulary belongs in its api tier.
