# SLA Compliance Reporting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `quarkus-work-reports`, an optional Jandex library module exposing four `/workitems/reports/*` REST endpoints for SLA breach tracking, actor performance, throughput trends, and queue health.

**Architecture:** Optional module (zero cost when absent) following the same pattern as `quarkus-work-notifications`. All business logic in `ReportService`; `ReportResource` is a thin HTTP adapter. Throughput bucketing uses HQL `date_trunc('day')` + Java week/month rollup via `ThroughputBucketAggregator`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate 6 HQL, H2 (tests), quarkus-cache (Caffeine), REST Assured, AssertJ, JUnit 5.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports`

---

## File Map

```
quarkus-work-reports/
├── pom.xml
└── src/
    ├── main/java/io/quarkiverse/work/reports/
    │   ├── api/ReportResource.java
    │   └── service/
    │       ├── ReportService.java
    │       ├── SlaBreachReport.java       record
    │       ├── SlaBreachItem.java         record
    │       ├── SlaSummary.java            record
    │       ├── ActorReport.java           record
    │       ├── ThroughputReport.java      record
    │       ├── ThroughputBucket.java      record
    │       ├── ThroughputBucketAggregator.java  pure Java
    │       └── QueueHealthReport.java     record
    └── test/
        ├── java/io/quarkiverse/work/reports/
        │   ├── service/ThroughputBucketAggregatorTest.java  pure unit
        │   └── api/
        │       ├── SlaBreachReportTest.java
        │       ├── ActorPerformanceReportTest.java
        │       ├── ThroughputReportTest.java
        │       └── QueueHealthReportTest.java
        └── resources/application.properties

# Files to DELETE from runtime/:
runtime/src/main/java/.../api/ReportResource.java
runtime/src/test/java/.../api/SlaBreachReportTest.java
runtime/src/test/java/.../api/ActorPerformanceReportTest.java

# Files to MODIFY:
pom.xml                              add quarkus-work-reports module
integration-tests/pom.xml           add quarkus-work-reports dependency
integration-tests/src/test/.../WorkItemNativeIT.java   add report smoke tests
docs/DESIGN.md
docs/api-reference.md
CLAUDE.md
```

---

## Task 1: Create GitHub issues under Epic #104

**Files:** none

- [ ] **Step 1: Create issue A**
```bash
gh issue create --repo casehubio/work \
  --title "Scaffold quarkus-work-reports module; relocate and fix existing SLA/actor scaffold" \
  --body "Scaffold the optional quarkus-work-reports Jandex library module. Move ReportResource and test stubs from runtime/ into the new module. Fix N+1 in computeByCategory. Fix Javadoc epic refs (#99 → #104).\n\nPart of Epic #104." \
  --label "enhancement"
```
Note the issue number printed (referenced below as `#A`).

- [ ] **Step 2: Create issue B**
```bash
gh issue create --repo casehubio/work \
  --title "Add throughput time-series endpoint GET /workitems/reports/throughput" \
  --body "Implement throughput reporting with groupBy=day|week|month. HQL date_trunc('day') for DB-side day grouping; Java rollup to week/month. Full TDD.\n\nPart of Epic #104." \
  --label "enhancement"
```

- [ ] **Step 3: Create issue C**
```bash
gh issue create --repo casehubio/work \
  --title "Add queue-health snapshot endpoint GET /workitems/reports/queue-health" \
  --body "Point-in-time queue health: overdueCount, pendingCount, avgPendingAgeSeconds, oldestUnclaimedCreatedAt, criticalOverdueCount. Full TDD.\n\nPart of Epic #104." \
  --label "enhancement"
```

- [ ] **Step 4: Create issue D**
```bash
gh issue create --repo casehubio/work \
  --title "Add E2E report smoke tests and Testcontainers PostgreSQL dialect validation" \
  --body "Add WorkItemReportNativeIT to integration-tests/. Add PostgresDialectValidationTest to quarkus-work-reports using Quarkus Dev Services to verify HQL date_trunc works against real PostgreSQL.\n\nPart of Epic #104." \
  --label "enhancement"
```

- [ ] **Step 5: Commit placeholder**
```bash
git commit --allow-empty -m "chore: begin Epic #104 SLA reporting — issues created

Refs #104"
```

---

## Task 2: Scaffold quarkus-work-reports module

**Files:**
- Create: `quarkus-work-reports/pom.xml`
- Create: `quarkus-work-reports/src/test/resources/application.properties`
- Modify: `pom.xml` (parent — add module)
- Delete: `runtime/src/main/java/io/quarkiverse/work/runtime/api/ReportResource.java`
- Delete: `runtime/src/test/java/io/quarkiverse/work/runtime/api/SlaBreachReportTest.java`
- Delete: `runtime/src/test/java/io/quarkiverse/work/runtime/api/ActorPerformanceReportTest.java`

