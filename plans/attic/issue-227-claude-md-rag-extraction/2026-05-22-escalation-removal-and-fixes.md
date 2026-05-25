# Escalation Removal and Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove deprecated `EscalationPolicy` SPI and its implementations; wire WorkBroker auto-assignment after `EscalateTo` breach decisions; close the already-fixed `scanRoots` issue; sync parent docs.

**Architecture:** `SlaBreachPolicy` replaced `EscalationPolicy` in commit `66c4541`. This plan removes all old dead code and adds the one missing runtime behaviour: after an `EscalateTo` decision, the routing strategy runs automatically so the item is pre-assigned (or stays PENDING for claim-first) without manual intervention. The `writeAudit()` helper is extended to carry a nullable detail string, preserving the diagnostic information previously written by `AutoRejectEscalationPolicy`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ. All builds via `scripts/mvn-test <module>`.

---

## File Map

### Deleted (work#215)
- `api/src/main/java/io/casehub/work/api/EscalationPolicy.java`
- `api/src/test/java/io/casehub/work/api/EscalationPolicyTest.java`
- `runtime/src/main/java/io/casehub/work/runtime/service/AutoRejectEscalationPolicy.java`
- `runtime/src/main/java/io/casehub/work/runtime/service/ReassignEscalationPolicy.java`
- `runtime/src/main/java/io/casehub/work/runtime/service/NotifyEscalationPolicy.java`
- `runtime/src/main/java/io/casehub/work/runtime/service/EscalationPolicyProducer.java`
- `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryEscalation.java`
- `runtime/src/main/java/io/casehub/work/runtime/service/ClaimEscalation.java`
- `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemExpiredEvent.java`
- `runtime/src/test/java/io/casehub/work/runtime/service/EscalationPolicyTest.java`
- `runtime/src/test/java/io/casehub/work/runtime/service/NotifyEscalationPolicyTest.java`

### Modified (work#215)
- `runtime/src/main/java/io/casehub/work/runtime/config/WorkItemsConfig.java` — remove 2 methods
- `runtime/src/test/java/io/casehub/work/runtime/config/WorkItemsConfigDefaultsTest.java` — remove 2 tests
- `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java` — remove overrides from 2 anonymous `WorkItemsConfig` impls
- `runtime/src/test/java/io/casehub/work/runtime/calendar/ConfigHolidayCalendarTest.java` — same
- `runtime/src/test/java/io/casehub/work/runtime/calendar/DefaultBusinessCalendarTest.java` — same

### Modified (work#216)
- `api/src/main/java/io/casehub/work/api/AssignmentTrigger.java` — add `SLA_ESCALATED`
- `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java` — inject `WorkItemAssignmentService`; fix `writeAudit()`; call `assign()` before `put()` in `executeEscalateTo()`
- `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java` — add setUp wiring + 3 new tests

### Modified (parent#43)
- `/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-work.md` — SPI section, scope field, scanRoots, Depends On

---

## Task 1: Add SLA_ESCALATED to AssignmentTrigger

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/AssignmentTrigger.java`

- [ ] **Step 1: Add the enum value**

```java
package io.casehub.work.api;

/** Lifecycle events that trigger worker (re-)selection. */
public enum AssignmentTrigger {
    /** WorkItem first created and persisted. */
    CREATED,
    /** WorkItem returned to pool by its assignee (release). */
    RELEASED,
    /** WorkItem delegated to a different pool or individual. */
    DELEGATED,
    /** WorkItem returned to an escalation pool after an SLA breach. */
    SLA_ESCALATED
}
```

- [ ] **Step 2: Run api tests**

```bash
scripts/mvn-test casehub-work-api
```
Expected: all tests pass (existing `AssignmentTrigger` tests are not broken; `triggers()` default adds SLA_ESCALATED automatically).

---

## Task 2: Write failing tests for work#216

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java`

