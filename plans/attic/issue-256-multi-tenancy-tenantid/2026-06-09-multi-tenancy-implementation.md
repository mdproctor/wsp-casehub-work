# Multi-Tenancy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add tenant isolation to every entity, store, event, and SSE stream in casehub-work; replace polling schedulers with per-item Quartz timers carrying tenant context.

**Architecture:** Application-managed tenancy filtering via `CurrentPrincipal.tenancyId()` in every store. Cross-tenant access via separate `@CrossTenant`-qualified store interfaces (engine pattern). Scheduler polling replaced with per-item Quartz JDBC-backed timers. `TenantContextRunner` for async/background tenant context establishment.

**Tech Stack:** Java 21, Quarkus 3.32, Hibernate ORM/Panache (blocking), Quartz JDBC store, H2 (tests), PostgreSQL (prod)

**Spec:** `specs/issue-256-multi-tenancy-tenantid/2026-06-08-multi-tenancy-design.md` (revision 6)

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module> -Dtest=<TestClass>`

**Key conventions:**
- `tenancyId` on Java fields/accessors; `tenancy_id` on SQL columns
- `TenancyConstants.DEFAULT_TENANT_ID = "278776f9-e1b0-46fb-9032-8bddebdcf9ce"`
- Stores stamp `tenancyId` on insert (when null); preserve on update (when already set)
- Tenant-scoped stores never check `isCrossTenantAdmin()` — unconditional filtering
- Cross-tenant stores are separate interfaces with narrowly scoped methods

---

## Phase 1: Foundation

### Task 1: Flyway V35 — runtime tenancy_id columns

**Files:**
- Create: `runtime/src/main/resources/db/work/migration/V35__tenancy_id.sql`

- [ ] **Step 1: Write the migration SQL**

```sql
-- V35: Add tenancy_id to all runtime entities
ALTER TABLE work_item ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE work_item_template ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE audit_entry ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE work_item_note ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE work_item_link ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE work_item_spawn_group ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE work_item_schedule ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE work_item_relation ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE routing_cursor ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE label_definition ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE label_vocabulary ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';
ALTER TABLE filter_rule ADD COLUMN tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce';

-- Indexes
CREATE INDEX idx_work_item_tenancy ON work_item(tenancy_id);
CREATE INDEX idx_work_item_template_tenancy ON work_item_template(tenancy_id);
CREATE INDEX idx_audit_entry_tenancy ON audit_entry(tenancy_id);
CREATE INDEX idx_work_item_note_tenancy ON work_item_note(tenancy_id);
CREATE INDEX idx_work_item_link_tenancy ON work_item_link(tenancy_id);
CREATE INDEX idx_work_item_spawn_group_tenancy ON work_item_spawn_group(tenancy_id);
CREATE INDEX idx_work_item_schedule_tenancy ON work_item_schedule(tenancy_id);
CREATE INDEX idx_work_item_relation_tenancy ON work_item_relation(tenancy_id);
CREATE INDEX idx_label_definition_tenancy ON label_definition(tenancy_id);
CREATE INDEX idx_label_vocabulary_tenancy ON label_vocabulary(tenancy_id);
CREATE INDEX idx_filter_rule_tenancy ON filter_rule(tenancy_id);

-- WorkItemTemplate: unique(name) → unique(name, tenancy_id)
ALTER TABLE work_item_template DROP CONSTRAINT IF EXISTS uq_work_item_template_name;
ALTER TABLE work_item_template ADD CONSTRAINT uq_work_item_template_name_tenant UNIQUE(name, tenancy_id);

-- RoutingCursor: PK (pool_hash) → PK (pool_hash, tenancy_id)
ALTER TABLE routing_cursor DROP CONSTRAINT IF EXISTS routing_cursor_pkey;
ALTER TABLE routing_cursor ADD PRIMARY KEY (pool_hash, tenancy_id);
```

- [ ] **Step 2: Add `tenancyId` field to WorkItem entity**

Add to `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`:

```java
@Column(name = "tenancy_id", nullable = false)
public String tenancyId;
```

Place it after the `version` field, before the core descriptive fields section.

- [ ] **Step 3: Add `tenancyId` field to all other runtime entities**

Add the same `@Column(name = "tenancy_id", nullable = false) public String tenancyId;` field to:
- `WorkItemTemplate.java`
- `AuditEntry.java`
- `WorkItemNote.java`
- `WorkItemLink.java`
- `WorkItemSpawnGroup.java`
- `WorkItemSchedule.java`
- `WorkItemRelation.java`
- `LabelDefinition.java`
- `LabelVocabulary.java`
- `FilterRule.java`

For `RoutingCursor.java`: add the field AND update the `@Id` annotation — RoutingCursor now has a composite PK of `(poolHash, tenancyId)`. This requires an `@IdClass` or `@EmbeddedId`. Use `@IdClass(RoutingCursorId.class)`:

Create `runtime/src/main/java/io/casehub/work/runtime/model/RoutingCursorId.java`:
```java
package io.casehub.work.runtime.model;

import java.io.Serializable;
import java.util.Objects;

public class RoutingCursorId implements Serializable {
    public String poolHash;
    public String tenancyId;

    public RoutingCursorId() {}

    public RoutingCursorId(String poolHash, String tenancyId) {
        this.poolHash = poolHash;
        this.tenancyId = tenancyId;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof RoutingCursorId that)) return false;
        return Objects.equals(poolHash, that.poolHash) && Objects.equals(tenancyId, that.tenancyId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(poolHash, tenancyId);
    }
}
```

Update `RoutingCursor.java`:
```java
@Entity
@Table(name = "routing_cursor")
@IdClass(RoutingCursorId.class)
public class RoutingCursor extends PanacheEntityBase {

    @Id
    @Column(name = "pool_hash", length = 64, nullable = false, updatable = false)
    public String poolHash;

    @Id
    @Column(name = "tenancy_id", nullable = false)
    public String tenancyId;
    // ... rest unchanged
}
```

- [ ] **Step 4: Update WorkItemTemplate unique constraint annotation**

In `WorkItemTemplate.java`, change:
```java
@UniqueConstraint(name = "uq_work_item_template_name", columnNames = "name")
```
to:
```java
@UniqueConstraint(name = "uq_work_item_template_name_tenant", columnNames = {"name", "tenancy_id"})
```

- [ ] **Step 5: Build runtime module to verify entity changes compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/resources/db/work/migration/V35__tenancy_id.sql
git add runtime/src/main/java/io/casehub/work/runtime/model/
git commit -m "feat(#256): V35 migration + tenancyId field on all runtime entities (Refs #256)"
```

