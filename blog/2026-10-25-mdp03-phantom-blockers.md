---
layout: post
title: "Phantom Blockers"
date: 2026-10-25
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
tags: [cognitive-architecture, project-hygiene, drive-expression]
---

# Phantom Blockers

I wanted to do a scenario run today. Full 300+ event GENERIC evaluation with the cognitive pipeline firing — channel weighting, drive expression, inner life, the lot. So I asked the obvious question: what's still blocking us?

The answer looked bad. Eight open issues across neocortex and examples. Three marked critical. The drive expression epic (#122) still open with unchecked items. The cognitive pipeline tuning issue (#102) with six sub-items. InnerLifeOrchestrator not wired into production. The picture said: you're not close.

The picture was wrong.

We started verifying. `InnerLifeOrchestrator.observe()` — the issue said zero production callers. But there it was in `PlaybookOrchestrator` at line 624, wired months ago. The fix was on main. The issue was never closed. `DriveProfileDerivationPhase` producing zero profiles — issue #584, still open. Commits on main: `19d62f3e`, `345d4c3b`. Fixed. Never closed. The somatic-only generation prompt — #585, still open. The code already has all four channels with Barrett, Easterbrook, and Frijda citations in the comments. Fixed. Never closed.

Six of the eight "blockers" were phantoms. The code had moved on. The issue tracker hadn't.

This is a specific failure mode: when implementation happens across sessions and repos, the close ceremony gets interrupted or forgotten. The fix lands on main via a cross-repo branch. The branch gets stamped. But the issue in the other repo stays open because nobody went back to tick the box. Over time, those open issues accumulate into a wall of apparent debt that doesn't exist.

The actual work this session was small but real. I wired `LlmSituationClassifier` into the `CapsProducer` CDI bean — one file change, replacing the rule-based no-op with actual LLM classification so channel weighting activates at runtime. And we went through every drive description across both social-config files, adding Frijda action tendencies where they were missing. Nine drives updated. The pattern was consistent: drives that only described what the character *feels* (jaw tightness, stomach clenching, restlessness) produced generic LLM behaviour. Drives that described what the character *does* — eyes scanning exits, hands closing into fists, feet moving before judgment — produced distinctive behaviour. Frijda was right: emotions without action readiness are psychologically incomplete, and computationally incomplete too.

We also fixed the gallantry/proving-worth conceptual bleed in the GENERIC profile. Gallantry had "the feeling needs an audience the way a fire needs air" — that's proving-worth territory. Gallantry is compulsive protective aid regardless of witness. The audience-seeking belongs to proving-worth. Clean separation matters because the CAPS channel weighting treats them differently.

After the housekeeping and the Frijda work, I created epic #149 — pre-run cognitive readiness — with the genuine remaining items. Three real blockers: behavioural attractor import (#686), attribute format mismatch (#688), and CAPS settling dampening (#111). Five quality items. One run validation. That's the actual distance to a full cognitive scenario run.

The gap between code landed and issues closed is a trust problem. If I'd taken the issue tracker at face value, I'd have estimated weeks of work before a run. The real answer is days. The lesson isn't "close your issues" — that's obvious. The lesson is that when implementation is distributed across repos and sessions, verification against the actual codebase is the only reliable source of truth. The tracker is an approximation. `git log --grep` is a fact.
