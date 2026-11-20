# Cognitive Pipeline Divergence Test Suite — Design Spec

## Goal

Prove that every cognitive pipeline capability produces measurably different output for different personalities. 48 capabilities identified, 6 currently tested, 42 untested. All tests run on the real CognitionTestHarness — no mocks, real CAPS engine, real derivation, real consolidation.

## Restructuring

### Extract shared fixtures

Create `DivergenceFixtures` in `io.casehub.neocortex.cognition.eval.divergence`:

```java
public final class DivergenceFixtures {
    public static final String TENANT = "test";

    public static final DispositionAxes OPEN_AXES = ...;
    public static final List<PersonalityFacet> OPEN_FACETS = ...;
    public static final DispositionAxes CAUTIOUS_AXES = ...;
    public static final List<PersonalityFacet> CAUTIOUS_FACETS = ...;

    public static DescriptorView openView() { ... }
    public static DescriptorView cautiousView() { ... }
    public static CognitiveDefaults openDefaults() { ... }
    public static CognitiveDefaults cautiousDefaults() { ... }

    // Pre-configured builder with both agents registered
    public static CognitionTestHarness.Builder pairedBuilder() { ... }
    // Builder with MindMapStore, both agents, behavioral synthesis
    public static CognitionTestHarness.Builder fullBuilder(InMemoryMindMapStore store) { ... }
}
```

### Migrate existing tests

Delete `PersonalityDifferentiationTest` after moving its 5 tests:

| Existing test | Target class |
|--------------|-------------|
| `derivedCopingDefaultsDiverge` | `DerivationDivergenceTest` |
| `searchBudgetsDivergeAfterTick` | `AppraisalDivergenceTest` |
| `promptRenderingDiverges` | `PromptSectionDivergenceTest` |
| `curiosityChannelDivergesOnHedonicType` | `AppraisalDivergenceTest` |
| `behavioralAttractorsDivergeAfterConsolidation` | `ConsolidationDivergenceTest` |

### Package structure

```
cognition/src/test/java/io/casehub/neocortex/cognition/eval/divergence/
  DivergenceFixtures.java
  DerivationDivergenceTest.java      (Priority 4 — 7 tests)
  AppraisalDivergenceTest.java       (Priority 2 — 4 tests)
  ConsolidationDivergenceTest.java   (Priority 2 — 4 tests)
  PromptSectionDivergenceTest.java   (Priority 3 — 7 tests)
  TheoryOfMindDivergenceTest.java    (Priority 1 — 4 tests)
  EmotionRegulationDivergenceTest.java (Priority 1 — 4 tests)
  TrajectoryDivergenceTest.java      (Priority 1 — not in original plan, covers multi-tick)
```

## Test Matrix

### Priority 1 — Cross-component wiring (highest risk)

#### TheoryOfMindDivergenceTest

| Test | Capabilities | Assertion |
|------|-------------|-----------|
| `tomPosteriorsDivergeOnSameObservation` | #25 ToM | Bayesian posterior on threat differs: anxious > open |
| `suspiciousnessScalesEvidence` | #25 ToM evidence layer | High-suspiciousness agent's behaviour evidence vector shifts toward avoidant |
| `relationalBiasingShiftsActivations` | #18 relational biasing | Trust overlay produces different CAPS input activations |
| `channelWeightsDivergeByPersonality` | #19 channel weighting | Different CAPS topology channel emphasis |

**Setup:** Two agents, each observing the other's action. Requires MindMap overlay nodes for relationship state.

#### EmotionRegulationDivergenceTest

| Test | Capabilities | Assertion |
|------|-------------|-----------|
| `regulationStrategyDiverges` | #27 regulation prediction, #3 regulation weights | Different regulation strategy selection for same emotion |
| `copingStrategyDiverges` | #28 coping prediction, #2 coping defaults | Different coping strategy for same stressor |
| `situationalDisplacementAccumulates` | #46 displacement, #47 deformation | 5+ ticks: cautious agent's anxiousness increases more than open agent's |
| `modeDetectionDiffers` | #48 mode detection | Same ambiguous input → different situational mode |

**Setup:** Multi-tick with emotion-triggering observations. Needs appraisal state seeding.

### Priority 2 — Tick pipeline

#### AppraisalDivergenceTest

