# Extract REST Endpoints into casehub-work-rest Module

**Issue:** casehubio/work#292
**Date:** 2026-07-05

## Summary

Extract all REST endpoints from `casehub-work` runtime into a new `casehub-work-rest` plain JAR module. Consumers that only need the Java SPI (e.g., casehub-engine composing work via CDI) no longer pay the JAX-RS coupling cost. REST becomes opt-in via explicit dependency inclusion.

## Module Shape

- **Folder:** `rest/`
- **ArtifactId:** `casehub-work-rest`
- **Packaging:** JAR (plain — not a Quarkus extension, no deployment module)
- **Package:** `io.casehub.work.rest` (clean break from `io.casehub.work.runtime.api`)
- **Discovery:** Jandex index at build time; Quarkus auto-discovers JAX-RS resources by classpath presence

### Design Rationale

The plain JAR + Jandex approach is justified from first principles:

1. **Plain JAR, not a Quarkus extension:** The REST layer contains no build-time processing, no CDI bean registration via `BuildItem`, and no native-image configuration. It is pure JAX-RS resource classes and DTOs. A Quarkus extension (with a deployment module) is only warranted when build-time augmentation is needed. REST resources discovered via Jandex need none.

2. **Jandex index for discovery:** Quarkus scans the Jandex index of every JAR on the classpath for JAX-RS-annotated classes (`@Path`, `@Provider`). Adding the `jandex-maven-plugin` to `rest/pom.xml` ensures all resource classes and exception mappers are discovered at build time without any explicit registration code.

3. **`provided` scope for `quarkus-rest-jackson`:** The consuming application (or the `casehub-work` extension itself) already brings `quarkus-rest-jackson` as a runtime dependency. Declaring it `provided` in `rest/` avoids version conflicts and duplicate classpath entries while still making the JAX-RS API available at compile time.

## What Moves to rest/

All files from `runtime/src/main/java/io/casehub/work/runtime/api/` except `AuditEntryResponse` (which moves to `api/` — see below), plus `FilterRuleResource` from `runtime/filter/`:

**Resources (12):**
- WorkItemResource, WorkItemBulkResource, WorkItemSpawnResource
- WorkItemTemplateResource, WorkItemScheduleResource, WorkItemInstancesResource
- WorkItemRelationResource, SpawnGroupResource, AuditResource
- VocabularyResource, AsyncApiResource
- FilterRuleResource (from `runtime/filter/`)

**Request DTOs (11):**
- CreateWorkItemRequest, CompleteRequest, CancelRequest, DelegateRequest
- EscalateRequest, ExtendRequest, FaultRequest, ObsoleteRequest
- ProgressRequest, RejectRequest, SuspendRequest

**Response DTOs (3):**
- WorkItemResponse, WorkItemWithAuditResponse, WorkItemLabelResponse

**Exception Mappers (5):**
- WorkItemNotFoundExceptionMapper, IllegalStateExceptionMapper
- MalformedCapabilityExceptionMapper, UnknownCapabilityExceptionMapper
- OptimisticLockExceptionMapper

**Utilities (1):**
- WorkItemMapper

**Total: 32 production files in rest/.**

## What Moves to api/

**AuditEntryResponse** moves to `api/` (package `io.casehub.work.api`), not `rest/`.

`AuditEntryResponse` is a plain Java record with no JAX-RS annotations. It is used as a cross-cutting read model — 24 files in `examples/` construct it directly via `new AuditEntryResponse(...)` to map from domain `AuditEntry` objects, completely bypassing `WorkItemMapper`. Making `examples/` depend on `casehub-work-rest` solely for a data record would create a false REST dependency on a module that is not a REST consumer.

Moving it to `api/` keeps the dependency graph honest: `examples/` depends on `api/` (which it already does) for the read model, and `rest/` imports `AuditEntryResponse` from `api/` (which it can, since `rest/` → `runtime/` → `api/`).

## What Stays in Runtime

