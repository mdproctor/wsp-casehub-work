# Design: Human Task Lifecycle Alignment

**Issue:** casehubio/work#240
**Date:** 2026-06-18
**Status:** Draft

---

## Goal

Align the WorkItem lifecycle state machine with industry specs (WS-HumanTask 1.1, OpenHumanTask, BPMN 2.0, CMMN 1.1) while maintaining coherence across all six lifecycle layers in the casehub platform. Produce a design that closes spec gaps, fixes cross-system inconsistencies, and establishes `docs/LIFECYCLE.md` in casehub-parent as the normative lifecycle reference for the entire platform.

---

## The Six Lifecycle Layers

The platform has six distinct lifecycle state machines, top to bottom:

| Layer | System | Enum | States | Terminal |
|---|---|---|---|---|
| 1 | Serverless Workflow (CNCF SDK) | `WorkflowStatus` | PENDING, RUNNING, WAITING, COMPLETED, FAULTED, CANCELLED, SUSPENDED | COMPLETED, FAULTED, CANCELLED |
| 2 | Case (casehub-engine) | `CaseStatus` | RUNNING, WAITING, SUSPENDED, COMPLETED, FAULTED, CANCELLED | COMPLETED, FAULTED, CANCELLED |
| 3 | PlanItem (casehub-engine blackboard) | `PlanItemStatus` | PENDING, RUNNING, DELEGATED, COMPLETED, FAULTED, REJECTED, CANCELLED | COMPLETED, FAULTED, REJECTED, CANCELLED |
| 4 | HumanTask (casehub-engine) | Definition type — no lifecycle enum | Expressed through PlanItem (DELEGATED) + WorkItem states | — |
| 5 | WorkItem (casehub-work) | `WorkItemStatus` | PENDING, ASSIGNED, IN_PROGRESS, DELEGATED, SUSPENDED, COMPLETED, REJECTED, CANCELLED, EXPIRED, ESCALATED | COMPLETED, REJECTED, CANCELLED, EXPIRED, ESCALATED |
| 6 | Commitment (casehub-qhorus) | `CommitmentState` | OPEN, ACKNOWLEDGED, FULFILLED, DECLINED, FAILED, DELEGATED, EXPIRED | FULFILLED, DECLINED, FAILED, DELEGATED, EXPIRED |

Auxiliary: `WorkEventType` (16 values), `MessageType` (9 speech acts), `GroupStatus` (3 values), `AssignmentTrigger` (5 values), `BreachDecision` (5 variants), `DeclineTarget` (2 values).

Four `BindingTarget` types determine how PlanItems enter their active state:
- `CapabilityTarget` → PlanItem RUNNING (Quartz job executing locally)
- `HumanTaskTarget` → PlanItem DELEGATED (creates WorkItem in casehub-work)
- `SubCaseTarget` → PlanItem DELEGATED (starts child case)
- `ExtensionTarget` → PlanItem DELEGATED (dispatches to external extension)

---

## Issues Found

### Issue 1 — FAULTED gap at WorkItem layer

Every layer except WorkItem distinguishes system failure from actor refusal:

| Layer | System failure | Actor refusal |
|---|---|---|
| WorkflowStatus | FAULTED | — |
| CaseStatus | FAULTED | — |
| PlanItemStatus | FAULTED | REJECTED |
| **WorkItemStatus** | **missing** | REJECTED |
| CommitmentState | FAILED | DECLINED |

When an AI agent hits an API timeout, context overflow, or unhandled exception, that is not REJECTED (deliberate refusal) — it is a system fault. Conflating the two corrupts trust scoring: REJECTED implies a deliberate decision to refuse; a system fault implies infrastructure failure with no actor judgment.

**Naming decision:** WS-HumanTask 1.1 and OpenHumanTask use `ERROR`. The rest of the casehub platform uses `FAULTED` (WorkflowStatus, CaseStatus, PlanItemStatus). Platform coherence wins — the name is **FAULTED**.

### Issue 2 — OBSOLETE gap

