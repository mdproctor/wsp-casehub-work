# Queue Health and Enriched List Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #357 — Add global queue health and enriched list endpoints for console UI
**Issue group:** #357

**Goal:** Add per-queue summary counts to the queue list endpoint and a global health KPI endpoint for the scaffold console's Queues tab.

**Architecture:** Both changes are additions to `QueueResource` in the `casehub-work-queues` module. The enriched list nests the existing `WorkItemSummary` per queue (already cached via `QueueMembershipService.summarize()`). The health endpoint aggregates those per-queue summaries into a KPI metrics array. No new modules, migrations, or caching changes.

**Tech Stack:** Java 21, Quarkus, JAX-RS, RestAssured

## Global Constraints

- All code in `casehub-work-queues` module — no cross-module changes
- Reuse `QueueMembershipService.summarize()` — do not duplicate aggregation logic
- Follow existing response patterns (`QueueTrendResponse` record style)
- Tests use `@QuarkusTest` + RestAssured (existing pattern in `QueueResourceTest`)

---

## Batch 1: Enriched list and health endpoints

### Task 1: Enrich `GET /queues` list with per-queue summary

**Files:**
- Modify: `queues/src/main/java/io/casehub/work/queues/api/QueueResource.java` — `list()` method
- Modify: `queues/src/test/java/io/casehub/work/queues/api/QueueResourceTest.java` — add test
- Modify: `docs/api-reference.md` — update `GET /queues` documentation

**Interfaces:**
- Consumes: `QueueMembershipService.summarize(SubjectViewSpec, Instant)` → `WorkItemSummary`
- Produces: `GET /queues` response now includes `summary` field per queue entry

- [ ] **Step 1: Write the failing test**

Add to `QueueResourceTest.java`:

```java
@Test
void listQueues_includesSummaryPerQueue() {
    // Create a label rule so WorkItems get labels matching the queue
    given().contentType(ContentType.JSON)
           .body("""
                 {"name":"Enriched list rule","scope":"ORG","conditionLanguage":"jexl",
                  "conditionExpression":"priority == 'HIGH'",
                  "actions":[{"type":"Add","label":"enriched-test/items"}]}""")
           .post("/label-rules").then().statusCode(201);

    // Create a queue
    given().contentType(ContentType.JSON)
           .body("""
                 {"name":"Enriched list queue","labelPattern":"enriched-test/**","scope":"ORG"}""")
           .post("/queues").then().statusCode(201);

    // Create a WorkItem that will be labelled into the queue
    given().contentType(ContentType.JSON)
           .body("""
                 {"title":"Enriched list item","priority":"HIGH","createdBy":"alice"}""")
           .post("/workitems").then().statusCode(201);

    // Verify list response includes summary
    given().get("/queues").then()
           .statusCode(200)
           .body("find { it.name == 'Enriched list queue' }.summary.total",
                 greaterThanOrEqualTo(1))
           .body("find { it.name == 'Enriched list queue' }.summary.byStatus.PENDING",
                 greaterThanOrEqualTo(1));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QueueResourceTest#listQueues_includesSummaryPerQueue -pl queues`
Expected: FAIL — `summary` field not present in response

- [ ] **Step 3: Implement enriched list**

Modify `QueueResource.list()` to call `membershipService.summarize()` for each queue and include the result:

```java
@GET
@Transactional
public List<Map<String, Object>> list() {
    final Instant now = Instant.now();
    return viewStore.findByTenancy(currentPrincipal.tenancyId()).stream()
                    .map(q -> {
                        final WorkItemSummary summary = membershipService.summarize(q, now);
                        return Map.<String, Object>of(
                                "id", q.id(), "name", q.name(),
                                "labelPattern", q.labelPattern(),
                                "scope", q.scope() != null ? q.scope().value() : "/",
                                "summary", summary);
                    })
                    .toList();
}
```

Add import: `import io.casehub.work.api.WorkItemSummary;`
Add import: `import java.time.Instant;` (already present — verify)

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QueueResourceTest#listQueues_includesSummaryPerQueue -pl queues`
Expected: PASS

- [ ] **Step 5: Run all QueueResourceTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QueueResourceTest -pl queues`
Expected: All pass — existing `listQueues_returnsCreated` test still works (it checks `name` field, not `summary`)

- [ ] **Step 6: Update api-reference.md**

In the `GET /queues` section, update the response body documentation to include the `summary` field. Add a field table showing the `WorkItemSummary` nested under each queue entry.

- [ ] **Step 7: Commit**

```bash
git add queues/src/main/java/io/casehub/work/queues/api/QueueResource.java \
       queues/src/test/java/io/casehub/work/queues/api/QueueResourceTest.java \
       docs/api-reference.md
git commit -m "feat(#357): enrich GET /queues with per-queue WorkItemSummary

Each queue entry in the list response now includes a summary field
with total, byStatus, byPriority, overdue, claimDeadlineBreached,
and oldestCreatedAt. Reuses cached QueueMembershipService.summarize().