---

### Task 2: Infrastructure — @CrossTenant, SystemCurrentPrincipal, CrossTenantProducer, TenantContextRunner

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/CrossTenant.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/WorkSystem.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/SystemCurrentPrincipal.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/CrossTenantProducer.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/TenantContextRunner.java`
- Create: `runtime/src/test/java/io/casehub/work/runtime/test/MutableCurrentPrincipal.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/TenantContextRunnerTest.java`

- [ ] **Step 1: Create @CrossTenant CDI qualifier**

```java
package io.casehub.work.runtime.repository;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;
import jakarta.inject.Qualifier;

@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE})
public @interface CrossTenant {}
```

- [ ] **Step 2: Create @WorkSystem qualifier**

```java
package io.casehub.work.runtime.service;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;
import jakarta.inject.Qualifier;

@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE})
public @interface WorkSystem {}
```

- [ ] **Step 3: Create SystemCurrentPrincipal**

```java
package io.casehub.work.runtime.service;

import java.util.Set;
import jakarta.enterprise.context.ApplicationScoped;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;

@ApplicationScoped
@WorkSystem
public class SystemCurrentPrincipal implements CurrentPrincipal {

    @Override
    public String actorId() {
        return "system";
    }

    @Override
    public Set<String> groups() {
        return Set.of();
    }

    @Override
    public String tenancyId() {
        return TenancyConstants.DEFAULT_TENANT_ID;
    }

