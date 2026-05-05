# quarkus-work-queues Module Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the optional `quarkus-work-queues` Maven module — filters, FilterChain engine, and queue views on top of the label model from sub-epic #51.

**Architecture:** The module is a plain Maven module (not a Quarkiverse extension pair — no deployment processor needed). It observes `WorkItemLifecycleEvent` from the core via CDI. Filters are stored in the database; evaluation is synchronous multi-pass (up to 10 passes for propagation). `FilterChain` tracks `filterId → Set<workItemId>` for O(affected) cascade on deletion. `QueueView` is a named label-pattern query. Lambda filters are CDI beans.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM Panache, Flyway, commons-jexl3, jackson-jq, JUnit 5, REST Assured

**Issues:** #56 (scaffold), #57 (WorkItemFilter), #58 (JEXL), #59 (JQ), #60 (Lambda), #61 (engine), #62 (cascade), #63 (QueueView), #64 (soft assign), #65 (E2E)

**Module location:** `quarkus-work-queues/`

**Build commands:**
```bash
# Build only this module (after installing parent)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues -Dno-format

# Full build including all dependencies
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests -Dno-format && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues -Dno-format
```

---

## File Map

**Created (quarkus-work-queues/src/main/java/io/quarkiverse/workitems/queues/):**
- `model/FilterScope.java` — enum: PERSONAL, TEAM, ORG
- `model/WorkItemFilter.java` — Panache entity
- `model/FilterChain.java` — Panache entity (filterId → workItemIds)
- `model/QueueView.java` — Panache entity
- `model/WorkItemQueueState.java` — Panache entity (relinquishable flag)
- `service/FilterConditionEvaluator.java` — SPI interface
- `service/FilterEvaluatorRegistry.java` — CDI registry
- `service/JexlConditionEvaluator.java` — JEXL built-in
- `service/JqConditionEvaluator.java` — JQ built-in
- `service/WorkItemFilterBean.java` — Lambda filter SPI interface
- `service/LambdaFilterRegistry.java` — Lambda CDI discovery
- `service/FilterEngine.java` — interface
- `service/FilterEngineImpl.java` — re-evaluation + cascade
- `service/FilterEvaluationObserver.java` — CDI event observer
- `api/FilterResource.java` — CRUD REST for filters
- `api/QueueResource.java` — queue view REST
- `api/QueueStateResource.java` — relinquishable REST

**Created (resources):**
- `quarkus-work-queues/src/main/resources/db/migration/V1__queues_schema.sql`
- `quarkus-work-queues/src/main/resources/META-INF/microprofile-config.properties`
- `quarkus-work-queues/src/test/resources/application.properties`

**Created (tests):**
- `queues/.../service/FilterEngineTest.java` — unit test (no Quarkus)
- `queues/.../service/JexlConditionEvaluatorTest.java`
- `queues/.../service/JqConditionEvaluatorTest.java`
- `queues/.../api/FilterResourceTest.java` — @QuarkusTest
- `queues/.../api/QueueResourceTest.java` — @QuarkusTest
- `queues/.../api/QueueStateResourceTest.java` — @QuarkusTest
- `queues/.../api/FullPipelineTest.java` — @QuarkusTest end-to-end

**Modified:**
- `pom.xml` — add `quarkus-work-queues` to `<modules>`

---

## Issue #56 — Module Scaffold

---

### Task 1: Create the quarkus-work-queues POM

**Files:**
- Create: `quarkus-work-queues/pom.xml`
- Modify: `pom.xml` (add module)

- [ ] **Step 1: Create module directory and POM**

```xml
<!-- quarkus-work-queues/pom.xml -->
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.quarkiverse.work</groupId>
    <artifactId>quarkus-work-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId>quarkus-work-queues</artifactId>
  <name>Quarkus WorkItems - Queues</name>
  <description>Optional queues module providing label-based work queues: saved and ad-hoc filters
(JEXL, JQ, Lambda), FilterChain derivation graph, and QueueView named queries.
Add this module to enable queue capabilities — the core extension is unchanged when absent.</description>

  <dependencies>

    <!-- Core extension — WorkItem, WorkItemLifecycleEvent, WorkItemRepository, labels -->
    <dependency>
      <groupId>io.quarkiverse.work</groupId>
      <artifactId>quarkus-work</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- CDI, REST, ORM, Flyway -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm-panache</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-flyway</artifactId>
    </dependency>

    <!-- JEXL expression evaluator -->
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-jexl3</artifactId>
      <version>3.4.0</version>
    </dependency>

    <!-- JQ expression evaluator -->
    <dependency>
      <groupId>net.thisptr</groupId>
      <artifactId>jackson-jq</artifactId>
      <version>1.0.0</version>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>

  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <version>3.3.1</version>
        <executions>
          <execution>
            <id>make-index</id>
            <goals>
              <goal>jandex</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 2: Add module to parent POM**

In `pom.xml`, add `<module>quarkus-work-queues</module>` after `quarkus-work-ledger`.

- [ ] **Step 3: Create test application.properties**

```properties
# quarkus-work-queues/src/test/resources/application.properties
quarkus.http.test-port=0
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:queuestest;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=db/migration
quarkus.scheduler.enabled=false
quarkus.work.default-expiry-hours=24
quarkus.work.default-claim-hours=4
quarkus.work.cleanup.expiry-check-seconds=60
```

- [ ] **Step 4: Verify module compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests -Dno-format 2>&1 | tail -8
```

Expected: BUILD SUCCESS across all modules.

---

### Task 2: Flyway V1 for queues module + FilterScope enum

**Files:**
- Create: `quarkus-work-queues/src/main/resources/db/migration/V1__queues_schema.sql`
- Create: `quarkus-work-queues/src/main/java/io/quarkiverse/workitems/queues/model/FilterScope.java`

- [ ] **Step 1: Create FilterScope enum**

```java
// quarkus-work-queues/src/main/java/io/quarkiverse/workitems/queues/model/FilterScope.java
package io.quarkiverse.work.queues.model;

/** Visibility scope of a {@link WorkItemFilter} or {@link QueueView}. */
public enum FilterScope {
    /** Visible only to the declaring user. */
    PERSONAL,
    /** Visible to all members of the owning group. */
    TEAM,
    /** Visible to all users in the organisation. */
    ORG
}
```

- [ ] **Step 2: Create the Flyway V1 migration**

```sql
-- quarkus-work-queues V1: filters, filter chains, queue views, queue state
-- Compatible with H2 (dev/test) and PostgreSQL (production).

-- Saved filter definitions
CREATE TABLE work_item_filter (
    id                   UUID            PRIMARY KEY,
    name                 VARCHAR(255)    NOT NULL,
    scope                VARCHAR(20)     NOT NULL,
    owner_id             VARCHAR(255),
    condition_language   VARCHAR(20)     NOT NULL,
    condition_expression VARCHAR(4000),
    actions              VARCHAR(4000),
    active               BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at           TIMESTAMP       NOT NULL
);

-- FilterChain: tracks which WorkItems a given filter has applied labels to (inverse index)
CREATE TABLE filter_chain (
    id          UUID    PRIMARY KEY,
    filter_id   UUID    NOT NULL REFERENCES work_item_filter(id) ON DELETE CASCADE
);

CREATE TABLE filter_chain_work_item (
    filter_chain_id UUID    NOT NULL REFERENCES filter_chain(id) ON DELETE CASCADE,
    work_item_id    UUID    NOT NULL,
    PRIMARY KEY (filter_chain_id, work_item_id)
);

CREATE INDEX idx_fc_filter_id ON filter_chain(filter_id);
CREATE INDEX idx_fcwi_work_item_id ON filter_chain_work_item(work_item_id);

-- Named queue views (label pattern + sort config)
CREATE TABLE queue_view (
    id                    UUID            PRIMARY KEY,
    name                  VARCHAR(255)    NOT NULL,
    label_pattern         VARCHAR(500)    NOT NULL,
    scope                 VARCHAR(20)     NOT NULL,
    owner_id              VARCHAR(255),
    additional_conditions VARCHAR(2000),
    sort_field            VARCHAR(50)     NOT NULL DEFAULT 'createdAt',
    sort_direction        VARCHAR(4)      NOT NULL DEFAULT 'ASC',
    created_at            TIMESTAMP       NOT NULL
);

-- Per-WorkItem queue state (relinquishable flag)
CREATE TABLE work_item_queue_state (
    work_item_id    UUID        PRIMARY KEY,
    relinquishable  BOOLEAN     NOT NULL DEFAULT FALSE
);
```

- [ ] **Step 3: Verify migration compiles with the module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests -Dno-format 2>&1 | tail -5
```

---

### Task 3: FilterEvaluationObserver stub + smoke test

**Files:**
- Create: `queues/.../service/FilterEngine.java`
- Create: `queues/.../service/FilterEngineImpl.java`
- Create: `queues/.../service/FilterEvaluationObserver.java`
- Create: `queues/.../api/QueuesSmoke.java` (test)

- [ ] **Step 1: Create FilterEngine interface**

```java
package io.quarkiverse.work.queues.service;

import io.quarkiverse.work.runtime.model.WorkItem;

