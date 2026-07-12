# Retrofit Work SPIs to NamedStrategy — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #287 — refactor(engine#634): retrofit work SPIs to extend NamedStrategy
**Issue group:** #287

**Goal:** Unify four ad-hoc SPI resolution mechanisms (CDI Instance scanning, @Named iteration, @Alternative/@Priority, @DefaultBean) into the platform-standard NamedStrategy + StrategyResolver convention.

**Architecture:** Each SPI interface extends `NamedStrategy` (adding `id()`). Each implementation returns a unique id and drops CDI selection annotations. Resolution sites inject `StrategyResolver` and resolve by config-specified id. Two new config properties (`casehub.work.sla.claim-policy`, `casehub.work.sla.breach-policy`) join the existing `casehub.work.routing.strategy`.

**Tech Stack:** Java 21, Quarkus 3.32, CDI, casehub-platform-api (NamedStrategy, StrategyResolver), casehub-platform (DefaultStrategyResolver)

## Global Constraints

- All SPIs live in `api/` — pure Java, no JPA, no REST
- All strategy implementations are `@ApplicationScoped` CDI beans
- `casehub-platform-api` is already a dependency of `api/` (provides `NamedStrategy`, `StrategyResolver`)
- `casehub-platform` (provides `DefaultStrategyResolver`) is test-scoped in `runtime/` and on the classpath in all real deployments
- Build with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and editing — never bash grep/Edit on `.java` files
- Pre-release: breaking changes are acceptable

---

### Task 1: Foundation — SPI interfaces extend NamedStrategy, implementations add id(), config adds SlaConfig