| Test | Capabilities | Assertion |
|------|-------------|-----------|
| `appraisalDimensionsDiverge` | #16 SEC, #8 appraisal weights | Different novelty/conduciveness/controllability scores |
| `emotionMappingDiverges` | #17 emotion mapping | Same SEC profile → different OCC emotion |
| `driveCompositionDiverges` | #21 drives, #22 drive modulation | Different motivational profile after same observation |
| `gutFeelingDiverges` | #24 gut feeling | Different somatic marker valence |
| `searchBudgetsDiverge` | #23 dynamic search budget (migrated) | Different breadth/depth/laterality/convergence |
| `curiosityChannelDiverges` | #5 curiosity channel (migrated) | I-type vs D-type hedonic |

**Setup:** Single tick with observation. Shared harness from `DivergenceFixtures.pairedBuilder()`.

#### ConsolidationDivergenceTest

| Test | Capabilities | Assertion |
|------|-------------|-----------|
| `behavioralAttractorsDiverge` | #39 behavioral synthesis (migrated) | Different attractor categories/strengths |
| `driveClassificationDiverges` | #41 drive derivation | Same experience → different drive labels |
| `experienceGraduationDiverges` | #42 experience consolidation | Different salience thresholds |
| `beliefRevisionRateDiverges` | #43 belief revision | Different confidence update rates |
| `goalPrioritizationDiverges` | #45 goal recognition | Different goal ordering |

**Setup:** Seeded MindMap with cognitive nodes + consolidation cycle.

### Priority 3 — Prompt rendering

#### PromptSectionDivergenceTest

| Test | Capabilities | Assertion |
|------|-------------|-----------|
| `behavioralPromptDiverges` | #31 behavioral prompt section | Different attractor labels in rendered text |
| `drivePromptDiverges` | #32 drive prompt section | Different motivational text |
| `moodPromptDiverges` | #33 mood prompt section | Different PAD values |
| `curiosityPromptDiverges` | #34 curiosity prompt section | Different gap focus |
| `shadowArchetypePromptDiverges` | #37 shadow archetype | Different displacement text |
| `needsPyramidPromptDiverges` | #38 needs pyramid | Different unmet needs |
| `attentionPromptDiverges` | #36 attention prompt | Different salience |
| `promptRenderingDiverges` | #30 composite (migrated) | Full prompt text differs |

**Setup:** Tick + render. Some sections need seeded MindMap state.

### Priority 4 — Derivation

#### DerivationDivergenceTest

| Test | Capabilities | Assertion |
|------|-------------|-----------|
| `moodBaselineDiverges` | #1 mood baseline | Different resting PAD |
| `copingDefaultsDiverge` | #2 coping defaults (migrated) | Different avoidance/confrontation/disengagement/suppression |
| `regulationWeightsDiverge` | #3 regulation weights | Different reappraisal/suppression/redirect strengths |
| `cbrStrategyDiverges` | #6 CBR strategy | Different retrieval mode and learning rate |
| `socialCognitionDiverges` | #7 social cognition | Different trust rate and conflict interpretation |
| `habituationConfigDiverges` | #9 habituation config | Different thresholds |
| `capacityConfigDiverges` | #10 cognitive capacity | Different working memory span |
| `expressionStylesDiverge` | #11 expression styles | Different personality labels |
| `recurringThoughtConfigDiverges` | #14 recurring thought | Different rumination thresholds |

**Setup:** `CognitiveDerivationEngine.derive()` only — no harness needed. Pure derivation assertions.

## Implementation order

1. `DivergenceFixtures` — shared constants and builders
2. `DerivationDivergenceTest` — migrate `derivedCopingDefaultsDiverge` + 8 new tests (easiest, validates fixtures)
3. `AppraisalDivergenceTest` — migrate 2 + 4 new tests
4. `ConsolidationDivergenceTest` — migrate 1 + 4 new tests
5. `PromptSectionDivergenceTest` — migrate 1 + 7 new tests
6. `TheoryOfMindDivergenceTest` — 4 new tests (most complex setup)
7. `EmotionRegulationDivergenceTest` — 4 new tests (multi-tick)
8. Delete `PersonalityDifferentiationTest`

Implementation order differs from priority order: we build fixtures first, then validate them with the simplest tests (derivation), then work up in complexity. But the priority order determines which tests to run first when validating the pipeline.

## Success criteria

- All 48 capabilities have at least one test proving personality divergence
- All tests pass on `mvn test -Dtest="io.casehub.neocortex.cognition.eval.divergence.*"`
- No test uses Mockito (real pipeline components only)
- Each test runs in < 500ms
