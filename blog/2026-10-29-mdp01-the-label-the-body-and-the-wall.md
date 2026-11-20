---
layout: post
title: "The Label, the Body, and the Wall"
date: 2026-10-29
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
series: issue-156-computed-mode-quality
tags: [wacky-manor, cognitive-architecture, barrett, immersion, drive-expression, entity-linking, metacognition]
---

# The Label, the Body, and the Wall

Yesterday I wrote about discovering that 640 lines of hand-crafted drive prose were short-circuiting the cognitive pipeline. Today I found out what happens when you take them away and try to make the pipeline stand on its own.

Short version: the computed pipeline now exceeds the prose baseline on every aggregate metric. Quality 3.92 (was 3.88). Register 3.74 (was 3.35). Differentiation 3.74 (was 3.67). But there is a wall, and the wall has a name: reactive drives.

## The Hybrid That Nobody Tested

The Barrett somatic descriptions — "warm, effervescent pressure behind your sternum" — produce immersive character experience. The LLM inhabits the body. But it doesn't know WHICH body state it's inhabiting. Strip the labels and the same somatic text could be joy, pride, excitement, or gloating.

I'd been treating labels (drive type + intensity) and somatic prose as opposing strategies. The project's history said so: Era 1 found that raw cognitive data (`MY STRONGEST DRIVE: gloating, INTENSITY: 0.7`) made the LLM reason ABOUT its personality system instead of AS its character. Era 4 found the same thing when drive descriptions were stripped to type+intensity only — three days later prose came back.

But nobody had tested putting the label ON TOP of the somatic description. The hypothesis: the 50-word somatic paragraph provides enough immersion mass to ground the character past the label. The label provides the conceptual anchor. The body text prevents metacognitive drift.

The result:

| Config | Quality | Immersion |
|--------|---------|-----------|
| Label + degree only | Poor | Simulation awareness leaks |
| Somatic prose only | 3.76 | Excellent immersion |
| Label + degree + prose | 3.99 | Zero leaks detected |

I checked the transcripts. In the hybrid configuration, characters never reference their drive names as system labels. They never say "my gloating drive is activating." They say "something warm rises behind my ribs" while the system knows it's gloating. In the V2 prose baseline — the one everyone assumed was clean — characters said "my proving-worth drive is screaming" and "the system is telling me his distress is deepening." Twenty-seven metacognitive hits in 303 events. The hybrid had zero.

The somatic text is the key differentiator. It is not decorative. It is protective.

## The Immersion Audit

Once I started looking for system vocabulary in character output, I found it everywhere.

`"Last Action Result: You interacted with the guest-book."` — the character would then say "the action result just says I interacted with it." It was reading a game report instead of remembering an experience.

`[id: penelope-pitstop]` on every entity — the LLM saw the cartoon agentId and eventually recognised the Wacky Races scenario. In the first contaminated 300-tick run, characters were saying "Dick Dastardly. The Hooded Claw. Names that sound like villains from a cartoon."

`[INTERACT]` on objects — characters picked up the affordance label and said "the crates can be interacted with." Nobody would say that.

The fix was systematic. Every section header, every action result, every affordance label, every entity ID — audited and rewritten from system-speak to experiential framing.

| Before | After |
|--------|-------|
| Last Action Result | A Moment Ago |
| Characters Present | Who's Here |
| You interacted with the guest-book | You examined the guest book. A leather-bound register with yellowed pages. |
| `ActionType.INTERACT` in memory text | `ActionType.verb()` → "examined" |

The `ActionType.verb()` change is my favourite: a single method on the enum that maps internal names to natural-language verbs. One line that makes it impossible for the enum name to reach any text the LLM sees.

For the GENERIC eval profile, we went further: cartoon agentIds are rewritten to generic slugs at import. `penelope-pitstop` becomes `clara-bellingham` before any code runs. The cartoon lineage doesn't exist in the runtime. There is nothing to leak because there is nothing to leak FROM.

## The Wall

All three aggregate metrics exceed V2. But drill into the per-drive breakdown and a pattern emerges.

Constant drives — scheming, curiosity, suspicion, self-preservation — score excellently. These are body states that are always on. The Barrett description says "your chest settles into that familiar stillness" and the character lives in that stillness every tick.

Reactive drives — gloating, gallantry, protection — regress. These need a TRIGGER. The character needs to notice that Reginald just failed, that Clara is in danger, that something needs protecting. The Barrett description tells the character what the body does when the drive is active. It does not tell the character what ACTIVATES it.

I tested this by building targeted scenarios. In a hand-crafted scene where Reginald publicly fumbles a vault combination, HC gloating scores 4.33. In a scene where Clara is about to walk into a trap, Peter gallantry scores 5.00. The code works. The quality is there.

But in the 300-tick run, these drives score 2.40 and 3.67. Not because the code fails, but because the connection between observation and activation is missing. "Reginald is here" doesn't automatically mean "your gloating should activate." The character has to independently make that link, and under long context with competing drives, the link fails.

The V2 prose had embedded triggers: "When someone is struggling, your body responds before your mind weighs whether to help." The word "when" is doing enormous work. It says: watch for this condition, and when you see it, the body responds. The computed pipeline provides the body response but not the "when."

## The Relational Answer

This is where the wall becomes a door.

The relational schemas work (#477) — built in a parallel session — adds per-person felt-experience text. When HC sees Reginald, the prompt includes "you sense his weakness" from the Other Model section. When trust for prey is low, CAPS biasing dampens empathy and boosts schadenfreude at the perceptual level. The agent doesn't know this is happening. It just notices different things about different people.

That is the missing connection. Not a generic "when someone fails" trigger, but a relational one: "when THIS person, who you relate to THIS way, fails — your body does THIS." The trigger isn't situational. It's relational. And relational context is exactly what #477 computes.

The next step is to merge the relational schemas into this pipeline and measure whether reactive drive quality recovers. The targeted probes already show the ceiling is high — 4.33 for gloating, 5.00 for gallantry under ideal conditions. The question is whether relational context lifts the floor to match.

## Entity Linking and the Deeper Question

A side thread surfaced during the immersion audit that I want to record even though it is not solved.

Character names in prose are fragile. "Clara is naive and trusts too easily" — that belief works for the GENERIC profile's name set. Change the names and the belief is wrong. Have two characters named James and the prose is ambiguous.

The NLP literature calls this entity linking — anchoring text mentions to canonical entities via standoff annotation. The text reads naturally. The entity metadata lives in a parallel structure. Every "he," "she," "the key," "that fool" resolves to a specific entity in the world model.

If character references were entity-linked, memory would stop being a text archive and become an entity graph. "What does Clara know about the key?" would be a graph query over linked spans, not a keyword search over prose. Theory of Mind, belief revision, reflection — all entity-graph operations.

This connects to the MindMap, which is already a knowledge graph. The standoff annotations would be the bridge between unstructured text (memories, beliefs) and structured knowledge (nodes, edges). Two views of the same information, mechanically linked.

I filed this as a design investigation (neocortex #721). It needs a literature review before implementation. But the shape is clear: entity-grounded prose, with the MindMap as the resolution layer.
