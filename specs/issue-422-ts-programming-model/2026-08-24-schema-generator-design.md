# Model-Canonical Schema Generator — Implementation Design

## Summary

Build a reflection-based generator that walks `io.casehub.api.model.CaseDefinition` and emits JSON Schema, reversing the current source-of-truth direction. Uses victools/jsonschema-generator with custom modules for domain-specific rules. Simultaneously expand YAML coverage to ~15 Java-only fields.

## Context

**Current flow (schema-canonical):**
```
CaseDefinition.yaml (hand-written, 1344 lines)
  → jsonschema2pojo (CasehubCodegen + CasehubRuleFactory)
  → io.casehub.model.* (~40 generated POJOs)
  → CaseDefinitionYamlMapper (1882 lines)
  → io.casehub.api.model.CaseDefinition (hand-written, 1019 lines)
```

**New flow (model-canonical):**
```
io.casehub.api.model.* (hand-written model types)
  → victools/jsonschema-generator (reflection)
  → CaseDefinition.yaml (generated)
```

Two issues: engine#975 (generator + structural equivalence), engine#976 (YAML expansion).

## Decisions

- **D1: Reflection over JavaParser** — Java 17+ reflection handles records, sealed interfaces, generics, Jakarta Validation annotations. Cross-repo types (worker-api, eidos-api, platform-api) resolve naturally from the classpath. No source file management needed.
- **D2: victools/jsonschema-generator** — Mature library for Java → JSON Schema via reflection. Custom modules for the two CasehubRuleFactory domain rules. Handles ~90% out of the box.
- **D3: @JsonPropertyDescription** — Schema `description` values come from Jackson annotations on model type fields. One-time migration from existing hand-written schema descriptions.
- **D4: New generator module** — `engine/generator/` alongside `codegen/`. Parallel run until structural equivalence is proven, then retire codegen.

## Module: engine/generator

### Dependencies