WS-HumanTask 1.1 and OpenHumanTask both have OBSOLETE — task superseded by context change. Distinct from CANCELLED:

- CANCELLED: a human or system deliberately stopped this work
- OBSOLETE: the case context changed, making this work irrelevant — it was never completed but never needed to be

Example: an IRB review WorkItem becomes OBSOLETE when the clinical trial is withdrawn. The reviewer did nothing wrong. Marking it CANCELLED misattributes the termination to a deliberate action.

No other layer in the platform has OBSOLETE. This is unique to human/agent tasks — workflow steps don't become obsolete, cases can be cancelled but not obsoleted.

### Issue 3 — DELEGATED has three incompatible meanings

| System | DELEGATED means | Terminal? |
|---|---|---|
| PlanItemStatus | Control passed to external actor — engine waiting | No |
| WorkItemStatus | Pre-acceptance hold — targeted to named actor | No |
| CommitmentState | Obligation transferred to new debtor via HANDOFF — original discharged | **Yes** |

PLATFORM.md already flags this as Known Overlap Risk #3. The WorkItem and Commitment meanings are dangerous: a developer building a Qhorus→WorkItem bridge expects DELEGATED obligations to be discharged (Qhorus semantics) but WorkItem DELEGATED means the item is still active.

No code change fixes this — the semantics are correct within each system. The fix is documentation: javadoc on all three must cross-reference the others and explicitly state the difference.

### Issue 4 — WorkEventType has 7+ gaps

Events fired as string names that have no `WorkEventType` enum value:

| Event fired | `eventType()` returns |
|---|---|
| DELEGATION_ACCEPTED | CREATED (silent fallback) |
| DELEGATION_DECLINED | CREATED |
| SLA_REASSIGNED | CREATED |
| SLA_EXTENDED | CREATED |
| BREACH_POLICY_MISCONFIGURED | CREATED |
| LABEL_ADDED | CREATED |
| LABEL_REMOVED | CREATED |

The `eventType()` method silently falls back to `WorkEventType.CREATED` for any unknown event name. Any observer switching on `WorkEventType` misclassifies these events. This is a correctness bug.

### Issue 5 — PlanItemStatus has no SUSPENDED

WorkItemStatus, CaseStatus, and WorkflowStatus all have SUSPENDED. PlanItemStatus does not. When a WorkItem is SUSPENDED, the PlanItem stays DELEGATED — the engine cannot distinguish "human is working" from "human has paused." Cosmetic for monitoring, does not affect execution correctness. Low priority.

### Issue 6 — ESCALATED naming overload

The word "escalation" is used for two different things:

1. **The action** — re-routing a WorkItem to new groups when an SLA breaches → status stays PENDING, event is `SLA_REASSIGNED`, trigger is `AssignmentTrigger.SLA_ESCALATED`
2. **The terminal state** — all re-routing branches exhausted → status is `ESCALATED`, set by `executeExhausted()`

The code behavior is correct. But the `WorkItemLifecycleAdapter` javadoc says "ESCALATED is not terminal" which conflates the action with the state. The javadoc is wrong.

### Issue 7 — Simple progress reporting gap

AI agents and human actors executing multi-step work have no way to report intermediate progress. Trust scoring benefits from progress signals: an agent reporting 80% before faulting has different trust implications than one that faults immediately.

---

## Design

### 1. WorkItemStatus — complete state machine (12 values)

```
                     ┌──────── release ──────────────────────┐
                     │                                       ▼
   PENDING ──claim──► ASSIGNED ──start──► IN_PROGRESS
      ▲                  │    │               │      │
      │                  │    └──reject───────┤      │
      │                  │                    │      │
      │         suspend──┤           suspend──┤      │
      │                  ▼                    ▼      │
      │              SUSPENDED ◄──────────────┘      │
      │              (resume restores priorStatus)   │
      │                                              │
      │◄── delegate ─────────────────────────────────┘
      │    (DELEGATED: pre-acceptance hold)
      │         │
      │    accept → ASSIGNED    decline → PENDING or DELEGATOR
      │
      ├──── complete (from IN_PROGRESS) ───────────── COMPLETED
      ├──── reject (from ASSIGNED|IN_PROGRESS) ────── REJECTED
      ├──── fault (from any non-terminal) ─────────── FAULTED      ← NEW
      ├──── cancel (from any non-terminal) ────────── CANCELLED
      ├──── obsolete (from any non-terminal) ──────── OBSOLETE     ← NEW
      ├──── SLA Fail ──────────────────────────────── EXPIRED
      ├──── SLA EscalateTo ────────────────────────── PENDING (new groups)
      └──── SLA Exhausted ─────────────────────────── ESCALATED
```

