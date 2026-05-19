# casehub-work — Session Handover
**Date:** 2026-05-19

## What Was Done This Session

**Small issues closed on main (no epic):**
- #181 — `permittedOutcomes` missing from `WorkItemWithAuditResponse`
- #188 — null `claimantId` guard in `claim()`; RoundRobin pre-filter for excluded users; SSE test calling claim with wrong param
- #183 — JsonNode TextNode validation at template creation (400 instead of silent double-encode)
- #174 — DB-level UNIQUE constraint on `WorkItemTemplate.name` (V28 migration); surfaced test isolation debt across 8+ `@QuarkusTest` classes using hardcoded template names in a shared H2 instance

**#186 — blocked-attempt audit (on `epic-exclusion-audit`, not yet merged):**
- `ExclusionPolicy.isExcluded() : boolean` → `check() : PolicyDecision` — denial reason now travels from the policy to audit entries and exception messages
- `PolicyDecision` record in `casehub-work-api` (Tier 1, pure Java); `ALLOW` constant avoids allocation
- `BlockedAttemptAuditService` with `@Transactional(REQUIRES_NEW)` writes `CLAIM_DENIED` / `DELEGATE_DENIED` — always returns normally (exceptions swallowed) so a transient DB failure doesn't convert a 409 into a 500
- Guard ordering fixed: status check before exclusion check — Claude caught this in code review; without the fix, phantom audit entries would appear for rejected-status operations
- 685 tests passing on the branch

Epic `epic-exclusion-audit` closed — branch retained until **2026-06-02**.

## Current State

- Project + workspace: both on `main`
- `epic-exclusion-audit` branch: 4 unmerged commits on project, pending merge to main
- Main test count: 675 (small issues only; #186 tests are on the epic branch)

## Immediate Next

**Merge `epic-exclusion-audit` to main** — 4 commits, 685 tests passing. No Flyway migrations. No conflicts expected.

Then: casehub-clinical Epic 4 (adverse event escalation) or engine#187 (SelectionContext 8th arg).

## Open / Next

| Priority | What |
|---|---|
| **merge now** | `epic-exclusion-audit` → main (685 tests, branch deletion due 2026-06-02) |
| next | casehub-clinical Epic 4 — adverse event escalation |
| engine | casehub-engine #187 — SelectionContext call sites need 8th arg (excludedUsers) |
| minor | #193 — `instantiate()` exclusion gap (assigneeId override against template excludedUsers — may already be covered by create() check; needs a test) |
| deferred | #192 — `CREATE_DENIED` audit; requires pre-generating WorkItem ID before exclusion check; orphaned audit entry semantics to resolve |
| blocked | casehubio/work#97 — event mesh (blocked on qhorus#131, #132) |
| debt | #182 WorkItemCreateRequest builder (24-param record unwieldy); #191/#91/#169 persistence-memory/ splits |

## Key References

- Blog: `blog/2026-05-19-mdp01-policy-owns-the-reason.md`
- Spec: `docs/specs/2026-05-19-exclusion-policy-audit-design.md`
- Garden: GE-20260519-28275c (REQUIRES_NEW must return normally), GE-20260519-23b704 (guard ordering), GE-20260519-685b4b (UNIQUE constraint test isolation)