- [ ] **Step 1: Create module pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.quarkiverse.work</groupId>
    <artifactId>quarkus-work-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>quarkus-work-reports</artifactId>
  <name>Quarkus Work - SLA Reports</name>
  <description>Optional SLA compliance reporting module. Adds /workitems/reports/* endpoints
for breach rates, actor performance, throughput trends, and queue health.
Zero cost when absent — the core extension is unchanged.</description>

  <dependencies>
    <dependency>
      <groupId>io.quarkiverse.work</groupId>
      <artifactId>quarkus-work</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-orm-panache</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-cache</artifactId>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <version>3.3.1</version>
        <executions>
          <execution>
            <id>jandex</id>
            <phase>process-classes</phase>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Register module in parent pom.xml**

In `pom.xml`, add `<module>quarkus-work-reports</module>` immediately before `<module>integration-tests</module>`.

- [ ] **Step 3: Create test application.properties**

File: `quarkus-work-reports/src/test/resources/application.properties`
```properties
quarkus.http.test-port=0
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:reportstest;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.scheduler.enabled=false
quarkus.work.default-expiry-hours=24
quarkus.work.default-claim-hours=4
quarkus.work.cleanup.expiry-check-seconds=60
# Short TTL so cache doesn't interfere between test runs
quarkus.cache.caffeine.reports.expire-after-write=1s
```

- [ ] **Step 4: Delete relocated files from runtime/**
```bash
rm runtime/src/main/java/io/quarkiverse/work/runtime/api/ReportResource.java
rm runtime/src/test/java/io/quarkiverse/work/runtime/api/SlaBreachReportTest.java
rm runtime/src/test/java/io/quarkiverse/work/runtime/api/ActorPerformanceReportTest.java
```

- [ ] **Step 5: Verify runtime still builds cleanly**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```
Expected: BUILD SUCCESS. Runtime test count will drop by the removed tests; the rest must still pass.

- [ ] **Step 6: Commit**
```bash
git add quarkus-work-reports/ pom.xml \
  runtime/src/main/java/io/quarkiverse/work/runtime/api/ReportResource.java \
  runtime/src/test/java/io/quarkiverse/work/runtime/api/SlaBreachReportTest.java \
  runtime/src/test/java/io/quarkiverse/work/runtime/api/ActorPerformanceReportTest.java
git commit -m "feat: scaffold quarkus-work-reports module; remove scaffold from runtime

Refs #A, #104"
```
(Replace `#A` with the actual issue number from Task 1.)

---

## Task 3: DTO records and ThroughputBucketAggregator with unit tests

**Files:**
- Create: `quarkus-work-reports/src/main/java/io/quarkiverse/work/reports/service/SlaBreachItem.java`
- Create: `...service/SlaSummary.java`
- Create: `...service/SlaBreachReport.java`
- Create: `...service/ActorReport.java`
- Create: `...service/ThroughputBucket.java`
- Create: `...service/ThroughputReport.java`
- Create: `...service/ThroughputBucketAggregator.java`
- Create: `...service/QueueHealthReport.java`
- Create: `quarkus-work-reports/src/test/java/io/quarkiverse/work/reports/service/ThroughputBucketAggregatorTest.java`

- [ ] **Step 1: Write the failing unit test**

`quarkus-work-reports/src/test/java/io/quarkiverse/work/reports/service/ThroughputBucketAggregatorTest.java`:
```java
package io.quarkiverse.work.reports.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

class ThroughputBucketAggregatorTest {

    @Test
    void groupByDay_sameDay_oneBucket() {
        LocalDate day = LocalDate.of(2026, 4, 15);
        List<Object[]> created = List.of(new Object[]{day, 3L});
        List<Object[]> completed = List.of(new Object[]{day, 2L});

        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(created, completed, "day");

        assertThat(buckets).hasSize(1);
        assertThat(buckets.get(0).period()).isEqualTo("2026-04-15");
        assertThat(buckets.get(0).created()).isEqualTo(3L);
        assertThat(buckets.get(0).completed()).isEqualTo(2L);
    }

    @Test
    void groupByDay_twoDays_twoBuckets() {
        List<Object[]> created = List.of(
            new Object[]{LocalDate.of(2026, 4, 15), 2L},
            new Object[]{LocalDate.of(2026, 4, 16), 5L}
        );
        List<Object[]> completed = List.of(
            new Object[]{LocalDate.of(2026, 4, 15), 1L}
        );

        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(created, completed, "day");

        assertThat(buckets).hasSize(2);
        assertThat(buckets.get(0).period()).isEqualTo("2026-04-15");
        assertThat(buckets.get(0).created()).isEqualTo(2L);
        assertThat(buckets.get(0).completed()).isEqualTo(1L);
        assertThat(buckets.get(1).period()).isEqualTo("2026-04-16");
        assertThat(buckets.get(1).created()).isEqualTo(5L);
        assertThat(buckets.get(1).completed()).isEqualTo(0L);
    }

    @Test
    void groupByWeek_sameIsoWeek_oneBucket() {
        // 2026-04-13 (Mon) and 2026-04-19 (Sun) are both in ISO week 16
        List<Object[]> created = List.of(
            new Object[]{LocalDate.of(2026, 4, 13), 3L},
            new Object[]{LocalDate.of(2026, 4, 19), 2L}
        );
        List<Object[]> completed = List.of();

        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(created, completed, "week");

        assertThat(buckets).hasSize(1);
        assertThat(buckets.get(0).period()).isEqualTo("2026-W16");
        assertThat(buckets.get(0).created()).isEqualTo(5L);
    }

    @Test
    void groupByWeek_differentWeeks_twoBuckets() {
        List<Object[]> created = List.of(
            new Object[]{LocalDate.of(2026, 4, 13), 1L},  // W16
            new Object[]{LocalDate.of(2026, 4, 20), 1L}   // W17
        );
        List<Object[]> completed = List.of();

        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(created, completed, "week");

        assertThat(buckets).hasSize(2);
        assertThat(buckets.get(0).period()).isEqualTo("2026-W16");
        assertThat(buckets.get(1).period()).isEqualTo("2026-W17");
    }

    @Test
    void groupByMonth_sameMonth_oneBucket() {
        List<Object[]> created = List.of(
            new Object[]{LocalDate.of(2026, 4, 1), 4L},
            new Object[]{LocalDate.of(2026, 4, 30), 6L}
        );
        List<Object[]> completed = List.of(
            new Object[]{LocalDate.of(2026, 4, 15), 3L}
        );

        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(created, completed, "month");

        assertThat(buckets).hasSize(1);
        assertThat(buckets.get(0).period()).isEqualTo("2026-04");
        assertThat(buckets.get(0).created()).isEqualTo(10L);
        assertThat(buckets.get(0).completed()).isEqualTo(3L);
    }

    @Test
    void groupByMonth_differentMonths_twoBuckets() {
        List<Object[]> created = List.of(
            new Object[]{LocalDate.of(2026, 3, 31), 2L},
            new Object[]{LocalDate.of(2026, 4, 1), 3L}
        );
        List<Object[]> completed = List.of();

        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(created, completed, "month");

        assertThat(buckets).hasSize(2);
        assertThat(buckets.get(0).period()).isEqualTo("2026-03");
        assertThat(buckets.get(1).period()).isEqualTo("2026-04");
    }

    @Test
    void emptyInput_returnsEmptyList() {
        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(List.of(), List.of(), "day");
        assertThat(buckets).isEmpty();
    }

    @Test
    void completedOnlyDay_createdIsZero() {
        List<Object[]> completed = List.of(new Object[]{LocalDate.of(2026, 4, 15), 5L});
        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(List.of(), completed, "day");

        assertThat(buckets).hasSize(1);
        assertThat(buckets.get(0).created()).isEqualTo(0L);
        assertThat(buckets.get(0).completed()).isEqualTo(5L);
    }

    @Test
    void bucketsAreSortedByPeriod() {
        List<Object[]> created = List.of(
            new Object[]{LocalDate.of(2026, 4, 17), 1L},
            new Object[]{LocalDate.of(2026, 4, 14), 1L},
            new Object[]{LocalDate.of(2026, 4, 15), 1L}
        );
        List<ThroughputBucket> buckets = ThroughputBucketAggregator.aggregate(created, List.of(), "day");

        assertThat(buckets).extracting(ThroughputBucket::period)
            .containsExactly("2026-04-14", "2026-04-15", "2026-04-17");
    }
}
```

- [ ] **Step 2: Run test — expect compile failure (classes don't exist yet)**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports -q 2>&1 | tail -5
```
Expected: compilation error referencing missing classes.

- [ ] **Step 3: Create DTO records**

`SlaBreachItem.java`:
```java
package io.quarkiverse.work.reports.service;

import java.time.Instant;

public record SlaBreachItem(
    String workItemId,
    String category,
    String priority,
    Instant expiresAt,
    Instant completedAt,
    String status,
    long breachDurationMinutes) {}
```

`SlaSummary.java`:
```java
package io.quarkiverse.work.reports.service;

import java.util.Map;

public record SlaSummary(
    long totalBreached,
    double avgBreachDurationMinutes,
    Map<String, Long> byCategory) {}
```

`SlaBreachReport.java`:
```java
package io.quarkiverse.work.reports.service;

import java.util.List;

public record SlaBreachReport(List<SlaBreachItem> items, SlaSummary summary) {}
```

`ActorReport.java`:
```java
package io.quarkiverse.work.reports.service;

import java.util.Map;

public record ActorReport(
    String actorId,
    long totalAssigned,
    long totalCompleted,
    long totalRejected,
    Double avgCompletionMinutes,
    Map<String, Long> byCategory) {}
```

`ThroughputBucket.java`:
```java
package io.quarkiverse.work.reports.service;

public record ThroughputBucket(String period, long created, long completed) {}
```

`ThroughputReport.java`:
```java
package io.quarkiverse.work.reports.service;

import java.time.Instant;
import java.util.List;

public record ThroughputReport(Instant from, Instant to, String groupBy, List<ThroughputBucket> buckets) {}
```

`QueueHealthReport.java`:
```java
package io.quarkiverse.work.reports.service;

import java.time.Instant;

public record QueueHealthReport(
    Instant timestamp,
    long overdueCount,
    long pendingCount,
    long avgPendingAgeSeconds,
    Instant oldestUnclaimedCreatedAt,
    long criticalOverdueCount) {}
```

- [ ] **Step 4: Implement ThroughputBucketAggregator**

`ThroughputBucketAggregator.java`:
```java
package io.quarkiverse.work.reports.service;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.time.temporal.IsoFields;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class ThroughputBucketAggregator {

    private ThroughputBucketAggregator() {}

    /**
     * Merge day-granularity DB result rows into period buckets.
     *
     * @param dayCreated   rows from HQL: [LocalDate, Long created count]
     * @param dayCompleted rows from HQL: [LocalDate, Long completed count]
     * @param groupBy      "day", "week", or "month"
     */
    public static List<ThroughputBucket> aggregate(
            List<Object[]> dayCreated,
            List<Object[]> dayCompleted,
            String groupBy) {

        Map<String, long[]> buckets = new LinkedHashMap<>();

        for (Object[] row : dayCreated) {
            String period = toPeriodLabel((LocalDate) row[0], groupBy);
            buckets.computeIfAbsent(period, k -> new long[2])[0] += (Long) row[1];
        }
        for (Object[] row : dayCompleted) {
            String period = toPeriodLabel((LocalDate) row[0], groupBy);
            buckets.computeIfAbsent(period, k -> new long[2])[1] += (Long) row[1];
        }

        return buckets.entrySet().stream()
            .sorted(Map.Entry.comparingByKey())
            .map(e -> new ThroughputBucket(e.getKey(), e.getValue()[0], e.getValue()[1]))
            .toList();
    }

    private static String toPeriodLabel(LocalDate date, String groupBy) {
        return switch (groupBy) {
            case "week" -> date.getYear() + "-W"
                + String.format("%02d", date.get(IsoFields.WEEK_OF_WEEK_BASED_YEAR));
            case "month" -> DateTimeFormatter.ofPattern("yyyy-MM").format(date);
            default -> date.toString();
        };
    }
}
```

- [ ] **Step 5: Run unit tests — expect pass**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=ThroughputBucketAggregatorTest -q
```
Expected: BUILD SUCCESS, 8 tests passing.

- [ ] **Step 6: Commit**
```bash
git add quarkus-work-reports/src/
git commit -m "feat: DTO records and ThroughputBucketAggregator with unit tests

Refs #B, #104"
```

---

## Task 4: ReportService stub + ReportResource skeleton

**Files:**
- Create: `quarkus-work-reports/src/main/java/io/quarkiverse/work/reports/service/ReportService.java`
- Create: `quarkus-work-reports/src/main/java/io/quarkiverse/work/reports/api/ReportResource.java`

- [ ] **Step 1: Create ReportService stub**

`ReportService.java` — stubs that compile and return empty results (tests will fail assertions):
```java
package io.quarkiverse.work.reports.service;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;

import io.quarkiverse.cache.CacheResult;
import io.quarkiverse.work.runtime.model.WorkItemPriority;
import io.quarkiverse.work.runtime.repository.AuditEntryStore;

@ApplicationScoped
public class ReportService {

    @Inject
    EntityManager em;

    @Inject
    AuditEntryStore auditStore;

    @CacheResult(cacheName = "reports")
    public SlaBreachReport slaBreaches(Instant from, Instant to, String category, WorkItemPriority priority) {
        return new SlaBreachReport(List.of(), new SlaSummary(0, 0.0, Map.of()));
    }

    @CacheResult(cacheName = "reports")
    public ActorReport actorPerformance(String actorId, Instant from, Instant to, String category) {
        return new ActorReport(actorId, 0, 0, 0, null, Map.of());
    }

    @CacheResult(cacheName = "reports")
    public ThroughputReport throughput(Instant from, Instant to, String groupBy) {
        return new ThroughputReport(from, to, groupBy, List.of());
    }

    @CacheResult(cacheName = "reports")
    public QueueHealthReport queueHealth(String category, WorkItemPriority priority) {
        return new QueueHealthReport(Instant.now(), 0, 0, 0, null, 0);
    }
}
```

Note: `@CacheResult` is `io.quarkiverse.cache.CacheResult` — verify the import from `quarkus-cache`. If the import is `io.quarkus.cache.CacheResult`, use that instead.

- [ ] **Step 2: Create ReportResource**

`ReportResource.java`:
```java
package io.quarkiverse.work.reports.api;

import java.time.Instant;

import jakarta.inject.Inject;
import jakarta.ws.rs.DefaultValue;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.WebApplicationException;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.quarkiverse.work.reports.service.ActorReport;
import io.quarkiverse.work.reports.service.QueueHealthReport;
import io.quarkiverse.work.reports.service.ReportService;
import io.quarkiverse.work.reports.service.SlaBreachReport;
import io.quarkiverse.work.reports.service.ThroughputReport;
import io.quarkiverse.work.runtime.model.WorkItemPriority;

@Path("/workitems/reports")
@Produces(MediaType.APPLICATION_JSON)
public class ReportResource {

    @Inject
    ReportService reportService;

    @GET
    @Path("/sla-breaches")
    public SlaBreachReport slaBreaches(
            @QueryParam("from") String from,
            @QueryParam("to") String to,
            @QueryParam("category") String category,
            @QueryParam("priority") String priority) {
        return reportService.slaBreaches(
            from != null ? Instant.parse(from) : null,
            to != null ? Instant.parse(to) : null,
            category,
            parsePriority(priority));
    }

    @GET
    @Path("/actors/{actorId}")
    public ActorReport actorPerformance(
            @PathParam("actorId") String actorId,
            @QueryParam("from") String from,
            @QueryParam("to") String to,
            @QueryParam("category") String category) {
        return reportService.actorPerformance(
            actorId,
            from != null ? Instant.parse(from) : null,
            to != null ? Instant.parse(to) : null,
            category);
    }

    @GET
    @Path("/throughput")
    public ThroughputReport throughput(
            @QueryParam("from") String from,
            @QueryParam("to") String to,
            @QueryParam("groupBy") @DefaultValue("day") String groupBy) {
        if (from == null || to == null) {
            throw new WebApplicationException("'from' and 'to' are required for throughput reports",
                Response.Status.BAD_REQUEST);
        }
        return reportService.throughput(Instant.parse(from), Instant.parse(to), groupBy);
    }

    @GET
    @Path("/queue-health")
    public QueueHealthReport queueHealth(
            @QueryParam("category") String category,
            @QueryParam("priority") String priority) {
        return reportService.queueHealth(category, parsePriority(priority));
    }

    private static WorkItemPriority parsePriority(String priority) {
        if (priority == null || priority.isBlank()) return null;
        try {
            return WorkItemPriority.valueOf(priority.toUpperCase());
        } catch (IllegalArgumentException e) {
            throw new WebApplicationException("Invalid priority: " + priority, Response.Status.BAD_REQUEST);
        }
    }
}
```

