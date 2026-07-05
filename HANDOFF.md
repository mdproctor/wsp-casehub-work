# HANDOFF — 2026-07-05

## Last Session

Designed, reviewed, and implemented #172 — CloudEvent WorkItem Bridge.
casehub-work is now CloudEvent-addressable: external systems fire a
`io.casehub.work.workitem.requested` CloudEvent to create WorkItems,
lifecycle events flow back as CloudEvents for correlation. Design review
(5 rounds, $14.58) drove significant improvements — dropped payloadType,
added TOCTOU mitigation, nuanced error handling. Filed #290 for HumanTask
adapter relocation from engine to work. Filed parent#345 for PLATFORM.md
doc sync.

## Immediate Next Step

Pick up #152 — split casehub-work-examples into core and full variants.
Run `/work` to start.

## Cross-Module

*Unchanged — `git show HEAD~2:HANDOFF.md`*

## What's Left

- Cross-repo briefing for Qhorus `work-and-workitems.md` — written but not committed to qhorus repo · XS · Low
- Branch hygiene — 11 unrecovered artifacts + 10 unstamped project branches on closed workspace branches · M · Low
- parent#345 — sync PLATFORM.md and casehub-work deep dive for CloudEvent inbound adapter · XS · Low

## What's Next

Five independent threads — not phases of one initiative.

**Distributed WorkItems (#92 epic)** — coherent chain: #172 ✅ → #290 → #97 → #95

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #290 | Relocate HumanTask adapter from engine to work | L | Med | new — filed this session |
| #97 | WorkItem event mesh — lifecycle events across services | L | High | needs #290 + Qhorus transport |
| #95 | Cross-service WorkItem federation | XL | High | needs #97 |

**External Integrations (#79 epic)** — blocked on upstream stability

| # | Description | Scale | Complexity | Blocked by |
|---|-------------|-------|------------|------------|
| #79 | External System Integrations | XL | Med | CaseHub/Qhorus not stable |
| #39 | ProvenanceLink — PROV-O causal graph | L | High | #79 |

**Standalone**

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #180 | Template versioning — immutable snapshots | L | Med | deferred from #170 |
| #152 | Split casehub-work-examples into core and full | M | Low | recommended next |
| #237 | Structured progress — schema-validated | L | High | ideas-capture |
| #238 | Saga compensation support | XL | High | ideas-capture; platform-wide |

## Key References

- Spec: `docs/specs/2026-07-04-cloudevent-workitem-bridge-design.md`
- Blog: `casehubio.github.io/_notes/2026-07-05-mdp01-where-does-the-cloudevent-stop.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
