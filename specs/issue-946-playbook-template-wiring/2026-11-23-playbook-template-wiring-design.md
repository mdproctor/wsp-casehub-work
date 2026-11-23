# Playbook Template Declaration and Executor Wiring

**Issue:** casehubio/platform#946
**Epic:** #943
**Scale:** M | **Complexity:** High

## Overview

Add a `templates:` section to playbook YAML metadata and wire template-based extraction into the playbook executor's event input pipeline. Messages arriving on a channel are matched against declared templates; matched messages become structured events fed to the playbook; unmatched messages pass through as plain conversation.

## YAML Schema

Templates are declared alongside existing metadata sections (`meta`, `params`, `orchestration`, `pools`):

```yaml
templates:
  vote:
    pattern: "I ${decision} this ${subject}[ because ${rationale}]"
    variables:
      decision: { type: enum, values: [approve, reject, abstain] }
  escalate:
    pattern: "Escalating to ${target}[ due to ${reason}]"
```

Each template has:
- `pattern` (required) — template string using `${var}` placeholders and `[optional]` segments
- `variables` (optional) — per-variable constraints (`type: enum, values: [...]` or `type: string, required: true/false`)

Add `templates` to `playbook-document.schema.json` properties. The schema already has `additionalProperties: true`, so templates work immediately; the schema entry adds IDE validation.

## Compilation

### TemplateSpec record (playbook-compiler)

```java
public record TemplateSpec(
    ProtocolDefinition protocolDefinition,
    Map<String, Map<String, VariableConstraint>> variableConstraints
) {}
```

- `protocolDefinition` — `ProtocolDefinition(playbookName, templateMap)` where `templateMap` is `Map<templateName, patternString>`
- `variableConstraints` — `Map<templateName, Map<variableName, VariableConstraint>>` parsed from the `variables:` declarations

### PlaybookCompiler changes

Add `parseTemplates()` following the existing `parseOrchestration()` / `parsePools()` pattern:

```java
if (content.containsKey("templates")) {
    templateSpec = parseTemplates(name, (Map<String, Object>) content.get("templates"));
}
```

`parseTemplates()`:
1. Iterates template entries
2. Extracts `pattern` string per template
3. Parses `variables` map into `VariableConstraint` instances (enum → `VariableConstraint.enumOf(values)`, string → `VariableConstraint.string()`, optional → `.optional()`)
4. Builds `ProtocolDefinition` from the name→pattern map
5. Returns `TemplateSpec`

### CompilationResult extension

Add `TemplateSpec templateSpec` field (nullable — defaults to null for playbooks without templates). Add a backward-compatible constructor.

## Executor Wiring

### TemplateMatchEvent record (yaml-step-runtime)

```java
public record TemplateMatchEvent(
    String type,
    Map<String, String> fields,
    double score,
    String originalInput
) {}
```

- `type` — template name (e.g. "vote", "escalate")
- `fields` — extracted variable values
- `score` — 1.0 for regex match, <1.0 for NLP fallback
- `originalInput` — the raw message text

### TemplateMatchingFilter (yaml-step-runtime)

A filter that sits in the executor's input path. Wraps a `ProtocolEvaluator` and per-template `VariableConstraint` maps.

```java
public final class TemplateMatchingFilter {
    private final ProtocolEvaluator evaluator;
    private final Map<String, Map<String, VariableConstraint>> constraints;

    public Optional<TemplateMatchEvent> match(String input) {
        // 1. Try ProtocolEvaluator.evaluate() with a fixed instance key
        // 2. On Matched: validate extracted params against constraints
        //    via TemplateExtractor.extract(input, constraints.get(templateName))
        // 3. Build TemplateMatchEvent with score 1.0 (regex) or from NLP
        // 4. On Unmatched: return empty
    }
}
```

Constraint enforcement: after `ProtocolEvaluator` returns a regex `Matched`, re-extract using `TemplateExtractor.extract(input, constraintMap)` to validate variable values. If constraint validation fails, the match is rejected and the evaluator continues to the next template / NLP fallback.

### PlaybookRunnerCore changes

1. Accept optional `NlpProtocolMatcher` in constructor (nullable)
2. After compilation, if `TemplateSpec` is present:
   - Create `ProtocolEvaluator` with optional NLP matcher
   - Register the `ProtocolDefinition`
   - Create `TemplateMatchingFilter` with the evaluator and constraints
3. Make the filter available to steps that consume channel messages
4. Expose the filter via `PlaybookRunConfig` or as a runner-scoped service

### Integration with ProtocolEvaluator

`ProtocolEvaluator` from #945 already supports:
- Regex matching via `TemplateRegistry`
- NLP fallback via optional `NlpProtocolMatcher`
- Keyword-scoped candidate selection via `TemplateKeywords`

The templates compiled from YAML produce a `ProtocolDefinition` that registers directly with `ProtocolEvaluator`. No new matching logic needed — the existing two-tier matching handles everything.

### NLP activation

`PlaybookRunnerCore` accepts an optional `NlpProtocolMatcher` via constructor injection. When present, the `ProtocolEvaluator` gets NLP fallback capability. When absent (no message-extraction module on classpath), regex-only matching. The CDI/DI layer handles wiring — no classpath detection magic in the runner.

## Module dependency changes

- **playbook-compiler** already depends on `coordination-core` (for `CoordinationSpec`). Adding `ProtocolDefinition` import costs nothing.
- **yaml-step-runtime** already depends on `yaml-core` (for `TemplateExtractor`, `VariableConstraint`). Adding `coordination-core` for `ProtocolEvaluator` is new but follows the existing dependency direction.

## Testing strategy

1. **TemplateSpec parsing** — unit tests in playbook-compiler: YAML template map → TemplateSpec with correct ProtocolDefinition and constraints
2. **TemplateMatchingFilter** — unit tests in yaml-step-runtime: regex match, NLP fallback, constraint enforcement, unmatched pass-through
3. **Integration** — playbook YAML with templates → compile → run with mock channel input → verify TemplateMatchEvent produced
4. **Constraint validation** — enum constraint rejects invalid values, required constraint rejects missing optional-segment variables

## Files changed

| Module | File | Change |
|--------|------|--------|
| yaml-core | `schema/playbook-document.schema.json` | Add `templates` property |
| playbook-compiler | `TemplateSpec.java` | New record |
| playbook-compiler | `PlaybookCompiler.java` | Add `parseTemplates()`, extend compile() |
| playbook-compiler | `CompilationResult.java` | Add `templateSpec` field |
| yaml-step-runtime | `TemplateMatchEvent.java` | New record |
| yaml-step-runtime | `TemplateMatchingFilter.java` | New class |
| yaml-step-runtime | `PlaybookRunnerCore.java` | Accept NlpProtocolMatcher, create filter from TemplateSpec |
| playbook-compiler | `PlaybookCompilerTest.java` | Template parsing tests |
| yaml-step-runtime | `TemplateMatchingFilterTest.java` | Matching + constraint tests |

## References

- `ProtocolDefinition.java` (coordination-core) — template container record
- `ProtocolEvaluator.java` (coordination-core) — two-tier matching engine (#945)
- `VariableConstraint.java` (yaml-core) — enum/string constraints (#944)
- `TemplateExtractor.java` (yaml-core) — regex extraction with constraints (#944)
- `PlaybookCompiler.java` (playbook-compiler) — compilation pipeline
- `PlaybookRunnerCore.java` (yaml-step-runtime) — executor entry point
- `playbook-document.schema.json` (yaml-core) — YAML schema
- `CompilationResult.java` (playbook-compiler) — compilation output record
