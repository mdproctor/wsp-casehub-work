# casehub-work — Session Handover
**Date:** 2026-05-18

## What Was Done This Session

### OHT gaps #170 + #171 — both closed and merged to main

**#170 — Schema-validated output (epic-output-schema):**
- `WorkItemTemplate.inputDataSchema` + `outputDataSchema` (JSON Schema TEXT), snapshotted to `WorkItem`
- `WorkItemFormSchema` entity + CRUD API deleted — template is now the sole type definition
- `FormSchemaValidationService` moved from resource to service layer
- V23/V24/V25 Flyway migrations

**#171 — Excluded users (epic-excluded-users):**
- `ExclusionPolicy` SPI in `casehub-work-api` (`@DefaultBean`: `CommaSeparatedExclusionPolicy`)
- `excludedUsers` TEXT on template + WorkItem; enforced at 5 assignment paths
- `SelectionContext` carries `excludedUsers` for external strategy SPI coherence
- V26/V27 Flyway migrations

### Cherry-pick damage incident — found and fixed
A separate Claude session merged both epics using `git cherry-pick -X ours`, silently dropping the REST surface and mapper for `excludedUsers` while keeping the service-layer enforcement. 41 files fixed. All 673 runtime tests passing. 30-point verification confirmed correct.

### Epic skill updated
- Step B8: branches never deleted automatically; EPIC-CLOSED.md written with 14-day deletion date; always return to main after close (with prompt)
- Workflow C: branch hygiene scan — checks Flyway V conflicts, code merge status, artifact promotion, closure marker, deletion eligibility

### Protocols and skills updated
- `flyway-version-range-allocation.md`: branch-level V-number reservation rule (`flyway-next-v` in `.meta`)
- `java-dev` skill: extended IntelliJ-first rule to cover bulk structural edits — never Python/bash for Java source manipulation

## Current State
- Both repos on `main`
- Both epic branches kept (deletion scheduled 2026-06-01): `epic-output-schema`, `epic-excluded-users`
- 673 runtime tests, 0 failures

## Immediate Next

**casehub-clinical — Epic 4 adverse event escalation** — next substantive feature work.

Or: address engine SelectionContext call sites (#187) — 2 files in casehub-engine need `excludedUsers` as 8th arg.

## Open / Next

| Priority | What |
|---|---|
| next | casehub-clinical Epic 4 adverse event escalation |
| engine | casehub-engine #187 — SelectionContext call sites need 8th arg (excludedUsers) |
| blocked | casehubio/work#97 — event mesh (blocked on qhorus#131, #132) |
| debt | #181 WorkItemWithAuditResponse missing permittedOutcomes; #182 WorkItemCreateRequest builder; #183 JsonNode TextNode validation |

## Key References
- Blog: `blog/2026-05-18-mdp01-template-as-type-system.md`
- Garden: GE-20260518-cf67e4 (cherry-pick -X ours), GE-20260518-d1e4b2 (Python Java parsing), GE-20260518-96bd10 (IntelliJ MCP stale cache)
