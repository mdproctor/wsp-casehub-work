# HANDOFF — casehub-work

**Date:** 2026-07-06
**Branch:** issue-292-extract-rest-module (closed → landed as ae56261 on main)
**Issues closed:** #292

## What Was Done

### #292 — Extract REST endpoints into casehub-work-rest module
- New `rest/` module (casehub-work-rest) — plain JAR + Jandex, opt-in REST surface
- 32 production files moved from runtime/api/ to rest/ (package `io.casehub.work.rest`)
- 40 test files moved (32 from api/, 6 spawn, 1 filter, 1 calendar — all hit REST endpoints)
- AuditEntryResponse moved to api/ as cross-cutting read model (not REST-specific)
- Downstream updated: queues/, integration-tests/, integration-tests-memory/ deps + imports
- Javadoc fixups (DeclineTarget, WorkItemRelation), MODULES.md, ARC42STORIES.MD updated
- Design review: 4 rounds, 16 issues raised, all resolved (spec substantially improved)
- Code review: 1 important finding (Mermaid diagram), fixed

## What's Left

- **parent#352** — sync casehub-work.md deep-dive for rest/ module extraction
- **Ledger module build failure** — pre-existing, unrelated
- **Cross-repo briefing for Qhorus** — `work-and-workitems.md` (from prior session)
- **Pre-existing test failures** — PermanentFilterRegistryTest (2 failures), WorkItemContextBuilderTest (1 failure) — not regressions

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
| #291 | Add types: Set<Path> to WorkItemTemplate and WorkItem | M | Med |
| #287 | Retrofit work SPIs to extend NamedStrategy | M | Med |
| #152 | Split casehub-work-examples into core and full | M | Low |
| #237 | Structured progress — schema-validated | L | High |
| #238 | Saga compensation support | XL | High |
