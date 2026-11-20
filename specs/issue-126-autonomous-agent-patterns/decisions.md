## D1: Pattern scope

**Choice:** Signal-driven orchestrator
**Alternatives:**
- Full feedback loop — bypasses eidos JPAF pipeline, computes trait deltas directly. Risks duplicating DefaultDispositionEvolution logic.
- Thin orchestrator — just wires SPIs with config, no signal translation SPI or safety rails. Too thin to be useful.
**Rationale:** Eidos already implements the full JPAF evaluation pipeline (DispositionSignalStore → DefaultDispositionHealth.probe() → DefaultDispositionEvolution.evaluate()). What's missing is the signal translation layer that maps real interaction outcomes (BehavioralSignal, QualitySignal, GoalOutcomeCounts) to disposition activation recordings, the orchestration that drives the periodic probe cycle, and safety rails the academic literature identifies as essential.
**Trade-offs:** Blocks depends on eidos SPI stability (DispositionSignalStore, DispositionHealth). If eidos's JPAF implementation changes, blocks must adapt.
**Sources:** DefaultDispositionHealth.java (eidos), DefaultDispositionEvolution.java (eidos), DispositionSignalStore.java (eidos), JPAF paper (arXiv:2601.10025), LLMPTBench (NeurIPS 2025), BFI-Adapt (arXiv:2608.06485)
**Exploration:** deep-analysis
**Status:** captured

## D2: Signal-to-activation mapping strategy

**Choice:** Layered mapping (profile-term-aware + vocabulary-structural)
**Alternatives:**
- Vocabulary-aware mapping only — maps via VocabularyTerm.opposite() for function-level activation. Only meaningful for JungianFunctionTerm; other 12 vocabulary frameworks return Optional.empty(), excluding their agents from personality evolution entirely.
- Axis-level mapping — maps to 5 DispositionAxis values. Architecturally incoherent: DefaultDispositionHealth.probe() iterates over dispositionProfile terms, not per-axis fields. Axis names recorded as activation terms would silently produce zero matches via counts.getOrDefault(dv.term(), 0).
- Raw event passthrough — passes raw domain events as activation terms, bypasses eidos JPAF infrastructure entirely.
**Rationale:** Two layers, both operating on dispositionProfile terms (the field consumed by DefaultDispositionHealth.probe()):
- Layer 1 (universal): Profile-term-aware mapping — the signal translator receives the AgentDescriptor, reads dispositionProfile terms, and maps domain events directly to those terms. Works for ANY vocabulary: Big Five agents activate "Openness"/"Conscientiousness", DISC agents activate "Dominance"/"Influence", Jungian agents activate "Ti"/"Ne". Every agent with a populated dispositionProfile participates in personality evolution.
- Layer 2 (enrichment): Vocabulary-structural mapping — for agents whose vocabulary supports structural navigation (e.g., JungianFunctionTerm.opposite()), the translator additionally infers indirect activations from vocabulary structure (e.g., negative event on Ti also activates shadow Fe). This enrichment requires vocabulary-specific structural knowledge not derivable from profile terms alone.
The translator already has access to AgentDescriptor (required for agentId/tenancyId in recordActivation), so reading dispositionProfile terms adds no dependencies.
**Trade-offs:** Two-layer mapping is more complex than either layer alone. Profile-term-aware mapping is direct but vocabulary-agnostic — agents with vocabulary structural enrichment get richer evolution trajectories (shadow function awareness). The mapping from domain event to specific profile term is SPI-defined, requiring per-domain-event-type implementations.
**Sources:** DispositionSignalStore.java (eidos), VocabularyTerm.opposite() (eidos API), AgentDisposition.dispositionProfile (eidos API), DefaultDispositionHealth.probe() (eidos runtime), GE-20260728-a53632 (vocabulary-generic structural navigation technique)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope)
**Status:** revised (R1-06, R1-07, R2-02: profile-term-aware mapping replaces axis-level mapping — aligns with probe pipeline's use of dispositionProfile terms)

## D3: Probe timing model

**Choice:** Batched periodic
**Alternatives:**
- Threshold-triggered — probe runs when activation count crosses threshold. Adapts to activity but no time-based decay anchor.
- Dual threshold + periodic — more complex, handles both high and low activity.
**Rationale:** Matches JPAF reflection model where evolution happens at episode boundaries, not per-event. Prevents micro-oscillation. Configurable frequency (per-case, daily, weekly) lets consumers tune to their domain. Time-based scheduling provides a natural anchor for decay application.
**Trade-offs:** Low-activity agents may accumulate stale activations between ticks. Mitigated by time-based decay running on the same schedule.
**Sources:** JPAF Algorithm 1 (reflection at episode boundaries), DefaultDispositionHealth.probe() (eidos)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope)
**Status:** captured

## D4: Asymmetric dampening strategy

**Choice:** Probe-time dampening with valence-tagged activations
**Alternatives:**
- Recording-time dampening — applies dampening factor at signal translation time, records already-dampened counts. Simple but permanently distorts data; retroactive recalibration impossible.
- Sentiment-aware decay rate — negative activations decay faster. Requires tracking signal source alongside counts, adds state to DispositionSignalStore.
- Fixed ratio cap — caps negative-to-positive activation ratio. Less granular than per-signal weighting.
**Rationale:** LLMPTBench (NeurIPS 2025) shows agentic frameworks exhibit exaggerated shifts under negative events. The signal translation SPI records activations with a valence tag (positive/negative). DispositionSignalStore gains a valence concept to distinguish positive from negative activation counts. DefaultDispositionHealth.probe() applies the dampening factor when computing effective weights: effectiveWeight = baseWeight + (positiveCount + negativeCount × dampeningFactor) × reinforcementDelta. The dampening factor is a configurable preference (per tenant), retroactively adjustable — changing the factor changes the next probe's computation without loss of historical data. This enables A/B testing of dampening factors across agent cohorts and analytical comparison of raw vs dampened signal distributions.
**Trade-offs:** Requires eidos SPI enhancement (valence parameter on recordActivation or separate positive/negative counts). More complex than recording-time dampening. Justified because preserving raw signal data with metadata is architecturally superior to permanently distorting it. Signal severity gradations are handled by the signal translation SPI (magnitude = how many activations to record) rather than the dampening mechanism (asymmetry = how to weight negative vs positive). Spec must document cascaded attenuation with default parameters: reinforcementDelta=0.06, overReinforcementThreshold=0.50, dampeningFactor=0.5.
**Sources:** LLMPTBench (NeurIPS 2025, OpenReview kVXePuKReA), BFI-Adapt (arXiv:2608.06485)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope), D2 (profile-term-aware + vocabulary-structural mapping)
**Status:** revised (R1-13: moved dampening from recording-time to probe-time to enable retroactive calibration)

## D5: Bounded displacement enforcement

**Choice:** L2 norm ceiling on effective weights
**Alternatives:**
- Per-function activation cap — caps max activation count per function. Doesn't account for different base weights.
- Weight-range clamping — clamps effective weights to JPAF differentiation ranges. Constrains output, not input.
- Weighted norm — assigns smaller penalty to dominant/auxiliary displacement (expected to be strong) and larger penalty to inferior displacement. More psychologically coherent but embeds Jungian theory into the safety rail.
**Rationale:** The L2 displacement metric is already exposed through the DispositionHealth SPI: DispositionStatus.Drifted(effectiveWeights, mostActivated, driftMagnitude) includes driftMagnitude which is the L2 distance. The orchestrator checks the ceiling via probe results. Between ticks, recording is gated by the halt flag (AtomicBoolean). No direct access to the private computeL2 method is needed.
**Halt flag state machine:** Complete coverage of the sealed DispositionStatus:
- Aligned → clear halt flag (no drift, recording continues)
- Drifted with driftMagnitude >= L2Ceiling → set halt flag (displacement too high)
- Drifted with driftMagnitude < L2Ceiling → clear halt flag (drift within bounds)
- EvolutionPending → trigger DispositionEvolution.evaluate(), then:
  - Evolved → apply new profile, call DispositionSignalStore.clear(), clear halt flag (new baseline = zero displacement)
  - Dampened → call DispositionSignalStore.decay() with factor, leave halt flag unchanged (next tick re-evaluates with reduced activations)
