# HANDOFF — 2026-06-18

## Last Session

Closed #267 (CrossTenantProducer hardcodes JPA types) and #269 (hardcoded
database name in 4 Mongo stores). Both were persistence-mongodb follow-ups
from #253. Garden entry GE-20260618-451d25 on the Panache `mongoCollection()`
`@GenerateBridge` type covariance gotcha.

## Immediate Next Step

#240 is next — L/High design work (human task lifecycle alignment). This is
not a code fix. Read the issue, understand the scope, then brainstorm before
any implementation.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #240 | design: human task lifecycle alignment | L | High | now critical — states, engine orchestration, spec gaps |

## Key References

- #267 spec: `docs/specs/2026-06-17-crosstenant-producer-mongodb-fix-design.md`
- #269 spec: `docs/specs/2026-06-18-hardcoded-db-name-fix-design.md`
- Blog: `blog/2026-06-17-mdp01-the-producer-that-forgot.md`
- Garden: GE-20260618-451d25 (Panache mongoCollection() @GenerateBridge covariance)
- Previous refs: `git show HEAD~1:HANDOFF.md`
