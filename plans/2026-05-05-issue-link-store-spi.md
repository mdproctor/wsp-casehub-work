# IssueLinkStore SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract Panache static calls from `WebhookEventHandler` and `IssueLinkService` behind an injectable `IssueLinkStore` SPI, making both classes fully unit-testable without CDI or a database.

**Architecture:** New `IssueLinkStore` interface in the issue-tracker module, with `JpaIssueLinkStore` as the default Panache-backed implementation and `InMemoryIssueLinkStore` in the testing module — identical pattern to `WorkItemStore`/`AuditEntryStore`. `WebhookEventHandler` injects `IssueLinkStore` + `WorkItemStore` (already exists). `IssueLinkService` injects `IssueLinkStore` replacing all direct Panache entity calls.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, CDI, JUnit 5, Mockito, AssertJ, H2 (integration tests)

---

## Prerequisites

Verify IntelliJ MCPs are available before starting. If either returns a 404 or error, stop and report.

All commits must reference `#161` and parent epic `#156`.

Build commands — always target specific modules:
```bash
# Test a module
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub-work-issue-tracker

# Install a module to local Maven repo
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -pl casehub-work-issue-tracker

# Test the testing module
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing
```

---

## File Map

**New files:**
- `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/IssueLinkStore.java` — SPI interface
- `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/jpa/JpaIssueLinkStore.java` — default JPA impl
- `testing/src/main/java/io/casehub/work/testing/InMemoryIssueLinkStore.java` — in-memory impl for tests
- `testing/src/test/java/io/casehub/work/testing/InMemoryIssueLinkStoreTest.java` — correctness tests

**Modified files:**
- `testing/pom.xml` — add `casehub-work-issue-tracker` compile dependency
- `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventHandler.java` — inject stores
- `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/webhook/WebhookEventHandlerTest.java` — update constructor usage + 5 new public `handle()` tests
- `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/service/IssueLinkService.java` — inject `IssueLinkStore`
- `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/service/IssueLinkServiceTest.java` — rewrite as pure unit tests
- `docs/ARCHITECTURE.md` — add new types
- `CLAUDE.md` — update issue-tracker + testing module entries
- `docs/DESIGN.md` — update test counts

---

## Task 1: IssueLinkStore SPI + JpaIssueLinkStore

**Files:**
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/IssueLinkStore.java`
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/jpa/JpaIssueLinkStore.java`

- [ ] **Step 1: Create the IssueLinkStore interface**

```java
package io.casehub.work.issuetracker.repository;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import io.casehub.work.issuetracker.model.WorkItemIssueLink;

/**
 * Store SPI for {@link WorkItemIssueLink} persistence.
 *
 * <p>
 * Replaces direct Panache static calls in
 * {@link io.casehub.work.issuetracker.webhook.WebhookEventHandler} and
 * {@link io.casehub.work.issuetracker.service.IssueLinkService} with an
 * injectable seam, enabling full unit testing without CDI or a database.
 *
 * <p>
 * The default implementation ({@link jpa.JpaIssueLinkStore}) uses Hibernate ORM
 * with Panache. The in-memory alternative is provided in
 * {@code casehub-work-testing} for application-level tests.
 *
 * <p>
 * Custom implementations register as {@code @ApplicationScoped @Alternative
 * @Priority(1)} CDI beans.
 */
public interface IssueLinkStore {

    /**
     * Find a link by its surrogate primary key.
     *
     * @param id the link UUID
     * @return the link, or empty if not found
     */
    Optional<WorkItemIssueLink> findById(UUID id);

    /**
     * Return all links for the given WorkItem, ordered by creation time ascending.
     *
     * @param workItemId the WorkItem UUID
     * @return list of links; may be empty, never null
     */
    List<WorkItemIssueLink> findByWorkItemId(UUID workItemId);

    /**
     * Find a specific link by WorkItem, tracker type, and external reference.
     *
     * @param workItemId the WorkItem UUID
     * @param trackerType the tracker type string (e.g. {@code "github"})
     * @param externalRef the tracker-specific reference (e.g. {@code "owner/repo#42"})
     * @return the link, or empty if not found
     */
    Optional<WorkItemIssueLink> findByRef(UUID workItemId, String trackerType, String externalRef);

    /**
     * Return all links for the given tracker type and external reference,
     * across all WorkItems. Used by webhook handlers that receive the tracker ref
     * but not the WorkItem ID.
     *
     * @param trackerType the tracker type string
     * @param externalRef the tracker-specific reference
     * @return list of links; may be empty, never null
     */
    List<WorkItemIssueLink> findByTrackerRef(String trackerType, String externalRef);

    /**
     * Persist or update a link.
     *
     * <p>
     * For new links (id is null), the entity's {@code @PrePersist} hook assigns a
     * UUID and {@code linkedAt} timestamp. For existing managed entities in a
     * JPA context, dirty-checking at transaction commit handles the update — an
     * explicit {@code save()} call makes the intent clear.
     *
     * @param link the link to persist; must not be null
     * @return the saved link
     */
    WorkItemIssueLink save(WorkItemIssueLink link);

    /**
     * Delete a link from the store.
     *
     * @param link the link to delete; must not be null
     */
    void delete(WorkItemIssueLink link);
}
```

- [ ] **Step 2: Create JpaIssueLinkStore**

```java
package io.casehub.work.issuetracker.repository.jpa;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.work.issuetracker.model.WorkItemIssueLink;
import io.casehub.work.issuetracker.repository.IssueLinkStore;

/**
 * Default JPA implementation of {@link IssueLinkStore} backed by Hibernate ORM
 * with Panache. Delegates to the static and instance methods on
 * {@link WorkItemIssueLink}.
 *
 * <p>
 * No query logic lives here — this is a thin adapter. All callers must ensure
 * an active JTA transaction exists before calling mutating methods.
 */
@ApplicationScoped
public class JpaIssueLinkStore implements IssueLinkStore {

    /**
     * {@inheritDoc}
     */
    @Override
    public Optional<WorkItemIssueLink> findById(final UUID id) {
        return Optional.ofNullable(WorkItemIssueLink.findById(id));
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public List<WorkItemIssueLink> findByWorkItemId(final UUID workItemId) {
        return WorkItemIssueLink.findByWorkItemId(workItemId);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public Optional<WorkItemIssueLink> findByRef(
            final UUID workItemId, final String trackerType, final String externalRef) {
        return Optional.ofNullable(WorkItemIssueLink.findByRef(workItemId, trackerType, externalRef));
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public List<WorkItemIssueLink> findByTrackerRef(final String trackerType, final String externalRef) {
        return WorkItemIssueLink.findByTrackerRef(trackerType, externalRef);
    }

    /**
     * {@inheritDoc}
     *
     * <p>
     * Calls {@link WorkItemIssueLink#persist()} which is a no-op for already-managed
     * entities; dirty-checking at transaction commit handles those updates.
     */
    @Override
    public WorkItemIssueLink save(final WorkItemIssueLink link) {
        link.persist();
        return link;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void delete(final WorkItemIssueLink link) {
        link.delete();
    }
}
```