- Services (`runtime/service/`)
- Stores and JPA entities (`runtime/repository/`, `runtime/model/`)
- CDI events, lifecycle emitters, CloudEvent adapter (`runtime/event/`)
- Filter engine (except FilterRuleResource) (`runtime/filter/`)
- Multi-instance coordinator, calendar, config
- Deployment module (`deployment/`) — unchanged

## Dependencies (rest/pom.xml)

```xml
<dependencies>
    <!-- Runtime module — services, stores, domain types -->
    <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-work</artifactId>
    </dependency>

    <!-- JAX-RS + JSON binding (provided by Quarkus at runtime) -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-rest-jackson</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

**Build plugins (production):**
```xml
<!-- Jandex index — Quarkus scans this to discover JAX-RS resources at build time.
     Without this plugin, the rest/ JAR has no index and all endpoints return 404. -->
<plugin>
    <groupId>io.smallrye</groupId>
    <artifactId>jandex-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>make-index</id>
            <goals><goal>jandex</goal></goals>
        </execution>
    </executions>
</plugin>
```
Version inherited from parent `pluginManagement`.

## Downstream Impact

Four internal modules need updates after extraction:

| Module | Change | Reason |
|--------|--------|--------|
| `queues/` | Add `<dependency>` on `casehub-work-rest` | QueueResource and QueueStateResource import WorkItemMapper and WorkItemResponse |
| `examples/` | Update imports from `io.casehub.work.runtime.api.AuditEntryResponse` to `io.casehub.work.api.AuditEntryResponse` | AuditEntryResponse moves to `api/`; no new module dependency needed (examples/ already depends on api/ transitively) |
| `integration-tests/` | Add `<dependency>` on `casehub-work-rest` | Tests hit REST endpoints |
| `integration-tests-memory/` | Add `<dependency>` on `casehub-work-rest` | `EphemeralDeploymentIT` hits REST endpoints (`/workitems` CRUD, lifecycle, inbox); without `casehub-work-rest` on classpath, all requests return 404 |

### External Consumer Verification

The following consumer repos were verified to have **zero** imports from `io.casehub.work.runtime.api`:

| Consumer | Verified | Result |
|----------|----------|--------|
| `casehub-engine/work-adapter` | Yes (IDE index search) | 0 references |
| `casehub-engine/casehub-engine-inbound` | Yes (IDE index search) | 0 references |
| `casehub-devtown` | No (not in workspace) | Expected 0 — uses only `casehub-work-api` types per architectural contract |
| `casehub-clinical` | No (not in workspace) | Expected 0 — same rationale |
| `casehub-life` | No (not in workspace) | Expected 0 — same rationale |

ASSUMPTION: `casehub-devtown`, `casehub-clinical`, and `casehub-life` do not import types from `io.casehub.work.runtime.api`. Verify via `grep -r "io.casehub.work.runtime.api"` in each repo before merging.

## Package Rename Migration

The package rename from `io.casehub.work.runtime.api` to `io.casehub.work.rest` affects every `import` statement referencing the old package. Verified reference counts within casehub-work:

| Type | Import sites outside runtime/ |
|------|-------------------------------|
| `AuditEntryResponse` | 24 files in `examples/` (→ rewrite to `io.casehub.work.api.AuditEntryResponse`) |
| `WorkItemMapper` | 2 files in `queues/` (→ rewrite to `io.casehub.work.rest.WorkItemMapper`) |
| `WorkItemResponse` | 1 file in `queues/` (→ rewrite to `io.casehub.work.rest.WorkItemResponse`) |

Additionally, all 32 test files in `runtime/src/test/.../api/` move to `rest/src/test/.../rest/` and get new package declarations.

This is mechanical (IDE rename-package refactoring) but the spec acknowledges the scope: it is not just adding a `<dependency>` — it is also a mass import migration across ~27 files outside `runtime/`.

## Tests

All 32 test files move from `runtime/src/test/java/io/casehub/work/runtime/api/` to `rest/src/test/java/io/casehub/work/rest/`:

**@QuarkusTest classes (31):**
- AsyncApiTest, AuditQueryTest, AuditResourceTest, InboxFilterTest, InboxSummaryTest
- LabelEndpointTest, MetricsEndpointTest, OpenApiTest
- MultiTenancyIT, WorkerSelectionStrategyIT, WorkItemCapabilityIT
- WorkItemBulkTest, WorkItemCloneTest, WorkItemDelegationTest
- WorkItemExcludedUsersTest, WorkItemExtendTest, WorkItemInstancesResourceTest
- WorkItemLinkTest, WorkItemNoteTest
- WorkItemOptimisticLockTest, WorkItemOutcomeValidationTest, WorkItemRelationTest
- WorkItemResourceTest, WorkItemScheduleClusterTest, WorkItemScheduleTest
- WorkItemSchemaValidationTest, WorkItemSSETest, WorkItemTemplatePatchTest
- WorkItemTemplateOutcomeTest, WorkItemTemplateSchemaTest, WorkItemTemplateTest

The 3 `*IT.java` files (MultiTenancyIT, WorkerSelectionStrategyIT, WorkItemCapabilityIT) use `@QuarkusTest` despite their suffix — the `IT` naming is a codebase convention, not an annotation indicator. Maven Failsafe is still required to run `*IT.java` files (Surefire skips them by default).

**Plain JUnit test (1):**
- WorkItemLabelMapperTest — no `@QuarkusTest` annotation; tests `WorkItemMapper` label mapping logic directly without CDI

**Service-level tests remaining in runtime/ (1):**
- WorkItemSmokeTest (in `runtime/src/test/.../runtime/`, not in `api/` — stays)

### Test Resource Files

The following test resources must be copied from `runtime/src/test/resources/` to `rest/src/test/resources/`:

1. **`application.properties`** — essential for @QuarkusTest boot:
   - `quarkus.http.test-port=0` (random port allocation)
   - `quarkus.datasource.db-kind=h2` + JDBC URL (H2 datasource)
   - `quarkus.hibernate-orm.database.generation=none` + `quarkus.flyway.migrate-at-start=true` (schema setup)
   - `quarkus.scheduler.start-mode=halted` (prevents @Scheduled auto-fire)
   - `casehub.work.*` config properties (default expiry/claim hours)
   - `quarkus.arc.exclude-types=io.casehub.platform.mock.MockGroupMembershipProvider` (CDI exclusion)

2. **`META-INF/services/io.quarkus.test.junit.callback.QuarkusTestBeforeEachCallback`** — registers `MutablePrincipalResetCallback` to reset the mutable principal between tests. The file content must reference the duplicated class's local package:
   ```
   io.casehub.work.rest.test.MutablePrincipalResetCallback
   ```
   A verbatim copy of runtime's service file (which references `io.casehub.work.runtime.test.MutablePrincipalResetCallback`) would silently fail — the SPI loader skips unresolvable entries, and tests would suffer from tenant state leakage between methods. This matches the established pattern: `queues/` has its own service file referencing `io.casehub.work.queues.test.MutablePrincipalResetCallback`.

Note: `rls-init.sql` stays in `runtime/src/test/resources/` — it is only used by `RlsEnforcementTest` (in `runtime/rls/`), not by any of the 32 API test files.

### Cross-Module Test Utilities

`MultiTenancyIT` imports `io.casehub.work.runtime.test.MutableCurrentPrincipal`, a test utility in `runtime/src/test/java/` — Maven does not share test sources transitively. The `QuarkusTestBeforeEachCallback` service file also references `MutablePrincipalResetCallback` from the same test package.

**Resolution:** Duplicate both classes to `rest/src/test/java/io/casehub/work/rest/test/`:
- `MutableCurrentPrincipal`
- `MutablePrincipalResetCallback`

This follows the established codebase pattern — `MutableCurrentPrincipal` is already independently duplicated in `queues/src/test/` and `persistence-mongodb/src/test/` for the same reason. Each module owns its copy.

Leaving MultiTenancyIT in `runtime/` is not viable: after extraction, the REST endpoints it tests are no longer in runtime/, and adding `casehub-work-rest` as a test dependency on runtime/ would create a Maven reactor cycle (rest/ → runtime/ at compile scope).

### @QuarkusTest Infrastructure (rest/pom.xml)

The `rest/` module needs the following test infrastructure. Configuration mirrors `integration-tests-memory/pom.xml`:

**Test dependencies:**
```xml
<!-- Quarkus test framework -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>

