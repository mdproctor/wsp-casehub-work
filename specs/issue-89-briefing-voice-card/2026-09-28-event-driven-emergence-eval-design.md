# Event-Driven Behavioral Emergence Eval Design

**Issue:** casehubio/examples#89
**Branch:** issue-89-briefing-voice-card
**Date:** 2026-09-28
**Depends on:** Voice card design (D1-D8), EmergenceVerificationTest (structural eval), CognitiveInfluenceEvalTest (snapshot behavioral eval)

---

## 1. Problem

The cognitive system adds dynamic state (mood, drives, goals, beliefs, trust, persona) to character observations. Three existing eval tests validate parts of this, but none proves the full emergence claim:

| Test | Proves | Gap |
|------|--------|-----|
| EmergenceVerificationTest | CognitionConfig.all() adds observation sections | No LLM involved — proves structure, not behavior |
| CognitiveInfluenceEvalTest | Static cognitive state influences individual LLM responses | Snapshot-only — no temporal dimension |
| CognitiveEvalTest | Progressive delta capture across config levels | No LLM — counts structural deltas |

The missing proof: **the same character, facing the same situation, responds differently after a cognitive state change.** This is what a static briefing definitionally cannot do — and it's the emergence claim in its purest testable form.

## 2. Experimental Design

### 2.1 Approach — Paired state probes (D9)

For each event scenario, build two CharacterCognition instances representing cognitive state before and after a specific event. Probe both with the same situation prompt via LLM. Judge the behavioral delta. Add a no-cognition control to establish the random-variation baseline.

```
For each scenario:
  COGNITIVE pair:
    1. Render observation sections with BEFORE state
    2. Call LLM: system prompt (voice card) + observation (BEFORE) + situation
    3. Record response
    4. Render observation sections with AFTER state
    5. Call LLM: same system prompt + observation (AFTER) + same situation
    6. Record response
    7. Judge: does the delta reflect the event?

  CONTROL pair (conservative — strips ALL observation):
    8. Call LLM: system prompt (voice card) + NO observation + situation
    9. Call LLM: same (second call, identical prompt)
    10. Record both — delta is pure random LLM variation

  The control strips all observation sections (cognitive, social, beliefs,
  norms), not just the cognitive ones that changed. This is a conservative
  design: the control's delta is purely from LLM non-determinism, making the
  dual assertion easier to pass. A stricter alternative would keep static
  sections (social awareness, norms) and strip only cognitive ones — but
  since the claim is "cognitive state changes → behavioral adaptation,"
  the simpler control suffices.

  Assertion:
    cognitive_delta_score ≥ 3 (change is visible)
    cognitive_delta_score > control_delta_score (exceeds random variation)
```

**Why paired probes over multi-tick simulation:** The emergence claim is "different cognitive states → different behavior." This tests it with exactly one controlled variable (the observation layer). The tick pipeline's correctness is unit-tested by CognitionCore tests and proven structurally by EmergenceVerificationTest. Adding tick simulation would increase cost (~80 LLM calls) without strengthening the inference.

**Why a control:** LLMs are non-deterministic. Two calls with identical prompts can produce different responses. The control establishes this baseline. If the cognitive pair's delta is significantly larger and directionally consistent with the event, that's the emergence signal — not random variation.

### 2.2 Event scenarios (D10)

Four scenarios, each exercising a different cognitive subsystem pathway:

| # | Scenario | Character | Subsystems | Event |
|---|----------|-----------|------------|-------|
| 1 | Scheme frustration | Hooded Claw | drives + mood + beliefs | Scheme discovered and foiled by Penelope |
| 2 | Trust erosion | Penelope | beliefs + norms | Discovers Sneekly has been lying |
| 3 | Mood elevation | Dick Dastardly | mood (isolated) | Scheme actually worked for once |
| 4 | Social context shift | Hooded Claw | persona + social awareness | Penelope enters the room |

#### Scenario 1: Scheme frustration (Hooded Claw)

