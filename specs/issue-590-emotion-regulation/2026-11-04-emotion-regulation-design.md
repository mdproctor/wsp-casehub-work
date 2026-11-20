# Emotion Regulation: Post-Appraisal Strategy Selection

**Issue:** casehubio/neocortex#590
**Branch:** issue-590-emotion-regulation
**Date:** 2026-11-04

## Overview

Characters currently feel emotions but cannot regulate them. This design adds a post-appraisal emotion regulation system based on Gross's (1998, 2015) process model, implementing all five regulation strategy families as a unified decision layer with distributed execution.

Regulation is the "I wanted to but held back" moment — the gap between feeling an emotion and expressing it. Without it, characters are purely reactive. With it, characters can suppress gloating, reappraise a threat as a challenge, distract from anxiety, avoid triggering situations, or modify their environment to reduce emotional load.

## Literature Foundation

| Model | Contribution | How used |
|-------|-------------|----------|
| Gross 1998 | Five strategy families: situation selection, situation modification, attentional deployment, cognitive change, response modulation | Strategy taxonomy — the enum of what regulation can do |
| Gross 2015 (EPM) | Identify → select → implement → monitor cycle | Control loop structure — the LLM prompt models this cycle |
| Bosse, Pontier & Treur 2010 | Quantitative dynamics: homeostatic baseline, persistence factor (β), adaptation rate (ζ) | Threshold calibration and adaptation — informs prompt context for cumulative regulation effects |
| Petter et al. 2025 | Strategy choice model: high intensity → distraction, low → reappraisal; cognitive load gates reappraisal | Strategy selection guidance — provided to LLM as structured decision factors |

## Architecture

### Pipeline Placement

A new `EmotionRegulationTickParticipant` runs in the **DERIVED** phase of `CognitionCore.tick()`, after `AppraisalTickParticipant`. It reads `AppraisalResult` (emotions, action tendencies) via a direct in-memory reference and writes `EmotionRegulationResult` for downstream consumption.

```
CognitionCore.tick() — DERIVED phase:
  drives.tick()                              ← direct call (NOT a participant), computes DriveAxis intensities
  runCustomParticipants(DERIVED):
    1. AppraisalTickParticipant               ← Scherer SEC → CognitiveEmotion + ActionTendency
    2. EmotionRegulationTickParticipant [NEW]  ← reads appraisal result, writes EmotionRegulationResult
    3. GutFeelingParticipant                  ← existing
```

Note: `drives.tick()` is a direct call in `CognitionCore.tick()`, executed _before_ `runCustomParticipants(CognitionPhase.DERIVED, context)`. It is not a registered participant. CapsSettler runs inside `CapsEngine` implementations — it is not part of the cognition tick pipeline. CAPS settling happens when `CapsEngine.settle()` is called from the code that drives CAPS network evaluation, which is separate from the per-tick DERIVED phase.

**Ordering constraint:** The participant list for DERIVED is insertion-ordered. `CognitionCore.configureRegulation()` must be called after `configureAppraisal()` to ensure insertion order. A unit test should verify that `EmotionRegulationTickParticipant` appears after `AppraisalTickParticipant` in the participant list for `CognitionPhase.DERIVED`.

### Relationship to Issue #590's CAPS Architecture

Issue #590 states: "Regulation strategies become CAUs that modulate drive expression in CAPS." This spec takes a different approach — regulation as an LLM decision layer outside CAPS, modifying drive rendering rather than participating in CAPS attractor dynamics. This divergence is intentional: D2 (from the decision review) chose LLM-first over mechanical CAPS integration because keyword-based regulation (the kind CAPS nodes would produce) has proven brittle in prior work.

The LLM decision layer is Phase 1. CAPS integration (regulation strategies as CAUs that feed into settling) is a natural Phase 2 extension — once the LLM-derived regulation patterns are stable and validated, the most effective patterns can be encoded as CAPS nodes. A follow-up issue should be filed for this CAPS integration path, and the epic remains open until that work is assessed.

### Activation Gate

Regulation does not fire every tick. A mechanical discrepancy threshold gates when the LLM generation call is made:

