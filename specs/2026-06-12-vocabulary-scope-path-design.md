# Replace VocabularyScope Enum with Path-Based Scope Hierarchy

**Issue:** casehubio/work#236
**Date:** 2026-06-12
**Status:** Approved

## Summary

Replace the fixed 4-level `VocabularyScope` enum (GLOBAL/ORG/TEAM/PERSONAL) with the platform's `Path` type from `casehub-platform-api`. Scope accessibility changes from ordinal comparison to `Path.isAncestorOf()`. The `ownerId` field is dropped — the scope path subsumes it. The `VocabularyScope` enum is deleted.

## Motivation

`VocabularyScope` is a parallel scope type that violates the platform boundary rule: "Do not define parallel path, scope, preference, or principal types." `Path` already provides `isAncestorOf()`, `root()`, `depth()`, and `segments()` — everything the vocabulary hierarchy needs, without a hardcoded 4-level ceiling.

The `ownerId` field is entangled with `scope` — they form a compound key where the enum constrains valid owner types. With `Path`, the compound key collapses: the path segments encode both the hierarchy tier (depth) and the identity (segment values). Keeping `ownerId` alongside a scope path creates two sources of truth with no consumer.

## Design

### Domain Model — `LabelVocabulary`

Replace the `scope` field type and drop `ownerId`:

```java
// Before
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 20)
public VocabularyScope scope;

@Column(name = "owner_id", length = 255)
public String ownerId;

// After
@Convert(converter = PathAttributeConverter.class)
@Column(nullable = false, length = 500)
public Path scope;
```

### PathAttributeConverter — Root Path Fix

`Path.parse("")` throws `IllegalArgumentException`. Root scope stores as `""` in the DB. The converter needs a root-path guard:

```java
@Override
public Path convertToEntityAttribute(final String value) {
    if (value == null) return null;
    if (value.isEmpty()) return Path.root();
    return Path.parse(value);
}
```

`convertToDatabaseColumn` is unchanged — `Path.root().value()` returns `""`.

### Service Layer — `LabelVocabularyService`

**`isScopeAccessible` → `isAccessibleFrom`:**

```java
public static boolean isAccessibleFrom(Path vocabScope, Path callerScope) {
    return vocabScope.equals(callerScope) || vocabScope.isAncestorOf(callerScope);
}
```

Root semantics: `Path.root().isAncestorOf(anyNonRoot)` returns true (empty segments → `!other.segments.isEmpty()`). Root-scoped vocabularies are accessible to everyone. A root caller sees only root-scoped vocabularies. Matches current GLOBAL behaviour.

**`listAccessible(VocabularyScope)` → `listAccessible(Path)`:** Same logic, new types.

**`findGlobalVocabulary()`:** Filter changes from `v.scope == VocabularyScope.GLOBAL` to `v.scope.equals(Path.root())`. Name unchanged — "global" is still the domain concept.

**`findOrCreateVocabulary` — simplified:**

```java
// Before: (VocabularyScope scope, String ownerId, String name)
// After:  (Path scope, String name)
```

Lookup changes from `scope == scope && ownerId.equals(v.ownerId)` to `scope.equals(v.scope)`.

### REST API — `VocabularyResource`

**Scope moves from URL path param to request body.** Path strings like `acme-corp/hr-team` contain `/` which collides with URL path structure. The scope is a property of the operation — it belongs in the body.

```java
// Before
record AddDefinitionRequest(String path, String description, String addedBy, String ownerId) {}
@POST @Path("/{scope}")
Response addDefinition(@PathParam("scope") String scopeStr, AddDefinitionRequest request)

// After
record AddDefinitionRequest(String path, String description, String addedBy, String scope) {}
@POST
Response addDefinition(AddDefinitionRequest request)
```

Scope parsing: null/blank → `Path.root()`, otherwise `Path.parse(scope)`.

