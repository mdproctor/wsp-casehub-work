# Progress Definition YAML Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #373 — Progress definition YAML
**Issue group:** #371, #372, #373

**Goal:** Add YAML-declared progress definitions loaded from classpath at startup, referenced by name in `ProgressCreateRequest`.

**Architecture:** `ProgressDefinition` record + `ProgressDefinitionRegistry` in progress-api. `ProgressDefinitionYamlLoader` in progress-runtime loads `META-INF/work-progress-definitions.yaml` at startup. `ProgressService.create()` resolves `definitionName` from the registry, filling shapeType/definition/defaults from the template.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- `ProgressDefinition` + `ProgressDefinitionRegistry` in `io.casehub.work.progress` (progress-api)
- `ProgressDefinitionYamlLoader` in `io.casehub.work.progress.runtime.service` (progress-runtime)
- No Flyway migration (in-memory registry, no persistence)
- Tests: plain JUnit 5 + AssertJ (no CDI container)
- Build progress-api: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-api`
- Build progress-runtime: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime`

---

## Batch 1: Foundation — ProgressDefinition + Registry

### Task 1: ProgressDefinition record + ProgressDefinitionRegistry

**Files:**
- Create: `progress-api/src/main/java/io/casehub/work/progress/ProgressDefinition.java`
- Create: `progress-api/src/main/java/io/casehub/work/progress/ProgressDefinitionRegistry.java`
- Test: `progress-api/src/test/java/io/casehub/work/progress/ProgressDefinitionTest.java`
- Test: `progress-api/src/test/java/io/casehub/work/progress/ProgressDefinitionRegistryTest.java`

**Interfaces:**
- Consumes: `com.fasterxml.jackson.databind.JsonNode` (existing)
- Produces:
  - `ProgressDefinition(String name, String shapeType, JsonNode definition, String rollbackPolicy, String visualisationMode)`
  - `ProgressDefinitionRegistry.register(ProgressDefinition)`, `.get(String name) → Optional<ProgressDefinition>`, `.getAll() → Collection<ProgressDefinition>`

- [ ] **Step 1: Write failing tests for ProgressDefinition**

```java
package io.casehub.work.progress;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.JsonNode;
import org.junit.jupiter.api.Test;

class ProgressDefinitionTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void constructValid() {
        JsonNode def = mapper.createArrayNode();
        var pd = new ProgressDefinition("doc-review", "step", def, "revert-to-previous", "linear");
        assertThat(pd.name()).isEqualTo("doc-review");
        assertThat(pd.shapeType()).isEqualTo("step");
        assertThat(pd.definition()).isEqualTo(def);
        assertThat(pd.rollbackPolicy()).isEqualTo("revert-to-previous");
        assertThat(pd.visualisationMode()).isEqualTo("linear");
    }

    @Test
    void constructMinimal() {
        var pd = new ProgressDefinition("simple", "percentage", null, null, null);
        assertThat(pd.name()).isEqualTo("simple");
        assertThat(pd.definition()).isNull();
    }

    @Test
    void nullNameThrows() {
        assertThatThrownBy(() -> new ProgressDefinition(null, "step", null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("name");
    }

    @Test
    void blankNameThrows() {
        assertThatThrownBy(() -> new ProgressDefinition("  ", "step", null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("name");
    }

    @Test
    void nullShapeTypeThrows() {
        assertThatThrownBy(() -> new ProgressDefinition("test", null, null, null, null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("shapeType");
    }
}
```

- [ ] **Step 2: Write failing tests for ProgressDefinitionRegistry**

```java
package io.casehub.work.progress;

import static org.assertj.core.api.Assertions.assertThat;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ProgressDefinitionRegistryTest {

    private ProgressDefinitionRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new ProgressDefinitionRegistry();
    }

    @Test
    void registerAndGet() {
        var def = new ProgressDefinition("doc-review", "step", null, null, null);
        registry.register(def);
        assertThat(registry.get("doc-review")).contains(def);
    }

    @Test
    void getUnknownReturnsEmpty() {
        assertThat(registry.get("nonexistent")).isEmpty();
    }

    @Test
    void duplicateNameOverwrites() {
        var def1 = new ProgressDefinition("doc-review", "step", null, null, null);
        var def2 = new ProgressDefinition("doc-review", "percentage", null, null, null);
        registry.register(def1);
        registry.register(def2);
        assertThat(registry.get("doc-review")).contains(def2);
    }

    @Test
    void getAll() {
        registry.register(new ProgressDefinition("a", "step", null, null, null));
        registry.register(new ProgressDefinition("b", "percentage", null, null, null));
        assertThat(registry.getAll()).hasSize(2);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-api -Dtest="ProgressDefinitionTest,ProgressDefinitionRegistryTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes do not exist

- [ ] **Step 4: Implement ProgressDefinition**

Use `ide_create_file`:

```java
package io.casehub.work.progress;