Note: probe() returns Drifted (not EvolutionPending) when the dominant exceeds overReinforcementThreshold, so EvolutionPending fires only when displacement has crossed an evolution threshold but NOT the over-reinforcement ceiling. EvolutionPending is an evolution trigger, not a halt condition.
**Complementary relationship with overReinforcementThreshold:** eidos's overReinforcementThreshold (default 0.50) is a per-function ceiling on the dominant's effective weight — fires when ONE function is disproportionately strong. The L2 ceiling is a global displacement bound — fires when the OVERALL profile drifts too far, catching distributed drift across multiple dimensions that no single per-function ceiling would detect. These are distinct geometric properties (per-dimension max vs global euclidean distance). Both mechanisms fire independently and serve complementary safety roles.
**Trade-offs:** L2 norm treats all axes equally — a shift in dominant function contributes the same as a shift in undifferentiated. This is intentional: the L2 ceiling is a safety rail, not a personality model. The JPAF pipeline handles psychological correctness (evolution detection); the L2 ceiling handles the "displacement accumulated without triggering evolution" case, which is pathological by definition. Bounded overshoot between probe ticks is an accepted trade-off of D3's periodic model.
**Sources:** DispositionStatus.Drifted.driftMagnitude (eidos API), JPAF Equations 1-3 (arXiv:2601.10025), DispositionPreferenceKeys.OVER_REINFORCEMENT_THRESHOLD (eidos, default 0.50)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope), D3 (batched periodic timing)
**Status:** revised (R1-16, R1-17: clarified L2 check mechanism via Drifted.driftMagnitude; R2-03: complete halt flag state machine covering all three DispositionStatus variants and post-evolution behavior)

## D6: Integration point

**Choice:** Application-driven CDI bean with tick() interface
**Alternatives:**
- Engine-coupled — driven by WorkOrchestrator or JobScheduler after task completion. Creates hard dependency between blocks and engine lifecycle. Blocks shouldn't know about engine.
- Standalone @Scheduled — CDI @Scheduled bean in blocks with fixed interval. Forces a specific scheduling framework; not all consuming apps use the same scheduler.
- Fully passive — triggered by signal recording (probe on every Nth recording). Contradicts D3's periodic model; couples probe cadence to interaction volume.
**Rationale:** The orchestrator is a CDI @ApplicationScoped bean that consuming apps wire into their lifecycle. It exposes a tick() method that applications call at their chosen cadence. This follows blocks' established pattern: SummarisationRunner, KeyedSummarisationRunner, and ObservationAccumulator all provide tick()/collect()/drain() methods that consuming apps invoke from their own scheduling mechanism (CDI @Scheduled, engine's JobScheduler, or explicit application code). Blocks is a library, not a framework — it doesn't own scheduling or lifecycle.
**Trade-offs:** Each consuming app must wire the tick call into its own scheduler. No out-of-the-box scheduling. Accepted because this is the same integration cost as SummarisationRunner and is consistent with blocks' architecture.
**Sources:** SummarisationRunner.tick() (blocks), KeyedSummarisationRunner.tick() (blocks), ObservationAccumulator.drain() (blocks)
**Exploration:** surfaced (R1-21: implicit decision made explicit)
**Depends on:** D1 (signal-driven orchestrator scope), D3 (batched periodic timing)
**Status:** captured

---

# MemoryHygiene (#120) Design Decisions

## D7: Scope — all five operations

**Choice:** Include all five operations: consolidation, forgetting, importance scoring, temporal versioning (§2.8), and cross-linking (§2.8)
**Alternatives:**
- Core three only (consolidation + forgetting + importance scoring) — smaller scope but defers P2-P3 fold-ins to separate issues
- Core three + integrity — adds integrity checks but still defers versioning and cross-linking
**Rationale:** Complete coverage avoids fragmented follow-up issues. The fold-ins (temporal versioning via existing supersession, cross-linking via feature maps) are lightweight additions that leverage existing store APIs.
**Trade-offs:** Larger implementation surface in a single issue
**Sources:** Research §2.2, §2.8; CbrCaseMemoryStore.supersede()/features()
**Exploration:** quick
**Status:** captured

## D8: Importance scoring placement — blocks SPIs with heuristic defaults

**Choice:** Define ImportanceScorer as a @FunctionalInterface SPI in blocks. Default implementations (ArousalScorer, SurpriseScorer) are heuristic approximations (@DefaultBean): word-list sentiment intensity for arousal, information entropy relative to agent's memory baseline for surprise. Production-quality psychological scoring requires LLM-backed consumer overrides — the defaults are functional but coarse.
**Alternatives:**
- In neocortex as retention extensions — closer to memory data but couples blocks to neocortex internals
- Split SPI in blocks / impl in neocortex — clean separation but adds cross-module coordination
**Rationale:** Blocks owns the orchestration pattern; importance scoring is a pluggable strategy within that pattern. Consumers override via CDI. Consistent with how blocks defines other SPIs (ConvergencePolicy, AcceptancePolicy, etc.). Heuristic defaults follow the HeuristicMessageSummariser precedent — zero LLM cost, consumers upgrade when needed.
**Trade-offs:** Heuristic defaults are coarse approximations of psychological constructs. LUFY uses LLM-based scoring — full fidelity requires consumer-provided LLM-backed implementations.
**Sources:** blocks scope criteria (CLAUDE.md); PersonalityEvolution TraitPressureSource pattern; HeuristicMessageSummariser precedent; LUFY (2024)
**Exploration:** quick
**Status:** revised (R1-05: clarified heuristic defaults, LLM-backed for production)

## D9: Orchestrator architecture — single-entry tick() + external composition in scheduler

**Choice:** MemoryHygieneOrchestrator has a single entry point: tick(agentId, tenantId) → HygieneTick. Runs importance scoring → consolidation (pass 1) → eviction. Consistent with PersonalityEvolutionOrchestrator.tick() and InnerLifeOrchestrator.tick(). Separate MemoryHygieneScheduler externally composes the full idle-time pipeline: calls tick() + ReflectionOrchestrator.reflect() + cross-linking + integrity checks. maintain() lives on the scheduler, not the orchestrator.
**Alternatives:**
- Dual-entry tick()/maintain() on orchestrator — introduces a new pattern not established by existing orchestrators
- Batch scheduler only — closer to CbrRetentionScheduler but no on-demand path
- Separate orchestrator + scheduler with shared pipeline stages — more flexible but more types
**Rationale:** Consistent with the established single-entry pattern (PersonalityEvolution, InnerLife). The orchestrator owns the core memory lifecycle (score → consolidate → evict). The scheduler composes higher-level operations externally. This keeps the orchestrator focused and testable.
**Trade-offs:** Scheduler has more responsibility (reflection, cross-linking, integrity) — but these are naturally batch-time concerns, not tick-time concerns
**Sources:** PersonalityEvolutionOrchestrator.tick(); InnerLifeOrchestrator.tick(); CbrRetentionScheduler
**Exploration:** quick
**Status:** revised (R1-03: single-entry tick(), maintain() moved to scheduler as external composition)

## D10: Consolidation strategy — two-pass (summarise + reflect)

**Choice:** Two-pass consolidation: Pass 1 (ContentSummariser) merges raw memories into consolidated entries. Pass 2 (ReflectionOrchestrator) generates higher-level insights from consolidated set. Tick runs pass 1 only; idle scheduler runs both.
**Alternatives:**
- SummarisationRunner composition only — simpler but misses knowledge synthesis
- ReflectionOrchestrator delegation only — generates insights but doesn't merge/reduce raw memories
**Rationale:** Two passes serve different purposes: data reduction (100→20 memories) vs knowledge synthesis ("user raises security concerns after deployments"). Tick mode stays cheap (heuristic summariser, no LLM). Idle mode leverages Sleeptime concept for full LLM-powered reflection.
**Trade-offs:** Idle scheduler has higher LLM cost. Reflection output is stored separately from CBR cases — reflections are `List<String>` (abstract insights) that don't fit the `CbrCase` contract (problem/solution/outcome). Stored as lightweight `ReflectionEntry` records to avoid polluting CBR retrieval results with structurally mismatched entries.
**Sources:** Research §2.2 (LUFY, MemGPT Sleeptime); ContentSummariser; TieredContentSummariser; ReflectionOrchestrator
**Exploration:** deep-analysis
**Status:** revised (R1-07: reflections stored as ReflectionEntry, not CBR cases — avoids retrieval pollution and semantic mismatch with CbrCase contract)

