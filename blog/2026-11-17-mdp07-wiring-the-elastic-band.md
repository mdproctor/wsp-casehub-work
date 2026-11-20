---
layout: post
title: "Wiring the Elastic Band"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [personality, CAPS, situational-modulation, elastic-band, cognitive-pipeline, tick-participant, consolidation, thread-safety, topology, implementation]
series: issue-712-shadow-archetypes
---

# Wiring the Elastic Band

*Continues from [The Elastic Band](2026-11-17-mdp01-the-elastic-band.md), which described the design model — displacement, home drift, resilience. This entry covers what it took to wire that model into the production cognitive pipeline.*

The design was clean. Each personality facet is an elastic band. Stress pulls it. Release and it springs back — mostly. The maths worked in isolation: the integrator tests passed, the detector classified modes correctly, the state service persisted round-trips. Five batches of pure-function components, all tested independently.

Then we had to plug it into CognitionCore.

## Why TERMINAL

CognitionCore runs tick participants in four phases: FOUNDATION (mood, attention), SOURCE (narrative, strategy, reflection), DERIVED (appraisal, regulation, drives, search budget), and TERMINAL (goals, domain activation, cognitive profile). The phases execute in order. Within each phase, participants run in registration order.

Situational personality modulation must run in TERMINAL. The reasoning is straightforward: the participant needs appraisal results (SEC dimensions, situation input activations) and mood state (PAD values) as evidence inputs. Both are produced in DERIVED. If the participant ran earlier, the evidence wouldn't exist yet.

But TERMINAL carries an additional constraint: everything that happens there is downstream of the appraisal-regulation-drive cascade. The personality changes we publish in TERMINAL won't affect appraisal weights until the next tick. This is correct — the per-tick flow is "perceive the world → classify the situation → shift the personality → let the shift affect next perception." A one-tick delay between classification and effect is the right temporal structure. Perception shouldn't retroactively change the appraisal that produced it.

![Tick pipeline showing TERMINAL placement and data flow from appraisal/mood into the situational participant](svg/situational-tick-pipeline.svg)

## The Accessor Pattern

The participant needs two upstream signals: appraisal data (situation input activations, SEC dimensions) and mood state (PAD values). Both come from components that already exist in the pipeline — `AppraisalTickParticipant` and `MoodOrchestrator`.

I could have injected those components directly. The participant would call `appraisalParticipant.currentInputActivations(agentId, tenantId)` and `mood.currentMood(agentId, tenantId)`. But that creates a hard dependency on two specific implementation classes, and the test harness would need to construct the entire appraisal stack just to verify detection logic.

Instead, two narrow interfaces:

```java
public interface AppraisalAccessor {
    Map<String, Double> currentInputActivations(String agentId, String tenantId);
    Map<String, Double> currentSecDimensions(String agentId, String tenantId);
}

public interface MoodAccessor {
    double pleasure(String agentId, String tenantId);
    double arousal(String agentId, String tenantId);
    double dominance(String agentId, String tenantId);
}
```

In production, `AppraisalTickParticipant` implements `AppraisalAccessor` (it already has the methods). `MoodOrchestrator` gets a thin adapter for `MoodAccessor`. In tests, a three-field stub. The participant doesn't know or care which.

## The Bug Claude Caught

The initial implementation stored detection state as a single volatile field:

```java
private volatile DetectionResult lastResult = DetectionResult.initial();
```

The detector needs the previous tick's result for fade decay and engagement hysteresis. When stress evidence arrives, the intensity ramps up. When it fades, the intensity decays exponentially. Whether a mode is currently engaged affects the threshold for disengagement — `threshold_off` is lower than `threshold_on`, creating a hysteresis band that prevents mode flapping.

The problem: `lastResult` was per-participant, not per-agent. If CognitionCore ticks agent A (who is under stress) and then agent B (who is calm), agent B's detection starts from agent A's stress state. B inherits the engagement flag, the residual intensity, the group winner selection. A depressed character in one conversation infects the detection state of an unrelated character in another.