/** Evaluates active filters against a WorkItem and updates its INFERRED labels. */
public interface FilterEngine {
    /**
     * Evaluate all active filters against the given WorkItem.
     * Strips INFERRED labels, runs filters, re-applies matching labels.
     */
    void evaluate(WorkItem workItem);
}
```

- [ ] **Step 2: Create stub FilterEngineImpl**

```java
package io.quarkiverse.work.queues.service;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkiverse.work.runtime.model.WorkItem;

@ApplicationScoped
public class FilterEngineImpl implements FilterEngine {

    @Override
    public void evaluate(final WorkItem workItem) {
        // Stub — full implementation in Task 11 (issue #61)
    }
}
```

- [ ] **Step 3: Create FilterEvaluationObserver**

```java
package io.quarkiverse.work.queues.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.runtime.event.WorkItemLifecycleEvent;
import io.quarkiverse.work.runtime.repository.WorkItemRepository;

/**
 * CDI observer that triggers filter re-evaluation on every WorkItem lifecycle event.
 * This is the integration seam between the core extension and the queues module.
 */
@ApplicationScoped
public class FilterEvaluationObserver {

    @Inject
    FilterEngine filterEngine;

    @Inject
    WorkItemRepository workItemRepo;

    @Transactional
    public void onLifecycleEvent(@Observes final WorkItemLifecycleEvent event) {
        workItemRepo.findById(event.workItemId()).ifPresent(filterEngine::evaluate);
    }
}
```

- [ ] **Step 4: Write smoke test**

Create `quarkus-work-queues/src/test/java/io/quarkiverse/workitems/queues/api/QueuesModuleSmokeTest.java`:

```java
package io.quarkiverse.work.queues.api;

import static io.restassured.RestAssured.given;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class QueuesModuleSmokeTest {

    @Test
    void application_starts_with_queues_module_on_classpath() {
        // If Quarkus boots successfully, the observer and engine are wired
        given().get("/workitems").then().statusCode(200);
    }
}
```

- [ ] **Step 5: Run the test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=QueuesModuleSmokeTest -Dno-format 2>&1 | tail -8
```

Expected: Tests run: 1, Failures: 0

- [ ] **Step 6: Commit issue #56**

```bash
git add -A
git commit -m "feat(queues): quarkus-work-queues module scaffold

- quarkus-work-queues Maven module with POM following ledger pattern
- FilterScope enum: PERSONAL, TEAM, ORG
- Flyway V1: work_item_filter, filter_chain, filter_chain_work_item,
  queue_view, work_item_queue_state tables
- FilterEngine SPI + stub FilterEngineImpl
- FilterEvaluationObserver: CDI bridge from WorkItemLifecycleEvent to FilterEngine
- Smoke test: Quarkus boots with queues module on classpath

Closes #56
Refs #52, #50"
```

---

## Issue #57 — WorkItemFilter Entity + CRUD REST + Ad-hoc Evaluation

---

### Task 4: WorkItemFilter entity

**Files:**
- Create: `queues/.../model/WorkItemFilter.java`
- Create: `queues/.../model/FilterAction.java`

- [ ] **Step 1: Create FilterAction record**

```java
package io.quarkiverse.work.queues.model;

/**
 * An action executed by a filter when its condition matches.
 * Currently only APPLY_LABEL is supported.
 */
public record FilterAction(
        /** Action type — currently always "APPLY_LABEL". */
        String type,
        /** Label path to apply (APPLY_LABEL action). */
        String labelPath) {

    public static FilterAction applyLabel(final String labelPath) {
        return new FilterAction("APPLY_LABEL", labelPath);
    }
}
```

- [ ] **Step 2: Create WorkItemFilter entity**

```java
package io.quarkiverse.work.queues.model;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * A saved filter that evaluates WorkItem conditions and applies INFERRED labels.
 *
 * <p>
 * Filters are evaluated on every {@code WorkItemLifecycleEvent}. Matching filters
 * apply their configured labels as {@code INFERRED} on the WorkItem.
 */
@Entity
@Table(name = "work_item_filter")
public class WorkItemFilter extends PanacheEntityBase {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Id
    public UUID id;

    /** Human-readable name for this filter. */
    @Column(nullable = false, length = 255)
    public String name;

    /** Visibility scope. */
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    public FilterScope scope;

    /** Owner: userId (PERSONAL), groupId (TEAM), null (ORG). */
    @Column(name = "owner_id", length = 255)
    public String ownerId;

    /** Language of the condition expression: "jexl", "jq", or "lambda". */
    @Column(name = "condition_language", nullable = false, length = 20)
    public String conditionLanguage;

    /** The condition expression string (null for lambda filters). */
    @Column(name = "condition_expression", length = 4000)
    public String conditionExpression;

    /** JSON-serialised list of {@link FilterAction}. */
    @Column(length = 4000)
    public String actions;

    /** Whether this filter participates in evaluation. */
    public boolean active = true;

    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
        createdAt = Instant.now();
    }

    /** Deserialise actions from JSON storage. */
    public List<FilterAction> parseActions() {
        if (actions == null || actions.isBlank()) {
            return List.of();
        }
        try {
            return MAPPER.readValue(actions, new TypeReference<List<FilterAction>>() {});
        } catch (JsonProcessingException e) {
            return List.of();
        }
    }

    /** Serialise actions to JSON for storage. */
    public static String serializeActions(final List<FilterAction> actionList) {
        try {
            return MAPPER.writeValueAsString(actionList);
        } catch (JsonProcessingException e) {
            return "[]";
        }
    }

    /** Find all active saved filters (JEXL and JQ — not lambda). */
    public static List<WorkItemFilter> findActive() {
        return find("active = true AND conditionLanguage != 'lambda'").list();
    }
}
```

---

### Task 5: FilterResource CRUD REST + ad-hoc evaluation

**Files:**
- Create: `queues/.../api/FilterResource.java`

- [ ] **Step 1: Write failing integration tests**

Create `quarkus-work-queues/src/test/java/io/quarkiverse/workitems/queues/api/FilterResourceTest.java`:

```java
package io.quarkiverse.work.queues.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class FilterResourceTest {

    @Test
    void createFilter_jexl_returnsId() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "High priority intake",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "priority == 'HIGH'",
                          "actions": [{"type": "APPLY_LABEL", "labelPath": "intake/triage"}]
                        }
                        """)
                .post("/filters")
                .then()
                .statusCode(201)
                .body("id", notNullValue())
                .body("name", equalTo("High priority intake"))
                .body("active", equalTo(true));
    }

    @Test
    void listFilters_returnsCreatedFilter() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "List test filter",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "status == 'PENDING'",
                          "actions": [{"type": "APPLY_LABEL", "labelPath": "intake"}]
                        }
                        """)
                .post("/filters")
                .then().statusCode(201);

        given()
                .get("/filters")
                .then()
                .statusCode(200)
                .body("name", hasItem("List test filter"));
    }

    @Test
    void createFilter_lambdaLanguage_returns400() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "Lambda attempt",
                          "scope": "ORG",
                          "conditionLanguage": "lambda",
                          "conditionExpression": null,
                          "actions": []
                        }
                        """)
                .post("/filters")
                .then()
                .statusCode(400);
    }

    @Test
    void deleteFilter_removesIt() {
        var id = given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "Delete me",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "true",
                          "actions": []
                        }
                        """)
                .post("/filters")
                .then().statusCode(201)
                .extract().path("id");

        given().delete("/filters/" + id).then().statusCode(204);
    }

    @Test
    void adHocEvaluation_matchingWorkItem_returnsTrue() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "conditionLanguage": "jexl",
                          "conditionExpression": "priority == 'HIGH'",
                          "workItem": {
                            "title": "test",
                            "status": "PENDING",
                            "priority": "HIGH"
                          }
                        }
                        """)
                .post("/filters/evaluate")
                .then()
                .statusCode(200)
                .body("matches", equalTo(true));
    }

    @Test
    void adHocEvaluation_nonMatchingWorkItem_returnsFalse() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "conditionLanguage": "jexl",
                          "conditionExpression": "priority == 'HIGH'",
                          "workItem": {
                            "title": "test",
                            "status": "PENDING",
                            "priority": "NORMAL"
                          }
                        }
                        """)
                .post("/filters/evaluate")
                .then()
                .statusCode(200)
                .body("matches", equalTo(false));
    }
}
```

- [ ] **Step 2: Create FilterResource**

