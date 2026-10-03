# Handoff — casehub-work

## Recent Changes

### 2026-10-03 — #414: Remove dead Panache dependencies
- **Branch:** `issue-414-remove-dead-panache-deps` (landed as c3cc572c on main)
- **Commit:** `c3cc572c feat(#414): remove dead Panache dependencies from 9 modules`
- Replaced `quarkus-hibernate-orm-panache` with `quarkus-hibernate-orm` in 8 runtime modules
- Removed `panache-deployment` and `panache-common-deployment` from deployment module
- Fixed surviving Panache static finder `WorkItemRelation.findByTargetAndType()` in `LedgerEventCapture` — converted to EntityManager JPQL query
- Closed #414 and epic #413

## Issues Closed
- casehubio/work#414 — cleanup: remove 9 dead Panache Maven dependencies
- casehubio/work#413 — epic: Spring completeness — close remaining gaps