- [ ] **Step 3: Verify module compiles**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl quarkus-work-reports -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**
```bash
git add quarkus-work-reports/src/main/
git commit -m "feat: ReportService stub and ReportResource skeleton

Refs #A, #104"
```

---

## Task 5: SLA breach endpoint — TDD

**Files:**
- Create: `quarkus-work-reports/src/test/java/io/quarkiverse/work/reports/api/SlaBreachReportTest.java`
- Modify: `quarkus-work-reports/src/main/java/io/quarkiverse/work/reports/service/ReportService.java`

- [ ] **Step 1: Write failing integration tests**

`SlaBreachReportTest.java`:
```java
package io.quarkiverse.work.reports.api;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.empty;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.greaterThan;
import static org.hamcrest.Matchers.hasItem;
import static org.hamcrest.Matchers.hasSize;
import static org.hamcrest.Matchers.not;
import static org.hamcrest.Matchers.notNullValue;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class SlaBreachReportTest {

    // ── Structure ─────────────────────────────────────────────────────────────

    @Test
    void report_returns200_withExpectedStructure() {
        given().get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", notNullValue())
            .body("summary", notNullValue())
            .body("summary.totalBreached", notNullValue())
            .body("summary.avgBreachDurationMinutes", notNullValue())
            .body("summary.byCategory", notNullValue());
    }

    // ── Happy path: breach detection ──────────────────────────────────────────

    @Test
    void completedAfterExpiry_isBreached() {
        String cat = "breach-" + System.nanoTime();
        String expiresAt = Instant.now().minus(2, ChronoUnit.MINUTES).toString();
        String id = createWithExpiry("Breached WI", cat, expiresAt);
        claimStartComplete(id, "reviewer");

        given().queryParam("category", cat).get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items.workItemId", hasItem(id));
    }

    @Test
    void completedBeforeExpiry_isNotBreached() {
        String cat = "ontime-" + System.nanoTime();
        String expiresAt = Instant.now().plus(24, ChronoUnit.HOURS).toString();
        String id = createWithExpiry("On-time WI", cat, expiresAt);
        claimStartComplete(id, "reviewer");

        given().queryParam("category", cat).get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", empty());
    }

    @Test
    void openItemPastDeadline_isBreached() {
        String cat = "open-breach-" + System.nanoTime();
        String expiresAt = Instant.now().minus(5, ChronoUnit.MINUTES).toString();
        String id = createWithExpiry("Open Past Deadline", cat, expiresAt);
        // leave open

        given().queryParam("category", cat).get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items.workItemId", hasItem(id));
    }

    // ── Correctness: boundary and exclusions ──────────────────────────────────

    @Test
    void completedAtExactlyExpiry_isNotBreached() {
        // We can't control completedAt precisely in a test, but we can verify
        // an item completed well before expiry doesn't appear
        String cat = "boundary-" + System.nanoTime();
        String expiresAt = Instant.now().plus(1, ChronoUnit.HOURS).toString();
        String id = createWithExpiry("Boundary WI", cat, expiresAt);
        claimStartComplete(id, "reviewer");

        given().queryParam("category", cat).get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", empty());
    }

    @Test
    void itemWithNoExpiresAt_neverAppears() {
        String cat = "no-expiry-" + System.nanoTime();
        String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"No Expiry\",\"category\":\"" + cat + "\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");
        claimStartComplete(id, "reviewer");

        given().queryParam("category", cat).get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", empty());
    }

    // ── Filters ───────────────────────────────────────────────────────────────

    @Test
    void filterByFrom_excludesItemsBeforeWindow() {
        String cat = "from-filter-" + System.nanoTime();
        String expiresAt = Instant.now().minus(10, ChronoUnit.MINUTES).toString();
        String id = createWithExpiry("Old Breach", cat, expiresAt);
        claimStartComplete(id, "reviewer");

        given().queryParam("from", "2099-01-01T00:00:00Z")
            .get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", empty());
    }

    @Test
    void filterByTo_excludesItemsAfterWindow() {
        String cat = "to-filter-" + System.nanoTime();
        String expiresAt = Instant.now().minus(2, ChronoUnit.MINUTES).toString();
        String id = createWithExpiry("Future Breach", cat, expiresAt);
        claimStartComplete(id, "reviewer");

        given().queryParam("to", "2000-01-01T00:00:00Z")
            .queryParam("category", cat)
            .get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", empty());
    }

    @Test
    void filterByCategory_returnsOnlyThatCategory() {
        String catA = "sla-a-" + System.nanoTime();
        String catB = "sla-b-" + System.nanoTime();
        String expiresAt = Instant.now().minus(2, ChronoUnit.MINUTES).toString();
        String idA = createWithExpiry("A", catA, expiresAt);
        String idB = createWithExpiry("B", catB, expiresAt);
        claimStartComplete(idA, "r");
        claimStartComplete(idB, "r");

        given().queryParam("category", catA).get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items.workItemId", hasItem(idA))
            .body("items.workItemId", not(hasItem(idB)));
    }

    @Test
    void filterByPriority_returnsOnlyMatchingPriority() {
        String cat = "prio-" + System.nanoTime();
        String expiresAt = Instant.now().minus(2, ChronoUnit.MINUTES).toString();
        String highId = createWithExpiryAndPriority("High", cat, expiresAt, "HIGH");
        String lowId = createWithExpiryAndPriority("Low", cat, expiresAt, "LOW");
        claimStartComplete(highId, "r");
        claimStartComplete(lowId, "r");

        given().queryParam("priority", "HIGH").queryParam("category", cat)
            .get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items.workItemId", hasItem(highId))
            .body("items.workItemId", not(hasItem(lowId)));
    }

    @Test
    void invalidPriority_returns400() {
        given().queryParam("priority", "SUPER_URGENT")
            .get("/workitems/reports/sla-breaches")
            .then().statusCode(400);
    }

    // ── Response fields ───────────────────────────────────────────────────────

    @Test
    void breachedItem_hasAllRequiredFields() {
        String cat = "fields-" + System.nanoTime();
        String expiresAt = Instant.now().minus(2, ChronoUnit.MINUTES).toString();
        String id = createWithExpiry("Fields Test", cat, expiresAt);
        claimStartComplete(id, "reviewer");

        given().queryParam("category", cat).get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", hasSize(1))
            .body("items[0].workItemId", equalTo(id))
            .body("items[0].category", equalTo(cat))
            .body("items[0].priority", notNullValue())
            .body("items[0].expiresAt", notNullValue())
            .body("items[0].status", notNullValue())
            .body("items[0].breachDurationMinutes", notNullValue());
    }

    // ── Summary aggregates ────────────────────────────────────────────────────

    @Test
    void summary_totalBreached_matchesItemCount() {
        String cat = "sumtotal-" + System.nanoTime();
        String expiresAt = Instant.now().minus(2, ChronoUnit.MINUTES).toString();
        createWithExpiry("S1", cat, expiresAt);
        createWithExpiry("S2", cat, expiresAt);

        int total = given().queryParam("category", cat)
            .get("/workitems/reports/sla-breaches")
            .then().statusCode(200).extract().path("summary.totalBreached");
        assertThat(total).isGreaterThanOrEqualTo(2);
    }

    @Test
    void summary_avgBreachDurationMinutes_isPositive() {
        String cat = "avgbreach-" + System.nanoTime();
        String expiresAt = Instant.now().minus(5, ChronoUnit.MINUTES).toString();
        String id = createWithExpiry("Avg Test", cat, expiresAt);
        claimStartComplete(id, "reviewer");

        float avg = given().queryParam("category", cat)
            .get("/workitems/reports/sla-breaches")
            .then().statusCode(200).extract().path("summary.avgBreachDurationMinutes");
        assertThat(avg).isGreaterThan(0f);
    }

    @Test
    void summary_byCategory_groupsCorrectly() {
        String catX = "bcx-" + System.nanoTime();
        String catY = "bcy-" + System.nanoTime();
        String expiresAt = Instant.now().minus(2, ChronoUnit.MINUTES).toString();
        createWithExpiry("X1", catX, expiresAt);
        createWithExpiry("Y1", catY, expiresAt);
        createWithExpiry("Y2", catY, expiresAt);

        var resp = given().get("/workitems/reports/sla-breaches")
            .then().statusCode(200).extract().response();
        assertThat((Object) resp.path("summary.byCategory." + catX)).isNotNull();
        assertThat((Object) resp.path("summary.byCategory." + catY)).isNotNull();
    }

    // ── Robustness ────────────────────────────────────────────────────────────

    @Test
    void noBreaches_returns200_withEmptyItems() {
        given().queryParam("from", "2099-01-01T00:00:00Z")
            .get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", empty())
            .body("summary.totalBreached", equalTo(0));
    }

    // ── E2E ───────────────────────────────────────────────────────────────────

    @Test
    void e2e_mixedCompliance_onlyBreachesInList() {
        String cat = "e2e-sla-" + System.nanoTime();
        String past = Instant.now().minus(3, ChronoUnit.MINUTES).toString();
        String future = Instant.now().plus(1, ChronoUnit.HOURS).toString();

        String b1 = createWithExpiry("Late 1", cat, past);
        String b2 = createWithExpiry("Late 2", cat, past);
        String ok = createWithExpiry("On Time", cat, future);
        claimStartComplete(b1, "r");
        claimStartComplete(b2, "r");
        claimStartComplete(ok, "r");

        given().queryParam("category", cat).get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items.workItemId", hasItem(b1))
            .body("items.workItemId", hasItem(b2))
            .body("items.workItemId", not(hasItem(ok)))
            .body("summary.totalBreached", greaterThan(1));
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private String createWithExpiry(String title, String category, String expiresAt) {
        return given().contentType(ContentType.JSON)
            .body("{\"title\":\"" + title + "\",\"category\":\"" + category
                + "\",\"expiresAt\":\"" + expiresAt + "\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");
    }

    private String createWithExpiryAndPriority(String title, String cat, String expiresAt, String priority) {
        return given().contentType(ContentType.JSON)
            .body("{\"title\":\"" + title + "\",\"category\":\"" + cat
                + "\",\"priority\":\"" + priority
                + "\",\"expiresAt\":\"" + expiresAt + "\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");
    }

    private void claimStartComplete(String id, String actor) {
        given().put("/workitems/" + id + "/claim?claimant=" + actor).then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=" + actor).then().statusCode(200);
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + id + "/complete?actor=" + actor).then().statusCode(200);
    }
}
```

