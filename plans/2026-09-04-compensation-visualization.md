# Compensation Visualization APIs — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #390 — saga: compensation visualization (design-time + runtime)
**Issue group:** #238 (saga compensation epic)

**Goal:** Expose compensation graph, saga timeline, and ledger chain data via GraphQL for claudony dashboard consumption.

**Architecture:** Three GraphQL additions in `engine-graphql` module: (1) `compensationGraph` field on `CaseDefinitionType` computed from `Binding.compensateRef`/`compensation`, (2) `compensationTimeline(caseId)` query joining PlanItemStore + EventLog, (3) `compensationChain(caseId)` query filtering ledger entries by `CompensationSupplement`. All computation inline in `engine-graphql` — no new modules.

**Tech Stack:** Java 21, SmallRye GraphQL, Quarkus, JUnit 5, Mockito, AssertJ

## Global Constraints

- Java 21 source level (on Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build/test target module: `mvn test -pl graphql` or `mvn test -pl api`
- `BindingTarget` is sealed: `permits CapabilityTarget, SubCaseTarget, JudgmentTarget, SignalTarget, ExtensionTarget`
- GraphQL DTOs use `@Type("Name")` annotation from `org.eclipse.microprofile.graphql`
- Test pattern: direct field injection (no CDI), Mockito mocks, AssertJ assertions
- All commits: `Refs #390 Refs #238`
- Slot engine repo: `/Users/mdproctor/claude/casehub/slots/169/engine`

---

## Batch 1: Design-time compensation graph

### Task 1: CompensationGraphProjection + CaseDefinitionType enrichment

**Files:**
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationGraphType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationNodeType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationEdgeType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/CompensationGraphProjection.java`
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/dto/CaseDefinitionType.java`
- Create: `graphql/src/test/java/io/casehub/engine/graphql/CompensationGraphProjectionTest.java`
- Test: `graphql/src/test/java/io/casehub/engine/graphql/CompensationGraphProjectionTest.java`

**Interfaces:**
- Consumes: `Binding.getName()`, `Binding.getCompensateRef()`, `Binding.isCompensation()`, `Binding.target()` (from engine-api)
- Produces: `CompensationGraphProjection.project(List<Binding>)` → `CompensationGraphType`; `CaseDefinitionType.compensationGraph()` field

- [ ] **Step 1: Write the failing test for CompensationGraphProjection**

Create `graphql/src/test/java/io/casehub/engine/graphql/CompensationGraphProjectionTest.java`:

```java
package io.casehub.engine.graphql;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CapabilityTarget;
import io.casehub.api.model.JudgmentTarget;
import io.casehub.api.model.Trigger;
import io.casehub.engine.graphql.dto.CompensationGraphType;
import io.casehub.worker.api.Capability;
import java.util.List;
import org.junit.jupiter.api.Test;

class CompensationGraphProjectionTest {

  @Test
  void projectFullCoverage() {
    Binding forward = Binding.builder()
        .name("irb-review")
        .judgment(JudgmentTarget.builder().title("IRB Review").scope("irb").build())
        .on(Trigger.onActivation())
        .compensateRef("irb-reversal")
        .build();
    Binding compensating = Binding.builder()
        .name("irb-reversal")
        .judgment(JudgmentTarget.builder().title("Reverse IRB").scope("irb-rev").build())
        .on(Trigger.onActivation())
        .compensation(true)
        .build();

    CompensationGraphType graph = CompensationGraphProjection.project(List.of(forward, compensating));

    assertThat(graph.nodes()).hasSize(2);
    assertThat(graph.edges()).hasSize(1);
    assertThat(graph.edges().get(0).sourceBinding()).isEqualTo("irb-review");
    assertThat(graph.edges().get(0).compensatingBinding()).isEqualTo("irb-reversal");
    assertThat(graph.gaps()).isEmpty();
  }

  @Test
  void projectDetectsGaps() {
    Binding withoutCompensation = Binding.builder()
        .name("data-export")
        .capability(Capability.of("exportService.export"))
        .on(Trigger.onActivation())
        .build();

    CompensationGraphType graph = CompensationGraphProjection.project(List.of(withoutCompensation));

    assertThat(graph.nodes()).hasSize(1);
    assertThat(graph.nodes().get(0).targetType()).isEqualTo("capability");
    assertThat(graph.edges()).isEmpty();
    assertThat(graph.gaps()).containsExactly("data-export");
  }

  @Test
  void projectCompensationOnlyBindingNotAGap() {
    Binding compensatingOnly = Binding.builder()
        .name("cleanup")
        .capability(Capability.of("cleanupService.run"))
        .on(Trigger.onActivation())
        .compensation(true)
        .build();
    Binding forward = Binding.builder()
        .name("process")
        .capability(Capability.of("processService.run"))
        .on(Trigger.onActivation())
        .compensateRef("cleanup")
        .build();

    CompensationGraphType graph = CompensationGraphProjection.project(List.of(compensatingOnly, forward));

    assertThat(graph.gaps()).isEmpty();
    assertThat(graph.nodes()).extracting("compensationOnly").containsExactly(true, false);
  }

  @Test
  void projectEmptyBindings() {
    CompensationGraphType graph = CompensationGraphProjection.project(List.of());

    assertThat(graph.nodes()).isEmpty();
    assertThat(graph.edges()).isEmpty();
    assertThat(graph.gaps()).isEmpty();
  }

  @Test
  void projectTargetTypeMapping() {
    Binding cap = Binding.builder().name("a").capability(Capability.of("x")).on(Trigger.onActivation()).build();
    Binding jt = Binding.builder().name("b").judgment(JudgmentTarget.builder().title("t").scope("s").build()).on(Trigger.onActivation()).build();

    CompensationGraphType graph = CompensationGraphProjection.project(List.of(cap, jt));

    assertThat(graph.nodes()).extracting("targetType").containsExactly("capability", "judgment");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=CompensationGraphProjectionTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5`
Expected: compilation error — `CompensationGraphProjection` and DTO classes don't exist yet

- [ ] **Step 3: Create GraphQL DTO records**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationNodeType.java`:

```java
package io.casehub.engine.graphql.dto;

import org.eclipse.microprofile.graphql.Type;

@Type("CompensationNode")
public record CompensationNodeType(
    String bindingName,
    String targetType,
    boolean compensationOnly) {}
```

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationEdgeType.java`:

```java
package io.casehub.engine.graphql.dto;

import org.eclipse.microprofile.graphql.Type;

@Type("CompensationEdge")
public record CompensationEdgeType(
    String sourceBinding,
    String compensatingBinding) {}
```

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationGraphType.java`:

```java
package io.casehub.engine.graphql.dto;

import java.util.List;
import org.eclipse.microprofile.graphql.Type;

@Type("CompensationGraph")
public record CompensationGraphType(
    List<CompensationNodeType> nodes,
    List<CompensationEdgeType> edges,
    List<String> gaps) {}
```

- [ ] **Step 4: Implement CompensationGraphProjection**

Create `graphql/src/main/java/io/casehub/engine/graphql/CompensationGraphProjection.java`:

```java
package io.casehub.engine.graphql;

import io.casehub.api.model.Binding;
import io.casehub.api.model.BindingTarget;
import io.casehub.api.model.CapabilityTarget;
import io.casehub.api.model.ExtensionTarget;
import io.casehub.api.model.JudgmentTarget;
import io.casehub.api.model.SignalTarget;
import io.casehub.api.model.SubCaseTarget;
import io.casehub.engine.graphql.dto.CompensationEdgeType;
import io.casehub.engine.graphql.dto.CompensationGraphType;
import io.casehub.engine.graphql.dto.CompensationNodeType;
import java.util.ArrayList;
import java.util.List;

public final class CompensationGraphProjection {

  private CompensationGraphProjection() {}

  public static CompensationGraphType project(List<Binding> bindings) {
    List<CompensationNodeType> nodes = new ArrayList<>();
    List<CompensationEdgeType> edges = new ArrayList<>();
    List<String> gaps = new ArrayList<>();

    for (Binding b : bindings) {
      nodes.add(new CompensationNodeType(
          b.getName(),
          targetTypeName(b.target()),
          b.isCompensation()));

      if (b.getCompensateRef() != null) {
        edges.add(new CompensationEdgeType(b.getName(), b.getCompensateRef()));
      } else if (!b.isCompensation()) {
        gaps.add(b.getName());
      }
    }
    return new CompensationGraphType(nodes, edges, gaps);
  }

  static String targetTypeName(BindingTarget target) {
    return switch (target) {
      case CapabilityTarget ignored -> "capability";
      case JudgmentTarget ignored -> "judgment";
      case SubCaseTarget ignored -> "sub-case";
      case SignalTarget ignored -> "signal";
      case ExtensionTarget ignored -> "extension";
    };
  }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=CompensationGraphProjectionTest 2>&1 | tail -5`
Expected: all 5 tests PASS

- [ ] **Step 6: Enrich CaseDefinitionType with compensationGraph field**

Modify `CaseDefinitionType.java` — add `compensationGraph` field to record and update `from()`:

```java
@Type("CaseDefinitionResponse")
public record CaseDefinitionType(
    String namespace,
    String name,
    String version,
    String title,
    String summary,
    List<String> capabilities,
    CompensationGraphType compensationGraph) {

  public static CaseDefinitionType from(CaseDefinition def) {
    List<String> capNames =
        def.getCapabilities() != null
            ? def.getCapabilities().stream().map(c -> c.name()).toList()
            : List.of();
    CompensationGraphType graph = def.getBindings() != null && !def.getBindings().isEmpty()
        ? CompensationGraphProjection.project(def.getBindings())
        : null;
    return new CaseDefinitionType(
        def.getNamespace(),
        def.getName(),
        def.getVersion(),
        def.getTitle(),
        def.getSummary(),
        capNames,
        graph);
  }
}
```

- [ ] **Step 7: Run full graphql module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql 2>&1 | tail -10`
Expected: all existing tests + CompensationGraphProjectionTest pass. Check that `CaseQueryResolverTest` and `EngineModelEnricherTest` still pass — they construct `CaseDefinitionType` and may need `compensationGraph` parameter added.

Fix any callers of `CaseDefinitionType` constructor that break due to the new field. Existing `CaseDefinitionType.from()` calls are fine — they go through the factory. Direct constructor calls in tests need the extra `null` argument.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/169/engine add graphql/src/main/java/io/casehub/engine/graphql/CompensationGraphProjection.java graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationGraphType.java graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationNodeType.java graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationEdgeType.java graphql/src/main/java/io/casehub/engine/graphql/dto/CaseDefinitionType.java graphql/src/test/java/io/casehub/engine/graphql/CompensationGraphProjectionTest.java
git -C /Users/mdproctor/claude/casehub/slots/169/engine commit -m "feat(#390): design-time compensation graph — GraphQL projection on CaseDefinitionType Refs #390 Refs #238"
```

---

## Batch 2: Runtime compensation timeline

### Task 2: CompensationTimeline query

**Files:**
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationTimelineType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/TimelineStepType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationStepType.java`
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java` (add query + injections)
- Create: `graphql/src/test/java/io/casehub/engine/graphql/CompensationTimelineQueryTest.java`
- Test: `graphql/src/test/java/io/casehub/engine/graphql/CompensationTimelineQueryTest.java`

**Interfaces:**
- Consumes: `CaseInstanceRepository.findByUuid()`, `PlanItemStore.findByCaseId()`, `CaseHubRuntime.eventLog()`, `CaseDefinitionRegistry.getCaseDefinition()`, `CaseDefinitionRegistry.findByIdentity()` (all already injected in CaseQueryResolver)
- Produces: `CaseQueryResolver.compensationTimeline(UUID caseId)` → `CompensationTimelineType`

- [ ] **Step 1: Write the failing test**

Create `graphql/src/test/java/io/casehub/engine/graphql/CompensationTimelineQueryTest.java`:

```java
package io.casehub.engine.graphql;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.JudgmentTarget;
import io.casehub.api.model.TaskStatus;
import io.casehub.api.model.Trigger;
import io.casehub.api.model.event.CaseEventLogRecord;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.TargetType;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.platform.api.identity.CurrentPrincipal;
import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CompensationTimelineQueryTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  private CaseQueryResolver resolver;
  private CaseInstanceRepository instanceRepository;
  private CaseDefinitionRegistry definitionRegistry;
  private CaseHubRuntime runtime;
  private PlanItemStore planItemStore;
  private CurrentPrincipal currentPrincipal;

  @BeforeEach
  void setUp() {
    resolver = new CaseQueryResolver();
    instanceRepository = mock(CaseInstanceRepository.class);
    definitionRegistry = mock(CaseDefinitionRegistry.class);
    runtime = mock(CaseHubRuntime.class);
    planItemStore = mock(PlanItemStore.class);
    currentPrincipal = mock(CurrentPrincipal.class);

    resolver.instanceRepository = instanceRepository;
    resolver.definitionRegistry = definitionRegistry;
    resolver.runtime = runtime;
    resolver.planItemStore = planItemStore;
    resolver.currentPrincipal = currentPrincipal;

    when(currentPrincipal.tenancyId()).thenReturn("test-tenant");
  }

  @Test
  void compensationTimelineReturnsNullForNonCompensatingCase() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = createInstance(caseId, CaseStatus.COMPLETED);
    when(instanceRepository.findByUuid(caseId, "test-tenant")).thenReturn(instance);
    when(runtime.eventLog(eq(caseId), any())).thenReturn(List.of());

    var result = resolver.compensationTimeline(caseId);

    assertThat(result).isNull();
  }

  @Test
  void compensationTimelineReturnsTimelineForCompensatingCase() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = createInstance(caseId, CaseStatus.COMPENSATING);
    when(instanceRepository.findByUuid(caseId, "test-tenant")).thenReturn(instance);

    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("case").version("1.0")
        .binding(Binding.builder().name("review")
            .judgment(JudgmentTarget.builder().title("t").scope("s").build())
            .on(Trigger.onActivation()).compensateRef("undo-review").build())
        .binding(Binding.builder().name("undo-review")
            .judgment(JudgmentTarget.builder().title("u").scope("s").build())
            .on(Trigger.onActivation()).compensation(true).build())
        .build();
    CaseMetaModel meta = instance.getCaseMetaModel();
    when(definitionRegistry.findByIdentity(meta.getNamespace(), meta.getName(), meta.getVersion()))
        .thenReturn(Optional.of(meta));
    when(definitionRegistry.getCaseDefinition(meta)).thenReturn(def);

    Instant now = Instant.now();
    PlanItemRecord forwardItem = PlanItemRecord.primitive(
        caseId, "pi-1", "review", TaskStatus.COMPLETED,
        now.minusSeconds(300), TargetType.JUDGMENT, null, "test-tenant",
        "Review", null, null);
    PlanItemRecord compensationItem = PlanItemRecord.primitive(
        caseId, "pi-2", "undo-review", TaskStatus.RUNNING,
        now.minusSeconds(10), TargetType.JUDGMENT, null, "test-tenant",
        "Undo review", null, null);
    when(planItemStore.findByCaseId(caseId, "test-tenant"))
        .thenReturn(List.of(forwardItem, compensationItem));

    ObjectNode startedMeta = MAPPER.createObjectNode();
    startedMeta.put("triggeredBy", "operator");
    startedMeta.put("reason", "Clinical trial withdrawn");
    CaseEventLogRecord startedEvent = new CaseEventLogRecord(
        CaseHubEventType.COMPENSATION_STARTED, EventStreamType.CASE,
        now.minusSeconds(30), startedMeta);
    when(runtime.eventLog(eq(caseId), eq(Set.of(
        CaseHubEventType.COMPENSATION_STARTED,
        CaseHubEventType.COMPENSATION_COMPLETED,
        CaseHubEventType.COMPENSATION_FAULTED))))
        .thenReturn(List.of(startedEvent));

    var result = resolver.compensationTimeline(caseId);

    assertThat(result).isNotNull();
    assertThat(result.caseId()).isEqualTo(caseId);
    assertThat(result.status()).isEqualTo("COMPENSATING");
    assertThat(result.triggeredBy()).isEqualTo("operator");
    assertThat(result.reason()).isEqualTo("Clinical trial withdrawn");
    assertThat(result.forwardSteps()).hasSize(1);
    assertThat(result.forwardSteps().get(0).bindingName()).isEqualTo("review");
    assertThat(result.compensationSteps()).hasSize(1);
    assertThat(result.compensationSteps().get(0).bindingName()).isEqualTo("undo-review");
    assertThat(result.compensationSteps().get(0).compensatesBinding()).isEqualTo("review");
  }

  private CaseInstance createInstance(UUID caseId, CaseStatus status) {
    CaseMetaModel meta = new CaseMetaModel("test", "case", "1.0");
    CaseInstance instance = new CaseInstance(meta, caseId);
    instance.setState(status);
    instance.tenancyId = "test-tenant";
    return instance;
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=CompensationTimelineQueryTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5`
Expected: compilation error — DTOs and `compensationTimeline` method don't exist

- [ ] **Step 3: Create timeline DTO records**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/TimelineStepType.java`:

```java
package io.casehub.engine.graphql.dto;

import java.time.Instant;
import org.eclipse.microprofile.graphql.Type;

@Type("TimelineStep")
public record TimelineStepType(
    String planItemId,
    String bindingName,
    String targetType,
    String status,
    Instant createdAt,
    Instant completedAt) {}
```

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationStepType.java`:

```java
package io.casehub.engine.graphql.dto;

import java.time.Instant;
import org.eclipse.microprofile.graphql.Type;

@Type("CompensationStep")
public record CompensationStepType(
    String planItemId,
    String bindingName,
    String targetType,
    String status,
    Instant createdAt,
    Instant completedAt,
    String compensatesBinding,
    String compensatesItemId) {}
```

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationTimelineType.java`:

```java
package io.casehub.engine.graphql.dto;

import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.eclipse.microprofile.graphql.Type;

@Type("CompensationTimeline")
public record CompensationTimelineType(
    UUID caseId,
    String status,
    String triggeredBy,
    String reason,
    Instant compensationStartedAt,
    Instant compensationCompletedAt,
    List<TimelineStepType> forwardSteps,
    List<CompensationStepType> compensationSteps) {}
```

- [ ] **Step 4: Implement compensationTimeline query in CaseQueryResolver**

Add to `CaseQueryResolver.java` — new query method. The resolver already injects `instanceRepository`, `definitionRegistry`, `runtime`, `planItemStore`, `currentPrincipal`.

```java
@Query
@Description("Compensation timeline for a case — forward steps and compensation steps with saga status")
public CompensationTimelineType compensationTimeline(UUID caseId) {
  String tenancyId = currentPrincipal.tenancyId();
  CaseInstance instance = instanceRepository.findByUuid(caseId, tenancyId);
  if (instance == null) return null;

  // Check for compensation events — if none exist, no timeline to show
  var compensationEvents = runtime.eventLog(caseId, Set.of(
      CaseHubEventType.COMPENSATION_STARTED,
      CaseHubEventType.COMPENSATION_COMPLETED,
      CaseHubEventType.COMPENSATION_FAULTED));
  if (compensationEvents.isEmpty()) return null;

  // Extract metadata from COMPENSATION_STARTED event
  String triggeredBy = null;
  String reason = null;
  Instant compensationStartedAt = null;
  Instant compensationCompletedAt = null;
  for (var event : compensationEvents) {
    if (event.eventType() == CaseHubEventType.COMPENSATION_STARTED) {
      compensationStartedAt = event.timestamp();
      if (event.payload() != null) {
        var payload = event.payload();
        if (payload.has("triggeredBy")) triggeredBy = payload.get("triggeredBy").asText();
        if (payload.has("reason")) reason = payload.get("reason").asText();
      }
    } else if (event.eventType() == CaseHubEventType.COMPENSATION_COMPLETED
        || event.eventType() == CaseHubEventType.COMPENSATION_FAULTED) {
      compensationCompletedAt = event.timestamp();
    }
  }

  // Load case definition to identify compensation bindings
  CaseMetaModel meta = instance.getCaseMetaModel();
  Set<String> compensationBindingNames = Set.of();
  Map<String, String> compensateRefMap = Map.of(); // compensatingName → originalName
  var metaOpt = definitionRegistry.findByIdentity(
      meta.getNamespace(), meta.getName(), meta.getVersion());
  if (metaOpt.isPresent()) {
    CaseDefinition def = definitionRegistry.getCaseDefinition(metaOpt.get());
    compensationBindingNames = def.getBindings().stream()
        .filter(Binding::isCompensation)
        .map(Binding::getName)
        .collect(Collectors.toSet());
    compensateRefMap = new HashMap<>();
    for (Binding b : def.getBindings()) {
      if (b.getCompensateRef() != null) {
        compensateRefMap.put(b.getCompensateRef(), b.getName());
      }
    }
  }

  // Load plan items and partition into forward vs compensation
  List<PlanItemRecord> planItems = planItemStore.findByCaseId(caseId, tenancyId);
  List<TimelineStepType> forwardSteps = new ArrayList<>();
  List<CompensationStepType> compensationSteps = new ArrayList<>();
  Map<String, String> finalCompensateRefMap = compensateRefMap;

  for (PlanItemRecord pi : planItems) {
    String targetType = pi.targetType() != null ? pi.targetType().name().toLowerCase().replace('_', '-') : "unknown";
    if (compensationBindingNames.contains(pi.bindingName())) {
      String compensatesBinding = finalCompensateRefMap.getOrDefault(pi.bindingName(), null);
      compensationSteps.add(new CompensationStepType(
          pi.planItemId(), pi.bindingName(), targetType,
          pi.status().name(), pi.createdAt(), pi.completedAt(),
          compensatesBinding, null));
    } else {
      forwardSteps.add(new TimelineStepType(
          pi.planItemId(), pi.bindingName(), targetType,
          pi.status().name(), pi.createdAt(), pi.completedAt()));
    }
  }

  return new CompensationTimelineType(
      caseId, instance.getState().name(), triggeredBy, reason,
      compensationStartedAt, compensationCompletedAt,
      forwardSteps, compensationSteps);
}
```

Add these imports to `CaseQueryResolver.java`:

```java
import io.casehub.api.model.Binding;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.graphql.dto.CompensationStepType;
import io.casehub.engine.graphql.dto.CompensationTimelineType;
import io.casehub.engine.graphql.dto.TimelineStepType;
import java.time.Instant;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=CompensationTimelineQueryTest 2>&1 | tail -10`
Expected: both tests PASS

- [ ] **Step 6: Run full graphql module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql 2>&1 | tail -10`
Expected: all tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/169/engine add graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationTimelineType.java graphql/src/main/java/io/casehub/engine/graphql/dto/TimelineStepType.java graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationStepType.java graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java graphql/src/test/java/io/casehub/engine/graphql/CompensationTimelineQueryTest.java
git -C /Users/mdproctor/claude/casehub/slots/169/engine commit -m "feat(#390): runtime compensation timeline — GraphQL query on CaseQueryResolver Refs #390 Refs #238"
```

---

## Batch 3: Ledger compensation chain

### Task 3: CompensationChain query

**Files:**
- Modify: `graphql/pom.xml` (add `casehub-engine-ledger` dependency)
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationLedgerEntryType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationChainType.java`
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java` (add query + injection)
- Create: `graphql/src/test/java/io/casehub/engine/graphql/CompensationChainQueryTest.java`
- Test: `graphql/src/test/java/io/casehub/engine/graphql/CompensationChainQueryTest.java`

**Interfaces:**
- Consumes: `CaseLedgerEntryRepository.findByCaseId(UUID)` (from engine-ledger), `JpaLedgerEntry.supplements` field, `CompensationSupplement` fields
- Produces: `CaseQueryResolver.compensationChain(UUID caseId)` → `CompensationChainType`

- [ ] **Step 1: Add engine-ledger dependency to graphql pom.xml**

Add to `graphql/pom.xml` under `<!-- Compile -->` dependencies:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-ledger</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger-api</artifactId>
    </dependency>
```

Note: `casehub-ledger-api` is managed via the parent BOM (no version needed). `casehub-engine-ledger` needs explicit version. Check parent pom.xml for ledger dependency management — if `casehub-engine-ledger` is managed, omit the version tag.

- [ ] **Step 2: Write the failing test**

Create `graphql/src/test/java/io/casehub/engine/graphql/CompensationChainQueryTest.java`:

```java
package io.casehub.engine.graphql;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.graphql.dto.CompensationChainType;
import io.casehub.ledger.model.CaseLedgerEntry;
import io.casehub.ledger.repository.CaseLedgerEntryRepository;
import io.casehub.ledger.runtime.model.supplement.JpaCompensationSupplement;
import io.casehub.platform.api.identity.CurrentPrincipal;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CompensationChainQueryTest {

  private CaseQueryResolver resolver;
  private CaseLedgerEntryRepository ledgerRepository;

  @BeforeEach
  void setUp() {
    resolver = new CaseQueryResolver();
    resolver.instanceRepository = mock(CaseInstanceRepository.class);
    resolver.definitionRegistry = mock(io.casehub.engine.common.spi.CaseDefinitionRegistry.class);
    resolver.runtime = mock(io.casehub.api.engine.CaseHubRuntime.class);
    resolver.planItemStore = mock(io.casehub.engine.common.spi.PlanItemStore.class);
    resolver.currentPrincipal = mock(CurrentPrincipal.class);
    ledgerRepository = mock(CaseLedgerEntryRepository.class);
    resolver.ledgerRepository = ledgerRepository;
  }

  @Test
  void compensationChainReturnsEmptyForCaseWithoutCompensation() {
    UUID caseId = UUID.randomUUID();
    when(ledgerRepository.findByCaseId(caseId)).thenReturn(List.of());

    CompensationChainType result = resolver.compensationChain(caseId);

    assertThat(result).isNotNull();
    assertThat(result.caseId()).isEqualTo(caseId);
    assertThat(result.entries()).isEmpty();
  }

  @Test
  void compensationChainFiltersToCompensationSupplementEntries() {
    UUID caseId = UUID.randomUUID();
    UUID originalEntryId = UUID.randomUUID();

    // Entry without compensation supplement
    CaseLedgerEntry normalEntry = new CaseLedgerEntry();
    normalEntry.caseId = caseId;
    normalEntry.eventType = "CaseStarted";
    normalEntry.caseStatus = "RUNNING";

    // Entry with compensation supplement
    CaseLedgerEntry compensationEntry = new CaseLedgerEntry();
    compensationEntry.caseId = caseId;
    compensationEntry.eventType = "CompensationStepCompleted";
    compensationEntry.caseStatus = "COMPENSATING";
    JpaCompensationSupplement supplement = new JpaCompensationSupplement();
    supplement.originalEntryId = originalEntryId;
    supplement.compensationReason = "Trial withdrawn";
    supplement.regulatoryBasis = "GDPR Art.17";
    supplement.compensationMode = "human-driven";
    compensationEntry.attach(supplement);

    when(ledgerRepository.findByCaseId(caseId))
        .thenReturn(List.of(normalEntry, compensationEntry));

    CompensationChainType result = resolver.compensationChain(caseId);

    assertThat(result.entries()).hasSize(1);
    var entry = result.entries().get(0);
    assertThat(entry.eventType()).isEqualTo("CompensationStepCompleted");
    assertThat(entry.originalEntryId()).isEqualTo(originalEntryId);
    assertThat(entry.compensationReason()).isEqualTo("Trial withdrawn");
    assertThat(entry.regulatoryBasis()).isEqualTo("GDPR Art.17");
    assertThat(entry.compensationMode()).isEqualTo("human-driven");
  }

  @Test
  void compensationChainReturnsNullForUnknownCase() {
    UUID caseId = UUID.randomUUID();
    when(ledgerRepository.findByCaseId(caseId)).thenReturn(List.of());

    CompensationChainType result = resolver.compensationChain(caseId);

    assertThat(result.entries()).isEmpty();
  }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=CompensationChainQueryTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5`
Expected: compilation error — DTOs and `compensationChain` method don't exist, `ledgerRepository` field not on resolver

- [ ] **Step 4: Create ledger chain DTO records**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationLedgerEntryType.java`:

```java
package io.casehub.engine.graphql.dto;

import java.time.Instant;
import java.util.UUID;
import org.eclipse.microprofile.graphql.Type;

@Type("CompensationLedgerEntry")
public record CompensationLedgerEntryType(
    UUID entryId,
    Instant timestamp,
    String eventType,
    String caseStatus,
    UUID causedByEntryId,
    UUID originalEntryId,
    String compensationReason,
    String regulatoryBasis,
    String compensationMode) {}
```

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationChainType.java`:

```java
package io.casehub.engine.graphql.dto;

import java.util.List;
import java.util.UUID;
import org.eclipse.microprofile.graphql.Type;

@Type("CompensationChain")
public record CompensationChainType(
    UUID caseId,
    List<CompensationLedgerEntryType> entries) {}
```

- [ ] **Step 5: Implement compensationChain query**

Add `ledgerRepository` field to `CaseQueryResolver.java`:

```java
@Inject CaseLedgerEntryRepository ledgerRepository;
```

Add import:

```java
import io.casehub.ledger.repository.CaseLedgerEntryRepository;
```

Add the query method:

```java
@Query
@Description("Ledger compensation chain — entries with CompensationSupplement for a case")
public CompensationChainType compensationChain(UUID caseId) {
  var entries = ledgerRepository.findByCaseId(caseId);
  List<CompensationLedgerEntryType> compensationEntries = entries.stream()
      .filter(e -> e.findSupplement(CompensationSupplement.class).isPresent())
      .map(e -> {
        CompensationSupplement cs = e.findSupplement(CompensationSupplement.class).orElseThrow();
        return new CompensationLedgerEntryType(
            e.id, e.recordedAt, e.eventType, e.caseStatus,
            e.causedByEntryId,
            cs.originalEntryId, cs.compensationReason,
            cs.regulatoryBasis, cs.compensationMode);
      })
      .toList();
  return new CompensationChainType(caseId, compensationEntries);
}
```

Add imports:

```java
import io.casehub.engine.graphql.dto.CompensationChainType;
import io.casehub.engine.graphql.dto.CompensationLedgerEntryType;
import io.casehub.ledger.api.model.supplement.CompensationSupplement;
import io.casehub.ledger.repository.CaseLedgerEntryRepository;
```

**Implementation note:** The exact field names on `CaseLedgerEntry` (`id`, `recordedAt`, `causedByEntryId`) come from the `JpaLedgerEntry` base class. Verify field names against the actual `JpaLedgerEntry` class — they may use JPA annotations with different accessor patterns. Check `findSupplement(Class)` method exists on `JpaLedgerEntry` — if not, iterate `supplements` list manually and check `instanceof CompensationSupplement`.

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=CompensationChainQueryTest 2>&1 | tail -10`
Expected: all 3 tests PASS

- [ ] **Step 7: Run full graphql module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql 2>&1 | tail -10`
Expected: all tests pass including previous batches

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/169/engine add graphql/pom.xml graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationChainType.java graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationLedgerEntryType.java graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java graphql/src/test/java/io/casehub/engine/graphql/CompensationChainQueryTest.java
git -C /Users/mdproctor/claude/casehub/slots/169/engine commit -m "feat(#390): ledger compensation chain — GraphQL query with CompensationSupplement filtering Refs #390 Refs #238"
```

---

## References

- `specs/issue-238-saga-compensation/2026-09-04-compensation-visualization-design.md` — design spec
- `specs/issue-238-saga-compensation/2026-09-01-saga-compensation-design.md` §11 — visual concept specification
- `engine/api/src/main/java/io/casehub/api/model/Binding.java` — compensateRef, compensation fields
- `engine/api/src/main/java/io/casehub/api/model/BindingTarget.java:27` — sealed permits list
- `engine/graphql/src/main/java/io/casehub/engine/graphql/dto/CaseDefinitionType.java` — GraphQL type to extend
- `engine/graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java` — query resolver to extend
- `engine/graphql/src/test/java/io/casehub/engine/graphql/CaseQueryResolverTest.java` — test pattern (mock injection)
- `engine/common/src/main/java/io/casehub/engine/common/spi/PlanItemStore.java:30` — PlanItem query SPI
- `engine/common/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java` — plan item DTO
- `engine/planning/src/main/java/io/casehub/engine/planning/compensation/CaseCompensationServiceImpl.java` — EventLog metadata format
- `engine/ledger/src/main/java/io/casehub/ledger/repository/CaseLedgerEntryRepository.java` — ledger query
- `ledger/api/src/main/java/io/casehub/ledger/api/model/supplement/CompensationSupplement.java` — supplement fields
- casehubio/work#390, casehubio/work#238