These tests will compile (SLA_ESCALATED now exists) but fail at runtime because:
- `auditEntry.detail` is always null today
- `service.assignmentService` field doesn't exist yet → won't compile until Task 3 adds the field

Add the following imports to `ExpiryLifecycleServiceTest.java` (at top with existing imports):

```java
import io.casehub.work.api.AssignmentDecision;
import io.casehub.work.api.AssignmentTrigger;
import io.casehub.work.api.PolicyDecision;
import io.casehub.work.api.SelectionContext;
import io.casehub.work.api.WorkerCandidate;
import io.casehub.work.core.strategy.WorkBroker;
```

Add a `CapturingStrategy` inner class inside `ExpiryLifecycleServiceTest`:

```java
static class CapturingStrategy implements io.casehub.work.api.WorkerSelectionStrategy {
    final java.util.List<SelectionContext> calls = new ArrayList<>();

    @Override
    public AssignmentDecision select(final SelectionContext ctx,
            final java.util.List<WorkerCandidate> candidates) {
        calls.add(ctx);
        return candidates.isEmpty()
                ? AssignmentDecision.noChange()
                : AssignmentDecision.assignTo(candidates.get(0).id());
    }
}
```

Add these three tests after the existing `// ── checkExpired — EscalateTo decision` block:

```java
// ── checkExpired — audit detail ──────────────────────────────────────────

@Test
void checkExpired_withFailDecision_recordsReasonAsAuditEntryDetail() {
    policy.willReturn(new BreachDecision.Fail("deadline-breach"));
    final WorkItem wi = expiredItem();
    service.checkExpired();
    final AuditEntry entry = auditStore.findByWorkItemId(wi.id).stream()
            .filter(e -> "EXPIRED".equals(e.event))
            .findFirst().orElseThrow();
    assertThat(entry.detail).isEqualTo("deadline-breach");
}

// ── checkExpired — EscalateTo auto-assignment ─────────────────────────────

@Test
void checkExpired_withEscalateToDecision_triggersAutoAssignmentWhenCandidatesAvailable() {
    // Override assignmentService with one that pre-assigns from candidateGroups
    final CapturingStrategy capturing = new CapturingStrategy();
    service.assignmentService = new WorkItemAssignmentService(
            capturing,
            group -> java.util.List.of(WorkerCandidate.of("escalation-worker")),
            id -> 0,
            new WorkBroker(),
            (userId, excluded) -> PolicyDecision.ALLOW);

    policy.willReturn(BreachDecision.EscalateTo.to("senior-reviewers"));
    final WorkItem wi = expiredItem();
    service.checkExpired();

    assertThat(capturing.calls).hasSize(1);
    assertThat(store.get(wi.id).orElseThrow().assigneeId).isEqualTo("escalation-worker");
    assertThat(store.get(wi.id).orElseThrow().status).isEqualTo(WorkItemStatus.ASSIGNED);
}

@Test
void checkExpired_withEscalateToDecision_remainsPendingWhenNoWorkersAvailable() {
    // The default service.assignmentService (wired in setUp) uses a no-op
    // strategy that returns noChange() — item stays PENDING
    policy.willReturn(BreachDecision.EscalateTo.to("senior-reviewers"));
    final WorkItem wi = expiredItem();
    service.checkExpired();
    assertThat(store.get(wi.id).orElseThrow().status).isEqualTo(WorkItemStatus.PENDING);
    assertThat(store.get(wi.id).orElseThrow().assigneeId).isNull();
}
```

- [ ] **Step 3: Attempt to compile**

```bash
scripts/mvn-compile runtime
```
Expected: FAIL — `service.assignmentService` field does not exist yet.

---

