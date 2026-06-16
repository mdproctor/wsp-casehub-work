# HANDOFF — 2026-06-17

## Last Session

Completed #263 — replaced `FilterScope` enum (PERSONAL/TEAM/ORG) with `Path` from
`casehub-platform-api` in the queues module. All 20 files updated, 104 tests green,
pushed to upstream (`casehubio/work`). Two design discoveries worth noting: `WorkItemFilterBean.scope()`
was dead code deleted from the interface; filter/queue scope is management metadata with
no execution enforcement (captured as protocol PP-20260616-4896da).

## Immediate Next Step

Branch is closed and pushed. Pick from What's Next.

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #240 | design: human task lifecycle alignment | L | High | now critical |
| #253 | feat: MongoDB store — complete drop-in replacement | M | Med | |

## Key References

- Garden: GE-20260616-e15321 (JAX-RS @Path + platform Path import collision)
- Blog: `2026-06-16-mdp01-filtering-out-the-enum.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
