# CaseDefinitionModule — Externalized Jackson Deserialization

## Summary

A Jackson `SimpleModule` subclass that registers custom deserializers for all polymorphic types in `CaseDefinition`, enabling direct `objectMapper.readValue()` deserialization from YAML/JSON. Follows protocol PP-20260825-7ad4b1 — no behavior annotations on domain types.

## Context

`CaseDefinitionYamlMapper` currently handles all deserialization manually: reads raw `JsonNode` trees, resolves polymorphic types by inspecting property names, and calls builders/setters. This works but prevents direct Jackson deserialization — which blocks TypeScript round-trip (write YAML in TS, deserialize directly in Java) and makes the `io.casehub.model.*` generated POJOs the only path from YAML to `CaseDefinition`.

The `CaseDefinitionModule` is the first step toward retiring both the generated POJOs and the manual mapping logic in `CaseDefinitionYamlMapper`.

## Module Location

`api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java`

Registered on any `ObjectMapper` that needs to deserialize `CaseDefinition`:
```java
ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
mapper.registerModule(new CaseDefinitionModule());
CaseDefinition def = mapper.readValue(yaml, CaseDefinition.class);
```

## Custom Deserializers

### 1. TriggerDeserializer

Named-property dispatch. Reads the single property key and delegates:

| YAML key | Java type |
|----------|-----------|
| `contextChange:` | `ContextChangeTrigger` |
| `schedule:` | `ScheduleTrigger` |
| `scopeActivated:` | `ScopeActivatedTrigger` |
| `cloudEvent:` | `CloudEventTrigger` |

Error on zero or multiple keys.

### 2. BindingTargetDeserializer

Named-property dispatch within `Binding`. The binding YAML has flat target properties — exactly one of:

| YAML key | Java type |
|----------|-----------|
| `capability:` (string) | `CapabilityTarget` — resolves capability name from `CaseDefinition.capabilities` |
| `subCase:` (object) | `SubCaseTarget` |
| `humanTask:` (object) | `HumanTaskTarget` |
| `signal:` (object) | `SignalTarget` |

`ExtensionTarget` excluded — engine-internal.

### 3. ExpressionEvaluatorDeserializer

Two-form pattern:
- String → `JQExpressionEvaluator(string)` (default language)
- `{lang: expr}` single-entry map → resolved via `ExpressionEngineRegistry` when available, or `JQExpressionEvaluator` when lang is `"jq"`

The definition-level `expressionLang` field affects the default — when set to non-`"jq"`, plain strings use that language instead. The deserializer reads `expressionLang` from the parent context.

### 4. CaseCompletionDeserializer

Two patterns:
- `doneWhen: <expression>` → `PredicateBasedCompletion(ExpressionEvaluator)`
- Goal-kind map → `GoalBasedCompletion`:
  ```yaml
  completion:
    success:
      allOf: [goal-a, goal-b]
    failure: goal-c
  ```
  Keys are goal kind names (`success`, `failure`, or custom). Values are `GoalExpression`.

### 5. GoalExpressionDeserializer

Recursive composition:
- String → `SingleGoalExpression(name)`
- Array of strings → `AllOfGoalExpression` (shorthand)
- `{allOf: [...]}` → `AllOfGoalExpression` with recursive children
- `{anyOf: [...]}` → `AnyOfGoalExpression` with recursive children

Children can be strings or nested `{allOf/anyOf}` objects.

### 6. SubCaseMappingDeserializer

Same two-form pattern as ExpressionEvaluator:
- String → `Expression(JQExpressionEvaluator(string))`
- `{lang: expr}` map → `Expression(resolved evaluator)`

## Mixins

Mixins suppress non-serializable fields without polluting domain types:

```java
@JsonIgnoreProperties({"function"})
abstract class WorkerMixin {}

@JsonIgnoreProperties({"costFunction"})
abstract class GoapActionMixin {}
```

Registered via `setMixIn(Worker.class, WorkerMixin.class)`.

## Property Name Mappings

YAML keys that differ from Java field names. Handled via `@JsonProperty` on mixin abstract methods, or via `setPropertyNamingStrategy` with overrides:

| YAML key | Java field | On type |
|----------|-----------|---------|
| `cbr` | `cbrConfig` | `CaseDefinitionSpec` |
| `adaptation` | `adaptationConfig` | `CaseDefinitionSpec` |
| `reflection` | `reflectionTrigger` | `CaseDefinitionSpec` |
| `monitoring` | `monitoringConfig` | `CaseDefinitionSpec` |
| `on` | trigger (on Binding) | `Binding` |
| `layers` | `layerNames` | `CaseDefinition` |
| `episodic` | `episodicMemoryConfig` | `CaseDefinition` |
| `context` | nested `contextStoreFactory` | `CaseDefinition` |

The `context` → `contextStoreFactory` mapping requires a custom deserializer or `@JsonUnwrapped`-equivalent logic since YAML has `context: { storeFactory: "..." }` but Java has a flat `String contextStoreFactory`.

## Structural Mapping: CaseDefinition ↔ YAML

The YAML structure has identity fields at root and a nested `spec:` block:
```yaml
namespace: acme
name: my-case
version: "1.0.0"
spec:
  capabilities: [...]
  workers: [...]
  bindings: [...]
```

Java has `CaseDefinition` with a `CaseDefinitionSpec spec` field. Jackson handles this naturally — `spec` is a regular field with its own type.

## What the Module Does NOT Handle

- **Validation** — signal name dedup, inbound mapping validation, worker service account type checks. These run post-deserialization.
- **Runtime wiring** — `ContextBridge<T>` creation from `contextType`, expression registry selection from `expressionLang`. These are call-site concerns.
- **DAG/HTN execution types** — `DagPlan`, `TaskNode`, snapshot types. These have their own serialization paths and are out of scope for this iteration.

## Implementation Strategy

Incremental — each deserializer is independently testable:

1. Module skeleton + ExpressionEvaluatorDeserializer (simplest, used everywhere)
2. GoalExpressionDeserializer (recursive, foundation for CaseCompletion)
3. CaseCompletionDeserializer (depends on GoalExpression)
4. TriggerDeserializer (independent)
5. SubCaseMappingDeserializer (same pattern as ExpressionEvaluator)
6. BindingTargetDeserializer (most complex — cross-references capabilities)
7. Mixins + property name mappings
8. Full CaseDefinition round-trip test

## Testing

Each deserializer gets unit tests with:
- Happy path (each variant)
- Error cases (invalid input, missing required fields)
- Round-trip: serialize → deserialize → assert structural equality

Integration test: load existing YAML fixtures via both `CaseDefinitionYamlMapper` (old path) and `CaseDefinitionModule` (new path), assert the resulting `CaseDefinition` instances are structurally equivalent.

## References

- PP-20260825-7ad4b1 — Jackson externalized serialization protocol
- `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — existing manual deserialization (1882+ lines)
- `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — domain type
- `api/src/main/java/io/casehub/api/model/CaseDefinitionSpec.java` — spec type
- D6 in `decisions.md` — scope decision
- casehubio/parent#422 — epic
