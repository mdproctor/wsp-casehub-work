# persistence-memory Module Extraction — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract in-memory store implementations from `testing/` into a new `persistence-memory/` module with correct CDI tier, thread-safe data structures, and updated documentation.

**Architecture:** `git mv testing/ persistence-memory/`, then refactor package names, update CDI annotations to `@Alternative @Priority(100)` (new Tier 3), replace `LinkedHashMap` with `ConcurrentHashMap`, restructure `InMemoryAuditEntryStore` from flat list to per-workItemId keyed concurrent map.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, Maven, ConcurrentHashMap, CopyOnWriteArrayList

**Spec:** `docs/superpowers/specs/2026-06-06-persistence-memory-module-design.md`

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`

**Test scripts:** Use `scripts/` helper scripts where available (they enforce hard timeouts).

---

### Task 1: Module rename via git mv

Rename the module directory to preserve file history.

**Files:**
- Rename: `testing/` → `persistence-memory/`

- [ ] **Step 1: git mv the module directory**

```bash
git mv testing/ persistence-memory/
```

- [ ] **Step 2: Commit the rename**

```bash
git add -A
git commit -m "refactor: git mv testing/ → persistence-memory/ (Refs #191)"
```

This preserves `git log --follow` history for all files. Package names and pom.xml are updated in subsequent tasks.

---

### Task 2: Update pom.xml — module identity and dependencies

Update the module's own pom.xml and the root pom.xml.

**Files:**
- Modify: `persistence-memory/pom.xml`
- Modify: `pom.xml` (root)

- [ ] **Step 1: Replace persistence-memory/pom.xml**

Replace the entire contents of `persistence-memory/pom.xml` with:

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-work-persistence-memory</artifactId>
  <name>CaseHub Work - In-Memory Persistence</name>
  <description>In-memory store implementations for CaseHub Work. Thread-safe, ephemeral (data lost on
restart). Tier 3 in the CDI priority ladder — beats JPA and MongoDB when on the classpath.
Use for tests, demos, and local evaluation without a datasource.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-work</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-work-issue-tracker</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>jakarta.enterprise</groupId>
      <artifactId>jakarta.enterprise.cdi-api</artifactId>
      <scope>provided</scope>
    </dependency>
    <!-- Testing of the persistence-memory module itself -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
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
            <id>make-index</id>
            <goals>
              <goal>jandex</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Update root pom.xml — replace testing with persistence-memory**

In `pom.xml` (root), replace `<module>testing</module>` with `<module>persistence-memory</module>`. Place it after `persistence-mongodb`:

```xml
    <module>persistence-mongodb</module>
    <module>persistence-memory</module>
```

Remove the old `<module>testing</module>` line (was at line 22).

- [ ] **Step 3: Verify the module compiles (will fail on package — expected)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl persistence-memory -am 2>&1 | tail -20
```

Expected: compilation errors about package `io.casehub.work.testing` — the sources still use the old package. This confirms Maven wiring is correct.

- [ ] **Step 4: Commit**

```bash
git add pom.xml persistence-memory/pom.xml
git commit -m "build: update pom.xml — module identity and dependencies (Refs #191)"
```

---

### Task 3: Package rename — main sources

Move all main sources from `io.casehub.work.testing` to `io.casehub.work.memory` and update CDI annotations.

**Files:**
- Move: `persistence-memory/src/main/java/io/casehub/work/testing/*.java` → `persistence-memory/src/main/java/io/casehub/work/memory/`
- Delete: `persistence-memory/src/main/java/io/casehub/work/testing/` (empty after move)

- [ ] **Step 1: Create the new package directory**

```bash
mkdir -p persistence-memory/src/main/java/io/casehub/work/memory
```

- [ ] **Step 2: Move all 5 store files to the new package**

```bash
git mv persistence-memory/src/main/java/io/casehub/work/testing/InMemoryWorkItemStore.java persistence-memory/src/main/java/io/casehub/work/memory/
git mv persistence-memory/src/main/java/io/casehub/work/testing/InMemoryAuditEntryStore.java persistence-memory/src/main/java/io/casehub/work/memory/
git mv persistence-memory/src/main/java/io/casehub/work/testing/InMemoryWorkItemNoteStore.java persistence-memory/src/main/java/io/casehub/work/memory/
git mv persistence-memory/src/main/java/io/casehub/work/testing/InMemoryIssueLinkStore.java persistence-memory/src/main/java/io/casehub/work/memory/
git mv persistence-memory/src/main/java/io/casehub/work/testing/InMemoryRoutingCursorStore.java persistence-memory/src/main/java/io/casehub/work/memory/
```

