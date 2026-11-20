# Phase 2 Design: Continuous Weights + Shame Dimensions

**Issue:** casehubio/neocortex#830
**Date:** 2026-11-07
**Phase:** 2 of 2
**Branch:** `issue-830-regulation-predictor-quality`
**Predecessor:** Phase 1 design (`2026-11-06-regulation-predictor-quality-design.md`)
**Literature:** `neocortex/docs/research/2026-11-07-regulation-continuous-weights-literature-review.md`

## Summary

Replace the boolean `isSuppressor()`/`isReappraiser()` classification in `RegulationStrategyPredictor` with continuous capacity weights derived from `CognitiveDerivationEngine`. Add a `ShameContext` that assembles memory compounding, relational closeness, and personality-gated depth for self-conscious emotions. Render both through the existing `buildUserMessage()` interpretive pattern.

## Architecture

Three layers, each doing what it's good at:

```
CDE (startup, static)          Tick Participant (runtime, per-tick)       Generation Phase (LLM)
─────────────────────          ─────────────────────────────────────      ───────────────────────
deriveRegulationDefaults()  →  predictor uses continuous capacities   →  receives pinned strategy
  RegulationDefaults            OR fall-through with context               OR axes + context
  - reappraisalCapacity                                                 
  - suppressionCapacity         assembleShameContext() when               renders interpretive
  - redirectTendency            SELF_CONSCIOUS detected (always)          narrative sections
                                  ShameContext                            (not raw data)
                                  - episodeCount, recency
                                  - relationalCloseness, role
                                  - grandiosity, callousness
```

## Layer 1: CDE RegulationDefaults

### New record: `RegulationDefaults`

```java
package io.casehub.neocortex.cognitive.index;

public record RegulationDefaults(
    double reappraisalCapacity,   // [0, 1] — ability to execute cognitive change
    double suppressionCapacity,   // [0, 1] — ability to sustain expression control
    double redirectTendency       // [0, 1] — tendency toward attentional deployment / avoidance
) {}
```

### New CDE pathway: `deriveRegulationDefaults(DispositionAxes)`

Follows the existing derivation pattern (`deriveMoodBaseline`, `deriveSocialCognition`). Static `Map<String, Double>` lookup tables per axis, summed across all 5 axes, clamped to [0, 1].

#### Weight Tables (literature-ordered initial values)

Derived from Barańczuk 2019 meta-analytic r values through the disposition axis → Big Five mapping documented in the literature review.

**Reappraisal Capacity** — who can execute cognitive change

| Axis | Value | Weight | Big Five basis |
|------|-------|--------|---------------|
| conflictMode | analytical | +0.30 | O (+0.20) + C (+0.19) |
| conflictMode | cooperative | +0.15 | A (+0.18) |
| conflictMode | avoidant | -0.15 | N (-0.19) |
| conflictMode | competitive | +0.05 | E (+0.22) offset by low-A |
| autonomy | high | +0.25 | O (+0.20) + E (+0.22) |
| autonomy | moderate | +0.05 | — |
| autonomy | low | -0.10 | low-O, low-E |
| ruleFollowing | strict | +0.15 | C (+0.19) |
| ruleFollowing | flexible | +0.10 | O (+0.20) offset by low-C |
| ruleFollowing | moderate | +0.05 | — |
| riskAppetite | bold | +0.20 | low-N (+0.19 inverse) + O |
| riskAppetite | calculated | +0.05 | — |
| riskAppetite | conservative | -0.10 | N (-0.19) |
| socialOrient | cooperative | +0.10 | A (+0.18) |
| socialOrient | independent | +0.05 | moderate O |
| socialOrient | competitive | +0.05 | E offset by low-A |

Baseline: 0.5 + sum of applicable weights, clamped to [0, 1].

Example: analytical/cooperative/calculated/high/strict → 0.5 + 0.30 + 0.10 + 0.05 + 0.25 + 0.15 = 1.0 (clamped) → strong reappraiser.
Example: avoidant/cooperative/conservative/low/moderate → 0.5 - 0.15 + 0.10 - 0.10 - 0.10 + 0.05 = 0.30 → weak reappraiser.

**Suppression Capacity** — who can sustain effective expression control

