# Design Decisions — #332 Multi-Instance Coordinated Rollback

## D1: Rollback target — timestamp and event-based

**Choice:** Support both timestamp-based and event-based subtree rollback targets
**Alternatives:**
- Timestamp only — simpler API, but no way to anchor rollback to a specific event in the tree's history
- Event only — precise but requires the caller to know a specific event ID
**Rationale:** Event-based resolves to a timestamp internally and delegates to the timestamp path. Minimal implementation cost for supporting both. They serve different use cases: timestamp for "undo to 3pm," event for "undo to when the parent was at this state."
**Trade-offs:** Slightly larger API surface (one extra query param). Negligible.
**Sources:** ProgressService.rollback(), ProgressService.rollbackToEvent() — existing single-instance precedent
**Exploration:** quick
**Status:** captured

## D2: Rollup suppression during coordinated rollback

**Choice:** Suppress rollup during the operation, then recompute once bottom-up
**Alternatives:**
- Let rollup cascade naturally — each rolled-back child triggers independent rollup recomputations. Intermediate states visible, OCC contention under fan-in (retry capped at 3 attempts).
**Rationale:** Coordinated rollback means the subtree moves atomically at the observable level. Piecemeal cascade contradicts the purpose. OCC storm under high fan-in (e.g., 20 children) is a practical failure mode.
**Trade-offs:** More complex than letting the existing cascade handle it. Requires a suppression mechanism on events and explicit bottom-up recomputation.
**Sources:** RollupObserver.java (lines 46-52, 55-66), progress-enhancements spec §3
**Exploration:** quick
**Status:** captured

## D3: Failure mode — best-effort with report

**Choice:** Best-effort with structured result report
**Alternatives:**
- All-or-nothing — one problematic node blocks the entire subtree rollback. Implies transactional semantics that don't match the observer model.
- Pre-validate then apply (two-phase) — adds a preview API the common case doesn't need. Caller can achieve the same by querying snapshots first.
**Rationale:** Progress model is an observer, not an orchestrator. All-or-nothing implies control semantics. Best-effort with a report gives full visibility — the caller (orchestrator) decides how to handle skipped nodes.
**Trade-offs:** Caller must inspect the result to know if all nodes were rolled back. No atomic guarantee.
**Sources:** Progress model design principles — "observation, not orchestration"
**Exploration:** quick
**Status:** captured

## D4: Nodes created after the target point — skip and document

**Choice:** Skip nodes created after the target rollback point. Report them as "skipped: created after target." Document clearly that the orchestrator is responsible for deciding their fate.
**Alternatives:**
- Cancel/fail them — progress model doesn't own lifecycle control, that's the orchestrator's job
- Detach them (remove parentProgressId) — destructive, hard to undo, violates observer principle
**Rationale:** Consistent with observer principle. The progress model records state; it doesn't control which nodes exist. Orchestrator observes the result report and can explicitly fail/cancel post-target nodes via existing APIs.
**Trade-offs:** Post-target nodes remain in the tree in their current state after rollback. Documentation must be explicit about this.
**Sources:** Progress model design principles — "observation, not orchestration"
**Exploration:** quick
**Status:** captured

## D5: rollbackPolicy during coordinated rollback — bypass

**Choice:** Bypass `rollbackPolicy = "denied"` during coordinated subtree rollback
**Alternatives:**
- Honour policy — nodes with denied policy are skipped. Contradicts the existing single-instance precedent where explicit rollback always bypasses policy.
**Rationale:** Consistent with established precedent. Single-instance `rollback()` and `rollbackToEvent()` already bypass policy. The spec rationale: policy protects against accidental backward movement via `PUT /state`; explicit rollback endpoints are deliberate undo actions. Subtree rollback is even more deliberate.
**Trade-offs:** No per-node opt-out from coordinated rollback. If "immutability/freeze" is needed, it's a separate concept not yet in the model.
**Sources:** progress-enhancements spec §3 — "Internal structure — policy bypass"
**Exploration:** quick
**Status:** captured

## D6: Service placement — new SubtreeRollbackService

**Choice:** New `SubtreeRollbackService` in `progress-runtime`
**Alternatives:**
- Add to ProgressService — keeps all progress operations together, but ProgressService is already 400 lines and the coordination concern (tree traversal, rollup suppression, bottom-up recompute, result aggregation) is distinct from single-instance state management.
**Rationale:** Clean separation: ProgressService = single-instance, SubtreeRollbackService = multi-instance coordination. Tree traversal (`collectDescendants`) moves from REST layer to a shared location. Rollup suppression mechanism stays isolated from per-instance logic.
**Trade-offs:** One more service class. Minor.
**Sources:** ProgressService.java (403 lines), ProgressResource.collectDescendants() (REST-only tree traversal)
**Exploration:** quick
**Status:** captured

## D7: Rollup suppression mechanism — coordinatedRollbackId on events

**Choice:** Add `UUID coordinatedRollbackId` field to `ProgressUpdatedEvent`. RollupObserver checks this and skips rollup when set. SubtreeRollbackService sets it on all events during the operation.
**Alternatives:**
- Thread-local/request-scoped flag — doesn't work because @ObservesAsync runs on a different thread
- Don't emit events during rollback, emit batch afterward — loses per-node event granularity that SSE consumers and audit trails expect
**Rationale:** Minimal change (one field on event record, one if-check in observer). All existing consumers still see standard per-node ROLLED_BACK events. The coordinatedRollbackId lets consumers correlate which events belong to the same coordinated operation. Pre-release project, so breaking record change is acceptable.
**Trade-offs:** Breaking change to ProgressUpdatedEvent record (gains a field). All callers constructing the event must be updated. Pre-release — acceptable.
**Depends on:** D2 (rollup suppression strategy)
**Sources:** RollupObserver.onProgressUpdated() (line 46), ProgressUpdatedEvent record
**Exploration:** quick
**Status:** captured

## D8: Scope clarification — not full saga compensation

**Choice:** #332 is coordinated subtree state rollback only. "Saga-style" in the issue scope refers to the coordination pattern (visiting each node), not the full BPMN compensation model. Full saga compensation remains in #238.
**Alternatives:** None — this is a scope clarification, not a design choice.
**Rationale:** #238 owns full saga compensation (compensating bindings, compensation handlers, COMPENSATING/COMPENSATED states). #332's ROLLED_BACK events with coordinatedRollbackId already provide the notification surface for orchestrators to observe and act on — no additional compensation SPI needed.
**Trade-offs:** Issue #332 description should be updated to remove misleading "saga-style" wording.
**Sources:** Issue #238 (saga compensation), issue #332 scope
**Exploration:** quick
**Status:** captured
