## D1: Pipeline placement for situational personality modulation

**Choice:** Split by time scale. A tick participant, `SituationalPersonalityParticipant` registered in `CognitionPhase.TERMINAL`, owns the fast state: mode detection, `current`, and exposure accumulation. A consolidation phase, `SituationalDeformationPhase`, owns the slow state: lasting `homeOffset` and `resilience`.
**Alternatives:**
- Single per-tick participant owning all state (round 1). Rejected in review R1-14: it mixes fast and slow time scales, which is the very reason PEO was rejected. It also makes lasting change depend on tick rate.
- Extend PersonalityEvolutionOrchestrator. It mixes time scales, it is never ticked in production (#1060), and it operates on Jungian terms only.
- Extend BehavioralSynthesisPhase. Consolidation cadence only, so it cannot drive per-turn state.
**Rationale:** State → trait consolidation (TESSERA, Wrzus & Roberts 2017) is a slow, repeated-exposure process. In neocortex, long-term change already runs in consolidation (`DriveAdaptationPhase`, CAPS weight learning). TERMINAL is the only phase that sees this tick's appraisal and post-appraisal mood.
**Trade-offs:** Two components share one state object, through `SituationalStateService` with per-key locking.
**Status:** revised (review round 1: R1-03, R1-14)

## D2: Elastic band state representation

**Choice:** Per facet, all state is relative (review round 2: R2-07): displacement `d` relative to home (`current = clamp01(home + d)`, never stored absolute); `homeOffset` relative to the Eidos baseline (lasting deformation, |offset| ≤ D_max); `resilience` ∈ [r_min, 1] (scales return speed, degraded by load, healed by rest); signed exposure and unsigned load accumulators drained by consolidation. Per mode: intensity and engaged flag.
**Alternatives:**
- Round-1 `current` / `home` / `elasticity` / `elasticity_floor`. Rejected: home drift was unbounded (R1-04); the floor never limited recovery, had no initial value and ratcheted to 0 (R1-05).
- Absolute `home`. Rejected: an offset preserves the lived deformation when the Eidos baseline is corrected.
**Rationale:** Bounded by construction (see the spec's invariants table). Grounded in set-point (Jeronimus 2014), resilience (Bonanno 2004) and sensitisation (Post 1992).
**Status:** revised (review round 1: R1-04, R1-05)

## D3: CDE integration path

**Choice:** A tenant-scoped `EffectiveCognitiveDefaults` stores each slot's inputs: the base `DescriptorView` from Eidos and an optional facet overlay. It re-derives via `merge(explicitLayer, CDE.derive(base.withFacets(overlay)))`, atomically. Situational modulation calls `updateFacets`; baseline wiring calls `updateBase`.
**Alternatives:**
- Round-1 `deriveAndMerge` through the bridge. Rejected: explicit-wins merge against the previously merged result makes every refresh a no-op (R1-01); agentId-keyed, non-atomic registry (R1-08); bridge not wired in production (R1-02); wrong owner (R1-12).
- Post-CDE modification of `CognitiveDefaults`. Rejected: it bypasses the PID-5 → FFM → parameter chains.
**Rationale:** One derivation entry point for baseline and situational inputs, which is #845's "single path, no forks". It also creates the convergence point that PEO evolution needs (#1060).
**Status:** revised (review round 1). Tracked as #1055.

## D4: Situational mode detection design

**Choice:** A per-mode weighted classifier over typed, normalised signals, split into exogenous signals (situation-classifier inputs, per-tick CAPS mediating units, relational schema) and endogenous signals (SEC dimensions, post-appraisal mood). Engagement uses hysteresis and a time-based fade. Modes are in two exclusive groups (affective: stress/flow; social: three frames).
**Invariant:** `w_N < θ_off`, so endogenous evidence alone can neither engage nor hold a mode. This makes the modulation → appraisal → detector loop structurally unable to lock in.
**Alternatives:**
- Round-1 classifier over CAPS output attractors, an undefined `threat` appraisal and raw PAD. Rejected (R1-06): it referenced nonexistent and structurally dead signals, used the consolidation-stale settling cache, and had no normalisation.
- Rule-based classifier. Less tunable.
- Secondary CAPS network. Over-engineered.
**Rationale:** Stress is defined by the transactional/SEC appraisal pattern (Lazarus & Folkman 1984); flow by challenge–skill balance (Csikszentmihalyi). CAPS contributes its if-then signature through mediating units (the "layer above CAPS" #712 asks for). The social subtypes now use the same classifier as the modes; round 1 used rules there (R1-15).
**Status:** revised (review round 1: R1-06, R1-07, R1-15)

## D5: Persistence of runtime personality state

**Choice:** A MindMap node (`cognitiveKind = situational-personality-state`) in the agent's cognitive subgraph, accessed through `SituationalStateService` (in-memory cache, per-key locks, write-through per tick).
**Alternatives:**
- Round-1 neocortex-local SQLite `SituationalPersonalityStore`. Rejected: it contradicts #845 Step 0c and the #712 design comment, which put runtime personality state in MindMap. The round-1 decision misquoted #845 to argue the opposite. It also needed its own SPI, in-memory default and backend, duplicating MindMap.
- Extend DispositionSignalStore. Its API is Jungian terms, and it is baseline-side.
**Rationale:** Documented architecture. Consolidation phases already operate on MindMap. Tenant scoping and the backend CDI ladder come with the MindMap SPI.
**Status:** reversed (review round 1: R1-11)

## D6: Where situational profiles (shadow rules) live

**Choice:** In Eidos: `AgentDisposition.situationalProfiles` holds authored or biographical profiles only (deltas on the AMPD 0–100 scale; facets from the closed `AmpdFacetName` enum). Family defaults (12 × 5) live in a versioned eidos-vocab table. `SituationalProfileResolver` applies them **at read time** for unauthored modes, never materialised into descriptors, so eval retuning needs no re-registration (revised in review round 2: R2-08). neocortex reads only the resolved list.
**Alternatives:**
- Per-family table in neocortex. Rejected (R1-09, R1-10):
  - every Creator degrades identically;
  - there is no path for biographical extraction (#712 Q4);
  - archetype → family resolution belongs with the archetype vocabulary that Eidos owns (`ArchetypeTerm.family()`), and a neocortex copy of the 48 → 12 mapping, or of the family-default table, would drift from it.
  - *Record correction (review round 3, R3-08):* rounds 1–2 also argued that neocortex `cognition` must stay framework-neutral and so must not depend on eidos-vocab's compile-scope `quarkus-arc`. That premise is false: `cognition` already imports `io.quarkus.arc.DefaultBean` (`CognitionDefaultBeans.java:27`), and this spec adds more `@DefaultBean` SPIs. The decision rests on ownership and drift alone. The Spring consequence of `@DefaultBean` is handled by explicit Spring beans (spec Part 0d).
- A generic profile for agents without an archetype. Rejected: it erases differentiation.
**Status:** new (review round 1). Tracked as casehubio/eidos#234; extraction is #1056.

## D7: Explicitly pinned cognitive-default fields under situational modulation

**Choice:** Pinned fields are immune. Explicit YAML is an authoring override and wins over every derivation. Explicit profiles may not pin facets, because facets have one source, Eidos.
**Status:** new (review round 1: R1-01)

## D9: Resting vs effective cognitive defaults

**Choice:** `EffectiveCognitiveDefaults` derives two results per slot. *Resting* is baseline + lasting offset; *effective* adds the transient displacement. The split follows the **kind of parameter read** (restated in review round 3, R3-06):
- **Learning-rule parameters** (rates, ceilings, gates of trust/role update, CAPS learning and initialisation, drive adaptation, strategy learning, graduation, PEO drift ceiling) come from resting. They are enforced by type: `restingFor` returns `LearningProfile`, and every learning rule takes a `LearningProfile`.
- **Behaviour, and the record of experience** produced under the current state (regulation strategy and outcome, appraisal of an action, goal formation), use effective `CognitiveDefaults`.

The round-2 wording ("anything that persists durable state reads resting", enforced by ArchUnit on `ConsolidationPhase`) contradicted the consumer list and could not police tick-time writers.
**Alternative:** Move the consolidation-maintained signals into the endogenous class (R2-02 option b). Rejected: it leaves transient state leaking into every durable learner, not only the detector's inputs.
**Rationale:** TESSERA. Transient states reach durable structure only through the slow, bounded lasting-change channel (homeOffset). This removes consolidation-speed feedback loops by construction.
**Status:** new (review round 2: R2-02)

## D10: Detector evidence form

**Choice:** `E = min(1, X·(1 + β_N·N)·(1 + β_R·R))`. Endogenous (SEC, mood) and relational evidence are gains on exogenous situational evidence. Situation inputs pass through `above_faint`, and mediating units are measured from the agent's resting settlement. The loader validates the faint-hold margin `G·x_faint < θ_off`.
**Alternative:** Rev 2's additive form with `w_N < θ_off`. Rejected: its margin was 0.02, so faint vocabulary held modes indefinitely (R2-01), and a standing relationship engaged social frames alone (R2-06).
**Rationale:** Mood-congruent appraisal amplifies congruent evidence rather than substituting for it (Bower 1981; Forgas 1995). `X = 0 ⇒ E = 0` exactly.
**Status:** new (review round 2: R2-01, R2-06). Supersedes the round-1 settled `w_N < θ_off` invariant.

## D11: One integrator on a fixed wall-clock hour grid

**Choice:** Fast relaxation and slow updates (fatigue, heal, drift, set-point) are integrated by one pure `advance(slot, to)` over wall-clock hours. Slow variables change only at hour closes. Any trigger (tick or consolidation) advances to now, with identical results. `SituationalDeformationPhase` is the trigger and audit point for idle agents.
**Alternative:** Slow updates only inside the ConsolidationPhase (round-1 settled decision). Rejected after R2-05: windowed, non-commuting updates depend on cadence, and the scheduler starves (#1063).
**Rationale:** Cadence invariance is exact. Resilience is current for the fast loop, so the `r↓ → L↑` feedback is captured. The intent of the round-1 decision is kept: slow variables are never updated per tick or at tick-rate-dependent rates.
**Status:** new (review round 2: R2-05). Amends the round-1 settled decision on slow-state placement.

## D8: Anchor of the situational target

**Choice:** `target = clamp01(home + clamp(Σ I·δ, ±S_max))`, anchored to the deformed home, not the immutable baseline. Boundedness comes from |homeOffset| ≤ D_max, not from moving the anchor.
**Alternative:** Anchor to the baseline (reviewer proposal, R1-04). Rejected: under stress, a permanently deformed character would be pulled partly back toward their original personality, so stress would undo deformation.
**Status:** new (review round 1)

## D12: Exposure dose weighted by sustained situational evidence

**Choice:** Load and signed exposure accrue at weight 1 inside exposure windows. A window opens or extends to `now + hold` (10 min) on each tick where a group winner has `X > 0` and `E ≥ θ_off`. Outside windows the relaxation tail accrues at `w_tail = 0.1`. `τ_fatigue` (12) and `τ_drift` (15) are re-tuned so chronic daily stress still saturates.
**Alternatives:**
- Unweighted tail (rev 3). Rejected (R3-05): the 6 h tail was about 80 % of any exposure's load, so a 10-minute and a 2-hour episode left the same trace. Duration collapsed into entry count.
- Two relaxation time constants. Rejected: it changes the observable state trajectory (how fast the character visibly returns) to fix a dose-accounting problem.
- Document duration-insensitivity as intended. Rejected: it contradicts TESSERA's "repeated, sustained" states and §Goal.
**Rationale:** Perseverative cognition prolongs stress activation beyond the stressor (Brosschot, Gerin & Thayer 2006), so the tail counts, but less than the stressor itself. Simulated: 30 days × 10 / 30 / 100 turns give o = 0.045 / 0.071 / 0.098 (monotone; 10 ≤ ½ · 100). Cadence invariance holds exactly.
**Status:** new (review round 3: R3-05)

## D13: One assembly path for CognitionCore

**Choice:** `CognitivePipelineBuilder` is the only constructor path; the `CognitionCore` constructor is package-private. `CognitionCoreFactory` (Quarkus CDI) and `AgencyAutoConfiguration` (Spring, with explicit beans for every Quarkus `@DefaultBean`) both build through it. agency `AgencyBeans` and the agentic-yaml `CognitionStack` migrate to it. `build()` fails fast on a half-wired situational configuration. Wiring tests boot both production graphs.
**Alternative:** Add the missing configure calls to each of the three hand-built roots. Rejected: that repeats the bridgeAll / PEO failure class (built, never assembled) three times over, and the next feature would need the same three edits.
**Status:** new (review round 3: R3-01). Tracked as casehubio/agency#484.

## D14: Per-classifier calibrated noise floor

**Choice:** The `above_floor` threshold `x_floor` is a per-classifier, per-model calibrated parameter: the 95th percentile of confidences on a frozen neutral / incidental-mention calibration corpus. The uncalibrated default is 0.6. The sub-floor hold margin `G·x_sub < θ_off` is validated with the measured `x_floor` and `m_floor`.
**Alternative:** The topology vocabulary band (0.3, rev 3). Rejected (R3-02): only the unused keyword classifier reads those bands; `LlmSituationClassifier` confidences are free-form.
**Status:** new (review round 3: R3-02). Refines D10.
