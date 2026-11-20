# InnerLife Pattern — Design Spec

**Issue:** casehubio/blocks#119
**Date:** 2026-08-18
**Branch:** issue-126-autonomous-agent-patterns

## Summary

InnerLife is a background thought loop in `io.casehub.blocks.agentic.personality` that enables agents to initiate conversations proactively rather than only responding when spoken to. It composes existing platform capabilities (ReflectionOrchestrator, AgentProvider) with consumer-provided context (AffordanceRenderer output) into a three-stage evaluation pipeline gated by composable civility constraints.

The pattern produces initiation decisions with content — consuming apps control dispatch. It does NOT own channel communication, scheduling, or message formatting.

## Architecture

### Existing Platform Capabilities (composed, not built)

1. **ReflectionOrchestrator** (neocortex) — `reflect(agentId, tenantId, since, maxSourceMemories) → List<String>` — generates reflective thoughts from accumulated memories
2. **AgentProvider** (platform-agent-api) — `invoke(AgentSessionConfig) → Multi<AgentEvent>` — LLM invocation returning a reactive stream of events. InnerLife follows the established text-collection-and-parse pattern (RoutingSupport, LlmDecomposition, LlmContentSummariser): collect `TextDelta` events, join into string, parse JSON

### Consumer-Provided Context

- **AffordanceRenderer** output — consumers pre-render affordance context via `AffordanceRenderer` and pass the result as the `channelContext` parameter to `tick()`. InnerLife is decoupled from the affordance rendering pipeline.

### What InnerLife Adds

1. **CivilityConstraint SPI** — composable pre-dispatch gating for social norms (rate limiting, gap enforcement, cooldown)
2. **Content quality gate** — novelty scoring + observation count + quiet-period bypass (System 1/2 fast path)
3. **InnerLifeOrchestrator** — periodic tick cycle: civility → content quality → reflect → score motivation → output. Maintains own per-agent event buffer and raw text buffer (not ObservationAccumulator — simpler lifecycle, no destructive drain or async rendering needed)
4. **Token-level novelty scorer** — zero-cost content novelty detection

### Evaluation Pipeline

```
tick() → CivilityConstraint.permitInitiation()    [D4 — zero cost]
              | permitted
         ContentQualityGate: novelty + obs count   [D3 — zero cost]
              | novel OR quiet-period bypass
         ReflectionOrchestrator.reflect()           [LLM call]
              | reflections
         AgentProvider.invoke() → motivation 0-1    [D2 — LLM call]
              | above threshold
         Initiated(content, channelHint, score)     [D1 — output]
```

Any stage that fails returns `Silent` immediately — no further cost.

### Relationship to Existing SPIs (D5, D6)

InnerLife is a blocks pattern (not a qhorus extension). It produces content dispatched VIA qhorus — it is a producer, not a qhorus component. CivilityConstraint is separate from ActivationRule — different architectural domains (social initiation vs orchestration activation). ActivationContext fields (activationCount, consecutiveIdleActivations, lastAggregationResult) reflect orchestration lifecycle, not social behaviour.

## CivilityConstraint SPI (D4)

### CivilityConstraint

```java
@FunctionalInterface
public interface CivilityConstraint {
    CivilityCheck permitInitiation(InitiationContext context);
}
```

### CivilityCheck

```java
public sealed interface CivilityCheck {
    record Permitted() implements CivilityCheck {}
    record Denied(String reason) implements CivilityCheck {}
}
```

### InitiationContext

```java
public record InitiationContext(
    Instant lastInitiationTimestamp,
    int initiationsInWindow,
    int consecutiveInitiationsWithoutResponse,
    AgentDescriptor descriptor) {}
```

`AgentDescriptor` (eidos-api) carries agent identity (`agentId`, `tenancyId`), personality (`disposition`), and domain context (`briefing`, `slot`). This is the platform's identity type — the correct domain for social-behaviour gating. `AgentRef` is an execution-domain type for agentic orchestration (WorkerAgent, ChannelAgent, etc.) and carries no identity or personality information. `initiationsInWindow` is computed by the orchestrator from a sliding window of initiation timestamps (see Per-Agent State).

### Default Implementations

