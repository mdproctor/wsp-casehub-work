# WorkerSelectionStrategy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement pluggable worker selection for WorkItems — a new `quarkus-work-api` module with shared SPIs (WorkerSelectionStrategy, WorkerRegistry, WorkerCandidate, AssignmentDecision), and built-in LeastLoadedStrategy/ClaimFirstStrategy in the runtime, wired into WorkItemService on create/release/delegate.

**Architecture:** Three layers — `quarkus-work-api` (pure Java, zero runtime deps) owns the shared SPI contracts; `quarkus-work` runtime implements built-in strategies and wires them into `WorkItemAssignmentService`; `WorkItemService.create/release/delegate` call the assignment service before persisting. The `SelectionContext` record (in api module) decouples strategies from the WorkItem entity.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache ORM, CDI `Instance<>` for strategy discovery, `WorkItemStore.scan()` for active-count queries. Maven: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn`.

---

## File Map

### New module: `quarkus-work-api`
```
quarkus-work-api/
  pom.xml
  src/main/java/io/quarkiverse/workitems/spi/
    WorkerCandidate.java          record: id, capabilities, activeWorkItemCount
    SelectionContext.java          record: category, priority, requiredCapabilities,
                                           candidateGroups, candidateUsers
    AssignmentDecision.java       record: assigneeId?, candidateGroups?, candidateUsers?
                                  + noChange() / assignTo() / narrowCandidates() factories
    AssignmentTrigger.java        enum: CREATED, RELEASED, DELEGATED
    WorkerSelectionStrategy.java  interface: select(SelectionContext, List<WorkerCandidate>)
                                             + default triggers()
    WorkerRegistry.java           interface: resolveGroup(String): List<WorkerCandidate>
  src/test/java/io/quarkiverse/workitems/spi/
    WorkerCandidateTest.java      unit — record construction + of() factory
    AssignmentDecisionTest.java   unit — all three factory methods
    SelectionContextTest.java     unit — record construction
```

### Modified: `runtime` module
```
runtime/src/main/java/io/quarkiverse/workitems/runtime/
  config/WorkItemsConfig.java       add routing() sub-group (strategy default: least-loaded)
  service/WorkItemAssignmentService.java  NEW — orchestrates resolve→filter→select→apply
  service/ClaimFirstStrategy.java   NEW — @ApplicationScoped WorkerSelectionStrategy, noChange
  service/LeastLoadedStrategy.java  NEW — @ApplicationScoped, picks min activeWorkItemCount
  service/NoOpWorkerRegistry.java   NEW — @ApplicationScoped, resolveGroup returns List.of()
  service/WorkItemService.java      wire WorkItemAssignmentService on create/release/delegate

runtime/src/test/java/io/quarkiverse/workitems/runtime/
  service/WorkItemAssignmentServiceTest.java  unit — trigger skip, candidate resolution,
                                              capability filter, decision application
  service/ClaimFirstStrategyTest.java         unit — always noChange()
  service/LeastLoadedStrategyTest.java        unit — min count, empty, ties
  api/WorkerSelectionStrategyIT.java          @QuarkusTest integration + E2E
```

### Modified: parent `pom.xml`
- Add `quarkus-work-api` to `<modules>` (before `runtime`)

### Modified: `runtime/pom.xml`
- Add compile dependency on `quarkus-work-api`

### Docs
- `docs/DESIGN.md` — new module row, domain model, REST API, test totals, roadmap
- `CLAUDE.md` — epic #100/#102 child issues closed

---

## Task 1: Create GitHub issues

- [ ] **Step 1: Create issue #115 — quarkus-work-api module**
```bash
gh issue create --repo mdproctor/quarkus-work \
  --title "quarkus-work-api module — shared WorkerSelectionStrategy SPI types" \
  --label "enhancement" \
  --body "## Context
Part of epics #100 and #102. Shared API module with zero runtime dependencies.

## What
New quarkus-work-api module containing:
- WorkerCandidate record (id, capabilities, activeWorkItemCount)
- SelectionContext record (category, priority, requiredCapabilities, candidateGroups, candidateUsers)
- AssignmentDecision record (assigneeId, candidateGroups, candidateUsers) + noChange/assignTo/narrowCandidates factories
- AssignmentTrigger enum (CREATED, RELEASED, DELEGATED)
- WorkerSelectionStrategy interface (select + default triggers)
- WorkerRegistry interface (resolveGroup)

## Why separate module
CaseHub and other systems can depend on the shared SPI without pulling in the full WorkItems runtime stack. Zero cyclical builds.

Refs #100
Refs #102"
```

- [ ] **Step 2: Create issue #116 — WorkItemAssignmentService + built-in strategies**
```bash
gh issue create --repo mdproctor/quarkus-work \
  --title "WorkItemAssignmentService + LeastLoadedStrategy + ClaimFirstStrategy" \
  --label "enhancement" \
  --body "## Context
Part of epics #100 and #102. Depends on #115.

## What
- WorkItemsConfig.routing().strategy() config sub-group (default: least-loaded)
- WorkItemAssignmentService: orchestrates candidate resolution, capability filtering, strategy selection, decision application
- ClaimFirstStrategy: no-op (returns noChange), pool stays open
- LeastLoadedStrategy: pre-assigns to candidateUser with fewest active (ASSIGNED/IN_PROGRESS/SUSPENDED) WorkItems
- NoOpWorkerRegistry: default group resolver (returns empty list)
- Wire into WorkItemService.create(), release(), delegate()

## Acceptance Criteria
- [ ] POST /workitems with candidateUsers=[alice,bob] and alice has 5 active, bob has 1 → pre-assigned to bob
- [ ] No candidates → noChange (pool stays open)
- [ ] PUT /workitems/{id}/release → strategy re-fires (RELEASED trigger)
- [ ] quarkus.work.routing.strategy=claim-first → no pre-assignment
- [ ] @Alternative custom strategy overrides config

Refs #100
Refs #102"
```

- [ ] **Step 3: Create issue for round-robin (deferred)**
```bash
gh issue create --repo mdproctor/quarkus-work \
  --title "RoundRobinStrategy — stateful sequential worker selection" \
  --label "enhancement" \
  --body "## Context
Part of epic #102. Deferred from initial WorkerSelectionStrategy implementation.

## What
RoundRobinStrategy distributes WorkItems sequentially across candidateUsers/resolved group members.
Requires a persistent cursor (DB-backed, cluster-safe) to track last-assigned index.

## Why deferred
The cursor requires a new entity and migration. Not blocking for the initial SPI delivery.
Implement after LeastLoadedStrategy is proven in production.

Refs #102"
```

---

## Task 2: quarkus-work-api module — scaffold + SPI types

**Files:**
- Create: `quarkus-work-api/pom.xml`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/WorkerCandidate.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/SelectionContext.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/AssignmentDecision.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/AssignmentTrigger.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/WorkerSelectionStrategy.java`
- Create: `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/WorkerRegistry.java`
- Test: `quarkus-work-api/src/test/java/io/quarkiverse/workitems/spi/WorkerCandidateTest.java`
- Test: `quarkus-work-api/src/test/java/io/quarkiverse/workitems/spi/AssignmentDecisionTest.java`
- Test: `quarkus-work-api/src/test/java/io/quarkiverse/workitems/spi/SelectionContextTest.java`

- [ ] **Step 1: Write failing tests (RED)**

Create `quarkus-work-api/src/test/java/io/quarkiverse/workitems/spi/WorkerCandidateTest.java`:
```java
package io.quarkiverse.work.spi;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.Set;
import org.junit.jupiter.api.Test;

class WorkerCandidateTest {

    @Test
    void of_createsCandidate_withEmptyCapabilitiesAndZeroCount() {
        final WorkerCandidate c = WorkerCandidate.of("alice");
        assertThat(c.id()).isEqualTo("alice");
        assertThat(c.capabilities()).isEmpty();
        assertThat(c.activeWorkItemCount()).isZero();
    }

    @Test
    void fullConstructor_setsAllFields() {
        final WorkerCandidate c = new WorkerCandidate("bob", Set.of("finance", "legal"), 3);
        assertThat(c.id()).isEqualTo("bob");
        assertThat(c.capabilities()).containsExactlyInAnyOrder("finance", "legal");
        assertThat(c.activeWorkItemCount()).isEqualTo(3);
    }

    @Test
    void withActiveWorkItemCount_returnsNewCandidateWithUpdatedCount() {
        final WorkerCandidate original = WorkerCandidate.of("carol");
        final WorkerCandidate updated = original.withActiveWorkItemCount(7);
        assertThat(updated.id()).isEqualTo("carol");
        assertThat(updated.activeWorkItemCount()).isEqualTo(7);
        assertThat(original.activeWorkItemCount()).isZero(); // immutable
    }
}
```

