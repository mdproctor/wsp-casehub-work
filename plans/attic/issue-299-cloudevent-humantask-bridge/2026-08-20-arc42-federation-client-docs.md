# ARC42 Federation/Client Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #360 — docs: add ARC42STORIES.MD coverage for federation/ and client/ modules
**Issue group:** #360

**Goal:** Add L8 Federation layer documentation to ARC42STORIES.MD — C4 diagram, Module Index, Layer × Chapter matrix, and full §9.4 layer entry.

**Architecture:** Single-file documentation change to `ARC42STORIES.MD`. Four insertion points: (1) C4 mermaid diagram, (2) Module Index table, (3) three Layer × Chapter matrix tables, (4) new §9.4 layer entry before §10. All content is specified in the design spec — this plan maps insertions to exact line ranges.

**Tech Stack:** Markdown (ARC42 conventions)

## Global Constraints

- All changes target `ARC42STORIES.MD` in the project repo
- Follow existing ARC42 template patterns exactly (match L1–L7 formatting)
- Federation source lives on `issue-92-distributed-workitems` branch — reference it as the source, not main
- No code changes — documentation only

---

## Batch 1: §5 Building Block View updates

### Task 1: Add L8 Federation to C4 diagram, Module Index, and Layer × Chapter matrix

**Files:**
- Modify: `ARC42STORIES.MD:167-175` (C4 diagram — add Container_Boundary after b7)
- Modify: `ARC42STORIES.MD:198` (Module Index — add rows after `flow/` row)
- Modify: `ARC42STORIES.MD:463,473,483` (three matrix tables — add L8 row after each L7 row)

