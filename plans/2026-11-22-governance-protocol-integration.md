# Governance Protocol Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #924 — feat: integrate TemplateExtractor with governance protocol runtime
**Issue group:** #924

**Goal:** Wire `TemplateExtractor` and `TemplateRegistry` into the governance
protocol runtime so protocol definitions declare response templates and the
runtime pattern-matches agent prose responses to extract structured parameters.

**Architecture:** Replace the hollow `PlaybookProtocolEvaluator` skeleton in
`coordination-core/protocol/` with three new types: `ProtocolDefinition`
(record — name + template map), `ProtocolMatchResult` (sealed — Matched/Unmatched),
and `ProtocolEvaluator` (class — register/evaluate using `TemplateRegistry`).
Retain existing advisory types. Delete `PlaybookProtocolEvaluator`.

**Tech Stack:** Java 21+, yaml-core (`TemplateExtractor`, `TemplateRegistry`),
JUnit 5, AssertJ

## Global Constraints

- `coordination-core` already depends on `yaml-core` — no new dependencies
- Thread-safe via `java.util.concurrent` — never `synchronized` (virtual thread pinning)
- Package: `io.casehub.platform.coordination.protocol`
- Zero CDI — `coordination-core` is framework-neutral

---

## Batch 1: Protocol types and evaluator

### Task 1: ProtocolDefinition record + ProtocolMatchResult sealed interface

**Files:**
- Create: `coordination-core/src/main/java/io/casehub/platform/coordination/protocol/ProtocolDefinition.java`
- Create: `coordination-core/src/main/java/io/casehub/platform/coordination/protocol/ProtocolMatchResult.java`
- Test: `coordination-core/src/test/java/io/casehub/platform/coordination/protocol/ProtocolDefinitionTest.java`
- Test: `coordination-core/src/test/java/io/casehub/platform/coordination/protocol/ProtocolMatchResultTest.java`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `ProtocolDefinition(String name, Map<String, String> templates)` — canonical constructor with defensive copy + null checks
  - `ProtocolMatchResult` sealed interface with `Matched(String templateName, Map<String, String> params)` and `Unmatched(String response)`

- [ ] **Step 1: Write ProtocolDefinition test**

```java
package io.casehub.platform.coordination.protocol;

import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ProtocolDefinitionTest {

    @Test
    void constructsWithNameAndTemplates() {
        var templates = Map.of("vote", "I ${decision} this design");
        var def = new ProtocolDefinition("review", templates);
        assertThat(def.name()).isEqualTo("review");
        assertThat(def.templates()).containsExactlyEntriesOf(Map.of("vote", "I ${decision} this design"));
    }

    @Test
    void defensiveCopyPreventsExternalMutation() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("vote", "I ${decision}");
        var def = new ProtocolDefinition("review", templates);
        templates.put("extra", "sneaky");
        assertThat(def.templates()).doesNotContainKey("extra");
    }

    @Test
    void templatesMapIsUnmodifiable() {
        var def = new ProtocolDefinition("review", Map.of("vote", "I ${decision}"));
        assertThatThrownBy(() -> def.templates().put("x", "y"))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void rejectsNullName() {
        assertThatThrownBy(() -> new ProtocolDefinition(null, Map.of()))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void rejectsNullTemplates() {
        assertThatThrownBy(() -> new ProtocolDefinition("review", null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void rejectsEmptyTemplates() {
        assertThatThrownBy(() -> new ProtocolDefinition("review", Map.of()))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f coordination-core/pom.xml test -Dtest=ProtocolDefinitionTest --batch-mode`
Expected: compilation failure — `ProtocolDefinition` does not exist

- [ ] **Step 3: Implement ProtocolDefinition**

```java
package io.casehub.platform.coordination.protocol;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;

public record ProtocolDefinition(String name, Map<String, String> templates) {

    public ProtocolDefinition {
        Objects.requireNonNull(name, "name");
        Objects.requireNonNull(templates, "templates");
        if (templates.isEmpty()) {
            throw new IllegalArgumentException("Protocol must have at least one template");
        }
        templates = Collections.unmodifiableMap(new LinkedHashMap<>(templates));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f coordination-core/pom.xml test -Dtest=ProtocolDefinitionTest --batch-mode`
Expected: all 6 tests PASS

- [ ] **Step 5: Write ProtocolMatchResult test**

