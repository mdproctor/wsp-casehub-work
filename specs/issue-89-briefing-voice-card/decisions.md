# Decisions — #89 Redefine Briefing as Voice Card

## D1: Briefing field structure

**Choice:** Structured sub-fields in the eidos descriptor — a single `voice` field of type `AgentVoiceProfile` record (like `AgentDisposition` already is), not flat fields scattered on `AgentDescriptor`.
**Alternatives:**
- Single text field (rewrite content only) — simpler but the constraint is social (authoring convention) not structural; doesn't prevent behavioral instructions creeping back
- Hybrid (voice field + briefing) — half-measure, two fields but no full separation
- 8+ flat fields on AgentDescriptor — bloats the already 26-field record; rejected after decision review R1-02
**Rationale:** Pre-release platform — the right time to harden foundations. Structural separation enforces the voice/behavior boundary at the API level, not just by convention. Follows the same principle as the write-content skill's Form/Mode/Voice taxonomy: what the agent is, how it behaves, and how it sounds are different concerns that should be structurally distinct. The `AgentVoiceProfile` record pattern matches `AgentDisposition` — a nested structured type rather than flat fields.
**Trade-offs:** Requires eidos API changes (new record type, renderer updates, vocabulary support). Every consuming app's descriptor YAML needs updating.
**Sources:** write-content skill (Form/Mode/Voice taxonomy), mark-proctor-voice.md (voice fingerprint structure), descriptors-composite.yaml (current briefing content), AgentDisposition (pattern for nested structured type)
**Exploration:** quick
**Revision:** R1-02 — voice profile as nested record, not flat fields on AgentDescriptor
**Status:** revised

## D2: Voice profile fields — layered voice authoring

**Choice:** Layered voice authoring with a `description` field as the primary voice identity signal. `AgentVoiceProfile` record contains: `description` (natural language voice identity phrase), `register` (vocabulary-resolved), `accent` (vocabulary-resolved), `catchphrases` (list), `speech_patterns` (list), `vocabulary_uses` (list), `vocabulary_avoids` (list), `quirks` (list), `personas` (optional map of named voice variants). Layer 1 (`description`) lets the LLM draw from training data for well-known characters. Layer 2 (structured fields) provides optional fine-tuning where the LLM's default isn't quite right. Both layers render together — description first, structured fields as supplements.
**Alternatives:**
- Pure structured, no freeform (R1-03, original choice) — works for cartoon characters but doesn't generalise: clinical and AML agents don't have catchphrases or quirks. Enumerated fields over-constrain well-known characters and waste tokens restating what the LLM already knows.
- Prose-only (single text field) — simpler but no programmatic hooks for persona selection, vocabulary resolution, or tooling.
- Template-based voice (R1-02 alternative) — templates render to prose, not programmatically inspectable; can't support persona selection (D3).
**Rationale:** For well-known characters (e.g., Wacky Races), the LLM's training data IS the voice definition — a brief description is sufficient. For original characters, structured fields add information the LLM doesn't have. The enumerated fields (catchphrases, vocabulary) were domain-specific to cartoon characters and didn't generalise to clinical, AML, or other domains. Layered authoring gives the best of both: token-efficient for known characters, precise for original ones.
**Trade-offs:** Authors could put behavioral instructions in the description field. The separation relies on authoring discipline rather than structural enforcement. Mitigated by the cognitive system carrying behavior independently.
**Sources:** Analysis of 4 Wacky Races characters, domain generality testing against clinical/AML use cases. R2-01 — revised after implementation showed enumerated fields redundant for well-known characters.
**Exploration:** deep-analysis
**Revision:** R2-01 — layered voice authoring (description as Layer 1, structured fields as optional Layer 2)
**Status:** revised

## D3: Hooded Claw dual-voice — personas map with cognitive switching

**Choice:** Multi-voice characters use a `personas` map in the voice profile (e.g., `sneekly: {register: obsequious, ...}`, `claw: {register: grandiose, ...}`). The switching logic lives in the cognitive layer via existing HARD constraints and goals. System prompt contains ALL personas; observation signals which is active. This preserves system prompt caching.
**Alternatives:**
- Single voice with prose instructions for switching — conflates voice and behavior
- Separate descriptors per persona — over-engineers identity
- Cache key includes active persona — loses caching benefit for persona-switching characters
**Rationale:** The switching rule is cognitive behavior (driven by `never-break-cover` constraint and `maintain-disguise` goal). The two voices are independently structurable. All personas in the system prompt preserves caching; the observation layer signals the active persona.
**Trade-offs:** System prompt is slightly larger (contains all persona variants). LLM must honour the persona selection signal from the observation layer.
**Sources:** Hooded Claw briefing analysis, CognitionCore constraint handling
**Depends on:** D2 (voice profile fields)
**Exploration:** deep-analysis
**Revision:** R1-06 — resolved caching conflict by putting all personas in system prompt
**Status:** revised

