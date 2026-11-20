## D1: Trust computation engine — Ledger's TrustScoreComputer with in-memory persistence

**Choice:** Use the ledger's Bayesian Beta trust model (`TrustScoreComputer`) with in-memory ledger stores (`casehub-ledger-memory`) rather than building a custom trust computation.
**Alternatives:**
- Custom event-counting heuristic in wacky-manor — simpler but duplicates what ledger already provides, misses temporal decay and credibility weighting
- Full persistent ledger pipeline with JPA — unnecessary for a game session; in-memory stores are designed for exactly this use case
**Rationale:** TrustScoreComputer is pure Java with no CDI dependency, designed for standalone use. The in-memory stores are already in wacky-manor's dependency tree. This makes wacky-manor an exemplar of lightweight ledger usage — real Bayesian Beta scoring without database overhead.
**Trade-offs:** Ledger entry/attestation objects are heavier than a simple counter, but the computation quality (temporal decay, credibility, Bayesian posterior) justifies the structure.
**Sources:** io.casehub.ledger.core.trust.TrustScoreComputer, io.casehub.ledger.memory.InMemoryLedgerEntryRepository, GE-20260429-42fb02 (Bayesian Beta trust returns 0.5 for no evidence)
**Exploration:** quick
**Status:** captured

## D2: Boundary — reusable framework split across three modules, YAML-configurable

**Choice:** Trust evolution framework classes split across three modules following existing architectural boundaries: (1) blocks-core — pure computation helpers (trust score computation wrapper, config spec types, overlay property model). Zero CDI. (2) blocks — CDI wiring (TrustEventRecorder observer, any CDI beans). Depends on quarkus-arc. (3) casehub-neocortex-mindmap-intelligence — TrustConsolidationPhase (implements ConsolidationPhase SPI, runs during consolidation, uses blocks-core computation, writes to overlay nodes). Trust event classification, scoring parameters, and thresholds remain YAML-configurable via a `TrustEvolutionConfigSpec` in `blocks/agentic-yaml/spec/cognition/`. Wacky-manor writes zero trust-specific Java — only YAML configuration.
**Alternatives:**
- All in blocks-core — violates blocks-core's "zero CDI, zero Spring" charter (CDI observer can't go there; ConsolidationPhase SPI is in neocortex)
- All in wacky-manor — not reusable; next cognitive agent rebuilds the same pipeline
- Split: framework in blocks, app-specific recorder in wacky-manor — the recorder logic is generic when parameterised by config; no app-specific Java needed
**Rationale:** blocks-core's pom.xml declares "Framework-neutral blocks POJOs — zero CDI, zero Spring." CDI observers (D7) require quarkus-arc → they go in blocks. The ConsolidationPhase SPI lives in casehub-neocortex-mindmap-intelligence → the trust consolidation phase goes there, following the pattern of all existing phases (AccessFrequencyPhase, ExperienceConsolidationPhase, MergeDetectionPhase, etc.). Pure computation in blocks-core, CDI wiring in blocks, consolidation in neocortex — each module does what it already does.
**Trade-offs:** blocks-core gains casehub-ledger-core as a provided dependency (pure Java, no CDI — consistent with its charter). neocortex-mindmap-intelligence gains a dependency on blocks-core for trust computation helpers.
**Sources:** blocks-core/pom.xml ("zero CDI, zero Spring"), blocks/pom.xml (quarkus-arc dependency), ConsolidationPhase SPI in neocortex, DSL Style Guide §YAML/Java Parity Principle
**Exploration:** deep-analysis
**Status:** revised (R1-02: three-module split replaces "all in blocks-core")

## D3: Per-relationship trust via attestor-filtered scoring with named capabilityTag

**Choice:** Each observer creates attestations (with their agentId as attestorId) on the target's ledger entries, using `capabilityTag="cognitive-trust"` instead of the default `"*"`. During consolidation, TrustScoreComputer is called with attestations filtered to a single attestorId, producing per-relationship Bayesian Beta scores.
**Alternatives:**
- Use `capabilityTag="*"` (the default) — contamines the engine's global trust score when JPA stores are shared (D9). `ExplicitGlobalAttestationsStrategy.selectAttestations()` explicitly includes `"*"` attestations in the global beta score computation via `TrustScoreCalculator.computeAll()`.
- Capability-tag namespacing (e.g. "relationship:penelope") — works but overloads the capability concept for something that's really attestor identity
- Global per-actor trust (no per-relationship) — loses the core insight: Penelope trusts HC differently than Muttley trusts HC
**Rationale:** TrustScoreComputer already accepts filtered attestation lists. Filtering by attestorId is the natural way to partition trust — it's what attestorId semantically means. Using a named capabilityTag ensures cognitive trust attestations are scoped as a distinct capability in `TrustScoreCalculator` — they appear in `capabilityScores["cognitive-trust"]` rather than polluting the global beta fallback. This is safe regardless of whether the backing store is in-memory or JPA.
**Trade-offs:** Requires N×M scoring calls during consolidation (N observers × M targets). Acceptable for game-scale character counts (< 20).
**Sources:** io.casehub.ledger.core.trust.TrustScoreComputer#compute, io.casehub.ledger.api.model.LedgerAttestation#attestorId, ExplicitGlobalAttestationsStrategy (filters for `"*"` only), TrustScoreCalculator.computeAll() (groups non-`"*"` attestations into capability-scoped scores)
**Exploration:** quick
**Status:** revised (R1-04/R2: capabilityTag changed from `"*"` to `"cognitive-trust"` to prevent collision with engine global trust scoring when JPA stores are shared)

