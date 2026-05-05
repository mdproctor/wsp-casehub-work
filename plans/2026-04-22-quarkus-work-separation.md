# quarkus-work / quarkus-work Separation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract a `quarkus-work-api` (pure-Java SPI) and `quarkus-work-core` (Jandex library with CDI, WorkBroker, filter engine) from the WorkItems mono-repo, leaving `quarkus-work` as a thin human-inbox layer on top.

**Architecture:** `quarkus-work-api` holds all shared contracts (WorkEventType, WorkLifecycleEvent, WorkloadProvider, EscalationPolicy, WorkerSelectionStrategy, WorkerRegistry, WorkerCandidate, SelectionContext, AssignmentDecision, AssignmentTrigger). `quarkus-work-core` holds all generic implementations (WorkBroker, LeastLoadedStrategy, ClaimFirstStrategy, NoOpWorkerRegistry, filter engine). `quarkus-work` implements the SPIs for the human-inbox domain. Two old modules — `quarkus-work-api` and `quarkus-work-filter-registry` — are deleted, their contents absorbed.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, CDI, commons-jexl3, JUnit 5, AssertJ, Rest-Assured, Mockito, H2, Flyway. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn`. All commits reference a GitHub issue + epic.

---

## File Map

### New module: `quarkus-work-api/`
```
quarkus-work-api/pom.xml
quarkus-work-api/src/main/java/io/quarkiverse/work/api/
  WorkEventType.java         (new enum)
  WorkLifecycleEvent.java    (new abstract class)
  WorkloadProvider.java      (new interface)
  EscalationPolicy.java      (moved from runtime/service — signature change)
  WorkerCandidate.java       (moved from quarkus-work-api/spi)
  SelectionContext.java       (moved)
  AssignmentDecision.java    (moved)
  AssignmentTrigger.java     (moved)
  WorkerSelectionStrategy.java (moved)
  WorkerRegistry.java        (moved)
quarkus-work-api/src/test/java/io/quarkiverse/work/api/
  WorkEventTypeTest.java     (new)
  WorkerCandidateTest.java   (moved from quarkus-work-api)
  AssignmentDecisionTest.java (moved)
  SelectionContextTest.java  (moved)
```

### New module: `quarkus-work-core/`
```
quarkus-work-core/pom.xml
quarkus-work-core/src/main/java/io/quarkiverse/work/core/strategy/
  WorkBroker.java            (new — extracted generic orchestration from WorkItemAssignmentService)
  LeastLoadedStrategy.java   (moved from runtime/service)
  ClaimFirstStrategy.java    (moved)
  NoOpWorkerRegistry.java    (moved)
quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/
  FilterAction.java          (moved from filterregistry/spi — signature change: WorkItem→Object)
  FilterDefinition.java      (moved)
  FilterEvent.java           (moved)
  ActionDescriptor.java      (moved)
  FilterRegistryEngine.java  (moved — observer WorkLifecycleEvent, uses event.context())
  JexlConditionEvaluator.java (moved — evaluate() takes Map not WorkItem; toMap() removed)
  PermanentFilterRegistry.java (moved)
  DynamicFilterRegistry.java (moved)
  FilterRule.java            (moved)
  FilterRuleResource.java    (moved)
quarkus-work-core/src/main/resources/db/migration/
  V3001__filter_rules.sql    (moved from quarkus-work-filter-registry)
quarkus-work-core/src/test/java/io/quarkiverse/work/core/strategy/
  WorkBrokerTest.java        (new unit tests)
  LeastLoadedStrategyTest.java (moved + re-packaged)
  ClaimFirstStrategyTest.java  (moved + re-packaged)
quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter/
  JexlConditionEvaluatorTest.java (moved — updated for new Map-based signature)
  FilterRegistryEngineTest.java   (moved — updated for WorkLifecycleEvent)
  DynamicFilterRegistryTest.java  (moved)
  PermanentFilterRegistryTest.java (moved)
  FilterRuleEvaluationTest.java   (moved — @QuarkusTest)
  FilterActionIntegrationTest.java (moved — @QuarkusTest)
```

### Modified: `runtime/` (quarkus-work)
```
event/WorkItemLifecycleEvent.java   record→class extends WorkLifecycleEvent; carries WorkItem
event/WorkItemContextBuilder.java   (new — toMap(WorkItem), moved from JexlConditionEvaluator)
service/JpaWorkloadProvider.java    (new — implements WorkloadProvider via JPA)
service/WorkItemAssignmentService.java  uses WorkBroker + JpaWorkloadProvider
service/EscalationPolicy.java       DELETE (moved to quarkus-work-api)
service/NotifyEscalationPolicy.java escalate(WorkLifecycleEvent) replaces two methods
service/AutoRejectEscalationPolicy.java  same
service/ReassignEscalationPolicy.java    same
service/ExpiryCleanupJob.java       calls escalationPolicy.escalate(event)
service/ClaimDeadlineJob.java       same
action/ApplyLabelAction.java        (moved to runtime — was in filter-registry)
action/OverrideCandidateGroupsAction.java (moved)
action/SetPriorityAction.java       (moved)
runtime/pom.xml                     swap quarkus-work-api→quarkus-work-core
```

### Modified: `quarkus-work-ai/`
```
pom.xml   swap quarkus-work-filter-registry dep → quarkus-work-core
filter/LowConfidenceFilterProducer.java  update imports
```

### Deleted
```
quarkus-work-api/      (entire module)
quarkus-work-filter-registry/  (entire module)
```

### Modified: root `pom.xml`
```
Remove: quarkus-work-api, quarkus-work-filter-registry modules
Add:    quarkus-work-api, quarkus-work-core modules
```

---

## Phase 0 — Issue Tracking

### Task 1: Create GitHub issue

- [ ] **Step 1: Create the issue**

```bash
gh issue create \
  --title "feat: quarkus-work-api + quarkus-work-core separation (WorkBroker + generic filter engine)" \
  --body "$(cat <<'EOF'
## Summary

Extract generic work management contracts and implementations into two new modules:
- **quarkus-work-api** — pure-Java SPI (WorkEventType, WorkLifecycleEvent, WorkloadProvider, EscalationPolicy, all routing SPI types)
- **quarkus-work-core** — Jandex library with WorkBroker, built-in strategies, filter engine

Absorbs: quarkus-work-api (renamed) and quarkus-work-filter-registry (dissolved).
Enables CaseHub to depend on quarkus-work-core for WorkBroker without pulling in human-inbox specifics.

Design spec: docs/superpowers/specs/2026-04-22-quarkus-work-separation-design.md

Refs #100, #102
EOF
)"
```

- [ ] **Step 2: Note the issue number** — all subsequent commits use `Refs #N` where N is the new issue.

---

## Phase 1 — quarkus-work-api Module

### Task 2: Scaffold quarkus-work-api

**Files:**
- Create: `quarkus-work-api/pom.xml`

- [ ] **Step 1: Create the module directory and pom.xml**

```xml
<!-- quarkus-work-api/pom.xml -->
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
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work-api</artifactId>
  <name>Quarkus Work - API (Shared SPI)</name>
  <description>Pure Java shared SPI contracts for work management. Zero runtime dependencies. CaseHub and quarkus-work depend on this without pulling in implementation details.</description>
  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

- [ ] **Step 2: Add to root pom.xml modules — add quarkus-work-api BEFORE quarkus-work-api**

In `pom.xml` `<modules>` section, add:
```xml
<module>quarkus-work-api</module>
```
Leave `quarkus-work-api` in place for now — it will be deleted later.

- [ ] **Step 3: Create source directories**

```bash
mkdir -p quarkus-work-api/src/main/java/io/quarkiverse/work/api
mkdir -p quarkus-work-api/src/test/java/io/quarkiverse/work/api
```

- [ ] **Step 4: Verify module compiles (empty)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl quarkus-work-api
```
Expected: BUILD SUCCESS

---

### Task 3: Move SPI routing types into quarkus-work-api

**Files:**
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkerCandidate.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/SelectionContext.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/AssignmentDecision.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/AssignmentTrigger.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkerSelectionStrategy.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkerRegistry.java`
- Create (test): `quarkus-work-api/src/test/java/io/quarkiverse/work/api/WorkerCandidateTest.java`
- Create (test): `quarkus-work-api/src/test/java/io/quarkiverse/work/api/AssignmentDecisionTest.java`
- Create (test): `quarkus-work-api/src/test/java/io/quarkiverse/work/api/SelectionContextTest.java`

- [ ] **Step 1: Write the failing tests first** (they fail to compile until code exists)

`quarkus-work-api/src/test/java/io/quarkiverse/work/api/WorkerCandidateTest.java`:
```java
package io.quarkiverse.work.api;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.Set;
import org.junit.jupiter.api.Test;

class WorkerCandidateTest {
    @Test void of_createsWithEmptyCapabilitiesAndZeroCount() {
        var c = WorkerCandidate.of("alice");
        assertThat(c.id()).isEqualTo("alice");
        assertThat(c.capabilities()).isEmpty();
        assertThat(c.activeWorkItemCount()).isZero();
    }
    @Test void withActiveWorkItemCount_returnsNewInstance() {
        var c = WorkerCandidate.of("bob").withActiveWorkItemCount(3);
        assertThat(c.activeWorkItemCount()).isEqualTo(3);
        assertThat(c.id()).isEqualTo("bob");
    }
}
```

`quarkus-work-api/src/test/java/io/quarkiverse/work/api/AssignmentDecisionTest.java`:
```java
package io.quarkiverse.work.api;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class AssignmentDecisionTest {
    @Test void noChange_allFieldsNull() {
        var d = AssignmentDecision.noChange();
        assertThat(d.assigneeId()).isNull();
        assertThat(d.candidateGroups()).isNull();
        assertThat(d.candidateUsers()).isNull();
        assertThat(d.isNoOp()).isTrue();
    }
    @Test void assignTo_setsAssigneeOnly() {
        var d = AssignmentDecision.assignTo("alice");
        assertThat(d.assigneeId()).isEqualTo("alice");
        assertThat(d.isNoOp()).isFalse();
    }
    @Test void narrowCandidates_setsGroupsAndUsers() {
        var d = AssignmentDecision.narrowCandidates("finance", "alice,bob");
        assertThat(d.candidateGroups()).isEqualTo("finance");
        assertThat(d.candidateUsers()).isEqualTo("alice,bob");
        assertThat(d.assigneeId()).isNull();
    }
}
```

`quarkus-work-api/src/test/java/io/quarkiverse/work/api/SelectionContextTest.java`:
```java
package io.quarkiverse.work.api;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class SelectionContextTest {
    @Test void record_storesAllFields() {
        var ctx = new SelectionContext("finance", "HIGH", "approval", "finance-team", "alice");
        assertThat(ctx.category()).isEqualTo("finance");
        assertThat(ctx.priority()).isEqualTo("HIGH");
        assertThat(ctx.requiredCapabilities()).isEqualTo("approval");
        assertThat(ctx.candidateGroups()).isEqualTo("finance-team");
        assertThat(ctx.candidateUsers()).isEqualTo("alice");
    }
    @Test void record_allowsNullFields() {
        var ctx = new SelectionContext(null, null, null, null, null);
        assertThat(ctx.category()).isNull();
    }
}
```

- [ ] **Step 2: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api 2>&1 | tail -20
```
Expected: compilation error (types not yet defined).

