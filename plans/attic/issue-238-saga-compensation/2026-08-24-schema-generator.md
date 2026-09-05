# Schema Generator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #975 — JavaParser SchemaWriter — model-canonical JSON Schema generation
**Issue group:** #975, #976

**Goal:** Build a reflection-based generator that produces JSON Schema from `CaseDefinition.class`, then expand YAML coverage to ~15 Java-only fields.

**Architecture:** New `engine/generator/` Maven module uses victools/jsonschema-generator to reflect on compiled `io.casehub.api.model.CaseDefinition` and emit Draft 2020-12 JSON Schema. Six custom victools modules handle domain-specific patterns (Worker extension point, CaseCompletion typed map, expression evaluator pattern, trigger/binding-target named-property `oneOf`, spec nesting, `unevaluatedProperties`). Structural equivalence test gates the build.

**Tech Stack:** victools/jsonschema-generator 4.37+, JakartaValidationModule, JacksonModule, networknt json-schema-validator (test), Jackson YAML (output)

## Global Constraints

- Draft 2020-12 JSON Schema (matches existing `CaseDefinition.yaml`)
- Generated schema committed to git; CI runs `git diff --exit-code`
- `unevaluatedProperties: false` default for all objects (not `additionalProperties`)
- Worker uses `additionalProperties: true` (plugin extension point)
- `_codegen*` properties excluded from equivalence comparison
- Cross-repo renames (worker-api `capabilities` → `capabilities`, `inputProjection` → `inputProjection`) tracked as separate issues — custom modules bridge the gap until those land

---

## Batch 1: Generator Module + Core Custom Modules

### Task 1: Generator Module Scaffold + Basic Schema Output

**Files:**
- Create: `generator/pom.xml`
- Create: `generator/src/main/java/io/casehub/generator/CaseHubSchemaGenerator.java`
- Create: `generator/src/test/java/io/casehub/generator/BasicSchemaGenerationTest.java`
- Modify: `pom.xml` (root) — add `<module>generator</module>`

**Interfaces:**
- Produces: `CaseHubSchemaGenerator.generate(Class<?> rootType): JsonNode` — generates JSON Schema from a Java type
- Produces: `CaseHubSchemaGenerator.generateToYaml(Class<?> rootType, Path output)` — writes YAML-formatted JSON Schema to file

- [ ] **Step 1: Write failing test — basic schema generation**

```java
package io.casehub.generator;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.JsonNode;
import org.junit.jupiter.api.Test;

class BasicSchemaGenerationTest {

    @Test
    void generate_caseDefinition_producesRootSchema() {
        var generator = new CaseHubSchemaGenerator();
        JsonNode schema = generator.generate(
            io.casehub.api.model.CaseDefinition.class);

        assertNotNull(schema);
        assertEquals("object", schema.get("type").asText());
        assertTrue(schema.has("properties"));
        assertTrue(schema.get("properties").has("namespace"));
        assertTrue(schema.get("properties").has("name"));
        assertTrue(schema.get("properties").has("version"));
    }
}
```

