# HANDOFF — 2026-07-14

## Last Session

Closed #288 — queue summary REST endpoint. Renamed `InboxSummaryBuilder` → `WorkItemSummaryBuilder` (inner record `InboxSummary` → `Summary`) to reflect general-purpose scope. Added `oldestCreatedAt` field. New `GET /queues/{id}/summary` endpoint. Design-reviewed (3 rounds, $9.10). Landed as `e955f77`.

Filed #305 (SQL push-down aggregation) and #306 (caching/materialised views) as out-of-scope follow-ons.

## Immediate Next Step

Pick from What's Next — no blockers outstanding.

## What's Left

- casehubio/work#300 — update #287 issue body to include SlaBreachPolicy scope · XS · Low
- casehubio/garden#4 — create `definable-entity-types-labels.md` protocol file · S · Low
- casehubio/parent#354 — PLATFORM.md protocol cross-reference · XS · Low
- casehubio/engine#653 — engine work-adapter update for types (source-breaking) · S · Med
- 6 consumer apps need POM update: `casehub-engine-work-adapter` → `casehub-work-engine-adapter` · XS each · Low
- casehubio/work#305 — SQL push-down aggregation for queue/inbox summary · M · Med
- casehubio/work#306 — caching or materialised views for summary endpoints · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #298 | Event contract refactoring — replace @ConsumeEvent with direct calls | M | Med | Deferred from #290 |
| #299 | CloudEvent bridge for cross-service HumanTask creation | M | Med | Deferred from #290 |
| #152 | Split examples into core and full variants | M | Low | |
| #305 | SQL push-down aggregation for queue/inbox summary | M | Med | New — from #288 |

## References

- Spec: `docs/specs/2026-07-14-queue-summary-endpoint-design.md`
- Design review: `~/adr/casehub-work/queue-summary-endpoint-20260714-041643/`
- Blog: `2026-07-14-mdp01-summary-that-was-already-there.md`
