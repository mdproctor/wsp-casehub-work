# API Surface Cleanup — Event Hierarchy, SPI Migration, Template Consolidation

**Issues:** #276, #277, #278 (single branch)
**Order:** #278 → #276 → #277
**Cross-repo:** engine#579 (source() → workItem() migration)

---

## #278 — Event Hierarchy Alignment

### Problem

Three competing event abstractions coexist:

1. `WorkLifecycleEvent` — abstract class in api/: `eventType()`, `context(): Map`, `source(): Object`
2. `WorkItemEvent` — interface in api/: `ref(): WorkItemRef` with typed defaults
3. `WorkItemLifecycleEvent` — final class in runtime/: extends #1, implements #2

`source()` returns `Object`, forcing all 5 callers to downcast to `WorkItem`. Every caller is in a module that already depends on runtime — the `Object` return buys nothing. `NotificationPayload` references `WorkLifecycleEvent` in api/ solely to have a non-runtime event type to hold.

### Design

**Enrich `WorkItemEvent`** (api/) with event metadata:

```java
public interface WorkItemEvent {
    WorkItemRef ref();
    WorkEventType eventType();
    Instant occurredAt();
    String actor();
    String detail();

    // existing default methods from ref()
    default UUID workItemId() { return ref().id(); }
    default WorkItemStatus status() { return ref().status(); }
    default String callerRef() { return ref().callerRef(); }
    default String assigneeId() { return ref().assigneeId(); }
    default String resolution() { return ref().resolution(); }
    default String candidateGroups() { return ref().candidateGroups(); }
    default String outcome() { return ref().outcome(); }
    default String tenancyId() { return ref().tenancyId(); }
}
```

**Delete `WorkLifecycleEvent`** abstract class. Method disposition:

| Method | Destination |
|--------|-------------|
| `eventType()` | Promoted to `WorkItemEvent` (abstract) |
| `context()` | Runtime-only on `WorkItemLifecycleEvent` (not on any interface) |
| `source()` | Replaced by typed `workItem(): WorkItem` on `WorkItemLifecycleEvent` |

**Update `WorkItemLifecycleEvent`** (runtime/):
- Remove `extends WorkLifecycleEvent`
- Keep `implements WorkItemEvent`
- Replace `source(): Object` with `workItem(): WorkItem` — returns null for wire-reconstructed events (via `fromWire()` factory); non-null for local events (via `of()` factories)
- `context()` stays as concrete method — throws or returns empty map when `workItem` is null (wire events)

**Update `NotificationPayload`** (api/):
- Field type: `WorkLifecycleEvent event` → `WorkItemEvent event`

**Observer migration:**

| Observer | Module | Change |
|----------|--------|--------|
| `FilterRegistryEngine` | runtime/ | `@Observes WorkLifecycleEvent` → `@Observes WorkItemLifecycleEvent` |
| `EscalationSummaryObserver` | ai/ | Same — observe concrete type, use `event.workItem()` |

**Notification channels** (Slack, Teams, HTTP):
- Currently: `(WorkItem) payload.event().source()`
- After: `((WorkItemLifecycleEvent) payload.event()).workItem()`
- All in notifications/ module (depends on runtime) — cast is legitimate

**FilterAction SPI:** stays `apply(Object workUnit, ...)`. Callers pass `event.workItem()` instead of `event.source()`. Runtime-internal, not in api/spi/.

**`WorkItemGroupLifecycleEvent`:** stays as-is. Tracked in #279 for future evaluation.

**Cross-repo:** engine#579 — `WorkItemLifecycleAdapter` migrates `event.source()` → `event.workItem()`.

### Test impact

- `WorkEventTypeTest`: anonymous `WorkLifecycleEvent` → anonymous `WorkItemEvent` implementation
- `WorkItemLifecycleEventTest`: update assertions for `workItem()` instead of `source()`
- `FilterRegistryEngineTest`: anonymous `WorkLifecycleEvent` → supply concrete `WorkItemLifecycleEvent`
- `WorkItemLifecycleEventOutcomeTest`: update if it uses `source()`

---

## #276 — SPI Migration to api/spi/

### Problem

14 consumer-facing SPI interfaces sit in `io.casehub.work.api` flat, mixed with value types, events, and exceptions. 2 SPIs from #275 are correctly in `api/spi/`. Inconsistency violates the `consumer-spi-placement` protocol.

### Design

Move 14 interfaces from `io.casehub.work.api` to `io.casehub.work.api.spi`:

1. BusinessCalendar
2. CapabilityRegistry
3. ClaimSlaPolicy
4. ExclusionPolicy
5. HolidayCalendar
6. InstanceAssignmentStrategy
7. NotificationChannel
8. SkillMatcher
9. SkillProfileProvider
10. SlaBreachPolicy
11. SpawnPort
12. WorkerRegistry
13. WorkerSelectionStrategy
14. WorkloadProvider

**Method:** IntelliJ `ide_move_file` for each — updates all imports across all modules.

**What stays in `io.casehub.work.api`:** value types (19 records/enums), events (WorkItemEvent, WorkItemGroupLifecycleEvent), exceptions (2), utilities (WorkCapabilities, WorkItemCallerRef, WorkItemCreateRequest).

---

## #277 — Template Consolidation

### Problem

Two template creation paths: `instantiate()` (3 positional-arg overloads, template-first) and `createFromTemplate(WorkItemCreateRequest)` (request-wins merge). Both call `workItemService.create()`. The REST endpoint uses the old path.

### Design

**Delete all `instantiate()` overloads** (3 methods) and **all `toCreateRequest()` static methods** (3 methods).

**Caller migration:**

| Caller | Module | Migration |
|--------|--------|-----------|
| `WorkItemTemplateResource.instantiate()` | runtime/ | Build `WorkItemCreateRequest` with templateId + overrides → `createFromTemplate()` |
| `WorkItemScheduleService.fireSchedule()` | runtime/ | Build request with templateId + createdBy → `createFromTemplate()` |
| `FormSchemaScenario` (2 calls) | examples/ | Build requests with templateId + overrides |
| ~15 test classes | runtime/test | Build `WorkItemCreateRequest` instead of positional args |

**Fix audit gap:** `createFromTemplate()` must add `auditDetail` for excludedGroups expansion (currently only in `instantiate()`). Add the `buildExpansionAuditDetail()` call after `mergeRequestWithTemplate()` when expanded users differ from the request's excludedUsers.

**Behavioral equivalence:** `mergeRequestWithTemplate()` (request-wins) is a superset of `toCreateRequest()` (template-first). For the fields callers currently override (title, assigneeId, createdBy, callerRef, payload), both produce identical results. Verified by analysis of both merge functions.

---

## Platform Coherence Review

| Check | Result |
|-------|--------|
| consumer-spi-placement | Aligned — all consumer-facing SPIs move to api/spi/ |
| spi-evolution-default-methods | N/A — new methods on `WorkItemEvent` are abstract (required, not optional) |
| work-event-type-enum-coverage | No new event types added |
| lifecycle-enum-classification-on-enum | No changes to lifecycle enums |
| bridge-module-spi-placement | N/A — no bridge SPIs affected |
| domain-event-not-workitem-lifecycle-for-triggers | N/A — observer changes preserve existing patterns |
| no-workarounds-fix-the-design | No wrappers or shims — `WorkLifecycleEvent` deleted, `source()` replaced |

## Out-of-scope (tracked)

- engine#579: migrate `WorkItemLifecycleAdapter` from `source()` to `workItem()`
- #279: evaluate `WorkItemGroupLifecycleEvent` hierarchy integration
