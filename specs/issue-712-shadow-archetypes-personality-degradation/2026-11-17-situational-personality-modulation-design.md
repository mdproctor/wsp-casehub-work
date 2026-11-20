# Situational Personality Modulation — Design Spec

Ref: casehubio/neocortex#712 — **Part 1: runtime consumption** (see §Scope)
Revision: 4 (adversarial review round 3). Every claim about existing code was re-verified against the neocortex slot tree in IntelliJ, against eidos, and against the agency checkout (production composition roots).

## Goal

Make a character's PID-5 personality *state* respond to the situation it is in (stress, flow, social context). That state feeds through the CDE, so appraisal, regulation, coping, search budget, capacity and expression parameters shift in an archetype-consistent way.

- Brief or mild exposure leaves no trace.
- Prolonged, repeated or extreme exposure leaves a *bounded* trace in the character's resting personality, and that trace reverses slowly (the elastic-band model).

## Scope

#712 asks for more than one deliverable. Its own scope line is "design the runtime consumption path first, then add extraction support". This spec covers the runtime consumption path. #712 stays open as the parent until the deferred items close.

| #712 requirement | Where |
|---|---|
| Stress mode, flow mode, social context (collaborative / competitive / hierarchical) | **This spec** |
| Per-character shadow rules carried in Eidos, with archetype-family defaults (Q1) | **This spec** (Part 0b contract); implemented in casehubio/eidos#234 |
| CAPS-triggered activation (Q2) | **This spec** (§Detection: per-tick CAPS settlement over situation inputs, read through mediating units) |
| Prompt rendering of shadow context (Q3) | v1 is implicit: facet shifts through the CDE, plus the facet-origin cue (§Facet coverage). Explicit `ShadowArchetypePromptSection` → #1059 |
| Biographical extraction of shadow patterns (Q4) | #1056 |
| Temporal drift (archetype salience across life phases) | #1057 |
| Narrative identity frame; classifying behavioural attractors as degraded archetype expression | #1058 (relates to #599) |
| CARMA: SEC evaluation aware of the situational mode | #1059 |

**Prerequisites** (designed here as Part 0, tracked separately):
- #1055 — effective/resting cognitive defaults and production Eidos population.
- casehubio/eidos#234 — `situationalProfiles`, the read-time family-default resolver and the closed facet vocabulary.
- casehubio/agency#484 — production composition roots assemble `CognitionCore` through `CognitivePipelineBuilder` (Part 0d).

**Pre-existing defects found during design** (filed, not fixed here):

| Issue | Defect |
|---|---|
| #1060 | PEO is never ticked |
| #1061 | `BehavioralSynthesisPhase` caches are agentId-keyed |
| #1062 | Per-tick CAPS settlements are duplicated |
| #1063 | `ConsolidationScheduler` loses tenants on restart and starves under load |
| #1064 | Bounded search followed by an in-memory filter loses nodes past the limit |
| #1065 | Relational schema has no hierarchy roles |

## Background

### What exists (verified)

