# Extract REST Endpoints into casehub-work-rest Module — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #292 — feat: extract REST endpoints into casehub-work-rest module
**Issue group:** #292

**Goal:** Extract all REST endpoints from `casehub-work` runtime into a new `casehub-work-rest` plain JAR module, making REST opt-in via explicit dependency inclusion.

**Architecture:** New `rest/` module (artifactId `casehub-work-rest`) — plain JAR with Jandex index, depends on `casehub-work` runtime. 32 production files move from `runtime/api/` + `runtime/filter/FilterRuleResource` to `rest/` with package rename `io.casehub.work.runtime.api` → `io.casehub.work.rest`. `AuditEntryResponse` moves to `api/` as a cross-cutting read model. 32 test files follow. Follows the `rest-adapter-module` protocol (`casehub/garden/docs/protocols/universal/rest-adapter-module.md`).

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven, Jandex, JAX-RS, H2 (test)

## Global Constraints

- Java 21 source, compiled on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- GroupId: `io.casehub`, version: `0.2-SNAPSHOT`
- Folder naming: `rest/` (not `casehub-work-rest/`) per `maven-submodule-folder-naming` protocol
- Use `mvn` not `./mvnw`
- Always target specific modules with `-pl <module>` — never run a full build
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all file moves, renames, and reference lookups — never bash grep
- After creating `rest/pom.xml`, run `ide_reload_project` so IntelliJ indexes the new module before using IDE move/rename
- Spec: `~/claude/public/casehub/work/specs/issue-292-extract-rest-module/2026-07-05-extract-rest-module-design.md`
- Protocol: `casehub/garden/docs/protocols/universal/rest-adapter-module.md`

---

### Task 1: Create rest/ module scaffold and move AuditEntryResponse to api/

**Why first:** The rest/ module must exist before files can move there. AuditEntryResponse must move to api/ before the runtime/api/ package is emptied — it's a cross-cutting read model used by 24 files in examples/ that should NOT depend on rest/.

**Files:**
- Create: `rest/pom.xml`
- Modify: `pom.xml` (parent — add `<module>rest</module>`)
- Move: `runtime/.../api/AuditEntryResponse.java` → `api/.../api/AuditEntryResponse.java`
- Modify: ~24 files in `examples/` (import rewrite: `io.casehub.work.runtime.api.AuditEntryResponse` → `io.casehub.work.api.AuditEntryResponse`)

**Produces:**
- Empty `rest/` module that compiles
- `AuditEntryResponse` in `api/` package `io.casehub.work.api`

- [ ] **Step 1: Create `rest/pom.xml`**

Create `rest/pom.xml` with:
- Parent: `casehub-work-parent` `0.2-SNAPSHOT`
- ArtifactId: `casehub-work-rest`
- Compile deps: `casehub-work` (`${project.version}`), `quarkus-rest-jackson` (provided)
- Test deps: `quarkus-junit5`, `rest-assured`, `casehub-platform` (runtime scope), `quarkus-jdbc-h2`, `assertj-core`, `quarkus-micrometer-registry-prometheus` — all test scope
- Build plugins: `jandex-maven-plugin` (make-index), `quarkus-maven-plugin` (build, generate-code, generate-code-tests), `maven-failsafe-plugin` (integration-test, verify)

Mirror structure from `integration-tests-memory/pom.xml` for test infrastructure. Jandex plugin version inherited from parent `pluginManagement` (3.1.2).

- [ ] **Step 2: Add rest module to parent pom**

Insert `<module>rest</module>` in parent `pom.xml` between `queues-postgres-broadcaster` and `integration-tests` (after line 36 in the modules list).

- [ ] **Step 3: Reload Maven project in IntelliJ**

Run `ide_reload_project` so IntelliJ indexes the new rest/ module.

