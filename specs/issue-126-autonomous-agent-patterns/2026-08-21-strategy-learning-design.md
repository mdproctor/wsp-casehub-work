# StrategyLearning Design — Multi-Level Reflection on Interaction Strategies

**Issue:** casehubio/blocks#124
**Package:** `io.casehub.blocks.agentic.social`
**Decisions:** D32–D41

## Overview

StrategyLearning is the fifth social cognition orchestrator. It evaluates engagement outcomes, detects what interaction strategies work (per-subject and globally), and produces actionable strategy guidelines for agent prompts.

**Cognitive boundary:** UserModel = perceive ("what IS this person like"). StrategyLearning = learn-and-adapt ("what WORKS with this person and in general"). MentalModel = reason about minds ("what does this person THINK"). PersonalityEvolution = self-model ("how am I CHANGING"). InnerLife = self-expression ("should I SPEAK now").

## Composition

| Component | Source | Role |
|-----------|--------|------|
| `ReflectionOrchestrator` | neocortex-memory-api | Generates reflective insights from accumulated memories (tier 3) |
| `ContentSummariser<EngagementSignal>` | blocks/summarisation | Textual summary of per-conversation engagement (tier 2) |
| `CbrCaseMemoryStore` | neocortex-memory-api | Stores engagement evidence cases with features + dimensional snapshots |
| `TrendAnalyzer` | neocortex-memory-api | Cross-case engagement trend detection (tier 3) |
| `AgentProvider` | platform-agent-api | LLM synthesis for strategy profile updates (tier 3) |
| `EngagementEvent` | neocortex-memory-api | Engagement signal data (responded, responseTimeMs, sentimentShift, etc.) |

## Types

### EngagementSignal (sealed interface)

Input signal type. Two variants. `subjectId` is a method parameter on `record()` (not on the signal) for API consistency with UserModel and MentalModel.

```java
public sealed interface EngagementSignal {

    record TurnOutcome(
        EngagementEvent event,
        Map<String, Double> dimensionalSnapshot,
        @Nullable String responseExcerpt
    ) implements EngagementSignal {
        public TurnOutcome {
            Objects.requireNonNull(event);
            Objects.requireNonNull(dimensionalSnapshot);
            dimensionalSnapshot = Map.copyOf(dimensionalSnapshot);
        }
        // dimensionalSnapshot: agent's current strategy dimensional scores
        //   at time of response (from StrategyProfile.dimensions()).
        //   Provides correlation data: "when verbosity was 0.7, engagement was X."
    }

    record ConversationOutcome(
        String conversationId,
        String conversationSummary,
        int turnCount
    ) implements EngagementSignal {
        public ConversationOutcome {
            Objects.requireNonNull(conversationId);
            if (conversationId.isBlank())
                throw new IllegalArgumentException("conversationId must not be blank");
            if (turnCount < 0)
                throw new IllegalArgumentException("turnCount must be >= 0");
        }
        // Conversation boundary marker — triggers tier 2 case storage
        // conversationId correlates with TurnOutcome.event.caseId()
    }
}
```

**Conversation correlation:** `TurnOutcome.event.caseId()` serves as the conversation identifier. Tier 2 matches TurnOutcomes to ConversationOutcomes via `event.caseId() == conversationOutcome.conversationId()`. This handles concurrent conversations with the same subject.

### StrategyProfile (record)

Output of the orchestrator. Per-agent strategy.

```java
public record StrategyProfile(
    String agentId,
    String tenantId,
    Map<String, Double> dimensions,
    List<String> guidelines,
    Instant lastReflection,
    int evidenceCount
) {
    public StrategyProfile {
        Objects.requireNonNull(agentId);
        Objects.requireNonNull(tenantId);
        Objects.requireNonNull(dimensions);
        Objects.requireNonNull(guidelines);
        dimensions = Map.copyOf(dimensions);
        guidelines = List.copyOf(guidelines);
    }

    public String toPromptSection() {
        if (guidelines.isEmpty()) return "";
        var sb = new StringBuilder("## Interaction Strategy\n\n");
        for (String guideline : guidelines) {
            sb.append("- ").append(guideline).append('\n');
        }
        return sb.toString();
    }
}
```