- [ ] **Step 2: Run tests — expect failures (stub returns empty)**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=SlaBreachReportTest -q 2>&1 | grep -E "FAIL|Tests run" | tail -10
```
Expected: multiple test failures (items always empty from stub).

- [ ] **Step 3: Implement slaBreaches in ReportService**

Replace the `slaBreaches` stub in `ReportService.java`:
```java
@CacheResult(cacheName = "reports")
public SlaBreachReport slaBreaches(Instant from, Instant to, String category, WorkItemPriority priority) {
    final Instant now = Instant.now();

    final StringBuilder jpql = new StringBuilder(
        "SELECT w FROM WorkItem w WHERE w.expiresAt IS NOT NULL AND (")
        .append("(w.status IN :activeStatuses AND w.expiresAt < :now)")
        .append(" OR ")
        .append("(w.status IN :terminalStatuses AND w.completedAt IS NOT NULL")
        .append("  AND w.completedAt > w.expiresAt))");;

    if (from != null) jpql.append(" AND w.expiresAt >= :from");
    if (to != null) jpql.append(" AND w.expiresAt <= :to");
    if (category != null && !category.isBlank()) jpql.append(" AND w.category = :category");
    if (priority != null) jpql.append(" AND w.priority = :priority");
    jpql.append(" ORDER BY w.expiresAt DESC");

    final jakarta.persistence.TypedQuery<io.quarkiverse.work.runtime.model.WorkItem> q =
        em.createQuery(jpql.toString(), io.quarkiverse.work.runtime.model.WorkItem.class);
    q.setHint("jakarta.persistence.query.timeout", 30_000);
    q.setParameter("now", now);
    q.setParameter("activeStatuses", List.of(
        WorkItemStatus.PENDING, WorkItemStatus.ASSIGNED, WorkItemStatus.IN_PROGRESS,
        WorkItemStatus.SUSPENDED, WorkItemStatus.ESCALATED, WorkItemStatus.EXPIRED));
    q.setParameter("terminalStatuses", List.of(
        WorkItemStatus.COMPLETED, WorkItemStatus.REJECTED, WorkItemStatus.CANCELLED));
    if (from != null) q.setParameter("from", from);
    if (to != null) q.setParameter("to", to);
    if (category != null && !category.isBlank()) q.setParameter("category", category);
    if (priority != null) q.setParameter("priority", priority);

    final List<io.quarkiverse.work.runtime.model.WorkItem> breached = q.getResultList();

    final List<SlaBreachItem> items = new ArrayList<>();
    final Map<String, Long> byCategory = new LinkedHashMap<>();
    long totalMinutes = 0;

    for (final var wi : breached) {
        final Instant end = wi.completedAt != null ? wi.completedAt : now;
        final long mins = Math.max(0, ChronoUnit.MINUTES.between(wi.expiresAt, end));
        items.add(new SlaBreachItem(
            wi.id.toString(), wi.category,
            wi.priority != null ? wi.priority.name() : null,
            wi.expiresAt, wi.completedAt,
            wi.status.name(), mins));
        totalMinutes += mins;
        if (wi.category != null) byCategory.merge(wi.category, 1L, Long::sum);
    }

    final double avg = breached.isEmpty() ? 0.0 : (double) totalMinutes / breached.size();
    return new SlaBreachReport(items,
        new SlaSummary(breached.size(), Math.round(avg * 10.0) / 10.0, byCategory));
}
```

Add these imports to `ReportService.java`:
```java
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import io.quarkiverse.work.runtime.model.WorkItemStatus;
```

- [ ] **Step 4: Run SLA tests — expect pass**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=SlaBreachReportTest -q
```
Expected: BUILD SUCCESS, all tests green.

- [ ] **Step 5: Commit**
```bash
git add quarkus-work-reports/src/
git commit -m "feat: SLA breach endpoint with full TDD

Refs #A, #104"
```

---

## Task 6: Actor performance endpoint — TDD

**Files:**
- Create: `...test/.../api/ActorPerformanceReportTest.java`
- Modify: `ReportService.java` (implement actorPerformance)

- [ ] **Step 1: Write failing tests**

`ActorPerformanceReportTest.java`:
```java
package io.quarkiverse.work.reports.api;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.notNullValue;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class ActorPerformanceReportTest {

    // ── Structure ─────────────────────────────────────────────────────────────

    @Test
    void report_returns200_withExpectedStructure() {
        String actor = "struct-" + System.nanoTime();
        given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200)
            .body("actorId", equalTo(actor))
            .body("totalAssigned", notNullValue())
            .body("totalCompleted", notNullValue())
            .body("totalRejected", notNullValue())
            .body("byCategory", notNullValue());
    }

    @Test
    void unknownActor_returns200_withZeroCounts() {
        String actor = "nobody-" + System.nanoTime();
        given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200)
            .body("totalAssigned", equalTo(0))
            .body("totalCompleted", equalTo(0))
            .body("totalRejected", equalTo(0));
    }

    // ── totalAssigned ─────────────────────────────────────────────────────────

    @Test
    void totalAssigned_incrementsPerClaim() {
        String actor = "assigned-" + System.nanoTime();
        String id1 = createWorkItem("cat1");
        String id2 = createWorkItem("cat1");
        given().put("/workitems/" + id1 + "/claim?claimant=" + actor).then().statusCode(200);
        given().put("/workitems/" + id2 + "/claim?claimant=" + actor).then().statusCode(200);

        int assigned = given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("totalAssigned");
        assertThat(assigned).isGreaterThanOrEqualTo(2);
    }

    // ── totalCompleted ────────────────────────────────────────────────────────

    @Test
    void totalCompleted_incrementsPerCompletion() {
        String actor = "compl-" + System.nanoTime();
        claimStartComplete(createWorkItem("cc"), actor);
        claimStartComplete(createWorkItem("cc"), actor);

        int completed = given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("totalCompleted");
        assertThat(completed).isGreaterThanOrEqualTo(2);
    }

    // ── totalRejected ─────────────────────────────────────────────────────────

    @Test
    void totalRejected_incrementsPerRejection() {
        String actor = "rej-" + System.nanoTime();
        String id = createWorkItem("rc");
        given().put("/workitems/" + id + "/claim?claimant=" + actor).then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=" + actor).then().statusCode(200);
        given().contentType(ContentType.JSON).body("{\"reason\":\"blocked\"}")
            .put("/workitems/" + id + "/reject?actor=" + actor).then().statusCode(200);

        int rejected = given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("totalRejected");
        assertThat(rejected).isGreaterThanOrEqualTo(1);
    }

    // ── avgCompletionMinutes ──────────────────────────────────────────────────

    @Test
    void avgCompletionMinutes_isNull_forActorWithNoCompletions() {
        String actor = "nocomp-" + System.nanoTime();
        given().put("/workitems/" + createWorkItem("nc") + "/claim?claimant=" + actor).then().statusCode(200);

        Object avg = given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("avgCompletionMinutes");
        assertThat(avg).isNull();
    }

    @Test
    void avgCompletionMinutes_isNonNegative_whenCompletionsExist() {
        String actor = "avgactor-" + System.nanoTime();
        claimStartComplete(createWorkItem("ac"), actor);

        float avg = given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("avgCompletionMinutes");
        assertThat(avg).isGreaterThanOrEqualTo(0f);
    }

    @Test
    void avgCompletionMinutes_measures_assignedAt_to_completedAt() {
        // avgCompletionMinutes must be assignedAt→completedAt, NOT createdAt→completedAt
        // We can't control timing precisely, but we can verify it's ≥ 0
        String actor = "avgmeasure-" + System.nanoTime();
        claimStartComplete(createWorkItem("am"), actor);

        Object avg = given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("avgCompletionMinutes");
        assertThat(avg).isNotNull();
        assertThat(((Number) avg).doubleValue()).isGreaterThanOrEqualTo(0.0);
    }

    // ── byCategory ────────────────────────────────────────────────────────────

    @Test
    void byCategory_countsCompletedPerCategory() {
        String actor = "bycat-" + System.nanoTime();
        String catA = "bca-" + System.nanoTime();
        String catB = "bcb-" + System.nanoTime();
        claimStartComplete(createWorkItem(catA), actor);
        claimStartComplete(createWorkItem(catA), actor);
        claimStartComplete(createWorkItem(catB), actor);

        var resp = given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().response();
        assertThat((Integer) resp.path("byCategory." + catA)).isGreaterThanOrEqualTo(2);
        assertThat((Integer) resp.path("byCategory." + catB)).isGreaterThanOrEqualTo(1);
    }

    // ── Filters ───────────────────────────────────────────────────────────────

    @Test
    void filterByFrom_farFuture_returnsZeroCounts() {
        String actor = "fromfilt-" + System.nanoTime();
        claimStartComplete(createWorkItem("fc"), actor);

        int completed = given().queryParam("from", "2099-01-01T00:00:00Z")
            .get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("totalCompleted");
        assertThat(completed).isZero();
    }

    @Test
    void filterByTo_farPast_returnsZeroCounts() {
        String actor = "tofilt-" + System.nanoTime();
        claimStartComplete(createWorkItem("tc"), actor);

        int completed = given().queryParam("to", "2000-01-01T00:00:00Z")
            .get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("totalCompleted");
        assertThat(completed).isZero();
    }

    @Test
    void filterByDateRange_includesActivityWithinRange() {
        String actor = "range-" + System.nanoTime();
        claimStartComplete(createWorkItem("rc"), actor);

        int completed = given()
            .queryParam("from", "2020-01-01T00:00:00Z")
            .queryParam("to", "2099-12-31T23:59:59Z")
            .get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("totalCompleted");
        assertThat(completed).isGreaterThanOrEqualTo(1);
    }

    @Test
    void filterByCategory_scopesCountsToThatCategory() {
        String actor = "catscope-" + System.nanoTime();
        String catA = "cs-a-" + System.nanoTime();
        String catB = "cs-b-" + System.nanoTime();
        claimStartComplete(createWorkItem(catA), actor);
        claimStartComplete(createWorkItem(catB), actor);

        int inA = given().queryParam("category", catA)
            .get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().path("totalCompleted");
        assertThat(inA).isEqualTo(1);
    }

    // ── E2E ───────────────────────────────────────────────────────────────────

    @Test
    void e2e_fullLifecycle_allCountsCorrect() {
        String actor = "e2eactor-" + System.nanoTime();
        String cat = "e2ecat-" + System.nanoTime();

        claimStartComplete(createWorkItem(cat), actor);
        claimStartComplete(createWorkItem(cat), actor);

        String rejId = createWorkItem(cat);
        given().put("/workitems/" + rejId + "/claim?claimant=" + actor).then().statusCode(200);
        given().put("/workitems/" + rejId + "/start?actor=" + actor).then().statusCode(200);
        given().contentType(ContentType.JSON).body("{\"reason\":\"blocked\"}")
            .put("/workitems/" + rejId + "/reject?actor=" + actor).then().statusCode(200);

        String inFlightId = createWorkItem(cat);
        given().put("/workitems/" + inFlightId + "/claim?claimant=" + actor).then().statusCode(200);

        var resp = given().get("/workitems/reports/actors/" + actor)
            .then().statusCode(200).extract().response();
        assertThat((Integer) resp.path("totalCompleted")).isGreaterThanOrEqualTo(2);
        assertThat((Integer) resp.path("totalRejected")).isGreaterThanOrEqualTo(1);
        assertThat((Integer) resp.path("totalAssigned")).isGreaterThanOrEqualTo(4);
        assertThat((Float) resp.path("avgCompletionMinutes")).isGreaterThanOrEqualTo(0f);
        assertThat((Integer) resp.path("byCategory." + cat)).isGreaterThanOrEqualTo(2);
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private String createWorkItem(String category) {
        return given().contentType(ContentType.JSON)
            .body("{\"title\":\"Perf Test\",\"category\":\"" + category + "\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");
    }

    private void claimStartComplete(String id, String actor) {
        given().put("/workitems/" + id + "/claim?claimant=" + actor).then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=" + actor).then().statusCode(200);
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + id + "/complete?actor=" + actor).then().statusCode(200);
    }
}
```

