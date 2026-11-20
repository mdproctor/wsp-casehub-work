# Design Spec — Belief Revision from Contradicting Evidence (#67)

## Summary

Characters have seeded beliefs (e.g. "Penelope is naive and trusts too easily") that never change regardless of gameplay. This adds a consolidation phase that uses LLM-assessed semantic contradiction detection to gradually erode belief confidence when accumulated evidence contradicts a belief, and ultimately supersedes stale beliefs with LLM-generated revised beliefs when confidence drops below a threshold.

## What Exists

**Already in neocortex (platform):**
- `ConsolidationPhase` — SPI interface: `String name()`, `void run(String tenantId, List<String> subgraphPriority)`
- `ConsolidationScheduler` — runs phases sequentially on a daemon thread, per-phase try-catch, gated by `IdleTracker.isIdle(Duration.ofMinutes(1))`
- `ExperienceConsolidationPhase` (@Priority 15) — graduates memories into cognitive nodes via `GraduationClassifier` and `GraduationScorer`
- `MindMapStore.supersede(targetId, supersedingId, reason, tenantId)` — marks target node as superseded; history tracked via `SupersessionStatus`
- `Confidence` record — `(ConfidenceOrigin origin, double value, Instant decayReference)` with `withValue(double)` for adjustment. Origins: STATED, INFERRED, SPECULATED, UNKNOWN
- `SupersessionStatus` — tracks supersession history: target, superseder, reason, timestamps, reinstatement
- `GraduationClassifier` — classifies memories into cognitive kinds; `GraduationResult(String cognitiveKind, ConfidenceOrigin confidenceOrigin, Map<String,String> properties)`

**Already in blocks-core:**
- `AgentProvider` — platform SPI for LLM invocation, already used by `UserModelOrchestrator` for LLM synthesis
- `DriveAdaptationPhase` (@Priority 17) — consolidation phase pattern to follow

**Already in wacky-manor:**
- `ManorCognitiveSeeder.seed()` — seeds Belieflike nodes in per-agent `beliefs-{agentId}` subgraphs with `Confidence.stated(0.8, now)`, trait `Belieflike`, property `subject`, provenance `manor-seed`
- `CharacterCognition.renderCognitiveSections()` — currently reads beliefs from `socialConfig.initialBeliefs()` (static YAML), not from MindMapStore

**Already in blocks-core (AGM framework):**
- `Belief<T>` record — `(String key, T value, int entrenchment)` — formal belief representation with epistemic ordering
- `BeliefSet<T>` — immutable keyed collection with AGM operations: `expand()`, `contract()`, `revise(belief, checker)`
- `ConsistencyChecker<T>` — `@FunctionalInterface boolean isConsistent(BeliefSet<T>)` — consistency hook
- `CognitiveObservationSections.beliefsSection(List<Belief<?>>, Set<String> revisedKeys)` — renders beliefs sorted by entrenchment, marks revised beliefs with `[REVISED]` prefix
- Initial beliefs in `social-config.yaml`: hooded-claw ("Penelope is naive and trusts too easily", "Peter Perfect is protective but predictable"), penelope-pitstop ("Sylvester Sneekly is a helpful estate manager", "Everyone here means well"), etc.

## Architecture

### 1. BeliefRevisionPhase (D24, D27)

New `BeliefRevisionPhase` in blocks-core, `@Priority(16)` — after ExperienceConsolidation@15 (which creates graduated nodes), before DriveAdaptation@17.

**Input:** `MindMapStore` (belief and graduated nodes), `AgentProvider` (LLM calls), `BeliefRevisionConfig` (decay/threshold parameters).

**Algorithm per consolidation cycle:**
1. List all `cognitive`-type subgraphs in the tenant
2. Scan ALL cognitive-type subgraphs, collecting: (a) Belieflike nodes from `beliefs-{agentId}` subgraphs (trait `Belieflike`, not superseded), and (b) newly graduated experience nodes from any cognitive subgraph (provenance `experience-consolidation`)
3. Use a cursor node (same pattern as `DriveAdaptationPhase`) to track which graduated nodes have already been processed — only process nodes newer than the cursor
4. Group by agent: beliefs via `node.principalId().id()` (returns agentId from `PrincipalId.agent(agentId)`), evidence via `node.properties().get("agent-id")` (string property). These are two different access paths that resolve to the same agentId string.
5. Per agent: if the agent has both active beliefs and new evidence, invoke LLM contradiction detection (§2)
6. Apply confidence decay for detected contradictions (§3)
7. If any belief's confidence drops below threshold, supersede it (§4)
8. Update cursor

