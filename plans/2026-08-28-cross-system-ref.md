# CrossSystemRef + Ledger Entry Back-Reference Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #365 — thread work-ledger entry ID back to engine on WorkItem completion
**Issue group:** #365, #366

**Goal:** Thread work-ledger entry IDs back to the engine on WorkItem completion, and introduce a CrossSystemRef interface for typed cross-system references.

**Architecture:** Two composable changes. Part 1 (#365): add a `ledgerEntryId` field to `WorkItemLifecycleEvent` set by `LedgerEventCapture` via a `LedgerEntryIdSetter` SPI, threaded through adapters to `PlanItemCompletionApplier` and `ActionGateCompletionApplier`. Part 2 (#366): introduce a minimal `CrossSystemRef` interface in work-api, make `CallerRef` extend it, rename `PlanItemCallerRef` → `PlanItemRef`, `GateCallerRef` → `GateRef`, `QhorusCallerRef` → `QhorusRef`.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (Jakarta), JPA/Hibernate

## Global Constraints

- Java 21 source level on Java 26 JVM
- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` for all builds
- Use `ide_refactor_rename` for type renames (never bash mv/sed)
- Use `ide_insert_member` / `ide_replace_member` for code edits
- Test with `mvn test -Dtest=ClassName -pl <module>` via scripts/
- All commits reference `Refs #365` or `Refs #366`

---

## Batch 1: Foundation — CrossSystemRef + ledgerEntryId event field

### Task 1: CrossSystemRef interface + LedgerEntryIdSetter SPI in work-api

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/CrossSystemRef.java`
- Create: `api/src/main/java/io/casehub/work/api/LedgerEntryIdSetter.java`
- Modify: `api/src/main/java/io/casehub/work/api/WorkItemLifecycleEvent.java`
- Modify: `api/src/main/java/io/casehub/work/api/WorkItemEvent.java`
- Test: `api/src/test/java/io/casehub/work/api/CrossSystemRefTest.java`
- Test: `api/src/test/java/io/casehub/work/api/WorkItemLifecycleEventTest.java` (existing — add tests)

**Interfaces:**
- Produces: `CrossSystemRef` interface with `String system()`, `String encode()`
- Produces: `LedgerEntryIdSetter` functional interface with `void set(WorkItemLifecycleEvent event, UUID ledgerEntryId)`
- Produces: `WorkItemLifecycleEvent.ledgerEntryIdSetter()` static accessor returning the `LedgerEntryIdSetter`
- Produces: `WorkItemEvent.ledgerEntryId()` default method returning `null`
- Produces: `WorkItemLifecycleEvent.ledgerEntryId()` getter returning the private field

- [ ] **Step 1: Write CrossSystemRef interface**

```java
package io.casehub.work.api;

public interface CrossSystemRef {
    String system();
    String encode();
}
```

- [ ] **Step 2: Write LedgerEntryIdSetter SPI**

```java
package io.casehub.work.api;

import java.util.UUID;

@FunctionalInterface
public interface LedgerEntryIdSetter {
    void set(WorkItemLifecycleEvent event, UUID ledgerEntryId);
}
```

- [ ] **Step 3: Add ledgerEntryId field and setter accessor to WorkItemLifecycleEvent**

Add to `WorkItemLifecycleEvent`:
- Private field: `private UUID ledgerEntryId;` (not final — set after construction by LedgerEventCapture)
- Getter: `public UUID ledgerEntryId() { return ledgerEntryId; }`
- Static accessor: `public static LedgerEntryIdSetter ledgerEntryIdSetter() { return (event, id) -> event.ledgerEntryId = id; }`
- Add `@Nullable UUID ledgerEntryId` parameter to `fromWire()` factory method — set the field from it

Use `ide_insert_member` to add the field, getter, and static method.

- [ ] **Step 4: Add default ledgerEntryId() to WorkItemEvent interface**

```java
default UUID ledgerEntryId() { return null; }
```

Use `ide_insert_member` on `WorkItemEvent.java`.

- [ ] **Step 5: Write tests**

In `CrossSystemRefTest` (new file):
- `crossSystemRef_interfaceContract` — verify a test implementation provides `system()` and `encode()`

In `WorkItemLifecycleEventTest` (existing):
- `ledgerEntryId_nullAfterConstruction` — verify `ledgerEntryId()` is null on a freshly created event
- `ledgerEntryId_setViaSetter_returnsValue` — verify `ledgerEntryIdSetter().set(event, id)` populates the getter
- `fromWire_withLedgerEntryId_preservesValue` — verify fromWire round-trips the ledgerEntryId

- [ ] **Step 6: Run tests**

Run: `scripts/test-module.sh api`
Expected: All new and existing tests pass

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/work/api/CrossSystemRef.java api/src/main/java/io/casehub/work/api/LedgerEntryIdSetter.java api/src/main/java/io/casehub/work/api/ api/src/test/
git commit -m "feat(api): add CrossSystemRef interface and ledgerEntryId field on WorkItemLifecycleEvent

Refs #365, Refs #366"
```

---

## Batch 2: Type renames — CallerRef extends CrossSystemRef

### Task 2: Rename PlanItemCallerRef → PlanItemRef, GateCallerRef → GateRef

**Files:**
- Rename: `PlanItemCallerRef` → `PlanItemRef` (use `ide_refactor_rename`)
- Rename: `GateCallerRef` → `GateRef` (use `ide_refactor_rename`)
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/CallerRef.java` — extend CrossSystemRef
- All referencing files auto-updated by IDE rename

**Interfaces:**
- Consumes: `CrossSystemRef` interface from Task 1
- Produces: `PlanItemRef` record (same as PlanItemCallerRef, renamed)
- Produces: `GateRef` record (same as GateCallerRef, renamed)
- Produces: `CallerRef extends CrossSystemRef` with `default String system() { return "engine"; }`

- [ ] **Step 1: Rename PlanItemCallerRef → PlanItemRef**

Use `ide_refactor_rename` on `engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCallerRef.java` line 28. This renames the class and updates all 64 references across source, test, and doc files.

- [ ] **Step 2: Rename GateCallerRef → GateRef**

Use `ide_refactor_rename` on `engine-adapter/src/main/java/io/casehub/work/engine/GateCallerRef.java` line 32. This renames the class and updates all 55 references.

- [ ] **Step 3: Make CallerRef extend CrossSystemRef**

Modify `CallerRef.java`:
```java
import io.casehub.work.api.CrossSystemRef;

public sealed interface CallerRef extends CrossSystemRef permits PlanItemRef, GateRef {
    UUID caseId();
    @Override default String system() { return "engine"; }
}
```

Add `encode()` implementations to `PlanItemRef` and `GateRef` (their existing `static encode()` methods become instance `encode()` overrides):

`PlanItemRef`:
```java
@Override public String encode() { return "case:" + caseId + "/pi:" + planItemId; }
```

`GateRef`:
```java
@Override public String encode() { return "case:" + caseId + "/gate:" + gateId; }
```

Keep the existing static `encode(UUID, String)` / `encode(UUID, long)` methods for backward compat with callers that construct the string without a record instance.

- [ ] **Step 4: Add CallerRef CrossSystemRef tests**

In `CallerRefTest` (existing), add:
- `planItemRef_implementsCrossSystemRef` — verify `system()` returns `"engine"` and `encode()` matches static encode
- `gateRef_implementsCrossSystemRef` — same for GateRef

- [ ] **Step 5: Run tests**

Run: `scripts/test-module.sh engine-adapter`
Expected: All existing + new tests pass. Renames are transparent.

- [ ] **Step 6: Commit**

```bash
git add engine-adapter/ api/ rest/
git commit -m "refactor(engine-adapter): rename PlanItemCallerRef/GateCallerRef, CallerRef extends CrossSystemRef

Refs #365, Refs #366"
```

### Task 3: Rename QhorusCallerRef → QhorusRef, implement CrossSystemRef

**Files:**
- Rename: `QhorusCallerRef` → `QhorusRef` (use `ide_refactor_rename`)
- All referencing files in qhorus/ auto-updated by IDE rename

**Interfaces:**
- Consumes: `CrossSystemRef` interface from Task 1
- Produces: `QhorusRef implements CrossSystemRef` with `system() = "qhorus"`

- [ ] **Step 1: Rename QhorusCallerRef → QhorusRef**

Use `ide_refactor_rename` on `qhorus/src/main/java/io/casehub/work/qhorus/QhorusCallerRef.java` line 5. Updates all 55 references.

- [ ] **Step 2: Make QhorusRef implement CrossSystemRef**

```java
import io.casehub.work.api.CrossSystemRef;

public record QhorusRef(UUID channelId, long messageId, String correlationId)
    implements CrossSystemRef {
    @Override public String system() { return "qhorus"; }
    // encode() already exists — satisfies CrossSystemRef.encode()
}
```

- [ ] **Step 3: Add QhorusRef CrossSystemRef test**

In `QhorusCallerRefTest` (now `QhorusRefTest` after rename), add:
- `qhorusRef_implementsCrossSystemRef` — verify `system()` returns `"qhorus"` and `encode()` matches existing format

- [ ] **Step 4: Run tests**

Run: `scripts/test-module.sh qhorus`
Expected: All existing + new tests pass.

- [ ] **Step 5: Commit**

```bash
git add qhorus/
git commit -m "refactor(qhorus): rename QhorusCallerRef → QhorusRef, implement CrossSystemRef

Refs #366"
```

---

## Batch 3: LedgerEntryId threading

### Task 4: Wire LedgerEventCapture to set ledgerEntryId

**Files:**
- Modify: `ledger/src/main/java/io/casehub/work/ledger/service/LedgerEventCapture.java`
- Test: `ledger/src/test/java/io/casehub/work/ledger/LedgerIntegrationTest.java` (existing — add test)

**Interfaces:**
- Consumes: `LedgerEntryIdSetter` from Task 1, `WorkItemLifecycleEvent.ledgerEntryIdSetter()` static accessor
- Produces: After `onWorkItemEvent()` completes, `event.ledgerEntryId()` returns the persisted entry's UUID

- [ ] **Step 1: Write the failing test**

In `LedgerIntegrationTest`, add:
```java
@Test
void ledgerEntryWritten_setsLedgerEntryIdOnEvent() {
    // Create and complete a WorkItem, capture the event
    // Assert event.ledgerEntryId() is non-null and matches the persisted entry ID
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `scripts/test-module.sh ledger`
Expected: FAIL — `ledgerEntryId()` returns null (setter not wired yet)

- [ ] **Step 3: Implement — add setter call to LedgerEventCapture**

In `LedgerEventCapture.onWorkItemEvent()`, after `ledgerRepo.save(entry, entry.tenancyId)` (line 148), add:

```java
WorkItemLifecycleEvent.ledgerEntryIdSetter().set(event, entry.id);
```

The `LedgerEntryIdSetter` accesses the private field via the static accessor. The `@Observes` method runs synchronously before any `@ObservesAsync` observer sees the event.

- [ ] **Step 4: Run test to verify it passes**

Run: `scripts/test-module.sh ledger`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add ledger/
git commit -m "feat(ledger): set ledgerEntryId on WorkItemLifecycleEvent after entry persisted

Refs #365"
```

### Task 5: Thread ledgerEntryId through adapters to completion appliers

**Files:**
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java`
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java`
- Modify: `engine-adapter/src/main/java/io/casehub/work/engine/ActionGateCompletionApplier.java`
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/WorkItemLifecycleAdapterTest.java` (existing — add tests)
- Test: `engine-adapter/src/test/java/io/casehub/work/engine/PlanItemCompletionApplierTest.java` (existing — add test)

**Interfaces:**
- Consumes: `WorkItemEvent.ledgerEntryId()` from Task 1, `LedgerEventCapture` wiring from Task 4
- Produces: `PlanItemCompletionApplier.apply(UUID caseId, String planItemId, WorkItemStatus status, WorkItemRef ref, UUID workLedgerEntryId)`
- Produces: `ActionGateCompletionApplier.apply(GateRef gateRef, WorkItemStatus status, WorkItemRef ref, String tenancyId, UUID workLedgerEntryId)`

- [ ] **Step 1: Write the failing test — adapter passes ledgerEntryId**

In `WorkItemLifecycleAdapterTest`, add:
```java
@Test
void workItemCompleted_passesLedgerEntryIdToApplier() {
    UUID ledgerEntryId = UUID.randomUUID();
    // Build a terminal event with ledgerEntryId set
    // Fire through adapter
    // Verify applier.apply() received the ledgerEntryId
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `scripts/test-module.sh engine-adapter`
Expected: FAIL — `apply()` doesn't accept ledgerEntryId yet

- [ ] **Step 3: Add UUID workLedgerEntryId parameter to PlanItemCompletionApplier.apply()**

Use `ide_replace_member` to update the `apply` method signature:
```java
@Transactional
public void apply(UUID caseId, String planItemId, WorkItemStatus status, WorkItemRef ref, UUID workLedgerEntryId) {
```

Add DEBUG log at the start of the method:
```java
LOG.debugf("PlanItem %s completion: workLedgerEntryId=%s", planItemId, workLedgerEntryId);
```

- [ ] **Step 4: Add UUID workLedgerEntryId parameter to ActionGateCompletionApplier.apply()**

Update `apply` signature:
```java
public void apply(final GateRef gateRef, final WorkItemStatus status, final WorkItemRef ref,
                  final String tenancyId, final UUID workLedgerEntryId) {
```

Also update `applyGroupCompletion` if it routes through the same path. Add DEBUG log.

- [ ] **Step 5: Update WorkItemLifecycleAdapter to pass ledgerEntryId**

In `onWorkItemLifecycle()`, change:
```java
applier.apply(piRef.caseId(), piRef.planItemId(), status, wie.ref());
```
to:
```java
applier.apply(piRef.caseId(), piRef.planItemId(), status, wie.ref(), wie.ledgerEntryId());
```

In `routeGate()`, change:
```java
gateApplier.apply(gateRef, status, ref, tenancyId);
```
to:
```java
gateApplier.apply(gateRef, status, ref, tenancyId, wie.ledgerEntryId());
```

(This requires passing `wie` or `wie.ledgerEntryId()` into `routeGate()` — update the private method signature.)

- [ ] **Step 6: Fix all existing test callsites**

Existing tests call `applier.apply(caseId, planItemId, status, ref)` — update to include `null` as the fifth argument. Use find-and-replace within the test files.

Similarly update `gateApplier.apply(...)` calls in `ActionGateHandlerTest`.

- [ ] **Step 7: Run tests**

Run: `scripts/test-module.sh engine-adapter`
Expected: All existing + new tests pass.

- [ ] **Step 8: Commit**

```bash
git add engine-adapter/
git commit -m "feat(engine-adapter): thread ledgerEntryId through adapters to completion appliers

Refs #365"
```

---

## Batch 4: Integration test + docs

### Task 6: LedgerEntryIdThreadingIT + contributor-guide update

**Files:**
- Create: `integration-tests/src/test/java/io/casehub/work/it/LedgerEntryIdThreadingIT.java`
- Modify: `docs/guides/contributor-guide.md` — update CallerRef references to PlanItemRef/GateRef

**Interfaces:**
- Consumes: All prior tasks — end-to-end verification

- [ ] **Step 1: Write integration test**

```java
@QuarkusTest
class LedgerEntryIdThreadingIT {

    @Inject WorkItemService workItemService;
    @Inject WorkItemLedgerEntryRepository ledgerRepo;

    @Test
    void completedWorkItem_ledgerEntryIdAvailableOnAsyncEvent() {
        // 1. Create a WorkItem with a callerRef
        // 2. Complete it
        // 3. Observe the WorkItemLifecycleEvent via a test @ObservesAsync CDI bean
        // 4. Assert event.ledgerEntryId() is non-null
        // 5. Assert it matches the ledger entry for this workItemId
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `scripts/test-module.sh integration-tests`
Expected: PASS — end-to-end threading works in a real Quarkus container

- [ ] **Step 3: Update contributor-guide.md**

Update the engine-adapter module description and CallerRef format section to reflect renamed types:
- `PlanItemCallerRef` → `PlanItemRef`
- `GateCallerRef` → `GateRef`
- Add mention of `CrossSystemRef` interface

- [ ] **Step 4: Commit**

```bash
git add integration-tests/ docs/guides/contributor-guide.md
git commit -m "test: add LedgerEntryIdThreadingIT, update contributor guide for renamed types

Refs #365, Refs #366"
```

---

## References

- [2026-08-28-cross-system-ref-design.md] — design spec this plan implements
- [engine-adapter/src/main/java/io/casehub/work/engine/CallerRef.java:39] — existing sealed interface
- [engine-adapter/src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java:68] — inbound adapter (@ObservesAsync)
- [engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java:92] — apply() method
- [engine-adapter/src/main/java/io/casehub/work/engine/ActionGateCompletionApplier.java:60] — gate apply()
- [ledger/src/main/java/io/casehub/work/ledger/service/LedgerEventCapture.java:72] — @Observes sync observer
- [api/src/main/java/io/casehub/work/api/WorkItemLifecycleEvent.java:13] — event class
- [qhorus/src/main/java/io/casehub/work/qhorus/QhorusCallerRef.java:5] — qhorus callerRef record
- [GitHub #365] — thread work-ledger entry ID back to engine
- [GitHub #366] — typed CrossSystemRef replacing callerRef parsing
