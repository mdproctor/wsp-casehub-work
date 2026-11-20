# Decisions — Emotion Regulation (#590)

## D1: Strategy scope

**Choice:** All five Gross strategies in a unified decision layer — no batching. Regulation is a decision system, not an execution system. The RegulationTickParticipant selects from all five strategy families; downstream consumers implement based on strategy type (drive rendering for strategies 3-5, goal pipeline for strategies 1-2).
**Alternatives:**
- Three post-appraisal only (distraction, reappraisal, suppression) — simpler but incomplete model, paints into a corner if goal-level strategies need different abstractions later
- Two core only (reappraisal + suppression) — misses distraction which Petter et al. show dominates at high intensity
- All five but batched (post-appraisal first, goal-level second) — unnecessary split; first-principles analysis shows all five trigger post-appraisal and differ only in implementation target
**Rationale:** From first principles: all five strategies share the same trigger (post-appraisal discrepancy detection). The difference is where implementation targets — internal state vs environment. A unified decision layer handles all five cleanly with distributed execution. No need for separate integration points.
**Sources:** Gross 1998 five-strategy process model; casehubio/neocortex#590 issue body
**Exploration:** quick
**Status:** captured

## D2: Computation approach

**Choice:** LLM-driven with literature-structured prompting. Bosse/Petter models provide the framework (strategy families, intensity thresholds, personality mappings) as structured prompt context; the LLM performs actual regulation reasoning. Mechanize later based on empirical evidence of what becomes formulaic.
**Alternatives:**
- Fully mechanical (Bosse differential equations + Petter RL selector) — deterministic and testable, but risks producing rigid/unconvincing regulation like prior keyword filtering issues
- Hybrid (mechanical selection, LLM rendering) — reasonable compromise but premature optimization before we know what quality looks like
**Rationale:** Quality first, optimize later. Published equations look clean but we don't know if they produce convincing character moments empirically. Prior bad experience with mechanical keyword filtering. The LLM gets the literature as structured input, not as a replacement for judgment.
**Trade-offs:** Non-deterministic initially; harder to unit test; higher token cost per tick. Mitigated by eval framework (drive expression eval can measure regulation quality).
**Sources:** Bosse et al. 2010 quantitative model; Petter et al. 2025 strategy choice; prior keyword filtering experience
**Exploration:** quick
**Status:** captured

## D3: LLM integration pattern

**Choice:** Per-tick LLM call in the regulation participant producing structured regulation output (strategy chosen, intensity dampening, narrative). Pattern: `CognitionCore.appraiseMood()` — uses `SessionPipeline.openSession()` with system prompt, streams response, parses JSON, with decay-based fallback for call failures.
**Alternatives:**
- Prompt-only (no generation call) — add regulation to renderInternalState() prompt sections, let character LLM handle it implicitly. Cheaper but regulation becomes unobservable and untestable
- Multi-phase generation (separate calls for identification, selection, implementation) — most faithful to Gross 2015 cycle but 3x token cost per tick
**Rationale:** Produces observable, testable regulation output stored on MindMap. Single LLM call balances quality with cost. Must include fallback logic (decay toward no-regulation when LLM call fails/times out), same as mood appraisal pattern.
**Trade-offs:** One extra LLM call per tick; regulation narrative quality depends on prompt engineering; per-tick LLM calls are unreliable (must handle failure gracefully)
**Sources:** CognitionCore.appraiseMood() per-tick LLM pattern; Gross 2015 extended process model
**Exploration:** quick
**Status:** revised (R1-02: corrected pattern reference from ConsolidationPhase to per-tick LLM pattern)

## D4: Pipeline placement

**Choice:** New RegulationTickParticipant in DERIVED phase, after AppraisalTickParticipant
**Alternatives:**
- Inside AppraisalTickParticipant — conflates emotion construction with regulation, grows an already complex class
- New phase between DERIVED and TERMINAL — maximum isolation but adds pipeline complexity, and regulation needs data from within DERIVED (CAPS output)
**Rationale:** Same pattern as existing participants. Clean separation of concerns — appraisal constructs, regulation modulates. Reads AppraisalResult, writes RegulationResult to MindMap for downstream consumption by DriveOrchestrator and renderInternalState().
**Trade-offs:** Ordering dependency within DERIVED phase — must run after appraisal
**Sources:** CognitionCore tick phase architecture; AppraisalTickParticipant pattern
**Exploration:** quick
**Status:** captured

## D5: Personality-strategy connection

**Choice:** Both personality-trait tendencies and current-state intensity/load as structured prompt context. The LLM integrates them rather than a mechanical priority rule.
**Alternatives:**
- Personality-primary (traits set default, intensity overrides only at extremes) — more predictable per-character but may miss dynamic moments
- Intensity-primary (Petter model drives selection, personality tilts tie-breakers) — more responsive but characters feel personality-less in regulation
**Rationale:** Consistent with D2 (quality first). Both dimensions are empirically validated — personality tendencies (Soleimani & Kobti extension of Bosse) and intensity-driven selection (Petter 2025). Letting the LLM integrate avoids premature mechanical weighting.
**Trade-offs:** Less predictable than a hard rule; regulation may vary across runs for same personality+intensity
**Depends on:** D2 (computation approach)
**Sources:** Petter et al. 2025 intensity model; Big Five → regulation strategy literature; Soleimani & Kobti personality-parameterized Bosse model
**Exploration:** quick
**Status:** captured

