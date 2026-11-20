# MemoryHygiene Pattern — Design Spec

**Issue:** casehubio/blocks#120
**Epic:** casehubio/blocks#126 (Autonomous Agent Patterns)
**Package:** `io.casehub.blocks.memory`

## Overview

Manages memory lifecycle for agents with persistent CBR memory stores: consolidation (merging related memories), forgetting (eviction of low-value memories), importance scoring, temporal versioning (invalidate-not-delete), cross-linking (consolidation provenance), and integrity checking (structural + semantic escalation).

Composes existing infrastructure — no new persistence layers:
- **neocortex**: `CbrCaseMemoryStore`, `CbrRetentionPolicy`, `TemporalDecay`, `ScopeDecay`, `ReflectionOrchestrator`
- **blocks/summarisation**: `ContentSummariser<T>`, `TieredContentSummariser<T>`

## Architecture

Two entry modes:

1. **Tick** — `MemoryHygieneOrchestrator.tick(agentId, tenantId)` → `HygieneTick`. On-demand, bounded cost. Runs: importance scoring → eviction → consolidation (pass 1: summarise). Eviction runs before consolidation to reduce the consolidation workload. Called by consuming apps at their chosen cadence.

2. **Maintain** — `MemoryHygieneScheduler.maintain(agentId, tenantId)` → `MaintenanceTick`. Full idle-time pipeline. Externally composes: tick() + reflection (pass 2) + cross-linking + integrity checks. Called during idle periods (MemGPT "Sleeptime" concept).

The orchestrator has a single entry point (`tick()`), consistent with `PersonalityEvolutionOrchestrator.tick()` and `InnerLifeOrchestrator.tick()`. The scheduler externally composes the full pipeline.

### Pipeline Stages

