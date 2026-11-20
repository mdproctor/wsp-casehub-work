# Decisions — issue-63-fix-cdi-config-beans

## blocks#283 — Directive-Minimal Architecture

## D1: Scope — full rewrite vs incremental patching

**Choice:** Full rewrite of system prompt structure, seeding mechanism, briefing text, and deduplication removal
**Alternatives:**
- Structured data only — move goals/constraints/disposition mechanically, leave briefing text. Safe but defers the real problem.
- Briefing text + structured data — refactor prose too but keep existing rendering pipeline. Half-measure.
**Rationale:** Pre-release with a single consumer (wacky-manor). Git makes this fully reversible. Patching defers the structural issue — the directive overpowering observation sections — without solving it.
**Trade-offs:** Larger scope means more files touched across 3 repos. Mitigated by git reversibility and empirical testing.
**Sources:** blocks#283 issue description, HANDOFF.md Phase C notes
**Exploration:** quick
**Status:** captured

## D2: System prompt content — what stays in the directive

**Choice:** Name + role + voice + hard constraints. Dynamic cognitive data moves to observation sections.
**Alternatives:**
- Name + role + voice only — maximally minimal but moves safety-critical constraints to observations where LLM attention is weaker. Empirically risky.
- Full character description stays — least disruptive but keeps behavioral instructions in directive, defeating the purpose.
**Rationale:** Hard constraints (HARD severity) are safety-critical — "never reveal your true identity," "do not break character." LLM system prompts receive higher attention weight than user-turn content; moving must-not-violate rules to observation sections risks weaker adherence. Soft constraints and dynamic cognitive constraints belong in observations.
**Trade-offs:** The directive is slightly larger than pure name+role+voice. Acceptable — hard constraints are few per character and structurally distinct.
**Sources:** descriptors-composite.yaml briefing fields, SystemPromptRenderer.render()
**Exploration:** quick
**Revised by:** R1-03 (decision review round 1) — original choice was "name + role + voice only" with hard constraints treated as observations. Reviewer correctly identified that LLM attention dynamics make system prompt placement important for safety-critical constraints.
**Status:** revised

## D3: Cognitive preamble — auto-generated from active subsystems

**Choice:** Auto-generated from active subsystems. A single renderer inspects which subsystems are active and composes one coherent paragraph.
**Alternatives:**
- YAML template per archetype — define cognitive instruction templates. Readable but drifts from actual wiring.
- Inline in briefing YAML — per-character authored text. Maximum control, maximum maintenance.
- Per-subsystem snippets — each subsystem contributes its own instruction text. Modular but fragmented prose.
**Rationale:** Auto-generation eliminates drift between what's wired and what's described. Single renderer owns prose quality — reads as coherent instruction, not a list of disconnected sentences. The cognitive preamble lives in the system prompt as part of the minimal directive — it tells the agent "you have a brain, here's how to use it" without duplicating the cognitive data itself. It replaces the per-section `DirectiveSection` wrapping (which prepends behavioral instructions to each observation section). With the preamble, each observation section presents raw cognitive state; the preamble provides the meta-instruction once.
**Trade-offs:** The renderer must be updated when new subsystem types are added. Acceptable — new subsystems are infrequent and the renderer is the natural place to document their cognitive role. Losing per-section DirectiveSection instructions means each section must be self-explanatory via its heading and structure.
**Preamble wording constraint:** The preamble describes cognitive architecture ("You have motivational drives that influence your priorities"), NOT current state ("some are stronger than others right now"). State-dependent claims conflict with the system prompt caching strategy — the preamble is part of the cached RenderedPrompt, and state-dependent text would require per-turn cache invalidation. Actual cognitive state is conveyed by observation sections, which are rebuilt every turn. The preamble should also disambiguate the two drive systems (see D16): "Your psychological needs — curiosity, competence, affiliation, autonomy — shift based on your interactions. Separately, your character motivations — the drives that define who you are — evolve based on your experiences."
**Implementation gaps (required changes):** (1) `CognitivePreambleGenerator.generate()` currently appends "some are stronger than others right now" when `drivesEnabled` is true — a state-dependent phrase that violates the caching constraint above. Must be reworded to architecture-only language (e.g., "You have motivational drives that influence your priorities"). (2) The generator does not check `config.characterDrivesEnabled()` and produces no drive disambiguation text. When character drives are enabled, the preamble must include the two-system disambiguation specified above.
**Sources:** CognitionCore.promptSections(), DirectiveSection wrapping pattern, RenderedPromptCache
**Exploration:** quick
**Clarified by:** R1-05 (decision review round 1) — added explicit relationship to DirectiveSection and placement in system prompt. R1-06 + R1-07 (decision review round 2) — preamble must describe architecture not current state (caching constraint); added drive disambiguation text for D16. Adversarial review R1-03 — flagged two implementation gaps in CognitivePreambleGenerator.
**Status:** captured

## D4: Templates — split into voice (directive) and behavioral seeds (neurocortex)

**Choice:** Template content is split: voice/style elements (speaking patterns, catchphrases, comedic conventions) stay in the directive as part of the character's voice definition. Behavioral pattern elements (strategies, disposition biases, norms) become seed data for cognitive subsystems.
**Alternatives:**
- All template content becomes cognitive seeds — but voice/style has no natural home in CognitionCore subsystems (no "speaking style orchestrator" exists).
- Templates stay entirely in system prompt — keeps behavioral instructions in the directive, defeating the purpose.
- Remove templates entirely — loses shared archetype reuse.
**Rationale:** Templates contain two distinct categories: voice/style ("expository soliloquy," "catchphrase repetition") and behavioral patterns ("scheme obsessively," "protect allies"). Voice/style is identity (stays in directive per D2). Behavioral patterns are cognitive state (moves to seeding per D5). Clean separation along the same boundary D2 establishes.
**Trade-offs:** Templates must be manually classified into voice vs behavioral. One-time effort for 4 templates.
**Sources:** templates.yaml (cartoon-villain, cartoon-hero, protector archetypes, hanna-barbera-cartoon-style)
**Exploration:** quick
**Revised by:** R1-08 (decision review round 1) — original choice moved all template content to seeding. Reviewer identified that voice/style content has no CognitionCore subsystem home and conflicts with D2's "voice stays in directive."
**Status:** revised

## D5: Neurocortex seeding — dual-layer with existing YAML

**Choice:** Both archetype-level (template seeds) and per-character seed data, stored in existing character YAML files. A loader reads seed sections and pushes into subsystems on first boot.
**Alternatives:**
- Separate seed YAML per character — clean separation but scatters character definition across files.
- Programmatic seeding only — Java code seeds based on traits. Maximum flexibility, zero declarative visibility.
**Rationale:** Keeps all character definition in one place. Two layers compose: template provides archetype defaults, per-character overrides specialize. Both are declarative and inspectable.
**Trade-offs:** Character YAML files grow larger with seed sections. Acceptable — they're already the single source of character definition.
**Sources:** descriptors-composite.yaml, social-config.yaml, trust-evolution.yaml
**Exploration:** quick
**Status:** captured

## D6: Backward compatibility — not needed

**Choice:** No backward compatibility mechanism. Rewrite rendering directly.
**Alternatives:**
- New BriefingMode.COGNITIVE with opt-in migration — clean migration lever but unnecessary indirection.
- Replace RICH semantics — redefine existing mode. Breaks nothing since there's only one consumer.
**Rationale:** Pre-release, single consumer (wacky-manor). No external dependencies on current behavior.
**Trade-offs:** None — this is the simplest path.
**Sources:** BriefingMode enum (EMPTY/NAME_ONLY/NAME_ROLE/RICH)
**Exploration:** quick
**Status:** captured

## D7: Implementation approach — custom renderer override, not eidos modification

**Choice:** Provide a custom `SystemPromptRenderer` implementation in blocks (or wacky-manor) that overrides the eidos `@DefaultBean`. The eidos `EidosSystemPromptRenderer` stays untouched. The custom renderer produces the minimal directive (name + role + voice + hard constraints + cognitive preamble). Existing `CognitionCore.promptSections()` pipeline handles observation sections. NeurocortexSeeder (in blocks) generalizes `ManorCognitiveSeeder` to read seed data from YAML and push into subsystems on first boot.
**Alternatives:**
- Modify EidosSystemPromptRenderer directly — eidos has 113+ references to SystemPromptRenderer across eval framework, A2A card generation, and eidos-examples. Changing the eidos renderer would invalidate eval baselines and impact non-cognitive consumers.
- New CognitivePromptComposer alongside existing renderer — unnecessary indirection.
- Invert pipeline (observation sections own everything including voice/style) — too risky without empirical evidence.
**Rationale:** `EidosSystemPromptRenderer` is `@ApplicationScoped` — not `@DefaultBean`. Overriding it requires `@Alternative @Priority(1)` on the replacement bean (a standard CDI mechanism). A `@DefaultBean` would cause an ambiguous dependency error since the existing bean is `@ApplicationScoped` and not automatically displaced. The `@Alternative @Priority(1)` approach is the correct CDI pattern for overriding a platform `@ApplicationScoped` bean without modifying the platform. CognitionCore already does the observation-side heavy lifting. NeurocortexSeeder generalizes the existing ManorCognitiveSeeder pattern rather than introducing a parallel seeder.
**Trade-offs:** Two SystemPromptRenderer implementations exist (eidos default + cognitive override). Acceptable — `@Alternative` with explicit priority makes the override intentional and priority-ordered.
**Sources:** EidosSystemPromptRenderer (`@ApplicationScoped`), SystemPromptRenderer SPI, ManorCognitiveSeeder, CognitionCore.promptSections()
**Exploration:** quick
**Revised by:** R1-02 (decision review round 1) — original choice was "rewrite SystemPromptRenderer.render()" which would modify the eidos platform renderer. Reviewer identified that SystemPromptRenderer lives in eidos-api/eidos, has 113+ consumers, and should not be modified. Round 2 (R1-02): corrected rationale — replaced incorrect `@DefaultBean` language with the actual CDI mechanism (`@Alternative @Priority(1)` overriding `@ApplicationScoped`).
**Depends on:** D1 (full rewrite scope), D2 (minimal directive), D5 (seeding mechanism)
**Status:** revised

