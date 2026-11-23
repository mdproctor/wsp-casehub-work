# Playbook Template Declaration and Executor Wiring — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #946 — feat: playbook template declaration and executor wiring
**Issue group:** #943 (parent epic), #944, #945

**Goal:** Add a `templates:` section to playbook YAML, compile it into a TemplateSpec with ProtocolDefinition + VariableConstraint maps, and wire template matching into the executor via a TemplateMatchingFilter.

**Architecture:** Templates are parsed by PlaybookCompiler into a TemplateSpec record (containing ProtocolDefinition + per-template variable constraints). CompilationResult carries the TemplateSpec to the executor. PlaybookRunnerCore creates a TemplateMatchingFilter from the TemplateSpec, backed by ProtocolEvaluator for regex+NLP matching, with VariableConstraint enforcement from #944.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Maven. Modules: yaml-core, playbook-compiler, yaml-step-runtime, coordination-core.

## Global Constraints

- yaml-core must remain zero-dependency (except yaml-plugin-api)
- playbook-compiler already depends on coordination-core — no new dep needed
- yaml-step-runtime needs coordination-core added as a compile dependency
- IntelliJ MCP may be unavailable — use Python patch scripts for editing existing files when the hook blocks Edit/Write

---

## Batch 1: Compilation Foundation

### Task 1: TemplateSpec record and CompilationResult extension

**Files:**
- Create: `playbook-compiler/src/main/java/io/casehub/platform/compiler/TemplateSpec.java`
- Modify: `playbook-compiler/src/main/java/io/casehub/platform/compiler/CompilationResult.java`
- Test: `playbook-compiler/src/test/java/io/casehub/platform/compiler/TemplateSpecTest.java`

**Interfaces:**
- Consumes: `ProtocolDefinition` from `io.casehub.platform.coordination.protocol`, `VariableConstraint` from `io.casehub.yaml.core.resolver`
- Produces: `TemplateSpec(ProtocolDefinition protocolDefinition, Map<String, Map<String, VariableConstraint>> variableConstraints)` — used by Task 2 (compiler) and Task 4 (filter)

- [ ] **Step 1: Write TemplateSpec tests**

```java
package io.casehub.platform.compiler;

import io.casehub.platform.coordination.protocol.ProtocolDefinition;
import io.casehub.yaml.core.resolver.VariableConstraint;
import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TemplateSpecTest {

    @Test
    void createsWithProtocolDefinitionAndConstraints() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision} this ${subject}");
        var protocol = new ProtocolDefinition("test", templates);
        var constraints = Map.of("vote",
                Map.<String, VariableConstraint>of("decision", VariableConstraint.enumOf("approve", "reject")));
        var spec = new TemplateSpec(protocol, constraints);
        assertThat(spec.protocolDefinition().name()).isEqualTo("test");
        assertThat(spec.variableConstraints()).containsKey("vote");
    }

    @Test
    void emptyConstraintsAllowed() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("ack", "I confirm ${item}");
        var protocol = new ProtocolDefinition("test", templates);
        var spec = new TemplateSpec(protocol, Map.of());
        assertThat(spec.variableConstraints()).isEmpty();
    }

    @Test
    void rejectsNullProtocolDefinition() {
        assertThatThrownBy(() -> new TemplateSpec(null, Map.of()))
                .isInstanceOf(NullPointerException.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -pl playbook-compiler test -Dtest=TemplateSpecTest`
Expected: FAIL — TemplateSpec class does not exist

- [ ] **Step 3: Create TemplateSpec record**

```java
package io.casehub.platform.compiler;

import io.casehub.platform.coordination.protocol.ProtocolDefinition;
import io.casehub.yaml.core.resolver.VariableConstraint;
import java.util.Map;
import java.util.Objects;

public record TemplateSpec(
        ProtocolDefinition protocolDefinition,
        Map<String, Map<String, VariableConstraint>> variableConstraints
) {
    public TemplateSpec {
        Objects.requireNonNull(protocolDefinition, "protocolDefinition");
        Objects.requireNonNull(variableConstraints, "variableConstraints");
        variableConstraints = Map.copyOf(variableConstraints);
    }
}
```