- [ ] **Step 2: Create `generator/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-engine-generator</artifactId>
    <name>CaseHub Engine :: Generator</name>
    <properties>
        <maven.deploy.skip>true</maven.deploy.skip>
        <version.victools.jsonschema>4.37.0</version.victools.jsonschema>
    </properties>
    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.github.victools</groupId>
            <artifactId>jsonschema-generator</artifactId>
            <version>${version.victools.jsonschema}</version>
        </dependency>
        <dependency>
            <groupId>com.github.victools</groupId>
            <artifactId>jsonschema-module-jakarta-validation</artifactId>
            <version>${version.victools.jsonschema}</version>
        </dependency>
        <dependency>
            <groupId>com.github.victools</groupId>
            <artifactId>jsonschema-module-jackson</artifactId>
            <version>${version.victools.jsonschema}</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
        </dependency>
        <!-- Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.networknt</groupId>
            <artifactId>json-schema-validator</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 3: Add module to root pom.xml**

Add `<module>generator</module>` to the `<modules>` section in the root `pom.xml`. Place it after `schema` (the generator depends on api, which compiles before it).

- [ ] **Step 4: Implement CaseHubSchemaGenerator**

```java
package io.casehub.generator;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.fasterxml.jackson.dataformat.yaml.YAMLGenerator;
import com.github.victools.jsonschema.generator.*;
import com.github.victools.jsonschema.module.jackson.JacksonModule;
import com.github.victools.jsonschema.module.jackson.JacksonOption;
import com.github.victools.jsonschema.module.jakarta.validation.JakartaValidationModule;
import com.github.victools.jsonschema.module.jakarta.validation.JakartaValidationOption;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class CaseHubSchemaGenerator {

    private final SchemaGenerator schemaGenerator;

    public CaseHubSchemaGenerator() {
        SchemaGeneratorConfigBuilder configBuilder = new SchemaGeneratorConfigBuilder(
            SchemaVersion.DRAFT_2020_12, OptionPreset.PLAIN_JSON);

        configBuilder.with(Option.DEFINITIONS_FOR_ALL_OBJECTS);
        configBuilder.with(Option.FLATTENED_ENUMS_FROM_TOSTRING);

        configBuilder.with(new JakartaValidationModule(
            JakartaValidationOption.INCLUDE_PATTERN_EXPRESSIONS,
            JakartaValidationOption.NOT_NULLABLE_FIELD_IS_REQUIRED));
        configBuilder.with(new JacksonModule(
            JacksonOption.RESPECT_JSONPROPERTY_ORDER));

        this.schemaGenerator = new SchemaGenerator(configBuilder.build());
    }

    public JsonNode generate(Class<?> rootType) {
        return schemaGenerator.generateSchema(rootType);
    }

    public void generateToYaml(Class<?> rootType, Path output) throws IOException {
        JsonNode schema = generate(rootType);
        ObjectMapper yamlMapper = new ObjectMapper(
            new YAMLFactory()
                .disable(YAMLGenerator.Feature.WRITE_DOC_START_MARKER)
                .enable(YAMLGenerator.Feature.MINIMIZE_QUOTES));
        Files.createDirectories(output.getParent());
        yamlMapper.writerWithDefaultPrettyPrinter().writeValue(output.toFile(), schema);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl generator -Dtest=BasicSchemaGenerationTest -q`
Expected: PASS — basic schema generated with `namespace`, `name`, `version` properties.

- [ ] **Step 6: Commit**

```bash
git add generator/ pom.xml
git commit -m "feat(#975): scaffold generator module with victools Refs #975"
```

### Task 2: Core Custom Modules — SpecNesting, UnevaluatedProperties, Worker, CaseCompletion

**Files:**
- Create: `generator/src/main/java/io/casehub/generator/module/SpecNestingModule.java`
- Create: `generator/src/main/java/io/casehub/generator/module/UnevaluatedPropertiesModule.java`
- Create: `generator/src/main/java/io/casehub/generator/module/WorkerSchemaModule.java`
- Create: `generator/src/main/java/io/casehub/generator/module/CaseCompletionSchemaModule.java`
- Modify: `generator/src/main/java/io/casehub/generator/CaseHubSchemaGenerator.java` — register modules
- Create: `generator/src/test/java/io/casehub/generator/module/SpecNestingModuleTest.java`
- Create: `generator/src/test/java/io/casehub/generator/module/WorkerSchemaModuleTest.java`
- Create: `generator/src/test/java/io/casehub/generator/module/CaseCompletionSchemaModuleTest.java`

**Interfaces:**
- Consumes: `CaseHubSchemaGenerator.generate(Class<?>)` from Task 1
- Produces: All four modules registered; generated schema has `spec:` nesting, `unevaluatedProperties`, Worker with `additionalProperties: true`, CaseCompletion with typed `additionalProperties`

- [ ] **Step 1: Write failing test — spec nesting**

```java
package io.casehub.generator.module;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.generator.CaseHubSchemaGenerator;
import org.junit.jupiter.api.Test;

class SpecNestingModuleTest {

    @Test
    void generate_nestedCapabilitiesUnderSpec() {
        var generator = new CaseHubSchemaGenerator();
        JsonNode schema = generator.generate(
            io.casehub.api.model.CaseDefinition.class);

        // Capabilities should be nested under spec, not at root
        assertFalse(schema.get("properties").has("capabilities"),
            "capabilities should not be at root level");

        JsonNode specProps = schema.path("properties").path("spec")
            .path("properties");
        assertTrue(specProps.has("capabilities"),
            "capabilities should be under spec");
        assertTrue(specProps.has("workers"),
            "workers should be under spec");
        assertTrue(specProps.has("bindings"),
            "bindings should be under spec");
    }
}
```

- [ ] **Step 2: Write failing test — Worker schema**

```java
package io.casehub.generator.module;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.generator.CaseHubSchemaGenerator;
import org.junit.jupiter.api.Test;

class WorkerSchemaModuleTest {

    @Test
    void workerSchema_hasAdditionalPropertiesTrue() {
        var generator = new CaseHubSchemaGenerator();
        JsonNode schema = generator.generate(
            io.casehub.api.model.CaseDefinition.class);

        JsonNode workerDef = schema.path("$defs").path("Worker");
        if (workerDef.isMissingNode()) {
            workerDef = schema.path("$defs").path("io.casehub.worker.api.Worker");
        }
        assertFalse(workerDef.isMissingNode(), "Worker should be in $defs");
        assertTrue(workerDef.path("additionalProperties").asBoolean(),
            "Worker should have additionalProperties: true");
    }

    @Test
    void workerSchema_hasCapabilitiesNotCapabilityNames() {
        var generator = new CaseHubSchemaGenerator();
        JsonNode schema = generator.generate(
            io.casehub.api.model.CaseDefinition.class);

        JsonNode workerDef = schema.path("$defs").path("Worker");
        if (workerDef.isMissingNode()) {
            workerDef = schema.path("$defs").path("io.casehub.worker.api.Worker");
        }
        JsonNode workerProps = workerDef.path("properties");
        assertTrue(workerProps.has("capabilities"),
            "Worker should have 'capabilities' property");
        assertFalse(workerProps.has("capabilityNames"),
            "Worker should NOT have 'capabilityNames'");
        assertFalse(workerProps.has("function"),
            "Worker should NOT have 'function'");
    }
}
```

- [ ] **Step 3: Write failing test — CaseCompletion schema**

```java
package io.casehub.generator.module;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.generator.CaseHubSchemaGenerator;
import org.junit.jupiter.api.Test;

class CaseCompletionSchemaModuleTest {

    @Test
    void caseCompletionSchema_hasTypedAdditionalProperties() {
        var generator = new CaseHubSchemaGenerator();
        JsonNode schema = generator.generate(
            io.casehub.api.model.CaseDefinition.class);

        JsonNode completionDef = schema.path("$defs").path("CaseCompletion");
        assertFalse(completionDef.isMissingNode(),
            "CaseCompletion should be in $defs");

        JsonNode additionalProps = completionDef.path("additionalProperties");
        assertTrue(additionalProps.has("$ref"),
            "CaseCompletion additionalProperties should be a $ref to GoalExpression");
    }
}
```

- [ ] **Step 4: Implement SpecNestingModule**

This module intercepts `CaseDefinition` and reorganizes its properties: identity fields stay at root, spec-level fields move under a `spec` property. Implementation uses victools `forTypesInGeneral()` with a custom `TypeAttributeOverrideV2` that post-processes the `CaseDefinition` schema node.

The list of spec-level fields matches the existing `CaseDefinitionSpec` in the YAML schema: `capabilities`, `workers`, `bindings`, `milestones`, `goals`, `completion`, `planningStrategy`, `decompositionStrategy`, `maxDecompositionDepth`, `agentRouting`, `implementationRouting`, `humanTaskRouting`, `candidateMatching`, `routingSignalWeights`, `cbr`, `channels`, `authorization`.

- [ ] **Step 5: Implement UnevaluatedPropertiesModule**

Replaces `additionalProperties: false` with `unevaluatedProperties: false` on all object schemas (Draft 2020-12 convention). Worker and `spec` override this.

- [ ] **Step 6: Implement WorkerSchemaModule**

When victools encounters `io.casehub.worker.api.Worker`, replace the entire schema with a hand-crafted definition matching the existing YAML schema structure: `name`, `description`, `capabilities` (array of strings), `executionPolicy`, `sequence`, `contextType`, `outputType`, `additionalProperties: true`. Required: `[name, capabilities]`.

- [ ] **Step 7: Implement CaseCompletionSchemaModule**

When victools encounters `io.casehub.api.model.CaseCompletion` (empty marker interface), replace with: `properties: { doneWhen: ExpressionOrOverride }`, `additionalProperties: { $ref: GoalExpression }`.

- [ ] **Step 8: Register all modules in CaseHubSchemaGenerator**

Add the four modules to the `SchemaGeneratorConfigBuilder` in `CaseHubSchemaGenerator`.

- [ ] **Step 9: Run all tests**

Run: `mvn test -pl generator -q`
Expected: All tests PASS.

- [ ] **Step 10: Commit**

```bash
git add generator/
git commit -m "feat(#975): core custom modules — spec nesting, worker, completion Refs #975"
```

---

## Batch 2: Type Mapping Modules + Structural Equivalence

### Task 3: ExpressionEvaluator, Trigger, and BindingTarget Modules

**Files:**
- Create: `generator/src/main/java/io/casehub/generator/module/ExpressionEvaluatorModule.java`
- Create: `generator/src/main/java/io/casehub/generator/module/TriggerModule.java`
- Create: `generator/src/main/java/io/casehub/generator/module/BindingTargetModule.java`
- Modify: `generator/src/main/java/io/casehub/generator/CaseHubSchemaGenerator.java` — register modules
- Create: `generator/src/test/java/io/casehub/generator/module/TypeMappingModulesTest.java`

**Interfaces:**
- Consumes: `CaseHubSchemaGenerator` with core modules from Task 2
- Produces: Generated schema has `ExpressionOrOverride` pattern for evaluator fields, `Trigger` as named-property `oneOf`, `BindingTarget` as named-property `oneOf`

- [ ] **Step 1: Write failing tests — ExpressionEvaluator, Trigger, BindingTarget**

```java
package io.casehub.generator.module;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.generator.CaseHubSchemaGenerator;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

class TypeMappingModulesTest {

    private static JsonNode schema;

    @BeforeAll
    static void generateSchema() {
        schema = new CaseHubSchemaGenerator().generate(
            io.casehub.api.model.CaseDefinition.class);
    }

    @Test
    void goalCondition_usesExpressionOrOverride() {
        JsonNode goalDef = schema.path("$defs").path("Goal");
        assertFalse(goalDef.isMissingNode());
        JsonNode conditionProp = goalDef.path("properties").path("condition");
        // Should be a $ref to ExpressionOrOverride or inline oneOf
        assertTrue(conditionProp.has("$ref") || conditionProp.has("oneOf"),
            "Goal.condition should use ExpressionOrOverride pattern");
    }

    @Test
    void trigger_hasNamedPropertyOneOf() {
        JsonNode triggerDef = schema.path("$defs").path("Trigger");
        assertFalse(triggerDef.isMissingNode());
        assertTrue(triggerDef.has("oneOf"),
            "Trigger should have oneOf");
        JsonNode props = triggerDef.path("properties");
        assertTrue(props.has("contextChange"),
            "Trigger should have contextChange property");
        assertTrue(props.has("schedule"),
            "Trigger should have schedule property");
    }

    @Test
    void binding_hasTargetOneOf() {
        JsonNode bindingDef = schema.path("$defs").path("Binding");
        assertFalse(bindingDef.isMissingNode());
        assertTrue(bindingDef.has("oneOf"),
            "Binding should have oneOf for target types");
        JsonNode props = bindingDef.path("properties");
        assertTrue(props.has("capability"),
            "Binding should have capability property");
        assertTrue(props.has("subCase"),
            "Binding should have subCase property");
        assertTrue(props.has("humanTask"),
            "Binding should have humanTask property");
        assertFalse(props.has("target"),
            "Binding should NOT have raw 'target' field");
    }
}
```

- [ ] **Step 2: Implement ExpressionEvaluatorModule**

When victools encounters a field typed as `ExpressionEvaluator` (or its implementations like `JQExpressionEvaluator`), emit the `ExpressionOrOverride` pattern: `oneOf: [{ type: string }, { type: object, minProperties: 1, maxProperties: 1, additionalProperties: { type: string } }]`. Define `ExpressionOrOverride` once in `$defs` and `$ref` to it.

- [ ] **Step 3: Implement TriggerModule**

When victools encounters `io.casehub.api.model.Trigger` (marker interface), replace with named-property `oneOf`: `contextChange` → `ContextChangeTrigger` schema, `cloudEvent` → `CloudEventTrigger` schema, `schedule` → `ScheduleTrigger` schema, `scopeActivated` → `ScopeActivatedTrigger` schema. The `oneOf` requires exactly one of these properties.

- [ ] **Step 4: Implement BindingTargetModule**

When victools encounters `io.casehub.api.model.BindingTarget` (sealed interface), replace the single `target` field in `Binding` with four flat properties: `capability` (string), `subCase` ($ref), `humanTask` ($ref), `signal` (object). The `oneOf` requires exactly one. `ExtensionTarget` is excluded (engine-internal).

- [ ] **Step 5: Register modules and run tests**

Run: `mvn test -pl generator -q`
Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git add generator/
git commit -m "feat(#975): type mapping modules — expression, trigger, binding target Refs #975"
```

### Task 4: @JsonPropertyDescription Migration + Structural Equivalence Test

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add `@JsonPropertyDescription` to fields
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java` — add descriptions
- Modify: `api/src/main/java/io/casehub/api/model/Goal.java` — add descriptions
- Modify: `api/src/main/java/io/casehub/api/model/Milestone.java` — add descriptions
- Modify: Multiple model type files — add descriptions (mechanical, copy from existing schema)
- Create: `generator/src/test/java/io/casehub/generator/StructuralEquivalenceTest.java`
- Create: `generator/src/test/java/io/casehub/generator/GeneratedSchemaValidationTest.java`
- Create: `generator/src/test/java/io/casehub/generator/SchemaComparator.java` (test utility)

**Interfaces:**
- Consumes: `CaseHubSchemaGenerator` with all modules from Tasks 1-3
- Produces: Structural equivalence test comparing generated vs existing schema; YAML fixture validation test

- [ ] **Step 1: Write failing test — structural equivalence**

```java
package io.casehub.generator;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import java.io.InputStream;
import org.junit.jupiter.api.Test;

class StructuralEquivalenceTest {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());
    private static final ObjectMapper JSON = new ObjectMapper();

    @Test
    void generatedSchema_structurallyEquivalentToExisting() throws Exception {
        // Load existing hand-written schema
        try (InputStream is = getClass().getClassLoader()
                .getResourceAsStream("schema/CaseDefinition.yaml")) {
            JsonNode existing = JSON.readTree(
                JSON.writeValueAsString(YAML.readTree(is)));

            // Generate new schema
            JsonNode generated = new CaseHubSchemaGenerator().generate(
                io.casehub.api.model.CaseDefinition.class);

            // Compare structurally (strips _codegen* properties)
            var result = SchemaComparator.compare(existing, generated);
            assertTrue(result.isEquivalent(),
                () -> "Structural differences found:\n" + result.differences());
        }
    }
}
```

- [ ] **Step 2: Write SchemaComparator utility**

```java
package io.casehub.generator;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import java.util.*;

public class SchemaComparator {

    public record Result(boolean isEquivalent, String differences) {}

    public static Result compare(JsonNode expected, JsonNode actual) {
        var diffs = new ArrayList<String>();
        ObjectNode cleanExpected = stripCodegenProperties(expected.deepCopy());
        compareNodes("", cleanExpected, actual, diffs);
        return new Result(diffs.isEmpty(), String.join("\n", diffs));
    }

    private static ObjectNode stripCodegenProperties(JsonNode node) {
        // Remove _codegen* properties from spec.properties
        // Recursive implementation walks the tree
        if (node.isObject()) {
            ObjectNode obj = (ObjectNode) node;
            List<String> toRemove = new ArrayList<>();
            obj.fieldNames().forEachRemaining(name -> {
                if (name.startsWith("_codegen")) toRemove.add(name);
            });
            toRemove.forEach(obj::remove);
            obj.fields().forEachRemaining(
                e -> stripCodegenProperties(e.getValue()));
        } else if (node.isArray()) {
            node.forEach(SchemaComparator::stripCodegenProperties);
        }
        return (ObjectNode) node;
    }

    private static void compareNodes(String path, JsonNode expected,
            JsonNode actual, List<String> diffs) {
        // Compare type, properties, required, oneOf, $ref,
        // validation constraints, additionalProperties/unevaluatedProperties
        // Skip description comparison initially (add later)
        // Recursive tree walk with meaningful diff output
        // Implementation: ~100 lines of recursive comparison
    }
}
```

- [ ] **Step 3: Write failing test — YAML fixture validation**

```java
package io.casehub.generator;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import com.networknt.schema.*;
import java.nio.file.*;
import java.util.Set;
import java.util.stream.Stream;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;

class GeneratedSchemaValidationTest {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());
    private static final ObjectMapper JSON = new ObjectMapper();

    static Stream<Path> exampleFiles() throws Exception {
        Path examplesDir = Path.of("../schema/src/main/resources/examples");
        if (!Files.exists(examplesDir)) return Stream.empty();
        return Files.list(examplesDir)
            .filter(p -> p.toString().endsWith(".yaml"));
    }

    @ParameterizedTest
    @MethodSource("exampleFiles")
    void existingYamlExamples_validateAgainstGeneratedSchema(Path yamlFile)
            throws Exception {
        JsonNode generatedSchema = new CaseHubSchemaGenerator().generate(
            io.casehub.api.model.CaseDefinition.class);
        JsonNode schemaAsJson = JSON.readTree(
            JSON.writeValueAsString(generatedSchema));

        JsonSchemaFactory factory = JsonSchemaFactory.getInstance(
            SpecVersion.VersionFlag.V202012);
        JsonSchema schema = factory.getSchema(schemaAsJson);

        JsonNode yamlNode = YAML.readTree(Files.readString(yamlFile));
        JsonNode jsonNode = JSON.readTree(JSON.writeValueAsString(yamlNode));
        Set<ValidationMessage> errors = schema.validate(jsonNode);

        assertTrue(errors.isEmpty(),
            () -> "Generated schema validation failed for "
                + yamlFile.getFileName() + ":\n" + errors);
    }
}
```

- [ ] **Step 4: Add @JsonPropertyDescription annotations to model types**

Copy description text from the existing `CaseDefinition.yaml` schema to `@JsonPropertyDescription` annotations on the corresponding Java fields/getters. Start with `CaseDefinition` fields (~30), then `Binding` (~20), `Capability` (~5), `Goal` (~4), `Milestone` (~6), and remaining types. Use `ide_edit_member` for each field.

This is mechanical work — each annotation looks like:
```java
@com.fasterxml.jackson.annotation.JsonPropertyDescription(
    "The CaseHub's namespace.")
