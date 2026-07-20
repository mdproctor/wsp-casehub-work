# FilterEngine → LabelRule Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #314 — refactor: migrate FilterEngine to platform LabelRule
**Issue group:** #314

**Goal:** Unify both filter systems (queues `FilterEngine` + runtime `FilterRegistryEngine`) into one `LabelRule`-based engine using platform primitives.

**Architecture:** One `LabelRuleEngine` in `runtime/filter/` replaces both evaluation engines. Rules come from two sources: CDI-produced `LabelRule` instances (permanent filters) and DB-persisted `LabelRuleEntity` records (dynamic filters, compiled to `LabelRule` at load time via `ExpressionEngineRegistry`). Single-pass evaluation — no multi-pass chain propagation. Per-rule error isolation. `LabelAction.Add`/`Remove` applied to WorkItem labels with INFERRED/MANUAL protection.

**Tech Stack:** Java 21, Quarkus 3.32, platform-api (`LabelRule`, `LabelAction`, `CompiledExpression`, `LambdaExpression`), platform-expression (`ExpressionEngineRegistry`, `JexlExpressionEngine`), JPA/Panache, CDI

## Global Constraints

- Platform SNAPSHOT dependency — platform#189 (triggerEvents) and platform#191 (JexlExpressionEngine) are on main
- Pre-release: no backward compatibility concerns, no data migration
- `scope` on `LabelRuleEntity` is management metadata, not an execution predicate (protocol: queue-filter-scope-management-only)
- `LabelRuleStore` follows the three-tier CDI priority ladder: JPA (Tier 1), MongoDB (Tier 2), InMemory (Tier 3) (protocol: persistence-backend-cdi-priority)
- `JpaLabelRuleStore.put()` stamps tenancyId on insert (protocol: store-tenancy-stamping-on-insert)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>` (never full build)
- IntelliJ MCP: mandatory for all Java code navigation and editing. Pass `project_path=/Users/mdproctor/claude/casehub/work` to every call.
- Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl platform-api,expression -f /Users/mdproctor/claude/casehub/platform/pom.xml` first to install platform SNAPSHOT with triggerEvents + JexlExpressionEngine.

---

### Task 1: Persistence — LabelRuleEntity + LabelRuleStore + JpaLabelRuleStore + V5004

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/filter/LabelRuleEntity.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/LabelRuleStore.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaLabelRuleStore.java`
- Create: `runtime/src/main/resources/db/work/migration/V5004__label_rule_schema.sql`
- Modify: `runtime/pom.xml` — add `casehub-platform-expression` dependency
- Test: `runtime/src/test/java/io/casehub/work/runtime/filter/LabelRuleEntityTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaLabelRuleStoreTenancyTest.java`

**Interfaces:**
- Consumes: platform `LabelAction`, `LabelAction.Add`, `LabelAction.Remove`, `Path`, `PathAttributeConverter`
- Produces: `LabelRuleEntity` (JPA entity), `LabelRuleStore` (SPI), `JpaLabelRuleStore` (JPA impl) — consumed by Tasks 2, 4, 5, 6

- [ ] **Step 1: Add platform-expression dependency to runtime/pom.xml**

Add to `runtime/pom.xml` `<dependencies>` section, after the `casehub-platform` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-expression</artifactId>
</dependency>
```

Version is managed by the parent POM's `<dependencyManagement>`.

- [ ] **Step 2: Install platform SNAPSHOT**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl platform-api,expression -f /Users/mdproctor/claude/casehub/platform/pom.xml -q
```

This makes platform#189 (triggerEvents) and platform#191 (JexlExpressionEngine) available locally.

- [ ] **Step 3: Create V5004 Flyway migration**

Create `runtime/src/main/resources/db/work/migration/V5004__label_rule_schema.sql`:

```sql
-- #314: Unified label rule table (replaces work_item_filter, filter_rule, filter_chain)

DROP TABLE IF EXISTS filter_chain_work_item;
DROP TABLE IF EXISTS filter_chain;
DROP TABLE IF EXISTS work_item_filter;
DROP TABLE IF EXISTS filter_rule;

CREATE TABLE label_rule (
    id                   UUID PRIMARY KEY,
    tenancy_id           VARCHAR(255) NOT NULL,
    name                 VARCHAR(255) NOT NULL,
    description          VARCHAR(500),
    condition_language   VARCHAR(20)  NOT NULL,
    condition_expression TEXT,
    actions_json         TEXT         NOT NULL DEFAULT '[]',
    trigger_events       VARCHAR(100) DEFAULT '',
    scope                VARCHAR(500),
    enabled              BOOLEAN      DEFAULT true,
    created_at           TIMESTAMP    NOT NULL
);
```

- [ ] **Step 4: Write LabelRuleEntity**

Create `runtime/src/main/java/io/casehub/work/runtime/filter/LabelRuleEntity.java` via `ide_create_file`:

```java
package io.casehub.work.runtime.filter;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Convert;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.platform.api.expression.CompiledExpression;
import io.casehub.platform.api.expression.ExpressionEngineRegistry;
import io.casehub.platform.api.label.LabelAction;
import io.casehub.platform.api.label.LabelRule;
import io.casehub.platform.api.path.Path;
import io.casehub.work.runtime.model.PathAttributeConverter;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