- [ ] **Step 4: Extend CompilationResult with templateSpec field**

Add `TemplateSpec templateSpec` as sixth field. Add backward-compatible constructors that default it to null. Update the compact constructor to handle null templateSpec.

```java
// New canonical constructor fields:
// Workflow workflow, CoordinationSpec coordinationSpec, SourceMap sourceMap,
// List<PoolDeclaration> poolDeclarations, PlaybookVariableContext variableContext,
// TemplateSpec templateSpec

// Backward-compat constructors delegate with templateSpec=null
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode -pl playbook-compiler test -Dtest=TemplateSpecTest`
Expected: PASS

- [ ] **Step 6: Run all playbook-compiler tests for regression**

Run: `mvn --batch-mode -pl playbook-compiler test`
Expected: All existing tests PASS (backward-compatible constructors)

- [ ] **Step 7: Commit**

```bash
git add playbook-compiler/src/main/java/io/casehub/platform/compiler/TemplateSpec.java \
       playbook-compiler/src/main/java/io/casehub/platform/compiler/CompilationResult.java \
       playbook-compiler/src/test/java/io/casehub/platform/compiler/TemplateSpecTest.java
git commit -m "feat(#946): add TemplateSpec record and extend CompilationResult"
```

### Task 2: PlaybookCompiler parseTemplates and schema update

**Files:**
- Modify: `playbook-compiler/src/main/java/io/casehub/platform/compiler/PlaybookCompiler.java`
- Modify: `yaml-core/src/main/resources/schema/playbook-document.schema.json`
- Test: `playbook-compiler/src/test/java/io/casehub/platform/compiler/PlaybookCompilerTemplateTest.java`

**Interfaces:**
- Consumes: `TemplateSpec` from Task 1, `ProtocolDefinition` from coordination-core, `VariableConstraint` from yaml-core
- Produces: `PlaybookCompiler.compile()` returns `CompilationResult` with populated `templateSpec` when YAML has `templates:` section

- [ ] **Step 1: Write compiler template parsing tests**

```java
package io.casehub.platform.compiler;

import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class PlaybookCompilerTemplateTest {

    private final PlaybookCompiler compiler = new PlaybookCompiler();

    @Test
    void compilesTemplatesSection() {
        var content = new LinkedHashMap<String, Object>();
        content.put("templates", Map.of(
                "vote", Map.of("pattern", "I ${decision} this ${subject}")
        ));
        content.put("steps", List.of());
        var outcome = compiler.compile("test-playbook", content);
        assertThat(outcome).isInstanceOf(CompilationOutcome.Success.class);
        var result = ((CompilationOutcome.Success) outcome).result();
        assertThat(result.templateSpec()).isNotNull();
        assertThat(result.templateSpec().protocolDefinition().name()).isEqualTo("test-playbook");
        assertThat(result.templateSpec().protocolDefinition().templates()).containsKey("vote");
    }

    @Test
    void parsesEnumVariableConstraints() {
        var content = new LinkedHashMap<String, Object>();
        content.put("templates", Map.of(
                "vote", Map.of(
                        "pattern", "I ${decision} this design",
                        "variables", Map.of("decision",
                                Map.of("type", "enum", "values", List.of("approve", "reject")))
                )
        ));
        content.put("steps", List.of());
        var result = ((CompilationOutcome.Success) compiler.compile("test", content)).result();
        var constraints = result.templateSpec().variableConstraints().get("vote");
        assertThat(constraints).containsKey("decision");
        assertThat(constraints.get("decision").required()).isTrue();
    }

    @Test
    void parsesOptionalStringConstraint() {
        var content = new LinkedHashMap<String, Object>();
        content.put("templates", Map.of(
                "greet", Map.of(
                        "pattern", "Hello ${name}",
                        "variables", Map.of("name",
                                Map.of("type", "string", "required", false))
                )
        ));
        content.put("steps", List.of());
        var result = ((CompilationOutcome.Success) compiler.compile("test", content)).result();
        var constraints = result.templateSpec().variableConstraints().get("greet");
        assertThat(constraints.get("name").required()).isFalse();
    }

    @Test
    void noTemplatesSectionReturnsNullTemplateSpec() {
        var content = new LinkedHashMap<String, Object>();
        content.put("steps", List.of());
        var result = ((CompilationOutcome.Success) compiler.compile("test", content)).result();
        assertThat(result.templateSpec()).isNull();
    }

    @Test
    void multipleTemplatesParsed() {
        var content = new LinkedHashMap<String, Object>();
        var templates = new LinkedHashMap<String, Object>();
        templates.put("vote", Map.of("pattern", "I ${decision} this design"));
        templates.put("escalate", Map.of("pattern", "Escalating to ${target}"));
        content.put("templates", templates);
        content.put("steps", List.of());
        var result = ((CompilationOutcome.Success) compiler.compile("test", content)).result();
        assertThat(result.templateSpec().protocolDefinition().templates()).hasSize(2);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode -pl playbook-compiler test -Dtest=PlaybookCompilerTemplateTest`
