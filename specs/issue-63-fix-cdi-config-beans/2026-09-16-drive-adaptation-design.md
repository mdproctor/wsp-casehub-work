# Drive Adaptation from Reward Signals

**Issue:** casehubio/examples#66
**Branch:** issue-63-fix-cdi-config-beans
**Date:** 2026-09-16

## Problem

Character motivational drives (e.g., Hooded Claw's "scheming" at 0.9, Penelope's "social-harmony" at 0.8) are static YAML values that never change regardless of what happens in the game. Characters should develop preferences from experience: actions that produce positive outcomes strengthen the associated drive; actions that produce negative outcomes weaken it. Penelope stays helpful because helping consistently produces pleasure — not because her config says so.

## Architecture

### Two Drive Systems (D16)

The platform has two independent drive systems that model motivation at different abstraction levels:

1. **SDT psychological needs** (blocks-core, `DriveOrchestrator`): Four fixed axes — CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY — dynamically computed each tick from interaction patterns (knowledge gaps, strategy trends, relationship familiarity, mental model pressure). Rendered by `DrivePromptSection` under "Psychological Needs."

2. **Character motivational drives** (application-level, `SocialConfig.Drive`): Free-form author-defined motivations — "scheming", "social-harmony", "gallantry" — expressing character identity. Currently static from YAML. **This is what #66 adapts.** Rendered under "Character Motivations."

They are complementary: SDT AFFILIATION being high and `social-harmony` being high are reinforcing signals at different levels of description. The cognitive preamble (D3) disambiguates them for the LLM.

### Data Flow

```
Bootstrap (once per scenario):
  social-config.yaml → ManorCognitiveSeeder → drive MindMap nodes
                                                (initial intensities)

Game loop (every tick):
  agent actions → experience memories (with PAD values)

Consolidation (periodic, when idle):
  ExperienceConsolidationPhase → graduated experience nodes
                                   (event-type, pleasure, arousal)
        ↓
  DriveAdaptationPhase → reads graduated nodes
                       → maps event-types to drives (per-character config)
                       → computes reward signal (configurable PAD axis)
                       → applies multiplicative intensity update
                       → writes updated drive nodes

Rendering (every prompt):
  CharacterDrivePromptSection → reads drive nodes from MindMap
                              → renders "Character Motivations" section
```

## Components

### 1. DriveAdaptationPhase (blocks-core)

**Location:** `blocks-core`, `io.casehub.blocks.agentic.social.drive.adaptation`
**Implements:** `ConsolidationPhase` with `@Priority(17)` — runs after ExperienceConsolidationPhase (15), before MergeDetectionPhase (20).

**Algorithm per agent per consolidation pass:**

1. **Load drive nodes** — query MindMapStore for nodes in the COGNITIVE subgraph with `cognitiveKind: "drive-intensity"` and matching `agent-id`. If no drive nodes found, emit WARNING log and skip (D14).

2. **Load new experience nodes** — query for nodes with `provenance: "experience-consolidation"` that have not yet been processed by this phase. DriveAdaptationPhase maintains its own sentinel node (name: `drive-adaptation-cursor`, in the COGNITIVE subgraph) with a `last-processed-node-id` property. On each pass, it loads experience nodes by subgraph and filters out nodes whose IDs are lexicographically ≤ the cursor. After processing, it updates the cursor to the last processed node's ID. This mirrors ExperienceConsolidationPhase's cursor pattern but operates on mindmap nodes rather than memory store entries.

3. **Aggregate reward per action-type** — for each experience node:
   - Read `event-type` property
   - Read the PAD value specified by the mapping's `rewardAxis` (default: pleasure)
   - Read `arousal` as the intensity modulator
   - Compute weighted reward: `reward = padValue * (1 + |arousal| * arousalWeight)`

4. **Map to drives** — using the per-character reinforcement config (`Map<String, List<DriveReinforcementEntry>>`), look up which drives are reinforced by each event-type. Each mapping entry specifies: drive-type, direction (POSITIVE/NEGATIVE), and reward axis.

5. **Apply multiplicative update** — for each drive with accumulated reward:
   ```
   delta = aggregatedReward * learningRate
   if direction == NEGATIVE: delta = -delta
   newIntensity = currentIntensity * (1 + delta)
   clamp(newIntensity, 0.1, 1.0)
   ```

6. **Write updated nodes** — update the `intensity` property on each drive's MindMap node. Update the phase cursor.

**Configuration (Smallrye config, prefix `casehub.drive-adaptation`):**

| Property | Default | Description |
|----------|---------|-------------|
| `learning-rate` | 0.05 | Scaling factor for intensity updates |
| `arousal-weight` | 0.3 | How much arousal modulates reward magnitude |
| `min-intensity` | 0.1 | Floor — drives never fully extinguish |
| `max-intensity` | 1.0 | Ceiling — existing range |
| `max-per-pass` | 20 | Maximum experience nodes to process per pass |