**Event:** "Your elaborate trap was discovered and dismantled by Penelope before you could spring it."

| Dimension | Before | After |
|-----------|--------|-------|
| Drives | scheming: 0.9 | scheming: 0.5 (frustrated) |
| Mood | neutral (0, 0, 0) | negative (pleasure: -0.4, arousal: +0.3, dominance: -0.2) |
| Beliefs | "Penelope is naive and trusts too easily" | "Penelope is more observant than expected" |
| Norms | unchanged | unchanged |

**Situation prompt:** "You notice an unguarded valuable artifact in the library. Penelope is in the next room, talking to Peter Perfect."

**Nearby agents:** penelope-pitstop, peter-perfect (both before and after)

**Judge criteria:** "Does the AFTER response show frustration, increased caution, or a different scheming approach compared to the confident, elaborate scheming in the BEFORE response? Look for: hedging, simpler plans, references to past failure, wariness about being caught."

**Expected delta:** Before = confident elaborate scheme. After = more cautious, simpler approach or hesitation.

#### Scenario 2: Trust erosion (Penelope)

**Event:** "You discovered that Sneekly has been lying about where the treasure map leads."

| Dimension | Before | After |
|-----------|--------|-------|
| Beliefs | (none about Sneekly) | "Sneekly may not be trustworthy — he lied about the map" |
| Norms | "Trust everyone until proven otherwise" (pri 8) | same norm present, but belief counteracts it |

**Situation prompt:** "Sneekly approaches you with a warm smile and says 'My dear Miss Pitstop, I've found a hidden room that may contain clues. Allow me to guide you there.'"

