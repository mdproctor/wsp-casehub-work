# db/work/migration Path Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move all casehub-work Flyway migrations from `classpath:db/migration` to `classpath:db/work/migration` and make the extension self-register that path via `FlywayConfigurationCustomizer` so JVM-mode consumers never need to configure it manually.

**Architecture:** A new `@ApplicationScoped` CDI bean `WorkItemsMigrationCustomizer` (in `runtime/`) implements `FlywayConfigurationCustomizer` and adds `classpath:db/work/migration` to whatever Flyway locations are already configured — idempotently and additively. The deployment processor gains a `@BuildStep` producing `NativeImageResourcePatternsBuildItem` to register SQL bytes for native image. All SQL migration files move from `db/migration/` to `db/work/migration/` in every module's `src/main/resources/`. Native-image consumers must also set `quarkus.flyway.locations=classpath:db/work/migration` at build time (JVM mode is automatic).

**Tech Stack:** Java 21, Quarkus 3.32.2, Flyway, H2 (tests), JUnit 5, AssertJ

**Spec:** `specs/issue-229-db-work-migration/2026-05-27-db-work-migration-path-design.md`

---

## File Map

| Action | File |
|---|---|
| **Create** | `runtime/src/main/java/io/casehub/work/runtime/flyway/WorkItemsMigrationCustomizer.java` |
| **Create** | `runtime/src/test/java/io/casehub/work/runtime/flyway/WorkItemsMigrationCustomizerTest.java` |
| **Modify** | `deployment/src/main/java/io/casehub/work/deployment/WorkItemsProcessor.java` |
| **Move dir** | `runtime/src/main/resources/db/migration/` → `runtime/src/main/resources/db/work/migration/` |
| **Move dir** | `ai/src/main/resources/db/migration/` → `ai/src/main/resources/db/work/migration/` |
| **Move dir** | `ledger/src/main/resources/db/migration/` → `ledger/src/main/resources/db/work/migration/` |
| **Move dir** | `notifications/src/main/resources/db/migration/` → `notifications/src/main/resources/db/work/migration/` |
| **Move dir** | `queues/src/main/resources/db/migration/` → `queues/src/main/resources/db/work/migration/` |
| **Move dir** | `issue-tracker/src/main/resources/db/migration/` → `issue-tracker/src/main/resources/db/work/migration/` |
| **Delete dir** | `core/src/main/resources/db/migration/` (empty) |
| **Move dir** | `db/migration/` (root DBA scripts) → `db/work/migration/` |
| **Modify** | `notifications/src/test/resources/application.properties` |
| **Modify** | `queues/src/test/resources/application.properties` |
| **Modify** | `queues-dashboard/src/main/resources/application.properties` |
| **Modify** | `examples/src/test/resources/application.properties` |
| **Modify** | `ledger/src/test/resources/application.properties` |
| **Modify** | `integration-tests/src/main/resources/application.properties` |
| **Create** | `~/claude/casehub/parent/docs/protocols/universal/flyway-extension-migration-registration.md` |
| **Modify** | `~/claude/casehub/parent/docs/PLATFORM.md` |
| **Modify** | `docs/FLYWAY.md` |
| **Modify** | `docs/GOTCHAS.md` |
| **Modify** | `docs/DESIGN.md` |

---

## Task 1: Write failing unit test for WorkItemsMigrationCustomizer

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/flyway/WorkItemsMigrationCustomizerTest.java`

- [ ] **Step 1.1: Create the test file**

```java
package io.casehub.work.runtime.flyway;

import org.flywaydb.core.api.Location;
import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.junit.jupiter.api.Test;

import java.util.Arrays;

import static org.assertj.core.api.Assertions.assertThat;

class WorkItemsMigrationCustomizerTest {

    private final WorkItemsMigrationCustomizer customizer = new WorkItemsMigrationCustomizer();

    private static final String WORK_PATH = "db/work/migration";

    @Test
    void addsWorkMigrationPathToConfiguredLocations() {
        FluentConfiguration config = new FluentConfiguration();
        config.locations("classpath:db/migration");

        customizer.customize(config);

        assertThat(config.getLocations())
                .extracting(Location::getPath)
                .containsExactlyInAnyOrder("db/migration", WORK_PATH);
    }