- [ ] **Step 3: Remove the empty old package directory**

```bash
rm -rf persistence-memory/src/main/java/io/casehub/work/testing
```

- [ ] **Step 4: Update package declarations and CDI annotations in all 5 files**

In each file, change:
- `package io.casehub.work.testing;` → `package io.casehub.work.memory;`
- `@Priority(1)` → `@Priority(100)`

Update Javadoc in each file:
- Remove "Not thread-safe — designed for single-threaded test use only." (thread safety is added in Task 5)
- Replace "quarkus-work-testing" / "casehub-work-testing" references with "casehub-work-persistence-memory"
- Replace "Call in @BeforeEach to isolate tests" with "Available for test isolation (@BeforeEach) and administrative reset."

**InMemoryWorkItemStore.java** — new Javadoc block:

```java
/**
 * In-memory implementation of {@link WorkItemStore} for ephemeral deployments
 * and tests. No datasource or Flyway configuration required.
 *
 * <p>
 * Tier 3 in the CDI priority ladder — {@code @Alternative @Priority(100)} beats
 * both JPA (Tier 1) and MongoDB (Tier 2) when on the classpath.
 *
 * <p>
 * Thread-safe. Data is ephemeral (lost on restart). Objects returned from the
 * store are shared references — concurrent field-level mutations to the same
 * object without calling {@link #put} are not guaranteed to be visible across
 * threads.
 */
```

**InMemoryAuditEntryStore.java** — new Javadoc block:

```java
/**
 * In-memory implementation of {@link AuditEntryStore} for ephemeral deployments
 * and tests. No datasource or Flyway configuration required.
 *
 * <p>
 * Tier 3 in the CDI priority ladder — {@code @Alternative @Priority(100)} beats
 * both JPA (Tier 1) and MongoDB (Tier 2) when on the classpath.
 *
 * <p>
 * Thread-safe. Data is ephemeral (lost on restart).
 *
 * <p>
 * <strong>Known limitation:</strong> Category filter in {@link AuditQuery} is
 * silently ignored — the filter requires access to the parent WorkItem's category,
 * which would create an inter-store dependency.
 */
```

**InMemoryIssueLinkStore.java** — new Javadoc block:

```java
/**
 * In-memory implementation of {@link IssueLinkStore} for ephemeral deployments
 * and tests. No datasource or Flyway configuration required.
 *
 * <p>
 * Tier 3 in the CDI priority ladder — {@code @Alternative @Priority(100)} beats
 * JPA (Tier 1) when on the classpath. No Tier 2 (MongoDB) exists for this SPI
 * yet (tracked as casehubio/work#253).
 *
 * <p>
 * Thread-safe. Data is ephemeral (lost on restart).
 */
```

**InMemoryWorkItemNoteStore.java** — new Javadoc block:

```java
/**
 * In-memory implementation of {@link WorkItemNoteStore} for ephemeral deployments
 * and tests. No datasource or Flyway configuration required.
 *
 * <p>
 * Tier 3 in the CDI priority ladder — {@code @Alternative @Priority(100)} beats
 * JPA (Tier 1) when on the classpath. No Tier 2 (MongoDB) exists for this SPI
 * yet (tracked as casehubio/work#253).
 *
 * <p>
 * Thread-safe. Data is ephemeral (lost on restart).
 */
```

**InMemoryRoutingCursorStore.java** — new Javadoc block:

```java
/**
 * In-memory {@link RoutingCursorStore} for ephemeral deployments and tests.
 *
 * <p>
 * Tier 3 in the CDI priority ladder — {@code @Alternative @Priority(100)} beats
 * JPA (Tier 1) when on the classpath. No Tier 2 (MongoDB) exists for this SPI
 * yet (tracked as casehubio/work#253).
 *
 * <p>
 * Thread-safe (lock-free CAS loop). Data is ephemeral (lost on restart).
 */
```

