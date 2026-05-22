# GitHub + Jira Inbound Webhooks — Phase 2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Receive inbound webhook events from GitHub Issues and Jira and drive WorkItem lifecycle transitions, closing the bidirectional sync loop.

**Architecture:** Each tracker has its own REST endpoint for HMAC verification and payload parsing. A shared `WebhookEventHandler` translates normalised `WebhookEvent` records into WorkItem transitions. Tracker vocabulary never crosses the provider boundary — all events are translated to normative speech acts (DONE/DECLINE/FAILURE) before touching WorkItem state.

**Tech Stack:** Java 21, Quarkus 3.32.2, JAX-RS, Jackson, Panache, JUnit 5, REST Assured, H2 (tests only)

---

## Prerequisites

This plan implements two GitHub issues:
- **#160** — WorkItemPriority rename (Tasks 1–2): CRITICAL→URGENT, NORMAL→MEDIUM
- **#156 Phase 2** — Inbound webhooks (Tasks 3–9)

#160 must land before #156 Phase 2 — label names change.

---

## File Map

**New files — `casehub-work-api`:**
- `src/main/java/io/casehub/work/api/NormativeResolution.java` — DONE/DECLINE/FAILURE enum

**New files — `casehub-work-issue-tracker`:**
- `src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventKind.java` — event kind enum
- `src/main/java/io/casehub/work/issuetracker/webhook/WebhookEvent.java` — normalised event record
- `src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventHandler.java` — transition engine
- `src/main/java/io/casehub/work/issuetracker/github/GitHubWebhookParser.java` — payload parser
- `src/main/java/io/casehub/work/issuetracker/github/GitHubWebhookResource.java` — REST endpoint
- `src/main/java/io/casehub/work/issuetracker/jira/JiraIssueTrackerConfig.java` — Jira config
- `src/main/java/io/casehub/work/issuetracker/jira/JiraWebhookParser.java` — payload parser
- `src/main/java/io/casehub/work/issuetracker/jira/JiraWebhookResource.java` — REST endpoint
- `src/main/resources/db/migration/V5001__priority_rename.sql` — rename stored values
- `src/test/java/io/casehub/work/issuetracker/webhook/WebhookEventHandlerTest.java`
- `src/test/java/io/casehub/work/issuetracker/github/GitHubWebhookParserTest.java`
- `src/test/java/io/casehub/work/issuetracker/github/GitHubWebhookResourceTest.java`
- `src/test/java/io/casehub/work/issuetracker/jira/JiraWebhookParserTest.java`
- `src/test/java/io/casehub/work/issuetracker/jira/JiraWebhookResourceTest.java`
- `src/test/resources/fixtures/github/issue-closed-completed.json`
- `src/test/resources/fixtures/github/issue-closed-not-planned.json`
- `src/test/resources/fixtures/github/issue-assigned.json`
- `src/test/resources/fixtures/github/issue-unassigned.json`
- `src/test/resources/fixtures/github/issue-edited-body.json`
- `src/test/resources/fixtures/github/issue-edited-title.json`
- `src/test/resources/fixtures/github/issue-labeled-priority.json`
- `src/test/resources/fixtures/github/issue-labeled-user.json`
- `src/test/resources/fixtures/github/issue-unlabeled.json`
- `src/test/resources/fixtures/jira/issue-resolved-done.json`
- `src/test/resources/fixtures/jira/issue-resolved-wont-do.json`
- `src/test/resources/fixtures/jira/issue-resolved-cannot-reproduce.json`
- `src/test/resources/fixtures/jira/issue-assigned.json`
- `src/test/resources/fixtures/jira/issue-unassigned.json`
- `src/test/resources/fixtures/jira/issue-updated-description.json`
- `src/test/resources/fixtures/jira/issue-updated-priority.json`

**Modified files:**
- `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemPriority.java` — rename values
- `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubIssueTrackerProvider.java` — update label strings, add `parseWebhookEvent`
- `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubIssueTrackerConfig.java` — add `webhook-secret`
- `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/spi/IssueTrackerProvider.java` — add `parseWebhookEvent` default
- `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/model/WorkItemIssueLink.java` — add `findByTrackerRef`
- All test files referencing `CRITICAL` or `NORMAL` priority

---

## Task 1: Rename WorkItemPriority enum values (Refs #160)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/model/WorkItemPriority.java`
- Modify: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubIssueTrackerProvider.java`

- [ ] **Step 1: Update the enum**

Replace the entire file content:

```java
package io.casehub.work.runtime.model;

/**
 * Priority level of a {@link WorkItem}, used to drive inbox ordering and
 * escalation policy thresholds. Aligns with Linear's priority vocabulary.
 */
public enum WorkItemPriority {

    /** Low priority — attend to when time permits. */
    LOW,

    /** Medium priority — standard processing order. */
    MEDIUM,

    /** High priority — handle before MEDIUM and LOW items. */
    HIGH,

    /** Urgent priority — requires immediate attention; handle before all others. */
    URGENT
}
```

- [ ] **Step 2: Update GitHubIssueTrackerProvider label strings**

In `priorityLabel()`, replace:
```java
case CRITICAL -> "priority:critical";
case NORMAL -> "priority:normal";
```
With:
```java
case URGENT -> "priority:urgent";
case MEDIUM -> "priority:medium";
```

In `labelColor()`, replace:
```java
if (name.startsWith("priority:critical")) return "E11D48";
if (name.startsWith("priority:normal")) return "3B82F6";
```
With:
```java
if (name.startsWith("priority:urgent")) return "E11D48";
if (name.startsWith("priority:medium")) return "3B82F6";
```

- [ ] **Step 3: Find and fix all test files referencing CRITICAL or NORMAL priority**

```bash
grep -rn "CRITICAL\|WorkItemPriority.NORMAL\|priority:critical\|priority:normal" \
  /Users/mdproctor/claude/casehub/work/casehub-work-issue-tracker/src/test \
  /Users/mdproctor/claude/casehub/work/runtime/src/test \
  /Users/mdproctor/claude/casehub/work/casehub-work-queues/src/test \
  /Users/mdproctor/claude/casehub/work/casehub-work-reports/src/test \
  /Users/mdproctor/claude/casehub/work/casehub-work-ai/src/test
```

For each occurrence, replace `CRITICAL` → `URGENT` and `NORMAL` → `MEDIUM`.

- [ ] **Step 4: Fix runtime code using NORMAL as default**

In `WorkItemService.java` line 71, replace:
```java
item.priority = request.priority() != null ? request.priority() : WorkItemPriority.NORMAL;
```
With:
```java
item.priority = request.priority() != null ? request.priority() : WorkItemPriority.MEDIUM;
```

Search for any other `WorkItemPriority.NORMAL` defaults across the codebase:
```bash
grep -rn "WorkItemPriority.NORMAL\|WorkItemPriority.CRITICAL" \
  /Users/mdproctor/claude/casehub/work --include="*.java" | grep -v test
```

- [ ] **Step 5: Compile to catch missed references**

```bash
scripts/mvn-compile runtime
scripts/mvn-compile casehub-work-issue-tracker
```

Fix any remaining compilation errors.

- [ ] **Step 6: Run tests for affected modules**

```bash
scripts/mvn-test runtime
scripts/mvn-test casehub-work-issue-tracker
```

Expected: all tests pass.

- [ ] **Step 7: Install runtime to local Maven repo**

```bash
scripts/mvn-install runtime
```

- [ ] **Step 8: Commit**

```bash
git add -p
git commit -m "refactor: rename WorkItemPriority CRITICAL->URGENT, NORMAL->MEDIUM (#160)

Aligns priority vocabulary with Linear (Urgent/High/Medium/Low).
CRITICAL was ITSM/incident severity, not a scheduling priority.
NORMAL was generic; MEDIUM is the standard term in task management.

Updates enum, GitHub label strings, label colors, default priority,
and all test references.

Closes #160
Refs #156"
```

---

## Task 2: Flyway migration for stored priority values (Refs #160)

**Files:**
- Create: `casehub-work-issue-tracker/src/main/resources/db/migration/V5001__priority_rename.sql`

- [ ] **Step 1: Write migration**

```sql
-- Rename WorkItemPriority stored values: CRITICAL->URGENT, NORMAL->MEDIUM
-- work_item table is in the runtime module (V1 schema); this migration runs
-- when casehub-work-issue-tracker is on the classpath alongside runtime.
UPDATE work_item SET priority = 'URGENT' WHERE priority = 'CRITICAL';
UPDATE work_item SET priority = 'MEDIUM' WHERE priority = 'NORMAL';
```

- [ ] **Step 2: Verify migration runs cleanly**

```bash
scripts/mvn-test casehub-work-issue-tracker
```

Expected: Flyway applies V5001 and all tests pass.

- [ ] **Step 3: Commit**

```bash
git add casehub-work-issue-tracker/src/main/resources/db/migration/V5001__priority_rename.sql
git commit -m "feat(migration): V5001 rename priority values CRITICAL->URGENT, NORMAL->MEDIUM (#160)

