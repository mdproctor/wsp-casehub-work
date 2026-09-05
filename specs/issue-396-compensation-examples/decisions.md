## D1: Example module split — work-standalone + engine-integrated

**Choice:** Both. Work-standalone examples (3 scenarios) in the existing `examples/` module, engine-integrated examples (2 scenarios) in a new module or the engine repo's examples. Each module targets a distinct developer journey: "I'm using casehub-work in my app" vs "I'm building case-driven workflows with the engine."
**Alternatives:**
- Work-standalone only — leaves engine saga orchestration undocumented in examples. Developers using the full platform miss the orchestration path.
- Engine-integrated only — skips the fundamental API tutorial. Developers using casehub-work standalone (without engine) have no compensation examples.
**Rationale:** The compensation API surface is split across two layers by design (D2, D3 in the saga spec). The developer learning journey mirrors this: first learn the work-level mechanics (compensate, guards, lifecycle), then learn how the engine orchestrates it at scale. Examples that blur this boundary misrepresent how the platform works.
**Trade-offs:** Two sets of examples to maintain. Acceptable — they exercise different layers and would diverge naturally.
**Sources:** examples/pom.xml (no engine-adapter dependency), WorkItemService.compensate() (work-level API), CaseCompensationService (engine-level orchestration), D2/D3 from saga spec decisions.md
**Exploration:** quick
**Status:** captured

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