| Implementation | Default | Logic |
|---|---|---|
| `MinimumGapConstraint` | 5 min | Denied if `now - lastInitiation < gap` |
| `MaxPerWindowConstraint` | 3/hour | Denied if `initiationsInWindow >= max` |
| `ConsecutiveInitiationCooldownConstraint` | 2 consecutive | Denied if `consecutiveWithoutResponse >= max` (cooldown until someone else posts) |

The orchestrator runs all constraints — first `Denied` wins. All checked before any LLM call.

## Content Quality Gate (D3)

### ContentQualityGate

```java
public record ContentQualityGate(double noveltyThreshold,
                                  int minObservations,
                                  Duration quietPeriodBypass) {
    public static final double DEFAULT_NOVELTY_THRESHOLD = 0.3;
    public static final int DEFAULT_MIN_OBSERVATIONS = 3;
    public static final Duration DEFAULT_QUIET_PERIOD = Duration.ofMinutes(30);

    public static ContentQualityGate defaults() {
        return new ContentQualityGate(DEFAULT_NOVELTY_THRESHOLD,
                DEFAULT_MIN_OBSERVATIONS, DEFAULT_QUIET_PERIOD);
    }
}
```

### Evaluation Logic

1. If `observationCount < minObservations` AND `timeSinceLastLlmEval < quietPeriodBypass` → skip (Silent)
2. Compute token-level Jaccard distance between current raw observation text and previous raw observation text
3. If `novelty < noveltyThreshold` AND `timeSinceLastLlmEval < quietPeriodBypass` → skip (Silent)
4. Otherwise → proceed to reflection + LLM scoring

**Observation text lifecycle:** "Current observation text" is the raw text buffer maintained by the orchestrator — a concatenation of `event.payload().toString()` values appended during each `observe()` call. This is NOT the rendered output of `ObservationAccumulator.drainObservation()`. The raw text buffer is zero-cost to read (no async, no destructive drain) and serves only the novelty comparison. The rendered observation text for the LLM prompt (step 4 of tick) is produced separately by rendering the event buffer when the LLM call proceeds.

The quiet-period bypass enables spontaneous initiation: after extended silence, the LLM decides whether elapsed time alone justifies speaking. Without it, tick() would be functionally reactive.

### TokenJaccardDistance

Package-private utility. Token-level Jaccard distance: `1 - |A ∩ B| / |A ∪ B|` where A and B are whitespace-tokenized word sets from the observation texts. Returns 1.0 for completely disjoint texts, 0.0 for identical texts.

Independent implementation for two reasons: (1) qhorus's `JaccardSimilarity` is package-private and inaccessible from blocks; (2) Apache Commons Text's `JaccardSimilarity` (on classpath via transitive dependency) operates at the **character level** — each character in the CharSequence is a set element. Character-level Jaccard is semantically meaningless for natural language novelty detection ("hello world" vs "world hello" would score as identical at character level but the word ordering change is irrelevant for both). Token-level Jaccard over whitespace-split word sets is the correct granularity for detecting whether new observations contain materially different content.

Designed to be replaceable — embedding-based novelty via `CbrSimilarityScorer` (neocortex-memory-api) is the natural upgrade when semantic precision justifies the added dependency and latency.

## Orchestrator

### InnerLifeOrchestrator

`@ApplicationScoped` CDI bean following the established `tick()` pattern.

```java
@ApplicationScoped
public class InnerLifeOrchestrator {

    InnerLifeOrchestrator(
        ReflectionOrchestrator reflectionOrchestrator,
        AgentProvider agentProvider,
        Instance<CivilityConstraint> civilityConstraints,
        InnerLifeConfig config);

    void observe(LevelEvent<?> event, AgentDescriptor descriptor);

    void observeResponse(AgentDescriptor descriptor);

    InnerLifeTick tick(AgentDescriptor descriptor, String channelContext);
}
```

### observe(event, descriptor)

Appends the event to the per-agent `eventBuffer` and appends `event.payload().toString()` to the per-agent `rawObservationText` buffer. Per-agent state is keyed by `descriptor.tenancyId() + ":" + descriptor.agentId()` — same composite key pattern as `PersonalityEvolutionOrchestrator.agentKey()`. Called by consuming app whenever channel activity occurs. Increments `observationCountSinceLastInitiation`. Updates `lastActivityTimestamp` for eviction tracking.