```
┌─────────────────────────────────────────────────────────────┐
│ tick() — on-demand                                          │
│                                                             │
│  1. Score importance    ImportanceScorer × each memory       │
│  2. Evict              Composite score < threshold → erase  │
│  3. Consolidate (P1)   Merge similar survivors via          │
│     └─ ContentSummariser → FeatureVectorCbrCase             │
│     └─ Supersede sources, annotate source_cases feature     │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│ maintain() — idle-time (scheduler composes externally)      │
│                                                             │
│  4. Reflect (P2)       ReflectionOrchestrator.reflect()     │
│     └─ Store as ReflectionEntry records                     │
│  5. Peer-link          Discover related memories by         │
│     └─ similarity; annotate with StringListVal features     │
│  6. Integrity check    IntegrityChecker → structural        │
│     └─ Escalate flagged anomalies to SemanticIntegrity      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Relationship with Existing Retention

MemoryHygiene **replaces** `CbrRetentionScheduler` for agents that use it. The orchestrator implements its own scan-and-evict loop via `retrieveSimilar` + `erase` — it cannot use `CbrRetentionPolicy.purge()` because purge uses simple filter criteria (maxAgeDays, maxCasesPerType, minTrustScore), not weighted composite scores.

Retrieval-time decay (`TemporalDecayCbrCaseMemoryStore` decorator) and eviction-time decay serve different purposes: retrieval decay modulates ranking during queries; eviction decay determines whether a memory is worth keeping at all. These are independent and intentionally separate.

Agents using MemoryHygiene should disable `CbrRetentionScheduler` for the same domain to avoid conflicting retention decisions.

## Type Catalogue

### Importance Scoring

| Type | Kind | Description |
|------|------|-------------|
| `ImportanceScorer` | `@FunctionalInterface` | `double score(ScoredCbrCase<? extends CbrCase> memory, Instant now)` → [0,1]. Input is the full scored case with text (`problem()`, `solution()`) and `features()`. |
| `ArousalScorer` | class, `@DefaultBean` | Heuristic approximation: word-list sentiment intensity from `CbrCase.problem()` + `CbrCase.solution()` text. Zero LLM cost. Production override with LLM-backed implementation for psychological fidelity. |
| `SurpriseScorer` | class, `@DefaultBean` | Heuristic approximation: information entropy of `CbrCase.features()` relative to agent's typical feature distribution. Zero LLM cost. |
| `CompositeImportanceScorer` | class | Weighted combination: `score = Σ(scorer_i.score() × weight_i) / Σ(weight_i)`. Constructor takes `List<WeightedScorer>`. |
| `WeightedScorer` | record | `(ImportanceScorer scorer, double weight)` — entry in the composite. Weight validated > 0. |

### Retention & Eviction

| Type | Kind | Description |
|------|------|-------------|
| `RetentionScore` | record | `(String caseId, String entityId, double importance, double recencyFactor, double scopeFactor, double trustFactor, double composite)` — full audit trail per memory. `entityId` needed for `EraseRequest`. `composite = (importance × iw + recency × rw + scope × sw + trust × tw) / (iw + rw + sw + tw)` (weighted arithmetic mean, all inputs [0,1]). |
| `RetentionConfig` | record | `retentionThreshold` [0,1] (default 0.1), `importanceWeight` (default 1.0), `recencyWeight` (default 1.0), `scopeWeight` (default 0.5), `trustWeight` (default 0.5). Validates: all weights ≥ 0, at least one > 0. |
| `MemoryHygieneConfig` | interface (`@ConfigMapping`) | `retentionConfig()`, `consolidationBatchSize()` (default 100), `maxReflectionSources()` (default 50), `crossLinkSimilarityThreshold()` (default 0.7), `memoryDomain()` (required), `caseTypes()` (required). |

### Consolidation

Consolidation is a two-step process:
1. **Text synthesis** — `ContentSummariser<ScoredCbrCase<? extends CbrCase>>` produces a `SummaryResult` (text + annotations) from a group of related memories.
2. **Case construction** — The orchestrator builds a `FeatureVectorCbrCase` from the `SummaryResult.text()` (as `problem`), merged features (union of source features), and `source_cases` provenance annotation (as `StringListVal`). This produces a storable `CbrCase`.

Sources are superseded via `CbrCaseMemoryStore.supersede(sourceId, mergedId, "hygiene-consolidation")`.

The tick uses `TieredContentSummariser` dispatch: small groups (≤5) get heuristic merging (feature union, text concatenation), larger groups get LLM-backed synthesis. `FeatureVectorCbrCase` supports `withFeatures()`, so cross-link annotations work without issues.

### Reflection Storage

Reflections from `ReflectionOrchestrator.reflect()` are `List<String>` — abstract insights that don't fit the CbrCase contract (problem/solution/outcome). Stored as lightweight records:

| Type | Kind | Description |
|------|------|-------------|
| `ReflectionEntry` | record | `(String agentId, String tenantId, String insight, Instant generatedAt, List<String> sourceCaseIds)` — a single reflection with provenance. |
| `ReflectionStore` | `@FunctionalInterface` | `void store(ReflectionEntry entry)`. SPI for reflection persistence. |
| `NoOpReflectionStore` | class, `@DefaultBean` | No-op implementation. Consumers override with real persistence. |

### Integrity Checking

| Type | Kind | Description |
|------|------|-------------|
| `IntegrityChecker` | `@FunctionalInterface` | `List<IntegrityViolation> check(String agentId, String tenantId, MemoryDomain domain)` |
| `IntegrityViolation` | record | `(String caseId, ViolationType type, String detail, boolean escalateToSemantic)` |
| `ViolationType` | enum | `ORPHANED_SUPERSESSION` (superseded case points to non-existent superseding case), `DUPLICATE_CASE` (same content stored twice), `MISSING_FEATURES` (required feature keys absent), `UNPROCESSED_STALE` (old memory never reviewed by hygiene pipeline), `SEMANTIC_CONFLICT` (contradictory memories — semantic checker only) |
| `SemanticIntegrityChecker` | `@FunctionalInterface` | `List<IntegrityViolation> checkSemantic(List<IntegrityViolation> flagged, String agentId, String tenantId)` |
| `NoOpSemanticIntegrityChecker` | class, `@DefaultBean` | Returns empty list. Consumers override with LLM-backed implementation. |
| `DefaultIntegrityChecker` | class, `@ApplicationScoped` | Structural checks via `CbrCaseMemoryStore.scan()` + `findSupersededCases()`. Sets `escalateToSemantic=true` on anomalies with high feature overlap but contradictory outcomes. Delegates flagged items to injected `SemanticIntegrityChecker`. |

### Orchestrator & Scheduler

| Type | Kind | Description |
|------|------|-------------|
| `MemoryHygieneOrchestrator` | `@ApplicationScoped` | Single entry: `HygieneTick tick(String agentId, String tenantId)`. Per-agent `ReentrantLock` for tick serialisation. Injects: `CbrCaseMemoryStore`, `ImportanceScorer`, `TemporalDecay`, `ScopeDecay`, `ContentSummariser`, `MemoryHygieneConfig`. |
| `HygieneTick` | sealed interface | `Idle(String reason)` — nothing to do (no memories, or below batch threshold). `Completed(int consolidated, int evicted, int totalScored, List<RetentionScore> scores)` — full audit. `Failed(String reason)` — pipeline error. |
| `MaintenanceTick` | sealed interface | `Completed(HygieneTick hygiene, int reflectionsGenerated, int crossLinksCreated, List<IntegrityViolation> violations)`. `Failed(String stage, String reason)` — which stage failed. |
| `MemoryHygieneScheduler` | class (not CDI) | Constructor: `(MemoryHygieneOrchestrator, ReflectionOrchestrator, ReflectionStore, IntegrityChecker, CbrCaseMemoryStore, MemoryHygieneConfig)`. `MaintenanceTick maintain(String agentId, String tenantId)` — composes: tick() + reflect + cross-link + integrity. Consumer constructs and wires into their scheduler. |

## Tick Pipeline Detail

The tick iterates over each configured `caseType` independently (matching `CbrRetentionScheduler`'s per-caseType loop). Scope is `Path.root()` — hygiene operates at the agent level, not scoped to a specific path.

### Step 1: Importance Scoring

```java
var now = Instant.now();
var domain = new MemoryDomain(config.memoryDomain());