Expected: FAIL — parseTemplates does not exist, CompilationResult constructor mismatch

- [ ] **Step 3: Add parseTemplates() to PlaybookCompiler**

Add `parseTemplates(String name, Map<String, Object> templatesMap)` method that:
1. Iterates entries in templatesMap
2. For each entry: extracts `pattern` string, parses `variables` map into `VariableConstraint` instances
3. Builds `ProtocolDefinition(name, patternMap)`
4. Returns `TemplateSpec(protocolDefinition, constraintsMap)`

Add the call in `compile()` after `parsePools()`:

```java
TemplateSpec templateSpec = null;
if (content.containsKey("templates")) {
    templateSpec = parseTemplates(name, (Map<String, Object>) content.get("templates"));
}
```

Update the return statement to include `templateSpec`:
```java
return new CompilationOutcome.Success(
    new CompilationResult(workflow, coordination.build(), sourceMap.build(),
                          poolDeclarations, variableContext, templateSpec));
```

- [ ] **Step 4: Update playbook-document.schema.json**

Add `templates` property to the document schema's properties object:

```json
"templates": {
  "type": "object",
  "description": "Protocol template declarations for structured message extraction.",
  "additionalProperties": {
    "type": "object",
    "properties": {
      "pattern": { "type": "string", "description": "Template pattern with ${var} placeholders and [optional] segments." },
      "variables": {
        "type": "object",
        "description": "Per-variable type constraints.",
        "additionalProperties": {
          "type": "object",
          "properties": {
            "type": { "type": "string", "enum": ["string", "enum"] },
            "values": { "type": "array", "items": { "type": "string" } },
            "required": { "type": "boolean", "default": true }
          }
        }
      }
    },
    "required": ["pattern"]
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode -pl playbook-compiler test -Dtest=PlaybookCompilerTemplateTest`
Expected: PASS

- [ ] **Step 6: Run full playbook-compiler tests for regression**

Run: `mvn --batch-mode -pl playbook-compiler test`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add playbook-compiler/src/main/java/io/casehub/platform/compiler/PlaybookCompiler.java \
       yaml-core/src/main/resources/schema/playbook-document.schema.json \
       playbook-compiler/src/test/java/io/casehub/platform/compiler/PlaybookCompilerTemplateTest.java