Also update `clear()` / `reset()` Javadoc in all files:

```java
/** Removes all stored entries. Available for test isolation ({@code @BeforeEach}) and administrative reset. */
```

- [ ] **Step 5: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl persistence-memory -am 2>&1 | tail -10
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "refactor: rename package to io.casehub.work.memory, Priority(100) (Refs #191)"
```

---

### Task 4: Package rename — test sources

Move tests from `io.casehub.work.testing` to `io.casehub.work.memory`.

**Files:**
- Move: `persistence-memory/src/test/java/io/casehub/work/testing/*.java` → `persistence-memory/src/test/java/io/casehub/work/memory/`
- Delete: `persistence-memory/src/test/java/io/casehub/work/testing/` (empty after move)

- [ ] **Step 1: Create the new test package directory**

```bash
mkdir -p persistence-memory/src/test/java/io/casehub/work/memory
```

- [ ] **Step 2: Move the 2 test files**

```bash
git mv persistence-memory/src/test/java/io/casehub/work/testing/InMemoryRepositoryTest.java persistence-memory/src/test/java/io/casehub/work/memory/
git mv persistence-memory/src/test/java/io/casehub/work/testing/InMemoryIssueLinkStoreTest.java persistence-memory/src/test/java/io/casehub/work/memory/
```

- [ ] **Step 3: Remove the empty old test package directory**

```bash
rm -rf persistence-memory/src/test/java/io/casehub/work/testing
```

- [ ] **Step 4: Update package declarations**

In both files, change:
- `package io.casehub.work.testing;` → `package io.casehub.work.memory;`

Update imports:
- `import io.casehub.work.testing.InMemoryWorkItemStore;` → remove (same package)
- `import io.casehub.work.testing.InMemoryAuditEntryStore;` → remove (same package)
- `import io.casehub.work.testing.InMemoryIssueLinkStore;` → remove (same package)

These imports are unnecessary when the test is in the same package as the class.

- [ ] **Step 5: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory 2>&1 | tail -20
```

Expected: All tests pass (existing behavior unchanged).

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "refactor: move tests to io.casehub.work.memory package (Refs #191)"
```

---

### Task 5: Thread-safe data structures — InMemoryWorkItemStore

Replace `LinkedHashMap` with `ConcurrentHashMap`.

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemStore.java`

- [ ] **Step 1: Write a concurrency test**

Add to `InMemoryRepositoryTest.java`:

```java
@Test
void put_concurrentWrites_nothingLost() throws Exception {
    final int threads = 8;
    final int perThread = 100;
    final var latch = new java.util.concurrent.CountDownLatch(1);
    final var executor = java.util.concurrent.Executors.newFixedThreadPool(threads);
    final var futures = new java.util.ArrayList<java.util.concurrent.Future<?>>();

    for (int t = 0; t < threads; t++) {
        futures.add(executor.submit(() -> {
            try { latch.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            for (int i = 0; i < perThread; i++) {
                workItemStore.put(workItem(WorkItemStatus.PENDING));
            }
        }));
    }
    latch.countDown();
    for (var f : futures) { f.get(5, java.util.concurrent.TimeUnit.SECONDS); }
    executor.shutdown();

    assertThat(workItemStore.findAll()).hasSize(threads * perThread);
}
```

- [ ] **Step 2: Run the test — expect failure (LinkedHashMap is not thread-safe)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryRepositoryTest#put_concurrentWrites_nothingLost 2>&1 | tail -20
```

Expected: FAIL (likely ConcurrentModificationException or wrong count).

- [ ] **Step 3: Replace LinkedHashMap with ConcurrentHashMap**

In `InMemoryWorkItemStore.java`, change:

```java
import java.util.LinkedHashMap;
```
to:
```java
import java.util.concurrent.ConcurrentHashMap;
```

And change the field:

```java
private final Map<UUID, WorkItem> store = new LinkedHashMap<>();
```
to:
```java
private final Map<UUID, WorkItem> store = new ConcurrentHashMap<>();
```

- [ ] **Step 4: Run tests — expect all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryRepositoryTest 2>&1 | tail -20
```

Expected: All tests pass including the new concurrency test.

- [ ] **Step 5: Commit**

```bash
git add persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemStore.java persistence-memory/src/test/java/io/casehub/work/memory/InMemoryRepositoryTest.java
git commit -m "feat: thread-safe InMemoryWorkItemStore — ConcurrentHashMap (Refs #191)"
```

---

### Task 6: Thread-safe data structures — InMemoryIssueLinkStore

Replace `LinkedHashMap` with `ConcurrentHashMap`.

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryIssueLinkStore.java`

- [ ] **Step 1: Write a concurrency test**

Add to `InMemoryIssueLinkStoreTest.java`:

```java
@Test
void save_concurrentWrites_nothingLost() throws Exception {
    final int threads = 8;
    final int perThread = 100;
    final var latch = new java.util.concurrent.CountDownLatch(1);
    final var executor = java.util.concurrent.Executors.newFixedThreadPool(threads);
    final var futures = new java.util.ArrayList<java.util.concurrent.Future<?>>();
    final UUID workItemId = UUID.randomUUID();

    for (int t = 0; t < threads; t++) {
        final int threadNum = t;
        futures.add(executor.submit(() -> {
            try { latch.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            for (int i = 0; i < perThread; i++) {
                store.save(link(workItemId, "github", "owner/repo#" + threadNum + "-" + i));
            }
        }));
    }
    latch.countDown();
    for (var f : futures) { f.get(5, java.util.concurrent.TimeUnit.SECONDS); }
    executor.shutdown();

    assertThat(store.findByWorkItemId(workItemId)).hasSize(threads * perThread);
}
```

- [ ] **Step 2: Run test — expect failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryIssueLinkStoreTest#save_concurrentWrites_nothingLost 2>&1 | tail -20
```

- [ ] **Step 3: Replace LinkedHashMap with ConcurrentHashMap**

In `InMemoryIssueLinkStore.java`, change:

```java
import java.util.LinkedHashMap;
```
to:
```java
import java.util.concurrent.ConcurrentHashMap;
```

And change the field:

```java
private final Map<UUID, WorkItemIssueLink> store = new LinkedHashMap<>();
```
to:
```java
private final Map<UUID, WorkItemIssueLink> store = new ConcurrentHashMap<>();
```

- [ ] **Step 4: Run tests — expect all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryIssueLinkStoreTest 2>&1 | tail -20
```

- [ ] **Step 5: Commit**

```bash
git add persistence-memory/src/main/java/io/casehub/work/memory/InMemoryIssueLinkStore.java persistence-memory/src/test/java/io/casehub/work/memory/InMemoryIssueLinkStoreTest.java
git commit -m "feat: thread-safe InMemoryIssueLinkStore — ConcurrentHashMap (Refs #191)"
```

---

### Task 7: Thread-safe data structures — InMemoryWorkItemNoteStore

Replace `LinkedHashMap` with `ConcurrentHashMap`.

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemNoteStore.java`

- [ ] **Step 1: Write a concurrency test**

Create `persistence-memory/src/test/java/io/casehub/work/memory/InMemoryWorkItemNoteStoreTest.java`:

```java
package io.casehub.work.memory;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.work.runtime.model.WorkItemNote;

class InMemoryWorkItemNoteStoreTest {

    private InMemoryWorkItemNoteStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryWorkItemNoteStore();
    }

    @Test
    void append_concurrentWrites_nothingLost() throws Exception {
        final int threads = 8;
        final int perThread = 100;
        final var latch = new CountDownLatch(1);
        final var executor = Executors.newFixedThreadPool(threads);
        final var futures = new java.util.ArrayList<Future<?>>();
        final UUID workItemId = UUID.randomUUID();

        for (int t = 0; t < threads; t++) {
            futures.add(executor.submit(() -> {
                try { latch.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                for (int i = 0; i < perThread; i++) {
                    final WorkItemNote note = new WorkItemNote();
                    note.workItemId = workItemId;
                    note.content = "note";
                    note.author = "test";
                    store.append(note);
                }
            }));
        }
        latch.countDown();
        for (var f : futures) { f.get(5, TimeUnit.SECONDS); }
        executor.shutdown();

        assertThat(store.findAll()).hasSize(threads * perThread);
    }
}
```

- [ ] **Step 2: Run test — expect failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryWorkItemNoteStoreTest 2>&1 | tail -20
```

- [ ] **Step 3: Replace LinkedHashMap with ConcurrentHashMap**

In `InMemoryWorkItemNoteStore.java`, change:

```java
import java.util.LinkedHashMap;
```
to:
```java
import java.util.concurrent.ConcurrentHashMap;
```

And change the field:

```java
private final Map<UUID, WorkItemNote> store = new LinkedHashMap<>();
```
to:
```java
private final Map<UUID, WorkItemNote> store = new ConcurrentHashMap<>();
```

- [ ] **Step 4: Run tests — expect all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryWorkItemNoteStoreTest 2>&1 | tail -20
```

- [ ] **Step 5: Commit**

```bash
git add persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemNoteStore.java persistence-memory/src/test/java/io/casehub/work/memory/InMemoryWorkItemNoteStoreTest.java
git commit -m "feat: thread-safe InMemoryWorkItemNoteStore — ConcurrentHashMap (Refs #191)"
```

---

### Task 8: AuditEntryStore restructure — per-workItemId keyed concurrent map

Replace flat `ArrayList<AuditEntry>` with `ConcurrentHashMap<UUID, CopyOnWriteArrayList<AuditEntry>>`.

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryAuditEntryStore.java`
- Modify: `persistence-memory/src/test/java/io/casehub/work/memory/InMemoryRepositoryTest.java`

- [ ] **Step 1: Write a concurrency test for append**

Add to `InMemoryRepositoryTest.java`:

```java
@Test
void append_concurrentWrites_nothingLost() throws Exception {
    final int threads = 8;
    final int perThread = 50;
    final UUID workItemId = UUID.randomUUID();
    final var latch = new java.util.concurrent.CountDownLatch(1);
    final var executor = java.util.concurrent.Executors.newFixedThreadPool(threads);
    final var futures = new java.util.ArrayList<java.util.concurrent.Future<?>>();

    for (int t = 0; t < threads; t++) {
        futures.add(executor.submit(() -> {
            try { latch.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            for (int i = 0; i < perThread; i++) {
                auditStore.append(auditEntry(workItemId, "EVENT"));
            }
        }));
    }
    latch.countDown();
    for (var f : futures) { f.get(5, java.util.concurrent.TimeUnit.SECONDS); }
    executor.shutdown();

    assertThat(auditStore.findByWorkItemId(workItemId)).hasSize(threads * perThread);
}
```

- [ ] **Step 2: Run test — expect failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryRepositoryTest#append_concurrentWrites_nothingLost 2>&1 | tail -20
```

- [ ] **Step 3: Rewrite InMemoryAuditEntryStore with keyed concurrent map**

Replace the full implementation:

```java
package io.casehub.work.memory;

import java.time.Instant;
import java.util.Comparator;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.work.runtime.model.AuditEntry;
import io.casehub.work.runtime.repository.AuditEntryStore;
import io.casehub.work.runtime.repository.AuditQuery;

/**
 * In-memory implementation of {@link AuditEntryStore} for ephemeral deployments
 * and tests. No datasource or Flyway configuration required.
 *
 * <p>
 * Tier 3 in the CDI priority ladder — {@code @Alternative @Priority(100)} beats
 * both JPA (Tier 1) and MongoDB (Tier 2) when on the classpath.
 *
 * <p>
 * Thread-safe. Data is ephemeral (lost on restart).
 *
 * <p>
 * <strong>Known limitation:</strong> Category filter in {@link AuditQuery} is
 * silently ignored — the filter requires access to the parent WorkItem's category,
 * which would create an inter-store dependency.
 */