    @Test
    void isIdempotentWhenWorkPathAlreadyPresent() {
        FluentConfiguration config = new FluentConfiguration();
        config.locations("classpath:db/work/migration");

        customizer.customize(config);

        long count = Arrays.stream(config.getLocations())
                .map(Location::getPath)
                .filter(WORK_PATH::equals)
                .count();
        assertThat(count).isEqualTo(1);
    }

    @Test
    void preservesExistingLocationsWhenAdding() {
        FluentConfiguration config = new FluentConfiguration();
        config.locations("classpath:db/ledger/migration");

        customizer.customize(config);

        assertThat(config.getLocations())
                .extracting(Location::getPath)
                .containsExactlyInAnyOrder("db/ledger/migration", WORK_PATH);
    }

    @Test
    void addsWorkPathAlongsideMultipleExistingLocations() {
        FluentConfiguration config = new FluentConfiguration();
        config.locations("classpath:db/ledger/migration", "classpath:db/migration");

        customizer.customize(config);

        assertThat(config.getLocations())
                .extracting(Location::getPath)
                .containsExactlyInAnyOrder("db/ledger/migration", "db/migration", WORK_PATH);
    }
}
```

- [ ] **Step 1.2: Run the test — confirm it fails with class-not-found**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemsMigrationCustomizerTest -pl runtime
```

Expected output: `COMPILATION ERROR` or `ClassNotFoundException: WorkItemsMigrationCustomizer`

---

## Task 2: Implement WorkItemsMigrationCustomizer + deployment BuildStep

**Files:**
- Create: `runtime/src/main/java/io/casehub/work/runtime/flyway/WorkItemsMigrationCustomizer.java`
- Modify: `deployment/src/main/java/io/casehub/work/deployment/WorkItemsProcessor.java`

- [ ] **Step 2.1: Create WorkItemsMigrationCustomizer**

```java
package io.casehub.work.runtime.flyway;

import io.quarkus.flyway.FlywayConfigurationCustomizer;
import jakarta.enterprise.context.ApplicationScoped;
import org.flywaydb.core.api.Location;
import org.flywaydb.core.api.configuration.FluentConfiguration;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

@ApplicationScoped
public class WorkItemsMigrationCustomizer implements FlywayConfigurationCustomizer {

    @Override
    public void customize(FluentConfiguration configuration) {
        Location target = new Location("classpath:db/work/migration");
        List<Location> locations = new ArrayList<>(Arrays.asList(configuration.getLocations()));
        // Location.getPath() strips the classpath: prefix — comparison is prefix-agnostic
        // e.g. new Location("classpath:db/work/migration").getPath() == "db/work/migration"
        if (locations.stream().noneMatch(l -> l.getPath().equals(target.getPath()))) {
            locations.add(target);
            configuration.locations(locations.toArray(new Location[0]));
        }
    }
}
```

- [ ] **Step 2.2: Run unit tests — confirm all 4 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemsMigrationCustomizerTest -pl runtime
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

- [ ] **Step 2.3: Add NativeImageResourcePatternsBuildItem to WorkItemsProcessor**

Replace the entire file content:

```java
package io.casehub.work.deployment;

import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;
import io.quarkus.deployment.builditem.nativeimage.NativeImageResourcePatternsBuildItem;

/**
 * Quarkus build-time processor for the WorkItems extension.
 * Registers the "workitems" feature and SQL migration resources for native image.
 */
class WorkItemsProcessor {

    private static final String FEATURE = "workitems";

    @BuildStep
    FeatureBuildItem feature() {
        return new FeatureBuildItem(FEATURE);
    }

    @BuildStep
    NativeImageResourcePatternsBuildItem registerMigrationResources() {
        return new NativeImageResourcePatternsBuildItem("db/work/migration/.*\\.sql");
    }
}
```

