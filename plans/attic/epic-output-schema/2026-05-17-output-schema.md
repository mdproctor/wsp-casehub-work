# Output Schema Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `inputDataSchema` and `outputDataSchema` to `WorkItemTemplate` (snapshotted onto `WorkItem` at instantiation), validate payload at create and resolution at complete in the service layer, and remove the now-redundant `WorkItemFormSchema` entity and its CRUD API.

**Architecture:** Template-level JSON Schema fields are snapshotted onto `WorkItem` at instantiation (same pattern as `permittedOutcomes`). `FormSchemaValidationService` (existing, retained) moves from resource injection to service injection. `WorkItemFormSchema`, its REST resource, and category-level validation in `WorkItemResource` are deleted — the template is the single source of truth for type-level contracts.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM / Panache, Flyway, `networknt/json-schema-validator` (already on classpath via `FormSchemaValidationService`), REST Assured + JUnit 5 for tests.

---

## Files to Create

| File | Purpose |
|------|---------|
| `runtime/src/main/resources/db/migration/V23__template_data_schemas.sql` | Add `input_data_schema`, `output_data_schema` to `work_item_template` |
| `runtime/src/main/resources/db/migration/V24__workitem_data_schemas.sql` | Add `input_data_schema`, `output_data_schema` to `work_item` |
| `runtime/src/main/resources/db/migration/V25__drop_form_schema.sql` | Drop `work_item_form_schema` table |
| `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemTemplateSchemaTest.java` | Template CRUD + instantiation snapshot |
| `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemSchemaValidationTest.java` | create() payload + complete() resolution validation |

## Files to Delete

| File | Reason |
|------|--------|
| `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemFormSchema.java` | Replaced by template-level schemas |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemFormSchemaResource.java` | Endpoint removed |
| `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemFormSchemaTest.java` | Tests code being removed |
| `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemFormSchemaValidationTest.java` | Tests code being removed |

## Files to Modify

| File | Change |
|------|--------|
| `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemTemplate.java` | Add `inputDataSchema`, `outputDataSchema` TEXT fields |
| `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java` | Add `inputDataSchema`, `outputDataSchema` TEXT fields |
| `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemCreateRequest.java` | Add `inputDataSchema`, `outputDataSchema` params |
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java` | Inject `FormSchemaValidationService`; validate in `create()` and `complete()` |
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java` | Pass schema fields in `toCreateRequest()` |
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemSpawnService.java` | Pass schema fields to `WorkItemCreateRequest` |
| `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java` | Pass schema fields in both request builders |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResource.java` | Remove `FormSchemaValidationService` injection + `WorkItemFormSchema` lookup |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemTemplateResource.java` | Add schema fields to `CreateTemplateRequest` + `toResponse()` |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResponse.java` | Add `inputDataSchema`, `outputDataSchema` |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemWithAuditResponse.java` | Add `inputDataSchema`, `outputDataSchema` |
| `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemMapper.java` | Map new fields in `toResponse()` and `toWithAudit()` |
| `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java` | Add both fields to JEXL map |
| Various test files | Add `null, null` to `WorkItemCreateRequest` constructors |

---

## Task 1: Write failing tests — template schema CRUD and snapshot

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemTemplateSchemaTest.java`

- [ ] **Step 1.1: Write the test class**

```java
package io.casehub.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.notNullValue;
import static org.hamcrest.Matchers.nullValue;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

/**
 * Tests that WorkItemTemplate carries inputDataSchema and outputDataSchema,
 * and that both are snapshotted onto WorkItem at instantiation. Refs #170.
 */
@QuarkusTest
class WorkItemTemplateSchemaTest {

    private static final String OUTPUT_SCHEMA = """
            {"type":"object","required":["decision"],
             "properties":{"decision":{"type":"string"},"notes":{"type":"string"}},
             "additionalProperties":false}
            """;

    private static final String INPUT_SCHEMA = """
            {"type":"object","required":["requestor"],
             "properties":{"requestor":{"type":"string"}},
             "additionalProperties":false}
            """;