## D11: Eviction policy — composite score threshold (replaces CbrRetentionScheduler)

**Choice:** Compute composite retention score from importance (arousal, surprise), recency (TemporalDecay), scope (ScopeDecay), and trust. Evict below configurable threshold. The orchestrator implements its own scan-and-evict loop using retrieveSimilar + erase — it does NOT use CbrRetentionPolicy.purge(), which cannot express weighted composite scores. For agents that use MemoryHygiene, the orchestrator replaces CbrRetentionScheduler's purge, not composes with it.
**Alternatives:**
- Staged filters (age → importance → capacity) — simpler but less nuanced
- Configurable EvictionStrategy SPI with composite default — adds SPI overhead for a single-strategy concern
- Compose with CbrRetentionPolicy.purge() — impossible because purge uses simple filter criteria (maxAgeDays, maxCasesPerType, minTrustScore), not weighted composite scores
**Rationale:** Single unified score is easier to tune and reason about. All signal types already produce [0,1] factors. Composite = weighted product of factors. Threshold is the one knob operators adjust. Retrieval-time decay (TemporalDecayCbrCaseMemoryStore decorator) and eviction-time decay serve different purposes: retrieval decay modulates ranking; eviction decay determines whether the memory is worth keeping at all.
**Trade-offs:** Weight tuning requires experimentation; no independent stage-level visibility. Agents using MemoryHygiene should disable CbrRetentionScheduler for the same domain to avoid conflicting retention decisions.
**Sources:** TemporalDecay; ScopeDecay; CbrRetentionPolicy; CbrRetentionScheduler; LUFY retention target (<10%); GE-20260804-eb75e0 (scan returns summaries without features — use retrieveSimilar)
**Exploration:** quick
**Status:** revised (R1-01: clarified relationship with neocortex retention — replaces, not composes)

## D12: Temporal versioning — supersession-based

**Choice:** Use existing CbrCaseMemoryStore.supersede(caseId, supersedingCaseId, reason) for temporal versioning. Old memory is marked superseded; new consolidated version replaces it. reinstate() available for rollback.
**Alternatives:**
- Validity windows (validFrom/validTo) — richer bi-temporal model but requires store API changes
- Soft-delete with archive domain — simple but grows storage
**Rationale:** Zero new store API needed. Supersession already provides invalidate-not-delete semantics. SupersessionStatus tracking gives audit trail. reinstate() gives rollback.
**Trade-offs:** No explicit time windows — supersession is binary (active/superseded), not temporal-range-based
**Sources:** CbrCaseMemoryStore.supersede()/reinstate()/getSupersessionStatus(); Research §2.8
**Exploration:** quick
**Status:** captured

## D13: Cross-linking — feature-based write-only annotations

**Choice:** Store cross-links as StringListVal feature values on CbrCase (e.g., "related_cases" → list of caseIds). Cross-links are write-only annotations that enrich consolidated cases — not a navigable graph. During consolidation, the orchestrator records which source cases were merged into the consolidated entry. No reverse traversal (finding "what links TO case B" requires scanning all cases).
**Alternatives:**
- Dedicated MemoryLinkStore SPI — richer graph model with bidirectional traversal, but adds persistence contract consumers must implement
- Supersession chains — limited to parent-child, no peer links
- Comma-separated StringVal — rejected: forces string parsing, doesn't use FeatureValue type system
**Rationale:** No store API change. StringListVal uses the type system correctly. Cross-links serve consolidation context ("these 5 memories were merged into this one"), not knowledge graph navigation. If navigable graph queries become needed, a dedicated link store can be added later without changing the annotation format.
**Trade-offs:** No reverse traversal. Not a Zettelkasten-style navigable graph — scoped to consolidation provenance. Adequate for the memory hygiene use case but not for general knowledge graph queries.
**Sources:** CbrCase.features(); FeatureValue.StringListVal; Research §2.8 (Zettelkasten/A-MEM)
**Exploration:** quick
**Status:** revised (R1-02: StringListVal replaces comma-separated StringVal; scoped to write-only annotations, not navigable graph)

## D14: Integrity checking — IntegrityChecker SPI with hybrid default

**Choice:** IntegrityChecker SPI. Default implementation does structural checks (orphaned supersessions, duplicate caseIds, missing features, unprocessed stale memories — i.e., old memories never reviewed by the hygiene pipeline), flags anomalies, escalates to optional SemanticIntegrityChecker (LLM-backed, @DefaultBean no-op). Consumers override semantic checker to activate.
**Alternatives:**
- Structural checks only — cheaper but misses contradictory memories
- Single strategy with escalation config — simpler but less flexible
- TieredIntegrityChecker — follows TieredContentSummariser pattern but integrity isn't batch-size-dependent
**Rationale:** Structural checks are always cheap and catch data consistency issues. Semantic escalation is opt-in — consumers who need contradiction detection activate it. SPI gives domain-specific flexibility.
**Trade-offs:** Default semantic checker is no-op — consumers must wire LLM-backed implementation to get semantic checks
**Sources:** Research §2.8 (memory integrity/safety); TieredContentSummariser pattern; IntakeClassifier SPI pattern
**Exploration:** quick
**Status:** captured

## D15: Package placement — blocks.memory (top-level)

**Choice:** New top-level package io.casehub.blocks.memory — parallel to blocks.summarisation
**Alternatives:**
- agentic.memory — signals "agentic-specific" but memory hygiene is orthogonal to agentic orchestration; a non-agentic app with CBR could use it
- agentic.personality — conflates memory lifecycle with personality
- agentic.personality.memory — nested sub-package, awkward nesting
**Rationale:** Memory hygiene is reusable infrastructure like summarisation, not an agentic interaction pattern like coalition or intention. blocks.summarisation is already used by both agentic and non-agentic consumers. Memory hygiene has the same profile — it composes neocortex memory APIs, not agentic orchestration primitives.
**Trade-offs:** Breaks the pattern of "all epic #126 types under agentic.*" — but architectural accuracy matters more than issue grouping
**Sources:** Existing package structure: blocks.summarisation (top-level, non-agentic); blocks.agentic.belief/coalition/intention (agentic-specific)
**Exploration:** quick
**Status:** revised (R1-09: moved from agentic.memory to blocks.memory — memory hygiene is infrastructure, not agentic-specific)

---

# UserModel (#122) Design Decisions

## D16: Profile subject identity — any interlocutor

**Choice:** Profile subject is any interlocutor the agent interacts with — human users, other agents, external systems. Keyed by generic string ID triple: `(agentId, subjectId, tenantId)`.
**Alternatives:**
- Other agents only — requires AgentDescriptor for subject, excludes human users which are the primary use case in devtown, clinical, ops, and wacky-manor
- Configurable per domain — SPI for subject identity. Over-engineered: identity is always a string; what varies per domain is profile fields, not subject identification
**Rationale:** Every real consumer's primary use case is modeling human users, not other agents. RelationshipEvent.otherAgentId() is already a generic String — no API changes needed. Agent-to-agent is a natural subset. Avoids the GE-20260811-e941cc pitfall (coupling to a specific identity type that excludes half the consumers).
**Trade-offs:** No access to subject's AgentDescriptor (personality, briefing). If the subject happens to be an agent, this enrichment context is optional — consumers can provide it but it's not required.
**Sources:** RelationshipEvent.java (neocortex-memory-api), ExperienceEvent.java (neocortex-memory-api), GE-20260811-e941cc (AgentDisposition vs DispositionProfile type split), Research §2.4, arXiv:2510.07925
**Exploration:** quick
**Status:** captured

## D17: Synthesis mechanism — tiered (heuristic + LLM)

