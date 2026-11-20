# Needs Pyramid — Satisfaction Tracking with Decay

**Issue:** casehubio/examples#68
**Branch:** issue-63-fix-cdi-config-beans
**Date:** 2026-09-18

## Problem

Without a needs hierarchy, a character with one strong reinforced drive will fixate on it — Penelope helps everyone all day while her tasks pile up and her personal goals stall. Drive adaptation (#66) makes characters develop preferences from experience, but nothing prevents a single dominant drive from consuming all of a character's attention. Real people don't fixate because neglected needs create increasing psychological pressure: hunger overrides curiosity, loneliness overrides ambition, mounting obligations override exploration.

The needs pyramid tracks satisfaction across five competing tiers. Each tier decays with neglect, creating urgency. Satisfied tiers lose priority (diminishing returns). The pyramid constrains drive adaptation — a drive can't strengthen indefinitely if its associated need tier is already saturated — and exposes satisfaction levels to GoalRevisionStrategy (#69) for goal prioritization.

## Architecture

### Satisfaction Routing (D30)

Satisfaction flows through the existing drive reinforcement layer via a global drive→tier mapping:

```
event (conflict_resolution)
  → per-character drive reinforcement (scheming for Claw, social-harmony for Penelope)  [D9, exists]
    → drive-to-tier mapping (scheming → SELF_EXPRESSION, social-harmony → SOCIAL)  [new, global]
      → tier satisfaction update
```

This achieves per-character behavior through ~15 drive-type entries rather than per-character event→tier mappings. The per-character specificity is handled by the existing reinforcement config (D9).

### Data Flow

```
Bootstrap (once per scenario):
  social-config.yaml → ManorCognitiveSeeder → need satisfaction nodes
                                               (all 5 tiers at initial 0.5)

Game loop (every tick):
  agent actions → experience memories (with event-type, PAD values)

Consolidation (periodic, when idle):
  ExperienceConsolidation@15 → graduated experience nodes
  BeliefRevision@16           → belief contradiction check
  DriveAdaptation@17          → drive intensity updates
                                 + need satisfaction updates (integrated, D33)
                                 + saturation constraint on learning rate (D31)
                                 + multiplicative decay toward resting level (D32)
  RelationshipStage@18        → familiarity/stage updates
  MergeDetection@20           → ...

Rendering (every prompt):
  NeedsPyramidPromptSection → reads satisfaction nodes from MindMap
                            → renders qualitative prose under "Inner Needs"
```

### Hierarchy Model (D40)

The pyramid uses non-strict hierarchy: neglected tiers increase goal priority weighting but do not block pursuit of higher tiers. A character with critically neglected Safety can still pursue Understanding goals — Safety-related goals just receive stronger priority weighting. This produces richer behavioral diversity than strict Maslow blocking.

### Tier Taxonomy (D39)

The 5 tiers are simulation-pragmatic categories, not a direct Maslow mapping:

```
        ┌─────────────┐
        │ Understanding│  curiosity, making sense of the mansion
        ├─────────────┤
        │ Self-express │  personal goals, values, identity
        ├─────────────┤
        │   Social     │  relationships, trust, belonging
        ├─────────────┤
        │    Tasks     │  assigned duties, commitments
        ├─────────────┤
        │   Safety     │  avoid threats, self-preservation
        └─────────────┘
```

TASKS replaces Maslow's "esteem" (obligation/duty in a game context). UNDERSTANDING replaces "self-actualization" (curiosity and sense-making, not transcendence). Simulated agents don't have metabolisms or transcendent experiences.

## Components

### 1. NeedTier Enum (blocks-core)

**Location:** `blocks-core`, `io.casehub.blocks.agentic.social.need`

```java
public enum NeedTier {
    SAFETY, TASKS, SOCIAL, SELF_EXPRESSION, UNDERSTANDING;
}
```

Decay rates and resting levels are purely configuration concerns (§7 NeedSatisfactionConfig).

**Decay rates** reflect urgency hierarchy: Safety (0.15) decays fastest — immediate threats demand fast response. Understanding (0.03) decays slowest — curiosity is patient.

**Resting levels** model baseline expectations (D32 revision):
- Safety: 0.6 — "no news is good news"; absence of threats means safety feels adequate
- Tasks: 0.3 — "obligations accumulate"; even without events, task pressure builds
- Social: 0.4 — moderate baseline; social needs persist in the background
- Self-expression: 0.4 — moderate baseline
- Understanding: 0.4 — moderate baseline

### 2. NeedTierMapping SPI (blocks-core)

**Location:** `blocks-core`, same package as NeedTier.

**SPI:** An injectable configuration providing `Map<String, Set<NeedTier>>` — drive-type string → set of tiers.

```java
public interface NeedTierMappingProvider {
    Map<String, Set<NeedTier>> tierMapping();
}
```

blocks-core defines the SPI; applications provide the mapping. Follows the DriveReinforcementConfig pattern (D13).

### 3. DriveAdaptationPhase Extension (blocks-core)

**Location:** `blocks-core`, `io.casehub.blocks.agentic.social.drive.adaptation`

Need satisfaction tracking is integrated into DriveAdaptationPhase (D33) rather than a separate consolidation phase. DriveAdaptationPhase gains a new constructor dependency on `NeedTierMappingProvider` and `NeedSatisfactionConfig`.

**Extended algorithm per agent per consolidation pass:**

Steps 1–4 are unchanged from the #66 spec (load drive nodes, load experience nodes, aggregate reward per action-type, map to drives).

5. **Accumulate tier satisfaction** — after mapping events to drives (step 4), accumulate satisfaction from raw experience node PAD values (not from the already-dampened `DriveReward.effectiveReward()`, to avoid double dampening). Drives not present in the tier mapping are silently skipped. This requires DriveAdaptationPhase to track per-tier accumulation from raw event data alongside its per-drive reward accumulation.

   For each experience node, extract the raw PAD value for the drive's configured reward axis (D12), then route through the drive→tier mapping:

   ```
   // Per-tier accumulators (raw, not dampened)
   tierRawSatisfaction = {}   // tier → cumulative raw satisfaction
   tierCount = {}             // tier → event count

   for each experienceNode:
       eventType = experienceNode.property("event-type")
       for each reinforcementEntry in reinforcementConfig.get(eventType):
           tiers = tierMapping.get(reinforcementEntry.driveType)
           if tiers == null: continue  // unmapped drive, skip
           padValue = extractPadValue(experienceNode, reinforcementEntry.rewardAxis)
           if padValue > 0:
               for tier in tiers:
                   tierRawSatisfaction[tier] += satisfactionIncrement × padValue
                   tierCount[tier] += 1
           elif padValue < 0:
               for tier in tiers:
                   tierRawSatisfaction[tier] -= dissatisfactionIncrement × |padValue|
                   tierCount[tier] += 1

   // Volume dampening applied once at the tier level (GE-20260820-d9129a)
   for each tier in tierRawSatisfaction:
       effectiveDelta = tierRawSatisfaction[tier] / (1 + log(tierCount[tier]))
       currentSatisfaction[tier] += effectiveDelta
       clamp(currentSatisfaction[tier], 0.0, 1.0)
   ```

   **Bidirectional model:** Positive events (positive PAD on the drive's reward axis) increase satisfaction above resting level. Negative events (negative PAD) actively decrease satisfaction below resting level — a character in danger feels less safe, not just "not more safe." The decay-toward-resting-level model (step 8) then restores equilibrium: a character whose Safety drops to 0.2 from a threat recovers toward 0.6 as the threat fades.

6. **Apply saturation constraint** — before applying drive intensity updates (step 5 in the original spec), scale the learning rate by tier satisfaction (D31). Drives without a tier mapping use the full learning rate (unconstrained):
   ```
   for each (driveType, reward) in driveRewards:
       tiers = tierMapping.get(driveType)
       if tiers != null:
           avgSatisfaction = average(currentSatisfaction[tier] for tier in tiers)
           effectiveLR = learningRate × (1 - avgSatisfaction)
       else:
           effectiveLR = learningRate  // unmapped drive, no constraint
       delta = reward.effectiveReward() × effectiveLR
       // ... rest of multiplicative update unchanged
   ```

   At 100% average tier satisfaction, drive strengthening stops entirely. At 0%, full learning rate applies. Smooth, cliff-free, self-balancing.

   **Timing:** `currentSatisfaction` includes step 5's increments — this is the current-cycle satisfaction, not stale data from the previous cycle. The satisfaction computed in step 5 is threaded to step 6 as an in-memory running total before being persisted in step 9.

7. **Apply multiplicative drive update** — unchanged from #66 spec (using `effectiveLR` from step 6).

8. **Apply tier decay** — at the end of the phase, decay all tiers toward their resting level (D32 revised):
   ```
   for each tier in NeedTier.values():
       restingLevel = config.restingLevel(tier)
       decayRate = config.decayRate(tier)
       satisfaction = restingLevel + (satisfaction - restingLevel) × (1 - decayRate)
   ```

   Tiers above resting level decay downward; tiers below resting level recover upward. Without events, all tiers settle at their resting level.

9. **Write updated satisfaction nodes** — update `satisfaction` property on each tier's MindMap node. Write updated drive nodes and cursor (unchanged from #66).

### 4. NeedsPyramidPromptSection (blocks-core)

**Location:** `blocks-core`, `io.casehub.blocks.agentic.social.prompt`
**Implements:** `PromptSection`

Reads satisfaction nodes from MindMapStore, formats them as qualitative natural-language prose under "Inner Needs" (D37):

```
## Inner Needs

Your sense of safety feels neglected — recent events have left you uneasy.
Your task commitments feel adequate.
Your social bonds feel well-met.
Your self-expression feels fulfilled — you've been true to yourself lately.
Your curiosity feels neglected — there's much you don't understand yet.
```

**Satisfaction bands:**

| Range | Band | Description |
|-------|------|-------------|
| 0.0–0.2 | Critically neglected | Strong urgency language, contextual note |
| 0.2–0.4 | Neglected | Urgency language, contextual note |
| 0.4–0.6 | Adequate | Neutral, no contextual note |
| 0.6–0.8 | Well-met | Positive language |
| 0.8–1.0 | Fulfilled | Strong positive, contextual note |

**Tier-specific prose vocabulary:**

| Tier | Noun phrase | Example neglected | Example fulfilled |
|------|------------|-------------------|-------------------|
| SAFETY | "sense of safety" | "recent events have left you uneasy" | "you feel secure in your surroundings" |
| TASKS | "task commitments" | "obligations are piling up" | "you're on top of your responsibilities" |
| SOCIAL | "social bonds" | "you've been isolated lately" | "your relationships feel strong" |
| SELF_EXPRESSION | "self-expression" | "you haven't been true to yourself" | "you've been true to yourself lately" |
| UNDERSTANDING | "curiosity" | "there's much you don't understand yet" | "the world around you makes sense" |

**Reachable-tier filtering:** The section only renders tiers reachable through the character's drives. It checks for `cognitiveKind: "drive-intensity"` nodes in the COGNITIVE subgraph — if none exist for the agent, the entire "Inner Needs" section is suppressed. For agents with drives, tiers not reachable through any of the agent's drive→tier mappings are omitted. This prevents character-incoherent rendering (e.g., a villain seeing "obligations are piling up" when they have no task-oriented drives).

**Registration:** Added to `CognitionCore.promptSections()` when needs pyramid is enabled (new `CognitionConfig` flag: `needsPyramidEnabled`). Guarded by `config.needsPyramidEnabled() && mindMapStore != null`, following the same pattern as `CharacterDrivePromptSection`.

### 5. ManorCognitiveSeeder Extension (wacky-manor)

**Location:** `ManorCognitiveSeeder` (already handles drives, beliefs, goals, relationships).

At scenario bootstrap, creates one MindMap node per need tier per agent in the COGNITIVE subgraph:

```java
for (NeedTier tier : NeedTier.values()) {
    NodeInput needNode = NodeInput.of(
            "need-" + tier.name().toLowerCase(),  // node name
            subgraphId)
        .withConfidence(MindMapConfidenceDefaults.forOrigin(
            ConfidenceOrigin.AUTHORED, Instant.now()))
        .withProvenance("need-satisfaction")
        .withProperties(Map.of(
            "cognitiveKind", "need-satisfaction",
            "agent-id", agentId,
            "tier", tier.name(),
            "satisfaction", String.valueOf(config.initialSatisfaction()),
            "resting-level", String.valueOf(config.restingLevel(tier))));
    mindMapStore.addNode(needNode, tenantId);
}
```

**Idempotent:** If a node with `cognitiveKind: "need-satisfaction"` and matching `agent-id` + `tier` already exists, skip seeding.

5 nodes per agent × 17 characters = 85 nodes total.

### 6. NeedTierMappingProvider (wacky-manor)

**Location:** `wacky-manor`, CDI `@ApplicationScoped` bean.

Provides the global drive→tier mapping for wacky-manor's 13 drive types:

```java
@ApplicationScoped
public class ManorNeedTierMappingProvider implements NeedTierMappingProvider {

    @Override
    public Map<String, Set<NeedTier>> tierMapping() {
        return Map.ofEntries(
            entry("scheming",         Set.of(SELF_EXPRESSION)),
            entry("self-preservation", Set.of(SAFETY)),
            entry("dominance",        Set.of(SELF_EXPRESSION)),
            entry("curiosity",        Set.of(UNDERSTANDING)),
            entry("social-harmony",   Set.of(SOCIAL)),
            entry("adventure",        Set.of(UNDERSTANDING, SOCIAL)),
            entry("gallantry",        Set.of(SOCIAL, TASKS)),
            entry("proving-worth",    Set.of(TASKS, SELF_EXPRESSION)),
            entry("protection",       Set.of(SAFETY, SOCIAL)),
            entry("greed",            Set.of(SELF_EXPRESSION)),
            entry("recognition",      Set.of(SELF_EXPRESSION, SOCIAL)),
            entry("loyalty",          Set.of(SOCIAL, TASKS)),
            entry("suspicion",        Set.of(SAFETY))
        );
    }
}
```

**Multi-tier mappings:**
- `adventure → UNDERSTANDING, SOCIAL` — adventure satisfies curiosity and social connection through shared experiences
- `gallantry → SOCIAL, TASKS` — chivalric duty satisfies social bonds and task obligations
- `protection → SAFETY, SOCIAL` — protecting others satisfies safety (neutralizing threats) and social bonds
- `proving-worth → TASKS, SELF_EXPRESSION` — demonstrating competence satisfies task obligations and identity
- `recognition → SELF_EXPRESSION, SOCIAL` — being acknowledged satisfies identity and belonging
- `loyalty → SOCIAL, TASKS` — devotion satisfies social bonds and duty

### 7. NeedSatisfactionConfig (blocks-core)

**Location:** `blocks-core`, same package as NeedTier.

Smallrye configuration with sensible defaults, following DriveAdaptationConfig pattern:

```java
@ConfigMapping(prefix = "casehub.needs-pyramid")
public interface NeedSatisfactionConfig {

    @WithDefault("0.05")
    double satisfactionIncrement();

    @WithDefault("0.05")
    double dissatisfactionIncrement();

    @WithDefault("0.5")
    double initialSatisfaction();

    NeedSatisfactionConfig.DecayConfig decay();

    interface DecayConfig {
        @WithDefault("0.15")
        double safety();

        @WithDefault("0.10")
        double tasks();

        @WithDefault("0.08")
        double social();

        @WithDefault("0.05")
        double selfExpression();

        @WithDefault("0.03")
        double understanding();
    }

    NeedSatisfactionConfig.RestingConfig resting();

    interface RestingConfig {
        @WithDefault("0.6")
        double safety();

        @WithDefault("0.3")
        double tasks();

        @WithDefault("0.4")
        double social();

        @WithDefault("0.4")
        double selfExpression();

        @WithDefault("0.4")
        double understanding();
    }
}
```

| Property | Default | Description |
|----------|---------|-------------|
| `satisfaction-increment` | 0.05 | Base satisfaction bump per positively-reinforced event |
| `dissatisfaction-increment` | 0.05 | Base satisfaction decrement per negatively-reinforced event |
| `initial-satisfaction` | 0.5 | Starting satisfaction for all tiers at bootstrap |
| `decay.safety` | 0.15 | Safety decay rate per cycle |
| `decay.tasks` | 0.10 | Tasks decay rate per cycle |
| `decay.social` | 0.08 | Social decay rate per cycle |
| `decay.self-expression` | 0.05 | Self-expression decay rate per cycle |
| `decay.understanding` | 0.03 | Understanding decay rate per cycle |
| `resting.safety` | 0.6 | Safety resting level (no-news-is-good-news) |
| `resting.tasks` | 0.3 | Tasks resting level (obligations accumulate) |
| `resting.social` | 0.4 | Social resting level |
| `resting.self-expression` | 0.4 | Self-expression resting level |
| `resting.understanding` | 0.4 | Understanding resting level |

## YAML Configuration

The drive→tier mapping is provided via the CDI bean (§6 above), not YAML. The full mapping for wacky-manor's 13 drive types:

| Drive Type | Tier(s) | Rationale |
|-----------|---------|-----------|
| `scheming` | SELF_EXPRESSION | Enacting schemes = expressing identity |
| `self-preservation` | SAFETY | Self-protection = safety |
| `dominance` | SELF_EXPRESSION | Power assertion = identity expression |
| `curiosity` | UNDERSTANDING | Investigating = understanding |
| `social-harmony` | SOCIAL | Maintaining relationships = social bonds |
| `adventure` | UNDERSTANDING, SOCIAL | Exploration + shared experiences |
| `gallantry` | SOCIAL, TASKS | Chivalric duty + obligation |
| `proving-worth` | TASKS, SELF_EXPRESSION | Competence demonstration + identity |
| `protection` | SAFETY, SOCIAL | Threat neutralization + caring for others |
| `greed` | SELF_EXPRESSION | Acquisition = identity expression |
| `recognition` | SELF_EXPRESSION, SOCIAL | Acknowledgment = identity + belonging |
| `loyalty` | SOCIAL, TASKS | Devotion = bonds + duty |
| `suspicion` | SAFETY | Vigilance = safety |

## Design Considerations

### Cold-Start Behavior

Characters start at 0.5 satisfaction (configurable). With the resting-level decay model (D32 revised), tiers settle toward their resting level without events:

- **Safety (resting 0.6):** Starts at 0.5, *rises* toward 0.6 — a new character in a peaceful environment feels increasingly safe, not decreasingly.
- **Tasks (resting 0.3):** Starts at 0.5, *falls* toward 0.3 — obligation pressure builds even without task events.
- **Social, Self-expression, Understanding (resting 0.4):** Start at 0.5, settle to 0.4 — gentle decay.

This avoids the false-urgency problem where a character in a peaceful scene reports "critically neglected Safety" after 50 minutes.

### Saturation Constraint Feedback Loop

The pyramid creates a self-regulating feedback loop with drive adaptation:

1. Drive reinforced → tier satisfaction increases
2. Tier satisfaction high → learning rate scaled down (`effectiveLR = LR × (1 - satisfaction)`)
3. Drive strengthening slows → other drives (and their tiers) become relatively more responsive
4. Neglected tiers decay → their drives' learning rates recover

This prevents any single drive from monopolizing adaptation and ensures characters develop balanced motivational profiles over time.

### Relationship with #69 (Goal Prioritization)

This issue **exposes** satisfaction levels; #69 **consumes** them. The interface between the two is the satisfaction nodes in the COGNITIVE subgraph — GoalRevisionStrategy reads `cognitiveKind: "need-satisfaction"` nodes and uses satisfaction levels as priority weights for goal reordering. The non-strict hierarchy model (D40) means satisfaction levels influence goal ordering through weighting, not through blocking.

### Characters Without Drives

Characters defined with only goals and no drives in social-config.yaml (e.g., Lazy Luke, Muttley) will have no drive reinforcement events and therefore no tier satisfaction updates. All their tiers will settle at resting levels. This is acceptable — these are simpler characters who don't participate in the full cognitive simulation. If a character needs the pyramid, it needs drives.

## Cross-Repo Scope

| Repo | Changes | Issue |
|------|---------|-------|
| **blocks** | `NeedTier` enum, `NeedTierMappingProvider` SPI, `NeedSatisfactionConfig`, `NeedsPyramidPromptSection`, `DriveAdaptationPhase` extension (satisfaction tracking, decay, constraint), `CognitionConfig` new flag | Existing casehubio/blocks branch extends |
| **examples/wacky-manor** | `ManorNeedTierMappingProvider` CDI bean, `ManorCognitiveSeeder` need-tier seeding | casehubio/examples#68 |
| **neocortex** | No changes — MindMapStore API is sufficient | None |

### CognitionConfig Changes (blocks)

`CognitionConfig` gains a new `needsPyramidEnabled` boolean. This requires updating: the record definition, `all()` and `none()` factories, the `with()` switch, and `AgenticYamlProcessor.discoverCognitionConfig()`. All constructors of `CognitionCore` that create `CognitionConfig` instances need the new field threaded through.

## Empirical Validation

1. **Resting-level convergence:** Run a scenario with no events for 30+ cycles. Verify all tiers converge to their resting levels (Safety→0.6, Tasks→0.3, others→0.4).

2. **Satisfaction from events:** Run a scenario where Penelope has repeated social_interaction events. Verify SOCIAL tier satisfaction rises above resting level. Verify SAFETY tier (unreinforced) settles at resting level.

3. **Saturation constraint:** In the same scenario, verify Penelope's social-harmony drive strengthening slows as SOCIAL satisfaction rises. Compare learning rate at satisfaction 0.3 vs 0.8.

4. **Cross-character divergence:** Run the same event sequence (conflict_resolution) for Hooded Claw and Penelope. Verify different tiers are satisfied (SELF_EXPRESSION for Claw via scheming, SOCIAL for Penelope via social-harmony).

5. **Volume dampening:** In a single consolidation pass with 10 social_interaction events, verify SOCIAL satisfaction increases by less than 10× the single-event increment.

6. **Rendering test:** Verify NeedsPyramidPromptSection renders the correct band labels for satisfaction values at 0.1 (critically neglected), 0.3 (neglected), 0.5 (adequate), 0.7 (well-met), 0.9 (fulfilled).

7. **Cold-start parity:** Compare agent behavior at tick 1 (initial 0.5 satisfaction, no constraint effect) with current non-pyramid behavior. The pyramid should be inert at cold start.

## References

- `DriveAdaptationPhase` — `blocks-core/.../social/drive/adaptation/DriveAdaptationPhase.java`
- `DriveReinforcementEntry` — `blocks-core/.../social/drive/adaptation/DriveReinforcementEntry.java`
- `DriveAdaptationConfig` — `blocks-core/.../social/drive/adaptation/DriveAdaptationConfig.java`
- `DriveOrchestrator` — `blocks-core/.../social/drive/DriveOrchestrator.java`
- `DrivePromptSection` — `blocks-core/.../social/prompt/DrivePromptSection.java`
- `CharacterDrivePromptSection` — `blocks-core/.../social/prompt/CharacterDrivePromptSection.java`
- `ConsolidationPhase` — `neocortex/mindmap-intelligence/.../consolidation/ConsolidationPhase.java`
- `ConsolidationScheduler` — `neocortex/mindmap-intelligence/.../consolidation/ConsolidationScheduler.java`
- `ExperienceConsolidationPhase` — `neocortex/mindmap-intelligence/.../consolidation/ExperienceConsolidationPhase.java`
- `MindMapStore` — `neocortex/mindmap-api/.../MindMapStore.java`
- `SocialConfig` — `wacky-manor/.../agent/SocialConfig.java`
- `ManorCognitiveSeeder` — `wacky-manor/.../agent/ManorCognitiveSeeder.java`
- `ActionImportanceScorer` — `wacky-manor/.../engine/ActionImportanceScorer.java`
- `ManorGoalRevisionStrategy` — `wacky-manor/.../agent/ManorGoalRevisionStrategy.java`
- `social-config.yaml` — `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`
- GE-20260714-439924 — multiplicative dampening
- GE-20260820-d9129a — volume factor
- Decisions — `specs/issue-63-fix-cdi-config-beans/decisions.md` (D30–D40)
- Issue #68 — Needs pyramid
- Issue #69 — Goal prioritization from unmet needs (downstream consumer)
- Issue #66 — Drive adaptation from reward signals (upstream dependency)
