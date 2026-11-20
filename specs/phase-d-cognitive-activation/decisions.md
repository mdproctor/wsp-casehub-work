# Decisions — Phase D Cognitive Activation

## D1: Phase D scope

**Choice:** #80 (CognitionCore.tick activation — all orchestrators) + #76 (directive-minimal YAML rewrite — extract and split briefings)
**Alternatives:**
- Tick only — defer seeding cleanup. Leaves duplication between briefing text and cognitive pipeline.
- Tick + seeding + norms — adds norm evolution. Largest scope, new consolidation phase. Deferred to Phase E.
**Rationale:** Tick activation gives characters a full inner life. Seeding cleanup removes duplication that would create contradictions once adaptation is running (briefing says "always scheme" while adapted drive says 0.1). Natural pairing.
**Trade-offs:** Norm evolution deferred. Characters still have static norms from YAML.
**Sources:** Audit report §9g, §9f; epic #77 checklist
**Exploration:** quick
**Status:** captured

## D2: Activation approach

**Choice:** Full stack wiring with progressive eval runs via CognitionConfig flags for empirical evidence. Wire all orchestrators in one pass. Evaluate incrementally by enabling one config flag at a time across test runs.
**Alternatives:**
- Incremental activation (separate issues per orchestrator) — buys safety not needed in pre-release demo
- Full stack without progressive eval — misses the empirical evidence for each subsystem's contribution
**Rationale:** Pre-release demo, single consumer. Full stack wiring is one implementation task. Progressive evaluation is a test/observation task using existing config flags. Separation lets us build fast and evaluate carefully.
**Trade-offs:** All-at-once wiring is harder to debug if something breaks. Mitigated by progressive eval runs isolating each subsystem.
**Sources:** CognitionConfig per-subsystem flags, ScenarioOrchestrator constructor
**Exploration:** quick
**Status:** captured

## D3: Eval output format

**Choice:** Delta-only output. Emit changes per tick per character — not snapshots. A character whose cognitive state didn't change produces no output. World events emitted only when they occur.
**Alternatives:**
- Full snapshot per tick — simple but produces huge JSON documents too large for LLM analysis
- Summary-only — loses the detail needed for cause-and-effect reasoning
**Rationale:** Output must be proportional to what happened, not to ticks × characters × sections. An LLM or human reviewer can read the delta stream and correlate cause-and-effect without drowning in repeated state.
**Trade-offs:** Delta computation adds complexity. Must track previous state per character to detect changes.
**Sources:** User requirement: "careful we don't get huge json documents"
**Exploration:** quick
**Status:** captured

## D4: Tick placement

**Choice:** Once per game tick, all characters, at the start of the cycle before any character acts. Batch-consistent.
**Alternatives:**
- Per-character before LLM call — inconsistent within a batch (earlier characters see stale state)
- Timer-driven — adds concurrency without clear benefit in a demo
**Rationale:** Characters are batched per game tick. Ticking all at cycle start keeps cognitive state consistent within a batch.
**Trade-offs:** Characters all update simultaneously rather than reacting to each other's actions within a tick. Acceptable — inter-tick reactions happen on the next cycle.
**Sources:** ScenarioOrchestrator.runAutonomousTicks
**Exploration:** quick
**Status:** captured

## D5: Store implementations

**Choice:** In-memory implementations for all orchestrator stores (NarrativeStore, UserProfileStore, MentalModelStore, StrategyStore).
**Alternatives:**
- MindMap-backed — persists across consolidation cycles, consistent with existing cognitive data model. More work.
- SQLite-backed — durable but adds schema management
**Rationale:** Pre-release demo, state lives for the scenario run duration. No persistence needed for evaluation. Simplest path.
**Trade-offs:** State lost between scenario runs. Acceptable for eval; revisit if persistence becomes needed.
**Sources:** Orchestrator constructor signatures
**Exploration:** quick
**Status:** captured

## D6: LLM-dependent orchestrators

**Choice:** Activate all, including StrategyLearningOrchestrator, UserModelOrchestrator, and MentalModelOrchestrator (which require LLM calls), and DriveOrchestrator (which depends on them).
**Alternatives:**
- Defer LLM-dependent — faster ticks but characters miss SDT drives and learned strategies
- Stub LLM calls — structure without substance. Defers the real test.
**Rationale:** The point is empirical evidence for each subsystem. Stubbing defeats that purpose. LLM latency per tick is acceptable in eval runs.
**Trade-offs:** Eval runs are slower (LLM calls per tick per character). Acceptable for controlled scenario with 2-3 characters.
**Sources:** CognitionCore.tick() phase ordering, DriveOrchestrator constructor
**Exploration:** quick
**Status:** captured

## D7: Seeding — directive-minimal rewrite

**Choice:** Extract and split existing briefings in descriptors-composite.yaml. Voice/role/hard-constraints stay in directive (system prompt). Behavioural instructions move to SocialConfig (observation pipeline). No remnants, no duplication.
**Alternatives:**
- Rewrite from scratch — more control but significant manual effort for 17 characters
- Defer — keeps current briefings. Risk of contradiction when adaptation runs (briefing says "always scheme", adapted drive says 0.1)
**Rationale:** Clean separation is the best-practice reference. After Phase D, the 17 characters demonstrate "how to seed an agent" without a briefing. No duplication means no contradictions when drives adapt.
**Trade-offs:** Editorial work for 17 characters. Mitigated by systematic extraction rather than per-character authoring.
**Sources:** D2 (original decision), D3 (cognitive preamble), descriptors-composite.yaml
**Exploration:** quick
**Status:** captured

