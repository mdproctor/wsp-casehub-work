# HANDOFF — 2026-06-22

*Updated: parent#284 closed — removed from backlog.*

## Last Session

Closed #271 and #272 — the last two follow-ups from the #270 review. Replaced
hardcoded terminal status lists in both JPA stores with a shared
`WorkItemStatus.TERMINAL_STATUSES` constant. Documented OCC as a `put()` contract
requirement. All four store backends now reference the same source of truth.

## Immediate Next Step

No this-repo work or cross-module blockers are queued. Check open issues for
the next piece of work: `gh issue list --repo casehubio/work --state open`.

## What's Next

No queued items — check open issues.

## Key References

- Blog: `blog/2026-06-22-mdp01-the-hardcoded-seven.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
