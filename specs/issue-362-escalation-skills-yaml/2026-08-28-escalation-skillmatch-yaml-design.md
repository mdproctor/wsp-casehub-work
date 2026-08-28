# Escalation and SkillMatch YAML Expansion — Design Spec

**Date:** 2026-08-28
**Status:** Draft
**Issue:** casehubio/work#362
**Parent:** casehubio/parent#422 — TypeScript programming model, Phase 2
**Decisions:** [decisions.md](decisions.md) D1-D3

## Motivation

`@Escalate` and `@SkillMatch` are annotation-only configuration in `casehub-work-annotations`. The parent#422 epic requires YAML parity for all declarative configurations across the platform. These two annotations map naturally to nested objects under the existing `humanTask:` YAML schema — adding them completes the work module's YAML surface.

## YAML Surface (D1)

```yaml
bindings:
  - name: review-binding
    humanTask:
      title: "Review investigation"
      candidateGroups: [senior-reviewers]
      claimDeadlineHours: 4
      escalation:
        onExpiry: "team-leads"
        onClaimDeadline: "managers"
        deadline: "PT4H"
        generateSummary: true
      skillMatch:
        strategy: "semantic"
        requiredCapabilities: ["java", "security-review"]
        minimumScore: 0.7
    on:
      contextChange:
        filter: ".readyForReview == true"
```

Both `escalation` and `skillMatch` are optional. All fields within each are optional — partial configuration is valid (e.g. `escalation:` with only `onExpiry`).

## Changes

### Engine: CaseDefinition.yaml — Schema (D1)

Add to `HumanTask` `$defs`:

```yaml
escalation:
  type: object
  unevaluatedProperties: false
  properties:
    onExpiry:
      type: string
      description: Candidate group to escalate to when the WorkItem expires.
    onClaimDeadline:
      type: string
      description: Candidate group to escalate to when claim deadline passes.
    deadline:
      type: string
      description: ISO-8601 duration for the escalation deadline.
    generateSummary:
      type: boolean
      default: true
      description: Whether to generate an AI summary on escalation.
skillMatch:
  type: object
  unevaluatedProperties: false
  properties:
    strategy:
      type: string
      description: Named WorkerSelectionStrategy (e.g. "semantic", "keyword").
    requiredCapabilities:
      type: array
      items:
        type: string
      description: Capabilities the assigned worker must possess.
    minimumScore:
      type: number
      minimum: 0
      maximum: 1
      description: Minimum skill-match score threshold.
```

### Engine: HumanTaskTarget — Model (D2)

Add two inner records and corresponding fields:

```java
public record EscalationConfig(
    String onExpiry,
    String onClaimDeadline,
    String deadline,
    boolean generateSummary
) {
    public EscalationConfig {
        if (deadline != null && !deadline.isEmpty()) {
            java.time.Duration.parse(deadline); // validate ISO-8601
        }
    }
}

public record SkillMatchConfig(
    String strategy,
    Set<String> requiredCapabilities,
    Double minimumScore
) {
    public SkillMatchConfig {
        if (minimumScore != null && minimumScore != -1.0
                && (minimumScore < 0.0 || minimumScore > 1.0)) {
            throw new IllegalArgumentException(
                "minimumScore must be in [0.0, 1.0] or -1.0, was: " + minimumScore);
        }
    }
}
```

Add fields to `HumanTaskTarget`:
- `private final EscalationConfig escalation;`
- `private final SkillMatchConfig skillMatch;`

Add builder methods:
- `escalation(EscalationConfig)`, `skillMatch(SkillMatchConfig)`

### Engine: BindingDeserializer — Parsing

Add to `deserializeHumanTask()`, following the existing pattern (check `node.has()`, parse nested object, set on builder):

```java
if (node.has("escalation")) {
    JsonNode esc = node.get("escalation");
    b.escalation(new HumanTaskTarget.EscalationConfig(
        textOrNull(esc, "onExpiry"),
        textOrNull(esc, "onClaimDeadline"),
        textOrNull(esc, "deadline"),
        esc.has("generateSummary") ? esc.get("generateSummary").asBoolean() : true
    ));
}
if (node.has("skillMatch")) {
    JsonNode sm = node.get("skillMatch");
    Set<String> caps = new LinkedHashSet<>();
    if (sm.has("requiredCapabilities")) {
        sm.get("requiredCapabilities").forEach(n -> caps.add(n.asText()));
    }
    b.skillMatch(new HumanTaskTarget.SkillMatchConfig(
        textOrNull(sm, "strategy"),
        caps.isEmpty() ? null : caps,
        sm.has("minimumScore") ? sm.get("minimumScore").asDouble() : null
    ));
}
```