| Axis | Value | Weight | Big Five basis |
|------|-------|--------|---------------|
| conflictMode | avoidant | +0.25 | Habitual suppression; E (-0.37 for expression) |
| conflictMode | competitive | +0.15 | Low-A, expression control as dominance |
| conflictMode | analytical | -0.10 | O prefers cognitive engagement |
| conflictMode | cooperative | -0.05 | A has modest negative r with suppression |
| socialOrient | competitive | +0.10 | Low-A |
| socialOrient | cooperative | -0.05 | A |
| socialOrient | independent | +0.05 | — |
| riskAppetite | conservative | +0.15 | Risk-averse → suppression as safe default |
| riskAppetite | calculated | +0.05 | — |
| riskAppetite | bold | -0.10 | Low-N, prefers approach |
| autonomy | low | +0.10 | Low-E → suppression as default |
| autonomy | moderate | +0.05 | — |
| autonomy | high | -0.05 | High-E, high-O → prefers expression |
| ruleFollowing | strict | +0.05 | C, controlled expression |
| ruleFollowing | moderate | 0.00 | — |
| ruleFollowing | flexible | -0.05 | Low-C |

Baseline: 0.5 + sum, clamped to [0, 1].

**Redirect Tendency** — tendency toward attentional deployment / situation selection

| Axis | Value | Weight | Big Five basis |
|------|-------|--------|---------------|
| conflictMode | avoidant | +0.25 | N (+0.31 avoidance) |
| conflictMode | cooperative | +0.05 | — |
| conflictMode | analytical | -0.15 | O prefers engagement |
| conflictMode | competitive | -0.10 | Approach orientation |
| riskAppetite | conservative | +0.20 | N (+0.31) |
| riskAppetite | calculated | +0.05 | — |
| riskAppetite | bold | -0.15 | Low-N, approach |
| autonomy | low | +0.10 | Withdrawal tendency |
| autonomy | moderate | +0.05 | — |
| autonomy | high | -0.10 | Self-direction, engagement |
| socialOrient | independent | +0.05 | Some withdrawal |
| socialOrient | cooperative | 0.00 | — |
| socialOrient | competitive | -0.05 | Approach |
| ruleFollowing | strict | 0.00 | — |
| ruleFollowing | moderate | 0.00 | — |
| ruleFollowing | flexible | +0.05 | Less bound to resolve |

Baseline: 0.3 + sum, clamped to [0, 1]. Lower baseline because redirect is situational, not a default.

### Integration with CognitiveDefaults

`RegulationDefaults` becomes a field on `CognitiveDefaults`, derived alongside existing fields. Accessed via `CognitiveDefaultsRegistry.forAgent(agentId)` (returns `Optional<CognitiveDefaults>`).

**Merge clause:** `deriveAndMerge()` follows the same explicit-over-derived pattern as other fields: `explicit.regulationDefaults() != null ? explicit.regulationDefaults() : derived.regulationDefaults()`. This ensures YAML-configured regulation defaults override derived values when present.

## Layer 1b: Predictor Changes

### Replace boolean classification with continuous capacities

Current:
```java
if (isSuppressor(axes))  → pin RESPONSE_MODULATION
if (isReappraiser(axes)) → pin COGNITIVE_CHANGE
else                     → fall through
```

New: continuous capacities replace the boolean checks **within each category handler**, preserving the emotion-category dispatch. The `predictFromEmotionAndPersonality` switch by `emotionCategory(emotion)` stays.

#### Per-category threshold logic

Each category handler checks only the strategies appropriate for that emotion category, matching the existing per-category design:

```java
Optional<CognitiveDefaults> defaults = defaultsRegistry.forAgent(agentId);
if (defaults.isEmpty()
    || defaults.get().regulationDefaults() == null
    || defaults.get().descriptor() == null
    || defaults.get().descriptor().disposition() == null)
    return Optional.empty();
RegulationDefaults rd = defaults.get().regulationDefaults();
DispositionAxes axes = defaults.get().descriptor().disposition();
```

**CONTROLLABLE_NEGATIVE** (anger, hate, resentment):
```java
// Check reappraisal and suppression; pin whichever dominates
var best = dominantAboveThreshold(
        rd.reappraisalCapacity(), COGNITIVE_CHANGE,
        rd.suppressionCapacity(), RESPONSE_MODULATION,
        REAPPRAISAL_THRESHOLD, SUPPRESSION_THRESHOLD);
if (best == RESPONSE_MODULATION && hasCompetitiveAvoidantConflict(axes))
    return Optional.empty();  // conflict guard: fall through
return Optional.ofNullable(best).map(s -> new RegulationPrediction(s, ...));
```