@ApplicationScoped
@Alternative
@Priority(100)
public class InMemoryAuditEntryStore implements AuditEntryStore {

    private final ConcurrentHashMap<UUID, CopyOnWriteArrayList<AuditEntry>> store = new ConcurrentHashMap<>();

    /** Removes all stored entries. Available for test isolation ({@code @BeforeEach}) and administrative reset. */
    public void clear() {
        store.clear();
    }

    @Override
    public void append(final AuditEntry entry) {
        if (entry.id == null) {
            entry.id = UUID.randomUUID();
        }
        if (entry.occurredAt == null) {
            entry.occurredAt = Instant.now();
        }
        store.computeIfAbsent(entry.workItemId, k -> new CopyOnWriteArrayList<>()).add(entry);
    }

    @Override
    public List<AuditEntry> findByWorkItemId(final UUID workItemId) {
        return store.getOrDefault(workItemId, new CopyOnWriteArrayList<>()).stream()
                .sorted(Comparator.comparing(e -> e.occurredAt))
                .toList();
    }

    @Override
    public List<AuditEntry> query(final AuditQuery query) {
        return store.values().stream()
                .flatMap(List::stream)
                .filter(e -> query.actorId() == null || query.actorId().equals(e.actor))
                .filter(e -> query.event() == null || query.event().equals(e.event))
                .filter(e -> query.from() == null || !e.occurredAt.isBefore(query.from()))
                .filter(e -> query.to() == null || !e.occurredAt.isAfter(query.to()))
                .sorted(Comparator.comparing((AuditEntry e) -> e.occurredAt).reversed())
                .skip((long) query.page() * query.size())
                .limit(query.size())
                .toList();
    }

