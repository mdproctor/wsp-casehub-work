# CrossTenantProducer MongoDB Backend Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `CrossTenantProducer` to resolve cross-tenant stores by interface instead of JPA concrete types, and provide MongoDB implementations so the cross-tenant path uses the active persistence backend.

**Architecture:** `CrossTenantProducer` switches from injecting `JpaCrossTenant*Store` concrete types to injecting `CrossTenant*Store` interfaces. Three new Mongo store classes in `persistence-mongodb` implement those interfaces with `@Alternative @Priority(1)` and use Panache document classes for queries/deletes. One CDI wiring test validates the full `@CrossTenant` → producer → Mongo resolution chain.

**Tech Stack:** Java 21, Quarkus 3.32.2, MongoDB Panache, CDI (`@Alternative @Priority(1)`), AssertJ

**Spec:** `specs/2026-06-17-crosstenant-producer-mongodb-fix-design.md`
**Issue:** casehubio/work#267

---

### Task 1: Fix CrossTenantProducer — inject by interface

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/CrossTenantProducer.java`

- [ ] **Step 1: Change the three field types from JPA concrete to interfaces**

Replace the import block and field declarations. Remove 3 JPA imports, keep 3 interface imports (already present):

```java
package io.casehub.work.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.runtime.repository.CrossTenant;
import io.casehub.work.runtime.repository.CrossTenantRoutingCursorStore;
import io.casehub.work.runtime.repository.CrossTenantWorkItemScheduleStore;
import io.casehub.work.runtime.repository.CrossTenantWorkItemStore;

/**
 * CDI producer for {@code @CrossTenant} store variants.
 *
 * <p>Each {@code @Produces} method validates that the active
 * {@code CurrentPrincipal} is the system principal (qualified with
 * {@link WorkSystem}) and that {@code isCrossTenantAdmin()} returns
 * {@code true}. This prevents accidental injection of cross-tenant
 * stores in request-scoped contexts where tenant isolation is required.
 *
 * <p><strong>Usage:</strong>
 * <pre>{@code
 *   @Inject @WorkSystem CurrentPrincipal systemPrincipal;
 *   @Inject @CrossTenant CrossTenantWorkItemStore crossTenantStore;
 * }</pre>
 *
 * <p>Only system-level services (background jobs, timer callbacks,
 * admin endpoints running in a {@link TenantContextRunner} block)
 * should inject {@code @CrossTenant} stores.
 */
@ApplicationScoped
public class CrossTenantProducer {

    @Inject
    @WorkSystem
    CurrentPrincipal systemPrincipal;

    @Inject
    CrossTenantWorkItemStore crossTenantWorkItemStore;

    @Inject
    CrossTenantWorkItemScheduleStore crossTenantScheduleStore;

    @Inject
    CrossTenantRoutingCursorStore crossTenantCursorStore;

    /**
     * Produces a {@code @CrossTenant} {@link CrossTenantWorkItemStore}.
     *
     * @return cross-tenant WorkItem store
     * @throws IllegalStateException if system principal is not a cross-tenant admin
     */
    @Produces
    @CrossTenant
    @ApplicationScoped
    public CrossTenantWorkItemStore produceWorkItemStore() {
        validateSystemPrincipal();
        return crossTenantWorkItemStore;
    }

    /**
     * Produces a {@code @CrossTenant} {@link CrossTenantWorkItemScheduleStore}.
     *
     * @return cross-tenant WorkItemSchedule store
     * @throws IllegalStateException if system principal is not a cross-tenant admin
     */
    @Produces
    @CrossTenant
    @ApplicationScoped
    public CrossTenantWorkItemScheduleStore produceScheduleStore() {
        validateSystemPrincipal();
        return crossTenantScheduleStore;
    }

    /**
     * Produces a {@code @CrossTenant} {@link CrossTenantRoutingCursorStore}.
     *
     * @return cross-tenant RoutingCursor store
     * @throws IllegalStateException if system principal is not a cross-tenant admin
     */
    @Produces
    @CrossTenant
    @ApplicationScoped
    public CrossTenantRoutingCursorStore produceCursorStore() {
        validateSystemPrincipal();
        return crossTenantCursorStore;
    }

