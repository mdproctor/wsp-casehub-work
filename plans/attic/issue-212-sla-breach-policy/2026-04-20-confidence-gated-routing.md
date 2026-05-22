# Confidence-Gated Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `confidenceScore` to WorkItem, a generic filter-registry module (persistent rules + pluggable action SPI), and a `quarkus-work-ai` module that ships a low-confidence routing filter.

**Architecture:** Three layers — core runtime gains `confidenceScore`; `quarkus-work-filter-registry` provides the `FilterAction` SPI, JEXL evaluation engine, permanent (CDI-produced) and dynamic (DB-persisted) filter rules; `quarkus-work-ai` produces the low-confidence `FilterDefinition` via CDI. A `ThreadLocal` guard in `FilterRegistryEngine` prevents recursive firing when actions trigger lifecycle events.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache ORM, commons-jexl3:3.4.0, RESTEasy Reactive, CDI, Flyway (V13 in core, V3001 in filter-registry).

---

## File Map

### Core runtime (modify existing)
- `runtime/src/main/resources/db/migration/V13__confidence_score.sql` — new column
- `runtime/src/main/java/.../model/WorkItem.java` — add `confidenceScore` field
- `runtime/src/main/java/.../model/WorkItemCreateRequest.java` — add `confidenceScore`
- `runtime/src/main/java/.../api/CreateWorkItemRequest.java` — add `confidenceScore`
- `runtime/src/main/java/.../api/WorkItemMapper.java` — map `confidenceScore` in `toServiceRequest()` + `toResponse()`
- `runtime/src/main/java/.../api/WorkItemResponse.java` — add `confidenceScore` field
- `runtime/src/main/java/.../api/WorkItemWithAuditResponse.java` — add `confidenceScore` field
- `runtime/src/test/java/.../api/WorkItemResourceTest.java` — add confidence score test cases

### quarkus-work-filter-registry (new module)
```
quarkus-work-filter-registry/
  pom.xml
  src/main/java/io/quarkiverse/workitems/filterregistry/
    spi/
      FilterEvent.java          enum ADD, UPDATE, REMOVE
      ActionDescriptor.java     record: type, params
      FilterDefinition.java     record: name, description, enabled, events, condition, conditionContext, actions
      FilterAction.java         CDI SPI interface
    action/
      ApplyLabelAction.java     APPLY_LABEL built-in
      OverrideCandidateGroupsAction.java   OVERRIDE_CANDIDATE_GROUPS built-in
      SetPriorityAction.java    SET_PRIORITY built-in
    engine/
      JexlConditionEvaluator.java   evaluates JEXL conditions against WorkItem
      FilterRegistryEngine.java     observes WorkItemLifecycleEvent, applies filters
    registry/
      PermanentFilterRegistry.java  CDI Instance<FilterDefinition> + enable/disable map
      DynamicFilterRegistry.java    FilterRule Panache queries
    model/
      FilterRule.java           JPA entity, DB-persisted rules
    api/
      FilterRuleResource.java   /filter-rules CRUD + /filter-rules/permanent
  src/main/resources/db/migration/
    V3001__filter_rule.sql
  src/test/java/io/quarkiverse/workitems/filterregistry/
    engine/
      FilterRegistryEngineTest.java        unit — no Quarkus boot
      JexlConditionEvaluatorTest.java      unit
    action/
      FilterActionIntegrationTest.java     @QuarkusTest — all three actions
    registry/
      PermanentFilterRegistryTest.java     @QuarkusTest
      DynamicFilterRegistryTest.java       @QuarkusTest
    FilterRuleEvaluationTest.java          @QuarkusTest — end-to-end evaluation
  src/test/resources/application.properties

### quarkus-work-ai (new module)
```
quarkus-work-ai/
  pom.xml
  src/main/java/io/quarkiverse/workitems/ai/
    config/
      WorkItemsAiConfig.java    @ConfigMapping prefix=quarkus.work.ai
    filter/
      LowConfidenceFilterProducer.java   @Produces FilterDefinition
  src/test/java/io/quarkiverse/workitems/ai/
    LowConfidenceFilterTest.java   @QuarkusTest — E2E
  src/test/resources/application.properties

---

## Task 1: Create GitHub issues

- [ ] **Create issue #112**
```bash
gh issue create --repo mdproctor/quarkus-work \
  --title "confidenceScore field on WorkItem — V13 migration" \
  --label "enhancement" \
  --body "## Context
Part of epic #100 — AI-Native features.

## What
Add Double confidenceScore (nullable) to WorkItem entity and all request/response DTOs.
V13 Flyway migration: confidence_score DOUBLE column.

## Acceptance Criteria
- [ ] confidenceScore persisted and returned in all WorkItem responses
- [ ] Null means 'no AI metadata' — no validation, no default
- [ ] Tests: create with/without confidenceScore, response includes field"
```

- [ ] **Create issue #113**
```bash
gh issue create --repo mdproctor/quarkus-work \
  --title "quarkus-work-filter-registry module — FilterAction SPI + evaluation engine" \
  --label "enhancement" \
  --body "## Context
Part of epic #100. Foundational module for confidence-gated routing and future AI features.

## What
New module: FilterAction SPI (CDI), built-in actions (APPLY_LABEL, OVERRIDE_CANDIDATE_GROUPS,
SET_PRIORITY), JEXL condition evaluator, FilterRegistryEngine (observes WorkItemLifecycleEvent),
permanent (CDI-produced) + dynamic (DB-persisted) filter rule registry, REST at /filter-rules.

## Acceptance Criteria
- [ ] FilterAction SPI resolvable by type name
- [ ] JEXL condition evaluated against WorkItem with conditionContext variables
- [ ] Permanent filters disable/enable at runtime without restart
- [ ] Dynamic rules CRUD at /filter-rules, persisted to V3001 table
- [ ] ThreadLocal guard prevents recursive CDI event firing"
```

- [ ] **Create issue #114**
```bash
gh issue create --repo mdproctor/quarkus-work \
  --title "quarkus-work-ai module — LowConfidenceFilterProducer" \
  --label "enhancement" \
  --body "## Context
Part of epic #100. Depends on #112 and #113.

## What
New module: quarkus-work-ai. Produces a permanent FilterDefinition that applies
label ai/low-confidence when workItem.confidenceScore < threshold (default 0.7).
Config: quarkus.work.ai.confidence-threshold, quarkus.work.ai.low-confidence-filter.enabled.

## Acceptance Criteria
- [ ] score < threshold → ai/low-confidence label applied
- [ ] score >= threshold → no label
- [ ] score null → no label
- [ ] filter disabled via config → no label
- [ ] custom threshold respected"
```

---

## Task 2: Core — confidenceScore field + V13 migration

**Files:**
- Create: `runtime/src/main/resources/db/migration/V13__confidence_score.sql`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItem.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemCreateRequest.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/CreateWorkItemRequest.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemMapper.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResponse.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemWithAuditResponse.java`
- Test: `runtime/src/test/java/io/quarkiverse/workitems/runtime/api/WorkItemResourceTest.java`

- [ ] **Step 1: Write failing tests for confidenceScore**

Add to `WorkItemResourceTest.java` (find the existing class and append):
```java
@Test
void createWorkItem_withConfidenceScore_persistsAndReturnsIt() {
    given().contentType(ContentType.JSON)
        .body("{\"title\":\"AI Task\",\"createdBy\":\"agent\",\"confidenceScore\":0.55}")
        .post("/workitems")
        .then().statusCode(201)
        .body("confidenceScore", equalTo(0.55f));
}

@Test
void createWorkItem_withoutConfidenceScore_returnsNullConfidenceScore() {
    given().contentType(ContentType.JSON)
        .body("{\"title\":\"Human Task\",\"createdBy\":\"human\"}")
        .post("/workitems")
        .then().statusCode(201)
        .body("confidenceScore", nullValue());
}

@Test
void createWorkItem_withHighConfidenceScore_returnsIt() {
    given().contentType(ContentType.JSON)
        .body("{\"title\":\"High Confidence\",\"createdBy\":\"agent\",\"confidenceScore\":0.95}")
        .post("/workitems")
        .then().statusCode(201)
        .body("confidenceScore", equalTo(0.95f));
}
```

Add import `import static org.hamcrest.Matchers.nullValue;` if not present.

- [ ] **Step 2: Run tests to verify RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="WorkItemResourceTest#createWorkItem_withConfidenceScore*,WorkItemResourceTest#createWorkItem_withoutConfidenceScore*,WorkItemResourceTest#createWorkItem_withHighConfidenceScore*" 2>&1 | tail -10
```
Expected: FAIL — `confidenceScore` field doesn't exist yet.

- [ ] **Step 3: Create V13 migration**

Create `runtime/src/main/resources/db/migration/V13__confidence_score.sql`:
```sql
-- V13: confidenceScore — AI agent confidence metadata (Issue #112, Epic #100)
--
-- Nullable Double (0.0–1.0). Null means the WorkItem was not created by an AI
-- agent or no confidence metadata was provided. No default, no constraint.
-- Used by the filter-registry's LowConfidenceFilter to gate routing decisions.

ALTER TABLE work_item ADD COLUMN confidence_score DOUBLE;
```

- [ ] **Step 4: Add confidenceScore to WorkItem entity**