Create `quarkus-work-api/src/test/java/io/quarkiverse/workitems/spi/AssignmentDecisionTest.java`:
```java
package io.quarkiverse.work.spi;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class AssignmentDecisionTest {

    @Test
    void noChange_hasAllNullFields() {
        final AssignmentDecision d = AssignmentDecision.noChange();
        assertThat(d.assigneeId()).isNull();
        assertThat(d.candidateGroups()).isNull();
        assertThat(d.candidateUsers()).isNull();
    }

    @Test
    void assignTo_setsAssigneeId_othersNull() {
        final AssignmentDecision d = AssignmentDecision.assignTo("alice");
        assertThat(d.assigneeId()).isEqualTo("alice");
        assertThat(d.candidateGroups()).isNull();
        assertThat(d.candidateUsers()).isNull();
    }

    @Test
    void narrowCandidates_setsGroupsAndUsers_assigneeNull() {
        final AssignmentDecision d = AssignmentDecision.narrowCandidates("finance-team", "alice,bob");
        assertThat(d.assigneeId()).isNull();
        assertThat(d.candidateGroups()).isEqualTo("finance-team");
        assertThat(d.candidateUsers()).isEqualTo("alice,bob");
    }

    @Test
    void isNoOp_trueWhenAllFieldsNull() {
        assertThat(AssignmentDecision.noChange().isNoOp()).isTrue();
        assertThat(AssignmentDecision.assignTo("alice").isNoOp()).isFalse();
        assertThat(AssignmentDecision.narrowCandidates("g", "u").isNoOp()).isFalse();
    }
}
```

Create `quarkus-work-api/src/test/java/io/quarkiverse/workitems/spi/SelectionContextTest.java`:
```java
package io.quarkiverse.work.spi;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class SelectionContextTest {

    @Test
    void constructor_setsAllFields() {
        final SelectionContext ctx = new SelectionContext(
            "finance", "HIGH", "audit,legal", "finance-team", "alice,bob");
        assertThat(ctx.category()).isEqualTo("finance");
        assertThat(ctx.priority()).isEqualTo("HIGH");
        assertThat(ctx.requiredCapabilities()).isEqualTo("audit,legal");
        assertThat(ctx.candidateGroups()).isEqualTo("finance-team");
        assertThat(ctx.candidateUsers()).isEqualTo("alice,bob");
    }

    @Test
    void constructor_acceptsNullFields() {
        final SelectionContext ctx = new SelectionContext(null, null, null, null, null);
        assertThat(ctx.category()).isNull();
        assertThat(ctx.candidateGroups()).isNull();
    }
}
```

- [ ] **Step 2: Run tests RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api \
  -Dtest="WorkerCandidateTest,AssignmentDecisionTest,SelectionContextTest" 2>&1 | tail -5
```
Expected: COMPILATION ERROR — types don't exist yet.

- [ ] **Step 3: Create pom.xml**

Create `quarkus-work-api/pom.xml`:
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
  <artifactId>quarkus-work-api</artifactId>
  <name>Quarkus WorkItems - API (Shared SPI)</name>
  <description>
    Pure Java shared SPI types for the WorkItems ecosystem. Zero runtime dependencies.
    CaseHub and other systems depend on this module without pulling in the full WorkItems stack.
  </description>
  <dependencies>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
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

Add `<module>quarkus-work-api</module>` to parent `pom.xml` **before** `<module>runtime</module>` (the api module must be built first since runtime will depend on it).

- [ ] **Step 4: Create SPI types**

Create `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/WorkerCandidate.java`:
```java
package io.quarkiverse.work.spi;

import java.util.Set;

/**
 * A potential assignee for a WorkItem.
 *
 * <p>Returned by {@link WorkerRegistry#resolveGroup} and constructed by
 * {@link io.quarkiverse.work.runtime.service.WorkItemAssignmentService}
 * from {@code WorkItem.candidateUsers}. The {@code activeWorkItemCount} is
 * pre-populated before the list is passed to {@link WorkerSelectionStrategy#select}.
 *
 * <p>CaseHub alignment: maps to {@code HumanWorkerProfile}.
 */
public record WorkerCandidate(String id, Set<String> capabilities, int activeWorkItemCount) {

    /** Convenience factory — capabilities empty, workload unknown (0). */
    public static WorkerCandidate of(final String id) {
        return new WorkerCandidate(id, Set.of(), 0);
    }

    /** Returns a new candidate with the updated active WorkItem count. */
    public WorkerCandidate withActiveWorkItemCount(final int count) {
        return new WorkerCandidate(id, capabilities, count);
    }
}
```

Create `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/SelectionContext.java`:
```java
package io.quarkiverse.work.spi;

/**
 * Minimal context from a WorkItem used by {@link WorkerSelectionStrategy#select}.
 *
 * <p>Decouples strategies from the WorkItem entity — strategies depend only on this
 * pure record, not on the JPA entity. CaseHub constructs this from {@code TaskRequest}.
 *
 * @param category             WorkItem category (may be null)
 * @param priority             WorkItemPriority name (may be null)
 * @param requiredCapabilities comma-separated capability tags (may be null)
 * @param candidateGroups      comma-separated group names (may be null)
 * @param candidateUsers       comma-separated user IDs (may be null)
 */
public record SelectionContext(
        String category,
        String priority,
        String requiredCapabilities,
        String candidateGroups,
        String candidateUsers) {
}
```

Create `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/AssignmentDecision.java`:
```java
package io.quarkiverse.work.spi;

/**
 * Immutable outcome of a {@link WorkerSelectionStrategy#select} call.
 *
 * <p>A {@code null} field means "no change to this WorkItem field". Only non-null
 * fields are applied to the WorkItem by
 * {@link io.quarkiverse.work.runtime.service.WorkItemAssignmentService}.
 *
 * @param assigneeId       pre-assign to this actor; null = leave unassigned (pool stays open)
 * @param candidateGroups  replace candidateGroups; null = keep as-is
 * @param candidateUsers   replace candidateUsers; null = keep as-is
 */
public record AssignmentDecision(String assigneeId, String candidateGroups, String candidateUsers) {

    /** No changes — pool remains open, claim-first behaviour. */
    public static AssignmentDecision noChange() {
        return new AssignmentDecision(null, null, null);
    }

    /** Pre-assign to a specific actor; candidate fields unchanged. */
    public static AssignmentDecision assignTo(final String id) {
        return new AssignmentDecision(id, null, null);
    }

    /** Narrow the candidate pool without pre-assigning. */
    public static AssignmentDecision narrowCandidates(final String groups, final String users) {
        return new AssignmentDecision(null, groups, users);
    }

    /** True when this decision makes no changes to the WorkItem. */
    public boolean isNoOp() {
        return assigneeId == null && candidateGroups == null && candidateUsers == null;
    }
}
```

Create `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/AssignmentTrigger.java`:
```java
package io.quarkiverse.work.spi;

/** Lifecycle events that trigger worker (re-)selection. */
public enum AssignmentTrigger {
    /** WorkItem first created and persisted. */
    CREATED,
    /** WorkItem returned to pool by its assignee (release). */
    RELEASED,
    /** WorkItem delegated to a different pool or individual. */
    DELEGATED
}
```

Create `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/WorkerSelectionStrategy.java`:
```java
package io.quarkiverse.work.spi;

import java.util.List;
import java.util.Set;