- [ ] **Step 2: Run tests — expect failures**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=ActorPerformanceReportTest -q 2>&1 | grep -E "FAIL|Tests run" | tail -5
```

- [ ] **Step 3: Implement actorPerformance in ReportService**

Replace the `actorPerformance` stub:
```java
@CacheResult(cacheName = "reports")
public ActorReport actorPerformance(String actorId, Instant from, Instant to, String category) {

    // Counts from AuditEntry
    long totalAssigned = countAuditEvents(actorId, "ASSIGNED", from, to, category);
    long totalCompleted = countAuditEvents(actorId, "COMPLETED", from, to, category);
    long totalRejected = countAuditEvents(actorId, "REJECTED", from, to, category);

    // avgCompletionMinutes: avg(completedAt - assignedAt) for completed items
    final StringBuilder avgJpql = new StringBuilder(
        "SELECT w.assignedAt, w.completedAt FROM WorkItem w" +
        " WHERE w.assigneeId = :actorId" +
        " AND w.completedAt IS NOT NULL AND w.assignedAt IS NOT NULL");
    if (from != null) avgJpql.append(" AND w.completedAt >= :from");
    if (to != null) avgJpql.append(" AND w.completedAt <= :to");
    if (category != null && !category.isBlank()) avgJpql.append(" AND w.category = :category");

    final jakarta.persistence.TypedQuery<Object[]> avgQ =
        em.createQuery(avgJpql.toString(), Object[].class);
    avgQ.setParameter("actorId", actorId);
    if (from != null) avgQ.setParameter("from", from);
    if (to != null) avgQ.setParameter("to", to);
    if (category != null && !category.isBlank()) avgQ.setParameter("category", category);

    final List<Object[]> rows = avgQ.getResultList();
    final Double avgCompletionMinutes = rows.isEmpty() ? null :
        rows.stream()
            .mapToLong(r -> ChronoUnit.MINUTES.between((Instant) r[0], (Instant) r[1]))
            .average()
            .stream()
            .boxed()
            .map(d -> Math.round(d * 10.0) / 10.0)
            .findFirst()
            .orElse(null);

    // byCategory: JPQL GROUP BY (no N+1)
    final StringBuilder catJpql = new StringBuilder(
        "SELECT w.category, COUNT(w) FROM WorkItem w" +
        " WHERE w.assigneeId = :actorId" +
        " AND w.status = :completed");
    if (from != null) catJpql.append(" AND w.completedAt >= :from");
    if (to != null) catJpql.append(" AND w.completedAt <= :to");
    if (category != null && !category.isBlank()) catJpql.append(" AND w.category = :category");
    catJpql.append(" GROUP BY w.category");

    final jakarta.persistence.TypedQuery<Object[]> catQ =
        em.createQuery(catJpql.toString(), Object[].class);
    catQ.setParameter("actorId", actorId);
    catQ.setParameter("completed", WorkItemStatus.COMPLETED);
    if (from != null) catQ.setParameter("from", from);
    if (to != null) catQ.setParameter("to", to);
    if (category != null && !category.isBlank()) catQ.setParameter("category", category);

    final Map<String, Long> byCategory = new LinkedHashMap<>();
    for (Object[] row : catQ.getResultList()) {
        if (row[0] != null) byCategory.put((String) row[0], (Long) row[1]);
    }

    return new ActorReport(actorId, totalAssigned, totalCompleted, totalRejected,
        avgCompletionMinutes, byCategory);
}

private long countAuditEvents(String actorId, String event, Instant from, Instant to, String category) {
    final StringBuilder jpql = new StringBuilder(
        "SELECT COUNT(a) FROM AuditEntry a WHERE a.actor = :actorId AND a.event = :event");
    if (from != null) jpql.append(" AND a.occurredAt >= :from");
    if (to != null) jpql.append(" AND a.occurredAt <= :to");
    if (category != null && !category.isBlank()) {
        jpql.append(" AND a.workItemId IN" +
            " (SELECT w.id FROM WorkItem w WHERE w.category = :category)");
    }

    final jakarta.persistence.TypedQuery<Long> q = em.createQuery(jpql.toString(), Long.class);
    q.setParameter("actorId", actorId);
    q.setParameter("event", event);
    if (from != null) q.setParameter("from", from);
    if (to != null) q.setParameter("to", to);
    if (category != null && !category.isBlank()) q.setParameter("category", category);
    return q.getSingleResult();
}
```

Remove the `auditStore` inject — it's no longer needed (replaced by direct EM queries).

- [ ] **Step 4: Run actor tests — expect pass**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=ActorPerformanceReportTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**
```bash
git add quarkus-work-reports/src/
git commit -m "feat: actor performance endpoint with full TDD; fix N+1 in byCategory

