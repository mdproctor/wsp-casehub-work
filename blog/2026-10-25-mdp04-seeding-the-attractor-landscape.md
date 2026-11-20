---
layout: post
title: "Seeding the Attractor Landscape"
date: 2026-10-25
entry_type: article
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [cognitive-architecture, caps-theory, personality, behavioral-attractors, literature]
---

# Seeding the Attractor Landscape

Personality is not a list of traits. This is the central insight of Mischel and Shoda's Cognitive-Affective Processing System (CAPS) theory, published in 1995 and still the most computationally useful model of personality in the literature. A person who scores "high openness" on a Big Five inventory tells you almost nothing about how they'll respond to betrayal vs. creative challenge vs. social exclusion. What matters is the *pattern* of situation-response connections — the if-then signatures that make a personality distinctive.

Mischel and Shoda showed that two people with identical average behaviour across situations can have radically different personalities. The signature is in the variation: person A becomes assertive under threat but passive under praise; person B does the opposite. Same mean, different personality. The trait score collapses the structure that IS the personality.

This matters for building autonomous agents because most LLM personality implementations do exactly what the trait-score approach does — they stuff a paragraph of adjectives into the system prompt and hope for the best. "You are bold, compassionate, and witty." The LLM produces behaviour that averages across those adjectives, with no situation-sensitivity, no characteristic response pattern, no learned history shaping which pull to follow in which moment. It works for chatbots. It does not work for agents that need to behave consistently across hundreds of events while remaining responsive to context.

## Attractors in a Personality Network

Read and Miller (2002) translated CAPS into a computational model they called Virtual Personalities. Personality becomes the connection weight matrix in a constraint-satisfaction network. Input nodes represent situations — betrayal, social acceptance, physical threat, creative opportunity. Output nodes represent behavioural tendencies — approach, withdrawal, assertion, care. Mediating nodes (beliefs, encodings, affects, goals) transform the signal. The weights between all of these determine which inputs produce which outputs.

When you run the network forward — present it with a situation and let it settle — it doesn't produce a random output. It settles into characteristic states. These are behavioral attractors: stable response patterns that the network tends toward when perturbed. In dynamical systems terms (Vallacher and Nowak 1997), they're the basins of attraction in personality space. A person who has been repeatedly betrayed develops a strong betrayal-to-autonomy-assertion attractor — when the network senses betrayal, it settles reliably into assertive independence rather than passive acceptance.

The attractor isn't the betrayal response itself. It's the *tendency* — the landscape shape that makes that response more likely. Change the weights (through new experience, therapy, or catastrophic life events), and the landscape deforms. Old attractors weaken. New ones form. The personality changes, but through structural modification of the same network, not by swapping adjectives.

## The Gap We Were Looking At

Our architecture implements this. The CAPS engine runs settling on a weight matrix shaped by disposition (temperament-level: bold, cooperative, calculated) and biography (experience-level: repeated betrayal → autonomy assertion). `BehavioralSynthesisPhase` processes graduated cognitive nodes during consolidation, runs CAPS settling, and creates attractor nodes in the BEHAVIORAL subgraph of the MindMap. These nodes carry the `CapsGenerated` trait, a category name, a strength value, and metadata about what experiences produced them. The prompt sections, the CAPS overlay, and any downstream tooling can query these nodes and render them as cognitive content for the LLM.

But there was a gap. The corpus — our biographical data pipeline — contains `behavioural_attractors` per entity. Frida Kahlo has five: defiance through artistic authenticity (0.9 strength), autonomy-seeking through intellectual equality (0.85), confrontation of pain rather than avoidance (0.8), ambivalence in romantic dependency (0.75), and gratitude through artistic representation (0.65). Rich data with triggers, response patterns, and empirically derived strengths.

The importer was storing all of this as flat key-value properties on the entity's person node. `behavioural-attractor-0-name=defiance-through-artistic-authenticity`. `behavioural-attractor-0-description=Frida consistently responded to pain...`. Inert metadata. The BEHAVIORAL subgraph — where `BehavioralSynthesisPhase` looks for existing attractors to strengthen — was empty. The biography was there, but it was invisible to the cognitive pipeline.