- [ ] **Step 2.4: Compile deployment module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-compile deployment
```

Expected: `BUILD SUCCESS`

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  runtime/src/main/java/io/casehub/work/runtime/flyway/WorkItemsMigrationCustomizer.java \
  runtime/src/test/java/io/casehub/work/runtime/flyway/WorkItemsMigrationCustomizerTest.java \
  deployment/src/main/java/io/casehub/work/deployment/WorkItemsProcessor.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(flyway): add WorkItemsMigrationCustomizer and native resource BuildStep

Implements FlywayConfigurationCustomizer to self-register classpath:db/work/migration
at JVM startup — additive, idempotent, default-datasource only (Quarkus routes
unqualified FlywayConfigurationCustomizer beans via DataSourceUtil.isDefault()).
NativeImageResourcePatternsBuildItem registers SQL bytes for native image.

Refs #229"
```

---

## Task 3: Move runtime SQL migrations and verify

**Files:**
- Move dir: `runtime/src/main/resources/db/migration/` → `runtime/src/main/resources/db/work/migration/`

- [ ] **Step 3.1: Create parent directory and git-move the directory**

```bash
mkdir -p /Users/mdproctor/claude/casehub/work/runtime/src/main/resources/db/work
git -C /Users/mdproctor/claude/casehub/work mv \
  runtime/src/main/resources/db/migration \
  runtime/src/main/resources/db/work/migration
```

Expected: no output (git mv is silent on success). Verify:

```bash
ls /Users/mdproctor/claude/casehub/work/runtime/src/main/resources/db/work/migration/ | head -5
```

Expected: SQL files listed (V1__initial_schema.sql, V2__label_schema.sql, etc.)

- [ ] **Step 3.2: Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-test runtime
```

Expected: all tests pass. The `WorkItemsMigrationCustomizer` adds `classpath:db/work/migration` to Flyway's location list at startup, so Quarkus discovers and applies all migrations from the new path.

If tests fail with `Table not found`: the customizer CDI bean is not being picked up. Check that the runtime JAR is correctly Jandex-indexed — ensure `io.quarkus:quarkus-flyway` is a non-test dependency in `runtime/pom.xml` (already confirmed present).

- [ ] **Step 3.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add -A runtime/src/main/resources/
git -C /Users/mdproctor/claude/casehub/work commit -m "chore(runtime): move migrations db/migration → db/work/migration

Refs #229"
```

---

## Task 4: Install runtime; move ai module migrations and verify

**Files:**
- Move dir: `ai/src/main/resources/db/migration/` → `ai/src/main/resources/db/work/migration/`

- [ ] **Step 4.1: Install runtime to local Maven repo**

The ai module depends on runtime at test time. Install so the customizer class is available:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-install runtime
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4.2: Move ai migrations**

```bash
mkdir -p /Users/mdproctor/claude/casehub/work/ai/src/main/resources/db/work
git -C /Users/mdproctor/claude/casehub/work mv \
  ai/src/main/resources/db/migration \
  ai/src/main/resources/db/work/migration
```

Verify:

```bash
ls /Users/mdproctor/claude/casehub/work/ai/src/main/resources/db/work/migration/
```

Expected: `V14__worker_skill_profile.sql  V4001__escalation_summary.sql`

- [ ] **Step 4.3: Run ai tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-test ai
```

Expected: all tests pass.

- [ ] **Step 4.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add -A ai/src/main/resources/
git -C /Users/mdproctor/claude/casehub/work commit -m "chore(ai): move migrations db/migration → db/work/migration

Refs #229"
```

---

## Task 5: Move ledger migrations, update ledger test properties, verify

**Files:**
- Move dir: `ledger/src/main/resources/db/migration/` → `ledger/src/main/resources/db/work/migration/`
- Modify: `ledger/src/test/resources/application.properties`

- [ ] **Step 5.1: Install ai to local Maven repo** (ledger depends on runtime; install ensures clean state)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-install ai
```

- [ ] **Step 5.2: Move ledger migrations**

```bash
mkdir -p /Users/mdproctor/claude/casehub/work/ledger/src/main/resources/db/work
git -C /Users/mdproctor/claude/casehub/work mv \
  ledger/src/main/resources/db/migration \
  ledger/src/main/resources/db/work/migration
