# CaseDefinitionModule Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #422 — TypeScript Programming Model
**Issue group:** #422

**Goal:** Build a Jackson `SimpleModule` that enables direct `objectMapper.readValue(yaml, CaseDefinition.class)` deserialization, following protocol PP-20260825-7ad4b1 (externalized serialization — no behavior annotations on domain types).

**Architecture:** Single `CaseDefinitionModule extends SimpleModule` in `api/.../converter/` registers custom deserializers for 6 polymorphic types, a `WorkerDeserializer` for record handling, mixins for non-serializable fields, and property name mappings. Constructor accepts nullable `ExpressionEngineRegistry` for non-JQ language support. Each deserializer is independently testable.

**Tech Stack:** Jackson 2.x (`SimpleModule`, `StdDeserializer`, `DeserializationContext`), existing engine-api model types

## Global Constraints

- No behavior annotations (`@JsonCreator`, `@JsonDeserialize`, `@JsonTypeInfo`) on domain types — protocol PP-20260825-7ad4b1
- Metadata annotations (`@JsonPropertyDescription`, `@JsonIgnore`, `@JsonProperty`) are fine
- Pre-release platform — breaking changes acceptable
- All tests run via `mvn test -pl api -Dtest=<TestClass>`

---

## Batch 1: Module Skeleton + Leaf Deserializers

### Task 1: Module skeleton + ExpressionEvaluatorDeserializer

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/ExpressionEvaluatorDeserializer.java`
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/ExpressionEvaluatorDeserializerTest.java`

**Interfaces:**
- Produces: `CaseDefinitionModule(ExpressionEngineRegistry)` — module with ExpressionEvaluator deserialization
- Produces: `ExpressionEvaluatorDeserializer` — string → `JQExpressionEvaluator`, `{lang: expr}` → registry-resolved

- [ ] **Step 1: Write failing test — string expression**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.converter.CaseDefinitionModule;
import io.casehub.api.model.evaluator.ExpressionEvaluator;
import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import org.junit.jupiter.api.Test;

class ExpressionEvaluatorDeserializerTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new CaseDefinitionModule(null));

    @Test
    void string_deserializesToJQExpression() throws Exception {
        ExpressionEvaluator result = mapper.readValue(
            "\"$.transaction.amount > 10000\"", ExpressionEvaluator.class);
        assertInstanceOf(JQExpressionEvaluator.class, result);
        assertEquals("$.transaction.amount > 10000",
            ((JQExpressionEvaluator) result).expression());
    }

    @Test
    void singleKeyMap_deserializesToJQWhenLangIsJq() throws Exception {
        ExpressionEvaluator result = mapper.readValue(
            "{\"jq\": \".amount > 100\"}", ExpressionEvaluator.class);
        assertInstanceOf(JQExpressionEvaluator.class, result);
        assertEquals(".amount > 100",
            ((JQExpressionEvaluator) result).expression());
    }

    @Test
    void singleKeyMap_nonJqWithoutRegistry_throws() {
        assertThrows(Exception.class, () ->
            mapper.readValue("{\"mvel\": \"amount > 100\"}", ExpressionEvaluator.class));
    }

    @Test
    void multiKeyMap_throws() {
        assertThrows(Exception.class, () ->
            mapper.readValue("{\"jq\": \"a\", \"mvel\": \"b\"}", ExpressionEvaluator.class));
    }

    @Test
    void nullValue_deserializesToNull() throws Exception {
        ExpressionEvaluator result = mapper.readValue("null", ExpressionEvaluator.class);
        assertNull(result);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=ExpressionEvaluatorDeserializerTest -q`
Expected: Compilation failure — classes don't exist yet.

- [ ] **Step 3: Create CaseDefinitionModule skeleton**

```java
package io.casehub.api.model.converter;

import com.fasterxml.jackson.databind.module.SimpleModule;
import io.casehub.api.model.converter.deser.ExpressionEvaluatorDeserializer;
import io.casehub.api.model.evaluator.ExpressionEvaluator;
import io.casehub.platform.api.expression.ExpressionEngineRegistry;

public class CaseDefinitionModule extends SimpleModule {

    public CaseDefinitionModule(ExpressionEngineRegistry registry) {
        super("CaseDefinitionModule");
        addDeserializer(ExpressionEvaluator.class,
            new ExpressionEvaluatorDeserializer(registry));
    }
}
```

- [ ] **Step 4: Implement ExpressionEvaluatorDeserializer**

```java
package io.casehub.api.model.converter.deser;

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.deser.std.StdDeserializer;
import io.casehub.api.model.evaluator.ExpressionEvaluator;
import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import io.casehub.platform.api.expression.ExpressionEngineRegistry;
import java.io.IOException;

public class ExpressionEvaluatorDeserializer extends StdDeserializer<ExpressionEvaluator> {

    static final String EXPRESSION_LANG_KEY = "casehub.expressionLang";
    private final ExpressionEngineRegistry registry;

    public ExpressionEvaluatorDeserializer(ExpressionEngineRegistry registry) {
        super(ExpressionEvaluator.class);
        this.registry = registry;
    }

    @Override
    public ExpressionEvaluator deserialize(JsonParser p, DeserializationContext ctxt)
            throws IOException {
        JsonNode node = p.readValueAsTree();
        if (node == null || node.isNull()) {
            return null;
        }
        String defaultLang = (String) ctxt.getAttribute(EXPRESSION_LANG_KEY);
        if (defaultLang == null) {
            defaultLang = JQExpressionEvaluator.TYPE;
        }
        if (node.isTextual()) {
            return createExpression(node.asText(), defaultLang);
        }
        if (node.isObject()) {
            if (node.size() != 1) {
                throw ctxt.weirdStringException(node.toString(),
                    ExpressionEvaluator.class,
                    "Expression override must be a single-key map {lang: expr}, got "
                        + node.size() + " keys");
            }
            var entry = node.fields().next();
            return createExpression(entry.getValue().asText(), entry.getKey());
        }
        throw ctxt.weirdStringException(node.toString(),
            ExpressionEvaluator.class,
            "Expression must be a string or single-key map {lang: expr}");
    }

    private ExpressionEvaluator createExpression(String expression, String lang) {
        if (JQExpressionEvaluator.TYPE.equals(lang)) {
            return new JQExpressionEvaluator(expression);
        }
        if (registry == null) {
            throw new IllegalArgumentException(
                "ExpressionEngineRegistry required for non-JQ language: " + lang);
        }
        return registry.create(expression, lang);
    }

    @Override
    public ExpressionEvaluator getNullValue(DeserializationContext ctxt) {
        return null;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=ExpressionEvaluatorDeserializerTest -q`
Expected: All 5 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java api/src/main/java/io/casehub/api/model/converter/deser/ api/src/test/java/io/casehub/api/model/converter/deser/
git commit -m "feat(#422): CaseDefinitionModule skeleton + ExpressionEvaluatorDeserializer Refs #422"
```

### Task 2: GoalExpressionDeserializer + CaseCompletionDeserializer

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/GoalExpressionDeserializer.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/CaseCompletionDeserializer.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java` — register both
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/GoalExpressionDeserializerTest.java`
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/CaseCompletionDeserializerTest.java`

**Interfaces:**
- Consumes: `CaseDefinitionModule` from Task 1
- Produces: `GoalExpressionDeserializer` — string/array/nested allOf/anyOf → `GoalExpression`
- Produces: `CaseCompletionDeserializer` — doneWhen/goal-kind map → `CaseCompletion`

- [ ] **Step 1: Write failing tests — GoalExpression**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.*;
import io.casehub.api.model.converter.CaseDefinitionModule;
import org.junit.jupiter.api.Test;

class GoalExpressionDeserializerTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new CaseDefinitionModule(null));

    @Test
    void string_deserializesToSingleGoal() throws Exception {
        GoalExpression result = mapper.readValue("\"done\"", GoalExpression.class);
        assertInstanceOf(SingleGoalExpression.class, result);
        assertEquals("done", ((SingleGoalExpression) result).goalName());
    }

    @Test
    void stringArray_deserializesToAllOf() throws Exception {
        GoalExpression result = mapper.readValue("[\"a\", \"b\"]", GoalExpression.class);
        assertInstanceOf(AllOfGoalExpression.class, result);
        assertEquals(2, ((AllOfGoalExpression) result).children().size());
    }

    @Test
    void nestedAllOf_deserializesRecursively() throws Exception {
        String json = "{\"allOf\": [\"a\", {\"anyOf\": [\"b\", \"c\"]}]}";
        GoalExpression result = mapper.readValue(json, GoalExpression.class);
        assertInstanceOf(AllOfGoalExpression.class, result);
        var children = ((AllOfGoalExpression) result).children();
        assertEquals(2, children.size());
        assertInstanceOf(SingleGoalExpression.class, children.get(0));
        assertInstanceOf(AnyOfGoalExpression.class, children.get(1));
    }
}
```

- [ ] **Step 2: Write failing tests — CaseCompletion**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.*;
import io.casehub.api.model.converter.CaseDefinitionModule;
import org.junit.jupiter.api.Test;

class CaseCompletionDeserializerTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new CaseDefinitionModule(null));

    @Test
    void doneWhen_deserializesToPredicateBased() throws Exception {
        String json = "{\"doneWhen\": \".status == \\\"done\\\"\"}";
        CaseCompletion result = mapper.readValue(json, CaseCompletion.class);
        assertInstanceOf(PredicateBasedCompletion.class, result);
    }

    @Test
    void goalKindMap_deserializesToGoalBased() throws Exception {
        String json = "{\"success\": \"goal-a\", \"failure\": \"goal-b\"}";
        CaseCompletion result = mapper.readValue(json, CaseCompletion.class);
        assertInstanceOf(GoalBasedCompletion.class, result);
    }

    @Test
    void doneWhenPlusGoalKind_throws() {
        String json = "{\"doneWhen\": \".x\", \"success\": \"g\"}";
        assertThrows(Exception.class, () ->
            mapper.readValue(json, CaseCompletion.class));
    }
}
```

- [ ] **Step 3: Implement GoalExpressionDeserializer**

Recursive deserializer: string → `SingleGoalExpression`, array → `AllOfGoalExpression`, object with `allOf`/`anyOf` → recursive children. Follow the pattern from `CaseDefinitionYamlMapper.parseGoalExpressionFromNode()` (line 1718).

- [ ] **Step 4: Implement CaseCompletionDeserializer**

Check for `doneWhen` key first (→ `PredicateBasedCompletion`). Otherwise treat keys as goal kinds using `resolveGoalKind()` logic from mapper (line 1693). Values are `GoalExpression` via the registered deserializer.

- [ ] **Step 5: Register both in CaseDefinitionModule**

Add `addDeserializer(GoalExpression.class, ...)` and `addDeserializer(CaseCompletion.class, ...)`.

- [ ] **Step 6: Run tests**

Run: `mvn test -pl api -Dtest="GoalExpressionDeserializerTest,CaseCompletionDeserializerTest" -q`
Expected: All tests PASS.

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/converter/deser/ api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java api/src/test/java/io/casehub/api/model/converter/deser/
git commit -m "feat(#422): GoalExpressionDeserializer + CaseCompletionDeserializer Refs #422"
```

### Task 3: TriggerDeserializer + SubCaseMappingDeserializer

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/TriggerDeserializer.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/SubCaseMappingDeserializer.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java` — register both
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/TriggerDeserializerTest.java`
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/SubCaseMappingDeserializerTest.java`

**Interfaces:**
- Consumes: `CaseDefinitionModule` from Task 1
- Produces: `TriggerDeserializer` — named-property dispatch for `contextChange`, `schedule`, `scopeActivated`
- Produces: `SubCaseMappingDeserializer` — string/map → `SubCaseMapping.Expression`

- [ ] **Step 1: Write failing tests — Trigger**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.*;
import io.casehub.api.model.converter.CaseDefinitionModule;
import org.junit.jupiter.api.Test;

class TriggerDeserializerTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new CaseDefinitionModule(null));

    @Test
    void contextChange_withFilter() throws Exception {
        String json = "{\"contextChange\": {\"filter\": \".amount > 100\"}}";
        Trigger result = mapper.readValue(json, Trigger.class);
        assertInstanceOf(ContextChangeTrigger.class, result);
        assertNotNull(((ContextChangeTrigger) result).filter());
    }

    @Test
    void contextChange_empty() throws Exception {
        String json = "{\"contextChange\": {}}";
        Trigger result = mapper.readValue(json, Trigger.class);
        assertInstanceOf(ContextChangeTrigger.class, result);
    }

    @Test
    void schedule_cron() throws Exception {
        String json = "{\"schedule\": {\"cron\": \"0 0 * * *\"}}";
        Trigger result = mapper.readValue(json, Trigger.class);
        assertInstanceOf(ScheduleTrigger.class, result);
    }

    @Test
    void schedule_every() throws Exception {
        String json = "{\"schedule\": {\"every\": \"PT1H\"}}";
        Trigger result = mapper.readValue(json, Trigger.class);
        assertInstanceOf(ScheduleTrigger.class, result);
    }

    @Test
    void scopeActivated() throws Exception {
        String json = "{\"scopeActivated\": {}}";
        Trigger result = mapper.readValue(json, Trigger.class);
        assertInstanceOf(ScopeActivatedTrigger.class, result);
    }

    @Test
    void unknownKey_throws() {
        assertThrows(Exception.class, () ->
            mapper.readValue("{\"cloudEvent\": {}}", Trigger.class));
    }

    @Test
    void multipleKeys_throws() {
        assertThrows(Exception.class, () ->
            mapper.readValue("{\"contextChange\": {}, \"schedule\": {}}", Trigger.class));
    }
}
```

- [ ] **Step 2: Write failing tests — SubCaseMapping**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.SubCaseMapping;
import io.casehub.api.model.converter.CaseDefinitionModule;
import org.junit.jupiter.api.Test;

class SubCaseMappingDeserializerTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new CaseDefinitionModule(null));

    @Test
    void string_deserializesToJqExpression() throws Exception {
        SubCaseMapping result = mapper.readValue("\".child\"", SubCaseMapping.class);
        assertInstanceOf(SubCaseMapping.Expression.class, result);
    }

    @Test
    void singleKeyMap_deserializesToExpression() throws Exception {
        SubCaseMapping result = mapper.readValue("{\"jq\": \".child\"}", SubCaseMapping.class);
        assertInstanceOf(SubCaseMapping.Expression.class, result);
    }
}
```

- [ ] **Step 3: Implement TriggerDeserializer**

Named-property dispatch following `CaseDefinitionYamlMapper.convertTrigger()` (line 1616). Read the single property key, delegate to the appropriate constructor. `ContextChangeTrigger` needs `filter` extracted as `ExpressionEvaluator`. `ScheduleTrigger` dispatches on `cron` vs `every` keys.

- [ ] **Step 4: Implement SubCaseMappingDeserializer**

Same two-form pattern as `ExpressionEvaluatorDeserializer`. String → `SubCaseMapping.Expression(JQExpressionEvaluator)`. Map → `SubCaseMapping.Expression(resolved evaluator)`.

- [ ] **Step 5: Register both in CaseDefinitionModule**

- [ ] **Step 6: Run tests**

Run: `mvn test -pl api -Dtest="TriggerDeserializerTest,SubCaseMappingDeserializerTest" -q`
Expected: All tests PASS.

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/converter/deser/ api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java api/src/test/java/io/casehub/api/model/converter/deser/
git commit -m "feat(#422): TriggerDeserializer + SubCaseMappingDeserializer Refs #422"
```

