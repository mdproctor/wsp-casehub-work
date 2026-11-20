---
layout: post
title: "The Wiring Problem — When Your Cognitive Architecture Works but Your Game Doesn't Know It"
date: 2026-09-20
entry_type: article
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, wiring, agent-design, wacky-manor, observation-pipeline]
series: issue-77-cognitive-wiring-audit
---

# The Wiring Problem — When Your Cognitive Architecture Works but Your Game Doesn't Know It

I ran a full-stack audit of the cognitive architecture across three repos and found something I should have caught months ago: every cognitive subsystem worked — drive adaptation, needs pyramid, belief revision, relationship stages, goal formation — and none of it reached the characters.

The game had two rendering pipelines. Pipeline A was `CognitionCore.promptSections()`, the platform-level system that returns `PromptSection` objects for mood, drives, narrative, strategy, goals, character motivations, and needs. Pipeline B was `CharacterCognition.renderCognitiveSections()`, wacky-manor's own rendering that returns `ObservationSection` objects for beliefs, norms, social awareness, and trust. The game used Pipeline B. All of Phase C went into Pipeline A.

The result: Hooded Claw's scheming drive could decay from 0.9 to 0.1 through repeated negative outcomes — the `DriveAdaptationPhase` computed this correctly, wrote the adapted intensity to MindMap nodes — and the character kept scheming at full intensity. The social awareness gate read from static YAML config, not from the adapted nodes. Drive adaptation existed as running code that changed nothing about behaviour.

## Two Pipelines, One Prompt

The two-pipeline architecture wasn't a bug in the original design. It was pragmatic — wacky-manor needed cognitive rendering before the platform-level infrastructure existed, so it built its own. When Phase C added `CognitionCore` with its orchestrators and prompt sections, the platform pipeline grew alongside the application pipeline without replacing it. Both worked. Neither knew about the other.

Bridging them turned out to be straightforward. `CognitionCore.promptSections()` returns sections that contribute markdown text with `## Header` prefixes. `CharacterCognition` needs `ObservationSection` objects. The adapter is eight lines:

```java
private static ObservationSection adaptPromptSection(String text) {
    if (text.startsWith("## ")) {
        int end = text.indexOf('\n');
        if (end > 0) {
            return ObservationSection.text(
                text.substring(3, end).strip(),
                text.substring(end).strip());
        }
    }
    // fallback: first line as header, rest as content
    int nl = text.indexOf('\n');
    if (nl > 0) {
        return ObservationSection.text(
            text.substring(0, nl).strip(),
            text.substring(nl).strip());
    }
    return ObservationSection.text("Cognitive State", text);
}
```

With this in place, every `PromptSection` contributed by `CognitionCore` — character drives, needs pyramid, goals — flows into the same observation pipeline the game already reads. Characters immediately gained self-awareness of their motivational drives, inner needs, and goals.

## Static Config Is a Lie

The more interesting fix was `shouldCompareSocially()`. This method gates whether a character generates social awareness — comparative perception of nearby characters. It's drive-gated: only characters with scheming or suspicion drives above 0.5 get social awareness. The design decision (D22) was explicit: adversarial behaviours are drive-gated, not familiarity-gated.

The problem: the method read from `SocialConfig.Drive` records loaded from YAML at startup. Drive adaptation writes updated intensities to MindMap nodes. The method never looked at MindMap.

The fix follows the same query pattern that `CharacterDrivePromptSection` already uses — filter the cognitive subgraph for `cognitiveKind: "drive-intensity"` nodes matching the agent, extract the adapted intensity, fall back to static config when no nodes exist. The call site in `renderSocialAwareness` resolves drives before passing them to the gate:

```java
if (!contextStrategy.shouldCompareSocially(resolveAdaptedDrives(), false)) {
    return List.of();
}
```

Now when Hooded Claw's scheming drive decays below 0.5 through consistent negative reinforcement, his social awareness actually turns off. Adaptation affects behaviour, not just rendering.

## The Audit Pattern

I found fifteen findings across the three repos. The distribution was telling: two critical (pipeline disconnection, CDI registration), three high (adapted drives, dead-code behavioural gates, unmerged feature branch), six medium, four low. The critical and high findings were all wiring problems — components that existed and worked but weren't connected to the thing that consumed them.

The low findings were the kind of thing you'd expect in pre-release code: static norms that don't evolve, a missing `@DefaultBean`, priority overlap risk between consolidation phases. The architecture itself was sound. The data model was correct. The seeding, storage, and retrieval patterns were consistent. What was missing was the last mile — the call from the rendering path to the infrastructure.

## What This Opens Up

The cognitive pipeline is now connected end-to-end for drives, needs, beliefs, goals, social awareness, trust, and behavioural gates. Characters can reason about their own motivational state in dialogue. A character whose scheming drive has decayed will behave differently, not just be described differently. Stage-gated behaviours (cooperation, disclosure) are now communicated as behavioural cues in the social awareness section — the LLM receives explicit guidance on what relationship level permits.

The biggest remaining gap is `CognitionCore.tick()`. The platform-level orchestrators for mood, SDT drives, narrative, strategy, mental model, and user model are all wired into `CognitionCore` but `tick()` is never called. These subsystems sit idle. Activating them would give characters emotional state, psychological drive dynamics, personal narrative arcs, and learned strategies — the full inner life the preamble already describes. That's Phase D.