```

Verify:

```bash
ls /Users/mdproctor/claude/casehub/work/ledger/src/main/resources/db/work/migration/
```

Expected: `V2001__work_item_ledger_entry.sql`

- [ ] **Step 5.3: Update ledger test application.properties**

In `ledger/src/test/resources/application.properties`, change:

```properties
quarkus.flyway.locations=db/migration,db/ledger/migration
```

to:

```properties
quarkus.flyway.locations=db/ledger/migration
```

The `db/work/migration` entry is no longer needed — `WorkItemsMigrationCustomizer` adds it automatically. The `db/ledger/migration` entry must remain until casehub-ledger adopts its own `FlywayConfigurationCustomizer`.

- [ ] **Step 5.4: Run ledger tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-test ledger
```

Expected: all tests pass.

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  ledger/src/main/resources/ \
  ledger/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/work commit -m "chore(ledger): move migrations db/migration → db/work/migration

Refs #229"
```

---

## Task 6: Move notifications migrations, update test properties, verify

**Files:**
- Move dir: `notifications/src/main/resources/db/migration/` → `notifications/src/main/resources/db/work/migration/`
- Modify: `notifications/src/test/resources/application.properties`

- [ ] **Step 6.1: Move notifications migrations**

```bash
mkdir -p /Users/mdproctor/claude/casehub/work/notifications/src/main/resources/db/work
git -C /Users/mdproctor/claude/casehub/work mv \
  notifications/src/main/resources/db/migration \
  notifications/src/main/resources/db/work/migration
```

Verify:

```bash
ls /Users/mdproctor/claude/casehub/work/notifications/src/main/resources/db/work/migration/
```

Expected: `V3000__notification_rules.sql`

- [ ] **Step 6.2: Update notifications test application.properties**

In `notifications/src/test/resources/application.properties`, remove the line:

```properties
quarkus.flyway.locations=db/migration
```

The `WorkItemsMigrationCustomizer` adds `db/work/migration` automatically; no explicit location config is needed.

- [ ] **Step 6.3: Run notifications tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-test notifications
```

Expected: all tests pass.

- [ ] **Step 6.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  notifications/src/main/resources/ \
  notifications/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/work commit -m "chore(notifications): move migrations db/migration → db/work/migration

Refs #229"
```

---

## Task 7: Move queues migrations, update queues and queues-dashboard properties, verify

**Files:**
- Move dir: `queues/src/main/resources/db/migration/` → `queues/src/main/resources/db/work/migration/`
- Modify: `queues/src/test/resources/application.properties`
- Modify: `queues-dashboard/src/main/resources/application.properties`

- [ ] **Step 7.1: Move queues migrations**

```bash
mkdir -p /Users/mdproctor/claude/casehub/work/queues/src/main/resources/db/work
git -C /Users/mdproctor/claude/casehub/work mv \
  queues/src/main/resources/db/migration \
  queues/src/main/resources/db/work/migration
```

Verify:

```bash
ls /Users/mdproctor/claude/casehub/work/queues/src/main/resources/db/work/migration/
```

Expected: `V2000__queues_schema.sql  V2002__queue_membership_tracking.sql`

- [ ] **Step 7.2: Update queues test application.properties**

In `queues/src/test/resources/application.properties`, remove the line:

```properties
quarkus.flyway.locations=db/migration
```

- [ ] **Step 7.3: Update queues-dashboard application.properties**

In `queues-dashboard/src/main/resources/application.properties`, remove the line:

```properties
quarkus.flyway.locations=db/migration
```

- [ ] **Step 7.4: Run queues tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-test queues
```

Expected: all tests pass.

- [ ] **Step 7.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  queues/src/main/resources/ \
  queues/src/test/resources/application.properties \
  queues-dashboard/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/work commit -m "chore(queues): move migrations db/migration → db/work/migration

Refs #229"
```

---

## Task 8: Move issue-tracker migrations and verify

**Files:**
- Move dir: `issue-tracker/src/main/resources/db/migration/` → `issue-tracker/src/main/resources/db/work/migration/`

- [ ] **Step 8.1: Move issue-tracker migrations**

```bash
mkdir -p /Users/mdproctor/claude/casehub/work/issue-tracker/src/main/resources/db/work
git -C /Users/mdproctor/claude/casehub/work mv \
  issue-tracker/src/main/resources/db/migration \
  issue-tracker/src/main/resources/db/work/migration
