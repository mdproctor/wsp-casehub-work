# MentalModel Design — Theory of Mind with BDI Tracking

**Issue:** casehubio/blocks#123
**Date:** 2026-08-20
**Decisions:** D23-D31

## Overview

MentalModel maintains a BDI (Beliefs, Desires, Intentions) model per actor the agent interacts with. It tracks what others know, want, and plan to do — enabling the agent to reason about other people's cognitive states, not just its own goals.

**Distinction from UserModel (D30):** UserModel tracks stable behavioral traits (communication style, preferences, topics of interest) — identity-level, slow-changing. MentalModel tracks dynamic cognitive state (current beliefs, desires, intentions) — situation-level, fast-changing. "User prefers concise communication" is UserModel. "User currently believes the deployment is risky" is MentalModel. Both are keyed by (agentId, subjectId, tenantId) but serve different consumer needs.

**Composition targets (issue #123):** BeliefSet + RelationshipEvent + GoapPlanner + EpistemicRule. All four are composed:
- `BeliefSet<String>` — AGM revision for belief conflict resolution
- `RelationshipEvent` — signal source via `RelationshipCue` variant
- `GoapWorldState` — confidence-aware projection via `project()`
- `EpistemicRule` — optional conversation-derived belief classification via `observeConversation()`

## Architecture

### Orchestrator: `MentalModelOrchestrator`

`@ApplicationScoped` CDI bean following the established social cognition pattern. Per-subject BDI state in `ConcurrentHashMap<String, SubjectMentalState>`. Per-subject `ReentrantLock` for tick concurrency. Four public methods:

- **`record(MentalStateSignal, agentId, subjectId, tenantId)`** — accumulates signals. Heuristic extraction of explicit cues from `VerbalCue` signals happens immediately (O(n) in signal text length, no LLM, no store write). Appends signal text to a bounded ring buffer for later LLM inference.

- **`tick(agentId, subjectId, tenantId) → MentalModelTick`** — re-evaluates all three BDI dimensions. Applies confidence decay. Evicts entries below confidence floor. Optionally invokes LLM when enough new signal has accumulated (gated by `minSignalsForInference` and `inferenceCooldown`). Persists via `MentalModelStore`. Returns sealed outcome: `Unchanged`, `Updated`, `Inferred` (LLM ran). When subject state is null but a stored snapshot exists, reloads from `MentalModelStore` before processing.

- **`project(agentId, subjectId, tenantId) → List<MentalProjection>`** — projects current BDI state into condition/confidence pairs. Mapping: each BDI entry with confidence above the projection floor produces one `MentalProjection` where `conditionKey` = entry key (e.g., "deployment_risk"), `value` = true (the entry exists and is believed/desired/intended), `confidence` = the entry's current confidence, `dimension` = BELIEF/DESIRE/INTENTION. Consumers filter by confidence threshold and merge into `GoapWorldState`.

- **`observeConversation(CommonGroundState, agentId, subjectId, tenantId)`** — optional composition with the conversation infrastructure. When a `CommonGroundAnalyser` has classified conversation points via `EpistemicRule`, the consumer passes the resulting `CommonGroundState`. The orchestrator extracts belief attributions: ESTABLISHED facts → high-confidence beliefs (0.9), PENDING claims → medium-confidence beliefs (0.5), DISPUTED points → low-confidence beliefs (0.3). This bridges the conversation epistemic layer into per-actor belief tracking.

### BDI State Container: `SubjectMentalState`

Package-private mutable state per subject:

```
SubjectMentalState:
  agentId, subjectId, tenantId     — identity triple
  beliefs: Map<String, AttributedState> — key=topic, with confidence + entrenchment
  desires: Map<String, AttributedState> — key=desire name, with confidence
  intentions: Map<String, AttributedState> — key=intention name, with confidence
  pendingSignals: int              — count since last inference
  signalBuffer: RingBuffer<String> — bounded (maxBufferSize), newest overwrites oldest
  lastSignalTimestamp: Instant     — for confidence decay
  lastInferenceTimestamp: Instant  — for cooldown gating
  lastActivityTimestamp: Instant   — for eviction
  currentSnapshot: MentalModelSnapshot — latest persisted state
```

### BDI Dimension Types

**Unified type:** All three BDI dimensions use `AttributedState` (R1-07). The `BdiDimension` enum distinguishes them. Different half-lives per dimension are configured in `MentalModelConfig`.

```java
record AttributedState(
    String key,              // e.g., "deployment_risk" or "quick_resolution"
    String description,      // e.g., "subject thinks deployments are risky"
    double confidence,       // [0,1], decays with time
    int entrenchment,        // reinforcement count (beliefs only; 0 for desires/intentions)
    Instant lastReinforced,  // timestamp of last confirming signal
    BdiDimension dimension   // BELIEF, DESIRE, INTENTION
) {}
```

**Belief revision via BeliefSet:** `BeliefSet<String>` is used specifically for AGM revision when contradictory evidence arrives — not as the primary belief container. The orchestrator maintains beliefs in `Map<String, AttributedState>`. When revision is needed (LLM detects contradiction or consumer signals conflict), the orchestrator:
1. Constructs a temporary `BeliefSet<String>` from the current beliefs map (mapping entrenchment from AttributedState)
2. Calls `revise(newBelief, consistencyChecker)`
3. Reads surviving beliefs back into the map, preserving confidence/timestamps from the original entries

This avoids extending `Belief<T>` with confidence (which would change the existing API). BeliefSet provides the AGM revision algorithm; AttributedState provides the temporal metadata.

**Default ConsistencyChecker:** Always consistent. This means `revise()` degenerates to `expand()` (simple overwrite by key) unless the consumer provides a domain-specific checker. AGM revision is opt-in infrastructure — semantic consistency checking (e.g., "trusts us" contradicts "distrusts us") requires domain knowledge that blocks cannot provide. The spec is honest about this: without a consumer-provided checker, belief updates are last-write-wins by key.

### Confidence Decay

Each BDI dimension decays at a configurable rate (D27):
- **Beliefs** decay slowly (default halfLife: 7 days). Well-established beliefs persist.
- **Desires** decay moderately (default halfLife: 1 day). Wants shift with context.
- **Intentions** decay quickly (default halfLife: 4 hours). Plans change frequently.

Decay formula: `confidence × Math.pow(0.5, elapsed / halfLife)`

Below a configurable floor (default 0.1), entries are evicted regardless of entrenchment. Entrenchment is orthogonal to confidence: it determines revision ordering when beliefs conflict (which survives AGM contraction), not whether a belief is still fresh enough to act on.

### GOAP Projection

`MentalProjection` record:
```java
record MentalProjection(
    String conditionKey,      // = AttributedState.key (e.g., "deployment_risk")
    boolean value,            // true = entry exists above projection floor
    double confidence,        // from AttributedState.confidence
    BdiDimension dimension    // BELIEF, DESIRE, INTENTION
) {}
```

**Projection mapping:** Each `AttributedState` entry with confidence ≥ `projectionFloor` (default 0.3) produces one `MentalProjection`. The `conditionKey` IS the entry's key — no transformation. Beliefs project as-is ("deployment_risk" → true at confidence 0.7). Desires and intentions project with their key as well. The value is always `true` — the entry's existence above the floor IS the condition.

Consumer usage:
```java
var projections = mentalModel.project(agentId, subjectId, tenantId);
var conditions = new HashMap<String, Boolean>();
for (var p : projections) {
    if (p.confidence() >= 0.5) {  // consumer's threshold
        conditions.put(p.conditionKey(), p.value());
    }
}
var worldState = new GoapWorldState(conditions);
```

The projection method is a pure function of current BDI state — no side effects, no engine dependency.

## Signal Model

### `MentalStateSignal` — sealed interface (D25, D31)

Per-orchestrator signal type, consistent with the social cognition pattern:

```java
sealed interface MentalStateSignal {
    String content();  // raw text for LLM inference

    record VerbalCue(String content, CueType type) implements MentalStateSignal {}
    // Explicit verbal statements: "I think X", "I want Y", "I plan to Z"

    record BehavioralCue(String content, String actionType) implements MentalStateSignal {}
    // Observable actions that imply mental state

    record ContextualCue(String content, Map<String, String> metadata) implements MentalStateSignal {}
    // Contextual information: deadlines, stress indicators, environmental factors

    record RelationshipCue(RelationshipEvent event) implements MentalStateSignal {
        @Override public String content() { return event.description(); }
    }
    // Wraps RelationshipEvent — composes the neocortex relationship layer
}
```

`CueType` enum: `BELIEF_STATEMENT`, `DESIRE_EXPRESSION`, `INTENTION_DECLARATION`.

The `RelationshipCue` variant fulfills the issue #123 composition mandate for RelationshipEvent. Consumers already producing `RelationshipEvent`s can wrap them as `RelationshipCue` to feed MentalModel. The quality signal on the event informs belief confidence (POSITIVE → higher confidence on inferred beliefs from that interaction).

### Heuristic Extraction (Tier 1)

On `record()`, extract explicit cues from `VerbalCue` signals:
- `BELIEF_STATEMENT` → expand belief (key derived from content, confidence 0.8)
- `DESIRE_EXPRESSION` → add desire (confidence 0.8)
- `INTENTION_DECLARATION` → add intention (confidence 0.8)

Heuristic extraction runs in O(n) of signal text length. False positives from figurative language are tolerated — LLM inference on the next tick can correct them via merge semantics.

### LLM Inference (Tier 2)

On `tick()`, when `pendingSignals >= minSignalsForInference` and `inferenceCooldown` has elapsed:

1. Assemble prompt with current BDI state + recent signals from ring buffer (most recent `maxSignalsInPrompt` entries)
2. Ask LLM to infer BDI updates
3. Parse structured JSON response
4. Apply updates via **merge semantics** (matching UserModel precedent):
   - Entries present in response: update description, reset confidence to LLM-provided value, reset lastReinforced
   - Entries absent from response: preserved unchanged (previous state survives)
   - New entries (key not in current state): added
   - For beliefs, if the ConsistencyChecker is non-trivial, run BeliefSet.revise() on the merged set

System prompt template:
```
You are analysing conversation signals to infer another person's mental state.
Given the current attributed beliefs, desires, and intentions, plus recent signals,
produce an updated BDI assessment.

Respond with JSON only:
{"beliefs": [{"key": "...", "text": "...", "confidence": 0.8}],
 "desires": [{"key": "...", "text": "...", "confidence": 0.7}],
 "intentions": [{"key": "...", "text": "...", "confidence": 0.5}]}

Only include NEW or CHANGED entries. Omit unchanged entries — they are preserved.
Only include entries with confidence >= 0.3.
```

## Persistence

### `MentalModelStore` SPI (D28)

```java
public interface MentalModelStore {
    void store(MentalModelSnapshot snapshot);
    Optional<MentalModelSnapshot> lookup(String agentId, String subjectId, String tenantId);
    List<MentalModelSnapshot> findByAgent(String agentId, String tenantId);
    void eraseSubject(String subjectId, String tenantId);
}
```

### `MentalModelSnapshot` — persisted state

```java
record MentalModelSnapshot(
    String agentId,
    String subjectId,
    String tenantId,
    List<AttributedState> beliefs,
    List<AttributedState> desires,
    List<AttributedState> intentions,
    Instant lastSignal,
    Instant lastInference,
    Instant snapshotCreated
) {}
```

Uses `AttributedState` directly — no separate `SnapshotBelief` type needed since `AttributedState` already carries entrenchment, confidence, and timestamp.

### `CbrMentalModelStore` — `@DefaultBean` adapter

Backs onto `CbrCaseMemoryStore`. Mental model snapshot stored as a CbrCase:
- `problem` = serialized BDI summary text
- `caseType` = "mental-model"
- Features: agentId, subjectId, tenantId (for lookup), BDI entries as JSON StringVal
- Supersession for versioning (each tick that persists supersedes the previous snapshot)

Same adapter pattern as `CbrUserProfileStore` (D20). Known limitation: lookup uses `retrieveSimilar` with feature-match query — this is a similarity search pretending to be an exact-key lookup. Acceptable for the same reason as UserProfileStore: the alternative is a new persistence SPI at the neocortex level, which is premature. If lookup becomes a bottleneck, the in-memory state (ConcurrentHashMap) serves most reads — the store is only consulted on cold-start reload.

## Configuration

### `MentalModelConfig`

```java
record MentalModelConfig(
    Duration beliefHalfLife,           // default: 7 days
    Duration desireHalfLife,           // default: 1 day
    Duration intentionHalfLife,        // default: 4 hours
    double confidenceFloor,            // default: 0.1 — eviction threshold
    double projectionFloor,            // default: 0.3 — projection threshold
    int minSignalsForInference,        // default: 3
    Duration inferenceCooldown,        // default: 5 minutes
    int maxSignalsInPrompt,            // default: 20
    int maxBufferSize,                 // default: 100 — ring buffer cap
    Duration evictionTimeout,          // default: 24 hours (for in-memory state)
    Duration expectedTickInterval      // default: 1 minute
) {}
```

**Config interaction:** With `expectedTickInterval` = 1 min and `inferenceCooldown` = 5 min, at most 1 in 5 ticks triggers LLM inference. The other 4 ticks apply decay, evict stale entries, and persist. This is intentional: most ticks should be cheap. LLM inference is the expensive exception, not the common path.

## Tick Outcome

### `MentalModelTick` — sealed interface

```java
sealed interface MentalModelTick {
    record Unchanged(String reason) implements MentalModelTick {}
    record Updated(MentalModelSnapshot snapshot) implements MentalModelTick {}
    record Inferred(MentalModelSnapshot snapshot,
                    MentalModelSnapshot previous) implements MentalModelTick {}
}
```

## Package Placement

`io.casehub.blocks.agentic.social` — alongside PersonalityEvolutionOrchestrator, InnerLifeOrchestrator, and UserModelOrchestrator. MentalModel is the fourth member of the social cognition family: how the agent models others' cognitive states (D22 extended).

**Cross-orchestrator integration:** MentalModel is architecturally independent — it doesn't call or depend on the other orchestrators. Integration happens at the consumer level: consumers can feed MentalModel projections into InnerLife's motivation assessment, or use UserModel's relationship stage to set confidence priors for MentalModel. These are consumer-side compositions, not built into blocks.

## Type Inventory

| Type | Kind | What it does |
|------|------|-------------|
| `MentalModelOrchestrator` | `@ApplicationScoped` CDI bean | Composition root: record()+tick()+project()+observeConversation() |
| `MentalStateSignal` | Sealed interface | Input signal: VerbalCue, BehavioralCue, ContextualCue, RelationshipCue |
| `CueType` | Enum | BELIEF_STATEMENT, DESIRE_EXPRESSION, INTENTION_DECLARATION |
| `AttributedState` | Record | Unified BDI entry: key, description, confidence, entrenchment, lastReinforced, dimension |
| `MentalProjection` | Record | GOAP projection: conditionKey + value + confidence + dimension |
| `BdiDimension` | Enum | BELIEF, DESIRE, INTENTION |
| `MentalModelTick` | Sealed interface | Tick outcome: Unchanged, Updated, Inferred |
| `MentalModelSnapshot` | Record | Persisted BDI state |
| `MentalModelStore` | Interface (SPI) | Persistence: store, lookup, findByAgent, eraseSubject |
| `CbrMentalModelStore` | `@DefaultBean @ApplicationScoped` | CbrCaseMemoryStore adapter |
| `MentalModelConfig` | Record | Configuration: decay rates, thresholds, cooldowns |

11 types total (6 new records/enums, 2 sealed interfaces, 1 CDI bean, 1 SPI interface, 1 default bean).

## Dependencies

**Compile (existing):** `casehub-neocortex-memory-api` (CbrCaseMemoryStore, RelationshipEvent — composed via RelationshipCue)
**Provided (existing):** `casehub-platform-agent-api` (AgentProvider for LLM inference)
**Internal (existing):** `blocks.agentic.belief` (BeliefSet, Belief, ConsistencyChecker — used for AGM revision), `blocks.conversation` (CommonGroundState, EpistemicStatus — composed via observeConversation())

No new external dependencies.

## References

- Research §2.5 — Mental Model (Theory of Mind) pattern definition
- ToMA (Findings of ACL 2026) — Infusing Theory of Mind into LLM agents
- Beyond Words (Findings of ACL 2025) — ToM-informed alignment
- TimeToM (Findings of ACL 2024) — Temporal reasoning in Theory of Mind
- DPT-Agent (2025) — Dual Process Theory with Theory of Mind
- BeliefSet.java (blocks/agentic/belief) — AGM belief revision infrastructure
- EpistemicRule.java, CommonGroundState.java (blocks/conversation) — epistemic classification
- GoapWorldState.java (engine-api) — world state record with .with() builder
- RelationshipEvent.java (neocortex-memory-api) — relationship interaction records
- UserModelOrchestrator.java (blocks/agentic/social) — precedent orchestrator
- InteractionSignal.java (blocks/agentic/social) — precedent signal model
- D23-D31 decisions — validated design choices
- Spec review R1-01 through R1-12 — addressed findings
