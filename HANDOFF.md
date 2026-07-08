# HANDOFF — 2026-07-08

## Last Session

Closed #290 — relocated `casehub-engine-work-adapter` to `casehub-work-engine-adapter`. Package renamed `io.casehub.workadapter` → `io.casehub.work.engine` via IntelliJ `ide_move_file` (22 files). Deleted broken `WorkItemCallerRef` from work-api. Fixed ARC42STORIES.MD callerRef format. Updated engine CLAUDE.md/DESIGN.md/javadocs, parent BOM. All 79 tests pass. Three repos pushed: work, engine, parent.

Design-reviewed adversarially (5 rounds, 14 issues, $15.46). Review caught: `ActionGateDeploymentHealthCheck` string literal, 6 consumer apps needing artifact updates, engine CLAUDE.md section migration, `WorkItemCallerRef` deletion over fix, callerRef format in ARC42STORIES §3. Filed #298 (event contract refactoring) and #299 (CloudEvent bridge) for deferred work.

## Immediate Next Step

The cross-repo issues from the #291 aftermath remain open: garden#4 (protocol file), parent#354 (PLATFORM.md cross-ref), engine#653 (work-adapter types update — note: artifact ID now `casehub-work-engine-adapter`). Or pick new feature work from What's Next.

## What's Left

- casehubio/garden#4 — create `definable-entity-types-labels.md` protocol file · S · Low
- casehubio/parent#354 — PLATFORM.md protocol cross-reference · XS · Low
- casehubio/engine#653 — engine work-adapter update for types (source-breaking) · S · Med
- 6 consumer apps need POM update: `casehub-engine-work-adapter` → `casehub-work-engine-adapter` (AML, clinical, devtown, life, fsitrading, SOC) · XS each · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #298 | Event contract refactoring — replace @ConsumeEvent with direct calls | M | Med | Deferred from #290 |
| #299 | CloudEvent bridge for cross-service HumanTask creation | M | Med | Deferred from #290 |
| #288 | Queue summary REST endpoint for dashboard cards | M | Med | Enhancement |
| #287 | Retrofit work SPIs to extend NamedStrategy | M | Med | engine#634 dep |
| #152 | Split examples into core and full variants | M | Low | |

## References

- Spec: `specs/2026-07-08-relocate-work-adapter-design.md` (workspace)
- Plan: `plans/2026-07-08-relocate-work-adapter.md` (workspace)
- Design review: `~/adr/casehub-work/relocate-work-adapter-20260708-141405/`
- Cross-repo: engine 1cb0622a+1cc96549, parent c2ab0f8a