**Cross-subgraph note:** Beliefs and graduated evidence typically live in different subgraphs — beliefs in `beliefs-{agentId}`, evidence in whichever cognitive subgraph `ExperienceConsolidationPhase.findOrCreateCognitiveSubgraph()` selects. The algorithm scans all cognitive subgraphs and groups by agent to bridge this split.

**Failure handling (D29):** LLM call failures are caught, logged as WARNING, and the phase returns without modifying beliefs. ConsolidationScheduler's per-phase try-catch ensures subsequent phases (DriveAdaptation@17, RelationshipStage@18) proceed normally.

### 2. LLM Contradiction Detection (D25)

One LLM call per agent per consolidation cycle. The prompt presents all active beliefs and all new evidence in a single batch.

**System prompt:**
```
You are analyzing a character's beliefs against recent evidence from their experiences.
For each belief that is contradicted by the evidence, identify the contradiction.
Respond with JSON only. If no contradictions are found, return {"contradictions": []}.
```

**User prompt:**
```
Character: {agentId}

Current beliefs:
1. [{beliefNodeId}] "{beliefText}" (confidence: {confidence})
2. [{beliefNodeId}] "{beliefText}" (confidence: {confidence})
...

Recent evidence:
- "{graduatedNodeText}" (event-type: {eventType})
- "{graduatedNodeText}" (event-type: {eventType})
...

For each belief contradicted by this evidence, provide:
- which belief (by ID)
- what evidence contradicts it
- how strong the contradiction is (0.0 = barely relevant, 1.0 = directly disproven)
- what the character should now believe (one sentence, from their perspective)
```

**Structured JSON output schema:**
```json
{
  "contradictions": [
    {
      "beliefNodeId": "node-id-123",
      "beliefText": "Penelope is naive and trusts too easily",
      "contradictingEvidence": "Penelope saw through the disguise and alerted Peter",
      "reasoning": "Penelope demonstrated perceptiveness, directly contradicting the belief that she is naive",
      "contradictionStrength": 0.8,
      "revisedBelief": "Penelope is more perceptive than she appears"
    }
  ]
}
```

The `contradictionStrength` field (0.0–1.0) modulates the confidence decay (§3). The `revisedBelief` field is used only when confidence drops below the supersession threshold — it is pre-generated in every response to avoid a second LLM call.

### 3. Confidence Decay (D26)

For each contradiction detected by the LLM:

```
effectiveDecay = beliefDecayPerContradiction × contradictionStrength
newConfidence = currentConfidence - effectiveDecay
```

- `beliefDecayPerContradiction` (default: 0.15) — configurable base decay per contradiction
- `contradictionStrength` (0.0–1.0) — from LLM output
- Confidence is clamped to [0.0, 1.0]

The decay creates a new `Confidence` with INFERRED origin and reset decay reference:

```java
new Confidence(ConfidenceOrigin.INFERRED, newConfidence, Instant.now())
```

- **Origin transition:** A seeded belief starts as STATED. After the first contradiction decay, its origin becomes INFERRED — the confidence is no longer author-stated, it has been computationally modified. This prevents `MindMapQuery.withConfidenceOrigin(STATED)` from returning machine-adjusted beliefs.
- **Decay reference reset:** Setting `decayReference` to `Instant.now()` prevents `ConfidenceDecayDecorator` (which applies exponential time-based decay using `decayReference` as the base timestamp) from compounding with contradiction-based decay. Without this reset, a belief decayed to 0.68 would be further reduced by time-based decay calculated from the original seed timestamp.

**Example progression** (default config, starting at 0.8):
- Strong contradiction (0.8): `0.8 - (0.15 × 0.8) = 0.68`
- Another strong (0.8): `0.68 - 0.12 = 0.56`
- Weak contradiction (0.3): `0.56 - 0.045 = 0.515`
- Strong contradiction (0.9): `0.515 - 0.135 = 0.38`
- Strong contradiction (0.8): `0.38 - 0.12 = 0.26` → below threshold (0.3), triggers supersession

### 4. Belief Supersession (D26, D28)

When a belief's confidence drops below `beliefSupersessionThreshold` (default: 0.3):

1. Create a new Belieflike node in the same subgraph:
   - Name: the `revisedBelief` text from the LLM's JSON output
   - Trait: `Belieflike`
   - Property `subject`: same as the superseded belief's subject
   - Confidence: `Confidence.inferred(0.6, Instant.now())` — INFERRED origin distinguishes evolved beliefs from authored (STATED) ones. Starting at 0.6 (not 0.8) reflects that the revised belief is less certain than an authored seed.
   - Provenance: `belief-revision`
   - PrincipalId: same agent as the superseded belief