- After appraisal produces `CognitiveEmotion` instances, compute the discrepancy: `max(|emotion.intensity - baseline|)` across ALL emotions in `AppraisalResult.emotions()`
- If the maximum discrepancy exceeds a personality-derived threshold, regulation fires
- All above-threshold emotions are provided to the LLM as context — the LLM produces a single `EmotionRegulationResult` covering the holistic emotional state, not one result per emotion
- Threshold is lower for high-neuroticism characters (they regulate more often) and higher for emotionally stable characters
- If appraisal is disabled (`config.appraisalEnabled() == false`), regulation does not fire — there is no emotion input to regulate

**Bootstrap values:** Starting estimates for empirical tuning. Baseline intensity = 0.3, threshold = 0.4 (regulation fires when any emotion intensity > 0.7). These are calibration parameters — the actual distribution of emotion intensities in the appraisal system will determine appropriate values. Bosse et al. (2010) provides the framework (homeostatic baseline + discrepancy detection), not the exact values — their parameterization (β, ζ) maps to the adaptation behavior in the LLM prompt (strategy fatigue, effectiveness decay), not directly to the activation threshold.

### Regulation as a Decision Layer

The core insight: regulation is a **decision system**, not an execution system. The `EmotionRegulationTickParticipant` selects a strategy from all five Gross families. The **implementation** is distributed to existing pipeline components based on strategy type:

| Strategy family | Target | Implementation consumer |
|----------------|--------|------------------------|
| Situation selection | Environment | Goal pipeline — avoidance constraints |
| Situation modification | Environment | Goal pipeline — situation-change goals |
| Attentional deployment | Attention/salience | `renderInternalState()` — attention redirection in prompt |
| Cognitive change | Appraisal meaning | `renderInternalState()` — reframed drive descriptions |
| Response modulation | Expression | `renderInternalState()` — dampened drive rendering |

**Access path to AppraisalResult:** `EmotionRegulationTickParticipant` obtains the `AppraisalTickParticipant` reference via `CognitionCore.configureRegulation()`, following the same pattern as `configureGutFeeling()`. It calls `appraisalParticipant.currentResult(agentId, tenantId)` to read the in-memory `AppraisalResult` from the same tick — no MindMap indirection for same-tick data.

```java
public void configureRegulation(AppraisalTickParticipant appraisalParticipant,
                                 SessionPipeline sessionPipeline,
                                 MindMapStore mindMapStore) {
    this.regulationParticipant = new EmotionRegulationTickParticipant(
            appraisalParticipant, sessionPipeline, mindMapStore, config, defaultsRegistry);
    addParticipant(CognitionPhase.DERIVED, this.regulationParticipant);
}
```

**Internal strategies (3-5):** Consumed by `CognitionObservationBuilder.renderInternalState()`. The integration point is a new regulation post-processing step inside `renderInternalState()`, executed AFTER `resolveRelationalTargets()` produces `TargetedDrive` objects and BEFORE the final `targetedDescriptionFor` lambda formats them. `resolveAdaptedDrives()` and `appraisalFilterDrives()` are NOT the integration points — the former constructs `DriveSnapshot` from MindMap intensity nodes (no suppression concept), the latter includes/excludes drives by CAPS situation-context overlap (not text modification).

**Access path from builder to regulation result:** `CognitionObservationBuilder` holds a `CognitionCore` reference (field `cognitionCore`, line 53). `CognitionCore` needs a new public accessor `regulationParticipant()` returning `@Nullable EmotionRegulationTickParticipant` — following the existing pattern of `mood()`, `drives()`, `goals()`, etc. The builder reads the regulation result via `cognitionCore.regulationParticipant().currentResult(agentId, tenantId)`.

```java
// CognitionCore — new accessor (alongside existing mood(), drives(), goals() etc.)
public @Nullable EmotionRegulationTickParticipant regulationParticipant() {
    return regulationParticipant;
}
```

The regulation integration in `renderInternalState()`:
1. After `resolveRelationalTargets()` produces the `TargetedDrive` list
2. Read `EmotionRegulationResult` from `cognitionCore.regulationParticipant().currentResult(agentId, tenantId)` (in-memory, same tick) — null-safe: if regulation is unconfigured or didn't fire this tick, skip
3. For drives listed in `EmotionRegulationResult.targetDrives()`, modify the description lambda to include regulation narrative
4. The regulation narrative text OVERLAYS the consolidated `DriveDescriptionGenerated` text from `readGeneratedDriveDescriptions()` — it appends regulation-colored text, not replaces the base description
5. Add a "Regulation" `ObservationSection` as an additional part in the `parts` list