**Default dimensions:**

| Dimension | Range | Default | Meaning |
|-----------|-------|---------|---------|
| `verbosity` | [0,1] | 0.5 | concise ↔ elaborate |
| `formality` | [0,1] | 0.5 | casual ↔ formal |
| `initiative` | [0,1] | 0.5 | reactive ↔ proactive |
| `directness` | [0,1] | 0.5 | indirect ↔ direct |
| `questionRate` | [0,1] | 0.5 | statements ↔ questions |

`toPromptSection()` provides the recommended injection pattern — consumers prepend this to their system prompt. Example:

```java
String systemPrompt = orchestrator.currentStrategy(agentId, tenantId)
    .map(StrategyProfile::toPromptSection)
    .orElse("") + baseSystemPrompt;
```

### StrategyLearningConfig (record)

```java
public record StrategyLearningConfig(
    int minSignalsForConversationCase,
    int minCasesForReflection,
    int maxReflectionSources,
    int maxGuidelines,
    double defaultDimensionValue,
    int maxBufferSize,
    Duration staleStateTimeout,
    MemoryDomain memoryDomain,
    String engagementCaseType,
    String profileCaseType
) {
    public StrategyLearningConfig {
        if (minSignalsForConversationCase < 1)
            throw new IllegalArgumentException("minSignalsForConversationCase must be >= 1");
        if (minCasesForReflection < 1)
            throw new IllegalArgumentException("minCasesForReflection must be >= 1");
        if (maxReflectionSources < 1)
            throw new IllegalArgumentException("maxReflectionSources must be >= 1");
        if (maxGuidelines < 1)
            throw new IllegalArgumentException("maxGuidelines must be >= 1");
        if (maxBufferSize < 1)
            throw new IllegalArgumentException("maxBufferSize must be >= 1");
        if (defaultDimensionValue < 0.0 || defaultDimensionValue > 1.0)
            throw new IllegalArgumentException("defaultDimensionValue must be in [0,1]");
        if (staleStateTimeout.isNegative() || staleStateTimeout.isZero())
            throw new IllegalArgumentException("staleStateTimeout must be positive");
        Objects.requireNonNull(memoryDomain);
        Objects.requireNonNull(engagementCaseType);
        Objects.requireNonNull(profileCaseType);
    }

    public static StrategyLearningConfig defaults() {
        return new StrategyLearningConfig(
            3, 5, 50, 10, 0.5, 100, Duration.ofHours(24),
            new MemoryDomain("strategy-learning"),
            "engagement-evidence", "strategy-profile");
    }
}
```

| Field | Default | Purpose |
|-------|---------|---------|
| `minSignalsForConversationCase` | 3 | Minimum TurnOutcome signals before storing a conversation case |
| `minCasesForReflection` | 5 | Minimum engagement cases before reflect() produces insights |
| `maxReflectionSources` | 50 | Max CBR cases to feed into reflect() LLM synthesis |
| `maxGuidelines` | 10 | Max guidelines in StrategyProfile |
| `defaultDimensionValue` | 0.5 | Starting value for each dimension |
| `maxBufferSize` | 100 | Max pending signals before dropping oldest |
| `staleStateTimeout` | 24h | Evict agent states with no activity |
| `memoryDomain` | `"strategy-learning"` | MemoryDomain for CBR case storage |
| `engagementCaseType` | `"engagement-evidence"` | CaseType for engagement evidence cases |
| `profileCaseType` | `"strategy-profile"` | CaseType for strategy profile storage |

### StrategyLearningTick (sealed interface — from tick())

Hierarchical: `Learned ⊃ Observed ⊃ NoChange`.

```java
public sealed interface StrategyLearningTick {
    record NoChange(String reason) implements StrategyLearningTick {}

    record Observed(
        int signalsProcessed,
        double engagementRate,
        double meanSentiment
    ) implements StrategyLearningTick {}

    record Learned(
        int signalsProcessed,
        double engagementRate,
        double meanSentiment,
        List<String> conversationsStored,
        int casesStored
    ) implements StrategyLearningTick {}
}
```

### StrategyReflection (sealed interface — from reflect())