    /**
     * Validates that the system principal has cross-tenant admin privileges.
     *
     * @throws IllegalStateException if {@code isCrossTenantAdmin()} returns false
     */
    private void validateSystemPrincipal() {
        if (!systemPrincipal.isCrossTenantAdmin()) {
            throw new IllegalStateException(
                "SystemCurrentPrincipal.isCrossTenantAdmin() must return true");
        }
    }
}
```

- [ ] **Step 2: Compile the runtime module to verify no breakage**

Run: `scripts/mvn-compile runtime`
Expected: BUILD SUCCESS — the interface imports are already present; only the 3 JPA concrete imports are removed.

- [ ] **Step 3: Run runtime tests to verify existing behaviour unchanged**

Run: `scripts/mvn-test runtime`
Expected: All tests pass. The JPA implementations still satisfy the unqualified interface injection points because they are the only `@ApplicationScoped` beans implementing those interfaces in the runtime module.

- [ ] **Step 4: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/service/CrossTenantProducer.java
git commit -m "fix(#267): CrossTenantProducer injects by interface, not JPA concrete type

Refs #267"
```

---

### Task 2: MongoCrossTenantWorkItemStore — implementation + tests

**Files:**
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemStore.java`
- Create: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemStoreTest.java`

- [ ] **Step 1: Write the test class**

```java
package io.casehub.work.mongodb;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.CrossTenant;
import io.casehub.work.runtime.repository.CrossTenantWorkItemStore;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MongoCrossTenantWorkItemStoreTest {

    @Inject
    MutableCurrentPrincipal principal;

    @Inject
    CrossTenantWorkItemStore unqualifiedStore;

    @Inject
    @CrossTenant
    CrossTenantWorkItemStore crossTenantStore;

    @BeforeEach
    void setUp() {
        principal.reset();
        MongoWorkItemDocument.deleteAll();
    }

    @Test
    void findActiveWithDeadlines_returnsItemsAcrossTenants() {
        principal.setTenancyId("tenant-a");
        persistWorkItem("Tenant A expiry", WorkItemStatus.PENDING,
                Instant.now().plus(1, ChronoUnit.HOURS), null);

        principal.setTenancyId("tenant-b");
        persistWorkItem("Tenant B claim", WorkItemStatus.ASSIGNED,
                null, Instant.now().plus(2, ChronoUnit.HOURS));

        List<WorkItem> results = unqualifiedStore.findActiveWithDeadlines();

        assertThat(results).hasSize(2)
                .extracting(w -> w.title)
                .containsExactlyInAnyOrder("Tenant A expiry", "Tenant B claim");
    }

    @Test
    void findActiveWithDeadlines_excludesTerminalStatuses() {
        principal.setTenancyId("tenant-a");
        persistWorkItem("Completed", WorkItemStatus.COMPLETED,
                Instant.now().plus(1, ChronoUnit.HOURS), null);
        persistWorkItem("Expired", WorkItemStatus.EXPIRED,
                Instant.now().plus(1, ChronoUnit.HOURS), null);
        persistWorkItem("Cancelled", WorkItemStatus.CANCELLED,
                Instant.now().plus(1, ChronoUnit.HOURS), null);
        persistWorkItem("Rejected", WorkItemStatus.REJECTED,
                Instant.now().plus(1, ChronoUnit.HOURS), null);
        persistWorkItem("Escalated", WorkItemStatus.ESCALATED,
                Instant.now().plus(1, ChronoUnit.HOURS), null);
        persistWorkItem("Active", WorkItemStatus.PENDING,
                Instant.now().plus(1, ChronoUnit.HOURS), null);

        List<WorkItem> results = unqualifiedStore.findActiveWithDeadlines();

        assertThat(results).hasSize(1);
        assertThat(results.get(0).title).isEqualTo("Active");
    }

    @Test
    void findActiveWithDeadlines_excludesItemsWithNoDeadlines() {
        principal.setTenancyId("tenant-a");
        persistWorkItem("No deadline", WorkItemStatus.PENDING, null, null);
        persistWorkItem("Has expiry", WorkItemStatus.PENDING,
                Instant.now().plus(1, ChronoUnit.HOURS), null);

        List<WorkItem> results = unqualifiedStore.findActiveWithDeadlines();

        assertThat(results).hasSize(1);
        assertThat(results.get(0).title).isEqualTo("Has expiry");
    }

    @Test
    void cdiWiring_crossTenantQualifier_resolvesToMongoViaProducer() {
        principal.setTenancyId("tenant-a");
        persistWorkItem("Wiring test", WorkItemStatus.PENDING,
                Instant.now().plus(1, ChronoUnit.HOURS), null);

        List<WorkItem> results = crossTenantStore.findActiveWithDeadlines();

        assertThat(results).hasSize(1);
        assertThat(results.get(0).title).isEqualTo("Wiring test");
    }

    private void persistWorkItem(String title, WorkItemStatus status,
            Instant expiresAt, Instant claimDeadline) {
        WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.tenancyId = principal.tenancyId();
        wi.title = title;
        wi.status = status;
        wi.expiresAt = expiresAt;
        wi.claimDeadline = claimDeadline;
        wi.createdBy = "test";
        wi.createdAt = Instant.now();
        wi.updatedAt = Instant.now();
        MongoWorkItemDocument.from(wi).persist();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail (implementation missing)**

Run: `scripts/mvn-compile persistence-mongodb`
Expected: FAIL — `MongoCrossTenantWorkItemStore` class not found.

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.work.mongodb;

import java.util.List;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import org.bson.Document;

import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.CrossTenantWorkItemStore;

@ApplicationScoped
@Alternative
@Priority(1)
public class MongoCrossTenantWorkItemStore implements CrossTenantWorkItemStore {

    private static final List<String> TERMINAL_STATUSES = List.of(
            WorkItemStatus.COMPLETED.name(),
            WorkItemStatus.REJECTED.name(),
            WorkItemStatus.CANCELLED.name(),
            WorkItemStatus.EXPIRED.name(),
            WorkItemStatus.ESCALATED.name());

    @Override
    public List<WorkItem> findActiveWithDeadlines() {
        Document filter = new Document("$and", List.of(
                new Document("status", new Document("$nin", TERMINAL_STATUSES)),
                new Document("$or", List.of(
                        new Document("expiresAt", new Document("$ne", null)),
                        new Document("claimDeadline", new Document("$ne", null))))));

        return MongoWorkItemDocument.<MongoWorkItemDocument>find(filter).list()
                .stream()
                .map(MongoWorkItemDocument::toDomain)
                .toList();
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoCrossTenantWorkItemStoreTest`
Expected: All 4 tests pass.