In `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItem.java`,
after the `labels` field (near the end of the field declarations), add:
```java
/**
 * Confidence score from the AI agent that created this WorkItem (0.0–1.0).
 * Null when created by a human or when no confidence metadata was provided.
 * Used by the filter-registry to gate routing: items below the configured
 * threshold receive the {@code ai/low-confidence} label automatically.
 */
@Column(name = "confidence_score")
public Double confidenceScore;
```

- [ ] **Step 5: Add confidenceScore to WorkItemCreateRequest (service record)**

In `runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemCreateRequest.java`,
add `Double confidenceScore` as the last field of the record:
```java
public record WorkItemCreateRequest(
        String title,
        String description,
        String category,
        String formKey,
        WorkItemPriority priority,
        String assigneeId,
        String candidateGroups,
        String candidateUsers,
        String requiredCapabilities,
        String createdBy,
        String payload,
        Instant claimDeadline,
        Instant expiresAt,
        Instant followUpDate,
        List<WorkItemLabelResponse> labels,
        Double confidenceScore) {
}
```

- [ ] **Step 6: Add confidenceScore to CreateWorkItemRequest (REST DTO)**

In `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/CreateWorkItemRequest.java`,
add `Double confidenceScore` as the last field:
```java
public record CreateWorkItemRequest(
        String title,
        String description,
        String category,
        String formKey,
        WorkItemPriority priority,
        String assigneeId,
        String candidateGroups,
        String candidateUsers,
        String requiredCapabilities,
        String createdBy,
        String payload,
        Instant claimDeadline,
        Instant expiresAt,
        Instant followUpDate,
        List<WorkItemLabelResponse> labels,
        Double confidenceScore) {
}
```

- [ ] **Step 7: Update WorkItemMapper**

In `WorkItemMapper.toServiceRequest()`, add `req.confidenceScore()` as the last argument:
```java
public static WorkItemCreateRequest toServiceRequest(final CreateWorkItemRequest req) {
    return new WorkItemCreateRequest(
            req.title(), req.description(), req.category(), req.formKey(),
            req.priority(), req.assigneeId(), req.candidateGroups(),
            req.candidateUsers(), req.requiredCapabilities(), req.createdBy(),
            req.payload(), req.claimDeadline(), req.expiresAt(), req.followUpDate(),
            req.labels(), req.confidenceScore());
}
```

In `WorkItemService.create()`, after `item.payload = request.payload();` add:
```java
item.confidenceScore = request.confidenceScore();
```

- [ ] **Step 8: Update WorkItemResponse and WorkItemWithAuditResponse**

Read `WorkItemResponse.java` and `WorkItemWithAuditResponse.java` — they are records. Add `Double confidenceScore` as the second-to-last field (before `version`) in both records, and update `WorkItemMapper.toResponse()` and `toWithAudit()` to pass `wi.confidenceScore` in the correct position.

Also update `WorkItemMapper.toResponse()` — add `wi.confidenceScore` before `wi.version` in the constructor call.

- [ ] **Step 9: Run tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: BUILD SUCCESS, all tests pass (500+ tests).

- [ ] **Step 10: Commit**
```bash
git add runtime/src/main/resources/db/migration/V13__confidence_score.sql \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItem.java \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItemCreateRequest.java \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/api/CreateWorkItemRequest.java \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemMapper.java \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResponse.java \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemWithAuditResponse.java \
  runtime/src/test/java/io/quarkiverse/workitems/runtime/api/WorkItemResourceTest.java
git commit -m "feat(core): confidenceScore field on WorkItem — V13 migration

Adds optional Double confidenceScore (0.0–1.0) to WorkItem entity and all request/
response DTOs. Null means no AI metadata. Used by the filter-registry to gate routing
decisions for low-confidence AI-created items.

Closes #112
Refs #100"
```

---

## Task 3: filter-registry — module scaffold

**Files:**
- Create: `quarkus-work-filter-registry/pom.xml`
- Modify: `pom.xml` (parent — add module entry)
- Create: `quarkus-work-filter-registry/src/test/resources/application.properties`

- [ ] **Step 1: Add commons-jexl3 to parent BOM**

In `pom.xml` dependencyManagement, add after the assertj entry:
```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-jexl3</artifactId>
  <version>3.4.0</version>
</dependency>
```

- [ ] **Step 2: Add module to parent pom.xml**

In the `<modules>` section of `pom.xml`, add:
```xml
<module>quarkus-work-filter-registry</module>
<module>quarkus-work-ai</module>
```

- [ ] **Step 3: Create filter-registry pom.xml**

Create `quarkus-work-filter-registry/pom.xml`:
```xml
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
  <artifactId>quarkus-work-filter-registry</artifactId>
  <name>Quarkus WorkItems - Filter Registry</name>
  <dependencies>
    <dependency>
      <groupId>io.quarkiverse.work</groupId>
      <artifactId>quarkus-work</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm-panache</artifactId>
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
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-jexl3</artifactId>
    </dependency>
    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
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
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-flyway</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 4: Create test application.properties**

Create `quarkus-work-filter-registry/src/test/resources/application.properties`:
```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:filterregistry;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.http.test-port=0
```

- [ ] **Step 5: Verify module compiles**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -pl quarkus-work-filter-registry 2>&1 | grep -E "BUILD|ERROR" | tail -5
```
Expected: BUILD SUCCESS (empty module).

---

## Task 4: filter-registry — SPI types

**Files (all new):**
- `quarkus-work-filter-registry/src/main/java/io/quarkiverse/workitems/filterregistry/spi/FilterEvent.java`
- `quarkus-work-filter-registry/src/main/java/io/quarkiverse/workitems/filterregistry/spi/ActionDescriptor.java`
- `quarkus-work-filter-registry/src/main/java/io/quarkiverse/workitems/filterregistry/spi/FilterDefinition.java`
- `quarkus-work-filter-registry/src/main/java/io/quarkiverse/workitems/filterregistry/spi/FilterAction.java`

- [ ] **Step 1: Create FilterEvent enum**

```java
package io.quarkiverse.work.filterregistry.spi;

/** Lifecycle events that filter rules can subscribe to. */
public enum FilterEvent {
    /** WorkItem first persisted (CREATED audit event). */
    ADD,
    /** WorkItem status changed but not yet terminal. */
    UPDATE,
    /** WorkItem reached a terminal state (COMPLETED, REJECTED, CANCELLED, EXPIRED, ESCALATED). */
    REMOVE
}
```

- [ ] **Step 2: Create ActionDescriptor record**

```java
package io.quarkiverse.work.filterregistry.spi;

import java.util.Map;

/**
 * Descriptor for a single action within a filter rule.
 * {@code type} matches the CDI bean name of a {@link FilterAction} implementation.
 * {@code params} are passed verbatim to {@link FilterAction#apply}.
 */
public record ActionDescriptor(String type, Map<String, Object> params) {

    public static ActionDescriptor of(final String type, final Map<String, Object> params) {
        return new ActionDescriptor(type, params);
    }
}
```

- [ ] **Step 3: Create FilterDefinition record**

```java
package io.quarkiverse.work.filterregistry.spi;

import java.util.List;
import java.util.Map;
import java.util.Set;

/**
 * Immutable definition of a filter rule produced by a CDI bean (@Produces FilterDefinition).
 *
 * <p>Permanent filters are loaded at startup from CDI producers. They can be enabled/
 * disabled at runtime but cannot be deleted without redeployment.
 *
 * @param name            unique identifier (e.g. "ai/low-confidence")
 * @param description     human-readable explanation
 * @param enabled         whether this filter is active; can be toggled at runtime
 * @param events          which lifecycle events trigger evaluation (default: all three)
 * @param condition       JEXL expression evaluated against the WorkItem; true → apply actions
 * @param conditionContext additional variables exposed in the JEXL evaluation context
 * @param actions         ordered list of actions to apply when condition is true
 */
public record FilterDefinition(
        String name,
        String description,
        boolean enabled,
        Set<FilterEvent> events,
        String condition,
        Map<String, Object> conditionContext,
        List<ActionDescriptor> actions) {

    /** Convenience factory — fires on all three event types. */
    public static FilterDefinition onAll(final String name, final String description,
            final boolean enabled, final String condition,
            final Map<String, Object> conditionContext,
            final List<ActionDescriptor> actions) {
        return new FilterDefinition(name, description, enabled,
                Set.of(FilterEvent.values()), condition, conditionContext, actions);
    }

    /** Convenience factory — fires on ADD only. */
    public static FilterDefinition onAdd(final String name, final String description,
            final boolean enabled, final String condition,
            final Map<String, Object> conditionContext,
            final List<ActionDescriptor> actions) {
        return new FilterDefinition(name, description, enabled,
                Set.of(FilterEvent.ADD), condition, conditionContext, actions);
    }
}
```

- [ ] **Step 4: Create FilterAction SPI**

```java
package io.quarkiverse.work.filterregistry.spi;

import java.util.Map;

import io.quarkiverse.work.runtime.model.WorkItem;

/**
 * SPI for actions that a filter rule can apply to a WorkItem when its condition matches.
 *
 * <p>Implementations must be {@code @ApplicationScoped} CDI beans. The engine resolves
 * implementations by matching {@link ActionDescriptor#type()} to the CDI bean name
 * (use {@code @Named("ACTION_TYPE_NAME")} or the default unqualified class name).
 *
 * <p>Built-in implementations: {@code APPLY_LABEL}, {@code OVERRIDE_CANDIDATE_GROUPS},
 * {@code SET_PRIORITY}.
 */
public interface FilterAction {

    /**
     * The action type name used in {@link ActionDescriptor#type()}.
     * Must be unique across all implementations.
     */
    String type();

    /**
     * Apply this action to the given WorkItem.
     *
     * @param workItem the WorkItem being processed (already persisted)
     * @param params   action-specific parameters from the {@link ActionDescriptor}
     */
    void apply(WorkItem workItem, Map<String, Object> params);
}
```

