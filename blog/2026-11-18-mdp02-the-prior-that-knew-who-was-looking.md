---
layout: post
title: "The Prior That Knew Who Was Looking"
date: 2026-11-18
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cognition, theory-of-mind, bayesian-inference, personality, cpd, bar-haim, blair, anxiety, callousness, appraisal]
series: cognitive-pipeline
---

# The Prior That Knew Who Was Looking

Yesterday's entry wired the Bayesian Theory of Mind engine into `CognitionCore.tick()`. Every agent now infers how the person in front of them is feeling — a three-node junction tree producing a posterior over positive, neutral, and distressed. The engine works. The bridge maps SEC dimensions and action tendencies into soft evidence. The inference runs every tick, every subject, at negligible cost.

The problem: every agent runs the same inference with the same priors. A callous manipulator reads distress signals identically to an empathic caregiver. An anxious agent assumes exactly the same baseline threat as a relaxed one. The Bayesian machinery was personality-blind.

This matters more than it might seem. Theory of Mind research consistently shows that personality shapes not just how people feel, but how they read others. Bar-Haim's 2007 meta-analysis of 172 studies demonstrated that anxious individuals show a reliable attentional bias toward threat — they don't just worry more, they literally perceive more threat in the same environment. Blair's work on callous-unemotional traits (2005, 2008, 2013) shows the complementary deficit: reduced empathic response means weaker coupling between observed distress signals and inferred emotional state. A callous observer sees avoidance behaviour and shrugs; an empathic one sees the same behaviour and infers distress.

## Two dimensions, two CPT modulations

The fix is a `TomPersonalityModulation` record carrying two facet scores extracted from `EffectiveCognitiveDefaults` at each tick: anxiousness and callousness. Both are already tracked in the personality facet system (PID-5 derived, [0,1] range), already modulated by situational displacement, already used by `CognitiveDerivationEngine` for a dozen other cognitive parameters. The ToM engine was the last holdout.

**Anxiety shifts the threat prior.** The original prior was `{NONE: 0.60, MILD: 0.30, SEVERE: 0.10}` — a reasonable baseline for a generally non-threatening world. At high anxiety (1.0), this shifts to `{0.30, 0.40, 0.30}`. At low anxiety (0.0), it relaxes to `{0.80, 0.15, 0.05}`. The interpolation is linear, keyed on the midpoint: anxiety = 0.5 reproduces the original prior exactly.

**Callousness weakens the distress→avoidant coupling.** The emotion→behaviour CPT's distressed row was `{APPROACH: 0.05, NEUTRAL: 0.20, AVOIDANT: 0.75}` — if someone is distressed, they'll probably avoid you. At high callousness (1.0), this shifts to `{0.15, 0.50, 0.35}`. The callous observer still sees avoidant behaviour but reads it as ambiguous rather than diagnostic. The coupling between "they're avoiding me" and "they're distressed" loosens.

![Personality-modulated Theory of Mind — anxiety shifts the threat prior, callousness weakens the distress-avoidance coupling](svg/personality-modulated-tom.svg)

## Why linear interpolation works here

A subtlety that turned out to be a feature: linear interpolation between two CPT rows that each sum to 1.0 always produces a row that sums to 1.0. No renormalisation needed. The proof is trivial — if `sum(a) = 1` and `sum(b) = 1`, then `sum(a + t*(b-a)) = sum(a) + t*(sum(b) - sum(a)) = 1 + t*(1-1) = 1`. This means the personality modulation can't produce an invalid probability table, regardless of the facet scores. The `BayesVariable` constructor validates row sums, so a bug would surface immediately as an `IllegalArgumentException` — but the maths guarantee it won't fire.

The design avoids the temptation to modulate the emotion→threat CPT as well. That table represents a causal relationship (threat causes emotional distress) that holds regardless of the observer's personality. What personality changes is the prior expectation of threat and the observer's ability to read behaviour as evidence. The causal mechanism stays fixed; the perceptual lens shifts.

## The wiring

`CognitionCore.tick()` already has access to `EffectiveCognitiveDefaults` — it uses them three lines earlier to set the memory modulation context. The new code extracts the two facets, builds a `TomPersonalityModulation`, and passes it to `TheoryOfMindBridge.infer()`. Missing facets default to 0.5 (anxiousness) and 0.0 (callousness), which reproduce the original CPTs exactly. Agents without personality data behave identically to before.

`TheoryOfMindBridge` gained an overload: `infer(AppraisalResult, TomPersonalityModulation)`. The original `infer(AppraisalResult)` delegates with `NEUTRAL`. Since the bridge already creates a fresh engine per call — three nodes, cheap — the personality-conditioned network is built at construction time. No shared mutable state between concurrent agent ticks.

## Cache eviction: the other thing

While the ToM personality work was the main thread, we also closed #1089 — unbounded cache growth in `SituationalStateService` and `SituationalPersonalityParticipant`. Both had `ConcurrentHashMap` caches keyed by agent ID with no eviction. With a fixed cast of 17 characters, this is harmless. With dynamic agent creation — which the architecture supports but hasn't needed yet — the maps grow without bound.

The fix follows the pattern already established by `EffectiveCognitiveDefaults.evictIdle()`: add a `lastAccess` timestamp map alongside the cache, update it on every read and write, and expose `evictIdle(Duration)` for periodic sweep. Nothing calls these methods yet — the periodic scheduler is a separate concern. The API is ready for when dynamic agents arrive.

## What this opens up

Personality now shapes not just what agents feel (CAPS/SEC appraisal), how they cope (regulation strategies), how they talk (expression styles), and what they remember (salience modulation) — but how they read other people. An anxious agent in a room with a neutral conversation partner will infer mild threat where a relaxed agent infers none. A callous agent watching someone withdraw will shrug it off where an empathic agent reads distress.

The next gap is suspiciousness. The ToM engine currently doesn't modulate how much an agent trusts its own inference — a suspicious agent should discount cooperative signals and amplify hostile ones, not at the CPT level but at the evidence weighting level. That's a different mechanism: soft evidence scaling rather than prior adjustment. It's worth doing, but it's a separate issue.