    @Override
    public long count(final AuditQuery query) {
        return store.values().stream()
                .flatMap(List::stream)
                .filter(e -> query.actorId() == null || query.actorId().equals(e.actor))
                .filter(e -> query.event() == null || query.event().equals(e.event))
                .filter(e -> query.from() == null || !e.occurredAt.isBefore(query.from()))
                .filter(e -> query.to() == null || !e.occurredAt.isAfter(query.to()))
                .count();
    }
}
```

- [ ] **Step 4: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory 2>&1 | tail -20
```

Expected: All tests pass (existing behavior preserved + new concurrency test passes).

- [ ] **Step 5: Commit**

```bash
git add persistence-memory/src/main/java/io/casehub/work/memory/InMemoryAuditEntryStore.java persistence-memory/src/test/java/io/casehub/work/memory/InMemoryRepositoryTest.java
git commit -m "feat: thread-safe InMemoryAuditEntryStore — keyed concurrent map (Refs #191)"
```

---

### Task 9: Consumer migration — update dependencies and imports

Update the 4 consumer modules from `casehub-work-testing` to `casehub-work-persistence-memory`.

**Files:**
- Modify: `ai/pom.xml`
- Modify: `notifications/pom.xml`
- Modify: `postgres-broadcaster/pom.xml`
- Modify: `queues-postgres-broadcaster/pom.xml`
- Modify: `ai/src/test/java/io/casehub/work/ai/skill/ResolutionHistorySkillProfileProviderTest.java`
- Modify: `ai/src/test/java/io/casehub/work/ai/suggestion/ResolutionSuggestionServiceTest.java`
- Modify: `ai/src/test/java/io/casehub/work/ai/escalation/EscalationSummaryServiceTest.java`

