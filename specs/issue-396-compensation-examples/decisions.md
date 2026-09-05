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