    @Test
    void createTemplate_withOutputDataSchema_storesAndReturnsIt() {
        given().contentType(ContentType.JSON)
                .body("{\"name\":\"Loan Approval\",\"category\":\"finance\"," +
                      "\"outputDataSchema\":" + OUTPUT_SCHEMA.strip().replace("\"", "\\\"") + "," +
                      "\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then()
                .statusCode(201)
                .body("outputDataSchema", notNullValue())
                .body("inputDataSchema", nullValue());
    }

    @Test
    void createTemplate_withBothSchemas_storesAndReturnsBoth() {
        given().contentType(ContentType.JSON)
                .body("{\"name\":\"Review Task\",\"category\":\"review\"," +
                      "\"inputDataSchema\":" + INPUT_SCHEMA.strip().replace("\"", "\\\"") + "," +
                      "\"outputDataSchema\":" + OUTPUT_SCHEMA.strip().replace("\"", "\\\"") + "," +
                      "\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then()
                .statusCode(201)
                .body("inputDataSchema", notNullValue())
                .body("outputDataSchema", notNullValue());
    }

    @Test
    void instantiateTemplate_withOutputDataSchema_snapshotsOntoWorkItem() {
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"Schema Template\",\"candidateGroups\":\"reviewers\"," +
                      "\"outputDataSchema\":" + OUTPUT_SCHEMA.strip().replace("\"", "\\\"") + "," +
                      "\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(201)
                .body("outputDataSchema", notNullValue())
                .body("inputDataSchema", nullValue())
                .body("templateId", equalTo(templateId));
    }

    @Test
    void instantiateTemplate_withoutSchemas_workItemSchemasAreNull() {
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"No Schema\",\"candidateGroups\":\"ops\",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(201)
                .body("outputDataSchema", nullValue())
                .body("inputDataSchema", nullValue());
    }
}
```

Note: the JSON body construction above is fragile. Use a cleaner approach with Jackson or a Map. Replace the body building with:

```java
// Helper to build a template body with schema
private String templateBody(final String name, final String outputSchema, final String inputSchema) {
    final var sb = new StringBuilder("{\"name\":\"" + name + "\",\"candidateGroups\":\"reviewers\",\"createdBy\":\"admin\"");
    if (outputSchema != null) sb.append(",\"outputDataSchema\":").append(outputSchema);
    if (inputSchema != null) sb.append(",\"inputDataSchema\":").append(inputSchema);
    sb.append("}");
    return sb.toString();
}
```

Use a simpler approach — Jackson ObjectMapper to build the JSON bodies:

```java
package io.casehub.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.notNullValue;
import static org.hamcrest.Matchers.nullValue;
import static org.hamcrest.Matchers.equalTo;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

@QuarkusTest
class WorkItemTemplateSchemaTest {

    private static final String OUTPUT_SCHEMA =
            "{\"type\":\"object\",\"required\":[\"decision\"]," +
            "\"properties\":{\"decision\":{\"type\":\"string\"}},\"additionalProperties\":false}";

    private static final String INPUT_SCHEMA =
            "{\"type\":\"object\",\"required\":[\"requestor\"]," +
            "\"properties\":{\"requestor\":{\"type\":\"string\"}},\"additionalProperties\":false}";

    @Test
    void createTemplate_withOutputDataSchema_storesAndReturnsIt() {
        given().contentType(ContentType.JSON)
                .body("{\"name\":\"Loan Approval\",\"category\":\"finance\"," +
                      "\"outputDataSchema\":" + OUTPUT_SCHEMA + ",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then()
                .statusCode(201)
                .body("outputDataSchema", notNullValue())
                .body("inputDataSchema", nullValue());
    }

    @Test
    void createTemplate_withBothSchemas_storesAndReturnsBoth() {
        given().contentType(ContentType.JSON)
                .body("{\"name\":\"Review Task\",\"candidateGroups\":\"reviewers\"," +
                      "\"inputDataSchema\":" + INPUT_SCHEMA + "," +
                      "\"outputDataSchema\":" + OUTPUT_SCHEMA + ",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then()
                .statusCode(201)
                .body("inputDataSchema", notNullValue())
                .body("outputDataSchema", notNullValue());
    }

    @Test
    void instantiateTemplate_withOutputDataSchema_snapshotsOntoWorkItem() {
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"Schema Template\",\"candidateGroups\":\"reviewers\"," +
                      "\"outputDataSchema\":" + OUTPUT_SCHEMA + ",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(201)
                .body("outputDataSchema", notNullValue())
                .body("inputDataSchema", nullValue())
                .body("templateId", equalTo(templateId));
    }

    @Test
    void instantiateTemplate_withoutSchemas_workItemSchemasAreNull() {
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"No Schema\",\"candidateGroups\":\"ops\",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(201)
                .body("outputDataSchema", nullValue())
                .body("inputDataSchema", nullValue());
    }
}
```

- [ ] **Step 1.2: Create the file**

Write the final version of `WorkItemTemplateSchemaTest.java` to disk at:
`runtime/src/test/java/io/casehub/work/runtime/api/WorkItemTemplateSchemaTest.java`

---

## Task 2: Write failing tests — service-layer schema validation

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemSchemaValidationTest.java`

- [ ] **Step 2.1: Write the test class**

```java
package io.casehub.work.runtime.api;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.containsString;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

/**
 * Tests that WorkItemService validates payload against inputDataSchema at create()
 * and resolution against outputDataSchema at complete(). Refs #170.
 */
@QuarkusTest
class WorkItemSchemaValidationTest {

    private static final String OUTPUT_SCHEMA =
            "{\"type\":\"object\",\"required\":[\"decision\"]," +
            "\"properties\":{\"decision\":{\"type\":\"string\"}},\"additionalProperties\":false}";

    private static final String INPUT_SCHEMA =
            "{\"type\":\"object\",\"required\":[\"requestor\"]," +
            "\"properties\":{\"requestor\":{\"type\":\"string\"}},\"additionalProperties\":false}";

    // ── outputDataSchema (resolution validation) ─────────────────────────────

    @Test
    void complete_validResolution_returns200() {
        final String id = workItemReadyToComplete(OUTPUT_SCHEMA);

        given().contentType(ContentType.JSON)
                .body("{\"resolution\":{\"decision\":\"approved\"},\"outcome\":null}")
                .put("/workitems/" + id + "/complete?actor=alice")
                .then()
                .statusCode(200);
    }

    @Test
    void complete_invalidResolution_returns400WithViolations() {
        final String id = workItemReadyToComplete(OUTPUT_SCHEMA);

        given().contentType(ContentType.JSON)
                .body("{\"resolution\":{\"wrong_field\":\"value\"}}")
                .put("/workitems/" + id + "/complete?actor=alice")
                .then()
                .statusCode(400)
                .body("error", containsString("outputDataSchema"));
    }

    @Test
    void complete_nullResolution_whenOutputSchemaSet_returns200() {
        // Null resolution is always valid — lenient behaviour
        final String id = workItemReadyToComplete(OUTPUT_SCHEMA);

        given().contentType(ContentType.JSON)
                .body("{}")
                .put("/workitems/" + id + "/complete?actor=alice")
                .then()
                .statusCode(200);
    }

    @Test
    void complete_noOutputSchema_anyResolutionAccepted() {
        final String id = workItemReadyToComplete(null);

        given().contentType(ContentType.JSON)
                .body("{\"resolution\":{\"anything\":true}}")
                .put("/workitems/" + id + "/complete?actor=alice")
                .then()
                .statusCode(200);
    }

    // ── inputDataSchema (payload validation at create) ────────────────────────

    @Test
    void instantiate_validDefaultPayload_returns201() {
        // Template has inputDataSchema + a defaultPayload that satisfies it
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"Input Schema Template\",\"candidateGroups\":\"ops\"," +
                      "\"inputDataSchema\":" + INPUT_SCHEMA + "," +
                      "\"defaultPayload\":{\"requestor\":\"eng-team\"}," +
                      "\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(201);
    }

    @Test
    void instantiate_invalidDefaultPayload_returns400() {
        // Template has inputDataSchema + a defaultPayload that violates it
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"Bad Payload Template\",\"candidateGroups\":\"ops\"," +
                      "\"inputDataSchema\":" + INPUT_SCHEMA + "," +
                      "\"defaultPayload\":{\"wrong_field\":\"value\"}," +
                      "\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(400);
    }

    @Test
    void instantiate_nullPayload_whenInputSchemaSet_returns201() {
        // Null payload is always valid — lenient behaviour
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"Null Payload Template\",\"candidateGroups\":\"ops\"," +
                      "\"inputDataSchema\":" + INPUT_SCHEMA + ",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then()
                .statusCode(201);
    }

    @Test
    void directCreate_noTemplate_noSchemaValidation() {
        // Direct creation never has schema validation — no template = no schema
        given().contentType(ContentType.JSON)
                .body("{\"title\":\"Ad hoc\",\"candidateGroups\":\"ops\",\"createdBy\":\"system\"}")
                .post("/workitems")
                .then()
                .statusCode(201);
    }

    // ── Helper ────────────────────────────────────────────────────────────────

    private String workItemReadyToComplete(final String outputDataSchema) {
        final String schemaJson = outputDataSchema != null
                ? ",\"outputDataSchema\":" + outputDataSchema
                : "";
        final String templateId = given().contentType(ContentType.JSON)
                .body("{\"name\":\"Completion Schema\",\"candidateGroups\":\"reviewers\"" +
                      schemaJson + ",\"createdBy\":\"admin\"}")
                .post("/workitem-templates")
                .then().statusCode(201).extract().path("id");

        final String id = given().contentType(ContentType.JSON)
                .body("{\"createdBy\":\"system\"}")
                .post("/workitem-templates/" + templateId + "/instantiate")
                .then().statusCode(201).extract().path("id");

        given().put("/workitems/" + id + "/claim?claimant=alice").then().statusCode(200);
        given().put("/workitems/" + id + "/start?actor=alice").then().statusCode(200);
        return id;
    }
}
```

- [ ] **Step 2.2: Create the file** at
  `runtime/src/test/java/io/casehub/work/runtime/api/WorkItemSchemaValidationTest.java`

---

## Task 3: Verify RED — tests fail to compile (new fields don't exist yet)

- [ ] **Step 3.1: Compile test sources**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | grep "ERROR" | head -20
```

Expected: compile errors referencing `outputDataSchema`, `inputDataSchema` on
`WorkItemResponse` and `WorkItemTemplate`. If tests compile and pass, check that they
genuinely exercise non-existent code paths.

---

## Task 4: Delete WorkItemFormSchema (entity, resource, tests)

These files test code being removed. Delete them before implementing to avoid
false-positive test coverage.

- [ ] **Step 4.1: Delete the files**

```bash
rm runtime/src/main/java/io/casehub/work/runtime/model/WorkItemFormSchema.java
rm runtime/src/main/java/io/casehub/work/runtime/api/WorkItemFormSchemaResource.java
rm runtime/src/test/java/io/casehub/work/runtime/api/WorkItemFormSchemaTest.java
rm runtime/src/test/java/io/casehub/work/runtime/api/WorkItemFormSchemaValidationTest.java
```

- [ ] **Step 4.2: Compile to surface all broken references**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | grep "ERROR" | grep -v "^\\[ERROR\\] $" | head -30
```

Expected errors: `WorkItemResource` references `WorkItemFormSchema` and
`FormSchemaValidationService` (will be removed in Task 8). These are expected — do not
fix them yet.

---

## Task 5: Flyway migrations

**Files:**
- Create: `runtime/src/main/resources/db/migration/V23__template_data_schemas.sql`
- Create: `runtime/src/main/resources/db/migration/V24__workitem_data_schemas.sql`
- Create: `runtime/src/main/resources/db/migration/V25__drop_form_schema.sql`

H2 does not support multi-column `ALTER TABLE ... ADD COLUMN x, ADD COLUMN y` — use
separate statements.

- [ ] **Step 5.1: Create V23**

```sql
-- Refs #170: schema-validated output data
ALTER TABLE work_item_template ADD COLUMN input_data_schema TEXT;
ALTER TABLE work_item_template ADD COLUMN output_data_schema TEXT;
```

- [ ] **Step 5.2: Create V24**

```sql
-- Refs #170: snapshotted schema fields on WorkItem
ALTER TABLE work_item ADD COLUMN input_data_schema TEXT;
ALTER TABLE work_item ADD COLUMN output_data_schema TEXT;
```

- [ ] **Step 5.3: Create V25**

```sql
-- Refs #170: WorkItemFormSchema removed — template is the type definition
DROP TABLE IF EXISTS work_item_form_schema;
```

---

## Task 6: Data model — WorkItemTemplate and WorkItem

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemTemplate.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItem.java`

- [ ] **Step 6.1: Add fields to WorkItemTemplate**

After the `outcomes` field (added in #169), add:

```java
/**
 * JSON Schema (draft-07) validating {@link WorkItem#payload} at instantiation.
 * Null means no input constraint. Snapshotted onto {@link WorkItem#inputDataSchema}
 * at instantiation time.
 */
@Column(name = "input_data_schema", columnDefinition = "TEXT")
public String inputDataSchema;

/**
 * JSON Schema (draft-07) validating {@link WorkItem#resolution} at completion.
 * Null means no output constraint. Snapshotted onto {@link WorkItem#outputDataSchema}
 * at instantiation time.
 */
@Column(name = "output_data_schema", columnDefinition = "TEXT")
public String outputDataSchema;
```

- [ ] **Step 6.2: Add fields to WorkItem**

After the `outcome` field (added in #169), add:

```java
/**
 * JSON Schema (draft-07) for {@link #payload}, snapshotted from the template at
 * instantiation. Null for WorkItems created without a template or from a template
 * that declares no input schema. Used by {@code WorkItemService.create()} to
 * validate the payload before persisting.
 */
@Column(name = "input_data_schema", columnDefinition = "TEXT")
public String inputDataSchema;

/**
 * JSON Schema (draft-07) for {@link #resolution}, snapshotted from the template at
 * instantiation. Null means no output constraint. Used by
 * {@code WorkItemService.complete()} to validate the resolution before completing.
 */
@Column(name = "output_data_schema", columnDefinition = "TEXT")
public String outputDataSchema;
```

---

## Task 7: WorkItemCreateRequest — add schema parameters

**File:** `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemCreateRequest.java`

- [ ] **Step 7.1: Add two new record components at the end**

The record currently ends with (after #169):
```java
UUID templateId,
List<String> permittedOutcomes)
```

Add two more after `permittedOutcomes`:

```java
UUID templateId,
List<String> permittedOutcomes,
/** JSON Schema string for payload validation; null = no constraint. */
String inputDataSchema,
/** JSON Schema string for resolution validation; null = no constraint. */
String outputDataSchema)
```

Also add `@param` entries to the Javadoc block at the top of the record.

---

## Task 8: Fix all WorkItemCreateRequest call sites

`WorkItemCreateRequest` now has 23 components. Every constructor call that was
21-arg needs two `null` appended. Run this to find them all:

```bash
grep -rn "new WorkItemCreateRequest(" /Users/mdproctor/claude/casehub/work/runtime/src --include="*.java" | grep -v test
```

Expected matches (all production code):
1. `WorkItemMapper.toServiceRequest()` — direct REST creation, no template
2. `WorkItemTemplateService.toCreateRequest()` — template instantiation (will set real values in Task 9)
3. `WorkItemSpawnService` — spawn creation (will set real values in Task 10)
4. `MultiInstanceSpawnService.buildParentRequest()` — will set real values in Task 10
5. `MultiInstanceSpawnService.buildChildRequest()` — will set real values in Task 10
6. `WorkItemService.cloneWorkItem()` — clone, no template

- [ ] **Step 8.1: Fix WorkItemMapper.toServiceRequest() — append `null, null`**

```java
// Before (last two params):
                req.claimDeadlineBusinessHours(), req.expiresAtBusinessHours(),
                null, null); // templateId and permittedOutcomes — not set for direct creation
// After:
                req.claimDeadlineBusinessHours(), req.expiresAtBusinessHours(),
                null, null,   // templateId, permittedOutcomes — not set for direct creation
                null, null);  // inputDataSchema, outputDataSchema — no template, no schema