**Interfaces:**
- Consumes: nothing
- Produces: L8 Federation layer defined in §5 (referenced by Task 2's layer entry)

- [ ] **Step 1: Add L8 Container_Boundary to C4 mermaid diagram**

Insert after line 174 (`  }` closing b7) and before line 175 (closing triple-backtick):

```markdown
  Container_Boundary(b8, "Layer: L8 Federation") {
    Container(federation, "casehub-work-federation", "Optional Quarkus extension", "FederationGuardStore (@Decorator on WorkItemStore.put()), FederationProxyService (@Decorator on WorkItemOperations), FederationReceiver (inbound CloudEvents → shadow WorkItems), FederationEventRouter (outbound lifecycle events → subscriptions), FederationSubscriptionService, Flyway V8000")
    Container(client, "casehub-work-client", "Pure Java", "WorkItemClient — zero-dependency HTTP client for remote WorkItem operations (claim, complete, reject, delegate, release)")
  }
```

- [ ] **Step 2: Add federation/ and client/ rows to Module Index**

Insert after line 198 (`flow/` row) and before line 199 (`integration-tests/` row):

```markdown
| `federation/` | `casehub-work-federation` | Optional extension | Cross-service federation; `FederationGuardStore` (`@Decorator` on `WorkItemStore.put()`), `FederationProxyService` (`@Decorator` on `WorkItemOperations`), `FederationReceiver` (inbound CloudEvents → shadow WorkItems), `FederationEventRouter` (outbound lifecycle events → subscriptions), subscription model with filter-on-creation lock-on. Flyway V8000. |
| `client/` | `casehub-work-client` | Pure Java library | Lightweight REST client for remote WorkItem operations (claim, complete, reject, delegate, release). No JPA, no CDI, no Quarkus dependencies. Used by `FederationProxy` and standalone consumers. |
```

- [ ] **Step 3: Add L8 Federation row to all three Layer × Chapter matrix tables**

After line 463 (L7 row in first table):
```markdown
| L8 Federation | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
```

After line 473 (L7 row in second table):
```markdown
| L8 Federation | — | — | — | — | — | — | — | — | — | — | — | — |
```

After line 483 (L7 row in third table):
```markdown
| L8 Federation | — | — | — | — | — | — | — | — |
```

- [ ] **Step 4: Verify markdown renders correctly**

Run: `grep -c "L8 Federation" ARC42STORIES.MD`
Expected: 4 occurrences (1 in C4 diagram description is implicit via b8, plus 3 matrix rows — actually 3 matrix rows. The C4 block says "Layer: L8 Federation" in the boundary label, so grep finds it there too = 4 total).

Also verify the mermaid block is valid — opening and closing backticks balanced:
Run: `grep -n 'Container_Boundary(b8' ARC42STORIES.MD`
Expected: one match

- [ ] **Step 5: Commit**

```bash
git add ARC42STORIES.MD
git commit -m "docs(#360): add L8 Federation to §5 Building Block View — C4 diagram, Module Index, matrix

Refs #360 Refs #92"
```

---

## Batch 2: §9.4 Layer Entry

### Task 2: Add L8 Federation layer entry to §9.4

**Files:**
- Modify: `ARC42STORIES.MD:1801` (insert new layer entry after L7's closing `---` and before `## §10`)

**Interfaces:**
- Consumes: L8 Federation defined in §5 (Task 1)
- Produces: complete layer entry for federation/ and client/ modules

- [ ] **Step 1: Insert L8 Federation layer entry**

Insert after line 1801 (`---` closing L7 section) and before line 1803 (`## §10 Architectural Decisions`). The full content below follows the exact template pattern established by L1–L7:

```markdown

### Layer — L8 Federation

**Participates in chapters:** —
**Architectural patterns:** Decorator, Shadow Replication, Pub-Sub (filter-on-creation + lock-on), Proxy
**Key protocols:** —
**Issues:** #92, #95
**Navigation:** `git log issue-92-distributed-workitems --grep="#92\|#95" --oneline`
**Completed:** in progress (issue-92-distributed-workitems branch)

#### What it adds

**Before:** WorkItems exist only within a single Quarkus application. A WorkItem created in service A is invisible to service B. No mechanism exists for cross-service task distribution, and local mutations of remotely-owned items are uncontrolled.
**After:** `casehub-work-federation` enables bidirectional cross-service WorkItem sharing via a publish-subscribe model. Outbound: `FederationEventRouter` publishes lifecycle events as CloudEvents to peer services that match subscription filters. Inbound: `FederationReceiver` creates read-only shadow copies with HMAC-verified integrity and version-based staleness rejection. `FederationGuardStore` enforces shadow immutability at the store layer. `FederationProxyService` transparently routes mutations on shadow items back to the owning service via `WorkItemClient` (a zero-dependency REST client in the `client/` module).

What this layer adds:
- **Shadow replication** — `FederationReceiver` processes inbound CloudEvents, creates/updates local shadow WorkItems with `originServiceId` + `originWorkItemId` + `originVersion` fields; rejects stale updates where incoming version ≤ existing version
- **Shadow immutability guard** — `FederationGuardStore` is a `@Decorator` on `WorkItemStore.put()` that blocks all mutations on shadow items (those with `originServiceId` set) unless `FederationSyncContext` is active (ThreadLocal flag set by `FederationReceiver` during sync)
- **Transparent proxy mutations** — `FederationProxyService` is a `@Decorator` on `WorkItemOperations` that intercepts claim/complete/reject/delegate/release on shadow items and routes them to the owning service via `FederationProxy` → `WorkItemClient` REST calls
- **Subscription model** — `FederationSubscriptionService` manages peer registrations with `callbackUrl`, `baseUrl`, `tenancyId`, `SubscriptionFilter` (candidateGroups, candidateUsers), HMAC secret, and auto-suspend after 5 consecutive delivery failures
- **Outbound event distribution** — `FederationEventRouter` observes `WorkItemLifecycleEvent` during `TransactionPhase.AFTER_SUCCESS`; creation events trigger filter-based subscription matching + lock-on; subsequent events reuse locked subscriptions; terminal events clean up tracking
- **HMAC transport security** — `HmacSigner` signs outbound CloudEvents and verifies inbound signatures; per-subscription secrets
- **Lightweight REST client** — `WorkItemClient` (in `client/` module) is pure Java with no Quarkus/CDI/JPA dependencies; five operations (claim, complete, reject, delegate, release); returns `ClientResponse` record with status code and optional JSON body

Not closed here: cross-service WorkItem creation (#95 federation path), Qhorus event bus integration (#97), federated audit trail reconciliation.

#### Accountability Gaps Closed

| Gap | What breaks without it | Closed by |
|---|---|---|
| No cross-service task visibility | WorkItems created in service A are invisible to agents/humans in service B | Shadow replication via `FederationReceiver` — CloudEvent-driven projections appear in the local inbox |
| Uncontrolled shadow mutation | Local code could mutate a shadow item, diverging from the owning service's authoritative state | `FederationGuardStore` `@Decorator` blocks `put()` on shadow items unless `FederationSyncContext` is active |
| No mutation routing for shadows | Claiming or completing a shadow item would modify only the local copy with no effect on the real item | `FederationProxyService` `@Decorator` intercepts lifecycle operations on shadows and proxies them to the owning service |
| No delivery reliability tracking | Failed event deliveries to peer services would go unnoticed | `FederationSubscriptionService.recordFailure()` tracks consecutive failures; auto-suspends subscription after 5 |

#### Key Files

**Federation module:**

`federation/src/main/java/io/casehub/work/federation/FederationGuardStore.java` — `@Decorator @Priority(APPLICATION)` on `WorkItemStore`; `put()` throws `FederatedWorkItemMutationException` if `originServiceId` is set and `FederationSyncContext` is not active. `get()` and `scan()` delegate transparently.

`federation/src/main/java/io/casehub/work/federation/FederationReceiver.java` — `@ApplicationScoped` inbound CloudEvent processor; verifies HMAC, parses CloudEvent JSON, runs in tenant context via `TenantContextRunner`, reconciles shadows by `(originServiceId, originWorkItemId)` tuple, rejects stale versions, namespaces `callerRef` to `federation:{sourceServiceId}:{originalCallerRef}`.

`federation/src/main/java/io/casehub/work/federation/FederationEventRouter.java` — `@ApplicationScoped` outbound event distributor; `@Observes(during = TransactionPhase.AFTER_SUCCESS) WorkItemLifecycleEvent`; skips shadow items (`originServiceId != null`); creation events → filter matching + lock-on; non-creation events → locked subscription lookup; terminal events → tracking cleanup.

`federation/src/main/java/io/casehub/work/federation/FederationProxyService.java` — `@Decorator @Priority(APPLICATION + 10)` on `WorkItemOperations`; intercepts claim, complete, reject, delegate, release on shadow items (detected by `originServiceId != null`); delegates to `FederationProxy` for remote execution. All other operations pass through to the local delegate.

`federation/src/main/java/io/casehub/work/federation/FederationProxy.java` — `@ApplicationScoped` bridge to `WorkItemClient`; resolves the owning service's base URL from active subscriptions matching `shadow.originServiceId`; maps `ClientResponse` to success/409 Conflict/503 Unavailable.

`federation/src/main/java/io/casehub/work/federation/FederationSyncContext.java` — `ThreadLocal<Boolean>` guard; `activate()` returns an `AutoCloseable` that sets the flag; `FederationGuardStore` checks `isActive()` before allowing shadow mutation. Used exclusively by `FederationReceiver` in a try-with-resources block.

`federation/src/main/java/io/casehub/work/federation/FederationConfig.java` — SmallRye `@ConfigMapping(prefix = "casehub.work.federation")`; `serviceId()` (default: `"default"`) identifies this service in CloudEvent `source` URN; `proxyTimeoutSeconds()` (default: `5`) configures `WorkItemClient` timeout.

`federation/src/main/java/io/casehub/work/federation/subscription/FederationSubscriptionService.java` — `@ApplicationScoped` subscription lifecycle; `register()` creates subscription with peerId, callbackUrl, baseUrl, tenancyId, filter, capabilities, HMAC secret; `matchSubscriptions()` evaluates `SubscriptionFilterEvaluator` against WorkItem; `lockOn()` creates tracking entry; `recordSuccess()`/`recordFailure()` track delivery; auto-suspends after 5 consecutive failures.

`federation/src/main/java/io/casehub/work/federation/subscription/FederationSubscriptionEntity.java` — Panache entity; `federation_subscription` table (V8000); `SubscriptionStatus` enum (ACTIVE, SUSPENDED).

`federation/src/main/java/io/casehub/work/federation/subscription/FederationTrackingEntity.java` — Panache entity; `federation_subscription_tracking` table (V8000); composite PK `(subscriptionId, workItemId)`.

`federation/src/main/java/io/casehub/work/federation/subscription/SubscriptionFilter.java` — record with `candidateGroups`, `candidateUsers`, `tenancyId`; defensive copies via `List.copyOf()`.

`federation/src/main/java/io/casehub/work/federation/subscription/SubscriptionFilterEvaluator.java` — stateless matcher; tenancyId must match; candidateGroups/candidateUsers intersected against WorkItem's CSV fields; empty filter matches all items in the tenant.

`federation/src/main/java/io/casehub/work/federation/transport/FederationTransport.java` — SPI interface for outbound event delivery; `send(cloudEventJson, callbackUrl, hmacSecret)`.

`federation/src/main/java/io/casehub/work/federation/transport/WebhookFederationTransport.java` — HTTP POST implementation of `FederationTransport`.

`federation/src/main/java/io/casehub/work/federation/transport/HmacSigner.java` — HMAC-SHA256 signing and verification for CloudEvent payloads.

`federation/src/main/java/io/casehub/work/federation/rest/FederationEventResource.java` — JAX-RS endpoint for receiving inbound federation CloudEvents.

`federation/src/main/java/io/casehub/work/federation/rest/FederationSubscriptionResource.java` — JAX-RS endpoint for subscription CRUD.

`federation/src/main/resources/db/work/migration/V8000__federation_subscription.sql` — creates `federation_subscription` and `federation_subscription_tracking` tables with index on `work_item_id`.

**Client module:**

`client/src/main/java/io/casehub/work/client/WorkItemClient.java` — pure Java HTTP client using `java.net.http.HttpClient`; five operations (claim, complete, reject, delegate, release); `ClientResponse` record with `statusCode`, `body` (JsonNode), and convenience predicates (`isSuccess()`, `isConflict()`, `isUnavailable()`). No Quarkus, CDI, or JPA dependency.

#### Key Wiring

**`FederationGuardStore` `@Decorator @Priority(APPLICATION)` on `WorkItemStore`.** The decorator intercepts `put()` only — `get()` and `scan()` delegate transparently. The guard checks two conditions: `item.originServiceId() != null` (this is a shadow) AND `!FederationSyncContext.isActive()` (we're not inside a federation sync). Both must be true to throw. This ensures that only `FederationReceiver` (which activates the sync context) can write shadow items; all other code paths — including `WorkItemService`, REST handlers, and integration modules — are blocked from mutating shadows.

**`FederationProxyService` `@Decorator @Priority(APPLICATION + 10)` on `WorkItemOperations`.** Priority +10 ensures it wraps after `FederationGuardStore` (which is at APPLICATION). The proxy only intercepts five "actor" operations (claim, complete, reject, delegate, release) on shadow items. All other operations — including create, start, suspend, resume, cancel, fault, obsolete, escalate — delegate directly. This is intentional: shadows should not be created locally (they arrive via `FederationReceiver`), and administrative operations like suspend/cancel are not proxied in the current design.

**`FederationSyncContext` ThreadLocal bypass.** `FederationReceiver.processInTenantContext()` wraps `workItemStore.put(projection)` in `try (var ctx = FederationSyncContext.activate()) { ... }`. The `AutoCloseable` pattern ensures the ThreadLocal is always cleaned up, even on exception. The decorator checks `isActive()` synchronously within the same thread — no async boundary can intervene.

**Filter-on-creation + lock-on subscription model.** `FederationEventRouter` evaluates `SubscriptionFilterEvaluator` only on creation events (`*.created`). Matching subscriptions are locked to the WorkItem via `FederationTrackingEntity(subscriptionId, workItemId)`. All subsequent lifecycle events for that WorkItem reuse the locked subscriptions without re-evaluating filters. This avoids re-evaluation cost on every event and ensures a WorkItem is consistently routed to the same set of peers for its entire lifecycle. Terminal events trigger `removeTracking()` cleanup.

**`@Observes(during = TransactionPhase.AFTER_SUCCESS)` on `FederationEventRouter`.** Outbound federation events fire only after the transaction commits — a rolled-back transition never publishes to peer services. This matches the L6 broadcaster pattern.

**Shadow items skip outbound routing.** `FederationEventRouter.onWorkItemLifecycle()` returns immediately if `workItem.originServiceId() != null`. This prevents federation echo: a shadow update received from service A is not re-published to other subscribers, avoiding infinite event loops in multi-service topologies.

**`callerRef` namespacing.** `FederationReceiver` rewrites `callerRef` to `federation:{sourceServiceId}:{originalCallerRef}`. This preserves the original callerRef for traceability while ensuring shadow callerRefs don't collide with locally-created items that may share the same original callerRef value.

**`FederationProxy` resolves owner URL from subscriptions.** When proxying a mutation on a shadow item, `FederationProxy` looks up active subscriptions with `peerId` matching `shadow.originServiceId()` to find the `baseUrl`. This means the subscription table serves double duty: outbound event routing AND inbound proxy URL resolution.

**Flyway V8000 at `db/work/migration/`.** Two tables: `federation_subscription` (subscription lifecycle) and `federation_subscription_tracking` (lock-on pairs). V8000 is in the federation module's reserved range.

#### Architectural Decisions

**Why `@Decorator` pattern rather than a separate `FederatedWorkItemStore` implementation:** A decorator wraps any underlying store — JPA, MongoDB, in-memory — without knowing which one is active. A separate implementation would need to duplicate or compose with every store variant. The decorator pattern also integrates naturally with CDI's priority-based selection, and the guard check (`originServiceId != null && !FederationSyncContext.isActive()`) is orthogonal to storage concerns. Tradeoff: decorators are globally active when the module is on the classpath — there is no way to disable the guard without removing the module.

**Why shadow immutability rather than conflict resolution:** Allowing local mutations on shadow items would require a conflict resolution strategy (last-writer-wins, merge, vector clocks). Shadow immutability is simpler: the owning service is the single source of truth; mutations are proxied back to it. Tradeoff: shadow items have higher write latency (remote call required) and are unavailable when the owning service is down.

**Why filter-on-creation + lock-on rather than re-evaluate on every event:** Re-evaluating subscription filters on every lifecycle event would be correct but expensive — each event would scan all active subscriptions and evaluate filters. Lock-on at creation time means subsequent events are O(1) lookup by workItemId. Tradeoff: if a subscription's filter changes after lock-on, the change does not affect already-locked WorkItems. This is acceptable because subscriptions represent stable service-to-service relationships, not dynamic query surfaces.

**Why zero-dependency client module:** `WorkItemClient` uses only `java.net.http.HttpClient` and Jackson (already on any Quarkus classpath). No Quarkus, CDI, or JPA dependency means it can be used in standalone Java applications, CLI tools, or test harnesses without pulling in the extension framework. Tradeoff: no MicroProfile REST Client features (retry, circuit breaker, health) — consumers must handle resilience themselves.

**Why auto-suspend after 5 consecutive failures:** A failing subscription generates error logs and wasted HTTP calls on every lifecycle event. Auto-suspend limits the blast radius. The threshold (5) is hardcoded — no configuration property. Tradeoff: a transient outage lasting more than 5 events permanently suspends the subscription; manual re-activation is required.

#### Pattern Introduced

Decorator-guarded shadow replication — a `@Decorator` on the store SPI enforces write protection on replicated items using a ThreadLocal context flag; only the replication receiver can bypass the guard. Combined with a second `@Decorator` on the operations SPI for transparent proxy routing, this creates a bidirectional federation where local consumers interact with shadow items identically to local items, but mutations are transparently routed to the owning service.

#### Pattern Anchor

`federation/src/main/java/io/casehub/work/federation/FederationGuardStore.java` — the `@Decorator` that enforces shadow immutability
`federation/src/main/java/io/casehub/work/federation/FederationProxyService.java` — the `@Decorator` that proxies shadow mutations to the owning service

#### Gotchas

**Symptom:** Inbound federation events are silently discarded with log message "Stale federation event discarded."
**Cause:** `FederationReceiver` rejects events where `incomingVersion <= shadow.originVersion()`. If the owning service sends events out of order (e.g. due to retry or parallel processing), later events may arrive with a lower version than an earlier-processed event.
**Fix:** Ensure the owning service publishes events with monotonically increasing `WorkItem.version` (guaranteed by `@Version` OCC). If events are retried, the retry carries the same version — a duplicate is harmless (version ==, rejected). True out-of-order delivery (version < current) indicates a bug in the transport layer or a concurrent modification on the owning service.

**Symptom:** `FederatedWorkItemMutationException` thrown when `WorkItemService` tries to update a shadow item during a legitimate federation sync.
**Cause:** `FederationSyncContext` is ThreadLocal. If `FederationReceiver.processInTenantContext()` runs on a different thread than `workItemStore.put()` (e.g. due to a Vert.x context switch or `@Transactional(REQUIRES_NEW)` spawning a new thread), the ThreadLocal flag is not visible.
**Fix:** Ensure `FederationReceiver` processing stays on the same thread from `FederationSyncContext.activate()` through `workItemStore.put()`. Do not use `REQUIRES_NEW` or async dispatch within the sync path.

**Symptom:** A peer service's subscription is permanently suspended after a brief network outage.
**Cause:** `FederationSubscriptionService.recordFailure()` increments `consecutiveFailures` and auto-suspends at 5. A burst of lifecycle events during the outage reaches the threshold quickly.
**Fix:** Manual re-activation required — update the `federation_subscription` row to set `status = 'ACTIVE'` and `consecutive_failures = 0`. Consider adding a REST endpoint for subscription re-activation in a future iteration.

**Symptom:** Shadow items appear with `callerRef = null` even though the original WorkItem had a callerRef.
**Cause:** `FederationReceiver` reads `callerRef` from `data.path("callerRef")` in the CloudEvent payload. If the owning WorkItem's callerRef was null, the namespacing logic produces null (the `projection.callerRef() != null` check returns false).
**Fix:** This is by design — shadow items with no original callerRef get no namespaced callerRef.

#### Pattern to Replicate

1. Define a `@Decorator @Priority(APPLICATION)` on the domain's store SPI that guards write operations on replicated items. Use a ThreadLocal context flag (with `AutoCloseable` cleanup) to allow the replication receiver to bypass the guard.
2. Define a second `@Decorator @Priority(APPLICATION + N)` on the domain's operations SPI that intercepts actor operations on replicated items and proxies them to the owning service via a lightweight REST client.
3. Implement an inbound receiver that validates transport security (HMAC), reconciles items by `(originServiceId, originItemId)` tuple with version-based staleness rejection, and activates the ThreadLocal context before writing.
4. Implement an outbound event router that observes domain lifecycle events during `AFTER_SUCCESS`, evaluates subscription filters on creation events only, locks matching subscriptions, and reuses locked subscriptions for subsequent events. Skip outbound routing for replicated items to prevent echo loops.
5. Define a subscription model with filter, callback URL, HMAC secret, and delivery reliability tracking (consecutive failure count with auto-suspend threshold).
6. Package the REST client as a separate zero-dependency module so it can be used by the federation proxy and by standalone consumers without pulling in the extension framework.
7. Reserve a Flyway version range (V8000+) for federation schema (subscription + tracking tables).
```

- [ ] **Step 2: Verify the layer entry is between L7 and §10**

Run: `grep -n "### Layer — L8\|## §10" ARC42STORIES.MD`
Expected: L8 line < §10 line, with L8 appearing after L7's closing `---`

- [ ] **Step 3: Verify all template sections are present**

Run: `grep -c "#### " ARC42STORIES.MD`
Compare with expected: previous count + 7 (What it adds, Accountability Gaps Closed, Key Files, Key Wiring, Architectural Decisions, Pattern Introduced + Pattern Anchor as one, Gotchas, Pattern to Replicate — actually the subsections are: What it adds, Accountability Gaps Closed, Key Files, Key Wiring, Architectural Decisions, Pattern Introduced, Pattern Anchor, Gotchas, Pattern to Replicate = 9 new `####` headings)

- [ ] **Step 4: Commit**

```bash
git add ARC42STORIES.MD
git commit -m "docs(#360): add L8 Federation layer entry to §9.4

Covers federation/ and client/ modules from issue-92-distributed-workitems branch.
Includes shadow replication, guard store decorator, proxy mutations, subscription model,
transport security, and lightweight REST client.

Refs #360 Refs #92"
```

---

## References

- [specs/issue-360-arc42-federation-client-docs/2026-08-20-arc42-federation-client-docs-design.md] — design spec this plan implements
- [ARC42STORIES.MD:135-203] — §5 Building Block View (C4 diagram + Module Index)
- [ARC42STORIES.MD:453-483] — Layer × Chapter matrix tables
- [ARC42STORIES.MD:1169-1801] — §9.4 Layer Entries (L1–L7 template examples)
- [docs/MODULES.md:37-38] — existing federation/ and client/ summaries
- [federation/ source on issue-92-distributed-workitems] — all implementation files
- [client/ source on issue-92-distributed-workitems] — WorkItemClient
- [GitHub #360] — documentation issue
- [GitHub #92] — parent epic (Distributed WorkItems)
- [GitHub #95] — cross-service federation (blocked)