<!-- CDI container + JPA backing (casehub-work extension provides services) -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- H2 datasource for test persistence -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-h2</artifactId>
    <scope>test</scope>
</dependency>

<!-- Assertions -->
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <scope>test</scope>
</dependency>

<!-- Prometheus registry — enables /q/metrics endpoint for MetricsEndpointTest -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
    <scope>test</scope>
</dependency>
```

**Not needed** (verified: none of the 32 API test files import these):
- `mockito-junit-jupiter` — only used by runtime/service/, runtime/rls/, runtime/event/ tests (all stay in runtime/)
- `awaitility` — only used by runtime/multiinstance/ tests (stays in runtime/)
- `testcontainers-postgresql` — only used by runtime/rls/RlsEnforcementTest (stays in runtime/)

**Build plugins:**
```xml
<!-- Required for @QuarkusTest CDI container boot -->
<plugin>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>build</goal>
                <goal>generate-code</goal>
                <goal>generate-code-tests</goal>
            </goals>
        </execution>
    </executions>
</plugin>

<!-- Required for *IT.java files (Maven Surefire skips them by default) -->
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## Javadoc Fixups

Two Javadoc `@link` references break after extraction because the target classes move to `io.casehub.work.rest` and the source files cannot depend on `rest/`:

1. **`api/src/main/java/io/casehub/work/api/DeclineTarget.java` line 15:** `{@link io.casehub.work.runtime.api.DelegateRequest#declineTarget()}` → replace with plain text: `DelegateRequest.declineTarget()` (in the REST module)