for (var caseType : config.caseTypes()) {
    // Retrieve all active memories for this agent in the configured domain
    // Empty features + minSimilarity(0.0) returns all cases (GE-20260804-eb75e0)
    var query = CbrQuery.of(tenantId, domain, Path.root(), caseType,
            Map.of(), config.consolidationBatchSize())
        .withMinSimilarity(0.0)
        .withProducerAgentId(agentId);
    var memories = store.retrieveSimilar(query, CbrCase.class);

    // Score each memory
    var scored = memories.stream()
        .map(m -> new RetentionScore(
            m.caseId(),
            importanceScorer.score(m, now),
            temporalDecay.factor(m.storedAt(), now),
            scopeDecay.factor(0),  // root scope = 0 distance
            m.cbrCase().trustScore() != null ? m.cbrCase().trustScore() : 1.0,
            computeComposite(config.retentionConfig(), ...)))
        .toList();
    // ... steps 2 and 3 follow per caseType
}
```

### Step 2: Eviction

Eviction runs before consolidation to reduce workload — remove obvious garbage first, then consolidate survivors.

```java
var toEvict = scored.stream()
    .filter(s -> s.composite() < config.retentionConfig().retentionThreshold())
    .toList();
for (var eviction : toEvict) {
    store.erase(new EraseRequest(
        eviction.entityId(), domain, tenantId, eviction.caseId()));
}
var survivors = scored.stream()
    .filter(s -> s.composite() >= config.retentionConfig().retentionThreshold())
    .toList();
```

### Step 3: Consolidation (Pass 1)

Group high-similarity survivors and merge:

1. Identify groups of similar memories (connected components above `crossLinkSimilarityThreshold`)
2. For each group: run `ContentSummariser` → produces `SummaryResult`
3. Build `FeatureVectorCbrCase` from `SummaryResult.text()` + merged features + `source_cases` `StringListVal`
4. Store consolidated case, supersede sources: `store.supersede(sourceId, mergedId, "hygiene-consolidation")`
5. `SupersessionStatus.supersededAt` provides temporal context for when the original memory was valid

## Maintain Pipeline Detail (Scheduler)

### Step 4: Reflection (Pass 2)

The scheduler knows which cases were processed by the tick (returned in `HygieneTick.Completed.scores()`). These scored caseIds are passed as `sourceCaseIds` to give reflections provenance.

```java
var hygieneTick = orchestrator.tick(agentId, tenantId);
var sourceCaseIds = switch (hygieneTick) {
    case HygieneTick.Completed c -> c.scores().stream().map(RetentionScore::caseId).toList();
    default -> List.<String>of();
};

