# VocabularyScope → Path Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the VocabularyScope enum with platform Path-based scope hierarchy, fixing a multi-tenant bug and adding race-safe vocabulary creation.

**Architecture:** LabelVocabulary.scope changes from `@Enumerated VocabularyScope` to `@Convert Path`. The store SPI gains `findByScope(Path)` and `findOrCreate(Path, String)` with REQUIRES_NEW race safety in the JPA impl. The REST endpoint moves scope from URL path param to request body. VocabularyScope enum is deleted.

**Tech Stack:** Java 21, Quarkus 3.32, Hibernate/Panache, Flyway, H2 (test) / PostgreSQL (prod), RestAssured (integration tests)

**Spec:** `wksp/specs/2026-06-12-vocabulary-scope-path-design.md`

**Issue:** casehubio/work#236

---

### Task 1: PathAttributeConverter — root path fix

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/PathAttributeConverter.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/model/PathAttributeConverterTest.java`

- [ ] **Step 1: Write failing root roundtrip tests**

Add three tests to the existing `PathAttributeConverterTest`:

```java
@Test
void convertToEntityAttribute_emptyString_returnsRoot() {
    assertThat(converter.convertToEntityAttribute("")).isEqualTo(Path.root());
}

@Test
void convertToDatabaseColumn_root_returnsEmptyString() {
    assertThat(converter.convertToDatabaseColumn(Path.root())).isEqualTo("");
}

@Test
void rootPath_roundTrips() {
    String stored = converter.convertToDatabaseColumn(Path.root());
    Path restored = converter.convertToEntityAttribute(stored);
    assertThat(restored).isEqualTo(Path.root());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PathAttributeConverterTest -pl runtime`

Expected: `convertToEntityAttribute_emptyString_returnsRoot` FAILS with `IllegalArgumentException: Path must not be blank`

- [ ] **Step 3: Fix the converter**

Replace the `convertToEntityAttribute` method in `PathAttributeConverter.java`:

```java
@Override
public Path convertToEntityAttribute(final String value) {
    if (value == null) return null;
    if (value.isEmpty()) return Path.root();
    return Path.parse(value);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=PathAttributeConverterTest -pl runtime`

Expected: All 7 tests PASS

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/work/runtime/model/PathAttributeConverter.java runtime/src/test/java/io/casehub/work/runtime/model/PathAttributeConverterTest.java
git commit -m "fix(#236): PathAttributeConverter root path roundtrip — empty string ↔ Path.root()

Refs #236"
```

---

### Task 2: Flyway V36 migration

**Files:**
- Create: `runtime/src/main/resources/db/work/migration/V36__vocabulary_scope_to_path.sql`

- [ ] **Step 1: Create the migration file**

```sql
-- V36: Replace VocabularyScope enum with Path-based scope hierarchy (#236)
-- Converts scope column from enum strings to path strings,
-- drops the redundant owner_id column, and enforces uniqueness.

-- Widen scope column from VARCHAR(20) to VARCHAR(500) for path strings
ALTER TABLE label_vocabulary ALTER COLUMN scope TYPE VARCHAR(500);

-- Convert enum values to path strings:
-- GLOBAL (owner_id NULL) → '' (root path)
-- ORG/TEAM/PERSONAL → owner_id value (identity preserved, tier lost)
UPDATE label_vocabulary SET scope = '' WHERE scope = 'GLOBAL';
UPDATE label_vocabulary SET scope = owner_id WHERE scope IN ('ORG', 'TEAM', 'PERSONAL');

-- Drop owner_id — subsumed by scope path
ALTER TABLE label_vocabulary DROP COLUMN owner_id;

-- One vocabulary per scope path per tenant (follows V35 uq_work_item_template_name_tenant pattern)
CREATE UNIQUE INDEX uq_label_vocabulary_scope_tenant ON label_vocabulary(scope, tenancy_id);
```

- [ ] **Step 2: Commit**

```
git add runtime/src/main/resources/db/work/migration/V36__vocabulary_scope_to_path.sql
git commit -m "feat(#236): V36 migration — scope enum→path, drop owner_id, add UNIQUE(scope, tenancy_id)

Refs #236"
```

---

### Task 3: Core refactoring — entity, store SPI, store impls, service, resource, delete enum

This task changes the type of `LabelVocabulary.scope` from `VocabularyScope` to `Path` and migrates all callers. The project does not compile between sub-steps — all steps must complete before verification.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/LabelVocabulary.java`
- Delete: `runtime/src/main/java/io/casehub/work/runtime/model/VocabularyScope.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/LabelVocabularyStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaLabelVocabularyStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryLabelVocabularyStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/LabelVocabularyService.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/api/VocabularyResource.java`

- [ ] **Step 1: Modify LabelVocabulary — scope enum → Path, drop ownerId**

Replace the full file content of `LabelVocabulary.java`:

```java
package io.casehub.work.runtime.model;

import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Convert;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.casehub.platform.api.path.Path;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * A named, scoped container of {@link LabelDefinition} entries.
 *
 * <p>
 * Vocabularies form a visibility hierarchy using {@link Path}-based scoping.
 * A user can apply any label declared in a vocabulary whose scope is an ancestor
 * of (or equal to) their own scope path. Root scope ({@code Path.root()}) is
 * visible to everyone.
 */
@Entity
@Table(name = "label_vocabulary")
public class LabelVocabulary extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "tenancy_id", nullable = false)
    public String tenancyId;

    @Convert(converter = PathAttributeConverter.class)
    @Column(nullable = false, length = 500)
    public Path scope;

    /** Human-readable name for this vocabulary. */
    @Column(nullable = false, length = 255)
    public String name;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
    }
}
```

- [ ] **Step 2: Delete VocabularyScope.java**

Delete: `runtime/src/main/java/io/casehub/work/runtime/model/VocabularyScope.java`

- [ ] **Step 3: Add findByScope and findOrCreate to LabelVocabularyStore SPI**

Add the import and two new methods to `LabelVocabularyStore.java`. Insert after the existing `import` block:

```java
import io.casehub.platform.api.path.Path;
```

Insert after the `scanAll()` method (before `delete`):

```java
    /**
     * Find the vocabulary at the given scope for the current tenant.
     *
     * @param scope the scope path to search for
     * @return an {@link Optional} containing the vocabulary, or empty if not found
     */
    Optional<LabelVocabulary> findByScope(Path scope);

    /**
     * Find the vocabulary at the given scope for the current tenant, or create one
     * if none exists. Thread-safe: implementations must handle concurrent creation
     * (via database constraints, synchronization, or native upsert).
     *
     * @param scope the scope path
     * @param name  human-readable name for a newly created vocabulary
     * @return the existing or newly created vocabulary; never null
     */
    default LabelVocabulary findOrCreate(Path scope, String name) {
        return findByScope(scope).orElseGet(() -> {
            LabelVocabulary vocab = new LabelVocabulary();
            vocab.scope = scope;
            vocab.name = name;
            return put(vocab);
        });
    }
