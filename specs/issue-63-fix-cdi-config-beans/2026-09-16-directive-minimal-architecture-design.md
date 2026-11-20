# Directive-Minimal Architecture Design

**Issue:** casehubio/blocks#283
**Branch:** issue-63-fix-cdi-config-beans
**Date:** 2026-09-16

## Problem

The current system prompt places cognitive data (goals, beliefs, strategies, constraints, disposition) alongside identity in the agent's briefing directive. The LLM treats system prompt content as authoritative instruction, overpowering dynamic cognitive sections from neurocortex subsystems that appear in the user prompt. Mood shifts, drive intensities, learned strategies, and trust perceptions read as supplementary context rather than active cognitive state.

The duplication compounds the problem: goals appear in both the system prompt (from eidos descriptor) and observation sections (from `CognitiveObservationSections.goalsSection()` and `EmergentGoalPromptSection`). Constraints appear in both channels. The directive version dominates, making the observation-side rendering decorative rather than functional.

## Architecture

### Before (current)

```
System prompt (directive):
  ├── Briefing text (identity + behavioral instructions + character knowledge)
  ├── Templates (voice/style + behavioral patterns)
  ├── Goals (static, from descriptor YAML)
  ├── Constraints (all severities)
  └── Disposition (MBTI, Enneagram)

User prompt (observations):
  ├── World state (room, characters, objects)
  ├── Cognitive sections (CognitionCore.promptSections())
  │     ├── Mood (MoodOrchestrator)
  │     ├── Drives (DriveOrchestrator)
  │     ├── Narrative (NarrativeOrchestrator)
  │     ├── Mental model / beliefs (MentalModelOrchestrator)
  │     ├── User model (UserModelOrchestrator)
  │     ├── Strategy (StrategyLearningOrchestrator)
  │     ├── Goals (GoalProposalOrchestrator)  ← DUPLICATE
  │     ├── Personality (disposition)         ← DUPLICATE
  │     └── Constraints                       ← DUPLICATE
  ├── CharacterCognition sections
  │     ├── Drives (from social-config.yaml)
  │     ├── Beliefs (from social-config.yaml)
  │     ├── Norms (from social-config.yaml)
  │     ├── Trust perceptions (from mindmap overlay)
  │     └── Social awareness (CognitiveProfile.compare())
  ├── Memories, reflections, insights
  └── Recent activity, plans, inventory
```

### After (proposed)

```
System prompt (minimal directive):
  ├── Identity: name, role, voice
  ├── Prime Directives (HARD severity constraints only)
  ├── Templates (voice/style only — behavioral content seeded per D4)
  └── Cognitive preamble (auto-generated: "you have a brain, here's how to use it")

User prompt (observations — sole source of cognitive state):
  ├── World state (unchanged)
  ├── Cognitive sections (CognitionCore.promptSections())
  │     ├── Mood, Drives, Narrative, Mental model, User model, Strategy, Goals
  │     ├── Personality
  │     └── Soft constraints
  ├── CharacterCognition sections (application-specific, no CognitionCore equivalent)
  │     ├── Character motivations (from social-config drives — free-form)
  │     ├── Initial beliefs (from social-config — authored knowledge)
  │     ├── Trust perceptions
  │     ├── Social awareness
  │     └── Norms
  ├── Memories, reflections, insights
  └── Recent activity, plans, inventory
```

## Components

### 1. CognitiveSystemPromptRenderer

**Location:** `blocks-core` (or `blocks-agentic` — wherever the cognitive pipeline lives)
**Package:** `io.casehub.blocks.agentic.social.prompt`

A `@Alternative @Priority(1)` CDI bean implementing `SystemPromptRenderer`. When present on the classpath, it overrides `EidosSystemPromptRenderer`. The eidos renderer stays untouched — no eidos changes.

**Renders:**
1. **Identity block:** Agent name + role sentence. Extracted from `AgentDescriptor.briefing()` — the briefing field is rewritten by content authors to contain only identity and voice content (see §4). The renderer uses it as-is.
2. **Voice block:** Speaking style, mannerisms, catchphrases. From descriptor `briefing` field plus template voice/style content. Templates are split per D4: behavioral patterns (strategies, archetype norms, disposition biases) are removed from templates and moved to seed data — see §4 Template Splitting.
3. **Prime Directives:** Only constraints with `severity: HARD` from `AgentDescriptor.constraints()`. Rendered under a "Prime Directives" heading in the system prompt — the naming signals their non-negotiable weight to the LLM.
4. **Cognitive preamble:** Auto-generated paragraph from active subsystems (see §2 below).