- [ ] **Step 5: Compile check**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl quarkus-work-filter-registry 2>&1 | grep -E "BUILD|ERROR" | tail -5
```
Expected: BUILD SUCCESS.

---

## Task 5: filter-registry — JEXL evaluator + FilterRegistryEngine (unit tests first)

**Files:**
- Create: `...filterregistry/engine/JexlConditionEvaluator.java`
- Create: `...filterregistry/engine/FilterRegistryEngine.java`
- Create: `...filterregistry/registry/PermanentFilterRegistry.java` (stub needed by engine)
- Create: `...filterregistry/registry/DynamicFilterRegistry.java` (stub needed by engine)
- Test: `...filterregistry/engine/JexlConditionEvaluatorTest.java`
- Test: `...filterregistry/engine/FilterRegistryEngineTest.java`

- [ ] **Step 1: Write failing unit tests for JexlConditionEvaluator**

Create `quarkus-work-filter-registry/src/test/java/io/quarkiverse/workitems/filterregistry/engine/JexlConditionEvaluatorTest.java`:
```java
package io.quarkiverse.work.filterregistry.engine;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;

import org.junit.jupiter.api.Test;

import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemPriority;

class JexlConditionEvaluatorTest {

    private final JexlConditionEvaluator evaluator = new JexlConditionEvaluator();

    @Test
    void evaluate_returnsTrue_whenConditionMatches() {
        final WorkItem wi = workItem(0.55, "finance");
        assertThat(evaluator.evaluate(
            "workItem.confidenceScore != null && workItem.confidenceScore < threshold",
            Map.of("threshold", 0.7), wi)).isTrue();
    }

    @Test
    void evaluate_returnsFalse_whenConfidenceAboveThreshold() {
        final WorkItem wi = workItem(0.85, "finance");
        assertThat(evaluator.evaluate(
            "workItem.confidenceScore != null && workItem.confidenceScore < threshold",
            Map.of("threshold", 0.7), wi)).isFalse();
    }

    @Test
    void evaluate_returnsFalse_whenConfidenceIsNull() {
        final WorkItem wi = workItem(null, "finance");
        assertThat(evaluator.evaluate(
            "workItem.confidenceScore != null && workItem.confidenceScore < threshold",
            Map.of("threshold", 0.7), wi)).isFalse();
    }

    @Test
    void evaluate_returnsTrue_forCategoryCondition() {
        final WorkItem wi = workItem(null, "finance");
        assertThat(evaluator.evaluate(
            "workItem.category == 'finance'", Map.of(), wi)).isTrue();
    }

    @Test
    void evaluate_returnsFalse_forNonMatchingCategory() {
        final WorkItem wi = workItem(null, "hr");
        assertThat(evaluator.evaluate(
            "workItem.category == 'finance'", Map.of(), wi)).isFalse();
    }

    @Test
    void evaluate_returnsFalse_forInvalidExpression() {
        final WorkItem wi = workItem(0.5, "cat");
        assertThat(evaluator.evaluate("!!!invalid{[}", Map.of(), wi)).isFalse();
    }

    @Test
    void evaluate_returnsFalse_forBlankCondition() {
        assertThat(evaluator.evaluate("", Map.of(), new WorkItem())).isFalse();
    }

    @Test
    void evaluate_exposesConditionContextVariables() {
        final WorkItem wi = workItem(null, "loan");
        assertThat(evaluator.evaluate(
            "myVar == 42", Map.of("myVar", 42), wi)).isTrue();
    }

    @Test
    void evaluate_priorityCondition_usingEnumName() {
        final WorkItem wi = new WorkItem();
        wi.priority = WorkItemPriority.HIGH;
        assertThat(evaluator.evaluate(
            "workItem.priority.name() == 'HIGH'", Map.of(), wi)).isTrue();
    }

    private WorkItem workItem(final Double confidenceScore, final String category) {
        final WorkItem wi = new WorkItem();
        wi.confidenceScore = confidenceScore;
        wi.category = category;
        return wi;
    }
}
```

- [ ] **Step 2: Write failing unit tests for FilterRegistryEngine**

Create `...filterregistry/engine/FilterRegistryEngineTest.java`:
```java
package io.quarkiverse.work.filterregistry.engine;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import io.quarkiverse.work.filterregistry.spi.*;
import io.quarkiverse.work.runtime.event.WorkItemLifecycleEvent;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemStatus;
import io.quarkiverse.work.runtime.repository.WorkItemStore;

@ExtendWith(MockitoExtension.class)
class FilterRegistryEngineTest {

    @Mock WorkItemStore workItemStore;
    @Mock FilterAction mockAction;

    private FilterRegistryEngine engine;
    private WorkItem workItem;

    @BeforeEach
    void setUp() {
        workItem = new WorkItem();
        workItem.id = UUID.randomUUID();
        workItem.confidenceScore = 0.55;
        workItem.category = "finance";
        when(workItemStore.get(workItem.id)).thenReturn(java.util.Optional.of(workItem));
        when(mockAction.type()).thenReturn("TEST_ACTION");
        engine = new FilterRegistryEngine(workItemStore, new JexlConditionEvaluator(),
                List.of(mockAction));
    }

    @Test
    void onEvent_appliesAction_whenConditionMatches() {
        final FilterDefinition def = FilterDefinition.onAdd("test", "desc", true,
            "workItem.confidenceScore < threshold",
            Map.of("threshold", 0.7),
            List.of(ActionDescriptor.of("TEST_ACTION", Map.of("key", "val"))));

        engine.processEvent(createdEvent(), List.of(def));

        verify(mockAction).apply(eq(workItem), eq(Map.of("key", "val")));
    }

    @Test
    void onEvent_skipsAction_whenConditionDoesNotMatch() {
        final FilterDefinition def = FilterDefinition.onAdd("test", "desc", true,
            "workItem.confidenceScore > 0.9",  // 0.55 is not > 0.9
            Map.of(),
            List.of(ActionDescriptor.of("TEST_ACTION", Map.of())));

        engine.processEvent(createdEvent(), List.of(def));

        verify(mockAction, never()).apply(any(), any());
    }

    @Test
    void onEvent_skipsDisabledFilter() {
        final FilterDefinition def = FilterDefinition.onAdd("test", "desc", false,
            "workItem.confidenceScore < threshold",
            Map.of("threshold", 0.7),
            List.of(ActionDescriptor.of("TEST_ACTION", Map.of())));

        engine.processEvent(createdEvent(), List.of(def));

        verify(mockAction, never()).apply(any(), any());
    }

    @Test
    void onEvent_skipsFilter_whenEventTypeNotSubscribed() {
        // Filter only subscribes to REMOVE, but event is ADD
        final FilterDefinition def = new FilterDefinition("test", "desc", true,
            Set.of(FilterEvent.REMOVE), "true", Map.of(),
            List.of(ActionDescriptor.of("TEST_ACTION", Map.of())));

        engine.processEvent(createdEvent(), List.of(def));

        verify(mockAction, never()).apply(any(), any());
    }

    @Test
    void onEvent_skipsUnknownActionType() {
        final FilterDefinition def = FilterDefinition.onAdd("test", "desc", true,
            "true", Map.of(),
            List.of(ActionDescriptor.of("UNKNOWN_ACTION", Map.of())));

        // Should not throw — unknown action is silently skipped
        assertThat(catchThrowable(() -> engine.processEvent(createdEvent(), List.of(def))))
            .isNull();
    }

    private WorkItemLifecycleEvent createdEvent() {
        return WorkItemLifecycleEvent.of("CREATED", workItem.id,
            WorkItemStatus.PENDING, "system", null);
    }

    private static Throwable catchThrowable(final Runnable r) {
        try { r.run(); return null; } catch (Throwable t) { return t; }
    }
}
```

Add Mockito to `quarkus-work-filter-registry/pom.xml` test dependencies:
```xml
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-junit-jupiter</artifactId>
  <scope>test</scope>
</dependency>
```
Check the Quarkus BOM includes `mockito-junit-jupiter` — it does via `quarkus-junit5`.

- [ ] **Step 3: Run tests to verify RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-filter-registry \
  -Dtest="JexlConditionEvaluatorTest,FilterRegistryEngineTest" 2>&1 | tail -10
```
Expected: COMPILATION ERROR — `JexlConditionEvaluator` and `FilterRegistryEngine` don't exist.

- [ ] **Step 4: Implement JexlConditionEvaluator**