```java
package io.casehub.platform.coordination.protocol;

import org.junit.jupiter.api.Test;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ProtocolMatchResultTest {

    @Test
    void matchedHoldsTemplateNameAndParams() {
        var result = new ProtocolMatchResult.Matched("vote", Map.of("decision", "approve"));
        assertThat(result.templateName()).isEqualTo("vote");
        assertThat(result.params()).containsEntry("decision", "approve");
    }

    @Test
    void matchedParamsAreUnmodifiable() {
        var result = new ProtocolMatchResult.Matched("vote", Map.of("decision", "approve"));
        org.assertj.core.api.Assertions.assertThatThrownBy(() -> result.params().put("x", "y"))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void unmatchedHoldsOriginalResponse() {
        var result = new ProtocolMatchResult.Unmatched("some random text");
        assertThat(result.response()).isEqualTo("some random text");
    }

    @Test
    void sealedTypeExhaustivenessViaSwitchExpression() {
        ProtocolMatchResult result = new ProtocolMatchResult.Matched("vote", Map.of());
        String outcome = switch (result) {
            case ProtocolMatchResult.Matched m -> "matched: " + m.templateName();
            case ProtocolMatchResult.Unmatched u -> "unmatched: " + u.response();
        };
        assertThat(outcome).isEqualTo("matched: vote");
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn -f coordination-core/pom.xml test -Dtest=ProtocolMatchResultTest --batch-mode`
Expected: compilation failure — `ProtocolMatchResult` does not exist

- [ ] **Step 7: Implement ProtocolMatchResult**

```java
package io.casehub.platform.coordination.protocol;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;

public sealed interface ProtocolMatchResult {

    record Matched(String templateName, Map<String, String> params) implements ProtocolMatchResult {
        public Matched {
            Objects.requireNonNull(templateName, "templateName");
            Objects.requireNonNull(params, "params");
            params = Collections.unmodifiableMap(new LinkedHashMap<>(params));
        }
    }

    record Unmatched(String response) implements ProtocolMatchResult {
        public Unmatched {
            Objects.requireNonNull(response, "response");
        }
    }
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `mvn -f coordination-core/pom.xml test -Dtest=ProtocolMatchResultTest --batch-mode`
Expected: all 4 tests PASS

- [ ] **Step 9: Commit**

```bash
git add coordination-core/src/main/java/io/casehub/platform/coordination/protocol/ProtocolDefinition.java \
      coordination-core/src/main/java/io/casehub/platform/coordination/protocol/ProtocolMatchResult.java \
      coordination-core/src/test/java/io/casehub/platform/coordination/protocol/ProtocolDefinitionTest.java \
      coordination-core/src/test/java/io/casehub/platform/coordination/protocol/ProtocolMatchResultTest.java
git commit -m "feat(#924): add ProtocolDefinition record and ProtocolMatchResult sealed interface"
```

### Task 2: ProtocolEvaluator + delete PlaybookProtocolEvaluator

**Files:**
- Create: `coordination-core/src/main/java/io/casehub/platform/coordination/protocol/ProtocolEvaluator.java`
- Delete: `coordination-core/src/main/java/io/casehub/platform/coordination/protocol/PlaybookProtocolEvaluator.java` (use `ide_refactor_safe_delete`)
- Test: `coordination-core/src/test/java/io/casehub/platform/coordination/protocol/ProtocolEvaluatorTest.java`

**Interfaces:**
- Consumes:
  - `ProtocolDefinition(String name, Map<String, String> templates)` from Task 1
  - `ProtocolMatchResult.Matched` / `ProtocolMatchResult.Unmatched` from Task 1
  - `ProtocolInstanceKey(String protocolName, String instanceId)` — existing type
  - `TemplateRegistry` from `io.casehub.yaml.core.resolver` — `builder().add(name, template).build()`, `match(input)` → `Optional<Match>`
- Produces:
  - `ProtocolEvaluator()` — no-arg constructor
  - `void register(ProtocolDefinition definition)` — compiles templates into TemplateRegistry, throws `IllegalArgumentException` on duplicate name
  - `ProtocolMatchResult evaluate(ProtocolInstanceKey key, String response)` — matches response, throws `IllegalArgumentException` on unknown protocol
  - `void deregister(String protocolName)` — removes registration
  - `boolean isRegistered(String protocolName)` — check registration

- [ ] **Step 1: Write ProtocolEvaluator test**

```java
package io.casehub.platform.coordination.protocol;

