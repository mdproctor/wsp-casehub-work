# Governance Protocol Integration Design

**Issue:** casehubio/platform#924
**Branch:** issue-924-governance-protocol-integ
**Date:** 2026-11-22

## Summary

Wire `TemplateExtractor` and `TemplateRegistry` into the governance protocol
runtime so that protocol definitions declare response templates and the runtime
pattern-matches agent prose responses to extract structured parameters
deterministically.

## Architecture

Replace the hollow `PlaybookProtocolEvaluator` skeleton with a template-first
`ProtocolEvaluator`. The evaluator's primary job is matching agent responses
against declared templates.

### Data flow

```
Agent prose response
        │
        ▼
ProtocolEvaluator.evaluate(key, responseText)
        │
        ├── lookup ProtocolInstanceKey → TemplateRegistry
        │
        ▼
TemplateRegistry.match(responseText)
        │
        ├── Match found → Matched(name, params)
        └── No match   → Unmatched(responseText)
```

## Components

All types live in `io.casehub.platform.coordination.protocol` within
`coordination-core`. No new modules.

### New types

#### `ProtocolDefinition` (record)

```java
public record ProtocolDefinition(String name, Map<String, String> templates) {
    // name: protocol identifier (e.g. "design-review-vote")
    // templates: ordered map of template name → template pattern
    //   e.g. {"approve": "I ${decision} this design[ because ${rationale}]",
    //         "escalate": "I escalate to ${target}[ with note ${note}]"}
    // Defensive copy + unmodifiable in canonical constructor
}
```

#### `ProtocolMatchResult` (sealed interface)

```java
public sealed interface ProtocolMatchResult {
    record Matched(String templateName, Map<String, String> params) implements ProtocolMatchResult {}
    record Unmatched(String response) implements ProtocolMatchResult {}
}
```

#### `ProtocolEvaluator` (class, replaces `PlaybookProtocolEvaluator`)

```java
public class ProtocolEvaluator {
    // ConcurrentHashMap<String, TemplateRegistry> — keyed by protocol name
    // Thread-safe (j.u.c, no synchronized — virtual-thread safe)

    public void register(ProtocolDefinition definition)
    // Compiles definition.templates() into a TemplateRegistry via Builder
    // Throws IllegalArgumentException on duplicate protocol name

    public ProtocolMatchResult evaluate(ProtocolInstanceKey key, String response)
    // Looks up TemplateRegistry by key.protocolName()
    // Calls registry.match(response)
    // Returns Matched or Unmatched
    // Throws IllegalArgumentException for unknown protocol name

    public void deregister(String protocolName)
    // Removes protocol registration

    public boolean isRegistered(String protocolName)
    // Check if protocol is registered
}
```

### Retained types (unchanged)

- `ProtocolInstanceKey(protocolName, instanceId)` — identifies a protocol
  instance. Used as the lookup key in `evaluate()`.
- `ProtocolAdvisory(severity, message, suggestedAction)` — downstream advisory
  type. Callers build advisories from `ProtocolMatchResult`.
- `AdvisorySeverity` (INFO/WARNING/CRITICAL) — retained for advisory use.
- `SuggestedAction` (LOG/ESCALATE/INVESTIGATE/REROUTE) — retained for advisory use.

### Deleted types

- `PlaybookProtocolEvaluator` — replaced by `ProtocolEvaluator`.

## Error handling

| Scenario | Behavior |
|----------|----------|
| Duplicate protocol name on register | `IllegalArgumentException` |
| Unknown protocol name on evaluate | `IllegalArgumentException` |
| Invalid template pattern on register | `IllegalArgumentException` (from `TemplateExtractor.compile()`) |
| Duplicate variable name in template | `IllegalArgumentException` (from `TemplateExtractor.compile()`) |
| Response doesn't match any template | `Unmatched(response)` — no exception |

All validation happens at `register()` time (fail-fast). `evaluate()` never
fails due to template issues — only due to unknown protocol names.

## Testing

- `ProtocolDefinitionTest` — constructor validation, defensive copy, immutability
- `ProtocolMatchResultTest` — sealed type coverage, Matched/Unmatched construction
- `ProtocolEvaluatorTest` — the main test class:
  - Register protocol with templates, evaluate matching responses
  - Evaluate with optional segment templates (present and absent)
  - Evaluate with unmatched response → `Unmatched`
  - Multiple protocols registered, correct dispatch by key
  - Duplicate registration rejection
  - Unknown protocol evaluation rejection
  - Thread-safety (concurrent register + evaluate)
  - Deregister and re-register

## Scope boundaries

**In scope:**
- `ProtocolDefinition`, `ProtocolMatchResult`, `ProtocolEvaluator`
- Delete `PlaybookProtocolEvaluator`
- Tests for all new types

**Out of scope:**
- YAML protocol definition parsing (future issue)
- Advisory generation from match results (caller concern)
- CoordinationStateStore integration (the evaluator is stateless)
- CDI wiring / `@ApplicationScoped` producers (not needed in coordination-core)

## References

- `yaml-core/src/main/java/.../resolver/TemplateExtractor.java` — template compilation + extraction
- `yaml-core/src/main/java/.../resolver/TemplateRegistry.java` — ordered multi-template matching
- `coordination-core/src/main/java/.../protocol/PlaybookProtocolEvaluator.java` — skeleton to replace
- casehubio/platform#921 — template extraction primitives
- casehubio/platform#923 — optional segment support
