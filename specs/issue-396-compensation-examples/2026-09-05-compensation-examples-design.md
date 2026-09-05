# Design: Compensation Examples — Practical End-User Scenarios

**Issue:** casehubio/work#396
**Date:** 2026-09-05
**Status:** Draft
**Parent:** #238 (saga compensation epic)

---

## 1. Problem

The examples module has 17 runnable scenarios covering lifecycle, ledger, queues, AI, and templates — but zero compensation scenarios. Compensation is the most complex capability added in #238 and the one developers most need to see working end-to-end. The API surface (`compensate()`, `CompensationStatus`, `CompensationLifecycleObserver`, guards) is non-obvious from javadoc alone — the interaction between original and compensating WorkItems, the automatic state transitions, and the intermediate COMPENSATING state all need demonstrated.

---

## 2. Scope

Three new scenarios in the existing `examples/` module, following the established REST+JSON pattern. Each scenario has a distinct learning objective and relatable domain.

Engine-integrated examples (saga orchestration, notification pipeline) are out of scope — filed separately in casehubio/engine where they belong in the dependency graph (D1).

---

## 3. Scenario 1: Expense Approval Reversal

**Learning objective:** Full compensation lifecycle with different actors, audit trail, and ledger causal chain. The "learn compensation" example.

**Story:** A finance analyst creates a $50K vendor payment approval. A senior finance officer claims and approves it. Three days later, internal audit discovers the invoice was for a cancelled project. A compliance officer compensates the approval.

**Transaction model:** Single `@Transactional`. Compensating WorkItem is created and completed within one call. Auto-`markCompensated` fires synchronously via `CompensationLifecycleObserver`. Developer sees the end-to-end flow resolving in one shot (D3).

### 3.1 Endpoint

`POST /examples/compensation/run`

### 3.2 Steps

| # | Actor | Action | API call |
|---|---|---|---|
| 1 | finance-analyst | Create vendor payment approval | `workItemService.create(request)` |
| 2 | senior-finance-officer | Claim the approval | `workItemService.claim(id, actor)` |
| 3 | senior-finance-officer | Start work | `workItemService.start(id, actor)` |
| 4 | senior-finance-officer | Approve the payment | `workItemService.complete(id, actor, resolution, outcome)` |
| 5 | internal-audit | Trigger compensation | `workItemService.compensate(originalId, compensatingRequest, triggeredBy, reason)` |
| 6 | compliance-officer | Claim compensating WorkItem | `workItemService.claim(compensatingId, actor)` |
| 7 | compliance-officer | Complete compensation | `workItemService.complete(compensatingId, actor, resolution, outcome)` |
| 8 | — | Verify original is COMPENSATED | `workItemService.findById(originalId)` → assert `compensationStatus == COMPENSATED` |

### 3.3 Response

```java
record CompensationResponse(
    String scenario,
    List<StepLog> steps,
    UUID originalWorkItemId,
    UUID compensatingWorkItemId,
    String compensationStatus,    // "COMPENSATED"
    String compensatingLink,      // compensatesWorkItemId on the compensating WI
    String triggeredBy,
    String reason,
    List<AuditEntryResponse> originalAuditTrail,
    List<AuditEntryResponse> compensatingAuditTrail
)
```

### 3.4 What the developer learns

- `compensate()` creates a **new WorkItem** — it's not a status change on the original
- The compensating WorkItem has its own lifecycle (claim → start → complete)
- Different actors can handle the compensation (compliance-officer, not the original approver)
- The original's `compensationStatus` transitions automatically (NONE → COMPENSATING → COMPENSATED) via `CompensationLifecycleObserver`
- The `compensatesWorkItemId` link connects compensating → original
- Both WorkItems have independent audit trails
- The `reason` on `compensate()` records **why** the reversal happened

---

## 4. Scenario 2: Multi-Step Loan Application Rollback

**Learning objective:** Application-driven reverse-order compensation of multiple correlated WorkItems. The "build your own saga" example.

**Story:** A loan application process has three sequential steps linked by callerRef: credit check → property valuation → final approval. All three complete. Then a regulatory audit discovers the credit check used outdated scoring data — all three must be reversed in dependency order.

**Transaction model:** Split transactions. Each compensation step is separate so the developer can observe the intermediate COMPENSATING state. A multi-step endpoint with a `phase` query parameter advances through the scenario (D3).

### 4.1 Endpoints

`POST /examples/loan-rollback/run` — executes all phases in sequence, returns the full result

### 4.2 Phases

