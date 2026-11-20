# Regulation Predictor Quality — Phase 1 Design

## Problem

RegulationStrategyPredictor (#818) passes its quality gate (floor 4, mean 4.6) but has two categories of weakness identified through 8 runs of empirical LLM judge testing:

1. **Suppressor + shame falls through** — No single Gross strategy scores 4+ for a coherent suppressor personality (cooperative/avoidant/conservative/low-autonomy) facing shame. The predictor correctly returns `Optional.empty()`, but the LLM then gets no personality context and makes an uninformed choice.

2. **Stable 4s that never reach 5** — suppressor-anger→RM (4, judge notes CC "also viable"), reappraiser-reproach→CC (4, judge notes strict rule-following resists reframing genuine standards violations).

## Approach

Sequential experimentation: Phase 1 improves LLM guidance for predictor fall-through cases, Phase 2 (separate design cycle) explores CognitiveDerivationEngine continuous regulation weights informed by Phase 1 results.

## Phase 1: Personality-Aware LLM Guidance on Fall-Through

### Architecture

Two changes to `EmotionRegulationGenerationPhase.buildUserMessage()`, following the existing conditional section pattern (fatigue, cognitive load, failure signals):

**Change 1: Disposition axes on fall-through**

When the predictor falls through (`pinnedStrategy == null`) and disposition axes are available, inject them into the user message:

```
Personality disposition (predictor did not pin a strategy — use these to 
select a personality-congruent regulation):
  conflict-mode: avoidant
  social-orient: cooperative
  risk-appetite: conservative
  autonomy: low
  rule-following: moderate
```

The explicit "predictor did not pin a strategy" phrasing tells the LLM this is a case where mechanical prediction was insufficient, encouraging careful personality-aware reasoning rather than defaulting.

**Change 2: Self-conscious emotion guidance**

When the dominant emotion is self-conscious (SHAME or REMORSE) and no strategy is pinned, inject literature-grounded guidance:

```
Self-conscious emotion guidance: Shame and remorse are self-evaluative — 
the emotion IS about the self's failure against its own standards. Pure 
suppression leaves the self-evaluation unaddressed (Webb et al. 2012). For 
avoidant personalities, temporal containment is adaptive: contain the 
expression now, with awareness that the self-evaluation will be processed 
when cognitive resources are available (Nathanson compass of shame; Yang 
et al. 2025). This is not suppression-as-avoidance — it is staged 
regulation where containment enables later reappraisal.
```

### Implementation Scope

**Files changed:**
- `EmotionRegulationGenerationPhase.java` — add two conditional sections to `buildUserMessage()`
- `EmotionRegulationTickParticipant.java` — pass `DispositionAxes` through to the generation phase when predictor falls through
- `RegulationStrategyPredictor.java` — add a method to expose the emotion category classification (for the self-conscious check in the user message builder)

**Files unchanged:**
- `RegulationStrategyPredictor` logic — the predictor's decision boundaries stay the same; we're improving what happens when it falls through, not changing when it falls through
- `SYSTEM_PROMPT` — generic guidance stays generic; emotion-specific guidance goes in the user message

### Data Flow

```
EmotionRegulationTickParticipant.tick()
  → strategyPredictor.predict(axes, emotion, intensity, fatigue, load)
  → Optional.empty() (fall-through)
  → generationPhase.generate(..., pinnedStrategy=null, axes=axes, emotionCategory=SELF_CONSCIOUS)
    → buildUserMessage()
      → [existing sections: effort, fatigue, load, failure signal, emotions, history]
      → [NEW: disposition axes section — when pinnedStrategy==null && axes!=null]
      → [NEW: self-conscious guidance — when dominant emotion is SHAME/REMORSE && pinnedStrategy==null]
    → LLM call
```

### Method Signature Change

`buildUserMessage()` needs two new parameters:
- `@Nullable DispositionAxes axes` — personality context for fall-through cases
- `@Nullable EmotionType dominantEmotionType` — for self-conscious emotion detection

These are passed from `EmotionRegulationTickParticipant`, which already has access to both via `defaultsRegistry` and the peak emotion selection logic.

### Eval Plan

1. Run existing regulation predictor judge eval (`caps-llm-classifier`) with the changes
2. Compare: suppressor-shame should now score (previously fell through with score -1)
3. Check for regression on existing 4+ scores
4. Check whether the two stable-4 cases (suppressor-anger, reappraiser-reproach) improve
5. Record findings in `neocortex/docs/research/` with full judge result comparison

### Success Criteria

- suppressor-shame gets a score (was -1/unscored)
- No regression below floor 4 on existing scored cases
- Mean ≥ 4.6 (current baseline)
- Phase 1 findings documented in neocortex research

### What Phase 1 Will Teach Us

- Does the LLM use disposition axes effectively when given them?
- Does self-conscious emotion guidance change the LLM's strategy selection?
- Which personality×emotion combinations still need mechanical prediction (informing Phase 2)?
- Is temporal containment a viable LLM-expressed concept, or does it need mechanical support?

## Phase 2: CognitiveDerivationEngine Continuous Weights (Deferred)

Architecture and weight tables deferred until Phase 1 eval results are available. Phase 1 will reveal:
- What the LLM handles well with guidance (→ leave to LLM)
- What needs mechanical prediction (→ build into CDE continuous weights)
- Whether the boolean isSuppressor/isReappraiser classification should be replaced or augmented

## References

- Webb, T. L., Miles, E., & Sheeran, P. (2012). Dealing with feeling: A meta-analysis of the effectiveness of strategies derived from the process model of emotion regulation. *Psychological Bulletin*, 138(4), 775-808.
- Gross, J. J. (2015). The extended process model of emotion regulation: Elaborations, applications, and future directions. *Psychological Inquiry*, 26, 130-137.
- Yang et al. (2025). Electrocortical modulation of reappraisal and suppression in avoidant and secure attachment. *PubMed* 41076579.
- Nathanson, D. L. (1992). *Shame and Pride: Affect, Sex, and the Birth of the Self*. (Compass of shame model — avoidance as temporal containment.)
- Hughes, D. J., et al. (2020). Personality traits and emotion regulation: A targeted review and recommendations. *Emotion*, 20(1), 63-74.
- Petter, J., et al. (2025). Personality and emotion regulation strategy selection. (Referenced in predictor codebase.)
- Aldao, A., Nolen-Hoeksema, S., & Schweizer, S. (2010). Emotion-regulation strategies across psychopathology: A meta-analytic review. *Clinical Psychology Review*, 30, 217-237.
- RegulationStrategyPredictor.java — lines 92-108 (self-conscious category, 7-run judge findings)
- EmotionRegulationGenerationPhase.java — lines 155-232 (buildUserMessage conditional sections pattern)
- EmotionRegulationTickParticipant.java — lines 149-184 (tryPredictStrategy, axes access)
- Issue #830 — 8 judge runs, floor 4, mean 4.6, distribution [4, 5, 4, 5, 5]
- Issue #818 — original regulation predictor implementation (closed)
