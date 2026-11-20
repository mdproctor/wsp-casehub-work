# Phase 1 — Wire Neocortex Memory Stack

**Date:** 2026-08-11
**Status:** Draft
**Branch:** issue-41-autonomous-agent-template
**Issue:** casehubio/examples#41
**Depends on:** Phase 2.5+ (autonomous characters) — complete

## Goal

Replace the manor's chronological memory recall with the full neocortex
memory stack: salience-scored retrieval, reflection, relationship tracking,
and memory decay. This is the foundation for the goal lifecycle phase
(engine SPIs) and trust/personality (Phase 3).

**Success criteria:**
- Characters recall salient memories (recent + important) rather than
  last-10 chronological
- Characters produce reflection insights after accumulating experience
- Characters have relationship memories for agents they interact with
- Old unimportant memories decay over time
- Observations include reflection insights and relationship context

## Architecture Context

See `decisions.md` for the full decision record. Key points:

- **Three-layer architecture** (D1): neocortex for memory, engine SPIs
  for goal/plan lifecycle (next phase), manor for execution loop
- **No `casehub-engine-api` dependency in this phase** (D4): the
  neocortex provides everything needed
- **Standalone reflection** (D2): separate LLM call, async, triggered
  by experience accumulation threshold
- **Per-turn goals preserved** (D3): `newGoals`/`dropGoals` stay in the
  LLM response format until the goal lifecycle phase replaces them

## No New Dependencies

The neocortex APIs needed are already in the pom.xml:
- `casehub-neocortex-memory-api` (compile)
- `casehub-neocortex-memory` (runtime)

No new Maven dependencies are required.

---

## 1. Salience-Scored Retrieval

### 1.1 Change: AgentExperienceService.recall()

Current:
```java
store.query(MemoryQuery.forEntity(agentId, MANOR_DOMAIN, tenantId)
    .withLimit(limit)
    .withOrder(MemoryOrder.CHRONOLOGICAL));
```

Change to:
```java
store.query(MemoryQuery.forEntity(agentId, MANOR_DOMAIN, tenantId)
    .withLimit(recallLimit)
    .withOrder(MemoryOrder.SALIENCE));
```

`recallLimit` configurable via constructor, default 20.

### 1.2 Importance Scoring on Experience Ingest

For salience to be meaningful (`salience = recency × importance`),
experience records must carry importance scores. Currently all records
have null importance (treated as 1.0), making salience equivalent to
recency.

`AgentExperienceService.ingest()` gains an `importance` parameter
derived from the action type at the call site:

| ActionType | Importance | Rationale |
|---|---|---|
| STEAL | 0.9 | High-impact interpersonal action |
| USE | 0.8 | Applies item to world — typically consequential |
| TAKE | 0.7 | Acquires resource |
| GIVE | 0.7 | Transfers resource to another character |
| PULL_ASIDE | 0.7 | Private conversation — socially significant |
| INTERACT | 0.6 | Activates world mechanism |
| dialogue (no action) | 0.5 | Social interaction |
| MOVE | 0.3 | Navigation — routine |
| LOOK | 0.2 | Observation — informational |
| WAIT | 0.1 | No action — minimal significance |

The importance is passed as metadata or set on the `Action` event. If
the `ExperienceRecorder` API doesn't support importance directly, set
it as metadata with key `importance` and file an issue to add first-class
support.

### 1.3 Impact

Characters will recall recent important actions over older routine ones.
A STEAL from 5 minutes ago outranks a MOVE from 2 minutes ago. This
changes character behavior — they'll reference consequential events more
often in their thinking.

---

## 2. Relationship Tracking

### 2.1 Change: AgentExperienceService.ingest()

`ingest()` gains a `targetAgentId` parameter. When non-null, adds
`ExperienceAttributeKeys.TARGET_AGENT` to the experience metadata:

```java
if (targetAgentId != null) {
    metadata.put(ExperienceAttributeKeys.TARGET_AGENT, targetAgentId);
}
```

The neocortex `RelationshipObserver` automatically creates relationship
memories from experience events with this attribute. No additional
wiring is needed — CDI event propagation handles it.

### 2.2 Call Site Changes

In `ScenarioOrchestrator.runAutonomousTicks()`, extract the target agent
from the response:

- `response.talkTo()` → target for dialogue experiences
- `response.action().target()` for GIVE, STEAL, PULL_ASIDE when the
  target is a character (not an object)

Pass as `targetAgentId` to `ingest()`.

In `CharacterAgentLoop.run()` (scripted mode), same extraction logic.

### 2.3 New Query: recallRelationships()

```java
public List<Memory> recallRelationships(String agentId,
                                         String otherAgentId, int limit) {
    return store.query(RelationshipQuery.forPair(agentId, otherAgentId, tenantId)
                    .withLimit(limit)
                    .withOrder(MemoryOrder.SALIENCE));
}
```

Called by `ObservationBuilder` for each character in the room.

---

## 3. Reflection Orchestration

### 3.1 New Class: ManorReflectionSynthesizer

Implements `io.casehub.neocortex.memory.reflection.ReflectionSynthesizer`.

