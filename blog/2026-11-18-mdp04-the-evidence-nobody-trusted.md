---
layout: post
title: "The Evidence Nobody Trusted"
date: 2026-11-18
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cognition, theory-of-mind, bayesian-inference, personality, evidence-weighting, suspiciousness, attributional-bias, fenigstein, combs]
series: cognitive-pipeline
---

# The Evidence Nobody Trusted

Two entries ago, we gave the Theory of Mind engine personality. Anxious agents perceive more threat from the same situation. Callous agents shrug off avoidance signals that would alarm an empathic observer. Both modulations work at the same level: they adjust the Bayesian network's conditional probability tables before inference begins. The *structure* of perception changes, but the *evidence* — what the agent actually observes — enters unfiltered.

That's the gap. A suspicious agent and a trusting agent see the same cooperative behaviour and feed the same evidence vector into the same (possibly identical) network. Suspiciousness should change how the evidence is interpreted, not how the network is wired.

## Why evidence weighting is a different mechanism

The distinction matters. In the Bayesian ToM engine, there are two layers where personality can intervene:

**Layer 1: Network construction (CPTs).** Anxiousness shifts the threat prior — before any evidence arrives, an anxious agent already assigns higher probability to threat. Callousness weakens the distress→avoidant coupling in the behaviour|emotion CPT — a callous agent's network encodes a weaker causal link between distress and avoidance. Both are structural: they define the agent's interpretive framework before the first observation.

**Layer 2: Evidence presentation.** The soft evidence vectors — probability distributions over threat levels and behaviour types — are computed by `TheoryOfMindBridge` from SEC appraisal dimensions and action tendencies. These vectors are what the junction tree actually conditions on. Scaling them before they enter the network changes what the agent "sees" rather than how the network reasons.

Suspiciousness is a Layer 2 phenomenon. The literature is clear on this. Fenigstein and Vanable's 1992 work on paranoia and interpretive bias found that suspicious individuals don't have different causal models of the world — they discount positive signals and amplify negative ones. Combs et al. (2007) demonstrated that this attributional bias operates on the same perceptual input as non-suspicious individuals. The same cooperative gesture gets weighted differently. The network that processes it can be identical.

![Three personality dimensions, two modulation layers — anxiousness and callousness modify CPTs, suspiciousness scales evidence vectors](svg/suspiciousness-evidence-weighting.svg)

## The scaling formula

The implementation is in `TheoryOfMindBridge.infer()`, between computing the behaviour distribution and passing it to the engine:

```
behaviorEvidence = behaviorDistribution(actionTendencies)
if suspiciousness > 0:
    transfer = approach × suspiciousness × 0.6
    approach -= transfer
    avoidant += transfer
    renormalize
```

The factor 0.6 caps the maximum transfer at 60% of the approach weight. At suspiciousness = 1.0 (maximum), a fully cooperative signal `{approach: 0.60, neutral: 0.20, avoidant: 0.20}` becomes `{0.24, 0.20, 0.56}`. The agent still registers some cooperative intent — paranoia distorts, it doesn't blind — but the balance tips sharply toward avoidant. At suspiciousness = 0.0, the evidence passes through unchanged. The guard `> 0.0` ensures zero overhead for the common case.

A property that falls out of the maths: the sum is preserved. Transferring weight from approach to avoidant doesn't change the total, so renormalisation is technically a no-op when the input already sums to 1.0. We keep it for defensive safety — if future changes to `behaviorDistribution` produce slightly unnormalised output, the scaling won't amplify the error.

## Why approach→avoidant and not approach→neutral?

A suspicious agent doesn't simply fail to register cooperation — they actively reinterpret it as potential hostility. "They're being nice to me" becomes "what do they want?" becomes a slight avoidant lean. The transfer goes directly to the hostile interpretation, not to uncertainty. This matches the attributional bias literature: suspicious individuals show a *hostile attribution bias*, not a general ambiguity bias. They don't become uncertain; they become certain about the wrong thing.

## Thread safety, in passing

The session's other fix was smaller: `CacheEvictionScheduler` switched from `ArrayList` to `CopyOnWriteArrayList` for its evictor registry (#1131). The access pattern — 2–3 registrations at startup, iteration every 60+ ticks — is the textbook COWAL case. Registration is rare and cheap; iteration is frequent and must be snapshot-safe. The fix prevents `ConcurrentModificationException` if `setEffectiveDefaults()` is called after ticks have started. The current wiring makes this impossible (everything configures before the first tick), but the public API contract should be safe regardless of call order.

## Three dimensions, one engine

The Theory of Mind engine now responds to three personality dimensions, each at a different level of the inference pipeline:

| Dimension | Layer | Mechanism | Literature |
|-----------|-------|-----------|-----------|
| Anxiousness | CPT (prior) | Shifts threat prior toward elevated threat | Bar-Haim et al. 2007 |
| Callousness | CPT (likelihood) | Weakens distress→avoidant coupling | Blair 2005, 2013 |
| Suspiciousness | Evidence | Transfers approach weight to avoidant | Fenigstein & Vanable 1992; Combs et al. 2007 |

The layering isn't accidental. Each dimension modulates a different aspect of the generative model: anxiousness changes what you expect, callousness changes what you think behaviour means, suspiciousness changes what you think you're seeing. An agent high on all three perceives more threat, reads less distress from avoidance, and discounts cooperative signals — a profile that coherently models a suspicious, callous, anxious observer whose Theory of Mind is systemically skewed toward hostile attribution.

The next gaps are known. The `tomFacetScore` default of 0.5 for missing facets is wrong for callousness and suspiciousness — both should default to 0.0 to match NEUTRAL (#1133). And `CognitionObservationBuilder` still bypasses personality modulation entirely, running all character observations through an unmodulated engine (#1134). Both are filed and queued.
