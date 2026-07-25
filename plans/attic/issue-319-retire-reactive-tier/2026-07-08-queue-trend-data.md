# Queue Trend Data Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #289 — Historical queue trend data endpoint for sparkline visualisation
**Issue group:** #289

**Goal:** Add periodic queue membership snapshots and a `GET /queues/{id}/trend` endpoint for sparkline visualisation.

**Architecture:** A Quartz `@Scheduled` heartbeat job snapshots queue member counts into a new `QueueSnapshot` JPA entity. Snapshot interval and retention are `PreferenceKey<DurationPreference>` preferences. A REST endpoint queries the snapshots for trend data. A shared `QueueMembershipService` extracts membership evaluation from `QueueResource` and adds a count-only path for the job.

**Tech Stack:** Java 21, Quarkus 3.32, Hibernate ORM/Panache, Flyway, Quartz scheduler, casehub-platform PreferenceProvider SPI

## Global Constraints

- Java 21 source level (running on Java 26 JVM)
- Flyway migration in range V2000–V2999 (queues module range); next safe: V2004
- Migration path: `db/work/migration/` (per repo-scoped migration path convention)
- Multi-tenancy: every entity has `tenancyId`, stores extend `TenantAwareStore`
- Preferences: use `PreferenceKey<DurationPreference>` (platform `DurationPreference` record), not `@ConfigProperty`
- Cross-tenant stores: follow `cross-tenant-store-minimal-surface` protocol (PP-20260609-2144e0)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Test scripts: use `scripts/` helpers when available for timeouts
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and refactoring

---

### Task 1: Add `countByQuery` to WorkItemStore SPI

Adds a count method to the store SPI so `QueueMembershipService.countMembers()` can count matching WorkItems without hydrating entities.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java`
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemStore.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/WorkItemStoreCountTest.java`

**Interfaces:**
- Consumes: `WorkItemQuery` (existing), `WorkItemStore.scan(WorkItemQuery)` (existing pattern)
- Produces: `WorkItemStore.countByQuery(WorkItemQuery): long` — used by Task 4 (`QueueMembershipService.countMembers`)

- [ ] **Step 1: Write failing test for countByQuery**

Create `WorkItemStoreCountTest.java` in `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/`:

```java
package io.casehub.work.runtime.repository.jpa;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.repository.WorkItemQuery;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

@QuarkusTest
class WorkItemStoreCountTest {

    @Inject
    WorkItemStore workItemStore;

    @Test
    @Transactional
    void countByQuery_returnsMatchingCount() {
        final WorkItem wi1 = TestWorkItemFactory.pending("count-test-1");
        wi1.persist();
        final WorkItem wi2 = TestWorkItemFactory.pending("count-test-2");
        wi2.persist();

        final long count = workItemStore.countByQuery(
                WorkItemQuery.builder().build());

        assertThat(count).isGreaterThanOrEqualTo(2);
    }

    @Test
    @Transactional
    void countByQuery_byLabelPattern_filtersCorrectly() {
        final WorkItem wi = TestWorkItemFactory.pending("label-count-test");
        wi.persist();

        final long count = workItemStore.countByQuery(
                WorkItemQuery.byLabelPattern("nonexistent-pattern/**"));

        assertThat(count).isEqualTo(0);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemStoreCountTest`
Expected: compilation failure — `countByQuery` method does not exist

- [ ] **Step 3: Add `countByQuery` default method to WorkItemStore interface**

Add to `WorkItemStore.java` after the `scan` method (around line 82):

```java
/**
 * Counts WorkItems matching the query without hydrating entities.
 * Default implementation delegates to {@link #scan} — JPA overrides
 * with a native COUNT query for efficiency.
 */
default long countByQuery(WorkItemQuery query) {
    return scan(query).size();
}
```

- [ ] **Step 4: Override in JpaWorkItemStore with a Panache count query**

Add to `JpaWorkItemStore.java`. Follow the existing `scan()` method's query-building pattern but use `count()` instead of `list()`:

```java
@Override
public long countByQuery(final WorkItemQuery query) {
    if (query.labelPattern() != null) {
        return WorkItem.count("SELECT COUNT(DISTINCT w) FROM WorkItem w JOIN w.labels l WHERE w.tenancyId = ?1 AND l.path LIKE ?2",
                currentPrincipal.tenancyId(),
                query.labelPattern().replace("**", "%").replace("*", "%"));
    }
    return scan(query).size();
}
```

Check the existing `scan()` label-pattern query in `JpaWorkItemStore` for the exact HQL pattern and adapt the COUNT version from it.

- [ ] **Step 5: Implement in InMemoryWorkItemStore**

The default method already delegates to `scan().size()` which works for the in-memory store. No override needed — verify the default works.

- [ ] **Step 6: Implement in MongoWorkItemStore**

Same as in-memory — the default delegates to `scan().size()`. Override only if a native MongoDB count query is warranted (check existing `MongoWorkItemStore.scan()` pattern). For now, the default suffices.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=WorkItemStoreCountTest`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemStore.java \
       runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java \
       runtime/src/test/java/io/casehub/work/runtime/repository/jpa/WorkItemStoreCountTest.java
git commit -m "feat(#289): add countByQuery to WorkItemStore SPI

Default delegates to scan().size(); JPA overrides with COUNT query
for label-pattern queries. Refs #289"
```

---

### Task 2: QueueSnapshot Entity, Flyway Migration, and Store

Creates the data layer: entity, database table, store interface, JPA implementation, and in-memory implementation.