## D8: Full stack scope — no issue split

**Choice:** #283 covers the full stack: rendering rewrite + seeding mechanism + briefing text stripping + duplication removal.
**Alternatives:**
- Split rendering and seeding into separate issues — smaller PRs but intermediate state (minimal directive with empty observations) is broken and untestable.
**Rationale:** The changes are coupled — you can't test the minimal directive without seeded subsystems providing the content that was removed from the directive.
**Trade-offs:** Larger PR. Mitigated by cross-repo issue tracking (each repo gets its own issue).
**Sources:** .plan queue (6 remaining issues depend on this architectural change)
**Exploration:** quick
**Depends on:** D1 (full rewrite scope)
**Status:** captured

---

## examples#66 — Drive Adaptation from Reward Signals

## D9: Action-type to drive mapping — per-character YAML

**Choice:** Per-character mapping in social-config.yaml. Each character declares which of its drives are reinforced by which event types.
**Alternatives:**
- Global mapping — single config maps event types to drive categories. Simpler but characters lose individual response profiles; every character with a "scheming" drive is reinforced identically.
- Hybrid (global defaults + per-character overrides) — more configuration complexity for marginal benefit at this stage.
**Rationale:** Characters should respond differently to the same event type. Hooded Claw's "scheming" is reinforced by conflict_resolution; Penelope's "social-harmony" is reinforced by social_interaction. The mapping is inherently per-character because it expresses motivational identity — what this character finds rewarding.
**Trade-offs:** More YAML per character. Acceptable — social-config.yaml already defines per-character drives, goals, norms, beliefs, and relationships.
**Sources:** SocialConfig.Drive, ActionImportanceScorer event types, issue #66
**Exploration:** quick
**Status:** captured

## D10: Persistence — MindMap nodes in COGNITIVE subgraph