- [ ] **Step 3: Copy SPI types from quarkus-work-api, update package to io.quarkiverse.work.api**

Copy each file verbatim, changing only the `package` declaration:
- `quarkus-work-api/.../spi/WorkerCandidate.java` → `quarkus-work-api/.../api/WorkerCandidate.java` (package `io.quarkiverse.work.api`)
- `SelectionContext.java` — same pattern
- `AssignmentDecision.java` — same
- `AssignmentTrigger.java` — same
- `WorkerSelectionStrategy.java` — same (update import of `AssignmentDecision`, `AssignmentTrigger`, `SelectionContext`, `WorkerCandidate` to `io.quarkiverse.work.api.*`)
- `WorkerRegistry.java` — same (update import of `WorkerCandidate`)

- [ ] **Step 4: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api
```
Expected: 5 tests pass, BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add quarkus-work-api/ pom.xml
git commit -m "$(cat <<'EOF'
feat: scaffold quarkus-work-api, move routing SPI types

WorkerCandidate, SelectionContext, AssignmentDecision, AssignmentTrigger,
WorkerSelectionStrategy, WorkerRegistry moved to io.quarkiverse.work.api.
quarkus-work-api kept in place temporarily pending full cutover.

Refs #ISSUE, #100, #102
EOF
)"
```

---

### Task 4: Add WorkEventType and WorkLifecycleEvent to quarkus-work-api

**Files:**
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkEventType.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkLifecycleEvent.java`
- Create (test): `quarkus-work-api/src/test/java/io/quarkiverse/work/api/WorkEventTypeTest.java`

- [ ] **Step 1: Write failing test**

`quarkus-work-api/src/test/java/io/quarkiverse/work/api/WorkEventTypeTest.java`:
```java
package io.quarkiverse.work.api;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.Map;
import org.junit.jupiter.api.Test;

class WorkEventTypeTest {
    @Test void allExpectedValuesExist() {
        assertThat(WorkEventType.values()).extracting(Enum::name).containsExactlyInAnyOrder(
            "CREATED", "ASSIGNED", "STARTED", "COMPLETED", "REJECTED",
            "DELEGATED", "RELEASED", "SUSPENDED", "RESUMED",
            "CANCELLED", "EXPIRED", "CLAIM_EXPIRED", "ESCALATED");
    }

    @Test void concreteEvent_implementsAbstractMethods() {
        // Verify WorkLifecycleEvent can be subclassed and methods called
        var event = new WorkLifecycleEvent() {
            @Override public WorkEventType eventType() { return WorkEventType.CREATED; }
            @Override public Map<String, Object> context() { return Map.of("id", "x"); }
            @Override public Object source() { return "test-source"; }
        };
        assertThat(event.eventType()).isEqualTo(WorkEventType.CREATED);
        assertThat(event.context()).containsKey("id");
        assertThat(event.source()).isEqualTo("test-source");
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api 2>&1 | tail -10
```

- [ ] **Step 3: Implement WorkEventType**

`quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkEventType.java`:
```java
package io.quarkiverse.work.api;

/**
 * Canonical lifecycle event vocabulary shared across all work-management systems.
 * WorkItems, CaseHub tasks, and future work-unit types all map to these values.
 */
public enum WorkEventType {
    CREATED,
    ASSIGNED,
    STARTED,
    COMPLETED,
    REJECTED,
    DELEGATED,
    RELEASED,
    SUSPENDED,
    RESUMED,
    CANCELLED,
    EXPIRED,
    /** Claim deadline passed without the work being claimed. */
    CLAIM_EXPIRED,
    ESCALATED
}
```

- [ ] **Step 4: Implement WorkLifecycleEvent**

`quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkLifecycleEvent.java`:
```java
package io.quarkiverse.work.api;

import java.util.Map;

/**
 * Abstract base for CDI lifecycle events fired when a work unit transitions state.
 *
 * <p>Observers declared as {@code @Observes WorkLifecycleEvent} receive any subtype,
 * including {@code WorkItemLifecycleEvent} from quarkus-work and future event
 * types from CaseHub.
 *
 * <p>Implementations carry the underlying work unit so observers can act on it without
 * an additional store lookup. The {@link #context()} map is pre-built by the work-unit
 * domain and used directly by the filter engine for JEXL condition evaluation.
 */
public abstract class WorkLifecycleEvent {

    /**
     * The canonical event type — what transition just occurred.
     */
    public abstract WorkEventType eventType();

    /**
     * A flat {@code Map<String, Object>} representing the work unit's current field values.
     * Used by the filter engine for JEXL condition evaluation.
     * Values must be directly accessible to JEXL (no JPMS-restricted reflection).
     */
    public abstract Map<String, Object> context();

    /**
     * The underlying work unit (e.g. {@code WorkItem} in quarkus-work).
     * {@code FilterAction} implementations downcast to the concrete type they expect.
     * Never null.
     */
    public abstract Object source();
}
```

- [ ] **Step 5: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api
```
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add quarkus-work-api/
git commit -m "$(cat <<'EOF'
feat(work-api): add WorkEventType enum and WorkLifecycleEvent abstract class

WorkEventType: canonical event vocabulary (CREATED..ESCALATED + CLAIM_EXPIRED).
WorkLifecycleEvent: abstract CDI event base; subclasses carry the work unit
and pre-built context map — filter engine needs no store lookup.

Refs #ISSUE, #100
EOF
)"
```

---

### Task 5: Add WorkloadProvider and EscalationPolicy to quarkus-work-api

**Files:**
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkloadProvider.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/work/api/EscalationPolicy.java`

- [ ] **Step 1: Write failing tests**

`quarkus-work-api/src/test/java/io/quarkiverse/work/api/WorkloadProviderTest.java`:
```java
package io.quarkiverse.work.api;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class WorkloadProviderTest {
    @Test void canImplementWithLambda() {
        WorkloadProvider p = workerId -> workerId.equals("alice") ? 3 : 0;
        assertThat(p.getActiveWorkCount("alice")).isEqualTo(3);
        assertThat(p.getActiveWorkCount("bob")).isZero();
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api 2>&1 | tail -10
```

- [ ] **Step 3: Implement WorkloadProvider**

`quarkus-work-api/src/main/java/io/quarkiverse/work/api/WorkloadProvider.java`:
```java
package io.quarkiverse.work.api;

/**
 * SPI for querying the active work count for a worker.
 *
 * <p>Implement as {@code @ApplicationScoped @Alternative @Priority(1)} to provide
 * domain-specific workload data. quarkus-work provides {@code JpaWorkloadProvider}.
 * CaseHub provides its own implementation against its task store.
 *
 * <p>Used by {@code WorkItemAssignmentService} to populate
 * {@link WorkerCandidate#activeWorkItemCount()} before passing candidates to
 * {@link WorkerSelectionStrategy#select}.
 */
@FunctionalInterface
public interface WorkloadProvider {

    /**
     * Returns the count of active (non-terminal) work units held by the given worker.
     *
     * @param workerId the worker identifier
     * @return active work count; 0 if unknown or no active items
     */
    int getActiveWorkCount(String workerId);
}
```

- [ ] **Step 4: Implement EscalationPolicy**

`quarkus-work-api/src/main/java/io/quarkiverse/work/api/EscalationPolicy.java`:
```java
package io.quarkiverse.work.api;

/**
 * SPI for escalation behaviour when work stalls past a deadline.
 *
 * <p>Implementations in quarkus-work cast {@code event.source()} to {@code WorkItem}
 * and can inspect {@code event.eventType()} to distinguish between
 * {@link WorkEventType#EXPIRED} (completion deadline missed) and
 * {@link WorkEventType#CLAIM_EXPIRED} (claim deadline missed without assignment).
 *
 * <p>Implementations are qualified with {@code @ExpiryEscalation} or
 * {@code @ClaimEscalation} in quarkus-work to distinguish the two injection points.
 */
public interface EscalationPolicy {

    /**
     * React to a stalled work unit.
     *
     * @param event the lifecycle event that triggered escalation; never null.
     *              Use {@code event.eventType()} to determine the escalation reason,
     *              {@code event.source()} to access the concrete work unit.
     */
    void escalate(WorkLifecycleEvent event);
}
```

- [ ] **Step 5: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api
```

- [ ] **Step 6: Commit**

```bash
git add quarkus-work-api/
git commit -m "$(cat <<'EOF'
feat(work-api): add WorkloadProvider and EscalationPolicy SPIs

WorkloadProvider: @FunctionalInterface for active work count per worker.
EscalationPolicy: generic escalate(WorkLifecycleEvent) replacing the two-method
WorkItems-specific interface (onExpired/onUnclaimedPastDeadline).

Refs #ISSUE, #100, #101
EOF
)"
```

---

## Phase 2 — quarkus-work-core Module

### Task 6: Scaffold quarkus-work-core

**Files:**
- Create: `quarkus-work-core/pom.xml`
- Create: `quarkus-work-core/src/main/resources/db/migration/V3001__filter_rules.sql`

- [ ] **Step 1: Create pom.xml**

```xml
<!-- quarkus-work-core/pom.xml -->
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
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work-core</artifactId>
  <name>Quarkus Work - Core (WorkBroker + Filter Engine)</name>
  <description>Generic work management implementations: WorkBroker, built-in WorkerSelectionStrategies, and the reactive filter engine. Jandex-indexed library — no deployment module needed.</description>
  <dependencies>
    <dependency>
      <groupId>io.quarkiverse.work</groupId>
      <artifactId>quarkus-work-api</artifactId>
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
      <groupId>org.mockito</groupId>
      <artifactId>mockito-junit-jupiter</artifactId>
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
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Add quarkus-work-core to root pom.xml modules** (after quarkus-work-api, before runtime):