### Engine: Generator — BindingTargetModule

If the victools generator is already active (engine#975), the inner records on `HumanTaskTarget` should be picked up automatically by reflection. Verify the generated schema includes the new nested objects. If the generator is not yet active, the hand-written schema change above is sufficient.

### Work: WorkItemCreateRequest — New fields (D3)

Add three fields:

| Field | Type | Maps from |
|-------|------|-----------|
| `escalationOnExpiry` | `String` | `EscalationConfig.onExpiry` |
| `escalationOnClaimDeadline` | `String` | `EscalationConfig.onClaimDeadline` |
| `escalationDeadline` | `String` | `EscalationConfig.deadline` |
| `escalationGenerateSummary` | `Boolean` | `EscalationConfig.generateSummary` |

`escalationDeadline` is the ISO-8601 duration for how long the escalated group has to act after escalation fires — distinct from the WorkItem's own `expiresAt`. `SlaBreachPolicy` reads it when constructing the escalated WorkItem.

Skill-match fields already exist: `routingStrategy`, `requiredCapabilities`, `minimumScore`.

### Work: HumanTaskScheduleHandler — Threading

In both `handleInlineMode` and `handleTemplateMode`, read the new config from `HumanTaskTarget` and pass to the `WorkItemCreateRequest.Builder`:

```java
if (target.escalation() != null) {
    requestBuilder
        .escalationOnExpiry(target.escalation().onExpiry())
        .escalationOnClaimDeadline(target.escalation().onClaimDeadline())
        .escalationDeadline(target.escalation().deadline())
        .escalationGenerateSummary(target.escalation().generateSummary());
}
if (target.skillMatch() != null) {
    requestBuilder
        .routingStrategy(target.skillMatch().strategy())
        .requiredCapabilities(toCsv(target.skillMatch().requiredCapabilities()))
        .minimumScore(target.skillMatch().minimumScore());
}
```

### Work: WorkItemEntity — Persistence

`WorkItemEntity` must store `escalationOnExpiry`, `escalationOnClaimDeadline`, and `escalationGenerateSummary` so `SlaBreachPolicy` can read them at breach time. Check existing columns — if absent, add a Flyway migration (V-number per `docs/FLYWAY.md` reservation procedure).

### Test Fixtures

**Engine tests:**
- `BindingDeserializerTest` — YAML snippet with `escalation:` and `skillMatch:` on humanTask; verify parsed `HumanTaskTarget` carries the config
- `HumanTaskTargetTest` — builder validation (invalid minimumScore, invalid deadline duration)
- Schema validation — YAML fixture validates against updated `CaseDefinition.yaml`

**Work tests:**
- `HumanTaskScheduleHandlerTest` — verify escalation/skillMatch fields flow from `HumanTaskTarget` through to `WorkItemCreateRequest`

### Backward Compatibility

- All new fields are optional with sensible defaults (no escalation, no skill-match)
- Existing YAML fixtures and annotation-based definitions are unaffected
- No breaking changes to `WorkItemCreateRequest` (additive fields only)

## References

- `annotations/runtime/src/main/java/io/casehub/work/annotations/Escalate.java` — annotation definition
- `annotations/runtime/src/main/java/io/casehub/work/annotations/SkillMatch.java` — annotation definition
- `engine/api/src/main/java/io/casehub/api/model/HumanTaskTarget.java` — target model
- `engine/api/src/main/java/io/casehub/api/model/converter/deser/BindingDeserializer.java` — YAML parsing
- `engine/schema/src/main/resources/schema/CaseDefinition.yaml:716` — HumanTask $defs
- `engine-adapter/src/main/java/io/casehub/work/engine/HumanTaskScheduleHandler.java` — adapter wiring
- `api/src/main/java/io/casehub/work/api/WorkItemCreateRequest.java` — create request DTO
- GE-20260511-3e5a75 — SlaBreachPolicy escalation pattern
- GE-20260522-f7db12 — stateless multi-tier escalation via candidateGroups
- casehubio/parent#422 — TypeScript programming model epic
