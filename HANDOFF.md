# HANDOFF — 2026-06-10

## Last Session

Closed 5 multi-tenancy issues on branch `issue-261-multi-tenancy-fixes`. #261 (test fixes), #260 (already done), #259 (protocol update), #258 (webhook tenant mapping via path segment), #257 (PostgreSQL RLS hardening — TenantAwareStore base class, WorkRlsPolicyApplicator, 26 stores migrated, module applicators, Testcontainers enforcement test). 7 commits, 56 files changed, 1541 insertions.

## Immediate Next Step

Run `work-end` to close branch `issue-261-multi-tenancy-fixes`. All code is committed and pushed to origin. The close needs: pre-close sweep (journal entry, arc42 scan, diary), artifact routing, journal merge, squash, rebase onto main, fork push, blessed repo delivery.

## What's Left

- `work-end` for `issue-261-multi-tenancy-fixes` — close the branch · XS · Low
- Garden push failed (pre-push hook) — 2 entries committed locally, need push · XS · Low
- `RlsEnforcementTest` requires Docker (Testcontainers) — not available on this machine; verify in CI · XS · Low
- Pre-existing CDI augmentation issue — full `mvn clean test -pl runtime` fails due to stale core jar; targeted tests pass · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #240 | design: human task lifecycle alignment | L | High | now critical |
| #236 | feat: replace VocabularyScope enum with Path-based scope hierarchy | M | Low | |
| #253 | feat: MongoDB store — complete drop-in replacement | M | Med | |

## Key References

- Spec (RLS): `specs/2026-06-10-postgresql-rls-hardening-design.md` (rev 5, 4 review rounds)
- Garden: GE-20260610-204ddb (stale Jandex CDI), GE-20260610-f990b2 (REQUIRES_NEW test isolation)
- ADRs: 0008–0010 (multi-tenancy from #256)
- Protocols: `docs/protocols/casehub/` — 3 multi-tenancy rules (updated #259)
- Previous refs: `git show HEAD~1:HANDOFF.md`
