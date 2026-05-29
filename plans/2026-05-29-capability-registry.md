# Capability Registry Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce a `Capability` value type that enforces kebab-case format at construction time, a `CapabilityRegistry` SPI for vocabulary governance with config-driven validation mode, and wire creation-time validation into `WorkItemService`.

**Architecture:** New API types (`Capability`, `CapabilityRegistry`, `WorkCapabilities`) live in `casehub-work-api`; default CDI implementations (`PermissiveCapabilityRegistry`, `CapabilityValidator`) live in `casehub-work-core`; `CapabilityParser` (dual strict/lenient parse) and exception mappers live in `runtime`. `WorkerCandidate.capabilities` and `SelectionContext.requiredCapabilities` change from `String`-based to `Set<Capability>` — this is an intentional breaking change at 0.2-SNAPSHOT. DB column and REST wire format stay as comma-separated String.

**Tech Stack:** Java 21, Quarkus CDI (`@DefaultBean`, `@ApplicationScoped`), MicroProfile Config (`@ConfigProperty`), JUnit 5 + AssertJ, `@QuarkusTest`

**Spec:** `specs/issue-220-capability-registry/2026-05-29-capability-registry-design.md`

---

## File Map

**Create:**
- `api/src/main/java/io/casehub/work/api/Capability.java`
- `api/src/main/java/io/casehub/work/api/MalformedCapabilityException.java`
- `api/src/main/java/io/casehub/work/api/ValidationMode.java`
- `api/src/main/java/io/casehub/work/api/UnknownCapabilityException.java`
- `api/src/main/java/io/casehub/work/api/CapabilityRegistry.java`
- `api/src/main/java/io/casehub/work/api/WorkCapabilities.java`
- `api/src/test/java/io/casehub/work/api/CapabilityTest.java`
- `api/src/test/java/io/casehub/work/api/CapabilityRegistryTest.java`
- `core/src/main/java/io/casehub/work/core/strategy/PermissiveCapabilityRegistry.java`
- `core/src/main/java/io/casehub/work/core/strategy/WorkCapabilitiesRegistry.java`
- `core/src/main/java/io/casehub/work/core/strategy/CapabilityValidator.java`
- `core/src/test/java/io/casehub/work/core/strategy/PermissiveCapabilityRegistryTest.java`
- `core/src/test/java/io/casehub/work/core/strategy/WorkCapabilitiesRegistryTest.java`
- `core/src/test/java/io/casehub/work/core/strategy/CapabilityValidatorTest.java`
- `runtime/src/main/java/io/casehub/work/runtime/service/CapabilityParser.java`
- `runtime/src/main/java/io/casehub/work/runtime/api/MalformedCapabilityExceptionMapper.java`
- `runtime/src/main/java/io/casehub/work/runtime/api/UnknownCapabilityExceptionMapper.java`
- `runtime/src/test/java/io/casehub/work/runtime/service/CapabilityParserTest.java`
- `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemCapabilityIT.java`

**Modify:**
- `api/src/main/java/io/casehub/work/api/WorkerCandidate.java` — `Set<String> capabilities` → `Set<Capability>`
- `api/src/main/java/io/casehub/work/api/SelectionContext.java` — `String requiredCapabilities` → `Set<Capability>`
- `api/src/test/java/io/casehub/work/api/WorkerCandidateTest.java` — update to use `Capability`
- `core/src/main/java/io/casehub/work/core/strategy/WorkBroker.java` — remove string parsing; use `Set<Capability>` directly
- `core/src/test/java/io/casehub/work/core/strategy/WorkBrokerTest.java` — update to use `Capability`
- `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java` — constructor-inject `CapabilityValidator`; call `CapabilityParser.parse()` in `create()`
- `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java` — use `CapabilityParser.parseLenient()` in `assign()`

---

## Task 1: `Capability` record + `MalformedCapabilityException`

**Files:**
- Create: `api/src/test/java/io/casehub/work/api/CapabilityTest.java`
- Create: `api/src/main/java/io/casehub/work/api/Capability.java`
- Create: `api/src/main/java/io/casehub/work/api/MalformedCapabilityException.java`

- [ ] **Step 1.1: Write the failing tests**

```java
// api/src/test/java/io/casehub/work/api/CapabilityTest.java
package io.casehub.work.api;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

class CapabilityTest {

    @ParameterizedTest
    @ValueSource(strings = {"legal-review", "a", "legal", "contract-analysis", "review-3", "x2"})
    void of_acceptsValidKebabCase(String id) {
        assertThat(Capability.of(id).id()).isEqualTo(id);
    }

    @ParameterizedTest
    @ValueSource(strings = {"Legal-Review", "legal_review", "legal-", "a--b", "3legal", "-legal", ""})
    void of_rejectsInvalidFormat(String id) {
        assertThatThrownBy(() -> Capability.of(id))
                .isInstanceOf(MalformedCapabilityException.class)
                .hasMessageContaining(id.isEmpty() ? "" : id);
    }

    @Test
    void of_rejectsNull() {
        assertThatThrownBy(() -> Capability.of(null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void equalCapabilities_areEqual() {
        assertThat(Capability.of("legal-review")).isEqualTo(Capability.of("legal-review"));
    }

    @Test
    void exceptionCarriesBadValue() {
        MalformedCapabilityException ex = null;
        try { Capability.of("Bad_Value"); }
        catch (MalformedCapabilityException e) { ex = e; }
        assertThat(ex).isNotNull();
        assertThat(ex.badValues()).containsExactly("Bad_Value");
    }
}
```

- [ ] **Step 1.2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapabilityTest -pl api
```
Expected: compilation failure — `Capability`, `MalformedCapabilityException` not found.

- [ ] **Step 1.3: Write `MalformedCapabilityException`**

```java
// api/src/main/java/io/casehub/work/api/MalformedCapabilityException.java
package io.casehub.work.api;

import java.util.List;

public class MalformedCapabilityException extends IllegalArgumentException {

    private final List<String> badValues;