```java
public sealed interface StrategyReflection {
    record NoChange(String reason) implements StrategyReflection {}

    record Reflected(
        StrategyProfile profile,
        List<String> newGuidelines,
        TrendProfile trends,
        int evidenceCases
    ) implements StrategyReflection {}
}
```

### StrategyStore (SPI)

```java
public interface StrategyStore {
    void store(StrategyProfile profile);
    Optional<StrategyProfile> lookup(String agentId, String tenantId);
    List<String> subjectInsights(String agentId, String subjectId, String tenantId);
    void eraseAgent(String agentId, String tenantId);
    void eraseSubject(String subjectId, String tenantId);
}
```

`CbrStrategyStore` (`@DefaultBean`) backs onto `CbrCaseMemoryStore`:
- Profile stored as CbrCase: guidelines as problem text, dimensions as features, placeholder solution (`"-"`)
- `subjectInsights()` queries engagement CBR cases filtered by subjectId (via `CbrFilter.contains("subjectId", subjectId)`), extracts engagement features, formats as template-based text: `"With {subjectId}: engagement rate {continuationRate}, avg response length {avgResponseLength}, sentiment trend {meanSentimentShift}"`
- `eraseSubject()` queries engagement CBR cases by subjectId feature filter, erases matching cases via `cbrStore.erase()`. Provides GDPR Art.17 erasure for subject data.

## Orchestrator

### StrategyLearningOrchestrator

Four entry points: `record()`, `tick()`, `reflect()`, `currentStrategy()`.

**Constructor dependencies:**
- `StrategyStore strategyStore`
- `CbrCaseMemoryStore cbrStore` — for engagement evidence cases
- `ReflectionOrchestrator reflectionOrchestrator` — tier 3 reflection
- `AgentProvider agentProvider` — tier 3 LLM synthesis
- `@Nullable ContentSummariser<EngagementSignal> summariser` — optional tier 2 text summary
- `StrategyLearningConfig config`

Package-private constructor adds `Clock clock` for testability (following MentalModelOrchestrator pattern).

**Internal state:** `ConcurrentHashMap<String, AgentLearningState>` keyed by `agentId:tenantId`.

```
AgentLearningState:
  - pendingTurns: ArrayDeque<TurnOutcomeEntry>  (bounded by maxBufferSize)
      TurnOutcomeEntry: (TurnOutcome signal, String subjectId)
  - pendingConversations: ArrayDeque<ConversationOutcomeEntry>
      ConversationOutcomeEntry: (ConversationOutcome signal, String subjectId)
  - totalSignals: int
  - totalResponded: int
  - sentimentSum: double
  - lastSignalTimestamp: Instant
  - lastTickTimestamp: Instant
  - lastReflectTimestamp: Instant
  - lastActivityTimestamp: Instant
  - currentProfile: StrategyProfile (loaded from store on first access)
```

### record(EngagementSignal signal, String agentId, String subjectId, String tenantId)

O(1). Appends to pending queue with subjectId. If `TurnOutcome`: increments counters, appends to pendingTurns. If `ConversationOutcome`: appends to pendingConversations. Follows the 4-parameter pattern of UserModel and MentalModel.

### tick(String agentId, String tenantId) → StrategyLearningTick

Cheap. Runs tiers 1-2 under per-agent `ReentrantLock`.

**Tier 1 (always, if pending turns exist):**
1. Drain pending TurnOutcome entries into a local list (snapshot — not discarded)
2. Update aggregate counters: totalSignals, totalResponded, sentimentSum
3. Compute engagement metrics: `engagementRate = totalResponded / totalSignals`, `meanSentiment = sentimentSum / totalSignals`
4. If no pending conversations → return `Observed`

**Tier 2 (if pending conversations exist or accumulated turn count ≥ minSignalsForConversationCase):**
1. Drain pending ConversationOutcome entries
2. For each conversation, collect TurnOutcome entries from the tier-1 local list matching `entry.signal().event().caseId() == conversation.conversationId()` AND `entry.subjectId() == conversation.subjectId()`
3. Extract structured features from matched TurnOutcome signals:
   - `avgResponseLength`: mean of `event.responseLength()` (null-safe, skip nulls)
   - `continuationRate`: fraction where `event.continued() == true` (null-safe)
   - `meanSentimentShift`: mean of `event.sentimentShift()` (null-safe)
   - `avgSnapshot_<dim>`: mean of snapshot values per dimension
   - `turnCount`: number of matched signals
   - `conversationTimestamp`: epoch millis of first signal (for temporal ordering)