**Multiplicative dampening (GE-20260714-439924):** The update formula uses `intensity *= (1 + delta)`, not `intensity += delta`. Multiplicative updates are self-dampening: a drive at 0.9 gaining 5% goes to 0.945 (+0.045 absolute), while a drive at 0.3 gaining 5% goes to 0.315 (+0.015 absolute). Strong drives change slowly; weak drives change even more slowly. This prevents runaway accumulation at game-loop frequency.

**Volume dampening (GE-20260820-d9129a):** When aggregating reward from multiple experience nodes of the same event-type in a single pass, apply diminishing returns: `effectiveReward = totalReward / (1 + log(count))`. A single strong interaction has more impact than many weak ones. This prevents a consolidation pass with many observations from dominating drive evolution.

### 2. DriveReinforcementConfig (blocks-core)

**Location:** `blocks-core`, same package as DriveAdaptationPhase.

**SPI:** An injectable configuration providing `Map<String, List<DriveReinforcementEntry>>` — event-type → list of reinforcement entries.

```java
public record DriveReinforcementEntry(
    String driveType,
    ReinforcementDirection direction,
    RewardAxis rewardAxis
) {
    public DriveReinforcementEntry {
        Objects.requireNonNull(driveType);
        if (direction == null) direction = ReinforcementDirection.POSITIVE;
        if (rewardAxis == null) rewardAxis = RewardAxis.PLEASURE;
    }
}

public enum ReinforcementDirection { POSITIVE, NEGATIVE }
public enum RewardAxis { PLEASURE, DOMINANCE, COMPOSITE }
```

**COMPOSITE** computes `0.6 * pleasure + 0.2 * arousal + 0.2 * dominance`.

The config is provided by the application. Wacky-manor supplies it via YAML loading and a CDI producer.

### 3. YAML Configuration (wacky-manor)

**Location:** `social-config.yaml`, new `reinforcement` section per character.

```yaml
penelope-pitstop:
  drives:
    - type: curiosity
      intensity: 0.7
      description: "Drawn to puzzles and mysteries"
    - type: social-harmony
      intensity: 0.8
      description: "Wants everyone to get along"
    - type: adventure
      intensity: 0.6
      description: "Delights in new experiences"
  reinforcement:
    social_interaction:
      - drive: social-harmony
      - drive: adventure
    conflict_resolution:
      - drive: social-harmony
        reward-axis: DOMINANCE
    observation:
      - drive: curiosity
      - drive: adventure
    trust_change:
      - drive: social-harmony

hooded-claw:
  drives:
    - type: scheming
      intensity: 0.9
      description: "Compelled to hatch elaborate plans"
    - type: self-preservation
      intensity: 0.7
      description: "Avoids direct confrontation"
    - type: dominance
      intensity: 0.6
      description: "Must be most powerful in every room"
  reinforcement:
    conflict_resolution:
      - drive: scheming
      - drive: dominance
        reward-axis: DOMINANCE
    social_interaction:
      - drive: scheming
        direction: NEGATIVE
    trust_change:
      - drive: self-preservation
    observation:
      - drive: scheming
```

