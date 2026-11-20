# UserModel Pattern — Design Spec

**Issue:** casehubio/blocks#122
**Epic:** casehubio/blocks#126 (Autonomous Agent Patterns)
**Date:** 2026-08-20
**Branch:** issue-126-autonomous-agent-patterns

## Summary

UserModel is a per-subject profile synthesis orchestrator in `io.casehub.blocks.agentic.social` that composes existing neocortex memory capabilities (RelationshipEvent, ExperienceRecorder, TrendAnalyzer, CbrCaseMemoryStore) into a structured behavioral profile for anyone an agent interacts with. The profile tracks relationship stage, interaction patterns, and LLM-synthesized open-ended dimensions (communication style, topics of interest, preferences).

The pattern does NOT duplicate neocortex's raw event storage. It fills the gap between "interaction events are recorded" and "the agent has a holistic understanding of this person" by providing tiered synthesis (heuristic for countable dimensions, LLM for open-ended), configurable relationship staging, and CBR-backed profile persistence with temporal versioning.

## Architecture

### Existing Platform Capabilities (composed, not built)

1. **RelationshipEvent / RelationshipQuery / QualitySignal** (neocortex-memory-api) — per-pair relationship event recording with POSITIVE/NEGATIVE/NEUTRAL quality signals, stored to the "relationship" MemoryDomain
2. **ExperienceRecorder / ExperienceQuery** (neocortex-memory-api) — sealed ExperienceEvent (Observation/Action/Outcome) recording with importance scoring and salient retrieval
3. **TrendAnalyzer / TrendProfile** (neocortex-memory-api) — time-series trend detection (slope, volatility, acceleration, change points) on CBR case features
4. **CbrCaseMemoryStore** (neocortex-memory-api) — CBR case storage with similarity-based retrieval, supersession, outcome recording
5. **ContentSummariser / TieredContentSummariser** (blocks/summarisation) — tiered content summarisation with dispatch by batch size
6. **AgentProvider** (platform-agent-api) — LLM invocation for open-ended synthesis

### What UserModel Adds

1. **InteractionSignal SPI** — sealed interface wrapping different signal types (relationship quality, experience events, custom domain signals) into a uniform recording API
2. **Tiered profile synthesis** — heuristic fold for core dimensions (familiarity score, interaction counts, stage transitions) + LLM synthesis for open-ended dimensions (communication style, topics, preferences)
3. **Relationship staging** — continuous familiarity score [0,1] with configurable threshold-to-stage mapping and inactivity decay
4. **UserModelOrchestrator** — CDI bean driving the `record()` + `tick()` cycle with per-subject state management
5. **CBR profile persistence** — profiles stored as CbrCases with dedicated schema, enabling similarity retrieval and temporal versioning via supersession

### Data Flow

```
Domain events → InteractionSignal.of(event)
                      |
record(signal, agentId, subjectId, tenantId)
  → increment counters (positive/negative/neutral)
  → append signal description to text buffer
  → update lastInteractionTimestamp
                      |
tick(agentId, subjectId, tenantId) → UserModelTick
  1. Heuristic fold:
     → compute familiarity score from signal counts + decay
     → resolve relationship stage from thresholds
     → update interaction frequency metrics
  2. LLM synthesis gate:
     → if textBufferSize >= minSignalsForSynthesis AND
        timeSinceLastSynthesis >= synthesisCooldown
     → invoke AgentProvider with accumulated text + current profile
     → parse structured response (topics, style, preferences)
  3. Persist:
     → build UserProfile record
     → store as CbrCase (supersede previous version)
     → clear text buffer, reset counters
  4. Return UserModelTick (Unchanged | Updated | Synthesized)
```

## InteractionSignal SPI

### InteractionSignal

```java
public sealed interface InteractionSignal {
    String description();
    QualitySignal quality();

    record RelationshipSignal(RelationshipEvent event) implements InteractionSignal {
        @Override public String description() { return event.description(); }
        @Override public QualitySignal quality() { return event.qualitySignal(); }
    }

    record ExperienceSignal(ExperienceEvent event, QualitySignal quality)
            implements InteractionSignal {
        @Override public String description() { return event.description(); }
    }

    record CustomSignal(String description, QualitySignal quality)
            implements InteractionSignal {}
}
```

`InteractionSignal` normalises diverse event types into a uniform recording contract. `RelationshipSignal` wraps `RelationshipEvent` directly (quality is already present). `ExperienceSignal` wraps `ExperienceEvent` — the caller provides quality since `ExperienceEvent` doesn't carry a quality signal. `CustomSignal` is an escape hatch for domain-specific signals that don't map to existing event types.