Refs #357"
```

---

### Task 2: Add `GET /queues/health` KPI endpoint

**Files:**
- Create: `queues/src/main/java/io/casehub/work/queues/api/QueueHealthMetric.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/api/QueueResource.java` — add `health()` method
- Create: `queues/src/test/java/io/casehub/work/queues/api/QueueHealthResourceTest.java`
- Modify: `docs/api-reference.md` — add `GET /queues/health` documentation

**Interfaces:**
- Consumes: `QueueMembershipService.summarize(SubjectViewSpec, Instant)` → `WorkItemSummary`
- Consumes: `SubjectViewStore.findByTenancy(String)` → `List<SubjectViewSpec>`
- Produces: `GET /queues/health` → `List<QueueHealthMetric>`

- [ ] **Step 1: Write the failing test**

Create `QueueHealthResourceTest.java`:

```java
package io.casehub.work.queues.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class QueueHealthResourceTest {

    @Test
    void health_returnsKpiMetrics() {
        given().contentType(ContentType.JSON)
               .body("""
                     {"name":"Health rule","scope":"ORG","conditionLanguage":"jexl",
                      "conditionExpression":"priority == 'HIGH' || priority == 'MEDIUM'",
                      "actions":[{"type":"Add","label":"health-test/items"}]}""")
               .post("/label-rules").then().statusCode(201);

        given().contentType(ContentType.JSON)
               .body("""
                     {"name":"Health test queue","labelPattern":"health-test/**","scope":"ORG"}""")
               .post("/queues").then().statusCode(201);

        given().contentType(ContentType.JSON)
               .body("""
                     {"title":"Health item 1","priority":"HIGH","createdBy":"alice"}""")
               .post("/workitems").then().statusCode(201);

        given().contentType(ContentType.JSON)
               .body("""
                     {"title":"Health item 2","priority":"MEDIUM","createdBy":"bob"}""")
               .post("/workitems").then().statusCode(201);

        given().get("/queues/health").then()
               .statusCode(200)
               .body("key", hasItems("total", "pending", "active", "overdue", "breached"))
               .body("find { it.key == 'total' }.value", greaterThanOrEqualTo(2))
               .body("find { it.key == 'pending' }.value", greaterThanOrEqualTo(2))
               .body("find { it.key == 'total' }.label", equalTo("Total"))
               .body("find { it.key == 'pending' }.status", equalTo("warning"));
    }

    @Test
    void health_noQueues_returnsZeroMetrics() {
        // Health with existing queues but the totals include zeros
        given().get("/queues/health").then()
               .statusCode(200)
               .body("key", hasItems("total", "pending", "active", "overdue", "breached"))
               .body("size()", equalTo(5));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QueueHealthResourceTest#health_returnsKpiMetrics -pl queues`
Expected: FAIL — 404 (endpoint doesn't exist)

- [ ] **Step 3: Create `QueueHealthMetric` record**

Create `queues/src/main/java/io/casehub/work/queues/api/QueueHealthMetric.java`:

```java
package io.casehub.work.queues.api;

public record QueueHealthMetric(String key, long value, String label, String status) {}
```

- [ ] **Step 4: Implement health endpoint**

Add to `QueueResource.java`:

```java
@GET
@Path("/health")
@Transactional
public List<QueueHealthMetric> health() {
    final Instant now = Instant.now();
    final var queues = viewStore.findByTenancy(currentPrincipal.tenancyId());

    long total = 0;
    long pending = 0;
    long active = 0;
    long overdue = 0;
    long breached = 0;

    for (final var queue : queues) {
        final var summary = membershipService.summarize(queue, now);
        total += summary.total();
        pending += summary.byStatus().getOrDefault("PENDING", 0L);
        active += summary.byStatus().getOrDefault("IN_PROGRESS", 0L)
                + summary.byStatus().getOrDefault("ASSIGNED", 0L);
        overdue += summary.overdue();
        breached += summary.claimDeadlineBreached();
    }

    return List.of(
            new QueueHealthMetric("total", total, "Total",
                    "neutral"),
            new QueueHealthMetric("pending", pending, "Pending",
                    pending > 0 ? "warning" : "neutral"),
            new QueueHealthMetric("active", active, "Active",
                    "neutral"),
            new QueueHealthMetric("overdue", overdue, "Overdue",
                    overdue > 0 ? "critical" : "neutral"),
            new QueueHealthMetric("breached", breached, "Claim SLA",
                    breached > 0 ? "critical" : "neutral"));
}
```

**JAX-RS path ordering note:** `@Path("/health")` must be registered before `@Path("/{id}")` to avoid the `{id}` path param consuming "health" as a UUID (which would 404). JAX-RS resolves literal segments before parameterised ones, so placing the method anywhere in the class is fine — no reordering needed.

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=QueueHealthResourceTest -pl queues`
Expected: PASS

- [ ] **Step 6: Run full queues module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues`
Expected: All pass — no regressions

- [ ] **Step 7: Update api-reference.md**

Add `GET /queues/health` section after the existing `GET /queues` section. Document the response format with the `QueueHealthMetric` fields (`key`, `value`, `label`, `status`) and the status threshold rules.

- [ ] **Step 8: Commit**

```bash
git add queues/src/main/java/io/casehub/work/queues/api/QueueHealthMetric.java \
       queues/src/main/java/io/casehub/work/queues/api/QueueResource.java \
       queues/src/test/java/io/casehub/work/queues/api/QueueHealthResourceTest.java \
       docs/api-reference.md
git commit -m "feat(#357): add GET /queues/health KPI endpoint

Aggregates per-queue cached summaries into a KPI metrics array
with total, pending, active, overdue, and claim SLA breach counts.
Status thresholds: overdue/breached > 0 → critical, pending > 0 → warning.

Refs #357"
```

---

## References

- [2026-08-17-queue-health-enriched-list-design.md] — design spec
- [QueueResource.java] — existing queue REST resource
- [QueueMembershipService.java:33] — cached summarize()
- [WorkItemSummary.java] — summary record
- [QueueResourceTest.java] — existing test patterns
- [QueueSummaryResourceTest.java] — summary test patterns
- [QueueTrendResponse.java] — response record pattern
- [PP-20260616-4896da] — queue-filter-scope-management-only protocol
- [GitHub #357] — issue