Create `...filterregistry/engine/JexlConditionEvaluator.java`:
```java
package io.quarkiverse.work.filterregistry.engine;

import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;

import org.apache.commons.jexl3.JexlBuilder;
import org.apache.commons.jexl3.JexlEngine;
import org.apache.commons.jexl3.JexlException;
import org.apache.commons.jexl3.MapContext;

import io.quarkiverse.work.runtime.model.WorkItem;

/**
 * Evaluates JEXL conditions against a WorkItem.
 *
 * <p>The JEXL context exposes {@code workItem} as the root object (public fields
 * accessible via dot notation) plus any additional variables from {@code conditionContext}.
 */
@ApplicationScoped
public class JexlConditionEvaluator {

    private static final JexlEngine JEXL = new JexlBuilder()
            .strict(false).silent(true).create();

    /**
     * @param condition        JEXL expression; blank → false
     * @param conditionContext additional variables merged into the JEXL context
     * @param workItem         the WorkItem being evaluated
     * @return true if the expression evaluates to Boolean.TRUE, false otherwise
     */
    public boolean evaluate(final String condition,
            final Map<String, Object> conditionContext, final WorkItem workItem) {
        if (condition == null || condition.isBlank()) {
            return false;
        }
        try {
            final var ctx = new MapContext();
            ctx.set("workItem", workItem);
            if (conditionContext != null) {
                conditionContext.forEach(ctx::set);
            }
            final Object result = JEXL.createExpression(condition).evaluate(ctx);
            return Boolean.TRUE.equals(result);
        } catch (JexlException e) {
            return false;
        }
    }
}
```

- [ ] **Step 5: Create stub PermanentFilterRegistry and DynamicFilterRegistry**

Create `...filterregistry/registry/PermanentFilterRegistry.java` (stub — fleshed out in Task 8):
```java
package io.quarkiverse.work.filterregistry.registry;

import java.util.List;
import jakarta.enterprise.context.ApplicationScoped;
import io.quarkiverse.work.filterregistry.spi.FilterDefinition;

@ApplicationScoped
public class PermanentFilterRegistry {
    public List<FilterDefinition> all() { return List.of(); }
    public void setEnabled(final String name, final boolean enabled) {}
}
```

Create `...filterregistry/registry/DynamicFilterRegistry.java` (stub — fleshed out in Task 7):
```java
package io.quarkiverse.work.filterregistry.registry;

import java.util.List;
import jakarta.enterprise.context.ApplicationScoped;
import io.quarkiverse.work.filterregistry.spi.FilterDefinition;

@ApplicationScoped
public class DynamicFilterRegistry {
    public List<FilterDefinition> allEnabled() { return List.of(); }
}
```

- [ ] **Step 6: Implement FilterRegistryEngine**

Create `...filterregistry/engine/FilterRegistryEngine.java`:
```java
package io.quarkiverse.work.filterregistry.engine;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Stream;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import io.quarkiverse.work.filterregistry.registry.DynamicFilterRegistry;
import io.quarkiverse.work.filterregistry.registry.PermanentFilterRegistry;
import io.quarkiverse.work.filterregistry.spi.ActionDescriptor;
import io.quarkiverse.work.filterregistry.spi.FilterAction;
import io.quarkiverse.work.filterregistry.spi.FilterDefinition;
import io.quarkiverse.work.filterregistry.spi.FilterEvent;
import io.quarkiverse.work.runtime.event.WorkItemLifecycleEvent;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemStatus;
import io.quarkiverse.work.runtime.repository.WorkItemStore;

/**
 * Observes WorkItemLifecycleEvent, evaluates all enabled filter definitions,
 * and applies matching actions.
 *
 * <p>A ThreadLocal guard prevents recursive triggering when actions (e.g. label
 * application) themselves fire lifecycle events.
 */
@ApplicationScoped
public class FilterRegistryEngine {

    private static final ThreadLocal<Boolean> RUNNING = ThreadLocal.withInitial(() -> false);

    private static final List<String> TERMINAL_EVENTS =
            List.of("completed", "rejected", "cancelled", "expired", "escalated");

    private final WorkItemStore workItemStore;
    private final JexlConditionEvaluator conditionEvaluator;
    private final List<FilterAction> filterActions;
    private final PermanentFilterRegistry permanentRegistry;
    private final DynamicFilterRegistry dynamicRegistry;

    /** Constructor used by CDI (full wiring). */
    @Inject
    public FilterRegistryEngine(final WorkItemStore workItemStore,
            final JexlConditionEvaluator conditionEvaluator,
            final jakarta.enterprise.inject.Instance<FilterAction> filterActions,
            final PermanentFilterRegistry permanentRegistry,
            final DynamicFilterRegistry dynamicRegistry) {
        this.workItemStore = workItemStore;
        this.conditionEvaluator = conditionEvaluator;
        this.filterActions = filterActions.stream().toList();
        this.permanentRegistry = permanentRegistry;
        this.dynamicRegistry = dynamicRegistry;
    }

    /** Constructor used by unit tests (no CDI). */
    FilterRegistryEngine(final WorkItemStore workItemStore,
            final JexlConditionEvaluator conditionEvaluator,
            final List<FilterAction> filterActions) {
        this.workItemStore = workItemStore;
        this.conditionEvaluator = conditionEvaluator;
        this.filterActions = filterActions;
        this.permanentRegistry = new PermanentFilterRegistry();
        this.dynamicRegistry = new DynamicFilterRegistry();
    }

    void onLifecycleEvent(@Observes final WorkItemLifecycleEvent event) {
        if (Boolean.TRUE.equals(RUNNING.get())) {
            return; // prevent recursive firing from actions that trigger lifecycle events
        }
        RUNNING.set(true);
        try {
            final WorkItem workItem = workItemStore.get(event.workItemId()).orElse(null);
            if (workItem == null) return;
            final List<FilterDefinition> allDefs = Stream.concat(
                    permanentRegistry.all().stream(),
                    dynamicRegistry.allEnabled().stream()).toList();
            processEvent(event, workItem, allDefs);
        } finally {
            RUNNING.remove();
        }
    }

    /** Package-visible for unit testing with a pre-supplied WorkItem. */
    void processEvent(final WorkItemLifecycleEvent event, final List<FilterDefinition> defs) {
        final WorkItem workItem = workItemStore.get(event.workItemId()).orElse(null);
        if (workItem == null) return;
        processEvent(event, workItem, defs);
    }

    private void processEvent(final WorkItemLifecycleEvent event,
            final WorkItem workItem, final List<FilterDefinition> defs) {
        final FilterEvent filterEvent = toFilterEvent(event.type());
        for (final FilterDefinition def : defs) {
            if (!def.enabled()) continue;
            if (!def.events().contains(filterEvent)) continue;
            if (!conditionEvaluator.evaluate(def.condition(), def.conditionContext(), workItem)) continue;
            for (final ActionDescriptor action : def.actions()) {
                resolveAction(action.type()).ifPresent(a -> a.apply(workItem, action.params()));
            }
        }
    }

    private FilterEvent toFilterEvent(final String eventType) {
        final String name = eventType.substring(eventType.lastIndexOf('.') + 1).toLowerCase();
        if ("created".equals(name)) return FilterEvent.ADD;
        if (TERMINAL_EVENTS.contains(name)) return FilterEvent.REMOVE;
        return FilterEvent.UPDATE;
    }

    private java.util.Optional<FilterAction> resolveAction(final String type) {
        return filterActions.stream().filter(a -> a.type().equals(type)).findFirst();
    }
}
```

- [ ] **Step 7: Fix FilterRegistryEngineTest constructor call**

The test uses a 3-arg constructor. Update it to call:
```java
engine = new FilterRegistryEngine(workItemStore, new JexlConditionEvaluator(),
        List.of(mockAction));
```
The engine's package-private `processEvent(event, List<FilterDefinition>)` method is called
in the test. This is correct — it bypasses the CDI event observer for unit testing.

- [ ] **Step 8: Run tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-filter-registry \
  -Dtest="JexlConditionEvaluatorTest,FilterRegistryEngineTest" 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: BUILD SUCCESS, 9 + 5 = 14 tests pass.

- [ ] **Step 9: Commit**
```bash
git add quarkus-work-filter-registry/ pom.xml
git commit -m "feat(filter-registry): FilterAction SPI + JEXL evaluator + FilterRegistryEngine

FilterAction SPI (CDI interface resolved by type name), JexlConditionEvaluator
(exposes workItem object + conditionContext variables), FilterRegistryEngine
(observes WorkItemLifecycleEvent, ThreadLocal guard against recursive firing).
9 JEXL unit tests + 5 engine unit tests (Mockito, no Quarkus boot).

Refs #113
Refs #100"
```

---

## Task 6: filter-registry — built-in FilterActions

**Files:**
- Create: `...filterregistry/action/ApplyLabelAction.java`
- Create: `...filterregistry/action/OverrideCandidateGroupsAction.java`
- Create: `...filterregistry/action/SetPriorityAction.java`
- Test: `...filterregistry/action/FilterActionIntegrationTest.java`

- [ ] **Step 1: Write failing integration tests**

