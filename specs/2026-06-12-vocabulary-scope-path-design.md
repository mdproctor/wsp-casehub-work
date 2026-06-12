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

**Full collapsed `addDefinition` implementation:**

```java
@POST
@Transactional
public Response addDefinition(final AddDefinitionRequest request) {
    if (request == null || request.path() == null || request.path().isBlank()) {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity(Map.of("error", "path is required")).build();
    }
    if (request.path().contains("*") || request.path().contains("?")) {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity(Map.of("error", "path must not contain wildcard characters")).build();
    }

    final io.casehub.platform.api.path.Path labelPath;
    try {
        labelPath = io.casehub.platform.api.path.Path.parse(request.path());
    } catch (IllegalArgumentException e) {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity(Map.of("error", "invalid path: " + e.getMessage())).build();
    }

    final io.casehub.platform.api.path.Path scopePath;
    try {
        scopePath = (request.scope() == null || request.scope().isBlank())
                ? io.casehub.platform.api.path.Path.root()
                : io.casehub.platform.api.path.Path.parse(request.scope());
    } catch (IllegalArgumentException e) {
        return Response.status(Response.Status.BAD_REQUEST)
                .entity(Map.of("error", "invalid scope: " + e.getMessage())).build();
    }

    final LabelVocabulary vocab = scopePath.equals(io.casehub.platform.api.path.Path.root())
            ? vocabularyService.findGlobalVocabulary()
            : vocabularyService.findOrCreateVocabulary(scopePath, scopePath.value());

    if (vocab == null) {
        return Response.status(Response.Status.INTERNAL_SERVER_ERROR)
                .entity(Map.of("error", "GLOBAL vocabulary not found — check Flyway V3 migration")).build();
    }

    final LabelDefinition def = vocabularyService.addDefinition(
            vocab.id, labelPath, request.description(),
            request.addedBy() != null ? request.addedBy() : "unknown");

    return Response.status(Response.Status.CREATED)
            .entity(Map.of("id", def.id, "path", def.path.value(), "scope", scopePath.value()))
            .build();
}
```

Key changes visible in the implementation:
- **Scope parsing:** null/blank → `Path.root()`, otherwise `Path.parse(scope)` with error handling.
- **Vocabulary name derivation:** auto-created vocabularies use `scopePath.value()` as the name — the path string IS the identity. The `name` parameter stays on `findOrCreateVocabulary` because it serves a display purpose ("HR Team Leave Vocabulary" vs "casehubio/hr-team"), but the REST endpoint derives it from the path for auto-creation.
- **Response serialization:** `scopePath.value()` returns a flat string, not a JSON object. Without `.value()`, Jackson would serialize `Path` as `{"value":"…","segments":[…]}`.
- **All GLOBAL/ORG/TEAM/PERSONAL branching eliminated** — the enum parsing, ownerId defaulting for PERSONAL, ownerId validation for ORG/TEAM, and scope-specific error messages all collapse to a single Path parse + root check.

**`GET /vocabulary` (listAll):** Currently hardcodes `VocabularyScope.PERSONAL` (= "show everything"). No `Path` equivalent of "deepest possible" exists. Change to list all definitions by scanning all vocabularies and flatmapping their definitions — no scope filter applied. This is what the endpoint was effectively doing (scope enforcement is deferred). The `listAccessible(Path)` method remains available for when scope enforcement is activated.

### Store/Repository Layer

No changes needed. `LabelVocabularyStore` SPI and implementations (JPA, InMemory) do generic CRUD without referencing `VocabularyScope`. The entity field type change is transparent via JPA converter.

### Flyway Migration — V36

File: `runtime/src/main/resources/db/work/migration/V36__vocabulary_scope_to_path.sql`

V36, not V5003. Per `docs/FLYWAY.md`, V1–V999 belongs to the runtime module (currently at V35). V5000–V5999 belongs to `casehub-work-issue-tracker`.

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

**`PathAttributeConverterTest` — root roundtrip test:** The root-path guard is the entire point of the converter change. Add test cases:

```java
@Test
void rootPath_roundtrips() {
    PathAttributeConverter converter = new PathAttributeConverter();
    // Path.root() → "" in database
    assertThat(converter.convertToDatabaseColumn(Path.root())).isEqualTo("");
    // "" in database → Path.root()
    assertThat(converter.convertToEntityAttribute("")).isEqualTo(Path.root());
}
```

### Cleanup

Delete `VocabularyScope.java`. All references eliminated by the changes above.

### ARC42STORIES.MD Updates

Three updates in the Layer 16 Key Files and Key Wiring sections:

1. **Remove VocabularyScope.java row** (~line 1452): file deleted.
2. **Update LabelVocabulary.java description** (~line 1450): change "scope enum GLOBAL/ORG/TEAM/PERSONAL" to "scope uses `Path` from `casehub-platform-api`; `PathAttributeConverter` stores as VARCHAR".
3. **Update Key Wiring section** (~line 1468): note that `LabelVocabulary.scope` now also uses `PathAttributeConverter` alongside `LabelDefinition.path`.

## Files Changed

| File | Change |
|------|--------|
| `runtime/.../model/VocabularyScope.java` | **Deleted** |
| `runtime/.../model/LabelVocabulary.java` | scope: enum → Path, drop ownerId |
| `runtime/.../model/PathAttributeConverter.java` | Root path guard in `convertToEntityAttribute` |
| `runtime/.../service/LabelVocabularyService.java` | All scope methods: enum → Path |
| `runtime/.../api/VocabularyResource.java` | Scope in body, drop ownerId, listAll without filter, full impl shown above |
| `runtime/.../resources/db/work/migration/V36__vocabulary_scope_to_path.sql` | Migration |
| `persistence-memory/.../InMemoryLabelVocabularyStore.java` | No changes needed |
| `runtime/test/.../LabelVocabularyServiceTest.java` | Rewrite scope tests |
| `runtime/test/.../PathAttributeConverterTest.java` | Add root roundtrip test |
| `runtime/test/.../JpaLabelVocabularyStoreTenancyTest.java` | GLOBAL → Path.root() |
| `runtime/test/.../JpaLabelDefinitionStoreTenancyTest.java` | GLOBAL → Path.root() |
| `examples/.../VocabularyScenario.java` | TEAM → Path.of("casehubio", TEAM_ID) |
| `ARC42STORIES.MD` | Remove VocabularyScope.java row, update LabelVocabulary description, update Key Wiring |

## Garden Entries Referenced

- `GE-20260522-a87fd7`: Path.parent() returns null for single-segment paths — root scope excluded from ancestor walk. Relevant to root handling but `isAncestorOf()` handles root correctly.

## Platform Coherence

- Removes parallel scope type per boundary rule
- Uses `Path` from `casehub-platform-api` as designed
- `VocabularyScope` is in `runtime/model/` (not `casehub-work-api`) — no external consumers, no cross-repo propagation
- No deferred concerns requiring separate issues
