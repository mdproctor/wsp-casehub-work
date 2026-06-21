# HANDOFF — 2026-06-21

## Last Session

Closed #270 (MongoWorkItemDocument field sync). Added 13 missing fields to
MongoWorkItemDocument with from()/toDomain() mapping. Implemented OCC in
MongoWorkItemStore.put() via version-checked replaceOne. Fixed buildFilter()
ignoring the outcome query dimension. Overrode 5 WorkItemStore methods with
proper MongoDB server-side filtered queries. Filed #271 (JPA terminal status
hardcoding) and #272 (WorkItemStore OCC contract docs).

## Immediate Next Step

Cross-repo follow-up: engine#539 remains the highest priority — the engine
adapter, applier, recovery service, and gate applier all need FAULTED/OBSOLETE
handling. This is engine-repo work, not casehub-work.

## Cross-Module

**We're blocking:**
- engine#539 — adapter/applier/recovery/gate need FAULTED + OBSOLETE · M · Med
- parent#284 — LIFECYCLE.md normative doc + PLATFORM.md updates · L · Med

## What's Left

- #271 — JPA countByParentAndAssignee hardcodes 4 of 7 terminal statuses · S · Low
- #272 — WorkItemStore.put() contract should document OCC requirement · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#539 | engine adapter/applier/recovery/gate for FAULTED/OBSOLETE | M | Med | spec sections 9a-9f; engine repo |
| parent#284 | LIFECYCLE.md + PLATFORM.md lifecycle protocol | L | Med | parent repo |
| qhorus#291 | CommitmentState.DELEGATED javadoc cross-reference | XS | Low | qhorus repo |
| engine#540 | PlanItemStatus.SUSPENDED observability gap | S | Med | engine repo |
| work#271 | JPA countByParentAndAssignee terminal status hardcoding | S | Low | this repo |
| work#272 | WorkItemStore.put() OCC contract docs | XS | Low | this repo |

## Key References

- Design spec: `docs/specs/2026-06-20-mongo-document-field-sync-design.md`
- Blog: `blog/2026-06-21-mdp01-thirteen-fields-that-werent-there.md`
- Plan: `plans/attic/issue-270-mongo-document-field-sync/2026-06-21-mongo-document-field-sync.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