## D6: Rendering in internal state

**Choice:** Regulation section + modified drives. A new 'Regulation' ObservationSection shows active regulation (strategy deployed, target emotion, effectiveness). Drive snapshot text is also modified to reflect dampening ("you feel the pull but hold it back").
**Alternatives:**
- Regulation section only — drives unchanged, regulation advisory only. Simpler but drives feel disconnected from regulation
- Modified drives only — regulation baked into drive descriptions. Saves prompt tokens but regulation becomes invisible as a distinct cognitive process
**Rationale:** Both channels reinforce the regulation signal. The separate section makes regulation observable and testable (eval can check for regulation presence). Modified drives ensure the character LLM doesn't ignore regulation when acting on drives.
**Trade-offs:** More prompt tokens per tick; requires coordination between regulation output and drive rendering
**Depends on:** D3 (RegulationGenerationPhase produces the content), D4 (pipeline placement determines data flow)
**Sources:** Issue #590 "regulation-colored drive text"; CognitionObservationBuilder.renderInternalState() architecture
**Exploration:** quick
**Status:** captured

## D7: Data model

**Choice:** Structured fields — strategy (enum of 5 Gross strategies), targetEmotion, originalIntensity, regulatedIntensity, effectiveness (0-1), narrative (free text). Parseable for drive modification and eval scoring, while narrative preserves LLM richness.
**Alternatives:**
- Narrative-only — maximum LLM flexibility but hard to consume programmatically for drive modification and eval
- Full Bosse state vector (norm level, persistence factor, adaptation rate) — over-commits to a model we chose not to mechanize (D2)
**Rationale:** Structured fields give eval measurability (intensity deltas, strategy distribution across runs) and drive rendering can read the intensity change mechanically. Narrative field preserves the qualitative regulation experience for the prompt.
**Trade-offs:** LLM must produce structured output (JSON parsing); fields may need revision as empirical testing reveals what matters
**Depends on:** D2 (computation approach), D3 (generation phase produces the output)
**Sources:** Drive expression eval framework (classify_emotions.py); DriveSnapshot data model pattern
**Exploration:** quick
**Status:** captured

## D8: Temporal scope (cross-tick regulation)

**Choice:** Per-tick with memory context. Each tick's regulation is independent but the generation phase prompt includes recent regulation history from MindMap (last 3-5 ticks). The LLM adapts naturally — a character suppressing anger for 5 ticks will show strain because the prompt says so.
**Alternatives:**
- Stateful regulation cycle (explicit identify→select→implement→monitor state machine) — faithful to Gross 2015 but adds persistent state management and mechanical complexity contradicting D2
- Per-tick only (stateless) — simplest but misses adaptation. Suppression increasing physiological arousal over time (Gross empirical finding) would be invisible
**Rationale:** Memory-based context preserves temporal awareness without adding state machines. The LLM sees "you have been suppressing anger for 4 ticks" and can reason about cumulative strain. Same approach as existing memory-informed cognition. Adaptation emerges from context, not mechanics.
**Trade-offs:** Depends on MindMap retention of regulation history; prompt length grows with regulation history window
**Depends on:** D7 (structured fields stored on MindMap enable history retrieval)
**Sources:** Gross 2015 monitoring cycle; Bosse et al. 2010 adaptation dynamics; MindMap memory architecture
**Exploration:** quick
**Status:** captured

## D9: Distributed execution model

**Choice:** Unified decision, distributed execution. The RegulationTickParticipant selects from all five strategies and writes RegulationResult to MindMap. Two downstream consumer categories:
- **Internal strategies (3-5):** attentional deployment, cognitive change, response modulation → consumed by `CognitionObservationBuilder.renderInternalState()` at render time. Integration point is `resolveAdaptedDrives()` and `appraisalFilterDrives()`, which read drive-intensity nodes from MindMap. Regulation modifies the *rendering* of character drives (scheming, gloating, etc.), not the `DriveAxis`-level computation in `DriveOrchestrator`.
- **Environmental strategies (1-2):** situation selection, situation modification → consumed by goal pipeline. Requires API extension to `DriveGoalFormationContext` (or a new regulation-aware goal formation strategy) to accept avoidance constraints and situation-change directives.
**Alternatives:**
- Separate regulation participants for internal vs environmental strategies — violates the insight that regulation is a single decision layer
- All execution within the regulation participant — crosses pipeline boundaries, regulation participant would need to know about goals
**Rationale:** First-principles analysis: regulation decides, existing pipeline components execute. `ModulationLayer` operates on `DriveAxis` (4 fundamental axes), NOT character-level drives — regulation must integrate at the rendering layer where character drives are resolved from MindMap. Goal pipeline integration requires explicit API changes (not a "natural extension").
**Trade-offs:** Two downstream consumer integration points; GoalFormationStrategy API must be extended; renderInternalState() must learn to read RegulationResult from MindMap
**Depends on:** D1 (unified scope), D4 (pipeline placement), D7 (structured fields include strategy type for routing)
**Sources:** Gross 1998 process model; CognitionObservationBuilder.resolveAdaptedDrives(); DriveGoalFormationContext record
**Exploration:** quick
**Status:** revised (R1-04: corrected integration point from DriveOrchestrator to renderInternalState(); R1-05: acknowledged GoalFormationStrategy API change requirement)