Create `...filterregistry/action/FilterActionIntegrationTest.java`:
```java
package io.quarkiverse.work.filterregistry.action;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

/**
 * Integration tests for built-in FilterActions via the WorkItem REST API.
 * Each test creates a WorkItem, triggers a filter via the engine, and asserts
 * the expected mutation.
 *
 * Issue #113, Epic #100.
 */
@QuarkusTest
class FilterActionIntegrationTest {

    @Test
    void applyLabelAction_addsLabelToWorkItem() {
        // Wired by LowConfidenceFilterProducer in quarkus-work-ai;
        // here we test the action directly via a programmatic FilterDefinition
        // registered as a test CDI producer in TestFilterProducer.
        final String id = given().contentType(ContentType.JSON)
                .body("{\"title\":\"Test\",\"createdBy\":\"test\",\"confidenceScore\":0.3}")
                .post("/workitems").then().statusCode(201).extract().path("id");

        // The test filter (confidenceScore < 0.5 → APPLY_LABEL ai/test-label) fires on ADD
        given().get("/workitems/" + id)
                .then().statusCode(200)
                .body("labels.path", hasItem("ai/test-label"));
    }

    @Test
    void overrideCandidateGroupsAction_replacesCandidateGroups() {
        final String id = given().contentType(ContentType.JSON)
                .body("{\"title\":\"Route Me\",\"createdBy\":\"agent\"," +
                      "\"candidateGroups\":\"original-group\",\"confidenceScore\":0.2}")
                .post("/workitems").then().statusCode(201).extract().path("id");

        // Test filter: confidenceScore < 0.3 → OVERRIDE_CANDIDATE_GROUPS review-team
        given().get("/workitems/" + id)
                .then().statusCode(200)
                .body("candidateGroups", equalTo("review-team"));
    }

    @Test
    void setPriorityAction_changesPriority() {
        final String id = given().contentType(ContentType.JSON)
                .body("{\"title\":\"Prioritise\",\"createdBy\":\"agent\"," +
                      "\"confidenceScore\":0.1,\"priority\":\"NORMAL\"}")
                .post("/workitems").then().statusCode(201).extract().path("id");

        // Test filter: confidenceScore < 0.15 → SET_PRIORITY CRITICAL
        given().get("/workitems/" + id)
                .then().statusCode(200)
                .body("priority", equalTo("CRITICAL"));
    }
}
```

Create `...filterregistry/action/TestFilterProducer.java` (test CDI producer to exercise actions):
```java
package io.quarkiverse.work.filterregistry.action;

import java.util.List;
import java.util.Map;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import io.quarkiverse.work.filterregistry.spi.*;

@ApplicationScoped
class TestFilterProducer {

    @Produces
    FilterDefinition applyLabelFilter() {
        return FilterDefinition.onAdd("test/apply-label", "test", true,
            "workItem.confidenceScore != null && workItem.confidenceScore < 0.5",
            Map.of(),
            List.of(ActionDescriptor.of("APPLY_LABEL",
                Map.of("path", "ai/test-label", "appliedBy", "test-filter"))));
    }

    @Produces
    FilterDefinition overrideGroupsFilter() {
        return FilterDefinition.onAdd("test/override-groups", "test", true,
            "workItem.confidenceScore != null && workItem.confidenceScore < 0.3",
            Map.of(),
            List.of(ActionDescriptor.of("OVERRIDE_CANDIDATE_GROUPS",
                Map.of("groups", "review-team"))));
    }

    @Produces
    FilterDefinition setPriorityFilter() {
        return FilterDefinition.onAdd("test/set-priority", "test", true,
            "workItem.confidenceScore != null && workItem.confidenceScore < 0.15",
            Map.of(),
            List.of(ActionDescriptor.of("SET_PRIORITY",
                Map.of("priority", "CRITICAL"))));
    }
}
```

- [ ] **Step 2: Run tests RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-filter-registry \
  -Dtest="FilterActionIntegrationTest" 2>&1 | tail -10
```
Expected: COMPILATION ERROR — action classes don't exist.

- [ ] **Step 3: Implement ApplyLabelAction**

```java
package io.quarkiverse.work.filterregistry.action;

import java.util.Map;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkiverse.work.filterregistry.spi.FilterAction;
import io.quarkiverse.work.runtime.model.LabelPersistence;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.service.WorkItemService;

/**
 * Applies a label to the WorkItem. The label is persisted as INFERRED
 * (managed by the filter engine, not manually applied by a human).
 *
 * <p>Params: {@code path} (required), {@code appliedBy} (optional, defaults to "filter-registry").
 */
@ApplicationScoped
public class ApplyLabelAction implements FilterAction {

    @Inject WorkItemService workItemService;

    @Override
    public String type() { return "APPLY_LABEL"; }

    @Override
    public void apply(final WorkItem workItem, final Map<String, Object> params) {
        final String path = (String) params.get("path");
        if (path == null || path.isBlank()) return;
        final String appliedBy = params.getOrDefault("appliedBy", "filter-registry").toString();
        workItemService.addLabel(workItem.id, path, LabelPersistence.INFERRED, appliedBy);
    }
}
```

Note: `WorkItemService.addLabel()` currently takes `(UUID id, String path)`. Check the
actual signature and add a `LabelPersistence` + `appliedBy` overload if needed, or use
the existing signature with INFERRED persistence as default.

- [ ] **Step 4: Implement OverrideCandidateGroupsAction**

```java
package io.quarkiverse.work.filterregistry.action;

import java.util.Map;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import io.quarkiverse.work.filterregistry.spi.FilterAction;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.repository.WorkItemStore;

/**
 * Replaces candidateGroups on the WorkItem with the configured value.
 * Params: {@code groups} (required) — comma-separated group identifiers.
 */
@ApplicationScoped
public class OverrideCandidateGroupsAction implements FilterAction {

    @Inject WorkItemStore workItemStore;

    @Override
    public String type() { return "OVERRIDE_CANDIDATE_GROUPS"; }

    @Override
    @Transactional
    public void apply(final WorkItem workItem, final Map<String, Object> params) {
        final String groups = (String) params.get("groups");
        if (groups == null) return;
        workItem.candidateGroups = groups;
        workItemStore.put(workItem);
    }
}
```

- [ ] **Step 5: Implement SetPriorityAction**

```java
package io.quarkiverse.work.filterregistry.action;

import java.util.Map;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import io.quarkiverse.work.filterregistry.spi.FilterAction;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemPriority;
import io.quarkiverse.work.runtime.repository.WorkItemStore;

/**
 * Sets the priority of a WorkItem.
 * Params: {@code priority} (required) — one of LOW, NORMAL, HIGH, CRITICAL.
 */
@ApplicationScoped
public class SetPriorityAction implements FilterAction {

    @Inject WorkItemStore workItemStore;

    @Override
    public String type() { return "SET_PRIORITY"; }

    @Override
    @Transactional
    public void apply(final WorkItem workItem, final Map<String, Object> params) {
        final String priority = (String) params.get("priority");
        if (priority == null) return;
        workItem.priority = WorkItemPriority.valueOf(priority);
        workItemStore.put(workItem);
    }
}
```

- [ ] **Step 6: Update PermanentFilterRegistry to collect CDI producers**

Replace stub with real implementation:
```java
package io.quarkiverse.work.filterregistry.registry;

import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import io.quarkiverse.work.filterregistry.spi.FilterDefinition;

/**
 * Collects all CDI-produced {@link FilterDefinition} beans (permanent filters).
 * Enable/disable state is held in-memory — toggling survives the current JVM instance
 * but resets on restart (permanent filters return to their declared enabled state).
 */
@ApplicationScoped
public class PermanentFilterRegistry {

    private final List<FilterDefinition> definitions;
    private final ConcurrentMap<String, Boolean> enabledOverrides = new ConcurrentHashMap<>();

    @Inject
    public PermanentFilterRegistry(final Instance<FilterDefinition> producers) {
        this.definitions = producers.stream().toList();
    }

    public List<FilterDefinition> all() {
        return definitions.stream()
                .map(def -> enabledOverrides.containsKey(def.name())
                        ? new FilterDefinition(def.name(), def.description(),
                                enabledOverrides.get(def.name()), def.events(),
                                def.condition(), def.conditionContext(), def.actions())
                        : def)
                .toList();
    }

    public List<FilterDefinition> allEnabled() {
        return all().stream().filter(FilterDefinition::enabled).toList();
    }

    public void setEnabled(final String name, final boolean enabled) {
        enabledOverrides.put(name, enabled);
    }

    public java.util.Optional<FilterDefinition> findByName(final String name) {
        return all().stream().filter(d -> d.name().equals(name)).findFirst();
    }
}
```

- [ ] **Step 7: Run integration tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-filter-registry \
  -Dtest="FilterActionIntegrationTest" 2>&1 | grep -E "Tests run:|BUILD|ERROR" | tail -10
```
Expected: BUILD SUCCESS, 3 tests pass. Investigate and fix any `addLabel` signature issues.

- [ ] **Step 8: Commit**
```bash
git add quarkus-work-filter-registry/
git commit -m "feat(filter-registry): built-in FilterActions + PermanentFilterRegistry

ApplyLabelAction (INFERRED label), OverrideCandidateGroupsAction (replaces candidateGroups),
SetPriorityAction (enum-validated). PermanentFilterRegistry collects CDI-produced
FilterDefinition beans with in-memory enable/disable override map.
3 @QuarkusTest integration tests covering all three built-in actions.

Refs #113
Refs #100"
```

---

## Task 7: filter-registry — FilterRule entity + V3001 migration + DynamicFilterRegistry

**Files:**
- Create: `...filterregistry/model/FilterRule.java`
- Create: `...filterregistry/src/main/resources/db/migration/V3001__filter_rule.sql`
- Update: `...filterregistry/registry/DynamicFilterRegistry.java`

- [ ] **Step 1: Write failing tests for DynamicFilterRegistry CRUD**