```

- [ ] **Step 4: Implement findByScope and findOrCreate in JpaLabelVocabularyStore**

Add imports at the top of `JpaLabelVocabularyStore.java`:

```java
import java.util.Optional;

import jakarta.persistence.PersistenceException;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;

import io.casehub.platform.api.path.Path;
```

Add the two methods after the existing `delete` method:

```java
    @Override
    public Optional<LabelVocabulary> findByScope(final Path scope) {
        return withTenantQuery(() ->
                LabelVocabulary.find("scope = ?1 AND tenancyId = ?2", scope, currentPrincipal.tenancyId())
                        .firstResultOptional());
    }

    @Override
    @Transactional(TxType.REQUIRES_NEW)
    public LabelVocabulary findOrCreate(final Path scope, final String name) {
        return withTenantQuery(() -> {
            final Optional<LabelVocabulary> existing = LabelVocabulary
                    .find("scope = ?1 AND tenancyId = ?2", scope, currentPrincipal.tenancyId())
                    .<LabelVocabulary>firstResultOptional();
            if (existing.isPresent()) {
                return existing.get();
            }
            final LabelVocabulary vocab = new LabelVocabulary();
            vocab.scope = scope;
            vocab.name = name;
            vocab.tenancyId = currentPrincipal.tenancyId();
            try {
                vocab.persistAndFlush();
                return vocab;
            } catch (PersistenceException e) {
                LabelVocabulary.getEntityManager().clear();
                return LabelVocabulary
                        .find("scope = ?1 AND tenancyId = ?2", scope, currentPrincipal.tenancyId())
                        .<LabelVocabulary>firstResultOptional()
                        .orElseThrow(() -> new IllegalStateException(
                                "Concurrent vocabulary creation failed", e));
            }
        });
    }