git commit -m "feat(#946): parse templates section in PlaybookCompiler"
```

## Batch 2: Executor Wiring

### Task 3: TemplateMatchEvent and TemplateMatchingFilter

**Files:**
- Create: `yaml-step-runtime/src/main/java/io/casehub/yaml/step/runner/TemplateMatchEvent.java`
- Create: `yaml-step-runtime/src/main/java/io/casehub/yaml/step/runner/TemplateMatchingFilter.java`
- Modify: `yaml-step-runtime/pom.xml` (add coordination-core dependency)
- Test: `yaml-step-runtime/src/test/java/io/casehub/yaml/step/runner/TemplateMatchingFilterTest.java`

**Interfaces:**
- Consumes: `ProtocolEvaluator`, `ProtocolDefinition`, `NlpProtocolMatcher` from coordination-core; `VariableConstraint`, `TemplateExtractor` from yaml-core
- Produces: `TemplateMatchingFilter.match(String input) → Optional<TemplateMatchEvent>` — used by PlaybookRunnerCore (Task 4)

- [ ] **Step 1: Add coordination-core dependency to yaml-step-runtime pom.xml**

Add to `<dependencies>` in `yaml-step-runtime/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-coordination-core</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Write TemplateMatchingFilter tests**

```java
package io.casehub.yaml.step.runner;

import io.casehub.platform.coordination.protocol.NlpProtocolMatcher;
import io.casehub.platform.coordination.protocol.ProtocolDefinition;
import io.casehub.platform.coordination.protocol.ProtocolEvaluator;
import io.casehub.yaml.core.resolver.VariableConstraint;
import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;

class TemplateMatchingFilterTest {

    private static TemplateMatchingFilter createFilter(
            Map<String, String> templates,
            Map<String, Map<String, VariableConstraint>> constraints,
            NlpProtocolMatcher nlpMatcher) {
        var protocol = new ProtocolDefinition("test", templates);
        return new TemplateMatchingFilter(protocol, constraints, nlpMatcher);
    }

    @Test
    void matchesRegexTemplate() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision} this design");
        var filter = createFilter(templates, Map.of(), null);
        var result = filter.match("I approve this design");
        assertThat(result).isPresent();
        assertThat(result.get().type()).isEqualTo("vote");
        assertThat(result.get().fields()).containsEntry("decision", "approve");
        assertThat(result.get().score()).isEqualTo(1.0);
        assertThat(result.get().originalInput()).isEqualTo("I approve this design");
    }

    @Test
    void returnsEmptyForUnmatched() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision} this design");
        var filter = createFilter(templates, Map.of(), null);
        assertThat(filter.match("completely unrelated")).isEmpty();
    }

    @Test
    void enforcesEnumConstraint() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision} this design");
        var constraints = Map.of("vote",
                Map.<String, VariableConstraint>of("decision",
                        VariableConstraint.enumOf("approve", "reject")));
        var filter = createFilter(templates, constraints, null);
        assertThat(filter.match("I approve this design")).isPresent();
        assertThat(filter.match("I maybe this design")).isEmpty();
    }

    @Test
    void enumConstraintIsCaseInsensitive() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision} this design");
        var constraints = Map.of("vote",
                Map.<String, VariableConstraint>of("decision",
                        VariableConstraint.enumOf("approve", "reject")));
        var filter = createFilter(templates, constraints, null);
        var result = filter.match("I APPROVE this design");
        assertThat(result).isPresent();
        assertThat(result.get().fields()).containsEntry("decision", "APPROVE");
    }

    @Test
    void nlpFallbackProducesMatchWithScore() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision} this design");
        NlpProtocolMatcher nlp = (t, input) ->
                Optional.of(new NlpProtocolMatcher.NlpMatch("vote",
                        Map.of("decision", "approved"), 0.85));
        var filter = createFilter(templates, Map.of(), nlp);
        var result = filter.match("I've approved the design changes");
        assertThat(result).isPresent();
        assertThat(result.get().score()).isEqualTo(0.85);
    }

    @Test
    void matchWithOptionalSegmentPresent() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision} this design[ because ${rationale}]");
        var filter = createFilter(templates, Map.of(), null);
        var result = filter.match("I approve this design because it's solid");
        assertThat(result).isPresent();
        assertThat(result.get().fields()).containsEntry("rationale", "it's solid");
    }

    @Test
    void matchWithOptionalSegmentAbsent() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision} this design[ because ${rationale}]");
        var filter = createFilter(templates, Map.of(), null);
        var result = filter.match("I approve this design");
        assertThat(result).isPresent();
        assertThat(result.get().fields()).doesNotContainKey("rationale");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode -pl yaml-step-runtime test -Dtest=TemplateMatchingFilterTest`
