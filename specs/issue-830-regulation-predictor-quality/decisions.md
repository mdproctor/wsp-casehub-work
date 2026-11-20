## D1: Implementation strategy — sequential experimentation

**Choice:** Try Approach 2 (LLM system prompt improvement) first, then Approach 3 (CognitiveDerivationEngine continuous weights). Phase 2 architecture deferred — its design will be informed by Phase 1 eval results and literature review, not upfront guesswork.
**Alternatives:**
- Approach 1 only (temporal RM pinning) — targeted shame fix but doesn't address 4→5 ceiling
- Pick one and commit — misses learning opportunity; empirical evidence should drive decisions
**Rationale:** Sequential experimentation lets each approach inform the next. Approach 2 is low-cost and reveals what LLM guidance can achieve, which calibrates how much mechanical precision Approach 3 needs. Findings documented in neocortex research docs.
**Trade-offs:** More total implementation time than picking one; risk of approach 2 being sufficient and approach 3 becoming unnecessary (acceptable — that's a good outcome)
**Sources:** Issue #830 analysis, 8 judge runs from #818 session
**Exploration:** quick
**Status:** captured

## D2: Pass disposition axes to LLM on predictor fall-through

**Choice:** Include personality disposition axes in the user message when the predictor falls through
**Alternatives:**
- System prompt only — generic shame guidance without personality context, less targeted
**Rationale:** The LLM needs the same personality information the predictor had to make personality-congruent strategy choices. The predictor falls through because no single strategy is mechanically predictable — but that doesn't mean personality is irrelevant to the choice.
**Trade-offs:** Slightly more LLM input tokens; the LLM may not use the axes effectively (eval will tell us)
**Sources:** EmotionRegulationGenerationPhase.java:155-232, RegulationStrategyPredictor.java:51-60
**Exploration:** quick
**Status:** captured

## D3: Shame guidance placement — conditional user message

**Choice:** Inject self-conscious emotion guidance as a conditional section in the user message (only when SHAME or REMORSE is dominant), not in the system prompt
**Alternatives:**
- System prompt (always visible) — simpler code but adds tokens to every regulation call
**Rationale:** Matches the existing conditional injection pattern used for fatigue, cognitive load, and strategy failure signals in `buildUserMessage()`. Keeps the system prompt generic; emotion-type-specific guidance only appears when relevant.
**Trade-offs:** Code must detect self-conscious emotions in the user message builder, adding a dependency on EmotionType classification
**Sources:** EmotionRegulationGenerationPhase.java:170-231 (conditional sections pattern), Webb et al. 2012 (strategy effectiveness varies by emotion type), Nathanson compass of shame (temporal avoidance→reappraisal sequence), Yang et al. 2025 (neural evidence for temporal suppression dynamics in avoidant attachment)
**Exploration:** deep-analysis
**Status:** captured

## D4: Phase 2 architecture — deferred to empirical evidence

**Choice:** Defer the CognitiveDerivationEngine architecture decision (replace boolean vs. augment) until Phase 1 eval results and literature review are available
**Alternatives:**
- Pre-decide now — requires guessing at what continuous weights should look like without knowing what the LLM can and can't handle
**Rationale:** Phase 1 results will reveal which personality×emotion combinations the LLM handles well with guidance vs. which need mechanical prediction. This directly informs whether continuous weights should replace or augment the boolean classification.
**Trade-offs:** Phase 2 requires its own design cycle; can't be planned in advance
**Sources:** User direction: "can we look at this through first principles and ensure it's a decision made in empirical testing and literature"
**Exploration:** quick
**Status:** captured

## D5: Phase 2 architecture — layered with interpretive rendering

**Choice:** Approach C — three-layer architecture: (1) CDE derives RegulationDefaults with continuous weights from personality (literature-grounded, static), (2) tick participant assembles ShameContext from runtime state when SELF_CONSCIOUS emotion detected (memory episodes, relational closeness, personality-gated depth), (3) generation phase renders both into psychologically grounded narrative sections, not raw data injection. ShameContext follows the existing buildUserMessage() pattern: computed values for testability + interpretive narrative that explains what each dimension means for this specific regulation episode.
**Alternatives:**
- Approach A (CDE-only, LLM handles runtime) — simpler but shame dimensions passed as unstructured prose, no mechanical integration of memory/relational context
- Approach B (full mechanical pipeline) — more deterministic but requires encoding shame thresholds the literature doesn't provide; violates literature-first principle
**Rationale:** Puts the mechanical boundary where the literature's precision ends. Continuous weights are specifiable from published personality-regulation correlations (Hughes 2020, Petter 2025). Shame dimension interactions (memory × relationship × personality depth) are qualitatively understood but lack quantitative thresholds — structured data + interpretive rendering avoids arbitrary numbers while giving the LLM psychologically grounded context, not keyword-filter data.
**Trade-offs:** More complex than A (three layers vs one), shame context rendering needs literature-grounded templates that may need iteration. Each layer independently testable.
**Sources:** Hughes et al. 2020 (Big Five → EPM stages), Petter et al. 2025 (intensity → strategy choice), end-to-end eval baseline (strategy mean 4.4, temporal containment 5.0), Phase 1 findings observations #2 #3 #5
**Exploration:** deep-analysis
**Depends on:** D4 (Phase 2 deferred to empirical evidence — now resolved by end-to-end eval)
**Status:** captured

## D6: Weight table derivation — literature-ordered, empirically calibrated

**Choice:** Derive initial weight tables from Big Five → regulation capacity mapping using Barańczuk (2019) meta-analytic correlations (132 studies, 46,345 participants). Disposition axes → Big Five approximation → regulation capacity direction and relative magnitude. Specific numerical values validated empirically via end-to-end eval.
**Alternatives:**
- Empirical-first (start with issue sketch values, tune) — faster but unprincipled initial values
- Full quantitative derivation from r values — not feasible; meta-analytic r for habitual use ≠ capacity/effectiveness (Hughes 2020 distinction)
**Rationale:** Literature constrains the ordering (analytical > cooperative > avoidant for reappraisal capacity) and relative magnitude (Openness r=0.20 vs Agreeableness r=0.18 — similar; vs Neuroticism r=-0.19 — opposite direction). But translating meta-analytic r to exact computational weights involves judgment the data doesn't fully constrain. The eval closes the gap.
**Trade-offs:** Initial weights are informed guesses within literature-constrained bounds; may need 2-3 eval iterations to calibrate
**Sources:** Barańczuk 2019 (meta-analysis, r values for Big Five × 8 strategies), Hughes et al. 2020 (capacity vs use distinction, EPM stages), Petter et al. 2025 (intensity → strategy choice)
**Exploration:** deep-analysis
**Depends on:** D5
**Status:** captured