### Design Rationale

The sealed interface serves two purposes:
1. **Type safety** — the orchestrator knows exactly what signal types exist and can pattern-match
2. **Extensibility** — `CustomSignal` lets domains contribute signals without modifying the sealed hierarchy. A future `MoodSignal` or `TaskOutcomeSignal` would become a new permit if usage patterns justify it

## Relationship Stage Model

### RelationshipStageConfig

```java
public record RelationshipStageConfig(
    List<StageTier> tiers,
    double decayRate,
    double positiveWeight,
    double negativeWeight) {

    public static RelationshipStageConfig defaults() {
        return new RelationshipStageConfig(
            List.of(
                new StageTier("stranger", 0.0),
                new StageTier("acquaintance", 0.2),
                new StageTier("familiar", 0.4),
                new StageTier("friend", 0.6),
                new StageTier("confidant", 0.8)),
            0.01,   // decay per tick (1% regression toward 0)
            1.0,    // positive signal weight
            0.5);   // negative signal weight (dampened, D4 precedent)
    }
}
```

### StageTier

```java
public record StageTier(String name, double threshold) {
    public StageTier {
        Objects.requireNonNull(name);
        if (threshold < 0.0 || threshold > 1.0)
            throw new IllegalArgumentException("threshold must be in [0,1]");
    }
}
```

### Familiarity Score Computation

The familiarity score is computed heuristically on each tick:

```java
double rawScore = (positiveCount * positiveWeight - negativeCount * negativeWeight)
                  / (positiveCount + negativeCount + neutralCount + 1);
// Normalise to [0,1]
double normalised = Math.max(0.0, Math.min(1.0, (rawScore + 1.0) / 2.0));
// Apply decay since last tick
double decayed = normalised * Math.pow(1.0 - decayRate, ticksSinceLastInteraction);
```

The `+1` in the denominator prevents division by zero and dampens early scores when signal count is low (Laplace smoothing). The decay factor applies per-tick regression toward 0 when no new interactions occur — inactive relationships naturally regress. `ticksSinceLastInteraction` is computed as `max(0, (now - lastInteractionTimestamp) / expectedTickInterval)` where `expectedTickInterval` is a config parameter (default: 1 hour). This converts wall-clock inactivity into tick-equivalent units regardless of actual tick frequency.

Stage is resolved by finding the highest tier whose threshold is ≤ the decayed score.

## UserProfile

### UserProfile Record

```java
public record UserProfile(
    String agentId,
    String subjectId,
    String tenantId,
    String relationshipStage,
    double familiarityScore,
    int totalInteractions,
    int positiveSignals,
    int negativeSignals,
    int neutralSignals,
    Instant lastInteraction,
    Instant profileCreated,
    Instant lastSynthesised,
    @Nullable String communicationStyle,
    @Nullable String topicsOfInterest,
    @Nullable String preferences,
    @Nullable String synthesisNotes,
    Map<String, String> metadata) {

    public UserProfile {
        Objects.requireNonNull(agentId);
        Objects.requireNonNull(subjectId);
        Objects.requireNonNull(tenantId);
        Objects.requireNonNull(relationshipStage);
        Objects.requireNonNull(lastInteraction);
        Objects.requireNonNull(profileCreated);
        Objects.requireNonNull(metadata);
        metadata = Map.copyOf(metadata);
    }
}
```

**Core fields (heuristic):** `relationshipStage`, `familiarityScore`, signal counts, interaction timestamps. Updated on every tick. Zero LLM cost.

**Open-ended fields (LLM-synthesised):** `communicationStyle`, `topicsOfInterest`, `preferences`, `synthesisNotes`. Updated only when LLM synthesis triggers. Nullable — absent until first synthesis.

**Extensible:** `metadata` map for domain-specific fields not covered by the core schema.

### UserProfileStore SPI

```java
public interface UserProfileStore {
    void store(UserProfile profile);
    Optional<UserProfile> lookup(String agentId, String subjectId, String tenantId);
    List<UserProfile> findByAgent(String agentId, String tenantId);
    void eraseSubject(String subjectId, String tenantId);
}
```

`store()` persists a profile, superseding any previous version for the same (agentId, subjectId, tenantId) triple. `lookup()` retrieves the current profile. `findByAgent()` returns all profiles an agent maintains. `eraseSubject()` removes ALL profiles across all agents for a given subject — the GDPR Art.17 erasure path.