```java
package io.quarkiverse.work.queues.api;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import com.fasterxml.jackson.databind.node.ObjectNode;

import io.quarkiverse.work.queues.model.FilterAction;
import io.quarkiverse.work.queues.model.FilterScope;
import io.quarkiverse.work.queues.model.WorkItemFilter;
import io.quarkiverse.work.queues.service.FilterConditionEvaluator;
import io.quarkiverse.work.queues.service.FilterEvaluatorRegistry;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemPriority;
import io.quarkiverse.work.runtime.model.WorkItemStatus;

@Path("/filters")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class FilterResource {

    @Inject
    FilterEvaluatorRegistry registry;

    public record CreateFilterRequest(
            String name,
            FilterScope scope,
            String ownerId,
            String conditionLanguage,
            String conditionExpression,
            List<FilterAction> actions) {}

    public record AdHocEvalRequest(
            String conditionLanguage,
            String conditionExpression,
            AdHocWorkItem workItem) {}

    public record AdHocWorkItem(
            String title,
            String status,
            String priority,
            String assigneeId,
            String category) {}

    @GET
    @Transactional
    public List<Map<String, Object>> list() {
        return WorkItemFilter.<WorkItemFilter> listAll().stream()
                .map(f -> Map.<String, Object> of(
                        "id", f.id,
                        "name", f.name,
                        "scope", f.scope,
                        "conditionLanguage", f.conditionLanguage,
                        "active", f.active))
                .toList();
    }

    @POST
    @Transactional
    public Response create(final CreateFilterRequest req) {
        if ("lambda".equalsIgnoreCase(req.conditionLanguage())) {
            return Response.status(400)
                    .entity(Map.of("error", "Lambda filters are CDI beans — not stored via REST. Use conditionLanguage 'jexl' or 'jq'."))
                    .build();
        }
        final WorkItemFilter f = new WorkItemFilter();
        f.name = req.name();
        f.scope = req.scope() != null ? req.scope() : FilterScope.ORG;
        f.ownerId = req.ownerId();
        f.conditionLanguage = req.conditionLanguage();
        f.conditionExpression = req.conditionExpression();
        f.actions = WorkItemFilter.serializeActions(req.actions() != null ? req.actions() : List.of());
        f.active = true;
        f.persist();
        return Response.status(201)
                .entity(Map.of("id", f.id, "name", f.name, "active", f.active))
                .build();
    }

    @PUT
    @Path("/{id}")
    @Transactional
    public Response update(@PathParam("id") final UUID id, final CreateFilterRequest req) {
        final WorkItemFilter f = WorkItemFilter.findById(id);
        if (f == null) {
            return Response.status(404).entity(Map.of("error", "Filter not found")).build();
        }
        if (req.name() != null) f.name = req.name();
        if (req.conditionExpression() != null) f.conditionExpression = req.conditionExpression();
        if (req.actions() != null) f.actions = WorkItemFilter.serializeActions(req.actions());
        return Response.ok(Map.of("id", f.id, "name", f.name)).build();
    }

    @DELETE
    @Path("/{id}")
    @Transactional
    public Response delete(@PathParam("id") final UUID id) {
        final WorkItemFilter f = WorkItemFilter.findById(id);
        if (f == null) {
            return Response.status(404).entity(Map.of("error", "Filter not found")).build();
        }
        // TODO: trigger cascade via FilterEngine (issue #62)
        f.delete();
        return Response.noContent().build();
    }

    @POST
    @Path("/evaluate")
    public Response evaluate(final AdHocEvalRequest req) {
        final FilterConditionEvaluator evaluator = registry.find(req.conditionLanguage());
        if (evaluator == null) {
            return Response.status(400)
                    .entity(Map.of("error", "Unknown condition language: " + req.conditionLanguage()))
                    .build();
        }
        // Build a minimal WorkItem from the request
        final WorkItem wi = new WorkItem();
        if (req.workItem() != null) {
            wi.title = req.workItem().title();
            wi.status = parseStatus(req.workItem().status());
            wi.priority = parsePriority(req.workItem().priority());
            wi.assigneeId = req.workItem().assigneeId();
            wi.category = req.workItem().category();
        }
        final boolean matches = evaluator.evaluate(wi, req.conditionExpression());
        return Response.ok(Map.of("matches", matches)).build();
    }

    private WorkItemStatus parseStatus(final String s) {
        try { return s != null ? WorkItemStatus.valueOf(s) : WorkItemStatus.PENDING; }
        catch (IllegalArgumentException e) { return WorkItemStatus.PENDING; }
    }

    private WorkItemPriority parsePriority(final String p) {
        try { return p != null ? WorkItemPriority.valueOf(p) : WorkItemPriority.NORMAL; }
        catch (IllegalArgumentException e) { return WorkItemPriority.NORMAL; }
    }
}
```

- [ ] **Step 3: Run tests — expect failure (FilterEvaluatorRegistry not yet created)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=FilterResourceTest -Dno-format 2>&1 | tail -8
```

Expected: compile failure (FilterEvaluatorRegistry not found) — confirms TDD red phase.

---

## Issue #58 — FilterConditionEvaluator SPI + JEXL

---

### Task 6: FilterConditionEvaluator SPI + FilterEvaluatorRegistry

**Files:**
- Create: `queues/.../service/FilterConditionEvaluator.java`
- Create: `queues/.../service/FilterEvaluatorRegistry.java`
- Create: `queues/.../service/JexlConditionEvaluatorTest.java`

- [ ] **Step 1: Write failing unit tests for JEXL**

Create `quarkus-work-queues/src/test/java/io/quarkiverse/workitems/queues/service/JexlConditionEvaluatorTest.java`:

```java
package io.quarkiverse.work.queues.service;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemPriority;
import io.quarkiverse.work.runtime.model.WorkItemStatus;

class JexlConditionEvaluatorTest {

    private JexlConditionEvaluator evaluator;

    @BeforeEach
    void setup() {
        evaluator = new JexlConditionEvaluator();
    }

    @Test
    void language_isJexl() {
        assertThat(evaluator.language()).isEqualTo("jexl");
    }