Closes #160"
```

---

## Task 3: NormativeResolution enum (Refs #156)

**Files:**
- Create: `casehub-work-api/src/main/java/io/casehub/work/api/NormativeResolution.java`

- [ ] **Step 1: Create the enum**

```java
package io.casehub.work.api;

/**
 * Normative resolution of a closed work item — grounded in speech act theory.
 *
 * <ul>
 *   <li>{@link #DONE} — work was fulfilled (COMMAND discharged successfully)</li>
 *   <li>{@link #DECLINE} — work was deliberately refused (won't be done)</li>
 *   <li>{@link #FAILURE} — work was attempted but could not be completed</li>
 * </ul>
 *
 * Maps to {@link io.casehub.work.runtime.model.WorkItemStatus}:
 * DONE → COMPLETED, DECLINE → CANCELLED, FAILURE → REJECTED.
 */
public enum NormativeResolution {
    /** Work fulfilled — the obligation was discharged. */
    DONE,
    /** Work refused — deliberate decision not to proceed. */
    DECLINE,
    /** Work attempted but could not be completed. */
    FAILURE
}
```

- [ ] **Step 2: Install casehub-work-api**

```bash
scripts/mvn-install casehub-work-api
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add casehub-work-api/src/main/java/io/casehub/work/api/NormativeResolution.java
git commit -m "feat(api): add NormativeResolution enum — DONE/DECLINE/FAILURE (#156)

Speech-act grounded resolution type for inbound tracker events.
Maps tracker close vocabulary to WorkItem terminal transitions
without leaking tracker-specific terms past the provider boundary.

Refs #156, #159"
```

---

## Task 4: WebhookEvent types and IssueTrackerProvider extension (Refs #156)

**Files:**
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventKind.java`
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEvent.java`
- Modify: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/spi/IssueTrackerProvider.java`
- Modify: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/model/WorkItemIssueLink.java`

- [ ] **Step 1: Create WebhookEventKind**

```java
package io.casehub.work.issuetracker.webhook;

/** The kind of inbound tracker event, normalised across GitHub and Jira. */
public enum WebhookEventKind {
    /** Issue closed — check {@link WebhookEvent#normativeResolution()} for DONE/DECLINE/FAILURE. */
    CLOSED,
    /** An actor was assigned to the issue. */
    ASSIGNED,
    /** The issue was unassigned. */
    UNASSIGNED,
    /** The issue title (GitHub) or summary (Jira) changed. */
    TITLE_CHANGED,
    /** The issue body (GitHub) or description (Jira) changed. */
    DESCRIPTION_CHANGED,
    /** The issue priority changed. */
    PRIORITY_CHANGED,
    /** A non-managed label was added. */
    LABEL_ADDED,
    /** A non-managed label was removed. */
    LABEL_REMOVED
}
```

- [ ] **Step 2: Create WebhookEvent record**

```java
package io.casehub.work.issuetracker.webhook;

import io.casehub.work.api.NormativeResolution;
import io.casehub.work.runtime.model.WorkItemPriority;

/**
 * Normalised inbound event from an external issue tracker.
 *
 * <p>Tracker-specific vocabulary (GitHub actions, Jira changelog fields) is
 * translated into this record by the {@link io.casehub.work.issuetracker.spi.IssueTrackerProvider}.
 * The {@link io.casehub.work.issuetracker.webhook.WebhookEventHandler} then applies
 * the appropriate WorkItem transition without knowing which tracker sent it.
 *
 * <p>Unused fields for a given {@link WebhookEventKind} are {@code null}.
 */
public record WebhookEvent(
        /** The tracker type: {@code "github"} or {@code "jira"}. */
        String trackerType,
        /** Tracker-specific issue reference, e.g. {@code "owner/repo#42"} or {@code "PROJ-1234"}. */
        String externalRef,
        /** The kind of event. */
        WebhookEventKind eventKind,
        /** The actor who triggered the event (GitHub login or Jira display name). */
        String actor,
        /** Resolution type — only set when {@code eventKind == CLOSED}. */
        NormativeResolution normativeResolution,
        /** New priority — only set when {@code eventKind == PRIORITY_CHANGED}. */
        WorkItemPriority newPriority,
        /** Label value — only set when {@code eventKind == LABEL_ADDED or LABEL_REMOVED}. */
        String labelValue,
        /** New title — only set when {@code eventKind == TITLE_CHANGED}. */
        String newTitle,
        /** New description — only set when {@code eventKind == DESCRIPTION_CHANGED}. */
        String newDescription,
        /** New assignee ID — only set when {@code eventKind == ASSIGNED}. */
        String newAssignee) {
}
```

- [ ] **Step 3: Add parseWebhookEvent default to IssueTrackerProvider**

Add after the `syncToIssue` default method:

```java
/**
 * Parse a raw inbound webhook payload into a normalised {@link WebhookEvent}.
 *
 * <p>The implementation is responsible for HMAC verification of the raw body
 * before calling this method. This method only parses — it does not verify.
 *
 * <p>Returns {@code null} if the payload represents an event type this provider
 * does not handle (e.g. {@code action: milestoned}) — the caller should return
 * HTTP 200 and take no action.
 *
 * <p>Default: throws {@link UnsupportedOperationException} — providers that do
 * not support inbound webhooks need not implement this.
 *
 * @param headers the HTTP request headers, lower-cased
 * @param body the raw request body as a UTF-8 string
 * @return a normalised event, or {@code null} if the event is not handled
 */
default io.casehub.work.issuetracker.webhook.WebhookEvent parseWebhookEvent(
        java.util.Map<String, String> headers, String body) {
    throw new UnsupportedOperationException(
            trackerType() + " does not support inbound webhooks");
}
```

- [ ] **Step 4: Add findByTrackerRef to WorkItemIssueLink**

Add after `findByRef`:

```java
/**
 * Return all links for a given tracker type and external ref, across all WorkItems.
 * Used by webhook handlers that receive the tracker ref but not the WorkItem ID.
 *
 * @param trackerType the tracker type string (e.g. {@code "github"})
 * @param externalRef the tracker-specific reference (e.g. {@code "owner/repo#42"})
 * @return list of links; may be empty if no WorkItem is linked to this ref
 */
public static List<WorkItemIssueLink> findByTrackerRef(
        final String trackerType, final String externalRef) {
    return list("trackerType = ?1 AND externalRef = ?2", trackerType, externalRef);
}
```

- [ ] **Step 5: Compile**

```bash
scripts/mvn-compile casehub-work-issue-tracker
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventKind.java \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEvent.java \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/spi/IssueTrackerProvider.java \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/model/WorkItemIssueLink.java
git commit -m "feat(issue-tracker): WebhookEvent types + IssueTrackerProvider extension (#156)

WebhookEventKind enum, WebhookEvent record (normalised cross-tracker event),
parseWebhookEvent default on IssueTrackerProvider, findByTrackerRef on
WorkItemIssueLink for webhook-driven WorkItem lookup without workItemId.

Refs #156"
```

---

## Task 5: WebhookEventHandler — core transition engine (Refs #156)

**Files:**
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventHandler.java`
- Create: `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/webhook/WebhookEventHandlerTest.java`

- [ ] **Step 1: Write failing tests first**