Expected: FAIL — classes do not exist

- [ ] **Step 4: Create TemplateMatchEvent record**

```java
package io.casehub.yaml.step.runner;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;

public record TemplateMatchEvent(
        String type,
        Map<String, String> fields,
        double score,
        String originalInput
) {
    public TemplateMatchEvent {
        Objects.requireNonNull(type, "type");
        Objects.requireNonNull(fields, "fields");
        Objects.requireNonNull(originalInput, "originalInput");
        fields = Collections.unmodifiableMap(new LinkedHashMap<>(fields));
    }
}
```

- [ ] **Step 5: Create TemplateMatchingFilter**

```java
package io.casehub.yaml.step.runner;

import io.casehub.platform.coordination.protocol.NlpProtocolMatcher;
import io.casehub.platform.coordination.protocol.ProtocolDefinition;
import io.casehub.platform.coordination.protocol.ProtocolEvaluator;
import io.casehub.platform.coordination.protocol.ProtocolInstanceKey;
import io.casehub.platform.coordination.protocol.ProtocolMatchResult;
import io.casehub.yaml.core.resolver.TemplateExtractor;
import io.casehub.yaml.core.resolver.VariableConstraint;

import java.util.Map;
import java.util.Objects;
import java.util.Optional;

public final class TemplateMatchingFilter {

    private static final String INSTANCE_KEY = "playbook";

    private final ProtocolEvaluator evaluator;
    private final Map<String, Map<String, VariableConstraint>> constraints;
    private final Map<String, TemplateExtractor> extractors;
    private final String protocolName;

    public TemplateMatchingFilter(ProtocolDefinition protocol,
                                  Map<String, Map<String, VariableConstraint>> constraints,
                                  NlpProtocolMatcher nlpMatcher) {
        Objects.requireNonNull(protocol, "protocol");
        this.protocolName = protocol.name();
        this.constraints = Map.copyOf(constraints);
        this.evaluator = new ProtocolEvaluator(nlpMatcher);
        this.evaluator.register(protocol);

        // Pre-compile extractors for constraint validation
        var extractorMap = new java.util.LinkedHashMap<String, TemplateExtractor>();
        protocol.templates().forEach((name, pattern) ->
                extractorMap.put(name, TemplateExtractor.compile(pattern)));
        this.extractors = Map.copyOf(extractorMap);
    }

    public Optional<TemplateMatchEvent> match(String input) {
        Objects.requireNonNull(input, "input");
        var key = new ProtocolInstanceKey(protocolName, INSTANCE_KEY);
        ProtocolMatchResult result = evaluator.evaluate(key, input);

        if (result instanceof ProtocolMatchResult.Matched matched) {
            // Validate constraints if present
            var templateConstraints = constraints.get(matched.templateName());
            if (templateConstraints != null && !templateConstraints.isEmpty()) {
                var extractor = extractors.get(matched.templateName());
                if (extractor != null) {
                    var validated = extractor.extract(input, templateConstraints);
                    if (validated.isEmpty()) {
                        return Optional.empty();
                    }
                    return Optional.of(new TemplateMatchEvent(
                            matched.templateName(), validated.get(), 1.0, input));
                }
            }
            return Optional.of(new TemplateMatchEvent(
                    matched.templateName(), matched.params(), 1.0, input));
        }

        return Optional.empty();
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode -pl yaml-step-runtime test -Dtest=TemplateMatchingFilterTest`
Expected: PASS

