---
layout: post
title: "The Mutation Nobody Claimed"
date: 2026-11-18
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [observability, cdi, cognitive-journal, mindmap, architecture]
---

# The Mutation Nobody Claimed

The cognitive journal records every significant event during a tick — appraisals, drive activations, CAPS settling, memory formation. Each entry carries `tenantId`, `agentId`, and an `EventRef` that ties it back to the triggering context. The journal is the audit trail for why an agent behaves the way it does.

Except for graph mutations. The MindMap — the knowledge graph that holds everything an agent knows — fires `GraphMutationRecorded` CDI events whenever nodes or edges change. Those events carried the tenant and the mutation data, but not the agent. A CDI listener observing these events couldn't write a journal entry because it had no idea which agent caused the change.

The `CognitiveEntry.GraphMutated` record type already existed, waiting for someone to construct it. The bridge to get there didn't.

## The two ways to get agentId into a decorator

The `MutationTrackingDecorator` wraps `MindMapStore` and intercepts every mutation. It doesn't know about agents — it operates at the graph layer, below the cognitive pipeline. The agent identity lives several layers up, in the tick orchestrator.

Two options surfaced:

1. **ThreadLocal context propagation** — the cognitive pipeline sets agentId on a ThreadLocal before triggering graph operations. The decorator reads it.
2. **Event enrichment** — add agentId directly to the `GraphMutationRecorded` record so downstream consumers see it on the event itself.

Both are needed, and they solve different halves of the problem. The ThreadLocal gets the data *into* the decorator (it can't take agentId as a method parameter without changing every MindMapStore method signature). The enriched event carries the data *out* to consumers, making each event self-contained.

![Mutation to journal bridge data flow](svg/2026-11-18-mutation-journal-bridge.svg)

This is a clean separation. The ThreadLocal is an implementation detail of how context crosses layer boundaries. The event is the public contract that consumers depend on. If someone replaces the ThreadLocal mechanism later — say, with a CDI request context or a Vert.x context — the event contract doesn't change.

## MutationContext already did half the work

`MutationContext` was already a ThreadLocal carrying the mutation `source` string — `"consolidation:MergeDetectionPhase"`, `"extraction"`, `"conversation-bridge"`. Every caller that mutates the graph was already setting it. Extending it with `setAgent()`/`agentId()` was the natural move:

```java
public static void setAgent(String agentId) { AGENT_ID.set(agentId); }
public static String agentId()              { return AGENT_ID.get(); }
```

And the existing `clear()` now removes both. Every caller already had `MutationContext.clear()` in a `finally` block, so the agentId gets cleaned up for free.

## The bridge itself

`GraphMutatedJournalBridge` is a CDI `@Observes` listener — synchronous, same thread as the event fire. It skips events with null agentId (non-agent context) and uses `Instance<CognitiveJournal>` rather than direct injection, following the same pattern as the other CDI observers in the codebase. If no journal bean is available, it silently does nothing. If the journal throws, the exception is caught rather than propagating back through the CDI event and blowing up the original MindMap operation.

The `EventRef` derivation maps the mutation source string to the journal's event taxonomy: a `"consolidation:*"` source becomes `EventRef.consolidation(...)`, everything else becomes `EventRef.event(source)`. This keeps the CDI event layer decoupled from the cognitive journal's type system — the bridge translates at the boundary.

## What this closes and what it opens

Every MindMap mutation can now be correlated to the agent and event context that caused it. When consolidation runs, it's attributed to `_system`. When a cognitive tick modifies the graph, the agent's identity flows through. The journal becomes a complete record of *why* the knowledge graph looks the way it does, not just what changed.

The mechanism is in place but not yet wired into every entry point. The extraction pipeline and conversation bridge run as async CDI events where ThreadLocal doesn't propagate — they'll need the agentId carried on the event itself, which is a different shape of change. That's follow-up work, and the foundation is here for it.
