# WorkItem Status & Lifecycle Fixes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix five lifecycle bugs and gaps: EXPIRED missing from isTerminal(), ESCALATED status never set, DELEGATED status lifecycle disconnected from DelegationState, GroupMembershipProvider callers verified, WorkItemService missing findById.

**Architecture:** Pure enum/service changes in the `runtime` and `api` modules; no new modules introduced. The delegation lifecycle redesign drops `DelegationState`, adds `DeclineTarget` preference key, and introduces two new REST endpoints. A single V34 Flyway migration covers the schema delta.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM 6 (Panache), AssertJ, JUnit 5, RestAssured (@QuarkusTest with H2 MODE=PostgreSQL), Maven.

**Build commands** (always from project root `/Users/mdproctor/claude/casehub/work`):
```bash
scripts/mvn-compile runtime         # compile only (fast)
scripts/mvn-test runtime            # unit tests
scripts/mvn-test runtime -Dtest=Foo # single class
scripts/mvn-install api             # publish api module (needed when api changes)
scripts/mvn-install core            # publish core module (needed when core changes)
```

---

## File Map

| Action | Path | Purpose |
|--------|------|---------|
| Modify | `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java` | Fix isTerminal() + add DELEGATED to isActive() |
| Create | `runtime/src/test/java/io/casehub/work/runtime/model/WorkItemStatusTest.java` | isTerminal()/isActive() enum tests |
| Modify | `api/src/main/java/io/casehub/work/api/BreachDecision.java` | Add `Exhausted` variant to sealed interface |
| Modify | `api/src/main/java/io/casehub/work/api/AssignmentTrigger.java` | Add `DELEGATION_DECLINED` |
| Create | `api/src/main/java/io/casehub/work/api/DeclineTarget.java` | Enum preference key for delegation decline target |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/api/DelegateRequest.java` | Add optional `declineTarget` field |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java` | Option B: BreachExecutionFailed + Exhausted + SLA_REASSIGNED rename |
| Modify | `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java` | New Chained-exhaustion and SLA_REASSIGNED tests; update stale assertion |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java` | Remove `delegationState`, add `delegationDeclineTarget` |
| Delete | `runtime/src/main/java/io/casehub/work/runtime/model/DelegationState.java` | No longer needed |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResponse.java` | Remove `delegationState` field |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemWithAuditResponse.java` | Remove `delegationState` field |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemMapper.java` | Remove `delegationState`, add `delegationDeclineTarget` |
| Modify | `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java` | Remove `delegationState`, add `delegationDeclineTarget` |
| Create | `runtime/src/main/resources/db/work/migration/V34__delegation_state_drop_and_decline_target.sql` | Drop old column, add new column |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemQuery.java` | Add DELEGATED to expired() status list |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java` | Update delegate(), add acceptDelegation(), declineDelegation(), findById() |
| Modify | `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java` | Update delegate test, add delegation lifecycle tests |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResource.java` | New accept/decline-delegation endpoints, update getById() |
| Create | `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemDelegationTest.java` | @QuarkusTest REST tests for accept/decline-delegation |
| Modify | `runtime/src/main/java/io/casehub/work/runtime/filter/FilterRegistryEngine.java` | Update REMOVE case comment |

---

## Task 1: Fix WorkItemStatus.isTerminal() (#243)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java`
- Create: `runtime/src/test/java/io/casehub/work/runtime/model/WorkItemStatusTest.java`

- [ ] **Step 1.1: Write failing tests**

Create `runtime/src/test/java/io/casehub/work/runtime/model/WorkItemStatusTest.java`:

```java
package io.casehub.work.runtime.model;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class WorkItemStatusTest {

    @Test
    void expired_isTerminal() {
        assertThat(WorkItemStatus.EXPIRED.isTerminal()).isTrue();
    }

    @Test
    void delegated_isNotTerminal() {
        assertThat(WorkItemStatus.DELEGATED.isTerminal()).isFalse();
    }

    @Test
    void pending_isActive() {
        assertThat(WorkItemStatus.PENDING.isActive()).isTrue();
    }

    @Test
    void assigned_isActive() {
        assertThat(WorkItemStatus.ASSIGNED.isActive()).isTrue();
    }

    @Test
    void inProgress_isActive() {
        assertThat(WorkItemStatus.IN_PROGRESS.isActive()).isTrue();
    }

    @Test
    void suspended_isActive() {
        assertThat(WorkItemStatus.SUSPENDED.isActive()).isTrue();
    }

    @Test
    void expired_isNotActive() {
        assertThat(WorkItemStatus.EXPIRED.isActive()).isFalse();
    }

    @Test
    void completed_isNotActive() {
        assertThat(WorkItemStatus.COMPLETED.isActive()).isFalse();
    }

    @Test
    void escalated_isTerminal() {
        assertThat(WorkItemStatus.ESCALATED.isTerminal()).isTrue();
    }

    @Test
    void noStatusIsNeitherActiveNorTerminal() {
        // All values must be either terminal, active, or a known non-active non-terminal state.
        // DELEGATED is the only currently-intended non-active, non-terminal status.
        for (WorkItemStatus s : WorkItemStatus.values()) {
            if (s.isTerminal()) {
                assertThat(s.isActive()).as("%s is both terminal and active", s).isFalse();
            }
        }
    }
}
```

- [ ] **Step 1.2: Run test to verify it fails**

```bash
cd /Users/mdproctor/claude/casehub/work
scripts/mvn-test runtime -Dtest=WorkItemStatusTest
```

Expected: `expired_isTerminal` FAILS with `expected: true but was: false`.

- [ ] **Step 1.3: Fix WorkItemStatus.isTerminal()**

In `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java`, update `isTerminal()`:

```java
public boolean isTerminal() {
    return switch (this) {
        case COMPLETED, REJECTED, CANCELLED, ESCALATED, EXPIRED -> true;
        default -> false;
    };
}
```

Leave `isActive()` unchanged for now (DELEGATED is added in Task 8 when the status is first actually set).

- [ ] **Step 1.4: Run tests**

```bash
scripts/mvn-test runtime -Dtest=WorkItemStatusTest
```

Expected: all pass.

- [ ] **Step 1.5: Run full runtime tests to check no regressions**

```bash
scripts/mvn-test runtime
```

Expected: all pass. The `InboxSummaryBuilderTest` and `WorkItemMetricsTest` may now exclude EXPIRED from active counts — this is correct behaviour.

- [ ] **Step 1.6: Commit**

```bash
cd /Users/mdproctor/claude/casehub/work
git add runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java \
        runtime/src/test/java/io/casehub/work/runtime/model/WorkItemStatusTest.java
git commit -m "fix(#243): add EXPIRED to WorkItemStatus.isTerminal()

EXPIRED is a dead-end state set by ExpiryLifecycleService.executeFail() with
completedAt set. No lifecycle rule transitions out of it. Previously missing
from isTerminal() caused EXPIRED items to be cancellable, extendable, countable
as active in metrics, and able to spawn children.

Refs #243, Refs #92"
```

---