2. **`runtime/src/main/java/io/casehub/work/runtime/model/WorkItemRelation.java` line 39:** `{@link io.casehub.work.runtime.api.WorkItemRelationResource}` → replace with plain text: `WorkItemRelationResource` (in the REST module)

## Parent POM Changes

Add `<module>rest</module>` to the parent `pom.xml` module list.

## Documentation Updates

### ARC42STORIES.MD

Three sections require updates when the `rest/` module is added:

1. **Layer Taxonomy (§4):** L2 currently says "WorkItemResource (7 sub-resources), DTOs, exception mappers, OpenAPI". Update to reflect that L2 content now lives in `rest/` as a separate module.

2. **Building Block View (§5):** Add `casehub-work-rest` container to the L1/L2 boundary in the Mermaid diagram, showing its dependency on `casehub-work` (runtime).

3. **Module Index (§5):** Add row:

| Folder | Artifact | Type | Purpose |
|--------|----------|------|---------|
| `rest/` | `casehub-work-rest` | Jandex library | JAX-RS resources (12), request/response DTOs, exception mappers, WorkItemMapper; opt-in REST surface |

Update the `runtime/` row to remove "REST resources (7)" from its purpose description.

## Breaking Change

This is a breaking change for existing consumers that depend on `casehub-work` and expect REST endpoints to be auto-registered. After this change, consumers must add an explicit dependency on `casehub-work-rest` to get the REST surface.

## Not in Scope

- REST endpoints in other optional modules (ledger, queues, reports, ai, notifications, issue-tracker) — these are self-contained and stay where they are
- No new REST endpoints or API changes — pure extraction
- No Flyway migrations
- Renaming `AuditEntryResponse` (it moves to `api/` with its current name; a rename to better reflect its read-model role is a separate concern)