```

Verify:

```bash
ls /Users/mdproctor/claude/casehub/work/issue-tracker/src/main/resources/db/work/migration/
```

Expected: `V5000__issue_link_schema.sql  V5001__priority_rename.sql`

- [ ] **Step 8.2: Run issue-tracker tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/mvn-test issue-tracker
```

Expected: all tests pass.

- [ ] **Step 8.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add issue-tracker/src/main/resources/
git -C /Users/mdproctor/claude/casehub/work commit -m "chore(issue-tracker): move migrations db/migration → db/work/migration

Refs #229"
```

---

## Task 9: Update examples and integration-tests properties, verify

**Files:**
- Modify: `examples/src/test/resources/application.properties`
- Modify: `integration-tests/src/main/resources/application.properties`

- [ ] **Step 9.1: Update examples test application.properties**

In `examples/src/test/resources/application.properties`, change:

```properties
quarkus.flyway.locations=db/migration,db/ledger/migration
```

to:

```properties
quarkus.flyway.locations=db/ledger/migration
```

The `db/work/migration` entry is handled by the customizer.

- [ ] **Step 9.2: Update integration-tests application.properties**

In `integration-tests/src/main/resources/application.properties`, add this line:

```properties
quarkus.flyway.locations=classpath:db/work/migration
```

This is required for native image (Quarkus's Flyway `FlywayProcessor.build()` pre-registers migration files at build time from `quarkus.flyway.locations`; the runtime customizer runs too late for native image). Setting it unconditionally also validates the customizer's idempotency guard in JVM mode (the customizer sees `db/work/migration` already present and skips adding a duplicate).

- [ ] **Step 9.3: Run integration-tests (JVM mode)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests
```

Expected: all `@QuarkusIntegrationTest` tests pass.

- [ ] **Step 9.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  examples/src/test/resources/application.properties \
  integration-tests/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/work commit -m "chore(config): remove db/migration location refs, add db/work/migration to integration-tests

Refs #229"
```

---

## Task 10: Clean up — empty core dir, root DBA scripts

**Files:**
- Delete dir: `core/src/main/resources/db/migration/` (empty)
- Move dir: `db/migration/` (root) → `db/work/migration/`

- [ ] **Step 10.1: Remove the empty core migration directory**

```bash
find /Users/mdproctor/claude/casehub/work/core/src/main/resources/db/migration -type f 2>/dev/null && echo "has files" || echo "empty"
```

If empty (expected), remove:

```bash
rmdir /Users/mdproctor/claude/casehub/work/core/src/main/resources/db/migration
rmdir /Users/mdproctor/claude/casehub/work/core/src/main/resources/db 2>/dev/null || true
```

If it contained a `.gitkeep`:

```bash
git -C /Users/mdproctor/claude/casehub/work rm core/src/main/resources/db/migration/.gitkeep
```

- [ ] **Step 10.2: Move root DBA reference scripts**

These files (`db/migration/V1__initial_schema.sql`, `db/migration/V1000–V1004` scripts) are DBA reference copies — not on any classpath. Moving them to `db/work/migration/` at the project root keeps naming consistent.

```bash
mkdir -p /Users/mdproctor/claude/casehub/work/db/work
git -C /Users/mdproctor/claude/casehub/work mv db/migration db/work/migration
```

Add a README to make the reference-only nature explicit:

```bash
cat > /Users/mdproctor/claude/casehub/work/db/work/migration/README.md << 'EOF'
# DBA Reference Scripts

These are reference copies of key schema migrations for manual review by DBAs.
They are NOT on any module classpath and are NOT run by Flyway automatically.

Runtime Flyway migrations live in each module's src/main/resources/db/work/migration/.
EOF
git -C /Users/mdproctor/claude/casehub/work add db/work/migration/README.md
```

- [ ] **Step 10.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add -A db/ core/src/main/resources/
git -C /Users/mdproctor/claude/casehub/work commit -m "chore: remove empty core migration dir, rename root DBA scripts to db/work/migration

Refs #229"
```