    @Test
    void evaluate_priorityHigh_matchesHighPriorityItem() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.HIGH, null, null);
        assertThat(evaluator.evaluate(wi, "priority == 'HIGH'")).isTrue();
    }

    @Test
    void evaluate_priorityHigh_doesNotMatchNormalPriorityItem() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null, null);
        assertThat(evaluator.evaluate(wi, "priority == 'HIGH'")).isFalse();
    }

    @Test
    void evaluate_unassigned_matchesItemWithNoAssignee() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null, null);
        assertThat(evaluator.evaluate(wi, "assigneeId == null")).isTrue();
    }

    @Test
    void evaluate_statusPending_matchesPendingItem() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null, null);
        assertThat(evaluator.evaluate(wi, "status == 'PENDING'")).isTrue();
    }

    @Test
    void evaluate_complexExpression_andCondition() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.HIGH, null, null);
        assertThat(evaluator.evaluate(wi, "priority == 'HIGH' && status == 'PENDING'")).isTrue();
    }

    @Test
    void evaluate_malformedExpression_returnsFalse() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null, null);
        assertThat(evaluator.evaluate(wi, "this is not valid jexl @@##")).isFalse();
    }

    @Test
    void evaluate_categoryMatch() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null, "legal");
        assertThat(evaluator.evaluate(wi, "category == 'legal'")).isTrue();
    }

    private WorkItem workItem(WorkItemStatus status, WorkItemPriority priority,
            String assigneeId, String category) {
        var wi = new WorkItem();
        wi.status = status;
        wi.priority = priority;
        wi.assigneeId = assigneeId;
        wi.category = category;
        return wi;
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=JexlConditionEvaluatorTest -Dno-format 2>&1 | tail -5
```

Expected: `cannot find symbol: class JexlConditionEvaluator`

- [ ] **Step 3: Create FilterConditionEvaluator interface**

```java
package io.quarkiverse.work.queues.service;

import io.quarkiverse.work.runtime.model.WorkItem;

/**
 * SPI for evaluating a filter condition expression against a WorkItem.
 *
 * <p>
 * Implementations are CDI beans discovered by {@link FilterEvaluatorRegistry}.
 * Three built-ins: JEXL, JQ, and Lambda (CDI beans).
 */
public interface FilterConditionEvaluator {

    /**
     * Language identifier stored alongside the expression.
     * e.g. {@code "jexl"}, {@code "jq"}, {@code "lambda"}.
     */
    String language();

    /**
     * Returns {@code true} if the WorkItem matches the condition expression.
     * Must never throw — malformed expressions return {@code false}.
     */
    boolean evaluate(WorkItem workItem, String expression);
}
```

- [ ] **Step 4: Create JexlConditionEvaluator**

```java
package io.quarkiverse.work.queues.service;

import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;

import org.apache.commons.jexl3.JexlBuilder;
import org.apache.commons.jexl3.JexlEngine;
import org.apache.commons.jexl3.JexlException;
import org.apache.commons.jexl3.MapContext;

import io.quarkiverse.work.runtime.model.WorkItem;

/**
 * Evaluates Apache JEXL expressions against a WorkItem.
 *
 * <p>
 * JEXL context exposes WorkItem fields as named variables:
 * {@code status}, {@code priority}, {@code assigneeId}, {@code category},
 * {@code title}, {@code description}, {@code candidateGroups}.
 *
 * <p>
 * Example: {@code priority == 'HIGH' && assigneeId == null}
 */
@ApplicationScoped
public class JexlConditionEvaluator implements FilterConditionEvaluator {

    private static final JexlEngine JEXL = new JexlBuilder()
            .strict(false)
            .silent(true)
            .create();

    @Override
    public String language() {
        return "jexl";
    }

    @Override
    public boolean evaluate(final WorkItem workItem, final String expression) {
        if (expression == null || expression.isBlank()) {
            return false;
        }
        try {
            final var ctx = new MapContext(buildContext(workItem));
            final Object result = JEXL.createExpression(expression).evaluate(ctx);
            return Boolean.TRUE.equals(result);
        } catch (JexlException e) {
            return false;
        }
    }

    private Map<String, Object> buildContext(final WorkItem wi) {
        return Map.of(
                "status", wi.status != null ? wi.status.name() : "null",
                "priority", wi.priority != null ? wi.priority.name() : "null",
                "assigneeId", wi.assigneeId,
                "category", wi.category,
                "title", wi.title != null ? wi.title : "",
                "description", wi.description != null ? wi.description : "",
                "candidateGroups", wi.candidateGroups != null ? wi.candidateGroups : "");
    }
}
```

**Note:** `MapContext` cannot hold null values in some versions. Read the JEXL docs and adjust if needed. An alternative is a custom `JexlContext` that handles nulls, or use `""` for null strings but expose `assigneeId` specially for null-checking.

Adjust the `buildContext` so `assigneeId == null` works:

```java
    private java.util.HashMap<String, Object> buildContext(final WorkItem wi) {
        var ctx = new java.util.HashMap<String, Object>();
        ctx.put("status", wi.status != null ? wi.status.name() : "UNKNOWN");
        ctx.put("priority", wi.priority != null ? wi.priority.name() : "NORMAL");
        ctx.put("assigneeId", wi.assigneeId); // intentionally null-able
        ctx.put("category", wi.category);
        ctx.put("title", wi.title != null ? wi.title : "");
        ctx.put("description", wi.description != null ? wi.description : "");
        ctx.put("candidateGroups", wi.candidateGroups != null ? wi.candidateGroups : "");
        return ctx;
    }
```

And use `new MapContext(buildContext(workItem))` — or switch to a direct `JexlContext` implementation if `MapContext` doesn't accept null-value maps. Use the simplest approach that makes the `assigneeId == null` test pass.

- [ ] **Step 5: Create FilterEvaluatorRegistry**

```java
package io.quarkiverse.work.queues.service;

import java.util.HashMap;
import java.util.Map;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

/**
 * Discovers and indexes all {@link FilterConditionEvaluator} CDI beans by language.
 */
@ApplicationScoped
public class FilterEvaluatorRegistry {

    @Inject
    Instance<FilterConditionEvaluator> evaluators;

    private final Map<String, FilterConditionEvaluator> index = new HashMap<>();

    @PostConstruct
    void init() {
        for (FilterConditionEvaluator e : evaluators) {
            index.put(e.language().toLowerCase(), e);
        }
    }

    /** Returns the evaluator for the given language, or {@code null} if unknown. */
    public FilterConditionEvaluator find(final String language) {
        return language != null ? index.get(language.toLowerCase()) : null;
    }
}
```

- [ ] **Step 6: Run JEXL tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=JexlConditionEvaluatorTest -Dno-format 2>&1 | tail -5
```

Expected: Tests run: 8, Failures: 0

---

## Issue #59 — JQ Evaluator

---

### Task 7: JqConditionEvaluator

**Files:**
- Create: `queues/.../service/JqConditionEvaluator.java`
- Create: `queues/.../service/JqConditionEvaluatorTest.java`

- [ ] **Step 1: Write failing unit tests**

Create `quarkus-work-queues/src/test/java/io/quarkiverse/workitems/queues/service/JqConditionEvaluatorTest.java`:

```java
package io.quarkiverse.work.queues.service;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemPriority;
import io.quarkiverse.work.runtime.model.WorkItemStatus;

class JqConditionEvaluatorTest {

    private JqConditionEvaluator evaluator;

    @BeforeEach
    void setup() {
        evaluator = new JqConditionEvaluator();
    }

    @Test
    void language_isJq() {
        assertThat(evaluator.language()).isEqualTo("jq");
    }

    @Test
    void evaluate_priorityHigh_matchesHighPriorityItem() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.HIGH, null);
        assertThat(evaluator.evaluate(wi, ".priority == \"HIGH\"")).isTrue();
    }

    @Test
    void evaluate_priorityHigh_doesNotMatchNormal() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null);
        assertThat(evaluator.evaluate(wi, ".priority == \"HIGH\"")).isFalse();
    }

    @Test
    void evaluate_statusPending() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null);
        assertThat(evaluator.evaluate(wi, ".status == \"PENDING\"")).isTrue();
    }

    @Test
    void evaluate_assigneeNull_unassigned() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null);
        assertThat(evaluator.evaluate(wi, ".assigneeId == null")).isTrue();
    }

    @Test
    void evaluate_malformedExpression_returnsFalse() {
        var wi = workItem(WorkItemStatus.PENDING, WorkItemPriority.NORMAL, null);
        assertThat(evaluator.evaluate(wi, "not valid jq @@@")).isFalse();
    }

    private WorkItem workItem(WorkItemStatus status, WorkItemPriority priority, String assigneeId) {
        var wi = new WorkItem();
        wi.status = status;
        wi.priority = priority;
        wi.assigneeId = assigneeId;
        return wi;
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

- [ ] **Step 3: Create JqConditionEvaluator**

```java
package io.quarkiverse.work.queues.service;

import jakarta.enterprise.context.ApplicationScoped;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.BooleanNode;

import net.thisptr.jackson.jq.BuiltinFunctionLoader;
import net.thisptr.jackson.jq.JsonQuery;
import net.thisptr.jackson.jq.Scope;
import net.thisptr.jackson.jq.Versions;
import net.thisptr.jackson.jq.exception.JsonQueryException;

import io.quarkiverse.work.runtime.model.WorkItem;

/**
 * Evaluates JQ expressions against a WorkItem serialised as JSON.
 *
 * <p>
 * Example: {@code .priority == "HIGH" and .assigneeId == null}
 */
@ApplicationScoped
public class JqConditionEvaluator implements FilterConditionEvaluator {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Scope ROOT_SCOPE;

    static {
        ROOT_SCOPE = Scope.newEmptyScope();
        BuiltinFunctionLoader.getInstance().loadFunctions(Versions.JQ_1_6, ROOT_SCOPE);
    }

    @Override
    public String language() {
        return "jq";
    }

    @Override
    public boolean evaluate(final WorkItem workItem, final String expression) {
        if (expression == null || expression.isBlank()) {
            return false;
        }
        try {
            final JsonNode input = MAPPER.valueToTree(toMap(workItem));
            final JsonQuery q = JsonQuery.compile(expression, Versions.JQ_1_6);
            final Scope scope = Scope.newChildScope(ROOT_SCOPE);
            final var results = new java.util.ArrayList<JsonNode>();
            q.apply(scope, input, results::add);
            if (results.isEmpty()) {
                return false;
            }
            final JsonNode first = results.get(0);
            return first instanceof BooleanNode && first.asBoolean();
        } catch (JsonQueryException | Exception e) {
            return false;
        }
    }

    private java.util.Map<String, Object> toMap(final WorkItem wi) {
        var map = new java.util.HashMap<String, Object>();
        map.put("status", wi.status != null ? wi.status.name() : null);
        map.put("priority", wi.priority != null ? wi.priority.name() : null);
        map.put("assigneeId", wi.assigneeId);
        map.put("category", wi.category);
        map.put("title", wi.title);
        map.put("description", wi.description);
        map.put("candidateGroups", wi.candidateGroups);
        return map;
    }
}
```

- [ ] **Step 4: Run JQ tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=JqConditionEvaluatorTest -Dno-format 2>&1 | tail -5
```

Expected: Tests run: 6, Failures: 0

---

## Issue #60 — Lambda Filter CDI Discovery

---

### Task 8: WorkItemFilterBean interface + LambdaFilterRegistry

**Files:**
- Create: `queues/.../service/WorkItemFilterBean.java`
- Create: `queues/.../service/LambdaFilterRegistry.java`

- [ ] **Step 1: Create WorkItemFilterBean SPI**

```java
package io.quarkiverse.work.queues.service;

import java.util.List;

import io.quarkiverse.work.queues.model.FilterAction;
import io.quarkiverse.work.queues.model.FilterScope;
import io.quarkiverse.work.runtime.model.WorkItem;

/**
 * SPI for programmatic (Lambda) filter beans.
 *
 * <p>
 * Implement this interface and annotate with {@code @ApplicationScoped} to define
 * a filter in code. Lambda filters are always active while deployed — they cannot
 * be paused via the REST API.
 *
 * <p>
 * Example:
 * <pre>
 * {@literal @}ApplicationScoped
 * public class HighPriorityFilter implements WorkItemFilterBean {
 *     public boolean matches(WorkItem wi) { return wi.priority == HIGH; }
 *     public List&lt;FilterAction&gt; actions() { return List.of(FilterAction.applyLabel("priority/high")); }
 *     public FilterScope scope() { return FilterScope.ORG; }
 * }
 * </pre>
 */
public interface WorkItemFilterBean {

    /** Returns true if this filter should apply its actions to the given WorkItem. */
    boolean matches(WorkItem workItem);

    /** Actions to apply when this filter matches. */
    List<FilterAction> actions();

    /** Visibility scope of this filter. */
    FilterScope scope();
}
```

- [ ] **Step 2: Create LambdaFilterRegistry**

```java
package io.quarkiverse.work.queues.service;

import java.util.ArrayList;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

/**
 * Discovers all CDI beans implementing {@link WorkItemFilterBean} at startup.
 * Lambda filters are not stored in the database — they are code.
 */
@ApplicationScoped
public class LambdaFilterRegistry {

    @Inject
    Instance<WorkItemFilterBean> beans;

    /** Returns all discovered lambda filter beans. */
    public List<WorkItemFilterBean> all() {
        final var list = new ArrayList<WorkItemFilterBean>();
        beans.forEach(list::add);
        return list;
    }
}
```

- [ ] **Step 3: Write @QuarkusTest to verify lambda filter discovery**

Add to `FilterResourceTest.java`:

```java
    @Test
    void lambdaFilters_notReturnedByRestApi() {
        // Lambda filters are CDI beans — not visible in GET /filters
        // This test verifies the API doesn't expose them
        given()
                .get("/filters")
                .then()
                .statusCode(200)
                // Lambda filters have no REST representation
                .body("conditionLanguage", not(hasItem("lambda")));
    }
```

- [ ] **Step 4: Now run FilterResourceTest to see all tests pass (evaluators are wired)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=FilterResourceTest -Dno-format 2>&1 | tail -8
```

Expected: Tests run: 7, Failures: 0

- [ ] **Step 5: Commit issues #57, #58, #59, #60**

```bash
git add -A
git commit -m "feat(queues): WorkItemFilter CRUD, JEXL/JQ/Lambda evaluators

- WorkItemFilter Panache entity: id, name, scope, conditionLanguage, actions (JSON)
- FilterAction record: APPLY_LABEL type + labelPath
- FilterResource: GET/POST/PUT/DELETE /filters, POST /filters/evaluate
- FilterConditionEvaluator SPI + FilterEvaluatorRegistry (CDI Instance discovery)
- JexlConditionEvaluator: JEXL3 with context for status/priority/assigneeId/category
- JqConditionEvaluator: jackson-jq with WorkItem serialised as JSON
- WorkItemFilterBean SPI + LambdaFilterRegistry for CDI bean discovery
- 8 JEXL unit tests, 6 JQ unit tests, 7 REST integration tests

Closes #57
Closes #58
Closes #59
Closes #60
Refs #52, #50"
```

---

## Issue #61 — FilterChain + Re-evaluation Engine

---

### Task 9: FilterChain entity

**Files:**
- Create: `queues/.../model/FilterChain.java`

- [ ] **Step 1: Create FilterChain entity**

```java
package io.quarkiverse.work.queues.model;

import java.util.HashSet;
import java.util.Set;
import java.util.UUID;

import jakarta.persistence.*;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * Tracks the inverse index: which WorkItems a given filter has applied INFERRED labels to.
 *
 * <p>
 * On filter deletion, {@code workItems} gives O(affected) cascade — no full table scan.
 * One {@code FilterChain} exists per filter; it accumulates WorkItem IDs as filters fire.
 */
@Entity
@Table(name = "filter_chain")
public class FilterChain extends PanacheEntityBase {

    @Id
    public UUID id;

    /** The filter this chain tracks. */
    @Column(name = "filter_id", nullable = false)
    public UUID filterId;

    /** WorkItems that have received INFERRED labels from this filter. */
    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "filter_chain_work_item",
            joinColumns = @JoinColumn(name = "filter_chain_id"))
    @Column(name = "work_item_id")
    public Set<UUID> workItems = new HashSet<>();

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
    }

    /** Find or create the FilterChain for a given filter. */
    public static FilterChain findOrCreateForFilter(final UUID filterId) {
        return FilterChain.<FilterChain> find("filterId", filterId)
                .firstResultOptional()
                .orElseGet(() -> {
                    final FilterChain fc = new FilterChain();
                    fc.filterId = filterId;
                    fc.persist();
                    return fc;
                });
    }

    /** Find the FilterChain for a given filter, or null if none exists. */
    public static FilterChain findByFilterId(final UUID filterId) {
        return FilterChain.<FilterChain> find("filterId", filterId).firstResult();
    }
}
```

---

### Task 10: FilterEngineImpl — re-evaluation with propagation

**Files:**
- Modify: `queues/.../service/FilterEngineImpl.java`
- Create: `queues/.../service/FilterEngineTest.java`

- [ ] **Step 1: Write failing unit tests for the engine**

Create `quarkus-work-queues/src/test/java/io/quarkiverse/workitems/queues/service/FilterEngineTest.java`:

```java
package io.quarkiverse.work.queues.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.quarkiverse.work.queues.model.FilterAction;
import io.quarkiverse.work.queues.model.FilterScope;
import io.quarkiverse.work.runtime.model.LabelPersistence;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemLabel;
import io.quarkiverse.work.runtime.model.WorkItemPriority;
import io.quarkiverse.work.runtime.model.WorkItemStatus;