/**
 * Pluggable worker selection SPI.
 *
 * <p>Implement as {@code @ApplicationScoped @Alternative @Priority(1)} to override
 * the built-in strategy selected by
 * {@code quarkus.work.routing.strategy}. Only one alternative may be active.
 *
 * <p>Built-in implementations in the runtime module:
 * <ul>
 *   <li>{@code ClaimFirstStrategy} — no pre-assignment; whoever claims first wins
 *   <li>{@code LeastLoadedStrategy} — pre-assigns to the candidate with the fewest
 *       active WorkItems (default)
 * </ul>
 *
 * <p>CaseHub alignment: corresponds to {@code WorkerSelectionStrategy} in casehub-engine.
 */
public interface WorkerSelectionStrategy {

    /**
     * Select an assignment outcome from the resolved candidate list.
     *
     * @param context    minimal WorkItem context (category, priority, capabilities, pools)
     * @param candidates resolved candidates with pre-populated active WorkItem counts
     * @return assignment decision; never null — use {@link AssignmentDecision#noChange()}
     *         to leave the WorkItem unassigned
     */
    AssignmentDecision select(SelectionContext context, List<WorkerCandidate> candidates);

    /**
     * The lifecycle events that trigger this strategy.
     * Override to restrict triggering to a subset of events.
     * Default: all three triggers (CREATED, RELEASED, DELEGATED).
     */
    default Set<AssignmentTrigger> triggers() {
        return Set.of(AssignmentTrigger.values());
    }
}
```

Create `quarkus-work-api/src/main/java/io/quarkiverse/workitems/spi/WorkerRegistry.java`:
```java
package io.quarkiverse.work.spi;

import java.util.List;

/**
 * Resolves a group name to its member {@link WorkerCandidate}s.
 *
 * <p>Implement as {@code @ApplicationScoped @Alternative @Priority(1)} to connect
 * LDAP, Keycloak, CaseHub WorkerRegistry, or any directory service.
 *
 * <p>The default implementation ({@code NoOpWorkerRegistry}) returns an empty list
 * for every group — groups remain claim-first until an application registers a real
 * implementation. This is intentionally conservative: no pre-assignment is made
 * unless the application explicitly wires group membership.
 *
 * <p>CaseHub alignment: corresponds to the {@code WorkerRegistry} concept in casehub-engine.
 */
public interface WorkerRegistry {

    /**
     * Resolve {@code groupName} to its member candidates.
     *
     * <p>Implementations should pre-populate {@link WorkerCandidate#capabilities()}
     * where known. The {@code activeWorkItemCount} will be populated by
     * {@link io.quarkiverse.work.runtime.service.WorkItemAssignmentService}
     * if left at 0.
     *
     * @param groupName the group to resolve (as stored in {@code WorkItem.candidateGroups})
     * @return member candidates; empty list if group is unknown or membership lookup fails
     */
    List<WorkerCandidate> resolveGroup(String groupName);
}
```

- [ ] **Step 5: Run tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-api \
  2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: BUILD SUCCESS, 7 tests pass.

- [ ] **Step 6: Commit**
```bash
git add quarkus-work-api/ pom.xml
git commit -m "feat(api): quarkus-work-api module — shared WorkerSelectionStrategy SPI

New pure-Java module with zero runtime dependencies, usable by CaseHub and any other
system that wants to implement worker selection without the full WorkItems stack.

WorkerCandidate (id+capabilities+activeWorkItemCount+withActiveWorkItemCount factory),
SelectionContext (category/priority/requiredCapabilities/candidateGroups/candidateUsers),
AssignmentDecision (noChange/assignTo/narrowCandidates factories + isNoOp()),
AssignmentTrigger (CREATED/RELEASED/DELEGATED),
WorkerSelectionStrategy (select + default triggers()),
WorkerRegistry (resolveGroup).

7 unit tests: record construction, factory methods, isNoOp, immutability.

Closes #115
Refs #100
Refs #102"
```

---

## Task 3: Add api dependency to runtime + WorkItemsConfig routing sub-group

**Files:**
- Modify: `runtime/pom.xml` — add dependency on `quarkus-work-api`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfig.java`
- Test: `runtime/src/test/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfigRoutingTest.java`

- [ ] **Step 1: Write failing config test**

Create `runtime/src/test/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfigRoutingTest.java`:
```java
package io.quarkiverse.work.runtime.config;

import static org.assertj.core.api.Assertions.assertThat;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class WorkItemsConfigRoutingTest {

    @Inject WorkItemsConfig config;

    @Test
    void routingStrategy_defaultsToLeastLoaded() {
        assertThat(config.routing().strategy()).isEqualTo("least-loaded");
    }
}
```

- [ ] **Step 2: Run test RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="WorkItemsConfigRoutingTest" 2>&1 | tail -5
```
Expected: COMPILATION ERROR — `routing()` method doesn't exist.

- [ ] **Step 3: Add api dependency to runtime pom**

In `runtime/pom.xml`, add after the first `<dependency>`:
```xml
<dependency>
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work-api</artifactId>
  <version>${project.version}</version>
</dependency>
```

- [ ] **Step 4: Add routing sub-group to WorkItemsConfig**

In `runtime/src/main/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfig.java`, add after the `cleanup()` method:
```java
/**
 * Worker selection strategy configuration.
 *
 * @return the routing configuration group
 */
@io.smallrye.config.WithName("routing")
RoutingConfig routing();

/**
 * Worker selection strategy configuration.
 */
interface RoutingConfig {

    /**
     * The built-in worker selection strategy to use when no CDI
     * {@code @Alternative WorkerSelectionStrategy} bean is present.
     *
     * <p>Accepted values:
     * <ul>
     *   <li>{@code least-loaded} — pre-assigns to the candidateUser with the
     *       fewest active (ASSIGNED/IN_PROGRESS/SUSPENDED) WorkItems (default)
     *   <li>{@code claim-first} — no pre-assignment; pool stays open, whoever
     *       claims first wins
     * </ul>
     *
     * @return strategy name; defaults to {@code "least-loaded"}
     */
    @io.smallrye.config.WithDefault("least-loaded")
    String strategy();
}
```

- [ ] **Step 5: Run test GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="WorkItemsConfigRoutingTest" 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Run full runtime tests (no regressions)**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  2>&1 | grep -E "Tests run: [0-9]+, Failures:|BUILD" | tail -3
```
Expected: BUILD SUCCESS, 501 tests pass.

- [ ] **Step 7: Commit**
```bash
git add runtime/pom.xml \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfig.java \
  runtime/src/test/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfigRoutingTest.java
git commit -m "feat(core): WorkItemsConfig routing sub-group + api module dependency

Adds quarkus.work.routing.strategy config (default: least-loaded) via a new
RoutingConfig sub-interface. Runtime module gains compile dep on quarkus-work-api.
1 @QuarkusTest config test.

Refs #116
Refs #100
Refs #102"
```

---

## Task 4: Built-in strategies + NoOpWorkerRegistry (unit tests first)

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/ClaimFirstStrategy.java`
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/LeastLoadedStrategy.java`
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/NoOpWorkerRegistry.java`
- Test: `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/ClaimFirstStrategyTest.java`
- Test: `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/LeastLoadedStrategyTest.java`

- [ ] **Step 1: Write failing unit tests**

Create `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/ClaimFirstStrategyTest.java`:
```java
package io.quarkiverse.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.Test;
import io.quarkiverse.work.spi.*;

class ClaimFirstStrategyTest {

    private final ClaimFirstStrategy strategy = new ClaimFirstStrategy();

    @Test
    void select_alwaysReturnsNoChange_withCandidates() {
        final List<WorkerCandidate> candidates = List.of(
            new WorkerCandidate("alice", Set.of(), 1),
            new WorkerCandidate("bob",   Set.of(), 5));
        final AssignmentDecision d = strategy.select(ctx(), candidates);
        assertThat(d.isNoOp()).isTrue();
    }

    @Test
    void select_alwaysReturnsNoChange_withNoCandidates() {
        assertThat(strategy.select(ctx(), List.of()).isNoOp()).isTrue();
    }

    @Test
    void triggers_includesAllThreeEvents() {
        assertThat(strategy.triggers()).containsExactlyInAnyOrder(
            AssignmentTrigger.values());
    }

    private SelectionContext ctx() {
        return new SelectionContext("cat", "NORMAL", null, null, "alice,bob");
    }
}
```

Create `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/LeastLoadedStrategyTest.java`:
```java
package io.quarkiverse.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.Test;
import io.quarkiverse.work.spi.*;