- [ ] **Step 3: Compile to verify both files are syntactically correct**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl casehub-work-issue-tracker 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Run existing tests to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub-work-issue-tracker 2>&1 | tail -8
```

Expected: all 87 tests pass

- [ ] **Step 5: Commit**

```bash
git add \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/IssueLinkStore.java \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/jpa/JpaIssueLinkStore.java
git commit -m "$(cat <<'EOF'
feat(issue-tracker): IssueLinkStore SPI + JpaIssueLinkStore (#161)

Injectable persistence seam for WorkItemIssueLink — follows the
WorkItemStore/AuditEntryStore pattern. JpaIssueLinkStore is the default
@ApplicationScoped Panache-backed implementation.

Refs #161, #156
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: InMemoryIssueLinkStore + pom + correctness tests

**Files:**
- Modify: `testing/pom.xml`
- Create: `testing/src/main/java/io/casehub/work/testing/InMemoryIssueLinkStore.java`
- Create: `testing/src/test/java/io/casehub/work/testing/InMemoryIssueLinkStoreTest.java`

- [ ] **Step 1: Add casehub-work-issue-tracker dependency to testing/pom.xml**

Add inside `<dependencies>` after the existing `casehub-work` dependency:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-work-issue-tracker</artifactId>
  <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Write the failing tests first**

Create `testing/src/test/java/io/casehub/work/testing/InMemoryIssueLinkStoreTest.java`:

```java
package io.casehub.work.testing;

import io.casehub.work.issuetracker.model.WorkItemIssueLink;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class InMemoryIssueLinkStoreTest {

    private InMemoryIssueLinkStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryIssueLinkStore();
    }

    private WorkItemIssueLink link(final UUID workItemId, final String trackerType, final String externalRef) {
        final WorkItemIssueLink link = new WorkItemIssueLink();
        link.workItemId = workItemId;
        link.trackerType = trackerType;
        link.externalRef = externalRef;
        link.status = "open";
        link.linkedBy = "test";
        return link;
    }

    // ── save + findById ───────────────────────────────────────────────────────

    @Test
    void save_assignsIdAndLinkedAt_whenNull() {
        final WorkItemIssueLink link = link(UUID.randomUUID(), "github", "owner/repo#1");

        final WorkItemIssueLink saved = store.save(link);

        assertThat(saved.id).isNotNull();
        assertThat(saved.linkedAt).isNotNull();
    }

    @Test
    void save_preservesExistingIdAndLinkedAt() {
        final UUID id = UUID.randomUUID();
        final Instant ts = Instant.parse("2026-01-01T00:00:00Z");
        final WorkItemIssueLink link = link(UUID.randomUUID(), "github", "owner/repo#2");
        link.id = id;
        link.linkedAt = ts;

        store.save(link);

        assertThat(store.findById(id)).isPresent().get()
                .satisfies(l -> {
                    assertThat(l.id).isEqualTo(id);
                    assertThat(l.linkedAt).isEqualTo(ts);
                });
    }

    @Test
    void findById_returnsEmpty_whenNotFound() {
        assertThat(store.findById(UUID.randomUUID())).isEmpty();
    }

    @Test
    void findById_returnsLink_afterSave() {
        final WorkItemIssueLink link = link(UUID.randomUUID(), "github", "owner/repo#3");
        store.save(link);

        assertThat(store.findById(link.id)).isPresent();
    }

    // ── findByWorkItemId ──────────────────────────────────────────────────────

    @Test
    void findByWorkItemId_returnsAllForWorkItem() {
        final UUID workItemId = UUID.randomUUID();
        store.save(link(workItemId, "github", "owner/repo#10"));
        store.save(link(workItemId, "github", "owner/repo#11"));
        store.save(link(UUID.randomUUID(), "github", "owner/repo#12")); // different WorkItem

        assertThat(store.findByWorkItemId(workItemId)).hasSize(2);
    }

    @Test
    void findByWorkItemId_returnsEmpty_whenNoLinks() {
        assertThat(store.findByWorkItemId(UUID.randomUUID())).isEmpty();
    }

    // ── findByRef ─────────────────────────────────────────────────────────────

    @Test
    void findByRef_returnsLink_whenExists() {
        final UUID workItemId = UUID.randomUUID();
        store.save(link(workItemId, "github", "owner/repo#20"));

        final Optional<WorkItemIssueLink> result = store.findByRef(workItemId, "github", "owner/repo#20");

        assertThat(result).isPresent();
        assertThat(result.get().workItemId).isEqualTo(workItemId);
    }

    @Test
    void findByRef_returnsEmpty_whenNotFound() {
        assertThat(store.findByRef(UUID.randomUUID(), "github", "owner/repo#99")).isEmpty();
    }

    @Test
    void findByRef_doesNotMatch_wrongWorkItem() {
        store.save(link(UUID.randomUUID(), "github", "owner/repo#30"));

        assertThat(store.findByRef(UUID.randomUUID(), "github", "owner/repo#30")).isEmpty();
    }

    // ── findByTrackerRef ──────────────────────────────────────────────────────

    @Test
    void findByTrackerRef_returnsAllMatchingLinks() {
        final UUID workItemId1 = UUID.randomUUID();
        final UUID workItemId2 = UUID.randomUUID();
        store.save(link(workItemId1, "github", "owner/repo#40"));
        store.save(link(workItemId2, "github", "owner/repo#40"));
        store.save(link(UUID.randomUUID(), "github", "owner/repo#41")); // different ref

        final List<WorkItemIssueLink> results = store.findByTrackerRef("github", "owner/repo#40");

        assertThat(results).hasSize(2);
        assertThat(results).allMatch(l -> "owner/repo#40".equals(l.externalRef));
    }

    @Test
    void findByTrackerRef_returnsEmpty_whenNoMatches() {
        assertThat(store.findByTrackerRef("github", "owner/repo#999")).isEmpty();
    }

    @Test
    void findByTrackerRef_doesNotMatch_differentTrackerType() {
        store.save(link(UUID.randomUUID(), "github", "owner/repo#50"));

        assertThat(store.findByTrackerRef("jira", "owner/repo#50")).isEmpty();
    }

    // ── delete ────────────────────────────────────────────────────────────────

    @Test
    void delete_removesLink() {
        final WorkItemIssueLink link = store.save(link(UUID.randomUUID(), "github", "owner/repo#60"));

        store.delete(link);

        assertThat(store.findById(link.id)).isEmpty();
    }

    @Test
    void delete_doesNotAffectOtherLinks() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink a = store.save(link(workItemId, "github", "owner/repo#70"));
        final WorkItemIssueLink b = store.save(link(workItemId, "github", "owner/repo#71"));

        store.delete(a);

        assertThat(store.findById(b.id)).isPresent();
        assertThat(store.findByWorkItemId(workItemId)).hasSize(1);
    }

    // ── clear ─────────────────────────────────────────────────────────────────

    @Test
    void clear_removesAllLinks() {
        store.save(link(UUID.randomUUID(), "github", "owner/repo#80"));
        store.save(link(UUID.randomUUID(), "github", "owner/repo#81"));

        store.clear();

        assertThat(store.findByTrackerRef("github", "owner/repo#80")).isEmpty();
        assertThat(store.findByTrackerRef("github", "owner/repo#81")).isEmpty();
    }
}
```

- [ ] **Step 3: Compile the test — expect failure (InMemoryIssueLinkStore doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl testing 2>&1 | grep -E "ERROR|error:" | head -5
```

Expected: compilation error on `InMemoryIssueLinkStore`

- [ ] **Step 4: Implement InMemoryIssueLinkStore**

Create `testing/src/main/java/io/casehub/work/testing/InMemoryIssueLinkStore.java`:

```java
package io.casehub.work.testing;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.work.issuetracker.model.WorkItemIssueLink;
import io.casehub.work.issuetracker.repository.IssueLinkStore;

/**
 * In-memory implementation of {@link IssueLinkStore} for use in tests of
 * applications that embed CaseHub Work with the issue-tracker module. No
 * datasource or Flyway configuration is required.
 *
 * <p>
 * Activate by including {@code casehub-work-testing} on the test classpath
 * alongside {@code casehub-work-issue-tracker}. CDI selects this bean over
 * the default JPA implementation via {@code @Alternative} and {@code @Priority(1)}.
 *
 * <p>
 * <strong>Not thread-safe</strong> — designed for single-threaded test use only.
 *
 * <p>
 * Call {@link #clear()} in a {@code @BeforeEach} method to isolate tests from
 * one another.
 */
@ApplicationScoped
@Alternative
@Priority(1)
public class InMemoryIssueLinkStore implements IssueLinkStore {

    // NOT thread-safe — designed for single-threaded test use
    private final Map<UUID, WorkItemIssueLink> store = new LinkedHashMap<>();

    /**
     * Clears all stored links. Call in {@code @BeforeEach} to isolate tests.
     */
    public void clear() {
        store.clear();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public Optional<WorkItemIssueLink> findById(final UUID id) {
        return Optional.ofNullable(store.get(id));
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public List<WorkItemIssueLink> findByWorkItemId(final UUID workItemId) {
        return store.values().stream()
                .filter(l -> workItemId.equals(l.workItemId))
                .sorted(java.util.Comparator.comparing(l -> l.linkedAt))
                .toList();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public Optional<WorkItemIssueLink> findByRef(
            final UUID workItemId, final String trackerType, final String externalRef) {
        return store.values().stream()
                .filter(l -> workItemId.equals(l.workItemId)
                        && trackerType.equals(l.trackerType)
                        && externalRef.equals(l.externalRef))
                .findFirst();
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public List<WorkItemIssueLink> findByTrackerRef(final String trackerType, final String externalRef) {
        return store.values().stream()
                .filter(l -> trackerType.equals(l.trackerType) && externalRef.equals(l.externalRef))
                .toList();
    }

    /**
     * {@inheritDoc}
     *
     * <p>
     * If {@code link.id} is {@code null}, a fresh {@link UUID} is assigned (replicating
     * what {@code @PrePersist} does in the JPA implementation). If {@code link.linkedAt}
     * is {@code null}, it is set to {@link Instant#now()}.
     */
    @Override
    public WorkItemIssueLink save(final WorkItemIssueLink link) {
        if (link.id == null) {
            link.id = UUID.randomUUID();
        }
        if (link.linkedAt == null) {
            link.linkedAt = Instant.now();
        }
        store.put(link.id, link);
        return link;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void delete(final WorkItemIssueLink link) {
        store.remove(link.id);
    }
}
```

- [ ] **Step 5: Run the tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing -Dtest=InMemoryIssueLinkStoreTest 2>&1 | tail -10
```

Expected: all 17 tests pass

- [ ] **Step 6: Run the full testing module suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing 2>&1 | tail -8
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7: Install casehub-work-issue-tracker so testing can resolve it**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -pl casehub-work-issue-tracker 2>&1 | tail -5
```

- [ ] **Step 8: Commit**

```bash
git add \
  testing/pom.xml \
  testing/src/main/java/io/casehub/work/testing/InMemoryIssueLinkStore.java \
  testing/src/test/java/io/casehub/work/testing/InMemoryIssueLinkStoreTest.java
git commit -m "$(cat <<'EOF'
feat(testing): InMemoryIssueLinkStore — in-memory IssueLinkStore for tests (#161)

LinkedHashMap-backed @Alternative @Priority(1) implementation.
Replicates @PrePersist UUID/linkedAt assignment for new links.
17 correctness tests covering all 6 SPI operations + clear().
testing/pom.xml gains casehub-work-issue-tracker compile dependency.

Refs #161, #156
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Refactor WebhookEventHandler + update tests

**Files:**
- Modify: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventHandler.java`
- Modify: `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/webhook/WebhookEventHandlerTest.java`

- [ ] **Step 1: Replace WebhookEventHandler with the refactored version**

Replace the entire file content of
`casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventHandler.java`:

```java
package io.casehub.work.issuetracker.webhook;

import java.util.ArrayList;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.jboss.logging.Logger;

import io.casehub.work.issuetracker.repository.IssueLinkStore;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemLabel;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.casehub.work.runtime.service.WorkItemService;

/**
 * Applies normalised {@link WebhookEvent} records to WorkItem state.
 * Called by tracker-specific webhook resources after HMAC verification and parsing.
 * Tracker vocabulary never reaches this class — all events arrive already normalised.
 */
@ApplicationScoped
public class WebhookEventHandler {

    private static final Logger LOG = Logger.getLogger(WebhookEventHandler.class);
    private static final String LINKED_WORKITEM_FOOTER = "\n\n---\n*Linked WorkItem:";

    @Inject
    IssueLinkStore linkStore;

    @Inject
    WorkItemStore workItemStore;

    @Inject
    WorkItemService workItemService;

    /** Package-private constructor for unit testing without CDI. */
    WebhookEventHandler(
            final IssueLinkStore linkStore,
            final WorkItemStore workItemStore,
            final WorkItemService workItemService) {
        this.linkStore = linkStore;
        this.workItemStore = workItemStore;
        this.workItemService = workItemService;
    }

    WebhookEventHandler() {
        // CDI no-arg constructor
    }

    /**
     * Look up all WorkItems linked to the event's externalRef and apply the transition to each.
     * Failures per WorkItem are logged and swallowed — prevents tracker retries.
     */
    @Transactional
    public void handle(final WebhookEvent event) {
        final var links = linkStore.findByTrackerRef(event.trackerType(), event.externalRef());

        if (links.isEmpty()) {
            LOG.debugf("No WorkItem linked to %s:%s — ignoring", event.trackerType(), event.externalRef());
            return;
        }

        for (final var link : links) {
            workItemStore.get(link.workItemId)
                    .ifPresent(wi -> handle(link.workItemId, wi, event));
        }
    }

    /** Package-private for unit testing. */
    void handle(final UUID workItemId, final WorkItem workItem, final WebhookEvent event) {
        if (workItem.status != null && workItem.status.isTerminal()) {
            LOG.debugf("WorkItem %s is terminal (%s) — skipping %s event",
                    workItemId, workItem.status, event.eventKind());
            return;
        }
        try {
            applyTransition(workItemId, workItem, event);
        } catch (final Exception e) {
            LOG.warnf("Failed to apply %s event to WorkItem %s: %s",
                    event.eventKind(), workItemId, e.getMessage());
        }
    }

    /** Package-private for unit testing. */
    void applyTransition(final UUID workItemId, final WorkItem workItem, final WebhookEvent event) {
        switch (event.eventKind()) {
            case CLOSED -> applyClosed(workItemId, event);
            case ASSIGNED -> applyAssigned(workItemId, workItem, event);
            case UNASSIGNED -> workItemService.release(workItemId, event.actor());
            case TITLE_CHANGED -> workItem.title = event.newTitle();
            case DESCRIPTION_CHANGED -> workItem.description = stripFooter(event.newDescription());
            case PRIORITY_CHANGED -> workItem.priority = event.newPriority();
            case LABEL_ADDED -> addLabel(workItem, event.labelValue());
            case LABEL_REMOVED -> removeLabel(workItem, event.labelValue());
        }
    }

    private void applyClosed(final UUID workItemId, final WebhookEvent event) {
        switch (event.normativeResolution()) {
            case DONE -> workItemService.complete(workItemId, event.actor(), null);
            case DECLINE -> workItemService.cancel(workItemId, event.actor(), null);
            case FAILURE -> workItemService.reject(workItemId, event.actor(), null);
        }
    }

    private void applyAssigned(final UUID workItemId, final WorkItem workItem, final WebhookEvent event) {
        if (workItem.status == WorkItemStatus.PENDING) {
            workItemService.claim(workItemId, event.newAssignee());
        } else {
            workItem.assigneeId = event.newAssignee();
        }
    }

    private String stripFooter(final String description) {
        if (description == null) return null;
        final int idx = description.indexOf(LINKED_WORKITEM_FOOTER);
        return idx >= 0 ? description.substring(0, idx) : description;
    }

    private void addLabel(final WorkItem workItem, final String path) {
        if (workItem.labels == null) workItem.labels = new ArrayList<>();
        final boolean exists = workItem.labels.stream().anyMatch(l -> path.equals(l.path));
        if (!exists) {
            final WorkItemLabel label = new WorkItemLabel();
            label.path = path;
            workItem.labels.add(label);
        }
    }

    private void removeLabel(final WorkItem workItem, final String path) {
        if (workItem.labels != null) {
            workItem.labels.removeIf(l -> path.equals(l.path));
        }
    }
}
```

- [ ] **Step 2: Update WebhookEventHandlerTest — fix constructor + add 5 new public handle() tests**

Replace the entire file content of
`casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/webhook/WebhookEventHandlerTest.java`:

```java
package io.casehub.work.issuetracker.webhook;

import io.casehub.work.api.NormativeResolution;
import io.casehub.work.issuetracker.model.WorkItemIssueLink;
import io.casehub.work.issuetracker.repository.IssueLinkStore;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemLabel;
import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.casehub.work.runtime.service.WorkItemService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNoException;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class WebhookEventHandlerTest {

    private IssueLinkStore linkStore;
    private WorkItemStore workItemStore;
    private WorkItemService workItemService;
    private WebhookEventHandler handler;
    private UUID workItemId;

    @BeforeEach
    void setUp() {
        linkStore = mock(IssueLinkStore.class);
        workItemStore = mock(WorkItemStore.class);
        workItemService = mock(WorkItemService.class);
        handler = new WebhookEventHandler(linkStore, workItemStore, workItemService);
        workItemId = UUID.randomUUID();
    }

    private WorkItem activeWorkItem() {
        WorkItem wi = new WorkItem();
        wi.id = workItemId;
        wi.status = WorkItemStatus.ASSIGNED;
        wi.assigneeId = "alice";
        wi.labels = new ArrayList<>();
        return wi;
    }

    private WorkItemIssueLink linkFor(final UUID wid) {
        final WorkItemIssueLink link = new WorkItemIssueLink();
        link.workItemId = wid;
        return link;
    }

    // ── handle(WebhookEvent) — public entry point tests ───────────────────────

    @Test
    void handle_singleLink_workItemFound_appliesTransition() {
        final WorkItem wi = activeWorkItem();
        when(linkStore.findByTrackerRef("github", "owner/repo#42"))
                .thenReturn(List.of(linkFor(workItemId)));
        when(workItemStore.get(workItemId)).thenReturn(Optional.of(wi));

        final WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        handler.handle(event);

        verify(workItemService).complete(workItemId, "alice", null);
    }

    @Test
    void handle_noLinksFound_noOp() {
        when(linkStore.findByTrackerRef("github", "owner/repo#99"))
                .thenReturn(List.of());

        final WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#99", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        handler.handle(event);

        verifyNoInteractions(workItemService);
    }

    @Test
    void handle_workItemNotFound_skipsGracefully() {
        when(linkStore.findByTrackerRef("github", "owner/repo#42"))
                .thenReturn(List.of(linkFor(workItemId)));
        when(workItemStore.get(workItemId)).thenReturn(Optional.empty());

        final WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        handler.handle(event);

        verifyNoInteractions(workItemService);
    }

    @Test
    void handle_multipleLinks_allGetTransition() {
        final UUID wid1 = UUID.randomUUID();
        final UUID wid2 = UUID.randomUUID();

        final WorkItem wi1 = new WorkItem();
        wi1.id = wid1;
        wi1.status = WorkItemStatus.ASSIGNED;
        wi1.labels = new ArrayList<>();

        final WorkItem wi2 = new WorkItem();
        wi2.id = wid2;
        wi2.status = WorkItemStatus.IN_PROGRESS;
        wi2.labels = new ArrayList<>();

        when(linkStore.findByTrackerRef("github", "owner/repo#42"))
                .thenReturn(List.of(linkFor(wid1), linkFor(wid2)));
        when(workItemStore.get(wid1)).thenReturn(Optional.of(wi1));
        when(workItemStore.get(wid2)).thenReturn(Optional.of(wi2));

        final WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        handler.handle(event);

        verify(workItemService).complete(wid1, "alice", null);
        verify(workItemService).complete(wid2, "alice", null);
    }

    @Test
    void handle_firstLinkThrows_secondStillProcessed() {
        final UUID wid1 = UUID.randomUUID();
        final UUID wid2 = UUID.randomUUID();

        final WorkItem wi1 = new WorkItem();
        wi1.id = wid1;
        wi1.status = WorkItemStatus.ASSIGNED;
        wi1.labels = new ArrayList<>();

        final WorkItem wi2 = new WorkItem();
        wi2.id = wid2;
        wi2.status = WorkItemStatus.ASSIGNED;
        wi2.labels = new ArrayList<>();

        when(linkStore.findByTrackerRef("github", "owner/repo#42"))
                .thenReturn(List.of(linkFor(wid1), linkFor(wid2)));
        when(workItemStore.get(wid1)).thenReturn(Optional.of(wi1));
        when(workItemStore.get(wid2)).thenReturn(Optional.of(wi2));
        doThrow(new RuntimeException("service error"))
                .when(workItemService).complete(eq(wid1), any(), any());

        final WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        assertThatNoException().isThrownBy(() -> handler.handle(event));
        verify(workItemService).complete(wid2, "alice", null);
    }

    // ── applyTransition / handle(UUID, WorkItem, WebhookEvent) ───────────────

    @Test
    void closed_done_callsComplete() {
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        handler.applyTransition(workItemId, activeWorkItem(), event);

        verify(workItemService).complete(workItemId, "alice", null);
    }

    @Test
    void closed_decline_callsCancel() {
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DECLINE, null, null, null, null, null);

        handler.applyTransition(workItemId, activeWorkItem(), event);

        verify(workItemService).cancel(workItemId, "alice", null);
    }

    @Test
    void closed_failure_callsReject() {
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.FAILURE, null, null, null, null, null);

        handler.applyTransition(workItemId, activeWorkItem(), event);

        verify(workItemService).reject(workItemId, "alice", null);
    }

    @Test
    void assigned_pending_callsClaim() {
        WorkItem wi = new WorkItem();
        wi.id = workItemId;
        wi.status = WorkItemStatus.PENDING;
        wi.labels = new ArrayList<>();

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.ASSIGNED,
                "alice", null, null, null, null, null, "bob");

        handler.applyTransition(workItemId, wi, event);

        verify(workItemService).claim(workItemId, "bob");
    }

    @Test
    void assigned_alreadyAssigned_updatesAssigneeDirectly() {
        WorkItem wi = activeWorkItem();

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.ASSIGNED,
                "alice", null, null, null, null, null, "carol");

        handler.applyTransition(workItemId, wi, event);

        verify(workItemService, never()).claim(any(), any());
        assertThat(wi.assigneeId).isEqualTo("carol");
    }

    @Test
    void unassigned_callsRelease() {
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.UNASSIGNED,
                "alice", null, null, null, null, null, null);

        handler.applyTransition(workItemId, activeWorkItem(), event);

        verify(workItemService).release(workItemId, "alice");
    }

    @Test
    void titleChanged_updatesTitle() {
        WorkItem wi = activeWorkItem();
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.TITLE_CHANGED,
                "alice", null, null, null, "New Title", null, null);

        handler.applyTransition(workItemId, wi, event);

        assertThat(wi.title).isEqualTo("New Title");
        verifyNoMoreInteractions(workItemService);
    }

    @Test
    void descriptionChanged_stripsFooter() {
        WorkItem wi = activeWorkItem();
        String rawBody = "The real description.\n\n---\n*Linked WorkItem: `" + workItemId + "`*";
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.DESCRIPTION_CHANGED,
                "alice", null, null, null, null, rawBody, null);

        handler.applyTransition(workItemId, wi, event);

        assertThat(wi.description).isEqualTo("The real description.");
    }

    @Test
    void priorityChanged_updatesPriority() {
        WorkItem wi = activeWorkItem();
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.PRIORITY_CHANGED,
                "alice", null, WorkItemPriority.HIGH, null, null, null, null);

        handler.applyTransition(workItemId, wi, event);

        assertThat(wi.priority).isEqualTo(WorkItemPriority.HIGH);
    }

    @Test
    void labelAdded_appendsLabel() {
        WorkItem wi = activeWorkItem();
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.LABEL_ADDED,
                "alice", null, null, "legal/nda", null, null, null);

        handler.applyTransition(workItemId, wi, event);

        assertThat(wi.labels).anyMatch(l -> "legal/nda".equals(l.path));
    }

    @Test
    void labelAdded_duplicate_notAddedTwice() {
        WorkItem wi = activeWorkItem();
        WorkItemLabel existing = new WorkItemLabel();
        existing.path = "legal/nda";
        wi.labels.add(existing);

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.LABEL_ADDED,
                "alice", null, null, "legal/nda", null, null, null);

        handler.applyTransition(workItemId, wi, event);

        assertThat(wi.labels).hasSize(1);
    }

    @Test
    void labelRemoved_removesLabel() {
        WorkItem wi = activeWorkItem();
        WorkItemLabel label = new WorkItemLabel();
        label.path = "legal/nda";
        wi.labels.add(label);

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.LABEL_REMOVED,
                "alice", null, null, "legal/nda", null, null, null);

        handler.applyTransition(workItemId, wi, event);

        assertThat(wi.labels).isEmpty();
    }

    @Test
    void terminalWorkItem_skippedSilently() {
        WorkItem wi = new WorkItem();
        wi.id = workItemId;
        wi.status = WorkItemStatus.COMPLETED;

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        handler.handle(workItemId, wi, event);

        verifyNoInteractions(workItemService);
    }

    @Test
    void transitionFailure_swallowed() {
        doThrow(new IllegalStateException("wrong status"))
                .when(workItemService).complete(any(), any(), any());

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        assertThatNoException().isThrownBy(() -> handler.handle(workItemId, activeWorkItem(), event));
    }
}
```

- [ ] **Step 3: Run the handler tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub-work-issue-tracker \
  -Dtest=WebhookEventHandlerTest 2>&1 | tail -10
```

