# Design Spec — Relationship Stage Thresholds (#70)

## Summary

Characters currently treat all relationships identically regardless of interaction history. This adds familiarity-based relationship stages (stranger → acquaintance → familiar → friend → confidant) that gate trust-related social behaviors and perception depth based on accumulated interaction volume.

## What Exists

**Already in blocks-core:**
- `RelationshipStageConfig` — 5-tier config with decay rate and positive/negative sentiment weights
- `StageTier` — named threshold record (`name`, `threshold`)
- `UserModelOrchestrator.computeFamiliarity(positive, negative, neutral, stageConfig, ticksSinceLastInteraction)` — volume factor formula with sentiment weighting and time decay
- `RelationshipStageConfig.resolveStage(familiarityScore)` — maps score to stage name
- `RelationshipStageConfigSpec` — nullable YAML spec record for per-character overrides
- `CognitionCompiler.compileRelationshipStage()` — compiles spec to config with defaults

**Already in wacky-manor:**
- `ManorCognitiveSeeder.seedPeople()` — seeds shared person nodes and per-observer overlay nodes on `people` subgraph
- `CharacterCognition.renderTrustSections()` — reads overlay properties to render trust summaries (pattern for familiarity rendering)
- `CharacterCognition.renderSocialAwareness()` — renders PAD-based perceptions via `PerceptionTranslator`
- `ManorContextStrategy.shouldCompareSocially()` — gates social behavior by drive intensity
- `AgentExperienceService.recallRelationships()` — queries memories per agent pair via `RelationshipQuery.forPair()`

## Architecture

### 1. Configuration (D19)

Per-character familiarity thresholds in `social-config.yaml`:

```yaml
hooded-claw:
  familiarity-thresholds:
    tiers:
      - name: stranger
        threshold: 0.0
      - name: acquaintance
        threshold: 0.3      # higher than default 0.2 — suspicious character
      - name: familiar
        threshold: 0.5
      - name: friend
        threshold: 0.75     # much harder to befriend
      - name: confidant
        threshold: 0.9
    decay-rate: 0.02         # faster decay — distrust grows with absence
    positive-weight: 0.8     # less trusting of positive signals
    negative-weight: 0.7     # more sensitive to negative signals
```

Characters without `familiarity-thresholds` use `RelationshipStageConfig.defaults()` (stranger@0.0, acquaintance@0.2, familiar@0.4, friend@0.6, confidant@0.8, decayRate=0.01, positiveWeight=1.0, negativeWeight=0.5).

`SocialConfig` gains a `RelationshipStageConfig stageConfig` field. `ManorSocialConfigLoader` parses the section using `RelationshipStageConfigSpec` and compiles via the existing pattern.

### 2. Consolidation Phase (D20, D21)

New `RelationshipStagePhase` in blocks-core, `@Priority(18)` — after DriveAdaptation@17, before Merge@20.

**Input:** `MindMapStore` (overlay nodes), `CaseMemoryStore` (interaction memories), per-agent `RelationshipStageConfig`.

**Algorithm per agent:**
1. List overlay nodes in `people` subgraph where `OverlayRef.AGENT_ID == agentId`
2. For each overlay, resolve the target agent from the shared node reference
3. Query `RelationshipQuery.forPair(agentId, targetAgentId, tenantId)` to get relationship memories
4. Classify each memory by PAD pleasure: `> 0.1` → positive, `< -0.1` → negative, else neutral
5. Compute ticks since last interaction: `Duration.between(latestMemoryTimestamp, now).toMillis() / consolidationIntervalMs`. This mirrors `UserModelOrchestrator.doTick()` which divides elapsed time by `config.expectedTickInterval()`. The consolidation interval (from `ConsolidationScheduler`) serves as the tick unit.
6. Call `UserModelOrchestrator.computeFamiliarity(positive, negative, neutral, stageConfig, ticksSinceLastInteraction)`
7. Map to stage via `stageConfig.resolveStage(familiarityScore)`
8. Persist on overlay node properties: `familiarity-score`, `familiarity-stage`, `familiarity-interaction-count`

**Config SPI:** The phase needs per-agent `RelationshipStageConfig`. It receives this via a functional interface `RelationshipStageConfigProvider` (`String agentId → RelationshipStageConfig`). Wacky-manor provides an implementation that reads from `SocialConfig.stageConfig()`.

**Diagnostic:** WARNING-level log when an overlay node has no memories (expected at cold start) vs when the `people` subgraph doesn't exist (configuration error, same diagnostic pattern as D14).

**Overlay property constants** in new `OverlayFamiliarityPropertyModel`:
```java
public static final String FAMILIARITY_SCORE = "familiarity-score";
public static final String FAMILIARITY_STAGE = "familiarity-stage";
public static final String FAMILIARITY_INTERACTION_COUNT = "familiarity-interaction-count";
```

### 3. Behavioral Gating (D22)

ManorContextStrategy gains stage-aware methods for **trust-unlocked behaviors only**:

```java
public boolean shouldDisclose(String stage) {
    return stageOrdinal(stage) >= stageOrdinal("friend");
}

public boolean shouldCooperate(String stage) {
    return stageOrdinal(stage) >= stageOrdinal("acquaintance");
}
```

Adversarial behaviors (scheming, suspicion) remain gated by drive intensity via `shouldCompareSocially()`. Stage influences *how* a character acts adversarially (crude against strangers, sophisticated against confidants) through the existing PerceptionTranslator rendering, but not *whether* they act adversarially.

