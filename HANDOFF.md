# Handoff — 2026-07-20

## What's Done

Migrated work-queues view infrastructure to platform-view toolkit (#312). QueueView entity, QueueViewStore, QueueMembershipTracker, QueueMembershipContext, QueueMembershipStore — all deleted (net -1076 lines). FilterEvaluationObserver delegates to SubjectViewOrchestrator. WorkItemViewQuery implements SubjectViewQuery<WorkItem>. Flyway V5003 migrates data. 88 queues + 37 examples tests green.

Also filed platform#188 (JPA tracker flush gap — fixed), platform#186 (cross-tenant store + boolean delete — landed), and work#314 (FilterEngine → LabelRule migration).

## Immediate Next Step

Start work on #314 — migrate FilterEngine/FilterEngineImpl to platform LabelRule. This is the labelling layer migration that completes the platform alignment (view layer done in #312, labelling layer next).

## Cross-Module

**We're enabling** (other modules waiting on us):
- `casehub-engine` — engine#730 (case queues) follows the same SubjectViewOrchestrator pattern. #312 validates the pattern; #314 aligns the labelling layer so both repos use LabelRule.

## What's Left

- Platform#185 (proactive membership cleanup on view deletion) — adopt when it lands · S · Low
- Add platform-view modules to `casehub-parent` BOM — currently version-managed in work's parent POM · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #314 | Migrate FilterEngine to platform LabelRule | M | Med | Immediate priority — completes platform alignment |
| #309 | Progress model: visualisation modes | M | Med | |
| #308 | Progress model: rollback control mechanism | S | Med | |
| #307 | Progress model: arbitrary JSON schema shape type | S | Low | |