## Task 3: Implement work#216 in ExpiryLifecycleService

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java` (setUp)

- [ ] **Step 1: Add import and inject WorkItemAssignmentService**

In `ExpiryLifecycleService.java`, add after the existing `@Inject WorkItemsConfig config;` field:

```java
@Inject
WorkItemAssignmentService assignmentService;
```

No import needed — `WorkItemAssignmentService` is in the same package (`io.casehub.work.runtime.service`).

- [ ] **Step 2: Extend writeAudit() to carry a nullable detail**

Replace the existing `writeAudit` method:

```java
private void writeAudit(final WorkItem item, final String event, final String detail, final Instant now) {
    final AuditEntry entry = new AuditEntry();
    entry.workItemId = item.id;
    entry.event = event;
    entry.actor = "system";
    entry.detail = detail;
    entry.occurredAt = now;
    auditStore.append(entry);
}
```

- [ ] **Step 3: Update all writeAudit() call sites**

There are three call sites. Replace them:

In `executeFail()` — pass the failure reason:
```java
writeAudit(item, "EXPIRED", fail.reason(), now);
```

In `executeEscalateTo()` — no detail:
```java
writeAudit(item, "ESCALATED", null, now);
```

In `executeExtend()` — no detail:
```java
writeAudit(item, "SLA_EXTENDED", null, now);
```

- [ ] **Step 4: Call assign() before put() in executeEscalateTo()**

Replace the full `executeEscalateTo()` method with:

```java
private BreachDecision.EscalateTo executeEscalateTo(
        final WorkItem item, final BreachDecision.EscalateTo escalate,
        final SlaBreachContext ctx, final Instant now) {
    item.candidateGroups = String.join(",", escalate.groups());
    item.assigneeId = null;
    item.status = WorkItemStatus.PENDING;

    if (ctx.breachType() == BreachType.COMPLETION_EXPIRED) {
        final Duration window = escalate.deadline() != null
                ? escalate.deadline()
                : Duration.ofHours(config.defaultExpiryHours());
        item.expiresAt = now.plus(window);
    } else {
        item.claimDeadline = computeNewClaimDeadline(item, now);
    }

    assignmentService.assign(item, AssignmentTrigger.SLA_ESCALATED);

    workItemStore.put(item);
    writeAudit(item, "ESCALATED", null, now);
    fireLifecycleEvent("ESCALATED", item);
    return escalate;
}
```

Add the missing import at the top of the file:
```java
import io.casehub.work.api.AssignmentTrigger;
```

- [ ] **Step 5: Wire assignmentService in test setUp**

In `ExpiryLifecycleServiceTest.setUp()`, add after `service.config = WorkItemServiceTest.testConfig();`:

```java
// Default: no-op strategy — existing EscalateTo tests expect PENDING
service.assignmentService = new WorkItemAssignmentService(
        (ctx, candidates) -> AssignmentDecision.noChange(),
        group -> java.util.List.of(),
        id -> 0,
        new WorkBroker(),
        (userId, excluded) -> PolicyDecision.ALLOW);
```

- [ ] **Step 6: Run runtime tests**

```bash
scripts/mvn-test runtime
```
Expected: all tests pass including the 3 new ones.

- [ ] **Step 7: Commit work#216**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  api/src/main/java/io/casehub/work/api/AssignmentTrigger.java \
  runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java \
  runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m \
  "feat(breach): SLA_ESCALATED trigger + auto-assignment after EscalateTo; audit detail on Fail

After EscalateTo breach decision, WorkItemAssignmentService.assign() is called with
SLA_ESCALATED so the routing strategy pre-assigns immediately (LeastLoaded/RoundRobin)
or leaves the item in the escalation pool (ClaimFirst). writeAudit() extended with a
nullable detail field; executeFail() now records fail.reason() as the audit detail —
preserving diagnostic information previously written by AutoRejectEscalationPolicy.

Closes casehubio/work#216 Refs casehubio/work#215"
```

---

## Task 4: Delete deprecated EscalationPolicy from api module (work#215)

**Files:**
- Delete: `api/src/main/java/io/casehub/work/api/EscalationPolicy.java`
- Delete: `api/src/test/java/io/casehub/work/api/EscalationPolicyTest.java`

- [ ] **Step 1: Delete the files**

