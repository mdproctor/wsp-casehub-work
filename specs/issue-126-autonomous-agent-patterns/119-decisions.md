## D1: Output contract

**Choice:** Initiation decision + content
**Alternatives:**
- Raw thoughts only — returns List<InnerThought> with motivation scores. Maximum flexibility but pushes dispatch complexity to every consumer.
- Full message dispatch — InnerLife posts directly via MessageDispatcher. Tighter coupling to qhorus, removes consumer control over dispatch timing and channel selection.
- Three-variant sealed type (Silent, Reflecting, Initiated) — Reflecting carries no actionable information for consumers; both Silent and Reflecting result in the same consumer action (do nothing).
**Rationale:** tick() returns a sealed type with two variants: Silent (no motivation or content) and Initiated(content, channelHint, motivationScore). The consuming app decides how to dispatch — post to channel, queue for human review, or discard. This matches blocks' role as a library (not a framework). The sealed type gives consumers exhaustive pattern matching where every variant maps to a distinct consumer action. The channelHint in Initiated is the LLM's suggestion based on affordance context provided by the consumer via AffordanceRenderer — the LLM reasons about channel names in observation text and suggests a target. No independent channel topology dependency.
**Trade-offs:** Consumers must wire dispatch logic. Accepted because dispatch is domain-specific (Discord bot posts differently from an email agent).
**Sources:** SummarisationRunner.tick() (blocks), PersonalityEvolutionOrchestrator.EvolutionTick (blocks, #118), Liu et al. CHI 2025 (proactive agents with inner thoughts)
**Exploration:** quick
**Status:** revised (R1-02: removed Reflecting variant — non-actionable for consumers; R1-04: clarified channelHint source as affordance-derived)

## D2: Motivation model

**Choice:** LLM motivation scoring via AgentProvider.invoke()
**Alternatives:**
- LLM-scored only — always asks LLM to score motivation. Natural but wastes calls during quiet periods. Addressed by D3 and D4 as pre-filters.
- Heuristic only — scores on observation count, time since last spoke, novelty. No LLM cost but less context-aware.
- Hybrid two-phase within D2 (Phase 1 heuristic + Phase 2 LLM) — original design. Created overlap: "minimum time gap" duplicated in D4's civility constraints, and "minimum observations" overlapped with D3's content quality gate. Replaced by clean pipeline separation.
**Rationale:** D2 is the LLM evaluation step in the pipeline: D4 (civility gate) → D3 (content quality gate) → D2 (LLM scoring). The LLM scores motivation 0.0–1.0 using structured output from AgentProvider.invoke() — the standard blocks pattern (LlmAgentRoutingStrategy, LlmContentSummariser, LlmDecomposition all use invoke() with structured output). No new AgentProvider method required. The prompt includes accumulated observations, affordance context, reflections, and personality. Above configurable threshold → Initiated. Below → Silent. Default thresholds are conservative ("erring toward silence") but fully configurable per-agent — domains requiring more proactive behaviour tune toward lower thresholds. Runtime adaptivity (thresholds adjusting based on engagement outcomes) is the concern of the Strategy Learning pattern (§2.6), not D2.
**Trade-offs:** Every tick that passes D4 and D3 gates incurs an LLM call. The conservative defaults in D4 and D3 ensure this happens infrequently during quiet periods.
**Sources:** Deng et al. AAAI 2025 (intelligence, adaptivity, civility triad), Liu et al. CHI 2025 (motivation threshold), AgentProvider.invoke() (platform-agent-api), LlmAgentRoutingStrategy (blocks)
**Exploration:** quick
**Depends on:** D1 (output contract), D3 (content gate precedes LLM scoring), D4 (civility gate precedes content gate)
**Status:** revised (R1-06: eliminated Phase 1 heuristic — timing moved to D4, observation count moved to D3; R1-08: specified AgentProvider.invoke() mechanism; R1-07: clarified adaptivity posture)

## D3: System 1/2 fast path

**Choice:** Content quality gate with quiet-period bypass (novelty + observation count + time escape)
**Alternatives:**
- Configurable skip rules — consumer-defined ActivationRule<ObservationResult> before reflection. More flexible but requires per-consumer configuration.
- Always reflect — LLM motivation score naturally handles staleness. Simpler but wastes LLM calls on quiet periods.
- Embedding-based novelty only — semantically precise but requires an embedding call per tick. Infrastructure exists (CbrSimilarityScorer via neocortex-memory-api) but adds a dependency and network call to what should be a zero-cost gate.
- Pure novelty + observation count gate (no time bypass) — original R1 revision. Creates a contradiction: tick() is defended as enabling spontaneous initiation (R1-03), but a pure content gate blocks the LLM when observations haven't changed, making tick() functionally reactive.
**Rationale:** Before the LLM motivation scoring (D2), check content quality: (1) novelty of accumulated observations against recent history, and (2) minimum observation count since last initiation. Low novelty OR insufficient observations → skip LLM, return Silent — UNLESS time since last LLM evaluation exceeds a configurable quiet-period threshold (default: 30 min). The quiet-period bypass is the mechanism that makes spontaneous initiation architecturally possible: after extended quiet, the LLM gets to decide whether elapsed time alone justifies speaking ("haven't chatted in a while — should I check in?"). Without this bypass, tick() and a reactive evaluate() produce identical behaviour — both only fire the LLM when observations change. The novelty mechanism is initially token-level similarity (pragmatic, zero-dependency, zero-cost) but the implementation should be replaceable — embedding-based novelty via CbrSimilarityScorer is the natural upgrade when semantic precision justifies the added dependency and latency. Note: qhorus.runtime.watchdog.JaccardSimilarity is package-private and inaccessible from blocks; implementation uses an independent token-level comparator.
**Trade-offs:** Token-level similarity is semantically imprecise — vocabulary overlap ≠ meaning overlap (e.g., opposing positions with shared vocabulary register as low novelty). Mitigated by: (a) false negatives are low-cost (the next tick catches novel content if it persists), (b) the novelty gate is not the only filter — D4 and D2 provide additional gating. The quiet-period bypass adds a time-tracking field (lastLlmEvaluationTimestamp) to InnerLife's internal state. Observation count check absorbed from former D2 Phase 1 to consolidate all content-quality evaluation in D3.
**Sources:** Research doc §2.1 (background thought loop between interactions), §2.8 (System 1/System 2 fast path fold-in), DPT-Agent 2025 (Dual Process Theory with ToM), JaccardSimilarity (qhorus runtime — reference implementation, package-private), CbrSimilarityScorer (neocortex-memory-api — embedding alternative)
**Exploration:** quick
**Depends on:** D2 (fast path gates before LLM scoring)
**Status:** revised (R1-10: acknowledged Jaccard limitations, noted pluggable mechanism and embedding upgrade path; R1-06/R1-11: absorbed observation count from D2 Phase 1; R2-02: added quiet-period bypass to resolve spontaneous initiation contradiction)

## D4: Civility constraints

**Choice:** Composable CivilityConstraint SPI with default implementations
**Alternatives:**
- Delegate to Watchdog — Watchdog is architecturally unsuitable: it is a post-hoc alerting system (evaluateAll() on a schedule) while InnerLife needs pre-dispatch gating (check before posting). Prior rationale incorrectly claimed Watchdog is exclusively channel-scoped — AGENT_STALE evaluates instance staleness via InstanceStore.scan() with no channel involvement. The real distinction is alerting vs gating.
- Hardcoded CivilityGuard with three constraints — violates blocks' SPI-first architectural identity. Blocks has 9+ consumer-implemented SPIs (ActivationRule, DecompositionStrategy, AggregationStrategy, TerminationCondition, RoutingStrategy, Summariser, ContentSummariser, ObservationRenderer, TraitPressureSource). Hardcoding prevents legitimate domain-specific extensions.
- Reuse ActivationRule<T> SPI — surface-level field similarity (activationCount, consecutiveIdleActivations) but different architectural domain. ActivationRule governs activation within agentic orchestration (ExecutionDriver lifecycle). InnerLife governs unprompted social initiation (calendar-time social norms). See D6.
**Rationale:** @FunctionalInterface CivilityConstraint with permitInitiation(InitiationContext). InitiationContext carries social-behaviour metrics: lastInitiationTimestamp, initiationsInWindow, consecutiveInitiationsWithoutResponse, agent. Three default implementations cover literature-identified civility requirements: MinimumGapConstraint (default: 5 min), MaxPerWindowConstraint (default: 3/hour), ConsecutiveInitiationCooldownConstraint (default: 2 consecutive → cooldown until response). All constraints checked as first step of tick(), before D3 or D2 — civility violations incur zero cost. Consumers extend for domain-specific rules (e.g., "don't initiate during business hours", "reduce frequency when user is AFK"). All timing/frequency constraints consolidated here from former D2 Phase 1 to eliminate overlap.
**Trade-offs:** More types than the hardcoded approach. Accepted because this follows blocks' established SPI pattern and enables legitimate domain-specific extensions the hardcoded approach would require wrapping for.
**Sources:** Liu et al. CHI 2025 (civility constraints), Deng et al. AAAI 2025 (adaptivity + civility), Watchdog (qhorus) — post-hoc alerting, ActivationRule (blocks/agentic) — different domain (see D6)
**Exploration:** quick
**Depends on:** D1 (civility gate runs before output production), D2 (bidirectional: D4 handles timing, D2 handles content quality)
**Status:** revised (R1-13: corrected Watchdog scope error; R1-14: composable SPI replaces hardcoded constraints; R1-15/R1-17: ActivationRule reuse evaluated and rejected)

## D5: InnerLife placement

**Choice:** Blocks pattern, not qhorus extension
**Alternatives:**
- qhorus SPI extension — InnerLife as a qhorus component with Watchdog integration. Couples agent behaviour patterns to messaging infrastructure.
- Hybrid — InnerLife in blocks with formal qhorus Watchdog integration for civility. Adds cross-layer coupling for architecturally distinct concerns.
**Rationale:** InnerLife is a compositional agent behaviour pattern in blocks alongside PersonalityEvolution (#118), the agentic orchestration patterns, and the summarisation patterns (ARC42STORIES.MD §5). It produces content dispatched VIA qhorus — it is a producer, not a qhorus component. Civility self-regulation (blocks CivilityConstraint, D4) and communication health monitoring (qhorus Watchdog) serve distinct purposes at different architectural layers: self-regulation gates initiation before it happens, monitoring detects unhealthy patterns after the fact. Both are needed — self-regulation doesn't eliminate monitoring, and monitoring shouldn't own self-regulation. Two rate-limiting systems at different layers is intentional design, not duplication.
**Sources:** ARC42STORIES.MD §5 (blocks building block view — agentic patterns), Watchdog (qhorus — alerting)
**Exploration:** surfaced (R1-18: implicit decision made explicit)
**Status:** captured

## D6: Relationship to ActivationRule SPI

**Choice:** Separate evaluation pipeline
**Alternatives:**
- Compose from ActivationRule implementations — reuse existing SPI for InnerLife's "should I initiate?" question.
- Extend ActivationContext with InnerLife fields — add lastInitiationTimestamp, initiationsInWindow to ActivationContext.
**Rationale:** ActivationRule<T> governs agent activation within agentic orchestration — called by ExecutionDriver during supervised/choreographed execution. ActivationContext fields reflect this: activationCount measures orchestration iterations, consecutiveIdleActivations measures idle rounds in an execution loop, event is the orchestration trigger, lastAggregationResult links to prior execution output. InnerLife's "should I initiate conversation?" operates on fundamentally different inputs (calendar time, social observation history, conversation context) in a different lifecycle (background thought loop, not orchestration execution). Extending ActivationContext with social-behaviour fields would pollute the agentic orchestration SPI with concerns it doesn't own. The CivilityConstraint SPI (D4) is the right abstraction — purpose-built for social initiation gating with the correct context fields.
**Sources:** ActivationRule (blocks/agentic/activation), ActivationContext (blocks/agentic/activation), ExecutionDriver (blocks/agentic/model)
**Exploration:** surfaced (R1-17: implicit decision made explicit)
**Status:** captured