Expected: 19 tests (14 original + 5 new) all pass

- [ ] **Step 4: Run the full module suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub-work-issue-tracker 2>&1 | tail -8
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git add \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventHandler.java \
  casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/webhook/WebhookEventHandlerTest.java
git commit -m "$(cat <<'EOF'
refactor(issue-tracker): WebhookEventHandler injects IssueLinkStore + WorkItemStore (#161)

Replaces Panache static calls with injectable SPIs — public handle(WebhookEvent)
is now fully unit-testable without CDI or a database.
WorkItem.findById -> workItemStore.get() (Optional, safer).
5 new public entry-point tests: happy path, empty links, WorkItem not found,
multiple links, exception-per-link swallowed.

Refs #161, #156
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Refactor IssueLinkService + rewrite unit tests

**Files:**
- Modify: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/service/IssueLinkService.java`
- Replace: `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/service/IssueLinkServiceTest.java`

- [ ] **Step 1: Replace IssueLinkService**

Replace the entire file content of
`casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/service/IssueLinkService.java`:

```java
package io.casehub.work.issuetracker.service;

import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.jboss.logging.Logger;

import io.casehub.work.issuetracker.github.GitHubIssueTrackerConfig;
import io.casehub.work.issuetracker.model.WorkItemIssueLink;
import io.casehub.work.issuetracker.repository.IssueLinkStore;
import io.casehub.work.issuetracker.spi.ExternalIssueRef;
import io.casehub.work.issuetracker.spi.IssueTrackerException;
import io.casehub.work.issuetracker.spi.IssueTrackerProvider;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItemStatus;

/**
 * Manages links between WorkItems and external issue tracker issues.
 *
 * <p>
 * Routes each operation to the {@link IssueTrackerProvider} whose
 * {@link IssueTrackerProvider#trackerType()} matches the link's stored type.
 * Multiple providers can coexist in the same application.
 *
 * <h2>Auto-close</h2>
 * <p>
 * Observes {@link WorkItemLifecycleEvent}: when a WorkItem is COMPLETED and
 * {@code casehub.work.issue-tracker.github.auto-close-on-complete=true},
 * all linked issues are closed via their respective providers. Close failures
 * are logged and swallowed — they do not roll back the WorkItem completion.
 */
@ApplicationScoped
public class IssueLinkService {

    private static final Logger LOG = Logger.getLogger(IssueLinkService.class);

    @Inject
    Instance<IssueTrackerProvider> providers;

    @Inject
    GitHubIssueTrackerConfig githubConfig;

    @Inject
    IssueLinkStore linkStore;

    /**
     * Link an existing issue to a WorkItem.
     *
     * <p>
     * Fetches the issue's current title, URL, and status from the remote tracker
     * and stores them as a cached snapshot. Idempotent — if the link already exists,
     * this is a no-op.
     *
     * @param workItemId the WorkItem UUID
     * @param trackerType the tracker type (must match a registered provider)
     * @param externalRef the tracker-specific issue reference
     * @param linkedBy the actor creating this link
     * @return the created or existing link
     * @throws IssueTrackerException if the remote fetch fails
     * @throws IllegalArgumentException if no provider is registered for trackerType
     */
    @Transactional
    public WorkItemIssueLink linkExistingIssue(
            final UUID workItemId,
            final String trackerType,
            final String externalRef,
            final String linkedBy) {

        final WorkItemIssueLink existing =
                linkStore.findByRef(workItemId, trackerType, externalRef).orElse(null);
        if (existing != null) {
            return existing;
        }

        final IssueTrackerProvider provider = providerFor(trackerType);
        final ExternalIssueRef ref = provider.fetchIssue(externalRef);

        final WorkItemIssueLink link = new WorkItemIssueLink();
        link.workItemId = workItemId;
        link.trackerType = trackerType;
        link.externalRef = externalRef;
        link.title = ref.title();
        link.url = ref.url();
        link.status = ref.status();
        link.linkedBy = linkedBy;
        return linkStore.save(link);
    }

    /**
     * Create a new issue in the remote tracker and link it to a WorkItem.
     *
     * @param workItemId the WorkItem UUID
     * @param trackerType the tracker type
     * @param title the issue title
     * @param body the issue body (markdown)
     * @param linkedBy the actor creating the issue
     * @return the created link
     * @throws IssueTrackerException if the remote call fails or creation is not supported
     */
    @Transactional
    public WorkItemIssueLink createAndLink(
            final UUID workItemId,
            final String trackerType,
            final String title,
            final String body,
            final String linkedBy) {

        final IssueTrackerProvider provider = providerFor(trackerType);
        final String externalRef = provider.createIssue(workItemId, title, body)
                .orElseThrow(() -> new IssueTrackerException(
                        "Provider '" + trackerType + "' does not support issue creation"));

        final ExternalIssueRef ref = provider.fetchIssue(externalRef);

        final WorkItemIssueLink link = new WorkItemIssueLink();
        link.workItemId = workItemId;
        link.trackerType = trackerType;
        link.externalRef = externalRef;
        link.title = ref.title();
        link.url = ref.url();
        link.status = "open";
        link.linkedBy = linkedBy;
        return linkStore.save(link);
    }

    /**
     * Return all links for a WorkItem, ordered by creation time.
     *
     * @param workItemId the WorkItem UUID
     * @return list of links; may be empty
     */
    @Transactional
    public List<WorkItemIssueLink> listLinks(final UUID workItemId) {
        return linkStore.findByWorkItemId(workItemId);
    }

    /**
     * Remove a specific link by ID.
     *
     * @param linkId the UUID of the link to remove
     * @param workItemId the WorkItem UUID (for ownership check)
     * @return true if deleted, false if the link was not found or belongs to a different WorkItem
     */
    @Transactional
    public boolean removeLink(final UUID linkId, final UUID workItemId) {
        final WorkItemIssueLink link = linkStore.findById(linkId).orElse(null);
        if (link == null || !link.workItemId.equals(workItemId)) {
            return false;
        }
        linkStore.delete(link);
        return true;
    }

    /**
     * Refresh the status and title of all links for a WorkItem by fetching from the remote trackers.
     *
     * <p>
     * Failed fetches for individual links are logged and skipped — partial sync is preferable
     * to an all-or-nothing failure.
     *
     * @param workItemId the WorkItem UUID
     * @return the number of links successfully synced
     */
    @Transactional
    public int syncLinks(final UUID workItemId) {
        final List<WorkItemIssueLink> links = linkStore.findByWorkItemId(workItemId);
        int synced = 0;
        for (final WorkItemIssueLink link : links) {
            try {
                final IssueTrackerProvider provider = providerFor(link.trackerType);
                final ExternalIssueRef ref = provider.fetchIssue(link.externalRef);
                link.title = ref.title();
                link.url = ref.url();
                link.status = ref.status();
                linkStore.save(link);
                synced++;
            } catch (final Exception e) {
                LOG.warnf("Sync failed for %s:%s — %s", link.trackerType, link.externalRef, e.getMessage());
            }
        }
        return synced;
    }

    /**
     * CDI observer: syncs WorkItem fields to all linked issues on every lifecycle transition,
     * and auto-closes issues when a WorkItem reaches a terminal status (if configured).
     *
     * <p>
     * Sync and close failures are logged and swallowed — they must not roll back the
     * WorkItem transition.
     */
    @Transactional
    public void onLifecycleEvent(@Observes final WorkItemLifecycleEvent event) {
        final List<WorkItemIssueLink> links = linkStore.findByWorkItemId(event.workItemId());
        if (links.isEmpty()) {
            return;
        }

        final io.casehub.work.runtime.model.WorkItem workItem =
                (io.casehub.work.runtime.model.WorkItem) event.source();

        for (final WorkItemIssueLink link : links) {
            final IssueTrackerProvider provider;
            try {
                provider = providerFor(link.trackerType);
            } catch (final IllegalArgumentException e) {
                LOG.warnf("No provider for %s — skipping sync for link %s", link.trackerType, link.id);
                continue;
            }

            try {
                provider.syncToIssue(link.externalRef, workItem);
            } catch (final Exception e) {
                LOG.warnf("syncToIssue failed for %s:%s — %s",
                        link.trackerType, link.externalRef, e.getMessage());
            }

            if (event.status() == WorkItemStatus.COMPLETED && githubConfig.autoCloseOnComplete()) {
                if (!"closed".equals(link.status)) {
                    try {
                        final String resolution = event.detail() != null
                                ? "WorkItem completed. " + event.detail()
                                : "WorkItem completed.";
                        provider.closeIssue(link.externalRef, resolution);
                        link.status = "closed";
                        linkStore.save(link);
                    } catch (final Exception e) {
                        LOG.warnf("Auto-close failed for %s:%s — %s",
                                link.trackerType, link.externalRef, e.getMessage());
                    }
                }
            }
        }
    }

    // ── Private ───────────────────────────────────────────────────────────────

    private IssueTrackerProvider providerFor(final String trackerType) {
        for (final IssueTrackerProvider provider : providers) {
            if (trackerType.equals(provider.trackerType())) {
                return provider;
            }
        }
        throw new IllegalArgumentException(
                "No IssueTrackerProvider registered for type '" + trackerType +
                        "'. Available: " + availableTypes());
    }

    private List<String> availableTypes() {
        return providers.stream().map(IssueTrackerProvider::trackerType).toList();
    }
}
```

- [ ] **Step 2: Write the new pure unit IssueLinkServiceTest**

Replace the entire file content of
`casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/service/IssueLinkServiceTest.java`:

```java
package io.casehub.work.issuetracker.service;

import io.casehub.work.issuetracker.github.GitHubIssueTrackerConfig;
import io.casehub.work.issuetracker.model.WorkItemIssueLink;
import io.casehub.work.issuetracker.repository.IssueLinkStore;
import io.casehub.work.issuetracker.spi.ExternalIssueRef;
import io.casehub.work.issuetracker.spi.IssueTrackerException;
import io.casehub.work.issuetracker.spi.IssueTrackerProvider;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.model.WorkItemStatus;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Iterator;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class IssueLinkServiceTest {

    private IssueLinkStore linkStore;
    private IssueTrackerProvider provider;
    private GitHubIssueTrackerConfig githubConfig;
    private IssueLinkService service;

    @BeforeEach
    void setUp() {
        linkStore = mock(IssueLinkStore.class);
        provider = mock(IssueTrackerProvider.class);
        githubConfig = mock(GitHubIssueTrackerConfig.class);

        when(provider.trackerType()).thenReturn("github");
        when(githubConfig.autoCloseOnComplete()).thenReturn(false);
        when(linkStore.save(any())).thenAnswer(inv -> {
            final WorkItemIssueLink link = inv.getArgument(0);
            if (link.id == null) link.id = UUID.randomUUID();
            return link;
        });

        // Build Instance<IssueTrackerProvider> mock that iterates over [provider]
        @SuppressWarnings("unchecked")
        final Instance<IssueTrackerProvider> instances = mock(Instance.class);
        when(instances.iterator()).thenReturn(List.<IssueTrackerProvider>of(provider).iterator());
        when(instances.stream()).thenReturn(List.<IssueTrackerProvider>of(provider).stream());

        service = new IssueLinkService(instances, githubConfig, linkStore);
    }

    // ── linkExistingIssue ─────────────────────────────────────────────────────

    @Test
    void linkExisting_fetchesAndSavesSnapshot() {
        final UUID workItemId = UUID.randomUUID();
        when(linkStore.findByRef(workItemId, "github", "owner/repo#42")).thenReturn(Optional.empty());
        when(provider.fetchIssue("owner/repo#42"))
                .thenReturn(new ExternalIssueRef("github", "owner/repo#42", "Fix the bug", "https://...", "open"));

        final WorkItemIssueLink result =
                service.linkExistingIssue(workItemId, "github", "owner/repo#42", "alice");

        assertThat(result.workItemId).isEqualTo(workItemId);
        assertThat(result.title).isEqualTo("Fix the bug");
        assertThat(result.status).isEqualTo("open");
        assertThat(result.linkedBy).isEqualTo("alice");
        verify(linkStore).save(any(WorkItemIssueLink.class));
    }

    @Test
    void linkExisting_isIdempotent_returnsExistingLink() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink existing = existingLink(workItemId, "github", "owner/repo#10");
        when(linkStore.findByRef(workItemId, "github", "owner/repo#10")).thenReturn(Optional.of(existing));

        final WorkItemIssueLink result =
                service.linkExistingIssue(workItemId, "github", "owner/repo#10", "bob");

        assertThat(result.id).isEqualTo(existing.id);
        verify(provider, never()).fetchIssue(any());
        verify(linkStore, never()).save(any());
    }

    @Test
    void linkExisting_throws_whenProviderThrowsNotFound() {
        final UUID workItemId = UUID.randomUUID();
        when(linkStore.findByRef(any(), any(), any())).thenReturn(Optional.empty());
        when(provider.fetchIssue("owner/repo#99999"))
                .thenThrow(IssueTrackerException.notFound("owner/repo#99999"));

        assertThatThrownBy(() ->
                service.linkExistingIssue(workItemId, "github", "owner/repo#99999", "alice"))
                .isInstanceOf(IssueTrackerException.class)
                .matches(e -> ((IssueTrackerException) e).isNotFound());
    }

    @Test
    void linkExisting_throws_whenNoProviderRegistered() {
        final UUID workItemId = UUID.randomUUID();
        when(linkStore.findByRef(any(), any(), any())).thenReturn(Optional.empty());

        assertThatThrownBy(() ->
                service.linkExistingIssue(workItemId, "jira", "PROJ-1", "alice"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("jira");
    }

    // ── createAndLink ─────────────────────────────────────────────────────────

    @Test
    void createAndLink_createsIssueAndReturnsLink() {
        final UUID workItemId = UUID.randomUUID();
        when(provider.createIssue(eq(workItemId), eq("Fix it"), any()))
                .thenReturn(Optional.of("owner/repo#99"));
        when(provider.fetchIssue("owner/repo#99"))
                .thenReturn(new ExternalIssueRef("github", "owner/repo#99", "Fix it", "https://...", "open"));

        final WorkItemIssueLink link =
                service.createAndLink(workItemId, "github", "Fix it", "Details", "system");

        assertThat(link.externalRef).isEqualTo("owner/repo#99");
        assertThat(link.title).isEqualTo("Fix it");
        verify(linkStore).save(any(WorkItemIssueLink.class));
    }

    @Test
    void createAndLink_throws_whenProviderDoesNotSupportCreation() {
        final UUID workItemId = UUID.randomUUID();
        when(provider.createIssue(any(), any(), any())).thenReturn(Optional.empty());

        assertThatThrownBy(() ->
                service.createAndLink(workItemId, "github", "Title", "Body", "alice"))
                .isInstanceOf(IssueTrackerException.class)
                .hasMessageContaining("does not support issue creation");
    }

    // ── listLinks ─────────────────────────────────────────────────────────────

    @Test
    void listLinks_delegatesToStore() {
        final UUID workItemId = UUID.randomUUID();
        final List<WorkItemIssueLink> links = List.of(
                existingLink(workItemId, "github", "owner/repo#1"),
                existingLink(workItemId, "github", "owner/repo#2"));
        when(linkStore.findByWorkItemId(workItemId)).thenReturn(links);

        assertThat(service.listLinks(workItemId)).hasSize(2);
        verify(linkStore).findByWorkItemId(workItemId);
    }

    // ── removeLink ────────────────────────────────────────────────────────────

    @Test
    void removeLink_found_deletesAndReturnsTrue() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink link = existingLink(workItemId, "github", "owner/repo#3");
        when(linkStore.findById(link.id)).thenReturn(Optional.of(link));

        assertThat(service.removeLink(link.id, workItemId)).isTrue();
        verify(linkStore).delete(link);
    }

    @Test
    void removeLink_notFound_returnsFalse() {
        when(linkStore.findById(any())).thenReturn(Optional.empty());

        assertThat(service.removeLink(UUID.randomUUID(), UUID.randomUUID())).isFalse();
        verify(linkStore, never()).delete(any());
    }

    @Test
    void removeLink_wrongWorkItem_returnsFalse() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink link = existingLink(workItemId, "github", "owner/repo#4");
        when(linkStore.findById(link.id)).thenReturn(Optional.of(link));

        assertThat(service.removeLink(link.id, UUID.randomUUID())).isFalse();
        verify(linkStore, never()).delete(any());
    }

    // ── syncLinks ─────────────────────────────────────────────────────────────

    @Test
    void syncLinks_updatesStatusAndSaves() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink link = existingLink(workItemId, "github", "owner/repo#6");
        when(linkStore.findByWorkItemId(workItemId)).thenReturn(List.of(link));
        when(provider.fetchIssue("owner/repo#6"))
                .thenReturn(new ExternalIssueRef("github", "owner/repo#6", "Issue", "https://...", "closed"));

        final int synced = service.syncLinks(workItemId);

        assertThat(synced).isEqualTo(1);
        assertThat(link.status).isEqualTo("closed");
        verify(linkStore).save(link);
    }

    @Test
    void syncLinks_continuesOnPartialFailure() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink good = existingLink(workItemId, "github", "owner/repo#7");
        final WorkItemIssueLink broken = existingLink(workItemId, "github", "owner/repo#BROKEN");
        when(linkStore.findByWorkItemId(workItemId)).thenReturn(List.of(good, broken));
        when(provider.fetchIssue("owner/repo#7"))
                .thenReturn(new ExternalIssueRef("github", "owner/repo#7", "Good", "https://...", "open"));
        when(provider.fetchIssue("owner/repo#BROKEN"))
                .thenThrow(IssueTrackerException.notFound("owner/repo#BROKEN"));

        final int synced = service.syncLinks(workItemId);

        assertThat(synced).isEqualTo(1);
        verify(linkStore).save(good);
        verify(linkStore, never()).save(broken);
    }

    // ── onLifecycleEvent ──────────────────────────────────────────────────────

    @Test
    void onLifecycleEvent_callsSyncToIssue_forEachLink() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink link1 = existingLink(workItemId, "github", "owner/repo#20");
        final WorkItemIssueLink link2 = existingLink(workItemId, "github", "owner/repo#21");
        when(linkStore.findByWorkItemId(workItemId)).thenReturn(List.of(link1, link2));

        final WorkItem wi = workItem(workItemId, WorkItemStatus.IN_PROGRESS, WorkItemPriority.HIGH);
        service.onLifecycleEvent(WorkItemLifecycleEvent.of("STARTED", wi, "alice", null));

        verify(provider).syncToIssue("owner/repo#20", wi);
        verify(provider).syncToIssue("owner/repo#21", wi);
    }

    @Test
    void onLifecycleEvent_noLinks_noSync() {
        final UUID workItemId = UUID.randomUUID();
        when(linkStore.findByWorkItemId(workItemId)).thenReturn(List.of());

        final WorkItem wi = workItem(workItemId, WorkItemStatus.PENDING, WorkItemPriority.MEDIUM);
        service.onLifecycleEvent(WorkItemLifecycleEvent.of("CREATED", wi, "system", null));

        verify(provider, never()).syncToIssue(any(), any());
    }

    @Test
    void onLifecycleEvent_autoClose_closesIssueAndSavesLink() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink link = existingLink(workItemId, "github", "owner/repo#30");
        link.status = "open";
        when(linkStore.findByWorkItemId(workItemId)).thenReturn(List.of(link));
        when(githubConfig.autoCloseOnComplete()).thenReturn(true);

        final WorkItem wi = workItem(workItemId, WorkItemStatus.COMPLETED, WorkItemPriority.MEDIUM);
        service.onLifecycleEvent(WorkItemLifecycleEvent.of("COMPLETED", wi, "alice", "done"));

        verify(provider).closeIssue(eq("owner/repo#30"), any());
        assertThat(link.status).isEqualTo("closed");
        verify(linkStore).save(link);
    }

    @Test
    void onLifecycleEvent_unknownProvider_skipsAndContinues() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink jiraLink = existingLink(workItemId, "jira", "PROJ-1");
        final WorkItemIssueLink ghLink = existingLink(workItemId, "github", "owner/repo#40");
        when(linkStore.findByWorkItemId(workItemId)).thenReturn(List.of(jiraLink, ghLink));

        final WorkItem wi = workItem(workItemId, WorkItemStatus.ASSIGNED, WorkItemPriority.LOW);
        service.onLifecycleEvent(WorkItemLifecycleEvent.of("ASSIGNED", wi, "alice", null));

        // github link still synced despite jira having no provider
        verify(provider).syncToIssue("owner/repo#40", wi);
    }

    @Test
    void onLifecycleEvent_syncFailure_swallowed() {
        final UUID workItemId = UUID.randomUUID();
        final WorkItemIssueLink link = existingLink(workItemId, "github", "owner/repo#50");
        when(linkStore.findByWorkItemId(workItemId)).thenReturn(List.of(link));
        doThrow(new RuntimeException("network error")).when(provider).syncToIssue(any(), any());

        final WorkItem wi = workItem(workItemId, WorkItemStatus.IN_PROGRESS, WorkItemPriority.HIGH);

        // Should not throw
        service.onLifecycleEvent(WorkItemLifecycleEvent.of("STARTED", wi, "alice", null));
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private WorkItemIssueLink existingLink(
            final UUID workItemId, final String trackerType, final String externalRef) {
        final WorkItemIssueLink link = new WorkItemIssueLink();
        link.id = UUID.randomUUID();
        link.workItemId = workItemId;
        link.trackerType = trackerType;
        link.externalRef = externalRef;
        link.status = "open";
        link.linkedBy = "test";
        return link;
    }

    private WorkItem workItem(final UUID id, final WorkItemStatus status, final WorkItemPriority priority) {
        final WorkItem wi = new WorkItem();
        wi.id = id;
        wi.status = status;
        wi.priority = priority;
        wi.title = "Test WorkItem";
        return wi;
    }
}
```

Note: `IssueLinkService` needs a constructor to inject these fields for unit testing. Add this package-private constructor to `IssueLinkService.java`:

```java
/** Package-private constructor for unit testing without CDI. */
IssueLinkService(
        final Instance<IssueTrackerProvider> providers,
        final GitHubIssueTrackerConfig githubConfig,
        final IssueLinkStore linkStore) {
    this.providers = providers;
    this.githubConfig = githubConfig;
    this.linkStore = linkStore;
}
```

And the no-arg CDI constructor (already implicitly exists — add it explicitly):

```java
IssueLinkService() {
    // CDI no-arg constructor
}
```

- [ ] **Step 3: Run the new unit tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub-work-issue-tracker \
  -Dtest=IssueLinkServiceTest 2>&1 | tail -15
```