```

- [ ] **Step 8.2: Fix WorkItemService.cloneWorkItem() — append `null, null`**

Find the clone constructor call (ends with `null, null, null, null`) and append two more nulls:
```java
                null, null, // no template provenance for clones
                null, null); // no schema for clones
```

- [ ] **Step 8.3: Fix all test call sites**

Run:
```bash
grep -rn "new WorkItemCreateRequest(" /Users/mdproctor/claude/casehub/work/runtime/src/test --include="*.java" | grep -v "WorkItemTemplateSchemaTest\|WorkItemSchemaValidationTest"
```

For each match, append `, null, null` (for `inputDataSchema` and `outputDataSchema`)
before the closing `)`. Use the Python script pattern from #169:

```python
import re

base = '/Users/mdproctor/claude/casehub/work/runtime/src/test/java/io/casehub/work/runtime'
files = [
    # list all test files that construct WorkItemCreateRequest
]

def fix_create_request(content):
    result = []
    i = 0
    while i < len(content):
        idx = content.find('new WorkItemCreateRequest(', i)
        if idx == -1:
            result.append(content[i:])
            break
        result.append(content[i:idx])
        start = idx + len('new WorkItemCreateRequest(')
        depth = 1
        j = start
        while j < len(content) and depth > 0:
            c = content[j]
            if c == '(':  depth += 1
            elif c == ')': depth -= 1
            j += 1
        inner = content[start:j-1]
        result.append('new WorkItemCreateRequest(' + inner + ', null, null)')
        i = j
    return ''.join(result)