```

- [ ] **Step 5: Implement findByScope and findOrCreate in InMemoryLabelVocabularyStore**

Add import at the top of `InMemoryLabelVocabularyStore.java`:

```java
import io.casehub.platform.api.path.Path;
```

Add the two methods after the existing `delete` method:

```java
    @Override
    public Optional<LabelVocabulary> findByScope(final Path scope) {
        final String tenancyId = currentPrincipal.tenancyId();
        return store.values().stream()
                .filter(v -> tenancyId.equals(v.tenancyId))
                .filter(v -> scope.equals(v.scope))
                .findFirst();
    }

    @Override
    public LabelVocabulary findOrCreate(final Path scope, final String name) {
        final String tenancyId = currentPrincipal.tenancyId();
        synchronized (store) {
            final Optional<LabelVocabulary> existing = store.values().stream()
                    .filter(v -> tenancyId.equals(v.tenancyId))
                    .filter(v -> scope.equals(v.scope))
                    .findFirst();
            if (existing.isPresent()) {
                return existing.get();
            }
            final LabelVocabulary vocab = new LabelVocabulary();
            vocab.id = UUID.randomUUID();
            vocab.scope = scope;
            vocab.name = name;
            vocab.tenancyId = tenancyId;
            store.put(vocab.id, vocab);
            return vocab;
        }
    }
```

- [ ] **Step 6: Rewrite LabelVocabularyService**

Replace the full file content of `LabelVocabularyService.java`:

```java
package io.casehub.work.runtime.service;

import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.platform.api.path.Path;
import io.casehub.work.runtime.model.LabelDefinition;
import io.casehub.work.runtime.model.LabelVocabulary;
import io.casehub.work.runtime.repository.LabelDefinitionStore;
import io.casehub.work.runtime.repository.LabelVocabularyStore;

@ApplicationScoped
public class LabelVocabularyService {

    @Inject
    LabelDefinitionStore labelDefinitionStore;

    @Inject
    LabelVocabularyStore labelVocabularyStore;

    /**
     * Returns true if a vocabulary at {@code vocabScope} is accessible
     * to a caller operating at {@code callerScope}.
     *
     * <p>
     * A vocabulary is accessible when its scope is an ancestor of (or equal to)
     * the caller's scope. Root scope is accessible to everyone.
     */
    public static boolean isAccessibleFrom(final Path vocabScope, final Path callerScope) {
        return vocabScope.equals(callerScope) || vocabScope.isAncestorOf(callerScope);
    }

    /**
     * Returns true if the given label {@code path} matches the {@code pattern}.
     *
     * <ul>
     * <li>Exact: {@code "legal"} matches only {@code "legal"}</li>
     * <li>Single wildcard {@code "legal/*"}: one segment below, not multiple</li>
     * <li>Multi wildcard {@code "legal/**"}: any path below {@code "legal/"}</li>
     * </ul>
     */
    public static boolean matchesPattern(final String pattern, final String path) {
        if (pattern.endsWith("/**")) {
            final String prefix = pattern.substring(0, pattern.length() - 3);
            return path.startsWith(prefix + "/");
        }
        if (pattern.endsWith("/*")) {
            final String prefix = pattern.substring(0, pattern.length() - 2);
            if (!path.startsWith(prefix + "/")) {
                return false;
            }
            final String remainder = path.substring(prefix.length() + 1);
            return !remainder.contains("/");
        }
        return pattern.equals(path);
    }

    /**
     * Returns true if the given path is declared in any accessible vocabulary.
     * Scope enforcement is deferred — any declared path is currently accepted.
     */
    @Transactional
    public boolean isDeclared(final Path path) {
        return !labelDefinitionStore.findByPath(path).isEmpty();
    }

    /**
     * Add a new label definition to the given vocabulary.
     */
    @Transactional
    public LabelDefinition addDefinition(final UUID vocabularyId, final Path path,
            final String description, final String createdBy) {
        final LabelDefinition def = new LabelDefinition();
        def.path = path;
        def.vocabularyId = vocabularyId;
        def.description = description;
        def.createdBy = createdBy;
        return labelDefinitionStore.put(def);
    }