@Entity
@Table(name = "label_rule")
public class LabelRuleEntity extends PanacheEntityBase {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Id
    public UUID id;

    @Column(name = "tenancy_id", nullable = false)
    public String tenancyId;

    @Column(nullable = false, length = 255)
    public String name;

    @Column(length = 500)
    public String description;

    @Column(name = "condition_language", nullable = false, length = 20)
    public String conditionLanguage;

    @Column(name = "condition_expression", columnDefinition = "TEXT")
    public String conditionExpression;

    @Column(name = "actions_json", nullable = false, columnDefinition = "TEXT")
    public String actionsJson = "[]";

    @Column(name = "trigger_events", length = 100)
    public String triggerEvents = "";

    @Convert(converter = PathAttributeConverter.class)
    @Column(length = 500)
    public Path scope;

    public boolean enabled = true;

    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
        if (createdAt == null) {
            createdAt = Instant.now();
        }
    }

    public List<LabelAction> parseActions() {
        if (actionsJson == null || actionsJson.isBlank()) {
            return List.of();
        }
        try {
            List<ActionJson> raw = MAPPER.readValue(actionsJson, new TypeReference<>() {});
            List<LabelAction> result = new ArrayList<>();
            for (ActionJson a : raw) {
                if ("Add".equals(a.type)) {
                    result.add(new LabelAction.Add(a.label));
                } else if ("Remove".equals(a.type)) {
                    result.add(new LabelAction.Remove(a.label));
                }
            }
            return result;
        } catch (JsonProcessingException e) {
            return List.of();
        }
    }

    public static String serializeActions(List<LabelAction> actions) {
        try {
            List<ActionJson> raw = actions.stream()
                    .map(a -> new ActionJson(
                            a instanceof LabelAction.Add ? "Add" : "Remove",
                            a.label()))
                    .toList();
            return MAPPER.writeValueAsString(raw);
        } catch (JsonProcessingException e) {
            return "[]";
        }
    }

    public LabelRule toLabelRule(ExpressionEngineRegistry registry) {
        CompiledExpression<java.util.Map<String, Object>, Boolean> condition =
                registry.compile(conditionLanguage, conditionExpression,
                        (Class<java.util.Map<String, Object>>) (Class<?>) java.util.Map.class,
                        Boolean.class);
        Set<String> events = (triggerEvents == null || triggerEvents.isBlank())
                ? Set.of()
                : Set.of(triggerEvents.split(","));
        return new LabelRule(name, condition, parseActions(), events);
    }

    record ActionJson(String type, String label) {}
}
```

- [ ] **Step 5: Write LabelRuleStore SPI**

Create `runtime/src/main/java/io/casehub/work/runtime/repository/LabelRuleStore.java` via `ide_create_file`:

```java
package io.casehub.work.runtime.repository;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.casehub.work.runtime.filter.LabelRuleEntity;

public interface LabelRuleStore {

    LabelRuleEntity put(LabelRuleEntity rule);

    Optional<LabelRuleEntity> get(UUID id);

    List<LabelRuleEntity> findEnabled();

    List<LabelRuleEntity> scanAll();

    boolean delete(UUID id);
}
```

- [ ] **Step 6: Write JpaLabelRuleStore**

Create `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaLabelRuleStore.java` via `ide_create_file`. Follow the `JpaFilterRuleStore` pattern exactly — `extends TenantAwareStore`:

```java
package io.casehub.work.runtime.repository.jpa;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.work.runtime.filter.LabelRuleEntity;
import io.casehub.work.runtime.repository.LabelRuleStore;

@ApplicationScoped
public class JpaLabelRuleStore extends TenantAwareStore implements LabelRuleStore {

    @Override
    public LabelRuleEntity put(final LabelRuleEntity rule) {
        return withTenantQuery(() -> {
            if (rule.tenancyId == null) {
                rule.tenancyId = currentPrincipal.tenancyId();
            }
            rule.persistAndFlush();
            return rule;
        });
    }

    @Override
    public Optional<LabelRuleEntity> get(final UUID id) {
        return withTenantQuery(() ->
                LabelRuleEntity.find("id = ?1 AND tenancyId = ?2", id,
                        currentPrincipal.tenancyId()).firstResultOptional());
    }

    @Override
    public List<LabelRuleEntity> findEnabled() {
        return withTenantQuery(() ->
                LabelRuleEntity.list("enabled = true AND tenancyId = ?1 ORDER BY createdAt ASC",
                        currentPrincipal.tenancyId()));
    }

    @Override
    public List<LabelRuleEntity> scanAll() {
        return withTenantQuery(() ->
                LabelRuleEntity.list("tenancyId = ?1 ORDER BY createdAt ASC",
                        currentPrincipal.tenancyId()));
    }