**`GET /vocabulary` (listAll):** Currently hardcodes `VocabularyScope.PERSONAL` (= "show everything"). No `Path` equivalent of "deepest possible" exists. Change to list all definitions by scanning all vocabularies and flatmapping their definitions — no scope filter applied. This is what the endpoint was effectively doing (scope enforcement is deferred). The `listAccessible(Path)` method remains available for when scope enforcement is activated.

### Store/Repository Layer

No changes needed. `LabelVocabularyStore` SPI and implementations (JPA, InMemory) do generic CRUD without referencing `VocabularyScope`. The entity field type change is transparent via JPA converter.

### Flyway Migration — V5003

```sql
-- Widen scope column
ALTER TABLE label_vocabulary ALTER COLUMN scope TYPE VARCHAR(500);

-- Convert enum values to path strings
UPDATE label_vocabulary SET scope = '' WHERE scope = 'GLOBAL';
UPDATE label_vocabulary SET scope = owner_id WHERE scope IN ('ORG', 'TEAM', 'PERSONAL');

-- Drop owner_id
ALTER TABLE label_vocabulary DROP COLUMN owner_id;
```

**Data loss acknowledgment:** ORG/TEAM/PERSONAL vocabularies become single-segment paths. The tier distinction is lost because the old schema never captured hierarchy — `ownerId="hr-team"` at TEAM scope didn't record which org. New vocabularies use proper hierarchical paths. The seeded GLOBAL row converts to `scope=''` (root).

**H2 compatibility:** `ALTER COLUMN ... TYPE` and `DROP COLUMN` work in both PostgreSQL and H2 `MODE=PostgreSQL`.

### Tests

**`LabelVocabularyServiceTest`:** Ordinal hierarchy tests → ancestor-based accessibility tests. Enum count test deleted. `matchesPattern` tests unchanged.

**`JpaLabelVocabularyStoreTenancyTest` / `JpaLabelDefinitionStoreTenancyTest`:** `VocabularyScope.GLOBAL` → `Path.root()` in test vocabulary creation.

**`VocabularyScenario`:** `VocabularyScope.TEAM` → `Path.of("casehubio", TEAM_ID)`. Demonstrates proper two-segment path following platform convention.

### Cleanup

Delete `VocabularyScope.java`. All references eliminated by the changes above.

## Files Changed

| File | Change |
|------|--------|
| `runtime/.../model/VocabularyScope.java` | **Deleted** |
| `runtime/.../model/LabelVocabulary.java` | scope: enum → Path, drop ownerId |
| `runtime/.../model/PathAttributeConverter.java` | Root path guard in `convertToEntityAttribute` |
| `runtime/.../service/LabelVocabularyService.java` | All scope methods: enum → Path |
| `runtime/.../api/VocabularyResource.java` | Scope in body, drop ownerId, listAll without filter |
| `runtime/.../resources/db/work/migration/V5003__vocabulary_scope_to_path.sql` | Migration |
| `persistence-memory/.../InMemoryLabelVocabularyStore.java` | No changes needed |
| `runtime/test/.../LabelVocabularyServiceTest.java` | Rewrite scope tests |
| `runtime/test/.../JpaLabelVocabularyStoreTenancyTest.java` | GLOBAL → Path.root() |
| `runtime/test/.../JpaLabelDefinitionStoreTenancyTest.java` | GLOBAL → Path.root() |
| `examples/.../VocabularyScenario.java` | TEAM → Path.of("casehubio", TEAM_ID) |

## Garden Entries Referenced

- `GE-20260522-a87fd7`: Path.parent() returns null for single-segment paths — root scope excluded from ancestor walk. Relevant to root handling but `isAncestorOf()` handles root correctly.

## Platform Coherence

- Removes parallel scope type per boundary rule
- Uses `Path` from `casehub-platform-api` as designed
- `VocabularyScope` is in `runtime/model/` (not `casehub-work-api`) — no external consumers, no cross-repo propagation
- No deferred concerns requiring separate issues
