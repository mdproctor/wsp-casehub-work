---
layout: post
title: "Why Decay Is Wrong for Memory Consolidation Triggers"
date: 2026-10-22
entry_type: article
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
tags: [cognitive-architecture, memory-consolidation, CUSUM, CLS-theory, CAPS, literature-review]
series: issue-650-elastic-session-scheduler
---

# Why Decay Is Wrong for Memory Consolidation Triggers

I spent a day tuning cognitive parameters for our multi-agent scenario — mood decay, session pool sizing, accumulator thresholds — and the tuning worked. Characters showed real mood differentiation: the Ant-Hill-Mob's pleasure went negative while the Hooded Claw's dominance collapsed from +0.95 to -0.32 as his disguise failed. But one thing didn't move at all: character drives. Across 50 ticks with 5 characters, zero drive adaptation. The accumulator was accumulating, but drives never changed.

The accumulator uses CUSUM (Cumulative Sum) with temporal decay. Each tick, the accumulated signal decays by 15%. For a character slowly becoming more suspicious over 20 ticks — one small signal per tick — the accumulation rate can sit below the decay rate. The signal never crosses the threshold. Consolidation never fires. The personality change is silently lost.

## What Was There and Why

The `DegreeAccumulator` was built as a dual-rate CUSUM: a fast channel for sudden behavioural shifts and a slow channel for gradual drift. Both channels have an allowance (noise filter), a threshold (firing point), and a decay factor. The fast channel decays at 0.85 per tick, the slow channel more gently.

CUSUM is a well-established statistical technique for detecting shifts in a process — it comes from industrial quality control, where you're monitoring a production line for deviations from a target value. The allowance filters noise, the threshold controls the false alarm rate, and decay makes old signals less relevant than recent ones. For detecting anomalies in a stationary process, this is exactly right.

The problem is that consolidation scheduling is not anomaly detection. We borrowed CUSUM because it looked like the right shape — "accumulate signal, fire when it crosses a threshold" — without examining whether its assumptions held. They don't. CUSUM assumes old signals are less relevant than new ones (hence decay). But for consolidation, old un-processed experiences are *more* urgent than recent ones, not less. A character who witnessed a betrayal 15 ticks ago and still hasn't consolidated that experience has a larger behavioural inconsistency than one who saw it 2 ticks ago.

Decay inverts the urgency model.

## What the Literature Says

Three independent lines of work converge on the same answer: consolidation triggers should be monotonic. No decay.

### Park et al. (2023) — Generative Agents

The Stanford "Smallville" paper used monotonic importance accumulation for reflection (their form of consolidation). Each observation gets an importance score from 1 to 10. When cumulative importance exceeds roughly 150, reflection fires. No decay — every experience counts until processed. Their threshold of ~150 importance points works out to roughly 15-20 interactions before reflection fires.

This is the simplest version of the right idea: count un-processed experiences, fire when there are enough.

### SCM: Sleep-Consolidated Memory (Shinde, 2026)

The most specific work on consolidation trigger design I found. SCM defines four triggers — consolidation fires when *any* is met:

- **Memory entropy > 0.9** — when the importance distribution becomes too uniform (too many equally important memories create decision paralysis)
- **Conflict density > 0.3** — when contradictions accumulate in the memory graph
- **Time elapsed > 1 hour** — hard ceiling
- **Manual forcing**

All three automatic triggers are monotonically increasing between consolidation events. None use decay. The time-elapsed trigger is the safety net — entropy or conflict usually fire sooner.

### Complementary Learning Systems (McClelland et al. 1995; Kumaran et al. 2016)

The foundational theory. The hippocampus does fast, one-shot learning; the neocortex does slow, gradual integration. Transfer between them happens via interleaved replay — hippocampal traces replay to the neocortex, interleaved with other experiences, to prevent catastrophic forgetting.

The 2016 update from Kumaran, Hassabis and McClelland is the key finding for our problem: replay frequency should be proportional to the *divergence* between new experience and existing structure. The more novel the experience — the more it contradicts the consolidated model — the more urgent the integration. This is the opposite of decay, which treats old signals as less important regardless of their divergence.

## What It Caused

In a 50-tick scenario run with 5 characters, the dual-rate accumulator with decay produced zero drive adaptation. Character drives remained at their seed values throughout — social-harmony=0.8, scheming=0.9, loyalty=0.9, whatever was configured at startup. The characters had rich mood dynamics (the mood system was tuned separately and worked well) but their underlying personality profiles were frozen.

The mechanism: with decay=0.85, each tick erodes 15% of accumulated signal. For a character accumulating one interaction per tick at an increment below the allowance threshold (0.15), the steady-state signal is roughly 6.7 — which can sit below the firing threshold indefinitely. Slow-building personality changes are permanently suppressed.

This is exactly the case CLS theory warns about. Gradual drift — a character who slowly becomes more suspicious, or gradually builds trust — produces small, consistent signals. Decay treats each one as less important than the last. The accumulator asymptotes below the threshold and consolidation never fires.

## What We'll Try Next

Replace the decay-based accumulator for consolidation scheduling with three monotonic signals:

**Event count** — the Park et al. baseline. Monotonic count of un-processed experiences per agent. Every experience counts until consolidated. Fire at threshold N (targeting 5-10 interactions based on the Smallville evaluation's finding that users notice behavioural inconsistency within 3-5 interactions).

**Divergence accumulation** — the CLS 2016 insight. Weight each experience by how much it diverges from the current consolidated personality model. When the appraisal strategy produces a result that contradicts the consolidated self-model, that experience contributes more to the accumulator. A character who witnesses a betrayal accumulates urgency faster than one having routine social interactions. Monotonic — divergence doesn't decay.

**Time ceiling** — the existing `manor.consolidation.interval-ticks` property. Unchanged. This is SCM's safety net — ensures consolidation fires even when event count and divergence stay low.

The `DegreeAccumulator` itself isn't wrong — it stays valid for CAPS situation classification, where detecting shifts in a stationary behavioural pattern is exactly the right model. The fix is using the right mechanism for the right job. CUSUM for classification, monotonic accumulation for consolidation.