private final String namespace;
```

- [ ] **Step 5: Implement full SchemaComparator and iterate until equivalence**

Complete the `SchemaComparator.compareNodes()` implementation. Run the structural equivalence test, fix generator modules based on differences found. Iterate until the test passes or all remaining differences are documented as known gaps (e.g., worker-api field name mismatches pending cross-repo renames).

- [ ] **Step 6: Run all tests**

Run: `mvn test -pl generator -q`
Expected: Structural equivalence passes (with documented exclusions for worker-api field names). YAML fixtures validate.

- [ ] **Step 7: Commit**

```bash
git add generator/ api/
git commit -m "feat(#975): structural equivalence + descriptions migration Refs #975"
```

---

## Batch 3: YAML Expansion (engine#976)

### Task 5: CaseDefinition-Level Field Expansion

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — parse new YAML fields
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` — add new fields (parallel run: both hand-written and generated schemas include them)
- Create: `schema/src/main/resources/examples/planning-config-example.yaml` — new example fixture
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` — new test cases

**Interfaces:**
- Consumes: `CaseHubSchemaGenerator` with all modules (new fields appear automatically in generated schema)
- Produces: YAML support for `goapActions`, `planningConstraints`, `monitoringConfig`, `portfolioConfig`, `workerServiceAccountIds`, `defaultQuorum`, `reflectionTrigger`, `adaptationConfig`, `recoveryPolicy`, `memoryRetrieval`, `maxAdaptations`, `humanTaskContextConstraints`, `humanTaskWorkloadConstraint`

- [ ] **Step 1: Write failing tests — YAML mapper parses new fields**

For each new field, write a test that:
1. Defines a YAML string with the new field
2. Parses it via `CaseDefinitionYamlMapper`
3. Asserts the field is set on the resulting `CaseDefinition`

Example for `planningConstraints`:
```java
@Test
void parsesPlanningConstraints() throws Exception {
    String yaml = """
        dsl: "0.1.0"
        namespace: test
        name: constraints-test
        version: "1.0.0"
        spec:
          planningConstraints:
            timeBudget: PT30M
            resourceLimit: 5
            costBudgets:
              tokens: 10000
          capabilities:
            - name: process
          bindings:
            - name: trigger
              capability: process
              on:
                contextChange: {}
        """;
    CaseDefinition def = parseYaml(yaml);
    assertNotNull(def.getPlanningConstraints());
    assertEquals(Duration.ofMinutes(30),
        def.getPlanningConstraints().timeBudget());
    assertEquals(5, def.getPlanningConstraints().resourceLimit());
}
```

Repeat for each field listed in the spec's YAML expansion table.

- [ ] **Step 2: Implement CaseDefinitionYamlMapper parsing for each field**

For each field, add parsing logic in `CaseDefinitionYamlMapper`. Follow existing patterns — read from the `spec` raw node, deserialize, call the setter. Example:

```java
// In the spec-parsing section of CaseDefinitionYamlMapper
JsonNode planningConstraintsNode = specNode.get("planningConstraints");
if (planningConstraintsNode != null) {
    PlanningConstraints pc = mapper.treeToValue(
        planningConstraintsNode, PlanningConstraints.class);
    definition.setPlanningConstraints(pc);
}
```

- [ ] **Step 3: Add @JsonPropertyDescription to new fields**

Add description annotations for each new field on `CaseDefinition`.

- [ ] **Step 4: Add new fields to hand-written schema (parallel run)**

Add the new fields to `schema/src/main/resources/schema/CaseDefinition.yaml` under the `CaseDefinitionSpec` section. This keeps both schemas in sync during the parallel run.

- [ ] **Step 5: Create YAML example fixture**

Create `planning-config-example.yaml` that exercises the new fields. Verify it passes both the existing `SchemaValidationTest` and the new `GeneratedSchemaValidationTest`.

- [ ] **Step 6: Run all tests**

Run: `mvn test -pl schema,api,generator -q`
Expected: All tests PASS — new fields parse from YAML, validate against both schemas.

- [ ] **Step 7: Commit**

```bash
git add api/ schema/ generator/
git commit -m "feat(#976): YAML expansion — planning, monitoring, recovery fields Refs #976"
```

### Task 6: Per-Worker/Per-Capability Annotation Fields

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` — add `cost:`, `effect:`, `softDependency:`, `customize:` fields
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — parse annotation fields from YAML
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` — test cases
- Create: `schema/src/main/resources/examples/goap-yaml-example.yaml` — GOAP worker example

**Interfaces:**
- Consumes: `CaseDefinitionYamlMapper` from Task 5
- Produces: YAML support for `cost:`, `effect:`, `softDependency:` on worker/capability definitions, `customize:` on case definition

- [ ] **Step 1: Write failing tests — per-worker annotation fields**

```java
@Test
void parsesGoapActionFieldsFromWorkerYaml() throws Exception {
    String yaml = """
        dsl: "0.1.0"
        namespace: test
        name: goap-test
        version: "1.0.0"
        spec:
          capabilities:
            - name: verify
          workers:
            - name: verifier
              capabilities: [verify]
              cost: 2.5
              effect:
                verified: true
              softDependency:
                - dataLoaded
          bindings:
            - name: trigger
              capability: verify
              on:
                contextChange: {}
        """;
    CaseDefinition def = parseYaml(yaml);
    var actions = def.getGoapActions();
    assertNotNull(actions);
    assertEquals(1, actions.size());
    assertEquals(2.5, actions.get(0).cost(), 0.001);
    assertTrue(actions.get(0).effects().get("verified"));
}

