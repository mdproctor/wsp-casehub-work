# PersonalityEvolution Pattern — Design Spec

**Issue:** casehubio/blocks#118
**Date:** 2026-08-18
**Branch:** issue-126-autonomous-agent-patterns

## Summary

PersonalityEvolution is a signal-driven orchestrator in `io.casehub.blocks.agentic.personality` that composes existing eidos and neocortex capabilities into a bounded personality drift feedback loop. Interaction outcomes (behavioral compliance, relationship quality, goal achievement) nudge personality traits within configurable ranges over time.

The pattern does NOT duplicate eidos's JPAF evaluation logic. It fills the gap between "interaction outcomes happen" and "the JPAF machinery processes them" by providing signal translation, periodic orchestration, safety rails, and outcome-based learning through CBR case recording.

## Architecture

### Existing Platform Capabilities (composed, not built)

1. **DispositionSignalStore** (eidos-api) — accumulates activation counts per cognitive function per agent, with decay support
2. **DefaultDispositionHealth.probe()** (eidos-runtime) — computes effective weights from `baseWeight + activationCount × Δw`, detects evolution thresholds (auxiliary surpasses dominant, shadow surpasses primary, structural reorganization)
3. **DefaultDispositionEvolution.evaluate()** (eidos-runtime) — applies 4 JPAF decision rules (dominant swap, dominant replacement, auxiliary replacement, structural reorganization) and normalizes weights
4. **CbrCaseMemoryStore** (neocortex-memory-api) — stores and retrieves CBR cases with outcome weighting, similarity-based retrieval, and supersession support
5. **PersonalityTransitionSchema** (neocortex-memory-api) — pre-defined CBR schema for personality transition cases: agent_id, old/new dominant, old/new auxiliary, trigger_type, outcome
6. **TrendAnalyzer** (neocortex-memory-api) — detects trends (slope, volatility, acceleration, change points) in time-series data within CBR cases

### What PersonalityEvolution Adds

1. **Signal translation SPI** — `TraitPressureSource<E>` maps domain events to disposition function activations using a two-layer strategy
2. **Orchestrator** — `PersonalityEvolutionOrchestrator` drives the periodic tick cycle: decay → probe → evaluate → persist
3. **Safety rails** — L2 displacement ceiling with halt flag, probe-time asymmetric dampening for negative signals
4. **CBR transition recording** — records personality transitions as CBR cases using `PersonalityTransitionSchema`, with outcome tracking for feedback learning

### Data Flow

```
Domain events → TraitPressureSource.translate() → DispositionSignalStore.recordActivation(valence)
                                                          |
tick() → DispositionSignalStore.decay() → DispositionHealth.probe()
              |                                    |
         L2 ceiling check              EvolutionPending?
              |                           | yes
         halt flag <--------  DispositionEvolution.evaluate()
                                    |
                              Evolved → persist new profile, record CBR case, clear signals
                              Dampened → decay signals, set halt flag
```

## Signal Translation SPI

### TraitPressureSource

```java
public interface TraitPressureSource<E> {
    Class<E> eventType();
    List<TraitActivation> translate(E event, AgentDescriptor descriptor);
}
```

Returns zero or more `TraitActivation` records. Zero means the event is irrelevant to this agent's personality. The SPI receives the full `AgentDescriptor` for access to `dispositionProfile`, `dispositionVocabulary`, and `agentId`/`tenancyId`.

`eventType()` enables runtime dispatch: `record()` matches incoming events to the source whose `eventType()` returns the event's class. This replaces CDI type-parameter inspection, which is erased at runtime.

### TraitActivation

```java
public record TraitActivation(String functionTerm, SignalValence valence) {}
```

The `functionTerm` must match a term in the agent's `dispositionProfile`. Activations with unrecognised terms are silently dropped (defensive — profile may have changed between event and recording).

**Valence targeting contract:** POSITIVE and NEGATIVE activations for a single event MUST target different function terms. POSITIVE activations target the function being naturally expressed. NEGATIVE activations target the compensating function experiencing pressure. This is enforced by the two-layer mapping design (D2), not by runtime validation.