## D4: Wacky-manor YAML integration — hand-rolled loader for now, DSL migration follow-up

**Choice:** For #65, add trust config to social-config.yaml using the existing ManorSocialConfigLoader pattern. The blocks-side TrustEvolutionConfigSpec is built (for DSL consumers), but wacky-manor doesn't use the agentic-yaml pipeline yet. A follow-up issue ports all of wacky-manor's social config to the blocks YAML DSL.
**Alternatives:**
- Bundle the full DSL migration into #65 — correct end state but doubles the scope; the migration touches drives, norms, beliefs, relationships, orchestrator wiring
- Skip the blocks spec type, only build for hand-rolled consumption — loses the DSL-first design; the spec type costs nothing to build alongside the framework classes
**Rationale:** Wacky-manor's config pipeline predates the blocks YAML work. The port is valuable (proves the DSL works end-to-end for cognitive agents) but orthogonal to trust evolution. Building the spec type now means the follow-up migration picks it up automatically.
**Trade-offs:** Short-term duplication — ManorSocialConfigLoader parses trust config into a new field on SocialConfig that parallels TrustEvolutionConfigSpec. The migration scope is larger than just trust: SocialConfig currently has drives, norms, beliefs, and relationships — all of these would also migrate to the blocks YAML DSL. The follow-up issue covers the full ManorSocialConfigLoader replacement, not just the trust config portion.
**Sources:** io.casehub.examples.manor.agent.ManorSocialConfigLoader, SocialConfig (drives/norms/beliefs/relationships — no trust field yet), social-config.yaml
**Exploration:** deep-analysis
**Status:** revised (R1-08: trade-off description corrected to reflect full migration scope)

## D5: Overlay property model — trust-score + alpha/beta posterior

**Choice:** Write three properties on person-entity overlay nodes during consolidation: `trust-score` (Bayesian Beta mean, 0.0-1.0), `trust-alpha`, and `trust-beta` (posterior parameters). Alpha and beta are stored for observability and evidence-strength rendering — knowing whether a score is based on strong evidence (high alpha+beta) vs weak evidence (low alpha+beta) enables richer observation text. Trust is always recomputed from the full attestation history via TrustScoreComputer on each consolidation cycle.
**Alternatives:**
- trust-score only — loses evidence-strength information; a score of 0.7 from 2 observations looks identical to 0.7 from 20 observations
- Encode trust in PAD dimensions (e.g. dominance = trust) — conflates two different concepts; PAD is affective perception, trust is epistemic
- O(1) continuation via stored alpha/beta as priors — rejected because TrustScoreComputer's temporal decay recomputes based on `now`, meaning old attestations' weights change over time; carrying forward alpha/beta freezes stale decay. Full recomputation is both correct and trivial at game scale (< 20 characters).
**Rationale:** Three properties cost nothing on overlay nodes. Alpha/beta expose the shape of the posterior for observation rendering ("You feel quite certain about X" vs "You're still forming an opinion"). Full recomputation via TrustScoreComputer preserves temporal decay correctness and keeps the API unmodified (D1).
**Trade-offs:** O(N) per consolidation cycle instead of O(1), where N is attestations per relationship. At game scale (< 20 characters, ~100s of attestations total), this is negligible.
**Sources:** io.casehub.neocortex.mindmap.NodeUpdate#withPropertiesToSet, io.casehub.ledger.core.trust.TrustScoreComputer.ActorScore (alpha, beta fields), TrustScoreComputer.compute() (always initialises alpha=1.0, beta=1.0)
**Exploration:** quick
**Status:** revised (R1-01: removed O(1) continuation claim; alpha/beta retained for observability, not continuation)

## D6: Consolidation ordering — @Priority(22), after merge detection