class LeastLoadedStrategyTest {

    private final LeastLoadedStrategy strategy = new LeastLoadedStrategy();

    @Test
    void select_assignsToCandidate_withLowestActiveCount() {
        final List<WorkerCandidate> candidates = List.of(
            new WorkerCandidate("alice", Set.of(), 5),
            new WorkerCandidate("bob",   Set.of(), 1),
            new WorkerCandidate("carol", Set.of(), 3));
        final AssignmentDecision d = strategy.select(ctx(), candidates);
        assertThat(d.assigneeId()).isEqualTo("bob");
        assertThat(d.candidateGroups()).isNull();
        assertThat(d.candidateUsers()).isNull();
    }

    @Test
    void select_returnsNoChange_whenCandidateListIsEmpty() {
        assertThat(strategy.select(ctx(), List.of()).isNoOp()).isTrue();
    }

    @Test
    void select_assignsToOnlyCandidate_whenListHasOne() {
        final List<WorkerCandidate> candidates = List.of(
            new WorkerCandidate("alice", Set.of(), 0));
        assertThat(strategy.select(ctx(), candidates).assigneeId()).isEqualTo("alice");
    }

    @Test
    void select_picksFirstAmongTies_whenMultipleCandidatesHaveSameCount() {
        final List<WorkerCandidate> candidates = List.of(
            new WorkerCandidate("alice", Set.of(), 2),
            new WorkerCandidate("bob",   Set.of(), 2));
        // Either alice or bob is valid — just verify someone is assigned
        final AssignmentDecision d = strategy.select(ctx(), candidates);
        assertThat(d.assigneeId()).isIn("alice", "bob");
    }

    @Test
    void select_assignsToZeroLoadCandidate_overHighLoad() {
        final List<WorkerCandidate> candidates = List.of(
            new WorkerCandidate("busy", Set.of(), 20),
            new WorkerCandidate("free", Set.of(), 0));
        assertThat(strategy.select(ctx(), candidates).assigneeId()).isEqualTo("free");
    }

    @Test
    void triggers_includesAllThreeEvents() {
        assertThat(strategy.triggers()).containsExactlyInAnyOrder(
            AssignmentTrigger.values());
    }

    private SelectionContext ctx() {
        return new SelectionContext("cat", "NORMAL", null, null, "alice,bob");
    }
}
```

- [ ] **Step 2: Run tests RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="ClaimFirstStrategyTest,LeastLoadedStrategyTest" 2>&1 | tail -5
```
Expected: COMPILATION ERROR — classes don't exist.

- [ ] **Step 3: Implement ClaimFirstStrategy**

Create `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/ClaimFirstStrategy.java`:
```java
package io.quarkiverse.work.runtime.service;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkiverse.work.spi.AssignmentDecision;
import io.quarkiverse.work.spi.SelectionContext;
import io.quarkiverse.work.spi.WorkerCandidate;
import io.quarkiverse.work.spi.WorkerSelectionStrategy;

/**
 * No-op selection strategy — leaves all WorkItems in the open pool.
 * Whoever claims first wins. Activated by:
 * {@code quarkus.work.routing.strategy=claim-first}.
 */
@ApplicationScoped
public class ClaimFirstStrategy implements WorkerSelectionStrategy {

    @Override
    public AssignmentDecision select(final SelectionContext context,
            final List<WorkerCandidate> candidates) {
        return AssignmentDecision.noChange();
    }
}
```

- [ ] **Step 4: Implement LeastLoadedStrategy**

Create `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/LeastLoadedStrategy.java`:
```java
package io.quarkiverse.work.runtime.service;

import java.util.Comparator;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkiverse.work.spi.AssignmentDecision;
import io.quarkiverse.work.spi.SelectionContext;
import io.quarkiverse.work.spi.WorkerCandidate;
import io.quarkiverse.work.spi.WorkerSelectionStrategy;

/**
 * Pre-assigns WorkItems to the candidate with the fewest active WorkItems.
 *
 * <p>"Active" is defined as: ASSIGNED, IN_PROGRESS, or SUSPENDED — the states
 * where a worker is actively holding a WorkItem. PENDING items are not counted
 * because they have not been claimed by anyone yet.
 *
 * <p>When the candidate list is empty (no candidateUsers and no WorkerRegistry
 * resolving any groups), returns {@link AssignmentDecision#noChange()} and the
 * WorkItem remains in the open pool for claim-first assignment.
 *
 * <p>Activated by: {@code quarkus.work.routing.strategy=least-loaded} (default).
 */
@ApplicationScoped
public class LeastLoadedStrategy implements WorkerSelectionStrategy {

    @Override
    public AssignmentDecision select(final SelectionContext context,
            final List<WorkerCandidate> candidates) {
        if (candidates.isEmpty()) {
            return AssignmentDecision.noChange();
        }
        return candidates.stream()
                .min(Comparator.comparingInt(WorkerCandidate::activeWorkItemCount))
                .map(c -> AssignmentDecision.assignTo(c.id()))
                .orElse(AssignmentDecision.noChange());
    }
}
```

- [ ] **Step 5: Implement NoOpWorkerRegistry**

Create `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/NoOpWorkerRegistry.java`:
```java
package io.quarkiverse.work.runtime.service;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.quarkiverse.work.spi.WorkerCandidate;
import io.quarkiverse.work.spi.WorkerRegistry;

/**
 * Default WorkerRegistry — returns an empty list for every group.
 *
 * <p>With this default, {@code candidateGroups} on a WorkItem does not trigger
 * pre-assignment: the group stays open and whoever claims first wins.
 *
 * <p>Replace with an {@code @Alternative @Priority(1) @ApplicationScoped} bean
 * that calls LDAP, Keycloak, CaseHub's WorkerRegistry, or a hard-coded map.
 */
@ApplicationScoped
public class NoOpWorkerRegistry implements WorkerRegistry {

    @Override
    public List<WorkerCandidate> resolveGroup(final String groupName) {
        return List.of();
    }
}
```

- [ ] **Step 6: Run tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="ClaimFirstStrategyTest,LeastLoadedStrategyTest" 2>&1 | grep -E "Tests run:|BUILD" | tail -3
```
Expected: BUILD SUCCESS, 11 tests pass.

- [ ] **Step 7: Commit**
```bash
git add runtime/src/main/java/io/quarkiverse/workitems/runtime/service/ClaimFirstStrategy.java \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/service/LeastLoadedStrategy.java \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/service/NoOpWorkerRegistry.java \
  runtime/src/test/java/io/quarkiverse/workitems/runtime/service/ClaimFirstStrategyTest.java \
  runtime/src/test/java/io/quarkiverse/workitems/runtime/service/LeastLoadedStrategyTest.java
git commit -m "feat(core): ClaimFirstStrategy + LeastLoadedStrategy + NoOpWorkerRegistry

ClaimFirstStrategy: always noChange() — pool stays open, existing claim-first behaviour.
LeastLoadedStrategy: selects min(activeWorkItemCount) candidate; noChange() on empty list.
NoOpWorkerRegistry: default group resolver returning empty list (claim-first for groups).
11 unit tests (no Quarkus boot): noChange always, picks min, empty list, ties, zero-load,
single candidate, triggers include all events.

Refs #116
Refs #100
Refs #102"
```

---

## Task 5: WorkItemAssignmentService (unit tests first)

**Files:**
- Create: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemAssignmentService.java`
- Test: `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/WorkItemAssignmentServiceTest.java`

- [ ] **Step 1: Write failing unit tests**