import com.fasterxml.jackson.databind.JsonNode;

public record ProgressDefinition(
        String name,
        String shapeType,
        JsonNode definition,
        String rollbackPolicy,
        String visualisationMode) {

    public ProgressDefinition {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("ProgressDefinition name is required");
        }
        if (shapeType == null || shapeType.isBlank()) {
            throw new IllegalArgumentException("ProgressDefinition shapeType is required");
        }
    }
}
```

- [ ] **Step 5: Implement ProgressDefinitionRegistry**

Use `ide_create_file`:

```java
package io.casehub.work.progress;

import java.util.Collection;
import java.util.Collections;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

import org.jboss.logging.Logger;

public class ProgressDefinitionRegistry {

    private static final Logger LOG = Logger.getLogger(ProgressDefinitionRegistry.class);
    private final Map<String, ProgressDefinition> definitions = new ConcurrentHashMap<>();

    public void register(ProgressDefinition definition) {
        ProgressDefinition prev = definitions.put(definition.name(), definition);
        if (prev != null) {
            LOG.warnf("ProgressDefinition '%s' overwritten", definition.name());
        }
    }

    public Optional<ProgressDefinition> get(String name) {
        return Optional.ofNullable(definitions.get(name));
    }

    public Collection<ProgressDefinition> getAll() {
        return Collections.unmodifiableCollection(definitions.values());
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-api -Dtest="ProgressDefinitionTest,ProgressDefinitionRegistryTest"`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add progress-api/src/main/java/io/casehub/work/progress/ProgressDefinition.java \
        progress-api/src/main/java/io/casehub/work/progress/ProgressDefinitionRegistry.java \
        progress-api/src/test/java/io/casehub/work/progress/ProgressDefinitionTest.java \
        progress-api/src/test/java/io/casehub/work/progress/ProgressDefinitionRegistryTest.java
git commit -m "feat(#373): ProgressDefinition record + registry

Named progress definition template with shapeType, definition JSON,
and optional defaults (rollbackPolicy, visualisationMode). In-memory
registry for lookup by name.

Refs #373"
```

---

## Batch 2: Loader + Service Integration

### Task 2: ProgressDefinitionYamlLoader

**Files:**
- Create: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/ProgressDefinitionYamlLoader.java`
- Create: `progress-runtime/src/test/resources/META-INF/work-progress-definitions.yaml` (test fixture)
- Test: `progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/ProgressDefinitionYamlLoaderTest.java`

**Interfaces:**
- Consumes: `ProgressDefinitionRegistry.register()` (Task 1), `StepDefinition`, `StepDefinitionValidator`
- Produces: `ProgressDefinitionYamlLoader` — `@Startup` bean, `loadFromClasspath()` (package-private for testing)

- [ ] **Step 1: Create test fixture YAML**

Create `progress-runtime/src/test/resources/META-INF/work-progress-definitions.yaml`:

```yaml
progressDefinitions:
  - name: document-review
    shapeType: step
    stages:
      - name: received
      - name: in-review
      - name: approved
      - name: rejected
        optional: true
    transitions:
      received: [in-review]
      in-review: [approved, rejected, received]
    rollbackPolicy: revert-to-previous
    visualisationMode: linear

  - name: simple-percentage
    shapeType: percentage
```

- [ ] **Step 2: Write failing tests**

```java
package io.casehub.work.progress.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.work.progress.ProgressDefinition;
import io.casehub.work.progress.ProgressDefinitionRegistry;

class ProgressDefinitionYamlLoaderTest {

    @Test
    void loadFromClasspath() {
        ProgressDefinitionRegistry registry = new ProgressDefinitionRegistry();
        ProgressDefinitionYamlLoader loader = new ProgressDefinitionYamlLoader();
        loader.registry = registry;

        loader.loadFromClasspath();

        assertThat(registry.getAll()).hasSize(2);
    }

    @Test
    void loadStepDefinition() {
        ProgressDefinitionRegistry registry = new ProgressDefinitionRegistry();
        ProgressDefinitionYamlLoader loader = new ProgressDefinitionYamlLoader();
        loader.registry = registry;

        loader.loadFromClasspath();

        ProgressDefinition docReview = registry.get("document-review").orElseThrow();
        assertThat(docReview.shapeType()).isEqualTo("step");
        assertThat(docReview.rollbackPolicy()).isEqualTo("revert-to-previous");
        assertThat(docReview.visualisationMode()).isEqualTo("linear");
        assertThat(docReview.definition()).isNotNull();
        assertThat(docReview.definition().has("steps")).isTrue();
        assertThat(docReview.definition().get("steps").size()).isEqualTo(4);
        assertThat(docReview.definition().has("transitions")).isTrue();
        assertThat(docReview.definition().get("transitions").get("received").size()).isEqualTo(1);
    }

    @Test
    void loadPercentageDefinition() {
        ProgressDefinitionRegistry registry = new ProgressDefinitionRegistry();
        ProgressDefinitionYamlLoader loader = new ProgressDefinitionYamlLoader();
        loader.registry = registry;

        loader.loadFromClasspath();

        ProgressDefinition simple = registry.get("simple-percentage").orElseThrow();
        assertThat(simple.shapeType()).isEqualTo("percentage");
        assertThat(simple.definition()).isNull();
    }

    @Test
    void stageOptionalFlag() {
        ProgressDefinitionRegistry registry = new ProgressDefinitionRegistry();
        ProgressDefinitionYamlLoader loader = new ProgressDefinitionYamlLoader();
        loader.registry = registry;

        loader.loadFromClasspath();

        ProgressDefinition docReview = registry.get("document-review").orElseThrow();
        var steps = docReview.definition().get("steps");
        boolean rejectedOptional = false;
        for (var step : steps) {
            if ("rejected".equals(step.get("name").asText())) {
                rejectedOptional = step.get("optional").asBoolean();
            }
        }
        assertThat(rejectedOptional).isTrue();
    }

    @Test
    void variableInterpolation() {
        System.setProperty("test.progress.name", "interpolated");
        try {
            String result = ProgressDefinitionYamlLoader.interpolate("${sys.test.progress.name}");
            assertThat(result).isEqualTo("interpolated");
        } finally {
            System.clearProperty("test.progress.name");
        }
    }

    @Test
    void interpolateNull() {
        assertThat(ProgressDefinitionYamlLoader.interpolate(null)).isNull();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime -Dtest=ProgressDefinitionYamlLoaderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 4: Implement ProgressDefinitionYamlLoader**

Use `ide_create_file`:

```java
package io.casehub.work.progress.runtime.service;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.work.progress.ProgressDefinition;
import io.casehub.work.progress.ProgressDefinitionRegistry;
import io.casehub.work.progress.StepDefinition;
import io.casehub.work.progress.validation.StepDefinitionValidator;
import io.quarkus.runtime.Startup;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Enumeration;
import java.util.List;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@ApplicationScoped
@Startup
public class ProgressDefinitionYamlLoader {

    private static final Logger LOG = Logger.getLogger(ProgressDefinitionYamlLoader.class);
    static final String RESOURCE_PATH = "META-INF/work-progress-definitions.yaml";
    private static final Pattern VAR_PATTERN = Pattern.compile("\\$\\{(env|sys)\\.([^}]+)}");
    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory());
    private static final ObjectMapper JSON_MAPPER = new ObjectMapper();
    private static final StepDefinitionValidator STEP_VALIDATOR = new StepDefinitionValidator();

    @Inject
    ProgressDefinitionRegistry registry;

    @PostConstruct
    void load() {
        loadFromClasspath();
    }

    void loadFromClasspath() {
        try {
            Enumeration<URL> resources = Thread.currentThread()
                    .getContextClassLoader().getResources(RESOURCE_PATH);
            var urls = Collections.list(resources);
            if (urls.isEmpty()) {
                LOG.info("No " + RESOURCE_PATH + " found on classpath");
                return;
            }
            for (URL url : urls) {
                LOG.infof("Loading progress definitions from %s", url);
                try (InputStream is = url.openStream()) {
                    JsonNode root = YAML_MAPPER.readTree(is);
                    JsonNode defs = root.get("progressDefinitions");
                    if (defs == null || !defs.isArray()) {
                        LOG.warnf("No progressDefinitions array in %s — skipping", url);
                        continue;
                    }
                    for (JsonNode node : defs) {
                        ProgressDefinition pd = parseDefinition(node);
                        registry.register(pd);
                    }
                }
            }
        } catch (IOException e) {
            throw new RuntimeException("Failed to discover " + RESOURCE_PATH, e);
        }
    }

    private ProgressDefinition parseDefinition(JsonNode node) {
        String name = interpolate(textOrNull(node, "name"));
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("progressDefinition entry missing required 'name'");
        }

        JsonNode stagesNode = node.get("stages");
        String shapeType = textOrNull(node, "shapeType");
        if (shapeType == null && stagesNode != null) {
            shapeType = "step";
        }
        if (shapeType == null) {
            throw new IllegalArgumentException(
                    "progressDefinition '" + name + "' requires shapeType (or stages to infer 'step')");
        }

        JsonNode definition = null;
        if (stagesNode != null && stagesNode.isArray()) {
            definition = buildStepDefinition(name, stagesNode, node.get("transitions"));
        }

        return new ProgressDefinition(name, shapeType, definition,
                interpolate(textOrNull(node, "rollbackPolicy")),
                interpolate(textOrNull(node, "visualisationMode")));
    }

    private JsonNode buildStepDefinition(String defName, JsonNode stagesNode, JsonNode transitionsNode) {
        List<StepDefinition> stepDefs = new ArrayList<>();
        for (JsonNode stage : stagesNode) {
            String stageName;
            boolean optional = false;
            List<String> dependsOn = List.of();
            String condition = null;

            if (stage.isTextual()) {
                stageName = interpolate(stage.asText());
            } else {
                stageName = interpolate(textOrNull(stage, "name"));
                if (stage.has("optional")) optional = stage.get("optional").asBoolean();
                if (stage.has("dependsOn")) {
                    List<String> deps = new ArrayList<>();
                    stage.get("dependsOn").forEach(d -> deps.add(d.asText()));
                    dependsOn = deps;
                }
                if (stage.has("condition")) condition = textOrNull(stage, "condition");
            }

            if (stageName == null || stageName.isBlank()) {
                throw new IllegalArgumentException(
                        "progressDefinition '" + defName + "' has a stage with no name");
            }
            stepDefs.add(new StepDefinition(stageName, optional, dependsOn, condition));
        }

        STEP_VALIDATOR.validate(stepDefs);

        ObjectNode result = JSON_MAPPER.createObjectNode();
        result.set("steps", JSON_MAPPER.valueToTree(stepDefs));

        if (transitionsNode != null && transitionsNode.isObject()) {
            var names = stepDefs.stream().map(StepDefinition::name).toList();
            transitionsNode.fields().forEachRemaining(entry -> {
                String from = entry.getKey();
                if (!names.contains(from)) {
                    throw new IllegalArgumentException(
                            "progressDefinition '" + defName + "' transition key '" + from + "' is not a defined stage");
                }
                entry.getValue().forEach(target -> {
                    if (!names.contains(target.asText())) {
                        throw new IllegalArgumentException(
                                "progressDefinition '" + defName + "' transition target '" + target.asText()
                                        + "' from '" + from + "' is not a defined stage");
                    }
                });
            });
            result.set("transitions", transitionsNode.deepCopy());
        }

        return result;
    }

    private static String textOrNull(JsonNode node, String field) {
        return node.has(field) && !node.get(field).isNull() ? node.get(field).asText() : null;
    }

    static String interpolate(String value) {
        if (value == null) return null;
        Matcher m = VAR_PATTERN.matcher(value);
        StringBuilder sb = new StringBuilder();
        while (m.find()) {
            String type = m.group(1);
            String key = m.group(2);
            String resolved = "env".equals(type) ? System.getenv(key) : System.getProperty(key);
            m.appendReplacement(sb, Matcher.quoteReplacement(resolved != null ? resolved : m.group(0)));
        }
        m.appendTail(sb);
        return sb.toString();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime -Dtest=ProgressDefinitionYamlLoaderTest`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/ProgressDefinitionYamlLoader.java \
        progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/ProgressDefinitionYamlLoaderTest.java \
        progress-runtime/src/test/resources/META-INF/work-progress-definitions.yaml
git commit -m "feat(#373): ProgressDefinitionYamlLoader — classpath YAML loading

Loads progress definitions from META-INF/work-progress-definitions.yaml
at startup. Parses stages to StepDefinition records, validates via
StepDefinitionValidator, stores transitions in definition JSON.
Variable interpolation for env/sys properties.

Refs #373"
```

### Task 3: ProgressCreateRequest definitionName + ProgressService resolution

**Files:**
- Modify: `progress-api/src/main/java/io/casehub/work/progress/ProgressCreateRequest.java`
- Modify: `progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/ProgressService.java:45-98`
- Modify: `progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/ProgressServiceTest.java`

**Interfaces:**
- Consumes: `ProgressDefinitionRegistry.get()` (Task 1), `ProgressDefinition` (Task 1)
- Produces: `ProgressCreateRequest.definitionName()` field, `ProgressService.create()` definition resolution

- [ ] **Step 1: Write failing test — definitionName resolution**

Add to `ProgressServiceTest.java` (after existing setUp):

```java
    @Test
    void createWithDefinitionNameResolvesFromRegistry() {
        ObjectMapper mapper = new ObjectMapper();
        ArrayNode def = mapper.createArrayNode();
        def.addObject().put("name", "draft").put("optional", false).putArray("dependsOn");
        def.addObject().put("name", "review").put("optional", false).putArray("dependsOn").add("draft");

        ObjectNode fullDef = mapper.createObjectNode();
        fullDef.set("steps", def);

        registry.register(new ProgressDefinition("my-workflow", "step", fullDef, "revert-to-previous", "linear"));

        ObjectNode state = mapper.createObjectNode();
        ObjectNode steps = state.putObject("steps");
        steps.putObject("draft").put("status", "pending");
        steps.putObject("review").put("status", "pending");

        ProgressCreateRequest request = new ProgressCreateRequest(
                "tenant1", "workitem", UUID.randomUUID().toString(),
                null, state, null, null, null, null, null, "my-workflow");

        ProgressInstance instance = service.create(request);
        assertThat(instance.shapeType()).isEqualTo("step");
        assertThat(instance.definition()).isEqualTo(fullDef);
        assertThat(instance.rollbackPolicy()).isEqualTo("revert-to-previous");
        assertThat(instance.visualisationMode()).isEqualTo("linear");
    }

    @Test
    void createWithDefinitionNameRequestOverridesTemplate() {
        ObjectMapper mapper = new ObjectMapper();
        registry.register(new ProgressDefinition("pct", "percentage", null, "revert-to-previous", "linear"));

        ProgressCreateRequest request = new ProgressCreateRequest(
                "tenant1", "workitem", UUID.randomUUID().toString(),
                "percentage", mapper.createObjectNode().put("value", 0),
                null, null, null, "custom-rollback", null, "pct");

        ProgressInstance instance = service.create(request);
        assertThat(instance.rollbackPolicy()).isEqualTo("custom-rollback");
        assertThat(instance.visualisationMode()).isEqualTo("linear");
    }

    @Test
    void createWithUnknownDefinitionNameThrows() {
        ProgressCreateRequest request = new ProgressCreateRequest(
                "tenant1", "workitem", UUID.randomUUID().toString(),
                null, new ObjectMapper().createObjectNode().put("value", 0),
                null, null, null, null, null, "nonexistent");

        assertThatThrownBy(() -> service.create(request))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Unknown progress definition");
    }
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime -Dtest="ProgressServiceTest#createWithDefinitionNameResolvesFromRegistry+createWithDefinitionNameRequestOverridesTemplate+createWithUnknownDefinitionNameThrows" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `ProgressCreateRequest` doesn't have `definitionName` parameter

- [ ] **Step 3: Add definitionName to ProgressCreateRequest**

Use `ide_replace_member` on `ProgressCreateRequest` — replace the record constructor to add the field. The record declaration changes to:

```java
public record ProgressCreateRequest(
        String tenancyId,
        String scopeType,
        String scopeId,
        String shapeType,
        JsonNode state,
        UUID parentProgressId,
        String rollupStrategyId,
        JsonNode definition,
        String rollbackPolicy,
        String visualisationMode,
        String definitionName
) {
    public ProgressCreateRequest {
        if (tenancyId == null || tenancyId.isBlank()) {
            throw new IllegalArgumentException("tenancyId is required");
        }
        if (scopeType == null || scopeType.isBlank()) {
            throw new IllegalArgumentException("scopeType is required");
        }
        if (scopeId == null || scopeId.isBlank()) {
            throw new IllegalArgumentException("scopeId is required");
        }
        if (definitionName == null) {
            if (shapeType == null || shapeType.isBlank()) {
                throw new IllegalArgumentException("shapeType is required when definitionName is not set");
            }
        }
        if (state == null) {
            throw new IllegalArgumentException("state is required");
        }
    }
}
```

- [ ] **Step 4: Fix existing call sites**

All existing `new ProgressCreateRequest(...)` calls need `, null` appended for the new `definitionName` parameter. Search for `new ProgressCreateRequest(` across:
- `progress-runtime/src/test/java/.../ProgressServiceTest.java` — update all helper methods
- `progress-runtime/src/test/java/.../SubtreeRollbackServiceTest.java` — update all constructors
- `progress-runtime/src/main/java/.../ProgressService.java` — if any internal construction exists
- Any other modules that construct `ProgressCreateRequest`

Use `ide_search_text` to find all call sites, then `ide_replace_text_in_file` for each.

- [ ] **Step 5: Add ProgressDefinitionRegistry to ProgressService constructor**

Add `ProgressDefinitionRegistry` parameter to the `ProgressService` constructor and field. Use `ide_insert_member` for the field, then update the constructor.

Update `ProgressService.create()` to resolve `definitionName`:

```java
public ProgressInstance create(ProgressCreateRequest request) {
    String shapeType = request.shapeType();
    JsonNode definition = request.definition();
    String rollbackPolicy = request.rollbackPolicy();
    String visualisationMode = request.visualisationMode();

    if (request.definitionName() != null) {
        ProgressDefinition def = registry.get(request.definitionName())
                .orElseThrow(() -> new IllegalArgumentException(
                        "Unknown progress definition: " + request.definitionName()));
        if (shapeType == null) shapeType = def.shapeType();
        if (definition == null) definition = def.definition();
        if (rollbackPolicy == null) rollbackPolicy = def.rollbackPolicy();
        if (visualisationMode == null) visualisationMode = def.visualisationMode();
    }

    // ... existing validation and creation logic using resolved values ...
}
```

- [ ] **Step 6: Update ProgressServiceTest setUp to include registry**

Add `ProgressDefinitionRegistry registry = new ProgressDefinitionRegistry();` to `setUp()`, pass to `ProgressService` constructor.

- [ ] **Step 7: Run all progress-runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl progress-runtime`
Expected: ALL PASS (including new definitionName tests and existing tests)

- [ ] **Step 8: Commit**

```bash
git add progress-api/src/main/java/io/casehub/work/progress/ProgressCreateRequest.java \
        progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/ProgressService.java \
        progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/ProgressServiceTest.java \
        progress-runtime/src/test/java/io/casehub/work/progress/runtime/service/SubtreeRollbackServiceTest.java
git commit -m "feat(#373): ProgressCreateRequest.definitionName + ProgressService resolution

ProgressCreateRequest gains optional definitionName field. When set,
ProgressService.create() resolves the named definition from
ProgressDefinitionRegistry, using template values as defaults.
Request fields override template defaults.

Refs #373"
```

---

## References

- [2026-08-30-progress-definition-yaml-design.md](../specs/issue-371-yaml-frontends/2026-08-30-progress-definition-yaml-design.md) — design spec
- `progress-api/src/main/java/io/casehub/work/progress/ProgressCreateRequest.java` — creation request record
- `progress-api/src/main/java/io/casehub/work/progress/StepDefinition.java` — step definition record
- `progress-core/src/main/java/io/casehub/work/progress/validation/StepDefinitionValidator.java` — step validation
- `progress-runtime/src/main/java/io/casehub/work/progress/runtime/service/ProgressService.java:66` — create method
- `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateYamlLoader.java` — YAML loading pattern
- GitHub #373 — focal issue