```java
package io.casehub.work.issuetracker.webhook;

import io.casehub.work.api.NormativeResolution;
import io.casehub.work.issuetracker.model.WorkItemIssueLink;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.service.WorkItemService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.util.List;
import java.util.UUID;

import static org.mockito.Mockito.*;

class WebhookEventHandlerTest {

    private WorkItemService workItemService;
    private WebhookEventHandler handler;
    private UUID workItemId;

    @BeforeEach
    void setUp() {
        workItemService = Mockito.mock(WorkItemService.class);
        handler = new WebhookEventHandler(workItemService);
        workItemId = UUID.randomUUID();
    }

    // -- helper --

    private WorkItem activeWorkItem() {
        WorkItem wi = new WorkItem();
        wi.id = workItemId;
        wi.status = WorkItemStatus.ASSIGNED;
        wi.assigneeId = "alice";
        wi.labels = new java.util.ArrayList<>();
        return wi;
    }

    // CLOSED + DONE → complete()
    @Test
    void closed_done_callsComplete() {
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        handler.applyTransition(workItemId, activeWorkItem(), event);

        verify(workItemService).complete(workItemId, "alice", null);
    }

    // CLOSED + DECLINE → cancel()
    @Test
    void closed_decline_callsCancel() {
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DECLINE, null, null, null, null, null);

        handler.applyTransition(workItemId, activeWorkItem(), event);

        verify(workItemService).cancel(workItemId, "alice", null);
    }

    // CLOSED + FAILURE → reject()
    @Test
    void closed_failure_callsReject() {
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.FAILURE, null, null, null, null, null);

        handler.applyTransition(workItemId, activeWorkItem(), event);

        verify(workItemService).reject(workItemId, "alice", null);
    }

    // ASSIGNED on PENDING WorkItem → claim()
    @Test
    void assigned_pending_callsClaim() {
        WorkItem wi = new WorkItem();
        wi.id = workItemId;
        wi.status = WorkItemStatus.PENDING;
        wi.labels = new java.util.ArrayList<>();

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.ASSIGNED,
                "alice", null, null, null, null, null, "bob");

        handler.applyTransition(workItemId, wi, event);

        verify(workItemService).claim(workItemId, "bob");
    }

    // ASSIGNED on already-assigned WorkItem → direct field update (no service call for claim)
    @Test
    void assigned_alreadyAssigned_updatesAssignee() {
        WorkItem wi = activeWorkItem(); // assigneeId = "alice"

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.ASSIGNED,
                "alice", null, null, null, null, null, "carol");

        handler.applyTransition(workItemId, wi, event);

        verify(workItemService, never()).claim(any(), any());
        assert "carol".equals(wi.assigneeId);
    }

    // UNASSIGNED → release()
    @Test
    void unassigned_callsRelease() {
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.UNASSIGNED,
                "alice", null, null, null, null, null, null);

        handler.applyTransition(workItemId, activeWorkItem(), event);

        verify(workItemService).release(workItemId, "alice");
    }

    // TITLE_CHANGED → workItem.title updated
    @Test
    void titleChanged_updatesTitle() {
        WorkItem wi = activeWorkItem();
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.TITLE_CHANGED,
                "alice", null, null, null, "New Title", null, null);

        handler.applyTransition(workItemId, wi, event);

        assert "New Title".equals(wi.title);
        verifyNoMoreInteractions(workItemService);
    }

    // DESCRIPTION_CHANGED → footer stripped, workItem.description updated
    @Test
    void descriptionChanged_stripsFooterAndUpdates() {
        WorkItem wi = activeWorkItem();
        String rawBody = "The real description.\n\n---\n*Linked WorkItem: `" + workItemId + "`*";
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.DESCRIPTION_CHANGED,
                "alice", null, null, null, null, rawBody, null);

        handler.applyTransition(workItemId, wi, event);

        assert "The real description.".equals(wi.description);
    }

    // PRIORITY_CHANGED → workItem.priority updated
    @Test
    void priorityChanged_updatesPriority() {
        WorkItem wi = activeWorkItem();
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.PRIORITY_CHANGED,
                "alice", null, WorkItemPriority.HIGH, null, null, null, null);

        handler.applyTransition(workItemId, wi, event);

        assert WorkItemPriority.HIGH == wi.priority;
    }

    // LABEL_ADDED → label appears in workItem.labels
    @Test
    void labelAdded_appendsLabel() {
        WorkItem wi = activeWorkItem();
        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.LABEL_ADDED,
                "alice", null, null, "legal/nda", null, null, null);

        handler.applyTransition(workItemId, wi, event);

        assert wi.labels.stream().anyMatch(l -> "legal/nda".equals(l.path));
    }

    // LABEL_ADDED duplicate → not added twice
    @Test
    void labelAdded_duplicate_notAddedTwice() {
        WorkItem wi = activeWorkItem();
        io.casehub.work.runtime.model.WorkItemLabel existing = new io.casehub.work.runtime.model.WorkItemLabel();
        existing.path = "legal/nda";
        wi.labels.add(existing);

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.LABEL_ADDED,
                "alice", null, null, "legal/nda", null, null, null);

        handler.applyTransition(workItemId, wi, event);

        assert wi.labels.size() == 1;
    }

    // LABEL_REMOVED → label removed from workItem.labels
    @Test
    void labelRemoved_removesLabel() {
        WorkItem wi = activeWorkItem();
        io.casehub.work.runtime.model.WorkItemLabel label = new io.casehub.work.runtime.model.WorkItemLabel();
        label.path = "legal/nda";
        wi.labels.add(label);

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.LABEL_REMOVED,
                "alice", null, null, "legal/nda", null, null, null);

        handler.applyTransition(workItemId, wi, event);

        assert wi.labels.isEmpty();
    }

    // Terminal WorkItem → skipped silently
    @Test
    void terminalWorkItem_skipped() {
        WorkItem wi = new WorkItem();
        wi.id = workItemId;
        wi.status = WorkItemStatus.COMPLETED;

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        handler.handle(workItemId, wi, event);

        verifyNoInteractions(workItemService);
    }

    // Transition failure → swallowed (no exception propagated)
    @Test
    void transitionFailure_swallowed() {
        doThrow(new IllegalStateException("wrong status"))
                .when(workItemService).complete(any(), any(), any());

        WebhookEvent event = new WebhookEvent(
                "github", "owner/repo#42", WebhookEventKind.CLOSED,
                "alice", NormativeResolution.DONE, null, null, null, null, null);

        // Should not throw
        handler.handle(workItemId, activeWorkItem(), event);
    }
}
```