**Terminal states (7):** COMPLETED, REJECTED, FAULTED, CANCELLED, OBSOLETE, EXPIRED, ESCALATED

**Active states (5):** PENDING, ASSIGNED, IN_PROGRESS, DELEGATED, SUSPENDED

```java
public enum WorkItemStatus {
    PENDING, ASSIGNED, IN_PROGRESS, COMPLETED, REJECTED, FAULTED,
    DELEGATED, SUSPENDED, CANCELLED, EXPIRED, ESCALATED, OBSOLETE;

    public boolean isTerminal() {
        return switch (this) {
            case COMPLETED, REJECTED, FAULTED, CANCELLED, OBSOLETE, EXPIRED, ESCALATED -> true;
            default -> false;
        };
    }

    public boolean isActive() {
        return switch (this) {
            case PENDING, ASSIGNED, IN_PROGRESS, SUSPENDED, DELEGATED -> true;
            default -> false;
        };
    }
}
```

### 2. New WorkItemService methods

**`fault(UUID id, String systemActorId, String errorDetail)`**
- Callable from any non-terminal state
- Sets status to FAULTED, completedAt to now, resolution to errorDetail
- Cancels expiry and claim deadline timers
- Fires `WorkItemLifecycleEvent("FAULTED", ...)` (sync + async)
- Typical callers: "system", AI agent IDs

**`obsolete(UUID id, String triggeredBy, String reason)`**
- Callable from any non-terminal state
- Sets status to OBSOLETE, completedAt to now, resolution to reason
- Cancels expiry and claim deadline timers
- Fires `WorkItemLifecycleEvent("OBSOLETE", ...)` (sync + async)
- Typical callers: "system", engine identity

**`progress(UUID id, String actorId, Integer percentComplete, String statusNote)`**
- Callable only from IN_PROGRESS
- Updates percentComplete and statusNote fields
- Fires `WorkItemLifecycleEvent("PROGRESS_UPDATE", ...)` (sync only — not a terminal event)
- Does NOT change status

**`faultFromSystem(UUID id, String actorId, String errorDetail)`**
- System variant accepting any non-terminal status (parallel to `completeFromSystem` / `rejectFromSystem`)
- Used by infrastructure (e.g. multi-instance coordinator, agent timeout handlers)
- Skips schema validation

### 3. REST API

| Method | Path | Body | Purpose |
|---|---|---|---|
| `PUT` | `/workitems/{id}/fault` | `{ "actor": "...", "errorDetail": "..." }` | System failure → FAULTED |
| `PUT` | `/workitems/{id}/obsolete` | `{ "actor": "...", "reason": "..." }` | Context superseded → OBSOLETE |
| `PUT` | `/workitems/{id}/progress` | `{ "percentComplete": 42, "statusNote": "..." }` | Progress update (IN_PROGRESS only) |

Naming follows existing pattern: verb paths (`claim`, `start`, `complete`, `reject`, `fault`, `obsolete`, `progress`).

### 4. WorkEventType — complete enum (25 values)

```java
public enum WorkEventType {
    CREATED, ASSIGNED, STARTED, COMPLETED, REJECTED, FAULTED,
    CANCELLED, OBSOLETE, EXPIRED, ESCALATED,
    DELEGATED, DELEGATION_ACCEPTED, DELEGATION_DECLINED,
    RELEASED, SUSPENDED, RESUMED,
    CLAIM_EXPIRED, SLA_REASSIGNED, SLA_EXTENDED, DEADLINE_EXTENDED,
    SPAWNED, SIGNAL_RECEIVED, PROGRESS_UPDATE,
    LABEL_ADDED, LABEL_REMOVED
}
```