## D4: Architecture — clean two-layer split with correct renderer ownership

**Choice:** Clean the existing two-layer architecture. `CognitiveSystemPromptRenderer` (blocks-core) owns the system prompt for cognitive apps — it already exists and was designed as part of the directive-minimal architecture (#63). Eidos provides identity data via `AgentDescriptor`. Blocks/app owns the observation (cognitive state). Non-cognitive apps continue using `EidosSystemPromptRenderer`.
**Alternatives:**
- Eidos owns everything — inverts dependency direction
- New composition module — adds complexity for something existing classes already handle
- Pretend eidos owns the system prompt — contradicts the codebase; `CognitiveSystemPromptRenderer` already overrides eidos renderer via `@Alternative @Priority(1)` pattern
**Rationale:** The two-layer split maps to the LLM API contract. `CognitiveSystemPromptRenderer` was explicitly designed to take over system prompt rendering for cognitive apps, producing a minimal directive (identity + voice + HARD constraints + preamble) while all dynamic cognitive data flows through the observation layer. This is the established design from #63.
**Trade-offs:** Two renderer paths (cognitive vs eidos default). Cognitive renderer must be updated to handle new voice profile fields.
**Sources:** CognitiveSystemPromptRenderer.java, directive-minimal architecture spec (#63), Phase D spec §4
**Exploration:** deep-analysis
**Revision:** R1-04 — corrected renderer ownership from "eidos" to "CognitiveSystemPromptRenderer (blocks)"
**Status:** revised

## D5: Remove duplication between layers

**Choice:** Kill `PersonalityPromptSection` — raw DispositionValue codes are a workaround; personality belongs in the system prompt rendered with vocabulary resolution. Keep `ConstraintPromptSection` for SOFT constraints — the HARD/SOFT split is intentional (#63 directive-minimal spec): HARD → system prompt as "Prime Directives", SOFT → observation (because soft constraints can be superseded by cognitive state). Distinguish authored goals (eidos descriptor) from emergent goals (GoalPromptSection).
**Alternatives:**
- Kill both PersonalityPromptSection and ConstraintPromptSection — wrong; SOFT constraints are deliberately in the observation layer
- Keep both — perpetuates raw-codes duplication for personality
**Rationale:** Personality is identity (eidos concept, vocabulary-resolved). SOFT constraints are contextual guidance the cognitive system might override — they belong in the observation layer where other cognitive state lives. The HARD/SOFT split was an explicit design decision in #63.
**Trade-offs:** Personality rendering moves to the system prompt renderer. Apps without `CognitiveSystemPromptRenderer` lose cognitive-context personality rendering.
**Sources:** CognitionCore.promptSections() lines 383-389 (SOFT-only filtering), directive-minimal spec §4.3 (constraint severity split), PersonalityPromptSection (raw codes)
**Depends on:** D4 (renderer ownership)
**Exploration:** deep-analysis
**Revision:** R1-07 — kept ConstraintPromptSection for SOFT constraints
**Status:** revised

## D6: ~~Cognitive preamble bridge~~

**Choice:** WITHDRAWN. `CognitiveSystemPromptRenderer` already calls `CognitivePreambleGenerator.generate(config)` directly (line 46). No bridge needed. The preamble is already handled by the existing renderer.
**Previous choice:** Extension data bridge from CognitionCore to eidos
**Reason for withdrawal:** R1-05 identified that the bridge solves a non-problem. The decision was based on the incorrect assumption that eidos owned the system prompt (corrected in D4 revision).
**Status:** withdrawn

## D7: Implementation scope — validate with 3-4 characters

**Choice:** Validate the design with 3-4 contrasting characters (Hooded Claw for dual-persona, Penelope for straightforward voice, Ant Hill Mob for ensemble, one more). Remaining 13 characters as a mechanical follow-up issue.
**Alternatives:**
- All 17 characters in this issue — more work, delays validation feedback
- Design only, no rewrites — misses the concrete validation that proves the design works
**Rationale:** The design needs empirical validation before scaling. Contrasting characters test the edge cases (dual-persona, ensemble, straightforward). Once validated, the remaining rewrites are mechanical.
**Trade-offs:** Two issues instead of one. The follow-up issue is low-risk but still work.
**Sources:** descriptors-composite.yaml (17 characters with varying complexity)
**Exploration:** quick
**Status:** captured

## D8: Emergence verification — three-run experimental design

**Choice:** Extend the Phase D eval infrastructure with three runs: (a) voice-only WITHOUT cognitive config — null hypothesis (does the LLM generate interesting behavior spontaneously?), (b) voice-only WITH full cognitive config — test (does the cognitive system drive behavior?), (c) compare b vs a. Persist output to a durable location (not target/). The comparison answers "does the cognitive system produce emergent behavior?" cleanly because it isolates the cognitive system's contribution.
**Alternatives:**
- Two-run design (full-briefing baseline vs voice-only) — confounded; can't distinguish "loss of instruction" from "emergence"
- Manual verification only — no reproducibility
**Rationale:** The three-run design (R1-08) isolates the variable correctly. Run (a) establishes what the LLM does with just voice. Run (b) shows what cognitive systems add. The delta (b - a) is the emergence signal. Output persisted outside target/ for reproducibility without polluting git history.
**Trade-offs:** Three eval runs instead of two. Non-deterministic LLM output means qualitative analysis, not exact diff.
**Sources:** CognitiveEvalTest.java (existing infrastructure), Phase D spec §5, R1-08 (methodology improvement)
**Depends on:** D7 (implementation scope — uses the same 3-4 characters)
**Exploration:** quick
**Revision:** R1-08 — three-run design isolates emergence correctly
**Status:** revised

## D9: Eval approach — paired state probes

**Choice:** Paired state probes — build two CharacterCognition instances (before-event and after-event cognitive state), probe both with the same situation prompt, judge the delta. Add a no-cognition control to establish random-variation baseline. ~4 LLM calls per scenario.
**Alternatives:**
- Multi-tick simulation — run CognitionCore.tick() N times, inject events at specific ticks, probe at designated ticks. More realistic but requires complex store setup (MindMapStore, memory stores, TrustEvolutionConfig). ~2N LLM calls per scenario. Rejected: the tick pipeline's correctness is unit-tested elsewhere; the interesting question is whether different cognitive states produce different behavior, not whether tick() processes events correctly.
- Narrative replay — full multi-turn conversation simulation. Most realistic but extremely expensive (~200 LLM calls), highly non-deterministic, and hard to judge ("what exactly are we measuring?"). Rejected: cost/signal ratio is poor.
**Rationale:** The emergence claim is "different cognitive states → different behavior." Approach A tests this with exactly one controlled variable (the cognitive state in the observation layer). Same voice card, same situation prompt, same character. The control run (no cognitive sections) establishes the random-variation baseline — if cognitive delta >> control delta, that's the emergence signal. Clean isolation, minimal cost, strongest inferential power.
**Trade-offs:** Doesn't prove the tick() mechanism naturally produces state evolution — only that state differences cause behavioral differences. Mitigated: tick correctness is tested by CognitionCore unit tests and EmergenceVerificationTest.
**Sources:** EmergenceVerificationTest (structural-only gap), CognitiveInfluenceEvalTest (snapshot-only gap), CognitionCore.tick() (tick pipeline), CharacterCognition.renderCognitiveSections() (section rendering)
**Depends on:** D8 (three-run emergence eval — this extends it to behavioral measurement)
**Exploration:** deep-analysis
**Status:** captured

## D10: Event scenarios — four cognitive-subsystem-spanning events

**Choice:** Four scenarios, each exercising a different cognitive subsystem:
1. **Scheme frustration (Hooded Claw)** — drives + mood. Before: scheming 0.9, neutral mood, belief "Penelope is naive." After: drive frustrated (0.5), negative mood, revised belief. Situation: unguarded valuable + Penelope nearby.
2. **Trust erosion (Penelope)** — beliefs + norms. Before: trusting norms, no suspicion. After: new belief about Sneekly's dishonesty. Situation: Sneekly offers guidance.
3. **Mood elevation (Dick Dastardly)** — mood in isolation. Before: neutral mood. After: elevated pleasure + dominance. Situation: Muttley awaiting orders.
4. **Social context shift (Hooded Claw)** — persona switching + social awareness. Before: alone (Claw). After: Penelope present (Sneekly). Situation: valuable artifact visible.
**Alternatives:**
- Fewer scenarios (2) — lower cost but weaker coverage; single failure invalidates the eval
- More scenarios (6+) — diminishing returns; 4 covers the key subsystems
- Different event selections — e.g., goal formation instead of mood elevation. Mood was chosen because it's the most isolated (pure PAD change, no SocialConfig restructuring), making it the cleanest single-variable test.
**Rationale:** Four scenarios span drives, beliefs/norms, mood, and persona/social — the major cognitive subsystem categories. Each scenario tests a different pathway from event → state change → behavioral adaptation. Together they prove the cognitive system's breadth, not just one lucky path.
**Trade-offs:** 16 LLM calls total (4 scenarios × 4 calls). Persona-switching scenario is partly structural (already proven by PersonaActivationSection), but the behavioral dimension (how the LLM adapts its response content, not just its persona label) is new.
**Sources:** SocialConfig (drives, norms, beliefs, persona-constraint), CharacterCognition.renderCognitiveSections(), social-config.yaml (existing character configs), CognitiveInfluenceEvalTest scenarios (prior art)
**Depends on:** D9 (eval approach)
**Exploration:** deep-analysis
**Status:** captured

## D11: Measurement strategy — paired LLM judge with dual assertion

**Choice:** LLM judge evaluates the paired delta — "given this event, does the AFTER response reflect the state change compared to BEFORE?" Score 0-5 on the same scale as CognitiveInfluenceEvalTest. Dual assertion: (1) cognitive delta score ≥ 3 (change is visible), (2) cognitive delta score > control delta score (change exceeds random variation). Markdown report persisted to `docs/eval/`.
**Alternatives:**
- Embedding distance — compute semantic similarity between before/after responses. Cheaper (no judge LLM call) but doesn't measure *adaptiveness*, only *difference*. Random variation also produces distance.
- Human evaluation — most accurate but not automatable, doesn't fit CI pipeline
- Structural diff only — count section differences. Already done by EmergenceVerificationTest; doesn't prove behavioral impact.
**Rationale:** The judge LLM can assess whether the response difference is *adaptive* (reflects the event) vs *random* (arbitrary variation). The dual assertion handles both type-I (false positive from random variation — caught by control comparison) and type-II (false negative from too-strict threshold — caught by ≥ 3 being a moderate bar) errors.
**Trade-offs:** Judge LLM adds cost and non-determinism. Mitigated: retry logic (existing pattern from CognitiveInfluenceEvalTest), and the dual assertion reduces false positives.
**Sources:** CognitiveInfluenceEvalTest.judgeResponse() (existing judge pattern), EvalJudgeProducer (judge infrastructure)
**Depends on:** D9 (eval approach), D10 (scenarios)
**Exploration:** quick
**Status:** captured

## D12: Test infrastructure — new test class extending existing patterns

**Choice:** New `EventDrivenEmergenceEvalTest` class — `@QuarkusTest @Tag("llm-eval")`. Uses existing `LlmTestSupport` and `CharacterCognition` rendering. Each scenario is a record `EventScenario(name, agentId, eventDescription, beforeState, afterState, situationPrompt, judgeCriteria, nearbyBefore, nearbyAfter)`. Output persisted to `docs/eval/` (durable, git-tracked).
**Alternatives:**
- Extend CognitiveInfluenceEvalTest — same class, different method. Rejected: the experimental design is fundamentally different (paired delta vs single-state probe). Mixing them obscures both.
- Standalone test (no @QuarkusTest) — rejected: needs AgentProvider for LLM calls, which requires Quarkus CDI.
**Rationale:** New class preserves the single-responsibility of each eval test. Reuses existing infrastructure (LlmTestSupport, SocialConfig, CharacterCognition) without modifying it. @Tag("llm-eval") ensures it runs only in the eval profile, not the standard suite.
**Trade-offs:** Another eval test class to maintain. Mitigated: shares infrastructure with CognitiveInfluenceEvalTest; if patterns diverge, extract a shared base.
**Sources:** CognitiveInfluenceEvalTest (pattern to follow), EmergenceVerificationTest (output persistence pattern), TestCdiBeans (CDI wiring for @QuarkusTest)
**Depends on:** D9 (eval approach)
**Exploration:** quick
**Status:** captured