**Choice:** Store adapted drive intensities as properties on drive-specific MindMap nodes in the COGNITIVE subgraph.
**Alternatives:**
- Extend CaseMemoryStore — designed for episodic memories with cursor-based scanning, not mutable state documents.
- New dedicated DriveStateStore — clean interface but introduces a new persistence layer that nothing else uses.
- Separate drive-specific subgraph (e.g., `drives-{agentId}`) — cleaner separation but proliferates subgraphs; ExperienceConsolidationPhase already uses a single `cognitive` type subgraph for all agents.
**Rationale:** Consistent with how trust (TrustConsolidationPhase), experience (ExperienceConsolidationPhase), and other cognitive state is already stored. MindMapStore is available in both consolidation and rendering paths. Nodes support properties, confidence, and PAD values — all useful for drive state.
**Node discrimination:** Drive nodes share the `cognitive` type subgraph used by ExperienceConsolidationPhase. They are distinguished from graduated experience nodes by: (1) `cognitiveKind: "drive-intensity"` (experience nodes use classifier-assigned `cognitiveKind` from `GraduationResult`); (2) `provenance: "drive-adaptation"` (experience nodes use `provenance: "experience-consolidation"`). Drive nodes carry: `agent-id` (agent scoping, same pattern as experience nodes), `drive-type` (e.g., "scheming", "social-harmony"), `intensity` (current adapted value, 0.0–1.0), `initial-intensity` (from seed data). DriveAdaptationPhase queries by `cognitiveKind: "drive-intensity"` + `agent-id` to locate drive nodes.
**Trade-offs:** MindMap nodes are a general-purpose knowledge graph mechanism, not a purpose-built drive store. Querying requires filtering by properties rather than typed queries. Acceptable — the same pattern works well for trust and experience.
**Subgraph location strategy:** ManorCognitiveSeeder creates per-agent subgraphs (`"beliefs-{agentId}"`, type `"cognitive"`). ExperienceConsolidationPhase's `findOrCreateCognitiveSubgraph()` uses `findFirst()` on cognitive-type subgraphs — in multi-agent scenarios, all graduated experience nodes land in whichever cognitive subgraph is enumerated first. DriveAdaptationPhase iterates each cognitive subgraph independently via `mindMapStore.nodesIn(subgraph.id(), tenantId)`, collecting both drive nodes and experience nodes from the same subgraph. In the current multi-agent setup, `findFirst()` concentrates all experience nodes in one subgraph, which means only the agent whose subgraph was selected receives drive adaptation from experiences. This is a pre-existing platform behavior — it predates D10 and affects any phase that reads experience nodes alongside its own node type. Fixing the `findFirst()` distribution is a separate concern that should be addressed at the ExperienceConsolidationPhase level.
**Sources:** MindMapStore, ExperienceConsolidationPhase (cognitiveKind property, agent-id property), TrustConsolidationPhase
**Exploration:** quick
**Clarified by:** R1-08 (decision review round 2) — added subgraph selection, node discrimination mechanism, and property model for drive nodes. Adversarial review R1-06 round 1 — added subgraph location strategy. Round 2 — corrected factual inaccuracy: DriveAdaptationPhase uses per-subgraph `nodesIn()`, not cross-subgraph `MindMapQuery`; acknowledged that the `findFirst()` experience distribution means only one agent's drives adapt from experiences in multi-agent setups.
**Depends on:** D9 (per-character mapping determines what's stored per drive)
**Status:** revised

## D11: Phase design — new DriveAdaptationPhase

**Choice:** New ConsolidationPhase implementation (`DriveAdaptationPhase`) running after ExperienceConsolidationPhase (priority ~17).
**Alternatives:**
- Extend ExperienceConsolidationPhase — couples experience graduation and drive adaptation. ExperienceConsolidationPhase is in neocortex (platform), while drive adaptation needs application-level config, creating a dependency direction problem.
- Event-driven via ConsolidationCompleted CDI event — decoupled but loses access to per-tenant consolidation context and can't participate in phase ordering.
**Rationale:** Clean SRP — experience graduation and drive adaptation are separate concerns. The phase reads graduated mindmap nodes that ExperienceConsolidationPhase just created, aggregates reward signals per action-type, and updates drive intensity nodes. Runs in the existing pipeline with proper ordering.
**Trade-offs:** Second pass over data (reads nodes that ExperienceConsolidationPhase just wrote). Acceptable — the node set is small (maxPerPass=20) and the read is by subgraph, not a full scan.
**Sources:** ConsolidationPhase SPI, ConsolidationScheduler phase ordering, ExperienceConsolidationPhase
**Exploration:** quick
**Status:** captured

## D12: Reward signal — configurable PAD axis with arousal modulation

**Choice:** Each drive reinforcement mapping specifies which PAD dimension serves as its reward signal (default: pleasure). Arousal modulates the magnitude. The reward axis is a per-mapping configuration — not per-drive, not global — because the same drive may respond to different PAD dimensions depending on the event type.
**Alternatives:**
- Pleasure-only (original choice) — simple but mismodels outcomes where instrumental success diverges from hedonic experience (e.g., successful but stressful conflict mediation weakens social-harmony under pleasure-only reward).
- Composite PAD reward (weighted combination of all three dimensions) — harder to reason about and configure.
- Per-drive PAD weighting (each drive specifies a single PAD dimension globally) — ignores that the same drive may need different signals for different event types.
**Rationale:** Pleasure is the right reward axis for most mappings — it captures "was this a good outcome." But for drives like social-harmony, dominance (agency/control in the situation) is a better reward signal for conflict-resolution events: a stressful mediation that succeeds is high-dominance, not high-pleasure. Making the axis configurable per mapping is consistent with D9's per-character philosophy — the reinforcement relationship is already per-character, extending it to include the reward dimension is architecturally coherent. Default to pleasure when not specified.
**Trade-offs:** Adds one optional field to each mapping entry. Marginal configuration cost for significantly better modeling of the pleasure/instrumental-success divergence.
**Sources:** Memory.pleasure(), Memory.arousal(), Memory.dominance(), issue #66, GE-20260714-439924 (multiplicative dampening)
**Exploration:** quick
**Revised by:** R1-03 (decision review round 2) — original pleasure-only choice mismodeled successful-but-unpleasant outcomes. Reviewer demonstrated concrete failure scenario (Penelope's social-harmony weakened by stressful conflict mediation). Per-mapping PAD axis selection is consistent with D9's per-character philosophy.
**Status:** revised

## D13: Code location — reusable phase in blocks-core

**Choice:** DriveAdaptationPhase lives in blocks-core with a structured config SPI: `Map<String, List<DriveReinforcementEntry>>` (event-type → list of reinforcement entries). Each `DriveReinforcementEntry` specifies: `driveType` (string), `direction` (POSITIVE or NEGATIVE, default POSITIVE), and `rewardAxis` (PLEASURE, DOMINANCE, or COMPOSITE, default PLEASURE). Wacky-manor provides only the YAML configuration.
**Alternatives:**
- Simple map `Map<String, List<String>>` (original choice) — lacks reinforcement direction and reward axis configuration. Cannot express negative reinforcement (e.g., socializing weakens scheming) or per-mapping PAD axis selection (per D12 revision).
- Wacky-manor only — inherently application-specific. But the adaptation mechanics (multiplicative update, clamping, arousal modulation) are generic and reusable.
- Neocortex — alongside other consolidation phases. Creates a neocortex dependency on drive concepts which are blocks-level.
**Rationale:** The adaptation algorithm is domain-agnostic: read reward signals from graduated nodes, map to drives using the structured SPI, apply directional multiplicative updates. The enriched SPI enables: (1) negative reinforcement — `social_interaction → scheming: NEGATIVE` means pleasant socializing weakens scheming identity; (2) per-mapping reward axis — `conflict_resolution → social-harmony: POSITIVE, rewardAxis=DOMINANCE` uses dominance instead of pleasure as the reward signal (per D12). Follows the blocks pattern where platform provides the engine and applications provide configuration.
**Trade-offs:** Richer SPI requires `DriveReinforcementEntry` record instead of plain strings. Marginal complexity for significantly better expressiveness. Blocks-core gains a new consolidation phase dependency on MindMapStore (neocortex). This dependency already exists via DriveOrchestrator → MemoryHygieneOrchestrator path.
**Sources:** blocks-core existing patterns (DriveOrchestrator, CognitionCore), ManorGraduationScorer, ActionImportanceScorer
**Exploration:** quick
**Revised by:** R1-04 (decision review round 2) — original `Map<String, List<String>>` SPI lacked reinforcement direction and reward axis selection. Enriched SPI is consistent with D9's per-character philosophy and D12's revised reward signal approach.
**Depends on:** D11 (new phase), D9 (mapping SPI), D12 (reward signal — per-mapping axis)
**Status:** revised

## D14: Bootstrapping — seeder writes drive nodes

**Choice:** ManorCognitiveSeeder (already enhanced in #283 for goal seeding) also seeds drive nodes into the COGNITIVE subgraph with initial intensities from social-config.yaml. DriveAdaptationPhase only reads/updates these nodes — never touches SocialConfig directly.
**Alternatives:**
- DriveConfigProvider SPI — blocks-core defines interface, wacky-manor implements to return initial intensities. Adds an SPI but avoids seeder dependency.
**Rationale:** Clean dependency direction. The seeder already runs at scenario bootstrap and handles beliefs, relationships, and goals. Adding drive seeding is natural. DriveAdaptationPhase in blocks-core has no dependency on SocialConfig — it works purely with MindMap nodes.
**Trade-offs:** Drive adaptation won't work without seeded nodes (no automatic fallback to static config). Acceptable — the seeder is mandatory for all cognitive features.
**Diagnostic requirement:** DriveAdaptationPhase must emit a `WARNING`-level log message when it finds no drive nodes for a given `agent-id` in the COGNITIVE subgraph. Unlike experience nodes (whose absence is expected at cold start), drive nodes are required at bootstrap — their absence is a configuration error. The diagnostic should log the agent-id and tenant-id to aid troubleshooting.
**Sources:** ManorCognitiveSeeder, D5 (neurocortex seeding from #283)
**Exploration:** quick
**Clarified by:** R1-11 (decision review round 2) — added diagnostic logging requirement for missing drive nodes.
**Depends on:** D10 (MindMap persistence), D13 (blocks-core location)
**Status:** captured

## D15: Rendering — CharacterDrivePromptSection in blocks-core

**Choice:** Create a `CharacterDrivePromptSection` (blocks-core PromptSection) that reads adapted drive nodes from MindMapStore and renders them in the observation section pipeline. CharacterCognition stops rendering drives directly.
**Alternatives:**
- CharacterCognition reads MindMapStore — keeps rendering in wacky-manor but adds MindMapStore dependency to CharacterCognition rendering, and misaligns with #283's directive-minimal architecture.
**Rationale:** Consistent with how platform SDT drives are rendered via DrivePromptSection. Aligns with #283's architecture where all cognitive state flows through CognitionCore observation sections. CharacterCognition's drive rendering (lines 106-111) is removed — one fewer concern in the application layer.
**Trade-offs:** Two drive prompt sections exist in the observation pipeline (platform SDT DrivePromptSection + character CharacterDrivePromptSection). They render different concepts — SDT psychological needs vs character motivational identity — under separate headings with explicit differentiation.
**Heading differentiation:** DrivePromptSection renders under "Psychological Needs" (via `CognitiveObservationSections.motivationalStateSection()`). CharacterDrivePromptSection renders under "Character Motivations". The cognitive preamble (D3) provides disambiguation text explaining the relationship between the two systems (see D16).
**Sources:** DrivePromptSection, CharacterCognition.renderCognitiveSections(), D7 (CognitiveSystemPromptRenderer from #283)
**Exploration:** quick
**Clarified by:** R1-07 (decision review round 2) — added explicit heading differentiation and preamble disambiguation requirement.
**Depends on:** D10 (MindMap persistence), D13 (blocks-core location), D16 (drive taxonomy relationship)
**Status:** captured

---

## Cross-Cutting Implicit Decisions (surfaced by decision review round 2)

## D16: Drive taxonomy — SDT drives and character motivations are intentionally independent

**Choice:** SDT drives (DriveOrchestrator, 4 fixed axes: CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY) and character motivations (SocialConfig.Drive, free-form: "scheming", "gallantry", "social-harmony") remain independent systems. They are rendered separately, adapted by separate mechanisms, and stored in separate locations. The cognitive preamble (D3) provides disambiguation text.
**Alternatives:**
- Convergence: character motivations as specializations of SDT axes — e.g., "scheming" maps to AUTONOMY. Loses the modeling distinction (SDT axes are universal psychological needs; character motivations are authored identity).
- Subsumption: one system replaces the other. SDT drives are platform-level and dynamically computed; character motivations are application-level and identity-defining. Neither subsumes the other's function.
- Explicit divergence with no disambiguation — current implicit state. The LLM receives two drive sections with no framing of their relationship.
**Rationale:** SDT drives and character motivations model motivation at different abstraction levels. SDT drives capture universal psychological needs (computed dynamically from interaction patterns by `DriveOrchestrator`). Character motivations capture character-specific motivational identity (authored per character, adapted from reward signals). They are complementary, not competing: SDT AFFILIATION being high and `social-harmony` being high are reinforcing signals at different levels of description. The preamble disambiguates this for the LLM.
**Trade-offs:** The LLM must interpret two drive systems. Mitigated by distinct section headings (D15) and preamble disambiguation (D3).
**Sources:** DriveOrchestrator (4-axis SDT model), SocialConfig.Drive (free-form character motivations), DrivePromptSection, CharacterDrivePromptSection
**Exploration:** quick (surfaced by R1-14, decision review round 2)
**Status:** captured

## D17: CognitionConfig CDI scope — application-level, not per-agent

**Choice:** `CognitionConfig` registered as a CDI synthetic bean via `SyntheticBeanBuildItem` with `@DefaultBean` scope. One config per application. Per-agent cognitive variation is expressed through the data fed to subsystems (SocialConfig), not through CognitionConfig flags.
**Alternatives:**
- Per-agent CognitionConfig (e.g., via `@RequestScoped` or a provider pattern) — would allow agents with different subsystem configurations in the same application. Adds significant CDI complexity for a capability not currently needed.
- No CDI registration (keep as plain record) — CognitiveSystemPromptRenderer can't inject it; must receive it through a different mechanism.
**Rationale:** All cognitive subsystem orchestrators (MoodOrchestrator, DriveOrchestrator, etc.) are `@ApplicationScoped` CDI beans — the same subsystem instances serve all agents. CognitionConfig flags describe which of these application-scoped subsystems are active. Per-agent variation in what subsystems exist would require per-agent CDI scoping of the subsystems themselves — a fundamentally different architecture. The CognitionConfig synthetic bean describes the application's cognitive architecture, matching the CDI scope of the subsystems it configures. If a future application needs agents with different cognitive subsystem configurations, that is a different deployment, not a per-agent CognitionConfig.
**Known limitation:** All agents in a given application share the same CognitionConfig. This is architecturally correct for the current design where subsystems are application-scoped.
**Sources:** CognitionConfig (plain record), SocialAvatarCognition (creates one CognitionCore for all agents), CognitionConfigBuildItem, AgenticYamlProcessor.discoverCognitionConfig()
**Exploration:** quick (surfaced by R1-09, decision review round 2)
**Status:** captured

## D18: CharacterCognition deduplication — CognitionCore sections separate from application sections

**Choice:** `CharacterCognition.renderCognitiveSections()` provides only application-specific content (beliefs, norms, social awareness, trust perceptions). CognitionCore sections are wired into the observation pipeline at the `ObservationBuilder` level, separate from CharacterCognition's sections.
**Alternatives:**
- CharacterCognition calls CognitionCore.promptSections() and aggregates both — creates a second path for CognitionCore sections alongside SocialAvatarCognition's `buildSections()`. In rendering paths where both contribute, CognitionCore sections would be duplicated.
- Remove from both CharacterCognition and SocialAvatarCognition, wire at a higher level — cleanest but requires both paths to be refactored simultaneously.
**Rationale:** The spec's "After" architecture shows CognitionCore sections and CharacterCognition sections as independent items in the observation pipeline. CharacterCognition should provide only application-specific content that has no CognitionCore equivalent: character motivations, initial beliefs, trust perceptions, social awareness, norms. CognitionCore sections (mood, SDT drives, narrative, mental model, user model, strategy, goals, personality, soft constraints) are provided by CognitionCore through the observation pipeline wiring, not aggregated by CharacterCognition.
**Current code state:** Verified — `CharacterCognition.renderCognitiveSections()` does not call `cognitionCore.promptSections()` in the current code. The `cognitionCore` field exists in CharacterCognition (assigned in constructor) but is unused — dead code to be removed. The separation described by this decision is already the implemented state. No code change is required for the deduplication itself.
**Sources:** CharacterCognition.renderCognitiveSections(), SocialAvatarCognition.buildSections(), spec §5 (deduplication)
**Exploration:** quick (surfaced by R1-15, decision review round 2)
**Revised by:** Adversarial review R1-07 — corrected to reflect current code state. The original text described "stops calling cognitionCore.promptSections()" but this call does not exist in current code.
**Depends on:** D7 (CognitiveSystemPromptRenderer), D15 (CharacterDrivePromptSection)
**Status:** revised

---

## examples#70 — Relationship Stage Thresholds

## D19: Configuration location — per-character in social-config.yaml, using existing 5-tier model

**Choice:** Add a `familiarity-thresholds` section per character in social-config.yaml. Uses the existing 5-tier model from `RelationshipStageConfig.defaults()`: stranger (0.0) → acquaintance (0.2) → familiar (0.4) → friend (0.6) → confidant (0.8). Characters without explicit thresholds use the application-level `RelationshipStageConfig` defaults. Per-character overrides are parsed as `RelationshipStageConfigSpec` (already exists in agentic-yaml) and compiled via `CognitionCompiler.compileRelationshipStage()`.
**Alternatives:**
- Separate familiarity.yaml — cleaner separation but scatters character definition across more files, diverging from the pattern where drives, norms, beliefs, and relationships all live in social-config.yaml.
- Application-level defaults only — simpler but loses per-character variation. A suspicious character like Hooded Claw should require more interactions to reach "friend" than sociable Penelope.
**Rationale:** social-config.yaml already defines per-character drives, goals, norms, beliefs, and relationships. Familiarity thresholds are character-specific social configuration — they belong with the rest. The per-character pattern is established by D9 (reinforcement mappings). Using the existing `RelationshipStageConfig` and `StageTier` types — already in blocks-core — avoids creating redundant types.
**Trade-offs:** social-config.yaml grows slightly larger. Marginal — it already handles six configuration categories per character.
**Sources:** RelationshipStageConfig.java, StageTier.java, RelationshipStageConfigSpec.java, CognitionCompiler.compileRelationshipStage(), SocialConfig.java, social-config.yaml, issue #70
**Exploration:** quick
**Revised by:** R1-02 (decision review) — original choice described only 4 stages (missing "familiar" tier). Existing `RelationshipStageConfig.defaults()` in blocks-core already defines 5 tiers. Adopted the existing model.
**Status:** revised

## D20: Code location — blocks-core (existing infrastructure)

**Choice:** Relationship stage computation reuses existing blocks-core infrastructure: `RelationshipStageConfig`, `StageTier`, `RelationshipStageConfigSpec`, `CognitionCompiler.compileRelationshipStage()`, and `UserModelOrchestrator.computeFamiliarity()`. The new `RelationshipStagePhase` (consolidation) and `OverlayFamiliarityPropertyModel` (property constants) are added to blocks-core alongside the existing types. Wacky-manor provides YAML configuration and rendering integration.
**Alternatives:**
- Wacky-manor only — keeps it application-level but duplicates the familiarity computation already in blocks-core.
- New blocks-relationship module — maximum isolation but unnecessary when the types and formula already exist in blocks-core.
**Rationale:** `UserModelOrchestrator.computeFamiliarity()` already implements the exact formula: volume factor + positive/negative weighting + time decay. `RelationshipStageConfig.resolveStage()` already maps familiarity score to stage name. No new computation logic is needed — only a consolidation phase that calls the existing function and persists results to overlay nodes. This is a wiring task, not an algorithm task.
**Trade-offs:** `computeFamiliarity()` is currently a static method inside `UserModelOrchestrator`. It may need to be extracted to a utility class if we want to avoid a dependency on UserModelOrchestrator from the consolidation phase. Alternatively, the consolidation phase can call it directly since it's static and side-effect-free.
**Sources:** UserModelOrchestrator.computeFamiliarity(), RelationshipStageConfig, StageTier, OverlayTrustPropertyModel, issue #70
**Exploration:** quick
**Revised by:** R1-02, R1-03 (decision review) — original choice proposed creating new familiarity computation logic. Reviewer identified that `RelationshipStageConfig`, `StageTier`, and `computeFamiliarity()` already exist in blocks-core with decay and sentiment weighting.
**Depends on:** D19 (per-character config)
**Status:** revised

## D21: Computation and persistence — RelationshipStagePhase consolidation

**Choice:** New `RelationshipStagePhase` (ConsolidationPhase implementation) in blocks-core. Runs during sleep consolidation. For each agent, scans the `people` subgraph for overlay nodes owned by that agent, queries CaseMemoryStore for relationship memories per pair using `RelationshipQuery.forPair()`, classifies interactions as positive/negative/neutral from Memory PAD pleasure values, calls `UserModelOrchestrator.computeFamiliarity(positive, negative, neutral, stageConfig, ticksSinceLastInteraction)`, maps result via `stageConfig.resolveStage(score)`, persists on overlay nodes.
**Alternatives:**
- On-demand in rendering — compute familiarity by counting memories at render time. Simpler but queries CaseMemoryStore on every prompt construction, scaling poorly with memory volume.
- Event-driven on ingest — update familiarity every time a memory is ingested. Real-time but couples memory recording to stage computation.
**Rationale:** Follows the established consolidation pattern. The formula already exists in `UserModelOrchestrator.computeFamiliarity()` — it uses volume factor (`1.0 - 1.0 / (1.0 + total * 0.1)`), positive/negative weighting from `RelationshipStageConfig`, and time-based decay. No new algorithm needed.
**Interaction classification:** Memories with `pleasure > 0.1` → positive. `pleasure < -0.1` → negative. Otherwise → neutral. Thresholds from Memory's existing PAD values (the same pleasure dimension the system already uses for emotional state).
**Overlay property model:** `familiarity-score` (double, 0.0–1.0), `familiarity-stage` (string: stranger/acquaintance/familiar/friend/confidant — 5 tiers matching `RelationshipStageConfig.defaults()`), `familiarity-interaction-count` (int, raw count for diagnostics).
**Priority ordering:** After DriveAdaptationPhase (~17) — relationship stage depends on accumulated interactions, not on drive state. Priority ~18. Verified: clean gap between Experience@15 and Merge@20.
**Trade-offs:** Stage updates are not real-time — a character who has many interactions in one turn won't see their stage change until the next consolidation cycle. Acceptable — relationship evolution should feel gradual, not instantaneous.
**Sources:** UserModelOrchestrator.computeFamiliarity(), RelationshipStageConfig, TrustConsolidationPhase pattern, DriveAdaptationPhase (priority ordering), OverlayTrustPropertyModel, CaseMemoryStore, RelationshipQuery, issue #70, GE-20260820-d9129a (volume factor)
**Exploration:** quick
**Revised by:** R1-03 (decision review) — original choice proposed a standalone volume-only formula ignoring existing decay and sentiment parameters. `computeFamiliarity()` already implements volume factor + positive/negative weighting + decay. Adopted the existing implementation.
**Depends on:** D20 (blocks-core location, existing computeFamiliarity), D19 (configurable thresholds)
**Status:** revised

## D22: Behavioral gating — trust-unlocked behaviors only; adversarial behaviors stay drive-gated

**Choice:** Add stage-aware gating to ManorContextStrategy for trust-unlocked behaviors only: `shouldDisclose(String stage)` (requires ≥ friend), `shouldCooperate(String stage)` (requires ≥ acquaintance). Adversarial behaviors (scheming, suspicion) remain gated by drive intensity via the existing `shouldCompareSocially()` — they are NOT stage-gated. The stage influences *how* a character schemes (crude against strangers, sophisticated against confidants), but not *whether* they scheme.
**Alternatives:**
- Stage-gate all behaviors including adversarial — treats scheming as a skill tree unlock. Semantically wrong: a villain schemes *more* against unknown threats, not less. Stage and motivation are orthogonal dimensions.
- Separate StageGate utility — cleaner separation but splits social behavior gating across two classes.
**Rationale:** Familiarity and behavioral motivation are orthogonal dimensions. Disclosure and cooperation are familiarity-gated — trust must be earned. Scheming and suspicion are drive-gated — character motivation determines whether the character acts adversarially at all. The existing `shouldCompareSocially()` already handles the motivation dimension via D9's drive intensity mechanism. Adding stage-gating only for trust-unlocked behaviors preserves this clean separation.
**Stage-to-behavior mapping (trust-unlocked only):** Stranger → surface interactions only, no disclosure. Acquaintance → basic cooperation, norm-gated disclosure. Familiar → cooperation preference, limited trust disclosure. Friend → trust disclosure, alliance formation. Confidant → full disclosure, strong loyalty bias.
**Trade-offs:** ManorContextStrategy grows moderately. The adversarial behavior gating is already handled by `shouldCompareSocially` — no new code needed for that dimension.
**Adapted intensity requirement:** Once D14's seeder populates drive nodes in the COGNITIVE subgraph, `shouldCompareSocially()` must read adapted drive intensities from MindMap nodes (queried by `cognitiveKind: "drive-intensity"` + `agent-id` + drive-type ∈ SOCIAL_AWARENESS_DRIVES), not from static `SocialConfig.Drive` records. The static config serves as fallback only when no seeded drive nodes exist. Without this change, drive adaptation (D11) modifies intensities that are rendered (D15) but never influence behavioral gating — making adaptation functionally inert for behavioral decisions. CharacterCognition already has access to `mindMapStore`, `agentId`, and `tenantId` — the call site in `renderSocialAwareness` can query adapted intensities and pass them to `shouldCompareSocially()`.
**Sources:** ManorContextStrategy.shouldCompareSocially(), issue #70 behavioral gates table
**Exploration:** quick
**Revised by:** R1-01 (decision review) — original choice included `shouldScheme` as a stage-gated method. Reviewer demonstrated that adversarial behaviors are drive-motivated, not familiarity-gated. Adversarial review R1-02 — added adapted intensity requirement; `shouldCompareSocially()` must read from MindMap nodes, not static SocialConfig.
**Depends on:** D19 (5-tier stage model), D21 (stage persisted on overlay), D11 (drive adaptation writes adapted intensities), D14 (seeder writes drive nodes)
**Status:** revised

## D23: Perception rendering — stage-gated in Social Awareness (5 tiers)

**Choice:** Stage-gate PerceptionTranslator output and integrate stage context into the existing `renderSocialAwareness` section. Stage label prefixes each character entry: "Peter Perfect (friend): seems more positive than you."
**Alternatives:**
- Always render, add stage context — keep existing perception output for all stages, prepend relationship framing. Simpler change but unrealistic — a stranger shouldn't have fine-grained emotional perception of someone they just met.
- Separate "Relationship Context" observation section — clear separation but adds yet another section to an already-rich prompt.
- RelationshipStagePromptSection in blocks-core — maximum platform reuse but heavy for rendering stage labels.
**Rationale:** Social Awareness is already where relationship perceptions are rendered. Adding stage context to existing entries is natural — the section answers "what do you notice about nearby characters?" The stage determines how much you notice. Avoiding a new section keeps prompt size bounded.
**Rendering tiers (5 stages):** STRANGER → skip (no perception for unknowns). ACQUAINTANCE → dominant dimension only, no trajectory. FAMILIAR → dominant dimension + second dimension hint, no trajectory. FRIEND → full dimension text, trajectory included. CONFIDANT → full output with trajectory and explicit stage note.
**Trade-offs:** PerceptionTranslator gains a stage parameter, changing its signature. Callers must supply stage. Acceptable — there's one caller (renderSocialAwareness).
**Sources:** PerceptionTranslator.translate(), CharacterCognition.renderSocialAwareness(), issue #70
**Exploration:** quick
**Revised by:** R1-02 (decision review) — original rendering tiers had 4 stages. Added "familiar" tier to match the 5-tier `RelationshipStageConfig.defaults()`.
**Depends on:** D21 (stage persisted on overlay), D22 (behavioral gating), D19 (5-tier model)
**Status:** revised

---

## examples#67 — Belief Revision from Contradicting Evidence

## D24: Pipeline slot — new BeliefRevisionPhase

**Choice:** New `BeliefRevisionPhase` (ConsolidationPhase implementation) at `@Priority(16)` — after ExperienceConsolidation@15 (which graduates events into cognitive nodes), before DriveAdaptation@17. Runs as a separate phase in the consolidation pipeline.
**Alternatives:**
- Inside ExperienceConsolidationPhase — inline contradiction check after each graduation. Tighter coupling, and ExperienceConsolidationPhase is in neocortex (platform) — adding LLM calls there creates a platform dependency on AgentProvider.
- Event-driven via CDI event — ExperienceConsolidationPhase fires GraduationCompleted, listener handles contradiction. Decoupled but loses consolidation context and phase ordering.
**Rationale:** Clean SRP — experience graduation and belief revision are separate concerns. The phase reads newly graduated nodes (created by ExperienceConsolidation@15) and checks them against existing Belieflike nodes. Follows the same pattern as DriveAdaptationPhase (reads graduated nodes, performs domain-specific analysis).
**Trade-offs:** Second pass over the subgraph's nodes. Acceptable — Belieflike nodes are few per agent (4-6 seeded beliefs) and the scan is by trait, not a full search.
**Sources:** ExperienceConsolidationPhase (@Priority 15), DriveAdaptationPhase (@Priority 17), ConsolidationPhase SPI, issue #67
**Exploration:** quick
**Status:** captured

## D25: LLM strategy — batch per agent

**Choice:** One LLM call per agent per consolidation cycle. The prompt presents all current beliefs and all newly graduated evidence nodes, asking: "Does any of this evidence contradict any of these beliefs? For each contradiction, explain which belief and why." The LLM returns structured JSON identifying contradicted beliefs and reasoning.
**Alternatives:**
- Pairwise per (belief, evidence) — one LLM call per pair. More precise but O(beliefs × evidence) calls per cycle. Expensive for characters with many beliefs.
- Pre-filter via embedding similarity, then LLM — reduces calls but adds embedding dependency and complexity.
**Rationale:** Amortizes LLM cost — one call checks all beliefs against all evidence for an agent. Characters have 4-6 beliefs and consolidation graduates ~5-20 nodes per cycle, so the prompt is bounded and fits in a single context window. The batch approach also gives the LLM cross-belief context — it can assess whether evidence contradicts one belief while reinforcing another.
**Trade-offs:** If a character accumulates many beliefs over time (dozens), the prompt may grow. Mitigated by only including active (non-superseded) beliefs. Also, the LLM may miss subtle contradictions when processing in batch. Acceptable at this stage — can switch to pairwise for high-value beliefs later.
**Sources:** UserModelOrchestrator LLM synthesis pattern, AgentProvider SPI, issue #67
**Exploration:** quick
**Status:** captured

## D26: Revision model — gradual confidence decay with variable strength, then supersede

**Choice:** Each contradicting event reduces the belief's confidence by `baseDecay × contradictionStrength`, where `baseDecay` is configurable (default: 0.15) and `contradictionStrength` (0.0–1.0) is provided by the LLM's structured JSON output (D25). When confidence drops below a configurable threshold (default: 0.3), the LLM generates a revised belief text, a new Belieflike node is created in the same subgraph, and the old one is superseded via `MindMapStore.supersede()`. The superseded belief remains in the mindmap — characters can "remember what they used to believe."
**Alternatives:**
- Immediate supersede on contradiction — any contradiction triggers immediate supersession. Dramatic but unrealistic — one contradicting event shouldn't overturn a long-held belief.
- Uniform decay (original choice) — every contradiction applies the same `baseDecay` regardless of strength. Mismodels: 3 weak contradictions ("mildly inconsistent") would supersede a belief that was never strongly contradicted.
- Accumulate count then batch-revise — track contradiction count on the belief node. Simpler than confidence decay but loses the nuance of varying contradiction strength.
**Rationale:** Variable-strength decay models how real belief revision works — a strong contradiction ("directly disproven by event X") erodes confidence more than a weak one ("mildly inconsistent with event Y"). The LLM already outputs structured JSON with reasoning per contradiction (D25); adding `contradictionStrength: 0.0–1.0` to the schema is a single field. The multiplication `baseDecay × contradictionStrength` is a single line. The confidence model already exists (`Confidence.withValue()`) and beliefs are seeded at 0.8 confidence. `MindMapStore.supersede()` already tracks supersession history via `SupersessionStatus`.
**Decay config:** `beliefDecayPerContradiction` (base decay, default: 0.15), `beliefSupersessionThreshold` (default: 0.3). The LLM output schema includes `contradictionStrength: 0.0–1.0` per identified contradiction; effective decay = `beliefDecayPerContradiction × contradictionStrength`. Configurable per application via a `BeliefRevisionConfig` record.
**Trade-offs:** The LLM-generated `contradictionStrength` is non-deterministic — the same contradiction may receive slightly different strength scores across runs. Acceptable — belief revision is inherently subjective, and the strength score modulates a configurable base decay, so extreme variation is bounded by the base value.
**Sources:** Confidence.withValue(), MindMapStore.supersede(), SupersessionStatus, issue #67
**Exploration:** quick
**Revised by:** Adversarial review R1-08 — original uniform decay treated all contradictions equally. Variable strength adds near-zero implementation cost (one schema field, one multiplication) while preventing incorrect supersession from accumulated weak contradictions.
**Depends on:** D24 (separate phase), D25 (LLM structured JSON output)
**Status:** revised

## D27: Code location — blocks-core

**Choice:** `BeliefRevisionPhase` lives in blocks-core alongside DriveAdaptationPhase and RelationshipStagePhase. A `BeliefRevisionConfig` record provides configurable decay and threshold parameters. Wacky-manor provides only the `AgentProvider` wiring (already available via CDI).
**Alternatives:**
- Wacky-manor only — application-level. But belief revision is a generic social cognition concept — any agent with seeded beliefs should support revision.
- New blocks-belief module — maximum isolation but heavy for one phase and one config record.
**Rationale:** blocks-core already depends on AgentProvider (via UserModelOrchestrator). Adding an LLM-calling consolidation phase follows the same dependency pattern. The belief revision algorithm is domain-agnostic — it works with any Belieflike nodes regardless of how they were seeded.
**Trade-offs:** blocks-core gains another consolidation phase and an LLM dependency in its consolidation pipeline. Acceptable — the LLM call is bounded (one per agent per cycle) and fails gracefully (no revision on failure, logged as warning).
**Sources:** DriveAdaptationPhase, UserModelOrchestrator (AgentProvider usage), blocks-core pom.xml, issue #67
**Exploration:** quick
**Depends on:** D24 (phase design), D25 (LLM strategy)
**Status:** captured

## D28: Revised belief text — LLM-generated

**Choice:** When confidence drops below threshold, the same LLM call that detected the final contradiction generates the revised belief text. Prompt: "Given that [character] believed '[old belief]' but evidence shows [contradicting evidence], what should they now believe? Write a single sentence from their perspective." The revised text becomes the name of the new Belieflike node.
**Alternatives:**
- Template-based — "[Old belief] is no longer certain — evidence suggests [summary]." Predictable but mechanical, doesn't capture character voice.
- No revised text — just supersede the old belief with no replacement. Simplest but the character loses the positive knowledge gained from the contradiction.
**Rationale:** LLM-generated text reads as something the character would think. The LLM already has the context (beliefs + evidence) from the contradiction detection call. Generating the revised text in the same call (or a follow-up) adds minimal cost. The revised belief inherits INFERRED confidence origin (vs STATED for seeded beliefs) — distinguishing authored from evolved beliefs.
**Trade-offs:** LLM-generated text is non-deterministic — the same contradiction may produce different revised beliefs on different runs. Acceptable — belief revision is inherently subjective, and the text is for cognitive rendering, not a deterministic data pipeline.
**Sources:** AgentProvider, Confidence (INFERRED origin), MindMapStore.addNode(), issue #67
**Exploration:** quick
**Depends on:** D26 (gradual decay triggers revision), D25 (LLM strategy)
**Status:** captured

## D29: Consolidation phases may include LLM calls

**Choice:** BeliefRevisionPhase (D24/D25) introduces LLM calls into the consolidation pipeline. This is explicitly permitted under the following constraints: (1) failure isolation — ConsolidationScheduler's per-phase try-catch catches exceptions, logs failures as WARNING, and subsequent phases (DriveAdaptation@17, RelationshipStage@18) proceed normally; (2) latency — consolidation runs asynchronously on a daemon thread with IdleTracker gating (`isIdle(Duration.ofMinutes(1))`), so LLM latency (2–30s) does not block user interactions; (3) cost — bounded to one LLM call per agent per consolidation cycle; (4) non-determinism — intentional for belief revision (the LLM judges what contradicts what); deterministic phases remain deterministic.
**Alternatives:**
- Deterministic-only consolidation — restrict all phases to synchronous, bounded-time operations. Precludes belief revision and any future LLM-powered cognitive processing in the consolidation pipeline.
- Separate LLM pipeline — run LLM-calling phases in a different scheduler with independent failure/timeout handling. Clean isolation but unnecessary indirection when ConsolidationScheduler already provides per-phase exception handling and idle-time gating.
**Rationale:** ConsolidationScheduler's existing architecture handles the key concerns: phases run sequentially with per-phase try-catch (verified in decompiled bytecode), failures are logged as WARNING and do not cascade, and the scheduler runs on a single daemon thread gated by `IdleTracker.isIdle()`. The qualitative difference (deterministic → LLM-calling) is real but the infrastructure already supports it. Making this an explicit decision surfaces the architectural shift for future phases that may also require LLM calls.
**Trade-offs:** Future consolidation phases that add LLM calls must respect the same constraints: bounded calls per cycle, graceful failure (skip on error, log WARNING), and no blocking of subsequent phases. No per-phase timeout enforcement exists in ConsolidationScheduler — phases are trusted to return. An LLM call that hangs indefinitely would block subsequent phases until the scheduler's next tick. Mitigation: AgentProvider implementations should enforce their own call timeouts.
**Sources:** ConsolidationScheduler (per-phase try-catch, IdleTracker gating, daemon thread), BeliefRevisionPhase (D24/D25), AgentProvider SPI
**Exploration:** quick (surfaced by adversarial review R1-04)
**Depends on:** D24 (BeliefRevisionPhase), D25 (LLM strategy)
**Status:** captured

---

## examples#68 — Needs Pyramid

## D30: Satisfaction routing — drive→tier mapping, not event→tier

**Choice:** Satisfaction flows through the existing drive reinforcement layer via a global drive→tier mapping table. When DriveAdaptationPhase reinforces a drive, the associated need tier receives a satisfaction bump. The per-character specificity is handled by the existing reinforcement config (D9) — no per-character need mapping required.
**Alternatives:**
- Global event→tier mapping — one table mapping event types directly to tiers. Fails psychological realism: conflict_resolution satisfies SELF_EXPRESSION for Hooded Claw (schemed successfully) but SOCIAL for Penelope (restored harmony). A global mapping can't express this.
- Per-character event→tier mapping — maximum control but redundant with D9's reinforcement config. Two per-character configs describing "what events are meaningful to this character" in different vocabularies, creating configuration burden (17 × 5 × 5 entries) and drift risk.
**Rationale:** The issue specification says "Drive-to-need mapping: each drive type maps to one or more need tiers (social-harmony → Social, scheming → Self-expression, self-preservation → Safety)." The mapping composes through existing infrastructure: event → per-character drive reinforcement (D9, exists) → drive→tier mapping (new, global) → tier satisfaction. This eliminates redundancy, enables the constraint loop ("a drive can't strengthen indefinitely if its need tier is saturated"), and achieves per-character behavior through ~15 drive-type entries instead of 425 per-character entries.
**Trade-offs:** Tiers not reachable through any of a character's drives will only experience decay, never satisfaction. This is intentionally correct — neglected needs creating pressure is the pyramid's purpose — but means characters with few drives will have more tiers under pressure. Mitigated by the non-strict Maslow model (pressure is weighting, not blocking).
**Sources:** Issue #68 (drive-to-need mapping description), D9 (per-character reinforcement), D12 (PAD reward axis), ActionImportanceScorer event types, SocialConfig.Drive
**Exploration:** deep-analysis (first-principles comparison of three architectures)
**Status:** captured

## D31: Saturation constraint — scaling factor on drive learning rate

**Choice:** DriveAdaptationPhase reads current tier satisfaction for each drive's mapped tier and scales its learning rate: `effectiveLR = learningRate × (1 - tierSatisfaction)`. At 100% satisfaction, drive strengthening stops. At 0%, full learning rate. One-cycle delay (reads satisfaction from previous cycle).
**Alternatives:**
- Hard cap with threshold — drive strengthening blocked entirely above a configurable satisfaction threshold (e.g., 0.8). Simpler but creates sudden behavioral cliffs where drives oscillate around the threshold.
- Diminishing returns curve — `effectiveLR = learningRate / (1 + satisfaction²)`. Non-linear, never fully stops. More organic but harder to reason about configuration and tune.
**Rationale:** Linear scaling is smooth, self-balancing, and predictable. A drive with 80% tier satisfaction strengthens at 20% of normal rate — proportional response, no cliffs. The one-cycle delay between satisfaction update and constraint application is acceptable since consolidation runs many cycles over a game session. The formula is a single multiplication, zero overhead.
**Trade-offs:** At very high satisfaction (0.95+), drives effectively freeze. This is by design — saturated needs don't need more reinforcement — but could feel unresponsive if satisfaction accumulates too quickly. Mitigated by tuning decay rate (D32) and satisfaction magnitude.
**Sources:** Issue #68 (constraint loop), D30 (drive→tier mapping), DriveAdaptationPhase learning rate config
**Exploration:** quick
**Depends on:** D30 (drive→tier mapping provides the tier lookup)
**Status:** captured

## D32: Decay model — multiplicative per-tier with configurable defaults

**Choice:** Each tier decays multiplicatively per consolidation cycle: `satisfaction *= (1 - decayRate)`. Per-tier rates with sensible defaults reflecting urgency: Safety=0.15, Tasks=0.10, Social=0.08, Self-expression=0.05, Understanding=0.03. Configurable via Smallrye properties (prefix `casehub.needs-pyramid.decay`) with per-tier overrides. Follows DriveAdaptationConfig pattern.
**Alternatives:**
- Uniform decay — same rate for all tiers. Simpler but loses the urgency hierarchy (Safety should decay faster than Understanding because immediate threats demand faster response).
- Additive decay — `satisfaction -= decayRate`. Linear, but doesn't self-dampen — low satisfaction drops at the same absolute rate as high satisfaction, creating faster urgency spiral than intended.
**Rationale:** Multiplicative decay is self-dampening (same pattern as GE-20260714-439924 for drive adaptation): a tier at 0.8 satisfaction losing 15% drops to 0.68 (-0.12 absolute), while a tier at 0.2 drops to 0.17 (-0.03 absolute). High satisfaction decays faster in absolute terms, low satisfaction decays slower — creating smooth urgency curves rather than cliff drops. Per-tier rates model the urgency hierarchy from the issue: "Safety decays faster than Understanding."
**Trade-offs:** More config knobs (5 decay rates). Acceptable — pre-release, single consumer, and the defaults should work without tuning.
**Decay toward resting level (revised per R1-05):** Unconditional decay toward zero creates false urgency — a character in a peaceful scene reports "critically neglected" Safety after 50 minutes. Revised formula: `satisfaction = restingLevel + (satisfaction - restingLevel) × (1 - decayRate)`. Each tier has a configurable `restingLevel`: Safety=0.6, Tasks=0.3, Social=0.4, Self-expression=0.4, Understanding=0.4. Tiers naturally settle at their resting level without events. Events push satisfaction above baseline (addressed) or below (actively threatened). Safety's high resting level (0.6) models "no news is good news" — absence of threats means safety feels adequate. Tasks' low resting level (0.3) models "obligations accumulate" — even without events, task pressure builds.
**Sources:** Issue #68 (per-tier decay rates), GE-20260714-439924 (multiplicative dampening), DriveAdaptationConfig pattern
**Exploration:** quick
**Revised by:** R1-05 (decision review) — unconditional decay toward zero created false urgency in peaceful environments. Resting-level model replaces decay-toward-zero.
**Status:** revised

## D33: Phase architecture — integrated into DriveAdaptationPhase

**Choice:** Need satisfaction updates are integrated into DriveAdaptationPhase rather than a separate ConsolidationPhase. After computing per-drive rewards (which it already does), DriveAdaptationPhase maps drive→tier (D30) and accumulates tier satisfaction in the same pass. Decay is applied at the end of the phase. The D31 saturation constraint reads current-cycle satisfaction (no stale data).
**Alternatives:**
- Separate NeedSatisfactionPhase@19 (original choice) — clean SRP but D31's saturation constraint already creates a bidirectional data dependency between the phases. DriveAdaptationPhase would need the drive→tier mapping AND tier satisfaction state to implement D31, meaning it already does most of the need-satisfaction work. A separate phase would re-read the same experience nodes, maintain a duplicate cursor, and operate on one-cycle-stale satisfaction for the constraint — duplication masquerading as separation.
- CDI event hybrid — loses consolidation phase ordering.
**Rationale:** D31's scaling factor requires DriveAdaptationPhase to read tier satisfaction AND know the drive→tier mapping. At that point, the phase already has: (a) which drives were reinforced, (b) by how much, (c) which tiers those drives map to. Computing tier satisfaction is a single accumulation step after the existing drive reward computation. Merging eliminates: the duplicate cursor, the re-read of 20 experience nodes, and the one-cycle delay for the saturation constraint. DriveAdaptationPhase gains a cohesive extended responsibility: "adapt drives and track their need satisfaction."
**Trade-offs:** DriveAdaptationPhase grows in scope. Acceptable — the addition is a small accumulation step after the existing drive reward computation, and the responsibility remains cohesive (drive adaptation and its need-satisfaction feedback loop).
**Sources:** DriveAdaptationPhase.accumulateRewards(), D31 (saturation constraint creates bidirectional dependency), D30 (drive→tier mapping)
**Exploration:** quick
**Revised by:** R1-04 (decision review) — bidirectional data dependency from D31 invalidated SRP justification for separation. Also resolves R1-06 (undisclosed dependency) and R1-09 (dual cursor debugging).
**Depends on:** D30 (drive→tier mapping), D31 (saturation constraint), D32 (decay model)
**Status:** revised

## D34: Rendering — NeedsPyramidPromptSection in observation pipeline

**Choice:** New `NeedsPyramidPromptSection` (blocks-core PromptSection) reads satisfaction levels from MindMap and renders them as an observation section. Characters consciously "feel" their needs and can reason about them in dialogue.
**Alternatives:**
- Behind-the-scenes only — satisfaction levels consumed only by GoalRevisionStrategy (#69) for silent goal reordering. Simpler prompt but characters can't reference or reason about their needs.
**Rationale:** Characters should be aware of their internal states — a character who "feels" their social needs are neglected will naturally seek out interaction, reference loneliness in dialogue, and make choices that address the gap. This is richer simulation and consistent with how drives (DrivePromptSection), mood (MoodPromptSection), and beliefs are already rendered as observable cognitive state.
**Trade-offs:** Adds one more observation section to an already-rich prompt. Acceptable — the section is compact (5 tiers, one line each) and provides high narrative value per token.
**Sources:** DrivePromptSection, CharacterDrivePromptSection (D15), MoodPromptSection, CognitionCore.promptSections()
**Exploration:** quick
**Depends on:** D33 (phase persists satisfaction state that the section reads)
**Status:** captured

## D35: Persistence and bootstrapping — MindMap nodes, initial 0.5

**Choice:** Satisfaction state stored as MindMap nodes in the COGNITIVE subgraph, one node per tier per agent. Properties: `cognitiveKind: "need-satisfaction"`, `agent-id`, `tier` (enum name), `satisfaction` (0.0–1.0), `provenance: "need-satisfaction"`. ManorCognitiveSeeder creates all 5 tier nodes per agent at bootstrap with `satisfaction: 0.5` (configurable default). Follows the same node discrimination pattern as drive nodes (D10).
**Alternatives:**
- Extend drive nodes with satisfaction properties — couples drive and need state on the same node. Clean SRP violation.
- New dedicated store — unnecessary when MindMapStore already handles all cognitive state.
**Rationale:** Identical pattern to drive intensity nodes (D10): property-based discrimination in the shared COGNITIVE subgraph, queried by `cognitiveKind` + `agent-id`. Initial 0.5 means characters start with moderate satisfaction — enough that no tier is in urgent neglect, but low enough that the first few events create visible satisfaction changes. The seeder already handles drives, beliefs, and goals — adding need tiers is natural.
**Trade-offs:** 5 additional MindMap nodes per agent (85 total for 17 characters). Negligible.
**Sources:** D10 (drive node pattern), ManorCognitiveSeeder, MindMapStore
**Exploration:** quick
**Depends on:** D33 (DriveAdaptationPhase reads/writes these nodes as part of integrated satisfaction tracking)
**Status:** captured

## D36: Satisfaction magnitude — small increment with volume dampening

**Choice:** Base satisfaction increment per event: configurable `satisfaction-increment` (default: 0.05). Scaled by the reward magnitude: `effectiveSatisfaction = satisfactionIncrement × |rewardSignal|`, where `rewardSignal` is the PAD value from the experience node (same axis as the drive's reward axis per D12). Volume dampening applied when multiple events address the same tier in one pass: `effectiveSatisfaction = totalSatisfaction / (1 + log(count))` — same formula as DriveAdaptation (GE-20260820-d9129a). Only positive reward signals produce satisfaction — negative outcomes (bad experiences) don't satisfy the need.
**Alternatives:**
- Fixed increment regardless of reward magnitude — simpler but a barely-positive conflict resolution produces the same satisfaction as a triumphant one.
- No volume dampening — many small events in one cycle could spike satisfaction. Contradicts GE-20260820-d9129a.
**Rationale:** The issue says "Satisfaction from a single action should be small (volume factor, not single-event jumps)." Scaling by reward magnitude means strong positive outcomes (high pleasure/dominance) produce more satisfaction than ambiguous ones. Volume dampening ensures a consolidation pass with many observations doesn't dominate tier satisfaction.
**Trade-offs:** Non-deterministic effective satisfaction (depends on PAD values, which are LLM-generated). Acceptable — same non-determinism as drive adaptation.
**Sources:** GE-20260820-d9129a (volume factor), D12 (PAD reward axis), DriveAdaptationPhase volume dampening
**Exploration:** quick
**Depends on:** D30 (drive→tier mapping), D33 (NeedSatisfactionPhase)
**Status:** captured

## D37: Rendering format — qualitative prose per tier

**Choice:** `NeedsPyramidPromptSection` renders satisfaction as qualitative natural language under an "Inner Needs" heading. Each tier maps satisfaction to a band with tier-specific prose: Safety→"sense of safety", Tasks→"task commitments", Social→"social bonds", Self-expression→"self-expression", Understanding→"curiosity." Bands: 0.0–0.2 (critically neglected, strong urgency language), 0.2–0.4 (neglected), 0.4–0.6 (adequate), 0.6–0.8 (well-met), 0.8–1.0 (fulfilled). Neglected tiers include a brief contextual note (e.g., "there's much you don't understand yet").
**Alternatives:**
- Quantitative ("Safety: 0.35") — precise but mechanical, breaks character immersion.
- Hybrid ("Safety: neglected (0.35)") — maximum information but mixes game mechanics with narrative voice.
**Rationale:** Consistent with how mood, drives, and beliefs are rendered as prose the character can internalize. Qualitative labels let the LLM interpret needs as psychological states rather than game metrics, producing more natural dialogue and decision-making.
**Trade-offs:** Loss of numeric precision. Acceptable — the LLM should respond to "neglected" vs "well-met" rather than 0.35 vs 0.72. The bands provide sufficient granularity.
**Sources:** DrivePromptSection rendering style, MoodPromptSection, CharacterDrivePromptSection (D15)
**Exploration:** quick
**Depends on:** D34 (rendering in prompt), D35 (persistence provides data)
**Status:** captured

## D38: Drive→tier mapping location — application-level config

**Choice:** The drive→tier mapping is provided as application-level configuration. Wacky-manor declares the mapping via a `NeedTierMappingProvider` CDI bean (or Smallrye config map). The mapping is a `Map<String, Set<NeedTier>>` — drive-type string → set of tiers. ~15 entries for wacky-manor's drive types. blocks-core defines the `NeedTier` enum and the SPI; applications provide the mapping.
**Alternatives:**
- Hardcoded in blocks-core — locks the drive→tier vocabulary to one application's drive types.
- YAML file — readable but introduces a new YAML schema for a small lookup table.
**Rationale:** Drive types are free-form strings defined per application ("scheming", "gallantry", "social-harmony"). The mapping from these application-specific strings to the universal NeedTier enum is inherently application-level. Follows the same pattern as DriveReinforcementConfig (D13) — blocks-core defines the SPI, applications provide configuration.
**Trade-offs:** Each new drive type requires a mapping entry. Acceptable — new drives are infrequent and the mapping is self-documenting (the drive name implies the tier).
**Sources:** D13 (DriveReinforcementConfig SPI pattern), NeedTier enum, SocialConfig.Drive
**Exploration:** quick
**Depends on:** D30 (drive→tier mapping architecture)
**Status:** captured

## D39: Tier taxonomy — simulation-pragmatic, not psychologically grounded

**Choice:** The 5 tiers (SAFETY, TASKS, SOCIAL, SELF_EXPRESSION, UNDERSTANDING) are simulation-pragmatic categories chosen for the wacky-manor game context, not a direct mapping of Maslow's hierarchy. TASKS replaces Maslow's "esteem" (which splits into TASKS for obligation/duty and SELF_EXPRESSION for identity/values). UNDERSTANDING replaces "self-actualization" (which in a game context is curiosity and sense-making, not transcendence).
**Alternatives:**
- Maslow-faithful tiers (physiological, safety, belongingness, esteem, self-actualization) — psychologically grounded but "physiological" and "self-actualization" have no meaningful game semantics. Simulated agents don't have metabolisms or transcendent experiences.
- Configurable tier list (application defines its own tiers) — maximum flexibility but over-engineering for pre-release. The enum can be extended later if needed.
**Rationale:** The pyramid serves the simulation, not psychology. TASKS creates pressure for obligation fulfillment (game-relevant). UNDERSTANDING creates pressure for exploration and mystery-solving (game-relevant). Future applications can extend the enum or define application-specific tier sets if the 5-tier model doesn't fit.
**Trade-offs:** The NeedTier enum in blocks-core is wacky-manor-pragmatic, not universal. If a future clinical or devtown application adopts it, TASKS as a fundamental need tier may feel domain-specific. Acceptable — the enum can be extended, and the resting-level model (D32) makes unused tiers inert.
**Sources:** Issue #68 (tier definition), Maslow's hierarchy of needs
**Exploration:** quick (surfaced by R1-07, decision review)
**Status:** captured

## D40: Hierarchy model — non-strict (priority weighting, not blocking)

**Choice:** The pyramid uses non-strict hierarchy: neglected tiers increase goal priority weighting but do not block pursuit of higher tiers. A character with critically neglected Safety can still pursue Understanding goals — Safety-related goals just receive stronger priority weighting.
**Alternatives:**
- Strict Maslow hierarchy — lower tiers must be satisfied before higher tiers can be pursued. Creates dramatic survival-focused behavior but makes characters monotonically fixated on their lowest-satisfied tier, reducing behavioral diversity.
- Threshold-gated — higher tiers suppressed only when lower tiers drop below a critical threshold (e.g., Safety < 0.2 blocks Understanding). Compromise but introduces cliff behavior and arbitrary thresholds.
**Rationale:** Issue #68 specifies: "The pyramid is not strict Maslow — lower tiers don't completely block higher ones; they increase priority weighting." Non-strict hierarchy produces richer character behavior — a character can feel unsafe AND curious simultaneously, with the relative urgency influencing but not dictating action. This is consumed by GoalRevisionStrategy (#69) which uses satisfaction levels as priority weights in goal reordering.
**Trade-offs:** The pyramid's effect on behavior is subtle (weighting) rather than dramatic (blocking). This means the pyramid primarily influences goal ordering rather than creating visible behavioral shifts. Acceptable — dramatic shifts should come from events and drive changes, not from a background needs mechanic.
**Sources:** Issue #68 (non-strict specification), GoalRevisionStrategy (#69), D31 (scaling factor is the primary behavioral constraint)
**Exploration:** quick (surfaced by R1-08, decision review)
**Status:** captured

## D41: Bidirectional satisfaction — negative events decrease tier satisfaction

**Choice:** Extend the satisfaction model to handle negative events. When `effectiveReward < 0`, tier satisfaction is decremented by `dissatisfactionIncrement × |padValue|`. This makes the model bidirectional: positive events push satisfaction above resting level, negative events push it below, and decay restores equilibrium.
**Alternatives:**
- Positive-only model (original D36) — satisfaction can only increase from events, decay provides the only downward pressure. Fails for active threats: a character in danger reports "adequate" Safety because negative events are silently ignored.
**Rationale:** The resting-level model (D32 revised) establishes an equilibrium that events should perturb in both directions. A character whose identity is blown should feel actively unsafe (Safety below resting level), not just "uneventful." The `dissatisfactionIncrement` config (default: 0.05, same as `satisfactionIncrement`) keeps the model symmetric and tunable.
**Trade-offs:** One additional config knob. Marginal.
**Sources:** Spec review R1-01 (concrete failure scenario with Hooded Claw's identity exposure)
**Exploration:** quick (surfaced by spec review R1-01)
**Depends on:** D36 (satisfaction magnitude), D32 (resting-level decay provides recovery path)
**Status:** captured

## D42: Rendering filter — only render reachable tiers, suppress for driveless characters

**Choice:** NeedsPyramidPromptSection only renders tiers reachable through the character's drives. Characters with no drive nodes in the COGNITIVE subgraph have the entire "Inner Needs" section suppressed. For characters with drives, tiers not reachable through any drive→tier mapping are omitted.
**Alternatives:**
- Render all tiers for all characters — creates character-incoherent prompts. Hooded Claw sees "obligations are piling up" (TASKS at resting 0.3 = "neglected") despite having no task orientation.
- Adjust resting levels to avoid "neglected" band — hides the problem without solving it. Unreachable tiers still provide static, meaningless prompt content.
**Rationale:** The pyramid's model tracks all tiers (GoalRevisionStrategy needs them), but the character should only perceive tiers connected to their motivational system. A villain doesn't "feel" task obligation pressure. The filter checks for `cognitiveKind: "drive-intensity"` nodes and the drive→tier mapping at render time.
**Trade-offs:** NeedsPyramidPromptSection gains a dependency on NeedTierMappingProvider and must query drive nodes. Acceptable — it already reads MindMapStore for satisfaction nodes.
**Sources:** Spec review R1-02 and R1-07 (TASKS resting level renders "neglected" for characters without TASKS drives, 12 driveless characters get inert prompts)
**Exploration:** quick (surfaced by spec review)
**Depends on:** D34 (rendering), D38 (mapping provider)
**Status:** captured

---

## examples#69 — Goal Prioritization from Unmet Needs

## D43: LLM role — consume numeric satisfaction, not re-assess

**Choice:** The LLM receives #68's computed numeric satisfaction levels directly and uses them to reorder goals. It does not independently re-assess satisfaction from cognitive context.
**Alternatives:**
- LLM re-assesses satisfaction from full cognitive state (beliefs, drives, norms, events) — richer but potentially contradicts #68's computed values, duplicates the satisfaction model, and adds LLM cost.
- Hybrid (numbers + cognitive context, LLM adjusts) — maximum information but complex prompt and unclear which signal the LLM should trust when they disagree.
**Rationale:** #68 already computes satisfaction per tier from drive reinforcement with decay and bidirectional events. Duplicating this assessment via LLM adds cost and non-determinism for no clear benefit. The LLM's value here is in *goal reordering* — judging which goals best address neglected needs given the character's personality — not in re-computing satisfaction. Per GE-20260919-3f610d: this is an LLM-as-reasoner consumer, so raw numeric satisfaction is the correct format.
**Trade-offs:** The LLM sees only the satisfaction numbers, not the events that produced them. It can't reason about "why" a tier is neglected, only "that" it is. Acceptable — the character's drives and goals provide enough context for reordering.
**Sources:** GE-20260919-3f610d (numeric vs qualitative), #68 (NeedSatisfactionPhase), ManorGoalRevisionStrategy, issue #69
**Exploration:** quick
**Depends on:** D30 (drive→tier mapping), D35 (satisfaction persistence)
**Status:** captured

## D44: Data access — inject MindMapStore into ManorGoalRevisionStrategy

**Choice:** ManorGoalRevisionStrategy receives MindMapStore via CDI constructor injection. In `revise()`, it queries for `cognitiveKind: "need-satisfaction"` nodes matching the agent-id from GoalRevisionContext, extracts per-tier satisfaction values, and includes them in the LLM prompt.
**Alternatives:**
- Extend GoalRevisionContext in engine-api — clean API but requires modifying the platform jar for every consumer, coupling the platform to the needs pyramid concept.
- Wrapper strategy pattern — NeedsAwareGoalRevisionStrategy wraps the existing strategy. Clean separation but unnecessary indirection for a single consumer.
**Rationale:** Follows the same pattern as CharacterDrivePromptSection and NeedsPyramidPromptSection — CDI-injected MindMapStore, query by cognitiveKind + agent-id. No platform jar changes needed for data access. GoalRevisionContext provides `agentId()` and `tenancyId()` which are sufficient for the MindMapStore query.
**Trade-offs:** ManorGoalRevisionStrategy gains a dependency on MindMapStore. Acceptable — this is an application-level strategy, not a platform SPI implementation.
**Sources:** CharacterDrivePromptSection (MindMapStore injection pattern), ManorGoalRevisionStrategy.java, GoalRevisionContext
**Exploration:** quick
**Depends on:** D43 (numeric consumption requires reading satisfaction nodes)
**Status:** captured

## D45: Output model — REPRIORITIZE action in GoalRevisionAction

**Choice:** Add `REPRIORITIZE` to the `GoalRevisionAction` enum in engine-api. Add an optional `Double newPriority` field to `GoalRevisionProposal.RevisedGoal`. ManorGoalEvaluator handles REPRIORITIZE by updating the goal's priority value.
**Alternatives:**
- Overload REVISE — when revisedDescription is null and newPriority is present, treat as reprioritization. No enum change but conflates two semantically distinct operations.
- Separate NeedsGoalPrioritizer SPI — parallel pipeline for priority adjustments. Clean separation but unnecessary for a single consumer, and fragments the goal revision flow.
**Rationale:** REVISE changes what a goal means. REPRIORITIZE changes how urgent it is. These are semantically distinct operations that the LLM should be able to recommend independently. Pre-release with a single consumer — extending the enum is the correct approach.
**Trade-offs:** Requires modifying engine-api (cross-repo). Acceptable — pre-release, and ManorGoalEvaluator (the only consumer) is in this repo.
**Sources:** GoalRevisionAction enum, GoalRevisionProposal.RevisedGoal, ManorGoalEvaluator.evaluate(), issue #69
**Exploration:** quick
**Depends on:** D43 (LLM needs an action to express priority changes)
**Status:** captured