```xml
<module>quarkus-work-core</module>
```

- [ ] **Step 3: Create directories and move V3001 migration**

```bash
mkdir -p quarkus-work-core/src/main/java/io/quarkiverse/work/core/strategy
mkdir -p quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter
mkdir -p quarkus-work-core/src/main/resources/db/migration
mkdir -p quarkus-work-core/src/test/java/io/quarkiverse/work/core/strategy
mkdir -p quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter
mkdir -p quarkus-work-core/src/test/resources
```

Copy `quarkus-work-filter-registry/src/main/resources/db/migration/V3001__filter_rules.sql`
to `quarkus-work-core/src/main/resources/db/migration/V3001__filter_rules.sql` (verbatim — no changes to SQL).

- [ ] **Step 4: Verify scaffold compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl quarkus-work-core
```
Expected: BUILD SUCCESS (empty source).

- [ ] **Step 5: Commit**

```bash
git add quarkus-work-core/ pom.xml
git commit -m "$(cat <<'EOF'
feat: scaffold quarkus-work-core module

Jandex library with Quarkus CDI/Panache/REST deps and quarkus-work-api.
V3001 filter_rules migration moved from quarkus-work-filter-registry.

Refs #ISSUE, #100
EOF
)"
```

---

### Task 7: Add WorkBroker to quarkus-work-core

**Files:**
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/strategy/WorkBroker.java`
- Create (test): `quarkus-work-core/src/test/java/io/quarkiverse/work/core/strategy/WorkBrokerTest.java`

- [ ] **Step 1: Write failing unit test**

`quarkus-work-core/src/test/java/io/quarkiverse/work/core/strategy/WorkBrokerTest.java`:
```java
package io.quarkiverse.work.core.strategy;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import io.quarkiverse.work.api.*;

class WorkBrokerTest {

    private WorkBroker broker;

    @BeforeEach void setUp() { broker = new WorkBroker(); }

    private SelectionContext ctx(String caps) {
        return new SelectionContext("finance", "HIGH", caps, null, "alice,bob");
    }

    @Test void skipsWhenTriggerNotInStrategyTriggers() {
        WorkerSelectionStrategy onlyCreated = new WorkerSelectionStrategy() {
            @Override public AssignmentDecision select(SelectionContext c, List<WorkerCandidate> cands) {
                return AssignmentDecision.assignTo("alice");
            }
            @Override public Set<AssignmentTrigger> triggers() {
                return Set.of(AssignmentTrigger.CREATED);
            }
        };
        var cands = List.of(WorkerCandidate.of("alice"), WorkerCandidate.of("bob"));
        var result = broker.apply(ctx(null), AssignmentTrigger.RELEASED, cands, onlyCreated);
        assertThat(result.isNoOp()).isTrue();
    }

    @Test void filtersOutCandidatesLackingRequiredCapabilities() {
        WorkerSelectionStrategy strategy = (c, cands) -> AssignmentDecision.assignTo(cands.get(0).id());
        var alice = new WorkerCandidate("alice", Set.of("approval", "legal"), 0);
        var bob   = new WorkerCandidate("bob",   Set.of("approval"), 0);
        // require both approval + legal — only alice qualifies
        var result = broker.apply(ctx("approval,legal"), AssignmentTrigger.CREATED, List.of(alice, bob), strategy);
        assertThat(result.assigneeId()).isEqualTo("alice");
    }

    @Test void noCapabilitiesFilter_passesAllCandidates() {
        WorkerSelectionStrategy strategy = (c, cands) -> {
            assertThat(cands).hasSize(2);
            return AssignmentDecision.noChange();
        };
        var cands = List.of(WorkerCandidate.of("alice"), WorkerCandidate.of("bob"));
        broker.apply(ctx(null), AssignmentTrigger.CREATED, cands, strategy);
    }

    @Test void emptyCandidatesAfterFilter_strategyReceivesEmptyList() {
        WorkerSelectionStrategy strategy = (c, cands) -> {
            assertThat(cands).isEmpty();
            return AssignmentDecision.noChange();
        };
        var alice = new WorkerCandidate("alice", Set.of("approval"), 0); // missing "legal"
        broker.apply(ctx("approval,legal"), AssignmentTrigger.CREATED, List.of(alice), strategy);
    }

    @Test void blankCapabilities_treatedAsNoFilter() {
        WorkerSelectionStrategy strategy = (c, cands) -> {
            assertThat(cands).hasSize(1);
            return AssignmentDecision.noChange();
        };
        broker.apply(ctx("  "), AssignmentTrigger.CREATED, List.of(WorkerCandidate.of("alice")), strategy);
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core 2>&1 | tail -10
```

- [ ] **Step 3: Implement WorkBroker**

`quarkus-work-core/src/main/java/io/quarkiverse/work/core/strategy/WorkBroker.java`:
```java
package io.quarkiverse.work.core.strategy;

import java.util.Arrays;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkiverse.work.api.AssignmentDecision;
import io.quarkiverse.work.api.AssignmentTrigger;
import io.quarkiverse.work.api.SelectionContext;
import io.quarkiverse.work.api.WorkerCandidate;
import io.quarkiverse.work.api.WorkerSelectionStrategy;

/**
 * Generic work assignment broker.
 *
 * <p>Applies trigger gating, capability filtering, and strategy dispatch.
 * Does not know about any specific work-unit type — all domain-specific
 * logic (candidate resolution, workload counting, decision application)
 * belongs in the caller (e.g. {@code WorkItemAssignmentService}).
 *
 * <p>CaseHub replaces its {@code TaskBroker} by delegating to this bean.
 */
@ApplicationScoped
public class WorkBroker {

    /**
     * Apply the strategy to a pre-resolved candidate list.
     *
     * @param context        the work unit's routing context
     * @param trigger        which lifecycle event triggered this call
     * @param candidates     resolved candidates with pre-populated activeWorkItemCount
     * @param strategy       the active selection strategy
     * @return the assignment decision; never null
     */
    public AssignmentDecision apply(
            final SelectionContext context,
            final AssignmentTrigger trigger,
            final List<WorkerCandidate> candidates,
            final WorkerSelectionStrategy strategy) {

        if (!strategy.triggers().contains(trigger)) {
            return AssignmentDecision.noChange();
        }
        final List<WorkerCandidate> filtered = filterByCapabilities(context, candidates);
        return strategy.select(context, filtered);
    }

    private List<WorkerCandidate> filterByCapabilities(
            final SelectionContext context, final List<WorkerCandidate> candidates) {
        if (context.requiredCapabilities() == null || context.requiredCapabilities().isBlank()) {
            return candidates;
        }
        final Set<String> required = Arrays.stream(context.requiredCapabilities().split(","))
                .map(String::trim).filter(s -> !s.isEmpty())
                .collect(Collectors.toSet());
        return candidates.stream()
                .filter(c -> c.capabilities().containsAll(required))
                .toList();
    }
}
```

- [ ] **Step 4: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core
```
Expected: 5 WorkBrokerTest tests pass.

- [ ] **Step 5: Commit**

```bash
git add quarkus-work-core/
git commit -m "$(cat <<'EOF'
feat(work-core): add WorkBroker — generic assignment orchestrator

Handles trigger gating and requiredCapabilities filtering, then delegates
to the active WorkerSelectionStrategy. Replaces CaseHub TaskBroker concept.
No domain-specific dependencies — pure work.api types only.

Refs #ISSUE, #100, #102
EOF
)"
```

---

### Task 8: Move strategies and NoOpWorkerRegistry to quarkus-work-core

**Files:**
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/strategy/LeastLoadedStrategy.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/strategy/ClaimFirstStrategy.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/strategy/NoOpWorkerRegistry.java`
- Create (test): `quarkus-work-core/src/test/java/io/quarkiverse/work/core/strategy/LeastLoadedStrategyTest.java`
- Create (test): `quarkus-work-core/src/test/java/io/quarkiverse/work/core/strategy/ClaimFirstStrategyTest.java`

- [ ] **Step 1: Write failing tests**

`quarkus-work-core/src/test/java/io/quarkiverse/work/core/strategy/LeastLoadedStrategyTest.java`:
```java
package io.quarkiverse.work.core.strategy;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.Test;
import io.quarkiverse.work.api.*;

class LeastLoadedStrategyTest {
    private final LeastLoadedStrategy strategy = new LeastLoadedStrategy();
    private final SelectionContext ctx = new SelectionContext(null, null, null, null, null);

    @Test void selectsCandidateWithFewestActiveItems() {
        var alice = new WorkerCandidate("alice", Set.of(), 5);
        var bob   = new WorkerCandidate("bob",   Set.of(), 1);
        var result = strategy.select(ctx, List.of(alice, bob));
        assertThat(result.assigneeId()).isEqualTo("bob");
    }

    @Test void emptyCandidates_returnsNoChange() {
        assertThat(strategy.select(ctx, List.of()).isNoOp()).isTrue();
    }

    @Test void singleCandidate_selected() {
        var result = strategy.select(ctx, List.of(WorkerCandidate.of("alice")));
        assertThat(result.assigneeId()).isEqualTo("alice");
    }

    @Test void tiedCandidates_picksFirstEncountered() {
        var alice = new WorkerCandidate("alice", Set.of(), 2);
        var bob   = new WorkerCandidate("bob",   Set.of(), 2);
        var result = strategy.select(ctx, List.of(alice, bob));
        assertThat(result.assigneeId()).isIn("alice", "bob");
    }

    @Test void triggersDefaultsToAllThree() {
        assertThat(strategy.triggers()).containsExactlyInAnyOrder(AssignmentTrigger.values());
    }
}
```

