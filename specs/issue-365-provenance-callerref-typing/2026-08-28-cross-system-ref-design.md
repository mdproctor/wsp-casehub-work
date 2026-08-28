# CrossSystemRef + Ledger Entry Back-Reference — casehubio/work#365, #366

**Date:** 2026-08-28
**Issues:** casehubio/work#365, casehubio/work#366
**Status:** Draft

---

## Problem

Two provenance gaps in the work→engine boundary:

1. **No inbound causedByEntryId (#365).** When `WorkItemLifecycleAdapter` observes a terminal WorkItem event and calls `PlanItemCompletionApplier`, no work-ledger entry ID flows back to the engine. The causal chain breaks: `engine SPAWN → work CREATED → work COMPLETED → engine [no back-reference]`.

2. **String-convention callerRef (#366).** Three independent parsing systems exist for the same concept — `CallerRef` (engine-adapter sealed interface), `QhorusCallerRef` (qhorus record), and `CallerRefParser` (engine-side duplicate). Each new integration module requires a new string format and parser. Cross-boundary graph traversal requires prefix-matching to know which system to query next.

These are tightly coupled: a typed `CrossSystemRef` with a `ledgerEntryId` field solves both — the type system replaces string parsing, and the ledger entry ID provides the back-reference.

---

## Design

### CrossSystemRef sealed interface (work-api)

```java
package io.casehub.work.api;

public sealed interface CrossSystemRef
    permits PlanItemRef, GateRef, QhorusRef, GenericRef {

  String system();
  String entityType();
  @Nullable UUID ledgerEntryId();

  String encode();

  static CrossSystemRef decode(String callerRef) { ... }
}
```

Four concrete types:

| Type | System | Format | Domain fields |
|------|--------|--------|---------------|
| `PlanItemRef` | `"engine"` | `case:{caseId}/pi:{planItemId}` | `UUID caseId()`, `String planItemId()` |
| `GateRef` | `"engine"` | `case:{caseId}/gate:{gateId}` | `UUID caseId()`, `long gateId()` |
| `QhorusRef` | `"qhorus"` | `qhorus:{channelId}/{messageId}/{correlationId}` | `UUID channelId()`, `long messageId()`, `String correlationId()` |
| `GenericRef` | varies | `{system}:{body}` | `String body()` |

`GenericRef` handles unknown prefixes — future integration modules (e.g. `flow:`) work without modifying the sealed hierarchy, though adding a typed variant is preferred.

### decode() backward compatibility

`CrossSystemRef.decode(String)` handles all existing callerRef format strings:

- `"case:xxx/pi:yyy"` → `PlanItemRef`
- `"case:xxx/gate:nnn"` → `GateRef`
- `"qhorus:xxx/nnn/zzz"` → `QhorusRef`
- `"other:..."` → `GenericRef`
- `null` or blank → `null`

`encode()` on each type produces the same string as today. No database migration — the TEXT column stores the same values.

### ledgerEntryId field

`UUID ledgerEntryId()` is nullable — the source system's ledger entry ID that caused this WorkItem to exist. Set at creation time when the source system provides it. Not all callerRefs will have a ledger entry ID (e.g. programmatic WorkItem creation without a ledger).

The field is narrowly scoped to ledger entries. Reasoning traces (engine#1007, stored in CaseMemoryStore) and EventLog entries flow through separate data paths and do not belong on CrossSystemRef.

### Ledger entry ID threading

The work-ledger entry ID reaches `WorkItemLifecycleAdapter` via a mutable field on `WorkItemLifecycleEvent`:

```
WorkItemService fires WorkItemLifecycleEvent
  ├─ @Observes: LedgerEventCapture
  │    ├─ writes WorkItemLedgerEntry (gets entry.id)
  │    └─ event.setLedgerEntryId(entry.id)    ← NEW
  │
  └─ @ObservesAsync: WorkItemLifecycleAdapter  (runs AFTER @Observes completes)
       ├─ reads event.ledgerEntryId()           ← NEW
       ├─ CrossSystemRef.decode(event.callerRef())
       └─ applier.apply(caseId, planItemId, status, ref, ledgerEntryId)
```

CDI guarantees: `@Observes` (synchronous) completes before `@ObservesAsync` fires. The field is guaranteed populated when the adapter reads it.

### Changes by module

#### work-api

- **New:** `CrossSystemRef` sealed interface + `PlanItemRef`, `GateRef`, `QhorusRef`, `GenericRef` records
- **Modified:** `WorkItemLifecycleEvent` — add `UUID ledgerEntryId` field with setter and getter
- **Modified:** `WorkItemEvent` — add `default UUID ledgerEntryId() { return null; }` to interface

#### engine-adapter

- **Deleted:** `CallerRef.java` (sealed interface), `PlanItemCallerRef.java`, `GateCallerRef.java`
- **Modified:** `WorkItemLifecycleAdapter` — switch from `CallerRef.parse()` to `CrossSystemRef.decode()`, pass `ledgerEntryId` to `PlanItemCompletionApplier`
- **Modified:** `PlanItemCompletionApplier.apply()` — add `UUID workLedgerEntryId` parameter (passed through to engine for `causedByEntryId` wiring)
- **Modified:** `HumanTaskScheduleHandler` — use `PlanItemRef.encode()` instead of `PlanItemCallerRef.encode()`
- **Modified:** `ActionGateWorkItemHandler` — use `GateRef.encode()` instead of `GateCallerRef.encode()`

#### qhorus

- **Deleted:** `QhorusCallerRef.java`
- **Modified:** `QhorusWorkItemLifecycleAdapter` — switch from `QhorusCallerRef.parse/isQhorus` to `CrossSystemRef.decode()` and `instanceof QhorusRef` check
- **Modified:** MCP tools (`request_human_work`) — use `QhorusRef.encode()` instead of `QhorusCallerRef.encode()`

#### ledger

- **Modified:** `LedgerEventCapture.onWorkItemEvent()` — after `ledgerRepo.save(entry)`, set `event.setLedgerEntryId(entry.id)` on the observed event

### What does NOT change

- **Database schema** — `callerRef` column stays TEXT, encoded strings are identical
- **REST API** — callerRef is returned as a string in API responses (unchanged)
- **Engine-side `CallerRefParser`** — stays as-is in `casehub-engine`. Follow-on issue to migrate to `CrossSystemRef` dependency or delete the duplicate.
- **Provenance supplement in LedgerEventCapture** — still writes `sourceEntityId = callerRef` string on CREATED entries (unchanged)

### PlanItemCompletionApplier — what the engine does with ledgerEntryId

`PlanItemCompletionApplier.apply()` gains a `UUID workLedgerEntryId` parameter. The work-side change stops here — the parameter is accepted and available for use. How the engine consumes it (adding a field to `PlanItemStateChangedEvent`, writing it to the engine's own ledger entry as `causedByEntryId`) is engine#205 scope. `PlanItemStateChangedEvent` is an engine-common record — modifying it requires a change in the engine repo, not here.

For now, `PlanItemCompletionApplier` logs the value at DEBUG level so the threading is visible and testable without modifying engine types.

---

## Tests

### work-api

1. `CrossSystemRefTest` — `decode()` round-trips for all four formats
2. `CrossSystemRefTest` — `decode()` handles null, blank, malformed input
3. `CrossSystemRefTest` — `encode()` produces backward-compatible strings matching existing format
4. `CrossSystemRefTest` — `decode()` unknown prefix produces `GenericRef`

### engine-adapter

5. `WorkItemLifecycleAdapterTest` — terminal event with `PlanItemRef` callerRef triggers `applier.apply()` with correct `ledgerEntryId`
6. `WorkItemLifecycleAdapterTest` — terminal event with `GateRef` callerRef routes to `gateApplier`
7. `WorkItemLifecycleAdapterTest` — null callerRef returns early (no NPE)
8. `PlanItemCompletionApplierTest` — `workLedgerEntryId` parameter accepted and logged at DEBUG

### qhorus

9. `QhorusWorkItemLifecycleAdapterTest` — terminal event with `QhorusRef` callerRef dispatches to channel
10. `QhorusWorkItemLifecycleAdapterTest` — non-qhorus callerRef is ignored

### ledger

11. `LedgerEventCaptureTest` — after ledger entry written, `event.ledgerEntryId()` is non-null and matches the persisted entry ID

---

## Out of scope

- Engine-side `CallerRefParser` migration — follow-on issue in casehubio/engine
- Engine-side wiring of `causedByEntryId` from the received `workLedgerEntryId` — engine#205 scope
- Reasoning traces as lineage DAG participants — engine#1007
- Platform-wide PROV-DM export — parent#431
- `WorkItemGroupLifecycleEvent` ledger entry threading — would follow the same pattern but is not in scope for these two issues

---

## References

- `engine-adapter/src/main/java/io/casehub/work/engine/CallerRef.java` — existing sealed interface (replaced)
- `engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCallerRef.java` — existing encode/parse (replaced)
- `engine-adapter/src/main/java/io/casehub/work/engine/GateCallerRef.java` — existing encode/parse (replaced)
- `qhorus/src/main/java/io/casehub/work/qhorus/QhorusCallerRef.java` — existing encode/parse (replaced)
- `engine/common/src/main/java/io/casehub/engine/common/spi/CallerRefParser.java` — engine-side duplicate (not changed)
- `api/src/main/java/io/casehub/work/api/WorkItemLifecycleEvent.java` — event class (modified)
- `engine-adapter/src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java` — inbound adapter (modified)
- `engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java` — completion path (modified)
- `ledger/src/main/java/io/casehub/work/ledger/service/LedgerEventCapture.java` — ledger write (modified)
- `docs/specs/2026-05-15-callerref-multi-instance-design.md` — prior callerRef design for multi-instance groups
- Slot 140 handoff — reasoning traces landing (validates "add to event, thread through pipeline" pattern)
- casehubio/engine#205 — complete causal lineage (consumes the ledgerEntryId we thread)
- casehubio/engine#1007 — reasoning traces as lineage participants (separate concern, not callerRef)
- casehubio/parent#363 — explainable decisions end-to-end (consumes cross-system lineage)
- casehubio/parent#431 — platform-wide PROV-DM export (consumes cross-system lineage)