**Does NOT render:** goals, soft constraints, disposition, drives, beliefs, strategies, narrative, or any dynamic cognitive data.

**Render format handling:** The renderer handles `RenderFormat.MARKDOWN` only. For `A2A_CARD` and `PROSE` formats, it delegates to a constructor-injected `EidosSystemPromptRenderer` instance, preserving A2A card generation and alternative format support.

**RenderedPrompt compatibility:** Returns a `RenderedPrompt` with `format: MARKDOWN`, `enriched: false` (no semantic enrichment — the minimal directive is authored text, not prose requiring LLM analysis), and a valid `descriptorHash`/`contextHash` for caching via `RenderedPromptCache`.

**Coherence validation:** The `CognitiveSystemPromptRenderer` returns `null` for `coherenceReport()`. The structural coherence checks in `EidosSystemPromptRenderer` (MBTI/Enneagram consistency, trait alignment) are designed for rich prose briefings and don't apply to the minimal directive format. Consumers that log coherence violations (e.g., `ScenarioOrchestrator.renderPrompt()`) will see null reports and skip logging — this is intentional.

### 2. Cognitive Preamble Generator

**Location:** `blocks-core`, same package as `CognitiveSystemPromptRenderer`
**Class:** `CognitivePreambleGenerator`

**CognitionConfig injection:** `CognitionConfig` is a plain record, not a CDI-managed bean. It is discovered at build time via `CognitionConfigBuildItem` (already implemented in `AgenticYamlProcessor.discoverCognitionConfig()`). The spec requires registering `CognitionConfig` as a CDI synthetic bean via `SyntheticBeanBuildItem` with `@DefaultBean` scope — this aligns with the existing build-time pipeline. `CognitiveSystemPromptRenderer` receives `CognitionConfig` via CDI constructor injection and passes it to `CognitivePreambleGenerator`.

Inspects which `CognitionCore` subsystems are active (via config flags: `config.moodEnabled()`, `config.drivesEnabled()`, etc.) and composes a single coherent paragraph that tells the agent how to use its cognitive faculties.

Example output when mood, drives, mental model, goals, and strategy are active:

> You have an inner life. Your emotional state colours how you respond — let it shape your tone and choices. You have motivational drives that pull at you; some are stronger than others right now. You hold beliefs about the people around you, formed from observation — act on them, update them when evidence contradicts. You have goals that emerged from your motivations — pursue them, reprioritise when circumstances change. You have learned strategies from past interactions — apply what worked, abandon what didn't.

This replaces `DirectiveSection` per-section wrapping. When `CognitiveSystemPromptRenderer` is active, `config.directivePrompts()` should be `false` — the preamble provides the meta-instruction once in the system prompt rather than prepending it to each observation section.

### 3. Neurocortex Seeding

**Generalises:** `ManorCognitiveSeeder` (currently in wacky-manor, manually instantiated)

The seeder reads seed data from character YAML and pushes initial state into cognitive subsystems on first boot (when subsystem state is empty).

**Seed sources (per character):**
- `social-config.yaml` → drives, norms, initial-beliefs, relationships (already exists)
- `descriptors-composite.yaml` → goals (currently in descriptor, moving to seed-only)
- `trust-evolution.yaml` → trust config (already exists, already seeded)

**Seed targets:**
| Source | Target subsystem | Already seeded? |
|--------|-----------------|----------------|
| `initial-beliefs` | MindMapStore (belief nodes) | Yes — ManorCognitiveSeeder |
| `relationships` | MindMapStore (person-entity overlays with PAD) | Yes — ManorCognitiveSeeder.seedPeople() |
| `drives` (character motivations) | N/A — stays rendered by CharacterCognition from SocialConfig | N/A — already observation-side |
| `norms` | Norms rendering (see GitHub issue below) | No — currently read from social-config at render time |
| `goals` | GoalProposalOrchestrator (via DriveGoalProposal) | No — currently from descriptor YAML |

**Drives — taxonomy clarification:** The platform's `DriveOrchestrator` uses a fixed four-axis SDT-derived model: `DriveAxis` = {CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY}. Each axis maps to a specific `DriveSource` implementation with its own calculation logic (`CuriosityDrive`, `CompetenceDrive`, `AffiliationDrive`, `AutonomyDrive`). Wacky-manor's `social-config.yaml` drives are free-form character motivations ("scheming", "gallantry", "social-harmony", etc.) — these are NOT the same concept. They describe the character's motivational identity, not SDT psychological needs.