**Phase 1 — Forward execution (3 WorkItems):**

| # | Actor | Action | CallerRef |
|---|---|---|---|
| 1 | loan-system | Create credit check | `loan:L-2026-001/credit-check` |
| 2 | credit-analyst | Claim + complete credit check | |
| 3 | loan-system | Create property valuation | `loan:L-2026-001/valuation` |
| 4 | property-surveyor | Claim + complete valuation | |
| 5 | loan-system | Create final approval | `loan:L-2026-001/approval` |
| 6 | senior-underwriter | Claim + complete approval | |

**Phase 2 — Compensate in reverse order:**

| # | Actor | Action | Compensates |
|---|---|---|---|
| 7 | regulatory-audit | Compensate final approval | approval |
| 8 | — | Query: approval is COMPENSATING | |
| 9 | compliance-officer | Complete compensating approval | |
| 10 | — | Query: approval is COMPENSATED | |
| 11 | regulatory-audit | Compensate valuation | valuation |
| 12 | compliance-officer | Complete compensating valuation | |
| 13 | regulatory-audit | Compensate credit check | credit-check |
| 14 | compliance-officer | Complete compensating credit check | |

**Phase 3 — Verify all compensated:**

| # | Action |
|---|---|
| 15 | Query all WorkItems by callerRef pattern `loan:L-2026-001/*` |
| 16 | Assert all 3 originals have `compensationStatus == COMPENSATED` |
| 17 | Assert each has a compensating WorkItem linked via `compensatesWorkItemId` |

### 4.3 Response

```java
record LoanRollbackResponse(
    String scenario,
    List<StepLog> steps,
    List<LoanStepSummary> forwardSteps,
    List<LoanStepSummary> compensationSteps,
    String compensationOrder   // "approval → valuation → credit-check"
)

record LoanStepSummary(
    String callerRef,
    UUID originalId,
    UUID compensatingId,
    String compensationStatus
)
```

### 4.4 What the developer learns

- Multiple WorkItems can be compensated in application-driven reverse order
- CallerRef provides correlation for finding related WorkItems
- The COMPENSATING intermediate state is visible between steps — queries can filter on it
- Each compensating WorkItem is independent — different actors, different resolutions
- The application (not the engine) drives the ordering when running standalone
- All 3 originals end up COMPENSATED with linked compensating WorkItems

### 4.5 Split transaction implementation

The scenario uses `@Transactional(REQUIRES_NEW)` on each phase's helper method, or `QuarkusTransaction.requiringNew()` to force intermediate commits. This ensures the COMPENSATING state is visible to queries between steps. The main `run()` method orchestrates the phases and collects results.

---

## 5. Scenario 3: Compensation Resilience

**Learning objective:** Guard semantics (all 3 guards) plus full lifecycle operations on the compensating WorkItem. The "defensive programming" example.

**Story:** A hospital patient referral system. A GP refers a patient to a specialist. Various compensation paths demonstrate what works, what fails, and why.

**Transaction model:** Split — the suspend/resume/complete lifecycle on the compensating WorkItem requires separate steps.

### 5.1 Endpoint

`POST /examples/compensation-resilience/run`

### 5.2 Sub-Scenarios

**5.2.1 — Guard: cannot compensate non-COMPLETED WorkItem**

| # | Actor | Action | Expected |
|---|---|---|---|
| 1 | gp-reception | Create referral (status: CREATED) | |
| 2 | gp-reception | Try to compensate it | `IllegalStateException`: "Only COMPLETED WorkItems can be compensated" |

**5.2.2 — Happy path + Guard: cannot double-compensate**

| # | Actor | Action | Expected |
|---|---|---|---|
| 3 | gp-reception | Create + complete a referral | status: COMPLETED |
| 4 | clinic-admin | Compensate the referral | compensatingWorkItem created, original: COMPENSATING |
| 5 | clinic-admin | Complete the compensating WorkItem | original: COMPENSATED |
| 6 | clinic-admin | Try to compensate the original again | `IllegalStateException`: "already has compensation activity" |

**5.2.3 — Guard: cannot compensate a compensating WorkItem**

| # | Actor | Action | Expected |
|---|---|---|---|
| 7 | — | Take the compensating WI from 5.2.2 | |
| 8 | clinic-admin | Try to compensate the compensating WI | `IllegalStateException`: "Compensating WorkItems cannot themselves be compensated" |

**5.2.4 — Lifecycle on compensating WorkItem: suspend → resume → complete**