**Environmental strategies (1-2):** Consumed by the goal pipeline, but NOT through the `DriveGoalFormationStrategy` decorator pattern. `GoalProposalOrchestrator.evaluateMappers()` only invokes `DriveGoalFormationStrategy` for drive axes whose `intensity >= config.proposalThreshold()`. Environmental regulation directives (e.g., "avoid this person") are emotion-triggered, not drive-triggered — if no drive exceeds the proposal threshold, a decorator would never fire and the directive would be silently dropped.

Instead, a new `EmotionRegulationGoalParticipant` runs in the **TERMINAL** phase, registered before `DriveGoalBridgeParticipant`:

```java
class EmotionRegulationGoalParticipant implements CognitionTickParticipant {
    private final EmotionRegulationTickParticipant regulationParticipant;
    private final GoalProposalOrchestrator goals;

    @Override
    public void tick(CognitionTickContext context) {
        var regulation = regulationParticipant.currentResult(
                context.agentId(), context.tenantId());
        if (regulation.isEmpty() || !regulation.get().isEnvironmentalStrategy()) return;

        var existing = goals.registeredGoals(context.agentId(), context.tenantId());
        var combined = new ArrayList<>(existing);
        combined.add(toGoalProposal(regulation.get()));
        goals.registerGoals(context.agentId(), context.tenantId(), combined);
    }
}
```

This participant:
1. Reads the regulation result from the DERIVED-phase participant's in-memory accessor
2. If the result is an environmental strategy (SITUATION_SELECTION or SITUATION_MODIFICATION), constructs a `DriveGoalProposal` and registers it via `GoalProposalOrchestrator.registerGoals()`
3. `DriveGoalBridgeParticipant` (which runs after in TERMINAL phase) picks up registered goals via `orchestrator.registeredGoals()` and bridges them to MindMap
4. This completely bypasses the drive-threshold mechanism — environmental regulation goals fire based on emotion state, not drive intensity

Registration order in `CognitionCore`:
```java
public void configureRegulationGoals(EmotionRegulationTickParticipant regulationParticipant) {
    if (goals == null) return;
    addParticipant(CognitionPhase.TERMINAL,
                   new EmotionRegulationGoalParticipant(regulationParticipant, goals));
    // Must be called BEFORE configureDriveGoalBridge() to ensure insertion order
}
```

### Regulation Target: Emotions Trigger, Drives Are Modulated

Regulation operates across two layers:

- **Trigger:** `CognitiveEmotion` from `AppraisalResult` — the discrepancy detection reads emotion intensity to decide whether regulation is needed
- **Modulation surface:** Character-level drives from MindMap (scheming, gloating, gallantry, dominance, etc.) — regulation modifies how these drives are rendered in the LLM prompt

This aligns with Gross: regulation operates on the emotional **response** (how the feeling manifests behaviorally), not on the appraisal itself. In this architecture, character drives ARE the behavioral response surface — they're how emotions manifest as motivational pulls.

### LLM Generation

The regulation LLM call follows the `CognitionCore.appraiseMood()` pattern:

1. Open a `SessionPipeline` session with a regulation system prompt
2. Provide structured context: current emotions (from appraisal), personality traits, strategy selection factors (intensity, cognitive load), recent regulation history
3. Stream and parse JSON response into `EmotionRegulationResult`
4. Fallback: if the LLM call fails or times out, decay toward no-regulation (same as mood fallback pattern `MOOD_FALLBACK_DECAY`)

**System prompt structure:**

```
You are modeling the emotion regulation process for a character.

REGULATION STRATEGIES (Gross 1998):
[five strategy descriptions with when-to-use guidance]

STRATEGY SELECTION FACTORS (Petter 2025):
- High intensity emotions → attentional deployment (distraction)
- Low-moderate intensity → cognitive change (reappraisal)

CHARACTER PERSONALITY:
[Big Five scores with regulation tendency mappings]
- High neuroticism → more suppression, more frequent regulation
- High openness → more reappraisal, more cognitive change
- High agreeableness → more situation modification in social contexts
- High conscientiousness → more response modulation (self-control)

CURRENT STATE:
[emotions from appraisal, drive intensities, recent regulation history]

REGULATION HISTORY (last 3-5 ticks):
[what strategies were used, cumulative effects]
Note: repeated suppression increases physiological strain (Gross empirical finding).
Adaptation: strategies used repeatedly become less effective (Bosse 2010 ζ parameter).

Respond with JSON: {strategy, targetEmotions[], targetDrives[], peakOriginalIntensity, 
regulatedIntensity, effectiveness, narrative,
regulationAxis (required for SITUATION_SELECTION/SITUATION_MODIFICATION, null otherwise),
environmentalDirective (required for SITUATION_SELECTION/SITUATION_MODIFICATION, null otherwise)}
```

