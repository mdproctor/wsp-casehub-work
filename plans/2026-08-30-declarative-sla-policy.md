# Declarative SLA Policy Config Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #372 — Declarative SLA policy config (deployment-wide YAML defaults)
**Issue group:** #371, #372, #373

**Goal:** Add a YAML-based `SlaBreachPolicy` implementation that reads deployment-wide SLA breach behavior from a classpath resource with application.yaml config property overrides.

**Architecture:** A `DeclarativeSlaBreachPolicy` (id: `"declarative"`) loads config from `META-INF/work-sla-defaults.yaml` at startup via `SlaDefaultsYamlLoader`. At breach time, it resolves the breach decision by walking `Path.parent()` from the WorkItem's scope through YAML scopes, then defaults, then a configurable fallback CDI policy. `BreachAction` is the intermediate sealed interface between YAML parsing and `BreachDecision` construction.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- All new classes in `io.casehub.work.runtime.service` (runtime module)
- Config prefix: `casehub.work.sla.declarative.*`
- No Flyway migration (in-memory config, no persistence)
- No changes to `ExpiryLifecycleService`
- No changes to SPI types (`SlaBreachPolicy`, `BreachDecision`, `SlaBreachContext`, `BreachedTask`)
- Tests: plain JUnit 5 + AssertJ (no CDI container for unit tests)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

---

## Batch 1: Foundation — BreachAction + SlaDeclarativeConfig

### Task 1: BreachAction sealed interface with parsing and conversion

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/BreachAction.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/BreachActionTest.java`

**Interfaces:**
- Consumes: `io.casehub.work.api.BreachDecision` (existing sealed interface)
- Produces: `BreachAction` sealed interface with:
  - `BreachAction.parse(Object yamlValue)` — parses YAML string or map
  - `BreachAction.parseColon(String configValue)` — parses colon-delimited config property
  - `BreachDecision toBreachDecision(Integer fallbackExtensionHours, int defaultExpiryHours)` — converts to runtime decision

- [ ] **Step 1: Write failing test — string shorthand parsing**

```java
package io.casehub.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Duration;

import org.junit.jupiter.api.Test;

import io.casehub.work.api.BreachDecision;

class BreachActionTest {

    @Test
    void parseStringFail() {
        BreachAction action = BreachAction.parse("fail");
        assertThat(action).isInstanceOf(BreachAction.FailAction.class);
        assertThat(((BreachAction.FailAction) action).reason()).isEqualTo("sla-breach");
    }

    @Test
    void parseStringExtend() {
        BreachAction action = BreachAction.parse("extend");
        assertThat(action).isInstanceOf(BreachAction.ExtendAction.class);
        assertThat(((BreachAction.ExtendAction) action).explicitDuration()).isNull();
    }

    @Test
    void parseStringExhausted() {
        BreachAction action = BreachAction.parse("exhausted");
        assertThat(action).isInstanceOf(BreachAction.ExhaustedAction.class);
        assertThat(((BreachAction.ExhaustedAction) action).reason()).isEqualTo("sla-exhausted");
    }

