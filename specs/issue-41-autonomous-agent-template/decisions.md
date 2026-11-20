# Decisions — Issue #41: Autonomous Agent Template

## D1: Three-layer cognitive architecture

**Choice:** Use the engine's public API (`casehub-engine-api`) for cognitive SPIs and data models where they are cleanly decoupled from the Worker execution model, use the neocortex directly for the memory layer, and implement only thin orchestration glue in the manor for the parts currently coupled to Workers.

**Alternatives:**
- Full engine integration (model game loop as CaseInstance with Workers) — fights the engine's process-orchestration design; Workers/Bindings don't map to continuous perceive-think-act loops
- Full re-implementation in manor (ignore engine entirely) — duplicates cognitive concepts without reusing the engine's public SPIs and data models; divergence risk

**Rationale:** The engine's cognitive capabilities (goal lifecycle, plan adaptation, reflection) are the platform value. The Worker/Binding execution model is context-specific. The engine's public API layer (`casehub-engine-api`) provides clean SPIs (`GoalFormationStrategy`, `GoalRevisionStrategy`) and data models. The manor depends on these where they are genuinely decoupled from Workers. Only the evaluator orchestration (when to call the SPIs) needs to be implemented in the manor.

**Trade-offs:** Some engine API types (`ReflectionTriggerConfig`, `CaseDefinition`) are semantically coupled to the Worker model and cannot be used directly — the manor uses application.properties for these concerns until the engine generalizes them. A platform issue should be filed to extract cognitive SPIs into a standalone module that doesn't require `CaseDefinition`.

**Revision (from decision review R1-04, R1-06, R1-15):** Original decision claimed all SPIs and data models were clean public APIs. Review identified that `ReflectionTriggerConfig.importanceWeights` uses Worker outcome types (SUCCESS/DECLINED/EXPIRED) and `GoalFormationContext` requires `CaseDefinition`, which drags the case model in. Revised to be explicit about which engine API types are usable vs coupled.

**Exploration:** deep-analysis
**Status:** revised

## D2: Standalone reflection

**Choice:** Reflection is a standalone LLM call, separate from the per-turn action loop, triggered periodically by experience accumulation thresholds. Config via application.properties (not `ReflectionTriggerConfig`).

**Alternatives:**
- Integrated per-turn (add reflections field to JSON response) — simpler, eliminates latency and threshold orchestration, but divides LLM attention across action selection and reflection in a single call. Worth revisiting if standalone proves too latent in short scenarios.
- Hybrid (standalone + per-turn observations) — unnecessary complexity; standalone reflection already reads all accumulated experience

**Rationale:** Reflection is a distinct cognitive process — "what have I learned?" is different from "what do I do now?" The engine uses the same pattern: experience accumulates, threshold triggers reflection, reflection produces insights. Insights later feed goal formation (Phase 2 of implementation).

**Trade-offs:** Additional LLM call per reflection cycle (infrequent — triggered by threshold, not per-turn). Reflection runs asynchronously in a virtual thread, so insights may lag by a few ticks. In short scenarios (~12 actions per character), the first insights arrive around tick 7-10 with `maxUnreflectedOutcomes=5`. Longer scenarios benefit substantially more.

**Orchestration:** After each `ingest()`, count unreflected experiences for the agent. If count >= `maxUnreflectedOutcomes` OR cumulative importance >= `importanceThreshold`, start reflection in a virtual thread. Reflection does not block the tick loop.

**Revision (from decision review R1-06, R1-07, R1-08, R1-09):** Dropped `ReflectionTriggerConfig` dependency (coupled to Worker outcomes). Added explicit orchestration design. Lowered default threshold. Acknowledged per-turn alternative more honestly.

**Depends on:** D1 (three-layer architecture — reflection is a neocortex service called by manor orchestration)
**Exploration:** quick
**Status:** revised

## D3: Per-turn goal management preserved in memory stack phase

**Choice:** Keep `newGoals`/`dropGoals` in the LLM response format for this phase. Replace with reflection-driven goal formation in the goal lifecycle phase.

**Alternatives:**
- Remove now (goals become static Eidos-only until reflection-driven formation is wired) — creates a regression in character behavior
- Per-turn goals as GoalFormationStrategy pass-through from day one — establishes the SPI contract immediately but adds implementation cost in a phase focused on memory, not goals

**Rationale:** The memory stack phase focuses on memory quality (salience, reflection, relationships). Goal lifecycle is a separate concern with its own design decisions (including the CaseDefinition question from D1). Removing per-turn goal management before the replacement is ready would make characters less autonomous.

**Trade-offs:**
- Behavioral contract entrenchment: characters and briefings are tuned for `newGoals`/`dropGoals`. Each phase that preserves them deepens the contract, making the eventual transition to reflection-driven formation a behavioral change that requires re-tuning.
- Dual-path transition gap: during the transition to reflection-driven goals, the system will briefly have two goal sources. The interaction protocol (override, merge, priority) must be designed in the goal lifecycle phase.
- Format gap: current `newGoals` is `{name, description}`. `AgentGoal` has richer structure (priority, visibility). The transition must handle this mismatch.

**Revision (from decision review R1-11, R1-12):** Added real trade-offs (was incorrectly "None"). Acknowledged per-turn-as-SPI alternative for Phase 2 consideration.

**Exploration:** quick
**Status:** revised

## D4: Defer engine-api dependency to goal lifecycle phase

**Choice:** Do not add `casehub-engine-api` as a dependency in the memory stack phase. The neocortex provides everything needed (salience, reflection, relationships, decay). Add engine-api in the goal lifecycle phase when SPIs are actually consumed.

**Alternatives:**
- Add engine-api now for `ReflectionTriggerConfig` — but R1-06 showed its importance weights are coupled to Worker outcome types
- Add engine-api now for forward-looking readiness — premature; introduces a dependency with no consumer

**Rationale:** The memory stack phase uses only neocortex APIs. Adding engine-api would bring in `CaseDefinition` and Worker-coupled types that aren't needed and create a misleading dependency. The goal lifecycle phase is where engine SPIs (`GoalFormationStrategy`, `GoalRevisionStrategy`) are actually consumed and where the CaseDefinition mapping question must be resolved.

**Trade-offs:** The goal lifecycle phase must add the dependency and resolve the CaseDefinition question. This is deferred work, not avoided work.

**Exploration:** quick (surfaced by decision review R1-04, R1-06, R1-15)
**Status:** captured