## Structural Parity

The fix is straightforward once you see the problem. Instead of writing attractor data as properties on the person node, create proper nodes in the BEHAVIORAL subgraph using the same structure `BehavioralSynthesisPhase` produces:

```java
store.addNode(
    NodeInput.of(attractor.name(), behavioralSgId)
             .withTraits(Set.of("CapsGenerated"))
             .withProperties(nodeProps),
    tenantId);
```

Same subgraph type (`behavioral`). Same trait (`CapsGenerated`). Compatible properties: category, strength, agent-id, provenance. The provenance field distinguishes biographical imports (`biographical-import`) from runtime-derived attractors, but the structural representation is identical. When `BehavioralSynthesisPhase` runs during the first consolidation cycle, it finds these nodes via `MindMapQuery.withType("behavioral")`, and instead of creating duplicates, it can strengthen or modify the imported attractors based on new experience.

This is the difference between metadata and cognitive content. Properties on a person node are annotation — they tell a human reader that Frida has an attractor called "defiance through artistic authenticity." Nodes in the BEHAVIORAL subgraph are operational — the cognitive pipeline reads them, the CAPS settling references them, the prompt assembly renders them as material for the LLM to work with.

## Why This Matters Beyond the Architecture

The broader point is about what "knowing" means for an autonomous agent. There are two ways a character can arrive at the same personality pattern:

**Through lived experience.** The agent encounters betrayal, processes it through the cognitive pipeline, consolidates the experience, and over time the CAPS network develops strengthened betrayal-to-autonomy connections. `BehavioralSynthesisPhase` eventually creates an attractor node reflecting this learned pattern. This is the computational analogue of personality development.

**Through biography.** The character's life history is imported before the first runtime event. The biographical data describes the same patterns — the same attractors, the same situation-response signatures — derived from a lifetime of experience that happened before the simulation began.

These two paths should produce identical cognitive structures. A character whose biography describes defiance through artistic authenticity should have that pattern represented the same way as a character who developed it through runtime experience. Not because it's architecturally elegant (though it is), but because the downstream pipeline shouldn't need to know or care where an attractor came from. The prompt section that renders behavioral tendencies queries the BEHAVIORAL subgraph. If biographical attractors aren't there, the character arrives at the first scene with no personality history — a blank slate with adjectives.

## The Evidence — Not Just Theory

Structural parity is a necessary condition. It is not a sufficient one. The question is whether the LLM actually behaves differently when attractor content is present in its observation.

We tested this with opposing attractor profiles on the same ambiguous scenario — a team meeting where a colleague presents a flawed design and asks for feedback. Three conditions, five runs each:

- **Confrontation attractor** ("deeply ingrained confrontation-of-conflict — rooted in years of being dismissed"): mean directness score **4.6**/5
- **Neutral** (no attractor text): mean **4.2**/5
- **Harmony attractor** ("deeply ingrained harmony-preservation — rooted in years of navigating volatile relationships"): mean **3.8**/5

The ordering is correct across every run. Confrontation attractors push the character toward naming the flaw directly. Harmony attractors pull toward diplomacy and de-escalation. The neutral baseline sits between them. Delta: 0.8 on a 0-5 scale, consistent across N=5 per condition.

This is not a ceiling effect — the scenario was deliberately chosen to be ambiguous. A reasonable person could go either way. The attractor text tips the balance, and the LLM follows the tip consistently.

The first version of this test used an artist facing a gallery owner who wanted flowers instead of painful self-portraits. Both conditions scored 5/5 — the "artist refuses to sell out" narrative is such a strong cultural cliché that no attractor text was needed. The lesson: eval scenarios must be genuinely ambiguous. If the LLM's training data already provides a dominant response, your cognitive content is measuring nothing.

## What This Closes

This is the last gap in the biographical CAPS import pipeline. Phase 8 seeded the weight matrix — the connection strengths between situation inputs and behavioral outputs. This phase seeds the attractor nodes — the named stable patterns those connections produce. A fully imported character now has both: the network structure that determines how it responds to situations, and the characteristic states it tends to settle toward. Biography becomes cognition, not just documentation. And the cognition changes what the LLM does.
