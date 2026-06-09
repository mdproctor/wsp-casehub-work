# HANDOFF — 2026-06-09

## Last Session

Multi-tenancy implementation for casehub-work (#256). 25 commits on `issue-256-multi-tenancy-tenantid`: `tenancy_id` on every entity (5 Flyway migrations), 25 store implementations (16 new tenant-scoped + 6 existing extended + 3 cross-tenant), CDI events with tenancyId, tenant-scoped SSE streams, polling schedulers replaced with per-item Quartz timers (18-transition lifecycle matrix), TenantContextRunner for async paths, @CrossTenant qualifier pattern matching engine. Spec at revision 6 after 5 review rounds. Code review passed — 2 findings fixed (SpawnGroupResource isolation gap, ReportService @CacheResult tenant leak). Protocol captured: PP-20260607-69eba2 (tenancyid-server-side-only). 3 out-of-scope issues filed: #257 (RLS), #258 (webhook tenant mapping), #259 (protocol update).

## Immediate Next Step

Complete work-end for `issue-256-multi-tenancy-tenantid`. The branch is implementation-complete but work-end was interrupted mid-procedure. Resume with: pre-close sweep (forage, protocol, ADR, diary), then Steps 2–10 of work-end. Two pending forage entries to submit: (1) @RequestScoped CDI bean displaces @DefaultBean in all request contexts, (2) @CacheResult cache key missing CDI-injected tenancyId = tenant data leak. #255 and #251 removed from COVERS — close only #256.

## Cross-Module

*Unchanged — retrieve with: `git show HEAD~1:HANDOFF.md`*

## What's Left

- `#234` — still blocked (connectors-core not built, Qhorus routing conflict, classification undefined) · S · High
- ledger exclude-types workaround — still temporary · XS · Low
- 6 workspace branches past deletion date · XS · Low
- 3 forage entries pending — run `forage SWEEP` with these descriptions:
  1. gotcha (score 10): @RequestScoped CDI bean silently displaces @DefaultBean @ApplicationScoped in all request contexts — defaults must match or tests break silently
  2. gotcha (score 12): Quarkus @CacheResult cache key omits CDI-injected tenancyId — cross-tenant data leak through cache
  3. gotcha (score 9): Upstream Maven SNAPSHOT breaking change breaks downstream CI silently — local .m2 has locally-installed jar, CI resolves published version with different signatures

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #240 | design: human task lifecycle alignment | L | High | #245 DELEGATED done without it — now critical |
| #255 | chore(persistence-memory): minor review findings | XS | Low | removed from #256 COVERS |
| #251 | OutcomeCodecs / PATCH template: minor quality gaps | XS | Low | removed from #256 COVERS |
| #236 | feat: replace VocabularyScope enum with Path-based scope hierarchy | M | Low | |

## Key References

- Spec: `specs/issue-256-multi-tenancy-tenantid/2026-06-08-multi-tenancy-design.md` (revision 6)
- Plan: `plans/2026-06-09-multi-tenancy-implementation.md`
- Protocol: PP-20260607-69eba2 (tenancyid-server-side-only) in casehub/garden
- Previous refs: `git show HEAD~1:HANDOFF.md`