---

## Batch 2: Complex Deserializers + Mixins

### Task 4: WorkerDeserializer

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/WorkerDeserializer.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java`
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/WorkerDeserializerTest.java`

**Interfaces:**
- Consumes: `CaseDefinitionModule` from Task 1
- Produces: `WorkerDeserializer` — builds `Worker` via `Worker.builder()`, skips `function` (→ `NONE`), preserves plugin blocks as raw properties

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.converter.CaseDefinitionModule;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import org.junit.jupiter.api.Test;

class WorkerDeserializerTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new CaseDefinitionModule(null));

    @Test
    void basicWorker_deserializes() throws Exception {
        String json = "{\"name\": \"analyser\", \"capabilities\": [\"analysis\"]}";
        Worker result = mapper.readValue(json, Worker.class);
        assertEquals("analyser", result.name());
        assertTrue(result.capabilityNames().contains("analysis"));
        assertEquals(WorkerFunction.NONE, result.function());
    }

    @Test
    void workerWithDescription_deserializes() throws Exception {
        String json = "{\"name\": \"w\", \"capabilities\": [\"c\"], \"description\": \"A worker\"}";
        Worker result = mapper.readValue(json, Worker.class);
        assertEquals("A worker", result.description());
    }
}
```

- [ ] **Step 2: Implement WorkerDeserializer**

Read all properties from the JSON tree. Build `Worker` via `Worker.builder()` — `name`, `capabilityNames`, `executionPolicy`, `description`. Set `function` to `WorkerFunction.NONE`. Preserve unknown properties (plugin blocks) — these are available for downstream `WorkerFunctionProvider` resolution.

- [ ] **Step 3: Register in CaseDefinitionModule + run tests**

Run: `mvn test -pl api -Dtest=WorkerDeserializerTest -q`
Expected: All tests PASS.

- [ ] **Step 4: Commit**

### Task 5: Mixins + Property Name Mappings

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionSpecMixin.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionMixin.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/GoapActionMixin.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java` — register mixins
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/PropertyMappingTest.java`

**Interfaces:**
- Consumes: `CaseDefinitionModule` from Task 1
- Produces: Mixins that map YAML property names to Java field names + suppress non-serializable fields

- [ ] **Step 1: Write failing tests — property name mapping**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.CaseDefinitionSpec;
import io.casehub.api.model.converter.CaseDefinitionModule;
import org.junit.jupiter.api.Test;

class PropertyMappingTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new CaseDefinitionModule(null));

    @Test
    void cbrYamlKey_mapsToCbrConfigField() throws Exception {
        String json = "{\"cbr\": {\"topK\": 5}}";
        CaseDefinitionSpec spec = mapper.readValue(json, CaseDefinitionSpec.class);
        assertNotNull(spec.getCbrConfig());
    }

    @Test
    void adaptationYamlKey_mapsToAdaptationConfigField() throws Exception {
        String json = "{\"adaptation\": {\"trigger\": \"every-step\"}}";
        CaseDefinitionSpec spec = mapper.readValue(json, CaseDefinitionSpec.class);
        assertNotNull(spec.getAdaptationConfig());
    }

    @Test
    void reflectionYamlKey_mapsToReflectionTriggerField() throws Exception {
        String json = "{\"reflection\": {\"enabled\": true}}";
        CaseDefinitionSpec spec = mapper.readValue(json, CaseDefinitionSpec.class);
        assertNotNull(spec.getReflectionTrigger());
    }

    @Test
    void monitoringYamlKey_mapsToMonitoringConfigField() throws Exception {
        String json = "{\"monitoring\": {\"enabled\": true}}";
        CaseDefinitionSpec spec = mapper.readValue(json, CaseDefinitionSpec.class);
        assertNotNull(spec.getMonitoringConfig());
    }
}
```