**Payload contract:** `event.payload().toString()` must produce human-readable text for the novelty gate (Jaccard distance) and LLM prompt to be meaningful. Consuming apps are responsible for ensuring event payloads have useful `toString()` implementations — domain records with default `toString()` (e.g., `MyRecord[field=value]`) are acceptable; opaque object references are not.

### observeResponse(descriptor)

Called when a non-self message is observed after an initiation. Resets `consecutiveInitiationsWithoutResponse` to 0. This is how the cooldown constraint knows someone responded.

### tick(descriptor, channelContext)

The periodic thought cycle. Synchronous — blocks on I/O via `.await().indefinitely()` (same pattern as PersonalityEvolutionOrchestrator). Consuming apps call tick() from their own scheduling mechanism, not from Vert.x event-loop threads.

0. **Snapshot:** Under the per-agent state lock, snapshot `eventBuffer` and `rawObservationText` (copy), then clear both live buffers. Release lock. All subsequent steps operate on the snapshot — `observe()` can keep appending to the (now empty) live buffers concurrently.
1. **Civility gate (D4):** Build `InitiationContext` from per-agent state (compute `initiationsInWindow` from sliding window timestamps). Run all `CivilityConstraint` instances. First `Denied` → return `Silent(reason)`.
2. **Content quality gate (D3):** Check novelty + observation count + quiet-period bypass using the **snapshotted** raw text. Fail → return `Silent`.
3. **Reflect:** Call `reflectionOrchestrator.reflect(agentId, tenantId, since, maxSourceMemories)` → `List<String>` reflections. Synchronous. `since` = `lastLlmEvaluationTimestamp` (or `Instant.EPOCH` on first evaluation) — reflections cover the period since the LLM last consumed reflection context, avoiding duplicate input.
4. **Score motivation (D2):** Render the **snapshotted** event buffer into observation text (most recent `maxObservationsInPrompt` events; default 50). Build `AgentSessionConfig` with system prompt and assembled user prompt (see Prompt Design below). Invoke `agentProvider.invoke(config)`, collect `TextDelta` events into a string via `.await().indefinitely()`, parse as JSON into `MotivationAssessment(double score, String content, String channelHint)` — a package-private record internal to the orchestrator. On parse failure (malformed JSON, missing fields, score outside [0,1]): log warning, return `Silent("parse failure")`.
5. **Threshold check:** If `score >= motivationThreshold` → return `Initiated(content, channelHint, score)`. Otherwise → return `Silent`.
6. **Update state:** Record `lastLlmEvaluationTimestamp`, store **snapshotted** raw text as `previousObservationText`. If Initiated: record `lastInitiationTimestamp`, add timestamp to sliding window, increment `consecutiveInitiationsWithoutResponse`, reset observation count. Prune stale entries from the agent state map (eviction scan).

### Prompt Design

**System prompt:**

```
You are an agent with an inner life. Given your personality, recent observations,
reflections, and available channels, decide whether you are motivated to initiate
a conversation right now.

Respond with JSON only: {"score": <0.0-1.0>, "content": "<what you want to say>",
"channelHint": "<suggested channel or null>"}

Score 0.0 = no motivation. Score 1.0 = strongly motivated. Only produce content
if you genuinely have something worth saying. If unmotivated, set score low and
content to empty string.
```

**User prompt assembly:**

```
Personality: {descriptor.name()} — {descriptor.briefing()}
Disposition: {formatted disposition profile}

Recent observations (most recent {maxObservationsInPrompt}, default 50):
{rendered event buffer — event.payload().toString() with timestamps}

Reflections:
{reflections joined with newlines, or "No recent reflections." if empty}

Available channels and context:
{channelContext parameter}
```

**Model:** Uses the default model from `AgentSessionConfig.of(systemPrompt, userPrompt)` — no model override. Consuming apps that need model control can provide a custom `AgentProvider` wrapper.

**Extension point:** A `SystemPromptCustomiser`-style hook is not included in the initial design. The system prompt is internal to InnerLife — it defines the motivation-scoring contract. If consumer customisation proves necessary, a `MotivationPromptCustomiser` SPI can be added as a follow-up without breaking the existing API.