@Test
void parsesCustomizeField() throws Exception {
    String yaml = """
        dsl: "0.1.0"
        namespace: test
        name: customize-test
        version: "1.0.0"
        spec:
          customize:
            - addLogging
            - addMetrics
          capabilities:
            - name: process
          bindings:
            - name: trigger
              capability: process
              on:
                contextChange: {}
        """;
    CaseDefinition def = parseYaml(yaml);
    // Customizer IDs should be parsed and available
    // (exact API depends on how customize is stored on CaseDefinition)
}
```

- [ ] **Step 2: Implement per-worker annotation field parsing in CaseDefinitionYamlMapper**

In the worker-parsing section of `CaseDefinitionYamlMapper`, read `cost`, `effect`, `softDependency` from each worker's raw YAML node. Construct `GoapAction` instances and add them to the definition via `setGoapActions()`.

- [ ] **Step 3: Add fields to hand-written schema**

Add `cost:`, `effect:`, `softDependency:` to the `Worker` definition in `CaseDefinition.yaml`. Add `customize:` to `CaseDefinitionSpec`.

- [ ] **Step 4: Create GOAP YAML example fixture**

- [ ] **Step 5: Run all tests**

Run: `mvn test -pl schema,api,generator -q`
Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git add api/ schema/ generator/
git commit -m "feat(#976): YAML expansion — GOAP annotations + customize Refs #976"
```

---

## References

- `specs/issue-422-ts-programming-model/2026-08-24-schema-generator-design.md` — design spec
- `codegen/src/main/java/io/casehub/codegen/CasehubRuleFactory.java` — two domain rules to reproduce
- `schema/src/main/resources/schema/CaseDefinition.yaml` — existing schema (1344 lines)
- `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — canonical model (1019 lines)
- `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — YAML mapper (1882 lines)
- `schema/src/test/java/io/casehub/model/SchemaValidationTest.java` — existing schema tests
- `reviews/casehub-slots/issue-422-schema-generator-20260824-044035/` — design review
- casehubio/parent#422 — epic
- casehubio/engine#975 — SchemaWriter issue
- casehubio/engine#976 — YAML expansion issue