### Data Model

```java
record EmotionRegulationResult(
    EmotionRegulationStrategy strategy,  // Gross 1998 five-family enum
    List<String> targetEmotions,         // CognitiveEmotion labels that triggered regulation
    List<String> targetDrives,           // character drives being modulated
    double peakOriginalIntensity,        // highest emotion intensity before regulation
    double regulatedIntensity,           // regulated intensity target
    double effectiveness,                // 0.0-1.0 how well the strategy worked
    String narrative,                    // character-experience text for prompt rendering

    // Environmental strategy fields — non-null when isEnvironmentalStrategy() is true
    @Nullable DriveAxis regulationAxis,          // which DriveAxis the goal maps to
    @Nullable String environmentalDirective      // actionable goal text (e.g. "Avoid being alone with Muttley")
) {
    public boolean isEnvironmentalStrategy() {
        return strategy == EmotionRegulationStrategy.SITUATION_SELECTION
            || strategy == EmotionRegulationStrategy.SITUATION_MODIFICATION;
    }
}

enum EmotionRegulationStrategy {
    SITUATION_SELECTION,       // Gross strategy 1 — avoid triggering contexts
    SITUATION_MODIFICATION,    // Gross strategy 2 — change the situation
    ATTENTIONAL_DEPLOYMENT,    // Gross strategy 3 — redirect focus (distraction)
    COGNITIVE_CHANGE,          // Gross strategy 4 — reappraise meaning
    RESPONSE_MODULATION        // Gross strategy 5 — suppress/amplify expression
}
```

**Environmental strategy fields:** When the LLM selects SITUATION_SELECTION or SITUATION_MODIFICATION, the JSON response includes two additional fields:

- `regulationAxis` — the `DriveAxis` this environmental goal relates to. The LLM infers this from emotional context:
  - Social avoidance/approach → `AFFILIATION`
  - Avoiding/modifying competitive or performance contexts → `COMPETENCE`
  - Avoiding/modifying threatening or constraining environments → `AUTONOMY`
  - Avoiding/modifying stimulation or novelty → `CURIOSITY`
- `environmentalDirective` — actionable goal text for `DriveGoalProposal.goalDescription()` (e.g., "Avoid being alone with Muttley", "Suggest moving the conversation to a different topic")

`EmotionRegulationGoalParticipant.toGoalProposal()` reads these fields to construct a valid `DriveGoalProposal`:

```java
private DriveGoalProposal toGoalProposal(EmotionRegulationResult regulation) {
    return new DriveGoalProposal(
            regulation.regulationAxis(),
            "regulation-" + regulation.strategy().name().toLowerCase(),
            regulation.environmentalDirective(),
            "Emotion regulation: " + regulation.strategy(),
            Math.min(1.0, regulation.peakOriginalIntensity()),
            GoalPriority.MEDIUM,
            Map.of("source", "emotion-regulation",
                   "targetEmotions", String.join(",", regulation.targetEmotions())));
}
```

`peakOriginalIntensity` is used as `driveIntensity` for proposal prioritization — the emotional intensity that triggered regulation is the appropriate urgency signal for the goal system.