- [ ] **Step 1: Update all 4 pom.xml files**

In each of `ai/pom.xml`, `notifications/pom.xml`, `postgres-broadcaster/pom.xml`, `queues-postgres-broadcaster/pom.xml`, replace:

```xml
      <artifactId>casehub-work-testing</artifactId>
```
with:
```xml
      <artifactId>casehub-work-persistence-memory</artifactId>
```

- [ ] **Step 2: Update Java imports in ai/ test sources**

In `ai/src/test/java/io/casehub/work/ai/skill/ResolutionHistorySkillProfileProviderTest.java`:
- `import io.casehub.work.testing.InMemoryWorkItemStore;` → `import io.casehub.work.memory.InMemoryWorkItemStore;`

In `ai/src/test/java/io/casehub/work/ai/suggestion/ResolutionSuggestionServiceTest.java`:
- `import io.casehub.work.testing.InMemoryWorkItemStore;` → `import io.casehub.work.memory.InMemoryWorkItemStore;`

In `ai/src/test/java/io/casehub/work/ai/escalation/EscalationSummaryServiceTest.java`:
- `import io.casehub.work.testing.InMemoryAuditEntryStore;` → `import io.casehub.work.memory.InMemoryAuditEntryStore;`
- `import io.casehub.work.testing.InMemoryWorkItemStore;` → `import io.casehub.work.memory.InMemoryWorkItemStore;`