- [ ] **Step 5: Commit**

```
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemStore.java
git add persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemStoreTest.java
git commit -m "feat(#267): MongoCrossTenantWorkItemStore with CDI wiring test

Refs #267"
```

---

### Task 3: MongoCrossTenantWorkItemScheduleStore — implementation + tests

**Files:**
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemScheduleStore.java`
- Create: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemScheduleStoreTest.java`

- [ ] **Step 1: Write the test class**

```java
package io.casehub.work.mongodb;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.runtime.model.WorkItemSchedule;
import io.casehub.work.runtime.repository.CrossTenantWorkItemScheduleStore;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MongoCrossTenantWorkItemScheduleStoreTest {

    @Inject
    MutableCurrentPrincipal principal;

    @Inject
    CrossTenantWorkItemScheduleStore store;

    @BeforeEach
    void setUp() {
        principal.reset();
        MongoWorkItemScheduleDocument.deleteAll();
    }

    @Test
    void findActive_returnsSchedulesAcrossTenants() {
        principal.setTenancyId("tenant-a");
        persistSchedule("Tenant A active", true);

        principal.setTenancyId("tenant-b");
        persistSchedule("Tenant B active", true);

        List<WorkItemSchedule> results = store.findActive();

        assertThat(results).hasSize(2)
                .extracting(s -> s.name)
                .containsExactlyInAnyOrder("Tenant A active", "Tenant B active");
    }

    @Test
    void findActive_excludesInactiveSchedules() {
        principal.setTenancyId("tenant-a");
        persistSchedule("Active", true);
        persistSchedule("Inactive", false);

        List<WorkItemSchedule> results = store.findActive();

        assertThat(results).hasSize(1);
        assertThat(results.get(0).name).isEqualTo("Active");
    }

    private void persistSchedule(String name, boolean active) {
        MongoWorkItemScheduleDocument doc = new MongoWorkItemScheduleDocument();
        doc.id = UUID.randomUUID().toString();
        doc.tenancyId = principal.tenancyId();
        doc.name = name;
        doc.templateId = UUID.randomUUID().toString();
        doc.cronExpression = "0 0 9 * * ?";
        doc.active = active;
        doc.createdBy = "test";
        doc.createdAt = Instant.now();
        doc.version = 0L;
        doc.persist();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail (implementation missing)**

Run: `scripts/mvn-compile persistence-mongodb`
Expected: FAIL — `MongoCrossTenantWorkItemScheduleStore` class not found.

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.work.mongodb;

import java.util.List;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import org.bson.Document;

import io.casehub.work.runtime.model.WorkItemSchedule;
import io.casehub.work.runtime.repository.CrossTenantWorkItemScheduleStore;

@ApplicationScoped
@Alternative
@Priority(1)
public class MongoCrossTenantWorkItemScheduleStore implements CrossTenantWorkItemScheduleStore {

    @Override
    public List<WorkItemSchedule> findActive() {
        Document filter = new Document("active", true);

        return MongoWorkItemScheduleDocument.<MongoWorkItemScheduleDocument>find(filter).list()
                .stream()
                .map(MongoWorkItemScheduleDocument::toDomain)
                .toList();
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoCrossTenantWorkItemScheduleStoreTest`
Expected: All 2 tests pass.