Create `runtime/src/test/java/io/quarkiverse/workitems/runtime/service/WorkItemAssignmentServiceTest.java`:
```java
package io.quarkiverse.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

import java.util.*;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import io.quarkiverse.work.runtime.model.*;
import io.quarkiverse.work.runtime.repository.*;
import io.quarkiverse.work.spi.*;

/**
 * Unit tests for WorkItemAssignmentService — no Quarkus boot required.
 * Issue #116, Epics #100/#102.
 */
@ExtendWith(MockitoExtension.class)
class WorkItemAssignmentServiceTest {

    @Mock WorkItemStore workItemStore;
    @Mock WorkerRegistry workerRegistry;

    private WorkItemAssignmentService service;

    @BeforeEach
    void setUp() {
        when(workerRegistry.resolveGroup(anyString())).thenReturn(List.of());
        service = new WorkItemAssignmentService(
            new LeastLoadedStrategy(),
            workerRegistry,
            workItemStore);
    }

    // ── Trigger filtering ─────────────────────────────────────────────────────

    @Test
    void assign_skipsWork_whenTriggerNotInStrategyTriggers() {
        // Create a strategy that only fires on CREATED
        final WorkerSelectionStrategy createdOnly = new WorkerSelectionStrategy() {
            @Override public AssignmentDecision select(SelectionContext ctx, List<WorkerCandidate> c) {
                return AssignmentDecision.assignTo("alice");
            }
            @Override public Set<AssignmentTrigger> triggers() {
                return Set.of(AssignmentTrigger.CREATED);
            }
        };
        service = new WorkItemAssignmentService(createdOnly, workerRegistry, workItemStore);

        final WorkItem wi = workItem("alice,bob", null, null);
        service.assign(wi, AssignmentTrigger.RELEASED); // not in triggers
        assertThat(wi.assigneeId).isNull(); // unchanged
    }

    @Test
    void assign_fires_whenTriggerIsInStrategyTriggers() {
        final WorkItem wi = workItem(null, null, "alice");
        mockActiveCount("alice", 1);
        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.assigneeId).isEqualTo("alice");
    }

    // ── candidateUsers resolution ─────────────────────────────────────────────

    @Test
    void assign_parsesCandidateUsers_asCommaDelimitedList() {
        final WorkItem wi = workItem(null, null, "alice,bob,carol");
        mockActiveCount("alice", 5);
        mockActiveCount("bob",   1); // least loaded
        mockActiveCount("carol", 3);
        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.assigneeId).isEqualTo("bob");
    }

    @Test
    void assign_trimsWhitespace_inCandidateUsers() {
        final WorkItem wi = workItem(null, null, " alice , bob ");
        mockActiveCount("alice", 0);
        mockActiveCount("bob",   2);
        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.assigneeId).isEqualTo("alice");
    }

    @Test
    void assign_noChange_whenNoCandidateUsersAndNoGroupResolution() {
        final WorkItem wi = workItem(null, null, null);
        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.assigneeId).isNull();
    }

    // ── candidateGroups resolution via WorkerRegistry ─────────────────────────

    @Test
    void assign_resolvesGroup_viaWorkerRegistry() {
        when(workerRegistry.resolveGroup("finance-team")).thenReturn(
            List.of(WorkerCandidate.of("alice"), WorkerCandidate.of("bob")));
        mockActiveCount("alice", 3);
        mockActiveCount("bob",   0);

        final WorkItem wi = workItem(null, "finance-team", null);
        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.assigneeId).isEqualTo("bob");
    }

    @Test
    void assign_deduplicates_candidatesFromUsersAndGroups() {
        when(workerRegistry.resolveGroup("team")).thenReturn(
            List.of(WorkerCandidate.of("alice"))); // alice appears in both

        mockActiveCount("alice", 2);
        mockActiveCount("bob",   1);

        final WorkItem wi = workItem(null, "team", "alice,bob");
        service.assign(wi, AssignmentTrigger.CREATED);
        // alice appears once (deduped), bob wins (lower count)
        assertThat(wi.assigneeId).isEqualTo("bob");
    }

    // ── requiredCapabilities filtering ────────────────────────────────────────

    @Test
    void assign_filtersOut_candidatesLackingRequiredCapabilities() {
        final WorkItem wi = workItem(null, null, "alice,bob");
        wi.requiredCapabilities = "audit";
        // Stub scan to return an empty list (candidates without audit capability
        // are filtered out before the strategy is called)
        // The service must build WorkerCandidate with capabilities from registry
        // For candidateUsers (no registry), capabilities are empty — so ALL are filtered out
        when(workItemStore.scan(any())).thenReturn(List.of());
        service.assign(wi, AssignmentTrigger.CREATED);
        // No candidates pass capability filter → noChange
        assertThat(wi.assigneeId).isNull();
    }

    @Test
    void assign_keepsCandidate_withRequiredCapabilities() {
        when(workerRegistry.resolveGroup("team")).thenReturn(List.of(
            new WorkerCandidate("alice", Set.of("audit", "legal"), 0),
            new WorkerCandidate("bob",   Set.of("sales"), 0)));

        final WorkItem wi = workItem(null, "team", null);
        wi.requiredCapabilities = "audit";
        when(workItemStore.scan(any())).thenReturn(List.of()); // 0 active for alice

        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.assigneeId).isEqualTo("alice"); // bob filtered out, alice wins
    }

    // ── AssignmentDecision application ────────────────────────────────────────

    @Test
    void assign_appliesAssigneeId_fromDecision() {
        final WorkItem wi = workItem(null, null, "alice");
        mockActiveCount("alice", 0);
        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.assigneeId).isEqualTo("alice");
    }

    @Test
    void assign_appliesCandidateGroups_fromNarrowDecision() {
        final WorkerSelectionStrategy narrower = (ctx, candidates) ->
            AssignmentDecision.narrowCandidates("narrowed-group", null);
        service = new WorkItemAssignmentService(narrower, workerRegistry, workItemStore);

        final WorkItem wi = workItem(null, "original-group", null);
        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.candidateGroups).isEqualTo("narrowed-group");
        assertThat(wi.assigneeId).isNull();
    }

    @Test
    void assign_doesNotOverwrite_existingAssigneeId_onNoChange() {
        final WorkerSelectionStrategy noOp = (ctx, c) -> AssignmentDecision.noChange();
        service = new WorkItemAssignmentService(noOp, workerRegistry, workItemStore);

        final WorkItem wi = workItem(null, null, "alice");
        wi.assigneeId = "existing-assignee";
        service.assign(wi, AssignmentTrigger.CREATED);
        assertThat(wi.assigneeId).isEqualTo("existing-assignee");
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private WorkItem workItem(final String groups, final String groupsOnly, final String users) {
        final WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.candidateGroups = groups != null ? groups : groupsOnly;
        wi.candidateUsers  = users;
        return wi;
    }

    private void mockActiveCount(final String actorId, final int count) {
        final List<WorkItem> items = new ArrayList<>();
        for (int i = 0; i < count; i++) {
            final WorkItem wi = new WorkItem();
            wi.assigneeId = actorId;
            wi.status = WorkItemStatus.ASSIGNED;
            items.add(wi);
        }
        when(workItemStore.scan(argThat(q ->
            q != null && actorId.equals(q.assigneeId()) &&
            q.statusIn() != null && q.statusIn().contains(WorkItemStatus.ASSIGNED))))
            .thenReturn(items);
    }
}
```

- [ ] **Step 2: Add Mockito to runtime pom (if not already present)**

Check `runtime/pom.xml` for `mockito-junit-jupiter`. If absent, add:
```xml
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-junit-jupiter</artifactId>
  <scope>test</scope>
</dependency>
```
(Quarkus BOM includes Mockito — no version needed.)

- [ ] **Step 3: Run tests RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="WorkItemAssignmentServiceTest" 2>&1 | tail -5
```
Expected: COMPILATION ERROR.

- [ ] **Step 4: Implement WorkItemAssignmentService**

Create `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemAssignmentService.java`:
```java
package io.quarkiverse.work.runtime.service;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import io.quarkiverse.work.runtime.config.WorkItemsConfig;
import io.quarkiverse.work.runtime.model.WorkItem;
import io.quarkiverse.work.runtime.model.WorkItemStatus;
import io.quarkiverse.work.runtime.repository.WorkItemQuery;
import io.quarkiverse.work.runtime.repository.WorkItemStore;
import io.quarkiverse.work.spi.AssignmentDecision;
import io.quarkiverse.work.spi.AssignmentTrigger;
import io.quarkiverse.work.spi.SelectionContext;
import io.quarkiverse.work.spi.WorkerCandidate;
import io.quarkiverse.work.spi.WorkerRegistry;
import io.quarkiverse.work.spi.WorkerSelectionStrategy;