---

## Task 11: Write protocol, update PLATFORM.md and project docs

**Files:**
- Create: `~/claude/casehub/parent/docs/protocols/universal/flyway-extension-migration-registration.md`
- Modify: `~/claude/casehub/parent/docs/PLATFORM.md`
- Modify: `docs/FLYWAY.md`
- Modify: `docs/GOTCHAS.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 11.1: Write the protocol file**

```bash
cat > /Users/mdproctor/claude/casehub/parent/docs/protocols/universal/flyway-extension-migration-registration.md << 'PROTOCOL'
---
id: PP-20260528-flyway-ext-reg
title: "Quarkus extensions must self-register their Flyway migration path"
type: rule
scope: universal
applies_to: "Any Quarkus extension in the casehubio ecosystem that ships Flyway migrations"
severity: high
refs:
  - PP-20260525-607b33 (flyway-repo-scoped-migration-path)
  - ../../PLATFORM.md (Implementation Protocols table)
violation_hint: "Consumer must manually add classpath:db/<repo>/migration to quarkus.flyway.locations — brittle, undiscoverable, breaks silently when a new consumer is onboarded"
created: 2026-05-28
---

Quarkus extensions that ship Flyway migrations must implement all three of the following:

## 1. Repo-scoped migration path

Migrations live at `src/main/resources/db/<repo>/migration/` and are served at
`classpath:db/<repo>/migration`. Never `classpath:db/migration` (the Quarkus default)
— that path is scanned transitively across all JARs on the classpath, causing version
collisions when multiple casehubio modules are co-deployed. See PP-20260525-607b33.

## 2. FlywayConfigurationCustomizer CDI bean (JVM self-registration)

The extension's **runtime module** ships an `@ApplicationScoped` CDI bean implementing
`io.quarkus.flyway.FlywayConfigurationCustomizer`. The `customize(FluentConfiguration)`
implementation adds `classpath:db/<repo>/migration` to the existing configured locations —
additively and idempotently:

```java
@ApplicationScoped
public class <Repo>MigrationCustomizer implements FlywayConfigurationCustomizer {

    @Override
    public void customize(FluentConfiguration configuration) {
        Location target = new Location("classpath:db/<repo>/migration");
        List<Location> locations = new ArrayList<>(Arrays.asList(configuration.getLocations()));
        // Location.getPath() strips the classpath: prefix — comparison is prefix-agnostic
        if (locations.stream().noneMatch(l -> l.getPath().equals(target.getPath()))) {
            locations.add(target);
            configuration.locations(locations.toArray(new Location[0]));
        }
    }
}
```

**Multi-datasource behaviour (confirmed from FlywayContainerProducer bytecode):**
An unqualified `FlywayConfigurationCustomizer` bean (no `@FlywayDataSource` annotation)
applies **only to the default datasource** — Quarkus routes unqualified customizers via
`DataSourceUtil.isDefault(datasourceName)` at lines 175–184 of `FlywayContainerProducer`.
No `@FlywayDataSource` qualifier is needed for extensions that use the default datasource.

**JVM mode:** The CDI bean is discovered automatically — it is a first-class
`@ApplicationScoped` bean in the extension runtime module, which is always Jandex-indexed
as part of the Quarkus extension contract. No `@BuildStep AdditionalBeans` registration
is needed.

## 3. NativeImageResourcePatternsBuildItem (native image resource registration)

The extension's **deployment module** produces a `@BuildStep` registering SQL resources:

```java
@BuildStep
NativeImageResourcePatternsBuildItem registerMigrationResources() {
    return new NativeImageResourcePatternsBuildItem("db/<repo>/migration/.*\\.sql");
}
```

## Native image limitation

`FlywayConfigurationCustomizer` runs at JVM startup — after the native image's Flyway
file registry is frozen by `FlywayProcessor.build()`. Native-image consumers must
explicitly configure the migration path at **build time**:

```properties
quarkus.flyway.locations=classpath:db/<repo>/migration
```

The `NativeImageResourcePatternsBuildItem` ensures the SQL bytes are present in the
native binary but does not populate the build-time file registry. Both steps are needed
for native image support.