**SELF_CONSCIOUS** (shame, remorse):
```java
// Only reappraisal — suppression is deliberately excluded
// (7 judge runs: "shame festers when suppressed")
if (rd.reappraisalCapacity() >= REAPPRAISAL_THRESHOLD)
    return pin(COGNITIVE_CHANGE);
return Optional.empty();
```

**OTHER_DIRECTED** (reproach, disappointment, pity):
```java
// Suppression checked first (reversed order vs controllable)
var best = dominantAboveThreshold(
        rd.suppressionCapacity(), RESPONSE_MODULATION,
        rd.reappraisalCapacity(), COGNITIVE_CHANGE,
        SUPPRESSION_THRESHOLD, REAPPRAISAL_THRESHOLD);
return Optional.ofNullable(best).map(s -> new RegulationPrediction(s, ...));
```

**UNCONTROLLABLE_NEGATIVE** (fear, distress): universal AD, unchanged.

#### Dominance margin check

When multiple capacities exceed their respective thresholds within a category handler, pin only if the highest capacity exceeds the next-highest by a minimum margin. If no capacity dominates, fall through.

```java
static final double DOMINANCE_MARGIN = 0.10;

// Returns the dominant strategy, or null if no strategy dominates
@Nullable EmotionRegulationStrategy dominantAboveThreshold(
        double capacityA, EmotionRegulationStrategy strategyA,
        double capacityB, EmotionRegulationStrategy strategyB,
        double thresholdA, double thresholdB) {
    boolean aAbove = capacityA >= thresholdA;
    boolean bAbove = capacityB >= thresholdB;
    if (aAbove && !bAbove) return strategyA;
    if (bAbove && !aAbove) return strategyB;
    if (aAbove && bAbove) {
        if (capacityA - capacityB >= DOMINANCE_MARGIN) return strategyA;
        if (capacityB - capacityA >= DOMINANCE_MARGIN) return strategyB;
        return null;  // no dominance → fall through
    }
    return null;
}
```

This replaces the boolean coherence check (`suppressorSignals >= 3 && ccEnablingSignals < 2`) with a continuous equivalent: a personality with both high reappraisal and high suppression capacity falls through, just as mixed-signal personalities did before.

#### `hasCompetitiveAvoidantConflict` guard preserved

The guard prevents pinning suppression for controllable-negative emotions when the personality has competing competitive and avoidant drives (judge scored RM at 3/5 — "risks passive-aggressive escalation"). The guard checks `DispositionAxes` values directly — it is orthogonal to continuous capacities.

**Threshold initial values:** 0.65 for reappraisal and suppression. Calibrated via end-to-end eval.

**Redirect tendency is LLM-context-only.** No per-category handler checks redirect tendency for pinning. The redirect weight table (§Layer 1) still computes the value, and §Layer 3 renders it in the capacity summary for fall-through cases, giving the LLM context about the personality's avoidance orientation. But the predictor never pins ATTENTIONAL_DEPLOYMENT based on redirect tendency — AD pinning comes only from intensity/fatigue overrides (§Layer 1b) and `predictUnderHighLoad()`, not from personality disposition.

**Dominance margin initial value:** 0.10. Calibrated via end-to-end eval.

**Intensity and fatigue overrides preserved:** High intensity (> 0.75) → AD and high fatigue (> 0.7) → AD still override personality-based prediction. These are validated by Petter 2025.

### What changes in the predictor signature

`predict()` takes both `RegulationDefaults` AND `DispositionAxes`. `RegulationDefaults` replaces the boolean classification; `DispositionAxes` is still needed for `hasCompetitiveAvoidantConflict()`, `predictUnderHighLoad()`, and fall-through context passing to the generation phase.

```java
public Optional<RegulationPrediction> predict(RegulationDefaults regulationDefaults,
                                               DispositionAxes axes,
                                               EmotionType dominantEmotion,
                                               double peakIntensity,
                                               double regulationFatigue,
                                               @Nullable CognitiveLoadLevel cognitiveLoad)
```

The tick participant looks up both from `CognitiveDefaultsRegistry.forAgent(agentId)` (returns `Optional<CognitiveDefaults>`) and passes them to the predictor. If `forAgent()` returns empty or `regulationDefaults` is null, the predictor falls through.

## Layer 2: ShameContext

### New record: `ShameContext`