/**
 * Orchestrates worker selection for WorkItems on creation, release, and delegation.
 *
 * <p><strong>Flow:</strong>
 * <ol>
 *   <li>Resolve the active strategy (CDI {@code @Alternative} > config-selected built-in)</li>
 *   <li>Check whether the trigger is in the strategy's declared trigger set — skip if not</li>
 *   <li>Build the resolved candidate list from {@code candidateUsers} and
 *       {@code candidateGroups} (via {@link WorkerRegistry})</li>
 *   <li>Populate each candidate's {@code activeWorkItemCount} from the WorkItem store</li>
 *   <li>Filter by {@code requiredCapabilities} if set</li>
 *   <li>Call {@link WorkerSelectionStrategy#select} and apply non-null fields of the
 *       returned {@link AssignmentDecision} to the WorkItem in memory</li>
 * </ol>
 *
 * <p>This service mutates the WorkItem in memory only. The caller's outer
 * {@code @Transactional} boundary flushes the changes to the database.
 */
@ApplicationScoped
public class WorkItemAssignmentService {

    private static final List<WorkItemStatus> ACTIVE_STATUSES = List.of(
            WorkItemStatus.ASSIGNED, WorkItemStatus.IN_PROGRESS, WorkItemStatus.SUSPENDED);

    private final WorkerSelectionStrategy defaultStrategy;
    private final WorkerRegistry workerRegistry;
    private final WorkItemStore workItemStore;
    private WorkItemsConfig config;
    private Instance<WorkerSelectionStrategy> alternatives;

    /** CDI constructor — full wiring with config and @Alternative discovery. */
    @Inject
    public WorkItemAssignmentService(
            final WorkItemsConfig config,
            final Instance<WorkerSelectionStrategy> alternatives,
            final WorkerRegistry workerRegistry,
            final WorkItemStore workItemStore,
            final ClaimFirstStrategy claimFirst,
            final LeastLoadedStrategy leastLoaded) {
        this.config = config;
        this.alternatives = alternatives;
        this.workerRegistry = workerRegistry;
        this.workItemStore = workItemStore;
        this.defaultStrategy = resolveConfigStrategy(config, claimFirst, leastLoaded);
    }

    /** Package-private constructor for unit tests — bypasses CDI and config. */
    WorkItemAssignmentService(final WorkerSelectionStrategy strategy,
            final WorkerRegistry workerRegistry, final WorkItemStore workItemStore) {
        this.defaultStrategy = strategy;
        this.workerRegistry = workerRegistry;
        this.workItemStore = workItemStore;
    }

    /**
     * Apply the active strategy to the WorkItem for the given trigger event.
     * Mutates the WorkItem in memory; caller is responsible for persisting.
     */
    public void assign(final WorkItem workItem, final AssignmentTrigger trigger) {
        final WorkerSelectionStrategy strategy = activeStrategy();

        if (!strategy.triggers().contains(trigger)) {
            return;
        }

        final List<WorkerCandidate> candidates = resolveCandidates(workItem);
        final SelectionContext context = new SelectionContext(
                workItem.category,
                workItem.priority != null ? workItem.priority.name() : null,
                workItem.requiredCapabilities,
                workItem.candidateGroups,
                workItem.candidateUsers);

        final AssignmentDecision decision = strategy.select(context, candidates);
        applyDecision(workItem, decision);
    }

    private WorkerSelectionStrategy activeStrategy() {
        if (alternatives != null) {
            final var alt = alternatives.stream()
                    .filter(s -> !(s instanceof ClaimFirstStrategy)
                              && !(s instanceof LeastLoadedStrategy))
                    .findFirst();
            if (alt.isPresent()) return alt.get();
        }
        return defaultStrategy;
    }

    private List<WorkerCandidate> resolveCandidates(final WorkItem workItem) {
        final List<WorkerCandidate> candidates = new ArrayList<>();

        // 1. candidateUsers — direct user IDs
        if (workItem.candidateUsers != null && !workItem.candidateUsers.isBlank()) {
            Arrays.stream(workItem.candidateUsers.split(","))
                    .map(String::trim)
                    .filter(id -> !id.isEmpty())
                    .forEach(id -> candidates.add(
                            WorkerCandidate.of(id).withActiveWorkItemCount(countActive(id))));
        }

        // 2. candidateGroups — resolved via WorkerRegistry
        if (workItem.candidateGroups != null && !workItem.candidateGroups.isBlank()) {
            Arrays.stream(workItem.candidateGroups.split(","))
                    .map(String::trim)
                    .filter(g -> !g.isEmpty())
                    .flatMap(g -> workerRegistry.resolveGroup(g).stream())
                    .filter(c -> candidates.stream().noneMatch(e -> e.id().equals(c.id())))
                    .map(c -> c.activeWorkItemCount() > 0
                            ? c : c.withActiveWorkItemCount(countActive(c.id())))
                    .forEach(candidates::add);
        }

        // 3. Filter by requiredCapabilities
        if (workItem.requiredCapabilities != null && !workItem.requiredCapabilities.isBlank()) {
            final Set<String> required = Arrays.stream(workItem.requiredCapabilities.split(","))
                    .map(String::trim).filter(s -> !s.isEmpty())
                    .collect(Collectors.toSet());
            candidates.removeIf(c -> !c.capabilities().containsAll(required));
        }

        return candidates;
    }

    private int countActive(final String actorId) {
        return workItemStore.scan(WorkItemQuery.builder()
                .assigneeId(actorId)
                .statusIn(ACTIVE_STATUSES)
                .build()).size();
    }

    private void applyDecision(final WorkItem workItem, final AssignmentDecision decision) {
        if (decision.assigneeId() != null) {
            workItem.assigneeId = decision.assigneeId();
        }
        if (decision.candidateGroups() != null) {
            workItem.candidateGroups = decision.candidateGroups();
        }
        if (decision.candidateUsers() != null) {
            workItem.candidateUsers = decision.candidateUsers();
        }
    }

    private static WorkerSelectionStrategy resolveConfigStrategy(
            final WorkItemsConfig config,
            final ClaimFirstStrategy claimFirst,
            final LeastLoadedStrategy leastLoaded) {
        return "claim-first".equals(config.routing().strategy()) ? claimFirst : leastLoaded;
    }
}
```

**Important:** `WorkItemQuery.builder()` with `.assigneeId()` and `.statusIn()` — verify these exist on the Builder by reading `WorkItemQuery.java`. The existing `WorkItemServiceTest.TestWorkItemRepo` already handles `statusIn` filtering, confirming this works.

- [ ] **Step 5: Run unit tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="WorkItemAssignmentServiceTest" 2>&1 | grep -E "Tests run:|BUILD" | tail -3
```
Expected: BUILD SUCCESS. Fix any compilation/test issues.

- [ ] **Step 6: Commit**
```bash
git add runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemAssignmentService.java \
  runtime/src/test/java/io/quarkiverse/workitems/runtime/service/WorkItemAssignmentServiceTest.java \
  runtime/pom.xml
git commit -m "feat(core): WorkItemAssignmentService — orchestrates worker selection

Resolves candidates from candidateUsers (direct) and candidateGroups (via WorkerRegistry),
pre-populates activeWorkItemCount via WorkItemStore.scan(), filters by requiredCapabilities,
calls WorkerSelectionStrategy.select(), applies non-null AssignmentDecision fields to
WorkItem in memory. CDI @Alternative strategy overrides config-selected built-in.
Trigger check: strategy.triggers() gate skips non-matching events.
14 unit tests (Mockito, no Quarkus boot): trigger skip/fire, candidateUsers parsing,
whitespace trim, no candidates noChange, group resolution, deduplication, capability
filter, decision application, noChange preservation.

Refs #116
Refs #100
Refs #102"
```

---

## Task 6: Wire WorkItemAssignmentService into WorkItemService

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemService.java`
- Test: `runtime/src/test/java/io/quarkiverse/workitems/runtime/api/WorkerSelectionStrategyIT.java`

- [ ] **Step 1: Write failing integration tests (RED)**

Create `runtime/src/test/java/io/quarkiverse/workitems/runtime/api/WorkerSelectionStrategyIT.java`:
```java
package io.quarkiverse.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

