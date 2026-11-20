---
title: "Proving Emergence — When the Cognitive System Earns Its Keep"
date: 2026-09-28
entry_type: note
subtype: diary
series: issue-89-briefing-voice-card
projects: [casehubio/examples, casehubio/blocks]
author: mdp
tags: [cognitive-systems, emergence, evaluation, mood, voice-cards, wacky-manor]
---

# Proving Emergence — When the Cognitive System Earns Its Keep

The voice card work is done — four characters stripped to voice-only
descriptors, persona switching wired, behavioral instructions extracted
to SocialConfig. The cognitive system is supposed to carry the behavior
that used to be scripted in the briefing. But "supposed to" isn't
evidence.

The structural emergence eval from last session proved the machinery
*exists*: CognitionConfig.all() adds observation sections that .none()
doesn't. MoodPromptSection showed up at tick 10. But a section existing
and a section influencing behavior are different claims. A character
with a mood section that reads "Pleasure: 0.00 (neutral)" doesn't
behave any differently than one without it.

## The Experimental Design

I wanted a clean test of the emergence claim: same character, same voice
card, same situation — different cognitive state. If the LLM responds
differently, the cognitive system produced it. If not, it's dead weight.

We settled on paired state probes. For each scenario, build two
CharacterCognition instances — one with pre-event state, one with
post-event state. Probe both with the same situation prompt. Add a
no-cognition control pair (identical prompts, no observation sections)
to establish the random-variation baseline. An LLM judge scores the
behavioral delta 0-5.

The dual assertion is the key: cognitive delta must be ≥3 (visible
change), AND must exceed the control delta (not random variation).
Without the control, you can't distinguish emergence from the LLM's
inherent non-determinism.

## Three Out of Four

Four scenarios, each targeting a different cognitive subsystem:

**Trust erosion (Penelope, 5/5):** Before the event, Penelope's beliefs
say "Sylvester Sneekly is a helpful estate manager." After discovering
he lied about the treasure map, the belief changes to "Sneekly may not
be trustworthy." Same situation: Sneekly offers to guide her to a
hidden room. The before response was warm, trusting, eager to follow.
The after response referenced the lie by name, refused to go without
others, and delivered a pointed aside. Control delta: 1. The belief
change flipped her behavior entirely.

**Persona switching (Hooded Claw, 4/5):** Before: alone in a room,
Claw persona active. After: Penelope enters, Sneekly persona active.
Same situation: a valuable compass on the mantelpiece. Before was open
scheming — "bait for an elaborate trap." After was performatively
innocent — "I shall take custody, wouldn't want it to fall into the
wrong hands." Control delta: 0. The persona activation section did
its job.

**Scheme frustration (Hooded Claw, 3/5):** Drives shifted from
scheming at 0.9 to 0.5 (frustrated), beliefs revised from "Penelope
is naive" to "more observant than expected." The LLM shifted from
immediate confident TAKE to cautious LOOK with hedging language.
Control delta: 0-1. The signal was clear but less dramatic than the
trust scenario — drives are a softer lever than beliefs.

**Mood elevation (Dick Dastardly, 1/5 → 3/5):** This one failed
first. And failing was the most useful thing the eval did.

## The Mood Gap

MoodPromptSection was rendering the PAD model as raw numbers:

```
Current emotional state:
- Pleasure: 0.60 (positive)
- Arousal: 0.20 (balanced)
- Dominance: 0.30 (balanced)
```

Claude the judge's verdict was blunt: "The AFTER response shows no
trace of the event — both responses are nearly identical in tone."
Score: 1. Control delta: 0. The mood subsystem was functionally dead.

The other three subsystems work because they communicate in natural
language. Beliefs say "Sneekly lied about the map" — the LLM reasons
from that. Norms say "Never help Penelope directly" — the LLM follows
that. Persona says "Active Voice: Sneekly" — the LLM adopts that.
Mood says "Pleasure: 0.60 (positive)" and the LLM does nothing,
because it has no training data mapping PAD dimensions to behavior.

The fix was straightforward: translate PAD to Russell's circumplex
emotion labels plus behavioral coloring. Instead of "Pleasure: 0.60
(positive)", render "You're feeling pleased, with a sense of
confidence. This colors your responses — you're more generous and
open, more assertive and commanding."

After the fix, mood-elevation scored 2-3 — borderline. The judge
noted "explicit 'good mood' in thinking, more grandiose
self-references, warmer tone toward Muttley." Not a clean pass like
the trust scenario, but the subsystem went from dead to alive.

## Why Mood Is Still Weak

Mood is inherently a weaker signal than beliefs, drives, or persona.
It *colors* behavior rather than *directing* it. "You're feeling
pleased" is softer guidance than "Sneekly lied" or "Active Voice:
Sneekly." The LLM has to interpret a mood label into behavioral
changes; with beliefs or personas, the behavioral implication is
already in the content.

The real fix isn't in the rendering — it's in the emotion
architecture. Blocks has three open issues that would make mood
load-bearing: #301 (goal emotions feeding into mood, so mood actually
*moves* in response to cognitive events rather than decaying toward
baseline), #302 (mood biasing subsequent appraisals, so a frustrated
character evaluates new situations more negatively), and the parent
epic #298 tying it together.

Without those, MoodPromptSection is a label on a number that barely
changes. The rendering fix was necessary — even with the emotion
architecture, the LLM needs emotional labels, not PAD numbers. But
it's the last mile, not the foundation. The foundation is the emotion
wiring in blocks and neocortex.

## What This Opens Up

The emergence eval is now a standing capability. Any change to the
cognitive system — a new subsystem, a rendering tweak, an architecture
change — can be validated by running the four scenarios. Each one
isolates a different pathway and measures whether the LLM actually
responds to it.

The blocks neocortex integration work (#311 epic, plus #301/#302 for
mood specifically) is the next priority. Nine issues across the epic,
three of them S-scale. The small ones wire individual neocortex
capabilities — reflection consumption, engagement streams, temporal
focus — that are already built in neocortex but not yet exposed
through blocks into the agent loop. Once wired, each one becomes a
new scenario in the emergence eval.

The pattern is becoming clear: build the subsystem in neocortex,
wire it through blocks, render it as actionable natural language in
the observation layer, and prove with the eval that the LLM responds
to it. Any subsystem that doesn't move the eval needle isn't earning
its keep.