- [ ] **Step 5: Commit**

```
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemScheduleStore.java
git add persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoCrossTenantWorkItemScheduleStoreTest.java
git commit -m "feat(#267): MongoCrossTenantWorkItemScheduleStore

Refs #267"
```

---

### Task 4: MongoCrossTenantRoutingCursorStore — implementation + tests

**Files:**
- Create: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoCrossTenantRoutingCursorStore.java`
- Create: `persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoCrossTenantRoutingCursorStoreTest.java`

- [ ] **Step 1: Write the test class**

```java
package io.casehub.work.mongodb;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.runtime.repository.CrossTenantRoutingCursorStore;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MongoCrossTenantRoutingCursorStoreTest {

    @Inject
    MutableCurrentPrincipal principal;

    @Inject
    CrossTenantRoutingCursorStore store;

    @BeforeEach
    void setUp() {
        principal.reset();
        MongoRoutingCursorDocument.deleteAll();
    }

    @Test
    void cleanupStale_deletesOldCursorsAcrossTenants() {
        Instant stale = Instant.now().minus(10, ChronoUnit.DAYS);
        Instant fresh = Instant.now();

        persistCursor("pool1:tenant-a", 3, stale);
        persistCursor("pool2:tenant-b", 7, stale);
        persistCursor("pool3:tenant-a", 1, fresh);

        Instant cutoff = Instant.now().minus(5, ChronoUnit.DAYS);
        long deleted = store.cleanupStale(cutoff);

        assertThat(deleted).isEqualTo(2);
        assertThat(MongoRoutingCursorDocument.count()).isEqualTo(1);
    }

    @Test
    void cleanupStale_returnsZero_whenNoneStale() {
        Instant fresh = Instant.now();
        persistCursor("pool1:tenant-a", 3, fresh);

        Instant cutoff = Instant.now().minus(5, ChronoUnit.DAYS);
        long deleted = store.cleanupStale(cutoff);

        assertThat(deleted).isEqualTo(0);
        assertThat(MongoRoutingCursorDocument.count()).isEqualTo(1);
    }

    @Test
    void cleanupStale_returnsZero_whenEmpty() {
        Instant cutoff = Instant.now().minus(5, ChronoUnit.DAYS);
        long deleted = store.cleanupStale(cutoff);

        assertThat(deleted).isEqualTo(0);
    }

    private void persistCursor(String id, long lastIndex, Instant lastAccessed) {
        MongoRoutingCursorDocument doc = new MongoRoutingCursorDocument();
        doc.id = id;
        doc.lastIndex = lastIndex;
        doc.lastAccessed = lastAccessed;
        doc.persist();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail (implementation missing)**

Run: `scripts/mvn-compile persistence-mongodb`
Expected: FAIL — `MongoCrossTenantRoutingCursorStore` class not found.

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.work.mongodb;

import java.time.Instant;
import java.util.Date;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import org.bson.Document;

import io.casehub.work.runtime.repository.CrossTenantRoutingCursorStore;

@ApplicationScoped
@Alternative
@Priority(1)
public class MongoCrossTenantRoutingCursorStore implements CrossTenantRoutingCursorStore {

    @Override
    public long cleanupStale(Instant cutoff) {
        return MongoRoutingCursorDocument.delete(
                new Document("lastAccessed", new Document("$lt", Date.from(cutoff))));
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `scripts/mvn-test persistence-mongodb -Dtest=MongoCrossTenantRoutingCursorStoreTest`
Expected: All 3 tests pass.

- [ ] **Step 5: Commit**

```
git add persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoCrossTenantRoutingCursorStore.java
git add persistence-mongodb/src/test/java/io/casehub/work/mongodb/MongoCrossTenantRoutingCursorStoreTest.java
git commit -m "feat(#267): MongoCrossTenantRoutingCursorStore

Refs #267"
```

---

### Task 5: Full test suite verification

**Files:** None (verification only)

- [ ] **Step 1: Run the full persistence-mongodb test suite**

Run: `scripts/mvn-test persistence-mongodb`
Expected: All tests pass — existing store tests plus 9 new cross-tenant tests.

- [ ] **Step 2: Run the runtime test suite (regression check)**

Run: `scripts/mvn-test runtime`
Expected: All tests pass — `CrossTenantProducer` now injects by interface, but JPA impls still satisfy the injection in the runtime test CDI container.

- [ ] **Step 3: Compile integration-tests to confirm no wiring issues**

Run: `scripts/mvn-compile integration-tests`
Expected: BUILD SUCCESS.