Character motivations from `social-config.yaml` remain rendered by `CharacterCognition` from `SocialConfig`. They are already on the observation side — the architectural target. There is no CognitionCore equivalent for free-form character motivations: `DrivePromptSection` renders the four-axis SDT model (dynamically computed by `DriveOrchestrator`), and `MentalModelPromptSection` renders BDI Theory of Mind (per-subject beliefs/desires/intentions about OTHER agents) — neither handles authored motivational identity. The four-axis `DriveOrchestrator` continues unchanged.

**Goals — seeding mechanism:** Static goals move from `descriptors-composite.yaml` to `social-config.yaml` with explicit `DriveAxis` annotations. The seeder constructs `DriveGoalProposal` records from the goal config:

```yaml
penelope-pitstop:
  goals:
    - name: solve-mystery
      description: "Uncover the secrets of Doily Manor"
      axis: CURIOSITY
      intensity: 0.8
      formation-reason: "character-definition"
    - name: maintain-harmony
      description: "Keep everyone getting along"
      axis: AFFILIATION
      intensity: 0.7
      formation-reason: "character-definition"
```

The seeder calls `GoalProposalOrchestrator.registerGoals()` with the constructed `DriveGoalProposal` list. `EmergentGoalPromptSection` becomes the sole goal renderer, replacing both `CognitiveObservationSections.goalsSection()` and descriptor-side goal rendering.

**Norms — deferred to GitHub issue:** A `NormsPromptSection` in `CognitionCore.promptSections()` is required for norms to follow the same observation-side rendering path as other cognitive data. This is NOT in scope for this spec. **GitHub issue to be filed on casehubio/blocks** to track: "Add NormsPromptSection to CognitionCore for observation-side norms rendering." Until implemented, norms remain rendered by `CharacterCognition` from `SocialConfig`.

**Key change — constraints:** Currently `CharacterCognition.renderCognitiveSections()` reads constraints from `SocialConfig` at render time and produces observation text directly. In the new model, constraints are split: HARD in system prompt, SOFT in `ConstraintPromptSection` (CognitionCore observation sections). Initial beliefs stay rendered by `CharacterCognition` from `SocialConfig` — they are already seeded into MindMapStore (for cognitive comparison via `CognitiveProfile.compare()`) but rendered directly from `SocialConfig`, not via `MentalModelPromptSection` (which serves BDI Theory of Mind, a different concept).

This requires:
- `ManorCognitiveSeeder` enhanced to construct and register `DriveGoalProposal` goals
- `CharacterCognition.renderCognitiveSections()` stops rendering constraints directly — SOFT constraints come from `ConstraintPromptSection` (CognitionCore)
- `CharacterCognition.renderCognitiveSections()` retains: character motivations (from social-config drives), initial beliefs (from social-config), trust perceptions, social awareness, and norms (pending norms issue)

**Seeder lifecycle:** Called once per agent at scenario bootstrap (same point where `ManorCognitiveSeeder.seed()` and `seedPeople()` are called today). Idempotent — if subsystem state already exists, seeding is skipped.

**Architecture question — seeder location:**
- The seeder SPI/interface belongs in `blocks-core` (alongside CognitionCore)
- The implementation that reads from `SocialConfig` + `AgentDescriptor` stays in wacky-manor (application-specific YAML formats)
- ManorCognitiveSeeder is enhanced rather than replaced — it already handles beliefs and relationships, it gains goal seeding (DriveGoalProposal construction and registration)

### 4. Descriptor YAML Changes

**`descriptors-composite.yaml`:**

Current briefing text mixes identity, voice, behavioral instructions, and character knowledge. Example (Penelope):

> You are Penelope Pitstop — a glamorous, sweet-natured Southern belle... You speak with a Southern drawl and use phrases like "Why, how delightful!"... You know Sylvester Sneekly as a helpful and charming estate manager.

In the new model, the `briefing` field is rewritten to contain only identity and voice. The eidos `AgentDescriptor` model is unchanged — `briefing` remains a single string field. The `CognitiveSystemPromptRenderer` consumes the same `AgentDescriptor`; the change is in what content authors put in the `briefing` field and how the renderer uses it.

```yaml
- agentId: penelope-pitstop
  name: Penelope Pitstop
  briefing: >-
    A glamorous, sweet-natured Southern belle and heiress.
    You speak with a Southern drawl and use phrases like
    "Why, how delightful!" and "Oh my stars!"
  # goals: REMOVED — seeded into GoalProposalOrchestrator via social-config.yaml
  constraints:
    - {name: never-break-cover, description: "...", severity: HARD}  # rendered in system prompt
    - {name: stay-in-character, description: "...", severity: SOFT}  # rendered in ConstraintPromptSection (observations)
  templates: [{ref: hanna-barbera-cartoon-style}]  # voice/style only post-D4 split
  disposition:  # kept in YAML but NOT rendered in directive — seeded into subsystems
    mbtiType: ESFJ
    enneagramType: helper
```