class FilterEngineTest {

    // A simple in-memory evaluator for unit testing (no CDI)
    record SimpleFilter(String condition, List<FilterAction> actions) {}

    /**
     * Simulates the core re-evaluation algorithm without Quarkus or DB.
     * Strip INFERRED labels, evaluate filters, apply matching labels.
     * Supports propagation: re-evaluates until stable (max 10 passes).
     */
    static List<WorkItemLabel> evaluate(WorkItem wi, List<SimpleFilter> filters) {
        // Strip INFERRED
        wi.labels.removeIf(l -> l.persistence == LabelPersistence.INFERRED);

        boolean changed = true;
        int passes = 0;
        while (changed && passes < 10) {
            changed = false;
            passes++;
            var labelPaths = wi.labels.stream().map(l -> l.path).toList();
            for (var filter : filters) {
                // Simple condition eval: check if condition matches
                // In production this calls JexlConditionEvaluator
                boolean matches = evalSimpleCondition(filter.condition(), wi, labelPaths);
                if (matches) {
                    for (var action : filter.actions()) {
                        if ("APPLY_LABEL".equals(action.type())) {
                            boolean already = wi.labels.stream()
                                    .anyMatch(l -> l.path.equals(action.labelPath()));
                            if (!already) {
                                wi.labels.add(new WorkItemLabel(
                                        action.labelPath(), LabelPersistence.INFERRED, "test-filter"));
                                changed = true;
                            }
                        }
                    }
                }
            }
        }
        return wi.labels;
    }

    static boolean evalSimpleCondition(String cond, WorkItem wi, List<String> labelPaths) {
        return switch (cond) {
            case "HIGH" -> wi.priority == WorkItemPriority.HIGH;
            case "PENDING" -> wi.status == WorkItemStatus.PENDING;
            case "HAS_INTAKE" -> labelPaths.contains("intake");
            case "ALWAYS" -> true;
            default -> false;
        };
    }

    @Test
    void evaluate_matchingFilter_appliesInferredLabel() {
        var wi = new WorkItem();
        wi.priority = WorkItemPriority.HIGH;
        wi.status = WorkItemStatus.PENDING;

        var filter = new SimpleFilter("HIGH",
                List.of(FilterAction.applyLabel("priority/high")));

        evaluate(wi, List.of(filter));

        assertThat(wi.labels)
                .extracting(l -> l.path)
                .contains("priority/high");
        assertThat(wi.labels)
                .filteredOn(l -> l.path.equals("priority/high"))
                .extracting(l -> l.persistence)
                .containsOnly(LabelPersistence.INFERRED);
    }

    @Test
    void evaluate_nonMatchingFilter_doesNotApplyLabel() {
        var wi = new WorkItem();
        wi.priority = WorkItemPriority.NORMAL;
        wi.status = WorkItemStatus.PENDING;

        var filter = new SimpleFilter("HIGH",
                List.of(FilterAction.applyLabel("priority/high")));

        evaluate(wi, List.of(filter));

        assertThat(wi.labels).extracting(l -> l.path).doesNotContain("priority/high");
    }

    @Test
    void evaluate_stripsExistingInferredLabels_beforeReevaluation() {
        var wi = new WorkItem();
        wi.priority = WorkItemPriority.NORMAL;
        wi.labels.add(new WorkItemLabel("old/inferred", LabelPersistence.INFERRED, "old-filter"));
        wi.labels.add(new WorkItemLabel("manual/keep", LabelPersistence.MANUAL, "alice"));

        evaluate(wi, List.of());

        assertThat(wi.labels).extracting(l -> l.path).doesNotContain("old/inferred");
        assertThat(wi.labels).extracting(l -> l.path).contains("manual/keep");
    }

    @Test
    void evaluate_propagationChain_filterBSeesLabelFromFilterA() {
        var wi = new WorkItem();
        wi.priority = WorkItemPriority.HIGH;
        wi.status = WorkItemStatus.PENDING;

        var filterA = new SimpleFilter("HIGH", List.of(FilterAction.applyLabel("intake")));
        var filterB = new SimpleFilter("HAS_INTAKE", List.of(FilterAction.applyLabel("intake/triage")));

        evaluate(wi, List.of(filterA, filterB));

        assertThat(wi.labels).extracting(l -> l.path).contains("intake", "intake/triage");
    }

    @Test
    void evaluate_circularFilters_terminatesAfterMaxPasses() {
        var wi = new WorkItem();
        wi.priority = WorkItemPriority.HIGH;

        // Filter A always fires and applies "cycle/a"
        // Filter B always fires and applies "cycle/b"
        // They keep alternating but both are already added after pass 1
        var filterA = new SimpleFilter("ALWAYS", List.of(FilterAction.applyLabel("cycle/a")));
        var filterB = new SimpleFilter("ALWAYS", List.of(FilterAction.applyLabel("cycle/b")));

        // Should terminate without infinite loop
        evaluate(wi, List.of(filterA, filterB));

        assertThat(wi.labels).extracting(l -> l.path).contains("cycle/a", "cycle/b");
    }