## Task 2: Add BreachDecision.Exhausted to sealed interface (#244)

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/BreachDecision.java`

- [ ] **Step 2.1: Add Exhausted variant to BreachDecision.java**

In `api/src/main/java/io/casehub/work/api/BreachDecision.java`, update `permits` clause and add the record:

```java
public sealed interface BreachDecision
        permits BreachDecision.Fail, BreachDecision.EscalateTo,
                BreachDecision.Extend, BreachDecision.Chained, BreachDecision.Exhausted {
```

Add inside the interface, after the `Chained` record:

```java
/**
 * All configured SLA breach policy branches have been exhausted.
 * The WorkItem transitions to {@link io.casehub.work.runtime.model.WorkItemStatus#ESCALATED}
 * and requires operator intervention to resolve.
 *
 * <p>Returned by the runtime when a {@link Chained} policy's primary and fallback both
 * throw {@code BreachExecutionFailed}. May also be returned directly by a
 * {@link SlaBreachPolicy} implementation when it determines no resolution is possible.
 */
record Exhausted(String reason) implements BreachDecision {}
```

- [ ] **Step 2.2: Publish the api module**

```bash
cd /Users/mdproctor/claude/casehub/work
scripts/mvn-install api
```

Expected: BUILD SUCCESS. This makes the new `Exhausted` type available to `runtime`.

- [ ] **Step 2.3: Compile runtime to verify no breakage**

```bash
scripts/mvn-compile runtime
```

Expected: BUILD SUCCESS. The `executeBreachDecision` switch in `ExpiryLifecycleService` uses a `default ->` branch so the new variant compiles cleanly. It will not be reachable until Task 3 adds the handler.

- [ ] **Step 2.4: Commit**

```bash
git add api/src/main/java/io/casehub/work/api/BreachDecision.java
git commit -m "feat(#244): add BreachDecision.Exhausted to sealed interface

Adds fifth variant to the SlaBreachPolicy decision type. Returned when all
Chained policy branches are exhausted, causing WorkItemStatus.ESCALATED to be
set on the WorkItem. Also allows SlaBreachPolicy implementations to signal
exhaustion directly.

Refs #244, Refs #92"
```

---

## Task 3: ExpiryLifecycleService — Option B implementation (#244)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java`

- [ ] **Step 3.1: Write failing tests**

Add these test methods to `ExpiryLifecycleServiceTest.java` (after the existing `checkExpired_*` tests):

```java
// ── #244: Chained exhaustion → ESCALATED ─────────────────────────────────

@Test
void checkExpired_withChainedTwoEmptyEscalateToDecisions_setsEscalatedStatus() {
    // Both primary and fallback fail → item should reach ESCALATED (terminal)
    policy.willReturn(
            BreachDecision.EscalateTo.to("group-a")
                .thenOnBreach(BreachDecision.EscalateTo.to("group-b")));
    // Use a service with an AssignmentService that will cause EscalateTo to throw
    // by having empty groups resolution (groups listed but no members)
    // We manufacture the exhaustion by directly constructing Chained(EscalateTo(empty), EscalateTo(empty))
    policy.willReturn(new BreachDecision.Chained(
            new BreachDecision.EscalateTo(Set.of(), null),
            new BreachDecision.EscalateTo(Set.of(), null)));

    final WorkItem wi = expiredItem();
    service.checkExpired();
    assertThat(store.get(wi.id).orElseThrow().status).isEqualTo(WorkItemStatus.ESCALATED);
}

@Test
void checkExpired_withChainedTwoEmptyEscalateToDecisions_setsCompletedAt() {
    policy.willReturn(new BreachDecision.Chained(
            new BreachDecision.EscalateTo(Set.of(), null),
            new BreachDecision.EscalateTo(Set.of(), null)));
    final WorkItem wi = expiredItem();
    service.checkExpired();
    assertThat(store.get(wi.id).orElseThrow().completedAt).isNotNull();
}

@Test
void checkExpired_withChainedTwoEmptyEscalateToDecisions_writesEscalatedAuditEntry() {
    policy.willReturn(new BreachDecision.Chained(
            new BreachDecision.EscalateTo(Set.of(), null),
            new BreachDecision.EscalateTo(Set.of(), null)));
    final WorkItem wi = expiredItem();
    service.checkExpired();
    assertThat(auditStore.findByWorkItemId(wi.id))
            .anyMatch(e -> "ESCALATED".equals(e.event) && "system".equals(e.actor));
}

@Test
void checkExpired_withChainedExhausted_firesExhaustedSlaBreachEvent() {
    policy.willReturn(new BreachDecision.Chained(
            new BreachDecision.EscalateTo(Set.of(), null),
            new BreachDecision.EscalateTo(Set.of(), null)));
    final WorkItem wi = expiredItem();
    service.checkExpired();
    assertThat(breachEvents).hasSize(1);
    assertThat(breachEvents.get(0).decision()).isInstanceOf(BreachDecision.Exhausted.class);
}

@Test
void checkExpired_withNonChainedEmptyEscalateTo_skipsItemAndWritesMisconfiguredAudit() {
    // A non-Chained EscalateTo(empty) should skip the item (no state change) and write audit
    policy.willReturn(new BreachDecision.EscalateTo(Set.of(), null));
    final WorkItem wi = expiredItem();
    final WorkItemStatus originalStatus = wi.status;
    service.checkExpired();
    // Item stays in original status
    assertThat(store.get(wi.id).orElseThrow().status).isEqualTo(originalStatus);
    // Audit entry written for observability
    assertThat(auditStore.findByWorkItemId(wi.id))
            .anyMatch(e -> "BREACH_POLICY_MISCONFIGURED".equals(e.event));
}

// ── #244: SLA_REASSIGNED rename ──────────────────────────────────────────

@Test
void checkExpired_withEscalateToDecision_writesSlaReassignedAuditEntry() {
    policy.willReturn(BreachDecision.EscalateTo.to("escalation-group"));
    final WorkItem wi = expiredItem();
    service.checkExpired();
    assertThat(auditStore.findByWorkItemId(wi.id))
            .anyMatch(e -> "SLA_REASSIGNED".equals(e.event));
}

@Test
void checkExpired_withEscalateToDecision_doesNotWriteEscalatedAuditEntry() {
    // "ESCALATED" audit must only appear when item reaches ESCALATED terminal status
    policy.willReturn(BreachDecision.EscalateTo.to("escalation-group"));
    final WorkItem wi = expiredItem();
    service.checkExpired();
    assertThat(auditStore.findByWorkItemId(wi.id))
            .noneMatch(e -> "ESCALATED".equals(e.event));
}
```

Add `import java.util.Set;` to the test imports if not already present.

- [ ] **Step 3.2: Run new tests to verify they fail**

```bash
scripts/mvn-test runtime -Dtest=ExpiryLifecycleServiceTest
```

Expected: multiple FAIL — the `ESCALATED` tests fail because Chained with empty EscalateTo doesn't set ESCALATED yet, and `SLA_REASSIGNED` tests fail because the event is still named `"ESCALATED"`.

- [ ] **Step 3.3: Implement Option B in ExpiryLifecycleService**

In `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java`:

**3.3a** — Add `BreachDecision.Exhausted` import at top:
```java
import io.casehub.work.api.BreachDecision;
```
(already imported — just confirming)

**3.3b** — Update `executeEscalateTo` to throw `BreachExecutionFailed` when groups empty (remove old inline LOG+Fail guard):

Replace the current `executeEscalateTo` method body opening block:
```java
// OLD — remove these lines from executeBreachDecision EscalateTo case:
if (escalate.groups().isEmpty()) {
    LOG.errorf("SlaBreachPolicy returned EscalateTo with empty groups for WorkItem %s" +
            " — applying Fail to avoid silent transaction rollback", item.id);
    yield executeFail(item, new BreachDecision.Fail("escalation-misconfigured"), now);
}
yield executeEscalateTo(item, escalate, ctx, now);
```

The `executeBreachDecision` EscalateTo case becomes simply:
```java
case BreachDecision.EscalateTo escalate -> {
    yield executeEscalateTo(item, escalate, ctx, now);
}
```

And add the guard to `executeEscalateTo` itself (at the very top, before any mutations):
```java
private BreachDecision.EscalateTo executeEscalateTo(
        final WorkItem item, final BreachDecision.EscalateTo escalate,
        final SlaBreachContext ctx, final Instant now) {
    if (escalate.groups().isEmpty()) {
        LOG.errorf("SlaBreachPolicy EscalateTo has empty groups for WorkItem %s — treating as policy failure",
                item.id);
        throw new BreachExecutionFailed("EscalateTo returned empty groups");
    }
    item.candidateGroups = String.join(",", escalate.groups());
    item.assigneeId = null;
    item.status = WorkItemStatus.PENDING;
    // ... rest of method unchanged
```

**3.3c** — Rename audit/lifecycle event in `executeEscalateTo` from `"ESCALATED"` to `"SLA_REASSIGNED"`:
```java
workItemStore.put(item);
writeAudit(item, "SLA_REASSIGNED", null, now);
fireLifecycleEvent("SLA_REASSIGNED", item);
return escalate;
```

**3.3d** — Update `executeBreachDecision` Chained case to handle double-failure and Exhausted:
```java
case BreachDecision.Chained chained -> {
    try {
        yield executeBreachDecision(item, chained.primary(), ctx, now);
    } catch (final BreachExecutionFailed e) {
        try {
            yield executeBreachDecision(item, chained.fallback(), ctx, now);
        } catch (final BreachExecutionFailed e2) {
            yield executeExhausted(item, "policy-exhausted", now);
        }
    }
}
case BreachDecision.Exhausted exhausted -> {
    yield executeExhausted(item, exhausted.reason(), now);
}
```

**3.3e** — Add `executeExhausted` private method:
```java
private BreachDecision.Exhausted executeExhausted(final WorkItem item, final String reason, final Instant now) {
    item.status = WorkItemStatus.ESCALATED;
    item.completedAt = now;
    workItemStore.put(item);
    writeAudit(item, "ESCALATED", reason, now);
    fireLifecycleEvent("ESCALATED", item);
    return new BreachDecision.Exhausted(reason);
}
```

**3.3f** — Wrap item processing in `checkExpired()` with `BreachExecutionFailed` catch:
```java
@Transactional
public void checkExpired() {
    final Instant now = Instant.now();
    for (final WorkItem item : workItemStore.scan(WorkItemQuery.expired(now))) {
        try {
            final SlaBreachContext ctx = buildBreachContext(item, BreachType.COMPLETION_EXPIRED, now);
            final BreachDecision leaf = executeBreachDecision(item, slaBreachPolicy.onBreach(ctx), ctx, now);
            slaBreachEventBus.fire(new SlaBreachEvent(ctx, leaf));
        } catch (final BreachExecutionFailed e) {
            LOG.errorf("SLA breach policy misconfigured for WorkItem %s — skipping this tick: %s",
                    item.id, e.getMessage());
            final AuditEntry entry = new AuditEntry();
            entry.workItemId = item.id;
            entry.event = "BREACH_POLICY_MISCONFIGURED";
            entry.actor = "system";
            entry.detail = e.getMessage();
            entry.occurredAt = now;
            auditStore.append(entry);
        }
    }
}
```

**3.3g** — Same `BreachExecutionFailed` wrap in `checkClaimDeadlines()`:
```java
@Transactional
public void checkClaimDeadlines() {
    final Instant now = Instant.now();
    for (final WorkItem item : workItemStore.scan(WorkItemQuery.claimExpired(now))) {
        try {
            if (item.lastReturnedToPoolAt != null) {
                item.accumulatedUnclaimedSeconds += Duration.between(item.lastReturnedToPoolAt, now).toSeconds();
            }
            item.lastReturnedToPoolAt = now;
            fireLifecycleEvent("CLAIM_EXPIRED", item);
            final SlaBreachContext ctx = buildBreachContext(item, BreachType.CLAIM_EXPIRED, now);
            final BreachDecision leaf = executeBreachDecision(item, slaBreachPolicy.onBreach(ctx), ctx, now);
            slaBreachEventBus.fire(new SlaBreachEvent(ctx, leaf));
        } catch (final BreachExecutionFailed e) {
            LOG.errorf("SLA breach policy misconfigured for WorkItem %s (claim) — skipping: %s",
                    item.id, e.getMessage());
            final AuditEntry entry = new AuditEntry();
            entry.workItemId = item.id;
            entry.event = "BREACH_POLICY_MISCONFIGURED";
            entry.actor = "system";
            entry.detail = e.getMessage();
            entry.occurredAt = now;
            auditStore.append(entry);
        }
    }
}
```

- [ ] **Step 3.4: Update stale test assertion**

In `ExpiryLifecycleServiceTest.java`, update the existing test `checkExpired_withEscalateToDecision_writesEscalatedAuditEntry` — it now asserts `"ESCALATED"` which is wrong post-rename. Change it to assert `"SLA_REASSIGNED"` instead (or delete it since `checkExpired_withEscalateToDecision_writesSlaReassignedAuditEntry` covers this). Delete the old test and keep the new one.

- [ ] **Step 3.5: Run all ExpiryLifecycleService tests**

```bash
scripts/mvn-test runtime -Dtest=ExpiryLifecycleServiceTest
```

Expected: all pass.

- [ ] **Step 3.6: Run full runtime tests**

```bash
scripts/mvn-test runtime
```

Expected: all pass.

- [ ] **Step 3.7: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java \
        runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java
git commit -m "feat(#244): implement ESCALATED terminal status via Chained policy exhaustion

- BreachExecutionFailed is now thrown from executeEscalateTo when groups is empty
  (before any state mutation), enabling the Chained fallback mechanism
- Chained handler: if both primary and fallback throw BreachExecutionFailed,
  calls executeExhausted() which sets WorkItemStatus.ESCALATED (terminal)
- Renames executeEscalateTo audit/lifecycle event from ESCALATED to SLA_REASSIGNED
  to distinguish reassignment (item still active) from terminal escalation
- Non-Chained empty-groups EscalateTo: item skipped for the tick, audit entry
  BREACH_POLICY_MISCONFIGURED written; batch not rolled back
- checkExpired() and checkClaimDeadlines() both wrapped with item-level catch

Refs #244, Refs #92"
```

---

## Task 4: API additions for delegation decline (#245)

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/AssignmentTrigger.java`
- Create: `api/src/main/java/io/casehub/work/api/DeclineTarget.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/DelegateRequest.java`

- [ ] **Step 4.1: Add DELEGATION_DECLINED to AssignmentTrigger**

In `api/src/main/java/io/casehub/work/api/AssignmentTrigger.java`:

```java
public enum AssignmentTrigger {
    /** WorkItem first created and persisted. */
    CREATED,
    /** WorkItem returned to pool by its assignee (release). */
    RELEASED,
    /** WorkItem delegated to a different pool or individual. */
    DELEGATED,
    /** WorkItem returned to an escalation pool after an SLA breach. */
    SLA_ESCALATED,
    /** WorkItem returned to pool after a decline of a targeted delegation. */
    DELEGATION_DECLINED
}
```

- [ ] **Step 4.2: Create DeclineTarget.java in casehub-work-api**

Create `api/src/main/java/io/casehub/work/api/DeclineTarget.java`:

```java
package io.casehub.work.api;

import java.util.Locale;

import io.casehub.platform.api.preferences.PreferenceKey;
import io.casehub.platform.api.preferences.SingleValuePreference;

/**
 * Determines where a delegated {@link io.casehub.work.runtime.model.WorkItem} returns
 * when the delegatee declines it.
 *
 * <p>Configure via preference key {@link #KEY} at any scope supported by
 * {@link io.casehub.platform.api.preferences.PreferenceProvider}. Override per
 * WorkItem at delegation time via {@link DelegateRequest#declineTarget()}.
 */
public enum DeclineTarget implements SingleValuePreference {

    /** Item returns to the general pool with candidateGroups unchanged. Default. */
    POOL,

    /** Item returns to the actor who delegated it (last entry in delegationChain). */
    DELEGATOR;

    /** Preference key for the scope-level default. Qualified name: {@code casehub.work.delegation.decline-target}. */
    public static final PreferenceKey<DeclineTarget> KEY =
            new PreferenceKey<>("casehub.work.delegation", "decline-target", POOL,
                    s -> DeclineTarget.valueOf(s.toUpperCase(Locale.ROOT)));
}
```

- [ ] **Step 4.3: Update DelegateRequest**

Replace `runtime/src/main/java/io/casehub/work/runtime/api/DelegateRequest.java`:

```java
package io.casehub.work.runtime.api;

import io.casehub.work.api.DeclineTarget;

/**
 * Request body for {@code PUT /workitems/{id}/delegate}.
 *
 * @param to the target actor to delegate to (required)
 * @param declineTarget optional instance-level override for where the item returns
 *        if the delegatee declines; null = use scope preference (default: POOL)
 */
public record DelegateRequest(String to, DeclineTarget declineTarget) {
}
```

- [ ] **Step 4.4: Publish api module**

```bash
scripts/mvn-install api
```

Expected: BUILD SUCCESS.

- [ ] **Step 4.5: Compile runtime**

```bash
scripts/mvn-compile runtime
```

Expected: BUILD SUCCESS.

- [ ] **Step 4.6: Commit**

```bash
git add api/src/main/java/io/casehub/work/api/AssignmentTrigger.java \
        api/src/main/java/io/casehub/work/api/DeclineTarget.java \
        runtime/src/main/java/io/casehub/work/runtime/api/DelegateRequest.java
git commit -m "feat(#245): add AssignmentTrigger.DELEGATION_DECLINED, DeclineTarget preference key, update DelegateRequest

DeclineTarget (POOL/DELEGATOR) is a SingleValuePreference with key
casehub.work.delegation.decline-target (default POOL). DelegateRequest gains
an optional declineTarget field for instance-level overrides.

Refs #245, Refs #92"
```

---

## Task 5: Drop DelegationState, add delegationDeclineTarget field, and V34 migration (#245)

**Files:**
- Create: `runtime/src/main/resources/db/work/migration/V34__delegation_state_drop_and_decline_target.sql`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`
- Delete: `runtime/src/main/java/io/casehub/work/runtime/model/DelegationState.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResponse.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemWithAuditResponse.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemMapper.java`
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java`

- [ ] **Step 5.1: Create V34 Flyway migration**

Create `runtime/src/main/resources/db/work/migration/V34__delegation_state_drop_and_decline_target.sql`:

```sql
-- V34: Replace DelegationState column with typed delegation decline target
-- delegation_state was PENDING/RESOLVED but was never transitioned to RESOLVED.
-- WorkItemStatus.DELEGATED now carries the pre-acceptance semantic directly.
ALTER TABLE work_item DROP COLUMN delegation_state;
ALTER TABLE work_item ADD COLUMN delegation_decline_target VARCHAR(10);
```

- [ ] **Step 5.2: Update WorkItem entity**

In `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`:

Remove these lines (the old DelegationState field):
```java
import io.casehub.work.runtime.model.DelegationState;
// ...
@Enumerated(EnumType.STRING)
@Column(name = "delegation_state")
public DelegationState delegationState;
```

Add the new field after the delegationChain field (around line 137):
```java
/**
 * Instance-level override for where this item returns if a delegation is declined.
 * Null = use scope preference (casehub.work.delegation.decline-target, default POOL).
 * Set by delegate() from DelegateRequest.declineTarget(); cleared after accept/decline.
 */
@Enumerated(EnumType.STRING)
@Column(name = "delegation_decline_target", length = 10)
public DeclineTarget delegationDeclineTarget;
```

Add import at top:
```java
import io.casehub.work.api.DeclineTarget;
```

Also update the delegationChain field comment (it currently says "JSON-serialised" which is wrong):
```java
/**
 * Comma-separated list of actorIds who have delegated this item (most recent delegator last).
 * Format is CSV; actorIds are UUIDs (hyphens only, no commas) so the format is unambiguous.
 * Structured delegation history (timestamps per hop) is deferred to #240.
 */
@Column(name = "delegation_chain")
public String delegationChain;
```

- [ ] **Step 5.3: Delete DelegationState.java**

```bash
rm /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/model/DelegationState.java
```

- [ ] **Step 5.4: Update WorkItemResponse.java**

Replace the `DelegationState delegationState` field with `DeclineTarget delegationDeclineTarget`. The full record signature update:

Remove:
```java
import io.casehub.work.runtime.model.DelegationState;
// ...
DelegationState delegationState,
```

Add after `createdBy`:
```java
import io.casehub.work.api.DeclineTarget;
// ...
DeclineTarget delegationDeclineTarget,
```

The record parameter order changes: `delegationState` → `delegationDeclineTarget`. Both are in the same position in the positional record.

- [ ] **Step 5.5: Update WorkItemWithAuditResponse.java**

Same change as Step 5.4 — replace `DelegationState delegationState` with `DeclineTarget delegationDeclineTarget`.

- [ ] **Step 5.6: Update WorkItemMapper.java**

In `toResponse()`, change:
```java
wi.createdBy, wi.delegationState, wi.delegationChain,
```
to:
```java
wi.createdBy, wi.delegationDeclineTarget, wi.delegationChain,
```

Same change in `toWithAudit()`.

Remove:
```java
import io.casehub.work.runtime.model.DelegationState;
```
(No longer needed — DelegationState is deleted.)

- [ ] **Step 5.7: Update MongoWorkItemDocument.java**

In `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemDocument.java`:

Remove:
```java
import io.casehub.work.runtime.model.DelegationState;
// ...
public String delegationState;
// ...
doc.delegationState = wi.delegationState != null ? wi.delegationState.name() : null;
// ...
wi.delegationState = delegationState != null ? DelegationState.valueOf(delegationState) : null;
```

Add:
```java
import io.casehub.work.api.DeclineTarget;
// ...
public String delegationDeclineTarget;
// ...
doc.delegationDeclineTarget = wi.delegationDeclineTarget != null ? wi.delegationDeclineTarget.name() : null;
// ...
wi.delegationDeclineTarget = delegationDeclineTarget != null ? DeclineTarget.valueOf(delegationDeclineTarget) : null;
```

- [ ] **Step 5.8: Update WorkItemService.java — remove DelegationState import and usage**

In `WorkItemService.java`, remove:
```java
import io.casehub.work.runtime.model.DelegationState;
// ...
item.delegationState = DelegationState.PENDING;  // in delegate()
```

(The rest of delegate() will be updated in Task 7.)

- [ ] **Step 5.9: Update WorkItemServiceTest.java — remove DelegationState import and stale test**

In `WorkItemServiceTest.java`:

Remove:
```java
import io.casehub.work.runtime.model.DelegationState;
```

Remove or update the test `delegate_setsDelegationStatePending` — it tests removed behaviour. Delete it (we'll replace it in Task 7 with `delegate_setsStatusDelegated`).

- [ ] **Step 5.10: Compile runtime**

```bash
scripts/mvn-compile runtime
```

Expected: BUILD SUCCESS. If any compilation errors remain, they will point to other import sites of `DelegationState` — fix each one by removing the reference.

- [ ] **Step 5.11: Run tests**

```bash
scripts/mvn-test runtime
```

Expected: all pass. The removed `delegate_setsDelegationStatePending` test is gone; the remaining delegate tests still pass since `delegate()` still has its other logic.

- [ ] **Step 5.12: Commit**

```bash
git add -A
git commit -m "feat(#245): drop DelegationState, add delegationDeclineTarget field, V34 migration

Breaking API change (acceptable for 0.2-SNAPSHOT): WorkItemResponse and
WorkItemWithAuditResponse lose delegationState, gain delegationDeclineTarget.
WorkItemStatus.DELEGATED now carries the pre-acceptance semantic directly.

V34 migration: DROP delegation_state, ADD delegation_decline_target VARCHAR(10).

Refs #245, Refs #92"
```

---

## Task 6: WorkItemQuery.expired() — add DELEGATED (#245)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemQuery.java`

- [ ] **Step 6.1: Write failing test**

Add to `ExpiryLifecycleServiceTest.java`:

```java
@Test
void checkExpired_withDelegatedItem_isPickedUpByExpiryJob() {
    policy.willReturn(new BreachDecision.Fail("sla-breached"));
    final WorkItem wi = expiredItem();
    wi.status = WorkItemStatus.DELEGATED;
    wi.assigneeId = "bob";
    store.put(wi);
    service.checkExpired();
    assertThat(store.get(wi.id).orElseThrow().status).isEqualTo(WorkItemStatus.EXPIRED);
}
```

- [ ] **Step 6.2: Run to verify it fails**

```bash
scripts/mvn-test runtime -Dtest=ExpiryLifecycleServiceTest#checkExpired_withDelegatedItem_isPickedUpByExpiryJob
```

Expected: FAIL — DELEGATED item not picked up (TestStore scan excludes it because DELEGATED is not in the hardcoded statusIn list).

Actually, the TestStore scan uses `!wi.status.isTerminal()` not a statusIn list. DELEGATED is not terminal so the TestStore WOULD pick it up. The real JPA query will also pick it up after we add DELEGATED to `statusIn`. So the test may already pass with the TestStore.

Run the test regardless to check:

```bash
scripts/mvn-test runtime -Dtest=ExpiryLifecycleServiceTest#checkExpired_withDelegatedItem_isPickedUpByExpiryJob
```

If it passes: proceed. If it fails for another reason: investigate.

- [ ] **Step 6.3: Update WorkItemQuery.expired()**

In `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemQuery.java`, update the `expired()` factory:

```java
public static WorkItemQuery expired(final Instant now) {
    return new Builder()
            .expiresAtOrBefore(now)
            .statusIn(List.of(
                    WorkItemStatus.PENDING,
                    WorkItemStatus.ASSIGNED,
                    WorkItemStatus.IN_PROGRESS,
                    WorkItemStatus.SUSPENDED,
                    WorkItemStatus.DELEGATED))
            .build();
}
```

- [ ] **Step 6.4: Run tests**

```bash
scripts/mvn-test runtime
```

Expected: all pass.

- [ ] **Step 6.5: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemQuery.java \
        runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java
git commit -m "feat(#245): include DELEGATED status in WorkItemQuery.expired()

DELEGATED items can expire — the SLA is a team obligation that does not pause
during internal hand-offs. If an item expires while DELEGATED, the expiry job
processes it normally; subsequent accept-delegation fails with status guard.

Refs #245, Refs #92"
```

---

## Task 7: WorkItemStatus.isActive() — add DELEGATED (#245)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/model/WorkItemStatusTest.java`

- [ ] **Step 7.1: Write failing test**

Add to `WorkItemStatusTest.java`:

```java
@Test
void delegated_isActive() {
    assertThat(WorkItemStatus.DELEGATED.isActive()).isTrue();
}
```

- [ ] **Step 7.2: Run to verify it fails**

```bash
scripts/mvn-test runtime -Dtest=WorkItemStatusTest#delegated_isActive
```

Expected: FAIL.

- [ ] **Step 7.3: Add DELEGATED to isActive()**

In `WorkItemStatus.java`, update `isActive()`:

```java
public boolean isActive() {
    return switch (this) {
        case PENDING, ASSIGNED, IN_PROGRESS, SUSPENDED, DELEGATED -> true;
        default -> false;
    };
}
```

- [ ] **Step 7.4: Run tests**

```bash
scripts/mvn-test runtime -Dtest=WorkItemStatusTest
```

Expected: all pass.

- [ ] **Step 7.5: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/model/WorkItemStatus.java \
        runtime/src/test/java/io/casehub/work/runtime/model/WorkItemStatusTest.java
git commit -m "feat(#245): add DELEGATED to WorkItemStatus.isActive()

DELEGATED is a live pre-acceptance state — the WorkItem requires action from
the delegatee. It is not terminal (can transition to ASSIGNED or PENDING) and
is now correctly classified as active.

Refs #245, Refs #92"
```

---

## Task 8: DELEGATED lifecycle in WorkItemService (#245)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java`

- [ ] **Step 8.1: Write failing tests for new behaviour**

In `WorkItemServiceTest.java`:

First, add `PreferenceProvider` import and inject it into setUp. The `WorkItemService` will need a `preferenceProvider` field (added in Step 8.2). In setUp, add after service construction:

```java
import io.casehub.platform.api.preferences.MapPreferences;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.work.api.DeclineTarget;
// ...

@BeforeEach
void setUp() {
    // ... existing setup ...
    service.preferenceProvider = scope -> new MapPreferences(Map.of()); // → POOL by default
}
```

Add tests:

```java
// ── delegate() produces DELEGATED status ──────────────────────────────────

@Test
void delegate_setsStatusDelegated() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    wi = service.delegate(wi.id, "alice", "bob");
    assertThat(wi.status).isEqualTo(WorkItemStatus.DELEGATED);
}

@Test
void delegate_setsAssigneeToBob() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    wi = service.delegate(wi.id, "alice", "bob");
    assertThat(wi.assigneeId).isEqualTo("bob");
}

@Test
void delegate_clearsClaimDeadline() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    wi = service.delegate(wi.id, "alice", "bob");
    assertThat(wi.claimDeadline).isNull();
}

@Test
void delegate_clearsLastReturnedToPoolAt() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    wi = service.delegate(wi.id, "alice", "bob");
    assertThat(wi.lastReturnedToPoolAt).isNull();
}

@Test
void delegate_withDeclineTarget_storesDeclineTarget() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    wi = service.delegate(wi.id, "alice", "bob", DeclineTarget.DELEGATOR);
    assertThat(wi.delegationDeclineTarget).isEqualTo(DeclineTarget.DELEGATOR);
}

@Test
void delegate_withNullDeclineTarget_leavesDeclineTargetNull() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    wi = service.delegate(wi.id, "alice", "bob", null);
    assertThat(wi.delegationDeclineTarget).isNull();
}

// ── acceptDelegation() ────────────────────────────────────────────────────

@Test
void acceptDelegation_transitionsToAssigned() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", null);
    wi = service.acceptDelegation(wi.id, "bob");
    assertThat(wi.status).isEqualTo(WorkItemStatus.ASSIGNED);
    assertThat(wi.assigneeId).isEqualTo("bob");
}

@Test
void acceptDelegation_setsAssignedAt() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", null);
    wi = service.acceptDelegation(wi.id, "bob");
    assertThat(wi.assignedAt).isNotNull();
}

@Test
void acceptDelegation_clearsDeclineTarget() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", DeclineTarget.DELEGATOR);
    wi = service.acceptDelegation(wi.id, "bob");
    assertThat(wi.delegationDeclineTarget).isNull();
}

@Test
void acceptDelegation_wrongActor_throws() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", null);
    assertThatThrownBy(() -> service.acceptDelegation(wi.id, "charlie"))
            .isInstanceOf(IllegalStateException.class);
}

@Test
void acceptDelegation_notDelegated_throws() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    assertThatThrownBy(() -> service.acceptDelegation(wi.id, "alice"))
            .isInstanceOf(IllegalStateException.class);
}

// ── declineDelegation() — POOL path ──────────────────────────────────────

@Test
void declineDelegation_poolPath_transitionsToPending() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", DeclineTarget.POOL);
    wi = service.declineDelegation(wi.id, "bob");
    assertThat(wi.status).isEqualTo(WorkItemStatus.PENDING);
    assertThat(wi.assigneeId).isNull();
}

@Test
void declineDelegation_poolPath_setsLastReturnedToPoolAt() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", DeclineTarget.POOL);
    wi = service.declineDelegation(wi.id, "bob");
    assertThat(wi.lastReturnedToPoolAt).isNotNull();
}

@Test
void declineDelegation_poolPath_recomputesClaimDeadline() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", DeclineTarget.POOL);
    wi = service.declineDelegation(wi.id, "bob");
    assertThat(wi.claimDeadline).isNotNull();
}

@Test
void declineDelegation_poolPath_clearsDeclineTarget() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", DeclineTarget.POOL);
    wi = service.declineDelegation(wi.id, "bob");
    assertThat(wi.delegationDeclineTarget).isNull();
}

// ── declineDelegation() — DELEGATOR path ─────────────────────────────────

@Test
void declineDelegation_delegatorPath_returnsToOriginalActor() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", DeclineTarget.DELEGATOR);
    wi = service.declineDelegation(wi.id, "bob");
    assertThat(wi.status).isEqualTo(WorkItemStatus.ASSIGNED);
    assertThat(wi.assigneeId).isEqualTo("alice");
}

// ── declineDelegation() — preference default ─────────────────────────────

@Test
void declineDelegation_usesPreferenceWhenNoInstanceOverride() {
    // Default preference is POOL → item returns to pool
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", null); // no instance override
    wi = service.declineDelegation(wi.id, "bob");
    assertThat(wi.status).isEqualTo(WorkItemStatus.PENDING); // POOL behaviour
}

// ── declineDelegation() — error cases ────────────────────────────────────

@Test
void declineDelegation_wrongActor_throws() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    service.delegate(wi.id, "alice", "bob", null);
    assertThatThrownBy(() -> service.declineDelegation(wi.id, "charlie"))
            .isInstanceOf(IllegalStateException.class);
}

@Test
void declineDelegation_notDelegated_throws() {
    WorkItem wi = service.create(basicRequest());
    service.claim(wi.id, "alice");
    assertThatThrownBy(() -> service.declineDelegation(wi.id, "alice"))
            .isInstanceOf(IllegalStateException.class);
}
```

Also update the stale test `delegate_setsNewAssigneeAndStatusPending`:

```java
// BEFORE (delete this):
void delegate_setsNewAssigneeAndStatusPending() {
    // ...
    assertThat(wi.status).isEqualTo(WorkItemStatus.PENDING);
}

// AFTER (delegate_setsStatusDelegated already added above — delete the old test)
```

- [ ] **Step 8.2: Run to verify failures**

```bash
scripts/mvn-test runtime -Dtest=WorkItemServiceTest
```

Expected: new tests FAIL — `delegate_setsStatusDelegated` fails (still returns PENDING), `acceptDelegation_*` and `declineDelegation_*` fail (methods don't exist).

- [ ] **Step 8.3: Implement delegate() changes in WorkItemService**

Add `@Inject` field for `PreferenceProvider` in `WorkItemService` (near the other `@Inject` fields, after `lifecycleEvent`):

```java
@Inject
io.casehub.platform.api.preferences.PreferenceProvider preferenceProvider;
```

Add imports:
```java
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.SettingsScope;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.path.Path;
import io.casehub.work.api.DeclineTarget;
```

Update `delegate()` signature and body:

```java
@Transactional
public WorkItem delegate(final UUID id, final String actorId, final String toAssigneeId,
        final DeclineTarget declineTarget) {
    final WorkItem item = requireWorkItem(id);
    if (item.status != WorkItemStatus.ASSIGNED && item.status != WorkItemStatus.IN_PROGRESS) {
        throw new IllegalStateException("Cannot delegate WorkItem in status: " + item.status);
    }
    final PolicyDecision delegateDecision = exclusionPolicy.check(toAssigneeId, item.excludedUsers);
    if (delegateDecision.denied()) {
        blockedAuditService.record(item.id, "DELEGATE_DENIED", actorId,
                "target:" + toAssigneeId + "; reason:" + delegateDecision.reason());
        throw new IllegalArgumentException(delegateDecision.reason());
    }
    if (item.owner == null) {
        item.owner = actorId;
    }
    item.delegationChain = item.delegationChain == null
            ? actorId
            : item.delegationChain + "," + actorId;

    // Fire strategy first while item still reflects current load
    assignmentService.assign(item, AssignmentTrigger.DELEGATED);

    // If strategy did not select a candidate, fall back to explicit target
    if (item.assigneeId == null || item.assigneeId.equals(actorId)) {
        item.assigneeId = toAssigneeId;
    }

    // DELEGATED unconditionally — overrides any ASSIGNED set by strategy
    item.status = WorkItemStatus.DELEGATED;
    // DELEGATED is directly addressed — pool SLA tracking does not apply
    item.claimDeadline = null;
    item.lastReturnedToPoolAt = null;
    // Store instance-level decline target override (null = use scope preference)
    item.delegationDeclineTarget = declineTarget;

    final WorkItem saved = workItemStore.put(item);
    audit(saved.id, "DELEGATED", actorId, "to:" + saved.assigneeId);
    if (lifecycleEvent != null) {
        lifecycleEvent.fire(WorkItemLifecycleEvent.of("DELEGATED", saved, actorId, "to:" + toAssigneeId));
    }
    return saved;
}
```

Remove any remaining reference to `buildClaimSlaContext` from the delegate path (it's no longer called here).

- [ ] **Step 8.4: Add acceptDelegation() to WorkItemService**

Add after the `delegate()` method:

```java
@Transactional
public WorkItem acceptDelegation(final UUID id, final String claimantId) {
    final WorkItem item = requireWorkItem(id);
    if (item.status != WorkItemStatus.DELEGATED) {
        throw new IllegalStateException(
                "Cannot accept delegation for WorkItem in status: " + item.status);
    }
    if (!claimantId.equals(item.assigneeId)) {
        throw new IllegalStateException(
                "Actor '" + claimantId + "' is not the designated delegatee for WorkItem " + id);
    }
    item.status = WorkItemStatus.ASSIGNED;
    item.assignedAt = Instant.now();
    item.delegationDeclineTarget = null;
    final WorkItem saved = workItemStore.put(item);
    audit(saved.id, "DELEGATION_ACCEPTED", claimantId, null);
    if (lifecycleEvent != null) {
        lifecycleEvent.fire(WorkItemLifecycleEvent.of("DELEGATION_ACCEPTED", saved, claimantId, null));
    }
    return saved;
}
```

- [ ] **Step 8.5: Add declineDelegation() to WorkItemService**

Add after `acceptDelegation()`:

```java
@Transactional
public WorkItem declineDelegation(final UUID id, final String actorId) {
    final WorkItem item = requireWorkItem(id);
    if (item.status != WorkItemStatus.DELEGATED) {
        throw new IllegalStateException(
                "Cannot decline delegation for WorkItem in status: " + item.status);
    }
    if (!actorId.equals(item.assigneeId)) {
        throw new IllegalStateException(
                "Actor '" + actorId + "' is not the designated delegatee for WorkItem " + id);
    }

    final DeclineTarget target = resolveDeclineTarget(item);
    item.delegationDeclineTarget = null;

    if (target == DeclineTarget.DELEGATOR && item.delegationChain != null) {
        final String[] chain = item.delegationChain.split(",");
        final String prevActor = chain[chain.length - 1].trim();
        // Restore to previous actor — no exclusion check: prevActor was a verified holder.
        item.assigneeId = prevActor;
        item.status = WorkItemStatus.ASSIGNED;
        item.assignedAt = Instant.now();
    } else {
        // POOL path
        item.assigneeId = null;
        item.status = WorkItemStatus.PENDING;
        final Instant now = Instant.now();
        item.lastReturnedToPoolAt = now;
        item.claimDeadline = claimSlaPolicy.computePoolDeadline(buildClaimSlaContext(item, now));
        assignmentService.assign(item, AssignmentTrigger.DELEGATION_DECLINED);
    }

    final WorkItem saved = workItemStore.put(item);
    audit(saved.id, "DELEGATION_DECLINED", actorId, null);
    if (lifecycleEvent != null) {
        lifecycleEvent.fire(WorkItemLifecycleEvent.of("DELEGATION_DECLINED", saved, actorId, null));
    }
    return saved;
}

private DeclineTarget resolveDeclineTarget(final WorkItem item) {
    if (item.delegationDeclineTarget != null) {
        return item.delegationDeclineTarget;
    }
    final Path scopePath = item.scope != null ? Path.parse(item.scope) : Path.root();
    final Preferences prefs = preferenceProvider.resolve(new SettingsScope(scopePath, Instant.now()));
    return prefs.getOrDefault(DeclineTarget.KEY);
}
```

- [ ] **Step 8.6: Add findById() to WorkItemService**

Add after `findByCallerRef()`:

```java
public Optional<WorkItem> findById(final UUID id) {
    return workItemStore.get(id);
}
```

- [ ] **Step 8.7: Check if delegate() has overloads elsewhere and update callers**

```bash
grep -rn "\.delegate(" /Users/mdproctor/claude/casehub/work/runtime/src/ --include="*.java" | grep -v "test"
```

The `WorkItemResource.delegate()` calls `workItemService.delegate(id, actor, body.to())` — this 3-arg version is gone. Update it in Task 9.

- [ ] **Step 8.8: Run unit tests**

```bash
scripts/mvn-test runtime -Dtest=WorkItemServiceTest
```

Expected: all new tests pass. The stale `delegate_setsNewAssigneeAndStatusPending` was deleted; `delegate_setsStatusDelegated` passes.

- [ ] **Step 8.9: Run full runtime tests**

```bash
scripts/mvn-test runtime
```

Expected: mostly pass. Compilation errors in `WorkItemResource` (3-arg `delegate()` call) will prevent full pass — that's fixed in Task 9.

- [ ] **Step 8.10: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java \
        runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java
git commit -m "feat(#245): implement DELEGATED lifecycle — delegate(), acceptDelegation(), declineDelegation(), findById()

delegate() now sets WorkItemStatus.DELEGATED unconditionally (after strategy);
claimDeadline and lastReturnedToPoolAt are cleared while DELEGATED.
acceptDelegation() transitions DELEGATED → ASSIGNED.
declineDelegation() resolves POOL vs DELEGATOR target (instance override →
scope preference → POOL default); POOL returns item to pool with fresh claim
deadline; DELEGATOR restores previous actor from delegationChain.
findById() exposes read access through the service layer.

Refs #245, Refs #92"
```

---

## Task 9: REST endpoints for delegation lifecycle (#245)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResource.java`
- Create: `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemDelegationTest.java`

- [ ] **Step 9.1: Write failing REST tests**

Create `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemDelegationTest.java`:

```java
package io.casehub.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.nullValue;

import org.junit.jupiter.api.Test;

import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
@TestTransaction
class WorkItemDelegationTest {

    private String createAndClaimItem(final String claimant) {
        final String id = given()
                .contentType(ContentType.JSON)
                .body("""
                        {"title":"Delegation test","priority":"MEDIUM","createdBy":"system"}
                        """)
                .when().post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        given().contentType(ContentType.JSON)
                .when().put("/workitems/" + id + "/claim?claimant=" + claimant)
                .then().statusCode(200);

        return id;
    }

    // ── delegate endpoint ─────────────────────────────────────────────────────

    @Test
    void delegate_returns200WithDelegatedStatus() {
        final String id = createAndClaimItem("alice");

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"to":"bob"}
                        """)
                .when().put("/workitems/" + id + "/delegate?actor=alice")
                .then()
                .statusCode(200)
                .body("status", equalTo("DELEGATED"))
                .body("assigneeId", equalTo("bob"));
    }

    @Test
    void delegate_withDeclineTarget_storesTarget() {
        final String id = createAndClaimItem("alice");

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"to":"bob","declineTarget":"DELEGATOR"}
                        """)
                .when().put("/workitems/" + id + "/delegate?actor=alice")
                .then()
                .statusCode(200)
                .body("delegationDeclineTarget", equalTo("DELEGATOR"));
    }

    // ── GET /{id} — findById path ─────────────────────────────────────────────

    @Test
    void getById_returns200WithDelegatedStatus() {
        final String id = createAndClaimItem("alice");

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"to":"bob"}
                        """)
                .when().put("/workitems/" + id + "/delegate?actor=alice")
                .then().statusCode(200);

        given()
                .when().get("/workitems/" + id)
                .then()
                .statusCode(200)
                .body("item.status", equalTo("DELEGATED"));
    }

    // ── accept-delegation endpoint ────────────────────────────────────────────

    @Test
    void acceptDelegation_returns200WithAssignedStatus() {
        final String id = createAndClaimItem("alice");

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"to":"bob"}
                        """)
                .when().put("/workitems/" + id + "/delegate?actor=alice")
                .then().statusCode(200);

        given()
                .when().put("/workitems/" + id + "/accept-delegation?claimant=bob")
                .then()
                .statusCode(200)
                .body("status", equalTo("ASSIGNED"))
                .body("assigneeId", equalTo("bob"));
    }

    @Test
    void acceptDelegation_wrongActor_returns409() {
        final String id = createAndClaimItem("alice");

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"to":"bob"}
                        """)
                .when().put("/workitems/" + id + "/delegate?actor=alice")
                .then().statusCode(200);

        given()
                .when().put("/workitems/" + id + "/accept-delegation?claimant=charlie")
                .then().statusCode(409);
    }

    // ── decline-delegation endpoint ───────────────────────────────────────────

    @Test
    void declineDelegation_poolPath_returns200WithPendingStatus() {
        final String id = createAndClaimItem("alice");

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"to":"bob","declineTarget":"POOL"}
                        """)
                .when().put("/workitems/" + id + "/delegate?actor=alice")
                .then().statusCode(200);

        given()
                .when().put("/workitems/" + id + "/decline-delegation?actor=bob")
                .then()
                .statusCode(200)
                .body("status", equalTo("PENDING"))
                .body("assigneeId", nullValue());
    }

    @Test
    void declineDelegation_delegatorPath_returns200WithAssignedToPreviousActor() {
        final String id = createAndClaimItem("alice");

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"to":"bob","declineTarget":"DELEGATOR"}
                        """)
                .when().put("/workitems/" + id + "/delegate?actor=alice")
                .then().statusCode(200);

        given()
                .when().put("/workitems/" + id + "/decline-delegation?actor=bob")
                .then()
                .statusCode(200)
                .body("status", equalTo("ASSIGNED"))
                .body("assigneeId", equalTo("alice"));
    }

    @Test
    void declineDelegation_wrongActor_returns409() {
        final String id = createAndClaimItem("alice");

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"to":"bob"}
                        """)
                .when().put("/workitems/" + id + "/delegate?actor=alice")
                .then().statusCode(200);

        given()
                .when().put("/workitems/" + id + "/decline-delegation?actor=charlie")
                .then().statusCode(409);
    }
}
```

- [ ] **Step 9.2: Run to verify failures**

```bash
scripts/mvn-test runtime -Dtest=WorkItemDelegationTest
```

Expected: compilation FAIL (accept/decline-delegation endpoints don't exist; delegate() takes wrong args).

- [ ] **Step 9.3: Update WorkItemResource**

**9.3a** — Update `delegate()` to pass `declineTarget` from request body:

```java
@PUT
@Path("/{id}/delegate")
@Consumes(MediaType.APPLICATION_JSON)
public Response delegate(@PathParam("id") final UUID id,
        @QueryParam("actor") final String actor,
        final DelegateRequest body) {
    try {
        return Response.ok(WorkItemMapper.toResponse(
                workItemService.delegate(id, actor, body.to(), body.declineTarget()))).build();
    } catch (final IllegalArgumentException e) {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity(Map.of("error", e.getMessage()))
                .build();
    }
}
```

**9.3b** — Add `acceptDelegation` endpoint:

```java
@PUT
@Path("/{id}/accept-delegation")
public Response acceptDelegation(@PathParam("id") final UUID id,
        @QueryParam("claimant") final String claimant) {
    try {
        return Response.ok(WorkItemMapper.toResponse(
                workItemService.acceptDelegation(id, claimant))).build();
    } catch (final IllegalStateException e) {
        return Response.status(Response.Status.CONFLICT)
                .entity(Map.of("error", e.getMessage()))
                .build();
    }
}
```

**9.3c** — Add `declineDelegation` endpoint:

```java
@PUT
@Path("/{id}/decline-delegation")
public Response declineDelegation(@PathParam("id") final UUID id,
        @QueryParam("actor") final String actor) {
    try {
        return Response.ok(WorkItemMapper.toResponse(
                workItemService.declineDelegation(id, actor))).build();
    } catch (final IllegalStateException e) {
        return Response.status(Response.Status.CONFLICT)
                .entity(Map.of("error", e.getMessage()))
                .build();
    }
}
```

**9.3d** — Update `getById()` to use `workItemService.findById()`:

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

**9.3e** — Update existing `delegate_returns200WithNewAssignee` test in `WorkItemResourceTest.java`: the status is now `"DELEGATED"` not `"PENDING"`. Find and update:

```java
.body("status", equalTo("DELEGATED"))
```

- [ ] **Step 9.4: Update FilterRegistryEngine comment**

In `runtime/src/main/java/io/casehub/work/runtime/filter/FilterRegistryEngine.java`, find the `toFilterEvent()` method and update the comment on `FilterEvent.REMOVE` case if there is one. Also update `FilterEvent.java` if it has a Javadoc listing terminal statuses to include EXPIRED and ESCALATED.

Find `FilterEvent.java`:

```bash
find /Users/mdproctor/claude/casehub/work -name "FilterEvent.java" | head -3
```

Read and update the comment:

```java
/** WorkItem reached a terminal state (COMPLETED, REJECTED, CANCELLED, EXPIRED, ESCALATED) or was reassigned (DELEGATED events). */
REMOVE,
```

- [ ] **Step 9.5: Run REST tests**

```bash
scripts/mvn-test runtime -Dtest=WorkItemDelegationTest
```

Expected: all pass.

- [ ] **Step 9.6: Run full runtime tests**

```bash
scripts/mvn-test runtime
```

Expected: all pass.

- [ ] **Step 9.7: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResource.java \
        runtime/src/test/java/io/casehub/work/runtime/api/WorkItemDelegationTest.java \
        runtime/src/test/java/io/casehub/work/runtime/api/WorkItemResourceTest.java \
        runtime/src/main/java/io/casehub/work/runtime/filter/FilterRegistryEngine.java
git commit -m "feat(#245): add accept-delegation and decline-delegation REST endpoints; update getById() via service

PUT /workitems/{id}/accept-delegation?claimant=X → ASSIGNED
PUT /workitems/{id}/decline-delegation?actor=X → PENDING or ASSIGNED (per policy)
GET /workitems/{id} now routes through workItemService.findById() instead of
directly accessing WorkItemStore.

Refs #245, Refs #92"
```

---

## Task 10: Verify and close #239 (GroupMembershipProvider callers)

**No code changes needed.**

- [ ] **Step 10.1: Verify GroupMembershipProvider callers**

```bash
grep -rn "membersOf\|GroupMembership" /Users/mdproctor/claude/casehub/work/runtime/src/main/java/ --include="*.java"
```

Expected: only `TemplateExpander` (uses `m.actorId()` correctly) and `NoOpGroupMembershipProvider` (returns `Set<GroupMember>`).

```bash
scripts/mvn-compile runtime
```

Expected: BUILD SUCCESS (no compilation errors related to GroupMember types).

- [ ] **Step 10.2: Close issue**

```bash
gh issue close 239 -R casehubio/work --comment "All callers in casehub-work already use Set<GroupMember> correctly: TemplateExpander uses m.actorId(), NoOpGroupMembershipProvider returns Set<GroupMember>. No code changes needed — platform#45 migration was complete."
```

- [ ] **Step 10.3: Commit note**

No code change — no commit needed. The issue is closed.

---

## Task 11: findById in WorkItemService (#241)

`findById()` was already added to `WorkItemService` in Task 8.5 and `getById()` was updated in Task 9.3d. This task verifies and commits the issue closure.

- [ ] **Step 11.1: Verify findById test coverage**

Check that `WorkItemDelegationTest.getById_returns200WithDelegatedStatus` covers the `findById` → service layer path. If additional unit-level coverage is desired, add to `WorkItemServiceTest.java`:

```java
@Test
void findById_returnsItem() {
    WorkItem wi = service.create(basicRequest());
    assertThat(service.findById(wi.id)).isPresent().get()
            .extracting(w -> w.id).isEqualTo(wi.id);
}

@Test
void findById_unknownId_returnsEmpty() {
    assertThat(service.findById(UUID.randomUUID())).isEmpty();
}
```

- [ ] **Step 11.2: Run test**

```bash
scripts/mvn-test runtime -Dtest=WorkItemServiceTest#findById_returnsItem,WorkItemServiceTest#findById_unknownId_returnsEmpty
```

Expected: pass.

- [ ] **Step 11.3: Commit and close issue**

```bash
git add runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java
git commit -m "test(#241): add findById unit tests to WorkItemServiceTest

findById() delegates to WorkItemStore.get() — the implementation was part of
Task 8 (delegation lifecycle). Tests confirm Optional semantics.

Closes #241, Refs #92"
```

```bash
gh issue close 241 -R casehubio/work --comment "Implemented as part of issue-243-status-lifecycle-fixes branch."
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|-----------------|------|
| EXPIRED in isTerminal() | Task 1 |
| isTerminal() call sites — cancel/extend/completeFromSystem/rejectFromSystem/SpawnService/MultiInstance/InboxSummary/Metrics | Task 1 (all affected by enum change) |
| WorkItemQuery.expired() independence noted | Task 6 (adds DELEGATED) |
| BreachDecision.Exhausted | Task 2 |
| SLA_REASSIGNED rename | Task 3 |
| Chained exhaustion → ESCALATED | Task 3 |
| Non-Chained empty-groups → skip + audit | Task 3 |
| BreachExecutionFailed private note | Task 3 (comment in code) |
| DELEGATED in isActive() | Task 7 |
| DELEGATED items expire (WorkItemQuery.expired()) | Task 6 |
| Drop DelegationState | Task 5 |
| delegationDeclineTarget typed field | Task 5 |
| DeclineTarget preference key | Task 4 |
| AssignmentTrigger.DELEGATION_DECLINED | Task 4 |
| delegate() sets DELEGATED unconditionally after assign() | Task 8 |
| claimDeadline/lastReturnedToPoolAt cleared in delegate() | Task 8 |
| acceptDelegation() | Task 8 + Task 9 |
| declineDelegation() — POOL path | Task 8 + Task 9 |
| declineDelegation() — DELEGATOR path | Task 8 + Task 9 |
| DELEGATOR path no exclusion check (intentional) | Task 8 |
| @Transactional on new methods | Task 8 |
| Race → 409 via state guard | Task 9 |
| SettingsScope construction | Task 8 |
| V34 migration | Task 5 |
| MongoWorkItemDocument updated | Task 5 |
| FilterRegistryEngine comment | Task 9 |
| Breaking API change acknowledged | Task 5 commit message |
| #239 verify + close | Task 10 |
| #241 findById + REST endpoint | Task 8 + Task 9 + Task 11 |

**Placeholder scan:** none found.

**Type consistency:**
- `DeclineTarget` defined in Task 4, used in Tasks 5, 8, 9 — consistent.
- `WorkItemStatus.DELEGATED` referenced from Tasks 1, 6, 7, 8, 9 — same enum constant.
- `AssignmentTrigger.DELEGATION_DECLINED` defined in Task 4, used in Task 8 — consistent.
- `BreachDecision.Exhausted` defined in Task 2, handled in Task 3 — consistent.
- `executeExhausted(item, reason, now)` — 3-param signature used consistently in Task 3.
