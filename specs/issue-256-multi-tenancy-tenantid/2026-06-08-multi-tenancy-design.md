# Multi-Tenancy — tenancyId Column and Filtering for casehub-work

**Issue:** casehubio/work#256
**Date:** 2026-06-08
**Status:** Design approved — revision 2

---

## Problem

casehub-work has zero tenant scoping. All WorkItems and WorkItemTemplates are shared across tenants. The platform foundation (`CurrentPrincipal.tenancyId()`, `TenancyConstants`) is complete; casehub-engine and casehub-eidos have implemented tenant scoping; casehub-work has not.

## Approach

Application-managed tenancy filtering. Every entity gets a `tenancy_id` column. Every store implementation injects `CurrentPrincipal` and applies `tenancyId` unconditionally to every query. Polling schedulers are replaced with per-item Quartz timers that carry tenant context as job data. A `TenantContextRunner` utility establishes tenant context for async and background paths. SSE event streams are tenant-scoped at the stream level, not just at serialisation.

---

## 1. Entity Changes

Add `tenancy_id VARCHAR(255) NOT NULL` to every JPA entity:

**Runtime:** WorkItem, WorkItemTemplate, AuditEntry, WorkItemNote, WorkItemLink, WorkItemSpawnGroup, WorkItemSchedule, WorkItemRelation, RoutingCursor, LabelDefinition, LabelVocabulary, FilterRule.

**Optional modules:** WorkItemQueueState, QueueView, WorkItemFilter, FilterChain, WorkItemQueueMembership (queues); WorkItemIssueLink (issue-tracker); WorkItemNotificationRule (notifications); WorkerSkillProfile, EscalationSummary (ai).