The `EmotionRegulation` prefix distinguishes intrapersonal regulation (this system — a character regulating their OWN emotions) from the existing `RegulationTracker` classes in neocortex (`io.casehub.neocortex.cognition.prompt.observation.RegulationTracker`) and wacky-manor (`io.casehub.examples.manor.agent.RegulationTracker`), which track interpersonal emotion trajectories (OTHER characters' distress levels via Theory of Mind).

### Rendering in Internal State

Two channels reinforce the regulation signal in the LLM prompt. Both execute inside `CognitionObservationBuilder.renderInternalState()`:

**1. Modified drive rendering** — after `resolveRelationalTargets()` produces the `TargetedDrive` list, a regulation post-processing step modifies drives that appear in `EmotionRegulationResult.targetDrives()`:

```
# Without regulation:
"gloating" (intensity: 0.8) — "A triumphant heat rises in your chest..."
[description from readGeneratedDriveDescriptions() MindMap node]

# With regulation (response modulation/suppression):
"gloating" (intensity: 0.8, regulated to 0.4) — "A triumphant heat rises in your chest..."
[original consolidated description preserved]
REGULATION: "The triumph is there but you hold it back — your jaw tightens with 
the effort of keeping your expression neutral."
[regulation narrative appended — not replacing the consolidated description]
```

The regulation narrative text is generated per-tick by the LLM. The consolidated drive description from `DriveDescriptionGenerationPhase` (which runs during MindMap consolidation, not per-tick) is preserved unchanged. Regulation appends to it — this is the correct temporal layering: consolidated descriptions are the character's baseline relationship with a drive; regulation is the per-tick modulation of that relationship.

The regulation narrative should integrate with Barrett's 4-channel framework (somatic, attentional, temporal, action tendency):

- **Suppression:** somatic channel shows physiological cost (increased tension, sympathetic activation)
- **Reappraisal:** temporal channel shifts (reframing changes how the moment feels)
- **Distraction:** attentional channel redirects (focus moves away from the trigger)

**2. Regulation ObservationSection** — added to the `parts` list in `renderInternalState()`, alongside self-standards, primed memories, and attractor lines:

```
## Regulation
You are actively regulating your emotional response.
Strategy: cognitive change (reappraisal)
Target: the anger you feel toward Muttley's betrayal
You are reframing this as a competitive challenge rather than a personal attack.
Effectiveness: moderate — the reframe holds but the sting persists.
```

### State Access: Same-Tick vs Cross-Tick

Two distinct access patterns, following existing conventions:

**Same-tick (in-memory accessor):** `EmotionRegulationTickParticipant` holds the current tick's result in a `ConcurrentHashMap<String, EmotionRegulationResult>`, exposed via `currentResult(String agentId, String tenantId)`. This is the same pattern as `AppraisalTickParticipant.currentResult()`. Consumers in the same tick (e.g., `renderInternalState()`, `EmotionRegulationGoalParticipant`) read from this accessor — no MindMap round-trip.

**Cross-tick (MindMap history):** After computing the regulation result, the participant writes it to MindMap as a node in `SubgraphTypes.COGNITIVE` with the properties defined below. History is retrieved following the existing `AppraisalTickParticipant.loadDriveIntensityNodes()` pattern — `MindMapQuery.withType()` filters by **subgraph type** (not by node property), then node properties are filtered in-memory:

```java
mindMapStore.search(MindMapQuery.of(tenantId, limit).withType(SubgraphTypes.COGNITIVE))
            .stream()
            .filter(n -> "emotion-regulation-result".equals(n.properties().get("cognitiveKind")))
            .filter(n -> agentId.equals(n.properties().get("agent-id")))
            .sorted(Comparator.comparing(n -> Instant.parse(n.properties().get("timestamp")),
                                          Comparator.reverseOrder()))
            .limit(historyWindow)
            .toList();
```

**MindMap node property schema** for `emotion-regulation-result` nodes:

| Property key | Type | Description |
|-------------|------|-------------|
| `cognitiveKind` | String | Always `"emotion-regulation-result"` — discriminator for filtering |
| `agent-id` | String | Agent this regulation belongs to |
| `timestamp` | String (ISO-8601 Instant) | `Instant.now()` at write time — ordering contract for history retrieval |
| `strategy` | String | `EmotionRegulationStrategy` enum name (e.g. `"COGNITIVE_CHANGE"`) |
| `targetEmotions` | String | Comma-separated emotion labels (e.g. `"anger,frustration"`) |
| `targetDrives` | String | Comma-separated drive types (e.g. `"gloating,scheming"`) |
| `peakOriginalIntensity` | String | Double as string (e.g. `"0.85"`) |
| `regulatedIntensity` | String | Double as string (e.g. `"0.45"`) |
| `effectiveness` | String | Double as string (e.g. `"0.6"`) |
| `narrative` | String | Regulation narrative text for prompt rendering |
| `regulationAxis` | String (nullable) | `DriveAxis` enum name — present only for environmental strategies |
| `environmentalDirective` | String (nullable) | Actionable goal text — present only for environmental strategies |

This mirrors the `EmotionRegulationResult` record fields. All values are strings — consistent with MindMap's `Map<String, String> properties()` contract. Timestamp-based ordering is done in-memory after the query since `MindMapQuery.withUpdatedAfter()` provides filtering but not sorting guarantees.

### Temporal Context (Cross-Tick Awareness)

Each tick's regulation is independent, but the generation phase prompt includes regulation history from MindMap (last 3-5 ticks). The LLM adapts naturally:

- "You have been suppressing anger for 4 ticks" → strain accumulates
- "Reappraisal worked well last tick" → strategy persistence
- "Distraction is losing effectiveness" → strategy switching

**Pruning:** Regulation nodes older than the history window (configurable, default 5) can be pruned during consolidation to prevent unbounded accumulation.

### Interaction with Existing Systems

**Interpersonal regulation (`RegulationTracker`):** Two existing `RegulationTracker` classes track OTHER characters' emotion trajectories: `io.casehub.neocortex.cognition.prompt.observation.RegulationTracker` (neocortex — distress trajectory tracking via Theory of Mind posteriors) and `io.casehub.examples.manor.agent.RegulationTracker` (wacky-manor — extends with `classifyBehavior()`). This new system handles INTRAPERSONAL regulation (a character's own emotions). All new types use the `EmotionRegulation` prefix (`EmotionRegulationResult`, `EmotionRegulationTickParticipant`, `EmotionRegulationStrategy`, `EmotionRegulationGenerationPhase`) to prevent naming collision.

