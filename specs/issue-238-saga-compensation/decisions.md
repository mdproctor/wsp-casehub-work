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

## D4: Compensation ordering — topological reverse

**Choice:** Compensating actions execute in reverse topological order of the dependency graph. Dependent steps compensate in reverse dependency order (C→B→A for a chain A→B→C). Independent steps compensate concurrently. Strict reverse-completion order falls out as the degenerate case for a linear chain.
**Alternatives:**
- Strict reverse-completion order — simpler but unnecessarily sequential for independent steps; not more correct, only slower
- Parallel compensation — all compensating actions created simultaneously, faster but unordered; unsafe for dependent steps
- Configurable per binding — maximum flexibility, adds complexity to case definition model
**Rationale:** The engine already has dependency information via Binding trigger conditions, produced/consumed keys, and EventLog completion order. A topological sort of the completion graph gives correct ordering for dependent steps (reverse dependency) while allowing concurrency for independent steps. This is more correct than strict reverse for cases with parallel branches — a step that had no causal relationship with another step in forward execution has no ordering constraint in reverse.
**Trade-offs:** More complex compensation coordinator implementation. Worth it — topological ordering is architecturally correct, not just faster.
**Sources:** BPMN 2.0 §10.4.3 compensation semantics, casehub-work#238 issue body (orchestrated saga section), Binding.produces/consumes/contextWrite dependency graph
**Exploration:** quick
**Status:** revised (R1-04: topological ordering is more correct than strict reverse for independent steps)

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
**Prerequisite:** CaseStatus must be registered in LIFECYCLE.md with `isTerminal()` and `isActive()` methods before adding compensation states. CaseStatus is currently unregistered (7 values, no lifecycle methods). Per LIFECYCLE.md §4, new lifecycle enum registration requires: define isTerminal()/isActive(), register in LIFECYCLE.md, audit all consumers, file cross-repo issues. This must be completed as a predecessor task.
**Depends on:** D6 (COMPENSATION_FAULTED)
**Sources:** CaseStatus.java (7 values, no isTerminal/isActive), LIFECYCLE.md (CaseStatus absent from registered state machines), BPMN 2.0 compensation model
**Exploration:** quick
**Status:** revised (R1-02: added CaseStatus LIFECYCLE.md registration prerequisite)

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

## D10: Qhorus integration — COMMAND with compensation context

**Choice:** Use existing COMMAND message type with compensation metadata (a `compensatesCommitmentId` field on the message content or artefactRefs). Agent is notified that their fulfilled commitment is being compensated via a standard COMMAND. CommitmentState stays FULFILLED; a new commitment is created for the compensating work (mirrors WorkItem separate-entity model).
**Alternatives:**
- New COMPENSATION_REQUESTED speech act — taxonomically inflating MessageType (currently 11 values) for a distinction that is contextual, not normatively distinct from COMMAND. Every new speech act propagates to MessageType consumers, MCP tools, channel projections, normative audit entries, and dashboards across all repos.
- CommitmentState gains COMPENSATED — breaks terminal-state invariant (same issue as WorkItem)
- Defer Qhorus integration — agents won't be notified of compensation
**Rationale:** A compensation request is semantically a directive — "undo your prior work." COMMAND already has the right normative semantics (creates obligation, requires correlationId). The compensation context is carried as metadata on the message, not as a new speech act type. The agent receives a COMMAND and discovers from context that it's compensating prior work. This avoids extending the speech-act taxonomy while conveying the same information.
**Trade-offs:** Compensation messages are less immediately distinguishable in audit views (they look like COMMANDs). But the CompensationSupplement on the ledger entry and the metadata on the message provide the context.
**Depends on:** D2 (separate compensating WorkItem)
**Sources:** MessageType.java (11 values: QUERY, COMMAND, RESPONSE, STATUS, DECLINE, HANDOFF, DONE, FAILURE, PROPOSE, JUDGMENT, EVENT), CommitmentState.java (7 values), QhorusWorkItemLifecycleAdapter
**Exploration:** quick
**Status:** revised (R1-05: COMMAND with metadata avoids taxonomy inflation; compensation is a directive, not a new speech act)

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

## D13: Compensation idempotency — state-guarded transitions

**Choice:** Compensation trigger is state-guarded and non-reentrant:
- COMPLETED → COMPENSATING: allowed (the normal trigger)
- COMPENSATING → reject with IllegalStateException (already compensating)
- COMPENSATED → reject (compensation complete, no re-compensation)
- COMPENSATION_FAULTED → allow manual retry (re-enters COMPENSATING from the faulted step)
- All other states → reject (only COMPLETED and COMPENSATION_FAULTED are valid entry points)
**Alternatives:**
- Allow re-compensation of COMPENSATED cases — creates recursive compensation chains; complex state management with no clear use case
- Reject COMPENSATION_FAULTED retry — leaves the case in a dead-end state with no recovery path
- Idempotent trigger (concurrent calls are no-ops) — hides race conditions rather than surfacing them
**Rationale:** COMPENSATION_FAULTED represents a partially-compensated state requiring human intervention. The operator's intervention is retrying from the faulted step. Rejecting retry from COMPENSATION_FAULTED would leave the case permanently stuck. Re-compensation of an already-COMPENSATED case has no semantic basis — the compensation is done.
**Trade-offs:** The retry path from COMPENSATION_FAULTED requires the engine to track which step faulted and resume from there.
**Sources:** R1-08 (reviewer-surfaced implicit decision)
**Exploration:** quick
**Status:** captured

## D14: Compensating WorkItems cannot be compensated

**Choice:** A compensating WorkItem (one with `compensatesWorkItemId != null`) cannot itself be compensated. `WorkItemService.compensate()` throws IllegalStateException if the target WorkItem has a non-null `compensatesWorkItemId`.
**Alternatives:**
- Allow chains (A → compensated by B → compensated by C) — creates recursive state management issues: what is A's compensationStatus when B is being compensated by C? Chain depth tracking and cycle prevention add complexity with no clear use case.
- Allow with depth limit — reduces but doesn't eliminate the state management complexity
**Rationale:** Compensation is a reversal of work, not a recursively composable operation. If compensating work itself needs reversal, that's a new forward operation (create a new WorkItem), not a meta-compensation. The `compensatesWorkItemId` link is a pair, not a chain.
**Trade-offs:** Operators who want to "undo a compensation" must create a new forward WorkItem manually. This is acceptable — the scenario is rare and the manual path is explicit.
**Sources:** R1-09 (reviewer-surfaced implicit decision)
**Exploration:** quick
**Status:** captured

## D15: Engine-work compensation interaction — skip already-compensated

**Choice:** When the engine's `HumanTaskCompensationHandler` encounters a WorkItem whose `compensationStatus` is already COMPENSATED (from prior operator action), it skips creating a new compensating WorkItem and marks the compensating PlanItem as COMPLETED. The engine proceeds to the next step.
**Alternatives:**
- Fail (unexpected state) — undesirable; operator action is a supported path
- Create a second compensating WorkItem — creates duplicate compensation
- Require operator to use engine path only — removes the escape valve that D3 provides
**Rationale:** D3 explicitly provides operator single-WorkItem compensation as an escape valve. The engine must handle the case where an operator already compensated a WorkItem before a full-scope case compensation is triggered. Skip-and-proceed is the only semantically correct behavior.
**Trade-offs:** The engine must read WorkItem compensationStatus via the work adapter. This creates a cross-module query, but it's a simple field read on an entity the handler already references.
**Sources:** R1-03/R1-12 (reviewer-surfaced implicit decision), D3 operator escape valve, D5 full scope
**Exploration:** quick
**Status:** captured
