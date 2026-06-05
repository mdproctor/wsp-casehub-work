# Design: Conditional Outcomes (#177) + PATCH Templates (#199)

**Branch:** `issue-177-outcomes-and-patch`  
**Issues:** #177 (primary), #199  
**Date:** 2026-06-05

---

## Overview

Two independent S-scale features shipped on one branch:

1. **#177 — Conditional Outcomes:** `Outcome` gains an optional JEXL `condition` field. At completion, if the chosen outcome has a condition, it is evaluated against the WorkItem context. Condition false → rejection.
2. **#199 — PATCH /workitem-templates/{id}:** JSON Merge Patch (RFC 7396) endpoint for partial template updates. Absent fields are left unchanged; null clears.

---

## #177 — Conditional Outcomes

### Data Model

**`Outcome` record** (`api/src/main/java/io/casehub/work/api/Outcome.java`):

```java
public record Outcome(String name, String displayName, String condition) {}
```

`condition` is a nullable JEXL expression string. Null means unconditional. Adding this field is a source-breaking change: all `new Outcome("name", "label")` call sites become `new Outcome("name", "label", null)`. The break is deliberate — callers become explicit.

**`WorkItem.permittedOutcomes` storage format change:**

Currently stores `["approved","rejected"]` (name strings). After this change stores full `Outcome` objects: `[{"name":"approved","displayName":"Approved","condition":"..."}]`.

No Flyway migration. `OutcomeCodecs.decodePermittedOutcomes()` uses a dual-format decoder: attempts `List<Outcome>` deserialization first; if that fails, falls back to `List<String>` and wraps each name into `Outcome(name, null, null)`. Existing rows degrade gracefully; new instantiations write the new format.

**Instantiation change:** `WorkItemTemplateService.instantiate()` currently calls `OutcomeCodecs.parseOutcomeNames()` to extract name strings and encode them into `permittedOutcomes`. It instead snapshots the full decoded `List<Outcome>` from the template — preserving displayName and condition as of template state at instantiation time. This is consistent with how `inputDataSchema` and `outputDataSchema` are already snapshotted.

### `OutcomeCodecs` Changes

| Method | Change |
|--------|--------|
| `decodePermittedOutcomes(String)` | Return type `List<String>` → `List<Outcome>`. Dual-format decoder. |
| `encodePermittedOutcomes(List<String>)` | Removed. |
| `encodePermittedOutcomes(List<Outcome>)` | New. Delegates to `encodeOutcomes()`. |
| `parseOutcomeNames(String)` | Unchanged — still extracts names from template outcomes JSON. |
| `encodeOutcomes(List<Outcome>)` | Unchanged. |
| `decodeOutcomes(String)` | Unchanged. |

### Service Layer — `WorkItemService.validateOutcome()`

Signature change: receives `resolution` and `actorId` in addition to `WorkItem` and `outcome` name (passed through from `complete()` and `reject()`).

Evaluation sequence:
1. If `item.permittedOutcomes` is null → return (unconstrained, existing behaviour).
2. If `outcome` is null/blank → throw (existing behaviour).
3. Decode `List<Outcome>` from `item.permittedOutcomes` using new decoder.
4. Find matching `Outcome` by name → if absent, throw (existing behaviour).
5. If `matching.condition()` is non-null and non-blank: evaluate with `JexlConditionEvaluator`.
   - Context: `WorkItemContextBuilder.toMap(item)` exposed as `workItem` variable, plus `resolution` and `actorId` as top-level variables.
   - If condition evaluates false → throw `IllegalArgumentException("outcome '${name}' condition not satisfied")`.

`WorkItemService` gains two injected dependencies: `JexlConditionEvaluator` and `WorkItemContextBuilder` — both `@ApplicationScoped`, already in the CDI graph.

### API Surface

**WorkItem responses:** `permittedOutcomes` field changes from `["approved","rejected"]` to full Outcome objects `[{"name":"approved","displayName":"Approved","condition":"..."}]`. Updated in `WorkItemMapper.toResponse()`.

