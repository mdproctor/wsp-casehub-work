# HANDOFF — 2026-06-02

## Last Session

Re-applied `update-claude-md` modular detection fix in cc-praxis (`issue-109` branch closed) — another Claude session had reverted it. Also: closed cc-praxis `issue-109-write-content-restructure`. New commits from parallel session landed on casehub/work main: WorkItemCallerRef utility, CI dispatch chain update, HttpWebhookChannel boundary doc.

*Previous session (2026-06-01):* Closed `issue-235-sxs-sweep` — all sweep items done except #234 (blocked). Full build green: 900+ tests, three PostgreSQL ITs, 25 native ITs. Fixes: CDI ambiguity (#247, #248), broadcaster flyway.locations.

## Immediate Next Step

**`#246`** — ARC42STORIES.MD migration (`DESIGN.md` + `ARCHITECTURE.md`).
Not yet started. Run `/work` to begin.

## Cross-Module

**Blocked by:**
- `casehub-platform` — needs Jandex in `casehub-platform-api` so the ledger identity enricher exclusion workaround can be removed and the no-ops are properly discovered. Text drafted for filing. Protocol PP-20260601-37179a captures the rule.

## What's Left

- `#234` — blocked (connectors-core not built, Qhorus routing conflict, classification undefined); documented and labeled · S · High
- ledger exclude-types workaround — temporary; remove when casehub-platform refreshes · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #246 | Migrate DESIGN.md + ARCHITECTURE.md → ARC42STORIES.MD | L | Med | — |
| parent#66 | Apply CLAUDE.md size discipline to remaining casehubio repos | L | Low | casehub-work is reference impl |

## Key References

- Garden: GE-20260601-b76fba (QuarkusTestResource cannot override build-time fixed properties)
- Garden: GE-20260601-7a3b38 (DefaultBean invisible when supertype JAR missing Jandex / stale SNAPSHOT)
- Protocol: PP-20260601-37179a (library JARs that ship CDI beans must include Jandex)
- Blog: `2026-06-01-mdp01-the-build-that-kept-giving.md`
- Platform issue text: drafted in session — file against casehubio/platform