```

Also run the same script on:
- `postgres-broadcaster/src/test/java/io/casehub/work/postgres/broadcaster/PostgresBroadcasterIT.java`

---

## Task 9: WorkItemTemplateService — pass schema fields in toCreateRequest()

**File:** `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`

- [ ] **Step 9.1: Update the 6-arg toCreateRequest() call**

Find the `return new WorkItemCreateRequest(` in `toCreateRequest()`. It currently ends with:
```java
                template.id,
                parseOutcomeNames(template.outcomes));
```

Change to:
```java
                template.id,
                parseOutcomeNames(template.outcomes),
                template.inputDataSchema,
                template.outputDataSchema);
```

No other changes to this file.

---

## Task 10: WorkItemSpawnService + MultiInstanceSpawnService — pass schema fields

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemSpawnService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/multiinstance/MultiInstanceSpawnService.java`

- [ ] **Step 10.1: WorkItemSpawnService — update constructor call**

Find the `new WorkItemCreateRequest(` in `WorkItemSpawnService`. It currently ends with:
```java
                template.id,
                WorkItemTemplateService.parseOutcomeNames(template.outcomes));
```

Change to:
```java
                template.id,
                WorkItemTemplateService.parseOutcomeNames(template.outcomes),
                template.inputDataSchema,
                template.outputDataSchema);
```

