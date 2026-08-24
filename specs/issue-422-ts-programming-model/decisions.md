## D1: Reflection over JavaParser

**Choice:** Use Java reflection for the model-canonical generator (both SchemaWriter and future TypeScriptWriter)
**Alternatives:**
- JavaParser — parses source files for richer info (Javadoc, builder patterns), but adds complexity (source file paths, CombinedTypeSolver for cross-repo types) without capability reflection doesn't already provide
- Hybrid (JavaParser for engine, reflection fallback for cross-repo) — two code paths for no gain
**Rationale:** Java 17+ reflection handles everything: records via `getRecordComponents()`, sealed interfaces via `getPermittedSubclasses()`, generics via `getGenericType()`, Jakarta Validation annotations (runtime-retained). Cross-repo types resolve naturally from the classpath. Descriptions come from annotations (`@JsonPropertyDescription`), not Javadoc.
**Trade-offs:** No Javadoc access — but descriptions via annotations are explicit, versioned, and don't rot. Acceptable trade.
**Sources:** CaseDefinition.java (1019 lines, references types from 4 repos), CasehubRuleFactory.java (existing codegen), spec 2026-08-23-typescript-programming-model-design.md
**Exploration:** quick
**Status:** captured

## D2: victools/jsonschema-generator as foundation

**Choice:** Use victools/jsonschema-generator library for Java → JSON Schema generation
**Alternatives:**
- Custom reflection walker — full control but reimplements what victools already handles (records, sealed interfaces, generics, Jakarta Validation, Jackson annotations). More code, more edge cases.
**Rationale:** Battle-tested library purpose-built for this. Custom modules for the two CasehubRuleFactory domain rules (Worker type reuse, CaseCompletion typed additionalProperties). Handles ~90% of the work out of the box.
**Trade-offs:** Library dependency. Acceptable — victools is well-maintained and widely used.
**Sources:** victools/jsonschema-generator GitHub, CasehubRuleFactory.java
**Exploration:** quick
**Status:** captured

## D3: @JsonPropertyDescription for schema descriptions

**Choice:** Add `@JsonPropertyDescription` annotations to model type fields to source JSON Schema `description` values
**Alternatives:**
- Companion metadata file — separate YAML/JSON file with descriptions; keeps model clean but creates a second maintenance surface
- Exclude descriptions from equivalence — defer descriptions, test structure only; faster initial delivery but weaker schema output
**Rationale:** Jackson annotation read automatically by victools. Descriptions stay with the code (single source of truth). One-time migration: copy existing hand-written schema descriptions to annotations.
**Trade-offs:** Adds annotations to model types — some noise, but the annotations are the description, not boilerplate.
**Depends on:** D2 (victools reads these annotations natively)
**Sources:** CaseDefinition.yaml (1344 lines of hand-written schema with descriptions), CaseDefinition.java
**Exploration:** quick
**Status:** captured

## D4: New 'generator' module alongside codegen

**Choice:** Create `engine/generator/` as a new Maven module. Both old codegen and new generator run during the parallel validation period.
**Alternatives:**
- Replace codegen in-place — simpler module structure but no parallel run period; must be correct immediately
**Rationale:** Parallel run catches edge cases the structural equivalence test might miss. Clean retirement: delete codegen/ when ready. Generator module depends on api module — Maven compiles api first, generator runs reflection on compiled classes.
**Trade-offs:** Two modules temporarily. Short-lived — codegen is deleted once equivalence is proven.
**Sources:** codegen/ (2 files), schema/pom.xml (exec-maven-plugin integration)
**Exploration:** quick
**Status:** captured