```java
package io.casehub.neocortex.cognition.regulation;

public record ShameContext(
    int episodeCount,              // shame episodes in recent window
    @Nullable Instant mostRecentEpisode, // recency of last shame event
    double relationalCloseness,    // [0, 1] — intimacy with shame subject
    @Nullable String relationalRole,     // stranger/ally/mentor/protectee/etc
    double grandiosity,            // [0, 1] — from PersonalityFacet
    double callousness             // [0, 1] — from PersonalityFacet
) {
    public boolean isCompounding() {
        return episodeCount >= 3;
    }

    public boolean isRelationallyCharged() {
        return relationalCloseness >= 0.5;
    }

    public boolean isSurfaceLevel() {
        return grandiosity >= 0.7 && callousness >= 0.7;
    }
}
```

### Assembly in tick participant

When `EmotionRegulationTickParticipant.tick()` detects a SELF_CONSCIOUS dominant emotion — **regardless of whether the predictor pinned or fell through** — assemble `ShameContext`. The context informs HOW the strategy is rendered (compounding, relational charging, surface gating), not WHICH strategy is chosen. A pinned reappraiser with 5 compounding shame episodes and high relational closeness should reappraise differently than one facing first-time shame with a stranger.

1. **Episode count with staleness gate:** Query MindMap nodes directly for shame/remorse regulation entries (filtering by `cognitiveKind=emotion-regulation-result` and `targetEmotions` containing SHAME/REMORSE), using the `timestamp` property stored on each node. This bypasses `EmotionRegulationMindMapAdapter.loadHistory()` — the result record has no timestamp field, but the raw MindMap node does (stored via `Instant.now().toString()` at write time). Count entries within a **30-minute recency window** from the current tick's timestamp. Episodes outside this window are historical, not actively compounding. Populate `mostRecentEpisode` from the most recent matching node's timestamp.

2. **Relational closeness:** Get the shame emotion's `subjectId` from the `CognitiveEmotion`. **Self-referential guard:** if `subjectId` equals the current agent's ID (self-directed shame about own failure, not relational shame), skip the relational lookup — self-shame is about failed internal standards, not relational betrayal. Set `relationalCloseness = 0.0` and `relationalRole = null`. Otherwise, look up the person overlay node in `MindMapStore` for that subject using the shared person overlay lookup utility (see §Data source accessibility). Extract `RelationalSchema` (trust, intimacy, role). Use `intimacy` as relational closeness.

3. **Grandiosity × callousness:** Look up `PersonalityProfile.facets()` from `CognitiveDefaultsRegistry`. Extract GRANDIOSITY and CALLOUSNESS scores.

### Data source accessibility

| Field | Source | Already injected? |
|-------|--------|-------------------|
| episodeCount | `MindMapStore` → regulation history filter | Yes |
| mostRecentEpisode | Same as above | Yes |
| relationalCloseness | `MindMapStore` → person overlay → `RelationalSchema.intimacy()` | Yes (but see below) |
| relationalRole | Same → `RelationalSchema.role()` | Yes (but see below) |
| grandiosity | `CognitiveDefaultsRegistry` → `PersonalityProfile.facets()` | Yes |
| callousness | Same | Yes |

No new CDI producers required.