- [ ] **Step 10.2: MultiInstanceSpawnService — update buildParentRequest()**

Find `buildParentRequest()`. Currently ends with:
```java
                template.id,
                WorkItemTemplateService.parseOutcomeNames(template.outcomes));
```

Change to:
```java
                template.id,
                WorkItemTemplateService.parseOutcomeNames(template.outcomes),
                template.inputDataSchema,
                template.outputDataSchema);
```

- [ ] **Step 10.3: MultiInstanceSpawnService — update buildChildRequest()**

Same pattern — same fix to the second `new WorkItemCreateRequest(` call in the file.

---

## Task 11: WorkItemService — inject validator, add schema validation

**File:** `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`

- [ ] **Step 11.1: Add FormSchemaValidationService injection**

`WorkItemService` currently uses constructor injection for its five main dependencies.
Add field injection for `FormSchemaValidationService` (consistent with how other optional
services are injected in this class):

```java
@Inject
FormSchemaValidationService schemaValidator;
```

Add the import:
```java
import io.casehub.work.runtime.service.FormSchemaValidationService;
```

- [ ] **Step 11.2: Set inputDataSchema and outputDataSchema in create()**

In `create()`, after the existing field assignments (after `item.permittedOutcomes = ...`):

```java
item.inputDataSchema = request.inputDataSchema();
item.outputDataSchema = request.outputDataSchema();
```

