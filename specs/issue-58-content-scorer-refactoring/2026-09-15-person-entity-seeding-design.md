# Person-Entity Node Seeding — Design Spec

**Issue:** casehubio/examples#61
**Parent:** casehubio/examples#53 (Phase B: mindmap-native cognitive integration)
**Branch:** issue-58-content-scorer-refactoring

## Problem

SocialComparison integration (#60) is wired, but the pipeline is inert — no entity nodes with PAD values and perspectival overlays exist in the mindmap. `CognitiveProfile.compare()` returns empty maps because `resolveNode()` finds nothing. The pipeline gracefully degrades but produces no meaningful output.

## Solution

Extend ManorCognitiveSeeder to create shared person nodes with perspectival overlays containing initial PAD values. Extend SocialConfig with relationship data so initial social dynamics are explicit and tunable via YAML.

## Changes

All changes are in **examples/wacky-manor** — no upstream repo changes needed.

### SocialConfig extension

Add `Relationship` record and `relationships` field:

```java
public record Relationship(String targetAgentId, double pleasure, double arousal, double dominance) {
    public Relationship {
        Objects.requireNonNull(targetAgentId);
        if (pleasure < -1.0 || pleasure > 1.0) throw new IllegalArgumentException("pleasure must be in [-1,1]");
        if (arousal < -1.0 || arousal > 1.0) throw new IllegalArgumentException("arousal must be in [-1,1]");
        if (dominance < -1.0 || dominance > 1.0) throw new IllegalArgumentException("dominance must be in [-1,1]");
    }
}
```

PAD range is [-1, 1] matching the standard PAD model (not [0, 1]).

### ManorSocialConfigLoader extension

Parse `relationships:` from YAML:

```yaml
penelope:
  relationships:
    - target: peter
      pleasure: 0.6
      arousal: 0.3
      dominance: 0.5
    - target: muttley
      pleasure: -0.2
      arousal: 0.7
      dominance: 0.3
```

### ManorCognitiveSeeder extension

New `seedPeople()` method:

1. **Create shared "people" subgraph** — idempotent, reuses if exists. Subgraph name: `"people"`.
2. **Create shared person nodes** — one per unique character, trait `"Entitylike"`, property `"agentId"` → character id. Neutral PAD (0.0/0.0/0.0) on the shared node — divergent values live in overlays.
3. **Create perspectival overlay nodes** — for each observer→target relationship in SocialConfig:
   - Trait: `"overlay"`
   - Ref: `OverlayRef.of(sharedPersonNodeId)`
   - Property: `OverlayRef.AGENT_ID` → observer's agentId
   - PAD: from `Relationship(pleasure, arousal, dominance)`
   - PrincipalId: `PrincipalId.agent(observerAgentId)`
   - Provenance: `"manor-seed"`

### Idempotency

`seedPeople()` checks for existing shared nodes before creating. If a person node for a character already exists, it skips creation. Overlays are created only if no overlay exists for the observer→target pair.

### Data flow

```
social-config.yaml
  → ManorSocialConfigLoader.load()
    → SocialConfig.relationships()
      → ManorCognitiveSeeder.seedPeople(allConfigs, tenantId)
        → MindMapStore.createSubgraph("people")
        → MindMapStore.addNode(shared person node per character)
        → MindMapStore.addNode(overlay per observer→target with PAD)
          → CognitiveProfile.compare() resolves shared + overlays
            → PerspectivalMerge.merge() applies PAD divergence
              → PerceptionTranslator.translate() produces output
```

## Testing

- **SocialConfig.Relationship**: construction, PAD range validation, null targetAgentId
- **ManorSocialConfigLoader**: parse relationships from YAML, missing relationships key defaults to empty
- **ManorCognitiveSeeder.seedPeople()**: creates shared subgraph, creates person nodes, creates overlay nodes with correct PAD/traits/refs, idempotency (second call creates no duplicates)
- **Integration**: seeded nodes are resolvable by CognitiveProfile.compare()

## References

- `io.casehub.examples.manor.agent.ManorCognitiveSeeder` — existing seeder (beliefs only)
- `io.casehub.examples.manor.agent.SocialConfig` — extended with Relationship
- `io.casehub.examples.manor.agent.ManorSocialConfigLoader` — extended to parse relationships
- `io.casehub.neocortex.mindmap.OverlayRef` — overlay node linking pattern
- `io.casehub.neocortex.cognitive.index.PerspectivalResolver` — loads overlays by "overlay" trait
- `io.casehub.neocortex.cognitive.index.CognitiveProfile#compare` — resolves shared node + overlays → EntityKnowledge
- `io.casehub.neocortex.mindmap.NodeInput` — withPad(), withTraits(), withRefs(), withProperties()
- `io.casehub.examples.manor.agent.PerceptionTranslator` — downstream consumer (produces output once data exists)
- casehubio/examples#60 — SocialComparison integration (design review R1-02 identified this gap)
