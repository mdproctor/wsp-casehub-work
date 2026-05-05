# casehub-work — Session Handover
**Date:** 2026-05-05

## Project Status

Build passing locally. Working tree clean. 93 tests in `casehub-work-issue-tracker`, 31 in `testing`.

## What Was Done This Session

### GitHub + Jira inbound webhooks — #156 Phase 2 (complete)

Full bidirectional sync now in place. Design involved:
- `NormativeResolution` enum (DONE/DECLINE/FAILURE) — speech-act grounded close mapping, grounded in Qhorus normative layer
- `WorkItemPriority` renamed CRITICAL→URGENT, NORMAL→MEDIUM (Linear alignment, #160) — V5001 Flyway migration
- Full inbound event mapping for both GitHub and Jira: close, assign/unassign, title, description, priority, labels
- GitHub/Jira parity principle established: always design for both simultaneously
- `WebhookEvent` record + `WebhookEventKind` enum, `GitHubWebhookParser`, `JiraWebhookParser`
- `GitHubWebhookResource` (HMAC-SHA256), `JiraWebhookResource` (query-param secret)
- Both fail-closed: 401 when secret not configured

### IssueLinkStore SPI — #161 (complete)

- `IssueLinkStore` interface + `JpaIssueLinkStore` (thin Panache adapter)
- `InMemoryIssueLinkStore` in `testing/` (`@Alternative @Priority(1)`)
- `WebhookEventHandler` now injects `IssueLinkStore` + `WorkItemStore` — public `handle(WebhookEvent)` fully unit-testable
- `IssueLinkService` injects `IssueLinkStore` — all Panache statics replaced
- `IssueLinkService.onLifecycleEvent` now `@Observes(during = AFTER_SUCCESS)`
- 34 new tests across modules

### Open issues created

- #159 — Normative alignment docs (WorkItem lifecycle → Qhorus speech acts)
- #160 — Priority rename (closed)
- #161 — IssueLinkStore SPI (closed)
- #162 (implied) — Comprehensive GitHub integration doc (comment on #156)

## Open / Next

| Priority | What |
|---|---|
| 1 | Create PR from fork to `casehubio/work` for all session changes |
| 2 | #156 Phase 3 — identity mapping (assigneeId ↔ GitHub login / Jira accountId) |
| 3 | #159 — normative alignment docs (low-effort docs task) |
| 4 | #97 — `casehub-work-qhorus` (blocked on qhorus#131 + #132) |

## Key References

- Spec: `docs/superpowers/specs/2026-05-04-github-jira-webhooks-phase2-design.md`
- Spec: `docs/superpowers/specs/2026-05-05-issue-link-store-spi-design.md`
- `IssueLinkStore` SPI: `casehub-work-issue-tracker/src/main/java/io/casehub/work/issuetracker/repository/IssueLinkStore.java`
- Blog: `blog/2026-05-05-mdp01-speech-acts-and-priority.md`
- Previous context: `git show HEAD~20:HANDOFF.md`
