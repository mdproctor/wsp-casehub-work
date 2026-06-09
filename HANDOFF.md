# HANDOFF — 2026-06-10

## Last Session

Closed branch `issue-256-multi-tenancy-tenantid` — full multi-tenancy across casehub-work. 197 files, 8 commits (squashed from 31), V35 migration, app-level tenant filtering on all stores, TenantContextRunner for async paths, @CrossTenant CDI qualifier. Closed #256, #255, #251. Pushed to both origin and upstream. Filed #261 for 12 pre-existing test failures (WorkItemTemplate.tenancyId null on direct Panache persist).

## Immediate Next Step

Pick up #261 (fix 12 test classes — set tenancyId before persist or use store put()). Run `/work` to start. XS/Low — mechanical fix, clear pattern.

## What's Left

- `#261` — 12 test classes fail: WorkItemTemplate.tenancyId null on direct Panache persist() · S · Low
- `#234` — still blocked (connectors-core not built) · S · High
- ledger exclude-types workaround — still temporary · XS · Low
- 6 workspace branches past deletion date · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #240 | design: human task lifecycle alignment | L | High | #245 DELEGATED done without it — now critical |
| #257 | feat: PostgreSQL RLS hardening for multi-tenancy | M | Med | defence-in-depth for #256 |
| #236 | feat: replace VocabularyScope enum with Path-based scope hierarchy | M | Low | |
| #253 | feat: MongoDB store — complete drop-in replacement | M | Med | |

## Key References

- Garden: GE-20260609-dac1a3 (@CacheResult tenant leak), GE-20260609-49bd08 (static Panache bypass), GE-20260609-a23a8b (context nesting)
- ADRs: 0008 (app-level filtering), 0009 (TenantContextRunner), 0010 (@CrossTenant qualifier)
- Protocols: `docs/protocols/casehub/` — 3 multi-tenancy rules
- Blog: `2026-06-09-mdp01-every-query-is-a-question-of-trust.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
