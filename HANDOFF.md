# HANDOFF — casehub-work

**Date:** 2026-07-05
**Branch:** issue-286-tenancy-query-versioning (closed → landed as 2367e48 on main)
**Issues closed:** #286, #180

## What Was Done

### #286 — Tenancy-aware query via WorkItemService
- Added `tenancyId` field to `WorkItemQuery` (AND filter, null = use current principal)
- Exposed `scan(WorkItemQuery)` on `WorkItemService` — consumers no longer need to reach into `WorkItemStore`
- All three store backends (JPA, MongoDB, InMemory) use `query.tenancyId()` when set, falling back to `currentPrincipal.tenancyId()`
- 5 unit tests in `WorkItemServiceTest`

### #180 — Template versioning
- Added `version` (long, starts at 1) to `WorkItemTemplate` — incremented on every PUT/PATCH
- Added `templateVersion` (Long) to `WorkItem` — set at instantiation from `template.version`
- Wired through `WorkItemCreateRequest`, `WorkItemTemplateService.mergeRequestWithTemplate()`, all REST responses, MongoDB document mappers
- Version increment placed after validation to avoid persisting on failed updates
- Flyway V41 migration
- 4 integration tests in `WorkItemTemplateTest`

## What's Left

- **Ledger module build failure** — pre-existing, unrelated to this branch. Cannot find symbols in ledger classes. Not a regression.
- **Cross-repo briefing for Qhorus** — `work-and-workitems.md` written but not committed to qhorus repo (from prior session)
- **parent#345** — sync PLATFORM.md for CloudEvent inbound adapter (from prior session)

## What's Next

**Distributed WorkItems (#92 epic)** — chain: #172 done → #290 → #97 → #95

| # | Description | Scale | Complexity |
|---|-------------|-------|------------|
| #290 | Relocate HumanTask adapter from engine to work | L | Med |
| #97 | WorkItem event mesh — lifecycle events across services | L | High |
| #95 | Cross-service WorkItem federation | XL | High |

**Standalone**

| # | Description | Scale | Complexity |
|---|-------------|-------|------------|
| #152 | Split casehub-work-examples into core and full | M | Low |
| #237 | Structured progress — schema-validated | L | High |
| #238 | Saga compensation support | XL | High |
