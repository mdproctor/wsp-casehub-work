## D1: Example module split — work-standalone here, engine-integrated in engine repo

**Choice:** Work-standalone examples (3 scenarios) in the existing `examples/` module of casehub-work. Engine-integrated examples (saga orchestration, notification pipeline) filed as a separate issue in casehubio/engine and built in the engine repo's examples. Engine is downstream of work — engine-integrated examples cannot live in the work repo without creating a circular dependency.
**Alternatives:**
- New `examples-engine` module in casehub-work — creates circular repo dependency (work → engine). Violates the dependency direction: engine depends on work, never the reverse.
- Work-standalone only, no engine examples — leaves saga orchestration undocumented. Deferred, not abandoned.
**Rationale:** The dependency graph is work → platform, engine → work. Engine-integrated examples require engine SNAPSHOTs and engine-adapter — they belong in the downstream repo. This branch (#396) delivers the 3 standalone examples. A follow-up issue in casehubio/engine delivers the orchestration examples.
**Trade-offs:** Examples split across repos. Acceptable — they match the dependency direction and the developer learning journey (standalone first, engine second).
**Sources:** examples/pom.xml (no engine dependency), engine-adapter/pom.xml (depends on casehub-work-api), D2/D3 from saga spec decisions.md
**Exploration:** quick
**Status:** revised (R1: corrected circular dependency — engine examples belong in engine repo)

## D2: Work-standalone scenario design — 3 scenarios with layered capability coverage

**Choice:** Three standalone scenarios, each teaching a distinct aspect of compensation:

1. **Expense Approval Reversal** — full compensation lifecycle with different actors, audit trail, ledger causal chain. The "learn compensation" example.
2. **Multi-Step Loan Application Rollback** — application-driven reverse-order compensation of 3 callerRef-correlated WorkItems. The "build your own saga" example.
3. **Compensation Resilience** — all 3 guards exercised in clinical context, plus suspend/resume/complete lifecycle on the compensating WorkItem. The "defensive programming" example.

**Alternatives:**
- Single comprehensive scenario — too much cognitive load; developer can't isolate what they're learning
- Many small scenarios (one per API call) — too fragmented; compensation is a multi-step flow that loses meaning when atomized
**Rationale:** Each scenario has a clear learning objective and a relatable domain. Together they cover the full standalone API surface without redundancy. The capability coverage matrix has no gaps and minimal overlap.
**Trade-offs:** 3 scenarios is more implementation work than 1. Worth it — compensation is the most complex capability in the module and needs layered teaching.
**Sources:** CancelScenario.java (existing pattern), WorkItemService.compensate/markCompensated (API surface), CompensationStatus enum, CompensationLifecycleObserver
**Exploration:** deep-analysis
**Status:** captured

## D3: Transaction model — both patterns across the 3 scenarios

**Choice:** Show both transaction patterns across the scenarios:
- **Expense Approval Reversal:** single `@Transactional` method. Compensate + complete compensating WorkItem in one call. Auto-`markCompensated` fires within the transaction. Developer sees the end-to-end flow resolving immediately — "here's how it works."
- **Multi-Step Loan Rollback:** split transactions. Compensate in one call, observe COMPENSATING intermediate state, complete the compensating WorkItem in a separate step. Developer sees what production looks like when compensating work takes time — queries by compensationStatus, the COMPENSATING → COMPENSATED transition across requests.
- **Compensation Resilience:** inherently split — suspend/resume on the compensating WorkItem requires separate steps. The intermediate lifecycle is the point of the example.

**Alternatives:**
- All single-transaction — hides the intermediate COMPENSATING state that developers need to handle in production (UI filtering, status polling, queue visibility)
- All split-transaction — buries the auto-`markCompensated` behavior under ceremony; the "just works" path is the one developers reach for first
**Rationale:** Developers encounter both patterns. Quick compensation (system-driven reversal) resolves in one transaction. Slow compensation (human-driven reversal) requires observing and querying intermediate state. Teaching both prevents the "it worked in the example but not in production" failure mode.
**Trade-offs:** Split-transaction examples need either multiple endpoints or a multi-step endpoint. The Loan Rollback scenario uses a step-by-step endpoint with query parameters to advance each phase.
**Sources:** CompensationLifecycleObserver.java (synchronous CDI @Observes), existing single-POST pattern (CancelScenario.java)
**Exploration:** quick
**Status:** captured
