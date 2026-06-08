# Multi-Tenancy — tenancyId Column and Filtering for casehub-work

**Issue:** casehubio/work#256
**Date:** 2026-06-08
**Status:** Design approved

---

## Problem

casehub-work has zero tenant scoping. All WorkItems and WorkItemTemplates are shared across tenants. The platform foundation (`CurrentPrincipal.tenancyId()`, `TenancyConstants`) is complete; casehub-engine and casehub-eidos have implemented tenant scoping; casehub-work has not.

## Approach

Application-managed tenancy filtering. Every entity gets a `tenancy_id` column. Every store implementation injects `CurrentPrincipal` and applies `tenancyId` unconditionally to every query. Polling schedulers are replaced with per-item Quartz timers that carry tenant context as job data. A `TenantContextRunner` utility establishes tenant context for async and background paths.

---

## 1. Entity Changes

Add `tenancy_id VARCHAR(255) NOT NULL` to every JPA entity:

**Runtime:** WorkItem, WorkItemTemplate, AuditEntry, WorkItemNote, WorkItemLink, WorkItemSpawnGroup, WorkItemSchedule, WorkItemRelation, RoutingCursor, LabelDefinition, LabelVocabulary, FilterRule.

**Optional modules:** WorkItemQueueState, QueueView, WorkItemFilter, FilterChain, WorkItemQueueMembership (queues); WorkItemIssueLink (issue-tracker); WorkItemNotificationRule (notifications); WorkerSkillProfile, EscalationSummary (ai).

**Collection table** `work_item_label`: no separate `tenancy_id` — owned by WorkItem via `@CollectionTable`, inherits parent's tenancy filter through JOINs.

**Constraint changes:**
- `WorkItemTemplate` unique constraint: `UNIQUE(name)` → `UNIQUE(name, tenancy_id)`
- `RoutingCursor` PK: `(pool_hash)` → `(pool_hash, tenancy_id)`

---

## 2. Store Layer — Tenancy Filtering

Every store implementation injects `CurrentPrincipal` and applies `tenancyId` unconditionally to every query. Per protocols `tenancy-repository-pattern` and `no-conditional-tenancy-filtering`.

