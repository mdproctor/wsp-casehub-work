---
layout: post
title: "Wiring the Coping Layer"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [coping-strategies, CAPS, personality, cognitive-architecture, Connor-Smith-Flachsbart, Rescorla-Wagner, topology, settling, literature-grounded]
series: issue-595-coping-strategies
---

# Wiring the Coping Layer

*Continues the coping pipeline work from [The Record That Knew Too Much](2026-11-17-mdp01-the-record-that-knew-too-much.md). That entry covered the CognitiveDefaults decomposition that made coping tendencies a first-class record. This entry covers what happened when those tendencies actually reached the CAPS engine.*

The coping output nodes have been sitting in the CAPS topology since the topology work a few weeks ago. Sixteen of them — planning, avoidance, positive reinterpretation, seek emotional support, the full Marsella and Gratch taxonomy. Structurally present, functionally inert. No connections feeding activation into them from the mediating layer. No personality-derived resting activations biasing them toward one character's coping style over another's.

A CAPS settling run would fire, activation would flow from input nodes through the mediating layer, and then stop. The regulation outputs lit up. The approach-avoidance outputs lit up. The coping outputs sat at zero. The architecture had the right shape but no current flowing through the coping region.

Two things needed to happen to make it work.

## Personality Seeds the Starting Position

The first problem: even before any situation-specific activation arrives, a character's personality should bias which coping strategies they reach for. Someone high in conscientiousness should have planning as a more accessible attractor. Someone high in neuroticism should have avoidance closer to the surface.

Connor-Smith and Flachsbart's 2007 meta-analysis gives us the mapping. They analysed 124 studies relating Big Five personality traits to coping strategy use. The CopingDefaults record — the one that emerged from last session's decomposition work — already carries the derived tendencies: planful, reinterpretation, avoidance, support-seeking, acceptance. Each is a continuous value between 0 and 1, derived from the PID-5 personality facets via the derivation engine.

The question was how to turn these into CAPS resting activations. The existing regulation and approach-avoidance outputs use values in the 0.1 to 0.3 range, applied conditionally from categorical disposition axes. Coping tendencies are continuous, so the approach is different.

I settled on centering at the population mean:

```
bias = (tendency − 0.5) × 0.5
```

A tendency of 0.5 — the midpoint — produces zero bias. The character has no dispositional lean toward or away from that strategy. A tendency of 0.8 (high conscientiousness → high planful) gives +0.15 resting activation on the planning node. A tendency of 0.2 (low neuroticism → low avoidance) gives −0.15 on the avoidance node — actively suppressing it below baseline.

The scale factor of 0.5 keeps the output range at [−0.25, +0.25], which sits comfortably within the existing activation magnitudes. The regulation nodes use 0.1 to 0.3 for their resting activations. The coping seeds land in the same neighbourhood without dominating the settling dynamics.

The mapping:

| CopingDefaults field | Topology node |
|---|---|
| planfulTendency | `planning` |
| reinterpretationTendency | `positive_reinterpretation` |
| avoidanceTendency | `avoidance` |
| supportSeekingTendency | `seek_instrumental_support` + `seek_emotional_support` |
| acceptanceTendency | `coping_acceptance` |

Support-seeking maps to both instrumental and emotional support nodes at equal weight. The literature doesn't distinguish the personality-based tendency between the two — the distinction is situational, not dispositional.

## Connections Make Activation Flow

Resting activations set the starting position, but without connections from the mediating layer, coping nodes can't respond to what's happening in the situation. A character with high threat sensitivity should see their avoidance attractor light up when danger appears — that requires a connection from `threat_sensitivity` to `avoidance`, not just a resting activation bias.

![Coping CAPS settling architecture](svg/coping-caps-settling.svg)

We added 18 connections from mediating nodes to coping outputs. The design follows the same Connor-Smith and Flachsbart mapping, now expressed through the CAPS mediating layer rather than directly from personality traits:

**Problem-focused coping.** `self_efficacy` feeds both `planning` (0.40) and `active_confrontation` (0.35). Agency — believing you can affect the outcome — is the prerequisite for problem-focused strategies. `BAS_activation` contributes to both as well, at lower weight. Approach motivation supports engagement with stressors.

**Avoidant coping.** `BIS_activation` is the primary driver — feeding `avoidance` (0.40), `mental_disengagement` (0.35), and `suppression` (0.30). `threat_sensitivity` adds a parallel pathway to avoidance and disengagement. And `self_efficacy` feeds avoidance with an inhibitory connection (−0.25): high agency suppresses avoidant coping. The idea is that someone who believes they can handle the situation is less likely to disengage from it.

**Support-seeking.** `other_safety` enables emotional support seeking — you don't disclose to someone you don't feel safe with. `other_reliability` enables instrumental support seeking — you don't ask for practical help from someone you can't rely on. `BAS_activation` adds a weaker connection to instrumental seeking, reflecting approach motivation for active help-seeking.

**Cognitive coping.** `self_efficacy` feeds `positive_reinterpretation` (0.40) — reframing a situation requires the cognitive resources that agency reflects. `self_worth` feeds both reinterpretation (0.25) and acceptance (0.30). And `BIS_activation` inhibits acceptance (−0.25): anxiety makes it harder to accept what you can't change.

## What's Still Inert

Seven coping output nodes have neither connections nor personality seeding: `denial`, `wishful_thinking`, `procrastination`, `resignation`, `seek_information`, `suppress_information`, `shift_responsibility`. These are the secondary strategies in each coping dimension — the ones the CopingStrategyPredictor resolves through its cognitive pipeline rather than through CAPS settling.

The predictor evaluates situational controllability, cognitive load, and self-conscious emotion to pick within a dimension. If CAPS settling activates the avoidance region, the predictor decides whether that means denial, mental disengagement, or procrastination based on the specifics. The CAPS topology doesn't need to distinguish between them — it just needs to light up the right region.

This is the same design pattern as the regulation layer: CAPS settles on the broad strategy family, the predictor refines within it. The topology handles "how strong is the pull toward avoidant coping?" The predictor handles "which specific avoidant strategy fits this situation?"

## What This Opens Up

Layer 2 coping settling is now functional end-to-end. Input activation flows through the mediating layer — BIS picks up threat, self-efficacy reflects agency, other-model nodes encode trust — and the coping output nodes respond with personality-shaped, situation-sensitive attractor strengths. The Rescorla-Wagner weight updates that run after each tick can now learn from coping outcomes, shifting attractor weights based on experience.

A character who repeatedly uses planning successfully in controllable situations will strengthen the self_efficacy → planning connection over time. A character who avoids threats that turn out to be manageable will gradually see the avoidance → planning shift that Connor-Smith and Flachsbart observe in longitudinal studies. The personality seeds provide the starting point. The topology connections provide the activation pathway. Learning provides the adaptation.

The coping layer went from architecturally present to computationally live in one session. The next piece is the topology connections between coping outputs and the existing regulation layer — a coping strategy like `positive_reinterpretation` should feed into `cognitive_change` as its regulation implementation. That bridge closes the gap between "what I'm trying to do about this situation" and "how I'm managing my emotional response to it."
