# HANDOFF — 2026-07-02

## Last Session

Closed #159 (normative alignment) on branch `issue-159-normative-alignment`.
Created `docs/NORMATIVE-ALIGNMENT.md` — complete mapping of all 12
WorkItemStatus → CommitmentState and all 26 WorkEventType → MessageType.
Updated LAYERING.md and ARC42STORIES.MD glossary with cross-references.
Cross-repo briefing written for Qhorus `work-and-workitems.md` update.

## Immediate Next Step

Pick up #152 — split casehub-work-examples into core and full variants.
Run `/work` to start.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- Cross-repo briefing for Qhorus `work-and-workitems.md` — written but not committed to qhorus repo · XS · Low

## What's Next

**Ready to pick up:**

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #152 | Split casehub-work-examples into core and full variants | M | Low | standalone |
| #172 | emit+listen SWF 1.0 bridge for casehub-work-flow | M | Med | idiomatic SWF async pattern |
| #180 | Template versioning — immutable snapshots for reproducibility | L | Med | deferred from #170 |

**Epics (active):**

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #92 | Distributed WorkItems — clustering + federation | XL | High | #93 ✅ #155 ✅; #95 #97 remain |
| #95 | Cross-service WorkItem federation | XL | High | under #92; create in A, resolve in B |
| #97 | WorkItem event mesh — lifecycle events across services | L | High | under #92; depends on Qhorus event mesh |

**Epics (blocked):**

| # | Description | Scale | Complexity | Blocked by |
|---|-------------|-------|------------|------------|
| #79 | External System Integrations | XL | Med | CaseHub/Qhorus not stable |
| #39 | ProvenanceLink — PROV-O causal graph | L | High | #79 (needs upstream integrations) |

**Ideas (captured, not specced):**

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #237 | Structured progress — schema-validated, hierarchical | L | High | ideas-capture |
| #238 | Saga compensation support across platform | XL | High | ideas-capture; platform-wide |

## Key References

- Blog: `blog/2026-07-02-mdp02-the-map-that-was-already-there.md`
- Normative alignment: `docs/NORMATIVE-ALIGNMENT.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