2. Supersede the old belief: `mindMapStore.supersede(oldNodeId, newNodeId, reason, tenantId)`
   - Reason: the LLM's `reasoning` text (e.g., "Penelope demonstrated perceptiveness, directly contradicting the belief that she is naive")

3. The superseded belief remains in the mindmap with `SupersessionStatus.superseded = true`. Characters can "remember what they used to believe" — the history is queryable.

### 5. Configuration (D27)

New `BeliefRevisionConfig` record in blocks-core:

```java
public record BeliefRevisionConfig(
    double beliefDecayPerContradiction,
    double beliefSupersessionThreshold,
    double revisedBeliefInitialConfidence
) {
    public static BeliefRevisionConfig defaults() {
        return new BeliefRevisionConfig(0.15, 0.3, 0.6);
    }
}
```

**CDI wiring:** `BeliefRevisionPhase` is NOT a CDI bean. Like `DriveAdaptationPhase`, it is instantiated via constructor by the application. Wacky-manor wires it in its CDI producer method alongside the other consolidation phases, passing `MindMapStore`, `AgentProvider`, and `BeliefRevisionConfig` as constructor arguments. The only annotation is `@Priority(16)` for phase ordering. No `@ApplicationScoped`, no `@Inject`.

### 6. Rendering Transition

`CharacterCognition.renderCognitiveSections()` currently reads beliefs from `socialConfig.initialBeliefs()` — static YAML parsed at boot. Without modifying this, the entire belief revision mechanism is invisible to the character's behavior: the LLM prompt continues showing original beliefs regardless of supersession.

**Required change:** Replace the static belief rendering with MindMapStore-sourced rendering:

1. Query `beliefs-{agentId}` subgraph for Belieflike nodes (non-superseded)
2. Map each `MindMapNode` to `Belief<String>` using: `key = node.property("subject")`, `value = node.name()`, `entrenchment = (int)(node.confidence().value() * 10)` (maps Confidence 0.0–1.0 to entrenchment 0–10, per the #54 design bridge)
3. Collect keys of recently superseded beliefs into a `Set<String> revisedKeys`
4. Render via `CognitiveObservationSections.beliefsSection(beliefs, revisedKeys)` — this already sorts by entrenchment (descending) and marks revised beliefs with `[REVISED]` prefix

This transitions CharacterCognition from Phase A (static config) to Phase B (MindMapStore-sourced), as planned in issue #52's design.

### Relationship to AGM Framework

blocks-core contains a formal AGM belief revision framework: `Belief<T>` (key, value, entrenchment), `BeliefSet<T>` (immutable set with `expand()`, `contract()`, `revise()`), and `ConsistencyChecker<T>` (boolean consistency check).

These two models serve different purposes and coexist:

- **AGM contraction** is binary — a belief is in the set or it isn't. Entrenchment determines which beliefs survive when consistency is violated. This is appropriate for logical consistency operations.
- **Confidence decay** is gradual epistemic erosion — continuous, per-contradiction, with variable strength. A belief doesn't disappear on the first contradiction; it weakens over time until superseded.

`ConsistencyChecker.isConsistent(BeliefSet)` is boolean — it cannot return *which* belief is contradicted or *how strongly*. The LLM batch assessment (D25) provides richer output: specific belief identification, reasoning, and contradiction strength. The AGM framework is not a substitute for the LLM assessment, but the rendering layer bridges both models: `CognitiveObservationSections.beliefsSection(List<Belief<?>>, Set<String> revisedKeys)` renders from `Belief<T>`, and the MindMapNode → Belief mapping (§6) feeds this rendering path regardless of whether the confidence change came from AGM contraction or gradual decay.

## Data Flow

```
scenario bootstrap
  └─ ManorCognitiveSeeder.seed()
       └─ Belieflike nodes created in beliefs-{agentId} subgraph
          confidence: 0.8, origin: STATED, provenance: manor-seed

interaction loop
  └─ recordExperience() stores memories

consolidation (sleep cycle)
  ├─ ExperienceConsolidationPhase (@Priority 15)
  │    └─ graduates memories → cognitive nodes (provenance: experience-consolidation)
  │
  ├─ BeliefRevisionPhase (@Priority 16)
  │    ├─ scans ALL cognitive subgraphs for Belieflike + new graduated nodes
  │    ├─ groups by agent (principalId for beliefs, agent-id property for evidence)
  │    ├─ LLM call: batch contradiction detection per agent
  │    │    └─ returns: [{beliefNodeId, contradictionStrength, revisedBelief}]
  │    ├─ for each contradiction:
  │    │    ├─ decay confidence: current - (baseDecay × strength)
  │    │    ├─ transition origin STATED → INFERRED, reset decayReference
  │    │    └─ if below threshold → supersede + create revised belief
  │    └─ update cursor
  │
  ├─ DriveAdaptationPhase (@Priority 17)
  ├─ RelationshipStagePhase (@Priority 18)
  ├─ MergeDetectionPhase (@Priority 20)
  └─ TrustConsolidationPhase (@Priority 22)

rendering (prompt construction)
  └─ CharacterCognition.renderCognitiveSections()
       └─ queries beliefs-{agentId} subgraph for non-superseded Belieflike nodes
          maps to Belief<T>, renders via CognitiveObservationSections.beliefsSection()
```

## Files Changed

### blocks-core (new)
| File | Change |
|------|--------|
| `BeliefRevisionPhase.java` | New ConsolidationPhase, @Priority(16) — LLM contradiction detection, confidence decay, supersession |
| `BeliefRevisionConfig.java` | Configuration record: decay, threshold, initial revised confidence |

### wacky-manor (modified)
| File | Change |
|------|--------|
| `CharacterCognition.java` | Replace static `socialConfig.initialBeliefs()` rendering with MindMapStore-sourced Belieflike node query + `CognitiveObservationSections.beliefsSection()` |

### wacky-manor (tests)
| File | Change |
|------|--------|
| `BeliefRevisionIntegrationTest.java` | End-to-end: seed beliefs → add graduated evidence → run phase → verify confidence decay and supersession |

## Trade-offs Acknowledged

- **LLM in consolidation (D29):** BeliefRevisionPhase is the first consolidation phase to invoke an LLM. ConsolidationScheduler's per-phase try-catch and idle-time gating handle failure isolation and latency. LLM calls that hang block subsequent phases until the next scheduler tick — AgentProvider implementations must enforce their own call timeouts.
- **Non-deterministic contradiction detection:** The LLM may assess the same (belief, evidence) pair differently across runs. Acceptable — belief revision is inherently subjective. The `contradictionStrength` modulates a configurable base decay, bounding extreme variation.
- **Batch prompt limitations (D25):** One LLM call per agent amortizes cost but may miss subtle pairwise contradictions. Characters have 4-6 beliefs and consolidation graduates ~5-20 nodes per cycle, so the prompt is bounded. Can switch to pairwise for high-value beliefs in a future iteration.
- **Uniform base decay (D26):** While `contradictionStrength` varies, the `baseDecay` is uniform across all beliefs. A deeply-held belief and a casual assumption both decay at the same base rate. Acceptable for v1 — per-belief decay rates would add configuration complexity with marginal benefit.
- **Revised belief confidence (D28):** Revised beliefs start at 0.6 (INFERRED), not 0.8 (STATED). This means a revised belief can itself be superseded more quickly by further contradicting evidence. Intentional — evolved beliefs should be less entrenched than authored ones.
- **Pre-generated revised text:** The `revisedBelief` field is generated in every LLM response, even when confidence hasn't reached the threshold yet. The unused text is discarded. This avoids a second LLM call when supersession does trigger, at the cost of generating text that may never be used.

## References

- `ConsolidationPhase.java` (neocortex-mindmap-intelligence) — SPI interface
- `ConsolidationScheduler` (neocortex-mindmap-intelligence) — per-phase try-catch, IdleTracker gating
- `ExperienceConsolidationPhase.java` (neocortex-mindmap-intelligence:88-153) — graduation pattern, @Priority(15)
- `DriveAdaptationPhase.java` (blocks-core) — consolidation phase pattern, cursor mechanism
- `MindMapStore.supersede()` (neocortex-mindmap-api) — supersession API
- `Confidence.java` (neocortex-cognitive-api) — confidence model with `withValue()`, STATED/INFERRED origins
- `SupersessionStatus.java` (neocortex-mindmap-api) — supersession history tracking
- `GraduationClassifier.java` (neocortex-memory-api) — classification interface
- `ManorCognitiveSeeder.seed()` (wacky-manor:100-109) — belief seeding with Belieflike trait
- `AgentProvider.java` (platform-agent-api) — LLM invocation SPI
- `UserModelOrchestrator.java` (blocks-core) — LLM synthesis pattern in blocks-core
- `Belief.java`, `BeliefSet.java`, `ConsistencyChecker.java` (blocks-core, `io.casehub.blocks.agentic.belief`) — AGM framework
- `CognitiveObservationSections.beliefsSection()` (blocks-core) — belief rendering with revised marking
- `CharacterCognition.renderCognitiveSections()` (wacky-manor:99-129) — current static belief rendering (to be modified)
- Issue #67, #52, #54