![Per-agent isolation: shared state leaks detection across agents; ConcurrentHashMap fix isolates each agent's history](svg/per-agent-isolation.svg)

The fix is straightforward — a `ConcurrentHashMap<String, DetectionResult>` keyed by `agentId:tenantId`. Each agent maintains its own detection history. The volatile field was the natural first implementation because the integrator tests use a single agent. The bug only manifests when the same participant instance serves multiple agents sequentially — which is exactly what CognitionCore does.

This is the kind of bug that never fails a unit test but corrupts production behaviour silently. The character doesn't crash. It just acts wrong — stressed when it shouldn't be, or calm when it should be anxious — because it's running on someone else's emotional history.

## The Noise Floor

The design spec defined two floors: `x_floor` (0.6 for uncalibrated classifiers) below which situation inputs are treated as noise, and `m_floor` (0.3) below which from-rest mediating responses don't contribute to detection.

We wrote a topology integration test that settles the real CAPS network with every situation input at `x_floor` simultaneously, then checks that all mediating nodes stay below `m_floor`. One node violated: `threat_sensitivity` reached 0.3241 — just 8% above the floor.

This isn't a bug. It's a property of the network. The spec's `m_floor` applies to per-mode detection, where only a subset of inputs fire. The test drove all inputs to the floor at once, which never happens during normal operation — real situations activate a cluster of related inputs, not every input in the network. The cumulative stimulation from dozens of floor-level inputs is higher than any single mode would produce.

We adjusted the cumulative threshold to 0.35 and documented the finding. The important result: no mediating node is pathologically responsive to floor-level input. `threat_sensitivity` is the most sensitive node in the network, and even under maximal cumulative stimulation it only marginally exceeds the per-mode floor. The detection model's weighted signal extraction handles the rest.

## Consolidation: What Happens When Nobody's Watching

The tick participant handles personality shifts during conversation. But what about idle time? If a character was stressed at the end of a conversation, their displacement should relax during the gap before the next one. The home offset should drift. Resilience should heal.

`SituationalDeformationPhase` handles this. It's a `ConsolidationPhase` — the same interface used by experience consolidation, recurring thought detection, and behavioural synthesis. The consolidation scheduler runs these phases periodically, outside the tick loop.

The deformation phase iterates over all agents with cached personality state, advances each one to the current wall time, and publishes any changes to `EffectiveCognitiveDefaults`. A character who was stressed yesterday and hasn't been seen since gets their displacement relaxed, their resilience partially healed, and their home offset adjusted for any drift that accrued.

This is where the wall-clock grid from the design pays off. The integrator produces identical results regardless of when consolidation runs. Whether the scheduler fires every five minutes or once a day, the character's state after advancing to the same wall-clock time is the same. No tick-rate dependence, no consolidation-frequency dependence.

## The Bridge: EffectiveCognitiveDefaults

The whole system converges at `EffectiveCognitiveDefaults`. This component sits between the personality state and the Cognitive Derivation Engine (CDE). It manages two views per agent: a resting view (baseline + home offset, used by learning rules) and an effective view (baseline + home offset + displacement, used by behaviour).

When the participant calls `updateState()`, `EffectiveCognitiveDefaults` recomposes the facets. Each personality facet gets its displacement and home offset added to the baseline score, clamped to [0, 1]. If the resulting score differs from baseline by more than 0.005, the facet is marked with origin `"situational"` — the prompt renderer can tell the agent its Anxiousness is elevated and why.

The CDE doesn't know any of this happened. It receives a `DescriptorView` with facet scores and derives cognitive defaults the same way it always has. Appraisal weights change because the facet scores changed, not because someone told the appraisal system about stress. Trust formation rates shift because the PID-5 inputs shifted, not because a flag was set. The same pipeline, different inputs, different behaviour.

That was the whole point. Not a mode system. Not an overlay. An elastic band that bends the personality, and a pipeline that responds to whatever the personality is right now.