| # | Actor | Action | Expected |
|---|---|---|---|
| 9 | gp-reception | Create + complete another referral | |
| 10 | clinic-admin | Compensate it | compensating WI created |
| 11 | clinic-admin | Claim the compensating WI | |
| 12 | clinic-admin | Suspend the compensating WI | "Doctor on leave" |
| 13 | — | Query: original is still COMPENSATING | |
| 14 | senior-doctor | Resume the compensating WI | |
| 15 | senior-doctor | Complete the compensating WI | original auto-COMPENSATED |

### 5.3 Response

```java
record CompensationResilienceResponse(
    String scenario,
    List<StepLog> steps,
    GuardResult nonCompletedGuard,
    GuardResult doubleCompensationGuard,
    GuardResult compensatorGuard,
    LifecycleResult suspendResumeLifecycle
)

record GuardResult(
    String description,
    boolean guardTriggered,
    String errorMessage
)

record LifecycleResult(
    UUID originalId,
    UUID compensatingId,
    List<String> compensatingStatuses,  // [CREATED, READY, IN_PROGRESS, SUSPENDED, IN_PROGRESS, COMPLETED]
    String finalCompensationStatus      // COMPENSATED
)
```

### 5.4 What the developer learns

- The three invariants of the compensation model and their error messages
- How to handle `IllegalStateException` from `compensate()` — these are not bugs, they're design constraints
- The compensating WorkItem has the **full** lifecycle — suspend, resume, delegate, escalate all work
- While the compensating WorkItem is suspended, the original stays COMPENSATING — the system reflects reality

---

## 6. Testing

Each scenario gets a test class following the existing pattern:

| Test class | Pattern | Validates |
|---|---|---|
| `CompensationScenarioTest` | `@QuarkusTest` + REST-assured POST | Full lifecycle, response fields, audit trail presence |
| `LoanRollbackScenarioTest` | `@QuarkusTest` + REST-assured POST | 3 originals compensated, correct order, callerRef queries |
| `CompensationResilienceScenarioTest` | `@QuarkusTest` + REST-assured POST | All 3 guards triggered, suspend/resume lifecycle, final states |

---

## 7. README Update

Add 3 entries to the scenarios table (numbers 18-20) and curl commands to the "Running All Scenarios" section.

---

## 8. File Structure

```
examples/src/main/java/io/casehub/work/examples/
  compensation/
    CompensationScenario.java
    CompensationResponse.java
  loanrollback/
    LoanRollbackScenario.java
    LoanRollbackResponse.java
    LoanStepSummary.java
  resilience/
    CompensationResilienceScenario.java
    CompensationResilienceResponse.java
    GuardResult.java
    LifecycleResult.java

examples/src/test/java/io/casehub/work/examples/
  compensation/CompensationScenarioTest.java
  loanrollback/LoanRollbackScenarioTest.java
  resilience/CompensationResilienceScenarioTest.java
```

---

## 9. Capability Coverage Matrix

| Capability | Ex 1 (Expense) | Ex 2 (Loan) | Ex 3 (Resilience) |
|---|---|---|---|
| `compensate()` API | yes | yes | yes |
| `markCompensated` (auto) | yes | yes | yes |
| `CompensationStatus` lifecycle | yes | yes | yes |
| `compensatesWorkItemId` link | yes | yes | yes |
| Different actors (original vs compensating) | yes | | yes |
| Audit trail (both lifecycles) | yes | | |
| Multi-WorkItem reverse compensation | | yes | |
| CallerRef correlation | | yes | |
| COMPENSATING intermediate state query | | yes | yes |
| Guard: non-COMPLETED | | | yes |
| Guard: double-compensation | | | yes |
| Guard: compensator rejection | | | yes |
| Suspend/resume on compensating WI | | | yes |
| Single-transaction pattern | yes | | |
| Split-transaction pattern | | yes | yes |
| Application-driven ordering | | yes | |

---

## References

- `WorkItemService.java:772-822` — `compensate()` and `markCompensated()` methods
- `CompensationStatus.java` — NONE/COMPENSATING/COMPENSATED enum
- `CompensationLifecycleObserver.java` — auto-markCompensated on compensating WI completion
- `CancelScenario.java` — existing example pattern (REST endpoint, StepLog, response record)
- `examples/README.md` — scenario listing and curl commands
- `decisions.md` D1-D3 — module split, scenario design, transaction model
- Saga spec `decisions.md` D2, D13, D14 — compensation model invariants (terminal preservation, idempotency, no meta-compensation)
