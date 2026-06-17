# HANDOFF — 2026-06-17

## Last Session

Completed #253: persistence-mongodb is now a true drop-in replacement for JPA.
Implemented all 11 remaining store SPIs (note, template, schedule, link, spawn group,
relation, vocabulary, definition, filter rule, routing cursor, issue link). OCC via
findOneAndUpdate for schedule and spawn group. Atomic cursor with $inc + floorMod.
5 unique compound indexes at startup. 122 tests passing. Code review approved.

Original issue scoped 3 stores — design review expanded to 11 after identifying
4 functional couplings and 4 missing stores the issue hadn't listed.

Filed #267 (CrossTenantProducer hardcodes JPA types — data integrity risk with
MongoDB active) and #269 (hardcoded database name in 4 stores). Garden entry
GE-20260617-daa4cb on $setOnInsert + $inc same-field conflict.

## Immediate Next Step

Branch is closed and pushed. Pick from What's Next.

## What's Left

- #267: CrossTenantProducer hardcodes JPA types — cleanup job and scheduler silently break with MongoDB · S · Med
- #269: hardcoded `"workitems"` database name in 4 stores · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #240 | design: human task lifecycle alignment | L | High | now critical |

## Key References

- MongoDB completion: #253 (closed — 11 stores, 42 files, 4342 insertions)
- CrossTenantProducer: #267 (open — data integrity when MongoDB active)
- Hardcoded DB name: #269 (open — 4 stores use mongoClient.getDatabase("workitems"))
- Spec: `specs/2026-06-17-mongodb-persistence-completion-design.md`
- Plan: `plans/2026-06-17-mongodb-persistence-completion.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