- [ ] **Step 11.3: Add payload validation in create()**

After setting the item fields and before `assignmentService.assign(item, ...)`, add:

```java
if (item.inputDataSchema != null) {
    final List<String> violations = schemaValidator.validate(item.inputDataSchema, item.payload);
    if (!violations.isEmpty()) {
        throw new IllegalArgumentException("payload violates inputDataSchema: " + violations);
    }
}
```

Add import if not already present:
```java
import java.util.List;
```

- [ ] **Step 11.4: Add resolution validation in complete()**

The `complete(UUID id, String actorId, String resolution, String outcome)` method.
After `validateOutcome(item, outcome)` and before setting status, add:

```java
if (item.outputDataSchema != null) {
    final List<String> violations = schemaValidator.validate(item.outputDataSchema, resolution);
    if (!violations.isEmpty()) {
        throw new IllegalArgumentException("resolution violates outputDataSchema: " + violations);
    }
}
```

Do the same for the 6-arg `complete(id, actorId, resolution, outcome, rationale, planRef)` —
same position.

Do NOT add validation to `completeFromSystem()` — system completions bypass schema checks.

---

## Task 12: WorkItemResource — remove FormSchema lookup

**File:** `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResource.java`

The resource currently validates payload and resolution against the category-level
`WorkItemFormSchema`. Both validations are being replaced by service-layer validation.