- [ ] **Step 2: Run test — verify it fails to compile (handler doesn't exist yet)**

```bash
scripts/mvn-compile casehub-work-issue-tracker
```

Expected: compilation errors on `WebhookEventHandler`.

- [ ] **Step 3: Implement WebhookEventHandler**

```java
package io.casehub.work.issuetracker.webhook;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.jboss.logging.Logger;

import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemLabel;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.service.WorkItemService;

/**
 * Applies normalised {@link WebhookEvent} records to WorkItem state.
 * Called by tracker-specific webhook resources after HMAC verification and parsing.
 * Tracker vocabulary never reaches this class.
 */
@ApplicationScoped
public class WebhookEventHandler {

    private static final Logger LOG = Logger.getLogger(WebhookEventHandler.class);
    private static final String LINKED_WORKITEM_FOOTER = "\n\n---\n*Linked WorkItem:";

    @Inject
    WorkItemService workItemService;

    /** Constructor for unit testing without CDI. */
    WebhookEventHandler(final WorkItemService workItemService) {
        this.workItemService = workItemService;
    }

    WebhookEventHandler() {
        // CDI
    }

    /**
     * Look up all WorkItems linked to the event's externalRef and apply the transition to each.
     * Failures per WorkItem are logged and swallowed — prevents tracker retries.
     */
    @Transactional
    public void handle(final WebhookEvent event) {
        final var links = io.casehub.work.issuetracker.model.WorkItemIssueLink
                .findByTrackerRef(event.trackerType(), event.externalRef());

        if (links.isEmpty()) {
            LOG.debugf("No WorkItem linked to %s:%s — ignoring", event.trackerType(), event.externalRef());
            return;
        }

        for (final var link : links) {
            final WorkItem workItem = WorkItem.findById(link.workItemId);
            if (workItem == null) continue;
            handle(link.workItemId, workItem, event);
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
            // Admin reassignment from tracker — update directly
            workItem.assigneeId = event.newAssignee();
        }
    }

    private String stripFooter(final String description) {
        if (description == null) return null;
        final int idx = description.indexOf(LINKED_WORKITEM_FOOTER);
        return idx >= 0 ? description.substring(0, idx) : description;
    }

    private void addLabel(final WorkItem workItem, final String path) {
        if (workItem.labels == null) workItem.labels = new java.util.ArrayList<>();
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

- [ ] **Step 4: Check WorkItemLabel structure**

Before compiling, verify `WorkItemLabel` has a `path` field:
```bash
grep -n "public.*path\|String path" \
  /Users/mdproctor/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/model/WorkItemLabel.java
```

If the field is named differently, update `addLabel` and `removeLabel` accordingly.

- [ ] **Step 5: Run the tests**

```bash
scripts/mvn-test casehub-work-issue-tracker -Dtest=WebhookEventHandlerTest
```

Expected: all 13 tests pass.

- [ ] **Step 6: Commit**

```bash
git add \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/webhook/WebhookEventHandler.java \
  casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/webhook/WebhookEventHandlerTest.java
git commit -m "feat(issue-tracker): WebhookEventHandler — normative transition engine (#156)

Applies normalised WebhookEvents to WorkItem state. Handles all 8 event
kinds: CLOSED (DONE/DECLINE/FAILURE), ASSIGNED, UNASSIGNED, TITLE_CHANGED,
DESCRIPTION_CHANGED, PRIORITY_CHANGED, LABEL_ADDED, LABEL_REMOVED.
Terminal WorkItems silently skipped. Transition failures logged + swallowed.
13 unit tests.

Refs #156"
```

---

## Task 6: GitHub webhook fixtures and parser (Refs #156)

**Files:**
- Create: `casehub-work-issue-tracker/src/test/resources/fixtures/github/*.json` (9 files)
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubWebhookParser.java`
- Create: `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/github/GitHubWebhookParserTest.java`

- [ ] **Step 1: Create GitHub fixture files**

`src/test/resources/fixtures/github/issue-closed-completed.json`:
```json
{
  "action": "closed",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Token expires too quickly.\n\n---\n*Linked WorkItem: `550e8400-e29b-41d4-a716-446655440000`*",
    "state": "closed",
    "state_reason": "completed",
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": { "login": "alice" },
    "labels": []
  },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

`src/test/resources/fixtures/github/issue-closed-not-planned.json`:
```json
{
  "action": "closed",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Token expires too quickly.",
    "state": "closed",
    "state_reason": "not_planned",
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": null,
    "labels": []
  },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "bob" }
}
```

`src/test/resources/fixtures/github/issue-closed-legacy.json` (no state_reason — legacy close):
```json
{
  "action": "closed",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Token expires too quickly.",
    "state": "closed",
    "state_reason": null,
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": null,
    "labels": []
  },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

`src/test/resources/fixtures/github/issue-assigned.json`:
```json
{
  "action": "assigned",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Token expires too quickly.",
    "state": "open",
    "state_reason": null,
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": { "login": "bob" },
    "labels": []
  },
  "assignee": { "login": "bob" },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

`src/test/resources/fixtures/github/issue-unassigned.json`:
```json
{
  "action": "unassigned",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Token expires too quickly.",
    "state": "open",
    "state_reason": null,
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": null,
    "labels": []
  },
  "assignee": { "login": "bob" },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

`src/test/resources/fixtures/github/issue-edited-body.json`:
```json
{
  "action": "edited",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Updated description text.",
    "state": "open",
    "state_reason": null,
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": null,
    "labels": []
  },
  "changes": {
    "body": { "from": "Original description." }
  },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

`src/test/resources/fixtures/github/issue-edited-title.json`:
```json
{
  "action": "edited",
  "issue": {
    "number": 42,
    "title": "Updated title",
    "body": "Description text.",
    "state": "open",
    "state_reason": null,
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": null,
    "labels": []
  },
  "changes": {
    "title": { "from": "Original title" }
  },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

`src/test/resources/fixtures/github/issue-labeled-priority.json`:
```json
{
  "action": "labeled",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Description.",
    "state": "open",
    "state_reason": null,
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": null,
    "labels": [{ "name": "priority:high" }]
  },
  "label": { "name": "priority:high" },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

`src/test/resources/fixtures/github/issue-labeled-user.json`:
```json
{
  "action": "labeled",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Description.",
    "state": "open",
    "state_reason": null,
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": null,
    "labels": [{ "name": "legal/nda" }]
  },
  "label": { "name": "legal/nda" },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

`src/test/resources/fixtures/github/issue-unlabeled.json`:
```json
{
  "action": "unlabeled",
  "issue": {
    "number": 42,
    "title": "Fix authentication bug",
    "body": "Description.",
    "state": "open",
    "state_reason": null,
    "html_url": "https://github.com/owner/repo/issues/42",
    "assignee": null,
    "labels": []
  },
  "label": { "name": "legal/nda" },
  "repository": { "full_name": "owner/repo" },
  "sender": { "login": "alice" }
}
```

- [ ] **Step 2: Write failing parser tests**

```java
package io.casehub.work.issuetracker.github;

import io.casehub.work.api.NormativeResolution;
import io.casehub.work.issuetracker.webhook.WebhookEvent;
import io.casehub.work.issuetracker.webhook.WebhookEventKind;
import io.casehub.work.runtime.model.WorkItemPriority;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class GitHubWebhookParserTest {

    private final GitHubWebhookParser parser = new GitHubWebhookParser();

    private String fixture(final String name) throws IOException {
        try (var stream = getClass().getResourceAsStream("/fixtures/github/" + name)) {
            return new String(stream.readAllBytes(), StandardCharsets.UTF_8);
        }
    }

    @Test
    void closedCompleted_parsesDone() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-closed-completed.json"));

        assertThat(event).isNotNull();
        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.CLOSED);
        assertThat(event.normativeResolution()).isEqualTo(NormativeResolution.DONE);
        assertThat(event.externalRef()).isEqualTo("owner/repo#42");
        assertThat(event.actor()).isEqualTo("alice");
        assertThat(event.trackerType()).isEqualTo("github");
    }

    @Test
    void closedNotPlanned_parsesDecline() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-closed-not-planned.json"));

        assertThat(event.normativeResolution()).isEqualTo(NormativeResolution.DECLINE);
    }

    @Test
    void closedLegacy_nullStateReason_parsesDone() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-closed-legacy.json"));

        assertThat(event.normativeResolution()).isEqualTo(NormativeResolution.DONE);
    }

    @Test
    void assigned_parsesAssigned() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-assigned.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.ASSIGNED);
        assertThat(event.newAssignee()).isEqualTo("bob");
        assertThat(event.actor()).isEqualTo("alice");
    }

    @Test
    void unassigned_parsesUnassigned() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-unassigned.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.UNASSIGNED);
        assertThat(event.actor()).isEqualTo("alice");
    }

    @Test
    void editedBody_parsesDescriptionChanged() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-edited-body.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.DESCRIPTION_CHANGED);
        assertThat(event.newDescription()).isEqualTo("Updated description text.");
    }

    @Test
    void editedTitle_parsesTitleChanged() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-edited-title.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.TITLE_CHANGED);
        assertThat(event.newTitle()).isEqualTo("Updated title");
    }

    @Test
    void labeledPriority_parsesPriorityChanged() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-labeled-priority.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.PRIORITY_CHANGED);
        assertThat(event.newPriority()).isEqualTo(WorkItemPriority.HIGH);
    }

    @Test
    void labeledUser_parsesLabelAdded() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-labeled-user.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.LABEL_ADDED);
        assertThat(event.labelValue()).isEqualTo("legal/nda");
    }

    @Test
    void unlabeled_parsesLabelRemoved() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-unlabeled.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.LABEL_REMOVED);
        assertThat(event.labelValue()).isEqualTo("legal/nda");
    }

    @Test
    void reopened_returnsNull() throws Exception {
        final String body = """
            {"action":"reopened","issue":{"number":42,"title":"T","body":"B",
             "state":"open","state_reason":null,"html_url":"https://github.com/o/r/issues/42",
             "assignee":null,"labels":[]},"repository":{"full_name":"o/r"},
             "sender":{"login":"alice"}}
            """;
        assertThat(parser.parse(Map.of(), body)).isNull();
    }
}
```

- [ ] **Step 3: Run tests — verify compilation failure**

```bash
scripts/mvn-compile casehub-work-issue-tracker
```

Expected: `GitHubWebhookParser` not found.

- [ ] **Step 4: Implement GitHubWebhookParser**

```java
package io.casehub.work.issuetracker.github;

import java.util.Map;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.work.api.NormativeResolution;
import io.casehub.work.issuetracker.webhook.WebhookEvent;
import io.casehub.work.issuetracker.webhook.WebhookEventKind;
import io.casehub.work.runtime.model.WorkItemPriority;

/**
 * Parses raw GitHub Issues webhook payloads into normalised {@link WebhookEvent} records.
 */
@ApplicationScoped
public class GitHubWebhookParser {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Set<String> MANAGED_LABEL_PREFIXES = Set.of("priority:", "status:", "category:");

    /**
     * Parse a raw GitHub webhook body into a {@link WebhookEvent}.
     *
     * @return the event, or {@code null} if the action is not handled
     */
    public WebhookEvent parse(final Map<String, String> headers, final String body) {
        try {
            final JsonNode root = MAPPER.readTree(body);
            final String action = root.path("action").asText("");
            final JsonNode issue = root.path("issue");
            final String repo = root.path("repository").path("full_name").asText("");
            final String number = String.valueOf(issue.path("number").asInt());
            final String externalRef = repo + "#" + number;
            final String actor = root.path("sender").path("login").asText("unknown");

            return switch (action) {
                case "closed" -> parseClosed(externalRef, actor, issue);
                case "assigned" -> parseAssigned(externalRef, actor, root);
                case "unassigned" -> new WebhookEvent("github", externalRef, WebhookEventKind.UNASSIGNED,
                        actor, null, null, null, null, null, null);
                case "edited" -> parseEdited(externalRef, actor, issue, root);
                case "labeled" -> parseLabeled(externalRef, actor, root);
                case "unlabeled" -> parseLabelRemoved(externalRef, actor, root);
                default -> null; // not handled — caller returns 200
            };
        } catch (final Exception e) {
            throw new io.casehub.work.issuetracker.spi.IssueTrackerException(
                    "Failed to parse GitHub webhook payload: " + e.getMessage(), e);
        }
    }

    private WebhookEvent parseClosed(final String externalRef, final String actor, final JsonNode issue) {
        final String stateReason = issue.path("state_reason").asText(null);
        final NormativeResolution resolution = "not_planned".equals(stateReason)
                ? NormativeResolution.DECLINE
                : NormativeResolution.DONE; // "completed" or null (legacy)
        return new WebhookEvent("github", externalRef, WebhookEventKind.CLOSED,
                actor, resolution, null, null, null, null, null);
    }

    private WebhookEvent parseAssigned(final String externalRef, final String actor, final JsonNode root) {
        final String assignee = root.path("assignee").path("login").asText(null);
        return new WebhookEvent("github", externalRef, WebhookEventKind.ASSIGNED,
                actor, null, null, null, null, null, assignee);
    }

    private WebhookEvent parseEdited(
            final String externalRef, final String actor, final JsonNode issue, final JsonNode root) {
        final JsonNode changes = root.path("changes");
        if (changes.has("body")) {
            return new WebhookEvent("github", externalRef, WebhookEventKind.DESCRIPTION_CHANGED,
                    actor, null, null, null, null, issue.path("body").asText(null), null);
        }
        if (changes.has("title")) {
            return new WebhookEvent("github", externalRef, WebhookEventKind.TITLE_CHANGED,
                    actor, null, null, null, issue.path("title").asText(null), null, null);
        }
        return null; // other field edited — not handled
    }

    private WebhookEvent parseLabeled(final String externalRef, final String actor, final JsonNode root) {
        final String labelName = root.path("label").path("name").asText("");
        if (isPriorityLabel(labelName)) {
            final WorkItemPriority priority = parsePriorityLabel(labelName);
            if (priority == null) return null;
            return new WebhookEvent("github", externalRef, WebhookEventKind.PRIORITY_CHANGED,
                    actor, null, priority, null, null, null, null);
        }
        if (isManagedLabel(labelName)) return null; // status:*, category:* — ignore
        return new WebhookEvent("github", externalRef, WebhookEventKind.LABEL_ADDED,
                actor, null, null, labelName, null, null, null);
    }

    private WebhookEvent parseLabelRemoved(final String externalRef, final String actor, final JsonNode root) {
        final String labelName = root.path("label").path("name").asText("");
        if (isManagedLabel(labelName)) return null;
        return new WebhookEvent("github", externalRef, WebhookEventKind.LABEL_REMOVED,
                actor, null, null, labelName, null, null, null);
    }

    private boolean isPriorityLabel(final String name) {
        return name.startsWith("priority:");
    }

    private boolean isManagedLabel(final String name) {
        return MANAGED_LABEL_PREFIXES.stream().anyMatch(name::startsWith);
    }

    private WorkItemPriority parsePriorityLabel(final String name) {
        return switch (name) {
            case "priority:urgent" -> WorkItemPriority.URGENT;
            case "priority:high" -> WorkItemPriority.HIGH;
            case "priority:medium" -> WorkItemPriority.MEDIUM;
            case "priority:low" -> WorkItemPriority.LOW;
            default -> null;
        };
    }
}
```

- [ ] **Step 5: Run the parser tests**

```bash
scripts/mvn-test casehub-work-issue-tracker -Dtest=GitHubWebhookParserTest
```

Expected: all 11 tests pass.

- [ ] **Step 6: Commit**

```bash
git add \
  casehub-work-issue-tracker/src/test/resources/fixtures/github/ \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubWebhookParser.java \
  casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/github/GitHubWebhookParserTest.java
git commit -m "feat(issue-tracker): GitHub webhook parser + fixtures (#156)

Parses GitHub Issues webhook payloads into normalised WebhookEvents.
Handles: closed (completed/not_planned/legacy), assigned, unassigned,
edited (title/body), labeled (priority + user labels), unlabeled.
Managed labels (priority:*, status:*, category:*) filtered on inbound.
11 parser unit tests with real GitHub payload fixtures.

Refs #156"
```

---

## Task 7: GitHubWebhookResource + config + integration test (Refs #156)

**Files:**
- Modify: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubIssueTrackerConfig.java`
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubWebhookResource.java`
- Create: `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/github/GitHubWebhookResourceTest.java`

- [ ] **Step 1: Add webhook-secret to GitHubIssueTrackerConfig**

Add after `autoCloseOnComplete()`:

```java
/**
 * Shared secret for verifying GitHub webhook signatures.
 * When present, every incoming webhook is verified via HMAC-SHA256
 * against the {@code X-Hub-Signature-256} header. Requests with a missing
 * or invalid signature are rejected with 401.
 * When absent, all incoming webhooks are rejected (fail-closed).
 *
 * @return the webhook secret, or empty if not configured
 */
@WithName("webhook-secret")
Optional<String> webhookSecret();
```

- [ ] **Step 2: Write failing integration test**

```java
package io.casehub.work.issuetracker.github;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.util.HexFormat;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

@QuarkusTest
class GitHubWebhookResourceTest {

    private static final String SECRET = "test-webhook-secret";
    private static final String ENDPOINT = "/workitems/github-webhook";

    private String sign(final String body) throws Exception {
        final Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(SECRET.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
        return "sha256=" + HexFormat.of().formatHex(mac.doFinal(body.getBytes(StandardCharsets.UTF_8)));
    }

    private String fixture(final String name) throws Exception {
        try (var stream = getClass().getResourceAsStream("/fixtures/github/" + name)) {
            return new String(stream.readAllBytes(), StandardCharsets.UTF_8);
        }
    }

    @Test
    void validSignature_returns200() throws Exception {
        final String body = fixture("issue-closed-completed.json");
        given()
            .header("Content-Type", "application/json")
            .header("X-Hub-Signature-256", sign(body))
            .body(body)
        .when()
            .post(ENDPOINT)
        .then()
            .statusCode(200);
    }

    @Test
    void invalidSignature_returns401() throws Exception {
        final String body = fixture("issue-closed-completed.json");
        given()
            .header("Content-Type", "application/json")
            .header("X-Hub-Signature-256", "sha256=deadbeef")
            .body(body)
        .when()
            .post(ENDPOINT)
        .then()
            .statusCode(401);
    }

    @Test
    void missingSignatureHeader_returns401() throws Exception {
        final String body = fixture("issue-closed-completed.json");
        given()
            .header("Content-Type", "application/json")
            .body(body)
        .when()
            .post(ENDPOINT)
        .then()
            .statusCode(401);
    }

    @Test
    void unhandledAction_returns200() throws Exception {
        final String body = """
            {"action":"reopened","issue":{"number":1,"title":"T","body":"B",
             "state":"open","state_reason":null,"html_url":"https://github.com/o/r/issues/1",
             "assignee":null,"labels":[]},"repository":{"full_name":"o/r"},
             "sender":{"login":"alice"}}
            """;
        given()
            .header("Content-Type", "application/json")
            .header("X-Hub-Signature-256", sign(body))
            .body(body)
        .when()
            .post(ENDPOINT)
        .then()
            .statusCode(200);
    }
}
```

Add to `casehub-work-issue-tracker/src/test/resources/application.properties`:
```properties
casehub.work.issue-tracker.github.webhook-secret=test-webhook-secret
```

- [ ] **Step 3: Run test — verify it fails (endpoint doesn't exist)**

```bash
scripts/mvn-test casehub-work-issue-tracker -Dtest=GitHubWebhookResourceTest
```

Expected: FAIL — 404 on POST /workitems/github-webhook.

- [ ] **Step 4: Implement GitHubWebhookResource**

```java
package io.casehub.work.issuetracker.github;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.HexFormat;
import java.util.Map;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.HeaderParam;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import org.jboss.logging.Logger;

import io.casehub.work.issuetracker.webhook.WebhookEvent;
import io.casehub.work.issuetracker.webhook.WebhookEventHandler;

/**
 * Receives inbound GitHub Issues webhook events.
 *
 * <p>Verifies the {@code X-Hub-Signature-256} HMAC before processing.
 * Returns 200 for all valid requests (including unhandled event types) to
 * prevent GitHub retry storms. Returns 401 on signature failure.
 */
@Path("/workitems/github-webhook")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class GitHubWebhookResource {

    private static final Logger LOG = Logger.getLogger(GitHubWebhookResource.class);
    private static final String HMAC_SHA256 = "HmacSHA256";

    @Inject
    GitHubIssueTrackerConfig config;

    @Inject
    GitHubWebhookParser parser;

    @Inject
    WebhookEventHandler handler;

    @POST
    public Response receive(
            @HeaderParam("X-Hub-Signature-256") final String signature,
            final String body) {

        final String secret = config.webhookSecret().orElse(null);
        if (secret == null) {
            LOG.warn("GitHub webhook received but casehub.work.issue-tracker.github.webhook-secret is not set — rejecting");
            return Response.status(Response.Status.UNAUTHORIZED).build();
        }

        if (!verifySignature(secret, body, signature)) {
            LOG.warn("GitHub webhook HMAC verification failed — rejecting");
            return Response.status(Response.Status.UNAUTHORIZED).build();
        }

        try {
            final WebhookEvent event = parser.parse(Map.of(), body);
            if (event != null) {
                handler.handle(event);
            }
        } catch (final Exception e) {
            LOG.warnf("GitHub webhook processing error (returning 200 to prevent retry): %s", e.getMessage());
        }

        return Response.ok().build();
    }

    private boolean verifySignature(final String secret, final String body, final String signature) {
        if (signature == null || signature.isBlank()) return false;
        try {
            final Mac mac = Mac.getInstance(HMAC_SHA256);
            mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), HMAC_SHA256));
            final String expected = "sha256=" +
                    HexFormat.of().formatHex(mac.doFinal(body.getBytes(StandardCharsets.UTF_8)));
            return MessageDigest.isEqual(
                    expected.getBytes(StandardCharsets.UTF_8),
                    signature.getBytes(StandardCharsets.UTF_8));
        } catch (final Exception e) {
            LOG.warnf("HMAC computation failed: %s", e.getMessage());
            return false;
        }
    }
}
```

- [ ] **Step 5: Run the integration test**

```bash
scripts/mvn-test casehub-work-issue-tracker -Dtest=GitHubWebhookResourceTest
```

Expected: all 4 tests pass.

- [ ] **Step 6: Run the full module test suite**

```bash
scripts/mvn-test casehub-work-issue-tracker
```

Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
git add \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubIssueTrackerConfig.java \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/github/GitHubWebhookResource.java \
  casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/github/GitHubWebhookResourceTest.java \
  casehub-work-issue-tracker/src/test/resources/application.properties
git commit -m "feat(issue-tracker): GitHubWebhookResource — POST /workitems/github-webhook (#156)

HMAC-SHA256 verification (X-Hub-Signature-256, constant-time compare).
Fail-closed: 401 when webhook-secret not configured.
Unhandled actions return 200 to prevent GitHub retry storms.
4 @QuarkusTest integration tests.

Refs #156"
```