**Files:**
- Create: `queues/src/main/java/io/casehub/work/queues/model/QueueSnapshot.java`
- Create: `queues/src/main/java/io/casehub/work/queues/repository/QueueSnapshotStore.java`
- Create: `queues/src/main/java/io/casehub/work/queues/repository/jpa/JpaQueueSnapshotStore.java`
- Create: `queues/src/main/resources/db/work/migration/V2004__queue_snapshot.sql`
- Create: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryQueueSnapshotStore.java`
- Modify: `persistence-memory/pom.xml` — add `casehub-work-queues` dependency
- Test: `queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaQueueSnapshotStoreTest.java`

**Interfaces:**
- Consumes: `TenantAwareStore` (existing base class), `QueueView.id` (existing entity)
- Produces: `QueueSnapshotStore` interface — used by Tasks 5 and 6; `QueueSnapshot` entity — used by Tasks 5 and 6; `InMemoryQueueSnapshotStore` — used by Task 5 tests

- [ ] **Step 1: Create QueueSnapshot entity**

```java
package io.casehub.work.queues.model;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

@Entity
@Table(name = "queue_snapshot")
public class QueueSnapshot extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "tenancy_id", nullable = false)
    public String tenancyId;

    @Column(name = "queue_view_id", nullable = false)
    public UUID queueViewId;

    @Column(name = "member_count", nullable = false)
    public long memberCount;

    @Column(name = "snapshot_at", nullable = false)
    public Instant snapshotAt;

    @PrePersist
    void prePersist() {
        if (id == null) id = UUID.randomUUID();
    }
}
```

- [ ] **Step 2: Create Flyway migration V2004**

Create `queues/src/main/resources/db/work/migration/V2004__queue_snapshot.sql`:

```sql
CREATE TABLE queue_snapshot (
    id            UUID         NOT NULL,
    tenancy_id    VARCHAR(255) NOT NULL,
    queue_view_id UUID         NOT NULL,
    member_count  BIGINT       NOT NULL,
    snapshot_at   TIMESTAMP    NOT NULL,
    CONSTRAINT pk_queue_snapshot PRIMARY KEY (id),
    CONSTRAINT fk_queue_snapshot_queue_view
        FOREIGN KEY (queue_view_id) REFERENCES queue_view(id) ON DELETE CASCADE,
    CONSTRAINT uq_queue_snapshot_tenant_queue_time
        UNIQUE (tenancy_id, queue_view_id, snapshot_at)
);

CREATE INDEX idx_queue_snapshot_trend
    ON queue_snapshot (tenancy_id, queue_view_id, snapshot_at);
```

- [ ] **Step 3: Create QueueSnapshotStore interface**

```java
package io.casehub.work.queues.repository;

import java.time.Instant;
import java.util.Collection;
import java.util.Map;
import java.util.UUID;
import java.util.List;

import io.casehub.work.queues.model.QueueSnapshot;

public interface QueueSnapshotStore {

    QueueSnapshot put(QueueSnapshot snapshot);

    List<QueueSnapshot> findByQueueAndPeriod(UUID queueViewId, Instant from, Instant to);

    Map<UUID, Instant> findLatestSnapshotTimes(Collection<UUID> queueViewIds);

    void deleteOlderThan(Instant cutoff);
}
```

- [ ] **Step 4: Create JpaQueueSnapshotStore**

Follow `JpaQueueViewStore` pattern — extend `TenantAwareStore`:

```java
package io.casehub.work.queues.repository.jpa;

import java.time.Instant;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.queues.model.QueueSnapshot;
import io.casehub.work.queues.repository.QueueSnapshotStore;

@ApplicationScoped
public class JpaQueueSnapshotStore implements QueueSnapshotStore {

    @Inject
    CurrentPrincipal currentPrincipal;

    @Override
    @Transactional
    public QueueSnapshot put(final QueueSnapshot snapshot) {
        snapshot.tenancyId = currentPrincipal.tenancyId();
        snapshot.persist();
        return snapshot;
    }

    @Override
    public List<QueueSnapshot> findByQueueAndPeriod(
            final UUID queueViewId, final Instant from, final Instant to) {
        return QueueSnapshot.find(
                "tenancyId = ?1 AND queueViewId = ?2 AND snapshotAt >= ?3 AND snapshotAt <= ?4 ORDER BY snapshotAt ASC",
                currentPrincipal.tenancyId(), queueViewId, from, to)
                .list();
    }

    @Override
    @SuppressWarnings("unchecked")
    public Map<UUID, Instant> findLatestSnapshotTimes(final Collection<UUID> queueViewIds) {
        if (queueViewIds.isEmpty()) return Map.of();
        final List<Object[]> rows = QueueSnapshot.find(
                "SELECT queueViewId, MAX(snapshotAt) FROM QueueSnapshot WHERE tenancyId = ?1 AND queueViewId IN ?2 GROUP BY queueViewId",
                currentPrincipal.tenancyId(), queueViewIds)
                .project(Object[].class).list();
        final Map<UUID, Instant> result = new HashMap<>();
        for (final Object[] row : rows) {
            result.put((UUID) row[0], (Instant) row[1]);
        }
        return result;
    }

    @Override
    @Transactional
    public void deleteOlderThan(final Instant cutoff) {
        QueueSnapshot.delete("tenancyId = ?1 AND snapshotAt < ?2",
                currentPrincipal.tenancyId(), cutoff);
    }
}
```

Note: check how `JpaQueueViewStore` handles queries — adapt the tenant-scoped pattern. The `findLatestSnapshotTimes` query uses Panache's projection API — verify the exact syntax works with aggregate queries. If not, fall back to `getEntityManager().createQuery()`.

- [ ] **Step 5: Write store integration test**

```java
package io.casehub.work.queues.repository.jpa;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import io.casehub.work.queues.model.QueueSnapshot;
import io.casehub.work.queues.repository.QueueSnapshotStore;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

@QuarkusTest
class JpaQueueSnapshotStoreTest {

    @Inject
    QueueSnapshotStore store;

    @Test
    @Transactional
    void put_and_findByQueueAndPeriod_roundTrips() {
        final UUID queueId = UUID.randomUUID();
        final Instant now = Instant.now();
        final QueueSnapshot snap = new QueueSnapshot();
        snap.queueViewId = queueId;
        snap.memberCount = 42;
        snap.snapshotAt = now;
        store.put(snap);

        final List<QueueSnapshot> found = store.findByQueueAndPeriod(
                queueId, now.minusSeconds(60), now.plusSeconds(60));
        assertThat(found).hasSize(1);
        assertThat(found.get(0).memberCount).isEqualTo(42);
    }