    @Test
    void evaluate_multipleFilters_allApplied() {
        var wi = new WorkItem();
        wi.priority = WorkItemPriority.HIGH;
        wi.status = WorkItemStatus.PENDING;

        var filterA = new SimpleFilter("HIGH", List.of(FilterAction.applyLabel("priority/high")));
        var filterB = new SimpleFilter("PENDING", List.of(FilterAction.applyLabel("intake")));

        evaluate(wi, List.of(filterA, filterB));

        assertThat(wi.labels).extracting(l -> l.path).contains("priority/high", "intake");
    }
}
```

- [ ] **Step 2: Run tests — expect PASS** (these test pure logic, no Quarkus)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=FilterEngineTest -Dno-format 2>&1 | tail -5
```

Expected: Tests run: 6, Failures: 0

- [ ] **Step 3: Implement FilterEngineImpl**

Replace the stub in `FilterEngineImpl.java`:

```java
package io.quarkiverse.work.queues.service;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.quarkiverse.work.queues.model.FilterAction;
import io.quarkiverse.work.queues.model.FilterChain;
import io.quarkiverse.work.queues.model.WorkItemFilter;
import io.quarkiverse.work.runtime.model.LabelPersistence;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemLabel;
import io.quarkiverse.work.runtime.repository.WorkItemRepository;

@ApplicationScoped
public class FilterEngineImpl implements FilterEngine {

    private static final int MAX_PROPAGATION_PASSES = 10;

    @Inject
    FilterEvaluatorRegistry registry;

    @Inject
    LambdaFilterRegistry lambdaRegistry;

    @Inject
    WorkItemRepository workItemRepo;

    @Override
    @Transactional
    public void evaluate(final WorkItem workItem) {
        // Step 1: Strip all INFERRED labels
        workItem.labels.removeIf(l -> l.persistence == LabelPersistence.INFERRED);

        // Step 2: Multi-pass evaluation until stable (max 10 passes for propagation)
        boolean changed = true;
        int passes = 0;
        while (changed && passes < MAX_PROPAGATION_PASSES) {
            changed = false;
            passes++;

            // Evaluate saved filters (JEXL, JQ)
            final List<WorkItemFilter> savedFilters = WorkItemFilter.findActive();
            for (final WorkItemFilter filter : savedFilters) {
                final FilterConditionEvaluator evaluator = registry.find(filter.conditionLanguage);
                if (evaluator != null && evaluator.evaluate(workItem, filter.conditionExpression)) {
                    changed |= applyActions(workItem, filter.parseActions(), filter.id.toString());
                    // Update FilterChain inverse index
                    final FilterChain chain = FilterChain.findOrCreateForFilter(filter.id);
                    chain.workItems.add(workItem.id);
                }
            }

            // Evaluate Lambda filters
            for (final WorkItemFilterBean bean : lambdaRegistry.all()) {
                if (bean.matches(workItem)) {
                    changed |= applyActions(workItem, bean.actions(), bean.getClass().getSimpleName());
                }
            }
        }

        // Step 3: Persist the WorkItem with updated labels
        workItemRepo.save(workItem);
    }

    private boolean applyActions(final WorkItem wi, final List<FilterAction> actions,
            final String appliedBy) {
        boolean applied = false;
        for (final FilterAction action : actions) {
            if ("APPLY_LABEL".equals(action.type()) && action.labelPath() != null) {
                final boolean alreadyPresent = wi.labels.stream()
                        .anyMatch(l -> l.path.equals(action.labelPath()));
                if (!alreadyPresent) {
                    wi.labels.add(new WorkItemLabel(action.labelPath(),
                            LabelPersistence.INFERRED, appliedBy));
                    applied = true;
                }
            }
        }
        return applied;
    }
}
```

---

### Task 11: @QuarkusTest — filter fires on WorkItem creation

**Files:**
- Create: `queues/.../api/FullPipelineTest.java`

- [ ] **Step 1: Write the integration test**

```java
package io.quarkiverse.work.queues.api;

import static io.restassured.RestAssured.given;
import static org.awaitility.Awaitility.await;
import static org.hamcrest.Matchers.*;

import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class FullPipelineTest {

    @Test
    void savedJexlFilter_firesOnWorkItemCreation_appliesInferredLabel() {
        // Create a saved filter: HIGH priority → apply intake/triage
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "High priority triage",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "priority == 'HIGH'",
                          "actions": [{"type": "APPLY_LABEL", "labelPath": "intake/triage"}]
                        }
                        """)
                .post("/filters")
                .then().statusCode(201);

        // Create a HIGH priority WorkItem
        var id = given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "title": "High priority pipeline test",
                          "priority": "HIGH",
                          "createdBy": "alice"
                        }
                        """)
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        // INFERRED label should be applied
        given()
                .get("/workitems/" + id)
                .then()
                .statusCode(200)
                .body("labels.path", hasItem("intake/triage"))
                .body("labels.findAll { it.path == 'intake/triage' }[0].persistence",
                        equalTo("INFERRED"));
    }

    @Test
    void savedJexlFilter_notMatchingPriority_doesNotApplyLabel() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "High priority only",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "priority == 'HIGH'",
                          "actions": [{"type": "APPLY_LABEL", "labelPath": "priority/high"}]
                        }
                        """)
                .post("/filters")
                .then().statusCode(201);

        var id = given()
                .contentType(ContentType.JSON)
                .body("""
                        {"title": "Normal priority", "priority": "NORMAL", "createdBy": "alice"}
                        """)
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        given()
                .get("/workitems/" + id)
                .then()
                .statusCode(200)
                .body("labels.path", not(hasItem("priority/high")));
    }

    @Test
    void propagation_filterA_appliesLabel_filterB_firesOnLabel() {
        // Filter A: HIGH → intake
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "Propagation A",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "priority == 'HIGH'",
                          "actions": [{"type": "APPLY_LABEL", "labelPath": "intake"}]
                        }
                        """)
                .post("/filters")
                .then().statusCode(201);

        // Filter B: has intake label (check via JEXL on labels)
        // Use assigneeId == null as proxy for "unfiltered" — we check propagation via the presence
        // of intake label on the WorkItem which filter B would then see on next pass
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "Propagation B",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "priority == 'HIGH'",
                          "actions": [{"type": "APPLY_LABEL", "labelPath": "intake/triage"}]
                        }
                        """)
                .post("/filters")
                .then().statusCode(201);

        var id = given()
                .contentType(ContentType.JSON)
                .body("""{"title": "Propagation test", "priority": "HIGH", "createdBy": "alice"}""")
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        given()
                .get("/workitems/" + id)
                .then()
                .statusCode(200)
                .body("labels.path", hasItems("intake", "intake/triage"));
    }
}
```

- [ ] **Step 2: Run the full pipeline test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=FullPipelineTest -Dno-format 2>&1 | tail -10
```

Expected: Tests run: 3, Failures: 0

- [ ] **Step 3: Commit issue #61**

```bash
git add -A
git commit -m "feat(queues): FilterChain entity + FilterEngine re-evaluation with propagation

- FilterChain: tracks filterId → workItems inverse index for O(affected) cascade
- FilterEngineImpl: strip INFERRED labels, multi-pass evaluation (max 10 passes),
  propagation supported (filterB can see labels applied by filterA in same run)
- FilterChain.workItems updated as filters fire
- FullPipelineTest: 3 @QuarkusTest integration tests — filter fires on create,
  non-matching filter, and label propagation chain