- [ ] **Step 12.1: Remove FormSchemaValidationService injection**

Delete the field:
```java
@Inject
FormSchemaValidationService schemaValidator;
```

Delete the imports:
```java
import io.casehub.work.runtime.model.WorkItemFormSchema;
import io.casehub.work.runtime.service.FormSchemaValidationService;
```

- [ ] **Step 12.2: Remove category-level payload validation from create()**

In the `create()` method, delete this block:
```java
if (request != null && request.category() != null && !request.category().isBlank()) {
    final WorkItemFormSchema schema = WorkItemFormSchema.findLatestByCategory(request.category());
    if (schema != null && schema.payloadSchema != null) {
        final List<String> violations = schemaValidator.validate(schema.payloadSchema, request.payload());
        if (!violations.isEmpty()) {
            final java.util.LinkedHashMap<String, Object> err = new java.util.LinkedHashMap<>();
            err.put("error", "payload violates form schema");
            err.put("violations", violations);
            return Response.status(Response.Status.BAD_REQUEST).entity(err).build();
        }
    }
}
```

The `try { ... } catch (IllegalArgumentException e)` block that wraps `workItemService.create()`
will now catch payload schema violations from the service layer. If that try-catch doesn't
already exist, add it:

```java
try {
    final WorkItem created = workItemService.create(WorkItemMapper.toServiceRequest(request));
    final URI location = URI.create("/workitems/" + created.id);
    return Response.created(location).entity(WorkItemMapper.toResponse(created)).build();
} catch (IllegalArgumentException e) {
    return Response.status(Response.Status.BAD_REQUEST)
            .entity(Map.of("error", e.getMessage()))
            .build();
}
```

- [ ] **Step 12.3: Remove category-level resolution validation from complete()**

In the `complete()` method, delete this block:
```java
final WorkItem current = workItemStore.get(id).orElse(null);
if (current != null && current.category != null && !current.category.isBlank()) {
    final WorkItemFormSchema schema = WorkItemFormSchema.findLatestByCategory(current.category);
    if (schema != null && schema.resolutionSchema != null) {
        final List<String> violations = schemaValidator.validate(schema.resolutionSchema, resolution);
        if (!violations.isEmpty()) {
            final java.util.LinkedHashMap<String, Object> err = new java.util.LinkedHashMap<>();
            err.put("error", "resolution violates form schema");
            err.put("violations", violations);
            return Response.status(Response.Status.BAD_REQUEST).entity(err).build();
        }
    }
}
```

The existing `catch (IllegalArgumentException e)` in `complete()` already maps to 400 and
will catch resolution schema violations from the service layer.

Note: the `workItemStore` field injection can remain — it is still used elsewhere in the resource.

---

## Task 13: WorkItemTemplateResource — add schema fields to CRUD

**File:** `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemTemplateResource.java`