    @Test
    @Transactional
    void findLatestSnapshotTimes_returnsBatchResult() {
        final UUID q1 = UUID.randomUUID();
        final UUID q2 = UUID.randomUUID();
        final Instant t1 = Instant.parse("2026-07-07T10:00:00Z");
        final Instant t2 = Instant.parse("2026-07-07T11:00:00Z");

        putSnapshot(q1, 10, t1);
        putSnapshot(q1, 12, t2);
        putSnapshot(q2, 5, t1);

        final Map<UUID, Instant> latest = store.findLatestSnapshotTimes(List.of(q1, q2));
        assertThat(latest.get(q1)).isEqualTo(t2);
        assertThat(latest.get(q2)).isEqualTo(t1);
    }

    @Test
    @Transactional
    void deleteOlderThan_prunesOldSnapshots() {
        final UUID queueId = UUID.randomUUID();
        final Instant old = Instant.parse("2026-06-01T00:00:00Z");
        final Instant recent = Instant.now();
        putSnapshot(queueId, 10, old);
        putSnapshot(queueId, 20, recent);

        store.deleteOlderThan(Instant.parse("2026-07-01T00:00:00Z"));

        final List<QueueSnapshot> remaining = store.findByQueueAndPeriod(
                queueId, Instant.EPOCH, Instant.MAX);
        assertThat(remaining).hasSize(1);
        assertThat(remaining.get(0).memberCount).isEqualTo(20);
    }

    private void putSnapshot(final UUID queueId, final long count, final Instant at) {
        final QueueSnapshot s = new QueueSnapshot();
        s.queueViewId = queueId;
        s.memberCount = count;
        s.snapshotAt = at;
        store.put(s);
    }
}
```

- [ ] **Step 6: Add casehub-work-queues dependency to persistence-memory pom.xml**

Add to `persistence-memory/pom.xml` dependencies:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-queues</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 7: Create InMemoryQueueSnapshotStore**

```java
package io.casehub.work.memory;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.work.queues.model.QueueSnapshot;
import io.casehub.work.queues.repository.QueueSnapshotStore;

@ApplicationScoped
@Alternative
@Priority(100)
public class InMemoryQueueSnapshotStore implements QueueSnapshotStore {

    private final List<QueueSnapshot> snapshots = new ArrayList<>();

    @Override
    public QueueSnapshot put(final QueueSnapshot snapshot) {
        if (snapshot.id == null) snapshot.id = UUID.randomUUID();
        snapshots.add(snapshot);
        return snapshot;
    }

    @Override
    public List<QueueSnapshot> findByQueueAndPeriod(
            final UUID queueViewId, final Instant from, final Instant to) {
        return snapshots.stream()
                .filter(s -> s.queueViewId.equals(queueViewId)
                        && !s.snapshotAt.isBefore(from)
                        && !s.snapshotAt.isAfter(to))
                .sorted((a, b) -> a.snapshotAt.compareTo(b.snapshotAt))
                .toList();
    }

    @Override
    public Map<UUID, Instant> findLatestSnapshotTimes(final Collection<UUID> queueViewIds) {
        final Map<UUID, Instant> result = new HashMap<>();
        for (final QueueSnapshot s : snapshots) {
            if (queueViewIds.contains(s.queueViewId)) {
                result.merge(s.queueViewId, s.snapshotAt,
                        (a, b) -> a.isAfter(b) ? a : b);
            }
        }
        return result;
    }

    @Override
    public void deleteOlderThan(final Instant cutoff) {
        snapshots.removeIf(s -> s.snapshotAt.isBefore(cutoff));
    }

    public void clear() {
        snapshots.clear();
    }
}
```

- [ ] **Step 8: Run store tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -Dtest=JpaQueueSnapshotStoreTest`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add queues/src/main/java/io/casehub/work/queues/model/QueueSnapshot.java \
       queues/src/main/java/io/casehub/work/queues/repository/QueueSnapshotStore.java \
       queues/src/main/java/io/casehub/work/queues/repository/jpa/JpaQueueSnapshotStore.java \
       queues/src/main/resources/db/work/migration/V2004__queue_snapshot.sql \
       queues/src/test/java/io/casehub/work/queues/repository/jpa/JpaQueueSnapshotStoreTest.java \
       persistence-memory/src/main/java/io/casehub/work/memory/InMemoryQueueSnapshotStore.java \
       persistence-memory/pom.xml
git commit -m "feat(#289): add QueueSnapshot entity, store, and V2004 migration

JPA entity with FK to queue_view (ON DELETE CASCADE), unique constraint
on (tenancy_id, queue_view_id, snapshot_at). Store SPI with put,
findByQueueAndPeriod, findLatestSnapshotTimes (batch), deleteOlderThan.
Refs #289"
```

---

### Task 3: CrossTenantQueueViewStore, Preferences, and CDI Producer

Adds the cross-tenant infrastructure for tenant discovery and the typed preference keys for snapshot interval and retention.

**Files:**
- Create: `queues/src/main/java/io/casehub/work/queues/repository/CrossTenantQueueViewStore.java`
- Create: `queues/src/main/java/io/casehub/work/queues/repository/jpa/JpaCrossTenantQueueViewStore.java`
- Create: `queues/src/main/java/io/casehub/work/queues/service/QueueCrossTenantProducer.java`
- Create: `queues/src/main/java/io/casehub/work/queues/config/QueueSnapshotInterval.java`
- Create: `queues/src/main/java/io/casehub/work/queues/config/QueueTrendRetention.java`
- Test: `queues/src/test/java/io/casehub/work/queues/config/QueuePreferenceKeyTest.java`

**Interfaces:**
- Consumes: `TenantAwareStore.withCrossTenantQuery()` (existing), `CrossTenant` qualifier (existing in `runtime/repository/`), `PreferenceKey<DurationPreference>` (platform-api), `CrossTenantProducer` pattern (existing in `runtime/service/`)
- Produces: `CrossTenantQueueViewStore.findDistinctTenancyIds(): List<String>` — used by Task 5; `QueueSnapshotInterval.KEY`, `QueueTrendRetention.KEY` — used by Task 5

- [ ] **Step 1: Create CrossTenantQueueViewStore interface**

```java
package io.casehub.work.queues.repository;