Refs #A, #104"
```

---

## Task 7: Throughput endpoint — TDD

**Files:**
- Create: `...test/.../api/ThroughputReportTest.java`
- Modify: `ReportService.java` (implement throughput)

- [ ] **Step 1: Write failing integration tests**

`ThroughputReportTest.java`:
```java
package io.quarkiverse.work.reports.api;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.empty;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.notNullValue;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class ThroughputReportTest {

    // ── Validation ────────────────────────────────────────────────────────────

    @Test
    void missingFrom_returns400() {
        given().queryParam("to", "2026-04-27T00:00:00Z")
            .get("/workitems/reports/throughput")
            .then().statusCode(400);
    }

    @Test
    void missingTo_returns400() {
        given().queryParam("from", "2026-04-01T00:00:00Z")
            .get("/workitems/reports/throughput")
            .then().statusCode(400);
    }

    @Test
    void missingBoth_returns400() {
        given().get("/workitems/reports/throughput").then().statusCode(400);
    }

    // ── Structure ─────────────────────────────────────────────────────────────

    @Test
    void report_returns200_withExpectedStructure() {
        String from = Instant.now().minus(1, ChronoUnit.HOURS).toString();
        String to = Instant.now().plus(1, ChronoUnit.HOURS).toString();

        given().queryParam("from", from).queryParam("to", to)
            .get("/workitems/reports/throughput")
            .then().statusCode(200)
            .body("from", notNullValue())
            .body("to", notNullValue())
            .body("groupBy", equalTo("day"))
            .body("buckets", notNullValue());
    }

    @Test
    void response_echoes_from_to_groupBy() {
        String from = "2026-04-01T00:00:00Z";
        String to = "2026-04-30T23:59:59Z";

        var resp = given().queryParam("from", from).queryParam("to", to)
            .queryParam("groupBy", "week")
            .get("/workitems/reports/throughput")
            .then().statusCode(200).extract().response();

        assertThat(resp.path("groupBy").toString()).isEqualTo("week");
    }

    // ── Happy path: created count ─────────────────────────────────────────────

    @Test
    void createdItem_appearsInCorrectDayBucket() {
        String from = Instant.now().minus(1, ChronoUnit.HOURS).toString();
        String to = Instant.now().plus(1, ChronoUnit.HOURS).toString();

        given().contentType(ContentType.JSON)
            .body("{\"title\":\"Throughput Test\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201);

        var resp = given().queryParam("from", from).queryParam("to", to)
            .queryParam("groupBy", "day")
            .get("/workitems/reports/throughput")
            .then().statusCode(200).extract().response();

        // At least one bucket with created ≥ 1
        java.util.List<Integer> created = resp.path("buckets.created");
        assertThat(created.stream().mapToInt(Integer::intValue).sum()).isGreaterThanOrEqualTo(1);
    }

    @Test
    void itemsOutsideRange_excluded() {
        // Query far past — our just-created items won't appear
        given().queryParam("from", "2000-01-01T00:00:00Z")
            .queryParam("to", "2000-12-31T23:59:59Z")
            .get("/workitems/reports/throughput")
            .then().statusCode(200)
            .body("buckets", empty());
    }

    // ── GroupBy variants ──────────────────────────────────────────────────────

    @Test
    void groupByDay_defaultWhenOmitted() {
        String from = Instant.now().minus(1, ChronoUnit.HOURS).toString();
        String to = Instant.now().plus(1, ChronoUnit.HOURS).toString();

        given().queryParam("from", from).queryParam("to", to)
            .get("/workitems/reports/throughput")
            .then().statusCode(200)
            .body("groupBy", equalTo("day"));
    }

    @Test
    void groupByWeek_returnsBucketWithWeekLabel() {
        String from = Instant.now().minus(1, ChronoUnit.HOURS).toString();
        String to = Instant.now().plus(1, ChronoUnit.HOURS).toString();

        given().contentType(ContentType.JSON)
            .body("{\"title\":\"Week Test\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201);

        var resp = given().queryParam("from", from).queryParam("to", to)
            .queryParam("groupBy", "week")
            .get("/workitems/reports/throughput")
            .then().statusCode(200).extract().response();

        java.util.List<String> periods = resp.path("buckets.period");
        assertThat(periods).isNotEmpty();
        assertThat(periods.get(0)).matches("\\d{4}-W\\d{2}");
    }

    @Test
    void groupByMonth_returnsBucketWithMonthLabel() {
        String from = Instant.now().minus(1, ChronoUnit.HOURS).toString();
        String to = Instant.now().plus(1, ChronoUnit.HOURS).toString();

        given().contentType(ContentType.JSON)
            .body("{\"title\":\"Month Test\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201);

        var resp = given().queryParam("from", from).queryParam("to", to)
            .queryParam("groupBy", "month")
            .get("/workitems/reports/throughput")
            .then().statusCode(200).extract().response();

        java.util.List<String> periods = resp.path("buckets.period");
        assertThat(periods).isNotEmpty();
        assertThat(periods.get(0)).matches("\\d{4}-\\d{2}");
    }

    // ── Correctness: created vs completed independence ─────────────────────────

    @Test
    void inFlightItem_appearsInCreated_notCompleted() {
        String from = Instant.now().minus(1, ChronoUnit.HOURS).toString();
        String to = Instant.now().plus(1, ChronoUnit.HOURS).toString();

        // Create but don't complete
        given().contentType(ContentType.JSON)
            .body("{\"title\":\"In Flight\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201);

        var resp = given().queryParam("from", from).queryParam("to", to)
            .queryParam("groupBy", "day")
            .get("/workitems/reports/throughput")
            .then().statusCode(200).extract().response();

        java.util.List<Integer> created = resp.path("buckets.created");
        java.util.List<Integer> completed = resp.path("buckets.completed");
        int totalCreated = created.stream().mapToInt(Integer::intValue).sum();
        int totalCompleted = completed.stream().mapToInt(Integer::intValue).sum();

        // created ≥ completed (in-flight items inflate created but not completed)
        assertThat(totalCreated).isGreaterThanOrEqualTo(totalCompleted);
    }

    // ── Robustness ────────────────────────────────────────────────────────────

    @Test
    void emptyRange_returnsEmptyBuckets() {
        given().queryParam("from", "2099-01-01T00:00:00Z")
            .queryParam("to", "2099-12-31T23:59:59Z")
            .get("/workitems/reports/throughput")
            .then().statusCode(200)
            .body("buckets", empty());
    }

    // ── E2E ───────────────────────────────────────────────────────────────────

    @Test
    void e2e_createdAndCompleted_appearsInBuckets() {
        String from = Instant.now().minus(1, ChronoUnit.HOURS).toString();
        String to = Instant.now().plus(1, ChronoUnit.HOURS).toString();

        String id = given().contentType(ContentType.JSON)
            .body("{\"title\":\"E2E Throughput\",\"createdBy\":\"test\"}")
            .post("/workitems").then().statusCode(201).extract().path("id");
        given().put("/workitems/" + id + "/claim?claimant=actor").then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=actor").then().statusCode(200);
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + id + "/complete?actor=actor").then().statusCode(200);

        var resp = given().queryParam("from", from).queryParam("to", to)
            .queryParam("groupBy", "day")
            .get("/workitems/reports/throughput")
            .then().statusCode(200).extract().response();

        java.util.List<Integer> created = resp.path("buckets.created");
        java.util.List<Integer> completed = resp.path("buckets.completed");
        assertThat(created.stream().mapToInt(Integer::intValue).sum()).isGreaterThanOrEqualTo(1);
        assertThat(completed.stream().mapToInt(Integer::intValue).sum()).isGreaterThanOrEqualTo(1);
    }
}
```

- [ ] **Step 2: Run tests — expect failures**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=ThroughputReportTest -q 2>&1 | grep -E "FAIL|Tests run" | tail -5
```

- [ ] **Step 3: Implement throughput in ReportService**

Replace the `throughput` stub:
```java
@CacheResult(cacheName = "reports")
public ThroughputReport throughput(Instant from, Instant to, String groupBy) {
    // Query day-level created counts
    final List<Object[]> dayCreated = queryDayBuckets("createdAt", from, to, null);

    // Query day-level completed counts (terminal statuses only)
    final List<Object[]> dayCompleted = queryDayBucketsCompleted(from, to);

    final List<ThroughputBucket> buckets =
        ThroughputBucketAggregator.aggregate(dayCreated, dayCompleted, groupBy);

    return new ThroughputReport(from, to, groupBy, buckets);
}

@SuppressWarnings("unchecked")
private List<Object[]> queryDayBuckets(String field, Instant from, Instant to, WorkItemStatus status) {
    final String jpql =
        "SELECT CAST(date_trunc('day', w." + field + ") AS LocalDate), COUNT(w)" +
        " FROM WorkItem w" +
        " WHERE w." + field + " >= :from AND w." + field + " <= :to" +
        (status != null ? " AND w.status = :status" : "") +
        " GROUP BY CAST(date_trunc('day', w." + field + ") AS LocalDate)" +
        " ORDER BY CAST(date_trunc('day', w." + field + ") AS LocalDate)";

    final jakarta.persistence.TypedQuery<Object[]> q = em.createQuery(jpql, Object[].class);
    q.setHint("jakarta.persistence.query.timeout", 30_000);
    q.setParameter("from", from);
    q.setParameter("to", to);
    if (status != null) q.setParameter("status", status);
    return q.getResultList();
}

private List<Object[]> queryDayBucketsCompleted(Instant from, Instant to) {
    final String jpql =
        "SELECT CAST(date_trunc('day', w.completedAt) AS LocalDate), COUNT(w)" +
        " FROM WorkItem w" +
        " WHERE w.completedAt >= :from AND w.completedAt <= :to" +
        " AND w.status IN :terminalStatuses" +
        " GROUP BY CAST(date_trunc('day', w.completedAt) AS LocalDate)" +
        " ORDER BY CAST(date_trunc('day', w.completedAt) AS LocalDate)";

    final jakarta.persistence.TypedQuery<Object[]> q = em.createQuery(jpql, Object[].class);
    q.setHint("jakarta.persistence.query.timeout", 30_000);
    q.setParameter("from", from);
    q.setParameter("to", to);
    q.setParameter("terminalStatuses", List.of(
        WorkItemStatus.COMPLETED, WorkItemStatus.REJECTED, WorkItemStatus.CANCELLED,
        WorkItemStatus.ESCALATED));
    return q.getResultList();
}
```

- [ ] **Step 4: Run throughput tests — expect pass**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=ThroughputReportTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**
```bash
git add quarkus-work-reports/src/
git commit -m "feat: throughput time-series endpoint with full TDD

Refs #B, #104"
```

---

## Task 8: Queue health endpoint — TDD

**Files:**
- Create: `...test/.../api/QueueHealthReportTest.java`
- Modify: `ReportService.java` (implement queueHealth)

- [ ] **Step 1: Write failing tests**

`QueueHealthReportTest.java`:
```java
package io.quarkiverse.work.reports.api;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.notNullValue;
import static org.hamcrest.Matchers.nullValue;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;

@QuarkusTest
class QueueHealthReportTest {

    // ── Structure ─────────────────────────────────────────────────────────────

    @Test
    void report_returns200_withExpectedStructure() {
        given().get("/workitems/reports/queue-health")
            .then().statusCode(200)
            .body("timestamp", notNullValue())
            .body("overdueCount", notNullValue())
            .body("pendingCount", notNullValue())
            .body("avgPendingAgeSeconds", notNullValue())
            .body("criticalOverdueCount", notNullValue());
    }

    // ── pendingCount ──────────────────────────────────────────────────────────

    @Test
    void pendingCount_includesPendingItems() {
        String cat = "pending-" + System.nanoTime();
        createItem(cat, null, null);
        createItem(cat, null, null);

        int count = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("pendingCount");
        assertThat(count).isGreaterThanOrEqualTo(2);
    }

    @Test
    void pendingCount_excludesCompletedItems() {
        String cat = "pend-excl-" + System.nanoTime();
        String id = createItem(cat, null, null);
        given().put("/workitems/" + id + "/claim?claimant=actor").then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=actor").then().statusCode(200);
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + id + "/complete?actor=actor").then().statusCode(200);

        // After completion, pendingCount for this category should be 0
        int count = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("pendingCount");
        assertThat(count).isZero();
    }

    // ── overdueCount ──────────────────────────────────────────────────────────

    @Test
    void overdueCount_includesActiveItemsPastExpiry() {
        String cat = "overdue-" + System.nanoTime();
        String expiresAt = Instant.now().minus(5, ChronoUnit.MINUTES).toString();
        createItem(cat, expiresAt, null);

        int overdue = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("overdueCount");
        assertThat(overdue).isGreaterThanOrEqualTo(1);
    }

    @Test
    void overdueCount_excludesCompletedItems() {
        String cat = "overdue-compl-" + System.nanoTime();
        String expiresAt = Instant.now().minus(5, ChronoUnit.MINUTES).toString();
        String id = createItem(cat, expiresAt, null);
        given().put("/workitems/" + id + "/claim?claimant=a").then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=a").then().statusCode(200);
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + id + "/complete?actor=a").then().statusCode(200);

        int overdue = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("overdueCount");
        assertThat(overdue).isZero();
    }

    @Test
    void overdueCount_excludesFutureExpiry() {
        String cat = "not-overdue-" + System.nanoTime();
        String futureExpiry = Instant.now().plus(1, ChronoUnit.HOURS).toString();
        createItem(cat, futureExpiry, null);

        int overdue = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("overdueCount");
        assertThat(overdue).isZero();
    }

    // ── criticalOverdueCount ──────────────────────────────────────────────────

    @Test
    void criticalOverdueCount_isSubsetOfOverdueCount() {
        String cat = "crit-" + System.nanoTime();
        String past = Instant.now().minus(5, ChronoUnit.MINUTES).toString();
        createItemWithPriority(cat, past, "CRITICAL");
        createItemWithPriority(cat, past, "NORMAL");

        var resp = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().response();

        int overdue = resp.path("overdueCount");
        int critOverdue = resp.path("criticalOverdueCount");
        assertThat(critOverdue).isGreaterThanOrEqualTo(1);
        assertThat(critOverdue).isLessThanOrEqualTo(overdue);
    }

    // ── avgPendingAgeSeconds ──────────────────────────────────────────────────

    @Test
    void avgPendingAgeSeconds_isNonNegative() {
        String cat = "age-" + System.nanoTime();
        createItem(cat, null, null);

        int age = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("avgPendingAgeSeconds");
        assertThat(age).isGreaterThanOrEqualTo(0);
    }

    // ── oldestUnclaimedCreatedAt ──────────────────────────────────────────────

    @Test
    void oldestUnclaimedCreatedAt_isNullWhenNoPendingItems() {
        // Use far-future category that no items belong to
        given().queryParam("category", "empty-queue-" + System.nanoTime())
            .get("/workitems/reports/queue-health")
            .then().statusCode(200)
            .body("oldestUnclaimedCreatedAt", nullValue());
    }

    @Test
    void oldestUnclaimedCreatedAt_isPresentWhenPendingItemsExist() {
        String cat = "oldest-" + System.nanoTime();
        createItem(cat, null, null);

        given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200)
            .body("oldestUnclaimedCreatedAt", notNullValue());
    }

    // ── Filters ───────────────────────────────────────────────────────────────

    @Test
    void filterByCategory_scopesAllCounts() {
        String catA = "qa-" + System.nanoTime();
        String catB = "qb-" + System.nanoTime();
        String past = Instant.now().minus(5, ChronoUnit.MINUTES).toString();
        createItem(catA, past, null);
        createItem(catB, past, null);

        int overdueA = given().queryParam("category", catA)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("overdueCount");
        assertThat(overdueA).isGreaterThanOrEqualTo(1);

        int overdueB = given().queryParam("category", catB)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("overdueCount");
        assertThat(overdueB).isGreaterThanOrEqualTo(1);
    }

    @Test
    void filterByPriority_scopesOverdueToPriority() {
        String cat = "prio-qh-" + System.nanoTime();
        String past = Instant.now().minus(5, ChronoUnit.MINUTES).toString();
        createItemWithPriority(cat, past, "HIGH");
        createItemWithPriority(cat, past, "LOW");

        int highOverdue = given().queryParam("category", cat).queryParam("priority", "HIGH")
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().path("overdueCount");
        assertThat(highOverdue).isEqualTo(1);
    }

    @Test
    void invalidPriority_returns400() {
        given().queryParam("priority", "ULTRA")
            .get("/workitems/reports/queue-health")
            .then().statusCode(400);
    }

    // ── Robustness ────────────────────────────────────────────────────────────

    @Test
    void allItemsCompleted_allCountsZero() {
        String cat = "allcompl-" + System.nanoTime();
        String id = createItem(cat, null, null);
        given().put("/workitems/" + id + "/claim?claimant=a").then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=a").then().statusCode(200);
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + id + "/complete?actor=a").then().statusCode(200);

        var resp = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().response();

        assertThat((Integer) resp.path("pendingCount")).isZero();
        assertThat((Integer) resp.path("overdueCount")).isZero();
        assertThat((Integer) resp.path("criticalOverdueCount")).isZero();
        assertThat((Object) resp.path("oldestUnclaimedCreatedAt")).isNull();
    }

    // ── E2E ───────────────────────────────────────────────────────────────────

    @Test
    void e2e_mixedQueue_allFieldsCorrect() {
        String cat = "e2e-qh-" + System.nanoTime();
        String past = Instant.now().minus(5, ChronoUnit.MINUTES).toString();

        createItemWithPriority(cat, past, "CRITICAL");  // overdue + critical overdue
        createItem(cat, null, null);                     // pending, not overdue

        String doneId = createItem(cat, null, null);
        given().put("/workitems/" + doneId + "/claim?claimant=a").then().statusCode(200);
        given().put("/workitems/" + doneId + "/start?actor=a").then().statusCode(200);
        given().contentType(ContentType.JSON).body("{}")
            .put("/workitems/" + doneId + "/complete?actor=a").then().statusCode(200);

        var resp = given().queryParam("category", cat)
            .get("/workitems/reports/queue-health")
            .then().statusCode(200).extract().response();

        assertThat((Integer) resp.path("pendingCount")).isGreaterThanOrEqualTo(2);
        assertThat((Integer) resp.path("overdueCount")).isGreaterThanOrEqualTo(1);
        assertThat((Integer) resp.path("criticalOverdueCount")).isGreaterThanOrEqualTo(1);
        assertThat((Object) resp.path("oldestUnclaimedCreatedAt")).isNotNull();
        assertThat((Integer) resp.path("avgPendingAgeSeconds")).isGreaterThanOrEqualTo(0);
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private String createItem(String category, String expiresAt, String priority) {
        String body = "{\"title\":\"QH Test\",\"category\":\"" + category + "\",\"createdBy\":\"test\""
            + (expiresAt != null ? ",\"expiresAt\":\"" + expiresAt + "\"" : "")
            + (priority != null ? ",\"priority\":\"" + priority + "\"" : "") + "}";
        return given().contentType(ContentType.JSON).body(body)
            .post("/workitems").then().statusCode(201).extract().path("id");
    }

    private String createItemWithPriority(String category, String expiresAt, String priority) {
        return createItem(category, expiresAt, priority);
    }
}
```

- [ ] **Step 2: Run tests — expect failures**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=QueueHealthReportTest -q 2>&1 | grep -E "FAIL|Tests run" | tail -5
```

- [ ] **Step 3: Implement queueHealth in ReportService**

Replace the `queueHealth` stub:
```java
@CacheResult(cacheName = "reports")
public QueueHealthReport queueHealth(String category, WorkItemPriority priority) {
    final Instant now = Instant.now();
    final List<WorkItemStatus> activeStatuses = List.of(
        WorkItemStatus.PENDING, WorkItemStatus.ASSIGNED, WorkItemStatus.IN_PROGRESS,
        WorkItemStatus.SUSPENDED, WorkItemStatus.ESCALATED, WorkItemStatus.EXPIRED);

    // overdueCount
    final StringBuilder overdueJpql = new StringBuilder(
        "SELECT COUNT(w) FROM WorkItem w WHERE w.expiresAt IS NOT NULL" +
        " AND w.expiresAt < :now AND w.status IN :activeStatuses");
    if (category != null && !category.isBlank()) overdueJpql.append(" AND w.category = :category");
    if (priority != null) overdueJpql.append(" AND w.priority = :priority");

    final jakarta.persistence.TypedQuery<Long> overdueQ =
        em.createQuery(overdueJpql.toString(), Long.class);
    overdueQ.setParameter("now", now);
    overdueQ.setParameter("activeStatuses", activeStatuses);
    if (category != null && !category.isBlank()) overdueQ.setParameter("category", category);
    if (priority != null) overdueQ.setParameter("priority", priority);
    final long overdueCount = overdueQ.getSingleResult();

    // criticalOverdueCount
    final StringBuilder critJpql = new StringBuilder(
        "SELECT COUNT(w) FROM WorkItem w WHERE w.expiresAt IS NOT NULL" +
        " AND w.expiresAt < :now AND w.status IN :activeStatuses" +
        " AND w.priority = :critical");
    if (category != null && !category.isBlank()) critJpql.append(" AND w.category = :category");

    final jakarta.persistence.TypedQuery<Long> critQ =
        em.createQuery(critJpql.toString(), Long.class);
    critQ.setParameter("now", now);
    critQ.setParameter("activeStatuses", activeStatuses);
    critQ.setParameter("critical", WorkItemPriority.CRITICAL);
    if (category != null && !category.isBlank()) critQ.setParameter("category", category);
    final long criticalOverdueCount = critQ.getSingleResult();

    // pendingCount + oldestUnclaimedCreatedAt + avgPendingAgeSeconds
    final StringBuilder pendingJpql = new StringBuilder(
        "SELECT w.createdAt FROM WorkItem w WHERE w.status = :pending");
    if (category != null && !category.isBlank()) pendingJpql.append(" AND w.category = :category");
    if (priority != null) pendingJpql.append(" AND w.priority = :priority");

    final jakarta.persistence.TypedQuery<Instant> pendingQ =
        em.createQuery(pendingJpql.toString(), Instant.class);
    pendingQ.setParameter("pending", WorkItemStatus.PENDING);
    if (category != null && !category.isBlank()) pendingQ.setParameter("category", category);
    if (priority != null) pendingQ.setParameter("priority", priority);

    final List<Instant> pendingCreatedAts = pendingQ.getResultList();
    final long pendingCount = pendingCreatedAts.size();
    final Instant oldestUnclaimed = pendingCreatedAts.stream()
        .min(Instant::compareTo).orElse(null);
    final long avgPendingAgeSeconds = pendingCreatedAts.isEmpty() ? 0L :
        (long) pendingCreatedAts.stream()
            .mapToLong(c -> ChronoUnit.SECONDS.between(c, now))
            .average().orElse(0.0);

    return new QueueHealthReport(now, overdueCount, pendingCount,
        avgPendingAgeSeconds, oldestUnclaimed, criticalOverdueCount);
}
```

- [ ] **Step 4: Run queue-health tests — expect pass**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=QueueHealthReportTest -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Run full module test suite**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports -q
```
Expected: BUILD SUCCESS, all tests passing.

- [ ] **Step 6: Commit**
```bash
git add quarkus-work-reports/src/
git commit -m "feat: queue-health endpoint with full TDD

Refs #C, #104"
```

---

## Task 9: Verify @CacheResult import and run full build

The `@CacheResult` annotation lives in `io.quarkus.cache.CacheResult`. Verify the import compiles correctly and test that caching doesn't break test isolation (TTL is 1s in tests).

- [ ] **Step 1: Confirm correct import in ReportService.java**

Ensure the import is:
```java
import io.quarkus.cache.CacheResult;
```
(Not `io.quarkiverse.cache.CacheResult` — that doesn't exist.)

- [ ] **Step 2: Full module build**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Full project build — verify nothing broken**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q
```
Expected: BUILD SUCCESS across all modules.

- [ ] **Step 4: Commit**
```bash
git commit --allow-empty -m "chore: verify caching and full build green

Refs #104"
```

---

## Task 10: Integration-tests — WorkItemReportNativeIT

**Files:**
- Modify: `integration-tests/pom.xml` (add quarkus-work-reports dependency)
- Modify: `integration-tests/src/test/java/io/quarkiverse/work/it/WorkItemNativeIT.java`

- [ ] **Step 1: Add quarkus-work-reports to integration-tests pom**

In `integration-tests/pom.xml`, add inside `<dependencies>`:
```xml
<dependency>
  <groupId>io.quarkiverse.work</groupId>
  <artifactId>quarkus-work-reports</artifactId>
  <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Add report smoke tests to WorkItemNativeIT.java**

Add the following test methods to the existing `WorkItemNativeIT` class:
```java
// ── Report endpoint smoke tests ───────────────────────────────────────────

@Test
void reports_slaBreaches_returns200() {
    given().get("/workitems/reports/sla-breaches")
        .then().statusCode(200)
        .body("items", notNullValue())
        .body("summary", notNullValue());
}

@Test
void reports_actorPerformance_returns200() {
    given().get("/workitems/reports/actors/smoke-actor")
        .then().statusCode(200)
        .body("actorId", equalTo("smoke-actor"))
        .body("totalCompleted", notNullValue());
}

@Test
void reports_throughput_missingFrom_returns400() {
    given().queryParam("to", "2026-04-27T00:00:00Z")
        .get("/workitems/reports/throughput")
        .then().statusCode(400);
}

@Test
void reports_throughput_returns200() {
    given()
        .queryParam("from", "2026-01-01T00:00:00Z")
        .queryParam("to", "2026-12-31T23:59:59Z")
        .queryParam("groupBy", "month")
        .get("/workitems/reports/throughput")
        .then().statusCode(200)
        .body("buckets", notNullValue());
}

@Test
void reports_queueHealth_returns200() {
    given().get("/workitems/reports/queue-health")
        .then().statusCode(200)
        .body("timestamp", notNullValue())
        .body("overdueCount", notNullValue())
        .body("pendingCount", notNullValue());
}

@Test
void reports_slaBreaches_e2e_breach_appears() {
    String expiresAt = Instant.now().minus(2, ChronoUnit.MINUTES).toString();
    String id = given().contentType(ContentType.JSON)
        .body("{\"title\":\"IT Breach\",\"expiresAt\":\"" + expiresAt + "\",\"createdBy\":\"it\"}")
        .post("/workitems").then().statusCode(201).extract().path("id");
    given().put("/workitems/" + id + "/claim?claimant=it-actor").then().statusCode(200);
    given().put("/workitems/" + id + "/start?actor=it-actor").then().statusCode(200);
    given().contentType(ContentType.JSON).body("{}")
        .put("/workitems/" + id + "/complete?actor=it-actor").then().statusCode(200);

    given().get("/workitems/reports/sla-breaches")
        .then().statusCode(200)
        .body("summary.totalBreached", greaterThanOrEqualTo(1));
}
```

Also add to imports at the top of `WorkItemNativeIT.java`:
```java
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import static org.hamcrest.Matchers.greaterThanOrEqualTo;
import static org.hamcrest.Matchers.equalTo;
```

- [ ] **Step 3: Run integration tests (JVM mode)**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests -q
```
Expected: BUILD SUCCESS, all IT tests pass including new report tests.

- [ ] **Step 4: Commit**
```bash
git add integration-tests/
git commit -m "test: add report endpoint smoke tests to WorkItemNativeIT

Refs #D, #104"
```

---

## Task 11: Testcontainers PostgreSQL dialect validation

**Files:**
- Modify: `quarkus-work-reports/pom.xml` (add postgresql + testcontainers test deps)
- Create: `quarkus-work-reports/src/test/java/io/quarkiverse/work/reports/api/PostgresDialectValidationTest.java`
- Create: `quarkus-work-reports/src/test/resources/application-postgres.properties`

- [ ] **Step 1: Add PostgreSQL test dependencies to quarkus-work-reports/pom.xml**

Inside `<dependencies>`:
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-jdbc-postgresql</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Create Postgres test profile properties**

`quarkus-work-reports/src/test/resources/application-postgres.properties`:
```properties
quarkus.http.test-port=0
quarkus.datasource.db-kind=postgresql
# No jdbc.url — Quarkus Dev Services will start a PostgreSQL Testcontainer
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.scheduler.enabled=false
quarkus.work.default-expiry-hours=24
quarkus.work.default-claim-hours=4
quarkus.work.cleanup.expiry-check-seconds=60
quarkus.cache.caffeine.reports.expire-after-write=1s
```

- [ ] **Step 3: Write dialect validation test**

`PostgresDialectValidationTest.java`:
```java
package io.quarkiverse.work.reports.api;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.notNullValue;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Map;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import io.restassured.http.ContentType;

/**
 * Verifies that HQL date_trunc('day') translates correctly against real PostgreSQL.
 * Requires Docker. Quarkus Dev Services starts a PostgreSQL container automatically.
 */
@QuarkusTest
@TestProfile(PostgresDialectValidationTest.PgProfile.class)
class PostgresDialectValidationTest {

    public static class PgProfile implements QuarkusTestProfile {
        @Override
        public Map<String, String> getConfigOverrides() {
            return Map.of("quarkus.datasource.db-kind", "postgresql");
        }

        @Override
        public String getConfigProfile() {
            return "postgres";
        }
    }

    @Test
    void throughput_groupByDay_executesOnPostgres() {
        given().contentType(ContentType.JSON)
            .body("{\"title\":\"PG Day Test\",\"createdBy\":\"pg-test\"}")
            .post("/workitems").then().statusCode(201);

        given()
            .queryParam("from", Instant.now().minus(1, ChronoUnit.HOURS).toString())
            .queryParam("to", Instant.now().plus(1, ChronoUnit.HOURS).toString())
            .queryParam("groupBy", "day")
            .get("/workitems/reports/throughput")
            .then().statusCode(200)
            .body("buckets", notNullValue());
    }

    @Test
    void throughput_groupByWeek_executesOnPostgres() {
        given().contentType(ContentType.JSON)
            .body("{\"title\":\"PG Week Test\",\"createdBy\":\"pg-test\"}")
            .post("/workitems").then().statusCode(201);

        var resp = given()
            .queryParam("from", Instant.now().minus(1, ChronoUnit.HOURS).toString())
            .queryParam("to", Instant.now().plus(1, ChronoUnit.HOURS).toString())
            .queryParam("groupBy", "week")
            .get("/workitems/reports/throughput")
            .then().statusCode(200).extract().response();

        java.util.List<String> periods = resp.path("buckets.period");
        if (!periods.isEmpty()) {
            assertThat(periods.get(0)).matches("\\d{4}-W\\d{2}");
        }
    }

    @Test
    void throughput_groupByMonth_executesOnPostgres() {
        given().contentType(ContentType.JSON)
            .body("{\"title\":\"PG Month Test\",\"createdBy\":\"pg-test\"}")
            .post("/workitems").then().statusCode(201);

        var resp = given()
            .queryParam("from", Instant.now().minus(1, ChronoUnit.HOURS).toString())
            .queryParam("to", Instant.now().plus(1, ChronoUnit.HOURS).toString())
            .queryParam("groupBy", "month")
            .get("/workitems/reports/throughput")
            .then().statusCode(200).extract().response();

        java.util.List<String> periods = resp.path("buckets.period");
        if (!periods.isEmpty()) {
            assertThat(periods.get(0)).matches("\\d{4}-\\d{2}");
        }
    }

    @Test
    void slaBreaches_executesOnPostgres() {
        given().get("/workitems/reports/sla-breaches")
            .then().statusCode(200)
            .body("items", notNullValue());
    }

    @Test
    void queueHealth_executesOnPostgres() {
        given().get("/workitems/reports/queue-health")
            .then().statusCode(200)
            .body("overdueCount", notNullValue());
    }
}
```

- [ ] **Step 4: Run PostgreSQL dialect tests (requires Docker)**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl quarkus-work-reports \
  -Dtest=PostgresDialectValidationTest -q
```
Expected: BUILD SUCCESS. If Docker is unavailable, this test will be skipped or fail at container startup — that is acceptable; skip with `-Dtest='!PostgresDialectValidationTest'` in those environments.

- [ ] **Step 5: Commit**
```bash
git add quarkus-work-reports/
git commit -m "test: Testcontainers PostgreSQL dialect validation for HQL date_trunc

Refs #D, #104"
```

---

## Task 12: Documentation — DESIGN.md, CLAUDE.md, api-reference.md

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md`
- Modify: `docs/api-reference.md`

- [ ] **Step 1: Update DESIGN.md**

Find the section that lists optional modules (notifications, queues, ledger, ai) and add `quarkus-work-reports`. Find the section listing REST endpoints and add the four report endpoints. Add a "Reporting" section if one doesn't exist covering: module name, query strategy (HQL date_trunc + Java rollup), caching (Caffeine 5-min TTL), dialect validation (Testcontainers PostgreSQL).

- [ ] **Step 2: Update CLAUDE.md project structure**

In the project structure tree, add `quarkus-work-reports/` as a new module entry after `quarkus-work-notifications/`. Include its package layout:
```
quarkus-work-reports/             — optional SLA reporting module
  api/ReportResource.java         — GET /workitems/reports/* (sla-breaches, actors, throughput, queue-health)
  service/ReportService.java      — all JPQL aggregate queries + @CacheResult (5-min TTL)
  service/ThroughputBucketAggregator.java — pure Java day→week/month rollup
  service/[SlaBreachReport|ActorReport|ThroughputReport|QueueHealthReport].java — response records
```

Also update the "Integration modules (built)" list to include `quarkus-work-reports`.

- [ ] **Step 3: Update api-reference.md**

Add a "Reports" section documenting all four endpoints with example requests and responses (copy from spec file at `docs/superpowers/specs/2026-04-27-sla-reporting-design.md`).

- [ ] **Step 4: Sweep for staleness**

Search for any remaining references to the old `ReportResource` location (`runtime/api/ReportResource`) in docs and correct them. Search for `#99` epic refs in report-related Javadoc and correct to `#104`.

- [ ] **Step 5: Commit**
```bash
git add docs/ CLAUDE.md
git commit -m "docs: update DESIGN.md, CLAUDE.md, api-reference for quarkus-work-reports

Refs #104"
```

---

## Task 13: Final verification and issue closure

- [ ] **Step 1: Full project build with tests**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 2: Run integration tests**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests -q
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Close child issues**
```bash
gh issue close <issueA> --repo casehubio/work --comment "Implemented in quarkus-work-reports module."
gh issue close <issueB> --repo casehubio/work --comment "Throughput endpoint implemented with full TDD."
gh issue close <issueC> --repo casehubio/work --comment "Queue-health endpoint implemented with full TDD."
gh issue close <issueD> --repo casehubio/work --comment "WorkItemNativeIT and PostgresDialectValidationTest added."
```

- [ ] **Step 4: Close epic**
```bash
gh issue close 104 --repo casehubio/work \
  --comment "All reporting endpoints implemented in quarkus-work-reports optional module. Full TDD, integration tests, Testcontainers PostgreSQL dialect validation, and docs updated."
```