### CbrUserProfileStore (default implementation)

`@DefaultBean @ApplicationScoped` implementation backed by `CbrCaseMemoryStore`. Maps `UserProfile` fields to `FeatureValue` types:

| Profile field | FeatureValue type | CBR feature key |
|---|---|---|
| `subjectId` | `StringVal` | `subject_id` |
| `relationshipStage` | `StringVal` | `relationship_stage` |
| `familiarityScore` | `NumberVal` | `familiarity_score` |
| `totalInteractions` | `NumberVal` | `total_interactions` |
| `positiveSignals` | `NumberVal` | `positive_signals` |
| `negativeSignals` | `NumberVal` | `negative_signals` |
| `neutralSignals` | `NumberVal` | `neutral_signals` |
| `communicationStyle` | `StringVal` | `communication_style` |
| `topicsOfInterest` | `StringVal` | `topics_of_interest` |
| `preferences` | `StringVal` | `preferences` |

The CbrCase `problem` field carries the profile summary text. `solution` is empty. `producerAgentId` is set to `agentId` for agent-scoped retrieval filtering (per GE-20260820-c19b68). The CbrCase convention (problem/solution/features) is contained entirely within this adapter — consumers interact only with `UserProfileStore` and `UserProfile`.

`eraseSubject()` queries all agents' profiles by `subject_id` feature and erases each via `CbrCaseMemoryStore.erase()`. This scanning erasure is acceptable for the GDPR use case (infrequent, batch, correctness-critical).

## Orchestrator

### UserModelOrchestrator

`@ApplicationScoped` CDI bean following the established `record()` + `tick()` pattern.

```java
@ApplicationScoped
public class UserModelOrchestrator {

    UserModelOrchestrator(
        UserProfileStore profileStore,
        AgentProvider agentProvider,
        UserModelConfig config);

    void record(InteractionSignal signal,
                String agentId, String subjectId, String tenantId);

    UserModelTick tick(String agentId, String subjectId, String tenantId);

    @Nullable UserProfile currentProfile(String agentId, String subjectId, String tenantId);
}
```

### record(signal, agentId, subjectId, tenantId)

1. Get or create per-subject state for `(agentId, subjectId, tenantId)`
2. Increment the appropriate signal counter (positive/negative/neutral) based on `signal.quality()`
3. Append `signal.description()` to the text buffer
4. Update `lastInteractionTimestamp`
5. Increment `totalInteractions`

All operations are O(1). No LLM, no store write.

### tick(agentId, subjectId, tenantId)

The periodic synthesis cycle. Synchronous — consuming apps call from their own scheduler.

1. **Acquire per-subject lock.** If no state exists for this subject, return `Unchanged("no signals")`.

2. **Snapshot and clear** signal counters and text buffer under the lock (same pattern as InnerLife step 0).

3. **Heuristic fold:**
   - Merge snapshotted counters into cumulative totals
   - Compute familiarity score using the formula from §Relationship Stage Model
   - Resolve relationship stage from configured tiers
   - If stage or familiarity score unchanged and no LLM synthesis triggered → return `Unchanged`
   - Otherwise → proceed

4. **LLM synthesis gate:**
   - If `textBuffer.length() >= config.minSignalsForSynthesis()` AND `timeSinceLastSynthesis >= config.synthesisCooldown()`:
     - Build prompt with current profile + accumulated signal descriptions
     - Invoke `agentProvider.invoke()`, collect text, parse structured JSON
     - Extract `communicationStyle`, `topicsOfInterest`, `preferences`, `synthesisNotes`
     - On parse failure: log warning, retain previous LLM-synthesised fields

5. **Build UserProfile** from heuristic fields + LLM fields (new or retained from previous).

6. **Persist:**
   - Store via `profileStore.store(profile)` — the store handles supersession internally
   - Update in-memory cached profile

7. **Return outcome:**
   - `Updated(profile)` — heuristic fields changed, no LLM synthesis
   - `Synthesised(profile, previousProfile)` — LLM synthesis ran, carries both for comparison

### currentProfile(agentId, subjectId, tenantId)

Returns the cached in-memory profile, or null if no profile exists. Does not query the store — the cached version is authoritative during the session. On restart, the first `tick()` loads the latest profile from the store.

### Prompt Design

**System prompt:**