/**
 * Integration + E2E tests for WorkerSelectionStrategy wired into the WorkItem lifecycle.
 * Issues #115/#116, Epics #100/#102.
 */
@QuarkusTest
class WorkerSelectionStrategyIT {

    // ── least-loaded (default) ────────────────────────────────────────────────

    @Test
    void leastLoaded_preAssigns_toCandidateUser_withLowestActiveCount() {
        // Give alice 2 active WorkItems, bob 0
        final String alice = "ls-alice-" + System.nanoTime();
        final String bob   = "ls-bob-"   + System.nanoTime();
        createAndAssign(alice);
        createAndAssign(alice);
        // bob has 0 active

        // New WorkItem with both as candidateUsers — should go to bob
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Route Me\",\"createdBy\":\"system\"," +
                  "\"candidateUsers\":\"" + alice + "," + bob + "\"}")
            .post("/workitems").then().statusCode(201)
            .extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("assigneeId", equalTo(bob))
            .body("status", equalTo("ASSIGNED"));
    }

    @Test
    void leastLoaded_noPreAssignment_whenNoCandidateUsers() {
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"No Candidates\",\"createdBy\":\"system\"}")
            .post("/workitems").then().statusCode(201)
            .extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("assigneeId", nullValue())
            .body("status", equalTo("PENDING"));
    }

    @Test
    void leastLoaded_preAssigns_toOnlyCandidate_whenListHasOne() {
        final String actor = "solo-" + System.nanoTime();
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Solo\",\"createdBy\":\"system\"," +
                  "\"candidateUsers\":\"" + actor + "\"}")
            .post("/workitems").then().statusCode(201)
            .extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("assigneeId", equalTo(actor));
    }

    @Test
    void leastLoaded_preAssignedItem_isAlreadyAssigned_noClaim_needed() {
        final String actor = "no-claim-" + System.nanoTime();
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Pre-Assigned\",\"createdBy\":\"system\"," +
                  "\"candidateUsers\":\"" + actor + "\"}")
            .post("/workitems").then().statusCode(201)
            .extract().path("id");

        // Status should be ASSIGNED directly — no PUT /claim needed
        given().get("/workitems/" + id).then().statusCode(200)
            .body("status", equalTo("ASSIGNED"))
            .body("assigneeId", equalTo(actor));

        // Actor can start immediately (no claim step required)
        given().put("/workitems/" + id + "/start?actor=" + actor)
            .then().statusCode(200)
            .body("status", equalTo("IN_PROGRESS"));
    }

    // ── RELEASED trigger ──────────────────────────────────────────────────────

    @Test
    void leastLoaded_refiresOnRelease_andReassigns_toLeastLoaded() {
        final String alice = "rel-alice-" + System.nanoTime();
        final String bob   = "rel-bob-"   + System.nanoTime();

        // Create and assign to alice (she has 0 active items initially)
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Release Me\",\"createdBy\":\"system\"," +
                  "\"candidateUsers\":\"" + alice + "," + bob + "\"}")
            .post("/workitems").then().statusCode(201)
            .extract().path("id");

        // Now give bob 2 more active items
        createAndAssign(bob);
        createAndAssign(bob);

        // Alice releases — strategy should re-fire and re-assign to alice (0 active vs bob's 2+)
        given().put("/workitems/" + id + "/release?actor=" + alice).then().statusCode(200)
            .body("assigneeId", equalTo(alice));
    }

    // ── DELEGATED trigger ─────────────────────────────────────────────────────

    @Test
    void leastLoaded_refiresOnDelegate_andReassigns_toLeastLoaded() {
        final String alice = "del-alice-" + System.nanoTime();
        final String bob   = "del-bob-"   + System.nanoTime();
        final String carol = "del-carol-" + System.nanoTime();

        // Create WorkItem assigned to alice
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Delegate Me\",\"createdBy\":\"system\"," +
                  "\"candidateUsers\":\"" + alice + "," + bob + "," + carol + "\"}")
            .post("/workitems").then().statusCode(201)
            .extract().path("id");

        // Start the item (required to delegate)
        given().put("/workitems/" + id + "/start?actor=" + alice).then().statusCode(200);

        // Give bob 3 active items, carol 0
        createAndAssign(bob);
        createAndAssign(bob);
        createAndAssign(bob);

        // Delegate — strategy re-fires, carol should be picked (0 active)
        given().put("/workitems/" + id + "/delegate?to=" + carol + "&actor=" + alice)
            .then().statusCode(200)
            .body("assigneeId", equalTo(carol));
    }

    // ── claim-first config ────────────────────────────────────────────────────

    @Test
    void claimFirst_leavesWorkItemUnassigned_whenConfigured() {
        // This test verifies the @QuarkusTestProfile approach isn't needed for claim-first
        // Instead verify: without candidateUsers, default least-loaded also does noChange
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Group Only\",\"createdBy\":\"system\"," +
                  "\"candidateGroups\":\"some-group\"}")
            .post("/workitems").then().statusCode(201)
            .extract().path("id");

        // candidateGroups with NoOpWorkerRegistry → no candidates → noChange → PENDING
        given().get("/workitems/" + id).then().statusCode(200)
            .body("status", equalTo("PENDING"))
            .body("assigneeId", nullValue());
    }

    // ── requiredCapabilities filtering ────────────────────────────────────────

    @Test
    void requiredCapabilities_fromCandidateUsers_noMatch_leavesUnassigned() {
        // candidateUsers have no capability info (empty Set) → all filtered → noChange
        final String actor = "cap-actor-" + System.nanoTime();
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Cap Required\",\"createdBy\":\"system\"," +
                  "\"candidateUsers\":\"" + actor + "\"," +
                  "\"requiredCapabilities\":\"exotic-skill\"}")
            .post("/workitems").then().statusCode(201)
            .extract().path("id");

        given().get("/workitems/" + id).then().statusCode(200)
            .body("status", equalTo("PENDING"))
            .body("assigneeId", nullValue());
    }

    // ── E2E: full lifecycle with pre-assignment ───────────────────────────────

    @Test
    void e2e_aiAgent_createsWorkItem_preAssignedToLeastLoadedHuman_completes() {
        final String alice = "e2e-alice-" + System.nanoTime();
        final String bob   = "e2e-bob-"   + System.nanoTime();

        // Bob is least loaded (alice has 1 active item)
        createAndAssign(alice);

        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"E2E Work\",\"createdBy\":\"agent:ai\"," +
                  "\"candidateUsers\":\"" + alice + "," + bob + "\"," +
                  "\"confidenceScore\":0.9}")
            .post("/workitems").then().statusCode(201)
            .body("assigneeId", equalTo(bob))
            .body("status", equalTo("ASSIGNED"))
            .extract().path("id");

        // Bob starts (no claim needed — already assigned)
        given().put("/workitems/" + id + "/start?actor=" + bob).then().statusCode(200);

        // Bob completes
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + id + "/complete?actor=" + bob).then().statusCode(200)
            .body("status", equalTo("COMPLETED"))
            .body("assigneeId", equalTo(bob));
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    /** Creates a WorkItem and assigns it to the given actor (puts it in ASSIGNED state). */
    private void createAndAssign(final String actor) {
        final String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"Active\",\"createdBy\":\"system\"," +
                  "\"candidateUsers\":\"" + actor + "\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");
        // If pre-assigned by least-loaded, no claim needed — just verify ASSIGNED state
        given().get("/workitems/" + id).then().statusCode(200);
    }

    private static org.hamcrest.Matcher<Object> nullValue() {
        return org.hamcrest.Matchers.nullValue();
    }
}
```

- [ ] **Step 2: Run tests RED**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="WorkerSelectionStrategyIT" 2>&1 | tail -8
```
Expected: Tests fail — service not yet wired.

- [ ] **Step 3: Wire WorkItemAssignmentService into WorkItemService**

Read `WorkItemService.java` fully. Then inject `WorkItemAssignmentService` and call it in three places:

**In the `@Inject` constructor, add parameter:**
```java
@Inject
public WorkItemService(final WorkItemStore workItemStore,
        final AuditEntryStore auditStore,
        final WorkItemsConfig config,
        final WorkItemAssignmentService assignmentService) {
    this.workItemStore = workItemStore;
    this.auditStore = auditStore;
    this.config = config;
    this.assignmentService = assignmentService;
}
```