**Character knowledge** ("You know Sylvester Sneekly as a helpful estate manager") becomes a seeded belief in `social-config.yaml`:

```yaml
penelope-pitstop:
  initial-beliefs:
    - {key: sneekly-identity, value: "Sylvester Sneekly is a helpful and charming estate manager"}
```

**Template splitting (D4):**

Templates currently contain two categories of content:
1. **Voice/style** — speaking patterns, performance conventions, catchphrases, comedic timing. Stays in the template (system prompt). Example: catchphrase delivery, theatrical outrage.
2. **Behavioral patterns** — strategies, goals, beliefs, constraints, archetype norms. Moves to seed data in `social-config.yaml` or descriptor `constraints`. Example: "scheme step by step" (cartoon-villain), "protect at all costs" (cartoon-protector).

Classification routing for behavioral content removed from templates:

| Behavioral type | Seed location | Rendering path |
|---|---|---|
| Goals/missions ("protect X at all costs") | `social-config.yaml` goals | GoalProposalOrchestrator → GoalPromptSection |
| Beliefs/suspicions ("suspicious of anyone too helpful") | `social-config.yaml` initial-beliefs | CharacterCognition (already observation-side) |
| Behavioral constraints ("never give up", "never confront directly") | descriptor `constraints` (SOFT severity) | ConstraintPromptSection |
| Archetype norms, genre conventions, behavioral tendencies | `social-config.yaml` norms | Pending NormsPromptSection (see GitHub issue) |

**Example — cartoon-villain post-split:**

Voice/style (stays in template):
- Theatrical outrage when foiled — performance style
- Catchphrase delivery (`${catchphrase}`) — speaking pattern

Behavioral (moves to `social-config.yaml` norms for characters referencing this template):
- "Monologue plans, explaining scheme step by step" → archetype norm
- "Gloat prematurely" → archetype norm
- "Plans always elaborate when simple would work" → archetype norm
- "Immediately planning next scheme" → archetype norm

Post-split, templates contain ONLY voice/style lines. Behavioral content from templates becomes archetype-level seed data in `social-config.yaml`: when a character references a template, the character's `social-config.yaml` entry includes the archetype's behavioral seeds (norms, goals, beliefs, constraints) alongside per-character seeds.

**Gap:** Most behavioral template content classifies as archetype norms. Until `NormsPromptSection` is implemented (tracked by the norms GitHub issue — see §3), behavioral template content classified as norms remains in templates. The norms GitHub issue scope now covers both social-config norms AND behavioral template norms. Goals, beliefs, and SOFT constraints from templates can be seeded immediately using existing subsystems.

### 5. Deduplication

Remove all observation-side content that duplicates what CognitionCore already renders:

| Currently duplicated | Remove from | Keep in |
|---------------------|-------------|---------|
| Goals | System prompt (descriptor) + `CognitiveObservationSections.goalsSection()` | `EmergentGoalPromptSection` (CognitionCore) — seeded via `registerGoals()` |
| Constraints | System prompt (all severities) + `CharacterCognition` direct render | System prompt as Prime Directives (HARD only) + `ConstraintPromptSection` (SOFT only, via CognitionCore) |
| Personality/disposition | System prompt + `SocialAvatarCognition.buildSections()` | `PersonalityPromptSection` (CognitionCore) |