```
You are analysing interaction history between an agent and a person to update
a behavioral profile. Given the current profile and recent interactions,
produce an updated assessment.

Respond with JSON only:
{
  "communicationStyle": "<how this person communicates — formal/casual, verbose/terse, etc.>",
  "topicsOfInterest": "<topics this person frequently discusses or cares about>",
  "preferences": "<observed preferences — timing, channel, format, approach>",
  "synthesisNotes": "<notable patterns, changes, or observations>"
}

If insufficient new information exists for a field, repeat the current value unchanged.
Empty string if no data at all.
```

**User prompt assembly:**

```
Current profile:
  Stage: {stage} (familiarity: {score})
  Interactions: {total} ({positive}↑ {negative}↓ {neutral}→)
  Previous communication style: {existing or "not yet assessed"}
  Previous topics: {existing or "not yet assessed"}
  Previous preferences: {existing or "not yet assessed"}

Recent interactions ({count} since last synthesis):
{text buffer — signal descriptions with quality labels}
```

### UserModelTick

```java
public sealed interface UserModelTick {
    record Unchanged(@Nullable String reason) implements UserModelTick {}
    record Updated(UserProfile profile) implements UserModelTick {}
    record Synthesised(UserProfile profile, @Nullable UserProfile previousProfile)
            implements UserModelTick {}
}
```

`Unchanged` — no signals recorded since last tick, or counters changed but stage/score didn't. `Updated` — heuristic fields changed (stage transition, score movement). `Synthesised` — LLM synthesis ran, `previousProfile` provided for comparison.

### Per-Subject State

In-memory, keyed by `agentId:subjectId:tenantId` in `ConcurrentHashMap`:

| Field | Type | Purpose |
|---|---|---|
| `positiveCount` | `int` | Snapshotted positive signals since last tick |
| `negativeCount` | `int` | Snapshotted negative signals since last tick |
| `neutralCount` | `int` | Snapshotted neutral signals since last tick |
| `textBuffer` | `StringBuilder` | Signal descriptions for LLM synthesis |
| `cumulativePositive` | `int` | Total positive signals across all ticks |
| `cumulativeNegative` | `int` | Total negative signals across all ticks |
| `cumulativeNeutral` | `int` | Total neutral signals across all ticks |
| `totalInteractions` | `int` | Total interaction count |
| `lastInteractionTimestamp` | `Instant` | For decay computation |
| `lastSynthesisTimestamp` | `Instant` | For synthesis cooldown gate |
| `lastTickTimestamp` | `Instant` | For decay factor |
| `currentProfile` | `UserProfile` | Cached latest profile |
| `lastActivityTimestamp` | `Instant` | For staleness-based eviction |

**Eviction:** Per-subject state entries not accessed for a configurable duration (default: 7 days) are evicted during `tick()` only. Same eviction pattern as InnerLifeOrchestrator.

**Restart resilience:** On restart, all in-memory state is lost. The first `tick()` for a subject loads the latest profile via `profileStore.lookup()`. Cumulative counters are recovered from the loaded profile's signal counts. LLM-synthesised fields are preserved. The text buffer is lost — this means the first post-restart tick may not trigger LLM synthesis if insufficient new signals have accumulated. Acceptable because the stored profile retains the last synthesis.

### Thread Safety

Per-subject `ReentrantLock` in `ConcurrentHashMap<String, ReentrantLock>`. The lock is acquired for the duration of `tick()` (same pattern as PersonalityEvolutionOrchestrator). `record()` synchronises on the per-subject state object for O(1) counter increments and buffer appends — never blocks on the tick lock.

### Configuration (UserModelConfig)

| Key | Default | Meaning |
|---|---|---|
| `minSignalsForSynthesis` | 5 | Minimum text buffer entries before LLM synthesis |
| `synthesisCooldown` | 1 hour | Minimum time between LLM synthesis runs |
| `decayRate` | 0.01 | Familiarity score decay per tick without interaction |
| `positiveWeight` | 1.0 | Weight of positive signals in familiarity computation |
| `negativeWeight` | 0.5 | Weight of negative signals (dampened, per LLMPTBench D4 precedent) |
| `stageConfig` | `RelationshipStageConfig.defaults()` | Stage tiers and thresholds |
| `expectedTickInterval` | 1 hour | Expected tick frequency — converts wall-clock inactivity to tick units for decay |
| `evictionTimeout` | 7 days | Remove per-subject state not accessed for this duration |
| `memoryDomain` | `"user-model"` | CbrCaseMemoryStore domain |
| `caseType` | `"user-profile"` | CbrCase type identifier |
| `maxObservationsInPrompt` | 50 | Max signal descriptions in LLM prompt |