All structural changes that compile without breaking existing behavior. CDI annotations are kept temporarily — resolution sites still use old mechanisms until Tasks 2–4.

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/spi/WorkerSelectionStrategy.java`
- Modify: `api/src/main/java/io/casehub/work/api/spi/InstanceAssignmentStrategy.java`
- Modify: `api/src/main/java/io/casehub/work/api/spi/ClaimSlaPolicy.java`
- Modify: `api/src/main/java/io/casehub/work/api/spi/SlaBreachPolicy.java`
- Modify: `core/src/main/java/io/casehub/work/core/strategy/LeastLoadedStrategy.java`
- Modify: `core/src/main/java/io/casehub/work/core/strategy/ClaimFirstStrategy.java`
- Modify: `core/src/main/java/io/casehub/work/core/strategy/RoundRobinStrategy.java`
- Modify: `core/src/main/java/io/casehub/work/core/policy/ContinuationPolicy.java`
- Modify: `core/src/main/java/io/casehub/work/core/policy/FreshClockPolicy.java`
- Modify: `core/src/main/java/io/casehub/work/core/policy/SingleBudgetPolicy.java`
- Modify: `core/src/main/java/io/casehub/work/core/policy/PhaseClockPolicy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/PoolAssignmentStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/RoundRobinAssignmentStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/ExplicitListAssignmentStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/CompositeInstanceAssignmentStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/NoOpSlaBreachPolicy.java`
- Modify: `ai/src/main/java/io/casehub/work/ai/skill/SemanticWorkerSelectionStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/config/WorkItemsConfig.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java` (testConfig)
- Test: `core/src/test/java/io/casehub/work/core/strategy/WorkBrokerTest.java` (anonymous impls need id)
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java` (anonymous impls need id)
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemAssignmentServiceTest.java` (anonymous impls need id)

**Interfaces:**
- Produces: `WorkerSelectionStrategy extends NamedStrategy`, `InstanceAssignmentStrategy extends NamedStrategy`, `ClaimSlaPolicy extends NamedStrategy`, `SlaBreachPolicy extends NamedStrategy` — all with `String id()` method
- Produces: `WorkItemsConfig.SlaConfig` with `String claimPolicy()` (default `"continuation"`) and `String breachPolicy()` (default `"no-op"`)

- [ ] **Step 1: Add `extends NamedStrategy` to all 4 SPI interfaces**

Use `ide_edit_member` on each interface declaration to add `extends NamedStrategy`. Update javadoc to reference NamedStrategy/config-based selection instead of `@Alternative @Priority` / `@Named`.

```java
// WorkerSelectionStrategy — add extends NamedStrategy
public interface WorkerSelectionStrategy extends NamedStrategy {

// InstanceAssignmentStrategy — add extends NamedStrategy
public interface InstanceAssignmentStrategy extends NamedStrategy {

// ClaimSlaPolicy — add extends NamedStrategy
public interface ClaimSlaPolicy extends NamedStrategy {

// SlaBreachPolicy — add extends NamedStrategy
public interface SlaBreachPolicy extends NamedStrategy {
```

- [ ] **Step 2: Add `id()` to all 13 implementations**

Use `ide_insert_member` to add the `id()` method to each implementation. Keep all existing CDI annotations for now.

```java
// LeastLoadedStrategy
@Override public String id() { return "least-loaded"; }

// ClaimFirstStrategy
@Override public String id() { return "claim-first"; }

// RoundRobinStrategy
@Override public String id() { return "round-robin"; }

// SemanticWorkerSelectionStrategy
@Override public String id() { return "semantic"; }

// ContinuationPolicy
@Override public String id() { return "continuation"; }

// FreshClockPolicy
@Override public String id() { return "fresh-clock"; }

// SingleBudgetPolicy
@Override public String id() { return "single-budget"; }

// PhaseClockPolicy
@Override public String id() { return "phase-clock"; }

// PoolAssignmentStrategy
@Override public String id() { return "pool"; }

// RoundRobinAssignmentStrategy (multiinstance)
@Override public String id() { return "round-robin"; }

// ExplicitListAssignmentStrategy
@Override public String id() { return "explicit"; }

// CompositeInstanceAssignmentStrategy
@Override public String id() { return "composite"; }

// NoOpSlaBreachPolicy
@Override public String id() { return "no-op"; }
```

- [ ] **Step 3: Fix anonymous/inner class implementations in tests**

Anonymous implementations of the SPIs now must implement `id()`. Find all anonymous impls and lambdas in test files that implement these SPIs and add an `id()` method.

Key locations:
- `WorkItemAssignmentServiceTest.java` — anonymous `WorkerSelectionStrategy` at line 57 (createdOnly), and lambdas at lines 170, 176, 181
- `ExpiryLifecycleServiceTest.java` — `FixedClaimSlaPolicy` inner class at line 109, `CapturingStrategy` at line 116, `TestSlaBreachPolicy` at line 130
- `WorkBrokerTest.java` — anonymous/lambda `WorkerSelectionStrategy` instances

For lambdas: lambdas implement SAM (single abstract method) interfaces. Adding `id()` as a second abstract method means `WorkerSelectionStrategy` is no longer a functional interface. Lambdas must be converted to anonymous inner classes.

```java
// Example: lambda → anonymous class
// Before (lambda):
(ctx, candidates) -> AssignmentDecision.noChange()
// After (anonymous class):
new WorkerSelectionStrategy() {
    @Override public String id() { return "test"; }
    @Override public AssignmentDecision select(SelectionContext ctx, List<WorkerCandidate> candidates) {
        return AssignmentDecision.noChange();
    }
}
```

- [ ] **Step 4: Add SlaConfig to WorkItemsConfig**

Use `ide_insert_member` to add the SlaConfig group to `WorkItemsConfig`:

```java
@io.smallrye.config.WithName("sla")
SlaConfig sla();

interface SlaConfig {
    @WithDefault("continuation")
    String claimPolicy();