```bash
rm /Users/mdproctor/claude/casehub/work/api/src/main/java/io/casehub/work/api/EscalationPolicy.java
rm /Users/mdproctor/claude/casehub/work/api/src/test/java/io/casehub/work/api/EscalationPolicyTest.java
```

- [ ] **Step 2: Run api tests**

```bash
scripts/mvn-test casehub-work-api
```
Expected: pass — no surviving code references `EscalationPolicy` in the api module.

---

## Task 5: Delete deprecated service classes from runtime (work#215)

**Files:**
- Delete: 7 files in `runtime/src/main/java/io/casehub/work/runtime/service/`
- Delete: 2 test files in `runtime/src/test/java/io/casehub/work/runtime/service/`

- [ ] **Step 1: Delete production files**

```bash
rm /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/service/AutoRejectEscalationPolicy.java
rm /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/service/ReassignEscalationPolicy.java
rm /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/service/NotifyEscalationPolicy.java
rm /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/service/EscalationPolicyProducer.java
rm /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/service/ExpiryEscalation.java
rm /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/service/ClaimEscalation.java
rm /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/service/WorkItemExpiredEvent.java
```

- [ ] **Step 2: Delete test files**

```bash
rm /Users/mdproctor/claude/casehub/work/runtime/src/test/java/io/casehub/work/runtime/service/EscalationPolicyTest.java
rm /Users/mdproctor/claude/casehub/work/runtime/src/test/java/io/casehub/work/runtime/service/NotifyEscalationPolicyTest.java
```

- [ ] **Step 3: Attempt compile**

```bash
scripts/mvn-compile runtime
```
Expected: FAIL — `WorkItemsConfig.escalationPolicy()` / `claimEscalationPolicy()` are still referenced in anonymous implementations across test files. Fix in Task 6.

---

## Task 6: Remove escalation config from WorkItemsConfig and fix all anonymous impls (work#215)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/config/WorkItemsConfig.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/config/WorkItemsConfigDefaultsTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/calendar/ConfigHolidayCalendarTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/calendar/DefaultBusinessCalendarTest.java`

- [ ] **Step 1: Update WorkItemsConfig.java**

Replace the class header comment (inside the `@ConfigMapping` Javadoc) — remove the line referencing `casehub.work.escalation-policy=reassign`. The new example block:

```java
/**
 * Configuration properties for the CaseHub Work extension.
 * All properties are prefixed with {@code casehub.work}.
 *
 * <p>
 * Example {@code application.properties} overrides:
 *
 * <pre>
 * casehub.work.default-expiry-hours=48
 * casehub.work.cleanup.expiry-check-seconds=30
 * </pre>
 */
```

Remove these two methods entirely from the interface (lines 42–59 in the current file):

```java
// DELETE:
@WithDefault("notify")
String escalationPolicy();

// DELETE:
@WithDefault("notify")
String claimEscalationPolicy();
```

- [ ] **Step 2: Remove two tests from WorkItemsConfigDefaultsTest.java**

Delete the `escalationPolicyIsNotify()` and `claimEscalationPolicyIsNotify()` test methods. The remaining file:

```java
package io.casehub.work.runtime.config;

import static org.assertj.core.api.Assertions.assertThat;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class WorkItemsConfigDefaultsTest {

    @Inject
    WorkItemsConfig config;

    @Test
    void defaultExpiryHoursIs24() {
        assertThat(config.defaultExpiryHours()).isEqualTo(24);
    }

    @Test
    void defaultClaimHoursIs4() {
        assertThat(config.defaultClaimHours()).isEqualTo(4);
    }

    @Test
    void cleanupExpiryCheckSecondsIs60() {
        assertThat(config.cleanup().expiryCheckSeconds()).isEqualTo(60);
    }
}
```

- [ ] **Step 3: Fix WorkItemServiceTest.java — first testConfig() (around line 168)**

Find and remove the two override methods from the first `testConfig()` anonymous class:

```java
// DELETE from testConfig():
@Override
public String escalationPolicy() {
    return "notify";
}

@Override
public String claimEscalationPolicy() {
    return "notify";
}
```

- [ ] **Step 4: Fix WorkItemServiceTest.java — second anonymous config (around line 1039)**

Find the second anonymous `WorkItemsConfig` inline (the `noClaimConfig`) and remove the same two overrides from it as well.

- [ ] **Step 5: Fix ConfigHolidayCalendarTest.java (around line 67)**

Find the anonymous `WorkItemsConfig` implementation in this file and remove:

```java
// DELETE:
@Override
public String escalationPolicy() {
    return "notify";
}

@Override
public String claimEscalationPolicy() {
    return "notify";
}
```

- [ ] **Step 6: Fix DefaultBusinessCalendarTest.java (around line 153)**

Same removal — find the anonymous `WorkItemsConfig` implementation in the static test config class and remove the two escalation method overrides.

- [ ] **Step 7: Run runtime tests**

```bash
scripts/mvn-test runtime
```
Expected: all tests pass. The three deleted test files are gone; the two config test methods are gone; the four anonymous impls no longer implement non-existent methods.

- [ ] **Step 8: Commit work#215**

```bash
git -C /Users/mdproctor/claude/casehub/work add -A
git -C /Users/mdproctor/claude/casehub/work commit -m \
  "chore: remove deprecated EscalationPolicy SPI and all built-in implementations

Removes EscalationPolicy (SPI interface), AutoRejectEscalationPolicy,
ReassignEscalationPolicy, NotifyEscalationPolicy, EscalationPolicyProducer,
ExpiryEscalation qualifier, ClaimEscalation qualifier, WorkItemExpiredEvent,
and associated tests. Removes escalationPolicy() and claimEscalationPolicy()
from WorkItemsConfig. No consuming repos implement EscalationPolicy.
Migrate to SlaBreachPolicy.onBreach() → BreachDecision.

Closes casehubio/work#215"
```

---

## Task 7: Close work#205

work#205 (`scanRoots` 3-param split) was fully implemented in commit `66c4541`. No code change needed.

- [ ] **Step 1: Close the issue**

```bash
gh issue close 205 --repo casehubio/work \
  --comment "Fixed in commit 66c4541: scanRoots(assignee, candidateUser, candidateGroups) is implemented with independent OR predicates for all three dimensions. No further code change required."
```

---

## Task 8: Sync parent docs (parent#43)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-work.md`

- [ ] **Step 1: Update Module Structure table — casehub-work-api row**

Replace:
```
| `casehub-work-api` | Pure-Java SPI (no Quarkus) | All SPIs: worker selection, registry, workload provision, escalation, spawn, skill profiling, notification channel |
```

With:
```
| `casehub-work-api` | Pure-Java SPI (no Quarkus) | All SPIs: worker selection, registry, workload provision, SLA breach policy, spawn, skill profiling, notification channel. Depends on `casehub-platform-api` for `Path` and `Preferences` used in `SlaBreachContext`. |
```

- [ ] **Step 2: Add SPI Reference section after "## Key Abstractions"**

Insert before the `---` separator that follows the CDI Events section:

```markdown
### SPI Reference (casehub-work-api)

| SPI | Method | Description |
|---|---|---|
| `WorkerSelectionStrategy` | `select(SelectionContext, List<WorkerCandidate>)` | Pluggable routing — LeastLoaded (default), ClaimFirst, RoundRobin built-in |
| `WorkerRegistry` | `resolveGroup(String)` | Resolves candidateGroup names to `WorkerCandidate` objects |
| `WorkloadProvider` | `getActiveWorkCount(String)` | Active WorkItem count per worker (used by LeastLoaded) |
| `SlaBreachPolicy` | `onBreach(SlaBreachContext) → BreachDecision` | SLA breach handling: returns `Fail`, `EscalateTo(groups, deadline)`, `Extend(by)`, or `Chained`. Replaces `@Deprecated EscalationPolicy` (removed). |
| `ExclusionPolicy` | `check(userId, excludedUsers) → PolicyDecision` | Conflict-of-interest user exclusion |
| `SpawnPort` | `spawn(SpawnRequest) → SpawnResult` | Child WorkItem creation with idempotency |
| `WorkerSelectionStrategy` | `triggers()` | Set of `AssignmentTrigger` values this strategy handles: `CREATED`, `RELEASED`, `DELEGATED`, `SLA_ESCALATED` |
```