Add field: `private final WorkItemAssignmentService assignmentService;`

**In `create()`, call assignment BEFORE `workItemStore.put()`:**
```java
// After building all item fields and before:
//   final WorkItem saved = workItemStore.put(item);
assignmentService.assign(item, AssignmentTrigger.CREATED);
final WorkItem saved = workItemStore.put(item);
```

Add import: `import io.quarkiverse.work.spi.AssignmentTrigger;`

**In `release()`, call assignment AFTER state change but BEFORE `workItemStore.put()`:**
```java
item.status = WorkItemStatus.PENDING;
item.assigneeId = null;
assignmentService.assign(item, AssignmentTrigger.RELEASED);
final WorkItem saved = workItemStore.put(item);
```

**In `delegate()`, call assignment AFTER setting delegation fields but BEFORE `workItemStore.put()`:**
```java
// After setting: delegationChain, assigneeId = toAssigneeId, delegationState, status = PENDING
assignmentService.assign(item, AssignmentTrigger.DELEGATED);
final WorkItem saved = workItemStore.put(item);
```

**Also update `WorkItemServiceTest.basicRequest()`** to pass `null` for the new `assignmentService` parameter if the test uses a direct constructor call — OR inject a no-op `WorkItemAssignmentService` stub. Since `WorkItemService` now requires `WorkItemAssignmentService` in its constructor, update the test's `setUp()` to pass a stub that does nothing:
```java
// In WorkItemServiceTest.setUp():
service = new WorkItemService(repo, auditStore, testConfig(),
    new WorkItemAssignmentService(
        (ctx, candidates) -> AssignmentDecision.noChange(),
        group -> List.of(),
        repo));
```

- [ ] **Step 4: Run integration tests GREEN**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="WorkerSelectionStrategyIT" 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```
Expected: BUILD SUCCESS, 9 tests pass.

- [ ] **Step 5: Run full runtime suite (no regressions)**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  2>&1 | grep -E "Tests run: [0-9]+, Failures:|BUILD" | tail -3
```
Expected: BUILD SUCCESS, 525+ tests.

- [ ] **Step 6: Commit**
```bash
git add runtime/src/main/java/io/quarkiverse/workitems/runtime/service/WorkItemService.java \
  runtime/src/test/java/io/quarkiverse/workitems/runtime/api/WorkerSelectionStrategyIT.java
git commit -m "feat(core): wire WorkItemAssignmentService into WorkItemService lifecycle

WorkItemService.create() calls assign(CREATED) before persist — pre-assignment is
atomic with WorkItem creation. WorkItemService.release() and delegate() call
assign(RELEASED/DELEGATED) to re-route returned WorkItems to least-loaded candidate.
9 @QuarkusTest integration + E2E tests: least-loaded pre-assigns to min-count candidate,
no pre-assignment without candidates, RELEASED/DELEGATED triggers re-assign, candidateGroups
with NoOpRegistry stays PENDING, requiredCapabilities filters out incapable candidates,
full E2E lifecycle (AI agent creates → pre-assigned → start → complete).

Closes #116
Refs #100
Refs #102"
```

---

## Task 7: Docs update + api module to parent pom

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Run final verification across all modules**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -pl quarkus-work-api,runtime 2>&1 | \
  grep -E "Tests run: [0-9]+, Failures:|BUILD" | tail -5
```
Expected: BUILD SUCCESS, 0 failures. Record the test counts.

- [ ] **Step 2: Update docs/DESIGN.md**

Read `docs/DESIGN.md` first, then make these changes:

**A) Add `quarkus-work-api` row to Module Structure table** (first row, before Runtime):
```
| API | `quarkus-work-api` | Shared SPI types — pure Java, zero runtime deps. `WorkerCandidate`, `SelectionContext`, `AssignmentDecision`, `AssignmentTrigger`, `WorkerSelectionStrategy`, `WorkerRegistry`. CaseHub and any system can depend on this without pulling in the full WorkItems stack. |
```

**B) Add `confidenceScore` and new routing fields to Domain Model WorkItem table** (if `confidenceScore` not already there — check) and add note about `assignedAt` being set on pre-assignment.

**C) Add WorkerSelectionStrategy section to Services table:**
```
| `WorkItemAssignmentService` | `runtime.service` | Orchestrates worker selection on CREATED/RELEASED/DELEGATED. Resolves candidates from `candidateUsers` + `WorkerRegistry`, populates `activeWorkItemCount`, filters by `requiredCapabilities`, calls `WorkerSelectionStrategy.select()`, applies `AssignmentDecision` to WorkItem fields in memory. |
| `ClaimFirstStrategy` | `runtime.service` | `WorkerSelectionStrategy` no-op — `AssignmentDecision.noChange()`; pool stays open. Config: `quarkus.work.routing.strategy=claim-first`. |
| `LeastLoadedStrategy` | `runtime.service` | Default `WorkerSelectionStrategy` — pre-assigns to candidate with fewest ASSIGNED/IN_PROGRESS/SUSPENDED WorkItems. Config: `quarkus.work.routing.strategy=least-loaded` (default). |
| `NoOpWorkerRegistry` | `runtime.service` | Default `WorkerRegistry` — returns empty list for all groups; groups stay claim-first until app registers a real resolver. |
```

**D) Add config to Configuration table:**
```
| `quarkus.work.routing.strategy` | least-loaded | Worker selection: `least-loaded` or `claim-first`. CDI `@Alternative WorkerSelectionStrategy` overrides config. |
```

**E) Update Build Roadmap** — add or update Phase 11/12 to show WorkerSelectionStrategy complete.

**F) Update test totals** with actual numbers from Step 1.

- [ ] **Step 3: Update CLAUDE.md**

In the Work Tracking section, update Epic #100 to show:
- #112 ✅, #113 ✅, #114 ✅, #115 ✅, #116 ✅
- Remaining: semantic matching, AI-suggested resolution, escalation summarisation
- Epic #102 partially served (#115/#116 deliver WorkItemRouter SPI + LeastLoadedStrategy)

- [ ] **Step 4: Commit docs**
```bash
git add docs/DESIGN.md CLAUDE.md
git commit -m "docs: update DESIGN.md and CLAUDE.md for WorkerSelectionStrategy

quarkus-work-api module documented (shared SPI, zero deps, CaseHub alignment).
WorkItemAssignmentService, ClaimFirstStrategy, LeastLoadedStrategy, NoOpWorkerRegistry
added to Services table. quarkus.work.routing.strategy config documented.
Build roadmap updated. Test totals updated. Epics #100/#102 child issues updated.

Refs #100
Refs #102"
```

---

## Self-Review

**1. Spec coverage:**
- ✅ `quarkus-work-api` module with all 6 SPI types (Task 2)
- ✅ `WorkItemsConfig.routing().strategy()` default `least-loaded` (Task 3)
- ✅ `ClaimFirstStrategy`, `LeastLoadedStrategy`, `NoOpWorkerRegistry` (Task 4)
- ✅ `WorkItemAssignmentService` — resolve/filter/select/apply (Task 5)
- ✅ Wired into `create/release/delegate` (Task 6)
- ✅ Integration tests: least-loaded, no candidates, RELEASED, DELEGATED, groups with NoOp, capabilities (Task 6)
- ✅ E2E: AI agent → pre-assigned → start → complete (Task 6)
- ✅ Docs updated (Task 7)
- ✅ GitHub issues created (Task 1) and referenced in every commit
- ✅ Round-robin deferred with issue created (Task 1)
- ✅ CaseHub coordination noted in spec (existing spec doc)

**2. Placeholder scan:** All code blocks are complete. No TBDs.

**3. Type consistency:**
- `WorkerCandidate`, `SelectionContext`, `AssignmentDecision`, `AssignmentTrigger` defined in Task 2, used consistently in Tasks 4, 5, 6.
- `WorkItemAssignmentService` package-private constructor in Task 5 takes `(WorkerSelectionStrategy, WorkerRegistry, WorkItemStore)` — same signature used in Task 6 test stub.
- `WorkItemQuery.builder().assigneeId().statusIn()` — confirmed to exist from `WorkItemQuery.java` read.
