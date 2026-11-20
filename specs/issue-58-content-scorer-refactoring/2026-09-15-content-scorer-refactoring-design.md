# ContentScorer Refactoring — Design Spec

**Issue:** casehubio/examples#58
**Parent:** casehubio/examples#53 (Phase B: mindmap-native cognitive integration)
**Branch:** issue-58-content-scorer-refactoring

## Problem

Consolidation (ExperienceConsolidationPhase) graduates all Tier 2 episodic events equally via GraduationScorer. Blocks has scoring heuristics (SurpriseScorer, ArousalScorer) that measure event importance, but they operate on `ScoredCbrCase` via `ConfidenceScorer` — a different interface and input type than `GraduationScorer(Memory)`. The scoring logic can't be composed across both contexts.

## Solution

Introduce `ContentScorer` and `ScoreableContent` in neocortex-memory-api as a domain-agnostic scoring layer. Scoring heuristics operate on extracted signals (text, metadata, timestamp), not domain types. Existing interfaces (ConfidenceScorer, GraduationScorer) become thin adapters over ContentScorer.

## Repos Touched

| Repo | Module | Changes |
|------|--------|---------|
| neocortex | memory-api | Add `ScoreableContent`, `ContentScorer`, `WeightedContentScorer` |
| blocks | (root) | Refactor `SurpriseScorer`, `ArousalScorer` to implement `ContentScorer` + retain `ConfidenceScorer` as adapter |
| examples | wacky-manor | Add `ActionImportanceScorer`, `ManorGraduationScorer` |

Dependency direction: `neocortex-memory-api ← blocks ← examples` — no new cross-repo deps.

## New Types (neocortex-memory-api)

### ScoreableContent

```java
package io.casehub.neocortex.memory.experience;

public record ScoreableContent(String text, Map<String, String> metadata, Instant timestamp) {
    public ScoreableContent {
        Objects.requireNonNull(text, "text required");
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }

    public static ScoreableContent fromMemory(Memory memory) {
        return new ScoreableContent(
            memory.text(),
            memory.attributes(),
            memory.createdAt());
    }
}
```

### ContentScorer

```java
package io.casehub.neocortex.memory.experience;

@FunctionalInterface
public interface ContentScorer {
    double score(ScoreableContent content);

    static ContentScorer composite(List<WeightedContentScorer> scorers) {
        // weighted mean, clamped to [0,1]
    }

    static ContentScorer weighted(ContentScorer scorer, double weight) {
        return new WeightedContentScorer(scorer, weight);
    }
}
```

### WeightedContentScorer

```java
package io.casehub.neocortex.memory.experience;

public record WeightedContentScorer(ContentScorer scorer, double weight) {
    public WeightedContentScorer {
        Objects.requireNonNull(scorer, "scorer required");
        if (weight <= 0.0) throw new IllegalArgumentException("weight must be positive");
    }
}
```

## Refactored Types (blocks)

### SurpriseScorer

Implements both `ContentScorer` and `ConfidenceScorer`. Real logic in `score(ScoreableContent)`, operating on metadata. `score(ScoredCbrCase, Instant)` extracts ScoreableContent from the CBR case and delegates.

```java
public final class SurpriseScorer implements ContentScorer, ConfidenceScorer {

    @Override
    public double score(ScoreableContent content) {
        Map<String, String> metadata = content.metadata();
        if (metadata == null || metadata.isEmpty()) return 0.5;
        int distinctValues = 0;
        for (var value : metadata.values()) {
            distinctValues += value.length();
        }
        return Math.clamp(Math.log1p(distinctValues) / 10.0, 0.0, 1.0);
    }

    @Override
    public double score(ScoredCbrCase<? extends CbrCase> memory, Instant now) {
        return score(toScoreableContent(memory, now));
    }

    private static ScoreableContent toScoreableContent(ScoredCbrCase<? extends CbrCase> m, Instant now) {
        // extract text from problem+solution, stringify features to Map<String,String>
    }
}
```

### ArousalScorer

Same dual-interface pattern. Real logic in `score(ScoreableContent)`, operating on text.

## New Types (examples/wacky-manor)

### ActionImportanceScorer

A game-specific `ContentScorer` that scores event-type significance from metadata attributes.

```java
public final class ActionImportanceScorer implements ContentScorer {
    private static final Map<String, Double> EVENT_WEIGHTS = Map.of(
        "conflict_resolution", 0.9,
        "trust_change", 0.8,
        "social_interaction", 0.6,
        "observation", 0.4,
        "idle", 0.1
    );

    @Override
    public double score(ScoreableContent content) {
        String eventType = content.metadata().getOrDefault("event-type", "unknown");
        return EVENT_WEIGHTS.getOrDefault(eventType, 0.3);
    }
}
```

### ManorGraduationScorer

Implements `GraduationScorer`, composes ContentScorer instances with game-specific weights.

```java
@ApplicationScoped
public class ManorGraduationScorer implements GraduationScorer {

    private final ContentScorer compositeScorer;

    public ManorGraduationScorer() {
        this.compositeScorer = ContentScorer.composite(List.of(
            ContentScorer.weighted(new ArousalScorer(), 0.3),
            ContentScorer.weighted(new ActionImportanceScorer(), 0.4)
        ));
    }

    @Override
    public double score(Memory memory) {
        ScoreableContent content = ScoreableContent.fromMemory(memory);
        double contentScore = compositeScorer.score(content);
        double confidence = memory.confidence() != null ? memory.confidence().value() : 0.5;
        return Math.clamp(contentScore * 0.7 + confidence * 0.3, 0.0, 1.0);
    }
}
```

The formula: `(arousal × 0.3 + actionImportance × 0.4) × 0.7 + confidence × 0.3` — the 0.7 normalises the content scorer composite (weights 0.3 + 0.4 = 0.7) so the three dimensions sum to 1.0.

## Impact

| Component | Change needed |
|-----------|--------------|
| MemoryHygieneOrchestrator | None — SurpriseScorer/ArousalScorer still implement ConfidenceScorer |
| CompositeConfidenceScorer | None |
| WeightedScorer | None |
| ConfidenceScorerRegistry | None |
| DefaultGraduationScorer | None — remains @DefaultBean fallback |
| ExperienceConsolidationPhase | None — consumes GraduationScorer SPI |

## Testing

- **ScoreableContent**: construction, fromMemory extraction, null/empty metadata
- **ContentScorer**: composite weighted mean calculation, single scorer, boundary values
- **SurpriseScorer**: dual interface — test both score(ScoreableContent) and score(ScoredCbrCase, Instant) produce consistent results
- **ArousalScorer**: dual interface — same consistency test
- **ActionImportanceScorer**: event-type mapping, unknown types default
- **ManorGraduationScorer**: formula verification, null confidence fallback, integration with ExperienceConsolidationPhase

## References

- `io.casehub.blocks.memory.ConfidenceScorer` — existing interface (blocks)
- `io.casehub.neocortex.memory.experience.GraduationScorer` — existing interface (memory-api)
- `io.casehub.blocks.memory.SurpriseScorer` — existing implementation (blocks)
- `io.casehub.blocks.memory.ArousalScorer` — existing implementation (blocks)
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ExperienceConsolidationPhase` — consumer
- `io.casehub.neocortex.mindmap.intelligence.consolidation.DefaultGraduationScorer` — default fallback
- GE-20260912-be7c74 — three-tier memory model (Tier 2 → Tier 3 graduation via importance scoring)
- GE-20260820-aa31ab — composite retention score masking (weighted mean caution)
- casehubio/examples#53 — parent issue (Phase B: mindmap-native cognitive integration)