---

## Task 8: Jira webhook fixtures and parser (Refs #156)

**Files:**
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/jira/JiraIssueTrackerConfig.java`
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/jira/JiraWebhookParser.java`
- Create: `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/jira/JiraWebhookParserTest.java`
- Create: `src/test/resources/fixtures/jira/*.json` (7 files)

- [ ] **Step 1: Create JiraIssueTrackerConfig**

```java
package io.casehub.work.issuetracker.jira;

import java.util.Optional;

import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;
import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithName;

/**
 * Configuration for the Jira issue tracker integration.
 *
 * <pre>{@code
 * # Shared secret for webhook verification (passed as query param by Jira)
 * casehub.work.issue-tracker.jira.webhook-secret=mysecret
 *
 * # Jira base URL (required for outbound calls — not needed for inbound webhooks only)
 * casehub.work.issue-tracker.jira.base-url=https://myorg.atlassian.net
 *
 * # Jira API token (email:token for cloud, PAT for server)
 * casehub.work.issue-tracker.jira.token=user@example.com:mytoken
 * }</pre>
 *
 * <h2>Webhook verification</h2>
 * <p>
 * Jira Cloud does not sign webhook payloads with HMAC. Instead, configure the
 * webhook URL with a secret query parameter:
 * {@code https://yourapp.example.com/workitems/jira-webhook?secret=mysecret}
 * and set {@code casehub.work.issue-tracker.jira.webhook-secret} to the same value.
 * If not configured, inbound webhooks are rejected (fail-closed).
 */
@ConfigMapping(prefix = "casehub.work.issue-tracker.jira")
@ConfigRoot(phase = ConfigPhase.RUN_TIME)
public interface JiraIssueTrackerConfig {

    /**
     * Shared secret verified against the {@code secret} query parameter.
     * Fail-closed: if not configured, all inbound webhooks are rejected.
     */
    @WithName("webhook-secret")
    Optional<String> webhookSecret();

    /** Jira base URL, e.g. {@code https://myorg.atlassian.net}. */
    @WithName("base-url")
    Optional<String> baseUrl();

    /** Jira API token. For Cloud: {@code email:api-token}. For Server: PAT. */
    Optional<String> token();
}
```

