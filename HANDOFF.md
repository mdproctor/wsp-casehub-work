# HANDOFF — 2026-06-09

## Last Session

Multi-tenancy implementation for casehub-work (#256). 28 commits on `issue-256-multi-tenancy-tenantid`: `tenancy_id` on every entity (5 Flyway migrations), 25 store implementations (16 new tenant-scoped + 6 existing extended + 3 cross-tenant), CDI events with tenancyId, tenant-scoped SSE streams, polling schedulers replaced with per-item Quartz timers (18-transition lifecycle matrix), TenantContextRunner for async paths, @CrossTenant qualifier pattern matching engine. Spec at revision 6 after 5 review rounds. Code review passed — 2 findings fixed (SpawnGroupResource isolation gap, ReportService @CacheResult tenant leak). Protocol captured: PP-20260607-69eba2 (tenancyid-server-side-only). 3 out-of-scope issues filed: #257 (RLS), #258 (webhook tenant mapping), #259 (protocol update). CI fix: 3 commits on main syncing casehub-work-ledger with upstream casehub-ledger tenancy-aware API (ledger#132 filed for @CrossTenant CDI issue). #256 closed on GitHub. Diary entry written.

## Immediate Next Step

Run `work-end` to complete the close. The branch is **already rebased onto project main** — the rebase step is done. The project repo is on `main` with all 28 branch commits. Remaining work-end steps:

1. `/git-squash upstream/main..HEAD` — 28 commits need compaction before push
2. Push to origin/main (fork) then upstream/main (blessed)
3. Mark closed (EPIC-CLOSED.md on workspace epic branch)
4. Return workspace to main

State: project on `main` (post-rebase), workspace on `issue-256-multi-tenancy-tenantid`.

## Pending Sweeps

3 forage entries identified but not yet submitted — run `forage CAPTURE` for each:

1. **gotcha (score 10):** @RequestScoped CDI bean silently displaces @DefaultBean @ApplicationScoped in all request contexts. A plain @RequestScoped bean (not @Alternative, not @DefaultBean) wins CDI resolution over a @DefaultBean @ApplicationScoped bean whenever request scope is active. In casehub-work, TenantScopedPrincipal (@RequestScoped) displaces MockCurrentPrincipal (@DefaultBean @ApplicationScoped) in REST request contexts — not just TenantContextRunner contexts. The defaults on the @RequestScoped bean must match the displaced @DefaultBean's behavior, or existing tests break silently with wrong tenancyId/actorId values. Stack: Quarkus 3.32, ArC CDI. Domain: jvm.

2. **gotcha (score 12):** Quarkus @CacheResult cache key omits CDI-injected fields — tenant data leak. @CacheResult derives cache keys from method parameters only. If a method injects CurrentPrincipal via CDI and reads tenancyId() from it (not a method parameter), two tenants with identical query parameters get the same cached result. This is a silent cross-tenant data leak through the cache layer. Fix: either make tenancyId an explicit method parameter, use a custom @CacheKeyGenerator, or remove caching. Stack: Quarkus 3.32, quarkus-cache. Domain: jvm.

3. **gotcha (score 9):** Upstream Maven SNAPSHOT breaking change breaks downstream CI silently — local build passes. When an upstream dependency publishes a new SNAPSHOT with changed method signatures, downstream CI resolves the new SNAPSHOT and breaks. Local builds pass because .m2/repository has a locally-installed version. The mismatch is invisible until CI runs. Root cause: `_remote.repositories` shows the local jar has no remote origin. Stack: Maven 3.x, GitHub Packages. Domain: jvm.

Protocol sweep: PP-20260607-69eba2 already captured. No other new protocols identified.
ADR: no new architectural decisions beyond what's in the spec.

## Cross-Module

**Blocked by:**
- `casehub-ledger` — ledger#132: @CrossTenant beans in published SNAPSHOT break downstream @QuarkusTest contexts. casehub-work-ledger tests fail in CI until resolved.

## What's Left

- `#234` — still blocked (connectors-core not built, Qhorus routing conflict, classification undefined) · S · High
- ledger exclude-types workaround — still temporary · XS · Low
- 6 workspace branches past deletion date · XS · Low
- ledger#132 — upstream @CrossTenant CDI issue blocking casehub-work-ledger tests · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #240 | design: human task lifecycle alignment | L | High | #245 DELEGATED done without it — now critical |
| #255 | chore(persistence-memory): minor review findings | XS | Low | removed from #256 COVERS |
| #251 | OutcomeCodecs / PATCH template: minor quality gaps | XS | Low | removed from #256 COVERS |
| #236 | feat: replace VocabularyScope enum with Path-based scope hierarchy | M | Low | |

## Key References

- Spec: `specs/issue-256-multi-tenancy-tenantid/2026-06-08-multi-tenancy-design.md` (revision 6)
- Plan: `plans/attic/issue-256-multi-tenancy-tenantid/2026-06-09-multi-tenancy-implementation.md`
- Protocol: PP-20260607-69eba2 (tenancyid-server-side-only) in casehub/garden
- Diary: `blog/2026-06-09-mdp01-every-query-every-entity.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
