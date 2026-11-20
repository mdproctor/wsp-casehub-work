---
layout: post
title: "Closing the Rebase Debt"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [rebase, technical-debt, LearningProfile, appraisal-weights, PersonalityMetadataEnricher, type-safety, consolidation, test-coverage, observability]
series: issue-712-shadow-archetypes
---

# Closing the Rebase Debt

*Continues from [The Rebase Nobody Asked For](2026-11-17-mdp08-the-rebase-nobody-asked-for.md), which described resolving semantic merge conflicts between #712 and canonical main. That entry explicitly deferred two items: the missing appraisal weight enrichment, and the orphaned `PersonalityMetadataEnricher`. This entry closes both.*

The rebase entry ended with a known gap. The resolved `addPersonalityMetadata` methods enriched disposition axes and PID-5 facets through `LearningProfile`, but appraisal weights were missing. I'd kept the type boundary — `LearningProfile` didn't expose `appraisalWeights()` — and deferred the fix rather than break a constraint in a conflict resolution commit.

The question was whether the type boundary was a wall or a door.

## The Appraisal Weight Gap

Situation classifier prompts use personality metadata to contextualise CAPS topology nodes. Before the rebase, the canonical `PersonalityMetadataEnricher` enriched three dimensions: disposition axes (`disposition.socialOrient`, etc.), PID-5 facet scores (`pid5.Anxiousness=0.72`), and appraisal weights (`appraisal.urgencyWeight`, `appraisal.fearOnsetThreshold`, etc.). After the rebase, the two consolidation phases — `BehavioralSynthesisPhase` and `DriveProfileDerivationPhase` — had inline enrichment covering the first two but not the third.

The gap matters because appraisal weights shape how an agent interprets situations. An agent with a high `fearOnsetThreshold` processes threat cues differently than one with a low threshold. Without these weights in the classifier prompt, the CAPS situation nodes see a personality with no emotional texture — disposition and facets describe *who* someone is, but appraisal weights describe *how they react*.

## LearningProfile as a Door

`LearningProfile` is a deliberate wrapper around `CognitiveDefaults`. It exposes the resting personality state — what consolidation phases should use for learning and classification — and hides the effective state, which includes situational displacement. The boundary exists so that memory classification is anchored to baseline personality, not reactive to momentary emotional state.

Appraisal weights are part of the resting personality. They're derived from emotion defaults, not from situational context. Adding `appraisalWeights()` to `LearningProfile` preserves the resting-only contract because the data *is* resting state — it just hadn't been exposed.

One line:

```java
public AppraisalWeights appraisalWeights() { return delegate.appraisalWeights(); }
```

The type boundary turned out to be extensible. The constraint it enforces — resting, not effective — was never about restricting which resting properties are visible. It was about which *kind* of state reaches consolidation phases.

## Consolidation: Reviving the Orphan

With `LearningProfile` now exposing appraisal weights, the orphaned `PersonalityMetadataEnricher` had a path back to life. The original utility took `CognitiveDefaultsRegistry` — the old per-agent lookup. I added a `LearningProfile` overload that accepts the profile directly and does all three enrichment dimensions in one place:

```java
public static Map<String, String> enrich(Map<String, String> metadata,
                                          LearningProfile profile,
                                          Set<String> allowedFacets) { ... }
```

The existing `CognitiveDefaultsRegistry` overload now delegates through `LearningProfile.from(defaults.get())`. Both consolidation phases collapse from 25 lines of inline enrichment to a three-line delegation. The duplicate code is gone and the enricher has production callers again.

![Before and after: inline duplication replaced by shared enricher](svg/2026-11-17-enrichment-consolidation.svg)

A side effect worth noting: the old `CognitiveDefaultsRegistry` overload had an early return when `descriptor() == null`, which skipped *all* enrichment — including appraisal weights that come from `emotionDefaults`, not `descriptor`. The new code through `LearningProfile` handles each dimension independently. If an agent has no descriptor but does have appraisal weights, those weights now reach the classifier prompt. The old behaviour was overly restrictive; the new behaviour is correct.

## The Other Two

Two smaller items closed the remaining branch debt.

`SituationalStateService` had a silently swallowed `IllegalArgumentException` when deserialising persisted mode names. If a `SituationalMode` enum value was removed in a code update, old persisted state referencing it would vanish with no trace. The forward-compatibility guard is correct — dropping unknown values is the right default — but invisible data loss makes debugging a guessing game. A `LOG.fine()` with the unrecognised mode name gives you the trail without cluttering production logs.

`MentalModelPromptSection` had no test coverage for its posterior rendering logic. The 60% confidence threshold — below which Bayesian ToM posteriors are suppressed from the prompt — was live code with no verification. Four tests now cover the boundary: above threshold (rendered with correct format), at threshold (not rendered — the check is strict `>`), below threshold (not rendered), and null posterior (no section). The tests exercise the full public API path through a mocked orchestrator, not just the private rendering method.

## The Pattern

Rebase conflict resolution creates a specific kind of technical debt: items you *understand* but deliberately defer because they don't belong in a merge commit. The rebase entry identified the appraisal weight gap and the orphaned enricher as follow-ups. This session closed them.

The turnaround matters. Deferred rebase debt that sits for weeks becomes mysterious — the context that made the deferral obvious is gone, and the next person to find the gap has to reconstruct the reasoning from scratch. Close it in the next session while the design decisions are still fresh. The fix was straightforward precisely because the reasoning hadn't decaye