## TrendAnalyzer Integration

When sufficient profile history exists (3+ superseded versions), `tick()` can optionally enrich the profile with trend metrics:

1. Load recent profile versions via `cbrStore.findSupersededCases()` or sequential retrieval
2. Extract time-series data: familiarity score over time, interaction frequency, positive/negative ratio
3. Run `TrendAnalyzer.analyze()` to compute slope (is the relationship improving or declining?), volatility (is it stable?), and change points (when did something shift?)
4. Store trend metrics as additional FeatureValues on the profile CbrCase

This is a follow-up enhancement, not part of the initial implementation. The profile schema and storage design accommodate it without changes.

## Package Structure

**`io.casehub.blocks.agentic.social`** (additions to existing package)

| Class | What it does |
|---|---|
| `InteractionSignal` | Sealed: `RelationshipSignal`, `ExperienceSignal`, `CustomSignal` |
| `UserProfile` | Record: core fields + LLM-synthesised fields + metadata |
| `UserModelTick` | Sealed: `Unchanged`, `Updated`, `Synthesised` |
| `UserModelOrchestrator` | CDI bean: `record()`, `tick()`, `currentProfile()` |
| `UserModelConfig` | Record with defaults and validation |
| `RelationshipStageConfig` | Record: stage tiers, decay rate, signal weights |
| `StageTier` | Record: `(String name, double threshold)` |
| `UserProfileStore` | SPI: `store()`, `lookup()`, `findByAgent()`, `eraseSubject()` |
| `CbrUserProfileStore` | `@DefaultBean`: CbrCaseMemoryStore-backed UserProfileStore |
| `UserProfileSchema` | Package-private: CbrFeatureSchema for profile storage |
| `SynthesisResult` | Package-private record: parsed LLM output |

## Testing Strategy

All tests are plain JUnit 5 + Mockito (no Quarkus runtime).

| Test class | Coverage |
|---|---|
| `InteractionSignalTest` | Sealed type exhaustiveness, quality extraction, description delegation |
| `RelationshipStageConfigTest` | Default tiers, custom tiers, threshold ordering validation |
| `FamiliarityScoreTest` | Score computation from signal counts, decay over ticks, boundary values, Laplace smoothing |
| `UserModelOrchestratorTest` | Full tick cycle: record signals → tick → Updated/Synthesised. Mock CbrCaseMemoryStore and AgentProvider. Profile persistence via store. Supersession of previous profile. |
| `LlmSynthesisGateTest` | Cooldown enforcement, minimum signal threshold, parse failure graceful degradation |
| `UserProfileTest` | Record validation, metadata immutability, null LLM fields |
| `UserModelTickTest` | Sealed type exhaustiveness, previousProfile in Synthesised |
| `ProfileRecoveryTest` | Restart recovery: load profile from store on first tick after restart |
| `EvictionTest` | Stale per-subject state evicted after timeout |

## References

- `RelationshipEvent.java`, `RelationshipQuery.java`, `QualitySignal.java` (neocortex-memory-api) — per-pair relationship event recording
- `ExperienceRecorder.java`, `ExperienceQuery.java`, `ExperienceEvent.java` (neocortex-memory-api) — experience event recording SPI
- `TrendAnalyzer.java`, `TrendProfile.java` (neocortex-memory-api) — time-series trend detection
- `CbrCaseMemoryStore` (neocortex-memory-api) — CBR case storage and retrieval
- `PersonalityEvolutionOrchestrator` (blocks/agentic/personality) — established record()+tick() pattern
- `InnerLifeOrchestrator` (blocks/agentic/personality) — established observe()+tick() pattern with per-agent state
- `TieredContentSummariser` (blocks/summarisation) — tiered dispatch pattern
- `AgentProvider.invoke()` (platform-agent-api) — LLM invocation pattern
- GE-20260820-c19b68 — CbrQuery has no producerAgentId filter, post-filter required
- GE-20260811-e941cc — AgentDisposition vs DispositionProfile type distinction
- GE-20260820-aa31ab — Composite retention score can mask values
- Research doc §2.4 — UserModel pattern description, composition targets
- "Enabling Personalized Long-term Interactions through Persistent Memory and User Profiles" (2025, arXiv:2510.07925)
- LD-Agent — modular long-term dialogue agent (NAACL 2025)
- Relationship science — perceived partner responsiveness (Smith, Bradbury, Karney, 2025)
- LLMPTBench (NeurIPS 2025) — asymmetric dampening for negative signals (D4 precedent)