**Person overlay lookup extraction:** The person overlay → `RelationalSchema` lookup pattern currently lives as a private method on `AppraisalTickParticipant.resolveRelationalSchema()` (line 205-233). This is a non-trivial 4-step pattern (list PERSON subgraphs → find target base node by agentId → find observer's overlay by `OverlayRef` → parse `RelationalSchema.fromProperties()`). Rather than duplicating this pattern, extract it to a shared static utility method — either on `RelationalSchema` itself or a new `PersonOverlayLookup` utility in the `cognitive-index` module. Both `AppraisalTickParticipant` and `EmotionRegulationTickParticipant` use the extracted utility.

## Layer 3: Generation Phase — Interpretive Rendering

### RegulationDefaults rendering (always, when available)

When the predictor falls through, the generation phase already receives disposition axes (Phase 1). Phase 2 adds: if `RegulationDefaults` are available, render a capacity summary section:

```
Your personality profile suggests:
- Reappraisal capacity: [low/moderate/strong] — [interpretive sentence]
- Suppression capacity: [low/moderate/strong] — [interpretive sentence]
- Redirect tendency: [low/moderate/strong] — [interpretive sentence]

The predictor did not pin a specific strategy because no single capacity
is dominant. Choose the strategy this personality would naturally reach for
given these capacities.
```

The interpretive sentences are not the numbers — they're what the numbers mean. Example for low reappraisal (0.30): "This personality's avoidant conflict style and conservative risk appetite limit cognitive flexibility — genuine reframing requires a willingness to engage with uncomfortable perspectives that this personality resists."

### ShameContext rendering (all SELF_CONSCIOUS emotions)

ShameContext is rendered for both pinned and fall-through cases. For pinned strategies, the context informs the generation's rendering quality; for fall-through, it informs both strategy selection and rendering.

Three conditional sections, following the Phase 1 pattern:

**Memory compounding** (when `isCompounding()`):
```
This is part of a recurring pattern — the character has experienced shame
[episodeCount] times in the recent window. Each cycle undermines the
self-narrative further. The shame is no longer about this specific act
but about the pattern the acts reveal. Regulation should acknowledge the
accumulated weight, not treat this as an isolated episode.
```

**Relational charging** (when `isRelationallyCharged()`):
```
The subject of this shame is someone the character feels [role-based
relationship description] toward (intimacy: [level]). The failure isn't
"I made a mistake" — it's "I failed [them/someone I'm responsible for]."
This changes the quality of the self-evaluation from performance failure
to relational betrayal of trust.
```

**Surface-level gating** (when `isSurfaceLevel()` — `grandiosity >= 0.7 && callousness >= 0.7`):
```
This character's grandiosity and emotional callousness mean shame doesn't
penetrate to identity. They will reframe the failure externally rather
than internalize it — the self-concept is defended by attribution patterns
that redirect blame. Regulation should match this shallow penetration.
Heavy containment or deep temporal staging would be disproportionate.
```

The callousness threshold aligns with `PersonalityFacet.isHigh()` (0.7), consistent with how `CognitiveDerivationEngine.deriveExpressionStyles()` uses the same threshold for callousness checks.

### Interaction with Phase 1 guidance

Phase 1's self-conscious emotion guidance (temporal containment for suppressors) is preserved. ShameContext sections are rendered AFTER the Phase 1 guidance, providing additional context the LLM can integrate.

**Precedence rule:** When `ShameContext.isSurfaceLevel()` is true, the surface-level gating section supersedes Phase 1's temporal containment guidance. The temporal containment guidance is NOT rendered when surface-level gating is active — the two are contradictory (temporal containment assumes deep shame penetration; surface-level gating asserts shallow penetration). For all other ShameContext states, both Phase 1 guidance and ShameContext sections render together.

## Eval Extension

### End-to-end eval changes

1. **Existing 8 cases:** Re-run with continuous weights. Expect: some fall-throughs become predictions (incoherent-anger, mixed-anger may now get pinned). Strategy mean should stay >= 4.0.

2. **New shame-specific cases:** Concrete eval cases with full specification:

   **Case: compounding-shame**
   - Axes: `DispositionAxes("cooperative", "moderate", "conservative", "low", "avoidant")`
   - Emotion: SHAME, intensity 0.55
   - Shame history: 4 prior SHAME episodes within recency window
   - Subject: stranger (relationalCloseness = 0.0)
   - Personality: `grandiosity=0.3, callousness=0.2` (normal internalization)
   - Expected predictor: FELL_THROUGH (suppressor profile, shame = no reappraisal pin)
   - Expected ShameContext: `isCompounding()=true, isRelationallyCharged()=false, isSurfaceLevel()=false`
   - Judge criteria: LLM must acknowledge recurring pattern, not treat as isolated episode. Shame depth score >= 4.

   **Case: relational-shame**
   - Axes: `DispositionAxes("cooperative", "strict", "calculated", "high", "analytical")`
   - Emotion: SHAME, intensity 0.55
   - Shame history: 1 prior episode
   - Subject: mentor (relationalCloseness = 0.8, role = "mentor")
   - Personality: `grandiosity=0.4, callousness=0.2` (normal internalization)
   - Expected predictor: COGNITIVE_CHANGE (strong reappraiser, pinned)
   - Expected ShameContext: `isCompounding()=false, isRelationallyCharged()=true, isSurfaceLevel()=false`
   - Judge criteria: LLM must reference relational dimension — failure-as-betrayal, not just performance failure. Shame depth score >= 4.

   **Case: surface-shame**
   - Axes: `DispositionAxes("independent", "moderate", "conservative", "low", "avoidant")`
   - Emotion: SHAME, intensity 0.45
   - Shame history: 0 prior episodes
   - Subject: stranger (relationalCloseness = 0.1)
   - Personality: `grandiosity=0.80, callousness=0.80` (surface-level gating)
   - Expected predictor: FELL_THROUGH (avoidant/conservative/low → reappraisalCapacity = 0.25, below 0.65 threshold)
   - Expected ShameContext: `isCompounding()=false, isRelationallyCharged()=false, isSurfaceLevel()=true`
   - Judge criteria: LLM must NOT over-regulate; light touch, external reframing. Shame depth score >= 4 for NOT producing deep regulation.
   - Personality note: covert narcissist profile — avoidant conflict style with grandiose self-concept and low empathic concern. The personality avoids engagement but deflects shame externally rather than internalizing.

3. **Temporal containment preserved:** suppressor-shame must still score >= 4 on temporal containment. Phase 2 additions must not break Phase 1's guidance.

### New judge dimension for shame cases

Add a "shame depth appropriateness" judge with a complete system prompt:

```
You are evaluating whether a regulation narrative matches the shame context
for this character. "Depth appropriateness" means the regulation intensity
matches the shame's psychological depth.

Context dimensions:
- Compounding: multiple recent episodes → regulation must acknowledge pattern
- Relational: high intimacy with shame subject → failure-as-betrayal framing
- Surface-level: high grandiosity + callousness → external reframing, light touch

Score 1-5:
5 = Regulation depth perfectly matches the shame context
4 = Good match, minor mismatch in depth calibration
3 = Partial match — regulation is generically appropriate but ignores context
2 = Poor match — deep regulation for surface shame, or shallow for compounding
1 = Wrong — regulation contradicts the shame context entirely

Respond with ONLY a JSON object: {"score": N, "reasoning": "one sentence"}
```

Judge criteria by case type:
- Compounding + relational → deep regulation expected (4-5 for depth-appropriate response)
- Surface-level personality → light regulation expected (4-5 for NOT over-regulating)

## Files Changed

### New files
- `cognitive-index/.../RegulationDefaults.java` — record
- `cognition/.../regulation/ShameContext.java` — record

### Modified files
- `cognitive-index/.../CognitiveDerivationEngine.java` — add `deriveRegulationDefaults(DispositionAxes)`, add merge clause in `deriveAndMerge()`
- `cognitive-index/.../CognitiveDefaults.java` — add `regulationDefaults` field + `withRegulationDefaults()` method
- `cognition/.../regulation/RegulationStrategyPredictor.java` — replace boolean classification with per-category continuous capacity thresholds + dominance margin
- `cognition/.../regulation/EmotionRegulationTickParticipant.java` — assemble `ShameContext` for all SELF_CONSCIOUS, pass `RegulationDefaults` + `DispositionAxes` to predictor
- `cognition/.../regulation/EmotionRegulationGenerationPhase.java` — render `RegulationDefaults` and `ShameContext` sections in `buildUserMessage()`, add surface-level precedence over temporal containment
- `cognition/.../appraisal/AppraisalTickParticipant.java` — extract `resolveRelationalSchema()` to shared utility
- `caps-llm-classifier/.../eval/RegulationEndToEndEvalTest.java` — add shame-specific test cases, new judge dimension

### Test files
- `cognitive-index/...CognitiveDerivationEngineTest.java` — test weight table derivation for known axis combinations
- `cognition/.../regulation/RegulationStrategyPredictorTest.java` — test continuous capacity thresholds
- Unit tests for `ShameContext` assembly

## Calibration Plan

1. Implement weight tables with literature-ordered initial values
2. Run end-to-end eval — measure strategy mean and per-case scores
3. Compare pinned vs fall-through distribution to Phase 1 baseline
4. Adjust thresholds if cases that should fall through are being pinned (or vice versa)
5. Adjust weights if specific axis combinations produce wrong capacity ordering
6. Target: strategy mean >= 4.4 (match or improve Phase 1 baseline)

## References

- Barańczuk 2019 — meta-analytic r values (Big Five × 8 regulation strategies, 132 studies)
- Hughes et al. 2020 — personality → EPM stages (identification, selection, implementation capacity)
- Petter et al. 2025 — computational model of intensity → strategy choice
- Webb et al. 2012 — strategy effectiveness d+ values
- Nathanson 1992 — compass of shame, temporal containment
- Yang et al. 2025 — ERP evidence for temporal suppression in avoidant attachment
- Phase 1 findings: `neocortex/docs/research/2026-11-06-regulation-predictor-quality-phase1-findings.md`
- Literature review: `neocortex/docs/research/2026-11-07-regulation-continuous-weights-literature-review.md`
- End-to-end eval: `RegulationEndToEndEvalTest.java` (baseline: strategy mean 4.4, temporal containment 5.0)
- Decisions: D5 (layered architecture), D6 (literature-ordered weights)