### InnerLifeTick (D1)

```java
public sealed interface InnerLifeTick {
    record Silent(@Nullable String reason) implements InnerLifeTick {}
    record Initiated(String content, @Nullable String channelHint,
                     double motivationScore) implements InnerLifeTick {}
}
```

`channelHint` is the LLM's suggestion based on affordance context provided by the consumer via the `channelContext` parameter. The LLM reasons about channel names in observation text and suggests a target. No independent channel topology dependency.

### Per-Agent State

In-memory, keyed by `tenancyId:agentId` in `ConcurrentHashMap`:

| Field | Type | Purpose |
|---|---|---|
| `eventBuffer` | `List<LevelEvent<?>>` | Buffered observations for LLM prompt rendering |
| `rawObservationText` | `StringBuilder` | Raw `payload.toString()` concatenation for novelty scoring |
| `lastInitiationTimestamp` | `Instant` | For civility gap check |
| `lastLlmEvaluationTimestamp` | `Instant` | For quiet-period bypass |
| `initiationTimestamps` | `Deque<Instant>` | Sliding window for rate limiting — `initiationsInWindow` is computed as count of timestamps where `now - ts < windowDuration`. Old entries pruned on each tick |
| `consecutiveInitiationsWithoutResponse` | `int` | For cooldown (reset via `observeResponse()`) |
| `previousObservationText` | `String` | For novelty scoring (Jaccard distance vs current rawObservationText) |
| `observationCountSinceLastInitiation` | `int` | For minimum observation check |
| `lastActivityTimestamp` | `Instant` | For staleness-based eviction |

**Eviction:** Per-agent state entries not observed for a configurable duration (default: 24 hours) are evicted during `tick()` only (step 6). Not during `observe()` — observe is the fast path (called on every channel event) and must stay O(1). The eviction scan during tick is negligible relative to the LLM call cost. The `lastActivityTimestamp` is updated on every `observe()` and `tick()` call.

**Restart resilience:** All state is in-memory (`@ApplicationScoped` lifecycle). On restart, all agents start with zero state — no initiations pending, no cooldowns active. This is acceptable because the first tick after restart evaluates fresh, and the conservative defaults (5 min gap, 3/hour max) prevent over-posting even without history. Same reasoning as PersonalityEvolutionOrchestrator.

### Thread Safety

Per-agent `ReentrantLock` for `tick()` (same pattern as PersonalityEvolutionOrchestrator). `observe()` synchronizes on the per-agent state object to append to `eventBuffer` and `rawObservationText` — O(1), never blocks on the tick lock. `tick()` step 0 acquires the same state-object lock to snapshot and clear the buffers, then releases it before the pipeline proceeds. This means `observe()` and tick's snapshot are mutually exclusive, but observe() never blocks for the duration of the LLM call. `observeResponse()` uses atomic operations on the per-agent state.

### Configuration (InnerLifeConfig)

| Key | Default | Meaning |
|---|---|---|
| `motivationThreshold` | 0.6 | LLM motivation score required to initiate |
| `noveltyThreshold` | 0.3 | Minimum Jaccard distance for content novelty |
| `minObservations` | 3 | Minimum observations before LLM evaluation |
| `quietPeriodBypass` | 30 min | Time before quiet-period triggers LLM evaluation regardless |
| `maxReflectionSources` | 10 | Max source memories for ReflectionOrchestrator |
| `maxObservationsInPrompt` | 50 | Max events rendered in LLM prompt (most recent N) |
| `windowDuration` | 1 hour | Time window for MaxPerWindowConstraint rate limiting |
| `evictionTimeout` | 24 hours | Remove per-agent state not observed for this duration |

## Evolution from Issue #119

Issue #119 listed composition targets: "ReflectionOrchestrator + Affordance + ActivationRule + Watchdog + JobScheduler." The spec departs from this list based on architectural analysis:

- **ActivationRule** → replaced by CivilityConstraint (D6: different architectural domain — orchestration activation vs social initiation)
- **Watchdog** → replaced by own civility gate (D4/D5: pre-dispatch gating vs post-hoc alerting)
- **JobScheduler** → consumer-driven scheduling (InnerLife is a library, not a framework; consumers call tick() from their own scheduling mechanism)

