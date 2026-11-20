---
layout: post
title: "Knowing When the Model Is Wrong"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [emotion-regulation, prediction-calibration, personality-prediction, observability, cognitive-architecture, CAPS, literature-grounded]
series: issue-921-prediction-calibration-analysis
---

# Knowing When the Model Is Wrong

The regulation predictor has been running for weeks. It reads an agent's personality weights — derived from Barańczuk's 2019 meta-analytic correlations between Big Five traits and emotion regulation strategies — and pins a strategy before the LLM ever sees the prompt. COGNITIVE_CHANGE for a reflective character. RESPONSE_MODULATION for an avoidant one. ATTENTIONAL_DEPLOYMENT when cognitive resources are spent.

It is, by construction, a population-level model. The weights say what works for the average person with a given personality profile. But agents are not averages. A competitive, analytical character might get COGNITIVE_CHANGE pinned at 0.70 effectiveness, while the LLM — reasoning about the specific situation, the conversation history, the social context — produces 0.85 for that same character because it can see what the predictor cannot.

Until now, we had no way to know.

## The Source Tracking That Made This Possible

Issue #844 added `strategySource` tracking to every regulation episode. Each time the regulation cascade fires, it records which layer made the decision:

- **SAFETY_GATE** — resource exhaustion forced ATTENTIONAL_DEPLOYMENT
- **CAPS** — connectionist attractor settling resolved a regulation strategy
- **PREDICTOR** — personality weights pinned the strategy
- **SHAME_CONTEXT** — surface-level shame override
- **LLM** — no layer pinned; the LLM reasoned from situation context

Every episode also records its effectiveness score — how well the chosen strategy actually worked for that agent in that situation. The data has been accumulating. What was missing was something to read it.

![Regulation cascade with calibration analysis](svg/2026-11-17-regulation-cascade-with-calibration.svg)

## What the Calibration Analyzer Does

`PredictionCalibrationAnalyzer` takes the accumulated source-grouped effectiveness data for a single agent and answers one question: is the predictor helping or hurting?

The mechanics are straightforward. Load all regulation episodes for an agent. Group by source. Compute mean effectiveness per source. Compare PREDICTOR against LLM. If the LLM consistently outperforms the predictor by more than 0.15 — a margin chosen because a 15% effectiveness gap represents a meaningful difference in regulation quality — the agent is flagged as potentially miscalibrated.

The report includes everything needed to act on the finding: episode counts per source, mean effectiveness per source, the raw delta, and an inconclusive flag when sample sizes are too small to draw conclusions. Twenty episodes per source is the default minimum — enough to smooth out the variance inherent in LLM-judged effectiveness scores.

## Why This Matters

Population-level personality models are useful starting points, not ground truths. Barańczuk's meta-analysis aggregates across hundreds of studies and thousands of participants. The correlations are real — conscientiousness does predict greater use of cognitive reappraisal, neuroticism does predict greater use of avoidant strategies. But within those population distributions, individual variation is substantial. An agent whose backstory includes formative experiences that cut against their personality type — a highly neurotic character who learned to plan under pressure through repeated exposure — may regulate better when the LLM reasons about their specific history than when the predictor applies the population weight.

This is the fundamental tension in any cognitive architecture that combines personality-grounded prediction with situational LLM reasoning. The predictor is fast, consistent, and personality-coherent. The LLM is slower, more variable, and situationally aware. Neither is universally better. The right answer is knowing which one to trust for each agent — and that requires measurement.

The calibration report is the first step toward adaptive confidence. Right now it flags. The next step — which belongs in the dynamic regulatory capacity work (#851) — is acting on the flag: reducing predictor confidence for agents where it consistently underperforms, letting more decisions fall through to the LLM. Not abandoning the predictor entirely — a miscalibrated predictor for one emotion category might still be accurate for others. The measurement needs to be granular.

## The Broader Pattern

Every time a cognitive pipeline stage makes a decision, it should be possible to ask: was that decision better than what the alternative would have produced? Source tracking makes the question answerable. Calibration analysis makes it automatic.

This applies beyond regulation. Drive modulation, attention allocation, coping strategy selection — anywhere a mechanical model competes with LLM reasoning, the same infrastructure could compare effectiveness per source. The pattern is: tag the decision maker, record the outcome, compare the populations.

The regulation predictor was the right place to start because it has the clearest binary: either the predictor pins a strategy, or the LLM picks one. The comparison is clean. Other pipeline stages will have more complex attribution — CAPS settling influences rather than pins, and the LLM sees CAPS outputs in its prompt even when CAPS doesn't dominate. Those are harder measurement problems. But the architecture is in place.