    @Test
    void parseUnknownStringThrows() {
        assertThatThrownBy(() -> BreachAction.parse("unknown"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BreachActionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `BreachAction` class does not exist

- [ ] **Step 3: Implement BreachAction — string shorthand parsing**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/work/runtime/service/BreachAction.java`:

```java
package io.casehub.work.runtime.service;

import java.time.Duration;
import java.util.Map;

import io.casehub.work.api.BreachDecision;

public sealed interface BreachAction {

    record FailAction(String reason) implements BreachAction {}
    record ExtendAction(Duration explicitDuration) implements BreachAction {}
    record EscalateToAction(String group, Duration deadline) implements BreachAction {}
    record ExhaustedAction(String reason) implements BreachAction {}

    @SuppressWarnings("unchecked")
    static BreachAction parse(Object yamlValue) {
        if (yamlValue instanceof String s) {
            return switch (s) {
                case "fail" -> new FailAction("sla-breach");
                case "extend" -> new ExtendAction(null);
                case "exhausted" -> new ExhaustedAction("sla-exhausted");
                default -> throw new IllegalArgumentException(
                        "Unknown SLA action: '" + s + "' — expected fail, extend, or exhausted");
            };
        }
        if (yamlValue instanceof Map<?, ?> map) {
            return parseMap((Map<String, Object>) map);
        }
        throw new IllegalArgumentException(
                "SLA action must be a string or object, got: " + yamlValue.getClass().getSimpleName());
    }

    private static BreachAction parseMap(Map<String, Object> map) {
        if (map.containsKey("fail")) {
            return new FailAction(String.valueOf(map.get("fail")));
        }
        if (map.containsKey("extend")) {
            Duration d = Duration.parse(String.valueOf(map.get("extend")));
            if (d.isZero() || d.isNegative()) {
                throw new IllegalArgumentException("extend duration must be positive, was: " + d);
            }
            return new ExtendAction(d);
        }
        if (map.containsKey("escalateTo")) {
            String group = String.valueOf(map.get("escalateTo"));
            Duration deadline = null;
            if (map.containsKey("deadline")) {
                deadline = Duration.parse(String.valueOf(map.get("deadline")));
                if (deadline.isZero() || deadline.isNegative()) {
                    throw new IllegalArgumentException("escalateTo deadline must be positive, was: " + deadline);
                }
            }
            return new EscalateToAction(group, deadline);
        }
        if (map.containsKey("exhausted")) {
            return new ExhaustedAction(String.valueOf(map.get("exhausted")));
        }
        throw new IllegalArgumentException(
                "SLA action object must contain one of: fail, extend, escalateTo, exhausted — got keys: " + map.keySet());
    }

    static BreachAction parseColon(String configValue) {
        if (configValue == null || configValue.isBlank()) {
            throw new IllegalArgumentException("SLA config property value must not be blank");
        }
        String[] parts = configValue.split(":", 3);
        return switch (parts[0]) {
            case "fail" -> new FailAction(parts.length > 1 ? parts[1] : "sla-breach");
            case "extend" -> {
                if (parts.length > 1) {
                    Duration d = Duration.parse(parts[1]);
                    if (d.isZero() || d.isNegative()) {
                        throw new IllegalArgumentException("extend duration must be positive, was: " + d);
                    }
                    yield new ExtendAction(d);
                }
                yield new ExtendAction(null);
            }
            case "escalateTo" -> {
                if (parts.length < 2) {
                    throw new IllegalArgumentException("escalateTo requires a group: escalateTo:<group>[:<deadline>]");
                }
                Duration deadline = null;
                if (parts.length > 2) {
                    deadline = Duration.parse(parts[2]);
                    if (deadline.isZero() || deadline.isNegative()) {
                        throw new IllegalArgumentException("escalateTo deadline must be positive, was: " + deadline);
                    }
                }
                yield new EscalateToAction(parts[1], deadline);
            }
            case "exhausted" -> new ExhaustedAction(parts.length > 1 ? parts[1] : "sla-exhausted");
            default -> throw new IllegalArgumentException(
                    "Unknown SLA action: '" + parts[0] + "' — expected fail, extend, escalateTo, or exhausted");
        };
    }

    default BreachDecision toBreachDecision(Integer fallbackExtensionHours, int defaultExpiryHours) {
        return switch (this) {
            case FailAction f -> new BreachDecision.Fail(f.reason());
            case ExtendAction e -> {
                Duration d = e.explicitDuration();
                if (d == null) {
                    int hours = fallbackExtensionHours != null ? fallbackExtensionHours : defaultExpiryHours;
                    d = Duration.ofHours(hours);
                }
                yield new BreachDecision.Extend(d);
            }
            case EscalateToAction e -> {
                var decision = BreachDecision.EscalateTo.to(e.group());
                if (e.deadline() != null) {
                    yield decision.withDeadline(e.deadline());
                }
                int hours = fallbackExtensionHours != null ? fallbackExtensionHours : defaultExpiryHours;
                yield decision.withDeadline(Duration.ofHours(hours));
            }
            case ExhaustedAction e -> new BreachDecision.Exhausted(e.reason());
        };
    }
}
```

- [ ] **Step 4: Run test to verify string shorthand tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BreachActionTest`
Expected: PASS

- [ ] **Step 5: Add remaining tests — map parsing, colon parsing, duration validation, toBreachDecision**

Add to `BreachActionTest`:

```java
    // ── Map parsing ──

    @Test
    void parseMapFail() {
        BreachAction action = BreachAction.parse(Map.of("fail", "custom-reason"));
        assertThat(action).isEqualTo(new BreachAction.FailAction("custom-reason"));
    }

    @Test
    void parseMapExtend() {
        BreachAction action = BreachAction.parse(Map.of("extend", "PT6H"));
        assertThat(action).isEqualTo(new BreachAction.ExtendAction(Duration.ofHours(6)));
    }

    @Test
    void parseMapEscalateTo() {
        BreachAction action = BreachAction.parse(Map.of("escalateTo", "team-leads", "deadline", "PT4H"));
        assertThat(action).isEqualTo(new BreachAction.EscalateToAction("team-leads", Duration.ofHours(4)));
    }

    @Test
    void parseMapEscalateToNoDeadline() {
        BreachAction action = BreachAction.parse(Map.of("escalateTo", "team-leads"));
        assertThat(action).isEqualTo(new BreachAction.EscalateToAction("team-leads", null));
    }

    @Test
    void parseMapExhausted() {
        BreachAction action = BreachAction.parse(Map.of("exhausted", "triage-sla-exceeded"));
        assertThat(action).isEqualTo(new BreachAction.ExhaustedAction("triage-sla-exceeded"));
    }

    @Test
    void parseMapUnknownKeyThrows() {
        assertThatThrownBy(() -> BreachAction.parse(Map.of("unknown", "value")))
                .isInstanceOf(IllegalArgumentException.class);
    }

    // ── Duration validation ──

    @Test
    void parseMapExtendZeroDurationThrows() {
        assertThatThrownBy(() -> BreachAction.parse(Map.of("extend", "PT0S")))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("extend duration must be positive");
    }

    @Test
    void parseMapExtendNegativeDurationThrows() {
        assertThatThrownBy(() -> BreachAction.parse(Map.of("extend", "-PT1H")))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("extend duration must be positive");
    }

    @Test
    void parseMapEscalateToZeroDeadlineThrows() {
        assertThatThrownBy(() -> BreachAction.parse(Map.of("escalateTo", "group", "deadline", "PT0S")))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("escalateTo deadline must be positive");
    }

    @Test
    void parseMapEscalateToNegativeDeadlineThrows() {
        assertThatThrownBy(() -> BreachAction.parse(Map.of("escalateTo", "group", "deadline", "-PT1H")))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("escalateTo deadline must be positive");
    }

    // ── Colon-delimited parsing ──

    @Test
    void parseColonFail() {
        assertThat(BreachAction.parseColon("fail"))
                .isEqualTo(new BreachAction.FailAction("sla-breach"));
    }

    @Test
    void parseColonFailWithReason() {
        assertThat(BreachAction.parseColon("fail:custom-reason"))
                .isEqualTo(new BreachAction.FailAction("custom-reason"));
    }

    @Test
    void parseColonExtend() {
        assertThat(BreachAction.parseColon("extend"))
                .isEqualTo(new BreachAction.ExtendAction(null));
    }

    @Test
    void parseColonExtendWithDuration() {
        assertThat(BreachAction.parseColon("extend:PT6H"))
                .isEqualTo(new BreachAction.ExtendAction(Duration.ofHours(6)));
    }

    @Test
    void parseColonEscalateTo() {
        assertThat(BreachAction.parseColon("escalateTo:team-leads"))
                .isEqualTo(new BreachAction.EscalateToAction("team-leads", null));
    }

    @Test
    void parseColonEscalateToWithDeadline() {
        assertThat(BreachAction.parseColon("escalateTo:team-leads:PT4H"))
                .isEqualTo(new BreachAction.EscalateToAction("team-leads", Duration.ofHours(4)));
    }

    @Test
    void parseColonEscalateToZeroDeadlineThrows() {
        assertThatThrownBy(() -> BreachAction.parseColon("escalateTo:group:PT0S"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("escalateTo deadline must be positive");
    }

    @Test
    void parseColonExhausted() {
        assertThat(BreachAction.parseColon("exhausted"))
                .isEqualTo(new BreachAction.ExhaustedAction("sla-exhausted"));
    }

    @Test
    void parseColonExhaustedWithReason() {
        assertThat(BreachAction.parseColon("exhausted:custom-reason"))
                .isEqualTo(new BreachAction.ExhaustedAction("custom-reason"));
    }

    @Test
    void parseColonMissingGroupThrows() {
        assertThatThrownBy(() -> BreachAction.parseColon("escalateTo"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("requires a group");
    }

    @Test
    void parseColonUnknownThrows() {
        assertThatThrownBy(() -> BreachAction.parseColon("unknown"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    // ── toBreachDecision ──

    @Test
    void failToBreachDecision() {
        BreachDecision d = new BreachAction.FailAction("reason").toBreachDecision(null, 24);
        assertThat(d).isEqualTo(new BreachDecision.Fail("reason"));
    }

    @Test
    void extendWithExplicitDurationToBreachDecision() {
        BreachDecision d = new BreachAction.ExtendAction(Duration.ofHours(6)).toBreachDecision(null, 24);
        assertThat(d).isEqualTo(new BreachDecision.Extend(Duration.ofHours(6)));
    }

    @Test
    void extendWithFallbackHoursToBreachDecision() {
        BreachDecision d = new BreachAction.ExtendAction(null).toBreachDecision(8, 24);
        assertThat(d).isEqualTo(new BreachDecision.Extend(Duration.ofHours(8)));
    }

    @Test
    void extendWithDefaultExpiryHoursToBreachDecision() {
        BreachDecision d = new BreachAction.ExtendAction(null).toBreachDecision(null, 24);
        assertThat(d).isEqualTo(new BreachDecision.Extend(Duration.ofHours(24)));
    }

    @Test
    void escalateToWithDeadlineToBreachDecision() {
        BreachDecision d = new BreachAction.EscalateToAction("team-leads", Duration.ofHours(4))
                .toBreachDecision(null, 24);
        assertThat(d).isEqualTo(BreachDecision.EscalateTo.to("team-leads").withDeadline(Duration.ofHours(4)));
    }

    @Test
    void escalateToWithFallbackHoursDeadlineToBreachDecision() {
        BreachDecision d = new BreachAction.EscalateToAction("team-leads", null)
                .toBreachDecision(8, 24);
        assertThat(d).isEqualTo(BreachDecision.EscalateTo.to("team-leads").withDeadline(Duration.ofHours(8)));
    }

    @Test
    void escalateToWithDefaultExpiryHoursDeadlineToBreachDecision() {
        BreachDecision d = new BreachAction.EscalateToAction("team-leads", null)
                .toBreachDecision(null, 24);
        assertThat(d).isEqualTo(BreachDecision.EscalateTo.to("team-leads").withDeadline(Duration.ofHours(24)));
    }

    @Test
    void exhaustedToBreachDecision() {
        BreachDecision d = new BreachAction.ExhaustedAction("reason").toBreachDecision(null, 24);
        assertThat(d).isEqualTo(new BreachDecision.Exhausted("reason"));
    }
```

- [ ] **Step 6: Run all BreachActionTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BreachActionTest`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/service/BreachAction.java \
        runtime/src/test/java/io/casehub/work/runtime/service/BreachActionTest.java
git commit -m "feat(#372): BreachAction sealed interface — YAML and config property parsing

Intermediate representation between YAML/config parsing and BreachDecision.
Supports string shorthand (fail, extend, exhausted), object form
(escalateTo with optional deadline), and colon-delimited config property
syntax. Validates positive durations for extend and escalateTo deadlines.

Refs #372"
```

### Task 2: SlaDeclarativeConfig record

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/SlaDeclarativeConfig.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/SlaDeclarativeConfigTest.java`

**Interfaces:**
- Consumes: `BreachAction` (Task 1), `io.casehub.platform.api.path.Path`
- Produces: `SlaDeclarativeConfig` record with:
  - `SlaDeclarativeConfig(BreachAction, BreachAction, Integer, Integer, Map<Path, ScopeConfig>)`
  - `ScopeConfig(BreachAction, BreachAction, Integer, Integer)`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Duration;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.path.Path;

class SlaDeclarativeConfigTest {

    @Test
    void constructWithDefaults() {
        var config = new SlaDeclarativeConfig(
                new BreachAction.FailAction("sla-breach"),
                new BreachAction.ExtendAction(null),
                4, 8, Map.of());
        assertThat(config.defaultOnCompletionExpiry()).isInstanceOf(BreachAction.FailAction.class);
        assertThat(config.defaultOnClaimExpiry()).isInstanceOf(BreachAction.ExtendAction.class);
        assertThat(config.extensionHours()).isEqualTo(4);
        assertThat(config.claimExtensionHours()).isEqualTo(8);
        assertThat(config.scopes()).isEmpty();
    }

    @Test
    void constructWithNullableFields() {
        var config = new SlaDeclarativeConfig(null, null, null, null, Map.of());
        assertThat(config.defaultOnCompletionExpiry()).isNull();
        assertThat(config.extensionHours()).isNull();
    }

    @Test
    void constructWithScopes() {
        var scope = new SlaDeclarativeConfig.ScopeConfig(
                new BreachAction.EscalateToAction("senior-reviewers", Duration.ofHours(24)),
                new BreachAction.EscalateToAction("team-leads", null),
                null, null);
        var config = new SlaDeclarativeConfig(null, null, null, null,
                Map.of(Path.parse("casehubio/clinical"), scope));
        assertThat(config.scopes()).hasSize(1);
        assertThat(config.scopes().get(Path.parse("casehubio/clinical"))).isEqualTo(scope);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SlaDeclarativeConfigTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `SlaDeclarativeConfig` class does not exist

- [ ] **Step 3: Implement SlaDeclarativeConfig**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/work/runtime/service/SlaDeclarativeConfig.java`:

```java
package io.casehub.work.runtime.service;

import java.util.Map;

import io.casehub.platform.api.path.Path;

public record SlaDeclarativeConfig(
        BreachAction defaultOnCompletionExpiry,
        BreachAction defaultOnClaimExpiry,
        Integer extensionHours,
        Integer claimExtensionHours,
        Map<Path, ScopeConfig> scopes) {

    public record ScopeConfig(
            BreachAction onCompletionExpiry,
            BreachAction onClaimExpiry,
            Integer extensionHours,
            Integer claimExtensionHours) {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SlaDeclarativeConfigTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/service/SlaDeclarativeConfig.java \
        runtime/src/test/java/io/casehub/work/runtime/service/SlaDeclarativeConfigTest.java
git commit -m "feat(#372): SlaDeclarativeConfig record — immutable config model

Holds parsed YAML defaults and scope map. All fields nullable.
ScopeConfig carries per-scope actions and extensionHours overrides.

Refs #372"
```

---

## Batch 2: Config + Loading — WorkItemsConfig + SlaDefaultsYamlLoader

### Task 3: WorkItemsConfig DeclarativeConfig + SlaDefaultsYamlLoader

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/config/WorkItemsConfig.java:148-170`
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/SlaDefaultsYamlLoader.java`
- Create: `runtime/src/test/resources/META-INF/work-sla-defaults.yaml` (test fixture)
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/SlaDefaultsYamlLoaderTest.java`

**Interfaces:**
- Consumes: `BreachAction.parse()` (Task 1), `SlaDeclarativeConfig` (Task 2), `WorkItemsConfig`
- Produces: `SlaDefaultsYamlLoader` with:
  - `SlaDeclarativeConfig loadedConfig` field (package-private, read by `DeclarativeSlaBreachPolicy`)
  - `WorkItemsConfig.SlaConfig.DeclarativeConfig` interface
  - `WorkItemsConfig.SlaConfig.DeclarativeConfig.DefaultsConfig` interface

- [ ] **Step 1: Add DeclarativeConfig to WorkItemsConfig**

Use `ide_insert_member` to add after `breachPolicy()` in `SlaConfig`:

```java
    DeclarativeConfig declarative();

    interface DeclarativeConfig {
        @WithDefault("no-op")
        String fallback();

        DefaultsConfig defaults();

        interface DefaultsConfig {
            Optional<String> onCompletionExpiry();
            Optional<String> onClaimExpiry();
            OptionalInt extensionHours();
            OptionalInt claimExtensionHours();
        }
    }
```

Add imports: `java.util.Optional`, `java.util.OptionalInt`.

- [ ] **Step 2: Write test fixture YAML**

Create `runtime/src/test/resources/META-INF/work-sla-defaults.yaml`:

```yaml
sla:
  defaults:
    onCompletionExpiry: fail
    onClaimExpiry: extend
    extensionHours: 4
    claimExtensionHours: 8
  scopes:
    casehubio/clinical:
      onCompletionExpiry:
        escalateTo: senior-reviewers
        deadline: PT24H
      onClaimExpiry:
        escalateTo: team-leads
    casehubio/clinical/triage:
      onCompletionExpiry:
        exhausted: triage-sla-exceeded
```

- [ ] **Step 3: Write failing test — YAML loading**

```java
package io.casehub.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Duration;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.path.Path;

class SlaDefaultsYamlLoaderTest {

    @Test
    void loadFromClasspath() {
        SlaDefaultsYamlLoader loader = new SlaDefaultsYamlLoader();
        SlaDeclarativeConfig config = loader.loadFromClasspath();

        assertThat(config).isNotNull();
        assertThat(config.defaultOnCompletionExpiry())
                .isEqualTo(new BreachAction.FailAction("sla-breach"));
        assertThat(config.defaultOnClaimExpiry())
                .isEqualTo(new BreachAction.ExtendAction(null));
        assertThat(config.extensionHours()).isEqualTo(4);
        assertThat(config.claimExtensionHours()).isEqualTo(8);
    }

    @Test
    void loadScopes() {
        SlaDefaultsYamlLoader loader = new SlaDefaultsYamlLoader();
        SlaDeclarativeConfig config = loader.loadFromClasspath();

        assertThat(config.scopes()).hasSize(2);

        var clinical = config.scopes().get(Path.parse("casehubio/clinical"));
        assertThat(clinical).isNotNull();
        assertThat(clinical.onCompletionExpiry())
                .isEqualTo(new BreachAction.EscalateToAction("senior-reviewers", Duration.ofHours(24)));
        assertThat(clinical.onClaimExpiry())
                .isEqualTo(new BreachAction.EscalateToAction("team-leads", null));

        var triage = config.scopes().get(Path.parse("casehubio/clinical/triage"));
        assertThat(triage).isNotNull();
        assertThat(triage.onCompletionExpiry())
                .isEqualTo(new BreachAction.ExhaustedAction("triage-sla-exceeded"));
    }

    @Test
    void variableInterpolation() {
        System.setProperty("test.sla.group", "interpolated-group");
        try {
            String result = SlaDefaultsYamlLoader.interpolate("${sys.test.sla.group}");
            assertThat(result).isEqualTo("interpolated-group");
        } finally {
            System.clearProperty("test.sla.group");
        }
    }

    @Test
    void interpolateNull() {
        assertThat(SlaDefaultsYamlLoader.interpolate(null)).isNull();
    }

    @Test
    void interpolatePlainString() {
        assertThat(SlaDefaultsYamlLoader.interpolate("plain-string")).isEqualTo("plain-string");
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SlaDefaultsYamlLoaderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `SlaDefaultsYamlLoader` class does not exist

- [ ] **Step 5: Implement SlaDefaultsYamlLoader**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/work/runtime/service/SlaDefaultsYamlLoader.java`:

```java
package io.casehub.work.runtime.service;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.platform.api.path.Path;
import io.casehub.work.runtime.config.WorkItemsConfig;
import io.quarkus.runtime.Startup;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.util.Collections;
import java.util.Enumeration;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@ApplicationScoped
@Startup
public class SlaDefaultsYamlLoader {

    private static final Logger LOG = Logger.getLogger(SlaDefaultsYamlLoader.class);
    static final String RESOURCE_PATH = "META-INF/work-sla-defaults.yaml";
    private static final Pattern VAR_PATTERN = Pattern.compile("\\$\\{(env|sys)\\.([^}]+)}");
    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory());

    @Inject
    WorkItemsConfig config;

    SlaDeclarativeConfig loadedConfig;

    @PostConstruct
    void load() {
        SlaDeclarativeConfig yamlConfig = loadFromClasspath();
        this.loadedConfig = mergeConfigOverrides(yamlConfig);
    }

    SlaDeclarativeConfig loadFromClasspath() {
        try {
            Enumeration<URL> resources = Thread.currentThread()
                    .getContextClassLoader().getResources(RESOURCE_PATH);
            var urls = Collections.list(resources);
            if (urls.isEmpty()) {
                LOG.info("No " + RESOURCE_PATH + " found on classpath — declarative SLA policy has no YAML config");
                return null;
            }

            BreachAction defaultOnCompletion = null;
            BreachAction defaultOnClaim = null;
            Integer extensionHours = null;
            Integer claimExtensionHours = null;
            boolean defaultsFound = false;
            Map<Path, SlaDeclarativeConfig.ScopeConfig> scopes = new LinkedHashMap<>();

            for (URL url : urls) {
                LOG.infof("Loading SLA defaults from %s", url);
                try (InputStream is = url.openStream()) {
                    JsonNode root = YAML_MAPPER.readTree(is);
                    JsonNode sla = root.get("sla");
                    if (sla == null) continue;

                    JsonNode defaults = sla.get("defaults");
                    if (defaults != null) {
                        if (defaultsFound) {
                            throw new IllegalStateException(
                                    "Multiple resources contribute sla.defaults — deployment-wide defaults must come from a single source. Conflicting resource: " + url);
                        }
                        defaultsFound = true;
                        if (defaults.has("onCompletionExpiry")) {
                            defaultOnCompletion = parseNodeAction(defaults.get("onCompletionExpiry"));
                        }
                        if (defaults.has("onClaimExpiry")) {
                            defaultOnClaim = parseNodeAction(defaults.get("onClaimExpiry"));
                        }
                        if (defaults.has("extensionHours")) {
                            extensionHours = defaults.get("extensionHours").asInt();
                            if (extensionHours <= 0) {
                                throw new IllegalArgumentException(
                                        "sla.defaults.extensionHours must be positive, was: " + extensionHours);
                            }
                        }
                        if (defaults.has("claimExtensionHours")) {
                            claimExtensionHours = defaults.get("claimExtensionHours").asInt();
                            if (claimExtensionHours <= 0) {
                                throw new IllegalArgumentException(
                                        "sla.defaults.claimExtensionHours must be positive, was: " + claimExtensionHours);
                            }
                        }
                    }

                    JsonNode scopesNode = sla.get("scopes");
                    if (scopesNode != null && scopesNode.isObject()) {
                        scopesNode.fields().forEachRemaining(entry -> {
                            String key = entry.getKey();
                            Path scopePath;
                            try {
                                scopePath = Path.parse(key);
                            } catch (Exception e) {
                                throw new IllegalArgumentException(
                                        "Scope key '" + key + "' is invalid — " + e.getMessage(), e);
                            }
                            if (scopePath.equals(Path.root())) {
                                throw new IllegalArgumentException(
                                        "Scope key '" + key + "' resolves to root — use the defaults section instead");
                            }
                            if (scopes.containsKey(scopePath)) {
                                LOG.warnf("Duplicate scope key '%s' — overwriting with entry from %s", key, url);
                            }
                            scopes.put(scopePath, parseScopeConfig(entry.getValue()));
                        });
                    }
                }
            }
            return new SlaDeclarativeConfig(defaultOnCompletion, defaultOnClaim,
                    extensionHours, claimExtensionHours, Map.copyOf(scopes));
        } catch (IOException e) {
            throw new RuntimeException("Failed to discover " + RESOURCE_PATH, e);
        }
    }

    SlaDeclarativeConfig mergeConfigOverrides(SlaDeclarativeConfig yamlConfig) {
        if (config == null) return yamlConfig;
        var dc = config.sla().declarative().defaults();

        BreachAction onCompletion = yamlConfig != null ? yamlConfig.defaultOnCompletionExpiry() : null;
        BreachAction onClaim = yamlConfig != null ? yamlConfig.defaultOnClaimExpiry() : null;
        Integer extHours = yamlConfig != null ? yamlConfig.extensionHours() : null;
        Integer claimExtHours = yamlConfig != null ? yamlConfig.claimExtensionHours() : null;
        Map<Path, SlaDeclarativeConfig.ScopeConfig> scopes = yamlConfig != null ? yamlConfig.scopes() : Map.of();

        if (dc.onCompletionExpiry().isPresent()) {
            BreachAction override = BreachAction.parseColon(dc.onCompletionExpiry().get());
            if (onCompletion != null) {
                LOG.warnf("SLA default for on-completion-expiry overridden by config property (config: %s; YAML had: %s)",
                        dc.onCompletionExpiry().get(), onCompletion);
            }
            onCompletion = override;
        }
        if (dc.onClaimExpiry().isPresent()) {
            BreachAction override = BreachAction.parseColon(dc.onClaimExpiry().get());
            if (onClaim != null) {
                LOG.warnf("SLA default for on-claim-expiry overridden by config property (config: %s; YAML had: %s)",
                        dc.onClaimExpiry().get(), onClaim);
            }
            onClaim = override;
        }
        if (dc.extensionHours().isPresent()) {
            int override = dc.extensionHours().getAsInt();
            if (override <= 0) {
                throw new IllegalArgumentException(
                        "casehub.work.sla.declarative.defaults.extension-hours must be positive, was: " + override);
            }
            if (extHours != null) {
                LOG.warnf("SLA default for extension-hours overridden by config property (config: %d; YAML had: %d)",
                        override, extHours);
            }
            extHours = override;
        }
        if (dc.claimExtensionHours().isPresent()) {
            int override = dc.claimExtensionHours().getAsInt();
            if (override <= 0) {
                throw new IllegalArgumentException(
                        "casehub.work.sla.declarative.defaults.claim-extension-hours must be positive, was: " + override);
            }
            if (claimExtHours != null) {
                LOG.warnf("SLA default for claim-extension-hours overridden by config property (config: %d; YAML had: %d)",
                        override, claimExtHours);
            }
            claimExtHours = override;
        }

        return new SlaDeclarativeConfig(onCompletion, onClaim, extHours, claimExtHours, scopes);
    }

    private BreachAction parseNodeAction(JsonNode node) {
        Object value = YAML_MAPPER.convertValue(node, Object.class);
        if (value instanceof String s) {
            return BreachAction.parse(interpolate(s));
        }
        return BreachAction.parse(value);
    }

    private SlaDeclarativeConfig.ScopeConfig parseScopeConfig(JsonNode node) {
        BreachAction onCompletion = null;
        BreachAction onClaim = null;
        Integer extHours = null;
        Integer claimExtHours = null;
        if (node.has("onCompletionExpiry")) {
            onCompletion = parseNodeAction(node.get("onCompletionExpiry"));
        }
        if (node.has("onClaimExpiry")) {
            onClaim = parseNodeAction(node.get("onClaimExpiry"));
        }
        if (node.has("extensionHours")) {
            extHours = node.get("extensionHours").asInt();
            if (extHours <= 0) {
                throw new IllegalArgumentException("scope extensionHours must be positive, was: " + extHours);
            }
        }
        if (node.has("claimExtensionHours")) {
            claimExtHours = node.get("claimExtensionHours").asInt();
            if (claimExtHours <= 0) {
                throw new IllegalArgumentException("scope claimExtensionHours must be positive, was: " + claimExtHours);
            }
        }
        return new SlaDeclarativeConfig.ScopeConfig(onCompletion, onClaim, extHours, claimExtHours);
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

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SlaDefaultsYamlLoaderTest`
Expected: PASS

- [ ] **Step 7: Verify compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime`
Expected: BUILD SUCCESS (WorkItemsConfig changes compile with SmallRye config processor)

- [ ] **Step 8: Run ide_diagnostics to check for errors**

Run: `ide_diagnostics` on `runtime/src/main/java/io/casehub/work/runtime/service/SlaDefaultsYamlLoader.java`
Expected: No errors

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/config/WorkItemsConfig.java \
        runtime/src/main/java/io/casehub/work/runtime/service/SlaDefaultsYamlLoader.java \
        runtime/src/test/java/io/casehub/work/runtime/service/SlaDefaultsYamlLoaderTest.java \
        runtime/src/test/resources/META-INF/work-sla-defaults.yaml
git commit -m "feat(#372): SlaDefaultsYamlLoader + WorkItemsConfig.DeclarativeConfig

Loads SLA defaults from META-INF/work-sla-defaults.yaml at startup.
Merges config property overrides for defaults section. Validates
positive extensionHours/claimExtensionHours. Logs WARN on config
property override. Fails fast on conflicting defaults from multiple
resources.

Refs #372"
```

---

## Batch 3: Policy + Integration — DeclarativeSlaBreachPolicy + IT

### Task 4: DeclarativeSlaBreachPolicy implementation

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/DeclarativeSlaBreachPolicy.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/DeclarativeSlaBreachPolicyTest.java`

**Interfaces:**
- Consumes: `SlaDefaultsYamlLoader.loadedConfig` (Task 3), `SlaDeclarativeConfig` (Task 2), `BreachAction` (Task 1), `SlaBreachPolicy`, `StrategyResolver`, `WorkItemsConfig`, `Path`
- Produces: `DeclarativeSlaBreachPolicy` implementing `SlaBreachPolicy` with `id() = "declarative"`

- [ ] **Step 1: Write failing test — scope hierarchy resolution**

```java
package io.casehub.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Duration;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.path.Path;
import io.casehub.platform.api.preferences.MapPreferences;
import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.BreachType;
import io.casehub.work.api.BreachedTask;
import io.casehub.work.api.SlaBreachContext;
import io.casehub.work.api.spi.SlaBreachPolicy;

class DeclarativeSlaBreachPolicyTest {

    private DeclarativeSlaBreachPolicy policy;
    private SlaDefaultsYamlLoader loader;
    private SlaBreachPolicy fallback;
    private int defaultExpiryHours;

    @BeforeEach
    void setUp() {
        defaultExpiryHours = 24;
        fallback = ctx -> new BreachDecision.Fail("fallback-fired");

        Map<Path, SlaDeclarativeConfig.ScopeConfig> scopes = new LinkedHashMap<>();
        scopes.put(Path.parse("casehubio/clinical"), new SlaDeclarativeConfig.ScopeConfig(
                new BreachAction.EscalateToAction("senior-reviewers", Duration.ofHours(24)),
                new BreachAction.EscalateToAction("team-leads", null),
                null, null));
        scopes.put(Path.parse("casehubio/clinical/triage"), new SlaDeclarativeConfig.ScopeConfig(
                new BreachAction.ExhaustedAction("triage-sla-exceeded"),
                null, null, null));

        var config = new SlaDeclarativeConfig(
                new BreachAction.FailAction("sla-breach"),
                new BreachAction.ExtendAction(null),
                4, 8, scopes);

        loader = new SlaDefaultsYamlLoader();
        loader.loadedConfig = config;

        policy = new DeclarativeSlaBreachPolicy();
        policy.loader = loader;
        policy.fallbackPolicyRef = fallback;
        policy.defaultExpiryHours = defaultExpiryHours;
    }

    private SlaBreachContext ctx(String scope, BreachType type) {
        return new SlaBreachContext(type,
                new BreachedTask(UUID.randomUUID(), null, "test", Set.of("group-a")),
                scope != null ? Path.parse(scope) : Path.root(),
                MapPreferences.empty());
    }

    @Test
    void id() {
        assertThat(policy.id()).isEqualTo("declarative");
    }

    // ── Scope resolution ──

    @Test
    void exactScopeMatch() {
        BreachDecision d = policy.onBreach(ctx("casehubio/clinical/triage", BreachType.COMPLETION_EXPIRED));
        assertThat(d).isEqualTo(new BreachDecision.Exhausted("triage-sla-exceeded"));
    }

    @Test
    void parentScopeMatch() {
        BreachDecision d = policy.onBreach(ctx("casehubio/clinical/triage", BreachType.CLAIM_EXPIRED));
        // triage has no onClaimExpiry → walks to parent casehubio/clinical
        assertThat(d).isInstanceOf(BreachDecision.EscalateTo.class);
        assertThat(((BreachDecision.EscalateTo) d).groups()).containsExactly("team-leads");
    }

    @Test
    void defaultsFallback() {
        BreachDecision d = policy.onBreach(ctx("casehubio/finance", BreachType.COMPLETION_EXPIRED));
        // no scope match → defaults: fail
        assertThat(d).isEqualTo(new BreachDecision.Fail("sla-breach"));
    }

    @Test
    void rootScopeGoesToDefaults() {
        BreachDecision d = policy.onBreach(ctx(null, BreachType.COMPLETION_EXPIRED));
        assertThat(d).isEqualTo(new BreachDecision.Fail("sla-breach"));
    }

    @Test
    void claimExpiryDefaults() {
        BreachDecision d = policy.onBreach(ctx("casehubio/finance", BreachType.CLAIM_EXPIRED));
        // defaults onClaimExpiry: extend with claimExtensionHours=8
        assertThat(d).isEqualTo(new BreachDecision.Extend(Duration.ofHours(8)));
    }

    @Test
    void delegatesToFallbackWhenNoMatch() {
        loader.loadedConfig = new SlaDeclarativeConfig(null, null, null, null, Map.of());
        BreachDecision d = policy.onBreach(ctx("casehubio/clinical", BreachType.COMPLETION_EXPIRED));
        assertThat(d).isEqualTo(new BreachDecision.Fail("fallback-fired"));
    }

    @Test
    void delegatesToFallbackWhenNoConfig() {
        loader.loadedConfig = null;
        BreachDecision d = policy.onBreach(ctx("casehubio/clinical", BreachType.COMPLETION_EXPIRED));
        assertThat(d).isEqualTo(new BreachDecision.Fail("fallback-fired"));
    }

    // ── Self-detection guard (R1-02) ──

    @Test
    void escalateToSkippedWhenAlreadyAtTargetGroup() {
        // candidateGroups already contains "senior-reviewers" → skip EscalateTo, walk to defaults
        var ctxAtTarget = new SlaBreachContext(BreachType.COMPLETION_EXPIRED,
                new BreachedTask(UUID.randomUUID(), null, "test", Set.of("senior-reviewers")),
                Path.parse("casehubio/clinical"), MapPreferences.empty());
        BreachDecision d = policy.onBreach(ctxAtTarget);
        // casehubio/clinical has EscalateTo(senior-reviewers) → skipped → defaults: fail
        assertThat(d).isEqualTo(new BreachDecision.Fail("sla-breach"));
    }

    @Test
    void escalateToNotSkippedWhenDifferentGroup() {
        // candidateGroups has "original-group" → EscalateTo fires normally
        BreachDecision d = policy.onBreach(ctx("casehubio/clinical", BreachType.COMPLETION_EXPIRED));
        assertThat(d).isInstanceOf(BreachDecision.EscalateTo.class);
        assertThat(((BreachDecision.EscalateTo) d).groups()).containsExactly("senior-reviewers");
    }

    // ── extensionHours resolution ──

    @Test
    void extendUsesDefaultExtensionHours() {
        BreachDecision d = policy.onBreach(ctx("casehubio/finance", BreachType.CLAIM_EXPIRED));
        assertThat(d).isEqualTo(new BreachDecision.Extend(Duration.ofHours(8)));
    }

    @Test
    void extendUsesScopeExtensionHoursWhenPresent() {
        Map<Path, SlaDeclarativeConfig.ScopeConfig> scopes = new LinkedHashMap<>();
        scopes.put(Path.parse("casehubio/clinical"), new SlaDeclarativeConfig.ScopeConfig(
                null, new BreachAction.ExtendAction(null), 12, null));
        loader.loadedConfig = new SlaDeclarativeConfig(null, null, 4, 8, scopes);

        BreachDecision d = policy.onBreach(ctx("casehubio/clinical", BreachType.CLAIM_EXPIRED));
        // scope has extensionHours=12 but no claimExtensionHours → extensionHours used as fallback
        assertThat(d).isEqualTo(new BreachDecision.Extend(Duration.ofHours(12)));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DeclarativeSlaBreachPolicyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `DeclarativeSlaBreachPolicy` class does not exist

- [ ] **Step 3: Implement DeclarativeSlaBreachPolicy**

Use `ide_create_file` to create `runtime/src/main/java/io/casehub/work/runtime/service/DeclarativeSlaBreachPolicy.java`:

```java
package io.casehub.work.runtime.service;

import io.casehub.platform.api.path.Path;
import io.casehub.platform.api.routing.StrategyResolver;
import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.BreachType;
import io.casehub.work.api.SlaBreachContext;
import io.casehub.work.api.spi.SlaBreachPolicy;
import io.casehub.work.runtime.config.WorkItemsConfig;
import io.quarkus.arc.Unremovable;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.inject.Provider;

@Unremovable
@ApplicationScoped
public class DeclarativeSlaBreachPolicy implements SlaBreachPolicy {

    @Inject SlaDefaultsYamlLoader loader;
    @Inject Provider<StrategyResolver> strategyResolverProvider;
    @Inject WorkItemsConfig config;

    volatile SlaBreachPolicy fallbackPolicyRef;
    int defaultExpiryHours;

    @Override
    public String id() { return "declarative"; }

    @PostConstruct
    void init() {
        if (id().equals(config.sla().declarative().fallback())) {
            throw new IllegalStateException(
                    "casehub.work.sla.declarative.fallback cannot be '" + id() + "' — infinite recursion");
        }
        defaultExpiryHours = config.defaultExpiryHours();
    }

    private SlaBreachPolicy fallbackPolicy() {
        if (fallbackPolicyRef == null) {
            fallbackPolicyRef = strategyResolverProvider.get()
                    .resolve(SlaBreachPolicy.class, config.sla().declarative().fallback());
        }
        return fallbackPolicyRef;
    }

    @Override
    public BreachDecision onBreach(SlaBreachContext context) {
        SlaDeclarativeConfig cfg = loader.loadedConfig;
        if (cfg == null) return fallbackPolicy().onBreach(context);

        java.util.Set<String> groups = context.task().candidateGroups();
        BreachAction action = resolveByScope(cfg, context.scope(), context.breachType(), groups);

        if (action == null) {
            action = switch (context.breachType()) {
                case COMPLETION_EXPIRED -> cfg.defaultOnCompletionExpiry();
                case CLAIM_EXPIRED -> cfg.defaultOnClaimExpiry();
            };
            if (action instanceof BreachAction.EscalateToAction esc
                    && groups.contains(esc.group())) {
                action = null;
            }
        }

        if (action == null) return fallbackPolicy().onBreach(context);

        Integer extHours = resolveExtensionHours(cfg, context.scope(), context.breachType());
        return action.toBreachDecision(extHours, defaultExpiryHours);
    }

    private BreachAction resolveByScope(SlaDeclarativeConfig cfg, Path scope, BreachType type,
                                        java.util.Set<String> candidateGroups) {
        Path current = scope;
        while (current != null && !current.equals(Path.root())) {
            SlaDeclarativeConfig.ScopeConfig sc = cfg.scopes().get(current);
            if (sc != null) {
                BreachAction action = switch (type) {
                    case COMPLETION_EXPIRED -> sc.onCompletionExpiry();
                    case CLAIM_EXPIRED -> sc.onClaimExpiry();
                };
                if (action != null) {
                    if (action instanceof BreachAction.EscalateToAction esc
                            && candidateGroups.contains(esc.group())) {
                        current = current.parent();
                        continue;
                    }
                    return action;
                }
            }
            current = current.parent();
        }
        return null;
    }

    private Integer resolveExtensionHours(SlaDeclarativeConfig cfg, Path scope, BreachType type) {
        Path current = scope;
        while (current != null && !current.equals(Path.root())) {
            SlaDeclarativeConfig.ScopeConfig sc = cfg.scopes().get(current);
            if (sc != null) {
                Integer hours = type == BreachType.CLAIM_EXPIRED
                        ? (sc.claimExtensionHours() != null ? sc.claimExtensionHours() : sc.extensionHours())
                        : sc.extensionHours();
                if (hours != null) return hours;
            }
            current = current.parent();
        }
        return type == BreachType.CLAIM_EXPIRED
                ? (cfg.claimExtensionHours() != null ? cfg.claimExtensionHours() : cfg.extensionHours())
                : cfg.extensionHours();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DeclarativeSlaBreachPolicyTest`
Expected: ALL PASS

- [ ] **Step 5: Run all runtime tests to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime` (use scripts/ timeout wrapper if available)
Expected: ALL PASS — no regressions in existing tests

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/service/DeclarativeSlaBreachPolicy.java \
        runtime/src/test/java/io/casehub/work/runtime/service/DeclarativeSlaBreachPolicyTest.java
git commit -m "feat(#372): DeclarativeSlaBreachPolicy — YAML-based SLA breach policy

Implements SlaBreachPolicy (id: 'declarative'). Resolves breach
decisions by walking Path.parent() through YAML scopes, then defaults,
then configurable fallback CDI policy. Uses Provider<StrategyResolver>
for lazy fallback resolution (avoids circular dependency). Self-reference
guard rejects fallback=declarative at startup.

Refs #372"
```

### Task 5: Integration test

**Files:**
- Create: `integration-tests/src/test/java/io/casehub/work/it/DeclarativeSlaBreachPolicyIT.java`
- Create: `integration-tests/src/test/resources/META-INF/work-sla-defaults.yaml` (IT test fixture)

**Interfaces:**
- Consumes: `DeclarativeSlaBreachPolicy` (Task 4), `WorkItemService`, `WorkItemStore`
- Produces: End-to-end verification that the declarative policy is active and applies YAML config

- [ ] **Step 1: Create IT test fixture YAML**

Create `integration-tests/src/test/resources/META-INF/work-sla-defaults.yaml`:

```yaml
sla:
  defaults:
    onCompletionExpiry: fail
    onClaimExpiry: extend
    extensionHours: 2
  scopes:
    test/escalation:
      onCompletionExpiry:
        escalateTo: senior-reviewers
        deadline: PT1H
```

- [ ] **Step 2: Add IT config property**

Add to `integration-tests/src/test/resources/application.properties`:

```properties
casehub.work.sla.breach-policy=declarative
```

- [ ] **Step 3: Write integration test**

```java
package io.casehub.work.it;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.UUID;

import io.casehub.work.api.WorkItem;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.api.spi.WorkItemStore;
import io.casehub.work.runtime.service.ExpiryLifecycleService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DeclarativeSlaBreachPolicyIT {

    @Inject WorkItemStore workItemStore;
    @Inject ExpiryLifecycleService expiryLifecycleService;

    @Test
    void declarativePolicyAppliesDefaultFail() {
        Instant pastExpiry = Instant.now().minusSeconds(60);
        WorkItem item = workItemStore.put(WorkItem.builder()
                .title("IT: declarative default fail")
                .status(WorkItemStatus.PENDING)
                .scope("test/no-scope-match")
                .expiresAt(pastExpiry)
                .createdBy("system")
                .build());

        expiryLifecycleService.expireItem(item.id());

        WorkItem updated = workItemStore.get(item.id()).orElseThrow();
        assertThat(updated.status()).isEqualTo(WorkItemStatus.EXPIRED);
        assertThat(updated.resolution()).isEqualTo("sla-breach");
    }

    @Test
    void declarativePolicyAppliesScopedEscalation() {
        Instant pastExpiry = Instant.now().minusSeconds(60);
        WorkItem item = workItemStore.put(WorkItem.builder()
                .title("IT: declarative scoped escalation")
                .status(WorkItemStatus.PENDING)
                .scope("test/escalation")
                .candidateGroups("original-group")
                .expiresAt(pastExpiry)
                .createdBy("system")
                .build());

        expiryLifecycleService.expireItem(item.id());

        WorkItem updated = workItemStore.get(item.id()).orElseThrow();
        assertThat(updated.status()).isEqualTo(WorkItemStatus.PENDING);
        assertThat(updated.candidateGroups()).isEqualTo("senior-reviewers");
    }
}
```

- [ ] **Step 4: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests -Dtest=DeclarativeSlaBreachPolicyIT`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add integration-tests/src/test/java/io/casehub/work/it/DeclarativeSlaBreachPolicyIT.java \
        integration-tests/src/test/resources/META-INF/work-sla-defaults.yaml \
        integration-tests/src/test/resources/application.properties
git commit -m "test(#372): integration test for declarative SLA breach policy

End-to-end verification: default fail action and scoped escalation
via YAML-declared config. Activates declarative policy via
casehub.work.sla.breach-policy=declarative in test config.

Refs #372"
```

---

## References

- [2026-08-30-declarative-sla-policy-design.md](../specs/issue-371-yaml-frontends/2026-08-30-declarative-sla-policy-design.md) — design spec this plan implements
- `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java:183` — resolveBreachDecision chain
- `runtime/src/main/java/io/casehub/work/runtime/service/NoOpSlaBreachPolicy.java` — existing default policy pattern
- `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateYamlLoader.java` — YAML loading pattern
- `runtime/src/main/java/io/casehub/work/runtime/config/WorkItemsConfig.java` — config interface
- `runtime/src/test/java/io/casehub/work/runtime/service/ExpiryLifecycleServiceTest.java` — test pattern (plain JUnit, in-memory stubs)
- GE-20260810-724b82 — StrategyResolver is config-driven
- GE-20260522-f7db12 — stateless multi-tier SLA escalation via candidateGroups
- GitHub #372 — focal issue
