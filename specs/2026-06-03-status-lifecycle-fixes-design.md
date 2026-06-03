# Design — WorkItem Status & Lifecycle Fixes

**Branch:** `issue-243-status-lifecycle-fixes`
**Date:** 2026-06-03
**Issues:** #243, #244, #245, #239 (verify), #241

---

## #243 — EXPIRED in isTerminal()

`WorkItemStatus.isTerminal()` currently returns `true` for COMPLETED, REJECTED, CANCELLED, ESCALATED — but not EXPIRED. EXPIRED is set by `ExpiryLifecycleService.executeFail()` with `completedAt` set; no lifecycle rule transitions out of it. It is unambiguously terminal.

**Fix:** add EXPIRED to the switch case.

```java
case COMPLETED, REJECTED, CANCELLED, ESCALATED, EXPIRED -> true;
```

DELEGATED stays non-terminal (PLATFORM.md Known Overlap Risk #3: Qhorus DELEGATED is terminal for the obligor; WorkItem DELEGATED is not — they must not be conflated).

EXPIRED is also absent from `isActive()` — correctly so (it's neither active nor terminal was the bug). No change to `isActive()`.

---

## #244 — ESCALATED gets real meaning (Option B)

### Problem

`BreachExecutionFailed` is never thrown, so the Chained fallback mechanism is dead code. `WorkItemStatus.ESCALATED` is in `isTerminal()` but never set on any WorkItem.

### Design

**Signal:** `executeEscalateTo` throws `BreachExecutionFailed` when `escalate.groups().isEmpty()` — before any state mutation. This is a pre-execution policy error, not a runtime failure.

**Guard removal:** the inline empty-groups guard in `executeBreachDecision`'s EscalateTo case is removed (it moves into `executeEscalateTo`).

**Chained handler:** gains a nested try-catch. If the fallback branch also throws `BreachExecutionFailed`, `executeExhausted()` is called.

**`executeExhausted(WorkItem item, Instant now)`:**
- `item.status = WorkItemStatus.ESCALATED`
- `item.completedAt = now`
- `workItemStore.put(item)`
- writes audit entry: event="ESCALATED", detail="policy-exhausted"
- fires lifecycle event: "ESCALATED"
- returns `BreachDecision.Fail("policy-exhausted")` as the leaf (no `BreachDecision.Exhausted` type exists in the API — semantic gap noted; the WorkItem status tells the true story)

**Batch safety:** `checkExpired()` and `checkClaimDeadlines()` wrap each item's processing in a try-catch for `BreachExecutionFailed`. A misconfigured non-Chained EscalateTo (empty groups) logs an error and skips that item; other items in the same tick are not rolled back. The item stays in its current state and is retried on the next tick.

**Semantics:** ESCALATED = all configured SLA breach policy branches were exhausted; operator intervention required. It is terminal and requires explicit operator action to resolve (currently: create a new WorkItem or cancel).

**FilterRegistryEngine:** already treats ESCALATED as a REMOVE event — no change needed.

---

## #245 — DELEGATED lifecycle

### Problem

`WorkItemStatus.DELEGATED` is never set — `delegate()` sets PENDING instead. `DelegationState` (PENDING/RESOLVED) is set but disconnected from status and never transitions to RESOLVED. No "items delegated to me" query is possible; no decline-delegation flow exists.

### Status classification change

`WorkItemStatus.DELEGATED` is added to `isActive()` — it is a live, pre-acceptance state. Stays non-terminal.

### Drop DelegationState

`DelegationState.java` is deleted. `WorkItemStatus.DELEGATED` replaces the pre-acceptance semantics. No other enum is needed — ASSIGNED (after accept) and PENDING (after decline) complete the picture.

Files updated on drop:
- `WorkItem.java` — remove `delegationState` field
- `MongoWorkItemDocument.java` — remove mapping
- `WorkItemResponse.java` — remove field
- `WorkItemWithAuditResponse.java` — remove field
- `WorkItemMapper.java` — remove mapping
- `WorkItemService.java` — remove assignment in `delegate()`

### New field: delegation decline target

`WorkItem.delegationDeclineTarget` — nullable String (`"POOL"` or `"DELEGATOR"`). Stores the instance-level override for where a declined delegation returns to.

- Written by `delegate()` if the request body specifies `declineTarget`
- Null = use scope preference

### New preference key: DeclineTarget

In `casehub-work-api`:

```java
public enum DeclineTarget implements SingleValuePreference {
    POOL, DELEGATOR;

    public static final PreferenceKey<DeclineTarget> KEY =
        new PreferenceKey<>("casehub.work.delegation", "decline-target", POOL,
                            s -> DeclineTarget.valueOf(s.toUpperCase(Locale.ROOT)));
}
```

Default: `POOL`. Namespace: `casehub.work.delegation`. Name: `decline-target`.

### delegate() changes

- Sets `item.status = WorkItemStatus.DELEGATED` (not PENDING)
- Removes `item.delegationState = DelegationState.PENDING`
- Reads optional `declineTarget` from request body; stores on `item.delegationDeclineTarget` if present
- assigneeId assignment logic unchanged (toAssigneeId or strategy result)
- `item.claimDeadline = null` — DELEGATED is a directly-addressed state; pool claim deadlines do not apply. The prior claimDeadline computation (which fired only on the PENDING branch) is removed from this path

### New service methods

**`acceptDelegation(UUID id, String claimantId)`:**
- Validates: `item.status == DELEGATED && item.assigneeId.equals(claimantId)`
- Sets: `item.status = ASSIGNED`, `item.assignedAt = now`
- Audit: "DELEGATION_ACCEPTED"
- Lifecycle event: "DELEGATION_ACCEPTED"

**`declineDelegation(UUID id, String actorId)`:**
- Validates: `item.status == DELEGATED && item.assigneeId.equals(actorId)`
- Resolves decline target: `item.delegationDeclineTarget` if non-null, else reads `DeclineTarget.KEY` from `PreferenceProvider` at the WorkItem's scope
- **POOL path:** `item.status = PENDING`, `item.assigneeId = null`, recompute `claimDeadline`, call `assignmentService.assign(item, RELEASED)`
- **DELEGATOR path:** parse previous actor from `delegationChain` (last entry); `item.assigneeId = prevActor`, `item.status = ASSIGNED`, `item.assignedAt = now`
- Audit: "DELEGATION_DECLINED"
- Lifecycle event: "DELEGATION_DECLINED"

`WorkItemService` gains `@Inject PreferenceProvider preferenceProvider`.

### V34 Flyway migration

`runtime/src/main/resources/db/work/migration/V34__delegation_state_drop_and_decline_target.sql`:

```sql
ALTER TABLE work_item DROP COLUMN delegation_state;
ALTER TABLE work_item ADD COLUMN delegation_decline_target VARCHAR(10);
```

### New REST endpoints

`WorkItemResource`:
- `PUT /workitems/{id}/accept-delegation?claimant={actorId}` → `acceptDelegation(id, claimant)`
- `PUT /workitems/{id}/decline-delegation?actor={actorId}` → `declineDelegation(id, actor)`
- `DelegateRequest` body gains optional `declineTarget` field (String, nullable; validated as POOL/DELEGATOR if present)

`claim()` guard unchanged: only PENDING — DELEGATED must use accept-delegation.

---

## #239 — GroupMembershipProvider callers

All production callers in casehub-work already use `Set<GroupMember>` correctly:
- `TemplateExpander` uses `m.actorId()` ✅
- `NoOpGroupMembershipProvider` returns `Set<GroupMember>` ✅

No WorkBroker, WorkerSelectionStrategy, or ExclusionPolicy calls `membersOf()` directly. Issue is complete — close on this branch.

---

## #241 — findById in WorkItemService

Add to `WorkItemService`:

```java
public Optional<WorkItem> findById(UUID id) {
    return workItemStore.get(id);
}
```

Update `WorkItemResource.getById()`:

```java
@GET
@Path("/{id}")
public WorkItemWithAuditResponse getById(@PathParam("id") final UUID id) {
    final WorkItem wi = workItemService.findById(id)
            .orElseThrow(() -> new WorkItemNotFoundException(id));
    final List<AuditEntry> trail = auditStore.findByWorkItemId(id);
    return WorkItemMapper.toWithAudit(wi, trail);
}
```

The `workItemStore` injection in `WorkItemResource` can be removed for `getById` (it remains for `listAll`, `inbox`, `inboxSummary`, and children/parent lookups).

---

## Implementation Order

1. #243: `WorkItemStatus.isTerminal()` + test
2. #244: `ExpiryLifecycleService` Option B wiring + tests
3. #245: Drop DelegationState, add DELEGATED lifecycle, V34 migration, REST endpoints + tests
4. #239: Verify + close issue
5. #241: `WorkItemService.findById()` + update REST endpoint + test
