---
layout: post
title: "Moving the Eval Tests Home"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [eval-migration, cognition, testing, observation-rendering]
series: issue-702-move-cognition-eval-tests
---

# Moving the Eval Tests Home

The previous entry described extracting `CognitionObservationBuilder` from `CharacterCognition` — the generic cognitive rendering logic freed from manor-specific types. That extraction existed to unblock this: moving all ten CharacterCognition-dependent eval tests from wacky-manor to neocortex's `caps-llm-classifier` module.

The tests evaluate neocortex capabilities — drive discrimination, attractor steering, personality emergence, priority signature effects — but they lived in wacky-manor because they needed `CharacterCognition` to assemble observations. With the builder extracted, the question became: what else do these tests need that's still manor-locked?

The answer was more than I expected. The simple tests (CognitiveInfluenceEvalTest, EmergentBehaviorEvalTest) don't use the mindmap at all — they call `renderCognitiveSections()` which produces social-config-based sections: tendencies, beliefs, norms, personality facets, formation memories, substrate, attachment style, relational schemas. The builder handles the cognitive sections (drives, attractors, differentiation), but these social sections came directly from `SocialConfig` — a manor type loaded from manor's YAML files.

I needed a neocortex-native equivalent. `EvalCharacterData` expanded from two fields (drives, formation memories) to nine, picking up every social dimension that `renderCognitiveSections()` renders. The YAML grew to match — five characters with full psychological profiles, all from the GENERIC profile that strips pop-culture references. Then `CognitiveTestObservation`: a test helper that assembles the same section ordering as `CharacterCognition.renderCognitiveSections()` — Lazarus appraisal flow, tendencies first, drives in the middle, attachment last — but from `EvalCharacterData` instead of `SocialConfig`.

The LLM eval tests also needed `LlmTestSupport` (which renders character system prompts from the eidos agent registry) and `AgentResponse` (which parses structured character responses). Both are manor types. `EvalLlmJudge` and `EvalAgentResponse` replaced them — thinner versions that use `EvalTestSupport`'s standalone `AgentProvider` instead of CDI injection.

With the infrastructure in place, the tests fell into three groups. The simple ones (CognitiveInfluenceEvalTest, EmergentBehaviorEvalTest, DriveExpressionEvalTest, RelationalModelEvalTest) migrated mechanically — swap `ManorSocialConfigLoader` for `EvalCharacterLoader`, swap `CharacterCognition` for `CognitiveTestObservation`, swap `LlmTestSupport` for `EvalLlmJudge`, drop `@QuarkusTest`.

The mindmap-aware tests (FocusedDriveEvalTest, PrioritySignatureReproducerTest, DriveProfilePipelineTest) needed more. They seed `InMemoryMindMapStore` with drives, formation memories, profiles, and differentiation texts, then run consolidation phases before rendering. `EvalMindMapSeeder` extracted the seeding pattern. The builder accepts the store and classifier at construction, and `CognitiveTestObservation.render()` takes an optional builder parameter — when present, it delegates cognitive rendering to the builder; when absent, it renders drive descriptions directly from the data.

The CDI tests (PropagationBoundaryTest, PipelineOutputTest) were the hardest call. They inject `MindMapStore`, `ConsolidationScheduler`, `ExperienceRecorder`, and `CapsEngine` — real CDI beans in a real Quarkus container. The original tests validate that the CDI wiring produces the right results end-to-end. Moving them means either constructing the full pipeline manually or accepting that we're testing something slightly different.

I went with manual construction. `InMemoryCapsEngine` (already in neocortex's `caps-testing` module) replaces the CDI-injected engine. For consolidation-dependent tests, the phases that need a classifier bypass `BehavioralSynthesisPhase` (package-private constructor) by seeding attractor nodes directly with the right traits. It tests the rendering pipeline rather than the full consolidation pipeline — a narrower claim, but the one these tests were actually designed to verify.

All ten tests compile and the deterministic ones pass. The `caps-llm-classifier` module is now a self-contained eval harness — drive discrimination, personality emergence, attractor steering, priority signatures, relational models, all testable without a single manor import.
