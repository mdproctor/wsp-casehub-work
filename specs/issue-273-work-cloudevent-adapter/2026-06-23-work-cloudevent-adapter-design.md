# WorkCloudEventAdapter — Design Spec

**Issue:** casehubio/work#273
**Date:** 2026-06-23

## Problem

casehub-ras needs to observe WorkItem lifecycle events for situational awareness (e.g. "3 SLA breaches in 10 minutes across the same case type") without coupling to casehub-work types. The platform-standard decoupling mechanism is CloudEvents — every other foundation module (IoT, Qhorus, Connectors) already has a CloudEvent adapter. casehub-work does not.

A prerequisite defect blocks a clean adapter: `WorkItemLifecycleEvent` is fired with a mix of `fire()` (sync) and `fireAsync()` (async) across different call sites. CDI treats these as independent channels — `fire()` delivers to `@Observes` only, `fireAsync()` delivers to `@ObservesAsync` only. No single observer annotation can catch all events. This also causes three existing bugs where sync observers miss async-only events.

## Scope

Three changes, one issue:

1. **Normalize event firing** — every `WorkItemLifecycleEvent` and `WorkItemGroupLifecycleEvent` firing site calls both `fire()` and `fireAsync()`
2. **Add `WorkCloudEventAdapter`** — bridges both event types to `CloudEvent` via `@ObservesAsync`
3. **File parent evaluation issue** — evaluate dual-channel CDI firing as a platform-wide standard

## Change 1: Event Firing Normalization

### Root cause

casehub-work is the only repo with mixed fire/fireAsync. All other repos (qhorus, iot, connectors, platform-streams) use async-only consistently. casehub-work's mixed pattern is an evolution artifact, not an intentional design.

### Existing bugs this fixes

| Observer | Annotation | Currently misses | Impact |
|----------|-----------|-----------------|--------|
| `LocalWorkItemEventBroadcaster` | `@Observes` | COMPLETED, REJECTED (async-only) | SSE streams incomplete — clients never see terminal events |
| `WorkItemMetrics` | `@Observes` | COMPLETED, REJECTED (async-only) | Completion/rejection counters not incremented |
| `FilterEvaluationObserver` | `@Observes` | COMPLETED, REJECTED (async-only) | Queue membership not updated on completion |

### Sites to change

Add the missing channel at each site. Order: `fire()` first (sync observers in-transaction), then `fireAsync()` (async observers post-commit).

**WorkItemService** — add `fireAsync()` where only `fire()` exists:
- `create()` — CREATED
- `claim()` — ASSIGNED
- `start()` — STARTED
- `delegate()` — DELEGATED
- `acceptDelegation()` — DELEGATION_ACCEPTED

**WorkItemService** — add `fire()` where only `fireAsync()` exists:
- `complete()` (both variants with/without rationale) — COMPLETED
- `reject()` (both variants with/without rationale) — REJECTED
- `completeFromSystem()` — COMPLETED
- `rejectFromSystem()` — REJECTED
- All remaining async-only sites (lines 635–736)

**Other classes** — add `fireAsync()` where only `fire()` exists:
- `ExpiryLifecycleService.fireLifecycleEvent()` — expiry events
- `WorkItemSpawnService.spawn()` — SPAWNED
- `QueueStateResource.pickup()` — ASSIGNED

**WorkItemGroupLifecycleEvent** — add `fire()` where only `fireAsync()` exists:
- `MultiInstanceGroupPolicy` — group status events

### Why dual-channel (not async-only)

casehub-work has sync observers with legitimate transactional consistency requirements:
- `FilterEvaluationObserver` — queue membership updates must be atomic with the WorkItem mutation
- `LocalWorkItemEventBroadcaster` — ordered SSE delivery within transaction boundary
- `WorkItemMetrics` — counters incremented within transaction boundary

Migrating these to `@ObservesAsync` would change the consistency model. Dual-channel preserves existing transactional guarantees while enabling async observers (CloudEvent adapter, MultiInstanceCoordinator).

## Change 2: WorkCloudEventAdapter

### Location

`runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventAdapter.java`

In `runtime/` — consistent with QhorusCloudEventAdapter placement. No new module; `cloudevents-core` is already transitive via `casehub-platform-api`.

### Structure

Single `@ApplicationScoped` class. Two `@ObservesAsync` methods. Follows the canonical 6-rule pattern (GE-20260621-629712).