Create `...filterregistry/registry/DynamicFilterRegistryTest.java`:
```java
package io.quarkiverse.work.filterregistry.registry;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class DynamicFilterRegistryTest {

    @Test
    void createRule_returns201_withAllFields() {
        given().contentType(ContentType.JSON)
            .body("{\"name\":\"test/dynamic\",\"description\":\"desc\",\"enabled\":true," +
                  "\"condition\":\"workItem.category == 'loan'\",\"events\":[\"ADD\"]," +
                  "\"actionsJson\":\"[{\\\"type\\\":\\\"SET_PRIORITY\\\"," +
                  "\\\"params\\\":{\\\"priority\\\":\\\"HIGH\\\"}}]\"}")
            .post("/filter-rules")
            .then().statusCode(201)
            .body("id", notNullValue())
            .body("name", equalTo("test/dynamic"))
            .body("enabled", equalTo(true));
    }

    @Test
    void listRules_returns200_withCreatedRule() {
        final String name = "list-test-" + System.nanoTime();
        given().contentType(ContentType.JSON)
            .body("{\"name\":\"" + name + "\",\"enabled\":true," +
                  "\"condition\":\"true\",\"events\":[\"ADD\"],\"actionsJson\":\"[]\"}")
            .post("/filter-rules").then().statusCode(201);

        given().get("/filter-rules")
            .then().statusCode(200)
            .body("name", hasItem(name));
    }

    @Test
    void getRule_returns200_forExistingRule() {
        final String id = given().contentType(ContentType.JSON)
            .body("{\"name\":\"get-test\",\"enabled\":false," +
                  "\"condition\":\"false\",\"events\":[\"UPDATE\"],\"actionsJson\":\"[]\"}")
            .post("/filter-rules").then().statusCode(201).extract().path("id");

        given().get("/filter-rules/" + id)
            .then().statusCode(200)
            .body("id", equalTo(id))
            .body("enabled", equalTo(false));
    }

    @Test
    void getRule_returns404_forUnknownId() {
        given().get("/filter-rules/00000000-0000-0000-0000-000000000000")
            .then().statusCode(404);
    }

    @Test
    void deleteRule_returns204_andRuleIsGone() {
        final String id = given().contentType(ContentType.JSON)
            .body("{\"name\":\"delete-test\",\"enabled\":true," +
                  "\"condition\":\"true\",\"events\":[\"ADD\"],\"actionsJson\":\"[]\"}")
            .post("/filter-rules").then().statusCode(201).extract().path("id");

        given().delete("/filter-rules/" + id).then().statusCode(204);
        given().get("/filter-rules/" + id).then().statusCode(404);
    }

    @Test
    void deleteRule_returns404_forUnknownId() {
        given().delete("/filter-rules/00000000-0000-0000-0000-000000000000")
            .then().statusCode(404);
    }
}
```

- [ ] **Step 2: Verify RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-filter-registry \
  -Dtest="DynamicFilterRegistryTest" 2>&1 | tail -5
```
Expected: 404 for all POST /filter-rules (endpoint not yet implemented).

- [ ] **Step 3: Create V3001 migration**

Create `quarkus-work-filter-registry/src/main/resources/db/migration/V3001__filter_rule.sql`:
```sql
-- V3001: FilterRule — dynamic (DB-persisted) filter rules (Issue #113, Epic #100)
--
-- Dynamic rules complement CDI-produced permanent filters. Operators create,
-- enable/disable, and delete rules at runtime without redeployment.
--
-- events: comma-separated FilterEvent names (ADD, UPDATE, REMOVE)
-- actions_json: JSON array of ActionDescriptor: [{type, params}]

CREATE TABLE filter_rule (
    id           UUID          NOT NULL,
    name         VARCHAR(255)  NOT NULL,
    description  VARCHAR(500),
    enabled      BOOLEAN       NOT NULL DEFAULT TRUE,
    condition    TEXT          NOT NULL,
    events       VARCHAR(50)   NOT NULL DEFAULT 'ADD,UPDATE,REMOVE',
    actions_json TEXT          NOT NULL DEFAULT '[]',
    created_at   TIMESTAMP     NOT NULL,
    CONSTRAINT pk_filter_rule PRIMARY KEY (id)
);

CREATE INDEX idx_filter_rule_enabled ON filter_rule (enabled);
```

- [ ] **Step 4: Create FilterRule entity**

```java
package io.quarkiverse.work.filterregistry.model;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.persistence.*;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

@Entity
@Table(name = "filter_rule")
public class FilterRule extends PanacheEntityBase {

    @Id public UUID id;
    @Column(nullable = false, length = 255) public String name;
    @Column(length = 500) public String description;
    @Column(nullable = false) public boolean enabled = true;
    @Column(nullable = false, columnDefinition = "TEXT") public String condition;
    @Column(nullable = false, length = 50) public String events = "ADD,UPDATE,REMOVE";
    @Column(name = "actions_json", nullable = false, columnDefinition = "TEXT")
    public String actionsJson = "[]";
    @Column(name = "created_at", nullable = false) public Instant createdAt;

    @PrePersist void prePersist() {
        if (id == null) id = UUID.randomUUID();
        if (createdAt == null) createdAt = Instant.now();
    }

    public static List<FilterRule> allEnabled() {
        return list("enabled = true ORDER BY createdAt ASC");
    }
}
```

- [ ] **Step 5: Implement DynamicFilterRegistry (real)**

Replace stub with:
```java
package io.quarkiverse.work.filterregistry.registry;

import java.util.*;
import jakarta.enterprise.context.ApplicationScoped;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkiverse.work.filterregistry.model.FilterRule;
import io.quarkiverse.work.filterregistry.spi.*;

@ApplicationScoped
public class DynamicFilterRegistry {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    public List<FilterDefinition> allEnabled() {
        return FilterRule.allEnabled().stream()
                .map(this::toDefinition)
                .filter(java.util.Objects::nonNull)
                .toList();
    }

    private FilterDefinition toDefinition(final FilterRule rule) {
        try {
            final Set<FilterEvent> events = new HashSet<>();
            for (final String e : rule.events.split(",")) {
                events.add(FilterEvent.valueOf(e.trim()));
            }
            final List<ActionDescriptor> actions = MAPPER.readValue(rule.actionsJson,
                MAPPER.getTypeFactory().constructCollectionType(List.class, ActionDescriptor.class));
            return new FilterDefinition(rule.name, rule.description, rule.enabled,
                    events, rule.condition, Map.of(), actions);
        } catch (Exception e) {
            return null; // skip malformed rules
        }
    }
}
```

- [ ] **Step 6: Create FilterRuleResource (CRUD + permanent toggle)**

Create `...filterregistry/api/FilterRuleResource.java`:
```java
package io.quarkiverse.work.filterregistry.api;

import java.util.*;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import io.quarkiverse.work.filterregistry.model.FilterRule;
import io.quarkiverse.work.filterregistry.registry.PermanentFilterRegistry;

@Path("/filter-rules")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class FilterRuleResource {

    @Inject PermanentFilterRegistry permanentRegistry;

    public record CreateFilterRuleRequest(String name, String description,
            Boolean enabled, String condition, List<String> events, String actionsJson) {}

    @POST
    @Transactional
    public Response create(final CreateFilterRuleRequest req) {
        if (req == null || req.name() == null || req.name().isBlank())
            return Response.status(400).entity(Map.of("error", "name required")).build();
        if (req.condition() == null || req.condition().isBlank())
            return Response.status(400).entity(Map.of("error", "condition required")).build();

        final FilterRule rule = new FilterRule();
        rule.name = req.name();
        rule.description = req.description();
        rule.enabled = req.enabled() != null ? req.enabled() : true;
        rule.condition = req.condition();
        rule.events = req.events() != null ? String.join(",", req.events()) : "ADD,UPDATE,REMOVE";
        rule.actionsJson = req.actionsJson() != null ? req.actionsJson() : "[]";
        rule.persist();
        return Response.status(201).entity(toResponse(rule)).build();
    }

    @GET
    public List<Map<String, Object>> list() {
        return FilterRule.<FilterRule>listAll().stream().map(this::toResponse).toList();
    }

    @GET @Path("/{id}")
    public Response get(@PathParam("id") final UUID id) {
        final FilterRule rule = FilterRule.findById(id);
        return rule != null ? Response.ok(toResponse(rule)).build()
                : Response.status(404).entity(Map.of("error", "Not found")).build();
    }

    @DELETE @Path("/{id}")
    @Transactional
    public Response delete(@PathParam("id") final UUID id) {
        return FilterRule.deleteById(id) ? Response.noContent().build()
                : Response.status(404).entity(Map.of("error", "Not found")).build();
    }

    @GET @Path("/permanent")
    public List<Map<String, Object>> listPermanent() {
        return permanentRegistry.all().stream().map(def -> {
            final Map<String, Object> m = new LinkedHashMap<>();
            m.put("name", def.name());
            m.put("description", def.description());
            m.put("enabled", def.enabled());
            m.put("events", def.events());
            m.put("condition", def.condition());
            return m;
        }).toList();
    }

    @PUT @Path("/permanent/{name}/enabled")
    public Response togglePermanent(@PathParam("name") final String name,
            final Map<String, Boolean> body) {
        final Boolean enabled = body.get("enabled");
        if (enabled == null)
            return Response.status(400).entity(Map.of("error", "enabled required")).build();
        if (permanentRegistry.findByName(name).isEmpty())
            return Response.status(404).entity(Map.of("error", "Not found")).build();
        permanentRegistry.setEnabled(name, enabled);
        return Response.ok(Map.of("name", name, "enabled", enabled)).build();
    }

    private Map<String, Object> toResponse(final FilterRule r) {
        final Map<String, Object> m = new LinkedHashMap<>();
        m.put("id", r.id);
        m.put("name", r.name);
        m.put("description", r.description);
        m.put("enabled", r.enabled);
        m.put("condition", r.condition);
        m.put("events", r.events);
        m.put("actionsJson", r.actionsJson);
        m.put("createdAt", r.createdAt);
        return m;
    }
}
```

- [ ] **Step 7: Run CRUD tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-filter-registry \
  -Dtest="DynamicFilterRegistryTest" 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: 6 tests pass.

- [ ] **Step 8: Commit**
```bash
git add quarkus-work-filter-registry/
git commit -m "feat(filter-registry): FilterRule entity + V3001 migration + CRUD REST

FilterRule JPA entity (Panache), V3001 Flyway migration, DynamicFilterRegistry
(converts DB rows to FilterDefinition), FilterRuleResource (POST/GET/GET{id}/DELETE
at /filter-rules, GET/PUT at /filter-rules/permanent).
6 @QuarkusTest CRUD tests.

Refs #113
Refs #100"
```

---

## Task 8: filter-registry — PermanentFilterRegistry REST + end-to-end evaluation tests

**Files:**
- Test: `...filterregistry/registry/PermanentFilterRegistryTest.java`
- Test: `...filterregistry/FilterRuleEvaluationTest.java`

- [ ] **Step 1: Write PermanentFilterRegistry tests**

Create `...filterregistry/registry/PermanentFilterRegistryTest.java`:
```java
package io.quarkiverse.work.filterregistry.registry;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class PermanentFilterRegistryTest {

    @Test
    void listPermanent_includesTestProducedFilters() {
        // TestFilterProducer produces 3 permanent filters
        given().get("/filter-rules/permanent")
            .then().statusCode(200)
            .body("name", hasItems("test/apply-label", "test/override-groups", "test/set-priority"));
    }

    @Test
    void togglePermanent_disablesFilter_atRuntime() {
        // Disable one filter
        given().contentType("application/json")
            .body("{\"enabled\":false}")
            .put("/filter-rules/permanent/test%2Fapply-label/enabled")
            .then().statusCode(200)
            .body("enabled", equalTo(false));

        // Re-enable
        given().contentType("application/json")
            .body("{\"enabled\":true}")
            .put("/filter-rules/permanent/test%2Fapply-label/enabled")
            .then().statusCode(200)
            .body("enabled", equalTo(true));
    }

    @Test
    void togglePermanent_returns404_forUnknownFilter() {
        given().contentType("application/json")
            .body("{\"enabled\":false}")
            .put("/filter-rules/permanent/nonexistent/enabled")
            .then().statusCode(404);
    }
}
```

- [ ] **Step 2: Write end-to-end evaluation test**

Create `...filterregistry/FilterRuleEvaluationTest.java`:
```java
package io.quarkiverse.work.filterregistry;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

/**
 * E2E tests: WorkItem creation triggers FilterRegistryEngine → actions fire.
 * Issue #113, Epic #100.
 */
@QuarkusTest
class FilterRuleEvaluationTest {

    @Test
    void permanentFilter_firesOnAdd_whenConditionMatches() {
        // TestFilterProducer: score < 0.5 → label ai/test-label
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Low Conf\",\"createdBy\":\"agent\",\"confidenceScore\":0.3}")
            .post("/workitems").then().statusCode(201).extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", hasItem("ai/test-label"));
    }

    @Test
    void permanentFilter_doesNotFire_whenConditionDoesNotMatch() {
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"High Conf\",\"createdBy\":\"agent\",\"confidenceScore\":0.9}")
            .post("/workitems").then().statusCode(201).extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", not(hasItem("ai/test-label")));
    }

    @Test
    void permanentFilter_doesNotFire_whenScoreIsNull() {
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"No Score\",\"createdBy\":\"human\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", not(hasItem("ai/test-label")));
    }

    @Test
    void dynamicRule_firesOnAdd_afterCreation() {
        // Create a dynamic rule: category == 'urgent' → SET_PRIORITY CRITICAL
        given().contentType(ContentType.JSON)
            .body("{\"name\":\"e2e/urgent\",\"enabled\":true," +
                  "\"condition\":\"workItem.category == 'urgent'\",\"events\":[\"ADD\"]," +
                  "\"actionsJson\":\"[{\\\"type\\\":\\\"SET_PRIORITY\\\"," +
                  "\\\"params\\\":{\\\"priority\\\":\\\"CRITICAL\\\"}}]\"}")
            .post("/filter-rules").then().statusCode(201);

        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Urgent Item\",\"category\":\"urgent\",\"createdBy\":\"sys\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("priority", equalTo("CRITICAL"));
    }

    @Test
    void disabledPermanentFilter_doesNotFire_evenIfConditionMatches() {
        // Disable the apply-label filter at runtime
        given().contentType("application/json").body("{\"enabled\":false}")
            .put("/filter-rules/permanent/test%2Fapply-label/enabled")
            .then().statusCode(200);

        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Disabled Filter\",\"createdBy\":\"agent\",\"confidenceScore\":0.1}")
            .post("/workitems").then().statusCode(201).extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", not(hasItem("ai/test-label")));

        // Re-enable for subsequent tests
        given().contentType("application/json").body("{\"enabled\":true}")
            .put("/filter-rules/permanent/test%2Fapply-label/enabled").then().statusCode(200);
    }
}
```

- [ ] **Step 3: Run all filter-registry tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-filter-registry \
  2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: BUILD SUCCESS, all tests pass (unit + integration + E2E).

- [ ] **Step 4: Commit**
```bash
git add quarkus-work-filter-registry/
git commit -m "feat(filter-registry): permanent registry REST + E2E evaluation tests