**Not duplicated — stays in CharacterCognition:** Character motivations (from social-config drives — free-form, no CognitionCore equivalent), initial beliefs (authored knowledge — distinct from `MentalModelPromptSection`'s BDI Theory of Mind), trust perceptions, social awareness, norms. These are application-specific content already on the observation side.

**Constraint severity filtering:** `CognitionCore.promptSections()` currently passes ALL constraints from `lastDescriptor.constraints()` to `ConstraintPromptSection` without filtering. This spec requires `CognitionCore.promptSections()` to filter for `severity != HARD` before constructing `ConstraintPromptSection`. HARD constraints are rendered as Prime Directives by `CognitiveSystemPromptRenderer` in the system prompt. Without this filtering, HARD constraints would appear in BOTH channels.

**PersonalityPromptSection deduplication:** `SocialAvatarCognition.buildSections()` (line 112) adds `PersonalityPromptSection` from the descriptor, then `core.promptSections()` (line 115) adds another `PersonalityPromptSection` from `CognitionCore` (line 347). This is an existing duplication within the observation pipeline. Fix: remove the explicit `PersonalityPromptSection` from `SocialAvatarCognition.buildSections()` — CognitionCore already provides it.

**CognitiveObservationSections callers:** Production code callers of `CognitiveObservationSections`:
- `DrivePromptSection` → `motivationalStateSection(DriveProfile)` — CognitionCore prompt section, unchanged
- `NarrativePromptSection` → `narrativeSection(NarrativeState)` — CognitionCore prompt section, unchanged
- `CharacterCognition.renderTrustSections()` → `trustSection(List<TrustSummary>)` — retained in CharacterCognition
- `goalsSection(List<AgentGoal>)` — called only from tests; safe to deprecate

The `CognitiveObservationSections` class is NOT removed — it remains as a rendering utility used by CognitionCore's prompt sections. Only `goalsSection()` is deprecated as goals move to `EmergentGoalPromptSection` via seeded `DriveGoalProposal` objects.

After deduplication, `CharacterCognition.renderCognitiveSections()` retains:
- Character motivations (from social-config drives — free-form, no CognitionCore equivalent)
- Initial beliefs (from social-config — authored knowledge, distinct from `MentalModelPromptSection`'s BDI Theory of Mind)
- Trust perceptions (from mindmap overlay — no CognitionCore equivalent)
- Social awareness (perspectival comparison — no CognitionCore equivalent)
- Norms (until `NormsPromptSection` is added — tracked by GitHub issue)
- Constraints removed: HARD → Prime Directives (system prompt), SOFT → `ConstraintPromptSection`

### 6. DirectiveSection Deprecation

`DirectiveSection` wrapping (`config.directivePrompts()`) is replaced by the cognitive preamble in the system prompt. When `CognitiveSystemPromptRenderer` is active:
- `config.directivePrompts()` returns `false`
- `CognitionCore.promptSections()` returns unwrapped sections
- Each section's heading and structure must be self-explanatory (section headings like "Your Current Mood", "Your Drives", "What You Believe" already achieve this)

The `DirectiveSection` class is not deleted — it remains available for non-cognitive rendering paths.

## Cross-Repo Scope

| Repo | Changes | Issue |
|------|---------|-------|
| **blocks** | `CognitiveSystemPromptRenderer`, `CognitivePreambleGenerator`, seeder SPI, `CognitionCore` config changes, norms prompt section | casehubio/blocks#283 |
| **examples/wacky-manor** | Descriptor YAML restructure, `ManorCognitiveSeeder` enhancement (motivational beliefs + goal seeding), `CharacterCognition` deduplication, `ScenarioOrchestrator` wiring | **TODO:** File issue on casehubio/examples before implementation begins |
| **neocortex** | No changes expected — consolidation pipeline already observation-side | None |

## Empirical Validation

This architectural change alters how the LLM processes agent personality. Validation requirements:

1. **Constraint adherence:** Hard constraints in system prompt must be followed as reliably as before. Test with Hooded Claw's `never-break-cover` constraint.
2. **Cognitive section influence:** Mood, drives, and trust perceptions should visibly influence agent behavior (not just appear in the prompt). Compare agent responses before/after the change.
3. **Cold-start quality:** Seeded initial state must produce agent behavior comparable to current briefing-heavy prompts.
4. **Multi-model sensitivity:** If eidos supports multiple LLM backends, verify that minimal-directive + rich-observations works across model families. System prompt vs user-turn attention weight varies by model.

The LLM eval framework (existing `llm-eval` Maven profile) can be extended with tests for these dimensions.

## References

- `EidosSystemPromptRenderer` — `casehub-eidos/runtime/renderer/EidosSystemPromptRenderer.java`
- `SystemPromptRenderer` SPI — `casehub-eidos-api`
- `CognitionCore.promptSections()` — `blocks-core/.../social/CognitionCore.java:344`
- `DirectiveSection` — `blocks-core/.../social/prompt/DirectiveSection.java`
- `ManorCognitiveSeeder` — `wacky-manor/.../agent/ManorCognitiveSeeder.java`
- `CharacterCognition` — `wacky-manor/.../agent/CharacterCognition.java`
- `ScenarioOrchestrator.renderPrompt()` — `wacky-manor/.../agent/ScenarioOrchestrator.java:543`
- `descriptors-composite.yaml` — `wacky-manor/src/main/resources/META-INF/eidos/`
- `social-config.yaml` — `wacky-manor/src/main/resources/META-INF/eidos/`
- `templates.yaml` — `wacky-manor/src/main/resources/META-INF/eidos/`
- Decision review findings — `reviews/casehub-slots/blocks-283-decision-20260916-104401/`
- Decisions — `specs/issue-63-fix-cdi-config-beans/decisions.md`