- [ ] **Step 3: Check for any other io.casehub.work.testing imports across the project**

```bash
grep -rn "io.casehub.work.testing" --include="*.java" --include="*.properties" | grep -v /target/ | grep -v persistence-memory/
```

Expected: no results. If any are found, update them.

- [ ] **Step 4: Verify consumer modules compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl ai,notifications,postgres-broadcaster,queues-postgres-broadcaster -am 2>&1 | tail -20
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Run consumer tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ai 2>&1 | tail -20
```

Expected: All ai/ tests pass. Run one consumer at a time to isolate failures.

- [ ] **Step 6: Commit**

```bash
git add ai/pom.xml notifications/pom.xml postgres-broadcaster/pom.xml queues-postgres-broadcaster/pom.xml
git add ai/src/test/
git commit -m "refactor: consumer migration — casehub-work-testing → persistence-memory (Refs #191)"
```

---

### Task 10: SPI Javadoc updates — correct the pre-existing error

Update the tier ladder documentation in each SPI interface.

**Files:**
- Modify: `../../../api/src/main/java/io/casehub/work/api/spi/WorkItemStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/AuditEntryStore.java`
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemNoteStore.java`
- Modify: `issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/IssueLinkStore.java`
- Modify: `core/src/main/java/io/casehub/work/core/strategy/RoutingCursorStore.java`

- [ ] **Step 1: Update WorkItemStore Javadoc**

Replace the CDI backend activation paragraph (lines 20–26) with:

```java
 * <strong>CDI backend activation (four-tier priority ladder):</strong><br>
 * Tier 0: {@code @DefaultBean} (no-op fallback) — not applicable to this SPI.<br>
 * Tier 1: {@code @ApplicationScoped} (JPA/SQL, default) — {@code casehub-work} runtime.<br>
 * Tier 2: {@code @Alternative @Priority(1)} (MongoDB) — {@code casehub-work-persistence-mongodb}.<br>
 * Tier 3: {@code @Alternative @Priority(100)} (in-memory, ephemeral) — {@code casehub-work-persistence-memory}.<br>
 * Adding a backend module to the classpath activates it automatically — no consumer changes.
```

- [ ] **Step 2: Update AuditEntryStore Javadoc**

Replace the CDI backend activation paragraph (lines 13–19) with the same four-tier text as WorkItemStore.

- [ ] **Step 3: Update WorkItemNoteStore Javadoc**

Replace the existing CDI paragraph (lines 12–15) with:

```java
 * <strong>CDI backend activation:</strong><br>
 * Tier 1: {@code @ApplicationScoped} (JPA/SQL, default) — {@code casehub-work} runtime.<br>
 * Tier 3: {@code @Alternative @Priority(100)} (in-memory, ephemeral) — {@code casehub-work-persistence-memory}.<br>
 * No Tier 2 (MongoDB) exists yet (tracked as casehubio/work#253).
```

- [ ] **Step 4: Update IssueLinkStore Javadoc**

Replace lines 12–25 with:

```java
 * Replaces direct Panache static calls with an injectable seam, enabling
 * full unit testing without CDI or a database.
 *
 * <p>
 * <strong>CDI backend activation:</strong><br>
 * Tier 1: {@code @ApplicationScoped} (JPA/SQL, default) — {@code casehub-work-issue-tracker}.<br>
 * Tier 3: {@code @Alternative @Priority(100)} (in-memory, ephemeral) — {@code casehub-work-persistence-memory}.<br>
 * No Tier 2 (MongoDB) exists yet (tracked as casehubio/work#253).
```

- [ ] **Step 5: Update RoutingCursorStore Javadoc**

Add after the existing atomicity paragraph:

```java
 * <p>
 * <strong>CDI backend activation:</strong><br>
 * Tier 0: {@code @DefaultBean} (no-op) — {@code casehub-work-core}.<br>
 * Tier 1: {@code @ApplicationScoped} (JPA/SQL, default) — {@code casehub-work} runtime.<br>
 * Tier 3: {@code @Alternative @Priority(100)} (in-memory, ephemeral) — {@code casehub-work-persistence-memory}.<br>
 * No Tier 2 (MongoDB) exists yet (tracked as casehubio/work#253).
```