import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ProtocolEvaluatorTest {

    private final ProtocolEvaluator evaluator = new ProtocolEvaluator();

    private static ProtocolDefinition votingProtocol() {
        var templates = new LinkedHashMap<String, String>();
        templates.put("approve", "I ${decision} this design[ because ${rationale}]");
        templates.put("escalate", "I escalate to ${target}");
        return new ProtocolDefinition("voting", templates);
    }

    @Test
    void matchesFirstTemplateWithParams() {
        evaluator.register(votingProtocol());
        var key = new ProtocolInstanceKey("voting", "inst-1");
        var result = evaluator.evaluate(key, "I approve this design because the error handling is solid");
        assertThat(result).isInstanceOf(ProtocolMatchResult.Matched.class);
        var matched = (ProtocolMatchResult.Matched) result;
        assertThat(matched.templateName()).isEqualTo("approve");
        assertThat(matched.params()).containsEntry("decision", "approve");
        assertThat(matched.params()).containsEntry("rationale", "the error handling is solid");
    }

    @Test
    void matchesWithOptionalSegmentAbsent() {
        evaluator.register(votingProtocol());
        var key = new ProtocolInstanceKey("voting", "inst-1");
        var result = evaluator.evaluate(key, "I approve this design");
        assertThat(result).isInstanceOf(ProtocolMatchResult.Matched.class);
        var matched = (ProtocolMatchResult.Matched) result;
        assertThat(matched.templateName()).isEqualTo("approve");
        assertThat(matched.params()).containsEntry("decision", "approve");
        assertThat(matched.params()).doesNotContainKey("rationale");
    }

    @Test
    void matchesSecondTemplate() {
        evaluator.register(votingProtocol());
        var key = new ProtocolInstanceKey("voting", "inst-1");
        var result = evaluator.evaluate(key, "I escalate to the review board");
        assertThat(result).isInstanceOf(ProtocolMatchResult.Matched.class);
        var matched = (ProtocolMatchResult.Matched) result;
        assertThat(matched.templateName()).isEqualTo("escalate");
        assertThat(matched.params()).containsEntry("target", "the review board");
    }

    @Test
    void returnsUnmatchedForNonMatchingResponse() {
        evaluator.register(votingProtocol());
        var key = new ProtocolInstanceKey("voting", "inst-1");
        var result = evaluator.evaluate(key, "I have no opinion");
        assertThat(result).isInstanceOf(ProtocolMatchResult.Unmatched.class);
        var unmatched = (ProtocolMatchResult.Unmatched) result;
        assertThat(unmatched.response()).isEqualTo("I have no opinion");
    }

    @Test
    void multipleProtocolsDispatchCorrectly() {
        evaluator.register(votingProtocol());
        var other = new ProtocolDefinition("ack",
                Map.of("confirm", "I confirm ${item}"));
        evaluator.register(other);

        var voteResult = evaluator.evaluate(
                new ProtocolInstanceKey("voting", "v1"), "I approve this design");
        assertThat(voteResult).isInstanceOf(ProtocolMatchResult.Matched.class);
        assertThat(((ProtocolMatchResult.Matched) voteResult).templateName()).isEqualTo("approve");

        var ackResult = evaluator.evaluate(
                new ProtocolInstanceKey("ack", "a1"), "I confirm the deployment");
        assertThat(ackResult).isInstanceOf(ProtocolMatchResult.Matched.class);
        assertThat(((ProtocolMatchResult.Matched) ackResult).templateName()).isEqualTo("confirm");
    }

    @Test
    void rejectsDuplicateRegistration() {
        evaluator.register(votingProtocol());
        assertThatThrownBy(() -> evaluator.register(votingProtocol()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("voting");
    }

    @Test
    void rejectsEvaluateForUnknownProtocol() {
        var key = new ProtocolInstanceKey("nonexistent", "inst-1");
        assertThatThrownBy(() -> evaluator.evaluate(key, "hello"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("nonexistent");
    }

    @Test
    void deregisterRemovesProtocol() {
        evaluator.register(votingProtocol());
        assertThat(evaluator.isRegistered("voting")).isTrue();
        evaluator.deregister("voting");
        assertThat(evaluator.isRegistered("voting")).isFalse();
    }

    @Test
    void deregisterAllowsReRegistration() {
        evaluator.register(votingProtocol());
        evaluator.deregister("voting");
        evaluator.register(votingProtocol());
        assertThat(evaluator.isRegistered("voting")).isTrue();
    }

    @Test
    void isRegisteredReturnsFalseForUnknown() {
        assertThat(evaluator.isRegistered("unknown")).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f coordination-core/pom.xml test -Dtest=ProtocolEvaluatorTest --batch-mode`
Expected: compilation failure — `ProtocolEvaluator` does not exist

- [ ] **Step 3: Implement ProtocolEvaluator**

```java
package io.casehub.platform.coordination.protocol;

import io.casehub.yaml.core.resolver.TemplateRegistry;

import java.util.Objects;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class ProtocolEvaluator {

    private final ConcurrentMap<String, TemplateRegistry> protocols = new ConcurrentHashMap<>();

    public void register(ProtocolDefinition definition) {
        Objects.requireNonNull(definition, "definition");
        TemplateRegistry.Builder builder = TemplateRegistry.builder();
        definition.templates().forEach(builder::add);
        TemplateRegistry registry = builder.build();
        if (protocols.putIfAbsent(definition.name(), registry) != null) {
            throw new IllegalArgumentException(
                    "Protocol already registered: " + definition.name());
        }
    }

    public ProtocolMatchResult evaluate(ProtocolInstanceKey key, String response) {
        Objects.requireNonNull(key, "key");
        Objects.requireNonNull(response, "response");
        TemplateRegistry registry = protocols.get(key.protocolName());
        if (registry == null) {
            throw new IllegalArgumentException(
                    "Unknown protocol: " + key.protocolName());
        }
        Optional<TemplateRegistry.Match> match = registry.match(response);
        return match.<ProtocolMatchResult>map(m ->
                        new ProtocolMatchResult.Matched(m.name(), m.values()))
                .orElseGet(() -> new ProtocolMatchResult.Unmatched(response));
    }

    public void deregister(String protocolName) {
        protocols.remove(protocolName);
    }

    public boolean isRegistered(String protocolName) {
        return protocols.containsKey(protocolName);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f coordination-core/pom.xml test -Dtest=ProtocolEvaluatorTest --batch-mode`
Expected: all 10 tests PASS

- [ ] **Step 5: Delete PlaybookProtocolEvaluator**

Use `ide_refactor_safe_delete` on `PlaybookProtocolEvaluator.java`.
It has zero external references (confirmed via `ide_search_text`).

- [ ] **Step 6: Run full coordination-core tests**

Run: `mvn -f coordination-core/pom.xml test --batch-mode`
Expected: all tests PASS, no regressions

- [ ] **Step 7: Commit**

```bash
git add coordination-core/src/main/java/io/casehub/platform/coordination/protocol/ProtocolEvaluator.java \
      coordination-core/src/test/java/io/casehub/platform/coordination/protocol/ProtocolEvaluatorTest.java
git rm coordination-core/src/main/java/io/casehub/platform/coordination/protocol/PlaybookProtocolEvaluator.java
git commit -m "feat(#924): add ProtocolEvaluator, delete hollow PlaybookProtocolEvaluator"
```

### Task 3: CLAUDE.md update + full build verification

**Files:**
- Modify: `CLAUDE.md` — update `coordination-core` module description

**Interfaces:**
- Consumes: all types from Tasks 1–2
- Produces: nothing (documentation + verification)

- [ ] **Step 1: Update CLAUDE.md coordination-core description**

Add to the `coordination-core` description in CLAUDE.md's module table:
the protocol package now contains `ProtocolDefinition` (record),
`ProtocolEvaluator` (class, `TemplateRegistry`-backed response matching),
and `ProtocolMatchResult` (sealed: Matched/Unmatched).

- [ ] **Step 2: Run full module build**

Run: `mvn -f coordination-core/pom.xml install --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#924): update coordination-core module description for protocol integration"
```

## References

- [2026-11-22-governance-protocol-integration-design.md] — design spec this plan implements
- `coordination-core/src/main/java/.../protocol/PlaybookProtocolEvaluator.java` — skeleton to delete
- `coordination-core/src/main/java/.../protocol/ProtocolInstanceKey.java` — existing type, retained
- `yaml-core/src/main/java/.../resolver/TemplateExtractor.java` — template compilation + extraction
- `yaml-core/src/main/java/.../resolver/TemplateRegistry.java` — ordered multi-template matching
- GitHub #924 — focal issue
- GitHub #921, #923 — template extraction primitives and optional segments