**Choice:** Tiered synthesis: heuristic fold for well-defined countable dimensions (relationship stage from signal counts, interaction frequency, recency), LLM synthesis for open-ended dimensions (communication style, topics of interest, behavioral patterns).
**Alternatives:**
- LLM synthesis on every tick — flexible but expensive, breaks bounded-cost tick principle established by PersonalityEvolution/InnerLife/MemoryHygiene
- Heuristic fold only — zero LLM cost but too rigid for open-ended dimensions like communication style and topics of interest that require natural language interpretation
**Rationale:** Follows TieredContentSummariser pattern established in blocks. PersonalityEvolution's core loop is pure heuristic (DispositionHealth.probe() is math). MemoryHygiene uses TieredContentSummariser. InnerLife gates heuristic checks before LLM calls. Most ticks will only update counters and check stage thresholds — LLM fires only when enough new textual signal has accumulated. An agent modeling dozens of subjects needs bounded per-tick cost.
**Trade-offs:** Two code paths (heuristic + LLM) are more complex than either alone. LLM synthesis quality depends on accumulated signal volume — sparse signals may produce low-quality synthesis.
**Sources:** TieredContentSummariser (blocks/summarisation), PersonalityEvolutionOrchestrator.tick() (blocks/agentic/personality), InnerLifeOrchestrator.tick() (blocks/agentic/personality), MemoryHygieneOrchestrator.tick() (blocks/memory)
**Exploration:** quick
**Depends on:** D16 (profile subject identity)
**Status:** captured

## D18: Relationship stage model — continuous score + threshold tiers

**Choice:** Maintain a continuous familiarity score [0,1] from accumulated QualitySignals. Configurable thresholds map score ranges to named stages. Score can decay with inactivity. Consumers define their own stage names and thresholds.
**Alternatives:**
- Discrete state machine — explicit transitions (stranger→acquaintance requires N positive signals). Clearer boundaries but rigid, hard to handle regression or domain-specific stages
- Multi-dimensional continuous — separate scores per dimension (trust, familiarity, rapport). Richer but more complex to configure and reason about
**Rationale:** Continuous score is the simplest model that supports regression (score decays with inactivity), domain-specific stages (consumers set their own thresholds and labels), and smooth transitions. Threshold tiers give consumers discrete categories when needed (e.g., "if stage >= FRIEND, use informal tone") without forcing a rigid state machine. Decay naturally handles staleness — inactive relationships regress.
**Trade-offs:** Single dimension may be reductive for complex relationships (you can trust someone professionally but not personally). Multi-dimensional is the upgrade path but adds configuration burden. Start simple.
**Sources:** Research §2.4 (stranger → acquaintance → friend → confidant), Relationship science — perceived partner responsiveness (Smith, Bradbury, Karney, 2025), TemporalDecay (neocortex-memory-api)
**Exploration:** quick
**Depends on:** D17 (tiered synthesis — familiarity score is a heuristic dimension)
**Status:** captured

## D19: Update cadence — event-driven record + periodic tick

**Choice:** record() accumulates signals immediately (cheap counter updates). tick() re-evaluates heuristic dimensions and optionally triggers LLM synthesis when enough new signal has accumulated. Matches PersonalityEvolution's record()+tick() dual-entry pattern. Consumer controls tick frequency.
**Alternatives:**
- Pure event-driven — profile updates on every recorded event. LLM synthesis on every event would be expensive; without LLM, open-ended dimensions never update
- Pure periodic tick — all signals batched, processed on tick. Misses the separation of cheap recording from expensive synthesis. Events between ticks have no immediate effect
**Rationale:** record() is O(1) — increment counters, append to event buffer. No LLM, no store write. tick() amortises the expensive operations (LLM synthesis, CBR store write). This separation is the established pattern across all three prior orchestrators (PersonalityEvolution, InnerLife, MemoryHygiene). The consumer controls both what events to record and how often to tick, matching blocks' "library, not framework" architecture.
**Trade-offs:** Profile is not instantly updated after record() — there's a lag until the next tick. For the use case (long-term relationship modeling), this lag is imperceptible and architecturally correct.
**Sources:** PersonalityEvolutionOrchestrator.record()/tick() (blocks/agentic/personality), InnerLifeOrchestrator.observe()/tick() (blocks/agentic/personality)
**Exploration:** quick
**Depends on:** D17 (tiered synthesis — tick triggers the synthesis)
**Status:** captured

## D20: Profile storage — UserProfileStore SPI backed by CbrCaseMemoryStore

**Choice:** `UserProfileStore` SPI with `store(UserProfile)`, `lookup(agentId, subjectId, tenantId) → Optional<UserProfile>`, `findByAgent(agentId, tenantId) → List<UserProfile>`, `eraseSubject(subjectId, tenantId)`. Default implementation (`CbrUserProfileStore`) backs onto `CbrCaseMemoryStore` internally — CbrCase convention (profile summary as problem, fields as features, supersession for versioning) is contained in the adapter. Consumers get a profile-oriented API; the CbrCase hack is hidden.
**Alternatives:**
- Raw CbrCaseMemoryStore — exposes CbrCase semantics (problem/solution/features) that don't map naturally to profiles. Every consumer must post-filter by producerAgentId AND caseType. Erasure by subjectId is impossible without scanning all cases. The "similarity search across profiles" benefit is secondary analytics, not a core retrieval pattern.
- In-memory only — lost on restart. Too limiting for long-term relationships.
**Rationale:** R1-03 (decision review): the reviewer identified that CbrCase is a semantic misfit for profile storage and proposed the adapter pattern. The primary access pattern is direct lookup by (agentId, subjectId, tenantId) — something a dedicated SPI expresses naturally. Supersession, trend enrichment, and similarity search are preserved in the backing implementation. The `eraseSubject()` method provides the GDPR Art.17 erasure path (R1-05) that raw CbrCaseMemoryStore cannot express.
**Trade-offs:** One additional SPI type (UserProfileStore) and one adapter class (CbrUserProfileStore). Minor cost for significant clarity.
**Sources:** CbrCaseMemoryStore (neocortex-memory-api), R1-03 (decision review finding), R1-05 (GDPR erasure gap), GE-20260820-c19b68 (producerAgentId post-filtering)
**Exploration:** quick
**Depends on:** D16 (profile subject identity — subjectId stored as feature)
**Status:** revised (R1-03: UserProfileStore SPI wrapping CbrCaseMemoryStore, not raw CbrCase exposure; R1-05: eraseSubject for GDPR compliance)

## D21: Profile structure — fixed core + extensible metadata

**Choice:** Core fields defined by UserModel: relationship stage (String), familiarity score (double), interaction count (int), last interaction timestamp, positive/negative signal counts. Open-ended dimensions (topics of interest, communication style, preferences) stored as LLM-synthesized text in a summary field plus domain-specific metadata via the CbrCase features Map.
**Alternatives:**
- Fully extensible ProfileDimension SPI — consumers declare what dimensions to track. Maximum flexibility but over-engineered; every consumer must configure dimensions before anything works
- Fixed schema only — all fields pre-defined. Too rigid: clinical needs different dimensions than gaming
**Rationale:** Fixed core gives consumers reliable, queryable fields for the common needs (relationship stage drives tone, interaction count drives personalization depth). Extensible metadata via features Map lets domains add whatever they need. LLM synthesis produces a free-text summary that captures nuances no fixed schema could anticipate. The CbrCase features Map is already designed for this — FeatureValue supports String, Number, and StringList values.
**Trade-offs:** LLM-synthesized summary is opaque — consumers can't query specific sub-fields without parsing. Mitigation: the LLM can be prompted to produce structured JSON that's stored as a StringVal feature, giving partial queryability.
**Sources:** FeatureValue (neocortex-memory-api), UserProfileSchema concept, HeuristicMessageSummariser precedent (fixed structure from message metadata), Research §2.4 (profile schema extensibility)
**Exploration:** quick
**Depends on:** D17 (tiered synthesis — core fields from heuristics, open-ended from LLM), D20 (CBR storage — features Map for extensibility)
**Status:** captured

## D22: Package placement — blocks.agentic.social