- [ ] **Step 2: Implement mixins**

`CaseDefinitionSpecMixin`:
```java
abstract class CaseDefinitionSpecMixin {
    @JsonProperty("cbr") abstract CbrConfig getCbrConfig();
    @JsonProperty("adaptation") abstract AdaptationConfig getAdaptationConfig();
    @JsonProperty("reflection") abstract ReflectionTriggerConfig getReflectionTrigger();
    @JsonProperty("monitoring") abstract MonitoringConfig getMonitoringConfig();
}
```

`CaseDefinitionMixin`:
```java
abstract class CaseDefinitionMixin {
    @JsonProperty("layers") abstract List<String> getLayerNames();
    @JsonProperty("episodic") abstract EpisodicMemoryConfig getEpisodicMemoryConfig();
}
```

`GoapActionMixin`:
```java
@JsonIgnoreProperties({"costFunction"})
abstract class GoapActionMixin {}
```

- [ ] **Step 3: Register mixins in CaseDefinitionModule**

```java
setMixIn(CaseDefinitionSpec.class, CaseDefinitionSpecMixin.class);
setMixIn(CaseDefinition.class, CaseDefinitionMixin.class);
setMixIn(GoapAction.class, GoapActionMixin.class);
```

- [ ] **Step 4: Run tests**

