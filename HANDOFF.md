# casehub-work — Session Handover
**Date:** 2026-05-21 (second session)

## What Was Done This Session

**All S/M backlog issues closed:**
- #195 — `WorkItemCreateRequest.toString()` log-safety comment
- #193 — `instantiate()` exclusion gap confirmed by test
- #194 — `LabelPersistence` → Tier 1 (casehub-work-api); `WorkItemLabelRequest` creation type
- #179 — `PUT /workitem-templates/{id}` update endpoint
- #178 — `?outcome=` filter on `GET /workitems`
- #176 — `reject()` accepts named outcome, validates against permittedOutcomes
- #117 — `RoundRobinStrategy` (cursor SPI, V29 migration, REQUIRES_NEW self-inject fix)
- #161 closed (already done), #168 closed (duplicate of #182)

**Upstream delivery:**
- 33 commits squashed to 14, pushed to `casehubio/work` main
- `epic-excluded-users` formally closed via work-end

**Deferred issues filed:**
- #196, #197 (fixed this session), #199 (PATCH alternative), #200 (inbox outcome filter), #201 (test isolation), #202 (cursor TTL), #203 (RoundRobinAssignmentStrategy inconsistency)

## Current State

- Both repos on `main`, both origin and upstream (`casehubio/work`) up to date
- 711 runtime tests, 32 core tests — all green
- V29 Flyway migration live
- `epic-exclusion-audit` and `epic-output-schema` branches still need work-end

## Immediate Next Step

Run `work-end` on `epic-exclusion-audit` and `epic-output-schema` — both merged to main but no `EPIC-CLOSED.md`.

## What's Left

- `epic-exclusion-audit` — work-end not run · XS · Low
- `epic-output-schema` — work-end not run · XS · Low
- #203 — `RoundRobinAssignmentStrategy` doesn't handle `"round-robin"` config · S · Low
- #202 — cursor row cleanup / TTL for stale `routing_cursor` rows · S · Low
- #200 — `?outcome=` filter on `GET /workitems/inbox` (trivial now cursor in place) · XS · Low
- casehub-engine #187 — SelectionContext 8th arg call sites in engine · S · Low (engine repo)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | work-end on epic-exclusion-audit + epic-output-schema | XS | Low | Do first |
| — | casehub-clinical Epic 4 — adverse event escalation | L | Med | — |
| #203 | RoundRobinAssignmentStrategy round-robin config | S | Low | — |
| #200 | ?outcome= on inbox endpoint | XS | Low | — |

## Key References

- Blog: `blog/2026-05-21-mdp02-retry-that-wasnt.md`
- Garden: GE-20260521-0e0122 (REQUIRES_NEW dead retry), GE-20260521-998034 (multi-catch subclass)
- Protocol: PP-20260521-903472 (built-in strategy registration, three-place atomicity)
