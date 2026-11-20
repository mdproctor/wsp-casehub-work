---
layout: post
title: "The Body Already Knows"
date: 2026-10-28
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
tags: [wacky-manor, cognitive-architecture, barrett, phenomenology, engrams, drive-expression]
---

# The Body Already Knows

I spent two iteration cycles polishing hand-crafted drive descriptions before realising I was polishing the wrong thing entirely.

The `social-config-generic.yaml` file had accumulated 640 lines of second-person somatic prose — paragraphs like "When someone speaks over you, heat climbs your neck before you've registered the words" — injected verbatim into every LLM prompt. I was iterating on that prose, tweaking word choices, adjusting somatic imagery, measuring the result with the eval instrument. The scores were moving. The approach felt productive.

It was templating. The cognitive architecture — CAPS, the consolidation pipeline, the appraisal strategy — was designed to compute this content per-turn from structured data. The hand-crafted descriptions short-circuited all of it.

## How It Got There

Git archaeology told the story. On October 6, a session explicitly stripped all drive descriptions to type and intensity only. The commit message said "sub-LLM appraisal computes the felt-state per-turn." Three days later, the descriptions came back — longer, more elaborate, justified as "somatic grounding." Each subsequent session found the computed pipeline output wasn't rich enough and, instead of fixing the pipeline, wrote better prose. The eval scores rewarded it because hand-crafted prompt engineering IS effective. It just bypasses the architecture.

## Barrett's Four Channels

The replacement is a universal instruction grounded in Lisa Feldman Barrett's constructionist theory and three supporting frameworks:

1. **Body** — what the body does before the mind labels it (Damasio's somatic markers)
2. **Attention** — what captures focus, what narrows (Easterbrook's cue utilization)
3. **Time** — temporal compression, stretching, urgency
4. **Action readiness** — what the body prepares to do (Frijda's action tendencies)

No per-drive templates. The LLM receives the drive type, the intensity, the character's personality scores, and the instruction to construct the felt experience across these four channels. The same instruction works for any character, any drive, any situation.

The key insight was treating personality scores as body calibration, not cognitive traits. "Callousness: 90" doesn't mean the character thinks cruel thoughts — it means the place where other people feel sympathy is quiet in their body. Attachment style is the somatic ground state: dismissive-avoidant bodies pull away from warmth; secure bodies settle into closeness. These shape how every drive manifests, and the LLM computes the specific expression from the structural data.

## Testing as a Function

The drive expression generation is a pure function: personality profile + drives + situation + people present → four-channel somatic text. We built a lightweight test harness that calls the LLM directly — no Quarkus, no CAPS, no MindMap. Ten test cases covering different characters in different situations with different people present. An LLM judge scores each output on four dimensions: bespoke (character-specific), somatic (body-first), situational (context-responsive), discovery (finding rather than reporting).

The V2 instruction scored 4.6 mean across all dimensions. Somatic hit 5.0 — a perfect score for body-first expression. The discovery dimension plateaued at 4.2 until we added consolidation-derived Barrett profiles — character-specific hints about how each drive tends to manifest in this person's body, computed from their personality and formation memories during the "sleep" consolidation phase. With profiles, discovery broke through to 4.5.

We also proved what doesn't work. V3 added structural constraints — banning drive names from the output, banning causal reasoning, forcing mid-sensation endings. Every constraint hurt. Banning cognitive processing severs the body-to-situation connection that makes the output rich. The ceiling on discovery isn't solvable by restriction; it needs richer input material.

## Engrams Replace Prose

The personality data was flowing through the wrong pipe. The `ManorCognitiveSeeder` was creating ad-hoc MindMap nodes — a parallel data path that duplicated what the biography pipeline already handles properly. The biography system uses structured YAML engrams: personality profiles with 13 psychological frameworks (AMPD, OCEAN, MBTI, Jungian function stack, Enneagram, DISC, Belbin, Thomas-Kilmann, SDI, Attachment, Locus of Control), plus archetypes, disposition axes, life events with PAD values and drive impacts, relational data, goals, and beliefs.

We wrote engrams for all five characters. Twenty-five YAML files, fully enriched, audited for cross-framework consistency. No prose anywhere — structured data throughout. The biography import pipeline loads them per-character on startup, writing personality properties to MindMap person nodes where the consolidation phase can read them.

## First Full Run

The first 5-character run with enriched engrams and Barrett instruction produced mixed results. AHM's protection and suspicion improved significantly — quality up 0.50, register up 0.63-0.70. DD's scheming register jumped 0.67. Peter's proving-worth quality rose 1.10.

But Peter's gallantry register dropped from 3.40 to 1.20, and his protection from 4.00 to 2.30. HC's gloating regressed across both dimensions. The root causes are identified: formation memories were over-zealously gated in computed mode (they're biographical data, not prose templates), and the BASELINE profile leaked cartoon character names through agentIds. Both are straightforward fixes.

The architecture is sound. The Barrett instruction produces character-specific somatic expression from structured data alone — proven at 4.7 in the harness, confirmed through the full pipeline with three characters producing relational dynamics visible in their thinking fields. What remains is closing the gaps where computed mode doesn't yet match the prose baseline, and that's iteration, not redesign.

The body already knows what it feels. The job of the architecture is to let it discover that — not to script the answer in advance.