    /**
     * List all label definitions accessible at the given scope (at or above).
     */
    @Transactional
    public List<LabelDefinition> listAccessible(final Path callerScope) {
        return labelVocabularyStore.scanAll().stream()
                .filter(v -> isAccessibleFrom(v.scope, callerScope))
                .flatMap(v -> labelDefinitionStore.findByVocabularyId(v.id).stream())
                .toList();
    }

    /**
     * List all label definitions across all vocabularies visible to the current tenant.
     * No scope filtering — returns everything.
     */
    @Transactional
    public List<LabelDefinition> listAllDefinitions() {
        return labelVocabularyStore.scanAll().stream()
                .flatMap(v -> labelDefinitionStore.findByVocabularyId(v.id).stream())
                .toList();
    }

    /**
     * Find an existing vocabulary at the given scope, or create one.
     * Delegates to the store's race-safe {@code findOrCreate}.
     */
    @Transactional
    public LabelVocabulary findOrCreateVocabulary(final Path scope, final String name) {
        return labelVocabularyStore.findOrCreate(scope, name);
    }
}
```

- [ ] **Step 7: Rewrite VocabularyResource**

Replace the full file content of `VocabularyResource.java`:

```java
package io.casehub.work.runtime.api;

import java.util.List;
import java.util.Map;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.casehub.work.runtime.model.LabelDefinition;
import io.casehub.work.runtime.model.LabelVocabulary;
import io.casehub.work.runtime.service.LabelVocabularyService;

@Path("/vocabulary")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class VocabularyResource {

    @Inject
    LabelVocabularyService vocabularyService;

    public record AddDefinitionRequest(String path, String description, String addedBy, String scope) {
    }

    /**
     * List all label definitions visible to the current tenant.
     * No scope filtering — scope enforcement is deferred.
     */
    @GET
    public List<Map<String, Object>> listAll() {
        return vocabularyService.listAllDefinitions().stream()
                .map(d -> Map.<String, Object> of(
                        "id", d.id,
                        "path", d.path.value(),
                        "vocabularyId", d.vocabularyId,
                        "description", d.description != null ? d.description : "",
                        "createdBy", d.createdBy,
                        "createdAt", d.createdAt))
                .toList();
    }

    /**
     * Add a label definition to the vocabulary at the given scope.
     * Scope is a Path string in the request body (null/blank = root/global).
     */
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

        final String vocabName = scopePath.value().isEmpty() ? "Global" : scopePath.value();
        final LabelVocabulary vocab = vocabularyService.findOrCreateVocabulary(scopePath, vocabName);

        final LabelDefinition def = vocabularyService.addDefinition(
                vocab.id, labelPath, request.description(),
                request.addedBy() != null ? request.addedBy() : "unknown");

        return Response.status(Response.Status.CREATED)
                .entity(Map.of("id", def.id, "path", def.path.value(), "scope", scopePath.value()))
                .build();
    }
}
```

- [ ] **Step 8: Verify main sources compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime`

Expected: BUILD SUCCESS. Test sources will not compile yet — that is expected.

Also verify persistence-memory:

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl persistence-memory`

Expected: BUILD SUCCESS.

- [ ] **Step 9: Commit**

```
git add -A
git commit -m "feat(#236): replace VocabularyScope with Path — entity, store SPI, service, resource

- LabelVocabulary.scope: VocabularyScope enum → Path with PathAttributeConverter
- Drop ownerId field — path encodes ownership hierarchy
- Delete VocabularyScope enum
- LabelVocabularyStore: add findByScope(Path), findOrCreate(Path, String)
- JpaLabelVocabularyStore: REQUIRES_NEW race-safe findOrCreate
- InMemoryLabelVocabularyStore: synchronized findOrCreate
- LabelVocabularyService: isAccessibleFrom(Path, Path), listAllDefinitions(), delete findGlobalVocabulary()
- VocabularyResource: scope in body, unified findOrCreateVocabulary

Refs #236"
```

---

### Task 4: Unit tests — LabelVocabularyServiceTest

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/LabelVocabularyServiceTest.java`