`quarkus-work-core/src/test/java/io/quarkiverse/work/core/strategy/ClaimFirstStrategyTest.java`:
```java
package io.quarkiverse.work.core.strategy;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.List;
import org.junit.jupiter.api.Test;
import io.quarkiverse.work.api.*;

class ClaimFirstStrategyTest {
    private final ClaimFirstStrategy strategy = new ClaimFirstStrategy();
    private final SelectionContext ctx = new SelectionContext(null, null, null, null, null);

    @Test void alwaysReturnsNoChange() {
        var result = strategy.select(ctx, List.of(WorkerCandidate.of("alice")));
        assertThat(result.isNoOp()).isTrue();
    }

    @Test void emptyList_returnsNoChange() {
        assertThat(strategy.select(ctx, List.of()).isNoOp()).isTrue();
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core 2>&1 | tail -10
```

- [ ] **Step 3: Copy LeastLoadedStrategy and ClaimFirstStrategy from runtime, update package and imports**

Copy `runtime/.../service/LeastLoadedStrategy.java` → `quarkus-work-core/.../strategy/LeastLoadedStrategy.java`:
- Package: `io.quarkiverse.work.core.strategy`
- Imports: `io.quarkiverse.work.api.*` (instead of `io.quarkiverse.work.spi.*`)

Copy `runtime/.../service/ClaimFirstStrategy.java` → same pattern.

Copy `runtime/.../service/NoOpWorkerRegistry.java` → `quarkus-work-core/.../strategy/NoOpWorkerRegistry.java`:
```java
package io.quarkiverse.work.core.strategy;

import java.util.List;
import jakarta.enterprise.context.ApplicationScoped;
import io.quarkiverse.work.api.WorkerCandidate;
import io.quarkiverse.work.api.WorkerRegistry;

/**
 * Default WorkerRegistry — returns empty list for all groups.
 * Groups remain claim-first until the application registers a real resolver.
 */
@ApplicationScoped
public class NoOpWorkerRegistry implements WorkerRegistry {
    @Override
    public List<WorkerCandidate> resolveGroup(final String groupName) {
        return List.of();
    }
}
```

- [ ] **Step 4: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core
```
Expected: 12+ tests pass.

- [ ] **Step 5: Commit**

```bash
git add quarkus-work-core/
git commit -m "$(cat <<'EOF'
feat(work-core): move routing strategies and NoOpWorkerRegistry

LeastLoadedStrategy, ClaimFirstStrategy, NoOpWorkerRegistry moved from
quarkus-work runtime to io.quarkiverse.work.core.strategy.
Copies in runtime retained temporarily — removed in Phase 5.

Refs #ISSUE, #100, #102
EOF
)"
```

---

### Task 9: Move filter SPI types to quarkus-work-core

**Files:**
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/FilterEvent.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/ActionDescriptor.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/FilterDefinition.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/FilterAction.java`

**Note on signature change:** `FilterAction.apply(WorkItem, Map)` becomes `FilterAction.apply(Object workUnit, Map)` — `workUnit` is the opaque domain object; implementations cast to the concrete type.

- [ ] **Step 1: Copy FilterEvent, ActionDescriptor, FilterDefinition verbatim** — update package to `io.quarkiverse.work.core.filter`, update cross-imports within the package.

FilterEvent has no deps — copy verbatim.
ActionDescriptor has no deps on WorkItem — copy verbatim.
FilterDefinition has no deps on WorkItem — copy verbatim (update import of FilterEvent, ActionDescriptor).

- [ ] **Step 2: Write failing test for FilterAction signature change**

`quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter/FilterActionTest.java`:
```java
package io.quarkiverse.work.core.filter;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class FilterActionTest {
    @Test void apply_receivesOpaqueWorkUnit() {
        final Object[] captured = new Object[1];
        FilterAction action = new FilterAction() {
            @Override public String type() { return "TEST"; }
            @Override public void apply(Object workUnit, Map<String, Object> params) {
                captured[0] = workUnit;
            }
        };
        Object domain = new Object();
        action.apply(domain, Map.of());
        assertThat(captured[0]).isSameAs(domain);
    }
}
```

- [ ] **Step 3: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core 2>&1 | tail -10
```

- [ ] **Step 4: Implement FilterAction with new signature**

`quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/FilterAction.java`:
```java
package io.quarkiverse.work.core.filter;

import java.util.Map;

/**
 * SPI for actions that a filter rule can apply to a work unit when its condition matches.
 *
 * <p>Implementations must be {@code @ApplicationScoped} CDI beans. The engine resolves
 * them by matching {@link ActionDescriptor#type()} to {@link #type()}.
 *
 * <p>The {@code workUnit} parameter is the domain-specific work object (e.g. {@code WorkItem}
 * in quarkus-work). Implementations cast to their expected concrete type.
 *
 * <p>Built-in implementations (in quarkus-work): {@code APPLY_LABEL},
 * {@code OVERRIDE_CANDIDATE_GROUPS}, {@code SET_PRIORITY}.
 */
public interface FilterAction {

    /** The action type name used in {@link ActionDescriptor#type()}. Must be unique. */
    String type();

    /**
     * Apply this action to the given work unit.
     *
     * @param workUnit the domain work object (cast to concrete type expected by this action)
     * @param params   action-specific parameters from the {@link ActionDescriptor}
     */
    void apply(Object workUnit, Map<String, Object> params);
}
```

- [ ] **Step 5: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core
```

- [ ] **Step 6: Commit**

```bash
git add quarkus-work-core/
git commit -m "$(cat <<'EOF'
feat(work-core): move filter SPI types (FilterEvent, ActionDescriptor, FilterDefinition, FilterAction)

FilterAction.apply() signature changes from (WorkItem, Map) to (Object workUnit, Map)
to break the dependency on the WorkItem entity. Implementations in quarkus-work
downcast workUnit to WorkItem.

Refs #ISSUE, #100
EOF
)"
```

---

### Task 10: Move JexlConditionEvaluator and filter registries to quarkus-work-core

**Files:**
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/JexlConditionEvaluator.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/PermanentFilterRegistry.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/DynamicFilterRegistry.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/FilterRule.java`
- Create (test): `quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter/JexlConditionEvaluatorTest.java`
- Create (test): `quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter/PermanentFilterRegistryTest.java`
- Create (test): `quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter/DynamicFilterRegistryTest.java`

**Note on JexlConditionEvaluator:** The `toMap(WorkItem)` method is removed. The `evaluate()` method changes from `(String, Map, WorkItem)` to `(String, Map, Map<String,Object>)` — the third parameter is the pre-built context map (previously built by `toMap()`; now provided by `WorkLifecycleEvent.context()`).

- [ ] **Step 1: Write failing test for new JexlConditionEvaluator signature**

`quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter/JexlConditionEvaluatorTest.java`:
```java
package io.quarkiverse.work.core.filter;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.Map;
import org.junit.jupiter.api.Test;

class JexlConditionEvaluatorTest {
    private final JexlConditionEvaluator eval = new JexlConditionEvaluator();

    @Test void evaluate_trueWhenConditionMatches() {
        var ctx = Map.<String, Object>of("category", "finance", "priority", "HIGH");
        assertThat(eval.evaluate("workItem.category == 'finance'", Map.of(), ctx)).isTrue();
    }

    @Test void evaluate_falseWhenConditionDoesNotMatch() {
        var ctx = Map.<String, Object>of("category", "legal");
        assertThat(eval.evaluate("workItem.category == 'finance'", Map.of(), ctx)).isFalse();
    }

    @Test void evaluate_conditionContextVariablesMerged() {
        var ctx = Map.<String, Object>of("score", 0.5);
        var condCtx = Map.<String, Object>of("threshold", 0.7);
        assertThat(eval.evaluate("workItem.score < threshold", condCtx, ctx)).isTrue();
    }

    @Test void evaluate_blankCondition_returnsFalse() {
        assertThat(eval.evaluate("", Map.of(), Map.of())).isFalse();
        assertThat(eval.evaluate(null, Map.of(), Map.of())).isFalse();
    }

    @Test void evaluate_invalidJexl_returnsFalse() {
        assertThat(eval.evaluate("!!!invalid!!!", Map.of(), Map.of())).isFalse();
    }