`stageOrdinal()` maps stage names to numeric order using the `RelationshipStageConfig.defaults()` tier list. This is a lookup, not a computation — stages are always the 5 fixed names.

### 4. Perception Rendering (D23)

`PerceptionTranslator.translate()` gains a `String stage` parameter. Rendering depth varies by stage:

| Stage | Perception output |
|-------|------------------|
| stranger | Empty — no perception for unknowns |
| acquaintance | Dominant PAD dimension only, no trajectory |
| familiar | Dominant + secondary dimension, no trajectory |
| friend | Full dimension text + trajectory annotation |
| confidant | Full output + trajectory + explicit relationship note |

`CharacterCognition.renderSocialAwareness()` reads `familiarity-stage` from the overlay node (same node walk as `renderTrustSections()`) and passes it to `PerceptionTranslator.translate()`. Each entry is prefixed with the stage label: "Peter Perfect (friend): seems more positive than you."

### 5. Seeding (initial state)

At scenario bootstrap, all overlay nodes start without familiarity properties (no `familiarity-score`, no `familiarity-stage`). This means all relationships start as `stranger` by default. The first consolidation cycle computes initial scores from whatever interaction history exists.

No seeding of initial familiarity is needed — the issue explicitly says stages are computed from interaction volume, not authored.

## Data Flow

```
scenario bootstrap
  └─ ManorCognitiveSeeder.seedPeople()
       └─ overlay nodes created (PAD values, trust, NO familiarity)

interaction loop
  └─ recordExperience() stores memories with targetAgentId attribute

consolidation (sleep cycle)
  └─ RelationshipStagePhase (@Priority 18)
       ├─ reads overlay nodes from people subgraph
       ├─ queries CaseMemoryStore for per-pair interaction memories
       ├─ classifies by PAD pleasure → positive/negative/neutral
       ├─ calls computeFamiliarity() → score
       ├─ calls resolveStage() → stage name
       └─ persists familiarity-score, familiarity-stage on overlay

rendering (prompt construction)
  └─ renderSocialAwareness()
       ├─ reads familiarity-stage from overlay node
       ├─ gates trust behaviors via shouldDisclose(stage), shouldCooperate(stage)
       └─ passes stage to PerceptionTranslator → depth-appropriate perception text
```

## Files Changed

### blocks-core (new)
| File | Change |
|------|--------|
| `RelationshipStagePhase.java` | New ConsolidationPhase, @Priority(18) |
| `OverlayFamiliarityPropertyModel.java` | Property key constants |
| `RelationshipStageConfigProvider.java` | Functional interface: agentId → RelationshipStageConfig |

### wacky-manor (modified)
| File | Change |
|------|--------|
| `SocialConfig.java` | Add `RelationshipStageConfig stageConfig` field |
| `ManorSocialConfigLoader.java` | Parse `familiarity-thresholds` section |
| `social-config.yaml` | Add per-character threshold overrides (2-3 characters) |
| `ManorContextStrategy.java` | Add `shouldDisclose(String)`, `shouldCooperate(String)` |
| `PerceptionTranslator.java` | Add `stage` parameter, stage-gated rendering |
| `CharacterCognition.java` | Read familiarity-stage from overlay, pass to PerceptionTranslator |

### wacky-manor (tests)
| File | Change |
|------|--------|
| `RelationshipStagePhaseTest.java` | Consolidation lifecycle: seed → interact → consolidate → verify stage |
| `ManorContextStrategyTest.java` | Stage gating: shouldDisclose, shouldCooperate |
| `PerceptionTranslatorTest.java` | Stage-gated rendering tiers |
| `ManorSocialConfigLoaderTest.java` | Threshold parsing |

## Trade-offs Acknowledged

- **Not real-time:** Stage updates happen during consolidation, not per-interaction. A burst of interactions in one turn won't immediately change stage. This is intentional — relationship evolution should feel gradual.
- **Volume factor saturation:** The formula asymptotically approaches 1.0. With default thresholds, ~60 interactions reach "confidant" threshold (0.8). This is by design — the formula prevents single interactions from jumping stages.
- **Adversarial behavior gap:** Stage doesn't gate scheming or suspicion. A character can scheme against a stranger. This is architecturally correct — motivation and familiarity are orthogonal dimensions.
- **Static `computeFamiliarity`:** The consolidation phase calls a static method on `UserModelOrchestrator`. If this coupling becomes problematic, extract to a utility class. Currently acceptable — the method is pure, side-effect-free, and already tested in `FamiliarityScoreTest`.

## References

- `RelationshipStageConfig.java` (blocks-core) — existing 5-tier config
- `UserModelOrchestrator.computeFamiliarity()` (blocks-core:110-121) — existing formula
- `StageTier.java` (blocks-core) — existing threshold record
- `OverlayTrustPropertyModel.java` (blocks-core) — pattern for overlay property constants
- `ManorCognitiveSeeder.seedPeople()` (wacky-manor:27-76) — overlay node creation pattern
- `CharacterCognition.renderTrustSections()` (wacky-manor:131-187) — overlay property reading pattern
- `PerceptionTranslator.translate()` (wacky-manor:17-59) — current PAD-based perception rendering
- `FamiliarityScoreTest.java` (blocks) — existing tests for computeFamiliarity
- GE-20260820-d9129a — volume factor formula (garden entry, offline)
- Issue #70, #61, #62