Expected: all 16 tests pass

- [ ] **Step 4: Run the full module suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub-work-issue-tracker 2>&1 | tail -8
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git add \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/service/IssueLinkService.java \
  casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/service/IssueLinkServiceTest.java
git commit -m "$(cat <<'EOF'
refactor(issue-tracker): IssueLinkService injects IssueLinkStore (#161)

Replaces all Panache static calls and active-record mutations with
IssueLinkStore. onLifecycleEvent now calls linkStore.save(link) explicitly
for the status="closed" update (was implicit dirty-check). IssueLinkServiceTest
rewritten as 16 pure Mockito unit tests — no @QuarkusTest or H2 needed.

Refs #161, #156
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Documentation sweep

**Files:**
- Modify: `docs/ARCHITECTURE.md`
- Modify: `CLAUDE.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Check what references need updating using IntelliJ search**

Use IntelliJ `search_in_files_by_text` to find all references to `WorkItemIssueLink` static methods and `Panache` in docs. Also search for stale test counts.

If IntelliJ search is unavailable, use:
```bash
grep -rn "findByWorkItemId\|findByRef\|findByTrackerRef\|IssueLinkService\|IssueLinkStore\|JpaIssueLinkStore\|InMemoryIssueLinkStore" \
  /Users/mdproctor/claude/casehub/work/docs \
  /Users/mdproctor/claude/casehub/work/CLAUDE.md
```

- [ ] **Step 2: Update docs/ARCHITECTURE.md**

In the `casehub-work-issue-tracker` section of the module map, add the repository package. Find the existing `model/` and `service/` entries for the issue-tracker module and add:

```
├── repository/
│   ├── IssueLinkStore.java            — SPI: findById, findByWorkItemId, findByRef, findByTrackerRef, save, delete
│   └── jpa/
│       └── JpaIssueLinkStore.java     — default Panache impl (@ApplicationScoped)
```

In the `testing/` module section, add:
```
├── InMemoryIssueLinkStore.java        — @Alternative, no datasource needed
```

- [ ] **Step 3: Update CLAUDE.md**

In the casehub-work-issue-tracker project structure section, add the `repository/` package with `IssueLinkStore` and `JpaIssueLinkStore`.

In the testing module section, add `InMemoryIssueLinkStore`.

In the Known gotchas section, add:
```
- `IssueLinkStore` is the SPI for `WorkItemIssueLink` persistence — inject it rather than calling `WorkItemIssueLink` Panache statics directly. `JpaIssueLinkStore` is the default; `InMemoryIssueLinkStore` in `casehub-work-testing` is the test alternative.
```

- [ ] **Step 4: Update docs/DESIGN.md**

Update the test count for `casehub-work-issue-tracker` to reflect:
- +17 new tests (`InMemoryIssueLinkStoreTest` in testing module)
- +5 new tests (`WebhookEventHandlerTest` public `handle()` tests)
- `IssueLinkServiceTest`: was ~15 `@QuarkusTest` tests, now 16 pure unit tests

Update the testing module test count to reflect the new `InMemoryIssueLinkStoreTest`.

- [ ] **Step 5: Run the full module suites to confirm all tests still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub-work-issue-tracker 2>&1 | tail -5
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl testing 2>&1 | tail -5
```

Expected: `BUILD SUCCESS` for both

- [ ] **Step 6: Commit**

```bash
git add docs/ARCHITECTURE.md CLAUDE.md docs/DESIGN.md
git commit -m "$(cat <<'EOF'
docs: update ARCHITECTURE.md, CLAUDE.md, DESIGN.md for IssueLinkStore (#161)

Add IssueLinkStore SPI, JpaIssueLinkStore, and InMemoryIssueLinkStore to
module maps and project structure. Update test counts. Add IssueLinkStore
gotcha to CLAUDE.md. Cross-references checked and updated.

Refs #161, #156
Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review

**Spec coverage:**
- ✅ `IssueLinkStore` SPI with 6 methods — Task 1
- ✅ `JpaIssueLinkStore` default impl — Task 1
- ✅ `InMemoryIssueLinkStore` in testing module — Task 2
- ✅ `testing/pom.xml` new dependency — Task 2
- ✅ `InMemoryIssueLinkStoreTest` — Task 2 (17 correctness tests)
- ✅ `WebhookEventHandler` injects `IssueLinkStore` + `WorkItemStore` — Task 3
- ✅ 5 new `handle(WebhookEvent)` unit tests — Task 3
- ✅ `IssueLinkService` injects `IssueLinkStore` — Task 4
- ✅ `IssueLinkServiceTest` rewritten as pure unit tests (16 tests) — Task 4
- ✅ `ARCHITECTURE.md`, `CLAUDE.md`, `DESIGN.md` updated — Task 5
- ✅ All commits reference `#161` and `#156`

**Type consistency:**
- `IssueLinkStore.findByRef()` returns `Optional<WorkItemIssueLink>` consistently across all tasks
- `IssueLinkStore.save()` returns `WorkItemIssueLink` consistently
- `WebhookEventHandler` test constructor takes `(IssueLinkStore, WorkItemStore, WorkItemService)` — matches implementation
- `IssueLinkService` test constructor takes `(Instance<IssueTrackerProvider>, GitHubIssueTrackerConfig, IssueLinkStore)` — matches implementation