### SignalValence

```java
public enum SignalValence { POSITIVE, NEGATIVE }
```

Lives in eidos-api (shared between blocks and eidos). POSITIVE signals reinforce a function. NEGATIVE signals indicate compensatory pressure — the function is being exercised reactively rather than naturally. At probe time, negative activations are weighted by the dampening factor (D4). Both valences represent activation (function exercise) — they differ in quality, not direction.

### Two-Layer Mapping (D2)

**Layer 1 — Profile-term-aware (universal):** The implementation reads `descriptor.disposition().dispositionProfile()` to get available terms. It maps the domain event to one or more of those terms. This works for ANY vocabulary framework — Big Five agents activate "openness"/"conscientiousness", DISC agents activate "dominance"/"influence", Jungian agents activate "ti"/"ne". Every agent with a populated `dispositionProfile` participates in personality evolution.

**Layer 2 — Vocabulary-structural (enrichment):** For agents whose vocabulary supports structural navigation (via `VocabularyTerm.opposite()` returning the shadow/counter function), the implementation additionally infers indirect activations. Example: a negative event on Ti also activates shadow Fe. This enrichment requires vocabulary-specific structural knowledge not derivable from profile terms alone. Agents without structural vocabulary simply skip Layer 2 — they still get Layer 1 evolution.

### Default Implementations

| Implementation | Event type | Mapping logic |
|---|---|---|
| `BehavioralSignalPressureSource` | `BehavioralSignal` | SUCCESS/COMPLIANT → POSITIVE activation of dominant function. DECLINE/VIOLATED → NEGATIVE activation of compensating function (vocabulary-structural if available). |
| `RelationshipPressureSource` | `RelationshipEvent` | POSITIVE quality → POSITIVE activation of the profile's highest-weighted function (dominant). NEGATIVE quality → NEGATIVE activation of the second-highest function (auxiliary). NEUTRAL quality → zero activations (no personality pressure from neutral events). When vocabulary structural navigation is available, NEGATIVE also activates the dominant's opposite (shadow). |
| `GoalOutcomePressureSource` | `GoalOutcomeCounts` | Success rate above configurable threshold (default 0.7) → POSITIVE activation of dominant. Success rate below configurable floor (default 0.3) → NEGATIVE activation of auxiliary. Between floor and threshold → no activation. |

Consumer repos provide domain-specific implementations for events not covered by the defaults (e.g., conversation deadlock → activate conflict-mode function).

## Orchestrator

### PersonalityEvolutionOrchestrator

`@ApplicationScoped` CDI bean. The orchestrator follows a periodic `tick()` cadence pattern — consuming apps call `tick()` at their chosen interval to drive the decay→probe→evaluate cycle. Unlike `SummarisationRunner` (plain class, synchronous, per-stream, async via CompletionStage), this orchestrator is a CDI singleton managing per-agent state with synchronous evaluation. The shared concept is periodic caller-driven processing; the implementation differs because personality evolution is stateful across agents and synchronous (evaluation is CPU-bound, not I/O-bound).

```java
@ApplicationScoped
public class PersonalityEvolutionOrchestrator {

    PersonalityEvolutionOrchestrator(
        DispositionSignalStore signalStore,
        DispositionHealth health,
        DispositionEvolution evolution,
        DispositionProfileStore profileStore,
        CbrCaseMemoryStore cbrStore,
        Instance<TraitPressureSource<?>> pressureSources);

    <E> void record(E event, AgentDescriptor descriptor);

    EvolutionTick tick(AgentDescriptor descriptor, CapabilityHealth.ProbeContext probeContext);
}
```

### record(event, descriptor)

1. Check halt flag for this agent — if set, skip (L2 ceiling exceeded, recording paused)
2. Find matching `TraitPressureSource<E>` by comparing `event.getClass()` against each source's `eventType()` (first match)
3. Call `translate(event, descriptor)` → `List<TraitActivation>`
4. For each activation, call `signalStore.recordActivation(agentId, tenancyId, functionTerm, valence)`