Run: `mvn test -pl api -Dtest=PropertyMappingTest -q`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

---

## Batch 3: Integration + Full Round-Trip

### Task 6: BindingDeserializer + CaseDefinitionDeserializer + Integration Test

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/BindingDeserializer.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionDeserializer.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java` — register both
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/BindingDeserializerTest.java`
- Create: `api/src/test/java/io/casehub/api/model/converter/deser/CaseDefinitionModuleIntegrationTest.java`

**Interfaces:**
- Consumes: All deserializers from Tasks 1-5
- Produces: `BindingDeserializer` — full binding with target dispatch, trigger, expressions
- Produces: `CaseDefinitionDeserializer` — top-level structural mapping, `expressionLang` context propagation
- Produces: Integration test comparing module path vs mapper path for existing YAML fixtures

- [ ] **Step 1: Write failing tests — Binding**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.*;
import io.casehub.api.model.converter.CaseDefinitionModule;
import org.junit.jupiter.api.Test;

class BindingDeserializerTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new CaseDefinitionModule(null));

    @Test
    void capabilityBinding_deserializes() throws Exception {
        String json = """
            {
              "name": "trigger",
              "capability": "process",
              "on": {"contextChange": {"filter": ".ready"}}
            }
            """;
        Binding result = mapper.readValue(json, Binding.class);
        assertEquals("trigger", result.name());
        assertInstanceOf(CapabilityTarget.class, result.target());
        assertInstanceOf(ContextChangeTrigger.class, result.trigger());
    }

    @Test
    void humanTaskBinding_deserializes() throws Exception {
        String json = """
            {
              "name": "review",
              "humanTask": {
                "candidateGroups": ["reviewers"]
              },
              "on": {"contextChange": {}}
            }
            """;
        Binding result = mapper.readValue(json, Binding.class);
        assertInstanceOf(HumanTaskTarget.class, result.target());
    }
}
```

- [ ] **Step 2: Write failing integration test**

```java
package io.casehub.api.model.converter.deser;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.converter.CaseDefinitionModule;
import io.casehub.api.model.converter.CaseDefinitionYamlMapper;
import java.io.InputStream;
import org.junit.jupiter.api.Test;