    @Test void evaluate_conditionContextOverridesWorkItem() {
        // threshold in conditionContext takes precedence over same key in workUnitContext
        var ctx = Map.<String, Object>of("threshold", 0.9);
        var condCtx = Map.<String, Object>of("threshold", 0.5);
        assertThat(eval.evaluate("workItem.threshold == 0.5", condCtx, ctx)).isTrue();
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core 2>&1 | tail -10
```

- [ ] **Step 3: Implement JexlConditionEvaluator (without toMap)**

`quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/JexlConditionEvaluator.java`:
```java
package io.quarkiverse.work.core.filter;

import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;

import org.apache.commons.jexl3.JexlBuilder;
import org.apache.commons.jexl3.JexlEngine;
import org.apache.commons.jexl3.JexlException;
import org.apache.commons.jexl3.MapContext;

/**
 * Evaluates JEXL conditions against a pre-built work unit context map.
 *
 * <p>The work unit is exposed as {@code workItem} in the JEXL context (key kept for
 * backward compatibility with existing filter rule expressions). Additional variables
 * from {@code conditionContext} are merged alongside.
 *
 * <p>The caller is responsible for building the context map. In quarkus-work,
 * {@code WorkItemContextBuilder.toMap(WorkItem)} provides it via
 * {@code WorkItemLifecycleEvent.context()}.
 */
@ApplicationScoped
public class JexlConditionEvaluator {

    private static final JexlEngine JEXL = new JexlBuilder()
            .strict(false).silent(true).create();

    /**
     * Evaluates a JEXL expression.
     *
     * @param condition       JEXL expression; blank/null → false
     * @param conditionContext additional variables merged into the JEXL context; may be null
     * @param workUnitContext  the work unit's field map, exposed as {@code workItem} in JEXL
     * @return true iff expression evaluates to Boolean.TRUE
     */
    public boolean evaluate(final String condition,
            final Map<String, Object> conditionContext,
            final Map<String, Object> workUnitContext) {
        if (condition == null || condition.isBlank()) {
            return false;
        }
        try {
            final var ctx = new MapContext();
            ctx.set("workItem", workUnitContext);
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

- [ ] **Step 4: Move PermanentFilterRegistry, DynamicFilterRegistry, FilterRule**

Copy from `quarkus-work-filter-registry`:
- `filterregistry/registry/PermanentFilterRegistry.java` → `io.quarkiverse.work.core.filter.PermanentFilterRegistry` — update package + imports (FilterDefinition now from `io.quarkiverse.work.core.filter`)
- `filterregistry/registry/DynamicFilterRegistry.java` → same pattern
- `filterregistry/model/FilterRule.java` → `io.quarkiverse.work.core.filter.FilterRule` — update package

Move `PermanentFilterRegistryTest.java` and `DynamicFilterRegistryTest.java` to `quarkus-work-core/src/test/...` with updated packages and imports.

- [ ] **Step 5: Run tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core
```
Expected: all tests pass including evaluator tests and registry tests.

- [ ] **Step 6: Commit**

```bash
git add quarkus-work-core/
git commit -m "$(cat <<'EOF'
feat(work-core): move JexlConditionEvaluator, filter registries, FilterRule

JexlConditionEvaluator.evaluate() now takes Map<String,Object> workUnitContext
instead of WorkItem — removes dependency on domain entity. toMap() removed
(moved to WorkItemContextBuilder in quarkus-work in Task 12).

Refs #ISSUE, #100
EOF
)"
```

---

### Task 11: Move FilterRegistryEngine and FilterRuleResource to quarkus-work-core

**Files:**
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/FilterRegistryEngine.java`
- Create: `quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/FilterRuleResource.java`
- Create (test): `quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter/FilterRegistryEngineTest.java`
- Create (test): `quarkus-work-core/src/test/resources/application.properties`

**Key changes in FilterRegistryEngine:**
1. Observer: `@Observes WorkItemLifecycleEvent` → `@Observes WorkLifecycleEvent`
2. Drops `WorkItemStore` dependency — gets work unit via `event.source()`
3. `conditionEvaluator.evaluate(...)` receives `event.context()` instead of `workItem`
4. `filterAction.apply(workUnit, params)` where `workUnit = event.source()`
5. `processEvent()` test method takes `WorkLifecycleEvent`

- [ ] **Step 1: Write failing FilterRegistryEngine test**

`quarkus-work-core/src/test/java/io/quarkiverse/work/core/filter/FilterRegistryEngineTest.java`:
```java
package io.quarkiverse.work.core.filter;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import io.quarkiverse.work.api.*;

class FilterRegistryEngineTest {

    private FilterRegistryEngine engine;
    private List<String> capturedWorkUnits;
    private FilterAction capturingAction;

    @BeforeEach void setUp() {
        capturedWorkUnits = new ArrayList<>();
        capturingAction = new FilterAction() {
            @Override public String type() { return "CAPTURE"; }
            @Override public void apply(Object workUnit, Map<String, Object> params) {
                capturedWorkUnits.add(workUnit.toString());
            }
        };
        engine = new FilterRegistryEngine(new JexlConditionEvaluator(), List.of(capturingAction));
    }

    private WorkLifecycleEvent event(WorkEventType type, Map<String, Object> ctx, Object source) {
        return new WorkLifecycleEvent() {
            @Override public WorkEventType eventType() { return type; }
            @Override public Map<String, Object> context() { return ctx; }
            @Override public Object source() { return source; }
        };
    }

    @Test void matchingCondition_actionFired() {
        var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("CAPTURE", Map.of())));
        var evt = event(WorkEventType.CREATED, Map.of("category", "finance"), "WORK_UNIT_1");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkUnits).containsExactly("WORK_UNIT_1");
    }

    @Test void nonMatchingCondition_actionNotFired() {
        var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'legal'", Map.of(),
                List.of(ActionDescriptor.of("CAPTURE", Map.of())));
        var evt = event(WorkEventType.CREATED, Map.of("category", "finance"), "UNIT");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkUnits).isEmpty();
    }

    @Test void disabledDefinition_actionNotFired() {
        var def = FilterDefinition.onAdd("test", "desc", false,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("CAPTURE", Map.of())));
        var evt = event(WorkEventType.CREATED, Map.of("category", "finance"), "UNIT");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkUnits).isEmpty();
    }

    @Test void wrongEventType_actionNotFired() {
        // def fires on ADD only; event is UPDATE
        var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("CAPTURE", Map.of())));
        var evt = event(WorkEventType.ASSIGNED, Map.of("category", "finance"), "UNIT");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkUnits).isEmpty();
    }

    @Test void unknownActionType_ignoredGracefully() {
        var def = FilterDefinition.onAdd("test", "desc", true,
                "workItem.category == 'finance'", Map.of(),
                List.of(ActionDescriptor.of("UNKNOWN_ACTION", Map.of())));
        var evt = event(WorkEventType.CREATED, Map.of("category", "finance"), "UNIT");
        engine.processEvent(evt, List.of(def));
        assertThat(capturedWorkUnits).isEmpty(); // no exception, just silently skipped
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core 2>&1 | tail -10
```

- [ ] **Step 3: Implement FilterRegistryEngine**

`quarkus-work-core/src/main/java/io/quarkiverse/work/core/filter/FilterRegistryEngine.java`:
```java
package io.quarkiverse.work.core.filter;

import java.util.List;
import java.util.stream.Stream;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import io.quarkiverse.work.api.WorkEventType;
import io.quarkiverse.work.api.WorkLifecycleEvent;

/**
 * Observes WorkLifecycleEvent (any subtype), evaluates all enabled filter definitions,
 * and applies matching actions via the FilterAction SPI.
 *
 * <p>A ThreadLocal guard prevents recursive triggering when actions (e.g. label application)
 * themselves fire lifecycle events.
 *
 * <p>The work unit is accessed via {@code event.source()} — no store lookup needed.
 * The context map for JEXL evaluation comes from {@code event.context()}.
 */
@ApplicationScoped
public class FilterRegistryEngine {

    private static final ThreadLocal<Boolean> RUNNING = ThreadLocal.withInitial(() -> false);

    private final JexlConditionEvaluator conditionEvaluator;
    private final List<FilterAction> filterActions;
    private final PermanentFilterRegistry permanentRegistry;
    private final DynamicFilterRegistry dynamicRegistry;

    @Inject
    public FilterRegistryEngine(final JexlConditionEvaluator conditionEvaluator,
            final Instance<FilterAction> filterActions,
            final PermanentFilterRegistry permanentRegistry,
            final DynamicFilterRegistry dynamicRegistry) {
        this.conditionEvaluator = conditionEvaluator;
        this.filterActions = filterActions.stream().toList();
        this.permanentRegistry = permanentRegistry;
        this.dynamicRegistry = dynamicRegistry;
    }

    /** Package-private constructor for unit tests — bypasses CDI. */
    FilterRegistryEngine(final JexlConditionEvaluator conditionEvaluator,
            final List<FilterAction> filterActions) {
        this.conditionEvaluator = conditionEvaluator;
        this.filterActions = filterActions;
        this.permanentRegistry = new PermanentFilterRegistry();
        this.dynamicRegistry = new DynamicFilterRegistry();
    }

    void onLifecycleEvent(@Observes final WorkLifecycleEvent event) {
        if (Boolean.TRUE.equals(RUNNING.get())) {
            return;
        }
        RUNNING.set(true);
        try {
            final List<FilterDefinition> allDefs = Stream.concat(
                    permanentRegistry.allEnabled().stream(),
                    dynamicRegistry.allEnabled().stream()).toList();
            applyFilters(event, allDefs);
        } finally {
            RUNNING.remove();
        }
    }

    /**
     * Package-visible for unit testing: evaluate definitions against the event without
     * going through the CDI observer or reentrancy guard.
     */
    void processEvent(final WorkLifecycleEvent event, final List<FilterDefinition> defs) {
        applyFilters(event, defs);
    }

    private void applyFilters(final WorkLifecycleEvent event, final List<FilterDefinition> defs) {
        final FilterEvent filterEvent = toFilterEvent(event.eventType());
        final Object workUnit = event.source();
        for (final FilterDefinition def : defs) {
            if (!def.enabled()) continue;
            if (!def.events().contains(filterEvent)) continue;
            if (!conditionEvaluator.evaluate(def.condition(), def.conditionContext(), event.context())) continue;
            for (final ActionDescriptor action : def.actions()) {
                filterActions.stream()
                        .filter(a -> a.type().equals(action.type()))
                        .findFirst()
                        .ifPresent(a -> a.apply(workUnit, action.params()));
            }
        }
    }

    private FilterEvent toFilterEvent(final WorkEventType eventType) {
        return switch (eventType) {
            case CREATED -> FilterEvent.ADD;
            case COMPLETED, REJECTED, CANCELLED, EXPIRED, ESCALATED -> FilterEvent.REMOVE;
            default -> FilterEvent.UPDATE;
        };
    }
}
```

- [ ] **Step 4: Move FilterRuleResource**

Copy `filterregistry/api/FilterRuleResource.java` → `io.quarkiverse.work.core.filter.FilterRuleResource` — update package and all imports (FilterRule, DynamicFilterRegistry, PermanentFilterRegistry, FilterDefinition now from `io.quarkiverse.work.core.filter`).

- [ ] **Step 5: Add test application.properties for @QuarkusTest in work-core**

`quarkus-work-core/src/test/resources/application.properties`:
```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:workcore;DB_CLOSE_DELAY=-1
quarkus.flyway.migrate-at-start=true
quarkus.http.test-port=0
```

- [ ] **Step 6: Move FilterRuleEvaluationTest and FilterActionIntegrationTest**

Copy `quarkus-work-filter-registry/src/test/.../FilterRuleEvaluationTest.java` → `quarkus-work-core/src/test/.../filter/FilterRuleEvaluationTest.java` — update package + imports. These are `@QuarkusTest` tests; they need the `quarkus-work` runtime on the classpath for full CDI context with WorkItem. Add `quarkus-work` as a `test` scope dependency in `quarkus-work-core/pom.xml`:
```xml
<dependency>
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work</artifactId>
  <version>${project.version}</version>
  <scope>test</scope>
</dependency>
```
(This is a test-only dep — no circular dep since runtime is not built yet. Wire after runtime is updated in Task 13.)

- [ ] **Step 7: Run unit tests only (skip @QuarkusTest for now)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core -Dtest="FilterRegistryEngineTest,JexlConditionEvaluatorTest,PermanentFilterRegistryTest,DynamicFilterRegistryTest,WorkBrokerTest,LeastLoadedStrategyTest,ClaimFirstStrategyTest"
```
Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git add quarkus-work-core/
git commit -m "$(cat <<'EOF'
feat(work-core): move FilterRegistryEngine and FilterRuleResource

Engine now observes WorkLifecycleEvent (any subtype) instead of WorkItemLifecycleEvent.
Drops WorkItemStore dependency — work unit from event.source(), context from event.context().
FilterEvent mapping switches from string parsing to WorkEventType enum match.

Refs #ISSUE, #100
EOF
)"
```

---

## Phase 3 — Update quarkus-work Runtime

### Task 12: Add WorkItemContextBuilder and update WorkItemLifecycleEvent

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/event/WorkItemContextBuilder.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/event/WorkItemLifecycleEvent.java`
- Create (test): `runtime/src/test/java/io/quarkiverse/workitems/runtime/event/WorkItemContextBuilderTest.java`

**WorkItemContextBuilder** extracts `toMap(WorkItem)` from `JexlConditionEvaluator`. The drift-protection test (verifying all WorkItem fields are in the map) moves here.

**WorkItemLifecycleEvent** changes from a record to a class extending `WorkLifecycleEvent`. Keeps all existing accessor methods so callers don't change.

- [ ] **Step 1: Write failing tests**

`runtime/src/test/java/io/quarkiverse/workitems/runtime/event/WorkItemContextBuilderTest.java`:
```java
package io.quarkiverse.work.runtime.event;

import static org.assertj.core.api.Assertions.assertThat;
import java.lang.reflect.Modifier;
import java.util.Arrays;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import io.quarkiverse.work.runtime.model.*;

class WorkItemContextBuilderTest {

    @Test void toMap_containsId() {
        WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.title = "Test";
        wi.status = WorkItemStatus.PENDING;
        wi.priority = WorkItemPriority.HIGH;
        Map<String, Object> map = WorkItemContextBuilder.toMap(wi);
        assertThat(map).containsKey("id");
        assertThat(map.get("id")).isEqualTo(wi.id);
    }

    @Test void toMap_containsAllPublicNonStaticWorkItemFields() {
        // Drift protection: any new field added to WorkItem must be added here too
        var expected = Arrays.stream(WorkItem.class.getFields())
                .filter(f -> !Modifier.isStatic(f.getModifiers()))
                .map(f -> f.getName())
                .toList();
        WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        Map<String, Object> map = WorkItemContextBuilder.toMap(wi);
        assertThat(map.keySet()).containsAll(expected);
    }

    @Test void toMap_returnsCorrectEnumReference() {
        WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.status = WorkItemStatus.IN_PROGRESS;
        wi.priority = WorkItemPriority.CRITICAL;
        Map<String, Object> map = WorkItemContextBuilder.toMap(wi);
        assertThat(map.get("status")).isEqualTo(WorkItemStatus.IN_PROGRESS);
        assertThat(map.get("priority")).isEqualTo(WorkItemPriority.CRITICAL);
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemContextBuilderTest -pl runtime 2>&1 | tail -10
```

- [ ] **Step 3: Implement WorkItemContextBuilder**

`runtime/src/main/java/io/quarkiverse/workitems/runtime/event/WorkItemContextBuilder.java`:

Copy the `toMap(WorkItem)` method body verbatim from the existing `JexlConditionEvaluator.toMap()` (it reads every public field of `WorkItem`). Class structure:

```java
package io.quarkiverse.work.runtime.event;

import java.util.HashMap;
import java.util.Map;

import io.quarkiverse.work.runtime.model.WorkItem;

/**
 * Builds the JEXL context map for a WorkItem.
 *
 * <p>Direct field access is used instead of reflection because Hibernate ORM bytecode
 * enhancement intercepts field storage — Field.get() via reflection returns stale values.
 * See JexlConditionEvaluator in quarkus-work-core for the evaluation contract.
 */
public final class WorkItemContextBuilder {

    private WorkItemContextBuilder() {}

    public static Map<String, Object> toMap(final WorkItem workItem) {
        // [copy the entire body of the existing JexlConditionEvaluator.toMap() verbatim]
        final Map<String, Object> map = new HashMap<>();
        map.put("id", workItem.id);
        map.put("version", workItem.version);
        map.put("title", workItem.title);
        // ... all fields as in the existing toMap() ...
        map.put("confidenceScore", workItem.confidenceScore);
        return map;
    }
}
```

- [ ] **Step 4: Run WorkItemContextBuilderTest — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemContextBuilderTest -pl runtime
```

- [ ] **Step 5: Update WorkItemLifecycleEvent from record to class**

`runtime/src/main/java/io/quarkiverse/workitems/runtime/event/WorkItemLifecycleEvent.java`:

The new class:
- Extends `WorkLifecycleEvent` (from `io.quarkiverse.work.api`)
- Carries the `WorkItem` entity as a private field (so `source()` can return it)
- Keeps all existing accessor methods (`type()`, `workItemId()`, `status()`, `actor()`, `detail()`, `rationale()`, `planRef()`, `occurredAt()`, `source()`, `subject()`) so all callers continue to compile
- Implements `eventType()`, `context()`, `source()` from `WorkLifecycleEvent`
- Updates static factory methods to take `WorkItem` entity directly (derive `id` and `status` from it)

```java
package io.quarkiverse.work.runtime.event;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import io.quarkiverse.work.api.WorkEventType;
import io.quarkiverse.work.api.WorkLifecycleEvent;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemStatus;

/**
 * CDI event fired on every WorkItem lifecycle transition.
 * Extends WorkLifecycleEvent so quarkus-work-core observers (e.g. FilterRegistryEngine)
 * receive it automatically.
 *
 * <p>The WorkItem entity is embedded so observers can access current state without
 * a separate store lookup. All existing accessor methods are preserved.
 */
public final class WorkItemLifecycleEvent extends WorkLifecycleEvent {

    private final String type;
    private final String sourceUri;
    private final String subject;
    private final UUID workItemId;
    private final WorkItemStatus status;
    private final Instant occurredAt;
    private final String actor;
    private final String detail;
    private final String rationale;
    private final String planRef;
    private final WorkItem workItem;

    private WorkItemLifecycleEvent(final String type, final String sourceUri, final String subject,
            final UUID workItemId, final WorkItemStatus status, final Instant occurredAt,
            final String actor, final String detail, final String rationale, final String planRef,
            final WorkItem workItem) {
        this.type = type;
        this.sourceUri = sourceUri;
        this.subject = subject;
        this.workItemId = workItemId;
        this.status = status;
        this.occurredAt = occurredAt;
        this.actor = actor;
        this.detail = detail;
        this.rationale = rationale;
        this.planRef = planRef;
        this.workItem = workItem;
    }

    public static WorkItemLifecycleEvent of(final String eventName, final WorkItem workItem,
            final String actor, final String detail) {
        return new WorkItemLifecycleEvent(
                "io.quarkiverse.work.workitem." + eventName.toLowerCase(),
                "/workitems/" + workItem.id,
                workItem.id.toString(),
                workItem.id, workItem.status, Instant.now(),
                actor, detail, null, null, workItem);
    }

    public static WorkItemLifecycleEvent of(final String eventName, final WorkItem workItem,
            final String actor, final String detail,
            final String rationale, final String planRef) {
        return new WorkItemLifecycleEvent(
                "io.quarkiverse.work.workitem." + eventName.toLowerCase(),
                "/workitems/" + workItem.id,
                workItem.id.toString(),
                workItem.id, workItem.status, Instant.now(),
                actor, detail, rationale, planRef, workItem);
    }

    // Existing accessors — unchanged names so all callers compile without modification
    public String type()           { return type; }
    public String source()         { return sourceUri; }
    public String subject()        { return subject; }
    public UUID workItemId()       { return workItemId; }
    public WorkItemStatus status() { return status; }
    public Instant occurredAt()    { return occurredAt; }
    public String actor()          { return actor; }
    public String detail()         { return detail; }
    public String rationale()      { return rationale; }
    public String planRef()        { return planRef; }

    // WorkLifecycleEvent abstract method implementations
    @Override
    public WorkEventType eventType() {
        final String name = type.substring(type.lastIndexOf('.') + 1).toUpperCase();
        try {
            return WorkEventType.valueOf(name);
        } catch (IllegalArgumentException e) {
            return WorkEventType.CREATED; // fallback — should never occur with known event names
        }
    }

    @Override
    public Map<String, Object> context() {
        return WorkItemContextBuilder.toMap(workItem);
    }

    @Override
    public Object source() {
        return workItem;
    }
}
```

**Note:** `source()` now returns `Object` (from `WorkLifecycleEvent`) rather than `String`. Check all callers of `event.source()` in the runtime — if any called `event.source()` expecting the URI string, they must be updated to use `event.sourceUri()` or just not call `source()`. Looking at the existing code, only the old record had `source` as the URI string. The new class renames it `sourceUri` internally and the public method `source()` returns `Object` (the WorkItem). **Verify no runtime callers used `event.source()` as the URI.** If `LedgerEventCapture` or similar used it, add a `sourceUri()` accessor.

- [ ] **Step 6: Find all callers of the old WorkItemLifecycleEvent.of() with UUID signature**

```bash
grep -rn "WorkItemLifecycleEvent.of(" runtime/src quarkus-work-ledger/src quarkus-work-queues/src work-flow/src 2>/dev/null | grep -v "workItem,"
```

Update every call site from `WorkItemLifecycleEvent.of("EVENT", item.id, item.status, ...)` to `WorkItemLifecycleEvent.of("EVENT", item, ...)`. The `WorkItemService` fires many such events — update all of them.

- [ ] **Step 7: Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all 525+ tests pass (same count as before).

- [ ] **Step 8: Commit**

```bash
git add runtime/
git commit -m "$(cat <<'EOF'
feat(workitems): add WorkItemContextBuilder, update WorkItemLifecycleEvent

WorkItemContextBuilder.toMap(WorkItem) extracted from JexlConditionEvaluator.
WorkItemLifecycleEvent: record→class extending WorkLifecycleEvent; embeds
WorkItem entity so filter engine needs no store lookup. All existing
accessors preserved — call sites unchanged.

Refs #ISSUE, #100
EOF
)"
```

---

### Task 13: Add JpaWorkloadProvider and update WorkItemAssignmentService

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/JpaWorkloadProvider.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemAssignmentService.java`
- Create (test): `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/JpaWorkloadProviderTest.java`

- [ ] **Step 1: Write failing test for JpaWorkloadProvider**

`runtime/src/test/java/io/quarkiverse/workitems/runtime/service/JpaWorkloadProviderTest.java`:
```java
package io.quarkiverse.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import io.quarkiverse.work.runtime.model.*;
import io.quarkiverse.work.testing.InMemoryWorkItemStore;

class JpaWorkloadProviderTest {

    @Test void returnsCountOfActiveWorkItemsForActor() {
        var store = new InMemoryWorkItemStore();
        var provider = new JpaWorkloadProvider(store);

        WorkItem wi1 = workItem("alice", WorkItemStatus.ASSIGNED);
        WorkItem wi2 = workItem("alice", WorkItemStatus.IN_PROGRESS);
        WorkItem wi3 = workItem("alice", WorkItemStatus.COMPLETED); // terminal — not counted
        WorkItem wi4 = workItem("bob",   WorkItemStatus.ASSIGNED);
        store.put(wi1); store.put(wi2); store.put(wi3); store.put(wi4);

        assertThat(provider.getActiveWorkCount("alice")).isEqualTo(2);
        assertThat(provider.getActiveWorkCount("bob")).isEqualTo(1);
        assertThat(provider.getActiveWorkCount("carol")).isZero();
    }

    @Test void suspended_countedAsActive() {
        var store = new InMemoryWorkItemStore();
        var provider = new JpaWorkloadProvider(store);
        store.put(workItem("alice", WorkItemStatus.SUSPENDED));
        assertThat(provider.getActiveWorkCount("alice")).isEqualTo(1);
    }

    private WorkItem workItem(String assignee, WorkItemStatus status) {
        var wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.assigneeId = assignee;
        wi.status = status;
        wi.title = "T";
        wi.createdBy = "test";
        return wi;
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JpaWorkloadProviderTest -pl runtime 2>&1 | tail -10
```

- [ ] **Step 3: Implement JpaWorkloadProvider**

`runtime/src/main/java/io/quarkiverse/workitems/runtime/service/JpaWorkloadProvider.java`:
```java
package io.quarkiverse.work.runtime.service;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.quarkiverse.work.api.WorkloadProvider;
import io.quarkiverse.work.runtime.model.WorkItemStatus;
import io.quarkiverse.work.runtime.repository.WorkItemQuery;
import io.quarkiverse.work.runtime.repository.WorkItemStore;

/**
 * WorkloadProvider backed by the JPA WorkItemStore.
 * Counts ASSIGNED, IN_PROGRESS, and SUSPENDED WorkItems for a given actor.
 */
@ApplicationScoped
public class JpaWorkloadProvider implements WorkloadProvider {

    private static final List<WorkItemStatus> ACTIVE_STATUSES = List.of(
            WorkItemStatus.ASSIGNED, WorkItemStatus.IN_PROGRESS, WorkItemStatus.SUSPENDED);

    private final WorkItemStore workItemStore;

    @Inject
    public JpaWorkloadProvider(final WorkItemStore workItemStore) {
        this.workItemStore = workItemStore;
    }

    @Override
    public int getActiveWorkCount(final String workerId) {
        return (int) workItemStore.scan(WorkItemQuery.builder()
                .assigneeId(workerId)
                .statusIn(ACTIVE_STATUSES)
                .build())
                .stream()
                .filter(wi -> workerId.equals(wi.assigneeId))
                .count();
    }
}
```

- [ ] **Step 4: Update WorkItemAssignmentService to use WorkBroker + JpaWorkloadProvider**

Replace the `countActive()` private method and the capability-filtering section in `resolveCandidates()` with calls to `JpaWorkloadProvider` and `WorkBroker`. The `assign()` method delegates the generic parts to `workBroker.apply()`:

```java
// Updated assign() method:
public void assign(final WorkItem workItem, final AssignmentTrigger trigger) {
    final WorkerSelectionStrategy strategy = activeStrategy();
    final List<WorkerCandidate> candidates = resolveCandidates(workItem);
    final SelectionContext context = new SelectionContext(
            workItem.category,
            workItem.priority != null ? workItem.priority.name() : null,
            workItem.requiredCapabilities,
            workItem.candidateGroups,
            workItem.candidateUsers);
    final AssignmentDecision decision = workBroker.apply(context, trigger, candidates, strategy);
    applyDecision(workItem, decision);
}
```

Remove the `countActive()` private method — replace calls with `workloadProvider.getActiveWorkCount(id)`.
Remove the capability-filtering from `resolveCandidates()` — WorkBroker now handles that.
Inject `WorkBroker workBroker` and `WorkloadProvider workloadProvider` (= `JpaWorkloadProvider`).
Update CDI constructor.
Update the test constructor accordingly.

- [ ] **Step 5: Run runtime tests — expect GREEN**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests pass (525+ still green).

- [ ] **Step 6: Commit**

```bash
git add runtime/
git commit -m "$(cat <<'EOF'
feat(workitems): add JpaWorkloadProvider, wire WorkBroker into WorkItemAssignmentService

JpaWorkloadProvider implements WorkloadProvider via JPA store (ASSIGNED/IN_PROGRESS/SUSPENDED).
WorkItemAssignmentService delegates trigger gating + capability filtering to WorkBroker;
retains candidate resolution, workload count population, and decision application.

Refs #ISSUE, #100, #102
EOF
)"
```

---

### Task 14: Update EscalationPolicy implementations

**Files:**
- Delete: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/EscalationPolicy.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/NotifyEscalationPolicy.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/AutoRejectEscalationPolicy.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/ReassignEscalationPolicy.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/ExpiryCleanupJob.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/ClaimDeadlineJob.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/EscalationPolicyProducer.java`

**Strategy:** `NotifyEscalationPolicy.escalate(event)` checks `event.eventType()` to distinguish EXPIRED vs CLAIM_EXPIRED and dispatches to the old logic (just refactored inline). Similarly for the other two.

- [ ] **Step 1: Update NotifyEscalationPolicy**

```java
@ApplicationScoped
public class NotifyEscalationPolicy implements EscalationPolicy {

    private static final Logger LOG = Logger.getLogger(NotifyEscalationPolicy.class);

    @Inject Event<WorkItemExpiredEvent> expiredEvent;

    @Override
    public void escalate(final WorkLifecycleEvent event) {
        final WorkItem workItem = (WorkItem) event.source();
        if (event.eventType() == WorkEventType.CLAIM_EXPIRED) {
            LOG.warnf("WorkItem %s unclaimed past deadline", workItem.id);
        } else {
            LOG.warnf("WorkItem %s expired (was %s)", workItem.id, workItem.status);
            expiredEvent.fire(new WorkItemExpiredEvent(workItem.id, workItem.status));
        }
    }
}
```

Update `AutoRejectEscalationPolicy` and `ReassignEscalationPolicy` the same way — cast `event.source()` to `WorkItem`, check `event.eventType()` where the two cases differ.

- [ ] **Step 2: Update ExpiryCleanupJob to build event and call escalate()**

Replace:
```java
escalationPolicy.onExpired(item);
```
With — build a lifecycle event for ESCALATED and pass it:
```java
final WorkItemLifecycleEvent escalatedEvent =
    WorkItemLifecycleEvent.of("ESCALATED", item, "system", null);
escalationPolicy.escalate(escalatedEvent);
lifecycleEvent.fire(escalatedEvent);
```
(Remove the separate `lifecycleEvent.fire(WorkItemLifecycleEvent.of("ESCALATED", ...))` line that followed the old call — it's now one event, fired once after the escalation policy runs.)

- [ ] **Step 3: Update ClaimDeadlineJob similarly**

Replace:
```java
claimEscalationPolicy.onUnclaimedPastDeadline(item);
lifecycleEvent.fire(WorkItemLifecycleEvent.of("ESCALATED", item.id, item.status, "system", null));
```
With:
```java
final WorkItemLifecycleEvent claimExpiredEvent =
    WorkItemLifecycleEvent.of("CLAIM_EXPIRED", item, "system", null);
claimEscalationPolicy.escalate(claimExpiredEvent);
lifecycleEvent.fire(claimExpiredEvent);
```

- [ ] **Step 4: Delete the old EscalationPolicy.java from runtime/service** and update `EscalationPolicyProducer` to import from `io.quarkiverse.work.api.EscalationPolicy`.

- [ ] **Step 5: Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/
git commit -m "$(cat <<'EOF'
feat(workitems): migrate EscalationPolicy to generic WorkLifecycleEvent-based SPI

Old two-method interface (onExpired/onUnclaimedPastDeadline) replaced by single
escalate(WorkLifecycleEvent). Impls cast event.source() to WorkItem and check
event.eventType() to distinguish EXPIRED vs CLAIM_EXPIRED.

Refs #ISSUE, #101
EOF
)"
```

---

### Task 15: Move filter actions to runtime, update runtime pom

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/action/ApplyLabelAction.java`
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/action/OverrideCandidateGroupsAction.java`
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/action/SetPriorityAction.java`
- Modify: `runtime/pom.xml`

- [ ] **Step 1: Copy ApplyLabelAction, OverrideCandidateGroupsAction, SetPriorityAction from quarkus-work-filter-registry to runtime/action/**

Update each file:
- Package: `io.quarkiverse.work.runtime.action`
- Import `FilterAction` from `io.quarkiverse.work.core.filter.FilterAction`
- Change `void apply(WorkItem workItem, Map<String, Object> params)` signature to `void apply(Object workUnit, Map<String, Object> params)` and add cast at top: `final WorkItem workItem = (WorkItem) workUnit;`

Example for `ApplyLabelAction`:
```java
package io.quarkiverse.work.runtime.action;

import java.util.ArrayList;
import java.util.Map;
import jakarta.enterprise.context.ApplicationScoped;
import io.quarkiverse.work.core.filter.FilterAction;
import io.quarkiverse.work.runtime.model.*;

@ApplicationScoped
public class ApplyLabelAction implements FilterAction {
    @Override public String type() { return "APPLY_LABEL"; }

    @Override
    public void apply(final Object workUnit, final Map<String, Object> params) {
        final WorkItem workItem = (WorkItem) workUnit;
        // [rest of body identical to existing ApplyLabelAction]
    }
}
```

- [ ] **Step 2: Update runtime/pom.xml**

Replace:
```xml
<dependency>
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work-api</artifactId>
  <version>${project.version}</version>
</dependency>
```
With:
```xml
<dependency>
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work-core</artifactId>
  <version>${project.version}</version>
</dependency>
```

- [ ] **Step 3: Update all runtime imports from io.quarkiverse.work.spi.* to io.quarkiverse.work.api.***

Files to update: `WorkItemAssignmentService`, `WorkItemService`, `LeastLoadedStrategy` (if still present — will be deleted in Task 16), any test files importing old SPI package.

```bash
find runtime/src -name "*.java" | xargs grep -l "io.quarkiverse.work.spi" | sort
```
Update each file's imports.

- [ ] **Step 4: Build runtime — expect clean compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl quarkus-work-api,quarkus-work-core,runtime
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/
git commit -m "$(cat <<'EOF'
feat(workitems): move filter actions to runtime, wire quarkus-work-core dependency

ApplyLabelAction, OverrideCandidateGroupsAction, SetPriorityAction moved from
quarkus-work-filter-registry to runtime/action. Each casts workUnit to WorkItem.
Runtime now depends on quarkus-work-core (was quarkus-work-api).

Refs #ISSUE, #100
EOF
)"
```

---

## Phase 4 — Delete Old Modules

### Task 16: Delete quarkus-work-api and quarkus-work-filter-registry

**Note:** These modules still have their old strategy/registry code. Remove them only after all consumers have been updated.

- [ ] **Step 1: Remove old modules from root pom.xml**

In `pom.xml` `<modules>`, remove:
```xml
<module>quarkus-work-api</module>
<module>quarkus-work-filter-registry</module>
```

- [ ] **Step 2: Delete the directories**

```bash
rm -rf quarkus-work-api/
rm -rf quarkus-work-filter-registry/
```

- [ ] **Step 3: Verify full build — no references to deleted modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl quarkus-work-api,quarkus-work-core,runtime,deployment,testing
```
Expected: BUILD SUCCESS.

- [ ] **Step 4: Remove old strategy/registry copies from runtime/service (LeastLoadedStrategy, ClaimFirstStrategy, NoOpWorkerRegistry)**

```bash
rm runtime/src/main/java/io/quarkiverse/workitems/runtime/service/LeastLoadedStrategy.java
rm runtime/src/main/java/io/quarkiverse/workitems/runtime/service/ClaimFirstStrategy.java
rm runtime/src/main/java/io/quarkiverse/workitems/runtime/service/NoOpWorkerRegistry.java
```

Update `WorkItemAssignmentService.activeStrategy()` to reference the new package:
- Import `io.quarkiverse.work.core.strategy.ClaimFirstStrategy`
- Import `io.quarkiverse.work.core.strategy.LeastLoadedStrategy`

- [ ] **Step 5: Run full build including all existing modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests -pl quarkus-work-api,quarkus-work-core,runtime,deployment,testing,work-flow,quarkus-work-ledger,quarkus-work-queues,quarkus-work-issue-tracker,quarkus-work-persistence-mongodb
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "$(cat <<'EOF'
chore: delete quarkus-work-api and quarkus-work-filter-registry

Both absorbed into quarkus-work-api and quarkus-work-core respectively.
Old LeastLoadedStrategy/ClaimFirstStrategy/NoOpWorkerRegistry copies removed
from runtime/service — now imported from io.quarkiverse.work.core.strategy.

Refs #ISSUE, #100, #102
EOF
)"
```

---

## Phase 5 — Update quarkus-work-ai

### Task 17: Update quarkus-work-ai

**Files:**
- Modify: `quarkus-work-ai/pom.xml`
- Modify: `quarkus-work-ai/src/main/java/io/quarkiverse/workitems/ai/filter/LowConfidenceFilterProducer.java`

- [ ] **Step 1: Update pom.xml — replace filter-registry dep with work-core**

Replace:
```xml
<dependency>
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work-filter-registry</artifactId>
  <version>${project.version}</version>
</dependency>
```
With:
```xml
<dependency>
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work-core</artifactId>
  <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Update imports in LowConfidenceFilterProducer**

Change:
```java
import io.quarkiverse.work.filterregistry.spi.ActionDescriptor;
import io.quarkiverse.work.filterregistry.spi.FilterDefinition;
```
To:
```java
import io.quarkiverse.work.core.filter.ActionDescriptor;
import io.quarkiverse.work.core.filter.FilterDefinition;
```

- [ ] **Step 3: Run quarkus-work-ai tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-ai
```
Expected: all 8 tests pass.

- [ ] **Step 4: Commit**

```bash
git add quarkus-work-ai/
git commit -m "$(cat <<'EOF'
feat(workitems-ai): update to quarkus-work-core dependency

Replace quarkus-work-filter-registry dep with quarkus-work-core.
Update FilterDefinition/ActionDescriptor imports to io.quarkiverse.work.core.filter.

Refs #ISSUE, #114
EOF
)"
```

---

## Phase 6 — Full Build and Integration Tests

### Task 18: Full build and test suite

- [ ] **Step 1: Build all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests
```
Expected: BUILD SUCCESS.

- [ ] **Step 2: Run all unit and @QuarkusTest tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```
Expected: all existing tests pass (943+ total, 0 failures).

- [ ] **Step 3: Run @QuarkusIntegrationTest suite (JVM mode)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests
```
Expected: 19 tests pass, BUILD SUCCESS.

- [ ] **Step 4: Confirm filter-registry integration tests pass in quarkus-work-core**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-core
```
Expected: all 40 former filter-registry tests pass under new package.

- [ ] **Step 5: Commit if clean (no code changes needed)**

If all green, proceed. If any test fails, fix it and commit the fix before continuing.

---

## Phase 7 — Documentation

### Task 19: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update the Project Structure section**

Replace the old module table with the new one reflecting `quarkus-work-api`, `quarkus-work-core`, and the deletion of `quarkus-work-api` and `quarkus-work-filter-registry`.

- [ ] **Step 2: Update Known Quarkiverse Gotchas section**

Remove any gotcha entries that reference `quarkus-work-filter-registry` or `quarkus-work-api` as standalone modules. Add if applicable: any new gotchas discovered during this refactor.

- [ ] **Step 3: Update package references** — any mention of `io.quarkiverse.work.spi` → `io.quarkiverse.work.api`; `io.quarkiverse.work.filterregistry` → `io.quarkiverse.work.core.filter`.

- [ ] **Step 4: Verify CLAUDE.md build commands still accurate** — confirm the module-specific test commands (`-pl runtime`, `-pl quarkus-work-queues`, etc.) are all still valid.

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
docs: update CLAUDE.md for quarkus-work-api/work-core separation

Module table updated: quarkus-work-api and quarkus-work-filter-registry
removed; quarkus-work-api and quarkus-work-core added. Package references updated.

Refs #ISSUE
EOF
)"
```

---

### Task 20: Update DESIGN.md and close issue

**Files:**
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Update the Component Structure table in DESIGN.md**

Replace the API row (quarkus-work-api) with two rows:
- `quarkus-work-api` — SPI types (WorkEventType, WorkLifecycleEvent, WorkloadProvider, EscalationPolicy, WorkerCandidate, SelectionContext, AssignmentDecision, AssignmentTrigger, WorkerSelectionStrategy, WorkerRegistry)
- `quarkus-work-core` — WorkBroker, LeastLoadedStrategy, ClaimFirstStrategy, NoOpWorkerRegistry, filter engine (FilterAction SPI, FilterRegistryEngine, JexlConditionEvaluator, PermanentFilterRegistry, DynamicFilterRegistry, FilterRule, FilterRuleResource)

Remove the `Filter Registry` row (dissolved into work-core). Update the `Runtime` row — it no longer contains strategies or filter actions in service/.

- [ ] **Step 2: Update the Services table** — remove LeastLoadedStrategy, ClaimFirstStrategy, NoOpWorkerRegistry (moved to work-core). Add JpaWorkloadProvider.

- [ ] **Step 3: Update the Build Roadmap** — mark Phase 12 complete, add a new completed entry for the quarkus-work separation.

- [ ] **Step 4: Scan DESIGN.md for drift** — read every section and verify it matches current code. Fix any stale references to old package names or deleted modules.

- [ ] **Step 5: Close the GitHub issue**

```bash
gh issue close ISSUE_NUMBER --comment "Implemented in quarkus-work-api and quarkus-work-core. All 943+ tests passing."
```

- [ ] **Step 6: Commit**

```bash
git add docs/DESIGN.md
git commit -m "$(cat <<'EOF'
docs: update DESIGN.md for quarkus-work separation

Component table updated: quarkus-work-api + quarkus-work-core added,
quarkus-work-api + quarkus-work-filter-registry removed.
Build Roadmap extended with Phase 13 complete.

Closes #ISSUE, Refs #100, #102
EOF
)"
```

---

### Task 21: Update HANDOFF.md

**Files:**
- Modify: `HANDOFF.md`

- [ ] **Step 1: Rewrite HANDOFF.md** — update Project Status (new module count, new test totals), What Was Built This Session (quarkus-work-api, quarkus-work-core, WorkBroker, WorkLifecycleEvent), Immediate Next Step (continue Epic #100 AI features: semantic skill matching).

- [ ] **Step 2: Commit**

```bash
git add HANDOFF.md
git commit -m "$(cat <<'EOF'
docs: session handover 2026-04-22 — quarkus-work separation complete

Refs #ISSUE
EOF
)"
```

---

## Self-Review Checklist

After completing all tasks, verify:

- [ ] `quarkus-work-api` has zero runtime dependencies (only test-scope JUnit/AssertJ)
- [ ] `quarkus-work-core` has no import of `io.quarkiverse.work.*` in main sources
- [ ] `quarkus-work` runtime has no import of `io.quarkiverse.work.spi.*` (old package gone)
- [ ] `quarkus-work-ai` imports from `io.quarkiverse.work.core.filter`
- [ ] All 943+ tests pass
- [ ] Integration tests (19) pass in JVM mode
- [ ] `FilterRegistryEngine` observer type is `WorkLifecycleEvent`, not `WorkItemLifecycleEvent`
- [ ] `FilterAction.apply()` signature is `(Object workUnit, Map<String,Object> params)`
- [ ] `JexlConditionEvaluator.evaluate()` takes `Map<String,Object>` as third arg, not `WorkItem`
- [ ] No file imports `io.quarkiverse.work.filterregistry.*`
- [ ] No file imports `io.quarkiverse.work.spi.*`
- [ ] DESIGN.md, CLAUDE.md, HANDOFF.md all reflect the new structure