The `eventType()` fallback in `WorkItemLifecycleEvent` changes from silently returning CREATED to letting `valueOf` throw. Any event fired without a corresponding enum value is a bug that tests must catch.

### 5. WorkItem entity — new columns

```java
@Column(name = "percent_complete")
public Integer percentComplete;

@Column(name = "status_note", columnDefinition = "TEXT")
public String statusNote;
```

### 6. Flyway migration

```sql
ALTER TABLE work_item ADD COLUMN percent_complete SMALLINT;
ALTER TABLE work_item ADD COLUMN status_note TEXT;
```

No migration for status enum — stored as STRING via `@Enumerated(EnumType.STRING)`.

### 7. Bridge mapping — PlanItemCompletionApplier

The adapter's `applyStatus()` switch gains two cases:

| WorkItemStatus | PlanItem transition | Rationale |
|---|---|---|
| COMPLETED | `markCompleted()` | Success |
| REJECTED | `markRejected()` | Actor refusal |
| FAULTED | `markFaulted()` | System failure — direct alignment |
| CANCELLED | `markCancelled()` | Deliberate stop |
| OBSOLETE | `markCancelled()` | Context-driven termination |
| EXPIRED | `markFaulted()` | Deadline breach — a type of fault |
| ESCALATED | context signal, stays DELEGATED | All breach policies exhausted |

This is a casehub-engine change (work-adapter module). Filed as engine issue.

### 8. WorkItemLifecycleAdapter — ESCALATED handling

The adapter already handles ESCALATED correctly in code (writes context signal, PlanItem stays DELEGATED). The class javadoc is wrong — says "ESCALATED is not terminal" when the WorkItem IS terminal.

Fix the javadoc: "ESCALATED is terminal — all SLA breach policy branches have been exhausted. The adapter writes a `workItemEscalated` signal to the case context so the engine can react (e.g. notify supervisor, create replacement task). The PlanItem stays DELEGATED. Distinct from the SLA_REASSIGNED event, which fires when an EscalateTo decision re-routes the WorkItem to new groups without terminating it."

### 9. DELEGATED cross-system documentation

All three DELEGATED declarations gain cross-references in javadoc:

- `WorkItemStatus.DELEGATED`: "Pre-acceptance hold — non-terminal. See `docs/LIFECYCLE.md` for cross-system DELEGATED semantics."
- `CommitmentState.DELEGATED`: "Obligation transferred — terminal. See `docs/LIFECYCLE.md` for cross-system DELEGATED semantics."
- `PlanItemStatus.DELEGATED`: "Control passed to external actor — non-terminal. See `docs/LIFECYCLE.md` for cross-system DELEGATED semantics."

### 10. PLATFORM.md updates

**Capability ownership table** — "Human task inbox" entry updated: "12 statuses" (was "10 statuses"). Add: "FAULTED: system failure (distinct from REJECTED actor refusal). OBSOLETE: context superseded. Simple progress: percentComplete + statusNote."

**New implementation protocol entry:**

> **Lifecycle coherence:** Any repo adding, renaming, or removing lifecycle states must update `docs/LIFECYCLE.md`. New states must be justified against the existing cross-system map — naming must be consistent with peer layers unless a documented reason exists for divergence. See `docs/LIFECYCLE.md` for the authoritative cross-system state machine map.

### 11. docs/LIFECYCLE.md (casehub-parent)

New document — the authoritative cross-system lifecycle reference. Contents:

1. **Contract** — this document is normative. Any repo adding lifecycle states must update it.
2. **The six lifecycle layers** — complete state machine for each, with state diagrams.
3. **Cross-layer bridges** — WorkItem→PlanItem mapping, Commitment→WorkItem conceptual parallels, Workflow→PlanItem mapping.
4. **Naming coherence** — where terminology aligns (FAULTED at 4 layers, COMPLETED at all layers, CANCELLED at all layers) and where it diverges (DELEGATED: 3 meanings; REJECTED vs DECLINED; EXPIRED at WorkItem/Commitment but not PlanItem/Case).
5. **Industry spec alignment** — WS-HumanTask 1.1, OpenHumanTask, BPMN 2.0, CMMN 1.1 mapping tables.
6. **Known gaps** — PlanItem SUSPENDED, COMPENSATING/COMPENSATED (BPMN), CMMN ManualActivation.
7. **Design rationale** — why FAULTED not ERROR, why OBSOLETE exists only at WorkItem layer, why DELEGATED semantics intentionally diverge.

---

## Cross-Repo Deliverables

| Repo | Change | Priority |
|---|---|---|
| casehub-work | Add FAULTED, OBSOLETE to WorkItemStatus; `fault()`, `obsolete()`, `progress()` methods; REST endpoints; WorkEventType alignment; Flyway migration; WorkItemResponse update | High |
| casehub-engine | PlanItemCompletionApplier: add FAULTED→markFaulted(), OBSOLETE→markCancelled(); fix WorkItemLifecycleAdapter javadoc | High |
| casehub-parent | Create docs/LIFECYCLE.md; update PLATFORM.md (capability ownership + lifecycle protocol) | High |
| casehub-work | DELEGATED javadoc cross-reference | Medium |
| casehub-qhorus | CommitmentState.DELEGATED javadoc cross-reference | Medium |
| casehub-engine | PlanItemStatus.DELEGATED javadoc cross-reference | Medium |
| casehub-engine | File issue: PlanItemStatus.SUSPENDED gap | Low |

---

## Industry Spec Alignment — After This Design

### vs WS-HumanTask 1.1

| WS-HT State | casehub-work | Notes |
|---|---|---|
| CREATED | PENDING | ✓ collapsed with READY |
| READY | PENDING | ✓ |
| RESERVED | ASSIGNED | ✓ |
| IN_PROGRESS | IN_PROGRESS | ✓ |
| SUSPENDED | SUSPENDED | ✓ with priorStatus restoration |
| COMPLETED | COMPLETED | ✓ |
| FAILED | REJECTED | ✓ actor's deliberate decision |
| ERROR | FAULTED | ✓ platform name, same semantics |
| EXITED | CANCELLED | ✓ |
| OBSOLETE | OBSOLETE | ✓ |

### vs OpenHumanTask

| OHT State | casehub-work | Notes |
|---|---|---|
| created / ready | PENDING | ✓ |
| reserved | ASSIGNED | ✓ |
| running | IN_PROGRESS | ✓ |
| completed | COMPLETED | ✓ |
| obsolete | OBSOLETE | ✓ |
| failed | REJECTED | ✓ |
| error | FAULTED | ✓ |
| exited | CANCELLED | ✓ |
| *(no suspended)* | SUSPENDED | ✓ casehub is richer |

### What casehub has beyond the specs

| Feature | Assessment |
|---|---|
| EXPIRED | Deadline-passed explicit. No spec equivalent. |
| ESCALATED | All breach policies exhausted — terminal. No spec equivalent. |
| DELEGATED (pre-acceptance) | Richer than WS-HT nomination. |
| Named outcomes | Beyond any spec — enables outcome-based routing. |
| ExclusionPolicy | Conflict-of-interest. No spec equivalent. |
| priorStatus restoration | WS-HT implies but does not specify. |
| delegationChain | Audit provenance. No spec equivalent. |
| SLA breach policy framework | Chained/EscalateTo/Extend/Exhausted — beyond any spec. |
| Simple progress (percentComplete + statusNote) | Operational. Not in any spec. |

### Remaining spec gaps (intentional)

| Gap | Spec | Reason |
|---|---|---|
| COMPENSATING / COMPENSATED | BPMN 2.0 | Tracked separately: casehub-work#238 |
| ManualActivation (enabled/disabled) | CMMN 1.1 | casehub auto-activates by design |
| PlanItem SUSPENDED | CMMN 1.1 | Observability gap — filed as engine issue |