- [ ] **Step 13.1: Add fields to CreateTemplateRequest**

Add two new fields to the `CreateTemplateRequest` record after `outcomes`:
```java
/** JSON Schema (draft-07) for payload validation at instantiation; null = no constraint. */
String inputDataSchema,
/** JSON Schema (draft-07) for resolution validation at completion; null = no constraint. */
String outputDataSchema,
String createdBy)
```

(Move `createdBy` to the last position after the new fields.)

- [ ] **Step 13.2: Set on entity in createTemplate()**

After `t.outcomes = WorkItemTemplateService.encodeOutcomes(request.outcomes());`:
```java
t.inputDataSchema = request.inputDataSchema();
t.outputDataSchema = request.outputDataSchema();
```

- [ ] **Step 13.3: Include in toResponse()**

After `m.put("outcomes", ...)`:
```java
m.put("inputDataSchema", t.inputDataSchema);
m.put("outputDataSchema", t.outputDataSchema);
```

---

## Task 14: WorkItemResponse, WorkItemWithAuditResponse, WorkItemMapper, WorkItemContextBuilder

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemResponse.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemWithAuditResponse.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/WorkItemMapper.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/event/WorkItemContextBuilder.java`

- [ ] **Step 14.1: WorkItemResponse — add two fields at end**

After `List<String> permittedOutcomes)`, add:
```java
/** JSON Schema for payload; snapshotted from template at instantiation. */
String inputDataSchema,
/** JSON Schema for resolution; snapshotted from template at instantiation. */
String outputDataSchema)
```

- [ ] **Step 14.2: WorkItemWithAuditResponse — add two fields at end**

After `String outcome)`, add:
```java
String inputDataSchema,
String outputDataSchema)
```

- [ ] **Step 14.3: WorkItemMapper — add to toResponse()**

After `wi.templateId, wi.outcome, OutcomeCodecs.decodePermittedOutcomes(wi.permittedOutcomes)`:
```java
wi.inputDataSchema,
wi.outputDataSchema)
```

- [ ] **Step 14.4: WorkItemMapper — add to toWithAudit()**

After `wi.templateId, wi.outcome`:
```java
wi.inputDataSchema,
wi.outputDataSchema)
```

- [ ] **Step 14.5: WorkItemContextBuilder — add to toMap()**

After `map.put("outcome", workItem.outcome);`:
```java
map.put("inputDataSchema", workItem.inputDataSchema);
map.put("outputDataSchema", workItem.outputDataSchema);
```

---

## Task 15: Verify GREEN — compile and run all tests

- [ ] **Step 15.1: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime 2>&1 | grep "ERROR" | head -20
```

Expected: no errors.

- [ ] **Step 15.2: Run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -8
```

Expected: all tests pass, count increases from 685 (removing ~55 FormSchema tests,
adding ~12 new schema validation tests → net ~630+12 = ~642+; exact count depends on
FormSchema test count).

- [ ] **Step 15.3: Run api module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api 2>&1 | tail -5
```

Expected: 37 tests, 0 failures (api module unchanged).

- [ ] **Step 15.4: Install and test postgres-broadcaster**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -pl runtime 2>&1 | tail -3
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl postgres-broadcaster -Dtest=WorkItemEventPayloadTest 2>&1 | tail -5
```

Expected: 8 tests, 0 failures.

---

## Task 16: Code review, commit, doc sync, journal

- [ ] **Step 16.1: Invoke superpowers:requesting-code-review**

Before committing. Fix any Important findings. Batch Minor nits into a single GitHub
issue if not addressed.

- [ ] **Step 16.2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add -p
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(template): schema-validated output data (Closes #170)

Refs #77"
```

- [ ] **Step 16.3: implementation-doc-sync**

Invoke `implementation-doc-sync` to sync DESIGN.md (add phase 22), CLAUDE.md,
casehub-work.md, and PLATFORM.md.

- [ ] **Step 16.4: Write journal entry**

Invoke `java-update-design` to update `design/JOURNAL.md` with the design decisions
from this epic.

- [ ] **Step 16.5: Epic close**

Invoke `/epic` to close `epic-output-schema` — promote artifacts, merge journal,
close issue #170.