```
@ApplicationScoped
WorkCloudEventAdapter
  - Event<CloudEvent> cloudEventBus  (injected)
  - ObjectMapper objectMapper        (injected — Rule 1)
  + onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent)
  + onGroupLifecycle(@ObservesAsync WorkItemGroupLifecycleEvent)
```

### Canonical pattern compliance

| Rule | Implementation |
|------|---------------|
| ObjectMapper injected | Constructor injection — never static |
| tenancyId null-safe | Guard before `withExtension("tenancyid", ...)` |
| fireAsync `.exceptionally()` | `LOG.warnf(ex, ...)`, return null |
| Serialisation error | Catch `JsonProcessingException` at WARN, fire with `new byte[0]` |
| Type from stable semantic field | WorkItem: reuse `event.type()`. Group: derive from `groupStatus()` |
| WARN severity | All degraded paths |
| datacontenttype | `application/json` |

### CloudEvent field mapping

**WorkItemLifecycleEvent → CloudEvent:**

| CloudEvent field | Source |
|-----------------|--------|
| `id` | `UUID.randomUUID().toString()` |
| `type` | `event.type()` — already `io.casehub.work.workitem.<event>` |
| `source` | `URI.create(event.sourceUri())` — already `/workitems/{id}` |
| `subject` | `event.subject()` — WorkItem UUID string |
| `time` | `event.occurredAt().atOffset(ZoneOffset.UTC)` |
| `datacontenttype` | `application/json` |
| `data` | `objectMapper.writeValueAsBytes(event)` — scalar fields only (`workItem` is `@JsonIgnore`) |
| `tenancyid` ext | `event.tenancyId()` — null-safe |

**WorkItemGroupLifecycleEvent → CloudEvent:**

| CloudEvent field | Source |
|-----------------|--------|
| `id` | `UUID.randomUUID().toString()` |
| `type` | `"io.casehub.work.group." + event.groupStatus().name().toLowerCase(Locale.ROOT)` |
| `source` | `URI.create("/workitems/groups/" + event.groupId())` |
| `subject` | `event.groupId().toString()` |
| `time` | `event.occurredAt().atOffset(ZoneOffset.UTC)` |
| `datacontenttype` | `application/json` |
| `data` | `objectMapper.writeValueAsBytes(event)` — all fields serializable |
| `tenancyid` ext | `event.tenancyId()` — null-safe |

### Dependencies

No new Maven dependencies. `cloudevents-core` 4.0.1 is already on the classpath transitively via `casehub-platform-api` → `casehub-work-api` → `casehub-work` (runtime).

## Change 3: Platform evaluation issue

File on `casehubio/parent`: **"evaluate: dual-channel CDI event firing as platform standard"**

Content:
- casehub-work normalized to dual-channel (fire + fireAsync) in work#273 because it has sync observers needing transactional consistency
- All other repos currently fire async-only — this works because they have no sync observers
- The evaluation question for each repo: "does any current or foreseeable observer need to run inside the producer's transaction?"
- Specific patterns to look for: observers that update derived views (like queue membership), observers that aggregate state (like metrics), observers that need ordering guarantees (like SSE broadcasting)
- Firing both costs nothing when a channel has no observers — CDI no-ops the empty channel
- This is an evaluation, not an adoption mandate

## Testing

### Adapter unit tests (pure JUnit 5, no container)

- `onWorkItemLifecycle` builds correct CloudEvent fields from a `WorkItemLifecycleEvent`
- `onGroupLifecycle` builds correct CloudEvent fields from a `WorkItemGroupLifecycleEvent`
- tenancyId null → extension omitted (not serialised as literal `"null"`)
- tenancyId present → extension set
- Serialisation failure → WARN logged, CloudEvent fired with empty data
- CloudEvent type derivation: WorkItem uses `event.type()`, Group uses `groupStatus().name().toLowerCase()`

### Firing normalization tests

- Verify sync observer receives events that were previously async-only (COMPLETED, REJECTED)
- Verify async observer receives events that were previously sync-only (CREATED, ASSIGNED)

### Integration test

- End-to-end: create → complete a WorkItem, verify `@ObservesAsync CloudEvent` observer receives both events with correct type, source, subject, tenancyId

## Platform doc updates

- **PLATFORM.md** Capability Ownership table: add WorkItem → CloudEvent adapter row
- **ARC42STORIES.MD**: update with adapter delivery