**Choice:** TrustConsolidationPhase runs at @Priority(22), after MergeDetectionPhase (20), before SchemaDiscoveryPhase (25). The phase lives in casehub-neocortex-mindmap-intelligence (per D2's three-module split).
**Alternatives:**
- @Priority(16) after ExperienceConsolidationPhase — false coupling; trust scoring reads from ledger entries, not from experience graduation output
- Same priority as another phase — non-deterministic ordering
- After CommunitySummaryPhase (30+) — unnecessarily late; community summaries could benefit from trust context
**Rationale:** Trust scoring reads from ledger entries created during tick processing. The only consolidation-phase dependency is MergeDetectionPhase — entity deduplication must complete before scoring so trust computations reference canonical entities, not duplicates that are about to be merged. Existing phase ordering: AccessFrequency(10), Experience(15), MergeDetection(20), TrustConsolidation(22), SchemaDiscovery(25), CommunitySummary(30), CuriosityRefresh(40).
**Trade-offs:** None significant.
**Sources:** MergeDetectionPhase (@Priority(20)), SchemaDiscoveryPhase (@Priority(25)), ConsolidationPhase SPI in neocortex
**Exploration:** quick
**Status:** revised (R1-07: corrected priority from 16 to 22 with proper dependency rationale)

## D7: Trust event recording — CDI event observer on action completion

**Choice:** The TrustEventRecorder in blocks observes a CDI event fired on action completion (see D10 for event design). It maps the action type to a verdict using the YAML-configured trust event table, creates a LedgerEntry for the acting character, and creates LedgerAttestations for affected and witnessing characters. TrustEventRecorder lives in the blocks module (CDI layer), not blocks-core (per D2's three-module split).
**Alternatives:**
- Direct call in the tick loop — couples the orchestrator to trust recording; every cognitive agent would need to add the call manually
- Post-action interceptor/decorator — more complex wiring for the same effect
- Direct service call (TrustEventService.record()) — simpler but loses the decoupling benefit; app must inject and call explicitly
**Rationale:** CDI observer is the established blocks/neocortex pattern for cross-cutting concerns (cf. ExtractionRequestedObserver in neocortex, which observes ExtractionRequested CDI events fired by ConversationBridge). The orchestrator fires the event; the recorder is a zero-touch add-on activated by YAML config presence.
**Trade-offs:** Requires a CDI action-completed event (D10). The event definition is a new blocks type that orchestrators must fire — but the alternative (direct calls in every orchestrator) is the current anti-pattern that D7 exists to eliminate.
**Sources:** blocks CDI observer patterns, ExtractionRequested/ExtractionRequestedObserver pattern in neocortex, ScenarioOrchestrator tick loop (line 387)
**Exploration:** quick
**Status:** revised (R1-03: acknowledged CDI event as explicit prerequisite, cross-referenced D10)

## D8: Observation threshold — configurable delta gating with TrustLevel/TrustSummary integration

**Choice:** Trust changes only surface in observations when the absolute delta since the last rendered trust observation exceeds a configurable threshold (default 0.15). Below that, the score updates on the overlay node but no observation text is generated. Observation rendering integrates with the existing TrustLevel enum and TrustSummary record in blocks-core, and feeds into CognitiveObservationSections.trustSection() for consistent rendering. Score-to-level mapping: score ≥ 0.7 → HIGH, 0.4 ≤ score < 0.7 → MODERATE, score < 0.4 → LOW. These thresholds are configurable in TrustEvolutionConfigSpec.
**Alternatives:**
- Always render — noisy; small fluctuations produce meaningless observations
- Fixed threshold levels only (HIGH/MODERATE/LOW transitions) — loses granularity; a 0.69→0.71 transition across the MODERATE→HIGH boundary fires but a 0.3→0.5 change within MODERATE doesn't
**Rationale:** Delta-based gating captures significant trust shifts regardless of which level band they fall in. TrustSummary(subjectName, level, reason) is the existing type for trust observations — the trust framework produces these during consolidation, and CognitiveObservationSections.trustSection() renders them. The evidence strength from D5's alpha/beta shapes the reason text ("You feel quite certain..." vs "Your initial impression...").
**Trade-offs:** Requires tracking "last rendered trust score" per relationship — one additional property on the overlay node (`trust-last-rendered`). Level boundary thresholds in TrustEvolutionConfigSpec must produce TrustSummary objects compatible with the existing trustSection() renderer.
**Sources:** io.casehub.blocks.summarisation.observation.affordance.TrustLevel, TrustSummary, CognitiveObservationSections.trustSection(), PerceptionTranslator (existing distance threshold pattern at 0.4)
**Exploration:** quick
**Status:** revised (R1-06: explicit TrustLevel/TrustSummary integration and score-to-level mapping)

## D9: Persistence agnostic — SPI-driven, durable stores configurable

**Choice:** All blocks framework classes depend on `LedgerEntryRepository` (SPI interface), not on any specific implementation. In-memory vs JPA-backed persistence is a deployment-time choice via Quarkus `selected-alternatives`. The framework works identically with both — wacky-manor uses in-memory for game sessions, but a production cognitive agent configures JPA-backed stores for durable trust history.
**Alternatives:**
- Hardcode in-memory assumption — locks the framework to ephemeral use cases; production agents can't persist trust history across restarts
- Require JPA — forces a database even for lightweight/demo scenarios
**Rationale:** The ledger already has this duality (`InMemoryLedgerEntryRepository` at @Priority(1) vs `JpaLedgerEntryRepository` as default). The trust framework simply inherits it by depending on the SPI, not the implementation. Zero additional work — just don't break the abstraction.
**Trade-offs:** When JPA stores are selected, the trust evolution framework and the engine share the same `LedgerEntryRepository` implementation. Trust evolution attestations become visible to `ComputedTrustScoreSource` and `TrustScoreCalculator`. D3 mitigates this by using `capabilityTag="cognitive-trust"` instead of `"*"` — `ExplicitGlobalAttestationsStrategy` only selects `"*"` attestations for the global beta score, so named-capability attestations are isolated into their own capability-scoped score and do not contaminate the engine's global trust computation.
**Sources:** io.casehub.ledger.api.spi.LedgerEntryRepository, io.casehub.ledger.memory.InMemoryLedgerEntryRepository, io.casehub.ledger.runtime.repository.jpa.JpaActorTrustScoreRepository, ComputedTrustScoreSource.computeFresh(), ExplicitGlobalAttestationsStrategy.selectAttestations()
**Exploration:** quick
**Status:** revised (R2-05: documented JPA store-sharing interaction and capabilityTag guard from D3)

## D10: Trust event trigger — TrustRelevantAction CDI event in blocks

**Choice:** Define a `TrustRelevantAction` CDI event type in the blocks module. The event carries actor ID, target ID, action type, and action result. Application orchestrators fire this event after action resolution; TrustEventRecorder (D7) observes it asynchronously.
**Alternatives:**
- Direct service call (TrustEventService.record()) — simpler API but requires every orchestrator to explicitly inject and call the service; loses cross-cutting decoupling
- Extend an existing engine lifecycle event — no existing event carries cognitive action context; engine events are work/case lifecycle, not agent-action-level
- Application-defined event — defeats framework reuse; every cognitive agent would define its own event type
**Rationale:** Follows the established neocortex pattern: ConversationBridge fires ExtractionRequested CDI event, ExtractionRequestedObserver handles it asynchronously. The event type lives in blocks because it's part of the trust framework contract — orchestrators import it from blocks, not from the application. The ScenarioOrchestrator currently calls `recordTrustEvent()` directly (line 387); replacing this with `event.fire(new TrustRelevantAction(...))` is mechanical.
**Trade-offs:** Introduces a new CDI event type that orchestrators must fire. Applications without CDI (pure Java orchestrators) would use the direct-call alternative.
**Sources:** ExtractionRequested/ExtractionRequestedObserver pattern in neocortex (see neocortex docs/specs/issue-295-knowledge-consolidation/2026-09-10-knowledge-consolidation-pipeline-design.md §3.4 for the event contract and observer implementation), ScenarioOrchestrator line 387, D7
**Exploration:** quick (surfaced by R1-03)
**Status:** revised (R2-04: added spec reference for ExtractionRequested precedent)

## D11: Trust granularity — per-actor-pair, not per-capability

**Choice:** Trust is computed per actor-pair (observer→subject). An agent's trust in another agent is a single scalar, not differentiated by action type or capability domain.
**Alternatives:**
- Per-capability-per-actor-pair — agent trusts X for code review but not for merge decisions. Richer model, matches engine-ledger's capability-scoped TrustWeightedAgentStrategy pattern
- Global per-actor — loses the core insight that Penelope trusts HC differently than Muttley trusts HC
**Rationale:** For the game context, per-actor-pair trust is sufficient and avoids the complexity of defining capability taxonomies for cognitive agents. All trust evolution attestations use `capabilityTag="cognitive-trust"` (D3), which scopes them as a distinct named capability in the ledger's `TrustScoreCalculator`. This prevents collision with engine-level global trust scoring regardless of whether in-memory or JPA stores are used (D9). A future enhancement could introduce multiple named capability tags (e.g., `"cognitive-trust:honesty"`, `"cognitive-trust:reliability"`) for per-capability cognitive trust without breaking the current model.
**Trade-offs:** Cognitive agents can't express differentiated trust ("I trust X for advice but not with my possessions"). Acceptable for game scope; the per-capability extension path is available via additional capabilityTag values without redesign.
**Sources:** D3 (attestor-filtered scoring, capabilityTag="cognitive-trust"), D9 (store-sharing guard), TrustScoreCalculator.computeAll() (capability-scoped scoring)
**Exploration:** quick (surfaced by R1-10)
**Status:** revised (R1-04/R2: removed false store-isolation claim, referenced D3's capabilityTag fix)
