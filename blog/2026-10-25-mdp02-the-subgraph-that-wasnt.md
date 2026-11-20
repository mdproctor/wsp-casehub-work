---
layout: post
title: "The Subgraph That Wasn't"
date: 2026-10-25
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [memory-seeding, corpus-import, behavioral-attractors, caps, biography]
---

# The Subgraph That Wasn't

Earlier today I wrote that behavioural attractors "had their import code."
That was true in a narrow sense. `EntityEnrichmentHandler` processed
`behaviouralAttractors` from the corpus. It didn't skip them. It stored
them as properties on the entity's PERSON node — `behavioural-attractor-0-name`,
`behavioural-attractor-0-description`, flat key-value pairs indexed by
position. Import complete. Data present. And completely invisible to
everything downstream.

The problem is that `BehavioralPromptSection` — the component that renders
attractor data into the LLM's observation — doesn't look at entity node
properties. It queries the BEHAVIORAL subgraph for nodes with the
`CapsGenerated` trait. That's what `BehavioralSynthesisPhase` creates at
runtime when the CAPS engine settles during consolidation: proper MindMap
nodes in a dedicated subgraph, with category, strength, agent-id, and
source provenance. The import path was writing to a different address than
the runtime path. Both worked. Neither knew about the other.

This is a pattern worth naming. The data model was correct — `BehaviouralAttractor`
deserialized from YAML without complaint. The storage code ran without error.
But the storage target was wrong. The enrichment handler followed the same
pattern it used for drive baselines and CAPS patterns: resolve the entity node,
attach properties. For those two types, that's correct — they decorate the
entity. Behavioural attractors are different. They're first-class cognitive
entities in their own subgraph, not metadata on a person.

The fix follows the runtime path exactly. `BiographyUtils.ensureSubgraph()`
creates the BEHAVIORAL subgraph if it doesn't exist.
Each attractor becomes a `NodeInput` with `CapsGenerated` trait and
`biographical-import` provenance. The properties match what
`BehavioralPromptSection` queries for: category, agent-id, strength,
description.

The converter was losing data too. The corpus carries five fields per
attractor — name, description, trigger, response, strength — but
`convert_corpus.py` was keeping only name and description. The
`BehaviouralAttractor` Java record matched: two fields, nothing more.
Three fields of biographical detail, silently discarded at the Python
boundary. Extending both — the converter to pass all five, the record to
receive them — means the imported nodes now carry trigger context and
strength values that `BehavioralSynthesisPhase` can use when it
encounters an existing biographical attractor during runtime
consolidation.

What's different between biographical and runtime attractors is
provenance, not structure. A biographical attractor says "this character
historically responded to physical suffering by painting visceral
detail, at strength 0.9." A runtime attractor says "across the last
twenty consolidation cycles, the CAPS engine settled on a
defiance-through-creativity pattern with EMA strength 0.73." Same
subgraph. Same trait. Same query path. Different origin stories. When
both exist, the runtime path can reinforce or revise the biographical
baseline — the weight matrix doesn't care where the initial signal came
from.
