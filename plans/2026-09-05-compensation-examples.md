# Compensation Examples Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #396 — saga: compensation examples — practical end-user scenarios
**Issue group:** #396

**Goal:** Add 3 runnable compensation scenarios to the examples module demonstrating
the full standalone compensation API surface.

**Architecture:** Each scenario is a JAX-RS endpoint class in the `examples/` module
following the existing pattern (CancelScenario, CreditDecisionScenario). Scenario 1
uses single-transaction. Scenarios 2 and 3 use split transactions via
`QuarkusTransaction.requiringNew()` to show intermediate COMPENSATING state.

**Tech Stack:** Java 21, Quarkus 3.32, casehub-work runtime, casehub-work-ledger,
H2 in-memory (dev), REST-assured + AssertJ (test)

## Global Constraints

- Follow existing example pattern: `@Path("/examples/<name>")`, `@POST @Path("/run")`,
  `StepLog` narrative, JSON response record
- No new dependencies — use only what `examples/pom.xml` already declares
- `findByCallerRef` for callerRef lookups (exact match — no pattern query available)
- Tests use `@QuarkusTest` + REST-assured POST + AssertJ assertions
- Scenario numbering continues from 17 (next: 18, 19, 20)

---

## Batch 1: Expense Approval Reversal (Scenario 18)

### Task 1: Expense Approval Reversal — scenario + response + test

**Files:**
- Create: `examples/src/main/java/io/casehub/work/examples/compensation/CompensationScenario.java`
- Create: `examples/src/main/java/io/casehub/work/examples/compensation/CompensationResponse.java`
- Create: `examples/src/test/java/io/casehub/work/examples/compensation/CompensationScenarioTest.java`

**Interfaces:**
- Consumes: `WorkItemService.create`, `.claim`, `.start`, `.complete`, `.compensate`, `.findById`; `AuditEntryStore.findByWorkItemId`
- Produces: `POST /examples/compensation/run` → `CompensationResponse` JSON

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.work.examples.compensation;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.restassured.response.Response;

@QuarkusTest
class CompensationScenarioTest {