- [ ] **Step 4: Move AuditEntryResponse to api/**

Use `ide_move_file` to move `runtime/src/main/java/io/casehub/work/runtime/api/AuditEntryResponse.java` to `api/src/main/java/io/casehub/work/api/`. IntelliJ will update the package declaration and all import references automatically (including the 24 examples/ files).

Verify with `ide_find_references` that no `io.casehub.work.runtime.api.AuditEntryResponse` imports remain.

- [ ] **Step 5: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,runtime,examples -q
```

- [ ] **Step 6: Commit**

```
feat(#292): create rest/ module scaffold and move AuditEntryResponse to api/

AuditEntryResponse is a cross-cutting read model (plain record, no JAX-RS
annotations) used by 24 files in examples/. Moving it to api/ keeps the
dependency graph honest — examples/ depends on api/ (which it already does),
not on the new rest/ module.

Refs #292
```

---

### Task 2: Move production files from runtime/ to rest/

**Why this order:** Production code moves before tests. All 31 remaining files in `runtime/api/` plus `FilterRuleResource` from `runtime/filter/` move to `rest/` with package rename to `io.casehub.work.rest`. Downstream consumers (queues/, integration-tests/, integration-tests-memory/) get updated dependencies and imports.

**Files:**
- Move: 31 files from `runtime/.../api/` → `rest/.../rest/` (package rename)
- Move: `runtime/.../filter/FilterRuleResource.java` → `rest/.../rest/`
- Modify: `queues/pom.xml` (add casehub-work-rest dep)
- Modify: `queues/.../QueueResource.java`, `QueueStateResource.java` (import rewrite)
- Modify: `integration-tests/pom.xml` (add casehub-work-rest dep)
- Modify: `integration-tests-memory/pom.xml` (add casehub-work-rest dep)
- Modify: `api/.../DeclineTarget.java` (fix Javadoc @link)
- Modify: `runtime/.../model/WorkItemRelation.java` (fix Javadoc @link)

**Consumes:** rest/ module from Task 1, AuditEntryResponse in api/ from Task 1
**Produces:** All REST resources in `io.casehub.work.rest`, runtime/ has no REST code

- [ ] **Step 1: Create target package directory**

Create `rest/src/main/java/io/casehub/work/rest/` directory structure.

- [ ] **Step 2: Move all 31 files from runtime/api/ to rest/**

Use `ide_move_file` for each file (resources, request DTOs, response DTOs, exception mappers, WorkItemMapper) from `runtime/src/main/java/io/casehub/work/runtime/api/` to `rest/src/main/java/io/casehub/work/rest/`.

IntelliJ handles the package rename and internal cross-references automatically. The files:

Resources (11): WorkItemResource, WorkItemBulkResource, WorkItemSpawnResource, WorkItemTemplateResource, WorkItemScheduleResource, WorkItemInstancesResource, WorkItemRelationResource, SpawnGroupResource, AuditResource, VocabularyResource, AsyncApiResource

Request DTOs (11): CreateWorkItemRequest, CompleteRequest, CancelRequest, DelegateRequest, EscalateRequest, ExtendRequest, FaultRequest, ObsoleteRequest, ProgressRequest, RejectRequest, SuspendRequest

Response DTOs (3): WorkItemResponse, WorkItemWithAuditResponse, WorkItemLabelResponse

Exception Mappers (5): WorkItemNotFoundExceptionMapper, IllegalStateExceptionMapper, MalformedCapabilityExceptionMapper, UnknownCapabilityExceptionMapper, OptimisticLockExceptionMapper

Utilities (1): WorkItemMapper

- [ ] **Step 3: Move FilterRuleResource from runtime/filter/**

Use `ide_move_file` to move `runtime/src/main/java/io/casehub/work/runtime/filter/FilterRuleResource.java` to `rest/src/main/java/io/casehub/work/rest/`.

- [ ] **Step 4: Add casehub-work-rest dependency to queues/**

Add to `queues/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-rest</artifactId>
    <version>${project.version}</version>
</dependency>
```

Verify QueueResource and QueueStateResource imports updated to `io.casehub.work.rest.WorkItemMapper` and `io.casehub.work.rest.WorkItemResponse` (IDE move should have done this; confirm with `ide_find_references`).

- [ ] **Step 5: Add casehub-work-rest dependency to integration-tests/ and integration-tests-memory/**

Add the same `casehub-work-rest` dependency to both `integration-tests/pom.xml` and `integration-tests-memory/pom.xml`.

- [ ] **Step 6: Fix Javadoc references**

1. `api/src/main/java/io/casehub/work/api/DeclineTarget.java` line 15: replace `{@link io.casehub.work.runtime.api.DelegateRequest#declineTarget()}` with plain text `DelegateRequest.declineTarget()` (in the REST module)

2. `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemRelation.java` line 39: replace `{@link io.casehub.work.runtime.api.WorkItemRelationResource}` with plain text `WorkItemRelationResource` (in the REST module)

- [ ] **Step 7: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,runtime,rest,queues -q
```

- [ ] **Step 8: Commit**

```
feat(#292): move 32 REST production files to rest/ module

Moves all JAX-RS resources, request/response DTOs, exception mappers,
and WorkItemMapper from runtime/api/ to rest/ (package io.casehub.work.rest).
FilterRuleResource moves from runtime/filter/.

Updates queues/, integration-tests/, integration-tests-memory/ dependencies.
Fixes two Javadoc @link references in DeclineTarget and WorkItemRelation.

Refs #292
```

---

### Task 3: Move test files and test infrastructure to rest/

**Why this order:** Tests follow production code. Test utilities (MutableCurrentPrincipal, MutablePrincipalResetCallback) must be duplicated because Maven doesn't share test sources transitively. Test resources (application.properties, META-INF/services) must be copied and adapted.

**Files:**
- Create: `rest/src/test/java/io/casehub/work/rest/test/MutableCurrentPrincipal.java`
- Create: `rest/src/test/java/io/casehub/work/rest/test/MutablePrincipalResetCallback.java`
- Create: `rest/src/test/resources/application.properties`
- Create: `rest/src/test/resources/META-INF/services/io.quarkus.test.junit.callback.QuarkusTestBeforeEachCallback`
- Move: 32 test files from `runtime/.../api/` → `rest/.../rest/`

**Consumes:** rest/ production code from Task 2
**Produces:** All 32 tests running in rest/ module

- [ ] **Step 1: Create test utilities**

Duplicate `MutableCurrentPrincipal` and `MutablePrincipalResetCallback` from `runtime/src/test/java/io/casehub/work/runtime/test/` to `rest/src/test/java/io/casehub/work/rest/test/`, updating the package declaration to `io.casehub.work.rest.test`.

Source files: read from `runtime/src/test/java/io/casehub/work/runtime/test/MutableCurrentPrincipal.java` and `MutablePrincipalResetCallback.java`. This duplication follows the established codebase pattern — the same classes are already independently duplicated in `queues/src/test/` and `persistence-mongodb/src/test/`.

- [ ] **Step 2: Create test resources**

Copy `runtime/src/test/resources/application.properties` to `rest/src/test/resources/application.properties` verbatim.

Create `rest/src/test/resources/META-INF/services/io.quarkus.test.junit.callback.QuarkusTestBeforeEachCallback` with content:
```
io.casehub.work.rest.test.MutablePrincipalResetCallback
```
(Note: must reference the LOCAL package `io.casehub.work.rest.test`, not the runtime package.)

- [ ] **Step 3: Move all 32 test files**

Use `ide_move_file` to move each test file from `runtime/src/test/java/io/casehub/work/runtime/api/` to `rest/src/test/java/io/casehub/work/rest/`.

All 32 files: AsyncApiTest, AuditQueryTest, AuditResourceTest, InboxFilterTest, InboxSummaryTest, LabelEndpointTest, MetricsEndpointTest, MultiTenancyIT, OpenApiTest, WorkerSelectionStrategyIT, WorkItemBulkTest, WorkItemCapabilityIT, WorkItemCloneTest, WorkItemDelegationTest, WorkItemExcludedUsersTest, WorkItemExtendTest, WorkItemInstancesResourceTest, WorkItemLabelMapperTest, WorkItemLinkTest, WorkItemNoteTest, WorkItemOptimisticLockTest, WorkItemOutcomeValidationTest, WorkItemRelationTest, WorkItemResourceTest, WorkItemScheduleClusterTest, WorkItemScheduleTest, WorkItemSchemaValidationTest, WorkItemSSETest, WorkItemTemplatePatchTest, WorkItemTemplateOutcomeTest, WorkItemTemplateSchemaTest, WorkItemTemplateTest.

After IDE move, verify MultiTenancyIT's import references `io.casehub.work.rest.test.MutableCurrentPrincipal` (the local copy, not runtime's).

- [ ] **Step 4: Run rest/ tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest
```

All 32 tests must pass (31 @QuarkusTest + 1 plain JUnit). For the 3 *IT.java files, also run:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl rest
```

- [ ] **Step 5: Run runtime/ tests to verify nothing broke**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Remaining runtime tests (services, stores, RLS, multi-instance, etc.) must still pass.

- [ ] **Step 6: Commit**

```
feat(#292): move 32 REST test files to rest/ module

Moves all test files from runtime/api/ to rest/ with test infrastructure:
- MutableCurrentPrincipal + MutablePrincipalResetCallback duplicated
  (follows established pattern — queues/ and persistence-mongodb/ do the same)
- application.properties copied from runtime test resources
- QuarkusTestBeforeEachCallback service file references local package

Refs #292
```

---

### Task 4: Documentation updates and final verification

**Files:**
- Modify: `docs/MODULES.md` (add rest/ row, update runtime/ row)
- Modify: `ARC42STORIES.MD` (§4 layer taxonomy, §5 building block view + module index)

**Consumes:** Complete extraction from Tasks 1-3

- [ ] **Step 1: Update docs/MODULES.md**

Add row to Core Modules table:
```
| `rest/` | JAX-RS REST surface (`casehub-work-rest`) | Plain JAR + Jandex. Resources (12), request/response DTOs, exception mappers, WorkItemMapper. Opt-in via dependency. |
```

Update `runtime/` row — remove "REST endpoints at `/workitems`" from its Purpose column.

- [ ] **Step 2: Update ARC42STORIES.MD**

1. **§4 Layer Taxonomy:** Update L2 description to note REST surface is now in `rest/` module.
2. **§5 Building Block View:** Add `casehub-work-rest` to module index table:

| Folder | Artifact | Type | Purpose |
|--------|----------|------|---------|
| `rest/` | `casehub-work-rest` | Jandex library | JAX-RS resources (12), request/response DTOs, exception mappers, WorkItemMapper; opt-in REST surface |

Update runtime/ row to remove REST from its purpose.

3. **§5 Mermaid diagram:** Add `casehub-work-rest` container showing dependency on `casehub-work`.

- [ ] **Step 3: Full build verification**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl api,runtime,rest,queues,examples,integration-tests-memory
```

- [ ] **Step 4: Commit**

```
docs(#292): update MODULES.md and ARC42STORIES.MD for rest/ module

Adds rest/ to the module map and updates the building block view
to reflect the REST extraction.

Refs #292
```

---

## Verification

After all 4 tasks:

1. `mvn compile -pl rest` — rest/ compiles with all 32 production files
2. `mvn test -pl rest` — all 32 tests pass in rest/
3. `mvn test -pl runtime` — remaining runtime tests still pass
4. `mvn compile -pl queues` — queues/ compiles with updated imports
5. `mvn compile -pl examples` — examples/ compiles with AuditEntryResponse from api/
6. `mvn verify -pl integration-tests-memory` — ephemeral deployment ITs pass
7. `ide_find_references` for `io.casehub.work.runtime.api` — zero remaining references (the old package is gone)
8. `runtime/src/main/java/io/casehub/work/runtime/api/` directory is empty or deleted
