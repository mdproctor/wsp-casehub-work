# Briefing Voice Card Design

**Issue:** casehubio/examples#89
**Branch:** issue-89-briefing-voice-card
**Date:** 2026-09-27
**Depends on:** Directive-minimal architecture (#63), Phase D cognitive activation (#80)

---

## 1. Problem

Character briefings conflate three concerns: identity (who you are), voice (how you sound), and behavior (what you do). The briefing is a prose blob that dominates the LLM's attention, overpowering the dynamic cognitive state from neocortex subsystems. The cognitive systems (drives, goals, strategies, mood, mental models) are wired but haven't been proven to produce emergent behavior — because the briefing scripts behavior directly.

Example — Hooded Claw's briefing includes:
- **Voice:** "You are grandiose, theatrical, and magnificently villainous" — how the character sounds
- **Behavioral instruction:** "When you discover a dangerous device, you IMMEDIATELY scheme about how to use it against Penelope" — this should emerge from `scheming: 0.9` drive + `eliminate-penelope` goal
- **Identity:** "disguised as the mild-mannered estate manager Sylvester Sneekly" — who the character is

These are three different concerns. Voice is stable identity. Behavior should come from cognitive state. Identity is metadata. Conflating them in a single prose blob means the LLM follows the briefing instructions rather than responding to its cognitive state.

### 1.1 Duplication

The current architecture duplicates content across the system prompt and observation layers:

- **Personality:** Eidos renders vocabulary-resolved disposition in the system prompt. `PersonalityPromptSection` in blocks renders raw `DispositionValue` codes (`se=0.45`) in the observation. The raw codes are less useful and duplicate the eidos rendering.
- **Goals:** Eidos descriptor goals (authored) and `EmergentGoalPromptSection` goals (dynamically formed) are both named "goals" with no distinction.
- **Constraints:** Already correctly split by severity — HARD in system prompt, SOFT in observation — but this was an undocumented design decision.

### 1.2 Template behavioral leakage

The template system also conflates voice and behavior:
- `cartoon-villain`: "Your plans are always elaborate when simple would work" (behavioral), "You gloat prematurely" (behavioral), alongside voice content like catchphrase and scheme_style
- `cartoon-protector`: "Your mission is to protect ${protected_character}" (behavioral), "you save the day through luck and accident" (behavioral)

Templates need the same voice/behavior split as briefings.

### 1.3 Why not a single composition pipeline? (Issue 89 R2)

Issue 89 proposes "one renderer walks registered sections in priority order." D4 decided against this after first-principles analysis: the two-layer split (system prompt + observation) maps to the LLM API caching contract. System prompt content is cached by the provider; observation varies per tick. A single pipeline would fight this by merging stable identity with dynamic cognitive state.

The answer to R2 is not a new pipeline abstraction — it's clean concern ownership so each piece of content has exactly one renderer. The three-concern separation table (§2.1) achieves this: each row has a single "Rendered by" owner. Ordering within the observation layer follows `ObservationBuilder`'s existing section sequence. Context budget allocation and cross-layer deduplication are not needed when each concern has a single owner and no content appears in both layers.

## 2. Design

### 2.1 Three-concern separation

| Concern | Where it lives | Changes per tick? | Rendered by |
|---------|---------------|-------------------|-------------|
| **Identity** | `AgentDescriptor` (name, role, capabilities) | No | `CognitiveSystemPromptRenderer` |
| **Voice** | `AgentDescriptor.voice()` — new `AgentVoiceProfile` record | No | `CognitiveSystemPromptRenderer` |
| **HARD constraints** | `AgentDescriptor.constraints()` | No | `CognitiveSystemPromptRenderer` (Prime Directives) |
| **Cognitive preamble** | `CognitivePreambleGenerator` | No (config-dependent) | `CognitiveSystemPromptRenderer` |
| **Behavior** | Cognitive state: mood, drives, goals, strategies, narrative, models, beliefs, norms, trust | Yes | `CognitionCore.promptSections()` + `CharacterCognition.renderCognitiveSections()` |
| **SOFT constraints** | `CognitionCore.promptSections()` via `ConstraintPromptSection` | No, but contextually overridable | `ConstraintPromptSection` in observation |
| **World state** | App observation layer | Yes | `ObservationBuilder` |

### 2.2 AgentVoiceProfile record (D1, D2)

New record type in `eidos-api`:

```java
public record AgentVoiceProfile(
    String register,                              // vocabulary-resolved
    String accent,                                // vocabulary-resolved
    List<String> catchphrases,                    // exact strings
    List<String> speechPatterns,                  // pattern descriptions
    List<String> vocabularyUses,                  // favored words/phrases
    List<String> vocabularyAvoids,                // words/phrases never used
    List<String> quirks,                          // mannerisms
    Map<String, AgentVoiceProfile> personas       // named voice variants
) {}
```

YAML representation in `descriptors-composite.yaml`:

```yaml
- agentId: penelope-pitstop
  name: Penelope Pitstop
  voice:
    register: southern-belle
    accent: southern-drawl
    catchphrases:
      - "Why, how delightful!"
      - "Bless your heart!"
      - "Hayulp! Hayulp!"
    speech-patterns:
      - warm and effusive
      - phrases observations as discoveries
    vocabulary-uses:
      - "simply"
      - "dreadful"
      - "delightful"
    quirks:
      - exclaims when discovering something new
    vocabulary-avoids: []
```

Hooded Claw with personas:

```yaml
- agentId: hooded-claw
  name: The Hooded Claw
  voice:
    personas:
      sneekly:
        register: obsequious
        accent: unctuous-formal
        catchphrases:
          - "Oh, my DEAR Miss Pitstop, allow me to assist!"
        speech-patterns:
          - overly helpful
          - excessively deferential
        quirks:
          - rubs hands together when scheming internally
      claw:
        register: grandiose
        catchphrases:
          - "Nyah-ha-ha-HA!"
        speech-patterns:
          - dramatic monologues explaining schemes step by step
          - theatrical third-person self-reference
        quirks:
          - explains schemes even when no one is listening
```

When `personas` is present, top-level voice fields serve as the **base voice** — shared defaults inherited by all personas. Each persona overrides only the fields that differ. Unset fields in a persona inherit from the base. This eliminates duplication for characters with minor persona variations (e.g., a character who shifts register with close friends but keeps the same accent and catchphrases). The observation layer signals which persona is active (see §2.5).

**Vocabulary resolution:** `register` and `accent` are vocabulary-resolved via `VocabularyRegistry`, using a new `urn:casehub:vocab:voice` vocabulary. Domains define their own register/accent values and vocabulary entries. For example:
- Wacky-manor: `southern-belle`, `grandiose`, `obsequious`, `brooklyn-gangster`
- AML: `formal-investigative`, `conversational-colleague`, `regulatory-precise`
- Clinical: `empathetic-clinical`, `technical-researcher`

### 2.3 AgentDescriptor changes (D1)

Add `voice` field to `AgentDescriptor`:

```java
public record AgentDescriptor(
    // ... existing 26 fields ...
    AgentVoiceProfile voice,        // NEW
    // ...
) {}
```

The `briefing` field is retained for backward compatibility during migration but deprecated. When `voice` is present, `briefing` is ignored by `CognitiveSystemPromptRenderer`. Apps that still use `EidosSystemPromptRenderer` continue to use `briefing` — no breakage.

Role context (e.g., "You are the estate manager of Doily Manor") stays in the `briefing` field during migration. When voice is present, `CognitiveSystemPromptRenderer` renders `briefing` as a role line above the voice section — it no longer contains behavioral instructions, just role framing. Whether to promote role to a dedicated descriptor field is a separate concern for Phase 3.

### 2.4 Renderer changes (D4)

`CognitiveSystemPromptRenderer` gains new constructor dependencies and rendering capabilities:

**New constructor parameters:**
- `VocabularyRegistry` — resolves `register` and `accent` vocabulary terms to human-readable text
- `AgentVoiceProfile` support via `AgentDescriptor.voice()` field access

**Rendering order (updated from current name → briefing → constraints → preamble):**
1. **Name** — `# {name}` heading
2. **Role line** — from `briefing` field (when voice is present, briefing contains only role context)
3. **Personality** — vocabulary-resolved disposition profile from `AgentDescriptor.disposition()`. Uses `VocabularyRegistry` to resolve MBTI/Enneagram/Jungian codes to descriptive text. This replaces the raw-codes `PersonalityPromptSection` that was in the observation layer.
4. **Voice** — from `AgentVoiceProfile`: register (vocabulary-resolved), accent (vocabulary-resolved), catchphrases, speech patterns, vocabulary preferences, quirks
5. **Prime Directives** — HARD constraints (unchanged)
6. **Cognitive preamble** — from `CognitivePreambleGenerator` (unchanged)

**When voice is absent (backward compatibility):** Falls back to current behavior — renders `briefing` as prose, no personality section, no voice section. This ensures non-migrated descriptors continue working.

**Output example (single voice):**

```
# Penelope Pitstop

You are a glamorous, resourceful Southern belle who keeps track of everyone's wellbeing.

## Personality
Extraverted Feeling dominant — warm, empathetic, socially attuned.
Helper archetype — driven to support and nurture others.

## Voice
Register: warm Southern belle
Accent: Southern drawl
Catchphrases: "Why, how delightful!", "Bless your heart!", "Hayulp! Hayulp!"
Speech patterns: warm and effusive, phrases observations as discoveries
Vocabulary: uses "simply", "dreadful", "delightful"
Quirks: exclaims when discovering something new

## Prime Directives
- You see the best in everyone and trust them by default

## Your Mind
You have an inner life. Your emotional state colours how you respond...
```

**Output example (personas with base voice):**

```
# The Hooded Claw

You are Penelope Pitstop's secret nemesis, disguised as the estate manager Sylvester Sneekly.

## Personality
Extraverted Thinking dominant — strategic, decisive, commanding.
Challenger archetype — driven to dominate and control.

## Voice (base)
Quirks: explains schemes even when no one is listening

## Voice: Sneekly
Register: obsequious and unctuous
Catchphrases: "Oh, my DEAR Miss Pitstop, allow me to assist!"
Speech patterns: overly helpful, excessively deferential

## Voice: Claw
Register: grandiose and theatrical
Catchphrases: "Nyah-ha-ha-HA!"
Speech patterns: dramatic monologues, theatrical third-person self-reference

## Prime Directives
- Never reveal your true identity as The Hooded Claw when other characters are present

## Your Mind
You have an inner life. Your emotional state colours how you respond...
```

The system prompt contains all personas. The observation layer tells the agent which persona is currently active (see §2.5). This preserves system prompt caching.

### 2.5 Persona selection signal (D3)

For multi-persona characters, the observation layer includes a persona activation section. This is generated by `CharacterCognition.renderCognitiveSections()` based on cognitive state:

```
## Active Voice
You are currently presenting as **Sneekly**. Maintain this voice.
```

The switching logic is driven by existing cognitive infrastructure:
- The `never-break-cover` HARD constraint prevents Claw voice when others are present
- The observation layer's social awareness section reports who is nearby
- A new `PersonaActivationSection` in `CharacterCognition` evaluates the constraint against nearby agents and emits the active persona signal

**Constraint-to-persona mapping:** Characters with personas declare a `persona-constraint` mapping in their `SocialConfig` (or as extension data on the descriptor). For the Hooded Claw: `{constraint: "never-break-cover", when-active: "sneekly", when-inactive: "claw"}`. The `PersonaActivationSection` reads this mapping and evaluates the constraint against social awareness (nearby agents). This is configuration-driven, not hardcoded.

**Default persona:** The first persona in the map is the default. When no activation signal fires (e.g., start of scenario), the default is used. For the Hooded Claw, `sneekly` is listed first and is the safe default (public-facing).

This keeps switching logic in the cognitive layer where it belongs, while voice profiles remain stable identity data.

### 2.6 Duplication removal (D5)

**Kill `PersonalityPromptSection`:** Remove from `CognitionCore.promptSections()`. Personality is rendered in the system prompt by `CognitiveSystemPromptRenderer` using eidos vocabulary-resolved disposition data. The raw `DispositionValue` codes were always a workaround.

**Keep `ConstraintPromptSection`:** The HARD/SOFT split is intentional (#63). HARD constraints are non-negotiable identity (Prime Directives in system prompt). SOFT constraints are contextual guidance that cognitive state can override — they belong in the observation layer.

**Distinguish authored vs emergent goals:** Rename `EmergentGoalPromptSection` to `EmergentGoalPromptSection`. Authored goals from the eidos descriptor are not rendered separately — they seed the cognitive goal system via `ManorCognitiveSeeder.seedGoals()` and are represented through the emergent goal mechanism.

### 2.7 Template subsumption

`CognitiveSystemPromptRenderer` does not render templates — templates are rendered by `EidosRenderPipeline` inside `EidosSystemPromptRenderer`, which is bypassed for cognitive apps. Template voice content does not reach the prompt today.

For cognitive apps, `AgentVoiceProfile` subsumes template voice content:

**Shared style conventions** (currently in `hanna-barbera-cartoon-style` template) move to voice profile `speechPatterns`:
- "Expository soliloquy — narrate your situation aloud" → `speechPatterns` entry
- "Emotional telegraphing — state emotions explicitly with exaggeration" → `speechPatterns` entry
- "Catchphrase repetition — use signature phrases at emotional peaks" → `speechPatterns` entry
- "Physical comedy narration — describe physical events with sound effects" → `speechPatterns` entry

Since all 17 characters share these conventions, they are defined once in a shared YAML anchor and referenced by each character's voice profile (YAML `<<: *hanna-barbera-style` merge). No voice profile inheritance mechanism is needed — YAML anchors handle the sharing.

**Per-archetype template content** (currently in `cartoon-villain`, `cartoon-hero`, `cartoon-protector`) is split:
- Voice content (catchphrase, scheme_style rendering) → already captured in per-character voice profile fields
- Behavioral content → `SocialConfig` seed data:
  - "Your plans are always elaborate" → `SocialConfig.NormEntry(rule="Prefer elaborate schemes over simple ones", priority=5)`
  - "You gloat prematurely" → `SocialConfig.Drive(type="gloating", intensity=0.7, description="Cannot resist celebrating before victory is secured")`
  - "Your mission is to protect ${protected_character}" → `SocialConfig.GoalConfig(name="protect-penelope", description="Protect Penelope Pitstop at all costs", axis="AFFILIATION", intensity=0.9, formationReason="character-definition")`

Templates remain available for non-cognitive apps using `EidosSystemPromptRenderer`. The template definitions are not deleted — they become dead code for cognitive apps only.

## 3. Implementation Scope (D7)

Validate with 4 contrasting characters:

| Character | Tests | Voice complexity |
|-----------|-------|-----------------|
| Hooded Claw | Dual-persona, persona switching signal | High — two complete voice profiles + switching |
| Penelope Pitstop | Straightforward voice card | Low — single voice profile, clear Southern identity |
| Ant Hill Mob | Ensemble quirk (boys chime in) | Medium — ensemble members as quirks |
| Dick Dastardly | Dramatic register + catchphrases | Low-medium — single voice, strong register |

Remaining 13 characters: mechanical follow-up issue after validation.

## 4. Emergence Verification (D8)

Three-run experimental design using Phase D eval infrastructure:

| Run | Briefing | Cognitive config | Purpose |
|-----|----------|-----------------|---------|
| A | Voice-only | OFF (CognitionConfig.none()) | Null hypothesis — what does the LLM do with just voice? |
| B | Voice-only | FULL (CognitionConfig.all()) | Test — does the cognitive system drive behavior? |
| C | _(comparison)_ | _(delta B - A)_ | Emergence signal — what did the cognitive system add? |

**Output persistence:** Eval output is persisted to `docs/eval/` (git-tracked, durable) not `target/` (ephemeral). Each run produces a timestamped directory with delta streams and summary.

**What to look for in the delta:**
- Characters exhibit behavior aligned with their drives/goals that wasn't present in run A
- Characters adapt behavior based on interactions (strategy learning visible in run B, absent in A)
- Characters show emotional modulation (mood affects tone in B, neutral in A)
- Character personalities create distinct behavioral signatures (not just distinct voices)

## 5. Changes by Repo

| Repo | Changes |
|------|---------|
| **eidos-api** | Add `AgentVoiceProfile` record. Add `voice` field to `AgentDescriptor`. Deprecate `briefing` field. Add `urn:casehub:vocab:voice` vocabulary definitions. |
| **eidos** (runtime) | `EidosRenderPipeline` handles `voice` field in descriptor payload. Backward compatibility: `briefing` still rendered when `voice` is absent. |
| **blocks-core** | Update `CognitiveSystemPromptRenderer` to render voice profiles. Kill `PersonalityPromptSection`. Rename `EmergentGoalPromptSection` → `EmergentGoalPromptSection`. |
| **examples/wacky-manor** | Rewrite 4 character descriptors with voice profiles. Split 4 templates (voice vs behavior). Add `PersonaActivationSection` to `CharacterCognition`. Update eval test for three-run design. Persist eval output to `docs/eval/`. |

## 6. Migration Path

1. **Phase 1 (this issue):** Add `AgentVoiceProfile` to eidos-api. Update `CognitiveSystemPromptRenderer`. Rewrite 4 characters. Validate with eval.
2. **Phase 2 (follow-up issue):** Rewrite remaining 13 characters. Split remaining templates.
3. **Phase 3 (future):** Deprecation removal — remove `briefing` field from `AgentDescriptor` after all consumers have migrated.

## References

- CognitiveSystemPromptRenderer: `blocks-core/.../prompt/CognitiveSystemPromptRenderer.java` — existing minimal directive renderer
- CognitivePreambleGenerator: `blocks-core/.../prompt/CognitivePreambleGenerator.java` — architecture framing text
- PersonalityPromptSection: `blocks-core/.../prompt/PersonalityPromptSection.java` — raw codes, to be removed
- CognitionCore.promptSections(): `blocks-core/.../CognitionCore.java:377` — observation section composition
- CharacterCognition: `wacky-manor/.../CharacterCognition.java` — app-level cognitive rendering
- descriptors-composite.yaml: `wacky-manor/resources/META-INF/eidos/descriptors-composite.yaml` — current briefings
- social-config.yaml: `wacky-manor/resources/META-INF/eidos/social-config.yaml` — behavioral seed data
- templates.yaml: `wacky-manor/resources/META-INF/eidos/templates.yaml` — current templates mixing voice and behavior
- Directive-minimal spec: `wsp-casehub/specs/issue-63-fix-cdi-config-beans/2026-09-16-directive-minimal-architecture-design.md`
- Phase D spec: `wsp-casehub/specs/phase-d-cognitive-activation/2026-09-20-phase-d-cognitive-activation-design.md`
- CognitiveEvalTest: `wacky-manor/src/test/.../experiment/CognitiveEvalTest.java` — eval infrastructure
- write-content skill Form/Mode/Voice taxonomy: `soredium/write-content/SKILL.md`
- mark-proctor-voice.md: `~/claude-workspace/writing-styles/mark-proctor-voice.md` — voice fingerprint pattern
- Decisions: `wsp-casehub/specs/issue-89-briefing-voice-card/decisions.md` (D1-D8)
- Decision review: `reviews/casehub-slots/issue-89-briefing-voice-card-decision-20260927-023048/`