### tick(descriptor, probeContext)

The periodic cycle. Consuming apps call this at their chosen cadence.

**D3 — Tick cadence:** Consuming apps are responsible for scheduling tick() calls. Recommended cadence: once per significant interaction batch (e.g., end of conversation turn, daily for long-running agents). Too frequent → unnecessary evaluation overhead. Too infrequent → delayed drift detection. Cadence should match the interaction rate — a Discord bot in an active channel ticks more often than a weekly email responder.

**D6 — Tick trigger policy:** Consuming apps choose between time-driven (fixed interval via `JobScheduler`) and event-driven (tick after N recorded events). Time-driven is simpler and sufficient for most cases. Event-driven suits bursty interaction patterns where long idle periods should not trigger unnecessary ticks.

Steps:

1. **Decay:** `signalStore.decay(agentId, tenancyId, decayFactor)`
2. **Probe:** `health.probe(descriptor, probeContext)` → `DispositionStatus`
3. **Process result** (D5 halt flag state machine):

| DispositionStatus | Condition | Action | Return |
|---|---|---|---|
| `Aligned` | — | Clear halt flag | `EvolutionTick.Stable` |
| `Drifted` | `magnitude < l2Ceiling` | Clear halt flag | `EvolutionTick.Drifting(magnitude)` |
| `Drifted` | `magnitude >= l2Ceiling` | Set halt flag | `EvolutionTick.Halted(magnitude)` |
| `EvolutionPending` | `Evolved` result | Persist new profile via `profileStore.update()`, record CBR transition case, `signalStore.clear()`, clear halt flag | `EvolutionTick.Evolved(prev, new, newProfile)` |
| `EvolutionPending` | `Dampened` result | `signalStore.decay(factor)`, set halt flag | `EvolutionTick.Dampened(factor)` |

Note: `probe()` returns `Drifted` (not `EvolutionPending`) when the dominant exceeds `overReinforcementThreshold`, so `EvolutionPending` fires only when displacement has crossed an evolution threshold but NOT the over-reinforcement ceiling.

Note: The `Dampened` path sets the halt flag to pause recording while dampening decay takes effect. Without this, new signals arriving between ticks would counteract the dampening decay, causing oscillation between EvolutionPending and Dampened. The halt flag is cleared on the next tick when displacement has fallen (Aligned or sub-ceiling Drifted).

### EvolutionTick

Sealed return type giving callers precise information about what happened:

```java
public sealed interface EvolutionTick {
    record Stable() implements EvolutionTick {}
    record Drifting(double magnitude) implements EvolutionTick {}
    record Halted(double magnitude) implements EvolutionTick {}
    record Evolved(String previousTypeLabel, String newTypeLabel,
                   List<DispositionValue> newProfile) implements EvolutionTick {}
    record Dampened(double decayFactor) implements EvolutionTick {}
}
```

`Evolved` carries the full `newProfile` for observability and downstream consumers. Field names align with `DispositionEvolution.EvolutionResult.Evolved`.

### Thread Safety

Per-agent locking via `ConcurrentHashMap<String, ReentrantLock>`. The lock is acquired for the duration of `tick()` to prevent concurrent ticks for the same agent from causing double-decay, race-on-Evolved, or inconsistent halt flag state. `record()` uses best-effort check-then-act on the halt flag (an `AtomicBoolean` per agent) — a race between halt check and recording is acceptable since the next `tick()` corrects it.

**Restart resilience:** Halt flags and per-agent locks are in-memory (`@ApplicationScoped` lifecycle). On application restart, all halt flags are lost and recording resumes for all agents. This is acceptable because the next `tick()` re-evaluates displacement — if the L2 ceiling is still exceeded, the halt flag is immediately re-set. The vulnerability window is bounded by one tick interval, and given the slow drift design (changes over weeks), a brief period of unhalted recording has negligible effect. For the same reason, cluster coordination is not needed — CaseHub agents are single-tenant per application instance.