- [ ] **Step 1: Rewrite LabelVocabularyServiceTest**

Replace the full file content:

```java
package io.casehub.work.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.path.Path;

class LabelVocabularyServiceTest {

    @Test
    void isAccessibleFrom_rootVisibleToAll() {
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.root(), Path.of("acme-corp"))).isTrue();
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.root(), Path.of("acme-corp", "hr-team"))).isTrue();
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.root(), Path.of("acme-corp", "hr-team", "jane"))).isTrue();
    }

    @Test
    void isAccessibleFrom_rootVisibleToRoot() {
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.root(), Path.root())).isTrue();
    }

    @Test
    void isAccessibleFrom_ancestorVisible() {
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp"), Path.of("acme-corp", "hr-team"))).isTrue();
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp"), Path.of("acme-corp", "hr-team", "jane"))).isTrue();
    }

    @Test
    void isAccessibleFrom_equalVisible() {
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp"), Path.of("acme-corp"))).isTrue();
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp", "hr-team"), Path.of("acme-corp", "hr-team"))).isTrue();
    }

    @Test
    void isAccessibleFrom_deeperNotVisibleToShallower() {
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp", "hr-team"), Path.of("acme-corp"))).isFalse();
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp", "hr-team", "jane"), Path.of("acme-corp"))).isFalse();
    }

    @Test
    void isAccessibleFrom_siblingNotVisible() {
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp"), Path.of("globex"))).isFalse();
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp", "hr-team"), Path.of("acme-corp", "sales"))).isFalse();
    }

    @Test
    void isAccessibleFrom_rootCallerSeesOnlyRoot() {
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp"), Path.root())).isFalse();
        assertThat(LabelVocabularyService.isAccessibleFrom(Path.of("acme-corp", "hr-team"), Path.root())).isFalse();
    }

    @Test
    void matchesPattern_exactMatch() {
        assertThat(LabelVocabularyService.matchesPattern("legal", "legal")).isTrue();
        assertThat(LabelVocabularyService.matchesPattern("legal", "legal/contracts")).isFalse();
    }

    @Test
    void matchesPattern_singleWildcard() {
        assertThat(LabelVocabularyService.matchesPattern("legal/*", "legal/contracts")).isTrue();
        assertThat(LabelVocabularyService.matchesPattern("legal/*", "legal/contracts/nda")).isFalse();
        assertThat(LabelVocabularyService.matchesPattern("legal/*", "legal")).isFalse();
    }

    @Test
    void matchesPattern_multiWildcard() {
        assertThat(LabelVocabularyService.matchesPattern("legal/**", "legal/contracts")).isTrue();
        assertThat(LabelVocabularyService.matchesPattern("legal/**", "legal/contracts/nda")).isTrue();
        assertThat(LabelVocabularyService.matchesPattern("legal/**", "legal")).isFalse();
    }
}
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LabelVocabularyServiceTest -pl runtime`

Expected: All 10 tests PASS

- [ ] **Step 3: Commit**

```
git add runtime/src/test/java/io/casehub/work/runtime/service/LabelVocabularyServiceTest.java
git commit -m "test(#236): rewrite LabelVocabularyServiceTest — ordinal hierarchy → Path ancestor tests

Refs #236"
```

---

### Task 5: Tenancy tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaLabelVocabularyStoreTenancyTest.java`
- Modify: `runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaLabelDefinitionStoreTenancyTest.java`

- [ ] **Step 1: Update JpaLabelVocabularyStoreTenancyTest**

Replace the import and `newVocabulary` helper. Change:

```java
import io.casehub.work.runtime.model.VocabularyScope;
```

to:

```java
import io.casehub.platform.api.path.Path;
```

Replace the `newVocabulary` method:

```java
    private LabelVocabulary newVocabulary(String name) {
        LabelVocabulary vocab = new LabelVocabulary();
        vocab.scope = Path.root();
        vocab.name = name;
        return vocab;
    }
```

- [ ] **Step 2: Update JpaLabelDefinitionStoreTenancyTest**

Replace the import. Change:

```java
import io.casehub.work.runtime.model.VocabularyScope;
```

to:

```java
import io.casehub.platform.api.path.Path;
```

Replace the `createVocabulary` helper:

```java
    private LabelVocabulary createVocabulary(String name) {
        LabelVocabulary vocab = new LabelVocabulary();
        vocab.scope = Path.root();
        vocab.name = name;
        return vocabularyStore.put(vocab);
    }
```

- [ ] **Step 3: Run tenancy tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JpaLabelVocabularyStoreTenancyTest -pl runtime`

Expected: All 4 tests PASS

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=JpaLabelDefinitionStoreTenancyTest -pl runtime`

Expected: All 5 tests PASS

- [ ] **Step 4: Commit**

```
git add runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaLabelVocabularyStoreTenancyTest.java runtime/src/test/java/io/casehub/work/runtime/repository/jpa/JpaLabelDefinitionStoreTenancyTest.java
git commit -m "test(#236): update tenancy tests — VocabularyScope.GLOBAL → Path.root()

Refs #236"
```

---

### Task 6: Integration tests — LabelEndpointTest

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/api/LabelEndpointTest.java`

- [ ] **Step 1: Rewrite vocabulary test methods**

The following tests must change because the endpoint moved from `POST /vocabulary/{scope}` to `POST /vocabulary` with scope in the body:

**Delete these tests entirely** (conceptually invalidated):
- `vocabulary_addDefinition_invalidScope_returns400` — "INVALID" is a valid single-segment Path
- `vocabulary_addDefinition_personal_defaultsOwnerToAddedBy` — PERSONAL-to-addedBy defaulting removed
- `vocabulary_addDefinition_org_missingOwnerId_returns400` — ownerId validation removed

**Update these tests** — change URL from `/vocabulary/GLOBAL` (or `/vocabulary/TEAM` etc.) to `/vocabulary`, add scope to request body, update response assertions:

Replace `vocabulary_addDefinition_appearsInList`:

```java
    @Test
    void vocabulary_addDefinition_appearsInList() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "test/unique-vocab-54", "description": "test label", "addedBy": "alice"}
                        """)
                .post("/vocabulary")
                .then()
                .statusCode(201)
                .body("path", equalTo("test/unique-vocab-54"));

        given()
                .get("/vocabulary")
                .then()
                .statusCode(200)
                .body("path", org.hamcrest.Matchers.hasItem("test/unique-vocab-54"));
    }
```

Replace `vocabulary_addDefinition_org_withOwnerId_succeeds`:

```java
    @Test
    void vocabulary_addDefinition_orgScope_succeeds() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "org/finance/approvals", "addedBy": "alice", "scope": "acme-corp"}
                        """)
                .post("/vocabulary")
                .then()
                .statusCode(201)
                .body("path", equalTo("org/finance/approvals"))
                .body("scope", equalTo("acme-corp"));
    }
```

Replace `vocabulary_addDefinition_team_withOwnerId_succeeds`:

```java
    @Test
    void vocabulary_addDefinition_teamScope_succeeds() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "team/sprint/review", "addedBy": "bob", "scope": "acme-corp/team-alpha"}
                        """)
                .post("/vocabulary")
                .then()
                .statusCode(201)
                .body("path", equalTo("team/sprint/review"))
                .body("scope", equalTo("acme-corp/team-alpha"));
    }
```

Replace `vocabulary_addDefinition_scopedTerm_appearsInListAll`:

```java
    @Test
    void vocabulary_addDefinition_scopedTerm_appearsInListAll() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "my/team/label", "addedBy": "charlie", "scope": "acme-corp/team-bravo"}
                        """)
                .post("/vocabulary")
                .then()
                .statusCode(201);

        given()
                .get("/vocabulary")
                .then()
                .statusCode(200)
                .body("path", org.hamcrest.Matchers.hasItem("my/team/label"));
    }
```

Replace `vocabulary_addDefinition_sameOwner_reuseVocabulary`:

```java
    @Test
    void vocabulary_addDefinition_sameScope_reuseVocabulary() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "personal/label/one", "addedBy": "dave", "scope": "casehubio/dave"}
                        """)
                .post("/vocabulary")
                .then()
                .statusCode(201);

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "personal/label/two", "addedBy": "dave", "scope": "casehubio/dave"}
                        """)
                .post("/vocabulary")
                .then()
                .statusCode(201);

        given()
                .get("/vocabulary")
                .then()
                .statusCode(200)
                .body("path", org.hamcrest.Matchers.hasItem("personal/label/one"))
                .body("path", org.hamcrest.Matchers.hasItem("personal/label/two"));
    }