## D8: Eval scenario and test infrastructure

**Choice:** Dedicated eval scenario as a permanent test suite. JUnit test classes under src/test/java/, run with a Maven profile (-Pcognitive-eval). Not ad-hoc scripts. Two layers: structural assertions (deterministic, CI-ready) and experiment output (persisted evidence for human review).
**Alternatives:**
- Full 17-character scenario — realistic but too noisy to attribute behaviour changes to specific orchestrators
- Ad-hoc scripts — not reproducible, not independently observable
**Rationale:** The test suite IS the experiment. Keeping it as a proper test suite makes the experiment reproducible, independently observable, and CI-ready. Anyone clones the repo, runs mvn test -Pcognitive-eval, and sees empirical evidence.
**Trade-offs:** Dedicated scenario may not capture emergent multi-character dynamics. Run full scenario separately after all subsystems validated.
**Sources:** Existing -Pllm-eval profile pattern
**Exploration:** quick
**Status:** captured

## D8a: Assertion layers

**Choice:** Two layers. Structural assertions (deterministic): "enabling mood produces a mood section", "delta output is non-empty when a new subsystem activates", "DriveOrchestrator only fires when dependencies present." Experiment output (non-deterministic): full delta stream persisted for human review. Structural tests are real pass/fail CI tests. Experiment output is evidence.
**Alternatives:**
- Assert on LLM content — too non-deterministic for reliable CI
- No assertions, only output — loses CI confidence that wiring works
**Rationale:** Structural assertions verify the plumbing. Experiment output verifies the value. Separating them means CI stays green while humans evaluate whether the subsystems produce meaningful behaviour change.
**Trade-offs:** Can't automatically verify "does mood make responses better" — that stays a human judgment.
**Sources:** LLM non-determinism constraints
**Exploration:** quick
**Status:** captured

## D9: SubjectResolver for tick()

**Choice:** Inline lambda capturing WorldState — returns agentIds of characters in the same room as the ticking agent. No new class; defined at the tick call site in runAutonomousTicks().
**Alternatives:**
- Named class ManorSubjectResolver — unnecessary indirection for a one-liner
- Static subjects (all active agents) — wrong semantics; per-subject orchestrators should only model characters the agent can actually perceive
**Rationale:** CognitionCore.tick() takes 4 args (agentId, tenantId, descriptor, SubjectResolver). The spec originally showed 3. The manor's "relevant subjects" are characters in the same room — the same set already computed for nearbyIds in the observation builder. Lambda captures `world` which is already in scope.
**Trade-offs:** Resolver is recreated each tick. Negligible cost — it's a lambda, not an object with state.
**Sources:** CognitionCore.tick() line 130, SubjectResolver.java, ScenarioOrchestrator line 259 (nearbyIds)
**Exploration:** quick
**Status:** captured

## D10: In-memory store implementations

**Choice:** Create local in-memory store implementations in wacky-manor production code (package-private in manor/agent/). Four trivial classes: InMemoryNarrativeStore, InMemoryUserProfileStore, InMemoryMentalModelStore, InMemoryStrategyStore.
**Alternatives:**
- Reuse CognitionStack's inner classes — they're in blocks test code, can't be referenced from production code in another repo
- Move stores to blocks production code — cross-repo change for a demo-only need
- Make CognitionStack stores public — production depending on test code
**Rationale:** Each store is 5–15 lines (ConcurrentHashMap + key-based lookup). Local copies avoid cross-repo dependencies and test-production coupling. Pre-release demo — no persistence needed (D5).
**Trade-offs:** Minor duplication with CognitionStack's test copies. Acceptable — they're trivial and independently evolvable.
**Sources:** CognitionStack.java lines 528–615 (reference implementations)
**Exploration:** quick
**Status:** captured

## D11: DriveOrchestrator construction — explicit DriveSource pattern

**Choice:** Construct the four DriveSource implementations explicitly (CuriosityDrive, CompetenceDrive, AffiliationDrive, AutonomyDrive), each wrapping its backing orchestrator, then pass to DriveOrchestrator's 7-arg constructor. Follow CognitionStack's proven pattern.
**Alternatives:**
- CDI-style constructor (takes orchestrators directly, creates drives internally) — designed for injection, not manual construction; loses control over per-axis thresholds and cooldowns
**Rationale:** Explicit construction gives control over AffiliationDrive threshold (0.3), cooldown (1h), and AutonomyDrive threshold (0.5). These parameters matter for tuning character behaviour during eval runs. CognitionStack uses this pattern successfully.
**Trade-offs:** More verbose construction. Worth it for tunability.
**Sources:** CognitionStack.from() lines 150–157, DriveOrchestrator constructors lines 38–92
**Exploration:** quick
**Depends on:** D5 (in-memory stores), D10 (store visibility)
**Status:** captured