**Personality → CDE.**
- `CognitiveDerivationEngine.derive(DescriptorView, eventRef, observer)` maps disposition axes, the Jungian profile and PID-5 facets to `CognitiveDefaults` (#845).
- `deriveAndMerge(explicit)` merges with explicit-wins semantics (`CognitiveDerivationEngine.java:297-322`).

**The Eidos → CDE path is not wired in production.**
- `EidosCognitiveProfileBridge.bridgeAll` has no production callers.
- No production `cognitive-profiles/*.yaml` exist.
- So every running agent gets `CognitiveDefaults.empty`.
- Re-registration is a no-op, because the previously merged result is passed as `explicit` (`EidosCognitiveProfileBridge.java:84-87`).
- The registry is agentId-keyed with a non-atomic copy-on-write.
- Consumers that enumerate agents (`allAgentIds()` / `allProfiles()`) see only the explicit layer:
  - `CognitiveAttentionAccumulator.addSignals:113` (broadcast, no tenant)
  - `SearchBudgetDescriptionGenerationPhase:108`
  - `CuriosityDescriptionGenerationPhase:114`
  - `DeclarativeRuleRegistry:112,142`
  - `CognitiveLoader:53`

**Appraisal.**
- `AppraisalResult.secDimensions` can carry relevance, novelty, urgency, conduciveness (bipolar) and controllability, among others. In production it is **always empty**: `LlmAppraisalStrategy.parseResponse` parses the dimensions (`:120`) but returns the 4-arg `AppraisalResult` (`:131`), which sets `secDimensions = Map.of()`. Only the keyword `SchererAppraisalStrategy` passes them.
- `AppraisalTickParticipant.results` is written only when appraisal runs (`:178`). Early returns at `:128` (no drives) and `:161` (no observation) leave the previous tick's result readable through `currentResult`.
- The same participant classifies the observation into CAPS input activations every tick it appraises (`:186-195`, after `RelationalBiasing` at `:190`). It uses them only for channel weights.

**Situation classifiers.**
- `RuleBasedSituationClassifier.classify` returns an empty result (`RuleBasedSituationClassifier.java:26-30`). Its keyword path `classifyByKeyword` is unused, preserved for a future NLP stage (#587, closed in favour of the graduated LLM + CBR classifier).
- The only working classifier is `LlmSituationClassifier` (optionally behind `CbrCachingSituationDecorator`). Its confidences are free-form LLM judgements, cut at 0.3 (`LlmSituationClassifier.java:27,138`). They are not drawn from the topology's vocabulary bands.
- `RelationalBiasing.apply` multiplies by `(1 + trust·scale)` with no clamp (`RelationalBiasing.java:43`), so activations can exceed 1.

**Composition roots.**
- Production `CognitionCore`s are hand-built in agency: `AgencyBeans:362-376` (Quarkus) and `AgencyAutoConfiguration:340` (Spring). The agentic-yaml eval harness `CognitionStack:176` is hand-built as well.
- None of them calls `configureAppraisal`, `configureRegulation` or `setDefaultsRegistry`, or wires a CAPS engine.
- `CognitionCoreFactory` / `CognitivePipelineBuilder`, which do wire appraisal, classifier, defaults and regulation, have no callers (agency#484).

**Mood.** PAD is in [-1, 1] and decays toward the global `MoodConfig.baseline()` by elapsed time (τ = 4 h default).

**Consumers of `CognitiveDefaults` that write durable state.** These run in consolidation and read the facets this feature modulates:

| Consumer | What it reads | Lines |
|---|---|---|
| `RelationalSchemaEvolutionPhase` | trust formation rate (Suspiciousness), Callousness damping, `resolveRole` on Manipulativeness/Callousness | `:131-176,184-223` |
| `BehavioralSynthesisPhase` CAPS weight learning | scaled by associative reach (Anxiousness) | `:245,256,504` |

**CAPS.**
- `BehavioralSynthesisPhase` settles on the consolidation cadence, and its cache is agentId-keyed (#1061).
- Per-tick settlements exist only inside regulation and coping (#1062).
- 19 of 51 output nodes are orphaned (#1050/#1051), and the `distrust` attractor is currently not produced (#1049).
- The situation-input vocabulary is banded: strong 0.7–0.9, moderate 0.4–0.6, faint 0.2–0.3 (`caps-topology.yaml:16-17`). Only the unused keyword path reads those bands (see Situation classifiers).

**MindMap.**
- Existing kind-based lookups take a bounded `search(limit 200)` and then filter in memory (#1064).
- `MindMapStore` offers deterministic `resolveNode(name, subgraphId, tenantId)`, `updateNode` and `nodesIn(subgraphId, tenantId)`.

**Consolidation cadence.** `ConsolidationScheduler` runs every 5 min, and only when the system is idle for 1 min. It covers only the in-memory `knownTenants` set, which `consolidateNow` alone populates (#1063).

**Tick structure** (`CognitionCore.tick`, `:286-379`), in order:
1. mood tick #1
2. FOUNDATION
3. SOURCE
4. per-subject
5. drives
6. DERIVED — appraisal, regulation and search budget read cognitive defaults here
7. appraisal → mood bridge, then mood and drive tick #2
8. goals
9. TERMINAL

Custom participants are ordered by `CognitionPhase` (`addParticipant`, `:1239`). The tick ref is `EventRef.tick(tickNumber)` (`:245`).

**PersonalityEvolutionOrchestrator.** Never ticked. It acts on Jungian terms only (#1060).

### Theoretical grounding

- **Whole Trait Theory** (Fleeson & Jayawickreme 2015).
  - A trait is a density distribution of personality states (descriptive part) produced by social-cognitive mechanisms (explanatory part).
  - Here, `current` is the state. The per-tick CAPS settlement over situation features (Mischel & Shoda 1995 if-then signatures) is the mechanism.
  - The detector reads CAPS *mediating* units (the "if" side), never output attractors. Attractors select behaviours, while facet state sets continuous cognitive parameters, so the two are not double-counted.
- **Transactional stress** (Lazarus & Folkman 1984), via Scherer's checks: stress = relevance × goal obstruction × low coping potential.
- **Flow** (Csikszentmihalyi 1990; Nakamura & Csikszentmihalyi 2002): matched high challenge and high skill, with an inverted-U in arousal.
- **Mood-congruent appraisal.** Affective state *amplifies* the appraisal of congruent situational evidence; it does not stand in for it (Bower 1981; Forgas 1995, Affect Infusion Model). This is why endogenous signals act as gains in §Detection.
- **State → trait consolidation** (TESSERA; Wrzus & Roberts 2017). Lasting change accrues from *repeated* states through slow processes. Transient state must not leak into durable learning except through the lasting-change channel (§Part 0a resting vs effective).
- **Set-point and partial return** (Jeronimus, Riese, Sanderman & Ormel 2014). **Magnitude of mean-level change** (Roberts, Walton & Viechtbauer 2006). **Resilience** (Bonanno 2004). **Sensitisation** (Post 1992).
- **Archetype shadow forms** (Pearson 1991; families per Mark & Pearson 2001 / Hartwell & Chen 2012) fix the *direction* of the deltas (qualitative consensus). Every *magnitude* is `provenance: estimated` and tuned by eval.

## Architecture

```
Eidos (class, per character)                               neocortex (runtime instance, per tenant × agent)
────────────────────────────                               ─────────────────────────────────────────────────────────────
AgentDisposition.ampdFacets (AmpdFacetName → 0..100)  ─┐    EffectiveCognitiveDefaults  (Part 0a)
AgentDisposition.situationalProfiles (authored only)   │      slot = base view + personality state  → resting, effective
SituationalProfileResolver (authored ∪ family default) │      pulls: DescriptorViewSource (Eidos) + PersonalityStateSource
   @ read time, versioned                              │    CognitionCore.tick
                                                       ├──►   start: updateBase(view)
                                                       │      DERIVED…: forAgent (effective)
                                                       └──►   TERMINAL: SituationalPersonalityParticipant
                                                                detector → SituationalStateService.advance + pull → publish
                                                            Consolidation: durable learners read restingFor(...)
                                                                SituationalDeformationPhase: advance + persist idle agents
                                                            MindMap subgraph "personality-state" (one node per agent)
```

### Class / instance model

| Concept | Owner / store | Updated by | Cadence |
|---|---|---|---|
| Baseline facets `b` | Eidos `AgentDisposition.ampdFacets` (0–100, closed facet enum) | engram import | character creation |
| Authored / biographical situational profiles | Eidos `AgentDisposition.situationalProfiles` | authoring, extraction (#1056) | character creation |
| Family-default profiles | eidos-vocab `archetype-situational-profiles.yaml` (versioned) | eval tuning | deployment |
| Effective profile set | `SituationalProfileResolver.resolve(descriptor)` | read time | per tick (cached per table version) |
| Detector configuration + dynamics parameters | neocortex `cognition` resource `situational-personality.yaml` | deployment | static |
| Displacement `d` (current − home) | MindMap `personality-state` node | `SituationalStateService` | per tick (pull) and continuous (relaxation) |
| Mode intensities, engaged flags, group winners | same node | tick participant | per tick |
| Lasting offset `o`, resilience `r`, open-hour accumulators | same node | `SituationalStateService.advance` on the hourly grid | hourly grid, whoever advances first |
| Resting / effective cognitive defaults | `EffectiveCognitiveDefaults` (derived, tenant-scoped, evictable) | Part 0a | on base or state change |

`current` is not stored. It is always `c = clamp01(home + d)` with `home = clamp01(b + o)`.

### Part 0a — Resting and effective cognitive defaults (prerequisite, #1055)

**Layers.**
- **Explicit layer.** `CognitiveDefaultsRegistry` is reduced to authored YAML (classpath and `CognitiveProfileWatcher`).
  - It is deployment configuration keyed by agentId.
  - Profiles that pin `personalityProfile.facets` are rejected at load.
  - Pinned fields are immune to modulation (D7).
- **`EffectiveCognitiveDefaults`** (`cognitive-index`, `@ApplicationScoped`) holds `ConcurrentHashMap<TenantAgentKey, Slot>`.
  - A slot holds its **inputs**: `baseView: DescriptorView` and `state: PersonalityState`. `PersonalityState` is per-facet `(o_f, d_f)`, empty when the agent has none.
  - A slot holds **two derived outputs**:
    - `resting = merge(explicit, CDE.derive(base ⊕ o))`
    - `effective = merge(explicit, CDE.derive(base ⊕ (o + d)))`
  - Composition is per facet over the **base view's facets only**: `x ⊕ Δ = clamp01(b_f + Δ_f)` for each `f ∈ base.facets`.
    - Facets absent from the base are never introduced, and state for facets no longer in the base is ignored (see §Facet coverage).
    - Facets with `|value − b_f| ≥ ε_publish` carry `origin = "situational"`.
  - No result is ever merged with a previous result. See §Concurrency protocol for how slots are recomputed and installed.
- **Two read paths**, chosen by the *kind of parameter read*, not by the kind of component (D9, restated in round 3 for R3-06).
  - **The rule.**
    - **Learning-rule parameters** govern *how* durable state updates: rates, ceilings, gates and thresholds of a learning or evolution rule. These are read from **resting**. If transient state changed the learning dynamics themselves, that would be the consolidation-speed loop of R2-02.
    - **Behaviour, and the record of experience produced under the current state,** read **effective**: a strategy chosen while stressed, an emotion appraised while stressed, and the stored outcome of either. That is what actually happened to the character, and recording it faithfully is correct.
    - Transient state therefore reaches *learning dynamics* only through the bounded, slow `homeOffset` channel (TESSERA's state → trait path). It still reaches the *experience being learned from*, as it should.
  - **Typed enforcement at the read site.**
    - `restingFor(tenantId, agentId)` returns a distinct type, `LearningProfile`. Its accessors are exactly the learning-rule parameters: `associativeReach`, `relationalComplexity`, `socialCognition` (`trustFormationRate`, `conflictInterpretation`), `attachment`, the facets learning rules branch on, `copingDefaults` and `dispositionAxes` for CAPS initialisation, and the recurring-thought and graduation configuration.
    - `forAgent(tenantId, agentId)` returns `CognitiveDefaults` (effective).
    - Every learning rule takes a `LearningProfile` in its signature: trust and role update, CAPS weight update and initialisation, drive adaptation, strategy learning, graduation scoring, PEO's drift ceiling. A learning rule therefore cannot be fed effective values without a compile error.
    - Fields that both behaviour and learning use, such as `associativeReach`, appear in both types with their respective semantics: the resting value in `LearningProfile`, the effective value in `CognitiveDefaults`.
    - Step 0a delivers the per-field classification as a reviewed table, with a test that enumerates every `CognitiveDefaults` component and asserts its view.
  - **Classification of the known consumers** (verified per read site):

    | Consumer | Reads | View |
    |---|---|---|
    | `RelationalSchemaEvolutionPhase` | `trustFormationRate`, `conflictInterpretation`, attachment, Callousness / Manipulativeness for trust damping and `resolveRole` | resting (learning rule) |
    | `RelationshipStagePhase` | stage transition gates | resting |
    | `BehavioralSynthesisPhase` | CAPS init (disposition, `copingDefaults`) and the `associativeReach`-scaled weight update | resting |
    | `DriveAdaptationPhase` (`:371`) | `associativeReach` as adaptation-rate scaler | resting |
    | `StrategyLearningOrchestrator` (`:73`) | capacity as learning-rate scaler | resting |
    | `DefaultGraduationScorer` (`:70`) | `relationalComplexity` scaling graduation | resting |
    | `PersonalityEvolutionOrchestrator` | `relationalComplexity` drift ceiling | resting |
    | `DriveProfileDerivationPhase`, `RecurringThoughtPhase` | derivation / recurrence configuration of durable nodes | resting |
    | `CuriosityDescriptionGenerationPhase`, `SearchBudgetDescriptionGenerationPhase` | personality descriptions persisted as durable text | resting |
    | `ActionAppraisalObserver` (`:142`) | appraisal weights for appraising an action's outcome as it happens | **effective** (experience) — reclassified from rev 3 |
    | `EmotionRegulationTickParticipant` | regulation defaults for strategy choice; writes regulation-result nodes; learns from `loadHistory` / `loadOutcomes` | **effective**. The strategy and its recorded outcome are experience. The predictor that learns from history has no personality-derived learning-rate parameter today; if one is added, it goes through `LearningProfile`. |
    | `CognitiveGoalOrchestrator`, goal formation | goal appraisal at tick time; writes goal nodes | effective (behaviour plus record) |
    | `AppraisalTickParticipant`, `SearchBudgetTickParticipant`, `ProactiveRegulationParticipant`, `DefaultCognitiveResourcePool`, `MindMapDriveSource`, `CognitiveAttentionAccumulator`, `CognitionCore` modulation context, prompt sections | per-tick behaviour | effective |
- **Population without a tick** (R1-02). Two SPIs in `cognitive-index`:
  - `DescriptorViewSource`:
    - `Optional<DescriptorView> find(String tenantId, String agentId)`
    - `List<String> agentIds(String tenantId)`

    Implemented in `cognitive-eidos` by `EidosDescriptorViewSource` over `AgentRegistry.findById(agentId, tenancyId)` and `find(AgentQuery.all(tenancyId))`. It has an empty `@DefaultBean` when no `AgentRegistry` exists.
  - `PersonalityStateSource`: `Optional<PersonalityStateSnapshot> find(String tenantId, String agentId, Instant now)`. Implemented by `SituationalStateService` (`cognition`); empty `@DefaultBean`.
    - It returns an immutable snapshot **advanced to `now`** (R3-04), produced by the pure integrator on a *copy*.
    - It takes **no lock**. It reads the slot's volatile last-published snapshot when the slot is cached, or the persisted node otherwise, and never writes. The next owner of the slot lock re-derives the same values from the same history, because the integrator is deterministic.
  - On a slot miss (any read path), the slot is created from both sources. Non-tick consumers, consumers right after a restart, and consumers after eviction therefore all see the agent's real resting and effective defaults, including lasting deformation, as of now.
- **Authoritative base source** (R3-04).
  - The registry (`DescriptorViewSource`) is authoritative whenever it knows the agent.
  - The tick's `context.descriptor()` is an **invalidation hint**: if its view differs from the stored base, the base is re-pulled from the registry. If the two still differ, the registry wins, and a `DescriptorMismatch` warning is journaled once per distinct pair.
  - The tick descriptor is used as the base only when the registry has no entry for the agent (an unregistered harness agent).
  - Registry updates reach live slots through an `AgentRegistryEvents` listener (eidos-api; framework-neutral, so it works on Quarkus and Spring alike). It invalidates the slot's base on `registered` / `updated` / `removed`, so agents that never tick still refresh.
  - Fallback where no event source is wired: bases older than 1 h are re-pulled on read.
- **Push paths.**
  - `updateBase(tenantId, view, ref)` — tick start (subject to the source rule above).
  - `updateState(tenantId, agentId, snapshot, ref)` — `SituationalStateService` on publish: tick catch-up, TERMINAL pull, or a consolidation hour close (§Dynamics).
- **Concurrency protocol** (R3-03).
  - **No calls out of `compute`.** Every miss or refresh:
    1. loads the inputs **outside** the map (registry I/O, MindMap reads, the integrator);
    2. derives `resting` / `effective` outside the map (pure CDE);
    3. installs with `compute`, whose lambda only compares the slot's `(baseVersion, stateVersion)` with the candidate's and keeps the newer.

    The bin lock is therefore held only for a version comparison: no blocking I/O, no recursion, no stall on colliding keys.
  - **Version stamps.**
    - `baseVersion` is a per-key invalidation epoch (an `AtomicLong` in a side map, no I/O). The registry listener and the tick-hint invalidation increment it. A load captures it *before* reading, so a load that raced an invalidation installs with the older epoch and loses. A descriptor hash would not work here: hashes have no order.
    - `stateVersion` is the `SituationalStateService` slot's monotonically increasing publish counter, persisted on the node.
    - A concurrent miss and publish can only install in version order; a stale miss result is discarded.
  - **Lock order.** `SituationalStateService` per-key lock → `EffectiveCognitiveDefaults` map, never the reverse.
    - `EffectiveCognitiveDefaults` never calls into `SituationalStateService` while holding anything.
    - `PersonalityStateSource.find` takes no lock, so the inversion cannot occur even through the pull path.
    - Publishing from inside the slot lock is allowed, because it only takes the (short, pure) bin lock afterwards.
- **Tenant-scoped enumeration.** `agentIds(tenantId)` = `DescriptorViewSource.agentIds(tenantId)` ∪ explicit-layer ids. It replaces `allAgentIds()` / `allProfiles()` for consumers that enumerate agents:
  - `CognitiveAttentionAccumulator`: broadcast signals are grouped by `AttentionSignal.tenantId()`, and its `perPrincipal` map is re-keyed by `(tenantId, principal)`.
  - `SearchBudgetDescriptionGenerationPhase`, `CuriosityDescriptionGenerationPhase`: these have a tenant from `run(tenantId, …)`.
- **Explicit-layer consumers by design.** `DeclarativeRuleRegistry` (authored trait and derived-edge rules), `CognitiveLoader` (vocabulary registration) and `CognitiveProfileWatcher` read authored content that Eidos derivation never produces, so they keep reading the explicit layer.
- **Lifecycle.**
  - An explicit-layer reload (watcher) recomputes every slot.
  - Slots idle for more than 24 h are evicted. That is safe because the next read re-pulls from the sources.
- **Removed.** `deriveAndMerge` is replaced by a pure `merge(explicit, derived)`. `bridgeAll` and `registerFromDescriptor` are deleted.
- **Bridge changes.**
  - `EidosCognitiveProfileBridge.toDescriptorView` becomes **public**. It maps facets through the closed `AmpdFacetName` enum (Part 0b), not `titleCase`.
  - New module edge: `cognition → cognitive-eidos`. It is acyclic.
- **Consumer migration.** About 108 production references move to `forAgent` (→ `CognitiveDefaults`) or `restingFor` (→ `LearningProfile`) per the rule above. This is mechanical and breaking by design; the type split makes every misclassification a compile error.
- **Observability.** Every recomputation passes the `EventRef` and the `DerivationObserver` (`StageComputed` / `DataFlowed`; production wiring is #981).

### Part 0c — Appraisal fixes the detector depends on (in this spec)

- `LlmAppraisalStrategy.parseResponse` returns `new AppraisalResult(emotions, tendencies, habituation, narrative, dims)` (the 5-arg form). This fixes production SEC dimensions for every consumer, not only this one.
- `AppraisalTickParticipant` stamps each stored result with the tick `EventRef`. It adds `currentResultAt(agentId, tenantId, EventRef)`, which returns empty when the result is not from that tick. `currentResult` keeps its existing semantics for prompt sections.
- `AppraisalTickParticipant.currentInputActivations(agentId, tenantId)` is a new accessor, cleared at the start of each tick.
- `CognitionTickContext` gains `EventRef tickEventRef`.
- `RelationalBiasing.apply` clamps biased confidences to [0, 1] (R3-02). With trust = −1 a `betrayal` 0.9 currently becomes 1.125. The clamp restores the [0,1] range every consumer of situation activations assumes, including channel weighting.
- The `LlmAppraisalStrategy` system prompt (`:44-47`) currently says traits "describe stable patterns". A clause is added: *"Traits marked `situational` are how the character is right now under present circumstances, shifted from their usual level (shown in brackets); let them colour the current appraisal."* Rendering becomes `Anxiousness (0.77): situational [usual 0.62]`.

### Part 0b — Situational profiles in Eidos (prerequisite, casehubio/eidos#234)

- **Closed facet vocabulary.**
  - New eidos-api enum `AmpdFacetName`: the 13 PID-5 facets, with canonical display names identical to the neocortex `PersonalityFacet` constants.
  - `AmpdFacet.facet` and `AmpdFacetShift.facet` become `AmpdFacetName`.
  - YAML accepts case, space and underscore variants and canonicalises them. An unknown name fails registration.
  - This removes the bridge `titleCase` path, which maps `emotional_lability` → `Emotional_lability` and so silently misses the CDE facet.
- **Contract.**
  - `record AmpdFacetShift(AmpdFacetName facet, int delta)`, with |delta| ≤ 50 on the 0–100 scale.
  - `record SituationalProfile(String mode, List<AmpdFacetShift> shifts, String provenance)`.
  - `AgentDisposition.situationalProfiles` holds **authored or biographical profiles only**.
  - Modes form a closed vocabulary: `stress`, `flow`, `social_collaborative`, `social_competitive`, `social_hierarchical`.
- **Read-time family defaults** (R2-08).
  - eidos-api `SituationalProfileResolver.resolve(AgentDescriptor) → List<SituationalProfile>`, implemented in eidos-runtime with an empty `@DefaultBean`.
  - It resolves `archetype` → family (`ArchetypeTerm.valueOf(..).family()`, else `ArchetypeFamily.valueOf`, else none). It returns the authored profiles plus the family default for every unauthored mode, with provenance `family-default:<FAMILY>@<tableVersion>`.
  - `archetype-situational-profiles.yaml` carries a `version`.
  - Family defaults are never materialised into descriptors, so an eval retune takes effect on deploy with no re-registration. The version in the provenance makes the active table observable in the journal.
  - Override granularity is the whole mode.
  - A null or unknown archetype with no authored profiles gives an empty list, and so no modulation (no generic profile). There is no archetype blending; per-character profiles (#1056) cover that.
- **Single conversion point.** `EidosCognitiveProfileBridge.toSituationalDeltas(List<SituationalProfile>) → Map<SituationalMode, Map<String, Double>>` (in `cognitive-eidos`) divides by 100 and maps `AmpdFacetName` → `PersonalityFacet` name. It is the only place profile deltas are converted. The participant calls `SituationalProfileResolver` and then this function.

```yaml
# eidos-vocab: archetype-situational-profiles.yaml (shape; deltas on the AMPD 0–100 scale)
version: 1
CREATOR:
  stress:
    description: "Obsessive self-validation, perfectionism paralysis"   # documentation only (narrative frame: #1058)
    provenance: estimated          # direction: consensus (Pearson 1991 Creator shadow); magnitude: estimated
    sources: [pearson-1991]
    shifts: {Anxiousness: +15, Emotional Lability: +20, Grandiosity: +10, Withdrawal: +10}
  flow:
    description: "Expansive creative expression, boundary dissolution"
    provenance: estimated
    sources: [csikszentmihalyi-1990, pearson-1991]
    shifts: {Risk Taking: +15, Impulsivity: +10, Grandiosity: +5, Anxiousness: -10}
  social_competitive:
    provenance: estimated
    sources: [pearson-1991]
    shifts: {Hostility: +10, Grandiosity: +15, Suspiciousness: +10}
  social_collaborative: {...}
  social_hierarchical:  {...}
CAREGIVER:
  stress:
    description: "Martyrdom, boundary collapse, resentful compliance"
    provenance: estimated
    sources: [pearson-1991]       # Caregiver shadow: martyr / enabler
    shifts: {Emotional Lability: +20, Anxiousness: +15, Callousness: -10, Withdrawal: -15}
  # ... 12 families × 5 modes = 60 profiles
```

### Part 0d — One assembly path for `CognitionCore` (prerequisite, casehubio/agency#484)

Every piece above is inert unless something assembles it. Today nothing does: every production `CognitionCore` is hand-built without appraisal, regulation, defaults or CAPS (§Background, Composition roots). That is the same failure class as `bridgeAll` and #1060: built, tested in isolation, never assembled. The fix is consolidation, not three more hand-wirings:

- **`CognitivePipelineBuilder` becomes the only way to construct a `CognitionCore`.** The `CognitionCore` constructor becomes package-private (same package as the builder). Breaking by design: each hand-built root fails to compile until it moves to the builder.
- **Builder extensions:**
  - `effectiveDefaults(EffectiveCognitiveDefaults)` — replaces `defaultsRegistry(...)` / `setDefaultsRegistry`.
  - `capsEngine(CapsEngine)` — passed to regulation, coping and the situational participant.
  - `behavioralSynthesis(...)` — trace source.
  - `situationalPersonality(SituationalStateService, SituationalProfileResolver, SituationalPersonalityConfig)` — registers the TERMINAL participant and the tick-start catch-up.
  - `participant(CognitionPhase, CognitionTickParticipant)` and `sectionCustomizer(...)` — for application-specific additions, applied *after* the platform participants of the same phase.
  - `build()` always calls `configureRegulation` with the CAPS engine when present. It fails fast with a descriptive error when `situationalPersonality` is configured but its required collaborators are absent: appraisal with a `SituationClassifier`, and `effectiveDefaults`. A half-wired feature must not start silently.
- **Quarkus:**
  - `CognitionCoreFactory` resolves every collaborator above via CDI and is the only Quarkus entry point.
  - `AgencyBeans` obtains its core from `CognitionCoreFactory.create(config)`. `CognitiveGoalOrchestrator` becomes a producer bean so the factory resolves it. The `EmergentGoalPromptSection` customizer moves to `sectionCustomizer`.
  - `SituationalDeformationPhase` is a CDI `ConsolidationPhase`, so the scheduler discovers it.
- **Spring:**
  - `AgencyAutoConfiguration` builds through `CognitivePipelineBuilder`.
  - `@DefaultBean` does nothing on Spring, so it declares explicit `@ConditionalOnMissingBean` beans for every SPI that is a `@DefaultBean` on Quarkus: `EffectiveCognitiveDefaults`, `DescriptorViewSource`, `PersonalityStateSource`, `SituationalProfileResolver`, `SituationalStateService`, the participant, the deformation phase, and the `AgentRegistryEvents` listener.
- **Eval harness.** agentic-yaml `CognitionStack` builds through the builder. Its classifier is the real `LlmSituationClassifier` for eval runs, and the scripted classifier (§Testing) for deterministic tests.
- **Wiring tests** (agency):
  - Boot the Quarkus bean graph and, separately, the Spring context. Assert that `SituationalPersonalityParticipant` is registered in TERMINAL, that appraisal has a `SituationClassifier`, that `effectiveDefaults` and the CAPS engine are set, and that `SituationalDeformationPhase` is among the scheduler's phases.
  - `CognitionCore` gains a read-only `participants(CognitionPhase)` accessor for this.

### Situational modes

```java
enum SituationalMode {
    STRESS(Group.AFFECTIVE), FLOW(Group.AFFECTIVE),
    SOCIAL_COLLABORATIVE(Group.SOCIAL), SOCIAL_COMPETITIVE(Group.SOCIAL), SOCIAL_HIERARCHICAL(Group.SOCIAL);
    enum Group { AFFECTIVE, SOCIAL }
}
```

- Modes are mutually exclusive within a group and orthogonal between groups.
- **Winner with switch margin.**
  - The group winner is the engaged mode with the highest intensity.
  - A challenger replaces the incumbent only when `I_challenger ≥ I_incumbent + Δ_switch` (default 0.10), or when the incumbent disengages.
  - Losing modes keep decaying and contribute no delta.
- At most two modes contribute (one per group). Their deltas add, then the per-facet clamp applies.

### Detection

#### Evidence model: endogenous and relational signals are gains on exogenous evidence

For each mode m, every tick:

```
X_m = Σ_s w_s · x_s                 exogenous situational evidence ∈ [0,1]   (signals below; Σ w_s = 1)
N_m = Σ_s v_s · n_s                 endogenous evidence ∈ [0,1]  (SEC, mood; absent/stale → 0)
R_m = relational context ∈ [0,1]    (per-mode transform of the interlocutor's schema; none → 0)

E_m = min(1, X_m · (1 + β_N · N_m) · (1 + β_R · R_m))
```

Mood-congruent appraisal and standing relationships amplify situational evidence. Neither can substitute for it.

- **Structural guarantee: `X_m = 0 ⇒ E_m = 0`, exactly.** No endogenous saturation or relationship can engage or hold a mode without situational evidence.
- **Quantified margin against sub-floor evidence** (R2-01, re-grounded for R3-02). Situation inputs pass through `above_floor(x_floor) = max(0, a − x_floor)/(1 − x_floor)`.
  - `x_floor` is a **per-classifier calibrated parameter**, not a topology constant. Rev 3's 0.3 was grounded in the topology vocabulary bands, which only the unused keyword path reads.
  - It is the noise floor of the classifier actually wired: the confidence below which an activation is indistinguishable from neutral or incidental mention.
  - Inputs at or below the floor contribute nothing.
  - **Calibration (pre-registered).**
    - Run the wired classifier over a fixed calibration corpus: 300 neutral utterances, plus 300 utterances that mention a threat / loss / mastery / social term incidentally (reported, hypothetical, negated, or about a third party). The corpus is authored with the eval set and frozen before tuning.
    - The corpus is split into a calibration half and a held-out half (eval). `x_floor` is the 95th percentile of all node confidences produced on the calibration half, recorded per classifier and model version in `situational-personality.yaml` (`classifiers.<impl>.x_floor`, with `calibrated_on` and the corpus hash).
    - If the wired classifier has no calibration entry, the loader uses a conservative uncalibrated default of **0.6**, which zeroes the reviewer's 0.5 "incidental mention". It journals `SituationalCalibrationMissing` once.
    - The default makes engagement harder, never easier: a strong 0.9 activation still yields 0.75.
  - Faint-vocabulary arithmetic from `classifyByKeyword` is irrelevant while that path is unused. If it is revived, it gets its own calibration entry, and its reinforcement term (two faint words → 0.35) is measured, not assumed.
  - CAPS mediating signals use `rise_from_rest` / `drop_from_rest` (below), measured against the agent's resting settlement, so they respond only to change.
  - **Mediating units.** CAPS settles on the raw (un-floored) activations, so sub-floor inputs can still move mediating units a little. Let `m_floor` be the largest from-rest mediating response when every situation input is at its classifier's `x_floor`.
  - Then `x_sub,m = Σ_{mediating s} w_s · m_floor` is the largest X that sub-floor evidence can produce. `G_m` is the largest possible gain: `(1+β_N)(1+β_R)` for modes with a relational source, `(1+β_N)` otherwise.
  - **Validated margin.** The loader enforces `G_m · x_sub,m < θ_off,m` for every mode. A conversation whose situational evidence never exceeds the calibrated noise floor therefore cannot hold a mode, however saturated N is.
  - `m_floor` is a config bound (default 0.3). An integration test settles the real topology with every input at `x_floor` and checks it (§Testing). The guarantee thus rests on two measured quantities, `x_floor` and `m_floor`, and a load-time inequality.
  - At defaults the hold margin `θ_off − G·x_sub` is **0.24** for stress, **0.195** for flow and collaborative/competitive, and **0.285** for hierarchical. Under rev 2's additive form it was 0.02.
  - **Residual risk, stated.** Evidence *above* the floor, such as a genuinely mentioned but mild threat, does hold a mode while N is saturated, for as long as it recurs. That is intended: real situational evidence, amplified by a congruent mood. The guarantee is "no lock-in on noise", not "no persistence under real but mild stressors".
- **What N can and cannot do.** With N = 1 and R = 0, engagement needs `X ≥ θ_on/(1+β_N)` = 0.33 and holding needs `X ≥ 0.28`, compared with 0.50 / 0.42 at N = 0. So mood and appraisal lower the bar for real situational evidence by at most a third.
- **Timing.** Modulation computed in TERMINAL at tick t is read in DERIVED at tick t+1. Slow learners read only resting defaults (Part 0a), so the transient state never enters consolidation-speed loops (R2-02).

#### Signals, sources and loop classes

| Signal | Accessor | Raw range | Transform | Freshness | Loop class |
|---|---|---|---|---|---|
| Situation input activations | `AppraisalTickParticipant.currentInputActivations` | [0,1] per node | `above_floor` | this tick; no observation → all 0 | **exogenous**: the classifier sees text plus `RelationalBiasing(targetSchema)`. The schema is maintained from resting defaults, so it changes only via `homeOffset` (slow, bounded). |
| CAPS mediating activations | per-tick `CapsEngine.settle(loadState, situationInputs)` by the participant (journaled as `CapsSettled`). Rest reference = `settle(loadState, {})`, cached per (tenant, agent, CAPS `generation`) | unipolar [0,1]; bipolar [-1,1] | `rise_from_rest` = max(0, a − a_rest)/(a_max − a_rest); `drop_from_rest` = max(0, a_rest − a)/(a_rest − a_min) | this tick; no inputs → 0 | **exogenous**: CAPS weights are initialised and learned from resting defaults only |
| Relational schema of the interlocutor | shared `RelationalSchemaResolver` (extracted from the duplicates in appraisal and regulation) for `targetAgentId` / `interactingAgents` | trust [-1,1] | `positive_part` (collaborative), `negative_part` (competitive) | consolidation-maintained | **gain R**: slow, resting-coupled |
| SEC dimensions | `AppraisalTickParticipant.currentResultAt(…, tickEventRef).secDimensions()` (Part 0c) | [0,1]; conduciveness [-1,1] | per signal | this tick only (stamp check) | **gain N**: endogenous |
| Post-appraisal mood | `MoodOrchestrator.currentMood`, read in TERMINAL | [-1,1] | `negative_part` / `positive_part` / `band(c,w)` | this tick | **gain N**: endogenous |

**Missing sources** (R2-06).
- Renormalisation across the remaining weights happens **only for structurally unavailable sources**:
  - no `SituationClassifier` bean (then X has no inputs, so the feature is inert);
  - no `CapsEngine` bean;
  - no CAPS state for this agent yet.
- A source that is available but **silent this tick** (no observation, failed classification, stale appraisal) scores **0**. "No evidence this tick" stays distinct from "fewer instruments".

**Social gate.** Social modes require an interlocutor (`targetAgentId` or `interactingAgents`). Without one, X = 0 for all social modes.

**Relational role.** `role` is not used. The platform's role vocabulary (`stranger / ally / mentor / tool / prey / threat / obstacle`, `RelationalSchemaEvolutionPhase.resolveRole`) carries no hierarchy (#1065). SOCIAL_HIERARCHICAL therefore rests on situation inputs and mediating units only.

#### Transforms

| Transform | Definition |
|---|---|
| `identity` | x |
| `positive_part` / `negative_part` | max(0, x) / max(0, −x) |
| `complement` | 1 − x |
| `band(c, w)` | max(0, 1 − \|x − c\|/w) — inverted U |
| `challenge_skill(a, b)` | √(a·b)·(1 − \|a − b\|) |
| `above_floor` | max(0, x − x_floor)/(1 − x_floor), with `x_floor` taken from the wired classifier's calibration entry (default 0.6 when uncalibrated) |
| `rise_from_rest` / `drop_from_rest` | as in the table above |

A signal may be a `product` of transformed factors.

#### Engagement and intensity

```
I_m ← I_m · exp(−Δt / τ_fade,m)                 (Δt from the injected Clock)
if !engaged_m and E_m ≥ θ_on,m:  engaged_m ← true
if engaged_m:  I_m ← max(I_m, E_m)
if engaged_m and I_m < θ_off,m:  engaged_m ← false; I_m ← 0
```

#### Configuration (`cognition/src/main/resources/situational-personality.yaml`)

```yaml
gains: {beta_N: 0.5, beta_R: 0.25, m_floor: 0.3, switch_margin: 0.10}
classifiers:                               # per-classifier noise floor (R3-02); pre-registered calibration
  LlmSituationClassifier: {x_floor: <calibrated>, model: <model id>, calibrated_on: <date>, corpus: <sha256>}
  uncalibrated_default: {x_floor: 0.6}
modes:
  stress:                                  # Lazarus & Folkman 1984; Scherer SEC
    group: affective
    exogenous:
      - {source: situation_input, nodes: [physical_threat, social_threat, psychological_threat, unpredictable_danger], aggregate: max, transform: above_floor, weight: 0.35}
      - {source: situation_input, nodes: [failure, scarcity, powerlessness, rejection, betrayal, abandonment], aggregate: max, transform: above_floor, weight: 0.25}
      - {source: caps_mediating, nodes: [FFFS_activation, BIS_activation, threat_sensitivity], aggregate: mean, transform: rise_from_rest, weight: 0.40}
    endogenous:
      - {source: sec, product: [{dim: relevance}, {dim: conduciveness, transform: negative_part}, {dim: controllability, transform: complement}], weight: 0.60}
      - {source: mood, axis: pleasure, transform: negative_part, weight: 0.25}
      - {source: mood, axis: arousal,  transform: positive_part, weight: 0.15}
    relational: none
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT2H
  flow:                                    # Csikszentmihalyi 1990; Nakamura & Csikszentmihalyi 2002
    group: affective
    exogenous:
      - {source: situation_input, nodes: [mastery, competence_recognition, choice_available, agency_granted], aggregate: max, transform: above_floor, weight: 0.5}
      - {source: caps_mediating, nodes: [BAS_activation], transform: rise_from_rest, weight: 0.3}
      - {source: caps_mediating, nodes: [self_efficacy], transform: rise_from_rest, weight: 0.2}
    endogenous:
      - {source: sec, product: [{dim: relevance}, {dim: conduciveness, transform: positive_part}, {challenge_skill: [urgency, controllability]}], weight: 0.60}
      - {source: mood, axis: arousal,  transform: "band(0.25, 0.6)", weight: 0.25}
      - {source: mood, axis: pleasure, transform: positive_part, weight: 0.15}
    relational: none
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT45M
  social_collaborative:
    group: social
    exogenous:
      - {source: situation_input, nodes: [acceptance, sharing, perceived_responsiveness], aggregate: max, transform: above_floor, weight: 0.6}
      - {source: caps_mediating, nodes: [other_safety, other_reliability], aggregate: mean, transform: rise_from_rest, weight: 0.4}
    endogenous:
      - {source: mood, axis: pleasure, transform: positive_part, weight: 1.0}
    relational: {field: trust, transform: positive_part}
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT30M
  social_competitive:
    group: social
    exogenous:
      - {source: situation_input, nodes: [competition, exclusion, dominance], aggregate: max, transform: above_floor, weight: 0.6}
      - {source: caps_mediating, nodes: [other_safety, other_reliability], aggregate: mean, transform: drop_from_rest, weight: 0.4}
    endogenous:
      - {source: sec, product: [{dim: relevance}, {dim: external-standards, transform: complement}], weight: 0.6}
      - {source: mood, axis: arousal, transform: positive_part, weight: 0.4}
    relational: {field: trust, transform: negative_part}
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT30M
  social_hierarchical:
    group: social
    exogenous:
      - {source: situation_input, nodes: [power_asymmetry, controlled, submission], aggregate: max, transform: above_floor, weight: 0.7}
      - {source: caps_mediating, nodes: [self_efficacy], transform: drop_from_rest, weight: 0.3}
    endogenous:
      - {source: mood, axis: dominance, transform: negative_part, weight: 1.0}
    relational: none                       # no hierarchy in RelationalSchema (#1065)
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT30M
```

**Load-time validation** fails startup on any of the following:
- unknown mode;
- weights within `exogenous` or within `endogenous` not summing to 1;
- `θ_off ≥ θ_on`;
- `β_N, β_R < 0`;
- **`G_m · x_sub,m ≥ θ_off,m`** for any mode (the sub-floor hold margin);
- a wired classifier with `x_floor` outside [0.3, 0.9];
- a situation node not in the `CapsTopology` input set;
- a mediating node not in the mediating set, or with no incoming connection;
- a situation-input signal whose transform is not `above_floor`;
- a mediating signal whose transform is not `*_from_rest`;
- an unknown SEC dimension or mood axis.

### Dynamics

All facet quantities are on the 0–1 scale. Per agent, state exists only for `f ∈ F_agent` (§Facet coverage).

| Symbol | Meaning | Stored |
|---|---|---|
| `b_f` | baseline (from the descriptor, every tick) | no |
| `o_f` | lasting offset; `home_f = clamp01(b_f + o_f)` | yes |
| `d_f` | displacement from home; `c_f = clamp01(home_f + d_f)` | yes |
| `r_f` | resilience ∈ [r_min, 1] | yes |
| `A_f`, `L_f` | signed / unsigned exposure in the **open hour** (facet·h) | yes |
| `hourCursor` | start of the open hour (wall-clock aligned) | yes |
| `exposedUntil` | end of the current evidence-sustained exposure window | yes |

**Everything is relative** (R2-07). `o` is relative to the baseline and `d` is relative to home. A corrected or re-extracted baseline moves the whole state with it: no spurious displacement, no fatigue and no reverse drift. Facets present in the baseline only after initialisation start at `o = d = 0`, `r = 1`. Facets that leave the baseline are dropped from state and journaled.

#### One integrator: `SituationalStateService.advance(slot, to)`

The fast and slow dynamics are integrated by **one pure function on a fixed grid of wall-clock hours**. Its result depends only on the event history (pull events with their times, and the baseline). It does **not** depend on tick rate, consolidation cadence or who triggers it (R2-05).

```
advance(slot, to):
  while slot.cursorTime < to:
    seg_end = min(to, slot.hourCursor + 1h, exposedUntil if cursorTime < exposedUntil)
    w = 1 if cursorTime < exposedUntil else w_tail          exposure weight (R3-05)
    relax over [cursorTime, seg_end]          for each facet, with k_f = r_f / τ_return:
        A_f += w·d_f·(1 − e^{−k_f·Δ})/k_f ;  L_f += w·|d_f|·(1 − e^{−k_f·Δ})/k_f ;  d_f ← d_f·e^{−k_f·Δ}
    if seg_end == hourCursor + 1h:            close the hour (slow step, exact order, 1 h resolution):
        r_f ← max(r_min, r_f − L_f/τ_fatigue)
        r_f ← 1 − (1 − r_f)·e^{−1h/τ_heal}
        step = (1 − r_f)·A_f/τ_drift ;  if sign(step)==sign(o_f): step ← step·(1 − |o_f|/D_max)
        o_f ← clamp(o_f + step, ±D_max) ;  o_f ← o_f·e^{−1h/τ_setpoint}
        A_f ← 0 ; L_f ← 0 ; hourCursor += 1h
  (fast-forward: once all |d_f| < 1e-6 the remaining closed hours have A = L = 0, so r and o
   advance in closed form: r ← 1 − (1−r)e^{−n/τ_heal}, o ← o·e^{−n/τ_setpoint})
```

- Because `d` is a displacement, a moving home carries the current state with it.
- `r` is always current to the last closed hour. The fast relaxation therefore sees `r`'s real history, including the `r ↓ → slower return → L ↑` feedback.
- **Exposure is weighted by sustained evidence** (R3-05).
  - Load and signed exposure accrue at full weight only inside an **exposure window**. A window opens or extends to `now + hold` (10 min) on every tick where a group winner has exogenous evidence `X > 0` and `E ≥ θ_off`.
  - Outside windows, the post-exposure relaxation tail accrues at `w_tail = 0.1`. That is perseverative cognition: the tail contributes, but less than the stressor itself (Brosschot, Gerin & Thayer 2006).
  - The displacement `d` relaxes identically in both cases. Only the *dose* that drives fatigue and drift is weighted.
  - Why this matters: in rev 3 the 6 h tail was about 80 % of the load of any exposure, so ten minutes and two hours left almost the same trace. Duration was collapsed into entry count. Now the dose is the time the character actually spent in the situation, plus a minor tail.
  - The constants were re-tuned to keep the chronic case: `τ_fatigue` 40 → 12 facet·h, `τ_drift` 50 → 15 h.

#### Tick-start catch-up (R3-04)

At tick start, before mood tick #1 and alongside `updateBase`, `CognitionCore` calls `SituationalStateService.catchUp(tenant, agent, now)`. This is `advance(slot, now)`, then a publish if the composed `o + d` moved by ≥ ε or any hour closed.

The first DERIVED appraisal after an idle gap therefore runs on the relaxed state, not the pre-idle one. The TERMINAL `advance` in step 1 below then covers only the few milliseconds of the tick itself.

#### Tick (`SituationalPersonalityParticipant`, TERMINAL)

1. `advance(slot, now)`.
2. Detect (§Detection) → intensities and group winners.
3. `Δ_f = clamp(Σ_{winners} I_m·δ_{m,f}, ±S_max)`.
4. If a winner exists, pull (event-based): `d_f ← d_f + α_pull·(Δ_f − d_f)`, then clamp so `home_f + d_f ∈ [0,1]`. If that winner has `X > 0` and `E ≥ θ_off`, set `exposedUntil ← max(exposedUntil, now + hold)`.
5. **Publish**:
   - **when:** `max_f |(o_f + d_f) − published_f| ≥ ε_publish`, a winner changed, any hour closed in step 1, or `published` is unknown (slot freshly loaded);
   - **how:** `EffectiveCognitiveDefaults.updateState(tenant, agent, snapshot, tickEventRef)` with `stateVersion` incremented, then journal `SituationalModulation`. `published` is in memory only, so a fresh load always republishes once (R2-03).
6. Persist (§Persistence) if anything changed.
7. Register the tenant with the consolidation scheduler (`registerTenant`, #1063). This is for audit cadence; correctness does not depend on it.

#### `SituationalDeformationPhase` (ConsolidationPhase)

- For each state node in the tenant's `personality-state` subgraph (`nodesIn`, one node per agent), it calls `advance(slot, now)` under the slot lock and persists.
- When any hour closed, it **publishes** via `updateState`, so drift, set-point decay and healing reach `restingFor` and `forAgent` for live slots without waiting for a tick (R3-04). It also journals `SituationalDeformationConsolidated`.
- It is a *trigger* that keeps idle agents' persisted state current and auditable.
- It is not where lasting change is *computed*: that is the grid, so starvation (#1063) delays audit entries but never changes results.
- This amends the round-1 settled decision "resilience and deformation only in a ConsolidationPhase", for the reason R2-05 exposed: a consolidation-only update is cadence-dependent and starves.
  - The intent of that decision still holds. Slow variables change only at hour closes, never per tick, and never at a tick-rate-dependent rate.
  - The audit point stays bounded at one entry per agent per hour of activity.

#### Invariants (property-tested)

| Invariant | Why |
|---|---|
| `I_m ∈ [0,1]`, `E_m ∈ [0,1]` | convex sums; min(1, ·) |
| `X_m = 0 ⇒ E_m = 0` | multiplicative gains |
| situation inputs at or below the calibrated floor cannot hold a mode | `above_floor(x_floor)` + measured `m_floor` + validated `G·x_sub < θ_off` |
| `\|Δ_f\| ≤ S_max`; `\|o_f\| ≤ D_max`; `r_f ∈ [r_min, 1]` | clamps, saturation, floor |
| `c_f ∈ [0,1]` | composition clamp; pull keeps `home + d` in range |
| result independent of tick rate and advance cadence | fixed wall-clock grid; exact relaxation integrals |

`PersonalityFacet` rejects out-of-range scores. All clamps live in the state machine.

#### Worked example (Creator, stress, Emotional Lability δ = +0.20; simulated with the integrator above)

Rev 4 defaults: `w_tail = 0.1`, `hold = 10 min`, `τ_fatigue = 12`, `τ_drift = 15`. Simulation: `agents/implementor/sim712_v2.py`.

| Scenario | r | o | \|c − b\| |
|---|---|---|---|
| One 10-turn exchange (1 turn/min, I = 1), then 24 h idle | 0.987 | +0.0001 | 0.004 |
| One 30-turn conversation, then 24 h idle | 0.981 | +0.0002 | 0.004 |
| One 100-turn conversation, then 24 h idle | 0.962 | +0.0008 | 0.005 |
| **Dose:** 10 turns daily × 30 days | 0.70 | +0.045 | — |
| **Dose:** 30 turns daily × 30 days | 0.57 | +0.071 | — |
| **Dose:** 100 turns daily × 30 days | 0.20 | +0.098 | 0.19 |
| 100 turns daily — day 7 | 0.75 | +0.026 | 0.04 |
| — day 14 | 0.53 | +0.069 | 0.10 |
| — day 21 | 0.33 | +0.092 | 0.15 |
| — day 30 + 24 h idle | 0.22 | +0.098 | 0.14 |
| — then 180 days without stress | 0.998 | +0.036 | 0.036 |

- **Dose-response** (R3-05). Lasting change grows monotonically with daily exposure and saturates: 10 / 30 / 100 turns per day for a month leaves o = 0.045 / 0.071 / 0.098. Ten minutes a day leaves 46 % of the trace of 100 minutes a day. In rev 3 the figure was 100 %.
- **The r-dependent relaxation matters** (R2-05). Under daily 100-turn stress, resilience reaches the floor in about four weeks. The effective return time `τ_return/r` then exceeds the gap between sessions, so the character stops returning between episodes: chronic stress.
- After six stress-free months, resilience has healed and about 37 % of the deformation remains (set-point decay e^{−1}), the partial return Jeronimus et al. report.
- Advancing the same 14-day history in 5-minute versus 7-day steps gives identical `r` and `o` to six decimals.

### Facet coverage and initialisation

- **Scale.** Runtime state is on [0,1]. Eidos values are converted once: baseline facets by `toDescriptorView`, deltas by `toSituationalDeltas`.
- **Only baseline facets** (`F_agent` = facets in `ampdFacets`) are modulated. There is no 0.5 seeding, because that would switch CDE Category A methods from the axes regime to the facets regime.
  - Profile shifts for absent facets are listed in `skippedFacets`.
  - Agents without `ampdFacets` are not modulated; the reason is journaled once.
- **Initial state** for a new (tenant, agent) or a newly present facet: `o = d = 0`, `r = 1`, `A = L = 0`, intensities 0, `hourCursor` = start of the current hour.
- **Provenance cue.** Shifted facets carry `origin = "situational"`. The LLM appraisal prompt renders the current and usual values and states that situational traits describe the character right now (Part 0c).
- **Discrete thresholds.** `isHigh` / `isLow` (0.7 / 0.3) branches are protected by `ε_publish`, engagement hysteresis and the winner switch margin. Eval measures flicker.

### Persistence

- **Dedicated subgraph.** New `SubgraphTypes.PERSONALITY_STATE = "personality-state"`. There is one subgraph per tenant named `personality-state`, created once and located via `listSubgraphs(tenantId)`. The subgraph count per tenant is small and the list is unbounded, so there is no limit issue.
  - Keeping state out of the COGNITIVE subgraph keeps it out of every COGNITIVE-typed retrieval, consolidation scan and rendering path. A 75-property numeric node is not a memory.
  - Implementation must confirm that MindMap decorators (confidence decay, derived edges) leave this subgraph type untouched. A test runs a full consolidation cycle and asserts the node is unchanged except by `SituationalDeformationPhase`.
  - **Tenant-level create-once** (R3-07).
    - The subgraph id is resolved *before* any slot lock is taken, under a per-tenant lock in `SituationalStateService`: re-check `listSubgraphs`, create if absent, then cache per tenant.
    - Lock order: tenant lock → (released) → slot lock. A tenant lock is never taken while a slot lock is held.
    - If `listSubgraphs` ever returns more than one `personality-state` subgraph (a pre-fix race, or two processes), the one with the lowest id is canonical. Nodes found in the others are migrated into it on discovery, and a `PersonalityStateSubgraphMerged` warning is journaled.
    - The same race exists in `BehavioralSynthesisPhase.findOrCreateBehavioralSubgraph` (`:482-490`); it is added to #1064.
- **Deterministic, case-safe identity** (R2-04, R3-07).
  - There is one node per agent, named `situational-personality-state:<id>`, where `<id>` is the lowercase base32 encoding of SHA-256(agentId), truncated to 26 characters (130 bits).
  - That name is immune to the store's case-insensitive matching: `resolveNode` compares names `COLLATE NOCASE` and checks the tenant-wide alias table on `lower(name)` (`SqliteMindMapStore:473-492`). Agent ids that differ only in case (`Frida` / `frida`) get distinct nodes.
  - The node carries the raw `agent-id`. On every resolve the service checks it for exact equality and treats a mismatch as a collision error (journaled, never silently merged).
  - The node id is cached in the slot.
- **Upsert.** Under the per-key lock: resolve, then `addNode` if absent, else `updateNode`. Writes after the first are always `updateNode` on the cached id. Bounded `search` + filter is never used (#1064 tracks the existing instances of that idiom).
- **Properties.**
  - `agent-id`, `schemaVersion`, `stateVersion`, `cursorTime`, `hourCursor`, `exposedUntil`
  - per facet: `facet.<Name>.displacement`, `.homeOffset`, `.resilience`, `.exposure`, `.load`
  - per mode: `mode.<mode>.intensity`, `.engaged`
  - per group: `winner.<group>`
- **Enumeration.** `SituationalDeformationPhase` iterates `nodesIn(personalityStateSubgraphId, tenantId)`. That set contains only state nodes, so its size equals the number of agents with state.
- **Why MindMap.** Unchanged from rev 2: #845 Step 0c; tenant-scoped SPI; backend ladder; consolidation operates on MindMap.
- **Caches and crash semantics.** `SituationalStateService` keeps a tenant-keyed slot cache with per-key locks and 24 h idle eviction (the same policy as `EffectiveCognitiveDefaults`). It writes through at the end of every tick or phase advance that changed state, so a crash loses at most the in-flight tick.

ASSUMPTION: a single neocortex process owns a given (tenant, agent) at a time. The per-key lock is in-process. Multi-instance ownership would need MindMap-level optimistic concurrency, which `MindMapStore` does not offer today.

### Pipeline integration

- `SituationalPersonalityParticipant` registers in TERMINAL through `CognitivePipelineBuilder.situationalPersonality(...)` (Part 0d). The same builder call installs the tick-start catch-up.
- It needs a non-null `context.descriptor()` and the tick `EventRef` (Part 0c).
- **Collaborators:** `SituationalStateService`, `SituationalProfileResolver` + `toSituationalDeltas`, `AppraisalTickParticipant`, `MoodOrchestrator`, `CapsEngine` (optional), `RelationalSchemaResolver`, `EffectiveCognitiveDefaults`, `CognitiveJournal`, `ConsolidationScheduler.registerTenant`, `Clock`.
- **Per-tick CAPS settlement.** When situation inputs exist, the participant calls `loadState` + `settle`, with the rest reference cached per CAPS generation. If the agent has no CAPS state, the mediating signals are structurally unavailable (renormalised). Settlements are not yet consolidated across regulation and coping (#1062).

### Observability

- **`SituationalModulation`** `(tenantId, agentId, EventRef, timestamp, intensities, winners, X, N, R, E per mode, compositeDelta, publishedFacets, skippedFacets, profileProvenance)`. Recorded on publish.
- **`SituationalDeformationConsolidated`** `(tenantId, agentId, EventRef, timestamp, hoursClosed, homeOffsetBefore/After, resilienceBefore/After, trigger = tick|consolidation)`.
- **Existing entries.** `CapsSettled` for the per-tick settlement. `StageComputed` / `DataFlowed` attribute facet → parameter changes for both resting and effective recomputations.
- **`PipelineTrace.SituationalPersonalitySnapshot`** `(intensities, winners, evidence, profileProvenance, baselineFacets, currentFacets, homeOffsets, resilience)`, aggregated in `CognitionCore.currentTrace`.

### Relationship to PersonalityEvolutionOrchestrator

- PEO is never ticked, its profile-store writes don't reach the descriptor, and it never reads facets (#1060).
- The convergence point is `EffectiveCognitiveDefaults`: evolved Jungian profiles arrive through the base view (`updateBase` / `DescriptorViewSource`), and situational state through `PersonalityStateSource` / `updateState`.
- PEO is a durable learner, so it reads `restingFor`.

## Parameters

All parameters are in `situational-personality.yaml`, and all are `provenance: estimated` and tuned by eval unless noted.

| Parameter | Default | Unit | Meaning | Source / rationale |
|---|---|---|---|---|
| `alpha_pull` | 0.30 | per turn | gap fraction closed per turn while a mode wins | saturates in ~9 turns |
| `tau_return` | 6 h | time | return time constant at r = 1 (÷ r) | slower than mood (4 h) |
| `S_max` | 0.25 | facet | max composite shift | ≈ 1 facet SD |
| `D_max` | 0.10 | facet | max lasting deformation | Roberts et al. 2006; Jeronimus et al. 2014 (≈ ½ SD) |
| `tau_fatigue` | 12 | facet·h (weighted) | weighted load to exhaust resilience | worked example (re-tuned with exposure weighting) |
| `tau_heal` | 30 d | time | resilience recovery | Bonanno 2004 |
| `r_min` | 0.2 | — | resilience floor | no ratchet to zero |
| `tau_drift` | 15 | h | weighted exposure → deformation | worked example (re-tuned) |
| `w_tail` | 0.1 | — | dose weight of the post-exposure relaxation tail | perseverative cognition (Brosschot et al. 2006); dose-response criterion |
| `hold` | 10 min | time | exposure window extension per evidence tick | covers inter-turn gaps within a conversation |
| `tau_setpoint` | 180 d | time | deformation decay to baseline | Jeronimus et al. 2014 |
| grid step | 1 h | time | slow-update resolution | ≪ every slow τ |
| `epsilon_publish` | 0.01 | facet | publish / journal quantum | below PID-5 resolution |
| `beta_N` / `beta_R` | 0.5 / 0.25 | — | endogenous / relational gain | mood-congruent amplification; bounded by the sub-floor hold margin |
| `x_floor` (per classifier) | calibrated (uncalibrated default 0.6) | confidence | classifier noise floor | 95th percentile on the pre-registered neutral/incidental corpus |
| `m_floor` | 0.3 | — | bound on from-rest mediating response to inputs at `x_floor` | checked by topology integration test |
| `switch_margin` | 0.10 | intensity | winner hysteresis | — |
| per mode `threshold_on / off` | 0.50 / 0.42 | — | engagement hysteresis | must satisfy the validated margins |
| per mode `fade_tau` | stress 2 h, flow 45 min, social 30 min | time | intensity fade | — |

## Testing

### Unit (pure functions)

- **Detector.**
  - Every transform.
  - Gain form: `X = 0 ⇒ E = 0` for all N, R.
  - Structural vs silent sources: no observation scores 0 (not renormalised); missing `CapsEngine` is renormalised.
  - Stale SEC (an appraisal from an earlier tick) contributes 0.
  - Social gate; group exclusion; winner switch margin (close stress/flow intensities never flip on consecutive ticks).
  - Hysteresis; time-based fade.
- **Lock-in properties** (R2-01, R3-02). For every situation-input vector with all activations ≤ `x_floor`, mediating from-rest responses ≤ `m_floor`, N = 1 and R = 1: the mode never engages, and an engaged mode disengages within `τ_fade·ln(1/θ_off)`. Config validation rejects any parameter set violating `G·x_sub < θ_off`. Activations above 1 from relational biasing are clamped (Part 0c), so the property's [0,1] premise holds.
- **Relational context** (R2-06). A trusted interlocutor with all situation inputs at 0 gives E = 0. A neutral stranger (trust 0) gives R = 0.
- **Integrator** (property-based over random event histories):
  - invariants table holds;
  - the same history advanced at 5 min, 1 h, 1 day and 7 day cadences, and ticked at 1 turn/s versus 1 turn/min of equal real time, gives `o, r` within 1e-6 (the grid is exact) and `A` within 1 %;
  - the worked-example table reproduces within tolerance;
  - an idle week returns `d` → 0;
  - facets with δ = 0 accrue no load;
  - **dose-response** (R3-05): `o` after 30 days is strictly increasing in daily turns (10 < 30 < 100), and `o(30 d × 10 turns) ≤ ½ · o(30 d × 100 turns)`;
  - the exposure window: load accrues at weight 1 only inside windows opened by evidence ticks, and at `w_tail` elsewhere;
  - tick-start catch-up (R3-04): after a 12 h gap, the effective defaults read in DERIVED reflect the relaxed `d`, not the pre-gap one.
- **Baseline change** (R2-07):
  - a baseline correction mid-episode accrues no load and no negative exposure (state is relative);
  - a facet added to the baseline appears in the effective facets, initialised at rest;
  - a removed facet disappears from state and from the effective facets and is journaled.
- **Config validation.** Every rejection rule above.
- **Part 0a.**
  - Re-derivation with changed state changes the effective appraisal weights while the resting ones stay unchanged.
  - A pinned explicit field survives.
  - An explicit profile with facets is rejected.
  - Two tenants with the same agentId stay independent.
  - Concurrent `updateState` loses no update.
  - A slot miss pulls from both sources.
  - Eviction followed by a read gives an identical effective result.
  - An explicit-layer reload re-merges all slots.
  - `agentIds(tenant)` includes Eidos-registered agents.
  - Typed views (R3-06): every learning rule's signature takes `LearningProfile`. A field-classification test enumerates every `CognitiveDefaults` component and asserts its view.
  - **Concurrency** (R3-03):
    - a miss-read and a publish on the same key, from 2–16 threads, never deadlock (bounded-time assertion) and install in version order;
    - the same on two keys forced into one `ConcurrentHashMap` bin;
    - a stale miss result (older `stateVersion` or `baseVersion`) is discarded;
    - no I/O inside `compute` (instrumented sources assert they are not called on the map's thread inside the lambda);
    - `PersonalityStateSource.find` takes no lock.
  - **Base source** (R3-04): the registry wins over a differing tick descriptor (mismatch journaled once); an unregistered agent uses the tick descriptor; an `AgentRegistryEvents.updated` invalidates and refreshes a slot whose agent never ticks.
  - Consolidation publish: an hour close in `SituationalDeformationPhase` changes `restingFor` for a live slot without a tick.
- **Part 0b** (eidos#234):
  - facet-name canonicalisation (`emotional_lability`, `EMOTIONAL LABILITY` → `EMOTIONAL_LABILITY`), and unknown names rejected;
  - the resolver: authored mode overrides the default, unknown archetype gives empty, provenance carries the table version;
  - a retuned table changes the resolution without re-registration.
- **Part 0c.** `LlmAppraisalStrategy` returns the parsed `secDimensions`. `currentResultAt` returns empty after an observation-less tick. `RelationalBiasing` output is in [0, 1] for trust ∈ [−1, 1].
- **Persistence edge cases** (R3-07):
  - concurrent first ticks of two agents in a new tenant create exactly one `personality-state` subgraph;
  - pre-existing duplicate subgraphs are merged into the lowest id;
  - agents `Frida` and `frida` get distinct nodes;
  - an `agent-id` mismatch on resolve is reported as a collision.

### Integration

- **Wiring** (Part 0d, agency#484): the Quarkus bean graph and the Spring context each produce a `CognitionCore` with the situational participant in TERMINAL, appraisal with a `SituationClassifier`, `effectiveDefaults`, the CAPS engine, and `SituationalDeformationPhase` among the scheduler's phases. `build()` fails fast on a half-wired configuration.
- **End to end.** A deterministic `ScriptedSituationClassifier` (test fixture: observation → declared activations) + `SqliteCapsEngine` + MindMap + a builder-assembled `CognitionCore`. `RuleBasedSituationClassifier` cannot be used: it returns no activations (§Background).
  1. a strong-threat observation engages stress;
  2. at tick t+1, appraisal and regulation read changed effective defaults;
  3. neutral observations disengage the mode and `d` relaxes;
  4. the journal entries carry the tick `EventRef`.
- **Noise-floor topology test.** Settle the real `caps-topology.yaml` with every situation input at the wired classifier's `x_floor` (and at the uncalibrated default 0.6). Assert that the from-rest mediating responses are ≤ `m_floor`, which grounds the validated margin.
- **Classifier calibration** (pre-registered, eval-tier). Run `LlmSituationClassifier` over the frozen 600-utterance neutral/incidental corpus and record the 95th-percentile confidence as `x_floor` with model id and corpus hash. Re-run on every classifier model change; the loader refuses a calibration entry whose model id does not match the wired model.
- **Restart** (R2-03). Deform an agent, restart, then *without a tick* read `forAgent` and `restingFor`. Both equal the pre-restart values, because the slot is pulled from `PersonalityStateSource`.
- **Persistence at scale** (R2-04). With more than 2000 COGNITIVE nodes in the tenant, the state node is resolved, updated in place (never duplicated), and enumerated by `SituationalDeformationPhase`.
- **No tick.** A consolidation phase reading `restingFor` for an Eidos agent that has not ticked since restart gets Eidos-derived defaults, not empty ones.
- **Tenancy.** The same agentId in two tenants: stress in A leaves B's state and defaults untouched.
- **Consolidation interleaving.** A deformation phase advancing during an active tick stream: there is no interleaving (lock), and hours close exactly once.

### Eval (pre-registered falsification criteria)

- **Differentiation.** Run the 12 families through an identical stress scenario. The mean pairwise distance between normalised effective `CognitiveDefaults` vectors must be greater during stress than at baseline (paired test, p < 0.05).
- **Coherence.** Under stress, the modulated run scores ≥ the unmodulated run (same seeds) by ≥ 0.5 on a 5-point archetype-consistency judge scale.
- **Recovery.**
  - A single 30-turn episode + 24 h idle: `max_f |c − b| ≤ 0.02`.
  - 30 daily episodes: stressed facets end with `o ∈ [0.05, D_max]` and `r ≤ 0.4`.
  - Then 180 stress-free days: `o ∈ [0.02, 0.05]` and `r ≥ 0.95`.
  - Facets the profile does not touch: `|o| < 0.005`.
- **Dose-response** (R3-05): over 30 simulated days in the eval harness, 10-turn daily stress leaves at most half the offset of 100-turn daily stress, and both exceed the single-episode offset by at least 10×.
- **Harness.** Eval runs on the agentic-yaml `CognitionStack`, assembled through `CognitivePipelineBuilder` (Part 0d) with the real `LlmSituationClassifier` and its calibrated `x_floor`.
- **Flicker.** ≤ 1 `isHigh`/`isLow` flip per facet per mode episode, and ≤ 1 group-winner change per minute.
- **Signal presence.** SEC dimensions are non-empty in ≥ 95 % of appraised ticks under `LlmAppraisalStrategy` (guards Part 0c).
- **Noise lock-in.** After a stress episode, a 100-turn conversation drawn from the held-out half of the incidental-mention corpus, run through the real `LlmSituationClassifier`, disengages within `τ_fade·ln(1/θ_off)`.

## Implementation ordering

| # | Step | Depends on |
|---|---|---|
| 0a | `EffectiveCognitiveDefaults` (resting/effective, sources, enumeration, eviction, reload), explicit-layer split, `merge`, consumer migration by the rule, public `toDescriptorView` (#1055) | 0b facet enum |
| 0b | Eidos: `AmpdFacetName`, `situationalProfiles`, `SituationalProfileResolver` (read-time, versioned table), 60 profiles (eidos#234) | — |
| 0c | `LlmAppraisalStrategy` SEC dims; `currentResultAt`; `currentInputActivations`; `tickEventRef`; prompt clause; `RelationalBiasing` clamp | — |
| 0d | `CognitivePipelineBuilder` as the single assembly path (package-private `CognitionCore` constructor, builder extensions, fail-fast); agency Quarkus and Spring roots and `CognitionStack` migrated; wiring tests (agency#484) | 0a, and step 6 for the situational registration |
| 0e | `LlmSituationClassifier` calibration corpus (frozen, split) and `x_floor` measurement | — |
| 1 | `situational-personality.yaml` + loader/validator (needs `CapsTopology`) | — |
| 2 | Integrator (`advance`) + pull, pure functions | — |
| 3 | Detector (gain form, transforms, winners) | 1 |
| 4 | `RelationalSchemaResolver` extraction; `toSituationalDeltas`; `SubgraphTypes.PERSONALITY_STATE`; `ConsolidationScheduler.registerTenant` | 0b |
| 5 | `SituationalStateService` (cache, locks, MindMap identity/upsert, `PersonalityStateSource`) | 2, 4 |
| 6 | `SituationalPersonalityParticipant` (TERMINAL) | 0a, 0c, 1–5 |
| 7 | `SituationalDeformationPhase` | 5 |
| 8 | Journal entries, `PipelineTrace` snapshot | 6, 7 |
| 9 | Eval | all |

Steps 0b, 0c, 0e, 1, 2 and 4 are independent. 0a needs the facet enum from 0b. 0d's builder work can start with 0a; its situational registration lands with step 6.

## References

- casehubio/neocortex#712 (parent).
  - Deferred: #1056, #1057, #1058, #1059.
  - Prerequisites: #1055, casehubio/eidos#234, casehubio/agency#484.
  - Related defects: #1060, #1061, #1062, #1063, #1064, #1065, #1049, #1050/#1051, #981, #599.
- casehubio/neocortex#845 — `examples/docs/specs/issue-845-cde-pid5-facets/2026-11-13-cde-pid5-facet-upgrade-design.md` (Step 0c).
- `neocortex/docs/research/era7-personality-trait-integration-research.md`.
- Code: `CognitiveDerivationEngine`, `CognitiveDefaultsRegistry`, `EidosCognitiveProfileBridge`, `CognitionCore`, `AppraisalTickParticipant`, `LlmAppraisalStrategy`, `EmotionRegulationTickParticipant`, `RelationalSchemaEvolutionPhase`, `BehavioralSynthesisPhase`, `CognitiveAttentionAccumulator`, `ConsolidationScheduler`, `MindMapStore`, `MoodOrchestrator`, `caps-topology.yaml`, eidos `AgentRegistry`, `ArchetypeDeriver`, `ArchetypeTerm`, `ArchetypeFamily`, `AmpdFacet`.
- Bonanno, G. A. (2004). Loss, trauma, and human resilience. *American Psychologist*, 59(1), 20–28.
- Bower, G. H. (1981). Mood and memory. *American Psychologist*, 36(2), 129–148.
- Brosschot, J. F., Gerin, W. & Thayer, J. F. (2006). The perseverative cognition hypothesis: a review of worry, prolonged stress-related physiological activation, and health. *Journal of Psychosomatic Research*, 60(2), 113–124.
- Csikszentmihalyi, M. (1990). *Flow: The Psychology of Optimal Experience*. Harper & Row.
- Fleeson, W. & Jayawickreme, E. (2015). Whole Trait Theory. *Journal of Research in Personality*, 56, 82–92.
- Forgas, J. P. (1995). Mood and judgment: the affect infusion model (AIM). *Psychological Bulletin*, 117(1), 39–66.
- Gore, W. L. & Widiger, T. A. (2013). The DSM-5 dimensional trait model and five-factor models of general personality. *Journal of Abnormal Psychology*, 122(3).
- Hartwell, M. & Chen, J. (2012). *Archetypes in Branding* (taxonomy only).
- Jeronimus, B. F., Riese, H., Sanderman, R. & Ormel, J. (2014). Mutual reinforcement between neuroticism and life experiences. *JPSP*, 107(4), 751–764.
- Jung, C. G. (1959). *The Archetypes and the Collective Unconscious*. CW 9i.
- Krueger, R. F. et al. (2012). PID-5. *Psychological Medicine*, 42(9).
- Lazarus, R. S. & Folkman, S. (1984). *Stress, Appraisal, and Coping*. Springer.
- Mark, M. & Pearson, C. S. (2001). *The Hero and the Outlaw*. McGraw-Hill.
- Mischel, W. & Shoda, Y. (1995). A cognitive-affective system theory of personality. *Psychological Review*, 102(2), 246–268.
- Nakamura, J. & Csikszentmihalyi, M. (2002). The concept of flow. In *Handbook of Positive Psychology*.
- Pearson, C. S. (1991). *Awakening the Heroes Within*. HarperCollins.
- Post, R. M. (1992). Transduction of psychosocial stress into the neurobiology of recurrent affective disorder. *American Journal of Psychiatry*, 149(8), 999–1010.
- Roberts, B. W., Walton, K. E. & Viechtbauer, W. (2006). Patterns of mean-level change in personality traits across the life course. *Psychological Bulletin*, 132(1), 1–25.
- Scherer, K. R. (2001). Appraisal considered as a process of multilevel sequential checking. In *Appraisal Processes in Emotion*.
- Wrzus, C. & Roberts, B. W. (2017). The TESSERA framework. *Personality and Social Psychology Review*, 21(3), 253–277.
