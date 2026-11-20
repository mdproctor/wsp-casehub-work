# Design: SocialComparison Integration (#317)

**Epic:** #311 (Neocortex cognitive integration)
**Scale:** M | **Complexity:** High
**Date:** 2026-09-29

## Problem

Neocortex provides `SocialComparison`, a pure static utility that computes multi-agent perspectival divergence metrics: PAD distance matrix, per-dimension pairwise differences, and trajectory alignment via cosine similarity. `CognitiveProfile.compare()` batches multi-agent perspective resolution. After #319, blocks has the raw `compare()` results cached in `CognitiveProfileParticipant`, but never runs `SocialComparison.compare()` to derive the richer divergence analysis. The full social comparison is only available in wacky-manor's app-level `CharacterCognition.renderSocialAwareness()`.

## Goal

Wire `SocialComparison.compare()` into blocks so agents can surface where they disagree about shared entities, detect emotional divergence, and identify misaligned trajectories — all rendered as structured natural language in prompts. Only relevant when `perspectiveComparisonEnabled` is true.

## Architecture

Two changes to existing code, one new class, one cleanup:

### 1. Extend CognitiveProfileParticipant (D1)

After calling `CognitiveProfile.compare()` for each entity, run `SocialComparison.compare()` on the result map. Store derived data alongside existing caches.

New field:
- `Map<String, PerspectivalComparison> lastSocialComparisons` — keyed by entity node ID

In the `tick()` method, after the existing compare block:

```java
if (config.perspectiveComparisonEnabled() && !cmp.isEmpty()) {
    comparisons.put(ek.node().id(), cmp);
    lastSocialComparisons.put(ek.node().id(),
            SocialComparison.compare(cmp));
}
```

New accessor:
- `Map<String, PerspectivalComparison> lastSocialComparisons()`

No new `CognitionConfig` flag — gated by existing `perspectiveComparisonEnabled`.

### 2. SocialComparisonPromptSection (D2)

New `PromptSection` implementation in `blocks-core/...prompt/`. Reads `PerspectivalComparison` data from the participant and renders structured natural language per entity.

Constructor takes: `Map<String, PerspectivalComparison>`, `double distanceThreshold` (default 0.4).

**Rendering format** per entity (only when divergence exceeds threshold):

```
Social divergence — [entity name]:
  [agent-a] ↔ [agent-b]: distance [N.NN]
    Dominant difference: [pleasure|arousal|dominance] — [agent-a] [+/-N.NN] vs [agent-b]
    Trajectory: [ALIGNED|MIXED|DIVERGENT] ([trend-a] vs [trend-b])
```

**Filtering rules** (D3):
- Skip pairs with PAD distance ≤ threshold (default 0.4)
- Skip pairs with trajectory agreement = ALIGNED
- Skip entities where all pairs are filtered out
- Return null (empty section) when nothing survives filtering
- Max 5 pairs per entity to bound output

### 3. Remove Comparison Rendering from EntityKnowledgePromptSection

The "How others see [entity]" block in `EntityKnowledgePromptSection.renderComparison()` is subsumed by `SocialComparisonPromptSection`. Remove the `renderComparison()` method and the comparison-related constructor parameter. The `comparisons` map is no longer needed in this section.

This also simplifies the `SocialAvatarCognition` constructor wiring — the EntityKnowledgePromptSection customizer no longer passes comparison data.

### 4. Wire via SocialAvatarCognition

In the existing CognitiveProfile wiring block in `SocialAvatarCognition(Builder)`, extend the `chainSectionCustomizer` to also add a `SocialComparisonPromptSection` when social comparisons are available:

```java
core.chainSectionCustomizer(sections -> {
    var result = new ArrayList<>(sections);
    var knowledge = profileParticipant.lastEntityKnowledge();
    if (!knowledge.isEmpty()) {
        result.add(new EntityKnowledgePromptSection(knowledge));
    }
    var socialComparisons = profileParticipant.lastSocialComparisons();
    if (!socialComparisons.isEmpty()) {
        result.add(new SocialComparisonPromptSection(socialComparisons));
    }
    return result;
});
```

No changes to BlocksBeans or BlocksAutoConfiguration — the wiring is internal to the existing CognitiveProfile block.

## Testing

### CognitiveProfileParticipantTest additions
- Verify `lastSocialComparisons()` populated when `perspectiveComparisonEnabled`
- Verify `lastSocialComparisons()` empty when disabled
- Verify PerspectivalComparison contains expected entity ID and agent count

### SocialComparisonPromptSectionTest
- Render with divergent pair — verify distance, dominant dimension, trajectory text
- Render with aligned pair below threshold — verify filtered out (null return)
- Render with mixed trajectories — verify MIXED label
- Verify max 5 pairs per entity limit
- Verify null return when all pairs filtered

### EntityKnowledgePromptSectionTest updates
- Remove comparison-related test cases
- Verify constructor no longer takes comparisons parameter

## Files Changed

| Module | File | Change |
|--------|------|--------|
| blocks-core | `CognitiveProfileParticipant.java` | Add SocialComparison step + cache |
| blocks-core | `prompt/SocialComparisonPromptSection.java` | New — renders PerspectivalComparison |
| blocks-core | `prompt/EntityKnowledgePromptSection.java` | Remove comparison rendering |
| blocks-core | `prompt/SocialAvatarCognition.java` | Update section customizer wiring |
| blocks | `CognitiveProfileParticipantTest.java` | Add social comparison assertions |
| blocks | `prompt/SocialComparisonPromptSectionTest.java` | New — unit tests |
| blocks | `prompt/EntityKnowledgePromptSectionTest.java` | Remove comparison tests |

## References

- `SocialComparison.java` — neocortex cognitive-index, static compare() utility
- `PerspectivalComparison.java` — record: perspectives, distances, dimensionDifferences, trajectoryAlignment
- `CognitiveProfileParticipant.java` — blocks-core, TERMINAL tick participant (#319)
- `EntityKnowledgePromptSection.java` — blocks-core, current comparison rendering to be removed
- `PerceptionTranslator.java` — wacky-manor, existing NL translation (0.4 distance threshold)
- `PadDistanceMatrix` — Euclidean distance between agent PAD vectors
- `PairwiseDifferences` — signed per-dimension differences per agent pair
- `TrajectoryAlignment` — cosine similarity + TrendAgreement (ALIGNED/MIXED/DIVERGENT)
- Issue #319 — CognitiveProfile deep integration (foundation)
- Issue #311 — Neocortex cognitive integration epic
