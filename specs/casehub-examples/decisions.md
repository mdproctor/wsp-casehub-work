# Decisions — blocks#317 SocialComparison Integration

## D1: Where SocialComparison.compare() runs

**Choice:** Extend CognitiveProfileParticipant
**Alternatives:**
- New SocialComparisonParticipant — cleaner separation but adds participant ordering dependency at TERMINAL phase
- Compute at render time — no tick overhead but computation during prompt assembly
**Rationale:** CognitiveProfileParticipant already calls CognitiveProfile.compare() and caches the raw Map<PrincipalId, EntityKnowledge>. Adding SocialComparison.compare() in the same tick is a natural extension — one participant, one tick, both raw and derived data cached.
**Trade-offs:** Participant grows in responsibility (entity resolution + social comparison). If SocialComparison computation becomes expensive, it can't be independently disabled without also disabling entity knowledge comparisons.
**Sources:** CognitiveProfileParticipant.java (blocks-core), SocialComparison.java (neocortex cognitive-index)
**Exploration:** quick
**Status:** captured

## D2: Prompt rendering approach

**Choice:** New SocialComparisonPromptSection
**Alternatives:**
- Extend EntityKnowledgePromptSection — fewer sections but mixes entity facts with divergence metrics
- Port PerceptionTranslator from wacky-manor — reuses existing NL but couples blocks to a specific rendering style
**Rationale:** SocialComparison data (PAD distance matrix, pairwise differences, trajectory alignment) is structurally different from entity knowledge (node properties, edges, memories). Separate section keeps concerns clear and can be toggled independently.
**Trade-offs:** Two sections may render for the same entity — EntityKnowledgePromptSection shows "what you know about X" while SocialComparisonPromptSection shows "how agents disagree about X". Overlap in the comparison block of EntityKnowledgePromptSection needs to be removed.
**Sources:** EntityKnowledgePromptSection.java (blocks-core), PerspectivalComparison.java (neocortex cognitive-index)
**Exploration:** quick
**Depends on:** D1 (participant caches PerspectivalComparison)
**Status:** captured

## D3: Filtering strategy for pairwise data

**Choice:** Significance threshold
**Alternatives:**
- Top-N pairs — predictable output size but might miss interesting low-distance pairs with divergent trajectories
- Self-centric only — reduces N² to N but loses third-party social intelligence
**Rationale:** PAD distance threshold (default 0.4) filters out emotionally aligned pairs. Trajectory alignment filter (skip ALIGNED, show DIVERGENT/MIXED) surfaces disagreement patterns. Output scales with actual divergence, not agent count.
**Trade-offs:** Threshold tuning — too low floods the prompt, too high misses subtle divergence. Default 0.4 matches PerceptionTranslator's existing threshold in wacky-manor.
**Sources:** PerceptionTranslator.java (wacky-manor), PadDistanceMatrix (neocortex cognitive-index)
**Exploration:** quick
**Status:** captured