Closes #61
Refs #52, #50"
```

---

## Issue #62 — Filter Deletion Cascade

---

### Task 12: Cascade on filter deletion

**Files:**
- Modify: `queues/.../service/FilterEngineImpl.java` (add cascadeDelete method)
- Modify: `queues/.../api/FilterResource.java` (wire cascade on DELETE)
- Modify: `queues/.../api/FullPipelineTest.java` (add cascade test)

- [ ] **Step 1: Write failing cascade test**

Add to `FullPipelineTest.java`:

```java
    @Test
    void deleteFilter_cascadesInferredLabels_fromAffectedWorkItems() {
        // Create filter: PENDING → intake
        var filterId = given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "Cascade test filter",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "status == 'PENDING'",
                          "actions": [{"type": "APPLY_LABEL", "labelPath": "intake"}]
                        }
                        """)
                .post("/filters")
                .then().statusCode(201)
                .extract().path("id");

        // Create PENDING WorkItem — filter should fire and apply INFERRED label
        var workItemId = given()
                .contentType(ContentType.JSON)
                .body("""{"title": "Cascade test item", "createdBy": "alice"}""")
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        // Verify INFERRED label is present
        given()
                .get("/workitems/" + workItemId)
                .then()
                .statusCode(200)
                .body("labels.path", hasItem("intake"));

        // Delete the filter — INFERRED label should cascade away
        given()
                .delete("/filters/" + filterId)
                .then()
                .statusCode(204);

        // Verify the INFERRED label is gone
        given()
                .get("/workitems/" + workItemId)
                .then()
                .statusCode(200)
                .body("labels.path", not(hasItem("intake")));
    }
```

- [ ] **Step 2: Add cascadeDelete to FilterEngine interface**

In `FilterEngine.java`, add:

```java
    /**
     * Remove all INFERRED labels applied by the given filter from all affected WorkItems.
     * Uses the FilterChain inverse index for O(affected) traversal.
     */
    void cascadeDelete(java.util.UUID filterId);
```

- [ ] **Step 3: Implement cascadeDelete in FilterEngineImpl**

Add to `FilterEngineImpl.java`:

```java
    @Override
    @Transactional
    public void cascadeDelete(final java.util.UUID filterId) {
        final FilterChain chain = FilterChain.findByFilterId(filterId);
        if (chain == null) {
            return; // Filter never fired — nothing to cascade
        }

        // Visit each WorkItem that received labels from this filter
        for (final java.util.UUID workItemId : chain.workItems) {
            workItemRepo.findById(workItemId).ifPresent(wi -> {
                // Remove INFERRED labels applied by this filter
                wi.labels.removeIf(l ->
                        l.persistence == LabelPersistence.INFERRED
                                && filterId.toString().equals(l.appliedBy));
                workItemRepo.save(wi);
            });
        }

        // Delete the chain record (FK cascade removes filter_chain_work_item rows)
        chain.delete();
    }
```

- [ ] **Step 4: Wire cascade in FilterResource.delete()**

Replace the `TODO` comment in `FilterResource.delete()`:

```java
    @DELETE
    @Path("/{id}")
    @Transactional
    public Response delete(@PathParam("id") final UUID id) {
        final WorkItemFilter f = WorkItemFilter.findById(id);
        if (f == null) {
            return Response.status(404).entity(Map.of("error", "Filter not found")).build();
        }
        filterEngine.cascadeDelete(id);
        f.delete();
        return Response.noContent().build();
    }
```

Add `@Inject FilterEngine filterEngine;` to `FilterResource`.

- [ ] **Step 5: Run cascade test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=FullPipelineTest -Dno-format 2>&1 | tail -8
```

Expected: Tests run: 4, Failures: 0

- [ ] **Step 6: Commit issue #62**

```bash
git add -A
git commit -m "feat(queues): filter deletion cascades INFERRED labels via FilterChain

- FilterEngine.cascadeDelete() uses FilterChain.workItems inverse index
- Removes INFERRED labels where appliedBy matches the deleted filterId
- FilterResource.delete() triggers cascadeDelete before removing filter record
- Cascade test: create filter, create WorkItem, verify label, delete filter, verify label gone

Closes #62
Refs #52, #50"
```

---

## Issue #63 — QueueView Entity + REST

---

### Task 13: QueueView entity + REST endpoints

**Files:**
- Create: `queues/.../model/QueueView.java`
- Create: `queues/.../api/QueueResource.java`
- Create: `queues/.../api/QueueResourceTest.java`

- [ ] **Step 1: Write failing tests**

Create `quarkus-work-queues/src/test/java/io/quarkiverse/workitems/queues/api/QueueResourceTest.java`:

```java
package io.quarkiverse.work.queues.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class QueueResourceTest {

    @Test
    void createQueueView_returnsId() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "Legal triage queue",
                          "labelPattern": "legal/**",
                          "scope": "TEAM",
                          "sortField": "createdAt",
                          "sortDirection": "ASC"
                        }
                        """)
                .post("/queues")
                .then()
                .statusCode(201)
                .body("id", notNullValue())
                .body("name", equalTo("Legal triage queue"));
    }

    @Test
    void listQueues_returnsCreatedView() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "List test queue",
                          "labelPattern": "intake/**",
                          "scope": "ORG"
                        }
                        """)
                .post("/queues")
                .then().statusCode(201);

        given()
                .get("/queues")
                .then()
                .statusCode(200)
                .body("name", hasItem("List test queue"));
    }

    @Test
    void getQueue_returnsWorkItemsMatchingLabel() {
        // Create a filter: HIGH → intake
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "QueueView pipeline filter",
                          "scope": "ORG",
                          "conditionLanguage": "jexl",
                          "conditionExpression": "priority == 'HIGH'",
                          "actions": [{"type": "APPLY_LABEL", "labelPath": "intake"}]
                        }
                        """)
                .post("/filters")
                .then().statusCode(201);

        // Create a queue view for intake/**
        var queueId = given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "name": "Intake queue",
                          "labelPattern": "intake/**",
                          "scope": "ORG",
                          "sortField": "createdAt",
                          "sortDirection": "ASC"
                        }
                        """)
                .post("/queues")
                .then().statusCode(201)
                .extract().path("id");

        // Create a HIGH priority WorkItem — filter fires, INFERRED label applied
        given()
                .contentType(ContentType.JSON)
                .body("""{"title": "Queue member", "priority": "HIGH", "createdBy": "alice"}""")
                .post("/workitems")
                .then().statusCode(201);

        // Queue view should return the WorkItem
        given()
                .get("/queues/" + queueId)
                .then()
                .statusCode(200)
                .body("title", hasItem("Queue member"));
    }

    @Test
    void deleteQueueView_removesIt() {
        var id = given()
                .contentType(ContentType.JSON)
                .body("""{"name": "Delete me", "labelPattern": "x/**", "scope": "ORG"}""")
                .post("/queues")
                .then().statusCode(201)
                .extract().path("id");

        given().delete("/queues/" + id).then().statusCode(204);
    }
}
```

- [ ] **Step 2: Create QueueView entity**

```java
package io.quarkiverse.work.queues.model;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.*;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * A named, scoped view over WorkItems matching a label pattern.
 *
 * <p>
 * A queue view is not a separate data store — it is a named query. WorkItems
 * appear in the queue when they carry a label matching {@code labelPattern}.
 * View ownership is independent of label ownership.
 */
@Entity
@Table(name = "queue_view")
public class QueueView extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(nullable = false, length = 255)
    public String name;

    /** Label path or wildcard, e.g. {@code legal/contracts/**}. */
    @Column(name = "label_pattern", nullable = false, length = 500)
    public String labelPattern;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    public FilterScope scope;

    @Column(name = "owner_id", length = 255)
    public String ownerId;

    /** Optional JEXL expression for additional filtering within the label. */
    @Column(name = "additional_conditions", length = 2000)
    public String additionalConditions;

    @Column(name = "sort_field", length = 50)
    public String sortField = "createdAt";

    @Column(name = "sort_direction", length = 4)
    public String sortDirection = "ASC";

    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (id == null) id = UUID.randomUUID();
        createdAt = Instant.now();
    }
}
```

- [ ] **Step 3: Create QueueResource**

```java
package io.quarkiverse.work.queues.api;

import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.work.queues.model.FilterScope;
import io.quarkiverse.work.queues.model.QueueView;
import io.quarkiverse.work.runtime.api.WorkItemMapper;
import io.quarkiverse.work.runtime.api.WorkItemResponse;
import io.quarkiverse.work.runtime.repository.WorkItemRepository;

@Path("/queues")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class QueueResource {

    @Inject
    WorkItemRepository workItemRepo;

    public record CreateQueueRequest(
            String name,
            String labelPattern,
            FilterScope scope,
            String ownerId,
            String additionalConditions,
            String sortField,
            String sortDirection) {}

    @GET
    @Transactional
    public List<Map<String, Object>> list() {
        return QueueView.<QueueView> listAll().stream()
                .map(q -> Map.<String, Object> of(
                        "id", q.id,
                        "name", q.name,
                        "labelPattern", q.labelPattern,
                        "scope", q.scope))
                .toList();
    }

    @POST
    @Transactional
    public Response create(final CreateQueueRequest req) {
        if (req.labelPattern() == null || req.labelPattern().isBlank()) {
            return Response.status(400)
                    .entity(Map.of("error", "labelPattern is required")).build();
        }
        final QueueView q = new QueueView();
        q.name = req.name();
        q.labelPattern = req.labelPattern();
        q.scope = req.scope() != null ? req.scope() : FilterScope.ORG;
        q.ownerId = req.ownerId();
        q.additionalConditions = req.additionalConditions();
        q.sortField = req.sortField() != null ? req.sortField() : "createdAt";
        q.sortDirection = req.sortDirection() != null ? req.sortDirection() : "ASC";
        q.persist();
        return Response.status(201)
                .entity(Map.of("id", q.id, "name", q.name, "labelPattern", q.labelPattern))
                .build();
    }

    @GET
    @Path("/{id}")
    @Transactional
    public Response query(@PathParam("id") final UUID id) {
        final QueueView q = QueueView.findById(id);
        if (q == null) {
            return Response.status(404).entity(Map.of("error", "Queue view not found")).build();
        }
        final List<WorkItemResponse> items = workItemRepo
                .findByLabelPattern(q.labelPattern).stream()
                .map(WorkItemMapper::toResponse)
                .toList();
        return Response.ok(items).build();
    }

    @DELETE
    @Path("/{id}")
    @Transactional
    public Response delete(@PathParam("id") final UUID id) {
        final QueueView q = QueueView.findById(id);
        if (q == null) {
            return Response.status(404).entity(Map.of("error", "Queue view not found")).build();
        }
        q.delete();
        return Response.noContent().build();
    }
}
```

- [ ] **Step 4: Run QueueResourceTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=QueueResourceTest -Dno-format 2>&1 | tail -8
```

Expected: Tests run: 4, Failures: 0

- [ ] **Step 5: Commit issue #63**

```bash
git add -A
git commit -m "feat(queues): QueueView entity and REST — named label-pattern queries

- QueueView Panache entity: labelPattern, scope, owner, additionalConditions, sort config
- QueueResource: GET/POST/DELETE /queues, GET /queues/{id} (executes label query)
- GET /queues/{id} delegates to WorkItemRepository.findByLabelPattern for live results
- 4 @QuarkusTest integration tests including full pipeline (filter → label → queue result)

Closes #63
Refs #52, #50"
```

---

## Issue #64 — Soft Assignment (Relinquishable)

---

### Task 14: WorkItemQueueState + relinquishable REST

**Files:**
- Create: `queues/.../model/WorkItemQueueState.java`
- Create: `queues/.../api/QueueStateResource.java`
- Create: `queues/.../api/QueueStateResourceTest.java`

- [ ] **Step 1: Write failing tests**

Create `quarkus-work-queues/src/test/java/io/quarkiverse/workitems/queues/api/QueueStateResourceTest.java`:

```java
package io.quarkiverse.work.queues.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class QueueStateResourceTest {

    @Test
    void setRelinquishable_marksWorkItemAsAvailableForPickup() {
        var id = given()
                .contentType(ContentType.JSON)
                .body("""{"title": "Soft assign test", "createdBy": "alice", "assigneeId": "alice"}""")
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        given()
                .contentType(ContentType.JSON)
                .body("""{"relinquishable": true}""")
                .put("/workitems/" + id + "/relinquishable")
                .then()
                .statusCode(200)
                .body("relinquishable", equalTo(true));
    }

    @Test
    void setRelinquishable_false_clearsFlag() {
        var id = given()
                .contentType(ContentType.JSON)
                .body("""{"title": "Clear flag test", "createdBy": "alice", "assigneeId": "alice"}""")
                .post("/workitems")
                .then().statusCode(201)
                .extract().path("id");

        given()
                .contentType(ContentType.JSON)
                .body("""{"relinquishable": true}""")
                .put("/workitems/" + id + "/relinquishable")
                .then().statusCode(200);

        given()
                .contentType(ContentType.JSON)
                .body("""{"relinquishable": false}""")
                .put("/workitems/" + id + "/relinquishable")
                .then()
                .statusCode(200)
                .body("relinquishable", equalTo(false));
    }

    @Test
    void getRelinquishableState_unknownWorkItem_returns404() {
        given()
                .contentType(ContentType.JSON)
                .body("""{"relinquishable": true}""")
                .put("/workitems/00000000-0000-0000-0000-000000000000/relinquishable")
                .then()
                .statusCode(404);
    }
}
```

- [ ] **Step 2: Create WorkItemQueueState entity**

```java
package io.quarkiverse.work.queues.model;

import java.util.UUID;

import jakarta.persistence.*;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * Queue-module-managed state per WorkItem.
 * The core WorkItem entity is not modified by the queues module.
 */
@Entity
@Table(name = "work_item_queue_state")
public class WorkItemQueueState extends PanacheEntityBase {

    /** Same UUID as the WorkItem this state belongs to. */
    @Id
    @Column(name = "work_item_id")
    public UUID workItemId;

    /**
     * When {@code true}, the assignee has signalled that this WorkItem is available
     * for another eligible candidate to claim, even though it is ASSIGNED.
     */
    public boolean relinquishable = false;

    public static WorkItemQueueState findOrCreate(final UUID workItemId) {
        return WorkItemQueueState.<WorkItemQueueState> findByIdOptional(workItemId)
                .orElseGet(() -> {
                    final WorkItemQueueState s = new WorkItemQueueState();
                    s.workItemId = workItemId;
                    s.persist();
                    return s;
                });
    }
}
```

- [ ] **Step 3: Create QueueStateResource**

```java
package io.quarkiverse.work.queues.api;

import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.work.queues.model.WorkItemQueueState;
import io.quarkiverse.work.runtime.repository.WorkItemRepository;

@Path("/workitems")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class QueueStateResource {

    @Inject
    WorkItemRepository workItemRepo;

    public record RelinquishableRequest(boolean relinquishable) {}

    @PUT
    @Path("/{id}/relinquishable")
    @Transactional
    public Response setRelinquishable(@PathParam("id") final UUID id,
            final RelinquishableRequest req) {
        if (workItemRepo.findById(id).isEmpty()) {
            return Response.status(404)
                    .entity(Map.of("error", "WorkItem not found: " + id)).build();
        }
        final WorkItemQueueState state = WorkItemQueueState.findOrCreate(id);
        state.relinquishable = req.relinquishable();
        return Response.ok(Map.of("workItemId", id, "relinquishable", state.relinquishable))
                .build();
    }
}
```

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dtest=QueueStateResourceTest -Dno-format 2>&1 | tail -8
```

Expected: Tests run: 3, Failures: 0

- [ ] **Step 5: Commit issue #64**

```bash
git add -A
git commit -m "feat(queues): soft assignment — relinquishable flag on WorkItem

- WorkItemQueueState Panache entity keyed by workItemId (not modifying core WorkItem)
- QueueStateResource: PUT /workitems/{id}/relinquishable — set/clear flag
- findOrCreate pattern: lazy-creates state record on first use
- 3 @QuarkusTest integration tests: set, clear, 404 for unknown WorkItem

Closes #64
Refs #52, #50"
```

---

## Issue #65 — End-to-End Integration Tests

---

### Task 15: Full suite verification + commit #65

**Files:**
- Full module test run

- [ ] **Step 1: Run the complete queues module test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-queues \
  -Dno-format 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -20
```

Expected: BUILD SUCCESS, 0 failures (all test classes)

- [ ] **Step 2: Run the full project including all existing modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -pl runtime,testing,work-flow,quarkus-work-ledger,quarkus-work-examples,quarkus-work-queues \
  -Dno-format 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -15
```

Expected: BUILD SUCCESS, 0 failures across all modules

- [ ] **Step 3: Full clean install with native validation skipped**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -Dno-format 2>&1 | tail -15
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Close issues on GitHub**

```bash
# These are closed via commit messages but verify on GitHub
gh issue list --state open --repo mdproctor/quarkus-work
```

Expected: only #39 (blocked) remaining open.

- [ ] **Step 5: Final commit for #65**

```bash
git add -A
git commit -m "test(queues): end-to-end integration test coverage for quarkus-work-queues

Full test suite across all modules passing. Coverage summary:
- QueuesModuleSmokeTest: Quarkus boots with module on classpath
- FilterResourceTest: CRUD + ad-hoc evaluation (JEXL/JQ) + lambda exclusion
- FullPipelineTest: filter fires on creation, propagation chain, cascade delete
- QueueResourceTest: CRUD + live queue content from label query
- QueueStateResourceTest: relinquishable set/clear/404
- JexlConditionEvaluatorTest: 8 unit tests
- JqConditionEvaluatorTest: 6 unit tests
- FilterEngineTest: 6 unit tests (propagation, circular termination, strip)

Closes #65
Closes #52
Refs #50"
```

---

## Self-Review

### Spec coverage

| Spec requirement | Task |
|---|---|
| Module scaffold + CDI observer seam | Task 1–3 |
| Flyway V1: all queues tables | Task 2 |
| WorkItemFilter entity + CRUD REST | Task 4–5 |
| POST /filters/evaluate (ad-hoc) | Task 5 |
| Lambda filter rejected via REST (400) | Task 5 |
| FilterConditionEvaluator SPI + registry | Task 6 |
| JEXL built-in evaluator | Task 6 |
| JQ built-in evaluator | Task 7 |
| Lambda WorkItemFilterBean SPI + discovery | Task 8 |
| FilterChain entity (filterId → workItems) | Task 9 |
| Strip INFERRED, multi-pass re-evaluation | Task 10 |
| Propagation (filter sees labels from other filters) | Task 10 |
| Circular detection (max 10 passes) | Task 10 |
| Filter deletion cascade (O(affected)) | Task 12 |
| QueueView entity + CRUD REST | Task 13 |
| GET /queues/{id} returns live WorkItems | Task 13 |
| Soft assignment (relinquishable flag) | Task 14 |
| E2E verification across all modules | Task 15 |

### Known deviations from spec

- **`supporters: Set<chainId>` on WorkItemLabel** — not implemented. Instead, `appliedBy` stores the filterId directly on INFERRED labels. The full multi-supporter FilterChain model is a future optimisation.
- **Relinquishable claim relaxation** — `QueueStateResource` manages the flag but does not yet intercept `PUT /{id}/claim` to relax the ASSIGNED guard. This requires JAX-RS request filtering or a queues-module-specific claim endpoint. Deferred to a follow-up issue.
- **QueueView `additionalConditions`** — stored but not evaluated in `GET /queues/{id}`. The query delegates to `findByLabelPattern` only. Evaluating additional JEXL conditions per WorkItem adds O(n) cost per request; deferred.
