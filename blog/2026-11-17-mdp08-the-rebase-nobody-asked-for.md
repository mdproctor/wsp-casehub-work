---
layout: post
title: "The Rebase Nobody Asked For"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [rebase, conflict-resolution, personality, PID-5, EffectiveCognitiveDefaults, LearningProfile, semantic-merge, git]
series: issue-712-shadow-archetypes
---

# The Rebase Nobody Asked For

*Continues from [Wiring the Elastic Band](2026-11-17-mdp07-wiring-the-elastic-band.md), which covered the tick pipeline integration. This entry covers landing the branch — specifically, what happens when two parallel development streams modify the same files for different reasons.*

The implementation was done. Ten commits, five batches, 79 tests. The elastic band model — situational personality displacement with exponential return — was wired from config loader to consolidation phase. Time to close the branch.

Then canonical main rejected the rebase. Three files, eleven conflict regions.

## What Happened

While #712 was building the situational personality system, slot 211 landed #1070–1072 on canonical main: a `PersonalityMetadataEnricher` extraction that wired PID-5 personality facets into the CAPS situation classifier. Both streams touched `BehavioralSynthesisPhase` and `DriveProfileDerivationPhase` — the two consolidation phases that classify episodic memories against CAPS topology nodes.

The #712 branch changed these files to migrate from `CognitiveDefaultsRegistry` (the old per-agent lookup) to `EffectiveCognitiveDefaults` (the new tenant-scoped, resting/effective split). The canonical commits changed the same files to extract inline disposition metadata enrichment into a shared `PersonalityMetadataEnricher` utility, adding PID-5 facets and appraisal weights alongside the disposition axes.

Same files, different intentions, incompatible assumptions about which types those files hold.

![Semantic conflict resolution showing how two development intents were combined](svg/rebase-semantic-resolution.svg)

## Why Ours/Theirs Doesn't Work

The temptation with a rebase conflict is to pick a side. `git checkout --ours` or `--theirs` on each file, move on. The problem: neither side is complete.

Take `--theirs` (keep the feature branch) and you lose PID-5 metadata enrichment entirely. The classifier prompts go back to disposition axes only — a functional regression against what canonical already ships. Take `--ours` (keep canonical) and you lose the `EffectiveCognitiveDefaults` migration — the whole point of the commit being replayed.

The correct resolution requires understanding *what each side intended*, not just what each side changed.

## Reading Intent From Code

Canonical's intent was clear from the extraction pattern: `addDispositionMetadata` became `addPersonalityMetadata`, which delegated to `PersonalityMetadataEnricher.enrich()`. The enricher took `CognitiveDefaultsRegistry` and a `Set<String>` PID-5 facet filter. It enriched metadata with three things: disposition axes, PID-5 facet scores, and appraisal weights. The name change — disposition to personality — signalled the broadened scope.

The feature branch's intent was equally clear from the type change: `CognitiveDefaultsRegistry` → `EffectiveCognitiveDefaults`, `forAgent(agentId)` → `restingFor(tenantId, agentId)`. The new type returns a `LearningProfile` — a deliberately constrained wrapper that exposes only the resting personality state, not effective (situationally displaced) values. This is a type-safety boundary: consolidation phases that derive learning parameters should see the resting profile, not the displaced one. Using `LearningProfile` makes it a compile error to accidentally use effective values where resting ones are needed.

Both intents are correct. Both need to survive.

## The Resolution Strategy

The merged result needed four things:

1. **`EffectiveCognitiveDefaults`** as the field type — the feature branch's migration is the whole reason this commit exists.
2. **`pid5FacetSet` and `Pid5FacetsConfig`** — canonical's PID-5 facet filtering, kept as CDI-injected config.
3. **`addPersonalityMetadata`** as the method name — canonical's broader name, since the method now does more than disposition.
4. **`LearningProfile` API** for the method body — the feature branch's tenant-scoped, type-safe access pattern.

The tricky part was number four. `LearningProfile` exposes `.dispositionAxes()` and `.facets()` — enough for disposition and PID-5 enrichment. It does not expose `.appraisalWeights()`. That's deliberate: appraisal weights are derived state that consolidation phases shouldn't consume. The canonical enricher accessed them through `CognitiveDefaults` directly, which had no such restriction.

I kept the type boundary. The resolved method enriches disposition axes and PID-5 facets through `LearningProfile`, and defers appraisal weight enrichment as a follow-up. The canonical `PersonalityMetadataEnricher` utility is now orphaned — no production callers — pending either an `EffectiveCognitiveDefaults` overload or deletion.

## What This Means in Practice

The classification metadata that feeds CAPS situation prompts now includes both disposition axes (`disposition.socialOrient`, etc.) and PID-5 facet scores (`pid5.Impulsivity=0.72`, etc.), filtered by `Pid5FacetsConfig`. It does this through the resting profile, not the effective one — so a character under stress still classifies memories against their baseline personality, not their displaced state. That's the right behaviour: memory classification should be anchored, not reactive.

The missing piece — appraisal weights in classification metadata — is a follow-up. It requires either extending `LearningProfile` or adding an `EffectiveCognitiveDefaults` overload to `PersonalityMetadataEnricher`. Neither is difficult, but neither belongs in a conflict resolution commit.

## The Broader Point

Semantic conflict resolution takes longer than mechanical resolution. For these eleven regions, I had to read both branches' full context: what `PersonalityMetadataEnricher` does, what `LearningProfile` exposes, what `restingFor` returns, where `pid5FacetSet` is initialised across four constructors. That's thirty minutes of reading code I didn't write, understanding design decisions I didn't make.

The alternative — a mechanical merge — would have compiled. It would have passed the existing tests. And it would have silently dropped PID-5 enrichment from classification, or silently broken the type-safety boundary between resting and effective defaults, or left uninitialised final fields in convenience constructors. The kind of bug that surfaces three weeks later when someone notices classifier quality degraded and nobody can explain why.

A rebase is not a merge tool problem. It's a design comprehension problem. The tool shows you *where* the conflict is. Understanding *why* both sides changed the same code — that's the actual work.
