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

## What Moves

All 32 files from `runtime/src/main/java/io/casehub/work/runtime/api/`:

**Resources (11 from runtime/api + 1 from runtime/filter = 12 total):**
- WorkItemResource, WorkItemBulkResource, WorkItemSpawnResource
- WorkItemTemplateResource, WorkItemScheduleResource, WorkItemInstancesResource
- WorkItemRelationResource, SpawnGroupResource, AuditResource
- VocabularyResource, AsyncApiResource
- FilterRuleResource (from runtime/filter/)

**Request DTOs (11):**
- CreateWorkItemRequest, CompleteRequest, CancelRequest, DelegateRequest
- EscalateRequest, ExtendRequest, FaultRequest, ObsoleteRequest
- ProgressRequest, RejectRequest, SuspendRequest

**Response DTOs (4):**
- WorkItemResponse, WorkItemWithAuditResponse, WorkItemLabelResponse, AuditEntryResponse

**Exception Mappers (5):**
- WorkItemNotFoundExceptionMapper, IllegalStateExceptionMapper
- MalformedCapabilityExceptionMapper, UnknownCapabilityExceptionMapper
- OptimisticLockExceptionMapper

**Utilities (1):**
- WorkItemMapper

**Total: 33 production files.**

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

Jandex plugin configured in build to generate the index.

## Downstream Impact

Three internal modules need a new `<dependency>` on `casehub-work-rest`:

| Module | Reason |
|--------|--------|
| `queues/` | QueueResource and QueueStateResource import WorkItemMapper and WorkItemResponse |
| `examples/` | 24 files import AuditEntryResponse for scenario responses |
| `integration-tests/` | Tests hit REST endpoints |

No external consumers are affected — these DTOs are not used outside the casehub-work project.

## Tests

Resource test classes move from `runtime/src/test/` to `rest/src/test/`:
- WorkItemResourceTest, WorkItemBulkResourceTest (if exists), WorkItemSpawnResourceTest
- WorkItemInstancesResourceTest, AuditResourceTest
- Any other `*ResourceTest.java` in `runtime/src/test/.../api/`

These are `@QuarkusTest` classes needing the full CDI container. Test-scoped dependencies on `casehub-work` runtime (and transitive JPA/H2) provide the backing services.

## Parent POM Changes

Add `<module>rest</module>` to the parent `pom.xml` module list.

## Breaking Change

This is a breaking change for existing consumers that depend on `casehub-work` and expect REST endpoints to be auto-registered. After this change, consumers must add an explicit dependency on `casehub-work-rest` to get the REST surface.

## Not in Scope

- REST endpoints in other optional modules (ledger, queues, reports, ai, notifications, issue-tracker) — these are self-contained and stay where they are
- No new REST endpoints or API changes — pure extraction
- No Flyway migrations