### Configuration

Via platform `PreferenceProvider`, per-tenant. Preference keys are defined in `PersonalityEvolutionConfig`:

| Key | Preference path | Default | Meaning |
|---|---|---|---|
| `decayFactor` | `casehub.blocks.personality.decay-factor` | 0.8 | Retention fraction per tick (0.8 = 20% decay per tick) |
| `l2Ceiling` | `casehub.blocks.personality.l2-ceiling` | 0.15 | Maximum L2 displacement from baseline before halt |
| `dampeningFactor` | `casehub.blocks.personality.dampening-factor` | 0.5 | Weight multiplier for NEGATIVE activations at probe time |

Cascaded attenuation with defaults: an agent receiving one negative activation per tick contributes `1 × 0.5 (dampening) × 0.06 (reinforcementDelta) = 0.03` to effective weight per tick, decaying by 20% each subsequent tick. This produces slow, bounded drift as intended.

### `CapabilityHealth.ProbeContext` Usage

`tick()` receives a `CapabilityHealth.ProbeContext` (nested record in `CapabilityHealth`). For personality evolution probing:
- `taskDomain`: use `"personality-evolution"` — distinguishes these probes from capability health probes
- `taskMetadata`: optionally carry tick context (e.g., `Map.of("tickSource", "scheduled")` or `Map.of("tickSource", "event-driven", "eventCount", "12")`)

Consuming apps construct the context — the orchestrator passes it through to `health.probe()` without interpretation.

## CBR Transition Recording

When `tick()` produces an `Evolved` result, the orchestrator records the transition as a CBR case:

```java
CbrCase transitionCase = CbrCase.of(PersonalityTransitionSchema.schema(), Map.of(
    "agent_id", FeatureValue.categorical(descriptor.agentId()),
    "old_dominant", FeatureValue.categorical(evolved.previousTypeLabel()),
    "new_dominant", FeatureValue.categorical(evolved.newTypeLabel()),
    "old_auxiliary", FeatureValue.categorical(previousAuxiliary),
    "new_auxiliary", FeatureValue.categorical(newAuxiliary),
    "trigger_type", FeatureValue.categorical(triggerType)
));
cbrStore.store(transitionCase, descriptor.agentId(), descriptor.tenancyId(),
               MemoryDomain.AGENT, "personality-evolution", null, null);
```

### Outcome Recording

The `outcome` field in `PersonalityTransitionSchema` is recorded asynchronously after the transition has had time to take effect. Consuming apps call `cbrStore.recordOutcome(caseId, tenancyId, outcome)` when they can assess whether the transition improved the agent's performance (e.g., better engagement, more natural behavior, improved goal completion). This closes the feedback loop — future transitions can consult similar past cases to inform dampening decisions.

### Supersession

When an agent undergoes a second evolution that supersedes a recent transition (e.g., reverting back), the previous transition case is superseded via `cbrStore.supersede()`. This prevents stale transitions from polluting similarity queries.

## Eidos SPI Enhancement

`DispositionSignalStore` gains two default methods for valence-aware storage. All new methods have defaults, so existing implementations (JPA, InMemory, NoOp) continue unchanged.

### New Types (eidos-api)

```java
// SignalValence enum (shared)
public enum SignalValence { POSITIVE, NEGATIVE }

// ValenceCounts record
public record ValenceCounts(int positive, int negative) {
    public int effective(double dampeningFactor) {
        return positive + (int) Math.round(negative * dampeningFactor);
    }
}
```

**`effective()` semantics:** Both positive and negative activations represent function exercise — positive through natural expression, negative through compensatory pressure. The formula is additive: full credit for positive activations plus dampened credit for negative activations. `dampeningFactor` (0.0–1.0) controls how much compensatory exercise contributes relative to natural exercise. A factor of 0.5 means compensatory activations count at half strength.