- [ ] **Step 6: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime,issue-tracker,core -am 2>&1 | tail -10
```

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemStore.java
git add runtime/src/main/java/io/casehub/work/runtime/repository/AuditEntryStore.java
git add runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemNoteStore.java
git add issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/IssueLinkStore.java
git add core/src/main/java/io/casehub/work/core/strategy/RoutingCursorStore.java
git commit -m "docs: correct SPI Javadoc — four-tier CDI priority ladder (Refs #191)"
```

---

### Task 11: Documentation — MODULES.md and module README

**Files:**
- Modify: `docs/MODULES.md`
- Create: `persistence-memory/README.md`

- [ ] **Step 1: Update MODULES.md**

In the Core Modules table, replace the `testing/` row with:

```markdown
| `persistence-memory/` | In-memory persistence (`casehub-work-persistence-memory`) | Thread-safe ConcurrentHashMap stores; Tier 3 `@Alternative @Priority(100)` — beats JPA and MongoDB. For tests, demos, and ephemeral deployment. |
```

Also add `persistence-mongodb/` to the Integration Modules table (pre-existing gap):

```markdown
| `casehub-work-persistence-mongodb/` | Optional MongoDB-backed WorkItemStore and AuditEntryStore; Tier 2 `@Alternative @Priority(1)` |
```

- [ ] **Step 2: Create persistence-memory/README.md**

```markdown
# CaseHub Work — In-Memory Persistence

Thread-safe, ephemeral in-memory stores for CaseHub Work. Data is lost on restart.

## When to use

- **Tests:** add as `<scope>test</scope>` dependency — in-memory stores beat JPA via CDI priority, no datasource config needed
- **Demos / local evaluation:** add at `compile` scope with datasource deactivation (see below)

## CDI Priority

Tier 3 in the platform CDI priority ladder — `@Alternative @Priority(100)`.

| Tier | Backend | Priority | Module |
|------|---------|----------|--------|
| 0 | No-op (@DefaultBean) | — | core/ (RoutingCursorStore only) |
| 1 | JPA (default) | @ApplicationScoped | runtime/ |
| 2 | MongoDB | @Priority(1) | persistence-mongodb/ |
| **3** | **In-memory (this module)** | **@Priority(100)** | **persistence-memory/** |

When both `persistence-mongodb` and `persistence-memory` are on the classpath, in-memory
wins. This is by design — typical use is `persistence-memory` at test scope alongside
a production backend.

## Thread safety

All stores use `ConcurrentHashMap` (or `ConcurrentHashMap` + `CopyOnWriteArrayList` for
audit entries). Weakly consistent iteration provides READ COMMITTED semantics — the same
isolation level as JPA stores on PostgreSQL.

Objects returned from the store are shared references. Concurrent field-level mutations
to the same object without calling `put()` are not guaranteed to be visible across threads.

## Ephemeral deployment

Add to `application.properties`:

```properties
quarkus.datasource.active=false
quarkus.hibernate-orm.active=false
```

These deactivate JPA extensions at build time. In-memory stores handle all operations.

## Known limitations

- **AuditEntryStore category filter** — silently ignored (requires inter-store dependency to resolve)
- **Shared object references** — see Thread Safety above
```

- [ ] **Step 3: Commit**

```bash
git add docs/MODULES.md persistence-memory/README.md
git commit -m "docs: update MODULES.md, add persistence-memory README (Refs #191)"
```

---

### Task 12: Full build verification

Run the complete build to catch any transitive issues.

- [ ] **Step 1: Run persistence-memory tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory 2>&1 | tail -20
```

- [ ] **Step 2: Run consumer module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ai 2>&1 | tail -20
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl notifications 2>&1 | tail -20
```

- [ ] **Step 3: Run integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl integration-tests 2>&1 | tail -30
```

- [ ] **Step 4: Check no stale references to casehub-work-testing remain**

```bash
grep -rn "casehub-work-testing\|io.casehub.work.testing" --include="*.java" --include="*.xml" --include="*.properties" --include="*.md" | grep -v /target/ | grep -v persistence-memory/
```

Expected: no results (MODULES.md and pom.xml already updated).

Note: if any are found in ARC42STORIES.MD — those are updated in Task 13 (doc-sync).
