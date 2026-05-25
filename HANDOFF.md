# Handover — 2026-05-25

## Last Session

Reduced CLAUDE.md from 51 KB to 15 KB by extracting triggered content to on-demand files (`docs/GOTCHAS.md`, `docs/FLYWAY.md`, `scripts/README.md`) and collapsing the Project Structure per-file tree to a module-level summary table. Formalised two protocols (claude-md-size-discipline, cross-repo-issues-in-parent) in casehubio/parent. Cross-repo tracking issue moved from casehubio/work#226 to casehubio/parent#66.

## Immediate Next Step

Merge `issue-227-claude-md-rag-extraction` to main — rebase onto main, push to origin/main, stamp branch closed, then pick up engine#330 (WorkItem.scope V31) in the engine session.

## Cross-Module

**We're blocking:**
- `casehub-engine` — needs `WorkItem.scope` (V31) for `HumanTaskTarget.scope` · S · Low · engine#330

**Blocked by:** nothing.

## What's Left

- `epic-excluded-users` — 7 days old, no commits · S · Low
- `epic-output-schema` — 7 days old, no commits · S · Low
- `epic-exclusion-audit` — 5 days old · S · Low
- EPIC-CLOSED.md missing on `issue-215-escalation-removal-and-fixes` workspace branch · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#330 | WorkItem.scope V31 — HumanTaskTarget propagation | S | Low | Engine session, not here |
| parent#66 | Apply CLAUDE.md size discipline to remaining casehubio repos | L | Low | Checklist in issue; casehub-work is the reference impl |

## Key References

- Branch: `issue-227-claude-md-rag-extraction` (project + workspace, ready to merge)
- Issues: casehubio/work#227 (closed), casehubio/parent#66 (open, cross-repo)
- Protocols: PP-20260525-8c361f (claude-md-size-discipline), PP-20260525-5b1efa (cross-repo-issues-in-parent)
- Garden: GE-20260525-58fcbf (always-needed vs triggered technique), GE-20260525-3fe619 (wc -c vs wc -l gotcha)
- Blog: `2026-05-25-mdp02-always-needed-vs-triggered.md`
