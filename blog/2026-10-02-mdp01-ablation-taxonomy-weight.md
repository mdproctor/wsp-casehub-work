---
title: "The Ablation That Worked — Proving the Taxonomy Carries the Weight"
date: 2026-10-02
entry_type: note
subtype: diary
projects: [casehubio/examples]
author: mdp
tags: [taxonomy, emergence, ablation, character-design, wacky-manor, llm]
---

# The Ablation That Worked — Proving the Taxonomy Carries the Weight

I had a nagging problem with the directive-minimal comparison runs. Both
transcripts looked great — characters behaved distinctively, voices were
strong, the stripped-down briefings produced equivalent behavior to the
verbose originals. But there was an obvious confound: Claude knows Wacky
Races. Say "You are Dick Dastardly" and the model brings scheming,
catchphrases, and a magnificent moustache from its own training data. The
comparison proved the taxonomy didn't *break* anything, but couldn't prove
it *contributed* anything.

The fix was an ablation test. Same taxonomy structure — same agentIds,
dispositions, constraints, drives, tendencies, social-config — but
different character names with no pop-culture footprint. Clara Bellingham
instead of Penelope Pitstop (warm Southern socialite). Vincent Marsh
instead of the Hooded Claw (theatrical English villain with a dual
persona). The Brixton Boys instead of the Ant Hill Mob (East London lads
instead of Brooklyn gangsters). Reginald Foxworth, James Hartwell.
Characters the model has never seen, where the YAML is the only source
of behavioral guidance.

The result was clear: the behavioral delta between Wacky Races and
generic characters was small. All five generics exhibited the same
patterns as their pop-culture counterparts. Marsh maintained his
Pemberton facade and schemed in private asides. The Brixton Boys
positioned protectively around Clara and voiced suspicions they
couldn't articulate. Foxworth lied about everything with theatrical
conviction. The taxonomy is load-bearing.

The Brixton Boys were the strongest evidence. The Ant Hill Mob speaks
Brooklyn gangster — "Hey boss, dat don't look right!" The Brixton Boys
speak East London — "Blimey, lads!", "Right dusty old gaff, innit?",
"legged it like headless chickens." Completely different surface
language, identical protective behaviour. The voice section drove the
accent; the tendencies and drives drove the behaviour. Neither came
from model priors, because the model has no priors about the Brixton
Boys.

A second insight fell out of the experiment. I'd been prescribing
specific catchphrases in the voice section — "Oh my goodness!", "Bless
your heart!" for Clara. Claude used them, but it felt constrained.
Replacing the phrase list with a style directive — "lean heavily into
Southern colloquialisms and regional idioms" — produced richer output.
"HUSH my mouth" is a deeper Southernism than anything I would have
prescribed. Foxworth started inventing credentials — the "Berkshire
Treasure Hunting Society", the "Foxworth Pressure Technique" — which
was funnier and more authentically upper-class-bluffer than my scripted
"Ha, SPLENDID!" The principle: prescribe structure, not content. Tell
the model where to fish, not which fish to catch.

One exception: the theatrical villain. Vincent Marsh's voice was
competent but didn't spontaneously generate a distinctive evil laugh.
Non-regional archetypes have less cultural material for the model to
draw from. For those, keep one or two prescribed anchors — a laugh, a
verbal tic — and let the style directive handle the rest.

This changes how I think about the character YAML. The briefing earns
its place as identity — one or two sentences. The voice section earns
its place as an accent anchor plus a directive to go deep. Tendencies,
drives, constraints, and norms carry the actual behaviour. Prescribed
catchphrases are optional anchors for characters whose archetype isn't
regionally grounded. Everything else the model can generate better than
I can prescribe it.