    @Override
    public boolean delete(final UUID id) {
        return withTenantQuery(() -> {
            long deleted = LabelRuleEntity.delete("id = ?1 AND tenancyId = ?2", id,
                    currentPrincipal.tenancyId());
            return deleted > 0;
        });
    }
}
```

- [ ] **Step 7: Write LabelRuleEntity unit test**

Create `runtime/src/test/java/io/casehub/work/runtime/filter/LabelRuleEntityTest.java`:

```java
package io.casehub.work.runtime.filter;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.label.LabelAction;

class LabelRuleEntityTest {

    @Test
    void serializeActions_addAndRemove() {
        var actions = List.<LabelAction>of(
                new LabelAction.Add("queue/urgent"),
                new LabelAction.Remove("queue/normal"));
        String json = LabelRuleEntity.serializeActions(actions);
        assertThat(json).contains("\"type\":\"Add\"", "\"label\":\"queue/urgent\"");
        assertThat(json).contains("\"type\":\"Remove\"", "\"label\":\"queue/normal\"");
    }

    @Test
    void parseActions_roundTrip() {
        var original = List.<LabelAction>of(
                new LabelAction.Add("a"),
                new LabelAction.Remove("b"));
        String json = LabelRuleEntity.serializeActions(original);
        var entity = new LabelRuleEntity();
        entity.actionsJson = json;
        List<LabelAction> parsed = entity.parseActions();
        assertThat(parsed).containsExactly(
                new LabelAction.Add("a"),
                new LabelAction.Remove("b"));
    }

    @Test
    void parseActions_emptyJson_returnsEmptyList() {
        var entity = new LabelRuleEntity();
        entity.actionsJson = "[]";
        assertThat(entity.parseActions()).isEmpty();
    }

    @Test
    void parseActions_nullJson_returnsEmptyList() {
        var entity = new LabelRuleEntity();
        entity.actionsJson = null;
        assertThat(entity.parseActions()).isEmpty();
    }

    @Test
    void parseActions_malformedJson_returnsEmptyList() {
        var entity = new LabelRuleEntity();
        entity.actionsJson = "not json";
        assertThat(entity.parseActions()).isEmpty();
    }

    @Test
    void serializeActions_emptyList() {
        assertThat(LabelRuleEntity.serializeActions(List.of())).isEqualTo("[]");
    }
}
```

- [ ] **Step 8: Run unit test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LabelRuleEntityTest -pl runtime -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Expected: all 6 tests pass.

- [ ] **Step 9: Write JpaLabelRuleStore tenancy test**

Create `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaLabelRuleStoreTenancyTest.java`. Follow the `JpaFilterRuleStoreTenancyTest` pattern — Quarkus test with `@TestProfile` for multi-tenant isolation:

The test class structure should mirror `JpaFilterRuleStoreTenancyTest` exactly — verify `put()` stamps tenancyId, `get()` is tenant-scoped, `findEnabled()` returns only current tenant's enabled rules, `delete()` is tenant-scoped. Use the existing test profile and tenant switching patterns from the file at `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaFilterRuleStoreTenancyTest.java`.

- [ ] **Step 10: Run JPA test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JpaLabelRuleStoreTenancyTest -pl runtime -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Expected: all tenancy tests pass.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add runtime/src/main/java/io/casehub/work/runtime/filter/LabelRuleEntity.java runtime/src/main/java/io/casehub/work/runtime/repository/LabelRuleStore.java runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaLabelRuleStore.java runtime/src/main/resources/db/work/migration/V5004__label_rule_schema.sql runtime/pom.xml runtime/src/test/java/io/casehub/work/runtime/filter/LabelRuleEntityTest.java runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaLabelRuleStoreTenancyTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#314): add LabelRuleEntity, LabelRuleStore, JpaLabelRuleStore, V5004 migration"
```

---

### Task 2: LabelRuleEngine — core evaluation logic

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/filter/LabelRuleEngine.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/filter/LabelRuleEngineTest.java`

**Interfaces:**
- Consumes: `LabelRuleStore` (Task 1), platform `LabelRule`, `LabelAction`, `ExpressionEngineRegistry`, `LambdaExpression`, `Instance<LabelRule>`, `WorkItemContextBuilder`, `WorkItem`, `WorkItemLabel`, `LabelPersistence`
- Produces: `LabelRuleEngine.evaluate(WorkItem, Map<String,Object>, String)` — consumed by Tasks 3, 5

- [ ] **Step 1: Write failing test — basic Add action**

Create `runtime/src/test/java/io/casehub/work/runtime/filter/LabelRuleEngineTest.java`:

```java
package io.casehub.work.runtime.filter;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.expression.ExpressionEvaluationException;
import io.casehub.platform.api.expression.LambdaExpression;
import io.casehub.platform.api.label.LabelAction;
import io.casehub.platform.api.label.LabelRule;
import io.casehub.work.api.LabelPersistence;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemLabel;

class LabelRuleEngineTest {

