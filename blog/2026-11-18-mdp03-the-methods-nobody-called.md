---
layout: post
title: "The Methods Nobody Called"
date: 2026-11-18
entry_type: note
subtype: diary
projects: [casehubio/neocortex]
tags: [cognitive-architecture, cache-management, infrastructure, cognition-core]
series: issue-1098-bdi-semantic-classifier
---

# The Methods Nobody Called

Three `evictIdle()` methods existed in the cognitive pipeline. All of them were public. All of them had clean implementations — cutoff time, stale entry scan, removal. None of them had a caller.

This is the gap between having a capability and using it. The methods were written as part of earlier work on situational personality caches and cognitive defaults. Each cache holds per-agent derived state: effective personality profiles, situational detection results, learning profiles. With a fixed cast of 17 characters, the caches stay small and nobody notices. With dynamic agent creation — agents spawned and destroyed at runtime — those caches grow without bound.

The fix isn't the eviction logic. That was already written. The fix is the scheduling.

## Why two thresholds

The obvious implementation: evict every N ticks. Run through the caches, remove anything idle, move on. But tick rate varies. A scenario with rapid-fire events might clock 60 ticks in under a minute. Running eviction that often is wasteful — the entries haven't had time to go stale.

The opposite problem: a system with slow ticks might go 20 minutes between eviction passes, even though entries have been idle for 15.

The solution is a dual threshold: evict only when **both** conditions are met. At least 60 ticks since the last eviction, AND at least 5 minutes of wall-clock time. The tick threshold prevents running eviction on every heartbeat in a fast scenario. The time threshold prevents running it when nothing has had time to go stale. Together, they give "whichever is longer" semantics.

![Cache eviction dual-threshold flow](svg/2026-11-18-cache-eviction-flow.svg)

The `CacheEvictionScheduler` tracks both counters and resets them together after each eviction pass. Registered evictors receive a configurable `maxIdle` duration — 10 minutes by default — so each cache decides for itself what "stale" means relative to that window.

## The configure-or-not pattern

`CognitionCore` already had a convention for optional infrastructure: nullable fields with a `configure*` method. `BehavioralSynthesisPhase`, `TheoryOfMindBridge`, `PipelineSnapshotStore` — all follow the same pattern. Set them and they participate in the tick pipeline. Leave them null and the tick skips them without ceremony.

The scheduler follows the same convention. `configureCacheEviction(scheduler)` wires it in. If nobody calls it, `tick()` just checks `if (scheduler != null)` and moves on. No config flag, no feature toggle in the already-crowded `CognitionConfig` record (26 boolean fields — adding a 27th for infrastructure plumbing felt wrong). The "flag" is implicit: configure it or don't.

```java
public void configureCacheEviction(CacheEvictionScheduler scheduler) {
    this.cacheEvictionScheduler = scheduler;
    if (effectiveDefaults != null) {
        scheduler.register(effectiveDefaults::evictIdle);
    }
}
```

`EffectiveCognitiveDefaults` auto-registers when the scheduler is configured. External caches — like `SituationalPersonalityParticipant` — register themselves via `scheduler.register(participant::evictIdle)`. The scheduler doesn't know or care what it's evicting. It just calls each registered `Consumer<Duration>` when the thresholds are met.

## What the review caught

Two issues surfaced during the code review pass. First: a throwing evictor would stop the iteration, leaving subsequent evictors un-called. The fix wraps each call in try-catch with a warning log — consistent with the `safeRun` pattern used elsewhere in `CognitionCore`.

Second: both `configureCacheEviction` and `setEffectiveDefaults` registered `effectiveDefaults::evictIdle` with the scheduler. If you called them in the wrong order — scheduler first, then defaults — the defaults registered once from each method, running eviction twice per cycle. The builder's call order happened to be safe (defaults first, scheduler second), but the public API didn't enforce it. The fix: registration only happens in `configureCacheEviction`. One registration path, no double-call risk.

## What this opens up

Right now, with 17 fixed characters, this is infrastructure waiting for its moment. The caches stay small and eviction is a no-op scan over a handful of entries. But the cognitive pipeline is heading toward dynamic agent creation — agents that arrive, interact, and leave. When that happens, the caches will grow proportionally, and the eviction scheduler will be the thing that keeps memory bounded without anyone having to think about it.

The pattern also establishes a convention for future caches. Any component that holds per-agent state can register with the scheduler via a single `Consumer<Duration>`. The scheduling policy — tick threshold, time threshold, max idle — lives in one place, configured once at startup.
