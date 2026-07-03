# HANDOFF — 2026-07-03

## Last Session

Backlog triage — grouped 10 open GitHub issues into 5 independent threads.
First-principles analysis of #172 revealed it's mis-scoped: the CloudEvent
contract is platform infrastructure (belongs in work-api), not a flow-specific
enhancement. Engine's `HumanTaskScheduleHandler` uses the same two-direction
pattern through different plumbing. Epic hygiene surfaced 11 unrecovered
specs/blogs on closed branches and 10 unstamped project branches.

## Immediate Next Step

Pick up #152 — split casehub-work-examples into core and full variants.
Run `/work` to start.

## Cross-Module

*Unchanged — `git show HEAD~2:HANDOFF.md`*

## What's Left

- Cross-repo briefing for Qhorus `work-and-workitems.md` — written but not committed to qhorus repo · XS · Low
- Branch hygiene — 11 unrecovered artifacts + 10 unstamped project branches on closed workspace branches · M · Low

## What's Next

Five independent threads — not phases of one initiative.

**Distributed WorkItems (#92 epic)** — coherent chain: #172 → #97 → #95

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #172 | CloudEvent contract + flow bridge (reframed from emit+listen) | M | Med | defines event types in work-api; flow bridge is first consumer |
| #97 | WorkItem event mesh — lifecycle events across services | L | High | needs #172's contract + Qhorus transport |
| #95 | Cross-service WorkItem federation | XL | High | needs #97; create in A, resolve in B |
| #92 | Epic parent — clustering + federation | XL | High | #93 ✅ #155 ✅; #172 #97 #95 remain |

**External Integrations (#79 epic)** — blocked on upstream stability

| # | Description | Scale | Complexity | Blocked by |
|---|-------------|-------|------------|------------|
| #79 | External System Integrations | XL | Med | CaseHub/Qhorus not stable |
| #39 | ProvenanceLink — PROV-O causal graph | L | High | #79 |

**Template evolution** — standalone

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #180 | Template versioning — immutable snapshots for reproducibility | L | Med | deferred from #170 |

**Project hygiene** — standalone

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #152 | Split casehub-work-examples into core and full variants | M | Low | — |

**Ideas (captured, not specced)** — unrelated to each other

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #237 | Structured progress — schema-validated, hierarchical | L | High | ideas-capture |
| #238 | Saga compensation support across platform | XL | High | ideas-capture; platform-wide |

## Key References

- Blog: `casehubio.github.io/_notes/2026-07-03-mdp01-five-threads-not-four-phases.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
