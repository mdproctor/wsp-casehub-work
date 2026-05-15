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

### Handover skill fix
- `handover` skill updated (both installed + cc-praxis source) to add "resume handover" workflow with workspace-awareness: checks CLAUDE.md for workspace path before reading HANDOFF.md

## Immediate Next Step

Start epic #92 child issue **#97** — WorkItem event mesh. The webhooks path is open now (notifications module already does outbound webhooks); check what #97 specifically requires beyond that before assuming it's done.

## Open / Next

| Priority | What |
|---|---|
| 1 | casehubio/work#97 — event mesh (webhooks path open, Qhorus path blocked) |
| 2 | casehubio/work#166 — multi-instance + callerRef (needs design session) |
| 3 | casehubio/work#169–171 — OHT gaps (named outcomes, output schema, excluded owners) |
| 4 | casehub-clinical — Epic 4 adverse event escalation |

## Key References

- Stale squash plans (can delete): `docs/squash-plan-2026-05-07.md`, `docs/squash-plan-v3-2026-05-08.md`
- Blog entry: `blog/2026-05-15-mdp01-swf-doesnt-have-a-human.md`
- Clinical context: `~/claude/casehub/clinical/wksp/HANDOFF.md`
