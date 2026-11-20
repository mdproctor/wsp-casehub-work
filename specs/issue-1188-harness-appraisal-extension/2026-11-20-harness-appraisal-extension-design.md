# Design: CognitionTestHarness lightweight appraisal extension

**Issue:** casehubio/neocortex#1188
**Scale:** M | **Complexity:** Med

## Problem

Several divergence tests fall back to derivation-level assertions because the harness doesn't wire a real appraisal participant. `PipelineTrace.appraisalResult()` is null and `capsInputActivations()` is empty after `tick()`. The existing `withAppraisalResult()` path injects a Mockito mock with pre-canned results — the mock's `tick()` does nothing, so no personality-sensitive computation runs during the pipeline.

## Design

### Component 1: `TestAppraisalStrategy`

**Location:** `cognition/src/test/java/io/casehub/neocortex/cognition/eval/TestAppraisalStrategy.java`

A rule-based `AppraisalStrategy` that produces personality-sensitive SEC dimensions without NLP. For each `appraise(AppraisalContext)` call:

1. **SEC dimensions from personality weights.** Use the `AppraisalWeights` already present in the context (derived by `CognitiveDerivationEngine` from personality facets). Compute:
   - `conduciveness` — base 0.0, shifted by `fearOnsetThreshold` (high threshold → more conducive)
   - `controllability` — base 0.5, shifted by `urgencyWeight` (high urgency → less controllable)
   - `relevance` — always 0.8 (observation was provided)
   - `novelty` — always 0.5 (neutral)

2. **Action tendencies from disposition axes.** Map `DispositionAxes.conflictMode()`:
   - `cooperative` → `APPROACH` at intensity 0.6
   - `avoidant` → `AVOIDANCE` at intensity 0.6
   - otherwise → `ATTENDING` at intensity 0.4

3. **Emotions from SEC dimensions.** Use `AlmaPadTable.project()` to derive PAD values from a base emotion type selected by conduciveness sign (positive → `CONTENTMENT`, negative → `ANXIETY`).

4. **Return.** `AppraisalResult` with SEC dimensions map, action tendencies, emotions, and empty habituation.

The strategy does not parse observation text. It produces different results for different personalities because `AppraisalWeights` and `DispositionAxes` differ.

### Component 2: Harness `Builder.withAppraisalEnabled(true)`

New builder method `withAppraisalEnabled(boolean)`. When `true`, the `build()` method:

1. Constructs `TestAppraisalStrategy`
2. Adds to the pipeline builder chain:
   - `.appraisal(testStrategy, new NoOpSalienceStrategy())`
   - `.situationClassifier(new RuleBasedSituationClassifier(capsEngine.topology()))` (only when `behavioralSynthesisEnabled` is also true)
3. This triggers `CognitivePipelineBuilder.build()` → `core.configureAppraisal()` which creates a real `AppraisalTickParticipant` registered at `CognitionPhase.APPRAISAL`

**Mutual exclusion:** `withAppraisalEnabled(true)` and `withAppraisalResult()` are incompatible (one creates a real participant, the other injects a mock). The builder throws `IllegalStateException` if both are set.

### Component 3: Upgraded divergence tests

Upgrade existing tests in `AppraisalDivergenceTest`:

- **`appraisalResultDivergesAtTickLevel`** (new test): Build harness with `withAppraisalEnabled(true)`, tick both agents with the same observation, assert `PipelineTrace.appraisalResult()` is non-null and SEC dimensions differ.
- **`driveCompositionDiverges`** — upgrade from checking `CognitiveDefaults.curiosity()` to checking `PipelineTrace.driveIntensities()` after tick.
- **`capsInputActivationsDiverge`** (new test): Build with appraisal + behavioral synthesis enabled, tick + consolidate, assert `PipelineTrace.capsInputActivations()` differ.

## Wiring path

```
Builder.build()
  → CognitivePipelineBuilder
      .appraisal(TestAppraisalStrategy, NoOpSalienceStrategy)
      .situationClassifier(RuleBasedSituationClassifier)
  → CognitivePipelineBuilder.build()
      → core.configureAppraisal(strategy, salience, defaults, classifier)
          → creates real AppraisalTickParticipant
          → registers at CognitionPhase.APPRAISAL
  → tick() runs AppraisalTickParticipant.tick()
      → calls appraisalStrategy.appraise()
      → populates results, channelWeights, inputActivations
  → currentTrace() reads from appraisalParticipant
      → PipelineTrace.appraisalResult() non-null
      → PipelineTrace.capsInputActivations() populated (via SituationClassifier)
```

## Testing

- `TestAppraisalStrategy` unit test: verify SEC dimensions differ for OPEN vs CAUTIOUS `AppraisalWeights`
- Upgraded divergence tests: verify PipelineTrace fields are populated and diverge
- Existing tests continue to pass (mock path unchanged, new path is opt-in)

## References

- `CognitionTestHarness.java:445-476` — current pipeline builder chain and mock appraisal path
- `CognitivePipelineBuilder.java:282-284` — `.appraisal()` wiring into `configureAppraisal()`
- `AppraisalTickParticipant.java:124-215` — real `tick()` method that runs strategies
- `PipelineTrace.java:13-26` — trace fields populated from appraisal participant
- `AppraisalDivergenceTest.java` — tests to upgrade
- `NoOpAppraisalStrategy.java`, `NoOpSalienceStrategy.java` — existing no-op implementations
- `RuleBasedSituationClassifier` — CAPS situation classifier already in production