- [ ] **Step 2: Create Jira fixture files**

`src/test/resources/fixtures/jira/issue-resolved-done.json`:
```json
{
  "webhookEvent": "jira:issue_updated",
  "issue": {
    "key": "PROJ-1234",
    "fields": {
      "summary": "Fix authentication bug",
      "description": "Token expires too quickly.",
      "status": {
        "name": "Done",
        "statusCategory": { "key": "done" }
      },
      "resolution": { "name": "Done" },
      "assignee": { "accountId": "abc123", "displayName": "Alice" },
      "priority": { "name": "High" },
      "labels": []
    }
  },
  "changelog": {
    "items": [
      { "field": "resolution", "fromString": null, "toString": "Done" },
      { "field": "status", "fromString": "In Progress", "toString": "Done" }
    ]
  },
  "user": { "displayName": "Alice", "accountId": "abc123" }
}
```

`src/test/resources/fixtures/jira/issue-resolved-wont-do.json`:
```json
{
  "webhookEvent": "jira:issue_updated",
  "issue": {
    "key": "PROJ-1234",
    "fields": {
      "summary": "Fix authentication bug",
      "description": "Token expires too quickly.",
      "status": { "name": "Done", "statusCategory": { "key": "done" } },
      "resolution": { "name": "Won't Do" },
      "assignee": null,
      "priority": { "name": "Medium" },
      "labels": []
    }
  },
  "changelog": {
    "items": [
      { "field": "resolution", "fromString": null, "toString": "Won't Do" }
    ]
  },
  "user": { "displayName": "Bob", "accountId": "def456" }
}
```

`src/test/resources/fixtures/jira/issue-resolved-cannot-reproduce.json`:
```json
{
  "webhookEvent": "jira:issue_updated",
  "issue": {
    "key": "PROJ-1234",
    "fields": {
      "summary": "Fix authentication bug",
      "description": "Token expires too quickly.",
      "status": { "name": "Done", "statusCategory": { "key": "done" } },
      "resolution": { "name": "Cannot Reproduce" },
      "assignee": null,
      "priority": { "name": "Low" },
      "labels": []
    }
  },
  "changelog": {
    "items": [
      { "field": "resolution", "fromString": null, "toString": "Cannot Reproduce" }
    ]
  },
  "user": { "displayName": "Alice", "accountId": "abc123" }
}
```

`src/test/resources/fixtures/jira/issue-assigned.json`:
```json
{
  "webhookEvent": "jira:issue_updated",
  "issue": {
    "key": "PROJ-1234",
    "fields": {
      "summary": "Fix authentication bug",
      "description": "Token expires too quickly.",
      "status": { "name": "To Do", "statusCategory": { "key": "new" } },
      "resolution": null,
      "assignee": { "accountId": "abc123", "displayName": "Alice" },
      "priority": { "name": "Medium" },
      "labels": []
    }
  },
  "changelog": {
    "items": [
      { "field": "assignee", "fromString": null, "toString": "Alice", "to": "abc123" }
    ]
  },
  "user": { "displayName": "Bob", "accountId": "def456" }
}
```

`src/test/resources/fixtures/jira/issue-unassigned.json`:
```json
{
  "webhookEvent": "jira:issue_updated",
  "issue": {
    "key": "PROJ-1234",
    "fields": {
      "summary": "Fix authentication bug",
      "description": "Token expires too quickly.",
      "status": { "name": "In Progress", "statusCategory": { "key": "indeterminate" } },
      "resolution": null,
      "assignee": null,
      "priority": { "name": "Medium" },
      "labels": []
    }
  },
  "changelog": {
    "items": [
      { "field": "assignee", "fromString": "Alice", "toString": null, "to": null }
    ]
  },
  "user": { "displayName": "Bob", "accountId": "def456" }
}
```