    @WithDefault("no-op")
    String breachPolicy();
}
```

Update `testConfig()` in `WorkItemServiceTest` to implement the new `sla()` method:

```java
@Override
public SlaConfig sla() {
    return new SlaConfig() {
        @Override public String claimPolicy() { return "continuation"; }
        @Override public String breachPolicy() { return "no-op"; }
    };
}
```

- [ ] **Step 5: Build api/, core/, runtime/ modules to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All tests pass. No behavioral change — resolution sites still use old mechanisms.

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#287): add NamedStrategy to work SPIs + id() to all implementations

All 4 SPI interfaces (WorkerSelectionStrategy, InstanceAssignmentStrategy,
ClaimSlaPolicy, SlaBreachPolicy) now extend NamedStrategy. All 13
implementations return their strategy id. SlaConfig added to WorkItemsConfig.

CDI selection annotations kept temporarily — resolution sites refactored
in subsequent commits.

Refs #287"
```

---

### Task 2: WorkerSelectionStrategy resolution — WorkItemAssignmentService

Refactor `WorkItemAssignmentService` to use `StrategyResolver` instead of CDI Instance scanning + config switch. Remove dead CDI annotations from `ClaimFirstStrategy` and `SemanticWorkerSelectionStrategy`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java`
- Modify: `core/src/main/java/io/casehub/work/core/strategy/ClaimFirstStrategy.java`
- Modify: `ai/src/main/java/io/casehub/work/ai/skill/SemanticWorkerSelectionStrategy.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemAssignmentServiceTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java` (setUp constructs WorkItemAssignmentService)
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java` (setUp constructs WorkItemAssignmentService)

**Interfaces:**
- Consumes: `StrategyResolver` (from casehub-platform-api), `WorkerSelectionStrategy extends NamedStrategy` (from Task 1)
- Produces: `WorkItemAssignmentService(StrategyResolver, WorkItemsConfig, WorkerRegistry, WorkloadProvider, WorkBroker, ExclusionPolicy)` constructor

- [ ] **Step 1: Write failing test — StrategyResolver-based resolution**

Add a test to `WorkItemAssignmentServiceTest` that constructs the service with a mock `StrategyResolver` and verifies the correct strategy is resolved by id:

```java
@Test
void assign_usesStrategyFromResolver() {
    var strategy = new LeastLoadedStrategy();
    var resolver = mock(StrategyResolver.class);
    when(resolver.resolve(WorkerSelectionStrategy.class, "least-loaded")).thenReturn(strategy);
    var config = WorkItemServiceTest.testConfig();
    service = new WorkItemAssignmentService(resolver, config, workerRegistry, workloadProvider, workBroker,
            (userId, excluded) -> PolicyDecision.ALLOW);

    when(workloadProvider.getActiveWorkCount("alice")).thenReturn(0);
    var wi = workItem(null, null, "alice");
    service.assign(wi, AssignmentTrigger.CREATED);
    assertThat(wi.assigneeId).isEqualTo("alice");
    verify(resolver).resolve(WorkerSelectionStrategy.class, "least-loaded");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemAssignmentServiceTest#assign_usesStrategyFromResolver -pl runtime`

Expected: FAIL — constructor with StrategyResolver does not exist yet.

- [ ] **Step 3: Refactor WorkItemAssignmentService**

Replace the CDI constructor and `activeStrategy()` method. Use `ide_edit_member` on:

1. **CDI constructor** — replace 9-param constructor with 6-param:
```java
@Inject
public WorkItemAssignmentService(
        final StrategyResolver strategyResolver,
        final WorkItemsConfig config,
        final WorkerRegistry workerRegistry,
        final WorkloadProvider workloadProvider,
        final WorkBroker workBroker,
        final ExclusionPolicy exclusionPolicy) {
    this.strategyResolver = strategyResolver;
    this.config = config;
    this.workerRegistry = workerRegistry;
    this.workloadProvider = workloadProvider;
    this.workBroker = workBroker;
    this.exclusionPolicy = exclusionPolicy;
}
```

2. **Fields** — replace `Instance<WorkerSelectionStrategy> alternatives`, `ClaimFirstStrategy claimFirst`, `LeastLoadedStrategy leastLoaded`, `RoundRobinStrategy roundRobin`, `WorkerSelectionStrategy fixedStrategy` with:
```java
private final StrategyResolver strategyResolver;
private final WorkItemsConfig config;
```
Keep: `workerRegistry`, `workloadProvider`, `workBroker`, `exclusionPolicy`.

3. **`activeStrategy()`** — replace body:
```java
private WorkerSelectionStrategy activeStrategy() {
    return strategyResolver.resolve(WorkerSelectionStrategy.class, config.routing().strategy());
}
```

4. **Remove the package-private test constructor** — tests now use a mock StrategyResolver.

- [ ] **Step 4: Update all tests that construct WorkItemAssignmentService**

Update `WorkItemAssignmentServiceTest.setUp()`:
```java
@BeforeEach
void setUp() {
    workBroker = new WorkBroker();
    lenient().when(workerRegistry.resolveGroup(anyString())).thenReturn(List.of());
    lenient().when(workloadProvider.getActiveWorkCount(anyString())).thenReturn(0);
    var resolver = mock(StrategyResolver.class);
    when(resolver.resolve(WorkerSelectionStrategy.class, "least-loaded")).thenReturn(new LeastLoadedStrategy());
    service = new WorkItemAssignmentService(resolver, WorkItemServiceTest.testConfig(),
            workerRegistry, workloadProvider, workBroker,
            (userId, excluded) -> PolicyDecision.ALLOW);
}
```

Update tests that construct `WorkItemAssignmentService` with a specific strategy (e.g., `assign_skipsWork_whenTriggerNotInStrategyTriggers`, `assign_setsCandidateGroups_fromNarrowDecision`, `assign_doesNotOverwrite_existingFields_onNoChange`):
```java
// For each test needing a custom strategy:
var resolver = mock(StrategyResolver.class);
when(resolver.resolve(WorkerSelectionStrategy.class, "least-loaded")).thenReturn(customStrategy);
service = new WorkItemAssignmentService(resolver, WorkItemServiceTest.testConfig(),
        workerRegistry, workloadProvider, workBroker,
        (userId, excluded) -> PolicyDecision.ALLOW);
```

Update `ExpiryLifecycleServiceTest.setUp()` — it constructs `WorkItemAssignmentService` with the old test constructor:
```java
var assignmentResolver = mock(StrategyResolver.class);
when(assignmentResolver.resolve(eq(WorkerSelectionStrategy.class), anyString()))
        .thenReturn(new CapturingStrategy());
service.assignmentService = new WorkItemAssignmentService(
        assignmentResolver, WorkItemServiceTest.testConfig(),
        group -> java.util.List.of(),
        id -> 0,
        new WorkBroker(),
        (userId, excluded) -> PolicyDecision.ALLOW);
```

Update `WorkItemServiceTest.setUp()` — same pattern.

- [ ] **Step 5: Remove dead CDI annotations**

Use `ide_edit_member` on:
- `ClaimFirstStrategy` — remove `@Alternative` and `@Priority(0)` from class declaration
- `SemanticWorkerSelectionStrategy` — remove `@Alternative` and `@Priority(1)` from class declaration

- [ ] **Step 6: Build and test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#287): refactor WorkItemAssignmentService to use StrategyResolver

Replace CDI Instance<> scanning + config switch with StrategyResolver.resolve().
Remove @Alternative/@Priority from ClaimFirstStrategy, SemanticWorkerSelectionStrategy.

Refs #287"
```

---

### Task 3: InstanceAssignmentStrategy resolution — MultiInstanceSpawnService + RoundRobinAssignmentStrategy

Refactor both resolution sites to use `StrategyResolver`. Remove `@Named` from all InstanceAssignmentStrategy implementations.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/RoundRobinAssignmentStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/PoolAssignmentStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/ExplicitListAssignmentStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/CompositeInstanceAssignmentStrategy.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/multiinstance/InstanceAssignmentStrategyTest.java`

**Interfaces:**
- Consumes: `StrategyResolver`, `InstanceAssignmentStrategy extends NamedStrategy` (from Task 1)
- Produces: `MultiInstanceSpawnService` using `StrategyResolver` for strategy resolution; `RoundRobinAssignmentStrategy(StrategyResolver, WorkItemsConfig)` constructor

- [ ] **Step 1: Write failing test — MultiInstanceSpawnService StrategyResolver resolution**

Add test to `InstanceAssignmentStrategyTest`:

```java
@Test
void resolveStrategy_byId_viaStrategyResolver() {
    var pool = new PoolAssignmentStrategy();
    var explicit = new ExplicitListAssignmentStrategy();
    var resolver = mock(StrategyResolver.class);
    when(resolver.resolve(InstanceAssignmentStrategy.class, "pool")).thenReturn(pool);
    when(resolver.resolve(InstanceAssignmentStrategy.class, "explicit")).thenReturn(explicit);

    // Verify pool is default
    var parent = parent("reviewers", null);
    var instances = List.of(item(), item());
    pool.assign((List) instances, new MultiInstanceContext(parent,
            new MultiInstanceConfig(2, 1, null, "pool", null, false, null)));
    assertThat(instances).allMatch(i -> "reviewers".equals(i.candidateGroups));
}
```

- [ ] **Step 2: Refactor MultiInstanceSpawnService**

Use `ide_edit_member` to replace the fields and `resolveStrategy()` method:

Fields — replace:
```java
// Remove:
@Inject @Named("pool") InstanceAssignmentStrategy defaultStrategy;
@Inject @Any Instance<InstanceAssignmentStrategy> strategies;

// Add:
@Inject StrategyResolver strategyResolver;
```

Method `resolveStrategy()` — replace body:
```java
private InstanceAssignmentStrategy resolveStrategy(final String name) {
    if (name == null || name.isBlank()) {
        return strategyResolver.resolve(InstanceAssignmentStrategy.class, "pool");
    }
    return strategyResolver.resolve(InstanceAssignmentStrategy.class, name);
}
```

- [ ] **Step 3: Refactor RoundRobinAssignmentStrategy**

Use `ide_edit_member` on the CDI constructor:

```java
@Inject
public RoundRobinAssignmentStrategy(
        final StrategyResolver strategyResolver,
        final WorkItemsConfig config) {
    this.workerSelectionStrategy = strategyResolver.resolve(
            WorkerSelectionStrategy.class, config.routing().strategy());
}
```

Keep the package-private test constructor `RoundRobinAssignmentStrategy(WorkerSelectionStrategy)`.

- [ ] **Step 4: Remove @Named from all InstanceAssignmentStrategy implementations**

Use `ide_edit_member` on class declarations:
- `PoolAssignmentStrategy` — remove `@Named("pool")`
- `RoundRobinAssignmentStrategy` — remove `@Named("roundRobin")`
- `ExplicitListAssignmentStrategy` — remove `@Named("explicit")`
- `CompositeInstanceAssignmentStrategy` — remove `@Named("composite")`

- [ ] **Step 5: Build and test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#287): refactor InstanceAssignmentStrategy resolution to StrategyResolver

MultiInstanceSpawnService and RoundRobinAssignmentStrategy now use
StrategyResolver instead of @Named/@Any Instance<> iteration.
Remove @Named from all InstanceAssignmentStrategy implementations.

Refs #287"
```

---

### Task 4: ClaimSlaPolicy + SlaBreachPolicy resolution — WorkItemService + ExpiryLifecycleService

Refactor both services to resolve policies via `StrategyResolver` + config. Remove `@Alternative` from ClaimSlaPolicy implementations and `@DefaultBean` from `NoOpSlaBreachPolicy`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/NoOpSlaBreachPolicy.java`
- Modify: `core/src/main/java/io/casehub/work/core/policy/FreshClockPolicy.java`
- Modify: `core/src/main/java/io/casehub/work/core/policy/SingleBudgetPolicy.java`
- Modify: `core/src/main/java/io/casehub/work/core/policy/PhaseClockPolicy.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemServiceTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java`

**Interfaces:**
- Consumes: `StrategyResolver`, `ClaimSlaPolicy extends NamedStrategy`, `SlaBreachPolicy extends NamedStrategy` (from Task 1), `WorkItemsConfig.SlaConfig` (from Task 1)
- Produces: `WorkItemService(WorkItemStore, AuditEntryStore, WorkItemsConfig, WorkItemAssignmentService, StrategyResolver, ExclusionPolicy, BlockedAttemptAuditService, CapabilityValidator, WorkItemTimerService)` constructor

- [ ] **Step 1: Write failing test — WorkItemService with StrategyResolver**

Update `WorkItemServiceTest.setUp()` to pass a mock `StrategyResolver` instead of `ContinuationPolicy`:

```java
@BeforeEach
void setUp() {
    repo = new TestWorkItemRepo();
    auditStore = new TestAuditRepo();
    var resolver = mock(StrategyResolver.class);
    when(resolver.resolve(ClaimSlaPolicy.class, "continuation"))
            .thenReturn(new io.casehub.work.core.policy.ContinuationPolicy());
    var assignmentResolver = mock(StrategyResolver.class);
    when(assignmentResolver.resolve(eq(WorkerSelectionStrategy.class), anyString()))
            .thenReturn(new WorkerSelectionStrategy() {
                @Override public String id() { return "test"; }
                @Override public AssignmentDecision select(SelectionContext c, List<WorkerCandidate> w) {
                    return AssignmentDecision.noChange();
                }
            });
    service = new WorkItemService(repo, auditStore, testConfig(),
            new WorkItemAssignmentService(assignmentResolver, testConfig(),
                    group -> List.of(), workerId -> 0, new WorkBroker(),
                    (userId, excluded) -> PolicyDecision.ALLOW),
            resolver,
            (userId, excluded) -> PolicyDecision.ALLOW,
            new BlockedAttemptAuditService(auditStore),
            new CapabilityValidator(ValidationMode.PERMISSIVE, () -> java.util.Set.of()),
            mock(WorkItemTimerService.class));
    service.preferenceProvider = scope -> new MapPreferences(Map.of());
    var outcomeValidator = new OutcomeValidator();
    outcomeValidator.conditionEvaluator = new JexlConditionEvaluator();
    service.outcomeValidator = outcomeValidator;
    service.lifecycleEmitter = mock(WorkItemLifecycleEmitter.class);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemServiceTest -pl runtime`

Expected: FAIL — constructor signature doesn't match.

- [ ] **Step 3: Refactor WorkItemService constructor**

Use `ide_edit_member` to replace the `ClaimSlaPolicy` parameter with `StrategyResolver`:

```java
@Inject
public WorkItemService(final WorkItemStore workItemStore,
        final AuditEntryStore auditStore,
        final WorkItemsConfig config,
        final WorkItemAssignmentService assignmentService,
        final StrategyResolver strategyResolver,
        final ExclusionPolicy exclusionPolicy,
        final BlockedAttemptAuditService blockedAuditService,
        final CapabilityValidator capabilityValidator,
        final WorkItemTimerService timerService) {
    this.workItemStore = workItemStore;
    this.auditStore = auditStore;
    this.config = config;
    this.assignmentService = assignmentService;
    this.claimSlaPolicy = strategyResolver.resolve(ClaimSlaPolicy.class, config.sla().claimPolicy());
    this.exclusionPolicy = exclusionPolicy;
    this.blockedAuditService = blockedAuditService;
    this.capabilityValidator = capabilityValidator;
    this.timerService = timerService;
}
```

- [ ] **Step 4: Refactor ExpiryLifecycleService**

Replace field injection of `ClaimSlaPolicy` and `SlaBreachPolicy` with `StrategyResolver` + `@PostConstruct`:

Use `ide_edit_member` to replace the fields:
```java
// Remove:
@Inject ClaimSlaPolicy claimSlaPolicy;
@Inject SlaBreachPolicy slaBreachPolicy;  // (find actual field — may be named differently)

// Add:
@Inject StrategyResolver strategyResolver;

ClaimSlaPolicy claimSlaPolicy;
SlaBreachPolicy slaBreachPolicy;
```

Use `ide_insert_member` to add `@PostConstruct`:
```java
@PostConstruct
void init() {
    this.claimSlaPolicy = strategyResolver.resolve(ClaimSlaPolicy.class, config.sla().claimPolicy());
    this.slaBreachPolicy = strategyResolver.resolve(SlaBreachPolicy.class, config.sla().breachPolicy());
}
```

- [ ] **Step 5: Update ExpiryLifecycleServiceTest**

Update `setUp()` to provide a mock StrategyResolver instead of directly setting policy fields:

```java
@BeforeEach
void setUp() {
    store = new TestStore();
    auditStore = new TestAuditStore();
    policy = new TestSlaBreachPolicy();
    breachEvents.clear();

    var resolver = mock(StrategyResolver.class);
    when(resolver.resolve(ClaimSlaPolicy.class, "continuation")).thenReturn(new FixedClaimSlaPolicy());
    when(resolver.resolve(SlaBreachPolicy.class, "no-op")).thenReturn(policy);

    service = new ExpiryLifecycleService();
    service.workItemStore = store;
    service.auditStore = auditStore;
    service.strategyResolver = resolver;
    service.preferenceProvider = EMPTY_PREFS;
    service.slaBreachEventBus = new CapturingBreachEventBus(breachEvents);
    service.lifecycleEmitter = mock(WorkItemLifecycleEmitter.class);
    service.config = WorkItemServiceTest.testConfig();
    // ... WorkItemAssignmentService setup same as Task 2 pattern
    service.timerService = mock(WorkItemTimerService.class);
    service.init(); // trigger @PostConstruct manually
}
```

- [ ] **Step 6: Remove dead CDI annotations**

Use `ide_edit_member` on class declarations:
- `FreshClockPolicy` — remove `@Alternative`
- `SingleBudgetPolicy` — remove `@Alternative`
- `PhaseClockPolicy` — remove `@Alternative`
- `NoOpSlaBreachPolicy` — remove `@DefaultBean`

- [ ] **Step 7: Build and test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git commit -m "feat(#287): refactor ClaimSlaPolicy + SlaBreachPolicy to StrategyResolver

WorkItemService and ExpiryLifecycleService now resolve policies via
StrategyResolver + config (casehub.work.sla.claim-policy,
casehub.work.sla.breach-policy). Remove @Alternative from ClaimSla
policies, @DefaultBean from NoOpSlaBreachPolicy.

Refs #287"
```

---

### Task 5: Cross-module build verification + integration tests

Full build across all modules. Fix any remaining compilation issues from the SPI changes propagating to modules not yet touched (e.g., engine-adapter, flow, examples).

**Files:**
- Possibly modify: test files in `integration-tests/`, `integration-tests-memory/`, `examples/`, `flow-examples/`, `engine-adapter/`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/config/WorkItemsConfigRoutingTest.java` (if it tests strategy names)
- Modify: `runtime/src/test/java/io/casehub/work/runtime/config/WorkItemsConfigDefaultsTest.java` (add SlaConfig defaults)

**Interfaces:**
- Consumes: All changes from Tasks 1–4

- [ ] **Step 1: Build all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

Expected: Build succeeds. Fix any compilation failures from:
- Anonymous `WorkerSelectionStrategy` lambdas in other modules (no longer a functional interface)
- Anonymous `ClaimSlaPolicy`/`SlaBreachPolicy`/`InstanceAssignmentStrategy` implementations missing `id()`
- Config test classes missing `sla()` method

- [ ] **Step 2: Fix any remaining compilation errors**

For each module that fails, add `id()` to anonymous implementations and `sla()` to config stubs. Use `ide_search_text` to find all anonymous impls:
- Search for `implements WorkerSelectionStrategy` across all modules
- Search for `implements ClaimSlaPolicy` across all modules
- Search for `implements SlaBreachPolicy` across all modules
- Search for `implements InstanceAssignmentStrategy` across all modules
- Search for `implements WorkItemsConfig` across all modules

- [ ] **Step 3: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests`

Expected: All integration tests pass. The default strategy is `least-loaded` via config, which is unchanged.

- [ ] **Step 4: Run integration-tests-memory**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests-memory`

Expected: All tests pass.

- [ ] **Step 5: Use `ide_diagnostics` for final verification**

Run `ide_diagnostics` on key modified files to catch any remaining issues:
- `WorkItemAssignmentService.java`
- `MultiInstanceSpawnService.java`
- `WorkItemService.java`
- `ExpiryLifecycleService.java`

- [ ] **Step 6: Commit any remaining fixes**

```bash
git commit -m "fix(#287): fix cross-module compilation after NamedStrategy retrofit

Update anonymous SPI implementations and config stubs across integration
tests, examples, and adapter modules.

Refs #287"
```

- [ ] **Step 7: Update javadoc on SPI interfaces**

Use `ide_edit_member` on each SPI interface to update activation instructions:
- Remove references to `@Alternative @Priority`, `@Named`, `@DefaultBean`
- Add references to `NamedStrategy`, `id()`, and config-based selection

Example for `WorkerSelectionStrategy`:
```java
/**
 * Pluggable worker selection SPI.
 *
 * <p>Implementations are CDI beans annotated {@code @ApplicationScoped} that return
 * a unique {@link #id()} string. The active strategy is selected by configuration:
 * {@code casehub.work.routing.strategy=<id>} (default: {@code "least-loaded"}).
 *
 * <p>Built-in implementations:
 * <ul>
 * <li>{@code LeastLoadedStrategy} (id: "least-loaded") — pre-assigns to fewest-active-items candidate (default)
 * <li>{@code ClaimFirstStrategy} (id: "claim-first") — no pre-assignment; whoever claims first wins
 * <li>{@code RoundRobinStrategy} (id: "round-robin") — sequential rotation across candidates
 * </ul>
 */
```

- [ ] **Step 8: Final commit**

```bash
git commit -m "docs(#287): update SPI javadoc for NamedStrategy activation model

Refs #287"
```