- [ ] **Step 3: Update WorkItem Entity section — add scope field**

After the line "10-status lifecycle from creation through terminal states..." add:

```markdown
**Key fields:** `scope VARCHAR(255)` (V31) — hierarchical scope path for SLA preference resolution via `casehub-platform-api`'s `Path` type; null defaults to org root scope. Populated by callers (e.g. casehub-engine via `HumanTaskTarget.scope` — see engine#330).
```

- [ ] **Step 4: Update Depends On section**

Replace:
```markdown
## Depends On

- `casehub-ledger` — optional only, via `casehub-work-ledger` module. Core has zero casehubio deps.
```

With:
```markdown
## Depends On

- `casehub-platform-api` — production dep in `casehub-work-api` module for `Path` and `Preferences` types used in `SlaBreachContext`. Zero-dep pure-Java module; does not force Quarkus on consumers.
- `casehub-platform` (mock module) — `test` scope only in `runtime` and `casehub-work-queues`; provides `MockPreferenceProvider @DefaultBean` for `@QuarkusTest` augmentation.
- `casehub-ledger` — optional only, via `casehub-work-ledger` module. Core has zero other casehubio deps.
```

- [ ] **Step 5: Add note about WorkItemStore.scanRoots after Core Services section**

After the line "See `docs/DESIGN.md` for service class structure and the M-of-N coordination model." add:

```markdown
**Inbox query:** `WorkItemStore.scanRoots(assignee, candidateUser, candidateGroups)` — three-param; each dimension is an independent OR predicate. Assignee matches on `assigneeId`; candidateUser matches on `candidateUsers LIKE`; candidateGroups matches on `candidateGroups LIKE` for each group. Returns root WorkItems (no `parentId`) including parents of visible children.
```

- [ ] **Step 6: Update Current State test count**

Replace the test count line:
```
- 637+ tests passing in runtime module; native image validated at 0.084s startup
```

With:
```
- 746+ tests passing in runtime module; native image validated
```

(The runtime test count from the handover was 746.)

- [ ] **Step 7: Commit in parent repo**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/repos/casehub-work.md
git -C /Users/mdproctor/claude/casehub/parent commit -m \
  "docs: sync casehub-work deep-dive for SlaBreachPolicy, scope field, scanRoots, platform-api dep

Closes casehubio/parent#43 Refs casehubio/work#212 casehubio/work#213"
```

---

## Self-Review Checklist

- [x] **Spec coverage:**
  - work#215: all 11 deletions + config removal + 4 test file fixes covered ✓
  - work#216: `SLA_ESCALATED` enum value, `writeAudit()` detail, `assign()` call, 3 new tests ✓
  - work#205: close-issue step ✓
  - parent#43: 5 doc changes + commit ✓
- [x] **No placeholders:** all steps have exact code or exact commands
- [x] **Type consistency:** `WorkItemAssignmentService` package-private constructor used consistently in Tasks 2 and 3; `AssignmentTrigger.SLA_ESCALATED` added in Task 1 before first test reference in Task 2
- [x] **`writeAudit()` call sites:** all 3 call sites updated in Task 3 Step 3; 4-arg signature consistent
- [x] **setUp wiring:** no-op `assignmentService` in `setUp()` preserves all existing EscalateTo tests that assert `PENDING`; capturing strategy wired per-test in the two new auto-assignment tests
