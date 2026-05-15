# casehub-work — Session Handover
**Date:** 2026-05-15

## What Was Done This Session

### casehubio/work#167 — Code quality fixes (complete ✅)
- `@Transactional` Javadoc on 5-arg `instantiate()` clarified: annotation inactive on delegation path, active for direct external callers
- `@BeforeEach @Transactional cleanup()` added to `WorkItemTemplateInstantiateTest` (FK-safe order: AuditEntry → WorkItemSpawnGroup → WorkItem → WorkItemTemplate)
- New integration test: 4-arg `instantiate()` via full CDI path confirms `callerRef == null`
- Commit: `c171931`

### casehubio/engine#245 — HumanTask binding verified and closed ✅
- All four modules confirmed: api (181), blackboard (160), runtime (506), work-adapter (14) — 861 tests total
- Required Podman running for runtime module (Quarkus Dev Services)
- Issue closed

### SWF/OHT research — standards positioning
- SWF 1.0 has no human task type; human-in-the-loop is `emit+listen` event pair
- Open Human Task (OHT) companion spec exists but isn't gaining traction
- CaseHub WorkItem already covers most OHT capabilities; three gaps worth closing
- New issues filed: #169 (named outcomes), #170 (output schema), #171 (excluded owners), #172 (emit+listen SWF bridge)

### casehubio/work#97 — confirmed fully blocked
- Webhook path (option 1) is covered by `casehub-work-notifications` — no work needed
- Qhorus event mesh (chosen approach) blocked on qhorus#131, qhorus#132, and `casehub-work-qhorus` module not existing
- Comment added to #97

## Immediate Next Step

**#166** — multi-instance + callerRef design session. Needs brainstorming before any implementation.

## Open / Next

| Priority | What |
|---|---|
| 1 | casehubio/work#166 — multi-instance + callerRef (needs design session) |
| 2 | casehubio/work#169–171 — OHT gaps (named outcomes, output schema, excluded owners) |
| 3 | casehub-clinical — Epic 4 adverse event escalation |
| blocked | casehubio/work#97 — event mesh (blocked on qhorus#131, #132) |

## Key References

- Stale squash plans (can delete): `docs/squash-plan-2026-05-07.md`, `docs/squash-plan-v3-2026-05-08.md`
- Blog entry: `blog/2026-05-15-mdp01-swf-doesnt-have-a-human.md`
- Clinical context: `~/claude/casehub/clinical/wksp/HANDOFF.md`
