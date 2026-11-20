---
layout: post
title: "The Body Doesn't Listen When You Talk About People"
date: 2026-10-29
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
series: era5-relational-cognition
tags: [wacky-manor, cognitive-architecture, barrett, attention-competition, relational-cognition, prompt-engineering]
---

# The Body Doesn't Listen When You Talk About People

Yesterday's entry ended with a wall. The computed cognitive pipeline beat
the hand-authored prose baseline on every aggregate metric — quality 3.92,
register 3.74 — but reactive drives wouldn't budge. Gloating, gallantry,
protection: drives that activate TOWARD specific people in specific
situations. The pipeline gave characters somatic awareness and drive labels.
It didn't give them a target.

The relational schema system was supposed to fix this. Five layers of
relationship-aware cognition: role labels, trust scores, empathic sensing,
CAPS biasing, dynamic evolution. Each character would know who in the room
was prey, who was ally, who was threat. The body would respond to the
person, not just the situation.

I expected the reactive drives to recover. Register stayed at 3.74 or
climbed. The numbers would close the gap between constant drives (scheming
at 5.00) and reactive drives (gloating at 2.40). That was the hypothesis.

## What Actually Happened

Register dropped from 3.74 to 3.13. Not noise — 70 evaluated events across
105 ticks, N=14 per character. The relational context didn't just fail to
help. It actively interfered with the somatic expression that was already
working.

The prompt had grown. Where Era 4 had one directive — feel your body, here
are your drives — Era 5 had six:

1. Feel your body (Barrett 4-channel instruction)
2. Here are your drives (somatic labels)
3. Here's what you think about these people (relational schemas)
4. Here's how you bond (attachment style)
5. Here's what you sense about them (empathic model)
6. Here's how trust modulates your responses (CAPS biasing)

Each section was well-crafted. Each was independently justified by
literature. Together they created a cognitive interrupt pattern that broke
the somatic flow.

The Barrett instruction says "notice your body." Two sections later, the
relational block says "here's what you think about Reginald: prey, trust
0.1, predictable." The character tries to feel its body AND analyse its
relationships simultaneously. The analysis wins. Register collapses.

## Attention Is Finite and Competitive

This is the finding I didn't expect. Adding information to an LLM prompt
doesn't just add — it takes away. Every token competes for attention during
generation. A section about relationship analysis steals attention from the
somatic instruction, even when both sections are high quality.

I've seen this pattern before in rule engines. Adding more rules doesn't
make the system smarter — past a threshold, rules compete and the system
starts thrashing. The same principle applies to prompt engineering. The
prompt is an attention budget. Spend it on one clear directive and the
output is focused. Split it across six directives and the output is
muddled.

The register numbers tell the story. Era 4's prompt had one directive:
feel. Register 3.74. Era 5's prompt had six directives: feel, analyse
people, recall attachment, sense empathically, modulate by trust. Register
3.13. More information, worse output.

## Less Is More, But Less Doesn't Mean Minimal

The relational context isn't wrong. The body DOES respond to specific
people. In real embodied cognition, you don't feel anxiety in the abstract
— you feel anxiety toward someone. The tightness in your chest isn't
generic; it's tightness-toward-Reginald.

The mistake was treating relational context as a SEPARATE section. A
cognitive analysis block sitting alongside a somatic instruction. Two
competing frames in one prompt.

The fix is architectural. Don't tell the character what it thinks about
people. Point the somatic sensation at the person:

```
GLOATING (0.7) → Reginald: A warm, effervescent pressure rises
behind your sternum...
```

Three characters: `→ Name`. The somatic description stays the same. It just
knows where it's pointed. No separate section. No cognitive interruption.
The drive already carries the relational context — not as analysis, but as
direction.

This mirrors how the body actually works. The body doesn't have a
"relationship analysis module" that feeds into a separate "somatic response
module." The relationship IS the somatic response. Tension toward one
person, settling near another. The body doesn't reason about trust — it
responds to the person with trust already baked into the response.

## What This Opens

The targeted somatic approach means the relational context folded into
#477 — roles, trust, empathic sensing — can still matter. But it has to
enter the prompt as somatic content, not cognitive content. The
CognitiveDerivationEngine already generates per-drive somatic descriptions.
Extending it to generate per-relationship somatic modulations is the next
step: "when prey is near, the warmth behind your sternum sharpens."

Consolidation produces these offline. Render-time attaches them to the
right person. The prompt stays the same length as Era 4. The relational
context is present but invisible — embedded in the somatic texture, not
competing with it.

The broader principle: when working with LLMs, more context is not always
better context. The right context, in the right structure, at the right
point in the attention flow. The body doesn't listen when you talk about
people. It listens when you point it at them.
