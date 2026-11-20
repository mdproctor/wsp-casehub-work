---
layout: post
title: "The Field That Wasn't Behavioral"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/examples]
tags: [wacky-manor, cognitive-architecture, prompt-engineering, block-tags, response-format, channel-emphasis, appraisal]
---

# The Field That Wasn't Behavioral

The response format for character ticks has accumulated behavioral directives embedded directly in JSON field descriptions. The `thinking` field alone carries a 100-word essay about somatic markers, attentional focus, temporal processing, and action tendencies — references to Damasio (1994), Easterbrook (1959), Barrett (2017), and Frijda (1986) — all wired into what's supposed to be a JSON schema definition. Issue #153 asked for an audit: classify each instruction as schema or behavioral, move behavioral content into the block tag system, reduce the response format to pure schema.

The classification was straightforward. The `thinking` field's multi-channel introspection guidance and the `moodImpact` field's "Be honest — negative is valid" anti-bias directive are behavioral. The value ranges, field types, and structural constraints are schema. Everything else (dialogue, talkTo, aside, action) was already clean.

## The Wrong Fix

The obvious fix: move the behavioral content into `[APPRAISAL]` and `[MOOD]` block tags via `CharacterCognition.renderTaggedSections()`, strip the response format to pure schema. This compiled, passed all 644 tests, and looked structurally correct.

It was wrong.

## Two Types of "Behavioral"

First-principles review surfaced a category error. The block tag system handles *character behavioral guidance* — how drives shape topic choice, how mood colours speech, how personality patterns influence responses. This is content about the character's nature and state. The existing directives in `DirectiveSection.java` follow this pattern:

- `[MOOD]` — "Your emotional state shapes how you speak — warmth shows in generosity of thought, arousal in the pace and intensity of your words"
- `[DRIVES]` — "These motivations pull at you right now. The strongest drive should steer what you choose to talk about"
- `[BEHAVIORAL]` — "These are your behavioral patterns. Established patterns are deep tendencies shaped by accumulated experience"

The content I was trying to move is different. "Notice your body — a tightness, a warmth, a numbness" is not character behavioral guidance. It's *response format calibration* — instructions about how to fill a specific JSON output field. The multi-channel thinking prompt tells the LLM what kind of content to write in the `thinking` field. "Be honest — negative is valid" tells it not to bias the `moodImpact` values toward positive. These are about the output format, not about the character.

Putting response format calibration into block tags creates three problems:

**Semantic collision.** The existing `[APPRAISAL]` content from `ChannelEmphasisPromptSection` provides dynamic state data — "your somatic channel is heightened right now." Adding a meta-instruction — "Notice your body" — to the same tag conflates state data with output guidance.

**Severed field connection.** Block-tagged sections render at the top of the observation prompt. The response format is at the bottom. The `[APPRAISAL]` directive says "notice your body" but doesn't reference the `thinking` field. Hundreds of tokens separate the instruction from the output field it was designed to calibrate.

**Non-cognitive regression.** Tagged sections flow through `CharacterCognition.renderTaggedSections()`, which only fires on the cognitive path. Characters without cognition silently lose all multi-channel guidance — worse than the original.

## The Corrected Approach

Response format calibration stays in the response format, but concise:

**Old** (100+ words): "What's happening inside you right now? Notice your body — a tightness, a warmth, a numbness. Notice what's grabbing your attention — what you can't look away from, what you're avoiding. Notice time — does the moment feel urgent, or suspended?..."

**New** (45 words): "your private inner monologue — attend to what feels vivid: body (tension, warmth), attention (what pulls, what you avoid), time (urgent or suspended), action pull. Raw, unperformed, sometimes confused — the character's voice belongs in dialogue. Shown to you next turn."

All four channels preserved. Uncertainty permission ("sometimes confused") preserved. Thinking/dialogue boundary preserved. Two things changed:

**"Attend to what feels vivid"** replaces the static "Notice body." This defers to the `ChannelEmphasisPromptSection` when it dynamically suppresses or foregrounds channels. If the cognitive pipeline says "Background your somatic," the static version contradicts it; the deferred version doesn't. When no channel emphasis is active, the four named channels serve as the baseline.

**Concise calibration lines** replace verbose preambles. `situationChangeDegree` and `moodImpact` descriptions were trimmed to their calibration anchors — the value ranges and anti-bias guidance — without the imperative framing.

## Adversarial Review Findings

A subagent adversarially challenged the implementation and produced six findings:

| Finding | Severity | Status |
|---------|----------|--------|
| ChannelEmphasis contradiction (static "Notice body" vs dynamic "Background somatic") | Medium | **Fixed** — "attend to what feels vivid" defers to dynamic emphasis |
| "Sometimes confused" permission dropped | Low-Medium | **Fixed** — restored |
| Embodiment framing shift (interrogative → parenthetical) | Low-Medium | **Open** — empirical question, needs thinking-field quality eval |
| moodImpact calibration adequate | Low | No action |
| Ordering correct (response format at end of prompt) | Low | No action |
| Non-cognitive path proportional | Low | No action |

## What Remains

Finding 3 — the shift from embodiment framing to parenthetical calibration — is an empirical question. The old version used indefinite articles and questions ("a tightness, a warmth, a numbness") that may produce more phenomenological inner monologue than the new parenthetical list ("body (tension, warmth)"). We have no eval instrument that measures thinking field register, so this can't be validated without building one.

This is the right trade-off for now. The architectural fix (ChannelEmphasis deference) addresses a real system contradiction. The condensation risk is second-order — the thinking field is shaped by many cognitive pipeline inputs, and the field description is one among them. But the risk exists, and the eval instrument is a legitimate follow-up.

## The Design Principle

**Response format calibration ≠ character behavioral guidance.** Both look like "behavioral directives" in the text, but they serve different systems. Block tags handle character state and identity — content the character processes as part of who they are. Response format calibration handles output field formatting — instructions the LLM follows when generating JSON. Mixing them creates semantic collision in the prompt, architectural coupling between systems, and invisible regression on code paths that don't load the cognitive pipeline.

Also fixed while here: pre-existing test compilation failures from #150's `MoodImpact` field addition, plus a null-action fallback in `AgentResponse.parse()`.