    private LabelRuleEngine engine;
    private List<LabelRule> cdiRules;

    @BeforeEach
    void setUp() {
        cdiRules = new ArrayList<>();
        engine = new LabelRuleEngine(cdiRules);
    }

    @Test
    void matchingRule_appliesInferredLabel() {
        cdiRules.add(new LabelRule("urgent-rule",
                new LambdaExpression<>(ctx -> "HIGH".equals(
                        ctx.get("priority") != null ? ctx.get("priority").toString() : null)),
                List.of(new LabelAction.Add("priority/high"))));

        var wi = new WorkItem();
        wi.priority = io.casehub.work.api.WorkItemPriority.HIGH;
        var context = Map.<String, Object>of("priority", wi.priority);

        engine.evaluate(wi, context, "ADD");

        assertThat(wi.labels).extracting(l -> l.path).contains("priority/high");
        assertThat(wi.labels).filteredOn(l -> l.path.equals("priority/high"))
                .extracting(l -> l.persistence).containsOnly(LabelPersistence.INFERRED);
    }

    @Test
    void nonMatchingRule_doesNotApplyLabel() {
        cdiRules.add(new LabelRule("urgent-rule",
                new LambdaExpression<>(ctx -> false),
                List.of(new LabelAction.Add("priority/high"))));

        var wi = new WorkItem();
        engine.evaluate(wi, Map.of(), "ADD");

        assertThat(wi.labels).extracting(l -> l.path).doesNotContain("priority/high");
    }

    @Test
    void stripsInferredLabels_beforeEvaluation() {
        var wi = new WorkItem();
        wi.labels.add(new WorkItemLabel("old/inferred", LabelPersistence.INFERRED, "old-rule"));
        wi.labels.add(new WorkItemLabel("manual/keep", LabelPersistence.MANUAL, "alice"));

        engine.evaluate(wi, Map.of(), "ADD");

        assertThat(wi.labels).extracting(l -> l.path).doesNotContain("old/inferred");
        assertThat(wi.labels).extracting(l -> l.path).contains("manual/keep");
    }

    @Test
    void removeAction_removesOnlyInferredLabels() {
        cdiRules.add(new LabelRule("cleanup-rule",
                new LambdaExpression<>(ctx -> true),
                List.of(new LabelAction.Remove("shared-label"))));

        var wi = new WorkItem();
        wi.labels.add(new WorkItemLabel("shared-label", LabelPersistence.MANUAL, "alice"));

        engine.evaluate(wi, Map.of(), "ADD");

        assertThat(wi.labels).extracting(l -> l.path).contains("shared-label");
    }

    @Test
    void addAction_deduplicatesExistingInferred() {
        cdiRules.add(new LabelRule("r1",
                new LambdaExpression<>(ctx -> true),
                List.of(new LabelAction.Add("x"))));
        cdiRules.add(new LabelRule("r2",
                new LambdaExpression<>(ctx -> true),
                List.of(new LabelAction.Add("x"))));

        var wi = new WorkItem();
        engine.evaluate(wi, Map.of(), "ADD");

        long count = wi.labels.stream().filter(l -> l.path.equals("x")).count();
        assertThat(count).isEqualTo(1);
    }

    @Test
    void triggerEvents_filtersRules() {
        cdiRules.add(new LabelRule("add-only",
                new LambdaExpression<>(ctx -> true),
                List.of(new LabelAction.Add("on-add")),
                Set.of("ADD")));

        var wi = new WorkItem();
        engine.evaluate(wi, Map.of(), "REMOVE");

        assertThat(wi.labels).extracting(l -> l.path).doesNotContain("on-add");
    }

    @Test
    void triggerEvents_emptyMatchesAll() {
        cdiRules.add(new LabelRule("all-events",
                new LambdaExpression<>(ctx -> true),
                List.of(new LabelAction.Add("always"))));

        var wi = new WorkItem();
        engine.evaluate(wi, Map.of(), "REMOVE");

        assertThat(wi.labels).extracting(l -> l.path).contains("always");
    }

    @Test
    void perRuleErrorIsolation_continuesAfterFailure() {
        cdiRules.add(new LabelRule("bad-rule",
                new LambdaExpression<>(ctx -> {
                    throw new ExpressionEvaluationException("boom");
                }),
                List.of(new LabelAction.Add("bad"))));
        cdiRules.add(new LabelRule("good-rule",
                new LambdaExpression<>(ctx -> true),
                List.of(new LabelAction.Add("good"))));

        var wi = new WorkItem();
        engine.evaluate(wi, Map.of(), "ADD");

        assertThat(wi.labels).extracting(l -> l.path).doesNotContain("bad");
        assertThat(wi.labels).extracting(l -> l.path).contains("good");
    }