**Insert vs update contract:**
- `put()` on insert (entity's tenancyId is null): stamp from `CurrentPrincipal.tenancyId()`
- `put()` on update (entity's tenancyId already set): preserve existing value, never overwrite
- Defence-in-depth: on child WorkItem spawn, assert parent's tenancyId matches `CurrentPrincipal.tenancyId()`

**Existing stores extended with tenant filtering:**
- `WorkItemStore` / `JpaWorkItemStore` — tenant predicate on `get()`, `scan()`, `scanAll()`, `findByCallerRef()`, `scanRoots()`, `countByParentAndAssignee()`; stamp on `put()`
- `AuditEntryStore` / `JpaAuditEntryStore` — tenant predicate on all queries
- `WorkItemNoteStore` / `JpaWorkItemNoteStore` — tenant predicate on all queries
- `RoutingCursorStore` / `JpaRoutingCursorStore` — tenant predicate on all queries
- `IssueLinkStore` / `JpaIssueLinkStore` — tenant predicate on all queries

**New store SPIs:**

| Store | Replaces | Key methods |
|-------|----------|-------------|
| `WorkItemTemplateStore` | Static `WorkItemTemplate.find/list/delete` | `put`, `get`, `getByName`, `scanAll`, `delete` |
| `WorkItemRelationStore` | Static `WorkItemRelation.find*` | `put`, `findBySource`, `findByTarget`, `findBySourceAndType`, `delete` |
| `WorkItemSpawnGroupStore` | Static `WorkItemSpawnGroup.find*` | `put`, `get`, `findByParentId`, `findByParentAndKey`, `findMultiInstanceByParentId` |
| `WorkItemScheduleStore` | Static `WorkItemSchedule.find*` | `put`, `get`, `scanAll`, `delete` |

Each gets JPA and InMemory implementations. MongoDB where the module exists.

**InMemory stores (persistence-memory):** inject `CurrentPrincipal`, filter the backing collection on `tenancyId` in every read, stamp on every write.

**MongoDB stores:** add `tenancyId` to every MongoDB filter document.

**ReportService:** hand-built EntityManager JPQL — injects `CurrentPrincipal` directly and adds `AND w.tenancyId = :tenancyId` to every query. Acceptable because ReportService is a data access class (builds and executes queries).

**Static Panache elimination:** after this work, zero static Panache calls remain in production code outside store implementations. Every entity access flows through a tenant-aware store.

---

## 3. Scheduler Architecture — Polling to Per-Item Timers

Replace polling schedulers with per-item Quartz timers. Each timer carries `{workItemId, tenancyId}` (or `{scheduleId, tenancyId}`) as Quartz job data.

**ExpiryCleanupJob → ExpiryTimerJob:**
- `WorkItemService.create()` with `expiresAt` → schedule Quartz job for that instant
- `WorkItemService.extend()` → reschedule the job
- `WorkItemService.complete()/cancel()/reject()` → cancel the job
- Job fires → `TenantContextRunner.runInTenantContext(tenancyId, () -> expireItem(workItemId))`

**ClaimDeadlineJob → ClaimDeadlineTimerJob:**
- Same pattern: schedule when `claimDeadline` is set, cancel on claim/terminal.

**WorkItemScheduleService → ScheduleTimerJob:**
- `WorkItemSchedule` creation → schedule Quartz cron/interval trigger with `{scheduleId, tenancyId}`
- Job fires → `TenantContextRunner.runInTenantContext(tenancyId, () -> instantiateFromSchedule(scheduleId))`

**RoutingCursorCleanupJob:** remains as periodic GC. Cross-tenant bulk delete is correct for pure housekeeping — no business logic, no events, idempotent. The `RoutingCursor.delete()` static Panache call moves to `RoutingCursorStore` with an explicit system-level cleanup method (no tenant filter, documented as GC-only).

**Startup recovery (RAM store):** on application boot, a `@Startup` bean runs a one-time cross-tenant scan:
- WorkItems: `status NOT IN (COMPLETED, REJECTED, CANCELLED, EXPIRED, ESCALATED) AND (expires_at IS NOT NULL OR claim_deadline IS NOT NULL)` — re-schedule timers for each, using the entity's `tenancyId` as job data.
- WorkItemSchedules: `active = true` — re-schedule Quartz triggers for each, using the entity's `tenancyId` as job data.

This is the one bounded cross-tenant operation — runs once at startup, not recurring. Uses a direct JPQL query (not through the tenant-scoped store) explicitly documented as system-level bootstrap.

---

## 4. TenantContextRunner

`@ApplicationScoped` bean in `runtime/service/`. Establishes a CDI request context with a synthetic `CurrentPrincipal` carrying a specific tenancyId.

```
tenantContextRunner.runInTenantContext(tenancyId, () -> { ... });
```

Internally:
1. Activates a CDI `RequestContext` via `Arc.container().requestContext()`
2. Registers a `CurrentPrincipal` instance with the given tenancyId and `actorId = "system"`
3. Runs the work
4. Tears down the context

**Used by:**
- Quartz expiry/claim-deadline/schedule job handlers
- `MultiInstanceCoordinator.onChildTerminal()` (`@ObservesAsync`) — reads tenancyId from event's WorkItem
- Startup recovery scan — iterates items, activates each item's tenant context
- Any future `@ObservesAsync` handler that needs tenant context

---

## 5. CDI Events

**WorkItemLifecycleEvent** — add `tenancyId` field. Factory methods read from `workItem.tenancyId`. `@JsonIgnore` on the field for client-facing SSE. Included in `fromWire()` for server-to-server distributed relay.

**WorkItemGroupLifecycleEvent** — add `tenancyId` field. Set from parent WorkItem's tenancyId at fire time.

**SlaBreachEvent** — add `tenancyId` field. With the timer architecture, the Quartz handler establishes tenant context before calling the expiry service, so the event is fired in the correct tenant context.

**WorkLifecycleEvent** (abstract superclass in `casehub-work-api`): no tenancyId accessor. The API module is consumed by `casehub-work-core` which has no CurrentPrincipal dependency. The concrete `WorkItemLifecycleEvent` in runtime carries it.

---

## 6. REST API Surface

No changes. `tenancyId` never appears in request DTOs, response DTOs, JSON Schemas, or OpenAPI specs. Per protocol `PP-20260607-69eba2` (`tenancyid-server-side-only`).

Request/response types already exist separately from entities:
- Request POJOs (`WorkItemCreateRequest` etc.) — no tenancyId field
- Response POJOs (`WorkItemResponse`, `WorkItemWithAuditResponse`) — mapper does not map tenancyId out
- Entity (`WorkItem`) — has tenancyId, internal only

---

## 7. Cross-Module SPI Boundary

**SpawnPort.spawn():** called by casehub-engine's work-adapter within the engine's tenant context. Flows through WorkItemSpawnService → WorkItemStore.put() → store stamps tenancyId from CurrentPrincipal. No SPI signature change needed.

**CaseSignalSink:** called by casehub-work on SLA escalation. With the timer architecture, the Quartz handler has already established tenant context. No signature change needed.

**Webhook resources (Jira, GitHub):** inbound with no CaseHub authentication. Webhook configuration must carry a tenancyId. Resource looks up config, establishes tenant context via TenantContextRunner, then processes. Out of scope for this issue — tracked separately.

---

## 8. Flyway Migrations

Each module's migration stays within its allocated V-number range.

| Migration | Module | Tables |
|-----------|--------|--------|
| V35 | runtime | work_item, work_item_template, audit_entry, work_item_note, work_item_link, work_item_spawn_group, work_item_schedule, work_item_relation, routing_cursor, label_definition, label_vocabulary, filter_rule |
| V2003 | queues | work_item_queue_state, queue_view, work_item_filter, filter_chain, work_item_queue_membership |
| V3001 | notifications | work_item_notification_rule |
| V4002 | ai | worker_skill_profile, escalation_summary |
| V5002 | issue-tracker | work_item_issue_link |

Each migration: `ALTER TABLE <t> ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce'`. Plus constraint and PK changes for WorkItemTemplate and RoutingCursor. Plus index on `tenancy_id` per table.

---

## 9. Testing Strategy

### Level 1: Store isolation tests

Every store implementation (JPA, InMemory, MongoDB) gets multi-tenant tests:
- Create data as tenant A → switch to tenant B → assert B cannot see/modify/delete A's data
- Assert `put()` on update preserves existing tenancyId

### Level 2: Timer and async context propagation tests

- Create WorkItem with `expiresAt` in tenant A → Quartz job fires → expiry in tenant A's context → tenant B untouched
- Same for claim deadline timers
- Multi-instance group: `@ObservesAsync` MultiInstanceCoordinator processes in correct tenant context
- WorkItemSchedule timer: instantiated WorkItem belongs to correct tenant

### Level 3: End-to-end integration tests

Two-tenant REST scenarios:
- Isolation: POST as tenant A, GET as tenant B → 404
- Lifecycle isolation: create/claim/complete as tenant A → tenant B audit empty
- Template isolation: same name in both tenants → both succeed, each sees only their own
- Spawn isolation: children visible only to parent's tenant
- Expiry isolation: both tenants' items expire independently with correct tenancyId on events
- Cross-tenant prevention: access tenant A's WorkItem ID as tenant B → 404 (not 403)

### Test infrastructure

`FixedCurrentPrincipal` from `casehub-platform-testing` with `setTenancyId()` for switching tenant context mid-test. Integration tests use different auth headers per tenant.

---

## 10. Protocols Referenced

| Protocol | Rule |
|----------|------|
| `tenancy-repository-pattern` | Filtering in data access layer, never at call sites |
| `no-conditional-tenancy-filtering` | Always on, never gated on deployment mode |
| `tenancyid-server-side-only` | Never in client-facing APIs (PP-20260607-69eba2) |
| `subcase-tenancyid-inherits-parent` | Children inherit parent's tenancyId |
| `spi-event-tenancyid-component-order` | tenancyId position in CDI event records |

---

## 11. Out of Scope (Separate Issues)

- PostgreSQL RLS hardening — future defence-in-depth layer on top of application-managed filtering
- Promotion of `TenantContextRunner` to `casehub-platform` — starts in casehub-work, promoted when other repos need it
- Webhook tenant mapping configuration for issue-tracker module (Jira/GitHub inbound)
- Update `tenancy-repository-pattern` protocol to acknowledge background/async paths that use TenantContextRunner instead of CurrentPrincipal directly
