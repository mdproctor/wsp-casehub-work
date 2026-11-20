# Phase D — Cognitive Activation

**Date:** 2026-09-20
**Scope:** #80 (CognitionCore.tick activation) + #76 (directive-minimal seeding)
**Depends on:** Phase C (infrastructure), Phase C.5 (wiring audit — complete)

---

## 1. Problem

CognitionCore has 9 orchestrators. Only GoalProposalOrchestrator is wired.
The other 8 are passed as null. `tick()` is never called. Characters have
cognitive infrastructure they can't use: no mood, no SDT drives, no personal
narrative, no learned strategies, no mental models, no user models, no
memory hygiene, no inner life.

Separately, character briefings in `descriptors-composite.yaml` still contain
behavioural instructions that duplicate (and will contradict) what the cognitive
pipeline now provides through observation sections.

## 2. End State

After Phase D:

- **All 9 orchestrators active.** `tick()` called at the start of each game
  tick cycle. Characters have mood, SDT drives, narrative arc, learned
  strategies, mental models of others, user behaviour models, and memory
  hygiene — alongside the existing character drives, needs pyramid, beliefs,
  goals, and social awareness.

- **Clean directive/observation separation.** System prompt contains name,
  role, voice, and hard constraints only. All behavioural data flows through
  observation sections. The cognitive preamble bridges the two. No duplication.

- **Empirical evidence for each subsystem.** A permanent test suite that runs
  the game scenario at progressive config levels and persists delta output.
  Structural assertions verify wiring. Experiment output shows each subsystem's
  behavioural contribution.

## 3. Tick Activation (#80)

### 3.1 Orchestrator Construction

All orchestrators constructed in `ScenarioOrchestrator` with in-memory store
implementations (D10). In-memory stores are local package-private classes in
`manor/agent/` — trivial ConcurrentHashMap wrappers following the pattern in
CognitionStack. Construction order respects dependencies:

```
Phase 1 (no dependencies):
  MoodOrchestrator(MoodConfig)
  NarrativeOrchestrator(InMemoryNarrativeStore)
  MemoryHygieneOrchestrator(cbrStore, confidenceScorer, temporalDecay, scopeDecay, ...)

Phase 2 (independent, need AgentProvider):
  UserModelOrchestrator(InMemoryUserProfileStore, agentProvider, UserModelConfig)
  MentalModelOrchestrator(InMemoryMentalModelStore, agentProvider, MentalModelConfig)
  StrategyLearningOrchestrator(InMemoryStrategyStore, cbrStore, reflectionOrch, agentProvider, config)

Phase 3 (depends on Phase 1 + 2 — explicit DriveSource pattern, D11):
  CuriosityDrive(memoryHygiene)
  CompetenceDrive(strategy)
  AffiliationDrive(userModel, 0.3, Duration.ofHours(1))
  AutonomyDrive(mentalModel, 0.5)
  DriveOrchestrator(curiosity, competence, affiliation, autonomy, mood, DriveComposer(), driveConfig)

Phase 4 (depends on Phase 3):
  InnerLifeOrchestrator(reflectionOrch, agentProvider, civilityConstraints, innerLifeConfig, driveOrchestrator)

Phase 5 (already wired):
  GoalProposalOrchestrator (existing — gains DriveOrchestrator dependency for tick-driven proposals)
```

### 3.2 Tick Call Site

In `ScenarioOrchestrator.runAutonomousTicks()`, at the start of each game
tick cycle, before any character acts:

```java
SubjectResolver resolver = (agentId, tenantId) ->
    world.charactersInRoom(world.character(agentId).currentRoom()).stream()
         .map(CharacterState::agentId)
         .filter(id -> !id.equals(agentId))
         .collect(Collectors.toSet());

for (var c : actingThisTick) {
    var desc = agentRegistry.findById(c.agentId(), ManorConstants.TENANCY_ID).orElse(null);
    cognitionCore.tick(c.agentId(), ManorConstants.TENANCY_ID, desc, resolver);
}
```

`SubjectResolver` (D9) is a `@FunctionalInterface` returning the set of
relevant subject IDs for per-subject orchestrators (UserModel, MentalModel).
In the manor, relevant subjects are characters in the same room — the same
set already used for `nearbyIds` in the observation builder. The lambda
captures `world` which is already in scope; no new class needed.

All characters tick at cycle start for batch consistency. A character's
cognitive state is fresh when their LLM call happens later in the cycle.

### 3.3 CognitionConfig Progression

The config flags control which orchestrators contribute to `tick()` and
`promptSections()`. Progressive activation order:

| Level | Config | What activates |
|-------|--------|----------------|
| 0 | Current (goals + characterDrives + needsPyramid) | Baseline — no tick effects |
| 1 | + mood | Emotional state colours responses |
| 2 | + narrative | Personal story arc, "who am I becoming" |
| 3 | + userModel + mentalModel | Models of other characters' behaviour and intentions |
| 4 | + strategy | Learned strategies from past interactions |
| 5 | + drives | SDT 4-axis psychological drives (needs mood, strategy, models) |
| 6 | + memoryHygiene + innerLife | Background maintenance, reflection |

Each level is a test case in the eval suite.

## 4. Directive-Minimal Seeding (#76)

### 4.1 Current State

`descriptors-composite.yaml` contains per-character briefings mixing:
- Voice and identity ("speaks with theatrical menace")
- Role ("secretly the Hooded Claw")
- Hard constraints ("never reveal your true identity")
- Behavioural instructions ("always scheme against Penelope")

### 4.2 Target State

**Directive (system prompt):** Name + role + voice + hard constraints only.
Rendered by `CognitiveSystemPromptRenderer`. Cached — no state-dependent content.