**casehub-work-ledger:** `WorkItemLedgerEntry` extends `LedgerEntry` (casehub-ledger) via JOINED inheritance. The base `LedgerEntry` has no `tenancyId` column — casehub-ledger manages tenancy at the repository query level, not on the entity. `WorkItemLedgerEntryRepository` and its JPA implementation must add tenant filtering to all query methods (`findByWorkItemId`, `findLatestByWorkItemId`, `findEarliestByWorkItemId`). The `work_item_ledger_entry` join table gets a `tenancy_id` column (V2002 migration in the ledger module's Flyway range). The base `ledger_entry` table is owned by casehub-ledger — casehub-work must not add columns to it.

**Collection table** `work_item_label`: no separate `tenancy_id` — owned by WorkItem via `@CollectionTable`, inherits parent's tenancy filter through JOINs.

**Constraint changes:**
- `WorkItemTemplate` unique constraint: `UNIQUE(name)` → `UNIQUE(name, tenancy_id)`
- `RoutingCursor` PK: `(pool_hash)` → `(pool_hash, tenancy_id)` — two tenants with identical candidate pools get independent cursor state

---

## 2. Store Layer — Tenancy Filtering

Every store implementation injects `CurrentPrincipal` and applies `tenancyId` unconditionally to every query. Per protocols `tenancy-repository-pattern` and `no-conditional-tenancy-filtering`.

### isCrossTenantAdmin() — principled position

`CurrentPrincipal.isCrossTenantAdmin()` is honoured. When true, the store omits the tenant filter and returns results across all tenants. This is not a conditional tenancy gate (which `no-conditional-tenancy-filtering` prohibits) — it is an authorization check. The protocol prohibits gating tenancy filtering on a *deployment mode* or *feature flag*. `isCrossTenantAdmin()` is an *identity claim* on the authenticated principal — the difference between "this deployment doesn't do tenancy" (prohibited) and "this principal is authorized to see all tenants" (permitted).

**Who gets `isCrossTenantAdmin() = true`:**
- `TenantContextRunner` when establishing system context for Quartz jobs and startup recovery (actorId = "system", crossTenantAdmin = true for the recovery scan specifically)
- `RoutingCursorCleanupJob` for GC
- Future admin/diagnostic endpoints (explicitly gated by `@RolesAllowed`)

**Who does NOT get it:** Normal REST requests. The `MockCurrentPrincipal` defaults to `crossTenantAdmin = false`.

### Insert vs update contract

- `put()` on insert (entity's tenancyId is null): stamp from `CurrentPrincipal.tenancyId()`
- `put()` on update (entity's tenancyId already set): preserve existing value, never overwrite
- Defence-in-depth: on child WorkItem spawn, assert parent's tenancyId matches `CurrentPrincipal.tenancyId()`

### Existing stores extended with tenant filtering

- `WorkItemStore` / `JpaWorkItemStore` — tenant predicate on `get()`, `scan()`, `scanAll()`, `findByCallerRef()`, `scanRoots()`, `countByParentAndAssignee()`; stamp on `put()`
- `AuditEntryStore` / `JpaAuditEntryStore` — tenant predicate on all queries
- `WorkItemNoteStore` / `JpaWorkItemNoteStore` — tenant predicate on all queries
- `RoutingCursorStore` / `JpaRoutingCursorStore` — tenant predicate on all queries; plus a `cleanupStale(Instant cutoff)` method with no tenant filter (GC-only, documented)
- `IssueLinkStore` / `JpaIssueLinkStore` — tenant predicate on all queries
- `WorkItemLedgerEntryRepository` / `JpaWorkItemLedgerEntryRepository` — tenant predicate on `findByWorkItemId()`, `findLatestByWorkItemId()`, `findEarliestByWorkItemId()`

### New store SPIs

| Store | Replaces | Key methods |
|-------|----------|-------------|
| `WorkItemTemplateStore` | Static `WorkItemTemplate.find/list/delete` | `put`, `get`, `getByName`, `scanAll`, `delete` |
| `WorkItemRelationStore` | Static `WorkItemRelation.find*` | `put`, `findBySource`, `findByTarget`, `findBySourceAndType`, `delete` |
| `WorkItemSpawnGroupStore` | Static `WorkItemSpawnGroup.find*` | `put`, `get`, `findByParentId`, `findByParentAndKey`, `findMultiInstanceByParentId` |
| `WorkItemScheduleStore` | Static `WorkItemSchedule.find*` | `put`, `get`, `scanAll`, `delete` |
| `LabelDefinitionStore` | Static `LabelDefinition.findByVocabularyId/findByPath` | `put`, `findByVocabularyId`, `findByPath`, `delete` |
| `LabelVocabularyStore` | Static `LabelVocabulary` calls | `put`, `get`, `scanAll`, `delete` |
| `FilterRuleStore` | Static `FilterRule.allEnabled` | `put`, `get`, `allEnabled`, `delete` |

Each gets JPA and InMemory implementations. MongoDB where the module exists.

**Optional module stores** (queues, notifications, ai) — entities in these modules that use static Panache calls (`QueueView.findById()`, `QueueView.listAll()`, `WorkItemNotificationRule.findEnabledForEventType()`, `WorkerSkillProfile.listAll()`, etc.) must also route through tenant-aware stores or inject `CurrentPrincipal` directly where the class is already a data access class.

**InMemory stores (persistence-memory):** inject `CurrentPrincipal`, filter the backing collection on `tenancyId` in every read, stamp on every write.

**MongoDB stores:** add `tenancyId` to every MongoDB filter document. Existing MongoDB documents have no `tenancyId` field. A `@Startup` migration bean backfills all documents without `tenancyId` to `TenancyConstants.DEFAULT_TENANT_ID`. This runs once, idempotently — documents already bearing a `tenancyId` are skipped.

**ReportService:** hand-built EntityManager JPQL — injects `CurrentPrincipal` directly and adds `AND w.tenancyId = :tenancyId` to every query. Acceptable because ReportService is a data access class (builds and executes queries).

**Static Panache elimination:** after this work, zero static Panache calls remain in production code outside store implementations. Every entity access flows through a tenant-aware store.

---

## 3. SSE Event Broadcasting — Tenant-Scoped Streams

### Problem

`LocalWorkItemEventBroadcaster` broadcasts to a single `BroadcastProcessor` shared across all subscribers. The `stream()` method filters by `workItemId` and `type` only — there is no tenant dimension. A tenant B subscriber sees every tenant A event on the hot stream. The same problem exists in `LocalWorkItemQueueEventBroadcaster` and both Postgres broadcaster variants.

### Design

**WorkItemEventBroadcaster SPI:** add `tenancyId` as a required filter parameter:
```
Multi<WorkItemLifecycleEvent> stream(UUID workItemId, String type, String tenancyId);
```

All implementations (`LocalWorkItemEventBroadcaster`, `PostgresWorkItemEventBroadcaster`) filter the stream by `tenancyId`. The SSE REST endpoint reads `tenancyId` from `CurrentPrincipal.tenancyId()` and passes it to `stream()`. A null `tenancyId` parameter is never permitted — it is always supplied by the endpoint.

**WorkItemQueueEventBroadcaster SPI:** add `tenancyId` as a required filter parameter:
```
Multi<WorkItemQueueEvent> stream(UUID queueViewId, String tenancyId);
```

Same pattern: all implementations filter; the REST endpoint supplies the value.

**WorkItemQueueEvent record:** add `tenancyId` field. Set from the WorkItem's `tenancyId` at event creation time in `FilterEvaluationObserver`.

### Wire DTOs for Postgres broadcasters

**WorkItemEventPayload** (postgres-broadcaster): add `tenancyId` field. `from()` reads it from `event.tenancyId()`. `toEvent()` passes it to `WorkItemLifecycleEvent.fromWire()`. Without this, the receiving node cannot scope the reconstructed event or apply tenant filtering on the stream.

**WorkItemQueueEvent** is used directly as the wire DTO in `PostgresWorkItemQueueEventBroadcaster` (serialised via ObjectMapper). Adding `tenancyId` to the record covers both the CDI event and the wire format.

---

## 4. Scheduler Architecture — Polling to Per-Item Timers

Replace polling schedulers with per-item Quartz timers. Each timer carries `{workItemId, tenancyId}` (or `{scheduleId, tenancyId}`) as Quartz job data.

### Quartz job store

**JDBC store** (`quartz.store-type=jdbc-cmt`). Timers survive restarts without a recovery scan. Quartz handles cluster dedup natively via database row locking — two nodes cannot fire the same trigger. This eliminates the startup recovery scan for the happy path. The RAM store alternative would require a recovery scan on every restart and has no cluster dedup.

**Scale characteristic:** per-item timers mean N Quartz jobs for N active WorkItems with deadlines. A system with 50K active items has 50K+ triggers. This is within Quartz JDBC store's design envelope — triggers are indexed rows, not in-heap objects.

**Startup recovery:** with JDBC store, timers that fired while the application was down are handled by Quartz's misfire policy. Set `org.quartz.jobStore.misfireThreshold` to a reasonable value (e.g. 60s). Misfired triggers execute immediately on startup. A separate recovery scan is needed only if JDBC store state is lost (disaster recovery) — in that case, a `@Startup` bean runs a one-time cross-tenant scan:
- WorkItems: `status NOT IN (COMPLETED, REJECTED, CANCELLED, EXPIRED, ESCALATED) AND (expires_at IS NOT NULL OR claim_deadline IS NOT NULL)` — re-schedule timers for each.
- WorkItemSchedules: `active = true` — re-schedule Quartz triggers for each.

This recovery scan uses `isCrossTenantAdmin() = true` context (see §2).

### ExpiryCleanupJob → ExpiryTimerJob

- `WorkItemService.create()` with `expiresAt` → schedule Quartz job for that instant
- `WorkItemService.extend()` → reschedule the job
- Terminal transitions → cancel the job
- Job fires → `TenantContextRunner.runInTenantContext(tenancyId, () -> expireItem(workItemId))`

### ClaimDeadlineJob → ClaimDeadlineTimerJob

- Same pattern: schedule when `claimDeadline` is set, cancel on claim/terminal.

### WorkItemScheduleService → ScheduleTimerJob

- `WorkItemSchedule` creation → schedule Quartz cron/interval trigger with `{scheduleId, tenancyId}`
- Job fires → `TenantContextRunner.runInTenantContext(tenancyId, () -> instantiateFromSchedule(scheduleId))`

### State transition × timer action matrix

Every state transition that touches `expiresAt` or `claimDeadline` needs a corresponding timer management call:

| Transition | Method | expiresAt timer | claimDeadline timer |
|-----------|--------|-----------------|---------------------|
| Create | `WorkItemService.create()` | SCHEDULE (from request or config default) | SCHEDULE (from request or config default, if > 0) |
| Claim | `WorkItemService.claim()` | — (unchanged) | CANCEL (item is claimed) |
| Start | `WorkItemService.start()` | — | — |
| Complete | `WorkItemService.complete()` | CANCEL | CANCEL |
| Reject | `WorkItemService.reject()` | CANCEL | CANCEL |
| Cancel | `WorkItemService.cancel()` | CANCEL | CANCEL |
| Extend | `WorkItemService.extend()` | RESCHEDULE (new expiresAt) | — |
| Delegate | `WorkItemService.delegate()` | — | CANCEL (delegatee has new acceptance window, not claim SLA) |
| Accept delegation | `WorkItemService.acceptDelegation()` | — | — |
| Decline delegation | `WorkItemService.declineDelegation()` | — | SCHEDULE (new claimDeadline from `ClaimSlaPolicy`) |
| Release | `WorkItemService.release()` | — | SCHEDULE (new claimDeadline from `ClaimSlaPolicy`) |
| Suspend | `WorkItemService.suspend()` | PAUSE (cancel timer, record remaining duration) | PAUSE |
| Resume | `WorkItemService.resume()` | RESUME (schedule with remaining duration from now) | RESUME |
| Expire (SLA Fail) | `ExpiryLifecycleService.executeFail()` | — (already fired) | CANCEL |
| Expire (SLA EscalateTo) | `ExpiryLifecycleService.executeEscalateTo()` | RESCHEDULE (new expiresAt window) | SCHEDULE (new claimDeadline) |
| Expire (SLA Extend) | `ExpiryLifecycleService.executeExtend()` | RESCHEDULE (now + extend.by()) | or RESCHEDULE (now + extend.by()) |
| Expire (Exhausted) | `ExpiryLifecycleService.executeExhausted()` | — (already fired) | CANCEL |
| Claim expired | `ExpiryLifecycleService.checkClaimDeadlines()` | — | — (already fired; policy may SCHEDULE a new one via EscalateTo) |
| Spawn child | `WorkItemSpawnService.spawn()` | SCHEDULE (if child has expiresAt) | SCHEDULE (if child has claimDeadline) |

**PAUSE/RESUME for suspend:** On suspend, cancel the timer and store the remaining duration on the entity (new field: `remainingExpirySeconds`, `remainingClaimSeconds` — transient or persisted, TBD during implementation). On resume, schedule a new timer for `now + remaining`.

### RoutingCursorCleanupJob

Remains as periodic GC. Cross-tenant bulk delete is correct for pure housekeeping — no business logic, no events, idempotent. Uses `RoutingCursorStore.cleanupStale(cutoff)` which applies no tenant filter (documented as GC-only).

---

## 5. TenantContextRunner

`@ApplicationScoped` bean in `runtime/service/`. Establishes a CDI request context with a synthetic `CurrentPrincipal` carrying a specific tenancyId.

```
tenantContextRunner.runInTenantContext(tenancyId, () -> { ... });
```

Internally:
1. Activates a CDI `RequestContext` via `Arc.container().requestContext()`
2. Registers a `CurrentPrincipal` instance with the given tenancyId, `actorId = "system"`, and `isCrossTenantAdmin() = false`
3. Runs the work
4. Tears down the context

A separate `runInCrossTenantContext()` variant sets `isCrossTenantAdmin() = true` — used only by the startup recovery scan.

**Used by:**
- Quartz expiry/claim-deadline/schedule job handlers — `runInTenantContext()`
- `MultiInstanceCoordinator.onChildTerminal()` (`@ObservesAsync`) — reads tenancyId from event's WorkItem
- Startup recovery scan — `runInCrossTenantContext()` for the scan, then `runInTenantContext()` per item if processing is needed
- `NotificationDispatcher` — currently dispatches to virtual threads via `CompletableFuture.runAsync()` which loses context; must wrap in `runInTenantContext()` using tenancyId from the event
- Any future `@ObservesAsync` handler that needs tenant context

---

## 6. CDI Events

**WorkItemLifecycleEvent** — add `tenancyId` field. Factory methods read from `workItem.tenancyId`. `@JsonIgnore` on the field for client-facing SSE. Included in `fromWire()` for server-to-server distributed relay.

**WorkItemGroupLifecycleEvent** — add `tenancyId` field. Set from parent WorkItem's tenancyId at fire time.

**SlaBreachEvent** — add `tenancyId` field. With the timer architecture, the Quartz handler establishes tenant context before calling the expiry service, so the event is fired in the correct tenant context.

**WorkItemQueueEvent** — add `tenancyId` field (see §3).

**WorkLifecycleEvent** (abstract superclass in `casehub-work-api`): no tenancyId accessor. The API module is consumed by `casehub-work-core` which has no CurrentPrincipal dependency. The concrete `WorkItemLifecycleEvent` in runtime carries it.

---

## 7. REST API Surface

No changes. `tenancyId` never appears in request DTOs, response DTOs, JSON Schemas, or OpenAPI specs. Per protocol `PP-20260607-69eba2` (`tenancyid-server-side-only`).

Request/response types already exist separately from entities:
- Request POJOs (`WorkItemCreateRequest` etc.) — no tenancyId field
- Response POJOs (`WorkItemResponse`, `WorkItemWithAuditResponse`) — mapper does not map tenancyId out
- Entity (`WorkItem`) — has tenancyId, internal only

---

## 8. Cross-Module SPI Boundary

**SpawnPort.spawn():** called by casehub-engine's work-adapter within the engine's tenant context. Flows through WorkItemSpawnService → WorkItemStore.put() → store stamps tenancyId from CurrentPrincipal. No SPI signature change needed.

**work-flow module (HumanTaskFlowBridge):** `requestApproval()` and `requestGroupApproval()` call `WorkItemService.create()` synchronously. The caller's request context propagates — `CurrentPrincipal.tenancyId()` is available if the Quarkus-Flow task function was invoked within a request context. `WorkItemFlowEventListener` is a synchronous `@Observes` observer — it inherits the caller's tenant context when resolving the `CompletableFuture`. The Uni returned by `requestApproval()` suspends waiting for the CompletableFuture; the WorkItem creation happens synchronously with tenant context; the completion event fires from a REST call (which has tenant context). No changes needed, but documented: **Quarkus-Flow callers must invoke `HumanTaskFlowBridge` within a tenant-scoped context.** If a flow task function runs on a background thread without request scope, the WorkItem will be created with the wrong (or default) tenancyId.

**Webhook resources (Jira, GitHub):** inbound with no CaseHub authentication. Webhook configuration must carry a tenancyId. Resource looks up config, establishes tenant context via TenantContextRunner, then processes. Out of scope for this issue — tracked as #258.

**CaseSignalSink:** referenced in PLATFORM.md and CLAUDE.md but does not exist as code. Removed from this spec. If/when CaseSignalSink is implemented, it will be called within a tenant context established by the Quartz timer handler (SLA escalation fires the signal).

---

## 9. Flyway Migrations

Each module's migration stays within its allocated V-number range.

| Migration | Module | Tables |
|-----------|--------|--------|
| V35 | runtime | work_item, work_item_template, audit_entry, work_item_note, work_item_link, work_item_spawn_group, work_item_schedule, work_item_relation, routing_cursor, label_definition, label_vocabulary, filter_rule |
| V2002 | ledger | work_item_ledger_entry |
| V2003 | queues | work_item_queue_state, queue_view, work_item_filter, filter_chain, work_item_queue_membership |
| V3001 | notifications | work_item_notification_rule |
| V4002 | ai | worker_skill_profile, escalation_summary |
| V5002 | issue-tracker | work_item_issue_link |

Each migration: `ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce'`. Plus constraint and PK changes for WorkItemTemplate and RoutingCursor. Plus index on `tenancy_id` per table.

**Reports module:** no migration needed. ReportService queries WorkItem and AuditEntry tables directly via EntityManager JPQL — those tables are covered by V35.

---

## 10. Testing Strategy

### Test infrastructure

`FixedCurrentPrincipal` does not exist in the platform. `MockCurrentPrincipal` is config-bound (`@ConfigProperty`) with no programmatic `setTenancyId()` method.

**Create `MutableCurrentPrincipal`** in casehub-work's test scope (`runtime/src/test/java/`): a `@ApplicationScoped @Alternative @Priority(100)` bean that implements `CurrentPrincipal` with mutable `tenancyId`, `actorId`, and `crossTenantAdmin` fields. This displaces `MockCurrentPrincipal` in test contexts. Multi-tenant tests call `setTenancyId()` to switch context mid-test:

```java
mutablePrincipal.setTenancyId("tenant-a");
workItemStore.put(itemA);
mutablePrincipal.setTenancyId("tenant-b");
assertThat(workItemStore.get(itemA.id)).isEmpty(); // tenant isolation
```

### Level 1: Store isolation tests

Every store implementation (JPA, InMemory, MongoDB) gets multi-tenant tests:
- Create data as tenant A → switch to tenant B → assert B cannot see/modify/delete A's data
- Assert `put()` on update preserves existing tenancyId
- Assert `isCrossTenantAdmin() = true` sees all tenants

### Level 2: Timer and async context propagation tests

- Create WorkItem with `expiresAt` in tenant A → Quartz job fires → expiry in tenant A's context → tenant B untouched
- Same for claim deadline timers
- Multi-instance group: `@ObservesAsync` MultiInstanceCoordinator processes in correct tenant context
- WorkItemSchedule timer: instantiated WorkItem belongs to correct tenant

### Level 3: SSE stream isolation tests

- Subscribe to SSE as tenant A → create WorkItem as tenant B → tenant A receives no events
- Subscribe to SSE as tenant A → create WorkItem as tenant A → tenant A receives the event
- Same for queue SSE streams

### Level 4: End-to-end integration tests

Two-tenant REST scenarios:
- Isolation: POST as tenant A, GET as tenant B → 404
- Lifecycle isolation: create/claim/complete as tenant A → tenant B audit empty
- Template isolation: same name in both tenants → both succeed, each sees only their own
- Spawn isolation: children visible only to parent's tenant
- Expiry isolation: both tenants' items expire independently with correct tenancyId on events
- Cross-tenant prevention: access tenant A's WorkItem ID as tenant B → 404 (not 403)
- Ledger isolation: WorkItemLedgerEntry visible only to its tenant

---

## 11. Platform Coherence — RLS Relationship

casehub-engine uses `TenantAwareRepository` with `SET LOCAL "casehub.tenancy_id"` and `RlsPolicyApplicator` for PostgreSQL Row-Level Security (in `casehub-engine/persistence-hibernate`). casehub-work uses blocking Hibernate ORM (not Reactive), so the `TenantAwareRepository` pattern does not directly port.

casehub-work implements application-level filtering only. This creates a platform divergence: in a shared deployment where both engine and work tables live in the same schema, engine tables have RLS enforced at the database level while work tables rely solely on application code.

**Path to convergence:** application-level filtering is the prerequisite for RLS — the `tenancy_id` column and the filtering logic must exist first. RLS policies will be added in #257 as a defence-in-depth layer on the same `tenancy_id` column. The blocking Hibernate equivalent of `TenantAwareRepository` (using `EntityManager.createNativeQuery("SET LOCAL ...")` within a `@Transactional` boundary) will be designed in that issue.

---

## 12. Protocols Referenced

| Protocol | Rule |
|----------|------|
| `tenancy-repository-pattern` | Filtering in data access layer, never at call sites |
| `no-conditional-tenancy-filtering` | Always on, never gated on deployment mode |
| `tenancyid-server-side-only` | Never in client-facing APIs (PP-20260607-69eba2) |
| `subcase-tenancyid-inherits-parent` | Children inherit parent's tenancyId |
| `spi-event-tenancyid-component-order` | tenancyId position in CDI event records |

---

## 13. Out of Scope (Separate Issues)

- **#257** — PostgreSQL RLS hardening — defence-in-depth layer; path to convergence with engine's `TenantAwareRepository` pattern (see §11)
- **#258** — Webhook tenant mapping configuration for issue-tracker module (Jira/GitHub inbound)
- **#259** — Update `tenancy-repository-pattern` protocol to acknowledge background/async paths that use TenantContextRunner instead of CurrentPrincipal directly
- Promotion of `TenantContextRunner` to `casehub-platform` — starts in casehub-work, promoted when other repos need it