var reflections = reflectionOrchestrator.reflect(agentId, tenantId, since, config.maxReflectionSources());
for (var insight : reflections) {
    reflectionStore.store(new ReflectionEntry(agentId, tenantId, insight, now, sourceCaseIds));
}
```

### Step 5: Peer-Link Discovery

Consolidation provenance (`source_cases` feature) is written during Step 3 as part of the tick. Step 5 discovers additional *peer* relationships — memories that are related but not similar enough to merge. The scheduler queries for pairs above a lower similarity threshold and annotates them with `related_cases` `StringListVal` features. This enriches retrieval context without merging.

### Step 6: Integrity Checking

`DefaultIntegrityChecker` gracefully degrades when `CbrCaseMemoryStore.scan()` is unsupported — it catches `UnsupportedOperationException` and skips scan-dependent checks (DUPLICATE_CASE, MISSING_FEATURES), logging a warning. Supersession-based checks (ORPHANED_SUPERSESSION) use `findSupersededCases()` which is a required method. UNPROCESSED_STALE detection uses `retrieveSimilar` with broad queries.

```java
var violations = integrityChecker.check(agentId, tenantId, domain);
// Structural checks that always work:
// - Orphaned supersessions via findSupersededCases()
// - Unprocessed stale via retrieveSimilar + age check
// Structural checks requiring scan() (graceful degradation):
// - Duplicate cases: same features+outcome within similarity threshold
// - Missing features: required feature keys absent
// Then escalates flagged items to SemanticIntegrityChecker
```

## Observability

Per epic #126 design principle 4 ("Observable — all state changes emit events consumable by existing platform infrastructure"), the orchestrator and scheduler emit `HygieneEvent` records via an injected `Consumer<HygieneEvent>` (no-op default):

| Type | Kind | Description |
|------|------|-------------|
| `HygieneEvent` | sealed interface | `MemoryEvicted(String caseId, RetentionScore score)`, `MemoryConsolidated(String mergedCaseId, List<String> sourceCaseIds)`, `ReflectionGenerated(String agentId, String insight)`, `IntegrityViolationDetected(IntegrityViolation violation)` |

Consumers wire the event sink to their observability infrastructure (platform EventSink, metrics, logging).

## Dependencies

**Compile:** `casehub-neocortex-memory-api` (CbrCaseMemoryStore, TemporalDecay, ScopeDecay, CbrRetentionPolicy, ReflectionOrchestrator, MemoryDomain, CbrCase, CbrQuery, FeatureValue, SupersessionStatus, EraseRequest)
**Compile:** `casehub-blocks` (ContentSummariser, TieredContentSummariser, SummarisationRunner — same module, different package)
**Provided:** `casehub-platform-agent-api` (AgentProvider — for LLM-backed implementations)
**Test:** JUnit 5, Mockito, AssertJ

## Testing Strategy

Plain JUnit 5 with Mockito — no Quarkus runtime (consistent with blocks test conventions).

| Test class | What it covers |
|-----------|----------------|
| `ImportanceScorerTest` | ArousalScorer, SurpriseScorer, CompositeImportanceScorer — boundary values, weight normalisation |
| `RetentionScoreTest` | Composite score computation, weight validation, threshold comparison |
| `MemoryHygieneOrchestratorTest` | Full tick pipeline: mock store → score → consolidate → evict. Per-agent locking. Idle/Completed/Failed outcomes. |
| `MemoryHygieneSchedulerTest` | maintain() composition: tick + reflect + cross-link + integrity. Stage failure isolation. |
| `DefaultIntegrityCheckerTest` | Each ViolationType detected. Semantic escalation for flagged items. |
| `ConsolidationTest` | Supersession of sources, feature merge, StringListVal provenance annotation |

## References

- Research: `docs/research/2026-08-16-autonomous-agent-patterns-landscape.md` §2.2, §2.8
- LUFY: "Should RAG Chatbots Forget Unimportant Conversations?" (2024, arXiv:2409.12524)
- MemGPT / Letta: "Sleeptime Agents" (ICLR 2024; Letta 2025)
- ACT-R memory activation model — power-law decay (recency × frequency)
- Memory OS of AI Agent (EMNLP 2025)
- GE-20260804-eb75e0 — scan() returns CbrCaseSummary without features
- GE-20260716-f292d3 — score-replacing decorators discard pre-applied decay
- GE-20260720-b7a8b9 — eraseEntity() is not domain-scoped
- CbrCaseMemoryStore (neocortex-memory-api)
- TemporalDecay, ScopeDecay (neocortex-memory-api)
- ReflectionOrchestrator (neocortex-memory-api)
- ContentSummariser, TieredContentSummariser (blocks/summarisation)
- PersonalityEvolutionOrchestrator, InnerLifeOrchestrator (blocks/agentic/personality)
- SupersessionStatus (neocortex-memory-api)