`src/test/resources/fixtures/jira/issue-updated-description.json`:
```json
{
  "webhookEvent": "jira:issue_updated",
  "issue": {
    "key": "PROJ-1234",
    "fields": {
      "summary": "Fix authentication bug",
      "description": "Updated description text.",
      "status": { "name": "In Progress", "statusCategory": { "key": "indeterminate" } },
      "resolution": null,
      "assignee": { "accountId": "abc123", "displayName": "Alice" },
      "priority": { "name": "High" },
      "labels": []
    }
  },
  "changelog": {
    "items": [
      { "field": "description", "fromString": "Original description.", "toString": "Updated description text." }
    ]
  },
  "user": { "displayName": "Alice", "accountId": "abc123" }
}
```

`src/test/resources/fixtures/jira/issue-updated-priority.json`:
```json
{
  "webhookEvent": "jira:issue_updated",
  "issue": {
    "key": "PROJ-1234",
    "fields": {
      "summary": "Fix authentication bug",
      "description": "Token expires too quickly.",
      "status": { "name": "In Progress", "statusCategory": { "key": "indeterminate" } },
      "resolution": null,
      "assignee": { "accountId": "abc123", "displayName": "Alice" },
      "priority": { "name": "Highest" },
      "labels": []
    }
  },
  "changelog": {
    "items": [
      { "field": "priority", "fromString": "Medium", "toString": "Highest" }
    ]
  },
  "user": { "displayName": "Alice", "accountId": "abc123" }
}
```

- [ ] **Step 3: Write failing parser tests**

```java
package io.casehub.work.issuetracker.jira;

import io.casehub.work.api.NormativeResolution;
import io.casehub.work.issuetracker.webhook.WebhookEvent;
import io.casehub.work.issuetracker.webhook.WebhookEventKind;
import io.casehub.work.runtime.model.WorkItemPriority;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class JiraWebhookParserTest {

    private final JiraWebhookParser parser = new JiraWebhookParser();

    private String fixture(final String name) throws IOException {
        try (var stream = getClass().getResourceAsStream("/fixtures/jira/" + name)) {
            return new String(stream.readAllBytes(), StandardCharsets.UTF_8);
        }
    }

    @Test
    void resolvedDone_parsesDone() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-resolved-done.json"));

        assertThat(event).isNotNull();
        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.CLOSED);
        assertThat(event.normativeResolution()).isEqualTo(NormativeResolution.DONE);
        assertThat(event.externalRef()).isEqualTo("PROJ-1234");
        assertThat(event.trackerType()).isEqualTo("jira");
        assertThat(event.actor()).isEqualTo("Alice");
    }

    @Test
    void resolvedWontDo_parsesDecline() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-resolved-wont-do.json"));

        assertThat(event.normativeResolution()).isEqualTo(NormativeResolution.DECLINE);
    }

    @Test
    void resolvedCannotReproduce_parsesFailure() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-resolved-cannot-reproduce.json"));

        assertThat(event.normativeResolution()).isEqualTo(NormativeResolution.FAILURE);
    }

    @Test
    void assigned_parsesAssigned() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-assigned.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.ASSIGNED);
        assertThat(event.newAssignee()).isEqualTo("abc123"); // accountId
        assertThat(event.actor()).isEqualTo("Bob");
    }

    @Test
    void unassigned_parsesUnassigned() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-unassigned.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.UNASSIGNED);
    }

    @Test
    void descriptionChanged_parsesDescriptionChanged() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-updated-description.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.DESCRIPTION_CHANGED);
        assertThat(event.newDescription()).isEqualTo("Updated description text.");
    }

    @Test
    void priorityChanged_highest_parsesUrgent() throws IOException {
        final WebhookEvent event = parser.parse(Map.of(), fixture("issue-updated-priority.json"));

        assertThat(event.eventKind()).isEqualTo(WebhookEventKind.PRIORITY_CHANGED);
        assertThat(event.newPriority()).isEqualTo(WorkItemPriority.URGENT);
    }
}
```

- [ ] **Step 4: Implement JiraWebhookParser**

```java
package io.casehub.work.issuetracker.jira;

import java.util.Map;
import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.work.api.NormativeResolution;
import io.casehub.work.issuetracker.spi.IssueTrackerException;
import io.casehub.work.issuetracker.webhook.WebhookEvent;
import io.casehub.work.issuetracker.webhook.WebhookEventKind;
import io.casehub.work.runtime.model.WorkItemPriority;

/**
 * Parses raw Jira webhook payloads ({@code jira:issue_updated}) into normalised
 * {@link WebhookEvent} records.
 *
 * <p>Jira sends a single {@code jira:issue_updated} event for all field changes.
 * The {@code changelog.items} array identifies what changed. This parser inspects
 * the changelog and returns the first handled event kind found (resolution >
 * assignee > priority > description > summary). If no handled change is found,
 * returns {@code null}.
 */
@ApplicationScoped
public class JiraWebhookParser {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Set<String> DECLINE_RESOLUTIONS = Set.of(
            "Won't Do", "Won't Fix", "Duplicate");
    private static final Set<String> FAILURE_RESOLUTIONS = Set.of(
            "Cannot Reproduce", "Incomplete");

    /**
     * Parse a raw Jira webhook payload.
     *
     * @return the event, or {@code null} if no handled change is present
     */
    public WebhookEvent parse(final Map<String, String> headers, final String body) {
        try {
            final JsonNode root = MAPPER.readTree(body);
            final String webhookEvent = root.path("webhookEvent").asText("");
            if (!"jira:issue_updated".equals(webhookEvent)) return null;

            final String issueKey = root.path("issue").path("key").asText(null);
            if (issueKey == null) return null;

            final String actor = root.path("user").path("displayName").asText("unknown");
            final JsonNode items = root.path("changelog").path("items");

            for (final JsonNode item : items) {
                final String field = item.path("field").asText("");
                final WebhookEvent event = parseChangelogItem(field, item, issueKey, actor, root);
                if (event != null) return event;
            }
            return null;

        } catch (final Exception e) {
            throw new IssueTrackerException("Failed to parse Jira webhook payload: " + e.getMessage(), e);
        }
    }

    private WebhookEvent parseChangelogItem(
            final String field, final JsonNode item, final String issueKey,
            final String actor, final JsonNode root) {

        return switch (field) {
            case "resolution" -> parseResolution(item, issueKey, actor, root);
            case "assignee" -> parseAssignee(item, issueKey, actor);
            case "priority" -> parsePriority(item, issueKey, actor, root);
            case "description" -> new WebhookEvent("jira", issueKey, WebhookEventKind.DESCRIPTION_CHANGED,
                    actor, null, null, null, null,
                    root.path("issue").path("fields").path("description").asText(null), null);
            case "summary" -> new WebhookEvent("jira", issueKey, WebhookEventKind.TITLE_CHANGED,
                    actor, null, null, null,
                    root.path("issue").path("fields").path("summary").asText(null), null, null);
            default -> null;
        };
    }

    private WebhookEvent parseResolution(
            final JsonNode item, final String issueKey, final String actor, final JsonNode root) {
        final String resolution = item.path("toString").asText(null);
        if (resolution == null) return null; // resolution cleared, not set — ignore
        final NormativeResolution normative = toNormativeResolution(resolution);
        return new WebhookEvent("jira", issueKey, WebhookEventKind.CLOSED,
                actor, normative, null, null, null, null, null);
    }

    private WebhookEvent parseAssignee(
            final JsonNode item, final String issueKey, final String actor) {
        final String toAccountId = item.path("to").asText(null);
        if (toAccountId == null || toAccountId.isBlank()) {
            return new WebhookEvent("jira", issueKey, WebhookEventKind.UNASSIGNED,
                    actor, null, null, null, null, null, null);
        }
        return new WebhookEvent("jira", issueKey, WebhookEventKind.ASSIGNED,
                actor, null, null, null, null, null, toAccountId);
    }

    private WebhookEvent parsePriority(
            final JsonNode item, final String issueKey, final String actor, final JsonNode root) {
        final String priorityName = root.path("issue").path("fields").path("priority").path("name").asText("");
        final WorkItemPriority priority = toWorkItemPriority(priorityName);
        if (priority == null) return null;
        return new WebhookEvent("jira", issueKey, WebhookEventKind.PRIORITY_CHANGED,
                actor, null, priority, null, null, null, null);
    }

    private NormativeResolution toNormativeResolution(final String resolution) {
        if (DECLINE_RESOLUTIONS.contains(resolution)) return NormativeResolution.DECLINE;
        if (FAILURE_RESOLUTIONS.contains(resolution)) return NormativeResolution.FAILURE;
        return NormativeResolution.DONE;
    }

    private WorkItemPriority toWorkItemPriority(final String jiraPriority) {
        return switch (jiraPriority) {
            case "Highest" -> WorkItemPriority.URGENT;
            case "High" -> WorkItemPriority.HIGH;
            case "Medium" -> WorkItemPriority.MEDIUM;
            case "Low", "Lowest" -> WorkItemPriority.LOW;
            default -> null;
        };
    }
}
```