**SocialConfig (observation pipeline):** Drives, beliefs, norms, goals,
reinforcement mappings. Rendered by CharacterCognition + CognitionCore.

**Cognitive preamble:** Architecture-level instruction generated by
`CognitivePreambleGenerator`. Tells the agent what subsystems it has
and how to use the observation data. Already implemented (#83).

### 4.3 Extraction Process

For each of the 17 characters:

1. Read the existing briefing text
2. Classify each sentence:
   - **Voice/identity** → stays in descriptor YAML `voice:` or `role:` field
   - **Hard constraint** → stays as `AgentConstraint` with `HARD` severity
   - **Behavioural instruction** → check if already represented in SocialConfig
     (drive, norm, belief, or goal). If yes, remove. If not, add to SocialConfig.
3. Write minimal directive — verify it produces the same character identity
   without behavioural duplication
4. Remove the old briefing text entirely — no commented-out remnants

### 4.4 Verification

After extraction, run the existing test suite. Each character should:
- Have the same identity (name, role, voice) in the system prompt
- Have all behavioural data in observation sections
- Produce no contradictions between directive and observation content

## 5. Cognitive Eval Test Suite

### 5.1 Infrastructure

- Maven profile: `-Pcognitive-eval`
- Test classes: `src/test/java/.../experiment/CognitiveEvalTest.java`
- Output: `target/cognitive-eval/<run-id>/`

### 5.2 Eval Scenario

Dedicated scenario: 2-3 characters, controlled interactions designed to
trigger each subsystem. Fixed world events at specific ticks:

| Tick | Event | Subsystems exercised |
|------|-------|---------------------|
| 1-3 | Characters meet, initial dialogue | mood, mentalModel |
| 4-5 | Conflict (item theft or betrayal) | mood shift, strategy, drives |
| 6-7 | Cooperation opportunity | userModel, strategy adaptation |
| 8-10 | Repeated interaction with same character | narrative arc, relationship evolution |

Characters selected for contrasting cognitive profiles (e.g., Hooded Claw
vs Penelope Pitstop — high scheming vs high social-harmony).

### 5.3 Two Assertion Layers

**Structural assertions (deterministic — CI tests):**

```java
@Test void moodSectionRenderedWhenEnabled()
@Test void narrativeSectionAbsentWhenDisabled()
@Test void deltaOutputNonEmptyWhenSubsystemActivated()
@Test void driveOrchestratorRequiresDependencies()
@Test void configLevelNProducesSupersetOfLevelNMinus1()
```

These verify the wiring works. They pass or fail. They run in CI.

**Experiment output (non-deterministic — persisted evidence):**

Each config level run produces:
- Delta stream: per-tick, per-character diffs of cognitive sections
- Only when something changed — characters with no cognitive state change
  produce no output for that tick
- World events only when they occur
- Metadata: tick number, config level, which sections changed, world context

Output persisted to `target/cognitive-eval/level-N-<label>/deltas.json`.
Human-reviewable. Comparable across levels.

### 5.4 Cross-Level Diff Report

The test suite generates a diff report between consecutive config levels
automatically. Not two similar files you have to compare — a single report
showing exactly what changed when a capability was enabled.

For each level transition (e.g., level 1 → level 2, "adding narrative"):

```
## Level 2: +narrative (on top of mood)

New sections appeared:
  hooded-claw tick 4: [Narrative] "A pattern is forming — each scheme
    fails, and Penelope seems more aware of the danger."
  penelope tick 6: [Narrative] "The manor reveals more of itself with
    each room. The Hooded Claw's presence feels closer."

Changed sections:
  (none — mood sections unchanged)

Behavioural shift:
  hooded-claw tick 7: response referenced personal arc (absent in level 1)
```

The report is generated per transition and written to
`target/cognitive-eval/diffs/level-1-to-2-narrative.md`.

A developer runs:
```bash
mvn test -Pcognitive-eval
```

And gets:
- Pass/fail on structural assertions
- `target/cognitive-eval/levels/` — raw delta stream per config level
- `target/cognitive-eval/diffs/` — cross-level diff reports showing
  exactly what each subsystem contributed
- `target/cognitive-eval/summary.md` — one-page overview: which subsystems
  produced observable changes, which didn't

## 6. Implementation Order

1. **Orchestrator construction** — wire all 9 into ScenarioOrchestrator
2. **Tick call site** — add tick() to the game loop
3. **Eval infrastructure** — Maven profile, scenario, delta capture
4. **Progressive eval runs** — run at each config level, verify structural assertions
5. **Seeding extraction** — extract and split all 17 character briefings
6. **Post-seeding eval** — re-run evals to verify no behavioural regression
7. **Full scenario run** — 17 characters with all subsystems active

## References

- Audit report: `wsp-casehub/audits/2026-09-19-cognitive-architecture-audit.md` §9g
- CognitionCore: `blocks-core/.../CognitionCore.java` — tick() method, constructor
- ScenarioOrchestrator: `wacky-manor/.../ScenarioOrchestrator.java` — current constructor call line 137
- Decisions: `wsp-casehub/specs/phase-d-cognitive-activation/decisions.md` (D1-D11)
- Phase C decisions: `wsp-casehub/specs/issue-63-fix-cdi-config-beans/decisions.md` (D2, D3)
- SubjectResolver: `blocks-core/.../SubjectResolver.java` — functional interface for per-subject tick resolution
- CognitionStack: `blocks-core/src/test/.../CognitionStack.java` — reference wiring pattern, in-memory store implementations
- DriveSource implementations: `blocks-core/.../drive/CuriosityDrive.java`, `CompetenceDrive.java`, `AffiliationDrive.java`, `AutonomyDrive.java`
