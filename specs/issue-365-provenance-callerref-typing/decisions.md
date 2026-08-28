## D1: CrossSystemRef type design

**Choice:** Sealed interface with integration-specific records
**Alternatives:**
- Single record with string fields — simpler but loses typed access to domain fields (caseId, channelId); no compile-time exhaustiveness
- Keep callerRef as String, add ledgerEntryId separately — doesn't solve #366 at all
**Rationale:** Type-safe sealed interface gives exhaustive switching and domain-specific accessors per integration module. `decode()` provides backward compatibility with existing callerRef format strings. `ledgerEntryId` is a first-class field on every variant.
**Trade-offs:** Adding a new integration module requires adding a new `permits` variant — but that's the right constraint (unknown formats use GenericRef fallback).
**Sources:** engine-adapter/src/main/java/io/casehub/work/engine/CallerRef.java, qhorus/src/main/java/io/casehub/work/qhorus/QhorusCallerRef.java, engine/common/src/main/java/io/casehub/engine/common/spi/CallerRefParser.java
**Exploration:** quick
**Status:** captured

## D2: ledgerEntryId field scope

**Choice:** UUID ledgerEntryId(), nullable — narrowly scoped to ledger entries
**Alternatives:**
- Generic sourceRef(system, id) pair — over-generalized for the actual use case; reasoning traces (engine#1007) are a different integration path, not callerRef
**Rationale:** CrossSystemRef is about creation provenance of a WorkItem — the ledger entry in the source system that caused this WorkItem to exist. Reasoning traces, EventLog entries, and memory store entries flow through different data paths and don't belong on CrossSystemRef.
**Trade-offs:** If someone ever wants callerRef to point to a non-ledger entity, they'd need to add a field. But this hasn't happened in practice and would indicate a design smell.
**Sources:** Slot 140 handoff (reasoning traces landing — confirms reasoning is stored in CaseMemoryStore, not on callerRef); engine#1007 (reasoning as lineage DAG participants — separate concern)
**Exploration:** quick
**Status:** captured

## D3: Ledger entry ID threading mechanism

**Choice:** Mutable field on WorkItemLifecycleEvent, set by LedgerEventCapture after writing the entry
**Alternatives:**
- Post-capture CDI event (WorkItemLedgerEntryCreatedEvent) — clean separation but new event class, dual-observation risk, more complex
- Query at adapter time — DB query on every terminal event, wrong dependency direction (engine-adapter shouldn't depend on work-ledger)
**Rationale:** CDI guarantees @Observes (sync) completes before @ObservesAsync fires. LedgerEventCapture writes the entry synchronously and sets the field. WorkItemLifecycleAdapter reads it asynchronously — field is guaranteed populated. WorkItemLifecycleEvent is already a mutable class.
**Trade-offs:** Relies on CDI observer ordering guarantee. Mutation of an event object is slightly unusual but narrowly scoped and documented.
**Sources:** ledger/src/main/java/io/casehub/work/ledger/service/LedgerEventCapture.java:72 (@Observes), engine-adapter/src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java:68 (@ObservesAsync), slot 140 handoff (reasoning uses same "add to event, thread through pipeline" pattern)
**Exploration:** quick
**Status:** captured

## D4: Migration path for CallerRef and QhorusCallerRef

**Choice:** Replace and delete in one step — clean break
**Alternatives:**
- Parallel types with adapters and deprecation — unnecessary caution for a pre-release project with no external consumers
**Rationale:** Pre-release stage, no external consumers. CallerRef sealed interface (engine-adapter) and QhorusCallerRef record (qhorus) both live in this repo. PlanItemCallerRef/GateCallerRef become PlanItemRef/GateRef in work-api. QhorusCallerRef becomes QhorusRef in work-api. encode() methods produce identical strings — backward compatible. Engine-side CallerRefParser stays as-is (follow-on issue in engine repo).
**Trade-offs:** Touches engine-adapter and qhorus module in one change. But both are in this repo and there are no external consumers.
**Sources:** CLAUDE.md (Stage: pre-release), engine-adapter/src/main/java/io/casehub/work/engine/CallerRef.java, qhorus/src/main/java/io/casehub/work/qhorus/QhorusCallerRef.java
**Exploration:** quick
**Status:** captured