This is correct because the valence targeting contract (§Signal Translation SPI) ensures positive and negative activations target different function terms. A given function receives either positive or negative activations for any single event, not both. The additive formula reflects that both forms of exercise increase the function's effective weight — compensatory exercise just does so at a reduced rate, preventing the exaggerated personality shifts under negative events identified by LLMPTBench.

### DispositionSignalStore Additions

```java
public interface DispositionSignalStore {
    // Existing — unchanged
    void recordActivation(String agentId, String tenancyId, String functionTerm);
    Map<String, Integer> activationCounts(String agentId, String tenancyId);
    void decay(String agentId, String tenancyId, double decayFactor);
    void clear(String agentId, String tenancyId);

    // New — valence-aware recording (default delegates to existing)
    default void recordActivation(String agentId, String tenancyId,
                                   String functionTerm, SignalValence valence) {
        recordActivation(agentId, tenancyId, functionTerm);
    }

    // New — valence-split counts for probe-time dampening
    default Map<String, ValenceCounts> valenceCounts(String agentId, String tenancyId) {
        var counts = activationCounts(agentId, tenancyId);
        var result = new java.util.LinkedHashMap<String, ValenceCounts>();
        counts.forEach((k, v) -> result.put(k, new ValenceCounts(v, 0)));
        return result;
    }
}
```

**Fallback behavior:** The default `valenceCounts()` wraps all existing counts as `ValenceCounts(count, 0)` — treating all activations as positive. This is semantically correct for the current system where no valence distinction exists. The dampening factor has no effect until store implementations override `recordActivation(4-arg)` and `valenceCounts()` to track valence separately. Implementations should be updated as part of the eidos prerequisite PR.

### DispositionProfileStore (eidos-api — new)

```java
@FunctionalInterface
public interface DispositionProfileStore {
    void update(String agentId, String tenancyId, List<DispositionValue> newProfile);
}
```

Abstracts profile persistence after evolution. Implementations decide the storage mechanism:
- In-memory: update a cached `AgentDescriptor`
- JPA: write to database
- Event-driven: publish a profile-changed event

### DefaultDispositionHealth.probe() Evolution

`probe()` evolves to use `valenceCounts()` instead of `activationCounts()`:

```java
// Before:
final double raw = dv.weight() + counts.getOrDefault(dv.term(), 0) * delta;

// After:
final var vc = valenceCounts.getOrDefault(dv.term(), new ValenceCounts(0, 0));
final double raw = dv.weight() + vc.effective(dampeningFactor) * delta;
```

The `dampeningFactor` is resolved from preferences via the key `casehub.blocks.personality.dampening-factor` (default 0.5), resolved alongside the existing `reinforcementDelta` and `overReinforcementThreshold` preferences.

**Implementation ordering:** The eidos-api and eidos-runtime changes (SignalValence, ValenceCounts, DispositionProfileStore, valence-aware probe) are a prerequisite PR to the eidos repo. The blocks PR depends on the updated eidos-api artifact.

## Complementary Safety Mechanisms

Two independent safety mechanisms protect against runaway drift:

| Mechanism | Owner | Scope | Trigger |
|---|---|---|---|
| `overReinforcementThreshold` | eidos (existing) | Per-function | Dominant effective weight exceeds ceiling (default 0.50) |
| `l2Ceiling` | blocks (new) | Global profile | L2 distance from baseline exceeds ceiling (default 0.15) |

These are distinct geometric properties. The per-function threshold catches one function becoming disproportionately strong. The L2 ceiling catches distributed drift across multiple dimensions that no single per-function ceiling would detect. Both fire independently.

## Package Structure

**`io.casehub.blocks.agentic.personality`** (new sub-package)

| Class | What it does |
|---|---|
| `TraitPressureSource<E>` | SPI: `eventType()` + `translate()` — domain event → trait activations |
| `TraitActivation` | Record: `(String functionTerm, SignalValence valence)` |
| `EvolutionTick` | Sealed: Stable, Drifting, Halted, Evolved, Dampened |
| `PersonalityEvolutionOrchestrator` | CDI bean: `record()`, `tick()` |
| `PersonalityEvolutionConfig` | Record with defaults, preference keys, and preference-based resolution |
| `BehavioralSignalPressureSource` | Default source for `BehavioralSignal` |
| `RelationshipPressureSource` | Default source for `RelationshipEvent` |
| `GoalOutcomePressureSource` | Default source for `GoalOutcomeCounts` |