**Template requests/responses:** `List<Outcome> outcomes` field in `CreateTemplateRequest`, `UpdateTemplateRequest`, and `toResponse()` already accepts `Outcome` objects — callers just start passing `condition` where needed. No structural change.

**JEXL filter rule context:** `WorkItemContextBuilder` continues to expose `permittedOutcomes` as `List<String>` (names only) — extracted from the new `List<Outcome>` before putting into the JEXL context map. Existing filter rules are unaffected.

### JEXL Condition Reference

Available variables at outcome condition evaluation time:

| Variable | Type | Description |
|----------|------|-------------|
| `workItem` | `Map<String,Object>` | Full WorkItem context (same as filter rules) |
| `resolution` | `String` | The completion payload |
| `actorId` | `String` | Who is completing/rejecting |

Example conditions:
- `workItem.priority == 'URGENT'` — outcome only available on urgent items
- `actorId.startsWith('mgr-')` — outcome only available to managers
- `resolution != null && resolution.contains('APPROVED')` — outcome tied to resolution content

---

## #199 — PATCH /workitem-templates/{id}

### Endpoint

```
PATCH /workitem-templates/{id}
Content-Type: application/merge-patch+json
```

Handler in `WorkItemTemplateResource`:

```java
@PATCH
@Path("/{id}")
@Consumes("application/merge-patch+json")
@Transactional
public Response patchTemplate(@PathParam("id") UUID id, JsonNode patch) { ... }
```

No new Maven dependency. `JsonNode` is already imported in this class (used for `inputDataSchema`/`outputDataSchema`).

### Patch Semantics (RFC 7396)

For each of the ~22 template fields:

| Patch field state | Action |
|-------------------|--------|
| Absent from patch | Leave existing value unchanged |
| Present, non-null | Update to provided value |
| Present, null | Clear the field (set to null) |

Special cases:
- `name` present and null → 400 (`name` is required when provided in a PATCH)
- `name` changed to a value that conflicts with another template → 409
- `createdBy` — not patchable (immutable, absent from `UpdateTemplateRequest` for the same reason)

### Validation and Response

Passes through `WorkItemTemplateValidationService.validate(t)` and `toResponse(t)` — same path as `PUT`. Returns 200 with the full updated template.

---

## Test Plan

### #177 — Unit Tests

**`OutcomeCodecs` (new or extended):**
- `decodePermittedOutcomes` with new object-array format
- `decodePermittedOutcomes` with old string-array format (fallback path)
- Round-trip: encode full `List<Outcome>` → decode → same list

**`WorkItemOutcomeValidationTest` (extend):**
- Outcome with null condition: name-only validation, accepted
- Outcome with condition evaluating true: accepted
- Outcome with condition evaluating false: `IllegalArgumentException`
- Condition references `resolution`: matched correctly
- Condition references `actorId`: matched correctly
- Old-format `permittedOutcomes`: fallback decoding, name validation still works

### #177 — Integration Test

Add to `CreditDecisionScenario` or `WorkItemLifecycleIT`:
- Create template with conditional outcome; instantiate; complete with satisfied condition → 200
- Same but condition not satisfied → 400

### #199 — Unit Tests (`WorkItemTemplatePatchTest`, new):
- PATCH absent field: existing value preserved
- PATCH field with non-null value: field updated
- PATCH field with null: field cleared
- PATCH `name` with null: 400
- PATCH `name` to conflicting name: 409
- PATCH non-existent template: 404
- Empty patch: 200, entity unchanged

### #199 — Integration Test

Add to `WorkItemTemplateIT`:
- Create template, PATCH one field, GET confirms only that field changed, others intact

---

## Out of Scope

- Condition syntax validation at template creation time (fail-fast on invalid JEXL) — deferred, not required for #177
- Condition evaluation at instantiation to filter `permittedOutcomes` — not the OHT model; evaluation is at completion time only