class CaseDefinitionModuleIntegrationTest {

    @Test
    void minimalDefinition_matchesMapperOutput() throws Exception {
        String yaml = """
            dsl: "0.1.0"
            namespace: test
            name: minimal
            version: "1.0.0"
            spec:
              capabilities:
                - name: process
              workers:
                - name: worker-1
                  capabilities: [process]
              bindings:
                - name: trigger
                  capability: process
                  on:
                    contextChange: {}
            """;

        CaseDefinition fromMapper = CaseDefinitionYamlMapper.load(
            new java.io.ByteArrayInputStream(yaml.getBytes()));

        ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory());
        yamlMapper.registerModule(new CaseDefinitionModule(null));
        CaseDefinition fromModule = yamlMapper.readValue(yaml, CaseDefinition.class);

        assertEquals(fromMapper.getNamespace(), fromModule.getNamespace());
        assertEquals(fromMapper.getName(), fromModule.getName());
        assertEquals(fromMapper.getVersion(), fromModule.getVersion());
        assertEquals(fromMapper.getCapabilities().size(),
            fromModule.getCapabilities().size());
        assertEquals(fromMapper.getWorkers().size(),
            fromModule.getWorkers().size());
        assertEquals(fromMapper.getBindings().size(),
            fromModule.getBindings().size());
    }
}
```

- [ ] **Step 3: Implement BindingDeserializer**

Read all properties from JsonNode. Dispatch target based on presence of `capability`/`subCase`/`humanTask`/`signal` keys. Read `on` key and delegate to `TriggerDeserializer`. Read expression-typed fields via registered `ExpressionEvaluator` deserializer. Build `Binding` via `Binding.builder()`.

For `CapabilityTarget`: store the capability name as a deferred reference. The `CapabilityTarget` constructor needs a `Capability` record — use a factory method that takes a string name and creates a minimal `Capability.of(name, null, null)` placeholder. Post-deserialization resolution (in `CaseDefinitionDeserializer`) replaces these with the real `Capability` objects.

- [ ] **Step 4: Implement CaseDefinitionDeserializer**

Top-level deserializer that:
1. Reads identity fields (`namespace`, `name`, `version`, `dsl`, `title`, `summary`)
2. Sets `expressionLang` as `DeserializationContext` attribute before delegating
3. Reads `context` object and extracts `storeFactory` → `contextStoreFactory`
4. Delegates `spec` to standard Jackson deserialization (with registered deserializers)
5. Post-deserialization: resolves `CapabilityTarget` capability references against `capabilities` list

- [ ] **Step 5: Register in CaseDefinitionModule + run all tests**

Run: `mvn test -pl api -Dtest="BindingDeserializerTest,CaseDefinitionModuleIntegrationTest,ExpressionEvaluatorDeserializerTest,GoalExpressionDeserializerTest,CaseCompletionDeserializerTest,TriggerDeserializerTest,SubCaseMappingDeserializerTest,WorkerDeserializerTest,PropertyMappingTest" -q`
Expected: All tests PASS.

- [ ] **Step 6: Run full api test suite to check for regressions**

Run: `mvn test -pl api -q`
Expected: All 1270+ tests PASS — the module is additive, no existing code changed.

- [ ] **Step 7: Commit**

```bash
git add api/
git commit -m "feat(#422): BindingDeserializer + CaseDefinitionDeserializer + integration test Refs #422"
```

---

## References

- `specs/issue-422-ts-programming-model/2026-08-25-case-definition-module-design.md` — design spec
- `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — reference implementation
- `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — domain type
- `api/src/main/java/io/casehub/api/model/CaseDefinitionSpec.java` — spec type
- PP-20260825-7ad4b1 — Jackson externalized serialization protocol
- casehubio/parent#422 — epic