**Choice:** All three social cognition orchestrators live in `io.casehub.blocks.agentic.social`: PersonalityEvolution (self-model), InnerLife (self-expression), UserModel (other-model).
**Alternatives:**
- blocks.agentic.personality — captures only the first orchestrator's concern. "Personality" doesn't describe user modeling or proactive initiation.
- blocks.user (new top-level) — signals infrastructure not agentic-specific, but UserModel is specifically about agent social cognition
- blocks.agentic.user (new sub-package) — fragments related concepts across multiple sub-packages
**Rationale:** R1-04 (decision review): no code exists in `blocks.agentic.personality` yet — all three orchestrators are being built on this branch. Zero cost to choose the right name from the start. The research doc §3.3 groups these patterns under "Conversation & Social." `social` captures the triad: how the agent changes (PersonalityEvolution), when it speaks (InnerLife), and what it knows about others (UserModel).
**Trade-offs:** None — the package doesn't exist yet, so there's nothing to rename.
**Sources:** PersonalityEvolutionOrchestrator, InnerLifeOrchestrator (both on this branch, not yet merged), R1-04 (decision review), Research §3.3, D15 precedent (MemoryHygiene chose top-level because it's domain-neutral)
**Exploration:** quick
**Depends on:** D16 (profile subject identity)
**Status:** revised (R1-04: renamed from blocks.agentic.personality to blocks.agentic.social — correct name at zero cost since no code is merged)

---

# MentalModel (#123) Design Decisions

## D23: BDI scope — full Beliefs + Desires + Intentions

**Choice:** Track all three BDI dimensions per actor: beliefs (what they know/think), desires (what they want), and intentions (what they plan to do).
**Alternatives:**
- Beliefs + goals only — skip intentions as too opaque. Loses the planning integration that makes ToM valuable for GOAP enrichment.
- Beliefs only — simplest, grounded in existing BeliefSet<T>. Defers the desires/intentions dimensions that academic literature (ToMA, DPT-Agent) shows are critical for social reasoning.
**Rationale:** The research consistently shows all three BDI dimensions are needed for effective ToM. Desires and intentions are harder to infer but still informative — even low-confidence inferences about what someone wants shape agent behavior (e.g., "user seems to want quick resolution" → prioritise conciseness). Confidence decay handles uncertainty: low-confidence inferences fade fast without reinforcing signals.
**Trade-offs:** Desires and intentions are speculative. Unlike beliefs (which can be explicitly stated and verified), D+I rely heavily on LLM inference quality. Mitigated by confidence scoring and decay.
**Sources:** Research §2.5, ToMA (ACL 2026), DPT-Agent (2025), Beyond Words (ACL 2025)
**Exploration:** quick
**Status:** captured

## D24: Inference mechanism — tiered (heuristic + LLM)

**Choice:** Two-tier inference: heuristic extraction for explicit signals ("I think X", "I want Y", "I plan to Z") + LLM inference for implicit signals (tone, context, behavioral patterns).
**Alternatives:**
- LLM-only inference — all BDI dimensions inferred by LLM. Simpler code but no heuristic fast-path; every tick requires LLM call. Breaks bounded-cost tick principle.
- Rule-based extraction only — pattern-matching on speech acts. Cheapest but misses the implicit signals that make ToM valuable (people rarely state their mental states explicitly).
**Rationale:** Follows the tiered pattern established by UserModel (D17): heuristic fold for well-defined countable dimensions, LLM for open-ended interpretation. Most ticks update counters and apply decay — LLM fires only when enough new signal has accumulated. Heuristic extraction catches explicit cues at zero LLM cost.
**Trade-offs:** Two code paths. Heuristic extraction can be noisy (false positives from figurative language). LLM inference quality varies with model capability.
**Sources:** UserModel D17 (tiered synthesis), TieredContentSummariser pattern, ToMA (ACL 2026), InnerLifeOrchestrator (heuristic gates before LLM)
**Exploration:** quick
**Depends on:** D23 (full BDI scope)
**Status:** captured

## D25: Input model — signal-based record()+tick()

**Choice:** Define MentalStateSignal as a sealed interface for domain events that carry cognitive cues. Consumers construct signals from their domain events and call record(). The orchestrator accumulates signals and processes on tick().
**Alternatives:**
- Message observer — MentalModel implements MessageObserver and directly observes channel messages. Tighter coupling to qhorus conversation infrastructure; excludes non-conversation signals (e.g., action outcomes, behavioral observations).
- Conversation state reader — reads ConversationState snapshots on tick. Requires an active conversation; can't model actors observed outside conversations.
**Rationale:** Consistent with the social cognition triad pattern. UserModel uses InteractionSignal (sealed interface with RelationshipSignal, ExperienceSignal, CustomSignal variants). PersonalityEvolution uses BehavioralSignal. record() is O(1) — no LLM, no store write. Decouples MentalModel from qhorus conversation infrastructure. Any domain event that carries cognitive cues (conversation messages, action outcomes, behavioral observations) can be wrapped as a MentalStateSignal.
**Trade-offs:** Consumers must translate their domain events into MentalStateSignals. Extra mapping layer, but this is the established blocks pattern and provides maximum composability.
**Sources:** InteractionSignal (blocks/agentic/social), BehavioralSignal (blocks/agentic/social), PersonalityEvolutionOrchestrator.record()/tick(), UserModelOrchestrator.record()/tick()
**Exploration:** quick
**Depends on:** D23 (full BDI scope), D24 (tiered inference)
**Status:** captured

## D26: GOAP integration — confidence-aware projection

**Choice:** MentalModel exposes a project(agentId, subjectId, tenantId) → List<MentalProjection> method. Each MentalProjection carries: conditionKey (String), value (boolean), confidence (double [0,1]), and source (BDI dimension enum). Consumers decide their own boolean threshold when mapping to GoapWorldState (e.g., only project conditions with confidence ≥ 0.5). Integration uses the current GoapWorldState API: `new GoapWorldState(conditions)` or chaining `.with(key, value)`.
**Alternatives:**
- Map<String, Boolean> projection — collapses confidence to boolean. Discards the uncertainty information that is MentalModel's core value (R1-05). A confidence-0.3 and confidence-0.9 inference become identical.
- Planning-aware orchestrator — MentalModel takes GoapAction definitions and reasons about action appropriateness. Tight coupling to engine GOAP API. Violates blocks scope.
- Defer GOAP integration — build MentalModel standalone. Misses the primary value proposition.
**Rationale:** Loose coupling via a projection method. MentalModel produces projections with confidence; consumers filter and map to their GoapWorldState. The projection is a pure function of current BDI state — no side effects, no GOAP dependency. Confidence-aware projection preserves the uncertainty that D27's confidence decay creates, letting consumers make threshold decisions at the point of consumption.
**Trade-offs:** Condition naming is consumer-defined — no standard vocabulary. MentalModel provides suggested condition names based on BDI dimensions, but consumers may need domain-specific mappings. Consumers must write the confidence→boolean threshold logic.
**Sources:** GoapWorldState (engine-api, record with Map<String, Boolean> conditions + .with() method), GoapPlanner (engine-api), Research §2.5 (GOAP integration depth)
**Exploration:** quick
**Depends on:** D23 (full BDI scope), D27 (confidence decay — confidence propagated to projection)
**Status:** revised (R1-05: Map<String, Boolean> → List<MentalProjection> with confidence; R1-04: fixed GoapWorldState API references to current API)

## D27: Temporal dynamics — confidence decay with entrenchment

**Choice:** Each attributed belief, desire, and intention carries a confidence score [0,1] that decays with time since last reinforcing signal. Beliefs additionally use BeliefSet entrenchment ordering for AGM-style revision. Configurable decay rate per BDI dimension (beliefs decay slowly, desires/intentions decay faster). Entries below a configurable floor are evicted.
**Alternatives:**
- Versioned snapshots — full timestamped history of mental state changes. Richer temporal queries but high storage cost for volatile state.
- Current state only — latest inferred state, no decay. Simplest but stale inferences persist indefinitely; an agent might act on a belief the subject held three months ago.
**Rationale:** Confidence decay naturally handles the core ToM challenge: we can never be certain about others' mental states, and our certainty decreases with elapsed time. TimeToM (ACL 2024) emphasizes temporal reasoning as critical for ToM accuracy. Entrenchment in BeliefSet provides formal revision ordering — deeply entrenched beliefs (reinforced by multiple signals) resist revision by contradictory evidence, which matches how real belief attribution works.
**Trade-offs:** Decay rate tuning requires experimentation. Too aggressive = useful inferences vanish too quickly. Too slow = stale state persists. Mitigated by separate decay rates for B, D, and I (beliefs are more stable than desires, which are more stable than intentions).
**Entrenchment vs confidence interaction (R1-10):** These are separate dimensions serving different purposes. Entrenchment (int) = revision resistance during AGM belief revision. Increases each time a reinforcing signal arrives. Used only by BeliefSet.revise() to determine which beliefs survive when the set is inconsistent. Confidence (double [0,1]) = temporal freshness. Decays with elapsed time since last reinforcing signal. Used for eviction (below floor → remove regardless of entrenchment) and projection weighting (higher confidence → stronger MentalProjection). They don't conflict: a belief with high entrenchment and low confidence was well-established historically but hasn't been reinforced recently — it gets evicted. Entrenchment only matters when new contradictory evidence forces revision between beliefs that are both above the confidence floor.
**Sources:** TimeToM (ACL 2024), BeliefSet.revise() (blocks/agentic/belief), TemporalDecay (neocortex-memory-api), UserModel D18 (familiarity decay)
**Exploration:** quick
**Depends on:** D23 (full BDI scope)
**Status:** revised (R1-10: clarified entrenchment vs confidence interaction — separate dimensions, eviction trumps entrenchment below confidence floor)

## D28: Persistence — MentalModelStore SPI

**Choice:** MentalModelStore SPI with store(MentalModelSnapshot), lookup(agentId, subjectId, tenantId) → Optional<MentalModelSnapshot>, findByAgent(agentId, tenantId) → List<MentalModelSnapshot>, eraseSubject(subjectId, tenantId). Default implementation backed by CbrCaseMemoryStore (same adapter pattern as CbrUserProfileStore). Beliefs stored as features; desires and intentions as JSON feature values.
**Alternatives:**
- In-memory with optional persist — primary state is ConcurrentHashMap, persistence is callback. State lost on restart, which is unacceptable for long-lived agents tracking relationship context over months.
- RelationshipEvent storage — record mental state observations as RelationshipEvents. No separate store. Reconstruction from history is expensive (scan + replay) and loses confidence/entrenchment metadata.
**Rationale:** Follows UserProfileStore (D20) pattern exactly. Dedicated SPI hides CbrCase impedance mismatch. The primary access pattern is direct lookup by (agentId, subjectId, tenantId). eraseSubject() provides GDPR compliance (established by D20). CbrCaseMemoryStore gives similarity search, supersession, and feature-based retrieval for free.
**Trade-offs:** One more SPI type + adapter class. Same cost as UserProfileStore; same clarity benefit.
**Sources:** UserProfileStore/CbrUserProfileStore (D20), CbrCaseMemoryStore (neocortex-memory-api), GDPR Art.17 erasure (D20 R1-05)
**Exploration:** quick
**Depends on:** D23 (full BDI scope), D27 (confidence decay — confidence stored as feature)
**Status:** captured

## D29: Orchestrator architecture — unified with per-subject BDI state

**Choice:** Single MentalModelOrchestrator with per-subject state holding all three BDI dimensions. ConcurrentHashMap<String, SubjectMentalState> + ReentrantLock per subject for tick concurrency. record()+tick()+project() public API.
**Alternatives:**
- Decomposed per-dimension — three trackers (BeliefTracker, DesireTracker, IntentionTracker) composed by a facade. More flexible but creates artificial boundaries between interdependent dimensions. Coordination overhead for cross-dimension consistency.
- Belief-centric with derived D+I — BeliefSet as primary, D+I derived by LLM from beliefs each tick. Simpler state but loses explicitly stated desires/intentions; D+I quality entirely depends on LLM.
**Rationale:** BDI dimensions are deeply interconnected: beliefs inform desires (knowing someone is stressed → inferring they want resolution), desires inform intentions (wanting resolution → planning to escalate). A unified state container lets the tiered inference engine reason across all three dimensions coherently. Follows the UserModelOrchestrator precedent (single orchestrator, per-subject state, record+tick).
**Trade-offs:** Single orchestrator is more complex internally than separate trackers. Accepted because the alternative (coordination between three trackers maintaining cross-dimension consistency) is harder to get right.
**Sources:** UserModelOrchestrator (blocks/agentic/social), PersonalityEvolutionOrchestrator (blocks/agentic/social), DPT-Agent (2025, unified BDI reasoning)
**Exploration:** quick
**Depends on:** D23 (full BDI), D24 (tiered inference), D25 (signal-based input)
**Status:** captured

## D30: MentalModel↔UserModel boundary

**Choice:** MentalModel and UserModel are complementary, not overlapping. UserModel tracks **stable behavioral traits** (communication style, preferences, topics of interest — slow-changing, identity-level). MentalModel tracks **dynamic cognitive state** (current beliefs, desires, intentions — fast-changing, situation-level). Both are keyed by (agentId, subjectId, tenantId) but serve different consumer needs and update at different cadences.
**Alternatives:**
- MentalModel subsumes UserModel's LLM synthesis — MentalModel produces the open-ended profile dimensions (communication style, topics) that UserModel currently generates. Eliminates duplicate LLM calls. But: UserModel is already implemented with 44 tests. Redesigning its synthesis pipeline introduces regression risk for no user-facing benefit. The boundary is also conceptually clean: "how they communicate" (UserModel) vs "what they currently think/want" (MentalModel).
- Shared LLM synthesis — both orchestrators share a single LLM call that produces both profile updates and BDI inferences. Reduces LLM cost but tightly couples two orchestrators that should evolve independently.
**Rationale:** The semantic boundary is genuine. "User prefers concise communication" (UserModel — stable trait, persists across contexts) is different from "User currently believes the deployment is risky" (MentalModel — situational belief, may change after a reassuring explanation). Consumers query different profiles for different purposes: UserModel for tone/style adaptation, MentalModel for action planning. Future consolidation is possible via a shared inference engine, but premature now.
**Trade-offs:** Potential duplicate LLM interpretation of the same conversation signals. Consumers calling both record() methods for the same event do pay twice. Accepted because the inference targets are different (behavioral patterns vs cognitive state) and each orchestrator's LLM prompt is tuned for its purpose.
**Sources:** UserModelOrchestrator (blocks/agentic/social), D16-D22 (UserModel decisions), R1-06 (decision review finding)
**Exploration:** quick (surfaced by R1-06)
**Depends on:** D23 (full BDI scope), D25 (signal-based input)
**Status:** captured

## D31: Per-orchestrator signal types (not unified)

**Choice:** MentalModel uses its own MentalStateSignal sealed interface, consistent with the per-orchestrator signal pattern. No unified SocialObservation bus.
**Alternatives:**
- Unified SocialObservation — single event type published to an EventStreamBus, all social cognition orchestrators subscribe. Reduces consumer integration from 4 record() calls to 1 publish(). But: each orchestrator extracts fundamentally different data from the same event (trait activation vs quality signals vs cognitive cues). A unified type becomes a kitchen-sink carrying all possible fields. The consumer saves one line but the orchestrators each need parsing/filtering logic for irrelevant fields.
- Adapter layer — consumers write one SocialObservation, adapters translate to per-orchestrator signals. Adds an abstraction layer without reducing total code.
**Rationale:** The 4 signal types extract orthogonal information. PersonalityEvolution cares about signal magnitude and polarity for trait activation. UserModel cares about quality signal counts for familiarity scoring. MentalModel cares about cognitive content (what was said/implied about beliefs/desires/intentions). Forcing these into one type loses the domain-specific contract each signal provides. The consumer cost (4 record() calls, each O(1)) is manageable. The orchestrator benefit (typed, validated input) outweighs it.
**Trade-offs:** Consumer writes 4 mapping functions per domain event type. N×M integration cost. Acceptable for the pattern library; consumers who find this onerous can write their own thin adapter.
**Sources:** InteractionSignal (blocks/agentic/social), EventStreamBus pattern (blocks/summarisation), R1-07 (decision review finding)
**Exploration:** quick (surfaced by R1-07)
**Depends on:** D25 (signal-based input)
**Status:** captured

---

# StrategyLearning (#124) Design Decisions

## D32: Reflection scope — per-agent output, per-subject evidence

**Choice:** StrategyLearning produces a per-agent StrategyProfile (global strategy defaults) but accumulates per-subject engagement evidence in CBR cases (subjectId as a feature). Periodic reflection analyzes both per-subject patterns ("humor works with User X") and cross-subject patterns ("I'm too verbose generally").
**Alternatives:**
- Pure per-agent (no per-subject data) — simpler but can't learn subject-specific strategies. Cross-subject aggregation loses the signal about what works with whom.
- Pure per-subject (like UserModel/MentalModel) — creates 5th per-subject orchestrator. The metacognitive value ("what works globally") is lost; that requires cross-subject analysis.
**Rationale:** The cognitive operation is "evaluate + learn from feedback." Per-subject evidence is essential (engagement varies by person), but the actionable output is a global strategy with per-subject annotations. The profile provides defaults; textual guidelines can include per-subject insights.
**Trade-offs:** CBR cases carry subjectId as a feature requiring post-filtering for per-subject analysis (GE-20260820-c19b68). Acceptable for periodic batch analysis.
**Sources:** Research §2.6, UserModelOrchestrator.findByAgent() pattern, GE-20260820-c19b68 (producerAgentId post-filtering)
**Exploration:** quick
**Status:** captured

## D33: Orchestrator API — record() + tick() + reflect()

**Choice:** Single StrategyLearningOrchestrator with three entry points: record() accumulates engagement signals (O(1)), tick() runs cheap tiers 1-2 (heuristic scoring + conversation case storage), reflect() runs expensive tier 3 (ReflectionOrchestrator + TrendAnalyzer + LLM synthesis → StrategyProfile update). Consumer calls tick() frequently and reflect() periodically at their chosen cadence.
**Alternatives:**
- Single tick() for all three tiers — pushes tier 3 cost into the tick loop. Contradicts MemoryHygiene precedent (D9) which separated tick() from maintain() for exactly this reason.
- Two separate classes (orchestrator + scheduler) — consistent with MemoryHygiene but adds a type when the composition is simpler (reflect() composes ReflectionOrchestrator + TrendAnalyzer directly, unlike maintain() which composes orchestrator + reflection + cross-linking + integrity).
- Three-tier explicit API — breaks established pattern.
**Rationale:** tick() + reflect() on the same class follows the MemoryHygiene principle ("tick is cheap, maintain is expensive") without the class split. Tier 3 composes external services (ReflectionOrchestrator, TrendAnalyzer) like MemoryHygieneScheduler.maintain(), but the composition is simpler — no cross-linking or integrity checks — so a separate scheduler class adds cost without benefit.
**Trade-offs:** Two consumer-facing cadence decisions (tick frequency, reflect frequency) instead of one. Acceptable; MemoryHygiene has the same dual-cadence pattern.
**Sources:** PersonalityEvolutionOrchestrator.record()/tick(), MemoryHygieneOrchestrator.tick() + MemoryHygieneScheduler.maintain() (dual-cadence precedent), MentalModelOrchestrator.record()/tick()/project()/observeConversation() (multi-entry-point precedent)
**Exploration:** quick
**Status:** revised (R1-06: tick()+reflect() split follows MemoryHygiene dual-cadence principle)

## D34: Output model — StrategyProfile with guidelines + dimensions

**Choice:** StrategyProfile record containing: ranked textual guidelines (List<String>, LLM-generated), dimensional scores (Map<String, Double>, [0,1] per dimension), last update timestamp, evidence count (how many CBR cases informed this profile), and extensible metadata. Consumers inject guidelines into agent system prompts via SystemPromptCustomiser or read dimensional scores directly.
**Alternatives:**
- Parameterized dimensions only (no text) — simpler but not actionable in prompts. Consumers must map numbers to prompt fragments.
- CBR case library only (no profile) — situational retrieval, no stable baseline. Rich but no persistent strategy.
**Rationale:** Combines structured dimensions (queryable, configurable) with LLM-generated text (actionable in prompts). Parallels UserProfile's structure: fixed core fields + LLM-synthesized summary.
**Trade-offs:** LLM synthesis is expensive — only runs on periodic tier. Guidelines may become stale between periodic ticks. Dimensional scores update on cheaper tiers.
**Sources:** UserProfile (blocks/agentic/social), SystemPromptCustomiser (blocks/prompt), Research §2.6
**Exploration:** quick
**Status:** captured

## D35: Signal type — EngagementSignal with dimensional snapshot

**Choice:** EngagementSignal sealed interface with variants: TurnOutcome(EngagementEvent event, Map<String, Double> dimensionalSnapshot, String responseExcerpt, String subjectId) — per-turn engagement data with the agent's dimensional scores at time of response; ConversationOutcome(String conversationId, String subjectId, String conversationSummary, int turnCount) — conversation-boundary marker for tier-2 analysis. Follows D31 per-orchestrator signal pattern.
**Alternatives:**
- Free-text strategyUsed: String — requires causal attribution the consumer can't reliably provide (R1-03). Is the response "formal" or "verbose"? Subjective, noisy. Labels would be unreliable.
- Raw EngagementEvent passthrough — no strategy context at all. The orchestrator sees outcomes but not what conditions produced them.
- LLM-inferred strategy from responseExcerpt — extra LLM call per turn, circular dependency.
**Rationale:** Dimensional snapshot provides reliable correlation data without causal attribution. "When verbosity was 0.7 and formality was 0.3, engagement was X" is objective and machine-readable. TrendAnalyzer detects patterns (SLOPE, DELTA) across snapshots correlated with engagement metrics. The LLM tier 3 reasons about correlations, not labels. The consumer reads current StrategyProfile dimensions and passes them as the snapshot — zero subjectivity, zero extra cost.
**Trade-offs:** Correlation ≠ causation. The orchestrator learns "high verbosity correlates with low engagement" but can't prove verbosity caused it. Mitigated by TrendAnalyzer detecting trends across multiple observations, and LLM synthesis reasoning about confounds.
**Sources:** EngagementEvent (neocortex-memory-api), InteractionSignal pattern (blocks/agentic/social), MentalStateSignal pattern (blocks/agentic/social), D31, R1-03 (decision review: causal attribution gap)
**Exploration:** quick
**Depends on:** D31 (per-orchestrator signal types), D38 (dimensional scores provide the snapshot values)
**Status:** revised (R1-03: replaced free-text strategyUsed with Map<String, Double> dimensionalSnapshot for objective correlation data)

## D36: Storage — StrategyStore SPI backed by CbrCaseMemoryStore

**Choice:** StrategyStore SPI with store(StrategyProfile), lookup(agentId, tenantId) → Optional<StrategyProfile>, subjectInsights(agentId, subjectId, tenantId) → List<String>, eraseAgent(agentId, tenantId). Default CbrStrategyStore backed by CbrCaseMemoryStore. Profile lookup is per-agent (no subjectId). subjectInsights() queries engagement CBR cases filtered by subjectId feature, extracts per-subject strategy insights. Same adapter pattern as CbrUserProfileStore (D20) and CbrMentalModelStore (D28).
**Alternatives:**
- In-memory only — profiles lost on restart. Unacceptable for learning that accumulates over weeks/months.
- Direct CbrCaseMemoryStore — exposes CbrCase semantics. Already rejected in D20 and D28.
- No per-subject retrieval (original D36) — leaves per-subject strategy gap (R1-08). The StrategyProfile is global; consumers at interaction time need "what works with User X" without manually querying CBR.
**Rationale:** Same pattern as the two prior stores, plus subjectInsights() to close the per-subject retrieval gap identified in R1-08. The method is a convenience query — internally filters engagement CBR cases by subjectId feature and returns summary insights. Placeholder solution field per GE-20260820-d4e011.
**Trade-offs:** subjectInsights() requires scanning CBR cases with post-filtering (GE-20260820-c19b68). Acceptable for per-interaction retrieval on bounded case sets.
**Sources:** UserProfileStore/CbrUserProfileStore (D20), MentalModelStore/CbrMentalModelStore (D28), GE-20260820-d4e011 (non-blank solution), GE-20260820-c19b68 (producerAgentId post-filtering), R1-08 (per-subject retrieval gap)
**Exploration:** quick
**Depends on:** D32 (per-agent scope + per-subject evidence)
**Status:** revised (R1-08: added subjectInsights() for per-subject strategy retrieval)

## D37: Three-tier reflection mechanics — tick() + reflect() split

**Choice:** tick() runs tiers 1-2 (cheap). reflect() runs tier 3 (expensive). Separated per revised D33.
- Tier 1 (per-response, every tick with new signals): Update aggregate engagement counters — signals processed, engagement rate (responded/total), mean sentiment trend, mean response length. NO dimensional score adjustment — tier 1 cannot attribute engagement changes to specific dimensions (R1-05). Zero LLM cost.
- Tier 2 (per-conversation, on ConversationOutcome signal or threshold): Two separate operations: (a) structured feature extraction — aggregate EngagementEvent fields directly into numeric features (avgResponseLength, continuationRate, meanSentimentShift, avgDimensionalSnapshot per dimension); (b) optional text summary via ContentSummariser<EngagementSignal> stored as StringVal feature. Assembled into a CbrCase and stored via CbrCaseMemoryStore. Builds the per-subject evidence base.
- Tier 3 (periodic, in reflect()): Full reflection pipeline. ReflectionOrchestrator.reflect() over accumulated memories. TrendAnalyzer on engagement CBR case history — requires CbrFeatureSchema with TimeSeries fields (spec will define schema). LLM synthesis correlates dimensional snapshots with engagement outcomes to produce updated StrategyProfile (new guidelines, revised dimensional scores). Stores profile via StrategyStore.
**Alternatives:**
- SummarisationRunner pipeline for tier aggregation — overhead disproportionate to engagement signal volume.
- Tier 1 adjusts dimensional scores heuristically — can't attribute engagement to dimensions from EngagementEvent metrics alone (R1-05). Only responseLength maps to verbosity; others are outcome metrics, not strategy indicators.
**Rationale:** Tier 1 accumulates data (cheap counters). Tier 2 builds evidence (structured features + optional text). Tier 3 reasons about evidence (LLM correlation analysis + trend detection). Each tier's cost matches its cadence. Structured feature extraction and text summarisation are separate operations (R1-04) — ContentSummariser produces text, not numeric aggregates.
**Trade-offs:** Tier 1 produces no dimensional insight — all dimensional adjustment is deferred to tier 3's LLM. Agents between periodic reflections operate on stale strategy. Accepted because dimensional adjustment from heuristics alone is unreliable.
**Sources:** UserModelOrchestrator.doTick() (tiered pattern), ContentSummariser (blocks/summarisation), ReflectionOrchestrator (neocortex-memory-api), TrendAnalyzer (neocortex-memory-api), MemoryHygieneScheduler.maintain() (reflection composition), R1-04 (separate summarisation from feature extraction), R1-05 (tier 1 cannot adjust dimensions)
**Exploration:** quick
**Depends on:** D33 (tick+reflect split), D35 (EngagementSignal with dimensional snapshot and ConversationOutcome)
**Status:** revised (R1-04: separate text summarisation from structured features; R1-05: tier 1 only updates counters, not dimensions; R1-06: tier 3 moved to reflect())

## D38: Strategy dimensions — fixed core + extensible metadata

**Choice:** Five core dimensions: verbosity (concise↔elaborate), formality (casual↔formal), initiative (reactive↔proactive), directness (indirect↔direct), questionRate (statements↔questions). Each double [0,1], default 0.5 (neutral). Consumers add domain-specific dimensions via CbrCase features on the backing store.
**Alternatives:**
- Fully configurable dimension set — maximum flexibility but every consumer must configure.
- Open-ended only (no numerics) — simple but not queryable.
**Rationale:** Same rationale as UserProfile D21 (fixed core for reliable queryable fields, extensible metadata for domain-specific needs). Five dimensions are universal communication strategy axes grounded in pragmatics literature. LLM reflection adjusts these based on engagement trend analysis.
**Trade-offs:** Five dimensions may be reductive — some strategies (humor, empathy) don't map cleanly to a single axis. Mitigated by textual guidelines capturing what dimensions can't.
**Sources:** Conversational pragmatics (politeness theory, speech act theory), Research §2.6 (strategy dimensions), UserProfile D21 (fixed core + extensible pattern)
**Exploration:** quick
**Depends on:** D34 (StrategyProfile output model)
**Status:** revised (R1-09: corrected theoretical grounding from CAT to conversational pragmatics broadly)

## D39: Package placement — blocks.agentic.social

**Choice:** io.casehub.blocks.agentic.social, alongside PersonalityEvolution, InnerLife, UserModel, MentalModel.
**Alternatives:** None worth exploring — StrategyLearning is social cognition (metacognitive self-improvement of interaction strategies).
**Rationale:** Fifth member of the social cognition package: self-model (PersonalityEvolution), self-expression (InnerLife), other-model (UserModel), other-mind (MentalModel), self-improvement (StrategyLearning).
**Trade-offs:** None.
**Sources:** D22 (package placement precedent)
**Exploration:** quick
**Depends on:** D32 (scope is social cognition)
**Status:** captured

## D40: Tick/reflect outcomes — hierarchical sealed interfaces

**Choice:** Two sealed outcome types matching the tick()/reflect() split:
- StrategyLearningTick (from tick()): NoChange(String reason) — nothing to do; Observed(int signalsProcessed, double engagementRate, double meanSentiment) — tier 1 ran, counters updated; Learned(int signalsProcessed, double engagementRate, double meanSentiment, List<String> conversationsStored, int casesStored) — tier 2 ran, includes tier 1 data. Hierarchical: Learned ⊃ Observed ⊃ NoChange.
- StrategyReflection (from reflect()): NoChange(String reason) — insufficient data; Reflected(StrategyProfile profile, List<String> newGuidelines, TrendProfile trends, int evidenceCases) — full reflection ran. Binary: either reflection ran or it didn't.
**Alternatives:**
- Four mutually exclusive variants on a single sealed interface — tiers are not mutually exclusive (R1-02). Tier 1+2 can fire in the same tick. A "highest tier wins" approach loses lower-tier data.
- Single record with nullable fields — loses type safety.
**Rationale:** Hierarchical sealed interface follows UserModelTick (Synthesised ⊃ Updated ⊃ Unchanged) and MentalModelTick (Inferred ⊃ Updated ⊃ Unchanged) patterns. Each higher variant subsumes lower-tier data. Separate StrategyReflection type for reflect() keeps the types clean — reflect() is a different entry point with different data.
**Trade-offs:** Two outcome types instead of one. Accepted because tick() and reflect() are different operations producing different data shapes.
**Sources:** UserModelTick (hierarchical: Synthesised ⊃ Updated ⊃ Unchanged), MentalModelTick (hierarchical: Inferred ⊃ Updated ⊃ Unchanged), R1-02 (multi-tier execution)
**Exploration:** quick
**Depends on:** D33 (tick+reflect split), D37 (three-tier mechanics)
**Status:** revised (R1-02: hierarchical variants replace mutually exclusive; split into tick/reflect outcome types)

## D41: Boundary with UserModel — perceive vs learn-and-adapt

**Choice:** StrategyLearning owns ALL strategy learning — both per-subject engagement evaluation and per-agent metacognition. UserModel stays purely observational (perceive). Clean cognitive boundary: UserModel = "what IS this person like" (descriptive). StrategyLearning = "what WORKS with this person and in general" (prescriptive). No overlap, no changes to UserModel.
**Alternatives:**
- Complementary layers with gap — UserModel = per-subject observation, StrategyLearning = per-agent global. Leaves per-subject strategy uncovered — nobody learns "humor works with User X."
- Extend UserModel for per-subject strategy — adds engagement processing and strategy output to UserModel. Blurs the perception/prescription boundary. UserModel's LLM prompt asks "how does this person communicate?" (observation); adding "what works with this person?" (strategy) makes it do two different cognitive operations.
**Rationale:** First-principles cognitive decomposition: perceive (observe traits/state) and learn (evaluate outcomes, extract patterns) are distinct operations. UserModel's LLM synthesis asks about the subject's behavior — descriptive. StrategyLearning's LLM synthesis asks what approach produces good engagement — prescriptive. Merging them conflates observation with optimization. The per-subject strategy gap is filled by StrategyLearning's CBR cases (subjectId as feature), not by extending UserModel.
**Trade-offs:** Two orchestrators read interaction data for different purposes. Consumers record signals to both. Acceptable per D31 (per-orchestrator signal types are orthogonal). Per-subject strategy retrieval is via StrategyStore.subjectInsights() (revised D36, R1-08) — queries engagement CBR cases by subjectId, returns per-subject strategy insights.
**Sources:** UserModelOrchestrator (blocks/agentic/social), D30 (MentalModel↔UserModel boundary), D31 (per-orchestrator signal types), fork analysis of UserModel LLM synthesis prompt
**Exploration:** deep-analysis (first-principles fork + code analysis)
**Depends on:** D32 (per-agent output scope), D35 (signal carries subjectId)
**Status:** captured