    public MalformedCapabilityException(String badValue) {
        super("Capability id must be lowercase kebab-case: " + badValue);
        this.badValues = List.of(badValue);
    }

    public List<String> badValues() {
        return badValues;
    }
}
```

- [ ] **Step 1.4: Write `Capability`**

```java
// api/src/main/java/io/casehub/work/api/Capability.java
package io.casehub.work.api;

import java.util.Objects;

/**
 * A validated capability name. Format: lowercase kebab-case only ([a-z][a-z0-9]*(-[a-z0-9]+)*).
 * Construction fails fast with {@link MalformedCapabilityException} on format violations.
 */
public record Capability(String id) {

    public Capability {
        Objects.requireNonNull(id);
        if (!id.matches("[a-z][a-z0-9]*(-[a-z0-9]+)*")) {
            throw new MalformedCapabilityException(id);
        }
    }

    public static Capability of(String id) {
        return new Capability(id);
    }
}
```

- [ ] **Step 1.5: Run to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapabilityTest -pl api
```
Expected: BUILD SUCCESS, 8 tests pass.

- [ ] **Step 1.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add api/src/main/java/io/casehub/work/api/Capability.java api/src/main/java/io/casehub/work/api/MalformedCapabilityException.java api/src/test/java/io/casehub/work/api/CapabilityTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(api): add Capability value type with kebab-case enforcement (Refs #220)"
```

---

## Task 2: `WorkCapabilities`, `ValidationMode`, `UnknownCapabilityException`

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/WorkCapabilities.java`
- Create: `api/src/main/java/io/casehub/work/api/ValidationMode.java`
- Create: `api/src/main/java/io/casehub/work/api/UnknownCapabilityException.java`

No isolated tests at this level — `ValidationMode` and `UnknownCapabilityException` are tested via `CapabilityValidatorTest` in Task 6.

- [ ] **Step 2.1: Write `ValidationMode`**

```java
// api/src/main/java/io/casehub/work/api/ValidationMode.java
package io.casehub.work.api;

public enum ValidationMode {
    /** Reject WorkItem creation if any required capability is not in the registry. */
    STRICT,
    /** Log a warning but proceed. */
    WARN,
    /** No registry check. Default. */
    PERMISSIVE
}
```

- [ ] **Step 2.2: Write `UnknownCapabilityException`**

```java
// api/src/main/java/io/casehub/work/api/UnknownCapabilityException.java
package io.casehub.work.api;

import java.util.List;

public class UnknownCapabilityException extends RuntimeException {

    private final List<String> unknownIds;

    public UnknownCapabilityException(List<Capability> unknown) {
        super("Unknown capabilities: " + unknown.stream().map(Capability::id).toList());
        this.unknownIds = unknown.stream().map(Capability::id).toList();
    }

    /** Capability ids as Strings — safe for JSON serialisation. */
    public List<String> unknownIds() {
        return unknownIds;
    }
}
```

- [ ] **Step 2.3: Write `WorkCapabilities`**

Look up what capabilities are used in the `examples/` module (search for `requiredCapabilities` in `examples/src/`) and include any that are real capability tags. Add at minimum the two shown below; add more if found.

```java
// api/src/main/java/io/casehub/work/api/WorkCapabilities.java
package io.casehub.work.api;

/**
 * Platform-canonical capability name constants.
 * Engine case definitions and worker registrations import from here.
 * Examples and test code import from this class — never the reverse.
 */
public final class WorkCapabilities {

    public static final Capability LEGAL_REVIEW      = Capability.of("legal-review");
    public static final Capability CONTRACT_ANALYSIS = Capability.of("contract-analysis");

    private WorkCapabilities() {}
}
```

- [ ] **Step 2.4: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api
```
Expected: BUILD SUCCESS.

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add api/src/main/java/io/casehub/work/api/WorkCapabilities.java api/src/main/java/io/casehub/work/api/ValidationMode.java api/src/main/java/io/casehub/work/api/UnknownCapabilityException.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(api): add WorkCapabilities constants, ValidationMode, UnknownCapabilityException (Refs #220)"
```

---

## Task 3: `CapabilityRegistry` SPI + delegation test

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/CapabilityRegistry.java`
- Create: `api/src/test/java/io/casehub/work/api/CapabilityRegistryTest.java`

- [ ] **Step 3.1: Write the failing test**

```java
// api/src/test/java/io/casehub/work/api/CapabilityRegistryTest.java
package io.casehub.work.api;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Set;

import org.junit.jupiter.api.Test;

class CapabilityRegistryTest {

    @Test
    void isKnown_delegatesToCapabilities_whenNotOverridden() {
        // Subclass overrides only capabilities() — isKnown() must reflect it.
        // Guard against GE-20260511-a5f47d: static-field-backed isKnown() that bypasses capabilities().
        final Capability legal = Capability.of("legal-review");
        final Capability audit = Capability.of("audit-sign");

        final CapabilityRegistry registry = () -> Set.of(legal);

        assertThat(registry.isKnown(legal)).isTrue();
        assertThat(registry.isKnown(audit)).isFalse();
    }

    @Test
    void isKnown_canBeOverriddenForEfficiency() {
        // Implementors may override isKnown() for DB-backed lookups — must still work correctly.
        final Capability known = Capability.of("legal-review");
        final CapabilityRegistry registry = new CapabilityRegistry() {
            @Override public Set<Capability> capabilities() { return Set.of(known); }
            @Override public boolean isKnown(Capability tag) { return tag.equals(known); }
        };

        assertThat(registry.isKnown(known)).isTrue();
        assertThat(registry.isKnown(Capability.of("other"))).isFalse();
    }
}
```

- [ ] **Step 3.2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapabilityRegistryTest -pl api
```
Expected: compilation failure — `CapabilityRegistry` not found.

- [ ] **Step 3.3: Write `CapabilityRegistry`**

```java
// api/src/main/java/io/casehub/work/api/CapabilityRegistry.java
package io.casehub.work.api;

import java.util.Set;

/**
 * SPI for known capability vocabulary.
 *
 * <p>The default implementation ({@link io.casehub.work.core.strategy.PermissiveCapabilityRegistry})
 * returns an empty set — no enforcement. Deploy an {@code @ApplicationScoped @Alternative @Priority(1)}
 * implementation to govern capability vocabulary.
 *
 * <p>Validation mode (STRICT / WARN / PERMISSIVE) is configured via
 * {@code casehub.work.capability-validation} — it is a deployment concern, not a registry concern.
 */
public interface CapabilityRegistry {

    /** Known capability vocabulary. Empty set means unmanaged (no enforcement). */
    Set<Capability> capabilities();

    /**
     * Returns true if {@code tag} is a known capability.
     *
     * <p>Matching is exact and case-sensitive. The {@link Capability} constructor enforces
     * lowercase kebab-case, so format violations are rejected before reaching this method.
     *
     * <p>Override when direct lookup is more efficient than loading {@link #capabilities()}
     * (e.g. a database-backed registry with {@code SELECT EXISTS}). Never back this method
     * with a static field — that silently bypasses subclass capability sets (GE-20260511-a5f47d).
     */
    default boolean isKnown(Capability tag) {
        return capabilities().contains(tag);
    }
}
```

- [ ] **Step 3.4: Run to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapabilityRegistryTest -pl api
```
Expected: BUILD SUCCESS, 2 tests pass.

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add api/src/main/java/io/casehub/work/api/CapabilityRegistry.java api/src/test/java/io/casehub/work/api/CapabilityRegistryTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(api): add CapabilityRegistry SPI (Refs #220)"
```

---

## Task 4: `PermissiveCapabilityRegistry`

**Files:**
- Create: `core/src/main/java/io/casehub/work/core/strategy/PermissiveCapabilityRegistry.java`
- Create: `core/src/test/java/io/casehub/work/core/strategy/PermissiveCapabilityRegistryTest.java`

- [ ] **Step 4.1: Write the failing test**

```java
// core/src/test/java/io/casehub/work/core/strategy/PermissiveCapabilityRegistryTest.java
package io.casehub.work.core.strategy;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

import io.casehub.work.api.Capability;

class PermissiveCapabilityRegistryTest {

    private final PermissiveCapabilityRegistry registry = new PermissiveCapabilityRegistry();

    @Test
    void capabilities_returnsEmptySet() {
        assertThat(registry.capabilities()).isEmpty();
    }

    @Test
    void isKnown_returnsFalse_forAnyCapability() {
        assertThat(registry.isKnown(Capability.of("legal-review"))).isFalse();
    }
}
```

- [ ] **Step 4.2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PermissiveCapabilityRegistryTest -pl core
```
Expected: compilation failure.

- [ ] **Step 4.3: Write `PermissiveCapabilityRegistry`**

```java
// core/src/main/java/io/casehub/work/core/strategy/PermissiveCapabilityRegistry.java
package io.casehub.work.core.strategy;

import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;

import io.casehub.work.api.Capability;
import io.casehub.work.api.CapabilityRegistry;

/**
 * Default CapabilityRegistry — no enforcement.
 * Displaced automatically by any application-scoped {@link CapabilityRegistry} in the deploying app.
 */
@DefaultBean
@ApplicationScoped
public class PermissiveCapabilityRegistry implements CapabilityRegistry {

    @Override
    public Set<Capability> capabilities() {
        return Set.of();
    }
}
```

- [ ] **Step 4.4: Run to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PermissiveCapabilityRegistryTest -pl core
```
Expected: BUILD SUCCESS.

- [ ] **Step 4.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add core/src/main/java/io/casehub/work/core/strategy/PermissiveCapabilityRegistry.java core/src/test/java/io/casehub/work/core/strategy/PermissiveCapabilityRegistryTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(core): add PermissiveCapabilityRegistry default (Refs #220)"
```

---

## Task 5: `WorkCapabilitiesRegistry`

**Files:**
- Create: `core/src/main/java/io/casehub/work/core/strategy/WorkCapabilitiesRegistry.java`
- Create: `core/src/test/java/io/casehub/work/core/strategy/WorkCapabilitiesRegistryTest.java`

- [ ] **Step 5.1: Write the failing test**

```java
// core/src/test/java/io/casehub/work/core/strategy/WorkCapabilitiesRegistryTest.java
package io.casehub.work.core.strategy;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

import io.casehub.work.api.Capability;
import io.casehub.work.api.WorkCapabilities;

class WorkCapabilitiesRegistryTest {

    private final WorkCapabilitiesRegistry registry = new WorkCapabilitiesRegistry();

    @Test
    void capabilities_containsAllWorkCapabilitiesConstants() {
        assertThat(registry.capabilities())
                .contains(WorkCapabilities.LEGAL_REVIEW, WorkCapabilities.CONTRACT_ANALYSIS);
    }

    @Test
    void isKnown_trueForKnownConstant() {
        assertThat(registry.isKnown(WorkCapabilities.LEGAL_REVIEW)).isTrue();
    }

    @Test
    void isKnown_falseForUnknownCapability() {
        assertThat(registry.isKnown(Capability.of("unknown-cap"))).isFalse();
    }
}
```

- [ ] **Step 5.2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCapabilitiesRegistryTest -pl core
```
Expected: compilation failure.

- [ ] **Step 5.3: Write `WorkCapabilitiesRegistry`**

```java
// core/src/main/java/io/casehub/work/core/strategy/WorkCapabilitiesRegistry.java
package io.casehub.work.core.strategy;

import java.util.Set;

import io.casehub.work.api.Capability;
import io.casehub.work.api.CapabilityRegistry;
import io.casehub.work.api.WorkCapabilities;

/**
 * Optional CapabilityRegistry backed by {@link WorkCapabilities} constants.
 *
 * <p>Not a CDI bean — instantiated by teams who want platform-vocabulary enforcement:
 * <pre>{@code
 * @ApplicationScoped @Alternative @Priority(1)
 * public class MyRegistry extends WorkCapabilitiesRegistry { }
 * }</pre>
 *
 * Or extend it and override {@link #capabilities()} to include domain-specific capabilities
 * in addition to the platform vocabulary.
 */
public class WorkCapabilitiesRegistry implements CapabilityRegistry {

    private static final Set<Capability> KNOWN = Set.of(
            WorkCapabilities.LEGAL_REVIEW,
            WorkCapabilities.CONTRACT_ANALYSIS
    );

    @Override
    public Set<Capability> capabilities() {
        return KNOWN;
    }
}
```

- [ ] **Step 5.4: Run to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkCapabilitiesRegistryTest -pl core
```
Expected: BUILD SUCCESS.

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add core/src/main/java/io/casehub/work/core/strategy/WorkCapabilitiesRegistry.java core/src/test/java/io/casehub/work/core/strategy/WorkCapabilitiesRegistryTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(core): add WorkCapabilitiesRegistry opt-in base class (Refs #220)"
```

---

## Task 6: `CapabilityValidator`

**Files:**
- Create: `core/src/main/java/io/casehub/work/core/strategy/CapabilityValidator.java`
- Create: `core/src/test/java/io/casehub/work/core/strategy/CapabilityValidatorTest.java`

- [ ] **Step 6.1: Write the failing tests**

```java
// core/src/test/java/io/casehub/work/core/strategy/CapabilityValidatorTest.java
package io.casehub.work.core.strategy;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNoException;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.Set;

import org.junit.jupiter.api.Test;

import io.casehub.work.api.Capability;
import io.casehub.work.api.CapabilityRegistry;
import io.casehub.work.api.UnknownCapabilityException;
import io.casehub.work.api.ValidationMode;
import io.casehub.work.api.WorkCapabilities;

class CapabilityValidatorTest {

    private final CapabilityRegistry registryWithLegal =
            () -> Set.of(WorkCapabilities.LEGAL_REVIEW);

    private CapabilityValidator validatorWith(ValidationMode mode, CapabilityRegistry registry) {
        return new CapabilityValidator(mode, registry);
    }

    @Test
    void permissive_alwaysPasses_evenForUnknownCapability() {
        final CapabilityValidator v = validatorWith(ValidationMode.PERMISSIVE, registryWithLegal);
        assertThatNoException().isThrownBy(
                () -> v.validate(Set.of(Capability.of("completely-unknown"))));
    }

    @Test
    void strict_passes_whenAllCapabilitiesKnown() {
        final CapabilityValidator v = validatorWith(ValidationMode.STRICT, registryWithLegal);
        assertThatNoException().isThrownBy(
                () -> v.validate(Set.of(WorkCapabilities.LEGAL_REVIEW)));
    }

    @Test
    void strict_throws_whenAnyCapabilityUnknown() {
        final CapabilityValidator v = validatorWith(ValidationMode.STRICT, registryWithLegal);
        final Capability unknown = Capability.of("audit-sign");
        assertThatThrownBy(() -> v.validate(Set.of(WorkCapabilities.LEGAL_REVIEW, unknown)))
                .isInstanceOf(UnknownCapabilityException.class)
                .satisfies(ex -> assertThat(((UnknownCapabilityException) ex).unknownIds())
                        .containsExactly("audit-sign"));
    }

    @Test
    void strict_throwsWithAllUnknownValues_whenMultipleUnknown() {
        final CapabilityValidator v = validatorWith(ValidationMode.STRICT, registryWithLegal);
        assertThatThrownBy(() -> v.validate(
                Set.of(Capability.of("audit-sign"), Capability.of("risk-assess"))))
                .isInstanceOf(UnknownCapabilityException.class)
                .satisfies(ex -> assertThat(((UnknownCapabilityException) ex).unknownIds())
                        .containsExactlyInAnyOrder("audit-sign", "risk-assess"));
    }

    @Test
    void warn_doesNotThrow_forUnknownCapability() {
        final CapabilityValidator v = validatorWith(ValidationMode.WARN, registryWithLegal);
        assertThatNoException().isThrownBy(
                () -> v.validate(Set.of(Capability.of("audit-sign"))));
    }

    @Test
    void emptySet_alwaysPasses_inAnyMode() {
        for (ValidationMode mode : ValidationMode.values()) {
            assertThatNoException().isThrownBy(
                    () -> validatorWith(mode, registryWithLegal).validate(Set.of()));
        }
    }
}
```

- [ ] **Step 6.2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapabilityValidatorTest -pl core
```
Expected: compilation failure.

- [ ] **Step 6.3: Write `CapabilityValidator`**

```java
// core/src/main/java/io/casehub/work/core/strategy/CapabilityValidator.java
package io.casehub.work.core.strategy;

import java.util.List;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import io.casehub.work.api.Capability;
import io.casehub.work.api.CapabilityRegistry;
import io.casehub.work.api.UnknownCapabilityException;
import io.casehub.work.api.ValidationMode;

@ApplicationScoped
public class CapabilityValidator {

    private static final Logger LOG = Logger.getLogger(CapabilityValidator.class);

    private final ValidationMode validationMode;
    private final CapabilityRegistry registry;

    @Inject
    public CapabilityValidator(
            @ConfigProperty(name = "casehub.work.capability-validation",
                            defaultValue = "PERMISSIVE") ValidationMode validationMode,
            CapabilityRegistry registry) {
        this.validationMode = validationMode;
        this.registry = registry;
    }

    /** For unit tests — bypasses CDI and config. */
    CapabilityValidator(ValidationMode validationMode, CapabilityRegistry registry) {
        this.validationMode = validationMode;
        this.registry = registry;
    }

    /**
     * Validates that all capabilities in the set are known to the registry.
     *
     * <p>Precondition: {@code capabilities} is non-null — use {@link
     * io.casehub.work.runtime.service.CapabilityParser#parse} or
     * {@link io.casehub.work.runtime.service.CapabilityParser#parseLenient} to produce the argument.
     *
     * <p>No-op in PERMISSIVE mode or when the set is empty.
     */
    public void validate(Set<Capability> capabilities) {
        if (validationMode == ValidationMode.PERMISSIVE || capabilities.isEmpty()) {
            return;
        }
        final List<Capability> unknown = capabilities.stream()
                .filter(c -> !registry.isKnown(c))
                .toList();
        if (unknown.isEmpty()) {
            return;
        }
        if (validationMode == ValidationMode.STRICT) {
            throw new UnknownCapabilityException(unknown);
        } else {
            LOG.warnf("WorkItem references unregistered capabilities: %s",
                    unknown.stream().map(Capability::id).toList());
        }
    }
}
```

- [ ] **Step 6.4: Run to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapabilityValidatorTest -pl core
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 6.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  core/src/main/java/io/casehub/work/core/strategy/CapabilityValidator.java \
  core/src/test/java/io/casehub/work/core/strategy/CapabilityValidatorTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(core): add CapabilityValidator with STRICT/WARN/PERMISSIVE modes (Refs #220)"
```

---

## Task 7: Breaking SPI changes — `WorkerCandidate`, `SelectionContext`, `WorkBroker`

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/WorkerCandidate.java`
- Modify: `api/src/main/java/io/casehub/work/api/SelectionContext.java`
- Modify: `api/src/test/java/io/casehub/work/api/WorkerCandidateTest.java`
- Modify: `core/src/main/java/io/casehub/work/core/strategy/WorkBroker.java`
- Modify: `core/src/test/java/io/casehub/work/core/strategy/WorkBrokerTest.java`

This task is a coordinated breaking change. Make all changes, then fix compilation errors, then run tests.

- [ ] **Step 7.1: Update `WorkerCandidate`**

Change `Set<String> capabilities` to `Set<Capability>`. Full file:

```java
// api/src/main/java/io/casehub/work/api/WorkerCandidate.java
package io.casehub.work.api;

import java.util.Set;

/**
 * A potential assignee for a WorkItem.
 *
 * <p>CaseHub alignment: corresponds to {@code HumanWorkerProfile}.
 */
public record WorkerCandidate(String id, Set<Capability> capabilities, int activeWorkItemCount) {

    /** Convenience factory — capabilities empty, workload 0. */
    public static WorkerCandidate of(final String id) {
        return new WorkerCandidate(id, Set.of(), 0);
    }

    /** Returns a new candidate with the updated active WorkItem count. */
    public WorkerCandidate withActiveWorkItemCount(final int count) {
        return new WorkerCandidate(id, capabilities, count);
    }
}
```

- [ ] **Step 7.2: Update `SelectionContext`**

Change `String requiredCapabilities` to `Set<Capability>`. Full file:

```java
// api/src/main/java/io/casehub/work/api/SelectionContext.java
package io.casehub.work.api;

import java.util.Set;

/**
 * Minimal WorkItem context passed to {@link WorkerSelectionStrategy#select}.
 *
 * <p>Decouples strategies from the WorkItem JPA entity.
 *
 * @param category WorkItem category (may be null)
 * @param priority WorkItemPriority name e.g. "HIGH" (may be null)
 * @param requiredCapabilities capabilities the assignee must possess (empty = no requirement);
 *     matched against worker capability tags using exact case-sensitive equality
 * @param candidateGroups comma-separated group names (may be null)
 * @param candidateUsers comma-separated user IDs (may be null)
 * @param title work item title — used by semantic matchers (may be null)
 * @param description work item description — used by semantic matchers (may be null)
 * @param excludedUsers comma-separated user IDs excluded from this WorkItem (may be null)
 */
public record SelectionContext(
        String category,
        String priority,
        Set<Capability> requiredCapabilities,
        String candidateGroups,
        String candidateUsers,
        String title,
        String description,
        String excludedUsers) {
}
```

- [ ] **Step 7.3: Update `WorkBroker`**

Remove the string-parsing logic — `SelectionContext` now carries `Set<Capability>` directly:

```java
// core/src/main/java/io/casehub/work/core/strategy/WorkBroker.java
package io.casehub.work.core.strategy;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.work.api.AssignmentDecision;
import io.casehub.work.api.AssignmentTrigger;
import io.casehub.work.api.SelectionContext;
import io.casehub.work.api.WorkerCandidate;
import io.casehub.work.api.WorkerSelectionStrategy;

/**
 * Generic work assignment broker.
 *
 * <p>Applies trigger gating, capability filtering, and strategy dispatch.
 * Does not know about any specific work-unit type.
 */
@ApplicationScoped
public class WorkBroker {

    public AssignmentDecision apply(
            final SelectionContext context,
            final AssignmentTrigger trigger,
            final List<WorkerCandidate> candidates,
            final WorkerSelectionStrategy strategy) {

        if (!strategy.triggers().contains(trigger)) {
            return AssignmentDecision.noChange();
        }
        final List<WorkerCandidate> filtered = filterByCapabilities(context, candidates);
        return strategy.select(context, filtered);
    }

    /**
     * Filters candidates to those possessing all required capabilities.
     *
     * <p>Capability matching is exact and case-sensitive — enforced by the {@link
     * io.casehub.work.api.Capability} constructor. Engine case definitions and worker
     * registrations must use identical {@link io.casehub.work.api.Capability} values.
     */
    private List<WorkerCandidate> filterByCapabilities(
            final SelectionContext context, final List<WorkerCandidate> candidates) {
        if (context.requiredCapabilities().isEmpty()) {
            return candidates;
        }
        return candidates.stream()
                .filter(c -> c.capabilities().containsAll(context.requiredCapabilities()))
                .toList();
    }
}
```

- [ ] **Step 7.4: Fix `WorkerCandidateTest`**

Update to use `Capability`:

```java
// api/src/test/java/io/casehub/work/api/WorkerCandidateTest.java
package io.casehub.work.api;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Set;

import org.junit.jupiter.api.Test;

class WorkerCandidateTest {

    @Test
    void of_createsCandidate_withEmptyCapabilitiesAndZeroCount() {
        final WorkerCandidate c = WorkerCandidate.of("alice");
        assertThat(c.id()).isEqualTo("alice");
        assertThat(c.capabilities()).isEmpty();
        assertThat(c.activeWorkItemCount()).isZero();
    }

    @Test
    void fullConstructor_setsAllFields() {
        final WorkerCandidate c = new WorkerCandidate(
                "bob",
                Set.of(Capability.of("finance"), Capability.of("legal-review")),
                3);
        assertThat(c.id()).isEqualTo("bob");
        assertThat(c.capabilities()).containsExactlyInAnyOrder(
                Capability.of("finance"), Capability.of("legal-review"));
        assertThat(c.activeWorkItemCount()).isEqualTo(3);
    }

    @Test
    void withActiveWorkItemCount_returnsNewCandidateWithUpdatedCount() {
        final WorkerCandidate original = WorkerCandidate.of("carol");
        final WorkerCandidate updated = original.withActiveWorkItemCount(7);
        assertThat(updated.id()).isEqualTo("carol");
        assertThat(updated.activeWorkItemCount()).isEqualTo(7);
        assertThat(original.activeWorkItemCount()).isZero();
    }
}
```

- [ ] **Step 7.5: Fix compilation in `core` (including `WorkBrokerTest`)**

Run to find all compilation errors in `core`:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl core 2>&1 | grep "error:"
```

Open `WorkBrokerTest` and update any `SelectionContext` constructor calls that pass `String requiredCapabilities` — change to `Set.of(Capability.of("tag"))` or `Set.of()` for no-requirement cases. Also update any `WorkerCandidate` constructions that use `Set<String>` — change to `Set<Capability>`.

Pattern to look for in `WorkBrokerTest`:
```java
// BEFORE — String-based
new SelectionContext(cat, prio, "legal-review", groups, users, title, desc, excluded)
new WorkerCandidate("alice", Set.of("legal-review"), 0)

// AFTER — Capability-based
new SelectionContext(cat, prio, Set.of(Capability.of("legal-review")), groups, users, title, desc, excluded)
new WorkerCandidate("alice", Set.of(Capability.of("legal-review")), 0)
```

- [ ] **Step 7.6: Run `api` and `core` tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,core
```
Expected: BUILD SUCCESS. All existing tests pass. Fix any remaining compilation failures before proceeding.

- [ ] **Step 7.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  api/src/main/java/io/casehub/work/api/WorkerCandidate.java \
  api/src/main/java/io/casehub/work/api/SelectionContext.java \
  api/src/test/java/io/casehub/work/api/WorkerCandidateTest.java \
  core/src/main/java/io/casehub/work/core/strategy/WorkBroker.java \
  core/src/test/java/io/casehub/work/core/strategy/WorkBrokerTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "refactor(api,core): WorkerCandidate + SelectionContext use Set<Capability>; WorkBroker drops string parsing (Refs #220)"
```

---

## Task 8: `CapabilityParser`

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/CapabilityParser.java`
- Create: `runtime/src/test/java/io/casehub/work/runtime/service/CapabilityParserTest.java`

- [ ] **Step 8.1: Write the failing tests**

```java
// runtime/src/test/java/io/casehub/work/runtime/service/CapabilityParserTest.java
package io.casehub.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Test;

import io.casehub.work.api.Capability;
import io.casehub.work.api.MalformedCapabilityException;
import io.casehub.work.api.WorkCapabilities;

class CapabilityParserTest {

    // --- parse() strict mode ---

    @Test
    void parse_returnsEmptySet_forNullInput() {
        assertThat(CapabilityParser.parse(null)).isEmpty();
    }

    @Test
    void parse_returnsEmptySet_forBlankInput() {
        assertThat(CapabilityParser.parse("   ")).isEmpty();
    }

    @Test
    void parse_trimsWhitespaceAroundTokens() {
        assertThat(CapabilityParser.parse("  legal-review , contract-analysis  "))
                .containsExactlyInAnyOrder(
                        WorkCapabilities.LEGAL_REVIEW, WorkCapabilities.CONTRACT_ANALYSIS);
    }

    @Test
    void parse_skipsEmptyTokensFromDoubleCommas() {
        assertThat(CapabilityParser.parse("legal-review,,contract-analysis"))
                .containsExactlyInAnyOrder(
                        WorkCapabilities.LEGAL_REVIEW, WorkCapabilities.CONTRACT_ANALYSIS);
    }

    @Test
    void parse_throws_onMalformedToken() {
        assertThatThrownBy(() -> CapabilityParser.parse("legal-review,legal_review"))
                .isInstanceOf(MalformedCapabilityException.class)
                .satisfies(ex -> assertThat(((MalformedCapabilityException) ex).badValues())
                        .containsExactly("legal_review"));
    }

    @Test
    void parse_returnsSingleElement() {
        assertThat(CapabilityParser.parse("legal-review"))
                .containsExactly(WorkCapabilities.LEGAL_REVIEW);
    }

    // --- parseLenient() lenient mode ---

    @Test
    void parseLenient_returnsEmptySet_forNullInput() {
        assertThat(CapabilityParser.parseLenient(null)).isEmpty();
    }

    @Test
    void parseLenient_skipsMalformedToken_andRetainsValid() {
        assertThat(CapabilityParser.parseLenient("legal-review,legal_review,contract-analysis"))
                .containsExactlyInAnyOrder(
                        WorkCapabilities.LEGAL_REVIEW, WorkCapabilities.CONTRACT_ANALYSIS);
    }

    @Test
    void parseLenient_returnsEmptySet_whenAllTokensMalformed() {
        assertThat(CapabilityParser.parseLenient("Legal_Review,BAD")).isEmpty();
    }
}
```

- [ ] **Step 8.2: Run to confirm failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapabilityParserTest -pl runtime
```
Expected: compilation failure.

- [ ] **Step 8.3: Write `CapabilityParser`**

```java
// runtime/src/main/java/io/casehub/work/runtime/service/CapabilityParser.java
package io.casehub.work.runtime.service;

import java.util.Arrays;
import java.util.Set;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import org.jboss.logging.Logger;

import io.casehub.work.api.Capability;
import io.casehub.work.api.MalformedCapabilityException;

/**
 * Parses comma-separated capability strings to {@code Set<Capability>}.
 *
 * <p>Two modes: {@link #parse} (strict — throws on bad format, for new input)
 * and {@link #parseLenient} (skips bad tokens with a warning, for existing DB rows).
 */
final class CapabilityParser {

    private static final Logger LOG = Logger.getLogger(CapabilityParser.class);

    private CapabilityParser() {}

    /**
     * Strict parse — throws {@link MalformedCapabilityException} on any malformed token.
     * Use for new user input (WorkItem creation).
     */
    static Set<Capability> parse(String raw) {
        if (raw == null || raw.isBlank()) {
            return Set.of();
        }
        return Arrays.stream(raw.split(","))
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .map(Capability::of)
                .collect(Collectors.toUnmodifiableSet());
    }

    /**
     * Lenient parse — skips malformed tokens with a WARN log entry.
     * Use when reading existing DB rows to build {@link io.casehub.work.api.SelectionContext}.
     */
    static Set<Capability> parseLenient(String raw) {
        if (raw == null || raw.isBlank()) {
            return Set.of();
        }
        return Arrays.stream(raw.split(","))
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .flatMap(s -> {
                    try {
                        return Stream.of(Capability.of(s));
                    } catch (MalformedCapabilityException e) {
                        LOG.warnf("Skipping malformed capability string in DB row: '%s'", s);
                        return Stream.empty();
                    }
                })
                .collect(Collectors.toUnmodifiableSet());
    }
}
```

- [ ] **Step 8.4: Run to confirm pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=CapabilityParserTest -pl runtime
```
Expected: BUILD SUCCESS.

- [ ] **Step 8.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  runtime/src/main/java/io/casehub/work/runtime/service/CapabilityParser.java \
  runtime/src/test/java/io/casehub/work/runtime/service/CapabilityParserTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(runtime): add CapabilityParser with strict and lenient modes (Refs #220)"
```

---

## Task 9: Exception mappers

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/api/MalformedCapabilityExceptionMapper.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/api/UnknownCapabilityExceptionMapper.java`

No unit tests — these are tested by the integration test in Task 11.

- [ ] **Step 9.1: Write `MalformedCapabilityExceptionMapper`**

```java
// runtime/src/main/java/io/casehub/work/runtime/api/MalformedCapabilityExceptionMapper.java
package io.casehub.work.runtime.api;

import java.util.Map;

import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;

import io.casehub.work.api.MalformedCapabilityException;

@Provider
public class MalformedCapabilityExceptionMapper implements ExceptionMapper<MalformedCapabilityException> {

    @Override
    public Response toResponse(final MalformedCapabilityException e) {
        return Response.status(400)
                .entity(Map.of(
                        "error", "MALFORMED_CAPABILITY",
                        "values", e.badValues(),
                        "message", "Capability id must be lowercase kebab-case"))
                .build();
    }
}
```

- [ ] **Step 9.2: Write `UnknownCapabilityExceptionMapper`**

```java
// runtime/src/main/java/io/casehub/work/runtime/api/UnknownCapabilityExceptionMapper.java
package io.casehub.work.runtime.api;

import java.util.Map;

import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;

import io.casehub.work.api.UnknownCapabilityException;

@Provider
public class UnknownCapabilityExceptionMapper implements ExceptionMapper<UnknownCapabilityException> {

    @Override
    public Response toResponse(final UnknownCapabilityException e) {
        return Response.status(400)
                .entity(Map.of(
                        "error", "UNKNOWN_CAPABILITY",
                        "values", e.unknownIds(),
                        "message", "Unknown capabilities: " + e.unknownIds()))
                .build();
    }
}
```

- [ ] **Step 9.3: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime
```
Expected: BUILD SUCCESS.

- [ ] **Step 9.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  runtime/src/main/java/io/casehub/work/runtime/api/MalformedCapabilityExceptionMapper.java \
  runtime/src/main/java/io/casehub/work/runtime/api/UnknownCapabilityExceptionMapper.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(runtime): add exception mappers for MalformedCapability and UnknownCapability (Refs #220)"
```

---

## Task 10: Wire `WorkItemService` and `WorkItemAssignmentService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java`

- [ ] **Step 10.1: Update `WorkItemService`**

Add `CapabilityValidator` to the constructor and call `CapabilityParser.parse()` in `create()`.

Locate the constructor (lines ~59–73) and add `CapabilityValidator capabilityValidator` as the last parameter. Add `this.capabilityValidator = capabilityValidator;` in the body. Add `private final CapabilityValidator capabilityValidator;` as a field.

Add the call at the top of `create()` (currently line ~76), immediately before `final WorkItem item = new WorkItem();`:

```java
// Add at top of create():
capabilityValidator.validate(CapabilityParser.parse(request.requiredCapabilities));
```

Add the import:
```java
import io.casehub.work.core.strategy.CapabilityValidator;
```

- [ ] **Step 10.2: Update `WorkItemAssignmentService`**

In `assign()` at line ~131, the `SelectionContext` constructor call passes `workItem.requiredCapabilities` (a raw String). Change this to use `CapabilityParser.parseLenient()`:

```java
// BEFORE (line ~134):
workItem.requiredCapabilities,

// AFTER:
CapabilityParser.parseLenient(workItem.requiredCapabilities),
```

Add the import:
```java
import io.casehub.work.runtime.service.CapabilityParser;
```

Note: `CapabilityParser` is package-private in `io.casehub.work.runtime.service` — `WorkItemAssignmentService` is in the same package, so access is fine.

- [ ] **Step 10.3: Fix any remaining compilation errors in `runtime`**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | grep "error:"
```

Common remaining breaks after these changes:
- `WorkItemAssignmentServiceTest` — if it constructs `SelectionContext` with a String, update to `Set.of(Capability.of("tag"))` or `Set.of()`
- Any test that constructs `WorkerCandidate` with `Set<String>` — update to `Set<Capability>`

- [ ] **Step 10.4: Run `runtime` tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: BUILD SUCCESS. Fix any failures before proceeding.

- [ ] **Step 10.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java \
  runtime/src/main/java/io/casehub/work/runtime/service/WorkItemAssignmentService.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(runtime): wire CapabilityValidator into WorkItemService.create(); parseLenient in assign() (Refs #220)"
```

---

## Task 11: Integration test

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemCapabilityIT.java`

- [ ] **Step 11.1: Write the integration test**

```java
// runtime/src/test/java/io/casehub/work/runtime/api/WorkItemCapabilityIT.java
package io.casehub.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.containsInAnyOrder;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.hasItem;

import java.util.Set;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import org.junit.jupiter.api.Test;

import io.casehub.work.api.Capability;
import io.casehub.work.api.CapabilityRegistry;
import io.casehub.work.api.WorkCapabilities;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(WorkItemCapabilityIT.StrictRegistryProfile.class)
class WorkItemCapabilityIT {

    public static class StrictRegistryProfile implements io.quarkus.test.junit.QuarkusTestProfile {
        @Override
        public java.util.Map<String, String> getConfigOverrides() {
            return java.util.Map.of("casehub.work.capability-validation", "STRICT");
        }
    }

    @Alternative
    @Priority(1)
    @ApplicationScoped
    public static class LegalOnlyRegistry implements CapabilityRegistry {
        @Override
        public Set<Capability> capabilities() {
            return Set.of(WorkCapabilities.LEGAL_REVIEW);
        }
    }

    @Test
    void create_succeeds_whenCapabilityIsInRegistry() {
        given()
            .contentType("application/json")
            .body("""
                {"title":"Review contract","requiredCapabilities":"legal-review","createdBy":"test"}
                """)
        .when()
            .post("/workitems")
        .then()
            .statusCode(201);
    }

    @Test
    void create_returns400_whenCapabilityHasWrongFormat() {
        given()
            .contentType("application/json")
            .body("""
                {"title":"Review contract","requiredCapabilities":"legal_review","createdBy":"test"}
                """)
        .when()
            .post("/workitems")
        .then()
            .statusCode(400)
            .body("error", equalTo("MALFORMED_CAPABILITY"))
            .body("values", hasItem("legal_review"));
    }

    @Test
    void create_returns400_whenCapabilityNotInRegistry() {
        given()
            .contentType("application/json")
            .body("""
                {"title":"Sign document","requiredCapabilities":"audit-sign","createdBy":"test"}
                """)
        .when()
            .post("/workitems")
        .then()
            .statusCode(400)
            .body("error", equalTo("UNKNOWN_CAPABILITY"))
            .body("values", hasItem("audit-sign"));
    }

    @Test
    void create_returns400_withAllUnknownCapabilities_whenMultipleUnknown() {
        given()
            .contentType("application/json")
            .body("""
                {"title":"Multi","requiredCapabilities":"audit-sign,risk-assess","createdBy":"test"}
                """)
        .when()
            .post("/workitems")
        .then()
            .statusCode(400)
            .body("error", equalTo("UNKNOWN_CAPABILITY"))
            .body("values", containsInAnyOrder("audit-sign", "risk-assess"));
    }

    @Test
    void create_succeeds_withNoRequiredCapabilities() {
        given()
            .contentType("application/json")
            .body("""
                {"title":"Open task","createdBy":"test"}
                """)
        .when()
            .post("/workitems")
        .then()
            .statusCode(201);
    }
}
```

- [ ] **Step 11.2: Run the integration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemCapabilityIT -pl runtime
```
Expected: BUILD SUCCESS, all 5 tests pass.

If tests fail due to `TestProfile`/`@Alternative` CDI wiring issues, check the existing pattern in `WorkItemResourceTest` or similar `@QuarkusTest` files in the module for how `@Alternative @Priority(1)` beans are wired in tests.

- [ ] **Step 11.3: Run full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```
Expected: BUILD SUCCESS, no regressions.

- [ ] **Step 11.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  runtime/src/test/java/io/casehub/work/runtime/api/WorkItemCapabilityIT.java
git -C /Users/mdproctor/claude/casehub/work commit -m "test(runtime): integration test for capability validation — STRICT, MALFORMED, UNKNOWN (Refs #220)"
```

---

## Task 12: Fix ripple effects in dependent modules

**Files:** (varies — search for uses of changed types)

The `ai/`, `work-flow/`, and `casehub-work-queues/` modules may reference `WorkerCandidate.capabilities()` as `Set<String>` or `SelectionContext.requiredCapabilities()` as `String`. Find and fix them.

- [ ] **Step 12.1: Find all compilation errors across the full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl ai,work-flow,casehub-work-queues 2>&1 | grep "error:" | head -40
```

- [ ] **Step 12.2: Fix each error**

Common patterns:
- `candidate.capabilities()` used as `Set<String>` → update to use `Capability::id` when String is needed: `candidate.capabilities().stream().map(Capability::id).collect(...)`
- `context.requiredCapabilities()` used as `String` → is now `Set<Capability>`; adapt caller
- `WorkerCandidate` constructor with `Set.of("tag")` → change to `Set.of(Capability.of("tag"))`

- [ ] **Step 12.3: Run tests in affected modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ai,work-flow,casehub-work-queues
```
Expected: BUILD SUCCESS.

- [ ] **Step 12.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add -u
git -C /Users/mdproctor/claude/casehub/work commit -m "refactor: update capability references in ai, work-flow, queues modules to Set<Capability> (Refs #220)"
```

---

## Task 13: Full build verification

- [ ] **Step 13.1: Build all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests
```
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 13.2: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,core,runtime
```
Expected: BUILD SUCCESS, no regressions.

- [ ] **Step 13.3: Final commit if any stray fixes needed**

```bash
git -C /Users/mdproctor/claude/casehub/work status
```
If clean: done. If uncommitted changes: commit with `fix(#220): ...`.
