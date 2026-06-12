# HANDOFF — 2026-06-12

## Last Session

Fixed all test failures across 15 modules after the multi-tenancy migration (#256). Root causes: `MutableCurrentPrincipal` state leaking between test classes, `quarkus.scheduler.enabled=false` removing the Scheduler CDI bean, tenancyId not set before Merkle hash computation, direct Panache persist without tenancyId in examples/scenarios. CI green. Designed #234 (InboundMessage → WorkItem) as `casehub-engine-inbound` bridge module — spec and issues filed in engine repo. Created `fix-ci` skill. Closed #261, #262, #234.

## Immediate Next Step

No trailing work in casehub-work. Pick from What's Next or switch to engine to implement `casehub-engine-inbound` (engine#468).

## What's Left

- ledger exclude-types workaround — still temporary · XS · Low
- `JpaWorkItemLedgerEntryRepository.save()` does raw `em.persist()` without the ledger save pipeline — fragile divergence from `JpaLedgerEntryRepository.save()` · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#468 | feat: casehub-engine-inbound bridge module | M | Med | spec in engine wksp |
| engine#469 | test: E2E Slack InboundMessage → Qhorus → WorkItem | S | Med | depends on engine#468 |
| #240 | design: human task lifecycle alignment | L | High | now critical |
| #236 | feat: replace VocabularyScope enum with Path-based hierarchy | M | Low | |
| #253 | feat: MongoDB store — complete drop-in replacement | M | Med | |

## Key References

- Garden: GE-20260612-ce4271 (principal leak), GE-20260612-40ee33 (scheduler bean), GE-20260612-c24e9d (hash pipeline)
- Skill: `fix-ci` — local-first CI fix workflow
- Spec: `casehub-engine/wksp/specs/2026-06-10-inbound-message-workitem-bridge.md`
- Blog: `2026-06-11-mdp01-the-tenancy-that-kept-leaking.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