**Eidos-api additions:**

| Class | What it does |
|---|---|
| `SignalValence` | Enum: POSITIVE, NEGATIVE |
| `ValenceCounts` | Record with `effective(dampeningFactor)` |
| `DispositionProfileStore` | `@FunctionalInterface` SPI for persisting evolved profiles |
| `DispositionSignalStore` | Two new default methods |

## Testing Strategy

All tests are plain JUnit 5 + Mockito (no Quarkus runtime).

| Test class | Coverage |
|---|---|
| `PersonalityEvolutionOrchestratorTest` | Core tick cycle: Stable/Drifting/Halted/Evolved/Dampened outcomes with mock signalStore, health, evolution, profileStore, cbrStore |
| `HaltFlagTest` | Recording stops when L2 ceiling exceeded, resumes after decay brings displacement within bounds. Dampened path sets halt flag — verifies oscillation prevention. |
| `BehavioralSignalPressureSourceTest` | Signal mapping for each BehavioralSignal value across Big Five, DISC, and Jungian profiles |
| `RelationshipPressureSourceTest` | Quality signal mapping across vocabulary types — POSITIVE, NEGATIVE, and NEUTRAL (zero activations) |
| `GoalOutcomePressureSourceTest` | Success/failure rate mapping with threshold boundaries |
| `ValenceCountsTest` | `effective()` computation with various dampening factors, edge cases (zero counts, zero dampening) |
| `EvolutionTickTest` | Sealed type exhaustiveness, field access, newProfile carried through Evolved |
| `CbrTransitionRecordingTest` | PersonalityTransitionSchema case recording on Evolved, outcome recording, supersession on revert |
| `TickConcurrencyTest` | Per-agent locking prevents double-decay and race-on-Evolved |
| `EventTypeDispatchTest` | `record()` correctly matches events to sources via `eventType()` |

## References

- `DefaultDispositionHealth.java` (eidos) — probe logic, L2 computation, evolution threshold detection
- `DefaultDispositionEvolution.java` (eidos) — JPAF decision rules implementation
- `DispositionSignalStore.java` (eidos) — activation signal accumulation SPI
- `VocabularyTerm.opposite()` (eidos-api) — vocabulary-generic structural navigation
- `SummarisationRunner.tick()` (blocks) — periodic caller-driven processing pattern
- `CbrCaseMemoryStore` (neocortex-memory-api) — CBR case storage and retrieval
- `PersonalityTransitionSchema` (neocortex-memory-api) — pre-defined transition case schema
- `TrendAnalyzer` (neocortex-memory-api) — trend detection for time-series CBR data
- `GE-20260811-e941cc` — AgentDisposition vs DispositionProfile type distinction
- `GE-20260728-a53632` — vocabulary-generic structural navigation technique
- `GE-20260729-172d18` — DefaultDispositionEvolution implements 4 JPAF decision rules
- JPAF paper (arXiv:2601.10025) — Jungian Personality Adaptation Framework: differentiation levels, decision rules, reinforcement-compensation, reflection mechanism
- LLMPTBench (NeurIPS 2025, OpenReview kVXePuKReA) — agentic frameworks exhibit exaggerated personality shifts under negative events
- BFI-Adapt (arXiv:2608.06485) — event-induced personality change benchmark, LLM agents shift indiscriminately and under-shift vs humans
- Takata et al. (2024, arXiv:2411.03252) — spontaneous emergence of agent individuality through social interactions
- Zeng et al. (ACL 2025, aclanthology.org/2025.findings-acl.1185) — dynamic personality in LLM agents, evolutionary modelling
- Li et al. (2024) — Cognition-Emotion-Growth architecture, feedback-loop personality development