- [ ] **Step 5: Run parser tests**

```bash
scripts/mvn-test casehub-work-issue-tracker -Dtest=JiraWebhookParserTest
```

Expected: all 7 tests pass.

- [ ] **Step 6: Commit**

```bash
git add \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/jira/JiraIssueTrackerConfig.java \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/jira/JiraWebhookParser.java \
  casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/jira/JiraWebhookParserTest.java \
  casehub-work-issue-tracker/src/test/resources/fixtures/jira/
git commit -m "feat(issue-tracker): Jira webhook parser + config + fixtures (#156)

Parses jira:issue_updated payloads via changelog inspection. Handles:
resolution (Done→DONE, Won't Do/Duplicate→DECLINE, Cannot Reproduce→FAILURE),
assignee set/cleared, priority (Highest→URGENT, High→HIGH, Medium→MEDIUM,
Low/Lowest→LOW), description, summary. First matching changelog item wins.
JiraIssueTrackerConfig at casehub.work.issue-tracker.jira.*.
7 parser unit tests with real Jira payload fixtures.

Refs #156"
```

---

## Task 9: JiraWebhookResource + integration test (Refs #156)

**Files:**
- Create: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/jira/JiraWebhookResource.java`
- Create: `casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/jira/JiraWebhookResourceTest.java`

- [ ] **Step 1: Write failing integration test**

```java
package io.casehub.work.issuetracker.jira;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import java.nio.charset.StandardCharsets;

import static io.restassured.RestAssured.given;

@QuarkusTest
class JiraWebhookResourceTest {

    private static final String SECRET = "test-jira-secret";
    private static final String ENDPOINT = "/workitems/jira-webhook";

    private String fixture(final String name) throws Exception {
        try (var stream = getClass().getResourceAsStream("/fixtures/jira/" + name)) {
            return new String(stream.readAllBytes(), StandardCharsets.UTF_8);
        }
    }

    @Test
    void validSecret_returns200() throws Exception {
        given()
            .header("Content-Type", "application/json")
            .queryParam("secret", SECRET)
            .body(fixture("issue-resolved-done.json"))
        .when()
            .post(ENDPOINT)
        .then()
            .statusCode(200);
    }

    @Test
    void wrongSecret_returns401() throws Exception {
        given()
            .header("Content-Type", "application/json")
            .queryParam("secret", "wrong-secret")
            .body(fixture("issue-resolved-done.json"))
        .when()
            .post(ENDPOINT)
        .then()
            .statusCode(401);
    }

    @Test
    void missingSecret_returns401() throws Exception {
        given()
            .header("Content-Type", "application/json")
            .body(fixture("issue-resolved-done.json"))
        .when()
            .post(ENDPOINT)
        .then()
            .statusCode(401);
    }

    @Test
    void unhandledEvent_returns200() throws Exception {
        final String body = "{\"webhookEvent\":\"jira:issue_created\",\"issue\":{\"key\":\"PROJ-1\"}}";
        given()
            .header("Content-Type", "application/json")
            .queryParam("secret", SECRET)
            .body(body)
        .when()
            .post(ENDPOINT)
        .then()
            .statusCode(200);
    }
}
```

Add to `src/test/resources/application.properties`:
```properties
casehub.work.issue-tracker.jira.webhook-secret=test-jira-secret
```

- [ ] **Step 2: Implement JiraWebhookResource**

```java
package io.casehub.work.issuetracker.jira;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.Map;

import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import org.jboss.logging.Logger;

import io.casehub.work.issuetracker.webhook.WebhookEvent;
import io.casehub.work.issuetracker.webhook.WebhookEventHandler;

/**
 * Receives inbound Jira webhook events.
 *
 * <p>Jira Cloud does not sign payloads with HMAC. Verification is done via a
 * shared secret passed as a query parameter. Configure your Jira webhook URL as:
 * {@code https://yourapp.example.com/workitems/jira-webhook?secret=<your-secret>}
 *
 * <p>Returns 200 for all valid requests (including unhandled events) to prevent
 * Jira retry storms. Returns 401 on secret mismatch.
 */
@Path("/workitems/jira-webhook")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class JiraWebhookResource {

    private static final Logger LOG = Logger.getLogger(JiraWebhookResource.class);

    @Inject
    JiraIssueTrackerConfig config;

    @Inject
    JiraWebhookParser parser;

    @Inject
    WebhookEventHandler handler;

    @POST
    public Response receive(
            @QueryParam("secret") final String secret,
            final String body) {

        final String configuredSecret = config.webhookSecret().orElse(null);
        if (configuredSecret == null) {
            LOG.warn("Jira webhook received but casehub.work.issue-tracker.jira.webhook-secret is not set — rejecting");
            return Response.status(Response.Status.UNAUTHORIZED).build();
        }

        if (!verifySecret(configuredSecret, secret)) {
            LOG.warn("Jira webhook secret mismatch — rejecting");
            return Response.status(Response.Status.UNAUTHORIZED).build();
        }

        try {
            final WebhookEvent event = parser.parse(Map.of(), body);
            if (event != null) {
                handler.handle(event);
            }
        } catch (final Exception e) {
            LOG.warnf("Jira webhook processing error (returning 200 to prevent retry): %s", e.getMessage());
        }

        return Response.ok().build();
    }

    private boolean verifySecret(final String expected, final String provided) {
        if (provided == null || provided.isBlank()) return false;
        return MessageDigest.isEqual(
                expected.getBytes(StandardCharsets.UTF_8),
                provided.getBytes(StandardCharsets.UTF_8));
    }
}
```

- [ ] **Step 3: Run integration tests**

```bash
scripts/mvn-test casehub-work-issue-tracker -Dtest=JiraWebhookResourceTest
```

Expected: all 4 tests pass.

- [ ] **Step 4: Run full module test suite**

```bash
scripts/mvn-test casehub-work-issue-tracker
```

Expected: all tests pass (~35–40 total including existing tests).

- [ ] **Step 5: Install to local Maven repo**

```bash
scripts/mvn-install casehub-work-issue-tracker
```

- [ ] **Step 6: Final commit**

```bash
git add \
  casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/jira/JiraWebhookResource.java \
  casehub-work-issue-tracker/src/test/java/io/casehub/work/issuetracker/jira/JiraWebhookResourceTest.java \
  casehub-work-issue-tracker/src/test/resources/application.properties
git commit -m "feat(issue-tracker): JiraWebhookResource — POST /workitems/jira-webhook (#156)

Query-param secret verification (constant-time compare). Jira Cloud does not
sign payloads with HMAC — configure webhook URL with ?secret=<value>.
Fail-closed: 401 when jira.webhook-secret not configured.
4 @QuarkusTest integration tests.

Closes #156 (Phase 2)
Refs #159"
```

---

## Self-Review

**Spec coverage check:**
- ✅ NormativeResolution DONE/DECLINE/FAILURE — Task 3
- ✅ WebhookEvent + WebhookEventKind — Task 4
- ✅ IssueTrackerProvider.parseWebhookEvent default — Task 4
- ✅ WorkItemIssueLink.findByTrackerRef — Task 4
- ✅ WebhookEventHandler with all 8 event kinds — Task 5
- ✅ Terminal WorkItem skip — Task 5
- ✅ Transition failure swallowed — Task 5
- ✅ Footer stripping on DESCRIPTION_CHANGED — Task 5
- ✅ Managed label filtering on inbound — Task 6 (parser)
- ✅ GitHub HMAC-SHA256 verification — Task 7
- ✅ GitHub fail-closed (no secret → 401) — Task 7
- ✅ GitHub fixture payloads for all event types — Task 6
- ✅ Jira query-param secret verification — Task 9
- ✅ Jira fail-closed — Task 9
- ✅ Jira DONE/DECLINE/FAILURE resolution mapping — Task 8
- ✅ Jira Highest→URGENT, Low/Lowest→LOW — Task 8
- ✅ Priority rename prerequisite — Tasks 1–2
- ✅ Flyway V5001 — Task 2
- ✅ ~35–40 tests across unit + integration tiers — Tasks 5–9