**Social regulation strategies:** Gross 2015 and Zaki & Williams 2013 describe social regulation (seeking comfort, co-regulation, social sharing). Out of scope for this issue — a separate GitHub issue must be filed (referenced from #590) to track social regulation as distinct work. The `EmotionRegulationStrategy` enum is extensible if social strategies are added later.

**Mood bridge:** Regulation output does not modify mood directly. Mood is derived from appraisal via PAD projection. However, sustained successful regulation may influence mood indirectly through the memory system (regulation experiences become memories that inform future mood appraisal).

### Personality-Strategy Connection

Both personality-trait tendencies and current-state factors are provided as structured prompt context. The LLM integrates them rather than a mechanical priority rule:

**Trait tendencies (from Big Five):**
| Trait | High → | Low → |
|-------|--------|-------|
| Neuroticism | More suppression, lower threshold | Less regulation needed |
| Openness | More reappraisal | More response modulation |
| Agreeableness | More situation modification | More avoidance |
| Conscientiousness | More response modulation | Less deliberate regulation |
| Extraversion | More social strategies (future) | More attentional deployment |

**State factors (per-tick):**
- Emotion intensity (Petter 2025): high → distraction, low → reappraisal
- Regulation history: repeated strategy → declining effectiveness (Bosse 2010)

**Deferred: Cognitive load.** Petter et al. (2025) include cognitive load as a strategy selection dimension (high load → simpler strategies like distraction/suppression, low load → complex strategies like reappraisal). No cognitive load signal currently exists in the codebase. A future enhancement could estimate it as a proxy from active drive count + active goal count + Theory of Mind tracking targets. Filed as a future enhancement — not included in the system prompt until a measurement signal exists.

## Evaluation Strategy

### A/B with Regression Gate

1. **Baseline run:** Same scenario with regulation disabled (toggle via configuration flag)
2. **Regulation run:** Same scenario with regulation enabled
3. **Regression check:** Existing drive expression eval (`classify_emotions.py`) scores must not regress beyond threshold on the regulation run
4. **Regulation scoring:** New eval dimension measures regulation quality:
   - Strategy appropriateness (does the chosen strategy match intensity/personality?)
   - Narrative coherence (does the regulation text make sense in context?)
   - Personality consistency (does a neurotic character regulate differently from a stable one?)
   - Selectivity (does regulation fire at appropriate moments, not every tick?)

### Configuration Toggle

A boolean flag enables/disables the regulation participant for A/B comparison:

```properties
manor.cognition.regulation.enabled=true   # default true
manor.cognition.regulation.threshold=0.4  # activation threshold, tunable
```

### Eval Workflow

```bash
# 1. Baseline run (regulation off)
# Start server with regulation disabled, run scenario, save transcript
# 2. Regulation run (regulation on)  
# Start server with regulation enabled, same scenario, save transcript
# 3. Compare
python3 wacky-manor/docs/eval/classify_emotions.py docs/eval/<baseline-run>
python3 wacky-manor/docs/eval/classify_emotions.py docs/eval/<regulation-run>
# 4. Check regression gate + score regulation quality
```

## Implementation Scope

All five strategies designed and implemented together — no batching. The unified decision layer handles all strategies; distributed execution is the only variation (renderInternalState for internal, goal pipeline for environmental).

### New Components (neocortex)
- `EmotionRegulationTickParticipant` — DERIVED phase participant, runs after appraisal
- `EmotionRegulationResult` / `EmotionRegulationStrategy` — data model (in `cognition-api`)
- `EmotionRegulationGenerationPhase` — LLM call for strategy selection + narrative (appraiseMood pattern)
- Regulation `ObservationSection` in `CognitionObservationBuilder`
- Regulation post-processing step in `renderInternalState()` (after `resolveRelationalTargets()`, before description formatting)
- `EmotionRegulationGoalParticipant` — TERMINAL phase participant for environmental strategies (situation selection/modification), registers goals via `GoalProposalOrchestrator.registerGoals()` independently of drive threshold
- `regulationEnabled` field added to `CognitionConfig` record (25th boolean field — requires updating `all()`, `withAllSubsystems()`, `none()`, `withDirectives()`, and adding `case "regulation"` to the `with()` switch)
- Configuration properties for enable/disable toggle and threshold

### Modified Components (neocortex)
- `CognitionCore` — add `configureRegulation()` method following `configureAppraisal()` pattern; store `regulationParticipant` as a private volatile field; add `regulationParticipant()` public accessor; register in DERIVED phase after appraisal; add `configureRegulationGoals()` for TERMINAL phase environmental strategy participant
- `CognitionObservationBuilder.renderInternalState()` — read `EmotionRegulationResult` via `cognitionCore.regulationParticipant().currentResult()`, render regulation section as an `ObservationSection`, modify drive text by appending regulation narrative to `targetedDescriptionFor` output

### Test Components (examples/wacky-manor)
- A/B eval harness (configuration toggle)
- Regulation eval dimension (classify_emotions.py extension or new script)
- Unit tests for activation gate, strategy selection, rendering integration

## References

- Gross, J.J. (1998). The emerging field of emotion regulation. *Review of General Psychology*, 2(3), 271-299.
- Gross, J.J. (2015). Emotion regulation: Current status and future prospects. *Psychological Inquiry*, 26(1), 1-26.
- Bosse, T., Pontier, M. & Treur, J. (2010). A computational model based on Gross' emotion regulation theory. *Cognitive Systems Research*, 11(3), 211-230.
- Petter, J. et al. (2025). Emotion regulation, fast or slow. *Emotion*, 25(5), 1273-1292.
- Zaki, J. & Williams, W.C. (2013). Interpersonal emotion regulation. *Emotion*, 13(5), 803-810.
- `CognitionCore.java` — tick phase architecture, appraiseMood() pattern, configureAppraisal()/configureGutFeeling() wiring patterns
- `AppraisalTickParticipant.java` — Scherer SEC appraisal, AppraisalResult, currentResult() in-memory accessor pattern
- `CognitionObservationBuilder.java` — renderInternalState() flow: resolveAdaptedDrives() → appraisalFilterDrives() → resolveRelationalTargets() → targetedDescriptionFor lambda → parts assembly
- `RegulationTracker.java` (neocortex + wacky-manor) — existing interpersonal distress trajectory tracking (NOT this system)
- `DriveOrchestrator.java` / `DriveComposer.java` — ModulationLayer, DriveAxis
- `DriveDescriptionGenerationPhase.java` — consolidation pipeline, Barrett 4-channel framework (not per-tick)
- `DriveGoalFormationStrategy.java` / `DriveGoalFormationContext.java` — goal formation SPI in cognition-api
- `GoalProposalOrchestrator.java` — DriveGoalFormationStrategy registration and invocation
- `CognitionConfig.java` — 24-field record, `with()` switch pattern for toggles
- `classify_emotions.py` — drive expression eval framework
- Protocol PP-20261017-d5853d — literature citation requirement for model-derived constructs
- Decision review: `/Users/mdproctor/reviews/casehub-slots/issue-590-decision-20261104-015751/`