## D10: Evaluation strategy

**Choice:** A/B with regression gate. Run the same scenario with regulation disabled (baseline) and enabled. A new regulation eval dimension scores regulation quality (strategy appropriateness, narrative coherence, personality consistency). The existing drive expression eval (classify_emotions.py) must not regress beyond a defined threshold. Selective rendering is tuned by comparing the two runs — if regulation-on degrades drive expression scores, the prompt is over-weighted toward regulation.
**Alternatives:**
- Single-run with regulation scoring — no controlled comparison, harder to attribute score changes to regulation vs run-to-run noise
- Progressive enablement (start at 1-in-5 ticks, increase) — more gradual but slower iteration and doesn't give clean A/B signal
**Rationale:** Empirical testing requires controlled comparison. The baseline run is the control; regulation-enabled is the treatment. Regression gating ensures we never ship regulation that degrades existing quality. Prompt tuning becomes an optimization loop: run A/B, measure delta, adjust prompt, repeat.
**Trade-offs:** Doubles eval run time (two runs per comparison); requires a toggle to disable regulation cleanly
**Depends on:** D7 (structured fields enable scoring), D6 (rendering changes are what gets measured)
**Sources:** Existing eval framework (classify_emotions.py, RelationalModelEvalTest); empirical-over-upfront-design principle
**Exploration:** quick
**Status:** captured

## D11: Activation gate

**Choice:** Discrepancy threshold — regulation fires only when the appraisal produces an emotion whose intensity exceeds a character-specific threshold derived from personality. Below threshold, no regulation needed (the emotion is within acceptable range). The threshold itself is discovered empirically by tuning against the A/B eval.
**Alternatives:**
- Always-on, LLM decides — generation phase runs every tick but can output "no regulation needed". More flexible but burns tokens on ticks where regulation is irrelevant
- Event-triggered only — fires only on new appraisal events, misses cumulative emotional buildup
**Rationale:** Prevents regulation from dominating quiet ticks. Personality-derived thresholds mean a high-neuroticism character regulates more often (lower threshold) while emotionally stable characters regulate rarely. The threshold is a mechanical gate — cheap to compute — that controls when the LLM generation call fires. Threshold values tuned empirically, not guessed.
**Trade-offs:** Requires a sensible initial threshold before empirical tuning; too high = regulation never fires, too low = fires every tick (same as always-on)
**Depends on:** D5 (personality connection), D10 (A/B eval provides the tuning signal)
**Sources:** Bosse et al. 2010 homeostatic baseline/norm deviation concept; Petter et al. 2025 intensity thresholds
**Exploration:** quick
**Status:** captured

## D12: Regulation target — emotions trigger, drives are modulated

**Choice:** Regulation is TRIGGERED by `CognitiveEmotion` from appraisal (the discrepancy detection reads emotion intensity) but MODULATES character-level drives in rendering. The flow: appraisal constructs emotion → regulation identifies emotion exceeding threshold → regulation selects strategy → strategy output modifies how character drives are rendered (e.g., suppression dampens "gloating" drive text, reappraisal reframes "scheming" drive).
**Alternatives:**
- Regulate emotions directly (modify AppraisalResult) — changes the appraisal output, but downstream systems that read AppraisalResult would see the regulated version, losing the original emotional signal
- Regulate drives at DriveAxis level (modify DriveOrchestrator output) — wrong granularity; DriveAxis has 4 axes, character drives have 10+ types
- Regulate emotions AND drives independently — over-specified; emotions are the trigger, drives are the expression surface
**Rationale:** Aligns with Gross: regulation operates on the emotional RESPONSE (how the feeling manifests), not on the appraisal itself. In the architecture, character drives ARE the emotional response — they're how emotions manifest as motivational pulls. CognitiveEmotion is the appraisal output (what was felt); character drives are the behavioral surface (how it pulls). Regulation sits between: reads what was felt, modifies how it pulls.
**Trade-offs:** D7 data model needs `targetEmotion` (trigger) AND `targetDrives` (what gets modulated) as separate fields
**Depends on:** D4 (pipeline placement), D7 (data model — needs field revision)
**Sources:** R1-13 review finding; CognitionObservationBuilder.resolveAdaptedDrives(); AppraisalTickParticipant.AppraisalResult
**Exploration:** quick
**Status:** captured