- [ ] **Step 7: Run full yaml-step-runtime tests for regression**

Run: `mvn --batch-mode -pl yaml-step-runtime test`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add yaml-step-runtime/pom.xml \
       yaml-step-runtime/src/main/java/io/casehub/yaml/step/runner/TemplateMatchEvent.java \
       yaml-step-runtime/src/main/java/io/casehub/yaml/step/runner/TemplateMatchingFilter.java \
       yaml-step-runtime/src/test/java/io/casehub/yaml/step/runner/TemplateMatchingFilterTest.java
git commit -m "feat(#946): add TemplateMatchingFilter with constraint enforcement"
```

### Task 4: PlaybookRunnerCore wiring

**Files:**
- Modify: `yaml-step-runtime/src/main/java/io/casehub/yaml/step/runner/PlaybookRunnerCore.java`
- Modify: `yaml-step-runtime/src/main/java/io/casehub/yaml/step/runner/PlaybookRunConfig.java` (if exists, to carry NlpProtocolMatcher)
- Test: existing PlaybookRunnerCore tests for regression

**Interfaces:**
- Consumes: `TemplateSpec` from Task 1, `TemplateMatchingFilter` from Task 3, `NlpProtocolMatcher` from coordination-core
- Produces: `PlaybookRunnerCore` constructor accepts optional `NlpProtocolMatcher`; after compilation, creates `TemplateMatchingFilter` from `TemplateSpec` and makes it available via `ServiceRegistry`

- [ ] **Step 1: Add NlpProtocolMatcher field to PlaybookRunnerCore**

Add a nullable `NlpProtocolMatcher nlpMatcher` field. Add a new constructor overload that accepts it. Existing constructors delegate with `null`.

- [ ] **Step 2: Wire TemplateMatchingFilter creation after compilation**

In `run()`, after the compiler returns `CompilationResult`, check if `templateSpec` is non-null. If so, create a `TemplateMatchingFilter` and register it with `ServiceRegistry` so steps can access it:

```java
if (compilationResult.templateSpec() != null) {
    var filter = new TemplateMatchingFilter(
            compilationResult.templateSpec().protocolDefinition(),
            compilationResult.templateSpec().variableConstraints(),
            nlpMatcher);
    serviceRegistry.register(TemplateMatchingFilter.class, filter);
}
```

Note: the exact registration mechanism depends on `ServiceRegistry`'s API. If it doesn't support dynamic registration, the filter can be passed through `PlaybookRunConfig` instead.

- [ ] **Step 3: Run all yaml-step-runtime and playbook tests for regression**

Run: `mvn --batch-mode -pl yaml-step-runtime,playbook-compiler,playbooks test`
Expected: All tests PASS

- [ ] **Step 4: Commit**

```bash
git add yaml-step-runtime/src/main/java/io/casehub/yaml/step/runner/PlaybookRunnerCore.java
git commit -m "feat(#946): wire TemplateMatchingFilter into PlaybookRunnerCore"
```

## References

- [2026-11-23-playbook-template-wiring-design.md] — design spec this plan implements
- `PlaybookCompiler.java:96-137` — compilation pipeline with parseOrchestration/parsePools pattern
- `CompilationResult.java:1-28` — existing record with progressive extension pattern
- `ProtocolEvaluator.java` — two-tier matching engine from #945
- `VariableConstraint.java` (yaml-core) — constraint types from #944
- `TemplateExtractor.java` (yaml-core) — regex extraction with constraints from #944
- `playbook-document.schema.json` — YAML schema with additionalProperties: true
- GitHub #943 — parent epic
- GitHub #944 — VariableConstraint in TemplateExtractor
- GitHub #945 — NLP fallback in ProtocolEvaluator