4. Optionally run `ContentSummariser<EngagementSignal>` for text summary (stored as StringVal feature)
5. Store CbrCase via `cbrStore.store()` with:
   - cbrCase: FeatureVectorCbrCase(problem=conversationSummary, solution="-", features=extracted, producerAgentId=agentId)
   - caseType: `config.engagementCaseType()`
   - domain: `config.memoryDomain()`
   - scope: `Path.root()`
6. Return `Learned` with tier 1 + tier 2 data

### reflect(String agentId, String tenantId) → StrategyReflection

Expensive. Runs tier 3 under per-agent `ReentrantLock`.

**Tier 3:**
1. Load current StrategyProfile from store (or use defaults if none)
2. Query engagement CBR cases:
   ```java
   CbrQuery.of(tenantId, config.memoryDomain(), Path.root(),
       config.engagementCaseType(), Map.of(), config.maxReflectionSources())
       .withMinSimilarity(0.0)
       .withFilter("agentId", CbrFilter.contains(agentId))
   ```
   Uses CbrFilter for server-side agentId filtering (avoids topK truncation before filter).
3. If cases < `minCasesForReflection` → return `NoChange("insufficient evidence")`
4. **Cross-case trend analysis via TrendAnalyzer:**
   - Collect retrieved case feature maps into `List<Map<String, FeatureValue>>`
   - Sort by `conversationTimestamp` feature (ascending)
   - Construct a programmatic `FeatureField.TimeSeries` schema with the numeric engagement features as inner fields and `conversationTimestamp` as the timestamp field
   - Apply `TrendSpec` with types: `SLOPE`, `DELTA`, `VOLATILITY`
   - Call `TrendAnalyzer.analyze(observations, schema)` → `TrendProfile`
   - Note: this is cross-case analysis (treating each case as an observation in a time series), NOT intra-case TimeSeries analysis. The schema is constructed at reflect() time, not declared on the stored cases.
5. Run `ReflectionOrchestrator.reflect(agentId, tenantId, lastReflectTimestamp, maxReflectionSources)` → reflective insights
6. Build LLM synthesis prompt (see below)
7. Invoke `agentProvider`:
   ```java
   var config = AgentSessionConfig.of(SYSTEM_PROMPT, userPrompt);
   String response = agentProvider.invoke(config)
       .filter(e -> e instanceof AgentEvent.TextDelta)
       .map(e -> ((AgentEvent.TextDelta) e).text())
       .collect().with(Collectors.joining())
       .await().atMost(Duration.ofMinutes(2));
   ```
8. Parse LLM JSON output. On parse failure: log warning, return `NoChange("parse failure")`, retain current profile (following UserModelOrchestrator precedent). Validation:
   - Malformed JSON → NoChange
   - Unknown dimension keys in deltas → ignore, log warning
   - Delta values outside [-0.2, +0.2] → clamp to range
   - Empty guidelines list → retain previous guidelines
9. Apply valid dimensional deltas to current scores (clamped to [0,1])
10. Build updated StrategyProfile
11. Store via `strategyStore.store()`
12. Return `Reflected` with new profile, guidelines, trends

### currentStrategy(String agentId, String tenantId) → Optional<StrategyProfile>

Returns in-memory profile if loaded, otherwise queries `strategyStore.lookup()`. Consistent with `UserModelOrchestrator.currentProfile()`.

### CBR Feature Schema

Engagement evidence cases use flat `FeatureField.Numeric` fields (NOT `FeatureField.TimeSeries` — the cases are flat records, not embedded time series). Cross-case trend analysis constructs a programmatic TimeSeries schema in reflect().

Package-private `EngagementCaseSchema` class defines the `CbrFeatureSchema`:

| Feature | Type | Range | Purpose |
|---------|------|-------|---------|
| `subjectId` | StringVal | — | Per-subject filtering via CbrFilter |
| `agentId` | StringVal | — | Per-agent filtering via CbrFilter |
| `conversationTimestamp` | NumberVal | [0, ∞] | Epoch millis for temporal ordering |
| `turnCount` | NumberVal | [1, 1000] | Conversation length |
| `avgResponseLength` | NumberVal | [0, 10000] | Mean response length in chars |
| `continuationRate` | NumberVal | [0, 1] | Fraction of turns where subject continued |
| `meanSentimentShift` | NumberVal | [-1, 1] | Mean sentiment change |
| `avgSnapshot_verbosity` | NumberVal | [0, 1] | Mean agent verbosity score during conversation |
| `avgSnapshot_formality` | NumberVal | [0, 1] | Mean agent formality score |
| `avgSnapshot_initiative` | NumberVal | [0, 1] | Mean agent initiative score |
| `avgSnapshot_directness` | NumberVal | [0, 1] | Mean agent directness score |
| `avgSnapshot_questionRate` | NumberVal | [0, 1] | Mean agent question rate |
| `conversationSummary` | StringVal | — | Optional text summary from ContentSummariser |

Schema registered via `cbrStore.registerSchema()` at construction time (if available) or lazily on first store.

### LLM Synthesis Prompt (tier 3)

```
You are a metacognitive strategy advisor for an AI agent. Analyze the agent's
interaction history and recommend strategy adjustments.

Current strategy dimensions:
{dimensions as key=value list}

Current guidelines:
{numbered guideline list, or "None yet" if first reflection}

Engagement trend analysis:
{TrendProfile metrics — slope, delta, volatility per feature}

Reflective insights:
{ReflectionOrchestrator output — list of reflective observations}

Per-subject engagement patterns:
{grouped by subjectId: engagement rate, mean sentiment, notable patterns}

Based on this evidence:
1. List up to {maxGuidelines} ranked strategy guidelines (most impactful first).
   Include per-subject insights where patterns differ significantly from global.
2. Suggest dimensional adjustments as JSON: {"verbosity": delta, "formality": delta, ...}
   where delta is in [-0.2, +0.2]. Only include dimensions that should change.

Respond as JSON:
{"guidelines": ["...", ...], "dimensionDeltas": {"verbosity": -0.1, ...}}
```

### Strategy Retrieval

Consumers inject strategy into agent prompts via `StrategyProfile.toPromptSection()`:

```java
// In consumer's PromptAssembler or system prompt builder:
String strategy = orchestrator.currentStrategy(agentId, tenantId)
    .map(StrategyProfile::toPromptSection)
    .orElse("");
String subjectContext = strategyStore
    .subjectInsights(agentId, subjectId, tenantId).stream()
    .map(s -> "- " + s)
    .collect(Collectors.joining("\n", "### Subject-specific:\n", "\n"));
String systemPrompt = strategy + subjectContext + baseSystemPrompt;
```

## Type Summary

| Type | Kind | Description |
|------|------|-------------|
| `EngagementSignal` | sealed interface | Input signal: TurnOutcome, ConversationOutcome |
| `StrategyProfile` | record | Per-agent strategy: dimensions + guidelines + toPromptSection() |
| `StrategyLearningConfig` | record | Configuration with full validation |
| `StrategyLearningTick` | sealed interface | tick() outcome: NoChange, Observed, Learned |
| `StrategyReflection` | sealed interface | reflect() outcome: NoChange, Reflected |
| `StrategyStore` | interface (SPI) | Strategy profile + subject erasure persistence |
| `CbrStrategyStore` | class (@DefaultBean) | CbrCaseMemoryStore-backed StrategyStore |
| `StrategyLearningOrchestrator` | class | Composition root: record() + tick() + reflect() + currentStrategy() |

8 new types. ~500 lines of production code estimated.

## Testing Strategy