**Key modeling decisions in the YAML:**
- `social_interaction → scheming: NEGATIVE` for Hooded Claw — pleasant socializing weakens his scheming identity (he's supposed to be plotting, not making friends).
- `conflict_resolution → social-harmony: reward-axis=DOMINANCE` for Penelope — a stressful but successful mediation strengthens her harmony drive (she maintained control) even though pleasure may be low. This addresses the D12 revision.
- Default `direction: POSITIVE` and `reward-axis: PLEASURE` when omitted.

### 4. Drive Node Seeding (wacky-manor)

**Location:** `ManorCognitiveSeeder` (already enhanced in #283 for goal seeding).

At scenario bootstrap, the seeder creates one MindMap node per drive per agent in the COGNITIVE subgraph:

```java
NodeInput driveNode = NodeInput.of(
        drive.type(),   // node name = drive type
        subgraphId)     // COGNITIVE subgraph
    .withConfidence(MindMapConfidenceDefaults.forOrigin(
        ConfidenceOrigin.AUTHORED, Instant.now()))
    .withProvenance("drive-adaptation")
    .withProperties(Map.of(
        "cognitiveKind", "drive-intensity",
        "agent-id", agentId,
        "drive-type", drive.type(),
        "intensity", String.valueOf(drive.intensity()),
        "initial-intensity", String.valueOf(drive.intensity()),
        "description", drive.description()));
```

**Idempotent:** If a node with `cognitiveKind: "drive-intensity"` and matching `agent-id` + `drive-type` already exists, skip seeding. Consistent with existing seeder idempotency for beliefs and relationships.

### 5. CharacterDrivePromptSection (blocks-core)

**Location:** `blocks-core`, `io.casehub.blocks.agentic.social.prompt`
**Implements:** `PromptSection`

Reads adapted drive nodes from MindMapStore, formats them as an observation section under "Character Motivations":

```
## Character Motivations

- scheming (92%) — Compelled to hatch elaborate plans
- self-preservation (68%) — Avoids direct confrontation
- dominance (63%) — Must be most powerful in every room
```

Sorted by intensity descending (same format as current CharacterCognition rendering). The section heading distinguishes this from the SDT "Psychological Needs" section rendered by `DrivePromptSection`.

**Registration:** Added to `CognitionCore.promptSections()` when drive adaptation is enabled (new `CognitionConfig` flag: `characterDrivesEnabled`). This ensures it participates in the standard observation section pipeline.

### 6. CharacterCognition Changes (wacky-manor)

**Remove drive rendering (D18):** `CharacterCognition.renderCognitiveSections()` lines 106-111 (the static drive rendering from SocialConfig) are removed. Drives are now rendered by `CharacterDrivePromptSection` through the CognitionCore pipeline.

**Remove CognitionCore delegation (D18):** `CharacterCognition.renderCognitiveSections()` lines 130-138 (the `cognitionCore.promptSections()` delegation) are removed. CognitionCore sections are wired at the `ObservationBuilder` level, not aggregated by CharacterCognition. CharacterCognition retains only application-specific sections: initial beliefs, norms, social awareness, trust perceptions.

### 7. Cognitive Preamble Update (D3)

The preamble generated by `CognitivePreambleGenerator` includes drive disambiguation text when both `drivesEnabled` and `characterDrivesEnabled` are true:

> Your psychological needs — curiosity, competence, affiliation, autonomy — shift based on your interactions. Separately, your character motivations — the drives that define who you are — evolve based on your experiences.

## Cross-Repo Scope

| Repo | Changes | Issue |
|------|---------|-------|
| **blocks** | `DriveAdaptationPhase`, `DriveReinforcementEntry`, `DriveReinforcementConfig`, `CharacterDrivePromptSection`, `CognitivePreambleGenerator` update, `CognitionConfig` new flag | Existing casehubio/blocks#283 scope extends |
| **examples/wacky-manor** | `social-config.yaml` reinforcement section, `ManorCognitiveSeeder` drive seeding, `CharacterCognition` deduplication, reinforcement config CDI producer | casehubio/examples#66 |
| **neocortex** | No changes — MindMapStore API is sufficient | None |

## Empirical Validation

1. **Convergence test:** Run a scenario with 50+ ticks. Verify drives converge toward stable values (no oscillation). A character that consistently succeeds at social interactions should see social-related drives strengthen.

2. **Floor/ceiling test:** Verify drives never drop below 0.1 or exceed 1.0 regardless of extreme reward signals.

3. **Negative reinforcement test:** Verify that Hooded Claw's scheming drive weakens after extended pleasant socializing (NEGATIVE direction mapping).

4. **PAD axis test:** Verify that Penelope's social-harmony strengthens after a stressful-but-successful mediation (DOMINANCE reward axis, low pleasure, high dominance).

5. **Cold-start parity:** Compare agent behavior at tick 1 (seeded drives = YAML values) with current static behavior. Should be identical.

## References

- `SocialConfig.Drive` — `wacky-manor/.../agent/SocialConfig.java:39`
- `DriveOrchestrator` — `blocks-core/.../social/drive/DriveOrchestrator.java`
- `DriveAxis` — `blocks-core/.../social/drive/DriveAxis.java`
- `DriveSource` — `blocks-core/.../social/drive/DriveSource.java`
- `DrivePromptSection` — `blocks-core/.../social/prompt/DrivePromptSection.java`
- `ExperienceConsolidationPhase` — `neocortex/mindmap-intelligence/.../consolidation/ExperienceConsolidationPhase.java`
- `ConsolidationPhase` — `neocortex/mindmap-intelligence/.../consolidation/ConsolidationPhase.java`
- `ConsolidationScheduler` — `neocortex/mindmap-intelligence/.../consolidation/ConsolidationScheduler.java`
- `ActionImportanceScorer` — `wacky-manor/.../engine/ActionImportanceScorer.java`
- `ManorGraduationScorer` — `wacky-manor/.../engine/ManorGraduationScorer.java`
- `CharacterCognition` — `wacky-manor/.../agent/CharacterCognition.java`
- `ManorCognitiveSeeder` — `wacky-manor/.../agent/ManorCognitiveSeeder.java`
- `social-config.yaml` — `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`
- GE-20260714-439924 — multiplicative dampening (additive penalties at game-loop frequency zero confidence in under 1 second)
- GE-20260820-d9129a — familiarity volume factor (Laplace smoothing alone lets single interactions jump too high)
- Decisions — `specs/issue-63-fix-cdi-config-beans/decisions.md` (D9–D18)