    @Test
    void run_expenseCompensation_originalIsCompensated() {
        final Response response = given()
                .contentType(ContentType.JSON)
                .when()
                .post("/examples/compensation/run")
                .then()
                .statusCode(200)
                .extract().response();

        assertThat(response.jsonPath().getString("scenario")).isEqualTo("expense-compensation");

        // Original and compensating WorkItem IDs are present and different
        final String originalId = response.jsonPath().getString("originalWorkItemId");
        final String compensatingId = response.jsonPath().getString("compensatingWorkItemId");
        assertThat(originalId).isNotNull();
        assertThat(compensatingId).isNotNull();
        assertThat(originalId).isNotEqualTo(compensatingId);

        // Final compensation status
        assertThat(response.jsonPath().getString("compensationStatus")).isEqualTo("COMPENSATED");

        // Compensating WorkItem links back to original
        assertThat(response.jsonPath().getString("compensatingLink")).isEqualTo(originalId);

        // Different actors
        assertThat(response.jsonPath().getString("triggeredBy")).isEqualTo("internal-audit");
        assertThat(response.jsonPath().getString("reason")).contains("cancelled project");

        // Steps logged
        final List<Map<String, Object>> steps = response.jsonPath().getList("steps");
        assertThat(steps).hasSize(8);

        // Both audit trails present
        final List<Map<String, Object>> originalAudit = response.jsonPath().getList("originalAuditTrail");
        assertThat(originalAudit).isNotEmpty();
        assertThat(originalAudit.stream()
                .anyMatch(e -> "COMPENSATION_STARTED".equals(e.get("event")))).isTrue();
        assertThat(originalAudit.stream()
                .anyMatch(e -> "COMPENSATION_COMPLETED".equals(e.get("event")))).isTrue();

        final List<Map<String, Object>> compensatingAudit = response.jsonPath().getList("compensatingAuditTrail");
        assertThat(compensatingAudit).isNotEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CompensationScenarioTest -pl examples`
Expected: FAIL — 404 (endpoint doesn't exist)

- [ ] **Step 3: Write the response record**

```java
package io.casehub.work.examples.compensation;

import java.util.List;
import java.util.UUID;

import io.casehub.work.examples.StepLog;
import io.casehub.work.api.AuditEntryResponse;

public record CompensationResponse(
        String scenario,
        List<StepLog> steps,
        UUID originalWorkItemId,
        UUID compensatingWorkItemId,
        String compensationStatus,
        String compensatingLink,
        String triggeredBy,
        String reason,
        List<AuditEntryResponse> originalAuditTrail,
        List<AuditEntryResponse> compensatingAuditTrail) {
}
```

- [ ] **Step 4: Write the scenario class**

```java
package io.casehub.work.examples.compensation;

import java.util.ArrayList;
import java.util.List;

import io.casehub.work.api.CompensationStatus;
import io.casehub.work.api.WorkItem;
import io.casehub.work.api.AuditEntryResponse;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.examples.StepLog;
import io.casehub.work.runtime.model.AuditEntry;
import io.casehub.work.runtime.repository.AuditEntryStore;
import io.casehub.work.runtime.service.WorkItemService;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.jboss.logging.Logger;

@Path("/examples/compensation")
@Produces(MediaType.APPLICATION_JSON)
public class CompensationScenario {

    private static final Logger LOG = Logger.getLogger(CompensationScenario.class);
    private static final String SCENARIO_ID = "expense-compensation";

    @Inject
    WorkItemService workItemService;

    @Inject
    AuditEntryStore auditStore;

    @POST
    @Path("/run")
    @Transactional
    public CompensationResponse run() {
        final List<StepLog> steps = new ArrayList<>();
        final int total = 8;

        // Step 1: finance-analyst creates vendor payment approval
        LOG.infof("[SCENARIO] Step 1/%d: finance-analyst creates $50K vendor payment approval", total);
        final WorkItem original = workItemService.create(WorkItemCreateRequest.builder()
                .title("Vendor payment approval: $50,000 — Acme Consulting")
                .description("Invoice INV-2026-847 for cancelled Project Atlas consulting services")
                .types(List.of("finance", "approval"))
                .priority(WorkItemPriority.HIGH)
                .candidateUsers("senior-finance-officer")
                .createdBy("finance-analyst")
                .payload("{\"vendor\": \"Acme Consulting\", \"amount\": 50000, \"invoice\": \"INV-2026-847\"}")
                .build());
        steps.add(new StepLog(1, "finance-analyst creates $50K vendor payment approval", original.id()));

        // Step 2: senior-finance-officer claims
        LOG.infof("[SCENARIO] Step 2/%d: senior-finance-officer claims the approval", total);
        workItemService.claim(original.id(), "senior-finance-officer");
        steps.add(new StepLog(2, "senior-finance-officer claims the approval", original.id()));

        // Step 3: senior-finance-officer starts work
        LOG.infof("[SCENARIO] Step 3/%d: senior-finance-officer starts reviewing the invoice", total);
        workItemService.start(original.id(), "senior-finance-officer");
        steps.add(new StepLog(3, "senior-finance-officer starts reviewing the invoice", original.id()));

        // Step 4: senior-finance-officer approves
        LOG.infof("[SCENARIO] Step 4/%d: senior-finance-officer approves the payment", total);
        workItemService.complete(original.id(), "senior-finance-officer",
                "Approved — vendor invoice verified against PO-2026-312", "APPROVED");
        steps.add(new StepLog(4, "senior-finance-officer approves the payment", original.id()));

        // Step 5: internal-audit triggers compensation
        LOG.infof("[SCENARIO] Step 5/%d: internal-audit discovers Project Atlas was cancelled — triggers compensation", total);
        final WorkItem compensating = workItemService.compensate(
                original.id(),
                WorkItemCreateRequest.builder()
                        .title("Reverse payment approval: INV-2026-847 — Project Atlas cancelled")
                        .description("Original approval was for a cancelled project. Reverse the payment authorisation and notify the vendor.")
                        .types(List.of("finance", "compensation"))
                        .priority(WorkItemPriority.URGENT)
                        .candidateUsers("compliance-officer")
                        .createdBy("internal-audit")
                        .payload("{\"originalInvoice\": \"INV-2026-847\", \"reversalReason\": \"Project Atlas cancelled 2026-08-15\"}")
                        .build(),
                "internal-audit",
                "Invoice INV-2026-847 is for cancelled project Atlas — payment must be reversed");
        steps.add(new StepLog(5, "internal-audit triggers compensation — compensating WorkItem created", compensating.id()));

        // Step 6: compliance-officer claims the compensating WorkItem
        LOG.infof("[SCENARIO] Step 6/%d: compliance-officer claims the compensation task", total);
        workItemService.claim(compensating.id(), "compliance-officer");
        steps.add(new StepLog(6, "compliance-officer claims the compensation task", compensating.id()));

        // Step 7: compliance-officer completes the compensation
        LOG.infof("[SCENARIO] Step 7/%d: compliance-officer reverses the payment authorisation", total);
        workItemService.complete(compensating.id(), "compliance-officer",
                "Payment reversal authorised — vendor notified, ref REV-2026-019", "REVERSED");
        steps.add(new StepLog(7, "compliance-officer reverses the payment authorisation", compensating.id()));

        // Step 8: verify original is COMPENSATED
        LOG.infof("[SCENARIO] Step 8/%d: verify original WorkItem is now COMPENSATED", total);
        final WorkItem finalOriginal = workItemService.findById(original.id()).orElseThrow();
        steps.add(new StepLog(8, "original WorkItem compensationStatus: " + finalOriginal.compensationStatus(), original.id()));

        // Collect audit trails
        final List<AuditEntryResponse> originalAudit = auditStore.findByWorkItemId(original.id()).stream()
                .map(a -> new AuditEntryResponse(a.id, a.event, a.actor, a.detail, a.occurredAt))
                .toList();
        final List<AuditEntryResponse> compensatingAudit = auditStore.findByWorkItemId(compensating.id()).stream()
                .map(a -> new AuditEntryResponse(a.id, a.event, a.actor, a.detail, a.occurredAt))
                .toList();

        return new CompensationResponse(
                SCENARIO_ID,
                steps,
                original.id(),
                compensating.id(),
                finalOriginal.compensationStatus().name(),
                compensating.compensatesWorkItemId().toString(),
                "internal-audit",
                "Invoice INV-2026-847 is for cancelled project Atlas — payment must be reversed",
                originalAudit,
                compensatingAudit);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CompensationScenarioTest -pl examples`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add examples/src/main/java/io/casehub/work/examples/compensation/
git add examples/src/test/java/io/casehub/work/examples/compensation/
git commit -m "feat(#396): scenario 18 — expense approval reversal compensation example Refs #396 Refs #238"
```

---

## Batch 2: Multi-Step Loan Rollback (Scenario 19)

### Task 2: Loan Rollback — scenario + response + test

**Files:**
- Create: `examples/src/main/java/io/casehub/work/examples/loanrollback/LoanRollbackScenario.java`
- Create: `examples/src/main/java/io/casehub/work/examples/loanrollback/LoanRollbackResponse.java`
- Create: `examples/src/main/java/io/casehub/work/examples/loanrollback/LoanStepSummary.java`
- Create: `examples/src/test/java/io/casehub/work/examples/loanrollback/LoanRollbackScenarioTest.java`

**Interfaces:**
- Consumes: `WorkItemService.create`, `.claim`, `.complete`, `.compensate`, `.findById`, `.findByCallerRef`
- Produces: `POST /examples/loan-rollback/run` → `LoanRollbackResponse` JSON

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.work.examples.loanrollback;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.restassured.response.Response;

@QuarkusTest
class LoanRollbackScenarioTest {

    @Test
    void run_loanRollback_allThreeCompensatedInReverseOrder() {
        final Response response = given()
                .contentType(ContentType.JSON)
                .when()
                .post("/examples/loan-rollback/run")
                .then()
                .statusCode(200)
                .extract().response();

        assertThat(response.jsonPath().getString("scenario")).isEqualTo("loan-rollback");

        // 3 forward steps, 3 compensation steps
        final List<Map<String, Object>> forwardSteps = response.jsonPath().getList("forwardSteps");
        assertThat(forwardSteps).hasSize(3);

        final List<Map<String, Object>> compensationSteps = response.jsonPath().getList("compensationSteps");
        assertThat(compensationSteps).hasSize(3);

        // Compensation order is reverse: approval → valuation → credit-check
        assertThat(response.jsonPath().getString("compensationOrder"))
                .isEqualTo("approval → valuation → credit-check");

        // All originals are COMPENSATED
        for (final Map<String, Object> step : forwardSteps) {
            assertThat(step.get("compensationStatus")).isEqualTo("COMPENSATED");
        }

        // Each has a compensating WorkItem
        for (final Map<String, Object> step : forwardSteps) {
            assertThat(step.get("compensatingId")).isNotNull();
        }

        // Steps logged
        final List<Map<String, Object>> steps = response.jsonPath().getList("steps");
        assertThat(steps).hasSizeGreaterThanOrEqualTo(15);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LoanRollbackScenarioTest -pl examples`
Expected: FAIL — 404

- [ ] **Step 3: Write the response records**

```java
package io.casehub.work.examples.loanrollback;

import java.util.UUID;

public record LoanStepSummary(
        String callerRef,
        UUID originalId,
        UUID compensatingId,
        String compensationStatus) {
}
```

```java
package io.casehub.work.examples.loanrollback;

import java.util.List;

import io.casehub.work.examples.StepLog;

public record LoanRollbackResponse(
        String scenario,
        List<StepLog> steps,
        List<LoanStepSummary> forwardSteps,
        List<LoanStepSummary> compensationSteps,
        String compensationOrder) {
}
```

- [ ] **Step 4: Write the scenario class**

The scenario uses `QuarkusTransaction.requiringNew()` to split transactions so the intermediate COMPENSATING state is visible between compensation steps.

```java
package io.casehub.work.examples.loanrollback;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import io.casehub.work.api.CompensationStatus;
import io.casehub.work.api.WorkItem;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.examples.StepLog;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import jakarta.inject.Inject;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.jboss.logging.Logger;

@Path("/examples/loan-rollback")
@Produces(MediaType.APPLICATION_JSON)
public class LoanRollbackScenario {

    private static final Logger LOG = Logger.getLogger(LoanRollbackScenario.class);
    private static final String SCENARIO_ID = "loan-rollback";
    private static final String LOAN_REF = "loan:L-2026-001";

    @Inject
    WorkItemService workItemService;

    @POST
    @Path("/run")
    public LoanRollbackResponse run() {
        final List<StepLog> steps = new ArrayList<>();

        // --- Phase 1: Forward execution (3 WorkItems) ---
        // Each step in its own transaction so data is committed and visible

        final UUID creditId = QuarkusTransaction.requiringNew().call(() -> {
            LOG.info("[SCENARIO] Step 1: loan-system creates credit check");
            final WorkItem wi = workItemService.create(WorkItemCreateRequest.builder()
                    .title("Credit check — Loan L-2026-001")
                    .description("Run credit scoring for applicant using current bureau data")
                    .types(List.of("loan", "credit-check"))
                    .priority(WorkItemPriority.HIGH)
                    .candidateUsers("credit-analyst")
                    .createdBy("loan-system")
                    .callerRef(LOAN_REF + "/credit-check")
                    .build());
            steps.add(new StepLog(1, "loan-system creates credit check", wi.id()));
            return wi.id();
        });

        QuarkusTransaction.requiringNew().run(() -> {
            LOG.info("[SCENARIO] Step 2: credit-analyst claims and completes credit check");
            workItemService.claim(creditId, "credit-analyst");
            workItemService.complete(creditId, "credit-analyst", "Score: 720 — approved", "PASS");
            steps.add(new StepLog(2, "credit-analyst completes credit check (score: 720)", creditId));
        });

        final UUID valuationId = QuarkusTransaction.requiringNew().call(() -> {
            LOG.info("[SCENARIO] Step 3: loan-system creates property valuation");
            final WorkItem wi = workItemService.create(WorkItemCreateRequest.builder()
                    .title("Property valuation — Loan L-2026-001")
                    .description("On-site property assessment for 42 Oak Lane")
                    .types(List.of("loan", "valuation"))
                    .priority(WorkItemPriority.HIGH)
                    .candidateUsers("property-surveyor")
                    .createdBy("loan-system")
                    .callerRef(LOAN_REF + "/valuation")
                    .build());
            steps.add(new StepLog(3, "loan-system creates property valuation", wi.id()));
            return wi.id();
        });

        QuarkusTransaction.requiringNew().run(() -> {
            LOG.info("[SCENARIO] Step 4: property-surveyor claims and completes valuation");
            workItemService.claim(valuationId, "property-surveyor");
            workItemService.complete(valuationId, "property-surveyor", "Valued at £320,000 — within LTV", "PASS");
            steps.add(new StepLog(4, "property-surveyor completes valuation (£320,000)", valuationId));
        });

        final UUID approvalId = QuarkusTransaction.requiringNew().call(() -> {
            LOG.info("[SCENARIO] Step 5: loan-system creates final approval");
            final WorkItem wi = workItemService.create(WorkItemCreateRequest.builder()
                    .title("Final approval — Loan L-2026-001")
                    .description("Senior underwriter review and final loan approval")
                    .types(List.of("loan", "approval"))
                    .priority(WorkItemPriority.HIGH)
                    .candidateUsers("senior-underwriter")
                    .createdBy("loan-system")
                    .callerRef(LOAN_REF + "/approval")
                    .build());
            steps.add(new StepLog(5, "loan-system creates final approval", wi.id()));
            return wi.id();
        });

        QuarkusTransaction.requiringNew().run(() -> {
            LOG.info("[SCENARIO] Step 6: senior-underwriter claims and completes approval");
            workItemService.claim(approvalId, "senior-underwriter");
            workItemService.complete(approvalId, "senior-underwriter", "Loan approved — all checks passed", "APPROVED");
            steps.add(new StepLog(6, "senior-underwriter approves the loan", approvalId));
        });

        // --- Phase 2: Compensate in reverse order ---

        final String reason = "Credit check used outdated scoring data — regulatory audit finding RA-2026-031";
        final List<LoanStepSummary> compensationSteps = new ArrayList<>();

        // Compensate approval (step 3 of 3 — reverse order)
        final UUID compensatingApprovalId = QuarkusTransaction.requiringNew().call(() -> {
            LOG.info("[SCENARIO] Step 7: regulatory-audit compensates final approval");
            final WorkItem comp = workItemService.compensate(approvalId,
                    WorkItemCreateRequest.builder()
                            .title("Reverse loan approval — L-2026-001")
                            .types(List.of("loan", "compensation"))
                            .priority(WorkItemPriority.URGENT)
                            .candidateUsers("compliance-officer")
                            .createdBy("regulatory-audit")
                            .build(),
                    "regulatory-audit", reason);
            steps.add(new StepLog(7, "regulatory-audit compensates final approval", comp.id()));
            return comp.id();
        });

        // Verify intermediate COMPENSATING state
        QuarkusTransaction.requiringNew().run(() -> {
            final WorkItem approvalNow = workItemService.findById(approvalId).orElseThrow();
            LOG.infof("[SCENARIO] Step 8: approval compensationStatus = %s (intermediate)", approvalNow.compensationStatus());
            steps.add(new StepLog(8, "approval is " + approvalNow.compensationStatus() + " (intermediate state)", approvalId));
        });

        QuarkusTransaction.requiringNew().run(() -> {
            LOG.info("[SCENARIO] Step 9: compliance-officer completes compensating approval");
            workItemService.claim(compensatingApprovalId, "compliance-officer");
            workItemService.complete(compensatingApprovalId, "compliance-officer", "Approval reversed", "REVERSED");
            steps.add(new StepLog(9, "compliance-officer completes compensating approval", compensatingApprovalId));
        });

        // Compensate valuation
        final UUID compensatingValuationId = QuarkusTransaction.requiringNew().call(() -> {
            LOG.info("[SCENARIO] Step 10: regulatory-audit compensates valuation");
            final WorkItem comp = workItemService.compensate(valuationId,
                    WorkItemCreateRequest.builder()
                            .title("Invalidate property valuation — L-2026-001")
                            .types(List.of("loan", "compensation"))
                            .priority(WorkItemPriority.URGENT)
                            .candidateUsers("compliance-officer")
                            .createdBy("regulatory-audit")
                            .build(),
                    "regulatory-audit", reason);
            steps.add(new StepLog(10, "regulatory-audit compensates valuation", comp.id()));
            return comp.id();
        });

        QuarkusTransaction.requiringNew().run(() -> {
            workItemService.claim(compensatingValuationId, "compliance-officer");
            workItemService.complete(compensatingValuationId, "compliance-officer", "Valuation invalidated", "REVERSED");
            steps.add(new StepLog(11, "compliance-officer completes compensating valuation", compensatingValuationId));
        });

        // Compensate credit check
        final UUID compensatingCreditId = QuarkusTransaction.requiringNew().call(() -> {
            LOG.info("[SCENARIO] Step 12: regulatory-audit compensates credit check");
            final WorkItem comp = workItemService.compensate(creditId,
                    WorkItemCreateRequest.builder()
                            .title("Invalidate credit check — L-2026-001")
                            .types(List.of("loan", "compensation"))
                            .priority(WorkItemPriority.URGENT)
                            .candidateUsers("compliance-officer")
                            .createdBy("regulatory-audit")
                            .build(),
                    "regulatory-audit", reason);
            steps.add(new StepLog(12, "regulatory-audit compensates credit check", comp.id()));
            return comp.id();
        });

        QuarkusTransaction.requiringNew().run(() -> {
            workItemService.claim(compensatingCreditId, "compliance-officer");
            workItemService.complete(compensatingCreditId, "compliance-officer", "Credit check invalidated — rescore required", "REVERSED");
            steps.add(new StepLog(13, "compliance-officer completes compensating credit check", compensatingCreditId));
        });

        // --- Phase 3: Verify all compensated ---
        return QuarkusTransaction.requiringNew().call(() -> {
            final WorkItem finalCredit = workItemService.findById(creditId).orElseThrow();
            final WorkItem finalValuation = workItemService.findById(valuationId).orElseThrow();
            final WorkItem finalApproval = workItemService.findById(approvalId).orElseThrow();

            steps.add(new StepLog(14, "verify: credit-check=" + finalCredit.compensationStatus()
                    + ", valuation=" + finalValuation.compensationStatus()
                    + ", approval=" + finalApproval.compensationStatus(), null));

            final List<LoanStepSummary> forwardSteps = List.of(
                    new LoanStepSummary(LOAN_REF + "/credit-check", creditId, compensatingCreditId, finalCredit.compensationStatus().name()),
                    new LoanStepSummary(LOAN_REF + "/valuation", valuationId, compensatingValuationId, finalValuation.compensationStatus().name()),
                    new LoanStepSummary(LOAN_REF + "/approval", approvalId, compensatingApprovalId, finalApproval.compensationStatus().name()));

            return new LoanRollbackResponse(
                    SCENARIO_ID,
                    steps,
                    forwardSteps,
                    List.of(
                            new LoanStepSummary(LOAN_REF + "/approval", approvalId, compensatingApprovalId, finalApproval.compensationStatus().name()),
                            new LoanStepSummary(LOAN_REF + "/valuation", valuationId, compensatingValuationId, finalValuation.compensationStatus().name()),
                            new LoanStepSummary(LOAN_REF + "/credit-check", creditId, compensatingCreditId, finalCredit.compensationStatus().name())),
                    "approval → valuation → credit-check");
        });
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LoanRollbackScenarioTest -pl examples`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add examples/src/main/java/io/casehub/work/examples/loanrollback/
git add examples/src/test/java/io/casehub/work/examples/loanrollback/
git commit -m "feat(#396): scenario 19 — multi-step loan rollback compensation example Refs #396 Refs #238"
```

---

## Batch 3: Compensation Resilience (Scenario 20) + README update

### Task 3: Compensation Resilience — scenario + response + test

**Files:**
- Create: `examples/src/main/java/io/casehub/work/examples/resilience/CompensationResilienceScenario.java`
- Create: `examples/src/main/java/io/casehub/work/examples/resilience/CompensationResilienceResponse.java`
- Create: `examples/src/main/java/io/casehub/work/examples/resilience/GuardResult.java`
- Create: `examples/src/main/java/io/casehub/work/examples/resilience/LifecycleResult.java`
- Create: `examples/src/test/java/io/casehub/work/examples/resilience/CompensationResilienceScenarioTest.java`

**Interfaces:**
- Consumes: `WorkItemService.create`, `.claim`, `.start`, `.complete`, `.compensate`, `.suspend`, `.resume`, `.findById`
- Produces: `POST /examples/compensation-resilience/run` → `CompensationResilienceResponse` JSON

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.work.examples.resilience;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.restassured.response.Response;

@QuarkusTest
class CompensationResilienceScenarioTest {

    @Test
    void run_compensationResilience_allGuardsAndLifecycle() {
        final Response response = given()
                .contentType(ContentType.JSON)
                .when()
                .post("/examples/compensation-resilience/run")
                .then()
                .statusCode(200)
                .extract().response();

        assertThat(response.jsonPath().getString("scenario")).isEqualTo("compensation-resilience");

        // Guard 1: non-COMPLETED
        assertThat(response.jsonPath().getBoolean("nonCompletedGuard.guardTriggered")).isTrue();
        assertThat(response.jsonPath().getString("nonCompletedGuard.errorMessage"))
                .contains("Only COMPLETED");

        // Guard 2: double-compensation
        assertThat(response.jsonPath().getBoolean("doubleCompensationGuard.guardTriggered")).isTrue();
        assertThat(response.jsonPath().getString("doubleCompensationGuard.errorMessage"))
                .contains("already has compensation activity");

        // Guard 3: compensator rejection
        assertThat(response.jsonPath().getBoolean("compensatorGuard.guardTriggered")).isTrue();
        assertThat(response.jsonPath().getString("compensatorGuard.errorMessage"))
                .contains("cannot themselves be compensated");

        // Lifecycle: suspend → resume → complete
        assertThat(response.jsonPath().getString("suspendResumeLifecycle.finalCompensationStatus"))
                .isEqualTo("COMPENSATED");
        final List<String> statuses = response.jsonPath().getList("suspendResumeLifecycle.compensatingStatuses");
        assertThat(statuses).contains("SUSPENDED");

        // Steps logged
        final List<Map<String, Object>> steps = response.jsonPath().getList("steps");
        assertThat(steps).hasSizeGreaterThanOrEqualTo(10);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CompensationResilienceScenarioTest -pl examples`
Expected: FAIL — 404

- [ ] **Step 3: Write the response records**

```java
package io.casehub.work.examples.resilience;

public record GuardResult(
        String description,
        boolean guardTriggered,
        String errorMessage) {
}
```

```java
package io.casehub.work.examples.resilience;

import java.util.List;
import java.util.UUID;

public record LifecycleResult(
        UUID originalId,
        UUID compensatingId,
        List<String> compensatingStatuses,
        String finalCompensationStatus) {
}
```

```java
package io.casehub.work.examples.resilience;

import java.util.List;

import io.casehub.work.examples.StepLog;

public record CompensationResilienceResponse(
        String scenario,
        List<StepLog> steps,
        GuardResult nonCompletedGuard,
        GuardResult doubleCompensationGuard,
        GuardResult compensatorGuard,
        LifecycleResult suspendResumeLifecycle) {
}
```

- [ ] **Step 4: Write the scenario class**

```java
package io.casehub.work.examples.resilience;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import io.casehub.work.api.WorkItem;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.examples.StepLog;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import jakarta.inject.Inject;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.jboss.logging.Logger;

@Path("/examples/compensation-resilience")
@Produces(MediaType.APPLICATION_JSON)
public class CompensationResilienceScenario {

    private static final Logger LOG = Logger.getLogger(CompensationResilienceScenario.class);
    private static final String SCENARIO_ID = "compensation-resilience";

    @Inject
    WorkItemService workItemService;

    @POST
    @Path("/run")
    public CompensationResilienceResponse run() {
        final List<StepLog> steps = new ArrayList<>();

        // --- Sub-scenario 1: Guard — cannot compensate non-COMPLETED ---
        final GuardResult nonCompletedGuard = QuarkusTransaction.requiringNew().call(() -> {
            LOG.info("[SCENARIO] Sub-scenario 1: Guard — cannot compensate non-COMPLETED WorkItem");
            final WorkItem referral = workItemService.create(WorkItemCreateRequest.builder()
                    .title("Patient referral: Smith → Cardiology")
                    .types(List.of("referral"))
                    .priority(WorkItemPriority.MEDIUM)
                    .candidateUsers("clinic-admin")
                    .createdBy("gp-reception")
                    .build());
            steps.add(new StepLog(1, "gp-reception creates referral (status: CREATED)", referral.id()));

            try {
                workItemService.compensate(referral.id(),
                        WorkItemCreateRequest.builder()
                                .title("Cancel referral").types(List.of("compensation"))
                                .createdBy("clinic-admin").build(),
                        "clinic-admin", "test");
                return new GuardResult("Cannot compensate non-COMPLETED WorkItem", false, "Guard did not fire");
            } catch (IllegalStateException e) {
                steps.add(new StepLog(2, "Guard triggered: " + e.getMessage(), referral.id()));
                return new GuardResult("Cannot compensate non-COMPLETED WorkItem", true, e.getMessage());
            }
        });

        // --- Sub-scenario 2: Happy path + Guard — cannot double-compensate ---
        final GuardResult doubleCompensationGuard;
        final UUID compensatingWiId;
        final UUID completedReferralId;

        // Create and complete a referral
        completedReferralId = QuarkusTransaction.requiringNew().call(() -> {
            final WorkItem referral = workItemService.create(WorkItemCreateRequest.builder()
                    .title("Patient referral: Jones → Orthopaedics")
                    .types(List.of("referral"))
                    .priority(WorkItemPriority.MEDIUM)
                    .candidateUsers("clinic-admin")
                    .createdBy("gp-reception")
                    .build());
            workItemService.claim(referral.id(), "clinic-admin");
            workItemService.start(referral.id(), "clinic-admin");
            workItemService.complete(referral.id(), "clinic-admin", "Referral accepted", "ACCEPTED");
            steps.add(new StepLog(3, "gp-reception creates and completes referral", referral.id()));
            return referral.id();
        });

        // Compensate it
        compensatingWiId = QuarkusTransaction.requiringNew().call(() -> {
            final WorkItem comp = workItemService.compensate(completedReferralId,
                    WorkItemCreateRequest.builder()
                            .title("Cancel referral: Jones — specialist unavailable")
                            .types(List.of("referral", "compensation"))
                            .priority(WorkItemPriority.HIGH)
                            .candidateUsers("clinic-admin")
                            .createdBy("clinic-admin")
                            .build(),
                    "clinic-admin", "Specialist unavailable — referral must be redirected");
            steps.add(new StepLog(4, "clinic-admin compensates the referral", comp.id()));

            // Complete the compensating WorkItem
            workItemService.claim(comp.id(), "clinic-admin");
            workItemService.complete(comp.id(), "clinic-admin", "Referral cancelled and patient notified", "CANCELLED");
            steps.add(new StepLog(5, "clinic-admin completes compensation", comp.id()));
            return comp.id();
        });

        // Try to compensate again
        doubleCompensationGuard = QuarkusTransaction.requiringNew().call(() -> {
            try {
                workItemService.compensate(completedReferralId,
                        WorkItemCreateRequest.builder()
                                .title("Second compensation attempt").types(List.of("compensation"))
                                .createdBy("clinic-admin").build(),
                        "clinic-admin", "test");
                return new GuardResult("Cannot double-compensate", false, "Guard did not fire");
            } catch (IllegalStateException e) {
                steps.add(new StepLog(6, "Guard triggered: " + e.getMessage(), completedReferralId));
                return new GuardResult("Cannot double-compensate", true, e.getMessage());
            }
        });

        // --- Sub-scenario 3: Guard — cannot compensate a compensating WorkItem ---
        final UUID finalCompensatingWiId = compensatingWiId;
        final GuardResult compensatorGuard = QuarkusTransaction.requiringNew().call(() -> {
            try {
                workItemService.compensate(finalCompensatingWiId,
                        WorkItemCreateRequest.builder()
                                .title("Meta-compensation attempt").types(List.of("compensation"))
                                .createdBy("clinic-admin").build(),
                        "clinic-admin", "test");
                return new GuardResult("Cannot compensate a compensating WorkItem", false, "Guard did not fire");
            } catch (IllegalStateException e) {
                steps.add(new StepLog(7, "Guard triggered: " + e.getMessage(), finalCompensatingWiId));
                return new GuardResult("Cannot compensate a compensating WorkItem", true, e.getMessage());
            }
        });

        // --- Sub-scenario 4: Lifecycle — suspend → resume → complete ---
        final LifecycleResult lifecycleResult;

        final UUID lifecycleOriginalId = QuarkusTransaction.requiringNew().call(() -> {
            final WorkItem referral = workItemService.create(WorkItemCreateRequest.builder()
                    .title("Patient referral: Brown → Neurology")
                    .types(List.of("referral"))
                    .priority(WorkItemPriority.MEDIUM)
                    .candidateUsers("clinic-admin")
                    .createdBy("gp-reception")
                    .build());
            workItemService.claim(referral.id(), "clinic-admin");
            workItemService.start(referral.id(), "clinic-admin");
            workItemService.complete(referral.id(), "clinic-admin", "Referral sent", "SENT");
            steps.add(new StepLog(8, "create and complete another referral", referral.id()));
            return referral.id();
        });

        final UUID lifecycleCompId = QuarkusTransaction.requiringNew().call(() -> {
            final WorkItem comp = workItemService.compensate(lifecycleOriginalId,
                    WorkItemCreateRequest.builder()
                            .title("Recall referral: Brown — incorrect specialist")
                            .types(List.of("referral", "compensation"))
                            .priority(WorkItemPriority.HIGH)
                            .candidateUsers("clinic-admin")
                            .createdBy("clinic-admin")
                            .build(),
                    "clinic-admin", "Wrong specialist — should be Neurosurgery not Neurology");
            steps.add(new StepLog(9, "compensate the referral", comp.id()));
            return comp.id();
        });

        final List<String> statusTrail = new ArrayList<>();

        // Claim
        QuarkusTransaction.requiringNew().run(() -> {
            workItemService.claim(lifecycleCompId, "clinic-admin");
            workItemService.start(lifecycleCompId, "clinic-admin");
            statusTrail.add(workItemService.findById(lifecycleCompId).orElseThrow().status().name());
        });

        // Suspend — doctor on leave
        QuarkusTransaction.requiringNew().run(() -> {
            workItemService.suspend(lifecycleCompId, "clinic-admin", "Doctor on leave — will resume Monday");
            statusTrail.add(workItemService.findById(lifecycleCompId).orElseThrow().status().name());
            steps.add(new StepLog(10, "compensating WorkItem suspended — doctor on leave", lifecycleCompId));
        });

        // Verify original is still COMPENSATING while suspended
        QuarkusTransaction.requiringNew().run(() -> {
            final WorkItem orig = workItemService.findById(lifecycleOriginalId).orElseThrow();
            steps.add(new StepLog(11, "original is still " + orig.compensationStatus() + " while compensating WI is suspended", lifecycleOriginalId));
        });

        // Resume — different doctor
        QuarkusTransaction.requiringNew().run(() -> {
            workItemService.resume(lifecycleCompId, "senior-doctor");
            statusTrail.add(workItemService.findById(lifecycleCompId).orElseThrow().status().name());
            steps.add(new StepLog(12, "senior-doctor resumes the compensating WorkItem", lifecycleCompId));
        });

        // Complete
        QuarkusTransaction.requiringNew().run(() -> {
            workItemService.complete(lifecycleCompId, "senior-doctor",
                    "Referral recalled — patient redirected to Neurosurgery", "RECALLED");
            statusTrail.add(workItemService.findById(lifecycleCompId).orElseThrow().status().name());
            steps.add(new StepLog(13, "senior-doctor completes — original auto-COMPENSATED", lifecycleCompId));
        });

        // Final verification
        lifecycleResult = QuarkusTransaction.requiringNew().call(() -> {
            final WorkItem finalOrig = workItemService.findById(lifecycleOriginalId).orElseThrow();
            return new LifecycleResult(lifecycleOriginalId, lifecycleCompId, statusTrail, finalOrig.compensationStatus().name());
        });

        return new CompensationResilienceResponse(
                SCENARIO_ID,
                steps,
                nonCompletedGuard,
                doubleCompensationGuard,
                compensatorGuard,
                lifecycleResult);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CompensationResilienceScenarioTest -pl examples`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add examples/src/main/java/io/casehub/work/examples/resilience/
git add examples/src/test/java/io/casehub/work/examples/resilience/
git commit -m "feat(#396): scenario 20 — compensation resilience (guards + lifecycle) example Refs #396 Refs #238"
```

### Task 4: Update README and run full test suite

**Files:**
- Modify: `examples/README.md`

**Interfaces:**
- Consumes: All 3 scenarios from Tasks 1-3
- Produces: Updated README with scenarios 18-20

- [ ] **Step 1: Add scenarios 18-20 to the README scenarios table**

After row 17 (Vocabulary Registration), add:

```markdown
| 18 | Expense Compensation | `POST /examples/compensation/run` | Full compensation lifecycle — create, approve, trigger compensation, different actor reverses, auto-COMPENSATED. Audit trail for both original and compensating WorkItems |
| 19 | Loan Rollback | `POST /examples/loan-rollback/run` | Multi-step reverse-order compensation — 3 callerRef-correlated WorkItems compensated in dependency-reverse order with observable intermediate COMPENSATING state |
| 20 | Compensation Resilience | `POST /examples/compensation-resilience/run` | All 3 compensation guards (non-COMPLETED, double-compensation, compensator rejection) plus suspend/resume lifecycle on compensating WorkItem |
```

- [ ] **Step 2: Add curl commands to the "Running All Scenarios" section**

```bash
curl -s -X POST http://localhost:8080/examples/compensation/run | jq '.scenario'
curl -s -X POST http://localhost:8080/examples/loan-rollback/run | jq '.scenario'
curl -s -X POST http://localhost:8080/examples/compensation-resilience/run | jq '.scenario'
```

- [ ] **Step 3: Update the expected test count**

Change "Expected: 30 tests, 0 failures." to "Expected: 33 tests, 0 failures." (or whatever the actual count is after running the full suite).

- [ ] **Step 4: Run the full examples test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples`
Expected: All tests pass (including existing 30 + 3 new)

- [ ] **Step 5: Commit**

```bash
git add examples/README.md
git commit -m "docs(#396): add compensation scenarios 18-20 to examples README Refs #396 Refs #238"
```

---

## References

- `specs/issue-396-compensation-examples/2026-09-05-compensation-examples-design.md` — design spec
- `specs/issue-396-compensation-examples/decisions.md` — D1-D3
- `WorkItemService.java:772-822` — `compensate()` and `markCompensated()`
- `CompensationStatus.java` — NONE/COMPENSATING/COMPENSATED
- `CompensationLifecycleObserver.java` — auto-markCompensated
- `CancelScenario.java` — existing example pattern
- `CancelScenarioTest.java` — existing test pattern
- `WorkItemQuery.java` — query capabilities (no callerRef filter)
- GitHub casehubio/work#396, casehubio/work#238