```java
public class ManorReflectionSynthesizer implements ReflectionSynthesizer {

    private final AgentProvider agentProvider;

    @Override
    public List<ReflectionEvent> synthesize(String agentId,
            String tenantId, List<Memory> sources, int targetLevel) {
        // Build prompt from source memories
        // Call LLM via agentProvider
        // Parse response into ReflectionEvent list
    }
}
```

**System prompt:** Not the character's persona prompt. Reflection is
analytical:

```
You are analyzing the recent experiences of an agent to identify
patterns, relationships, and strategic insights. Each insight should
be one clear, specific sentence. Focus on:
- Patterns in other agents' behavior
- Cause-and-effect relationships between actions
- Strategic implications for the agent's goals
- Social dynamics and trust signals

Return a JSON array of insights:
[{"insight": "...", "importance": 0.0-1.0}]
```

**Importance:** Reflection insights get importance 0.7-0.9 (synthesized
knowledge, more valuable than raw experience).

**Model:** Uses `GatedAgentProvider` — same concurrency gate as character
LLM calls. Reflection competes for LLM slots, which naturally throttles
it during busy ticks.

### 3.2 Reflection Trigger Logic

New class: `ManorReflectionTrigger`. Tracks per-agent state:

```java
public class ManorReflectionTrigger {
    private final Map<String, Integer> unreflectedCounts;
    private final Map<String, Double> cumulativeImportance;
    private final int maxUnreflected;          // default 5
    private final double importanceThreshold;  // default 3.0

    public boolean shouldReflect(String agentId, double actionImportance) {
        // increment counts and importance
        // return true if either threshold exceeded
    }

    public void reset(String agentId) {
        // called after reflection completes
    }
}
```

### 3.3 Integration in AgentExperienceService

After `recorder.record(event)` in `ingest()`:

```java
if (reflectionTrigger.shouldReflect(agentId, importance)) {
    Thread.ofVirtual().name(agentId + "-reflect").start(() -> {
        try {
            reflectionOrchestrator.reflect(agentId, tenantId,
                lastReflectionTime.get(agentId), maxSourceMemories);
            reflectionTrigger.reset(agentId);
            lastReflectionTime.put(agentId, Instant.now());
        } catch (Exception e) {
            log.warnf("%s: reflection failed (non-fatal): %s",
                agentId, e.getMessage());
        }
    });
}
```

**Thread safety:** The `ManorReflectionTrigger` uses `ConcurrentHashMap`
and `AtomicInteger`/`DoubleAdder` for lock-free updates. The neocortex
`CaseMemoryStore` is thread-safe by contract. The `ReflectionOrchestrator`
handles its own synchronization.

**Failure mode:** Reflection failure is non-fatal. The agent continues
without insights. The trigger resets to avoid repeated failed attempts
on the same batch.

### 3.4 Configuration

```properties
manor.reflection.enabled=true
manor.reflection.max-unreflected=5
manor.reflection.importance-threshold=3.0
manor.reflection.max-source-memories=15
```

---

## 4. Enhanced Observations

### 4.1 New Section: Insights

After "Past Experience", before "Last Action Result":

```
== Insights ==
- I've noticed Sneekly is always nearby when something goes wrong
- The brass key might open the cabinet in the kitchen
```

Populated by querying the `reflection` domain:

```java
store.query(MemoryQuery.forEntity(agentId,
    ReflectionEvents.DOMAIN, tenantId)
    .withLimit(5)
    .withOrder(MemoryOrder.SALIENCE));
```

Only shown if insights exist. Omitted on first few turns when no
reflection has fired.

### 4.2 New Section: Relationship Notes

After "Characters Present", for each character in the room:

```
== About Sneekly ==
- You recall: Sneekly offered you tea with unusual insistence
- You recall: Sneekly was seen near the kitchen alone
```

Populated by `recallRelationships(agentId, otherAgentId, 3)` for
each character in the room. Only shown if relationship memories exist.

### 4.3 Change: Past Experience

The existing "Past Experience" section already shows memories. The only
change is that memories are now salience-scored rather than chronological,
and the limit increases from 10 to 20. The section renders identically.

### 4.4 ObservationBuilder Signature Change

`buildObservation()` gains an `AgentExperienceService` parameter (or
the pre-queried reflection and relationship memories are passed in):

```java
public static String buildObservation(
    CharacterState character, WorldState world,
    List<AgentGoal> goals,
    PartitionedDrain<String> drain,
    List<Memory> memories,
    List<Memory> reflections,
    Map<String, List<Memory>> relationshipMemories,
    Set<String> observerTags)
```

The caller (ScenarioOrchestrator/CharacterAgentLoop) queries memories,
reflections, and relationships before calling `buildObservation()`.

---

## 5. Memory Decay

### 5.1 Trigger Point

Memory decay runs after each reflection cycle completes. Reflection
synthesizes raw memories into insights, then the raw memories become
candidates for decay.

```java
// After reflection completes successfully:
if (decayEnabled) {
    store.purge(new MemoryRetentionPolicy(
        tenantId, MANOR_DOMAIN, maxAgeHours, minImportance));
}
```

