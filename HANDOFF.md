# Handoff — 2026-07-20

## What's Done

Migrated work-queues view infrastructure to platform-view toolkit (#312). QueueView entity, QueueViewStore, QueueMembershipTracker, QueueMembershipContext, QueueMembershipStore — all deleted (net -1076 lines). FilterEvaluationObserver delegates to SubjectViewOrchestrator. WorkItemViewQuery implements SubjectViewQuery<WorkItem>. Flyway V5003 migrates data. 88 queues + 37 examples tests green.

Also filed platform#188 (JPA tracker flush gap — fixed), platform#186 (cross-tenant store + boolean delete — landed), and work#314 (FilterEngine → LabelRule migration).

## Cross-Repo Commits (this session)

- **casehubio/platform** — `a3c1d94` on main: `feat(#189): add triggerEvents to LabelRule for event-scoped evaluation`. Adds `Set<String> triggerEvents` to `LabelRule` record with backward-compatible 3-arg constructor and `evaluate(rules, context, event)` overload. Platform#189 created and closed.
- **casehubio/platform** — `bb43338` on main: `feat(#191): add JexlExpressionEngine to platform expression module`. JEXL joins MVEL and JQ as a platform-level expression engine. Uses strict(false) but NOT silent(true). Platform#191 created and closed.

## Immediate Next Step

Continue brainstorming #314 — migrate FilterEngine/FilterEngineImpl to platform LabelRule. Design decisions so far:
- Unify both filter systems (queues FilterEngine + runtime FilterRegistryEngine) into one LabelRule-based engine
- LabelRule for label mutations; non-label actions (OVERRIDE_CANDIDATE_GROUPS, SET_PRIORITY) become downstream observers
- Single-pass evaluation (drop multi-pass — no production usage of chain propagation)
- Event-type filtering via platform's new `triggerEvents` on LabelRule (platform#189, landed)

## Cross-Module

**We're enabling** (other modules waiting on us):
- `casehub-engine` — engine#730 (case queues) follows the same SubjectViewOrchestrator pattern. #312 validates the pattern; #314 aligns the labelling layer so both repos use LabelRule.

## What's Left

- Platform#185 (proactive membership cleanup on view deletion) — adopt when it lands · S · Low
- Add platform-view modules to `casehub-parent` BOM — currently version-managed in work's parent POM · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #314 | Migrate FilterEngine to platform LabelRule | M | Med | In progress — brainstorming |
| #309 | Progress model: visualisation modes | M | Med | |
| #308 | Progress model: rollback control mechanism | S | Med | |
| #307 | Progress model: arbitrary JSON schema shape type | S | Low | |