    @Override
    public boolean isCrossTenantAdmin() {
        return true;
    }
}
```

- [ ] **Step 4: Write failing test for TenantContextRunner**

```java
package io.casehub.work.runtime.service;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class TenantContextRunnerTest {

    @Inject
    TenantContextRunner tenantContextRunner;

    @Test
    void runInTenantContext_establishes_principal_with_given_tenancyId() {
        var result = new String[1];
        tenantContextRunner.runInTenantContext("test-tenant-abc", () -> {
            CurrentPrincipal principal = io.quarkus.arc.Arc.container()
                .instance(CurrentPrincipal.class).get();
            result[0] = principal.tenancyId();
        });
        assertThat(result[0]).isEqualTo("test-tenant-abc");
    }

    @Test
    void runInTenantContext_principal_is_not_cross_tenant_admin() {
        var result = new boolean[1];
        tenantContextRunner.runInTenantContext("test-tenant-xyz", () -> {
            CurrentPrincipal principal = io.quarkus.arc.Arc.container()
                .instance(CurrentPrincipal.class).get();
            result[0] = principal.isCrossTenantAdmin();
        });
        assertThat(result[0]).isFalse();
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TenantContextRunnerTest`
Expected: FAIL — `TenantContextRunner` class not found

- [ ] **Step 6: Implement TenantContextRunner**

```java
package io.casehub.work.runtime.service;

import java.util.Set;
import jakarta.enterprise.context.ApplicationScoped;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.arc.Arc;
import io.quarkus.arc.ManagedContext;

@ApplicationScoped
public class TenantContextRunner {

    public void runInTenantContext(String tenancyId, Runnable work) {
        ManagedContext requestContext = Arc.container().requestContext();
        boolean wasActive = requestContext.isActive();
        if (!wasActive) {
            requestContext.activate();
        }
        try {
            var syntheticPrincipal = new SyntheticPrincipal(tenancyId);
            Arc.container().beanManager()
                .createInstance()
                .select(CurrentPrincipal.class)
                .stream()
                .findFirst();
            // Use CDI programmatic lookup with the synthetic principal
            // The cleaner approach: use Arc's InstanceHandle to set the bean
            // For Quarkus Arc, we use InjectableContext to add the bean
            var handle = Arc.container().instance(CurrentPrincipal.class);
            // Actually, in Quarkus Arc we need a different approach.
            // Use a ThreadLocal on a @RequestScoped bean.
            throw new UnsupportedOperationException("Implement with RequestScoped ThreadLocal pattern");
        } finally {
            if (!wasActive) {
                requestContext.terminate();
            }
        }
    }

    private record SyntheticPrincipal(String tenancyId) implements CurrentPrincipal {
        @Override public String actorId() { return "system"; }
        @Override public Set<String> groups() { return Set.of(); }
        @Override public boolean isCrossTenantAdmin() { return false; }
    }
}
```

**Implementation note:** The exact CDI mechanism for establishing a synthetic `CurrentPrincipal` in a programmatically activated request context depends on Quarkus Arc's API. The implementation must:
1. Activate a request context if not already active
2. Make the synthetic `CurrentPrincipal` resolvable via `@Inject CurrentPrincipal` within the scope
3. Run the work
4. Tear down

The recommended Quarkus pattern is a `@RequestScoped` bean with a `ThreadLocal` holder that `TenantContextRunner` sets before invoking the work. This avoids CDI synthetic bean registration. The implementor should explore `Arc.container().requestContext()` + a `TenantHolder` `@RequestScoped` bean whose `tenancyId()` reads from the ThreadLocal. The tests above verify the contract.

- [ ] **Step 7: Create MutableCurrentPrincipal for tests**

```java
package io.casehub.work.runtime.test;

import java.util.Set;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;

@ApplicationScoped
@Alternative
@Priority(100)
public class MutableCurrentPrincipal implements CurrentPrincipal {

    private String actorId = "test-user";
    private String tenancyId = TenancyConstants.DEFAULT_TENANT_ID;
    private boolean crossTenantAdmin = false;
    private Set<String> groups = Set.of();

    @Override
    public String actorId() { return actorId; }

    @Override
    public Set<String> groups() { return groups; }

    @Override
    public String tenancyId() { return tenancyId; }

    @Override
    public boolean isCrossTenantAdmin() { return crossTenantAdmin; }

    public void setActorId(String actorId) { this.actorId = actorId; }
    public void setTenancyId(String tenancyId) { this.tenancyId = tenancyId; }
    public void setCrossTenantAdmin(boolean crossTenantAdmin) { this.crossTenantAdmin = crossTenantAdmin; }
    public void setGroups(Set<String> groups) { this.groups = groups; }

    public void reset() {
        this.actorId = "test-user";
        this.tenancyId = TenancyConstants.DEFAULT_TENANT_ID;
        this.crossTenantAdmin = false;
        this.groups = Set.of();
    }
}
```

- [ ] **Step 8: Run tests and verify TenantContextRunner passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TenantContextRunnerTest`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/repository/CrossTenant.java
git add runtime/src/main/java/io/casehub/work/runtime/service/WorkSystem.java
git add runtime/src/main/java/io/casehub/work/runtime/service/SystemCurrentPrincipal.java
git add runtime/src/main/java/io/casehub/work/runtime/service/TenantContextRunner.java
git add runtime/src/test/java/io/casehub/work/runtime/test/MutableCurrentPrincipal.java
git add runtime/src/test/java/io/casehub/work/runtime/service/TenantContextRunnerTest.java
git commit -m "feat(#256): infrastructure — @CrossTenant, SystemCurrentPrincipal, TenantContextRunner, MutableCurrentPrincipal (Refs #256)"
```

---

### Task 3: Extend WorkItemStore with tenant filtering + isolation tests

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStoreTenancyTest.java`

- [ ] **Step 1: Write failing tenant isolation test**

```java
package io.casehub.work.runtime.repository.jpa;

import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.casehub.work.runtime.test.MutableCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class JpaWorkItemStoreTenancyTest {

    private static final String TENANT_A = "tenant-a";
    private static final String TENANT_B = "tenant-b";

    @Inject WorkItemStore store;
    @Inject MutableCurrentPrincipal principal;

    @BeforeEach
    void resetPrincipal() {
        principal.reset();
    }

    @Test
    @Transactional
    void get_returns_empty_for_other_tenants_item() {
        principal.setTenancyId(TENANT_A);
        var item = createWorkItem("Tenant A item");
        var saved = store.put(item);

        principal.setTenancyId(TENANT_B);
        assertThat(store.get(saved.id)).isEmpty();
    }

    @Test
    @Transactional
    void put_stamps_tenancyId_on_insert() {
        principal.setTenancyId(TENANT_A);
        var item = createWorkItem("Stamped item");
        var saved = store.put(item);
        assertThat(saved.tenancyId).isEqualTo(TENANT_A);
    }

    @Test
    @Transactional
    void put_preserves_tenancyId_on_update() {
        principal.setTenancyId(TENANT_A);
        var item = createWorkItem("Original tenant");
        var saved = store.put(item);

        principal.setTenancyId(TENANT_B);
        saved.title = "Updated title";
        var updated = store.put(saved);
        assertThat(updated.tenancyId).isEqualTo(TENANT_A);
    }

    @Test
    @Transactional
    void scan_returns_only_current_tenant_items() {
        principal.setTenancyId(TENANT_A);
        store.put(createWorkItem("A1"));
        store.put(createWorkItem("A2"));

        principal.setTenancyId(TENANT_B);
        store.put(createWorkItem("B1"));

        principal.setTenancyId(TENANT_A);
        var results = store.scan(io.casehub.work.runtime.repository.WorkItemQuery.all());
        assertThat(results).hasSize(2);
        assertThat(results).allSatisfy(wi -> assertThat(wi.tenancyId).isEqualTo(TENANT_A));
    }

    private WorkItem createWorkItem(String title) {
        var item = new WorkItem();
        item.title = title;
        item.status = WorkItemStatus.PENDING;
        item.createdBy = "test";
        return item;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaWorkItemStoreTenancyTest`
Expected: FAIL — no tenant filtering in JpaWorkItemStore

- [ ] **Step 3: Implement tenant filtering in JpaWorkItemStore**

Inject `CurrentPrincipal` into `JpaWorkItemStore`. In `put()`: if `workItem.tenancyId == null`, stamp from `currentPrincipal.tenancyId()`. In `get()`: add `AND tenancyId = :tenancyId`. In `scan()`: prepend `tenancyId = :tenancyId` to the JPQL. Apply the same to `findByCallerRef()`, `scanRoots()`, `countByParentAndAssignee()`, `scanAll()`.

Pattern for every query method:
```java
@Inject CurrentPrincipal currentPrincipal;

@Override
public Optional<WorkItem> get(final UUID id) {
    return WorkItem.find("id = ?1 AND tenancyId = ?2", id, currentPrincipal.tenancyId())
        .firstResultOptional();
}
```

For `put()`:
```java
@Override
public WorkItem put(final WorkItem workItem) {
    if (workItem.tenancyId == null) {
        workItem.tenancyId = currentPrincipal.tenancyId();
    }
    workItem.persistAndFlush();
    return workItem;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaWorkItemStoreTenancyTest`
Expected: PASS

- [ ] **Step 5: Run existing WorkItemStore tests to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All existing tests pass (MockCurrentPrincipal provides DEFAULT_TENANT_ID; all existing data has DEFAULT_TENANT_ID from V35 migration)

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java
git add runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStoreTenancyTest.java
git commit -m "feat(#256): tenant filtering on JpaWorkItemStore — put/get/scan/scanRoots (Refs #256)"
```

---

### Task 4: Extend AuditEntryStore, WorkItemNoteStore, RoutingCursorStore with tenant filtering

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaAuditEntryStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemNoteStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaRoutingCursorStore.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaAuditEntryStoreTenancyTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemNoteStoreTenancyTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaRoutingCursorStoreTenancyTest.java`

- [ ] **Step 1: Write failing tenant isolation tests for each store**

Same pattern as Task 3: create data as tenant A, switch to tenant B, assert invisible. One test class per store.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaAuditEntryStoreTenancyTest,JpaWorkItemNoteStoreTenancyTest,JpaRoutingCursorStoreTenancyTest"`
Expected: FAIL

- [ ] **Step 3: Inject CurrentPrincipal and add tenant predicate to every query method in each store**

Same pattern as Task 3. For `RoutingCursorStore`: remove any `cleanupStale` method from the tenant-scoped store — GC is exclusively on `CrossTenantRoutingCursorStore` (Task 10).

- [ ] **Step 4: Run tests to verify they pass**

Expected: PASS

- [ ] **Step 5: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/repository/jpa/
git add runtime/src/test/java/io/casehub/work/runtime/repository/jpa/
git commit -m "feat(#256): tenant filtering on AuditEntryStore, WorkItemNoteStore, RoutingCursorStore (Refs #256)"
```

---

### Task 5: New store — WorkItemTemplateStore

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemTemplateStore.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemTemplateStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java` — replace all static Panache calls
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemTemplateResource.java` — replace all static Panache calls
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemTemplateStoreTenancyTest.java`

- [ ] **Step 1: Define WorkItemTemplateStore SPI**

```java
package io.casehub.work.runtime.repository;

import java.util.List;
import java.util.Optional;
import java.util.UUID;
import io.casehub.work.runtime.model.WorkItemTemplate;

public interface WorkItemTemplateStore {
    WorkItemTemplate put(WorkItemTemplate template);
    Optional<WorkItemTemplate> get(UUID id);
    Optional<WorkItemTemplate> getByName(String name);
    List<WorkItemTemplate> scanAll();
    boolean delete(UUID id);
}
```

- [ ] **Step 2: Write failing tenant isolation test**

Same pattern: create template as tenant A, query as tenant B → empty. Test `getByName` returns only current tenant's template. Test that two tenants can create templates with the same name.

- [ ] **Step 3: Run test to verify it fails**

Expected: FAIL

- [ ] **Step 4: Implement JpaWorkItemTemplateStore**

```java
package io.casehub.work.runtime.repository.jpa;

import java.util.List;
import java.util.Optional;
import java.util.UUID;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.casehub.work.runtime.repository.WorkItemTemplateStore;

@ApplicationScoped
public class JpaWorkItemTemplateStore implements WorkItemTemplateStore {

    @Inject CurrentPrincipal currentPrincipal;

    @Override
    public WorkItemTemplate put(WorkItemTemplate template) {
        if (template.tenancyId == null) {
            template.tenancyId = currentPrincipal.tenancyId();
        }
        template.persistAndFlush();
        return template;
    }

    @Override
    public Optional<WorkItemTemplate> get(UUID id) {
        return WorkItemTemplate.find("id = ?1 AND tenancyId = ?2",
            id, currentPrincipal.tenancyId()).firstResultOptional();
    }

    @Override
    public Optional<WorkItemTemplate> getByName(String name) {
        return WorkItemTemplate.find("name = ?1 AND tenancyId = ?2",
            name, currentPrincipal.tenancyId()).firstResultOptional();
    }

    @Override
    public List<WorkItemTemplate> scanAll() {
        return WorkItemTemplate.find("tenancyId = ?1 ORDER BY name ASC",
            currentPrincipal.tenancyId()).list();
    }

    @Override
    public boolean delete(UUID id) {
        return WorkItemTemplate.delete("id = ?1 AND tenancyId = ?2",
            id, currentPrincipal.tenancyId()) > 0;
    }
}
```

- [ ] **Step 5: Wire WorkItemTemplateService and WorkItemTemplateResource to use the store**

Replace every `WorkItemTemplate.findById()`, `WorkItemTemplate.listAllByName()`, `WorkItemTemplate.deleteById()`, `WorkItemTemplate.find("name", ...)` with corresponding `templateStore` calls. Inject `WorkItemTemplateStore` into both classes.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaWorkItemTemplateStoreTenancyTest,WorkItemTemplateTest,WorkItemTemplateSchemaTest,WorkItemTemplateOutcomeTest,WorkItemTemplateInstantiateTest,WorkItemTemplateServiceTest,WorkItemTemplatePatchTest"`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemTemplateStore.java
git add runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemTemplateStore.java
git add runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java
git add runtime/src/main/java/io/casehub/work/runtime/api/WorkItemTemplateResource.java
git add runtime/src/test/java/
git commit -m "feat(#256): WorkItemTemplateStore — tenant-scoped template access, eliminate static Panache (Refs #256)"
```

---

### Task 6: New stores — WorkItemRelationStore, WorkItemSpawnGroupStore, WorkItemScheduleStore, WorkItemLinkStore

**Files:**
- Create: SPI interfaces in `runtime/src/main/java/io/casehub/work/runtime/repository/`
- Create: JPA implementations in `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/`
- Modify: All callers that use static Panache on these entities
- Test: Tenancy isolation tests per store

Follow the same TDD pattern as Task 5 for each store. For each:
1. Define the SPI interface with the methods needed (from the spec's store table)
2. Write tenant isolation test
3. Implement JPA store with `CurrentPrincipal` injection
4. Wire callers to use the store (eliminate static Panache calls)
5. Run tests

**Key callers to wire:**
- `WorkItemRelation`: `WorkItemRelationResource`, `WorkItemResource`, `LedgerEventCapture`, `WorkItemSpawnService`
- `WorkItemSpawnGroup`: `WorkItemSpawnService`, `WorkItemSpawnResource`, `MultiInstanceGroupPolicy`, `WorkItemInstancesResource`, `WorkItemService.claim()`
- `WorkItemSchedule`: `WorkItemScheduleService`, `WorkItemScheduleResource`
- `WorkItemLink`: `WorkItemResource`

- [ ] **Step 1: Define all four SPI interfaces**
- [ ] **Step 2: Write tenant isolation tests for all four**
- [ ] **Step 3: Implement all four JPA stores**
- [ ] **Step 4: Wire all callers — eliminate every static Panache call for these entities**
- [ ] **Step 5: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/repository/
git add runtime/src/main/java/io/casehub/work/runtime/repository/jpa/
git add runtime/src/main/java/io/casehub/work/runtime/service/
git add runtime/src/main/java/io/casehub/work/runtime/api/
git add runtime/src/main/java/io/casehub/work/runtime/multiinstance/
git add runtime/src/test/java/
git commit -m "feat(#256): RelationStore, SpawnGroupStore, ScheduleStore, LinkStore — tenant-scoped (Refs #256)"
```

---

### Task 7: New stores — LabelDefinitionStore, LabelVocabularyStore, FilterRuleStore

**Files:**
- Create: SPI interfaces in `runtime/src/main/java/io/casehub/work/runtime/repository/`
- Create: JPA implementations in `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/`
- Modify: All callers (VocabularyResource, FilterRuleResource, DynamicFilterRegistry, FilterRegistryEngine)
- Test: Tenancy isolation tests per store

Same TDD pattern. Wire callers to eliminate `LabelDefinition.findByVocabularyId()`, `LabelDefinition.findByPath()`, `LabelVocabulary` static calls, `FilterRule.allEnabled()`.

- [ ] **Step 1–5: Same pattern as Task 5/6**
- [ ] **Step 6: Run full runtime test suite**
- [ ] **Step 7: Commit**

---

### Task 8: Cross-tenant stores

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/CrossTenantWorkItemStore.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/CrossTenantWorkItemScheduleStore.java`
- Create: `runtime/src/main/java/io/casehub/work/runtime/repository/CrossTenantRoutingCursorStore.java`
- Create: JPA implementations in `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/`
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/CrossTenantProducer.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/CrossTenantWorkItemStoreTest.java`

- [ ] **Step 1: Define cross-tenant store interfaces**

```java
package io.casehub.work.runtime.repository;

import java.util.List;
import io.casehub.work.runtime.model.WorkItem;

public interface CrossTenantWorkItemStore {
    List<WorkItem> findActiveWithDeadlines();
}
```

```java
package io.casehub.work.runtime.repository;

import java.util.List;
import io.casehub.work.runtime.model.WorkItemSchedule;

public interface CrossTenantWorkItemScheduleStore {
    List<WorkItemSchedule> findActive();
}
```

```java
package io.casehub.work.runtime.repository;

import java.time.Instant;

public interface CrossTenantRoutingCursorStore {
    long cleanupStale(Instant cutoff);
}
```

- [ ] **Step 2: Write test — cross-tenant store sees all tenants**

```java
@QuarkusTest
class CrossTenantWorkItemStoreTest {

    @Inject WorkItemStore tenantStore;
    @Inject @CrossTenant CrossTenantWorkItemStore crossTenantStore;
    @Inject MutableCurrentPrincipal principal;

    @Test
    @Transactional
    void findActiveWithDeadlines_returns_items_from_all_tenants() {
        principal.setTenancyId("tenant-a");
        var itemA = createItemWithExpiry("A item");
        tenantStore.put(itemA);

        principal.setTenancyId("tenant-b");
        var itemB = createItemWithExpiry("B item");
        tenantStore.put(itemB);

        var all = crossTenantStore.findActiveWithDeadlines();
        assertThat(all).hasSizeGreaterThanOrEqualTo(2);
    }
}
```

- [ ] **Step 3: Implement JPA cross-tenant stores (no tenant predicate)**
- [ ] **Step 4: Implement CrossTenantProducer**

```java
package io.casehub.work.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.runtime.repository.CrossTenant;
import io.casehub.work.runtime.repository.CrossTenantWorkItemStore;
import io.casehub.work.runtime.repository.CrossTenantWorkItemScheduleStore;
import io.casehub.work.runtime.repository.CrossTenantRoutingCursorStore;
import io.casehub.work.runtime.repository.jpa.JpaCrossTenantWorkItemStore;
import io.casehub.work.runtime.repository.jpa.JpaCrossTenantWorkItemScheduleStore;
import io.casehub.work.runtime.repository.jpa.JpaCrossTenantRoutingCursorStore;

@ApplicationScoped
public class CrossTenantProducer {

    @Inject @WorkSystem CurrentPrincipal systemPrincipal;
    @Inject JpaCrossTenantWorkItemStore crossTenantWorkItemStore;
    @Inject JpaCrossTenantWorkItemScheduleStore crossTenantScheduleStore;
    @Inject JpaCrossTenantRoutingCursorStore crossTenantCursorStore;

    @Produces @CrossTenant @ApplicationScoped
    public CrossTenantWorkItemStore produceWorkItemStore() {
        validateSystemPrincipal();
        return crossTenantWorkItemStore;
    }

    @Produces @CrossTenant @ApplicationScoped
    public CrossTenantWorkItemScheduleStore produceScheduleStore() {
        validateSystemPrincipal();
        return crossTenantScheduleStore;
    }

    @Produces @CrossTenant @ApplicationScoped
    public CrossTenantRoutingCursorStore produceCursorStore() {
        validateSystemPrincipal();
        return crossTenantCursorStore;
    }

    private void validateSystemPrincipal() {
        if (!systemPrincipal.isCrossTenantAdmin()) {
            throw new IllegalStateException(
                "SystemCurrentPrincipal.isCrossTenantAdmin() must return true");
        }
    }
}
```

- [ ] **Step 5: Wire RoutingCursorCleanupJob to use @CrossTenant CrossTenantRoutingCursorStore**

Replace `RoutingCursor.delete(...)` with `@Inject @CrossTenant CrossTenantRoutingCursorStore` and call `cleanupStale(cutoff)`.

- [ ] **Step 6: Run tests**
- [ ] **Step 7: Commit**

---

## Phase 2: CDI Events and SSE

### Task 9: CDI events — add tenancyId

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEvent.java`
- Modify: `api/src/main/java/io/casehub/work/api/WorkItemGroupLifecycleEvent.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/SlaBreachEvent.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/event/WorkItemQueueEvent.java`
- Modify: All fire sites in `WorkItemService`, `ExpiryLifecycleService`, `MultiInstanceGroupPolicy`, `FilterEvaluationObserver`

- [ ] **Step 1: Add `tenancyId` to WorkItemLifecycleEvent**

Add `private final String tenancyId` field. Update `of()` factory to read `workItem.tenancyId`. Update `fromWire()` to accept `tenancyId` parameter. Add `@JsonIgnore` on the `tenancyId()` accessor. Add a `tenancyId()` getter.

- [ ] **Step 2: Add `tenancyId` to WorkItemGroupLifecycleEvent**

Add `private final String tenancyId` field. Update `of()` factory to accept `tenancyId`. Add `tenancyId()` getter.

- [ ] **Step 3: Add `tenancyId` to SlaBreachEvent**

Add field, update constructor.

- [ ] **Step 4: Add `tenancyId` to WorkItemQueueEvent record**

Change record to: `WorkItemQueueEvent(UUID workItemId, UUID queueViewId, String queueName, QueueEventType eventType, String tenancyId)`

- [ ] **Step 5: Update all fire sites**

In `WorkItemService`: the `workItem.tenancyId` is already set by the store's `put()` — `WorkItemLifecycleEvent.of()` reads it from the entity. No change needed at fire sites if the factory reads from `workItem.tenancyId`.

In `MultiInstanceGroupPolicy`: update `WorkItemGroupLifecycleEvent.of()` call to pass parent's `tenancyId`.

In `FilterEvaluationObserver`: update `WorkItemQueueEvent` construction to include `tenancyId`.

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests pass

- [ ] **Step 7: Commit**

---

### Task 10: SSE broadcasters — tenant-scoped streams

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemEventBroadcaster.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/LocalWorkItemEventBroadcaster.java`
- Modify: `postgres-broadcaster/src/main/java/io/casehub/work/postgres/broadcaster/WorkItemEventPayload.java`
- Modify: `postgres-broadcaster/src/main/java/io/casehub/work/postgres/broadcaster/PostgresWorkItemEventBroadcaster.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/service/WorkItemQueueEventBroadcaster.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/service/LocalWorkItemQueueEventBroadcaster.java`
- Modify: `queues-postgres-broadcaster/src/main/java/io/casehub/work/queues/postgres/broadcaster/PostgresWorkItemQueueEventBroadcaster.java`
- Modify: SSE REST endpoints that call `stream()` — pass `currentPrincipal.tenancyId()`

- [ ] **Step 1: Add `tenancyId` parameter to WorkItemEventBroadcaster.stream()**

```java
Multi<WorkItemLifecycleEvent> stream(UUID workItemId, String type, String tenancyId);
```

- [ ] **Step 2: Update LocalWorkItemEventBroadcaster — filter stream by tenancyId**

```java
@Override
public Multi<WorkItemLifecycleEvent> stream(UUID workItemId, String type, String tenancyId) {
    Multi<WorkItemLifecycleEvent> source = processor.toHotStream();
    source = source.filter(e -> tenancyId.equals(e.tenancyId()));
    // ... existing workItemId and type filters
    return source;
}
```

- [ ] **Step 3: Add `tenancyId` to WorkItemEventPayload**

Add `@JsonProperty("tenancyId") String tenancyId` to the record. Update `from()` and `toEvent()`.

- [ ] **Step 4: Update PostgresWorkItemEventBroadcaster — filter stream by tenancyId**

- [ ] **Step 5: Same for WorkItemQueueEventBroadcaster and its implementations**

Add `tenancyId` parameter to `stream(UUID queueViewId, String tenancyId)`. Update both implementations.

- [ ] **Step 6: Update SSE REST endpoints to pass tenancyId from CurrentPrincipal**

In `WorkItemResource.streamEvents()`: inject `CurrentPrincipal`, pass `currentPrincipal.tenancyId()` to `broadcaster.stream()`.
In `QueueResource.streamQueueEvents()`: same pattern.

- [ ] **Step 7: Run tests across affected modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,postgres-broadcaster,queues,queues-postgres-broadcaster`
Expected: All tests pass

- [ ] **Step 8: Commit**

---

## Phase 3: Optional Module Stores

### Task 11: Optional module Flyway migrations + entity tenancyId fields

**Files:**
- Create: `queues/src/main/resources/db/work/migration/V2003__tenancy_id.sql`
- Create: `notifications/src/main/resources/db/work/migration/V3001__tenancy_id.sql`
- Create: `ai/src/main/resources/db/work/migration/V4002__tenancy_id.sql`
- Create: `issue-tracker/src/main/resources/db/work/migration/V5002__tenancy_id.sql`
- Modify: All optional-module entity classes — add `tenancyId` field

- [ ] **Step 1: Write migration SQL for each module** (same ADD COLUMN pattern as V35)
- [ ] **Step 2: Add `tenancyId` field to each entity class**
- [ ] **Step 3: Build each module to verify compilation**
- [ ] **Step 4: Commit**

---

### Task 12: Queues module stores (5 stores)

**Files:**
- Create: `QueueViewStore`, `QueueMembershipStore`, `WorkItemFilterStore`, `FilterChainStore`, `QueueStateStore` — SPIs and JPA implementations in `queues/src/main/java/`
- Modify: `FilterEvaluationObserver`, `QueueMembershipTracker`, `FilterEngineImpl`, `FilterResource`, `QueueResource`, `QueueStateResource`, `WorkItemQueueMetrics`
- Test: Tenancy isolation tests per store

Follow Task 5 pattern for each store. Wire all 13 static Panache calls listed in the spec.

- [ ] **Steps 1–6: TDD cycle per store, wire callers**
- [ ] **Step 7: Run full queues test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues`
Expected: All tests pass

- [ ] **Step 8: Commit**

---

### Task 13: Notifications, AI, Issue-tracker, Ledger module stores

**Files:**
- Create: `NotificationRuleStore` (notifications module)
- Create: `WorkerSkillProfileStore`, `EscalationSummaryStore` (ai module)
- Modify: `IssueLinkStore` / `JpaIssueLinkStore` — add tenant filtering (already exists, extend)
- Modify: `WorkItemLedgerEntryRepository` / `JpaWorkItemLedgerEntryRepository` — add tenant filtering
- Modify: All callers in each module
- Test: Tenancy isolation tests

**NotificationDispatcher special handling:** requires both (a) tenant-scoped rule loading via `NotificationRuleStore.findEnabledForEventType()` in the AFTER_SUCCESS observer, and (b) `TenantContextRunner.runInTenantContext()` wrapping for `CompletableFuture.runAsync()` channel dispatch.

- [ ] **Steps 1–6: TDD cycle per store/module**
- [ ] **Step 7: Run tests per module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notifications
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ai
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl issue-tracker
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger
```

- [ ] **Step 8: Commit per module**

---

### Task 13b: ReportService — tenant-scoped JPQL

**Files:**
- Modify: `reports/src/main/java/io/casehub/work/reports/service/ReportService.java`
- Test: `reports/src/test/java/io/casehub/work/reports/service/ReportServiceTenancyTest.java`

ReportService builds JPQL via `EntityManager.createQuery()`. Inject `CurrentPrincipal` and add `AND w.tenancyId = :tenancyId` to every query. This is a data access class — injecting CurrentPrincipal directly is acceptable per spec.

- [ ] **Step 1: Write failing test — reports scoped to tenant**
- [ ] **Step 2: Add tenant predicate to every EntityManager query in ReportService**
- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl reports`
Expected: All tests pass

- [ ] **Step 4: Commit**

---

## Phase 4: Scheduler Architecture

### Task 14: WorkItemTimerService — scheduling infrastructure

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTimerService.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTimerServiceTest.java`

- [ ] **Step 1: Write failing test for timer scheduling**

```java
@QuarkusTest
class WorkItemTimerServiceTest {

    @Inject WorkItemTimerService timerService;

    @Test
    void scheduleExpiry_creates_quartz_trigger() {
        UUID workItemId = UUID.randomUUID();
        String tenancyId = "test-tenant";
        Instant fireAt = Instant.now().plusSeconds(3600);

        timerService.scheduleExpiry(workItemId, tenancyId, fireAt);

        assertThat(timerService.hasExpiryTimer(workItemId)).isTrue();
    }

    @Test
    void cancelExpiry_removes_quartz_trigger() {
        UUID workItemId = UUID.randomUUID();
        timerService.scheduleExpiry(workItemId, "t", Instant.now().plusSeconds(3600));
        timerService.cancelExpiry(workItemId);
        assertThat(timerService.hasExpiryTimer(workItemId)).isFalse();
    }

    @Test
    void rescheduleExpiry_updates_fire_time() {
        UUID workItemId = UUID.randomUUID();
        Instant first = Instant.now().plusSeconds(3600);
        Instant second = Instant.now().plusSeconds(7200);

        timerService.scheduleExpiry(workItemId, "t", first);
        timerService.rescheduleExpiry(workItemId, second);

        assertThat(timerService.hasExpiryTimer(workItemId)).isTrue();
    }
}
```

- [ ] **Step 2: Implement WorkItemTimerService**

```java
package io.casehub.work.runtime.service;

import java.time.Instant;
import java.util.Date;
import java.util.UUID;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.quartz.*;

@ApplicationScoped
public class WorkItemTimerService {

    @Inject Scheduler scheduler;

    public void scheduleExpiry(UUID workItemId, String tenancyId, Instant fireAt) {
        JobDetail job = JobBuilder.newJob(ExpiryTimerJob.class)
            .withIdentity("expiry-" + workItemId, "work-expiry")
            .usingJobData("workItemId", workItemId.toString())
            .usingJobData("tenancyId", tenancyId)
            .build();

        Trigger trigger = TriggerBuilder.newTrigger()
            .withIdentity("expiry-trigger-" + workItemId, "work-expiry")
            .startAt(Date.from(fireAt))
            .build();

        try {
            if (scheduler.checkExists(job.getKey())) {
                scheduler.deleteJob(job.getKey());
            }
            scheduler.scheduleJob(job, trigger);
        } catch (SchedulerException e) {
            throw new RuntimeException("Failed to schedule expiry timer for " + workItemId, e);
        }
    }

    public void cancelExpiry(UUID workItemId) {
        try {
            scheduler.deleteJob(JobKey.jobKey("expiry-" + workItemId, "work-expiry"));
        } catch (SchedulerException e) {
            // Idempotent — job may not exist
        }
    }

    public void rescheduleExpiry(UUID workItemId, Instant newFireAt) {
        try {
            TriggerKey key = TriggerKey.triggerKey("expiry-trigger-" + workItemId, "work-expiry");
            Trigger newTrigger = TriggerBuilder.newTrigger()
                .withIdentity(key)
                .startAt(Date.from(newFireAt))
                .build();
            scheduler.rescheduleJob(key, newTrigger);
        } catch (SchedulerException e) {
            throw new RuntimeException("Failed to reschedule expiry for " + workItemId, e);
        }
    }

    public boolean hasExpiryTimer(UUID workItemId) {
        try {
            return scheduler.checkExists(JobKey.jobKey("expiry-" + workItemId, "work-expiry"));
        } catch (SchedulerException e) {
            return false;
        }
    }

    // Same pattern for scheduleClaimDeadline / cancelClaimDeadline / rescheduleClaimDeadline
    // using group "work-claim-deadline"
}
```

- [ ] **Step 3: Create ExpiryTimerJob and ClaimDeadlineTimerJob Quartz job classes**

```java
package io.casehub.work.runtime.service;

import java.util.UUID;
import jakarta.inject.Inject;
import org.quartz.Job;
import org.quartz.JobExecutionContext;

public class ExpiryTimerJob implements Job {

    @Inject TenantContextRunner tenantContextRunner;
    @Inject ExpiryLifecycleService expiryService;

    @Override
    public void execute(JobExecutionContext context) {
        String workItemId = context.getJobDetail().getJobDataMap().getString("workItemId");
        String tenancyId = context.getJobDetail().getJobDataMap().getString("tenancyId");

        tenantContextRunner.runInTenantContext(tenancyId, () -> {
            expiryService.expireItem(UUID.fromString(workItemId));
        });
    }
}
```

- [ ] **Step 4: Run tests**
- [ ] **Step 5: Commit**

---

### Task 15: Wire timer management into WorkItemService lifecycle transitions

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryLifecycleService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemSpawnService.java`
- Remove: `runtime/src/main/java/io/casehub/work/runtime/service/ExpiryCleanupJob.java`
- Remove: `runtime/src/main/java/io/casehub/work/runtime/service/ClaimDeadlineJob.java`

Implement the state transition × timer action matrix from the spec:

- [ ] **Step 1: Inject WorkItemTimerService into WorkItemService**

- [ ] **Step 2: Add timer calls to each lifecycle method**

In `create()` — after `workItemStore.put(item)`:
```java
if (saved.expiresAt != null) {
    timerService.scheduleExpiry(saved.id, saved.tenancyId, saved.expiresAt);
}
if (saved.claimDeadline != null) {
    timerService.scheduleClaimDeadline(saved.id, saved.tenancyId, saved.claimDeadline);
}
```

In `claim()` — after assignment: `timerService.cancelClaimDeadline(saved.id);`

In `complete()`, `reject()`, `cancel()`:
```java
timerService.cancelExpiry(saved.id);
timerService.cancelClaimDeadline(saved.id);
```

In `extend()`: `timerService.rescheduleExpiry(saved.id, newExpiresAt);`

In `delegate()`: `timerService.cancelClaimDeadline(saved.id);`

In `declineDelegation()`, `release()`:
```java
timerService.scheduleClaimDeadline(saved.id, saved.tenancyId, saved.claimDeadline);
```

In `suspend()`:
```java
timerService.cancelExpiry(saved.id);
timerService.cancelClaimDeadline(saved.id);
```

In `resume()`:
```java
if (saved.expiresAt != null) timerService.scheduleExpiry(saved.id, saved.tenancyId, saved.expiresAt);
if (saved.claimDeadline != null) timerService.scheduleClaimDeadline(saved.id, saved.tenancyId, saved.claimDeadline);
```

- [ ] **Step 3: Wire timer calls in ExpiryLifecycleService**

In `executeEscalateTo()` — after setting new `expiresAt`/`claimDeadline`:
```java
timerService.rescheduleExpiry(item.id, item.expiresAt);
timerService.scheduleClaimDeadline(item.id, item.tenancyId, item.claimDeadline);
```

In `executeExtend()` — after setting new deadline: reschedule the appropriate timer.

In `executeFail()`, `executeExhausted()` — cancel claim deadline timer.

- [ ] **Step 4: Refactor ExpiryLifecycleService — extract per-item expiry method**

Create `expireItem(UUID workItemId)` that loads and processes a single item. The Quartz job calls this. Remove the batch scan from `checkExpired()`.

- [ ] **Step 5: Delete ExpiryCleanupJob and ClaimDeadlineJob**

These are replaced by Quartz timer jobs.

- [ ] **Step 6: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: Tests that relied on the old @Scheduled pattern need updating — they should now use `WorkItemTimerService` or test expiry by directly calling `expiryService.expireItem()`.

- [ ] **Step 7: Commit**

---

### Task 16: Startup recovery bean

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/service/TimerRecoveryStartup.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/service/TimerRecoveryStartupTest.java`

- [ ] **Step 1: Write test for recovery scan**
- [ ] **Step 2: Implement TimerRecoveryStartup**

```java
package io.casehub.work.runtime.service;

import io.startup.Startup;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.casehub.work.runtime.repository.CrossTenant;
import io.casehub.work.runtime.repository.CrossTenantWorkItemStore;
import io.casehub.work.runtime.repository.CrossTenantWorkItemScheduleStore;

@ApplicationScoped
@Startup
public class TimerRecoveryStartup {

    @Inject @CrossTenant CrossTenantWorkItemStore crossTenantWorkItemStore;
    @Inject @CrossTenant CrossTenantWorkItemScheduleStore crossTenantScheduleStore;
    @Inject WorkItemTimerService timerService;

    void onStartup(@Observes StartupEvent event) {
        var items = crossTenantWorkItemStore.findActiveWithDeadlines();
        for (var item : items) {
            if (item.expiresAt != null) {
                timerService.scheduleExpiry(item.id, item.tenancyId, item.expiresAt);
            }
            if (item.claimDeadline != null) {
                timerService.scheduleClaimDeadline(item.id, item.tenancyId, item.claimDeadline);
            }
        }
        // Schedule recurring triggers for active WorkItemSchedules
        var schedules = crossTenantScheduleStore.findActive();
        for (var schedule : schedules) {
            timerService.scheduleRecurring(schedule.id, schedule.tenancyId, /* cron/interval */);
        }
    }
}
```

- [ ] **Step 3: Run tests**
- [ ] **Step 4: Commit**

---

## Phase 5: InMemory and MongoDB Stores

### Task 17: InMemory store tenant filtering (persistence-memory)

**Files:**
- Modify: All `InMemory*Store` classes in `persistence-memory/src/main/java/`
- Test: Tenancy isolation tests in `persistence-memory/src/test/java/`

For each `InMemory*Store`: inject `CurrentPrincipal`, filter backing collections by `tenancyId` on every read, stamp on every write. Create new InMemory implementations for the new store SPIs (WorkItemTemplateStore, etc.).

- [ ] **Step 1: Add tenant filtering to existing InMemory stores**
- [ ] **Step 2: Create InMemory implementations for new store SPIs**
- [ ] **Step 3: Write tenancy isolation tests**
- [ ] **Step 4: Run persistence-memory tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: All tests pass

- [ ] **Step 5: Commit**

---

### Task 18: MongoDB store tenant filtering + data migration

**Files:**
- Modify: `MongoWorkItemStore.java`, `MongoAuditEntryStore.java` in `persistence-mongodb/src/main/java/`
- Create: MongoDB migration bean for backfilling `tenancyId` on existing documents
- Test: Tenancy isolation tests

- [ ] **Step 1: Add tenancyId to all MongoDB filter documents**
- [ ] **Step 2: Create @Startup migration bean for backfilling**
- [ ] **Step 3: Write tenancy isolation tests**
- [ ] **Step 4: Run persistence-mongodb tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-mongodb`
Expected: All tests pass

- [ ] **Step 5: Commit**

---

## Phase 6: @ObservesAsync Handlers

### Task 19: Wire TenantContextRunner into @ObservesAsync handlers

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceCoordinator.java`
- Modify: `queues-dashboard/src/main/java/.../QueueDashboard.java`
- Modify: `notifications/src/main/java/.../NotificationDispatcher.java`

- [ ] **Step 1: MultiInstanceCoordinator — wrap in runInTenantContext**

```java
@ObservesAsync WorkItemLifecycleEvent event → {
    String tenancyId = ((WorkItem) event.source()).tenancyId;
    tenantContextRunner.runInTenantContext(tenancyId, () -> {
        // existing logic — all store calls now have tenant context
    });
}
```

- [ ] **Step 2: QueueDashboard — wrap in runInTenantContext**

- [ ] **Step 3: NotificationDispatcher — tenant-scoped rule loading + async dispatch**

The AFTER_SUCCESS observer has CurrentPrincipal (request context still alive). Rule loading via `NotificationRuleStore.findEnabledForEventType()` is tenant-scoped. The `CompletableFuture.runAsync()` dispatch wraps in `tenantContextRunner.runInTenantContext(tenancyId, ...)`.

- [ ] **Step 4: Run tests across affected modules**
- [ ] **Step 5: Commit**

---

## Phase 7: Integration Tests

### Task 20: End-to-end multi-tenant integration tests

**Files:**
- Create: `integration-tests/src/test/java/io/casehub/work/it/MultiTenancyIntegrationTest.java`

- [ ] **Step 1: Write two-tenant REST isolation tests**

```java
@QuarkusIntegrationTest
class MultiTenancyIntegrationTest {

    @Test
    void workItem_invisible_across_tenants() {
        // POST as tenant A → 201
        // GET same ID as tenant B → 404
    }

    @Test
    void template_same_name_different_tenants() {
        // POST template "review" as tenant A → 201
        // POST template "review" as tenant B → 201 (not a conflict)
        // GET templates as tenant A → 1 result
    }

    @Test
    void audit_isolated_per_tenant() {
        // create + complete as tenant A
        // GET audit as tenant B → empty
    }
}
```

- [ ] **Step 2: Write SSE stream isolation test**

Subscribe to SSE as tenant A, fire event as tenant B, assert tenant A receives nothing.

- [ ] **Step 3: Write expiry isolation test**

Create WorkItems with short expiry in both tenants. Wait for timer fire. Assert each tenant's item expired independently with correct tenancyId on events.

- [ ] **Step 4: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests`
Expected: All tests pass

- [ ] **Step 5: Commit**

---

## Phase 8: Verification

### Task 21: Static Panache audit + full build

- [ ] **Step 1: Verify zero static Panache calls in production code**

Use IntelliJ MCP to search for `WorkItem.find`, `WorkItem.findById`, `WorkItem.listAll`, `WorkItem.count`, `WorkItem.delete` etc. across all production source directories. Exclude store implementations and test code.

- [ ] **Step 2: Full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS, all modules pass

- [ ] **Step 3: Native image integration tests**

Run: `JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home mvn verify -Pnative -pl integration-tests`
Expected: All tests pass

- [ ] **Step 4: Final commit — update docs**

Update `ARC42STORIES.MD` and `CLAUDE.md` to reflect the multi-tenancy architecture.

```bash
git commit -m "docs(#256): update ARC42STORIES.MD and CLAUDE.md for multi-tenancy (Refs #256)"
```