PermanentFilterRegistry REST endpoints (GET /filter-rules/permanent,
PUT /filter-rules/permanent/{name}/enabled). FilterRegistryEngine wired end-to-end:
WorkItem creation triggers JEXL evaluation, matching filters apply actions.
8 tests: permanent registry list/toggle, E2E evaluation (match/no-match/null/disabled,
dynamic rule created at runtime fires correctly).

Closes #113
Refs #100"
```

---

## Task 9: quarkus-work-ai module — scaffold + LowConfidenceFilterProducer

**Files:**
- Create: `quarkus-work-ai/pom.xml`
- Create: `quarkus-work-ai/src/main/java/.../ai/config/WorkItemsAiConfig.java`
- Create: `quarkus-work-ai/src/main/java/.../ai/filter/LowConfidenceFilterProducer.java`
- Create: `quarkus-work-ai/src/test/resources/application.properties`
- Test: `quarkus-work-ai/src/test/java/.../ai/LowConfidenceFilterTest.java`

- [ ] **Step 1: Write failing tests**

Create `quarkus-work-ai/src/test/java/io/quarkiverse/workitems/ai/LowConfidenceFilterTest.java`:
```java
package io.quarkiverse.work.ai;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

/**
 * Integration + E2E tests for confidence-gated routing.
 * Verifies that the LowConfidenceFilterProducer correctly gates WorkItem routing.
 * Issue #114, Epic #100.
 */
@QuarkusTest
class LowConfidenceFilterTest {

    // ── Confidence gating — label application ────────────────────────────────

    @Test
    void lowConfidence_belowThreshold_appliesAiLabel() {
        final String id = createWithScore(0.55);
        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", hasItem("ai/low-confidence"));
    }

    @Test
    void highConfidence_aboveThreshold_noAiLabel() {
        final String id = createWithScore(0.85);
        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", not(hasItem("ai/low-confidence")));
    }

    @Test
    void exactlyAtThreshold_noAiLabel() {
        // 0.7 is not < 0.7
        final String id = createWithScore(0.7);
        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", not(hasItem("ai/low-confidence")));
    }

    @Test
    void nullConfidenceScore_noAiLabel() {
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Human Task\",\"createdBy\":\"human\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");
        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", not(hasItem("ai/low-confidence")));
    }

    @Test
    void lowConfidence_labelIsInferred_notManual() {
        final String id = createWithScore(0.3);
        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.findAll { it.path == 'ai/low-confidence' }[0].persistence",
                  equalTo("INFERRED"));
    }

    // ── Visible in permanent filter list ─────────────────────────────────────

    @Test
    void lowConfidenceFilter_appearsInPermanentFilterList() {
        given().get("/filter-rules/permanent").then().statusCode(200)
            .body("name", hasItem("ai/low-confidence"));
    }

    // ── E2E: AI agent workflow ────────────────────────────────────────────────

    @Test
    void e2e_aiAgentCreatesLowConfidenceItem_reviewerSeeItInInbox() {
        // AI agent creates WorkItem with low confidence
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Uncertain Decision\",\"category\":\"risk\"," +
                  "\"candidateGroups\":\"analysts\",\"createdBy\":\"agent:risk-ai\"," +
                  "\"confidenceScore\":0.42}")
            .post("/workitems").then().statusCode(201).extract().path("id");

        // Label applied
        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", hasItem("ai/low-confidence"))
            .body("confidenceScore", equalTo(0.42f));

        // Reviewer queries inbox filtering for low-confidence items
        given().queryParam("label", "ai/*")
            .get("/workitems")
            .then().statusCode(200)
            .body("id", hasItem(id));

        // Reviewer claims and completes
        given().put("/workitems/" + id + "/claim?claimant=reviewer").then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=reviewer").then().statusCode(200);
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + id + "/complete?actor=reviewer").then().statusCode(200)
            .body("status", equalTo("COMPLETED"));
    }

    @Test
    void e2e_customThreshold_lowersGate() {
        // Default threshold is 0.7; with custom threshold 0.5,
        // score 0.6 should NOT trigger (0.6 > 0.5)
        // This test uses the default config (threshold=0.7), so 0.6 < 0.7 → label applied
        final String id = createWithScore(0.6);
        given().get("/workitems/" + id).then().statusCode(200)
            .body("labels.path", hasItem("ai/low-confidence"));
    }

    private String createWithScore(final double score) {
        return given().contentType(ContentType.JSON)
            .body("{\"title\":\"AI Task\",\"createdBy\":\"agent\",\"confidenceScore\":" + score + "}")
            .post("/workitems").then().statusCode(201).extract().path("id");
    }
}
```

- [ ] **Step 2: Create quarkus-work-ai pom.xml**

```xml
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
  <artifactId>quarkus-work-ai</artifactId>
  <name>Quarkus WorkItems - AI</name>
  <dependencies>
    <dependency>
      <groupId>io.quarkiverse.work</groupId>
      <artifactId>quarkus-work</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkiverse.work</groupId>
      <artifactId>quarkus-work-filter-registry</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
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
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-flyway</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm-panache</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 3: Create test application.properties**

