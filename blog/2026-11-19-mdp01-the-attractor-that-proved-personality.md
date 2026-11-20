---
layout: post
title: "The Attractor That Proved Personality"
date: 2026-11-19
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [caps, behavioral-attractors, personality, test-harness, cognition, mischel-shoda, disposition, divergence]
series: cognitive-pipeline
---

# The Attractor That Proved Personality

The cognitive pipeline has four layers of personality differentiation. We proved the first three in earlier sessions: derived coping defaults diverge, search budgets diverge, and prompt rendering diverges between an open and a cautious agent given the same situation. The fourth layer — behavioral attractors — is the one that actually matters. If two personalities settle into identical attractor basins given different experiences, the entire CAPS architecture is decorative.

## What behavioral attractors are

Mischel and Shoda's 1995 CAPS model (Cognitive-Affective Processing System) rejects trait theory's assumption that personality is a bag of static values. Personality is a network: cognitive-affective units — encodings, expectancies, goals, competencies, self-regulatory strategies — connected by activation weights that differ between individuals. When a situation activates the network, activity propagates through these weighted connections until the network settles into a stable pattern. Those stable patterns are attractors.

The same situation fed into two networks with different connection weights will settle into different attractors. That's the whole point. A bold agent and an anxious agent watching the same confrontation should end up with different activated behavioral responses — not because we told them to respond differently, but because their network wiring routes the activation differently.

We built the network (CapsEngine), the topology (120+ nodes across threat, reward, coping, and behavioral output clusters), and the settling algorithm. We built the consolidation phase that processes graduated experience memories through CAPS and writes the resulting attractors into the behavioral subgraph. What we hadn't done was test, in isolation, that the full pipeline actually produces different attractors for different personalities.

## What was missing from the test harness

CognitionTestHarness is the unit-test entry point for the cognitive pipeline. It wires up an isolated CognitionCore with in-memory stores, mock CDI, and no container overhead. Four existing tests prove personality differentiation at earlier pipeline stages:

- `derivedCopingDefaultsDiverge` — a cautious agent has higher avoidance and suppression tendencies
- `searchBudgetsDivergeAfterTick` — search breadth, depth, and convergence differ after a tick
- `promptRenderingDiverges` — the rendered prompt text differs for the same observation
- `curiosityChannelDivergesOnHedonicType` — anxious agents get D-type (deprivation) curiosity, open agents get I-type (interest)

None of these test the final layer: does the CAPS engine actually settle differently, and do the resulting behavioral attractors diverge?

The harness couldn't test this because BehavioralSynthesisPhase needs a CapsEngine (with a loaded topology and per-agent weight initialization), a SituationClassifier, and graduated experience nodes in the MindMap. The harness didn't wire any of this.

## Three failures before the test passed

The first version constructed InMemoryCapsEngine and RuleBasedSituationClassifier in the Builder, pre-initialized CAPS state for each agent using their disposition axes, and passed the BehavioralSynthesisPhase as a consolidation phase. Clean compile, test runs, assertion fails: `capsSettlingResult` returns empty.

The classifier returns empty activations (it's a stub that always produces `List.of()`). Without classifier activations, the only input comes from drive-intensity nodes via `mergeDriveIntensities`. I hadn't seeded any.

Second attempt: added drive-intensity nodes to the cognitive subgraph — curiosity and scheming drives for both agents, same intensity values. Test runs, settling results appear. Both agents produce identical attractors. Same inputs, same topology. The connection weights differ because disposition initialization is personality-specific, but 10 iterations of settling with identical input activations converge to the same basin regardless of initial weight differences.

Third attempt found the real bug. I was initializing CAPS state in the Builder with `"test-tenant"` as the tenant ID, but `consolidate()` runs with whatever tenant the test passes — `"test"`. BehavioralSynthesisPhase.run() calls `capsEngine.loadState("test", "open")`, finds nothing (we stored under `"test-tenant"`), and auto-initializes both agents with identical default axes. All the disposition-specific weight initialization was invisible.

## Lazy initialization and personality-appropriate drives

The fix has two parts. First, CAPS initialization moved from the Builder into `consolidate()` — lazy initialization using the caller's actual tenant ID. The harness stores the CapsEngine and the agentDefaults map; on first consolidation, it checks each agent's state and initializes with their descriptor's disposition axes if missing.

Second, the test gives each agent personality-appropriate drive inputs rather than identical ones. The open agent gets adventure (0.9) and curiosity (0.8) — these map through DriveCompetitionDetector to explore, BAS_activation, and approach nodes. The cautious agent gets suspicion (0.9) and self-preservation (0.8) — mapping to threat_sensitivity, BIS_activation, and FFFS_activation. Different inputs, different weights, different settling.

![CAPS behavioral attractor pipeline — from personality definition through settling to divergent attractors](svg/caps-attractor-pipeline.svg)

The test now proves what the architecture promises: `withBehavioralSynthesis(true)` on the Builder, seed personality-appropriate drives, call `consolidate()`, read `capsSettlingResult()`. The open agent and cautious agent produce measurably different attractor profiles — different categories, different strengths. The network routes activation differently because the wiring is different.

```
var openByCategory = openAttractors.stream()
    .collect(Collectors.toMap(CapsAttractor::category, CapsAttractor::strength, Math::max));
var cautiousByCategory = cautiousAttractors.stream()
    .collect(Collectors.toMap(CapsAttractor::category, CapsAttractor::strength, Math::max));
// regulation, learning, coping, conflict, approach_avoidance — all diverge
```

## The ScopedValue that couldn't

A side-task on the same branch attempted to migrate MutationContext from ThreadLocal to ScopedValue (JEP 506). The design is clean — `ScopedValue.where(CURRENT, source).run(action)` replaces the set/try/finally/clear pattern, with automatic scoping and no cleanup burden. All six usage sites were migrated. All tests rewritten.

Compile fail. `maven.compiler.release` is 21. ScopedValue is finalized in JDK 25. The API exists in the runtime (JDK 26), but the cross-compilation target treats it as a preview feature. Raising the minimum to 25 is a cross-cutting change across all repos. The code was reverted and the issue (#1162) blocked with a comment explaining the prerequisite.

Good engineering design meeting JDK reality. The migration is ready when the platform catches up.

## What this opens up

The harness can now test the full cognitive differentiation pipeline in isolation — no container, no CDI, no external dependencies. Five layers of personality divergence, each testable independently: coping defaults, search budgets, prompt rendering, curiosity channels, and now behavioral attractors.

The next gap is trajectory assertions. The test proves attractors diverge, but doesn't check the settling trajectory — how many iterations to convergence, what convergence type (stable vs oscillation), what the saturation ratio is. Those dimensions could also differentiate personalities and would catch regressions in the settling algorithm.