```xml
<dependencies>
  <!-- Reflects on these compiled classes -->
  <dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-api</artifactId>
  </dependency>
  <!-- Generator library -->
  <dependency>
    <groupId>com.github.victools</groupId>
    <artifactId>jsonschema-generator</artifactId>
  </dependency>
  <dependency>
    <groupId>com.github.victools</groupId>
    <artifactId>jsonschema-module-jakarta-validation</artifactId>
  </dependency>
  <dependency>
    <groupId>com.github.victools</groupId>
    <artifactId>jsonschema-module-jackson</artifactId>
  </dependency>
  <!-- Test: structural equivalence + YAML validation -->
  <dependency>
    <groupId>com.networknt</groupId>
    <artifactId>json-schema-validator</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

Maven compiles `api` first (dependency ordering), then `generator` reflects on compiled classes.

### Components

#### CaseHubSchemaGenerator

Main entry point. Configures victools:

- Schema version: Draft 2020-12 (matches existing schema)
- Modules: JakartaValidationModule, JacksonModule, WorkerSchemaModule, CaseCompletionSchemaModule
- Entry type: `CaseDefinition.class`
- Output: YAML file (JSON Schema in YAML format, matching existing convention)

Configuration:
- `unevaluatedProperties: false` as the default for all object types (existing schema convention). Worker overrides with `additionalProperties: true`. `CaseDefinitionSpec` overrides with `unevaluatedProperties: true`. victools defaults to `additionalProperties` — a custom type attribute resolver emits `unevaluatedProperties` instead, matching the existing schema's Draft 2020-12 convention.
- Top-level required fields: `[dsl, namespace, name, version, spec]`
- `$schema`, `$id`, `title`, `description` set on the root schema to match existing values.

#### WorkerSchemaModule

victools custom module. Handles the Worker type reuse rule from `CasehubRuleFactory`.

When the generator encounters `io.casehub.worker.api.Worker`:
- Emits schema with `additionalProperties: true` (Worker is an extension point — `agent:`, `do:`, `mcp:`, `a2a:`, `react:` blocks are plugin-supplied)
- Required: `[name, capabilities]`
- Properties: `name`, `description`, `capabilities` (array of strings), `executionPolicy`, `sequence`, `contextType`, `outputType`

The hand-written `io.casehub.model.Worker` in `schema/src/main/java/` and its `WorkerMarshaller` stay — they serve the generated POJO deserialization path (old flow) during the parallel period.

#### CaseCompletionSchemaModule

victools custom module. Handles the typed additionalProperties rule from `CasehubRuleFactory`.

When the generator encounters `CaseCompletion`:
- Emits `additionalProperties: { $ref: "#/$defs/GoalExpression" }` (not `Map<String, Object>`)
- Properties: `doneWhen` (ExpressionOrOverride)
- Document order = evaluation priority (existing convention)

#### ExpressionOrOverrideModule

victools custom module. When the generator encounters fields typed as `ExpressionEvaluator` or declared with the expression-or-override pattern:
- Emits the `ExpressionOrOverride` schema: `oneOf: [string, object with single-property additionalProperties]`
- Matches the existing `$ref: "#/$defs/ExpressionOrOverride"` usage

#### SpecExtensionModule

The `CaseDefinitionSpec` type uses `unevaluatedProperties: true` (it's an extension point for plugin modules). This module overrides the default `unevaluatedProperties: false` for this specific type.

### Build Integration

- exec-maven-plugin at `generate-resources` phase runs `CaseHubSchemaGenerator`
- Generated schema written to `generator/src/main/resources/schema/CaseDefinition.generated.yaml`
- Generated file committed to git (deterministic output)
- CI: regenerate + `git diff --exit-code` on generated file. Non-zero = model changed without regenerating.

### Structural Equivalence Test

`StructuralEquivalenceTest` compares the generated schema against the existing `schema/src/main/resources/schema/CaseDefinition.yaml`:

**Exclusions:** The existing schema has `_codegenAgent`, `_codegenAgentModel`, `_codegenOpenAi`, etc. — fake properties that force jsonschema2pojo to generate POJO types. These are codegen artifacts with no runtime meaning. The generated schema does not include them. The equivalence test strips `_codegen*` properties before comparison.

1. Parse both as Jackson `JsonNode` trees
2. Walk recursively, comparing:
   - Property names and nesting
   - Type declarations (`type`, `$ref`, `oneOf`, `allOf`, `anyOf`)
   - Required arrays (order-independent)
   - Validation constraints: `pattern`, `minimum`, `maximum`, `minLength`, `maxLength`, `enum`, `minItems`, `uniqueItems`
   - `additionalProperties` / `unevaluatedProperties`
   - `default` values
3. Description text compared after `@JsonPropertyDescription` migration is complete
4. Meaningful diff output on failure — shows the path, expected value, and actual value

### YAML Fixture Validation Test

`GeneratedSchemaValidationTest` runs all existing example YAML files against the generated schema using networknt json-schema-validator. Same approach as the existing `SchemaValidationTest` but loading the generated schema instead of the hand-written one.

## @JsonPropertyDescription Migration

One-time copy of existing schema descriptions onto model type fields. Scope:

- `CaseDefinition` — ~30 fields
- `Binding` — ~20 fields
- `Capability` — ~5 fields
- `Goal` — ~4 fields
- `Milestone` — ~6 fields
- `CaseCompletion` — ~2 fields
- `OutcomePolicy` — ~4 fields
- `SubCase` — ~10 fields
- `HumanTaskTarget` — ~15 fields
- `Trigger` types — ~10 fields
- LLM model types (OpenAI, Anthropic, etc.) — ~40 fields total
- `GoalExpression`, `Use`, `LabelRule`, `InboundSignalMapping` — ~15 fields

Total: ~150 annotations. Mechanical work — copy description text from YAML schema to Java annotation.

## YAML Expansion (engine#976)

### Fields to Add

Fields that exist on `CaseDefinition` Java model but have no YAML schema entry. All are declaratively expressible:

| Field | Type | YAML key |
|-------|------|----------|
| `goapActions` | `List<GoapAction>` | `goapActions:` (without `CostFunction` — static `cost` only) |
| `goalToEffectKeys` | `Map<String, Set<String>>` | `goalToEffectKeys:` |
| `planningConstraints` | `PlanningConstraints` | `planningConstraints:` |
| `monitoringConfig` | `MonitoringConfig` | `monitoring:` |
| `portfolioConfig` | `PortfolioConfig` | `portfolioConfig:` |
| `workerServiceAccountIds` | `Map<String, String>` | `workerServiceAccountIds:` |
| `defaultQuorum` | `QuorumConfig` | `defaultQuorum:` |
| `reflectionTrigger` | `ReflectionTriggerConfig` | `reflection:` |
| `humanTaskContextConstraints` | `List<ContextConstraint>` | `humanTaskConstraints:` |
| `humanTaskWorkloadConstraint` | `WorkloadConstraint` | `humanTaskWorkload:` |
| `maxAdaptations` | `Integer` | `maxAdaptations:` |
| `recoveryPolicy` | `RecoveryPolicy` | `recoveryPolicy:` |
| `memoryRetrieval` | `MemoryRetrievalConfig` | `memoryRetrieval:` |

### Fields That Stay Java-Only

| Field | Why |
|-------|-----|
| `defaultWorkerBridge` | `ContextBridge<?>` — runtime object, not serializable |
| `GoapAction.costFunction` | `CostFunction` — `@FunctionalInterface`, requires code |
| `agentDescriptors` | `AgentDescriptor` from eidos — tracked in eidos#143 |
| `cognitiveDemands` | Already in schema (on `Capability`, not `CaseDefinition`) |

### Per-Field Work

For each field:
1. Add `@JsonPropertyDescription` annotation with description text
2. Update `CaseDefinitionYamlMapper` to parse from YAML raw node
3. Add YAML test fixture exercising the field
4. Verify existing tests still pass (backward compatible)

The generator automatically includes new fields in the generated schema.

## Issue Ordering

1. **engine#975** — Generator module + structural equivalence + @JsonPropertyDescription migration
2. **engine#976** — YAML expansion (incremental, one field at a time)

#975 gates #976: the generator must produce a structurally equivalent schema before expanding it.

## What Stays

- `json-schema-validator` (networknt) — more valuable than before: validates the *generated* schema
- Hand-written `Worker.java` + `WorkerMarshaller.java` in schema module — custom Jackson serialization
- `CaseDefinitionYamlMapper` — remains until the two-class problem is collapsed (longer-term)
- `codegen/` — stays during parallel run, retired after sustained green CI

## What Gets Retired (after equivalence proven)

- `codegen/` module (2 files)
- exec-maven-plugin + build-helper-maven-plugin in `schema/pom.xml`
- ~40 generated `io.casehub.model.*` POJOs (replaced by `io.casehub.api.model.*` as canonical)
- Hand-written `CaseDefinition.yaml` in `schema/src/main/resources/schema/` (replaced by generated)

## References

- `codegen/src/main/java/io/casehub/codegen/CasehubRuleFactory.java` — two domain rules to reproduce
- `codegen/src/main/java/io/casehub/codegen/CasehubCodegen.java` — existing codegen entry point
- `schema/src/main/resources/schema/CaseDefinition.yaml` — existing hand-written schema (1344 lines)
- `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — canonical model (1019 lines)
- `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — YAML mapper (1882 lines)
- `schema/src/test/java/io/casehub/model/SchemaValidationTest.java` — existing schema tests
- `specs/main/2026-08-23-typescript-programming-model-design.md` — roadmap spec (blocks workspace)
- victools/jsonschema-generator: https://github.com/victools/jsonschema-generator
- casehubio/parent#422 — epic
- casehubio/engine#975 — SchemaWriter issue
- casehubio/engine#976 — YAML expansion issue
