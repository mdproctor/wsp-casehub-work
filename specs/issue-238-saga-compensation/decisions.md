## D1: Scope

**Choice:** Full cross-platform design — spec covers all 5 repos (work, engine, ledger, qhorus, connectors), decomposed into sequenced child issues for implementation.
**Alternatives:**
- casehub-work only — implement work states first, defer engine/ledger/qhorus/connectors
- work + engine only — design the orchestration core, defer periphery
**Rationale:** Compensation is a cross-cutting concern. Designing each repo in isolation risks inconsistent models and rework. The spec defines the conceptual model; child issues scope the implementation.
**Trade-offs:** Larger spec, longer design phase. Worth it to prevent cross-repo incoherence.
**Sources:** casehub-work#238 issue body (scope section), lifecycle alignment spec (#240 §13)
**Exploration:** quick
**Status:** captured

## D2: Compensation model — separate compensating WorkItem

**Choice:** Original WorkItem stays COMPLETED (terminal invariant preserved). A new compensating WorkItem is created with `compensatesWorkItemId` linking back. Original gets a denormalized `compensationStatus` field (NONE/COMPENSATING/COMPENSATED).
**Alternatives:**
- Post-terminal status transition (COMPLETED → COMPENSATING) — breaks isTerminal() contract, requires auditing every consumer across the platform
- Metadata-only (ledger tracking) — no WorkItemStatus changes, loses lifecycle visibility (SSE events, filter rules, queue management)
**Rationale:** The lifecycle alignment spec (#240) just completed a massive audit of all isTerminal() consumers for FAULTED/OBSOLETE. Breaking the terminal invariant again would create the same class of bugs. Compensation is new work — a separate entity is the right model.
**Trade-offs:** Queries become slightly more complex ("is this WorkItem effectively compensated?" requires checking compensationStatus). Indirect causal relationship (linked WorkItem vs status on the same entity).
**Sources:** WorkItemStatus.java (12 values, isTerminal()), lifecycle alignment spec (#240 §9a-9f hardcoded status audit), LedgerEntry.causedByEntryId
**Exploration:** quick
**Status:** captured

## D3: Compensation trigger model

**Choice:** Engine-driven (primary) + explicit operator action (secondary). Engine fires compensate event on bindings (BPMN-style). Operators can trigger compensation on a single WorkItem via REST API. Both create compensating WorkItems.
**Alternatives:**
- Engine-driven only — prevents ad-hoc compensation, limits standalone casehub-work deployments
- Any caller via API — maximum flexibility but no coordination guarantee for multi-step sagas
**Rationale:** The engine owns saga coordination (reverse-ordering, scope tracking). Operator action is the escape valve for targeted fixes on individual WorkItems without case context.
**Trade-offs:** Two entry points to compensation (engine and operator) — must share the same underlying mechanism to avoid divergence.
**Sources:** casehub-work#238 issue body (choreography vs orchestration section)
**Exploration:** quick
**Status:** captured

## D4: Compensation ordering — strict reverse

**Choice:** Compensating actions execute in strict reverse-completion order. Step N compensated before step N-1.
**Alternatives:**
- Parallel compensation — all compensating actions created simultaneously, faster but unordered
- Configurable per binding — maximum flexibility, adds complexity to case definition model
**Rationale:** Deterministic, auditable, matches BPMN 2.0 compensation semantics. The engine already tracks PlanItem completion order via EventLog. Reverse ordering ensures causal consistency — later steps that depended on earlier ones are undone first.
**Trade-offs:** Slower than parallel. For independent steps, reverse ordering is unnecessarily sequential — but correctness trumps speed for compensation.
**Sources:** BPMN 2.0 compensation semantics, casehub-work#238 issue body (orchestrated saga section)
**Exploration:** quick
**Status:** captured

## D5: Partial compensation — full scope only

**Choice:** Compensation always runs from the latest completed step backward. No cherry-picking individual steps at the engine level.
**Alternatives:**
- Partial allowed — engine can compensate a subset, requires tracking which steps were/weren't compensated
**Rationale:** Partial compensation creates complex state management (which steps are compensated, which aren't) and risks inconsistent domain state. Operator single-item compensation (D3) is the escape valve for targeted fixes.
**Trade-offs:** Cannot surgically compensate step 5 without compensating steps 1-4 at the case level. The operator API handles the surgical case.
**Sources:** Saga pattern literature, casehub-work#238 open questions section
**Exploration:** quick
**Status:** captured

## D6: Compensation failure — COMPENSATION_FAULTED terminal state

**Choice:** New terminal state COMPENSATION_FAULTED on CaseStatus. When a compensating action fails (FAULTED, REJECTED), the case enters COMPENSATION_FAULTED. Requires human intervention. No automatic retry.
**Alternatives:**
- Retry then fault — configurable retry count, adds complexity
- Propagate as FAULTED — simpler but loses the distinction between original faults and compensation faults
**Rationale:** Compensation failure is semantically distinct from execution failure. A case that faulted during execution is different from one that faulted during compensation — the latter is in a partially-compensated state that requires careful intervention.
**Trade-offs:** New terminal state on CaseStatus. All CaseStatus consumers must handle it. Worth it for the semantic clarity.
**Sources:** casehub-work#238 open questions ("Can compensation itself fail?")
**Exploration:** quick
**Status:** captured

## D7: Case-level model — new CaseStatus values

**Choice:** CaseStatus gains COMPENSATING, COMPENSATED, and COMPENSATION_FAULTED. A COMPLETED case transitions to COMPENSATING when the engine fires a compensate event. COMPENSATED is terminal after all compensating bindings complete.
**Alternatives:**
- Compensation as a child case — mirrors WorkItem separate-entity decision, but cases are heavyweight
- Planner-internal only — hides compensation from case-level observers
**Rationale:** The case IS the thing being compensated. It makes sense for CaseStatus to reflect this. Unlike WorkItem (where compensation creates new work), case compensation is a state of the orchestration itself.
**Trade-offs:** COMPLETED → COMPENSATING is a post-terminal transition at the case level. CaseStatus consumers must be audited. This is acceptable because CaseStatus has fewer consumers than WorkItemStatus, and the semantic fit is strong.
**Depends on:** D6 (COMPENSATION_FAULTED)
**Sources:** CaseStatus.java (7 values), BPMN 2.0 compensation model
**Exploration:** quick
**Status:** captured

## D8: Worker binding model — compensate: block on any Binding

**Choice:** Any Binding in the case definition can declare an optional `compensate:` block pointing to another Binding. The engine invokes the compensating binding when compensation is triggered. Worker-agnostic — a HumanTask can be compensated by a Workflow step and vice versa. HTN and Workflow workers can opt in.
**Alternatives:**
- Compensating binding per worker type — fragments the model across worker implementations
- Compensation handler SPI — clean separation but adds another SPI surface
**Rationale:** The engine is worker-agnostic by design. Compensation should follow the same pattern: the case definition declares what compensates what; workers execute. Any worker that supports the binding model gets compensation for free.
**Trade-offs:** Workers that don't support the binding's target type can't provide compensating actions. This is by design — not all workers need compensation.
**Sources:** casehub-engine Binding/BindingTarget model, user clarification (HTN and Workflow workers)
**Exploration:** quick
**Status:** captured

## D9: Ledger model — CompensationSupplement

**Choice:** New LedgerSupplement subclass (CompensationSupplement) alongside ComplianceSupplement and ProvenanceSupplement. Attaches to any LedgerEntry to mark it as a compensation record. Carries: originalEntryId, compensationReason, regulatoryBasis. Uses existing causedByEntryId for the causal chain.
**Alternatives:**
- New COMPENSATION LedgerEntryType — conflates entry type with compensation nature (a compensation entry is still an EVENT)
- Domain data only — loses type safety and queryability
**Rationale:** The supplement model is designed exactly for this: cross-cutting concerns attached to any entry without changing the core schema. CompensationSupplement follows the same pattern as ComplianceSupplement (GDPR) and ProvenanceSupplement (workflow source). causedByEntryId already provides the causal chain.
**Trade-offs:** New supplement table and migration. Acceptable — supplements are self-contained.
**Sources:** LedgerEntry.java (supplements, causedByEntryId), LedgerEntryType.java, ComplianceSupplement, ProvenanceSupplement
**Exploration:** quick
**Status:** captured

## D10: Qhorus integration — COMPENSATION_REQUESTED speech act

**Choice:** New message type COMPENSATION_REQUESTED on the channel. Agent is notified that their fulfilled commitment is being compensated. CommitmentState stays FULFILLED; a new commitment is created for the compensating work (mirrors WorkItem separate-entity model).
**Alternatives:**
- CommitmentState gains COMPENSATED — breaks terminal-state invariant (same issue as WorkItem)
- Defer Qhorus integration — agents won't be notified of compensation
**Rationale:** Consistent with D2 (separate entity model). The original commitment was fulfilled — that's a historical fact. Compensation is a new obligation requiring a new commitment. The speech act notifies the agent of the context change.
**Trade-offs:** Agents must handle a new speech act type. The new commitment's lifecycle is standard (OPEN → FULFILLED/DECLINED/etc).
**Depends on:** D2 (separate compensating WorkItem)
**Sources:** CommitmentState.java (7 values), QhorusWorkItemLifecycleAdapter, casehub-work#238 (Qhorus section)
**Exploration:** quick
**Status:** captured

## D11: Visualization — included in spec

**Choice:** The spec includes a visualization section covering: (1) design-time case-definition view showing bindings + compensating bindings in YAML, (2) runtime saga execution timeline, (3) ledger causal chain graph. Implementation deferred to child issues.
**Alternatives:**
- Separate visualization issue — risks disconnecting visualization requirements from the data model
**Rationale:** Visualization requirements influence what data to expose and index. Designing them alongside the model ensures the right fields are available.
**Trade-offs:** Larger spec. Worth it — sagas are complex; visualization is the primary tool for understanding them.
**Sources:** User request, YAML frontends epic (#371)
**Exploration:** quick
**Status:** captured

## D12: YAML format — primary, with programmatic API

**Choice:** YAML is the primary format for declaring compensating bindings in case definitions. Also available programmatically via the Java API. Consistent with the YAML frontends epic (#371). Existing tooling supports creating visualizers from YAML definitions.
**Alternatives:**
- Programmatic only initially — fragments the case definition experience
**Rationale:** YAML is the established format for case definitions. Existing tooling (from #371) makes visualization creation straightforward. Compensating bindings should be first-class in the YAML schema.
**Trade-offs:** YAML schema must be extended. The yaml-core module from #379 handles variable resolution.
**Depends on:** D8 (compensate: block on Binding)
**Sources:** YAML frontends epic (#371), yaml-core migration (#379), user clarification
**Exploration:** quick
**Status:** captured
