---
layout: post
title: "Why Every Drive Looked the Same (and What Fixed It)"
date: 2026-10-16
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
tags: [caps, drive-expression, appraisal-filtering, cognitive-architecture, profile-derivation]
series: issue-115-drive-expression-gap
---

# Why Every Drive Looked the Same (and What Fixed It)

Gallantry rendered when PP was alone in a workshop repairing a mechanism. No social partner present, no warmth, no courtesy — pure solo mechanical task. Gallantry should not activate here. But it did, every time.

The investigation took a full day and went through five distinct fix attempts before landing on the right one. Each attempt taught something about where the real problem lived.

## Getting to reproduction

The first obstacle wasn't the bug itself — it was model routing. The `LlmSituationClassifier` called `withModel("haiku")` to use a fast model for JSON classification, but the string `"haiku"` never reached the CLI. Two bugs stacked: the `RoutingAgentProvider` rejected `"haiku"` because no model with that ID existed in the seed catalog (it's registered as `claude-haiku-4-5`), and even if resolution succeeded, `ClaudeAgentClient` silently dropped the resolved model ID when building CLI options.

We fixed both — model passthrough in the client, unrecognized-string fallback in the router — and could finally run the pipeline test. The `DriveProfilePipelineTest` seeded all 17 characters, derived profiles via LLM, classified five PP scenarios, and checked which drives rendered. It took twenty minutes and timed out intermittently during profile derivation.

The first optimisation: seed only the characters the test actually uses. One character instead of seventeen. Seven minutes instead of twenty. Still slow, but reproducible.

## The social context gate — first fix

The obvious fix: gallantry is a social drive. It needs a social partner to render. I added a `context: social` field to `SocialConfig.Drive`, marked gallantry as social in the YAML, and gated it in the appraisal filter — if `nearbyAgentIds` is empty and the drive is social, filter it.

This worked. PP-alone-workshop passed. But the fix was human priming. I'd told the system which drives were social instead of letting it figure that out from the drive's own data. Every new drive would need a manual classification decision.

## Relative overlap — second fix

The deeper issue was the overlap check. The filter compared LLM-derived drive profiles against situation activations. If ANY single input node overlapped, the drive rendered. With profiles containing ten or fifteen generic nodes — `mastery`, `competence_recognition`, `agency_granted`, `reward` — everything overlapped with everything.

The fix: relative overlap. Instead of `overlap > 0`, use `overlap / profileTotal >= 0.15`. The situation must activate a meaningful fraction of the drive's profile pattern, not just any one shared node. This worked for the unit tests, but the pipeline test remained fragile — different LLM runs produced different profiles, and the profiles were always too broad.

## Kitchen-sink profiles — the root cause

The derived profiles for gallantry looked like this:

```
reward:0.850, acceptance:0.750, physical_threat:0.850, performance_judged:0.650,
competence_recognition:0.850, social_threat:0.450, agency_granted:0.600
```

Seven nodes spanning five CAPS categories: reward, social, threat, achievement, power. Every situation touches at least one of these. The profile doesn't describe gallantry — it describes "a memory that involved helping someone," which is so generic it matches workshops, danger, puzzles, and social warmth equally.

The derivation was asking the wrong question. "What situation features does this memory evoke?" produces a comprehensive inventory of everything the memory touches. A memory about helping a girl at school touches acceptance (warmth), reward (satisfaction), competence (fixing things), physical_threat (danger), mastery (skill). All true. All useless for discrimination.

## Authored triggers — the wrong direction

I tried replacing LLM-derived profiles with hand-authored trigger lists in the YAML — explicit CAPS input nodes that each drive responds to. Gallantry triggers on `acceptance`, `sharing`, `secure_attachment`. Scheming triggers on `dominance`, `agency_granted`, `competition`.

The mechanism worked. The unit tests passed. But the design was wrong. Authoring triggers is scripting behaviour, not emergence. The whole point of the cognitive architecture is that drives emerge from the character's psychology — from memories and descriptions — not from explicit rules. Every new drive would need manual trigger authoring. The system would be as expressive as its author's imagination, not the LLM's capacity to discover patterns.