import java.util.List;

public interface CrossTenantQueueViewStore {
    List<String> findDistinctTenancyIds();
}
```

- [ ] **Step 2: Create JpaCrossTenantQueueViewStore**

Follow `JpaCrossTenantWorkItemStore` pattern exactly:

```java
package io.casehub.work.queues.repository.jpa;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.casehub.work.queues.model.QueueView;
import io.casehub.work.queues.repository.CrossTenantQueueViewStore;

@ApplicationScoped
public class JpaCrossTenantQueueViewStore extends TenantAwareStore implements CrossTenantQueueViewStore {

    @Override
    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public List<String> findDistinctTenancyIds() {
        return withCrossTenantQuery(() ->
            QueueView.find("SELECT DISTINCT tenancyId FROM QueueView")
                .project(String.class).list());
    }
}
```

Note: check whether `TenantAwareStore` is in `queues/repository/jpa/` or imported from `runtime/repository/jpa/`. The queues module may have its own or reuse the runtime's. Use `ide_find_class` for `TenantAwareStore` in the queues module to verify the import path. If queues has its own, extend that one.

- [ ] **Step 3: Create QueueCrossTenantProducer**

Follow `CrossTenantProducer` in `runtime/service/` exactly:

```java
package io.casehub.work.queues.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.work.runtime.repository.CrossTenant;
import io.casehub.work.runtime.service.WorkSystem;
import io.casehub.work.queues.repository.CrossTenantQueueViewStore;

@ApplicationScoped
public class QueueCrossTenantProducer {

    @Inject
    @WorkSystem
    CurrentPrincipal systemPrincipal;

    @Inject
    CrossTenantQueueViewStore crossTenantQueueViewStore;

    @Produces
    @CrossTenant
    @ApplicationScoped
    public CrossTenantQueueViewStore produceQueueViewStore() {
        if (!systemPrincipal.isCrossTenantAdmin()) {
            throw new IllegalStateException(
                "SystemCurrentPrincipal.isCrossTenantAdmin() must return true");
        }
        return crossTenantQueueViewStore;
    }
}
```

- [ ] **Step 4: Create QueueSnapshotInterval preference key**

```java
package io.casehub.work.queues.config;

import java.time.Duration;
import io.casehub.platform.api.preferences.PreferenceKey;
import io.casehub.platform.api.preferences.DurationPreference;

public final class QueueSnapshotInterval {
    private QueueSnapshotInterval() {}

    public static final PreferenceKey<DurationPreference> KEY =
            new PreferenceKey<>("casehub.work.queues", "snapshot-interval",
                    new DurationPreference(Duration.ofHours(1)),
                    s -> new DurationPreference(Duration.parse(s)));
}
```

- [ ] **Step 5: Create QueueTrendRetention preference key**

```java
package io.casehub.work.queues.config;

import java.time.Duration;
import io.casehub.platform.api.preferences.PreferenceKey;
import io.casehub.platform.api.preferences.DurationPreference;

public final class QueueTrendRetention {
    private QueueTrendRetention() {}

    public static final PreferenceKey<DurationPreference> KEY =
            new PreferenceKey<>("casehub.work.queues", "trend-retention",
                    new DurationPreference(Duration.ofDays(7)),
                    s -> new DurationPreference(Duration.parse(s)));
}
```

- [ ] **Step 6: Write preference key unit test**

```java
package io.casehub.work.queues.config;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Duration;
import org.junit.jupiter.api.Test;

class QueuePreferenceKeyTest {

    @Test
    void snapshotInterval_defaultIsOneHour() {
        assertThat(QueueSnapshotInterval.KEY.defaultValue().duration())
                .isEqualTo(Duration.ofHours(1));
    }

    @Test
    void snapshotInterval_parsesIso8601() {
        final var parsed = QueueSnapshotInterval.KEY.parse("PT30M");
        assertThat(parsed.duration()).isEqualTo(Duration.ofMinutes(30));
    }

    @Test
    void trendRetention_defaultIsSevenDays() {
        assertThat(QueueTrendRetention.KEY.defaultValue().duration())
                .isEqualTo(Duration.ofDays(7));
    }

    @Test
    void trendRetention_parsesIso8601() {
        final var parsed = QueueTrendRetention.KEY.parse("P30D");
        assertThat(parsed.duration()).isEqualTo(Duration.ofDays(30));
    }

    @Test
    void qualifiedNames_followNamespace() {
        assertThat(QueueSnapshotInterval.KEY.qualifiedName())
                .isEqualTo("casehub.work.queues.snapshot-interval");
        assertThat(QueueTrendRetention.KEY.qualifiedName())
                .isEqualTo("casehub.work.queues.trend-retention");
    }
}
```

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -Dtest=QueuePreferenceKeyTest`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add queues/src/main/java/io/casehub/work/queues/repository/CrossTenantQueueViewStore.java \
       queues/src/main/java/io/casehub/work/queues/repository/jpa/JpaCrossTenantQueueViewStore.java \
       queues/src/main/java/io/casehub/work/queues/service/QueueCrossTenantProducer.java \
       queues/src/main/java/io/casehub/work/queues/config/QueueSnapshotInterval.java \
       queues/src/main/java/io/casehub/work/queues/config/QueueTrendRetention.java \
       queues/src/test/java/io/casehub/work/queues/config/QueuePreferenceKeyTest.java
git commit -m "feat(#289): add CrossTenantQueueViewStore, snapshot preferences

