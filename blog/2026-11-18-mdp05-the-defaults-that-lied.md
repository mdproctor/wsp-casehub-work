---
layout: post
title: "The Defaults That Lied"
date: 2026-11-18
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cognition, theory-of-mind, bayesian-inference, personality, pid-5, defaults, observation-builder, evidence-weighting]
series: cognitive-pipeline
---

# The Defaults That Lied

[Continues from [The Evidence Nobody Trusted](2026-11-18-mdp04-the-evidence-nobody-trusted.md)]

The previous entry ended with two known gaps. Both turned out to illustrate the same principle: personality modulation only works when it actually reaches the code that makes decisions. Getting the mechanism right matters less than getting the wiring right.

## The wrong default

`tomFacetScore` extracts a PID-5 personality facet from an agent's profile. When a facet is absent — no explicit score for that dimension — it falls back to a default. The default was 0.5 for everything.

That's correct for anxiousness. The `NEUTRAL` baseline is `(0.5, 0.0, 0.0)` — a midpoint that says "neither particularly anxious nor particularly calm." But callousness and suspiciousness have a different neutral: 0.0. Zero callousness means full empathic coupling in the behaviour|emotion CPT. Zero suspiciousness means no evidence scaling — approach signals pass through unmodified.

An agent with no explicit personality was getting callousness = 0.5, which weakens the distress→avoidant coupling in the Bayesian network by half. And suspiciousness = 0.5, which transfers 30% of cooperative behaviour evidence toward avoidant. Every agent without explicit PID-5 facets was moderately callous and moderately suspicious by default.

The fix is a one-parameter change: `tomFacetScore` now takes a `defaultValue` argument. Anxiousness passes 0.5; callousness and suspiciousness pass 0.0. The defaults match `NEUTRAL` exactly.

## The path that skipped everything

The second fix was more structural. `CognitionObservationBuilder.observeCharacterState()` creates a `TheoryOfMindEngine` per observed character, feeds in behaviour and threat evidence, and infers an emotional posterior. This is the path the cognitive pipeline uses to build Theory of Mind observations during prompt construction.

It was creating engines with no arguments — always `NEUTRAL` modulation. It was feeding behaviour as hard enum evidence — a one-hot vector, `APPROACH` or `AVOIDANT`, with no scaling. All three personality dimensions were bypassed.

The bridge path (`TheoryOfMindBridge.infer()`) already did everything right: construct the engine with personality modulation, convert appraisal dimensions to soft evidence, scale behaviour vectors by suspiciousness. The observation builder was a parallel path that never got the same treatment. Pre-existing before the suspiciousness work in #1127, but invisible while only anxiousness existed — a single missing dimension is harder to notice than three.

![Before and after: CognitionObservationBuilder now routes through personality-modulated ToM](svg/tom-bypass-fixed.svg)

The fix resolves the observer's `TomPersonalityModulation` once per builder (lazy, cached — the builder is per-tick, so the personality is stable) and passes it to every engine it creates. Behaviour evidence is converted from the enum to a soft distribution and scaled through `TheoryOfMindBridge.scaleBehaviorEvidence` when suspiciousness is non-zero. The suspiciousness guard (`> 0.0`) preserves the zero-overhead path for the common case.

Two visibility changes were necessary: `CognitionCore.tomModulation` went from private to public (the builder needs personality modulation from a different package), and `TheoryOfMindBridge.scaleBehaviorEvidence` from package-private to public (same reason). Both are pure query methods with no side effects — the exposure is safe.

## Why parallel paths diverge

The pattern is worth naming because it recurs. A bridge or orchestrator implements the full mechanism. A secondary consumer creates the same underlying engine directly, skipping the bridge. When the bridge gains a new capability, the secondary path doesn't. The divergence is silent because both paths produce plausible output — the unmodulated engine still infers, still returns posteriors, still looks correct. You only notice when you audit whether personality modulation is actually reaching every inference site.

The fix for the pattern is the same as the fix for the code: route through the bridge, not around it. The observation builder now resolves modulation from `CognitionCore` and applies the same evidence scaling the bridge uses. If a fourth personality dimension is added to the ToM engine, it reaches both paths through the same mechanism.

The defaults were wrong and the wiring was incomplete. Both were small fixes — a parameter here, a constructor argument there. The gap between "the mechanism exists" and "the mechanism is used everywhere" is where personality modulation actually lives.