```

Replace `vocabulary_addDefinition_wildcardPath_returns400`:

```java
    @Test
    void vocabulary_addDefinition_wildcardPath_returns400() {
        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "legal/*", "addedBy": "alice"}
                        """)
                .post("/vocabulary")
                .then()
                .statusCode(400)
                .body("error", org.hamcrest.Matchers.containsString("wildcard"));

        given()
                .contentType(ContentType.JSON)
                .body("""
                        {"path": "legal/?", "addedBy": "alice"}
                        """)
                .post("/vocabulary")
                .then()
                .statusCode(400)
                .body("error", org.hamcrest.Matchers.containsString("wildcard"));
    }
```

Replace `vocabulary_addDefinition_pathStoredAndRetrievableByPath`:

```java
    @Test
    void vocabulary_addDefinition_pathStoredAndRetrievableByPath() {
        String uniquePath = "test/converter-round-trip-99";

        given()
                .contentType(ContentType.JSON)
                .body("{\"path\": \"" + uniquePath + "\", \"description\": \"converter test\", \"addedBy\": \"alice\"}")
                .post("/vocabulary")
                .then()
                .statusCode(201)
                .body("path", equalTo(uniquePath));

        given()
                .get("/vocabulary")
                .then()
                .statusCode(200)
                .body("path", org.hamcrest.Matchers.hasItem(uniquePath));
    }
```

Replace `vocabulary_addDefinition_emptyPath_returns400`:

```java
    @Test
    void vocabulary_addDefinition_emptyPath_returns400() {
        given()
                .contentType(ContentType.JSON)
                .body("{\"path\": \"\", \"addedBy\": \"alice\"}")
                .post("/vocabulary")
                .then()
                .statusCode(400);
    }
```

Remove the `VocabularyScope` import — it is no longer needed in this file.

- [ ] **Step 2: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=LabelEndpointTest -pl runtime`

Expected: All tests PASS (17 tests — 20 original minus 3 deleted)

- [ ] **Step 3: Commit**

```
git add runtime/src/test/java/io/casehub/work/runtime/api/LabelEndpointTest.java
git commit -m "test(#236): rewrite LabelEndpointTest — scope in body, delete invalidated tests

Deleted: invalidScope (valid Path), personal_defaultsOwnerToAddedBy, org_missingOwnerId
Refs #236"
```

---

### Task 7: VocabularyScenario example

**Files:**
- Modify: `examples/src/main/java/io/casehub/work/examples/vocabulary/VocabularyScenario.java`

- [ ] **Step 1: Update VocabularyScenario**

Replace the `VocabularyScope` import:

```java
import io.casehub.work.runtime.model.VocabularyScope;
```

with (no replacement — remove the import entirely).

Replace line 93 (the `findOrCreateVocabulary` call):

```java
        final LabelVocabulary vocab = vocabularyService.findOrCreateVocabulary(
                VocabularyScope.TEAM, TEAM_ID, "HR Team Leave Vocabulary");
```

with:

```java
        final LabelVocabulary vocab = vocabularyService.findOrCreateVocabulary(
                io.casehub.platform.api.path.Path.of("casehubio", TEAM_ID), "HR Team Leave Vocabulary");
```

Replace line 109 (the `listAccessible` call):

```java
        final List<LabelDefinition> accessible = vocabularyService.listAccessible(VocabularyScope.TEAM);
```

with:

```java
        final List<LabelDefinition> accessible = vocabularyService.listAccessible(
                io.casehub.platform.api.path.Path.of("casehubio", TEAM_ID));
```

- [ ] **Step 2: Verify examples compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl examples`

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```
git add examples/src/main/java/io/casehub/work/examples/vocabulary/VocabularyScenario.java
git commit -m "refactor(#236): VocabularyScenario — VocabularyScope.TEAM → Path.of(casehubio, hr-team)

Refs #236"
```

---

### Task 8: Documentation — ARC42STORIES.MD and api-reference.md

**Files:**
- Modify: `ARC42STORIES.MD` (lines ~1449–1465)
- Modify: `docs/api-reference.md` (lines ~601–643)

- [ ] **Step 1: Update ARC42STORIES.MD Key Files table**

At ~line 1449, replace the `LabelVocabulary.java` row:

```
| `runtime/src/main/java/io/casehub/work/runtime/model/LabelVocabulary.java` | Scoped container for LabelDefinition entries; scope enum GLOBAL/ORG/TEAM/PERSONAL |
```

with:

```
| `runtime/src/main/java/io/casehub/work/runtime/model/LabelVocabulary.java` | Scoped container for LabelDefinition entries; `scope` uses `Path` from `casehub-platform-api` with `PathAttributeConverter` |
```

At ~line 1452, delete the `VocabularyScope.java` row entirely:

```
| `runtime/src/main/java/io/casehub/work/runtime/model/VocabularyScope.java` | Enum: GLOBAL → ORG → TEAM → PERSONAL visibility hierarchy |
```

At ~line 1465, in the Key Wiring paragraph about `LabelDefinition.path`, append a sentence:

After `VocabularyResource` must use the fully qualified name...`:

```
`LabelVocabulary.scope` also uses `PathAttributeConverter` — root scope stores as empty string, non-root as the path value string.
```

- [ ] **Step 2: Update docs/api-reference.md Vocabulary API section**

Replace the Vocabulary API section (~lines 601–643) with:

```markdown
## Vocabulary API (quarkus-work core)

Labels must be declared in a vocabulary before they can be applied. Vocabularies are scoped using `Path`-based hierarchy — a vocabulary at an ancestor path is visible to all descendant scopes. Root scope (empty/null) is visible to everyone. A root vocabulary is auto-created per tenant on first use, seeded with common labels (`intake`, `intake/triage`, `priority/high`, `priority/critical`, `legal`, `legal/contracts`, `legal/compliance`).

---

### GET /vocabulary

Lists all label definitions visible to the current tenant (all scopes, no filtering).

**Response:** `200 OK` — array of definition objects: `{id, path, vocabularyId, description, createdBy, createdAt}`

```bash
curl http://localhost:8080/vocabulary
```

---

### POST /vocabulary

Adds a label definition to the vocabulary at the given scope. If no vocabulary exists at the scope, one is auto-created.

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `path` | string | yes | Label path to declare, e.g. `finance/invoices` |
| `description` | string | no | Human-readable description |
| `addedBy` | string | no | User ID declaring the label |
| `scope` | string | no | Scope path for the vocabulary (null/blank = root/global), e.g. `acme-corp/hr-team` |

**Response:** `201 Created` — `{id, path, scope}`

**Errors:** `400` if path is blank, contains wildcards, or scope is invalid.

```bash
# Add to root (global) vocabulary
curl -X POST http://localhost:8080/vocabulary \
  -H 'Content-Type: application/json' \
  -d '{"path": "finance/invoices", "description": "Invoice approval items", "addedBy": "alice"}'

# Add to org-scoped vocabulary
curl -X POST http://localhost:8080/vocabulary \
  -H 'Content-Type: application/json' \
  -d '{"path": "hr/leave", "description": "Leave requests", "addedBy": "alice", "scope": "acme-corp"}'

# Add to team-scoped vocabulary
curl -X POST http://localhost:8080/vocabulary \
  -H 'Content-Type: application/json' \
  -d '{"path": "sprint/review", "description": "Sprint reviews", "addedBy": "bob", "scope": "acme-corp/hr-team"}'
```
```

- [ ] **Step 3: Commit**

```
git add ARC42STORIES.MD docs/api-reference.md
git commit -m "docs(#236): update ARC42STORIES + api-reference for Path-based vocabulary scope

Refs #236"
```

---

### Task 9: Full build verification

- [ ] **Step 1: Run full runtime module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All tests PASS. This covers: PathAttributeConverterTest, LabelVocabularyServiceTest, JpaLabelVocabularyStoreTenancyTest, JpaLabelDefinitionStoreTenancyTest, LabelEndpointTest, and all other runtime tests.

- [ ] **Step 2: Run persistence-memory module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`

Expected: BUILD SUCCESS (compilation only — persistence-memory has no tests, but compile verifies InMemoryLabelVocabularyStore).

- [ ] **Step 3: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

Expected: BUILD SUCCESS across all modules. Verify no VocabularyScope references remain anywhere.