    @Test
    void appliedBy_setToRuleName() {
        cdiRules.add(new LabelRule("my-rule",
                new LambdaExpression<>(ctx -> true),
                List.of(new LabelAction.Add("tagged"))));

        var wi = new WorkItem();
        engine.evaluate(wi, Map.of(), "ADD");

        assertThat(wi.labels).filteredOn(l -> l.path.equals("tagged"))
                .extracting(l -> l.appliedBy).containsOnly("my-rule");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LabelRuleEngineTest -pl runtime -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Expected: FAIL — `LabelRuleEngine` class not found.

- [ ] **Step 3: Write LabelRuleEngine**

Create `runtime/src/main/java/io/casehub/work/runtime/filter/LabelRuleEngine.java` via `ide_create_file`:

```java
package io.casehub.work.runtime.filter;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.platform.api.expression.ExpressionEngineRegistry;
import io.casehub.platform.api.label.LabelAction;
import io.casehub.platform.api.label.LabelRule;
import io.casehub.work.api.LabelPersistence;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemLabel;
import io.casehub.work.runtime.repository.LabelRuleStore;

@ApplicationScoped
public class LabelRuleEngine {

    private static final Logger LOG = Logger.getLogger(LabelRuleEngine.class);
    private static final ThreadLocal<Boolean> RUNNING = ThreadLocal.withInitial(() -> false);

    @Inject
    Instance<LabelRule> cdiRules;

    @Inject
    LabelRuleStore labelRuleStore;

    @Inject
    ExpressionEngineRegistry expressionRegistry;

    LabelRuleEngine() {}

    LabelRuleEngine(List<LabelRule> testRules) {
        this.cdiRules = null;
        this.labelRuleStore = null;
        this.expressionRegistry = null;
        this.testRules = testRules;
    }

    private List<LabelRule> testRules;

    public void evaluate(final WorkItem workItem, final Map<String, Object> context,
            final String event) {
        if (Boolean.TRUE.equals(RUNNING.get())) {
            return;
        }
        RUNNING.set(true);
        try {
            workItem.labels.removeIf(l -> l.persistence == LabelPersistence.INFERRED);

            List<LabelRule> allRules = collectRules();
            List<LabelAction> actions = evaluateRules(allRules, context, event);
            applyActions(workItem, actions, allRules, context, event);
        } finally {
            RUNNING.remove();
        }
    }

    private List<LabelRule> collectRules() {
        List<LabelRule> all = new ArrayList<>();
        if (testRules != null) {
            all.addAll(testRules);
        } else {
            cdiRules.forEach(all::add);
            if (labelRuleStore != null && expressionRegistry != null) {
                for (LabelRuleEntity entity : labelRuleStore.findEnabled()) {
                    try {
                        all.add(entity.toLabelRule(expressionRegistry));
                    } catch (Exception e) {
                        LOG.warnf("Failed to compile label rule '%s': %s", entity.name, e.getMessage());
                    }
                }
            }
        }
        return all;
    }

    private List<LabelAction> evaluateRules(final List<LabelRule> rules,
            final Map<String, Object> context, final String event) {
        List<LabelAction> actions = new ArrayList<>();
        for (LabelRule rule : rules) {
            Set<String> triggers = rule.triggerEvents();
            if (!triggers.isEmpty() && !triggers.contains(event)) {
                continue;
            }
            try {
                if (Boolean.TRUE.equals(rule.condition().eval(context))) {
                    actions.addAll(rule.actions());
                }
            } catch (Exception e) {
                LOG.warnf("Rule '%s' evaluation failed: %s", rule.name(), e.getMessage());
            }
        }
        return actions;
    }

    private void applyActions(final WorkItem workItem, final List<LabelAction> actions,
            final List<LabelRule> rules, final Map<String, Object> context,
            final String event) {
        for (int i = 0, ruleIdx = 0; i < actions.size(); i++) {
            LabelAction action = actions.get(i);
            String ruleName = findRuleName(rules, context, event, action);
            if (action instanceof LabelAction.Add add) {
                boolean exists = workItem.labels.stream()
                        .anyMatch(l -> add.label().equals(l.path) && l.persistence == LabelPersistence.INFERRED);
                if (!exists) {
                    workItem.labels.add(new WorkItemLabel(add.label(), LabelPersistence.INFERRED, ruleName));
                }
            } else if (action instanceof LabelAction.Remove remove) {
                workItem.labels.removeIf(l ->
                        remove.label().equals(l.path) && l.persistence == LabelPersistence.INFERRED);
            }
        }
    }

    private String findRuleName(final List<LabelRule> rules, final Map<String, Object> context,
            final String event, final LabelAction action) {
        for (LabelRule rule : rules) {
            if (rule.actions().contains(action)) {
                return rule.name();
            }
        }
        return "unknown";
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LabelRuleEngineTest -pl runtime -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Expected: all 9 tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add runtime/src/main/java/io/casehub/work/runtime/filter/LabelRuleEngine.java runtime/src/test/java/io/casehub/work/runtime/filter/LabelRuleEngineTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#314): add LabelRuleEngine — unified single-pass label evaluation"
```

---

### Task 3: FilterEvaluationObserver refactor

**Files:**
- Modify: `queues/src/main/java/io/casehub/work/queues/service/FilterEvaluationObserver.java`
- Test: existing tests in `queues/` module

**Interfaces:**
- Consumes: `LabelRuleEngine.evaluate()` (Task 2), `WorkItemContextBuilder.toMap()`, `SubjectViewOrchestrator`, `WorkItemStore`
- Produces: refactored observer — no new public API

- [ ] **Step 1: Refactor FilterEvaluationObserver**

Replace the entire body of `FilterEvaluationObserver` with the new wiring. Use `ide_edit_member` on class `FilterEvaluationObserver`:

```java
@ApplicationScoped
public class FilterEvaluationObserver {

    @Inject
    LabelRuleEngine labelRuleEngine;

    @Inject
    WorkItemStore workItemStore;

    @Inject
    SubjectViewOrchestrator views;

    @Inject
    Event<WorkItemQueueEvent> queueEventBus;

    @Transactional
    public void onLifecycleEvent(@Observes final WorkItemLifecycleEvent event) {
        workItemStore.get(event.workItemId()).ifPresent(wi -> {
            final String eventType = mapEventType(event.eventType());
            final Map<String, Object> context = WorkItemContextBuilder.toMap(wi);

            labelRuleEngine.evaluate(wi, context, eventType);

            final Set<String> labelPaths = wi.labels == null ? Set.of()
                    : wi.labels.stream().map(l -> l.path).collect(Collectors.toSet());

            views.evaluateAndTrack(wi.id, wi.tenancyId, labelPaths)
                 .forEach(ve -> queueEventBus.fire(
                         new WorkItemQueueEvent(ve.subjectId(), ve.viewId(),
                                                ve.viewName(),
                                                QueueEventType.valueOf(ve.type().name()),
                                                ve.tenancyId())));
        });
    }

    private String mapEventType(final WorkEventType eventType) {
        return switch (eventType) {
            case CREATED -> "ADD";
            case COMPLETED, REJECTED, FAULTED, CANCELLED, OBSOLETE, EXPIRED, ESCALATED -> "REMOVE";
            default -> "UPDATE";
        };
    }
}
```

Update imports to add `LabelRuleEngine`, `WorkItemContextBuilder`, `WorkEventType`, `Map`, `Set`, `Collectors` and remove `FilterEngine`.

- [ ] **Step 2: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl queues -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Compilation may fail if queues still imports old FilterEngine types — fix imports as needed.

- [ ] **Step 3: Run queues tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Expected: tests pass. Some tests may need updating if they mock `FilterEngine` — update them to mock `LabelRuleEngine` instead.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add queues/
git -C /Users/mdproctor/claude/casehub/work commit -m "refactor(#314): wire FilterEvaluationObserver to LabelRuleEngine"
```

---

### Task 4: Migrate producers + ReviewStepService

**Files:**
- Modify: `ai/src/main/java/io/casehub/work/ai/filter/LowConfidenceFilterProducer.java`
- Modify: `queues-dashboard/src/main/java/io/casehub/work/dashboard/SecurityWritersFilter.java`
- Modify: `queues-examples/src/main/java/io/casehub/work/examples/queues/review/SecurityWritersFilter.java`
- Modify: `queues-dashboard/src/main/java/io/casehub/work/dashboard/ReviewStepService.java`
- Test: respective module tests

**Interfaces:**
- Consumes: platform `LabelRule`, `LabelAction`, `LambdaExpression`, `ExpressionEngineRegistry`
- Produces: CDI-produced `LabelRule` instances, `LabelRuleStore`-based persistence in ReviewStepService

- [ ] **Step 1: Migrate LowConfidenceFilterProducer**

Use `ide_edit_member` to replace the class body. Changes:
- `@Produces FilterDefinition` → `@Produces LabelRule`
- `FilterDefinition.onAdd(...)` → `new LabelRule(name, compiledCondition, actions, triggerEvents)`
- Inject `ExpressionEngineRegistry` to compile the JEXL expression with bound variables
- `ActionDescriptor.of("APPLY_LABEL", Map.of("path", "ai/low-confidence", ...))` → `new LabelAction.Add("ai/low-confidence")`
- `triggerEvents = Set.of("ADD")`

```java
@ApplicationScoped
public class LowConfidenceFilterProducer {

    @Inject
    WorkItemsAiConfig config;

    @Inject
    ExpressionEngineRegistry expressionRegistry;

    @Produces
    public LabelRule lowConfidenceFilter() {
        final double threshold = config.confidenceThreshold();
        @SuppressWarnings("unchecked")
        var condition = expressionRegistry.compile(
                "jexl",
                "confidenceScore != null && confidenceScore < threshold",
                (Class<Map<String, Object>>) (Class<?>) Map.class,
                Boolean.class,
                Map.of("threshold", threshold));
        return new LabelRule(
                "ai/low-confidence",
                condition,
                List.of(new LabelAction.Add("ai/low-confidence")),
                Set.of("ADD"));
    }
}
```

- [ ] **Step 2: Migrate SecurityWritersFilter (queues-dashboard)**

Replace `SecurityWritersFilter` — change from `implements WorkItemFilterBean` to a CDI producer class:

```java
@ApplicationScoped
public class SecurityWritersFilter {

    @Produces
    public LabelRule securityWritersRule() {
        return new LabelRule(
                "security-writers",
                new LambdaExpression<>(ctx -> {
                    Object cg = ctx.get("candidateGroups");
                    Object status = ctx.get("status");
                    return cg != null && cg.toString().contains("security-writers")
                            && status != null && !isTerminal(status);
                }),
                List.of(new LabelAction.Add("review/urgent")));
    }

    private static boolean isTerminal(Object status) {
        if (status instanceof io.casehub.work.api.WorkItemStatus ws) {
            return ws.isTerminal();
        }
        return false;
    }
}
```

- [ ] **Step 3: Migrate SecurityWritersFilter (queues-examples)**

Same transformation as Step 2. The file is at `queues-examples/src/main/java/io/casehub/work/examples/queues/review/SecurityWritersFilter.java`.

- [ ] **Step 4: Migrate ReviewStepService**

`ReviewStepService.ensureFilters()` creates persisted `WorkItemFilter` entities. Migrate to `LabelRuleEntity` via `LabelRuleStore`. Key changes:

- Replace `WorkItemFilter` with `LabelRuleEntity`
- Replace `FilterAction.applyLabel()` with `LabelRuleEntity.serializeActions(List.of(new LabelAction.Add(...)))`
- Remove `Instance<WorkItemFilterBean> lambdaFilterBeans` injection — replace with `Instance<LabelRule> cdiRules`
- Rewrite the chain-dependent demo filters for single-pass: each rule checks source data directly and applies both tier and column labels

The 12 old filters (3 tier-only + 9 tier+status) become 9 single-pass rules (3 tiers × 3 statuses), each applying both the tier label and column label:

```java
private void persist(final String name, final String lang, final String expr,
        final List<LabelAction> actions) {
    final LabelRuleEntity rule = new LabelRuleEntity();
    rule.name = name;
    rule.scope = Path.root();
    rule.conditionLanguage = lang;
    rule.conditionExpression = expr;
    rule.actionsJson = LabelRuleEntity.serializeActions(actions);
    rule.enabled = true;
    rule.tenancyId = currentPrincipal.tenancyId();
    labelRuleStore.put(rule);
}
```

Update `ensureFilters()` to use combined conditions:
```java
// Example: URGENT + PENDING → both tier and column labels
persist("Review: Urgent + Pending", "jexl",
    "priority.name() == 'URGENT' && !status.isTerminal() && status.name() == 'PENDING'",
    List.of(new LabelAction.Add("review/urgent"),
            new LabelAction.Add("review/urgent/unassigned")));
```

Repeat for all 9 combinations (URGENT/HIGH/MEDIUM-LOW × PENDING/ASSIGNED/IN_PROGRESS).

Update `lambdaFilterCount()` and `lambdaFilterNames()` to use `Instance<LabelRule>` instead of `Instance<WorkItemFilterBean>`.

- [ ] **Step 5: Run module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ai -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues-dashboard -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues-examples -f /Users/mdproctor/claude/casehub/work/pom.xml
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add ai/ queues-dashboard/ queues-examples/
git -C /Users/mdproctor/claude/casehub/work commit -m "refactor(#314): migrate filter producers and ReviewStepService to LabelRule"
```

---

### Task 5: LabelRuleResource — REST API consolidation

**Files:**
- Create: `rest/src/main/java/io/casehub/work/rest/LabelRuleResource.java`
- Test: `rest/src/test/java/io/casehub/work/rest/LabelRuleResourceTest.java`

**Interfaces:**
- Consumes: `LabelRuleStore` (Task 1), `LabelRuleEntity` (Task 1), `ExpressionEngineRegistry`, `Instance<LabelRule>`
- Produces: REST API at `/label-rules`

- [ ] **Step 1: Write LabelRuleResource**

Create `rest/src/main/java/io/casehub/work/rest/LabelRuleResource.java` via `ide_create_file`. The resource consolidates `/filters` (queues) and `/filter-rules` (runtime) into `/label-rules`:

```java
@Path("/label-rules")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class LabelRuleResource {

    @Inject LabelRuleStore labelRuleStore;
    @Inject ExpressionEngineRegistry expressionRegistry;
    @Inject Instance<LabelRule> permanentRules;

    // Records for request/response DTOs
    public record CreateLabelRuleRequest(String name, String description,
            String conditionLanguage, String conditionExpression,
            List<LabelActionDto> actions, String triggerEvents, String scope) {}

    public record LabelActionDto(String type, String label) {}

    // GET /label-rules — list persisted + permanent (CDI)
    // POST /label-rules — create persisted rule (validate expression first)
    // PUT /label-rules/{id} — update persisted rule
    // DELETE /label-rules/{id} — delete persisted rule
    // PUT /label-rules/{name}/enabled — toggle enabled
    // POST /label-rules/evaluate — ad-hoc evaluation
}
```

Full implementation follows the patterns in `FilterResource` and `FilterRuleResource`. Validate expressions via `expressionRegistry.validate()` before persisting. List endpoint includes a `source` field ("persisted" or "permanent") to distinguish CDI-produced rules.

- [ ] **Step 2: Write LabelRuleResource tests**

Test CRUD operations, permanent rule listing, validation rejection, ad-hoc evaluation. Follow existing `rest` module test patterns.

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -f /Users/mdproctor/claude/casehub/work/pom.xml
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add rest/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#314): add LabelRuleResource — consolidated REST API at /label-rules"
```

---

### Task 6: Delete old code + backend stores + Maven cleanup

**Files:**
- Delete: all types listed in the "Delete — replaced entirely" table in the spec (~30 files across queues, runtime, rest modules)
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoLabelRuleStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryLabelRuleStore.java`
- Modify: `queues/pom.xml` — remove `commons-jexl3`
- Modify: `runtime/pom.xml` — remove `commons-jexl3`
- Delete tests for deleted types

**Interfaces:**
- Consumes: `LabelRuleStore` SPI (Task 1)
- Produces: `MongoLabelRuleStore`, `InMemoryLabelRuleStore`

- [ ] **Step 1: Create MongoLabelRuleStore**

Create `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoLabelRuleStore.java`. Follow `MongoFilterRuleStore` pattern — same CDI priority (`@Alternative @Priority(1)`), same tenant-scoped CRUD, adapted for `LabelRuleEntity`.

- [ ] **Step 2: Create InMemoryLabelRuleStore**

Create `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryLabelRuleStore.java`. Follow `InMemoryFilterRuleStore` pattern — same CDI priority (`@Alternative @Priority(100)`), ConcurrentHashMap-backed, adapted for `LabelRuleEntity`.

- [ ] **Step 3: Delete old types using ide_refactor_safe_delete**

Delete in dependency order (leaf types first, then types they depend on):

**queues module:**
1. `FilterChainStore` interface + `JpaFilterChainStore` impl
2. `FilterChain` entity
3. `FilterAction` record (queues/model)
4. `WorkItemFilterBean` interface
5. `LambdaFilterRegistry`
6. `WorkItemExpressionEvaluator` SPI
7. `ExpressionDescriptor` record
8. `FilterEvaluatorRegistry`
9. `JexlConditionEvaluator` (queues)
10. `JqConditionEvaluator`
11. `WorkItemFilterStore` interface + `JpaWorkItemFilterStore` impl
12. `WorkItemFilter` entity
13. `FilterEngine` interface
14. `FilterEngineImpl`
15. `FilterResource`

**runtime module:**
1. `ActionDescriptor` record
2. `FilterEvent` enum
3. `FilterAction` SPI interface
4. `ApplyLabelAction`
5. `OverrideCandidateGroupsAction`
6. `SetPriorityAction`
7. `FilterDefinition` record
8. `PermanentFilterRegistry`
9. `DynamicFilterRegistry`
10. `JexlConditionEvaluator` (runtime)
11. `FilterRegistryEngine`
12. `FilterRule` entity
13. `FilterRuleStore` interface + `JpaFilterRuleStore` impl

**persistence modules:**
1. `MongoFilterRuleStore`
2. `InMemoryFilterRuleStore`

**rest module:**
1. `FilterRuleResource`

Use `ide_refactor_safe_delete` for each — it checks for remaining references. If references exist, the delete will tell you what still depends on the type.

- [ ] **Step 4: Delete old test classes**

Delete all test files for deleted types: `FilterEngineTest`, `FilterRegistryEngineTest`, `JpaWorkItemFilterStoreTenancyTest`, `JpaFilterRuleStoreTenancyTest`, `ApplyLabelActionTest`, `OverrideCandidateGroupsActionTest`, `SetPriorityActionTest`, `PostgresWorkItemEventBroadcasterFilterTest` (if it references old types), test `FilterProducer` classes.

- [ ] **Step 5: Remove commons-jexl3 from queues/pom.xml and runtime/pom.xml**

The JEXL dependency is now transitive via `casehub-platform-expression`. Remove the direct `commons-jexl3` dependency from both modules.

- [ ] **Step 6: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ai -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues-dashboard -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues-examples -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb -f /Users/mdproctor/claude/casehub/work/pom.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -f /Users/mdproctor/claude/casehub/work/pom.xml
```

- [ ] **Step 7: Run integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests -f /Users/mdproctor/claude/casehub/work/pom.xml
```

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add .
git -C /Users/mdproctor/claude/casehub/work commit -m "refactor(#314): delete old filter types, add backend stores, remove jexl3 direct deps

Deletes ~30 types across queues, runtime, rest, and persistence modules.
Adds MongoLabelRuleStore and InMemoryLabelRuleStore.
JEXL now transitive via casehub-platform-expression.

Closes #314"
```