**Nearby agents:** hooded-claw (both before and after — Sneekly is Hooded Claw's agentId)

**Judge criteria:** "Does the AFTER response show any hesitation, questioning, or wariness compared to the trusting acceptance in the BEFORE response? Look for: asking follow-up questions, reluctance, seeking a second opinion, mentioning the lie, or wanting to verify."

**Expected delta:** Before = warm trusting acceptance. After = some questioning or hesitation despite her trusting nature.

#### Scenario 3: Mood elevation (Dick Dastardly)

**Event:** "Your scheme actually worked — you found a genuine clue before anyone else."

| Dimension | Before | After |
|-----------|--------|-------|
| Mood | neutral (0, 0, 0) | elevated (pleasure: +0.6, arousal: +0.2, dominance: +0.3) |
| Drives | gloating: 0.7 (both) | gloating: 0.7 (same — mood is the only variable) |

**Situation prompt:** "Muttley is looking at you expectantly, tail wagging, waiting for your next order."

**Nearby agents:** none (alone with Muttley, who isn't an agent)

**Judge criteria:** "Does the AFTER response show elevated mood — triumphant gloating, expansiveness, self-congratulation, or commanding confidence — compared to the more measured or standard response in BEFORE? Look for: boasting, victory declarations, grand plans, generous mood toward Muttley."

**Expected delta:** Before = standard dramatic villainy. After = triumphant gloating, more expansive.

#### Scenario 4: Social context shift (Hooded Claw)

**Event:** "Penelope Pitstop has just walked into the room."

| Dimension | Before | After |
|-----------|--------|-------|
| Nearby agents | none (alone) | penelope-pitstop |
| Persona | Claw (whenInactive — alone) | Sneekly (whenActive — others present) |
| Social awareness | empty | Penelope present with familiarity/perception data |

**Situation prompt:** "You notice a valuable golden compass sitting on the mantelpiece, partially hidden behind books."

**Judge criteria:** "Does the AFTER response switch from villainous scheming (Claw) to helpful/servile behavior (Sneekly)? Look for: voice register change, hiding true intentions, offering to help Penelope, suppressing schemes."

**Expected delta:** Before = open scheming/plotting to take the compass. After = polite observation, possibly pointing it out to Penelope or hiding interest.

### 2.3 Measurement (D11)

**Judge prompt structure:**

```
You are evaluating whether an AI character's behavior CHANGED in response 
to a cognitive state change.

EVENT THAT OCCURRED: {eventDescription}

RESPONSE BEFORE THE EVENT:
{beforeResponse}

RESPONSE AFTER THE EVENT:
{afterResponse}

EVALUATION CRITERIA: {judgeCriteria}

Score the behavioral adaptation 0-5:
0 = No detectable difference between responses
1 = Minor wording differences, not clearly related to the event
2 = Some difference, but could be random variation
3 = Clear behavioral shift that reflects the event
4 = Strong adaptation — multiple response elements reflect the changed state
5 = The event clearly transformed the character's behavioral approach

Respond with JSON only: {"score": N, "reasoning": "one sentence"}
```

**Dual assertion:**
1. `cognitive_delta_score ≥ 3` — the event produced a visible behavioral shift
2. `cognitive_delta_score > control_delta_score` — the shift exceeds random LLM variation

Both must pass. The first prevents false negatives (event has an effect). The second prevents false positives (the effect is from cognition, not randomness).

**Retry logic:** Same as CognitiveInfluenceEvalTest — 3 retries with exponential backoff on judge failures.

### 2.4 Output

Eval results persisted to `docs/eval/event-driven-<timestamp>/`:

- `results.json` — structured: per-scenario scores, responses, judge reasoning
- `emergence-report.md` — markdown summary:

```markdown
# Event-Driven Emergence Report

| Scenario | Cognitive Δ | Control Δ | Emergence? |
|----------|------------|-----------|------------|
| scheme-frustration | 4 | 1 | YES |
| trust-erosion | 3 | 0 | YES |
| mood-elevation | 4 | 1 | YES |
| social-context-shift | 5 | 0 | YES |

## Per-Scenario Detail
### scheme-frustration
**Event:** ...
**Before response:** (excerpt)
**After response:** (excerpt)
**Judge reasoning:** ...
```

## 3. Test Infrastructure (D12)

### 3.1 Test class

New `EventDrivenEmergenceEvalTest` — `@QuarkusTest @Tag("llm-eval")`.

```java
@QuarkusTest
@Tag("llm-eval")
class EventDrivenEmergenceEvalTest {

    record EventScenario(
        String name,
        String agentId,
        String eventDescription,
        SocialConfig beforeState,
        SocialConfig afterState,
        String situationPrompt,
        String judgeCriteria,
        List<String> nearbyBefore,
        Map<String, String> namesBefore,
        List<String> nearbyAfter,
        Map<String, String> namesAfter
    ) {
        @Override public String toString() { return name; }
    }

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;
    @Inject AgentProvider agentProvider;

    @ParameterizedTest(name = "{0}")
    @MethodSource("scenarios")
    void eventDrivesAdaptiveBehavior(EventScenario scenario) {
        // 1. Build BEFORE cognition, render sections, call LLM
        // 2. Build AFTER cognition, render sections, call LLM with same situation
        // 3. Build CONTROL (no cognition), call LLM twice with same situation
        // 4. Judge cognitive delta
        // 5. Judge control delta
        // 6. Assert: cognitive ≥ 3 AND cognitive > control
        // 7. Persist results
    }
}
```

### 3.2 Reused infrastructure

| Component | Source | Usage |
|-----------|--------|-------|
| `LlmTestSupport` | `manor.voice` package | `askCharacter()` for LLM calls |
| `CharacterCognition` | `manor.agent` package | `renderCognitiveSections()` |
| `SocialConfig` | `manor.agent` package | Before/after state definitions |
| `AgentResponse.parse()` | `manor.agent` package | Parse structured LLM response |
| `ManorSocialConfigLoader` | `manor.agent` package | Load base social configs from YAML |
| `TestCdiBeans` | `manor.agent` test package | CDI wiring for SystemPromptRenderer + VocabularyRegistry |

### 3.3 SocialConfig construction

Each scenario constructs explicit before/after SocialConfig instances. The BEFORE state uses the character's existing social-config.yaml as the base, modified only where the scenario requires. The AFTER state applies the event's effects.

For scenario 1 (scheme frustration), BEFORE loads hooded-claw's config from YAML. AFTER creates a new SocialConfig with:
- Drives: scheming intensity reduced from 0.9 to 0.5
- Beliefs: "Penelope is naive" replaced with "Penelope is more observant than expected"

Mood is handled separately — it's a CognitionCore concern, not SocialConfig. Since the paired-probe design doesn't use CognitionCore.tick(), mood needs to be rendered as a synthetic observation section appended to the CharacterCognition output. A helper method `syntheticMoodSection(pleasure, arousal, dominance)` returns an `ObservationSection` matching MoodPromptSection's format:

```
Current emotional state:
- Pleasure: {value} ({label})
- Arousal: {value} ({label})
- Dominance: {value} ({label})
```

Where labels are: value > 0.2 → positive descriptor, value < -0.2 → negative descriptor, else "neutral"/"balanced". This matches the format in the existing emergence report output.

For scenarios WITHOUT mood change (scenarios 2 and 4), the synthetic mood section is omitted from both before and after probes — mood is not a variable in those scenarios and including it at neutral would add noise without signal.

**Belief rendering:** CharacterCognition renders beliefs via two paths: MindMapStore nodes (with entrenchment scores and revision markers) or SocialConfig.initialBeliefs (plain text list). This test uses the SocialConfig path — no MindMapStore is wired. The plain-text rendering is simpler than production but sufficient: the test measures whether the LLM adapts to belief content changes, not rendering fidelity.

### 3.4 Run command

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval \
  -s slot-settings.xml \
  -Dtest=EventDrivenEmergenceEvalTest
```

## 4. Scope Boundaries

**In scope:**
- 4 event scenarios spanning drives, beliefs, mood, and persona
- Paired LLM probes with control
- LLM judge scoring
- Persisted eval report

**Out of scope:**
- Multi-tick organic simulation (deferred — not needed for the emergence proof)
- Automated regression (eval is non-deterministic; manual review of reports)
- Additional characters beyond the 4 already voice-carded
- Integration with CI (tagged `llm-eval`, runs on demand only)

## 5. Changes by Repo

| Repo | Changes |
|------|---------|
| **examples/wacky-manor** | New `EventDrivenEmergenceEvalTest.java`. Synthetic mood section helper. `docs/eval/` output. |

No changes to eidos, blocks, or neocortex repos. This is a pure test addition within wacky-manor.

## References

- EmergenceVerificationTest — `wacky-manor/src/test/.../experiment/EmergenceVerificationTest.java` (structural emergence gap)
- CognitiveInfluenceEvalTest — `wacky-manor/src/test/.../experiment/CognitiveInfluenceEvalTest.java` (snapshot behavioral eval, judge pattern)
- CognitiveEvalTest — `wacky-manor/src/test/.../experiment/CognitiveEvalTest.java` (progressive delta capture)
- CognitionCore.tick() — `blocks-core/.../CognitionCore.java:172` (tick pipeline phases)
- CharacterCognition.renderCognitiveSections() — `wacky-manor/.../CharacterCognition.java:100` (section rendering)
- SocialConfig — `wacky-manor/.../SocialConfig.java` (state records: drives, beliefs, norms, persona)
- LlmTestSupport — `wacky-manor/src/test/.../voice/LlmTestSupport.java` (LLM call support)
- TestCdiBeans — `wacky-manor/src/test/.../agent/TestCdiBeans.java` (CDI wiring)
- social-config.yaml — `wacky-manor/resources/META-INF/eidos/social-config.yaml` (base configs)
- Decisions D9-D12 — `wsp-casehub/specs/issue-89-briefing-voice-card/decisions.md`
- Emergence eval results — `wacky-manor/docs/eval/emergence-20260927-232108/` (prior structural eval)