CrossTenantQueueViewStore with REQUIRES_NEW transaction and CDI producer.
Two PreferenceKey<DurationPreference> constants: snapshot-interval (1h default),
trend-retention (7d default). Refs #289"
```

---

### Task 4: QueueMembershipService — Extract and Add Count Path

Extracts the membership evaluation logic from `QueueResource.query()` into a shared service and adds the `countMembers()` method for the snapshot job.

**Files:**
- Create: `queues/src/main/java/io/casehub/work/queues/service/QueueMembershipService.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/api/QueueResource.java` — refactor `query()` to use `QueueMembershipService`
- Test: `queues/src/test/java/io/casehub/work/queues/service/QueueMembershipServiceTest.java`

**Interfaces:**
- Consumes: `WorkItemStore.scan(WorkItemQuery)` (existing), `WorkItemStore.countByQuery(WorkItemQuery)` (from Task 1), `FilterEvaluatorRegistry` (existing), `ExpressionDescriptor` (existing)
- Produces: `QueueMembershipService.evaluateMembers(QueueView): List<WorkItem>` — used by refactored `QueueResource.query()`; `QueueMembershipService.countMembers(QueueView): int` — used by Task 5

- [ ] **Step 1: Write failing test for countMembers**

```java
package io.casehub.work.queues.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.queues.model.QueueView;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.repository.WorkItemStore;

class QueueMembershipServiceTest {

    private WorkItemStore workItemStore;
    private FilterEvaluatorRegistry evaluatorRegistry;
    private QueueMembershipService service;

    @BeforeEach
    void setUp() {
        workItemStore = mock(WorkItemStore.class);
        evaluatorRegistry = mock(FilterEvaluatorRegistry.class);
        service = new QueueMembershipService(workItemStore, evaluatorRegistry);
    }

    @Test
    void countMembers_noAdditionalConditions_usesCountQuery() {
        when(workItemStore.countByQuery(any())).thenReturn(5L);
        final QueueView queue = new QueueView();
        queue.labelPattern = "finance/**";

        assertThat(service.countMembers(queue)).isEqualTo(5);
    }

    @Test
    void countMembers_withAdditionalConditions_fallsBackToFullEval() {
        final WorkItem wi1 = new WorkItem();
        final WorkItem wi2 = new WorkItem();
        when(workItemStore.scan(any())).thenReturn(List.of(wi1, wi2));
        final var jexl = mock(WorkItemExpressionEvaluator.class);
        when(evaluatorRegistry.find("jexl")).thenReturn(jexl);
        when(jexl.evaluate(any(), any())).thenReturn(true);

        final QueueView queue = new QueueView();
        queue.labelPattern = "finance/**";
        queue.additionalConditions = "item.priority == 'HIGH'";

        assertThat(service.countMembers(queue)).isEqualTo(2);
    }