## Reference implementation

casehub-work: `WorkItemsMigrationCustomizer` (runtime), `WorkItemsProcessor` (deployment).
PROTOCOL
```

- [ ] **Step 11.2: Update PLATFORM.md Implementation Protocols table**

In `~/claude/casehub/parent/docs/PLATFORM.md`, in the Implementation Protocols table, add a row after the existing Flyway migration rules row:

Find the line:
```
| [Flyway migration rules](protocols/flyway-migration-rules.md) | Version namespace ranges; `MODE=PostgreSQL` in all H2 test URLs |
```

Add after it:
```
| [Flyway extension self-registration](protocols/universal/flyway-extension-migration-registration.md) | Extensions implement `FlywayConfigurationCustomizer` to self-register `classpath:db/<repo>/migration`; `NativeImageResourcePatternsBuildItem` for native; native consumers must also set `quarkus.flyway.locations` at build time |
```

Also update the Persistence table entry for WorkItem tables. Find:
```
| WorkItem tables | `casehub-work` runtime | Flyway V1–V999 at `classpath:db/work/migration` (migration pending — see casehubio/work#229) |
```

Change to:
```
| WorkItem tables | `casehub-work` runtime | Flyway V1–V999 at `classpath:db/work/migration`; auto-registered via `WorkItemsMigrationCustomizer` |
```

- [ ] **Step 11.3: Update docs/FLYWAY.md**

Replace the section header and location references throughout. The key changes:

1. At the top of the "casehub-ledger Prerequisites" section, update the code block:

Find:
```properties
quarkus.flyway.locations=db/migration,db/ledger/migration
```

Replace with:
```properties
quarkus.flyway.locations=db/ledger/migration
```

And update the surrounding prose to say: "casehub-work migrations are now at `classpath:db/work/migration` and are auto-registered by `WorkItemsMigrationCustomizer`. Only the ledger path needs to be declared explicitly — until casehub-ledger adopts its own `FlywayConfigurationCustomizer`."

2. Add a new section after "Version Range Ownership":

```markdown
## Migration Path and Self-Registration

casehub-work migrations live at `classpath:db/work/migration` (per platform protocol
PP-20260525-607b33). The extension self-registers this path via `WorkItemsMigrationCustomizer`
(`@ApplicationScoped FlywayConfigurationCustomizer` in the runtime module).

**JVM mode:** Automatic. No `quarkus.flyway.locations` configuration needed for casehub-work.

**Native image mode:** The customizer runs too late for native image (Quarkus pre-registers
migration files at build time). Native-image consumers must add to build-time config:
```properties
quarkus.flyway.locations=classpath:db/work/migration
```
```

- [ ] **Step 11.4: Update docs/GOTCHAS.md**

Find the existing classpath collision gotcha entry (references casehub-work V1–V21 conflicts with casehub-qhorus). Update it to note the resolution:

Add after the existing entry:

```
**Resolution (work#229):** casehub-work migrations moved to `classpath:db/work/migration`.
`WorkItemsMigrationCustomizer` auto-registers this path in JVM mode. Native image
consumers must configure `quarkus.flyway.locations=classpath:db/work/migration` at
build time (see docs/FLYWAY.md).
```

- [ ] **Step 11.5: Update docs/DESIGN.md**

In the Flyway history section of DESIGN.md, add an entry for this migration path change. Find the Flyway section and add:

```markdown
**work#229 (2026-05-28):** Renamed all module migration paths from `db/migration/` to
`db/work/migration/`. Added `WorkItemsMigrationCustomizer` (JVM self-registration) and
`NativeImageResourcePatternsBuildItem` (native image resource registration). Protocol
`PP-20260528-flyway-ext-reg` written to casehub-parent.
```

- [ ] **Step 11.6: Commit docs and protocol**

```bash
git -C /Users/mdproctor/claude/casehub/work add \
  docs/FLYWAY.md \
  docs/GOTCHAS.md \
  docs/DESIGN.md
git -C /Users/mdproctor/claude/casehub/work commit -m "docs: update FLYWAY.md, GOTCHAS.md, DESIGN.md for db/work/migration rename

Refs #229"

git -C /Users/mdproctor/claude/casehub/parent add \
  docs/protocols/universal/flyway-extension-migration-registration.md \
  docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "protocol: add flyway-extension-migration-registration (PP-20260528)

Quarkus extensions must implement FlywayConfigurationCustomizer to self-register
their repo-scoped migration path. Reference implementation: casehub-work.

Refs casehubio/work#229"
```

---

## Task 12: File casehub-ledger GitHub issue

- [ ] **Step 12.1: Create the issue**

```bash
gh issue create \
  --repo casehubio/ledger \
  --title "feat: implement FlywayConfigurationCustomizer for classpath:db/ledger/migration (PP-20260528)" \
  --body "## Context

Platform protocol PP-20260528 (flyway-extension-migration-registration) requires all
casehubio Quarkus extensions that ship Flyway migrations to self-register their path
via \`FlywayConfigurationCustomizer\`, so consumers never need to manually configure
\`quarkus.flyway.locations\`.

casehub-work adopted this pattern in work#229 — \`WorkItemsMigrationCustomizer\` now
auto-registers \`classpath:db/work/migration\` for JVM-mode consumers.

## What to implement

In the \`casehub-ledger\` runtime module, add:

\`\`\`java
@ApplicationScoped
public class LedgerMigrationCustomizer implements FlywayConfigurationCustomizer {
    @Override
    public void customize(FluentConfiguration configuration) {
        Location target = new Location(\"classpath:db/ledger/migration\");
        List<Location> locations = new ArrayList<>(Arrays.asList(configuration.getLocations()));
        if (locations.stream().noneMatch(l -> l.getPath().equals(target.getPath()))) {
            locations.add(target);
            configuration.locations(locations.toArray(new Location[0]));
        }
    }
}
\`\`\`

In the deployment module, add:
\`\`\`java
@BuildStep
NativeImageResourcePatternsBuildItem registerMigrationResources() {
    return new NativeImageResourcePatternsBuildItem(\"db/ledger/migration/.*\\\\.sql\");
}
\`\`\`

## Impact when done

casehub-work consumers currently declare:
\`\`\`properties
quarkus.flyway.locations=db/ledger/migration
\`\`\`
in their test \`application.properties\` (examples, ledger module). Once casehub-ledger
ships this customizer, those explicit declarations can be removed.

## Native image note

Same limitation applies — native-image consumers must still configure
\`quarkus.flyway.locations\` at build time. See protocol PP-20260528 for details.

Refs: casehubio/work#229, protocol PP-20260528-flyway-ext-reg"
```

---

## Task 13: Final verification — run all affected modules

- [ ] **Step 13.1: Run check-build across all affected modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) scripts/check-build runtime ai ledger notifications queues issue-tracker
```

Expected: all pass.

- [ ] **Step 13.2: Verify no stale db/migration references in source**

```bash
grep -r "db/migration" \
  /Users/mdproctor/claude/casehub/work \
  --include="*.java" \
  --include="*.properties" \
  --include="*.yaml" \
  --include="*.yml" \
  | grep -v target \
  | grep -v "db/work/migration" \
  | grep -v "db/ledger/migration" \
  | grep -v ".md"
```

Expected: no output. Any remaining `db/migration` reference (excluding `db/work/migration` and `db/ledger/migration`) is a missed update.

- [ ] **Step 13.3: Verify db/work/migration directories exist in all expected modules**

```bash
find /Users/mdproctor/claude/casehub/work -path "*/src/main/resources/db/work/migration" -type d | sort
```

Expected output:
```
.../ai/src/main/resources/db/work/migration
.../issue-tracker/src/main/resources/db/work/migration
.../ledger/src/main/resources/db/work/migration
.../notifications/src/main/resources/db/work/migration
.../queues/src/main/resources/db/work/migration
.../runtime/src/main/resources/db/work/migration
```

- [ ] **Step 13.4: Verify old db/migration directories are gone from all modules**

```bash
find /Users/mdproctor/claude/casehub/work \
  -path "*/src/main/resources/db/migration" \
  -not -path "*/target/*" \
  -type d 2>/dev/null
```

Expected: no output (all old directories removed).