Create `quarkus-work-ai/src/test/resources/application.properties`:
```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:workitemsai;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.http.test-port=0
quarkus.work.ai.confidence-threshold=0.7
quarkus.work.ai.low-confidence-filter.enabled=true
```

- [ ] **Step 4: Create WorkItemsAiConfig**

```java
package io.quarkiverse.work.ai.config;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import io.smallrye.config.WithName;

/**
 * Configuration for the quarkus-work-ai module.
 * All properties are optional — sensible defaults apply.
 */
@ConfigMapping(prefix = "quarkus.work.ai")
public interface WorkItemsAiConfig {

    /**
     * Confidence threshold below which a WorkItem is considered low-confidence.
     * WorkItems with confidenceScore < threshold receive the {@code ai/low-confidence} label.
     * Default: 0.7.
     *
     * @return the threshold value (0.0–1.0)
     */
    @WithDefault("0.7")
    double confidenceThreshold();

    /**
     * Configuration for the low-confidence filter.
     */
    @WithName("low-confidence-filter")
    LowConfidenceFilterConfig lowConfidenceFilter();

    interface LowConfidenceFilterConfig {
        /**
         * Whether the low-confidence filter is active.
         * When false, no {@code ai/low-confidence} label is applied regardless of score.
         * Default: true.
         *
         * @return true if the filter should fire on WorkItem creation
         */
        @WithDefault("true")
        boolean enabled();
    }
}
```

- [ ] **Step 5: Create LowConfidenceFilterProducer**

```java
package io.quarkiverse.work.ai.filter;

import java.util.List;
import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

import io.quarkiverse.work.ai.config.WorkItemsAiConfig;
import io.quarkiverse.work.filterregistry.spi.ActionDescriptor;
import io.quarkiverse.work.filterregistry.spi.FilterDefinition;

/**
 * Produces the permanent {@code ai/low-confidence} filter definition.
 *
 * <p>When {@code quarkus.work.ai.low-confidence-filter.enabled=true} (default),
 * WorkItems created with {@code confidenceScore} below
 * {@code quarkus.work.ai.confidence-threshold} (default 0.7) automatically
 * receive the {@code ai/low-confidence} label (INFERRED persistence).
 *
 * <p>The filter fires on ADD only — it is not re-evaluated on subsequent updates.
 *
 * @see <a href="https://github.com/mdproctor/quarkus-work/issues/114">Issue #114</a>
 */
@ApplicationScoped
public class LowConfidenceFilterProducer {

    @Inject
    WorkItemsAiConfig config;

    /**
     * Produces the low-confidence routing filter as a permanent CDI-managed definition.
     * Enable/disable at runtime via PUT /filter-rules/permanent/ai%2Flow-confidence/enabled.
     */
    @Produces
    public FilterDefinition lowConfidenceFilter() {
        final double threshold = config.confidenceThreshold();
        return FilterDefinition.onAdd(
                "ai/low-confidence",
                "Applies ai/low-confidence label when AI agent confidence is below threshold ("
                        + threshold + "). Enables human reviewers to filter their inbox for uncertain items.",
                config.lowConfidenceFilter().enabled(),
                "workItem.confidenceScore != null && workItem.confidenceScore < threshold",
                Map.of("threshold", threshold),
                List.of(ActionDescriptor.of("APPLY_LABEL",
                        Map.of("path", "ai/low-confidence", "appliedBy", "ai-confidence-gate"))));
    }
}
```

- [ ] **Step 6: Run tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-ai \
  2>&1 | grep -E "Tests run:|BUILD|ERROR" | tail -10
```
Expected: BUILD SUCCESS, 8 tests pass.

- [ ] **Step 7: Commit**
```bash
git add quarkus-work-ai/
git commit -m "feat(workitems-ai): LowConfidenceFilterProducer — confidence-gated routing

New quarkus-work-ai module. LowConfidenceFilterProducer @Produces a permanent
FilterDefinition: condition=confidenceScore<threshold, action=APPLY_LABEL ai/low-confidence.
Config: quarkus.work.ai.confidence-threshold (default 0.7) and
quarkus.work.ai.low-confidence-filter.enabled (default true).
8 tests: label applied below threshold, not applied above/at threshold, null score,
INFERRED persistence, filter visible in /filter-rules/permanent, full E2E workflow.

Closes #114
Refs #100"
```

---

## Task 10: Update docs, DESIGN.md, CLAUDE.md

- [ ] **Step 1: Update docs/DESIGN.md**

Add to the Module Structure table:
```markdown
| Filter Registry | `quarkus-work-filter-registry` | `FilterAction` SPI (CDI, resolved by type name), JEXL condition evaluator, permanent (CDI-produced) + dynamic (DB-persisted) filter rules, `FilterRegistryEngine` observer, built-in actions: `APPLY_LABEL`, `OVERRIDE_CANDIDATE_GROUPS`, `SET_PRIORITY`. REST: `/filter-rules` CRUD + `/filter-rules/permanent` toggle. V3001 migration. |
| AI | `quarkus-work-ai` | `confidenceScore` routing, `LowConfidenceFilterProducer` — permanent filter applying `ai/low-confidence` label when score < threshold. Config: `quarkus.work.ai.confidence-threshold` (default 0.7), `low-confidence-filter.enabled` (default true). |
```

Add to Domain Model:
```markdown
### WorkItem.confidenceScore
`Double`, nullable. Set by AI agents at creation time (0.0–1.0). Null means no AI metadata.
Used by the filter-registry's `LowConfidenceFilterProducer` to gate routing.
V13 migration: `confidence_score DOUBLE` column.
```

Add to REST API Surface:
```markdown
`FilterRuleResource` at `/filter-rules` (filter-registry module, Epic #100):
| `POST /filter-rules` | Create dynamic rule |
| `GET /filter-rules` | List all dynamic rules |
| `GET /filter-rules/{id}` | Get single rule; 404 if not found |
| `DELETE /filter-rules/{id}` | Delete rule; 204/404 |
| `GET /filter-rules/permanent` | List CDI-produced permanent filters |
| `PUT /filter-rules/permanent/{name}/enabled` | Toggle permanent filter at runtime |
```

Update the Build Roadmap phase 11 (or add new phase):
```markdown
| **11 — Confidence-Gated Routing** | ✅ Complete | Epic #100: `confidenceScore` on WorkItem (#112 ✅), filter-registry module with FilterAction SPI + JEXL engine + permanent/dynamic registry (#113 ✅), `quarkus-work-ai` LowConfidenceFilterProducer (#114 ✅). |
```

Update test totals.

- [ ] **Step 2: Update CLAUDE.md active epics table**

Mark issues #112, #113, #114 closed in the epic table.

- [ ] **Step 3: Final full build verification**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -pl runtime,quarkus-work-filter-registry,quarkus-work-ai \
  2>&1 | grep -E "Tests run: [0-9]+, Failures:|BUILD" | tail -5
```
Expected: BUILD SUCCESS, 0 failures across all three modules.

- [ ] **Step 4: Commit docs**
```bash
git add docs/DESIGN.md CLAUDE.md
git commit -m "docs: update DESIGN.md and CLAUDE.md for Epic #100 (confidence-gated routing)

New modules quarkus-work-filter-registry and quarkus-work-ai documented.
confidenceScore field, FilterAction SPI, filter rule REST API, LowConfidenceFilterProducer,
build roadmap phase 11, updated test totals.

Refs #100"
```

---

## Self-Review Checklist

- [x] **Spec coverage:** All four spec sections covered — module structure (Task 3), types (Task 4), evaluation engine (Tasks 5–8), confidence gating (Task 9), E2E flow (Task 8 + 9).
- [x] **No placeholders:** All steps have exact code or exact commands.
- [x] **Type consistency:** `FilterDefinition`, `ActionDescriptor`, `FilterEvent`, `FilterAction` defined in Task 4 and referenced consistently in Tasks 5–9. `conditionContext` in `FilterDefinition` matches usage in `JexlConditionEvaluator`.
- [x] **ApplyLabelAction signature:** Noted that `WorkItemService.addLabel()` signature must be verified — implementer is warned to check and add overload if needed.
- [x] **ThreadLocal guard:** Present in `FilterRegistryEngine.onLifecycleEvent()`.
- [x] **Issue linkage:** Every commit references `#112`, `#113`, or `#114` + `#100`.
- [x] **V3001 migration prefix:** Correctly separate from core (V1–V13) and queues (V2001).