    @Test
    void evaluateMembers_returnsFilteredWorkItems() {
        final WorkItem wi1 = new WorkItem();
        when(workItemStore.scan(any())).thenReturn(List.of(wi1));

        final QueueView queue = new QueueView();
        queue.labelPattern = "legal/**";

        assertThat(service.evaluateMembers(queue)).containsExactly(wi1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -Dtest=QueueMembershipServiceTest`
Expected: compilation failure — `QueueMembershipService` does not exist

- [ ] **Step 3: Create QueueMembershipService**

Follows the spec's `QueueMembershipService` definition:

```java
package io.casehub.work.queues.service;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.work.queues.model.QueueView;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.repository.WorkItemQuery;
import io.casehub.work.runtime.repository.WorkItemStore;

@ApplicationScoped
public class QueueMembershipService {

    private final WorkItemStore workItemStore;
    private final FilterEvaluatorRegistry evaluatorRegistry;

    @Inject
    public QueueMembershipService(
            final WorkItemStore workItemStore,
            final FilterEvaluatorRegistry evaluatorRegistry) {
        this.workItemStore = workItemStore;
        this.evaluatorRegistry = evaluatorRegistry;
    }

    public List<WorkItem> evaluateMembers(final QueueView queue) {
        var candidates = workItemStore.scan(
                WorkItemQuery.byLabelPattern(queue.labelPattern));
        if (queue.additionalConditions != null
                && !queue.additionalConditions.isBlank()) {
            final var jexl = evaluatorRegistry.find("jexl");
            if (jexl != null) {
                candidates = candidates.stream()
                        .filter(wi -> jexl.evaluate(wi,
                                ExpressionDescriptor.of("jexl",
                                        queue.additionalConditions)))
                        .toList();
            }
        }
        return candidates;
    }

    public int countMembers(final QueueView queue) {
        if (queue.additionalConditions == null
                || queue.additionalConditions.isBlank()) {
            return (int) workItemStore.countByQuery(
                    WorkItemQuery.byLabelPattern(queue.labelPattern));
        }
        return evaluateMembers(queue).size();
    }
}
```

- [ ] **Step 4: Refactor QueueResource.query() to use QueueMembershipService**

In `QueueResource.java`, inject `QueueMembershipService` and replace the inline membership evaluation in `query()`:

```java
@Inject
QueueMembershipService membershipService;
```

Replace lines 132-142 in `query()` with:

```java
final var candidates = membershipService.evaluateMembers(q);
```

Remove the `FilterEvaluatorRegistry` injection from `QueueResource` if it's no longer used directly (check other methods first).

- [ ] **Step 5: Run existing queue tests to verify refactoring is safe**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues`
Expected: all existing tests PASS (refactoring preserved behavior)

- [ ] **Step 6: Run new unit tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -Dtest=QueueMembershipServiceTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add queues/src/main/java/io/casehub/work/queues/service/QueueMembershipService.java \
       queues/src/main/java/io/casehub/work/queues/api/QueueResource.java \
       queues/src/test/java/io/casehub/work/queues/service/QueueMembershipServiceTest.java
git commit -m "refactor(#289): extract QueueMembershipService from QueueResource

Shared evaluateMembers() and countMembers() — prevents drift between
REST endpoint and snapshot job. countMembers uses countByQuery fast path
when no additionalConditions. Refs #289"
```

---

### Task 5: QueueSnapshotJob — Scheduled Snapshot and Cleanup

The main job that takes periodic snapshots and prunes old data. Depends on Tasks 2, 3, and 4.

**Files:**
- Create: `queues/src/main/java/io/casehub/work/queues/service/QueueSnapshotJob.java`
- Test: `queues/src/test/java/io/casehub/work/queues/service/QueueSnapshotJobTest.java`

**Interfaces:**
- Consumes: `CrossTenantQueueViewStore.findDistinctTenancyIds()` (Task 3), `QueueViewStore.scanAll()` (existing), `QueueSnapshotStore.put/findLatestSnapshotTimes/deleteOlderThan` (Task 2), `QueueMembershipService.countMembers()` (Task 4), `QueueSnapshotInterval.KEY` / `QueueTrendRetention.KEY` (Task 3), `TenantContextRunner` (existing), `PreferenceProvider` (platform-api)
- Produces: periodic `QueueSnapshot` rows in the database — consumed by Task 6 endpoint

- [ ] **Step 1: Write failing unit test — snapshot persisted when interval elapsed**

```java
package io.casehub.work.queues.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.preferences.DurationPreference;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.work.queues.config.QueueSnapshotInterval;
import io.casehub.work.queues.config.QueueTrendRetention;
import io.casehub.work.queues.model.QueueSnapshot;
import io.casehub.work.queues.model.QueueView;
import io.casehub.work.queues.repository.CrossTenantQueueViewStore;
import io.casehub.work.queues.repository.QueueSnapshotStore;
import io.casehub.work.queues.repository.QueueViewStore;
import io.casehub.work.runtime.service.TenantContextRunner;

class QueueSnapshotJobTest {

    private CrossTenantQueueViewStore crossTenantStore;
    private QueueViewStore queueViewStore;
    private QueueSnapshotStore snapshotStore;
    private QueueMembershipService membershipService;
    private PreferenceProvider preferenceProvider;
    private TenantContextRunner tenantContextRunner;
    private QueueSnapshotJob job;

    @BeforeEach
    void setUp() {
        crossTenantStore = mock(CrossTenantQueueViewStore.class);
        queueViewStore = mock(QueueViewStore.class);
        snapshotStore = mock(QueueSnapshotStore.class);
        membershipService = mock(QueueMembershipService.class);
        preferenceProvider = mock(PreferenceProvider.class);
        tenantContextRunner = mock(TenantContextRunner.class);

        // TenantContextRunner just runs the runnable
        doAnswer(inv -> {
            ((Runnable) inv.getArgument(1)).run();
            return null;
        }).when(tenantContextRunner).runInTenantContext(any(), any());

        // Default preferences
        when(preferenceProvider.get(QueueSnapshotInterval.KEY))
                .thenReturn(new DurationPreference(Duration.ofHours(1)));
        when(preferenceProvider.get(QueueTrendRetention.KEY))
                .thenReturn(new DurationPreference(Duration.ofDays(7)));

        job = new QueueSnapshotJob(crossTenantStore, queueViewStore,
                snapshotStore, membershipService, preferenceProvider,
                tenantContextRunner);
    }

    @Test
    void tick_snapshotsQueue_whenIntervalElapsed() {
        final QueueView queue = testQueue();
        when(crossTenantStore.findDistinctTenancyIds()).thenReturn(List.of("t1"));
        when(queueViewStore.scanAll()).thenReturn(List.of(queue));
        when(snapshotStore.findLatestSnapshotTimes(any())).thenReturn(Map.of());
        when(membershipService.countMembers(queue)).thenReturn(42);

        job.tick();

        verify(snapshotStore).put(argThat(s ->
                s.queueViewId.equals(queue.id) && s.memberCount == 42));
    }

    @Test
    void tick_skipsQueue_whenWithinInterval() {
        final QueueView queue = testQueue();
        when(crossTenantStore.findDistinctTenancyIds()).thenReturn(List.of("t1"));
        when(queueViewStore.scanAll()).thenReturn(List.of(queue));
        when(snapshotStore.findLatestSnapshotTimes(any()))
                .thenReturn(Map.of(queue.id, Instant.now()));

        job.tick();

        verify(snapshotStore, never()).put(any());
    }

    @Test
    void tick_callsRetentionCleanup() {
        when(crossTenantStore.findDistinctTenancyIds()).thenReturn(List.of("t1"));
        when(queueViewStore.scanAll()).thenReturn(List.of());

        job.tick();

        verify(snapshotStore).deleteOlderThan(any());
    }

    @Test
    void tick_continuesAfterFailure() {
        final QueueView q1 = testQueue();
        final QueueView q2 = testQueue();
        when(crossTenantStore.findDistinctTenancyIds()).thenReturn(List.of("t1"));
        when(queueViewStore.scanAll()).thenReturn(List.of(q1, q2));
        when(snapshotStore.findLatestSnapshotTimes(any())).thenReturn(Map.of());
        when(membershipService.countMembers(q1))
                .thenThrow(new RuntimeException("boom"));
        when(membershipService.countMembers(q2)).thenReturn(10);

        job.tick();

        verify(snapshotStore).put(argThat(s -> s.queueViewId.equals(q2.id)));
    }

    private QueueView testQueue() {
        final QueueView q = new QueueView();
        q.id = UUID.randomUUID();
        q.name = "test-queue-" + q.id;
        q.labelPattern = "test/**";
        return q;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -Dtest=QueueSnapshotJobTest`
Expected: compilation failure — `QueueSnapshotJob` does not exist

- [ ] **Step 3: Implement QueueSnapshotJob**

```java
package io.casehub.work.queues.service;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.jboss.logging.Logger;

import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.work.queues.config.QueueSnapshotInterval;
import io.casehub.work.queues.config.QueueTrendRetention;
import io.casehub.work.queues.model.QueueSnapshot;
import io.casehub.work.queues.model.QueueView;
import io.casehub.work.queues.repository.CrossTenantQueueViewStore;
import io.casehub.work.queues.repository.QueueSnapshotStore;
import io.casehub.work.queues.repository.QueueViewStore;
import io.casehub.work.runtime.service.TenantContextRunner;
import io.quarkus.scheduler.Scheduled;

@ApplicationScoped
public class QueueSnapshotJob {

    private static final Logger LOG = Logger.getLogger(QueueSnapshotJob.class);

    private final CrossTenantQueueViewStore crossTenantStore;
    private final QueueViewStore queueViewStore;
    private final QueueSnapshotStore snapshotStore;
    private final QueueMembershipService membershipService;
    private final PreferenceProvider preferenceProvider;
    private final TenantContextRunner tenantContextRunner;

    @Inject
    public QueueSnapshotJob(
            final CrossTenantQueueViewStore crossTenantStore,
            final QueueViewStore queueViewStore,
            final QueueSnapshotStore snapshotStore,
            final QueueMembershipService membershipService,
            final PreferenceProvider preferenceProvider,
            final TenantContextRunner tenantContextRunner) {
        this.crossTenantStore = crossTenantStore;
        this.queueViewStore = queueViewStore;
        this.snapshotStore = snapshotStore;
        this.membershipService = membershipService;
        this.preferenceProvider = preferenceProvider;
        this.tenantContextRunner = tenantContextRunner;
    }

    @Scheduled(identity = "queue-snapshot-heartbeat", every = "5m", delayed = "30s")
    @Transactional(Transactional.TxType.NOT_SUPPORTED)
    public void tick() {
        final List<String> tenantIds = crossTenantStore.findDistinctTenancyIds();
        for (final String tenancyId : tenantIds) {
            tenantContextRunner.runInTenantContext(tenancyId, () ->
                    processForTenant(tenancyId));
        }
    }

    private void processForTenant(final String tenancyId) {
        try {
            final Duration interval = preferenceProvider
                    .get(QueueSnapshotInterval.KEY).duration();
            final Duration retention = preferenceProvider
                    .get(QueueTrendRetention.KEY).duration();

            final List<QueueView> queues = queueViewStore.scanAll();
            if (queues.isEmpty()) return;

            final List<UUID> queueIds = queues.stream()
                    .map(q -> q.id).toList();
            final Map<UUID, Instant> latestTimes =
                    snapshotStore.findLatestSnapshotTimes(queueIds);
            final Instant now = Instant.now();

            for (final QueueView queue : queues) {
                final Instant lastSnapshot = latestTimes.get(queue.id);
                if (lastSnapshot != null
                        && Duration.between(lastSnapshot, now).compareTo(interval) < 0) {
                    continue;
                }
                snapshotQueue(queue, now);
            }

            snapshotStore.deleteOlderThan(now.minus(retention));
        } catch (final Exception e) {
            LOG.warnf("Queue snapshot tick failed for tenant %s: %s",
                    tenancyId, e.getMessage());
        }
    }

    private void snapshotQueue(final QueueView queue, final Instant now) {
        try {
            final int count = membershipService.countMembers(queue);
            final QueueSnapshot snapshot = new QueueSnapshot();
            snapshot.queueViewId = queue.id;
            snapshot.memberCount = count;
            snapshot.snapshotAt = now;
            snapshotStore.put(snapshot);
        } catch (final Exception e) {
            LOG.warnf("Snapshot failed for queue %s (%s): %s",
                    queue.id, queue.name, e.getMessage());
        }
    }
}
```

Note: the spec says `prepareSnapshot` and `snapshotQueue` each run in `REQUIRES_NEW`. For the unit-testable version above, the `@Transactional` annotations are on the store/service methods, not on the job's private methods. In the JPA integration path, the stores' own transactions provide the boundaries. Verify this works — if the stores don't have their own `@Transactional`, add `@Transactional(REQUIRES_NEW)` annotations to the relevant store methods (`put`, `deleteOlderThan`).

- [ ] **Step 4: Run unit tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -Dtest=QueueSnapshotJobTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add queues/src/main/java/io/casehub/work/queues/service/QueueSnapshotJob.java \
       queues/src/test/java/io/casehub/work/queues/service/QueueSnapshotJobTest.java
git commit -m "feat(#289): add QueueSnapshotJob — periodic queue membership snapshots

@Scheduled heartbeat every 5m, snapshots when preference interval elapsed.
Per-tenant processing via TenantContextRunner. Retention cleanup per-tenant.
Individual queue failures do not stop other snapshots. Refs #289"
```

---

### Task 6: REST Endpoint — GET /queues/{id}/trend

Adds the trend endpoint to `QueueResource` and the response DTO.

**Files:**
- Create: `queues/src/main/java/io/casehub/work/queues/api/QueueTrendResponse.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/api/QueueResource.java` — add trend endpoint
- Test: `queues/src/test/java/io/casehub/work/queues/api/QueueTrendIntegrationTest.java`
- Modify: `docs/api-reference.md` — document new endpoint

**Interfaces:**
- Consumes: `QueueViewStore.get(UUID)` (existing), `QueueSnapshotStore.findByQueueAndPeriod()` (Task 2)
- Produces: `GET /queues/{id}/trend?period=24h` → `QueueTrendResponse` JSON

- [ ] **Step 1: Create QueueTrendResponse DTO**

```java
package io.casehub.work.queues.api;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record QueueTrendResponse(
        UUID queueViewId,
        String queueName,
        String period,
        List<DataPoint> dataPoints) {

    public record DataPoint(Instant snapshotAt, long memberCount) {}
}
```

- [ ] **Step 2: Write failing integration test**

```java
package io.casehub.work.queues.api;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import io.casehub.work.queues.model.QueueSnapshot;
import io.casehub.work.queues.model.QueueView;
import io.casehub.work.queues.repository.QueueSnapshotStore;
import io.casehub.work.queues.repository.QueueViewStore;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

@QuarkusTest
class QueueTrendIntegrationTest {

    @Inject
    QueueViewStore queueViewStore;

    @Inject
    QueueSnapshotStore snapshotStore;

    @Test
    @Transactional
    void trend_returnsSnapshotDataPoints() {
        final QueueView q = new QueueView();
        q.name = "trend-test";
        q.labelPattern = "trend/**";
        q.scope = io.casehub.platform.api.path.Path.root();
        q.sortField = "createdAt";
        q.sortDirection = "ASC";
        queueViewStore.put(q);

        final Instant now = Instant.now();
        putSnapshot(q.id, 10, now.minusSeconds(3600));
        putSnapshot(q.id, 15, now);

        final var response = given()
                .when().get("/queues/" + q.id + "/trend?period=24h")
                .then().statusCode(200)
                .extract().as(QueueTrendResponse.class);

        assertThat(response.queueViewId()).isEqualTo(q.id);
        assertThat(response.queueName()).isEqualTo("trend-test");
        assertThat(response.dataPoints()).hasSize(2);
        assertThat(response.dataPoints().get(0).memberCount()).isEqualTo(10);
        assertThat(response.dataPoints().get(1).memberCount()).isEqualTo(15);
    }

    @Test
    void trend_returns404_forUnknownQueue() {
        given()
                .when().get("/queues/" + UUID.randomUUID() + "/trend")
                .then().statusCode(404);
    }

    @Test
    @Transactional
    void trend_returnsEmptyDataPoints_whenNoSnapshots() {
        final QueueView q = new QueueView();
        q.name = "empty-trend";
        q.labelPattern = "empty/**";
        q.scope = io.casehub.platform.api.path.Path.root();
        q.sortField = "createdAt";
        q.sortDirection = "ASC";
        queueViewStore.put(q);

        final var response = given()
                .when().get("/queues/" + q.id + "/trend?period=24h")
                .then().statusCode(200)
                .extract().as(QueueTrendResponse.class);

        assertThat(response.dataPoints()).isEmpty();
    }

    @Transactional
    void putSnapshot(final UUID queueId, final long count, final Instant at) {
        final QueueSnapshot s = new QueueSnapshot();
        s.queueViewId = queueId;
        s.memberCount = count;
        s.snapshotAt = at;
        snapshotStore.put(s);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -Dtest=QueueTrendIntegrationTest`
Expected: 404 on `/queues/{id}/trend` — endpoint does not exist yet

- [ ] **Step 4: Add trend endpoint to QueueResource**

Add to `QueueResource.java`:

```java
@Inject
QueueSnapshotStore snapshotStore;

@GET
@Path("/{id}/trend")
@Transactional
public Response trend(@PathParam("id") final UUID id,
                      @jakarta.ws.rs.QueryParam("period") final String periodParam) {
    final QueueView q = queueViewStore.get(id).orElse(null);
    if (q == null) {
        return Response.status(404)
                .entity(Map.of("error", "Queue view not found")).build();
    }

    final java.time.Duration period = parsePeriod(
            periodParam != null ? periodParam : "24h");
    final Instant now = Instant.now();
    final Instant from = now.minus(period);

    final var snapshots = snapshotStore.findByQueueAndPeriod(q.id, from, now);
    final var dataPoints = snapshots.stream()
            .map(s -> new QueueTrendResponse.DataPoint(s.snapshotAt, s.memberCount))
            .toList();

    return Response.ok(new QueueTrendResponse(
            q.id, q.name, period.toString(), dataPoints)).build();
}

private static java.time.Duration parsePeriod(final String input) {
    if (input.startsWith("P") || input.startsWith("p")) {
        return java.time.Duration.parse(input);
    }
    final String normalized = input.toLowerCase();
    if (normalized.endsWith("h")) {
        return java.time.Duration.ofHours(
                Long.parseLong(normalized.substring(0, normalized.length() - 1)));
    }
    if (normalized.endsWith("d")) {
        return java.time.Duration.ofDays(
                Long.parseLong(normalized.substring(0, normalized.length() - 1)));
    }
    return java.time.Duration.parse("PT" + input);
}
```

- [ ] **Step 5: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues -Dtest=QueueTrendIntegrationTest`
Expected: PASS

- [ ] **Step 6: Run all queue tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queues`
Expected: all PASS

- [ ] **Step 7: Update api-reference.md**

Add after the `GET /queues/{id}/events` section (around line 1170):

```markdown
### GET /queues/{id}/trend

Historical queue membership trend data for sparkline visualisation.

**Path parameter:** `id` — UUID

**Query parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `period` | Duration string | `24h` | How far back (`1h`, `24h`, `7d`, `30d`). Accepts ISO-8601 (`PT24H`, `P7D`) and shorthand. |

**Response:** `200 OK`
**Body:** `QueueTrendResponse`

| Field | Type | Description |
|---|---|---|
| `queueViewId` | UUID | |
| `queueName` | string | |
| `period` | string | ISO-8601 duration |
| `dataPoints` | DataPoint[] | Ordered by `snapshotAt` ascending |

DataPoint: `snapshotAt` (instant), `memberCount` (long)

**Error:** `404` — queue not found

---
```

- [ ] **Step 8: Commit**

```bash
git add queues/src/main/java/io/casehub/work/queues/api/QueueTrendResponse.java \
       queues/src/main/java/io/casehub/work/queues/api/QueueResource.java \
       queues/src/test/java/io/casehub/work/queues/api/QueueTrendIntegrationTest.java \
       docs/api-reference.md
git commit -m "feat(#289): add GET /queues/{id}/trend endpoint

Returns historical queue membership data points for sparkline rendering.
Period parameter with shorthand (24h, 7d) and ISO-8601 (PT24H, P7D).
Empty dataPoints when no snapshots exist yet. Closes #289"
```