| Test | What it verifies |
|------|-----------------|
| EngagementSignal validation | TurnOutcome requires non-null event/snapshot; ConversationOutcome requires non-blank conversationId |
| StrategyProfile defaults | Default dimensions are all 0.5, empty guidelines |
| StrategyProfile.toPromptSection() | Formats guidelines as markdown list; empty guidelines returns empty string |
| StrategyLearningConfig validation | Rejects all 7 invalid field values |
| CbrStrategyStore round-trip | store() → lookup() returns equivalent profile; eraseAgent() clears |
| CbrStrategyStore subjectInsights | Returns template-formatted insights filtered by subjectId |
| CbrStrategyStore eraseSubject | Erases engagement cases matching subjectId |
| record() API consistency | Takes (signal, agentId, subjectId, tenantId) — 4 params like siblings |
| tick() with no signals | Returns NoChange |
| tick() with TurnOutcome signals | Returns Observed with correct engagement metrics |
| tick() signal retention | Tier 1 drains into local list; tier 2 reads same list (no signal loss) |
| tick() with ConversationOutcome | Returns Learned, stores CBR case with correct features |
| tick() conversation correlation | Matches TurnOutcomes to ConversationOutcomes via caseId/conversationId |
| tick() concurrent conversations | Two conversations with same subject stored as separate cases |
| tick() feature extraction | avgResponseLength, continuationRate, meanSentimentShift computed correctly |
| tick() dimensional snapshot averaging | avgSnapshot_* features correctly averaged across signals |
| tick() null-safe extraction | Null responseLength/sentimentShift/continued fields skipped |
| tick() CBR store parameters | Correct caseType, domain, scope Path.root() passed |
| reflect() with insufficient cases | Returns NoChange |
| reflect() CbrFilter query | Uses CbrFilter.contains("agentId", agentId) for server-side filtering |
| reflect() cross-case TrendAnalyzer | Programmatic TimeSeries schema, sorted by conversationTimestamp |
| reflect() LLM synthesis | Guidelines extracted, dimensions adjusted, profile stored |
| reflect() dimensional clamping | Deltas outside [-0.2, +0.2] clamped; final scores clamped to [0,1] |
| reflect() parse failure | Malformed JSON → NoChange, previous profile retained |
| reflect() unknown dimension keys | Ignored with warning |
| reflect() empty guidelines | Previous guidelines retained |
| reflect() per-subject grouping | Cases grouped by subjectId for per-subject analysis |
| Concurrency | Concurrent record() + tick() on same agent — no lost signals |
| Stale state eviction | Agent states evicted after staleStateTimeout (via Clock injection) |
| currentStrategy() | Returns in-memory profile or loads from store |
| Clock injection | Package-private constructor accepts Clock for deterministic testing |

~48 tests estimated.

## References

- Research §2.6 (StrategyLearning pattern definition)
- Metacognitive Learning (ICML 2025, Liu & Van Der Schaar)
- Self-Learning Agents Enhanced by Multi-level Reflection (EMNLP 2025)
- Reflexion: Language Agents with Verbal Reinforcement Learning (NeurIPS 2023, Shinn et al.)
- MIRROR: Cognitive Inner Monologue Between Conversational Turns (2025, arXiv:2506.00430)
- EngagementEvent (neocortex-memory-api: `io.casehub.neocortex.memory.engagement`)
- ReflectionOrchestrator (neocortex-memory-api: `io.casehub.neocortex.memory.reflection`)
- TrendAnalyzer / TrendProfile (neocortex-memory-api: `io.casehub.neocortex.memory.cbr`)
- ContentSummariser (blocks: `io.casehub.blocks.summarisation`)
- UserModelOrchestrator (blocks: `io.casehub.blocks.agentic.social`) — tiered pattern precedent
- MemoryHygieneScheduler (blocks: `io.casehub.blocks.memory`) — tick()+maintain() dual-cadence precedent
- CbrUserProfileStore / CbrMentalModelStore — CbrCase adapter pattern precedent
- GE-20260820-c19b68 (producerAgentId post-filtering)
- GE-20260804-eb75e0 (scan() returns summaries — use retrieveSimilar)
- GE-20260820-d4e011 (non-blank solution placeholder)
- GE-20260813-keyed-summarisation-runner-api (KeyedSummarisationRunner API differences)
- Decisions D32–D41 in `decisions.md`
- Decision review R1-01 through R1-11
- Spec review R1-02 through R1-16