### 5.2 Policy

- `maxAgeHours=168` (7 days) — memories older than 7 days with
  importance below threshold are purged
- `minImportance=0.2` — only purge truly unimportant memories
  (WAIT, LOOK)
- Reflection domain is NOT purged — insights persist

### 5.3 Configuration

```properties
manor.decay.enabled=true
manor.decay.max-age-hours=168
manor.decay.min-importance=0.2
```

---

## 6. Wiring in ScenarioOrchestrator

The orchestrator creates and wires the new components:

```java
// In runScenario(), after creating observationService:
var reflectionSynthesizer = new ManorReflectionSynthesizer(gatedProvider);
var reflectionOrchestrator = new DefaultReflectionOrchestrator(
    store, reflectionSynthesizer);
var reflectionTrigger = new ManorReflectionTrigger(
    maxUnreflected, importanceThreshold);
var experienceService = new AgentExperienceService(
    recorder, store, tenantId,
    reflectionOrchestrator, reflectionTrigger,
    reflectionEnabled, decayEnabled, maxAgeHours, minImportance,
    maxSourceMemories, recallLimit);
```

The `AgentExperienceService` constructor grows to accept reflection
and decay configuration. This is constructor injection — no CDI
changes needed.

---

## 7. File Changes Summary

| File | Change |
|---|---|
| `AgentExperienceService.java` | Add importance parameter to `ingest()`, add `targetAgentId` parameter, switch to SALIENCE order, add reflection trigger evaluation, add `recallReflections()` and `recallRelationships()`, add memory decay after reflection |
| `ManorReflectionSynthesizer.java` | **New** — implements `ReflectionSynthesizer`, LLM-powered insight generation |
| `ManorReflectionTrigger.java` | **New** — tracks unreflected experience count and cumulative importance per agent |
| `ObservationBuilder.java` | Add "Insights" and "Relationship Notes" sections, accept reflection and relationship memories as parameters |
| `ScenarioOrchestrator.java` | Wire reflection components, pass target agent ID to `ingest()`, query reflections and relationships for observation building |
| `CharacterAgentLoop.java` | Pass target agent info to `ingest()` |
| `application.properties` | Add `manor.reflection.*` and `manor.decay.*` config |

---

## 8. Test Plan

### Unit Tests

- **`AgentExperienceServiceTest`**
  - Salience retrieval returns higher-importance recent memories first
  - Importance mapping: STEAL → 0.9, WAIT → 0.1, etc.
  - TARGET_AGENT metadata set when targetAgentId provided
  - Reflection trigger fires after maxUnreflected experiences
  - Reflection trigger fires when cumulative importance exceeds threshold
  - Reflection trigger resets after successful reflection
  - Memory decay purges old low-importance memories
  - Memory decay does not purge reflection domain

- **`ManorReflectionSynthesizerTest`**
  - Given source memories, produces ReflectionEvent list
  - Each event has insight text and importance
  - Mock LLM returns parseable JSON
  - Handles LLM failure gracefully (returns empty list)

- **`ManorReflectionTriggerTest`**
  - Count-based threshold fires at correct count
  - Importance-based threshold fires at correct cumulative value
  - Reset clears both counters
  - Thread-safe under concurrent updates

- **`ObservationBuilderTest`**
  - Insights section renders when reflections provided
  - Insights section omitted when no reflections
  - Relationship notes render per character in room
  - Relationship notes omitted when no relationship memories exist
  - Past Experience section uses salience-ordered memories

### Integration Test

- **`ReflectionIntegrationTest`**
  - Multi-tick scenario: character ingests 5+ experiences with varying
    importance → reflection fires → insights stored in reflection domain
    → subsequent observation includes insights
  - Verify reflection runs asynchronously (ingest returns immediately)
  - Verify relationship memories auto-created from TARGET_AGENT metadata

---

## 9. What's Deferred

| Capability | Deferred to | Reason |
|---|---|---|
| Goal lifecycle (AgentGoal, formation/revision/abandonment) | Goal lifecycle phase | Requires resolving CaseDefinition mapping (D1 revision) |
| Plan structure (replace currentPlan string) | Goal lifecycle phase | Coupled to goal lifecycle |
| `casehub-engine-api` dependency | Goal lifecycle phase | SPIs not consumed in memory phase (D4) |
| Trust scoring (AgentTrustProvider) | Phase 3 | Requires trust model design |
| Personality evolution | Phase 3 | Requires disposition transition model |
| Personality-weighted retrieval | Phase 3 | Depends on personality evolution |
| CBR integration | Future | Valuable but adds complexity; validate basic memory stack first |

## 10. Platform Issues to File

- **Engine:** Generalize `ReflectionTriggerConfig.importanceWeights` to
  accept arbitrary outcome type keys instead of hardcoded Worker outcome
  types
- **Engine:** Extract cognitive SPIs (`GoalFormationStrategy`,
  `GoalRevisionStrategy`) into a standalone module that doesn't require
  `CaseDefinition` — enables autonomous agents without the case model