The decisions doc (D4, D5, D6) records the full rationale. The issue description on GitHub should be updated to reflect the actual composition when the spec is implemented.

## Package Structure

**`io.casehub.blocks.agentic.personality`** (additions to existing package)

ARC42STORIES.MD §5 needs updating to include the `personality` sub-package (currently unlisted — PersonalityEvolution from #118 is also missing). This is an implementation task.

| Class | What it does |
|---|---|
| `CivilityConstraint` | `@FunctionalInterface` SPI: `permitInitiation(InitiationContext) → CivilityCheck` |
| `CivilityCheck` | Sealed: `Permitted`, `Denied(reason)` |
| `InitiationContext` | Record: lastInitiationTimestamp, initiationsInWindow, consecutiveInitiationsWithoutResponse, descriptor |
| `MinimumGapConstraint` | Default civility: denied if gap too short |
| `MaxPerWindowConstraint` | Default civility: denied if rate exceeded |
| `ConsecutiveInitiationCooldownConstraint` | Default civility: denied after N unanswered initiations |
| `ContentQualityGate` | Record: novelty threshold, min observations, quiet-period bypass |
| `InnerLifeTick` | Sealed: `Silent(reason)`, `Initiated(content, channelHint, motivationScore)` |
| `InnerLifeOrchestrator` | CDI bean: `observe()`, `observeResponse()`, `tick()` |
| `InnerLifeConfig` | Record with defaults and validation (same pattern as `PersonalityEvolutionConfig`) |
| `MotivationAssessment` | Package-private record: `(double score, String content, String channelHint)` — JSON parse target in tick step 4. Not part of public API; parsed values flow into `InnerLifeTick.Initiated` |
| `TokenJaccardDistance` | Package-private: token-level novelty scoring |

## Testing Strategy

All tests are plain JUnit 5 + Mockito (no Quarkus runtime).

| Test class | Coverage |
|---|---|
| `CivilityConstraintTest` | All three default constraints: gap, rate, cooldown — boundary values, edge cases |
| `ContentQualityGateTest` | Novelty scoring, observation count, quiet-period bypass, combined evaluation |
| `TokenJaccardDistanceTest` | Token-level similarity: identical text, disjoint text, partial overlap, empty input |
| `InnerLifeOrchestratorTest` | Full pipeline: civility denied → Silent, low novelty → Silent, low motivation → Silent, high motivation → Initiated. Mock ReflectionOrchestrator and AgentProvider. |
| `InnerLifeTickTest` | Sealed type exhaustiveness, field access |
| `ObserveAndStateTest` | Observation accumulation, state tracking (lastInitiation, consecutiveWithoutResponse reset on non-self message) |

## References

- `ReflectionOrchestrator` (neocortex) — reflection generation SPI
- `AffordanceRenderer` (blocks/summarisation/observation/affordance) — consumer-side action vocabulary rendering (not composed by InnerLife; output passed via `channelContext`)
- `ActivationRule` / `ActivationContext` (blocks/agentic/activation) — separate domain (D6)
- `PersonalityEvolutionOrchestrator` (blocks/agentic/personality) — established tick() pattern
- `AgentProvider.invoke()` (platform-agent-api) — `invoke(AgentSessionConfig) → Multi<AgentEvent>` — text-collection-and-parse pattern
- `RoutingSupport.invokeAndCollect()` (blocks/routing/agent) — reference implementation of the text-collection-and-parse pattern
- `Watchdog` / `WatchdogConditionType` (qhorus-api) — channel-level alerting (distinct from agent-level civility, D5)
- `AgentDescriptor` (eidos-api) — agent identity and personality record
- Research doc §2.1 — InnerLife pattern description, capability composition
- Research doc §2.8 — System 1/2 fast path fold-in, novelty engine fold-in
- Liu, Fang, Shi, Wu, Igarashi, Chen — "Proactive Conversational Agents with Inner Thoughts" (CHI 2025, arXiv:2501.00383) — 82% user preference for agents with inner thoughts
- Deng et al. — "Human-Centered Proactive Agents: Intelligence, Adaptivity, Civility" (AAAI 2025) — civility constraints taxonomy
- DPT-Agent (2025) — Dual Process Theory with Theory of Mind
