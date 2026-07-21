# Handoff — 2026-07-21

## What's Done

**#314 — Migrated FilterEngine to platform LabelRule** (this session). Unified both filter systems (queues `FilterEngine` + runtime `FilterRegistryEngine`) into one `LabelRuleEngine` using platform primitives. 83 files changed, net -2,315 lines. Single-pass evaluation, per-rule error isolation, INFERRED/MANUAL label protection.

Platform changes landed on main:
- platform#189 (`a3c1d94`) — `LabelRule.triggerEvents` for event-scoped evaluation
- platform#191 (`bb43338`) — `JexlExpressionEngine` in platform expression module

Follow-up issues filed:
- work#316 — label-triggered CDI observer pattern for non-label side effects
- work#317 — update ARC42STORIES.MD and api-reference.md for LabelRule migration

Prior session: #312 migrated the VIEW layer (QueueView → SubjectViewSpec, membership tracking → ViewMembershipTracker).

## Cross-Module

**We're enabling:**
- `casehub-engine#730` — engine case queues can now use the same `LabelRule` + `ExpressionEngineRegistry` for labelling
- Platform expression engines (JEXL, JQ, MVEL) shared across all repos via `ExpressionEngineRegistry`

## Known Issues

- `queues-examples` and `queues-dashboard` example scenarios have remaining compilation issues from the migration (field name mismatches in demo filter creation code). Core modules compile clean.
- `engine-adapter` has a pre-existing `ActionGateCompletionApplier` compilation error (unrelated to #314)

## What's Left

- work#317 — doc sync: update ARC42STORIES.MD and api-reference.md for LabelRule types · M · Low
- work#316 — label-triggered CDI observer pattern for non-label side effects · S · Low
- Platform#185 (proactive membership cleanup on view deletion) — adopt when it lands · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #317 | Docs: update ARC42 + api-reference for LabelRule | M | Low | |
| #309 | Progress model: visualisation modes | M | Med | |
| #308 | Progress model: rollback control mechanism | S | Med | |
| #307 | Progress model: arbitrary JSON schema shape type | S | Low | |