## The compositional answer

The fix isn't a single mechanism. It's three layers, each handling a different aspect of the problem.

**Discriminative derivation.** The classifier sees each memory alongside the agent's OTHER drives. Instead of "what features does this memory contain?", the question becomes "what features are distinctive to this drive, compared to the agent's other drives?" The metadata tells the classifier: *this memory serves gallantry; the agent also has protection and proving-worth; return only features that would NOT equally activate those other drives.*

The result: gallantry gets `acceptance:0.800, sharing:0.700, secure_attachment:0.750, reward:0.500` — four nodes concentrated in the social/relationship category. Proving-worth gets `mastery:0.900, competence_recognition:0.850, performance_judged:0.800, failure:0.700` — four nodes in the achievement category. No overlap between the two profiles.

**Relative overlap threshold.** With narrow profiles, the 0.15 threshold works cleanly. Gallantry's four social nodes produce 0.000 overlap with workshop activations (mastery, competence_recognition). Proving-worth's four achievement nodes produce 0.492 overlap. The filter doesn't need to know which drives are "social" — the profile structure carries that information.

**Derive and pin.** Derivation runs once per character definition. The result is stored as a `DriveProfileInferred` node in the cognitive subgraph. Runtime uses the pinned profile — no LLM calls for filtering, no non-determinism between test runs. Re-derive when memories or descriptions change.

## What the numbers look like

From the 13-scenario cross-matrix test (three characters, solo and social environments, competing signals):

| Scenario | gallantry | proving-worth | scheming | self-preservation |
|---|---|---|---|---|
| PP alone workshop | **0.000** | 0.492 ✓ | — | — |
| PP alone crash | **0.000** | 0.000 | — | — |
| PP with Clara warmth | **0.582** ✓ | 0.154 ✓ | — | — |
| HC alone plotting | — | — | **0.600** ✓ | 0.000 |
| HC alone danger | — | — | 0.000 | **0.881** ✓ |
| HC trap but heard | — | — | 0.133 | **0.373** ✓ |

Gallantry at 0.000 in solo scenarios — not because a social gate blocked it, but because its profile nodes don't exist in workshop or danger activations. The discrimination happened at derivation time, not at filter time.

The entire cross-matrix runs in 0.2 seconds. No LLM, no Quarkus boot. Fixed profiles, mock classifiers, deterministic assertions. The pipeline test with real LLM classification adds ~5 minutes — one LLM call per scenario, no profile derivation at runtime.

## Pipeline validation — the gap that remains

The discriminative prompt improved the LLM output. Gallantry's profile narrowed from seven nodes to five, and `acceptance:0.800` became the top node — the right signal in the right place. But `physical_threat:0.850` and `competence_recognition:0.750` survived. The LLM understood the intent but didn't fully constrain its output. Four of eight pipeline scenarios passed; four failed because generic nodes still leaked through.

The unit test matrix with hand-curated discriminative profiles passes 13/13. The pipeline test with LLM-derived profiles passes 4/8. The gap is prompt engineering: the discrimination metadata nudges the LLM in the right direction but doesn't force the output to match the structure the filter needs. Constraining the output count ("at most 4-5 nodes") or providing explicit contrast examples ("acceptance is distinctive to gallantry; competence_recognition is not, because proving-worth also responds to it") are the next iterations.

This is the derive-and-pin workflow in practice. Run the derivation. Inspect the profiles. Pin the ones that work. Iterate on the prompt for the ones that don't. The fast test matrix validates in 0.2 seconds; the pipeline test validates the full LLM chain when the prompt is ready.

## The question that changed the answer

The difference between a kitchen-sink profile and a discriminative profile is one sentence in the classifier's metadata:

> "Return only the situation features that are distinctive to this drive — features that would NOT equally activate the other drives."

Classification asks "what's here?" Discrimination asks "what distinguishes this from that?" The profiles were broad because the question was broad. Narrow the question, narrow the profile. The LLM was capable of discrimination — it just wasn't being asked. The prompt got it 60% of the way. The remaining 40% is refining how strongly the discrimination constraint is expressed.
