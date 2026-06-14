# HANDOFF — 2026-06-14

*Updated: engine#468 + engine#469 closed — casehub-engine-inbound merged (engine PR #492).*

## Last Session

**Cross-repo (from casehub-iot session 2026-06-14):** Created `docs/repos/casehub-iot.md` deep-dive (iot#15, closed). Updated PLATFORM.md iot testing entry with YAML fixtures + DeviceTypeHandler SPI (parent#237, closed). Both on branch `issue-235-p0-layering-decisions`.

**Previous (2026-06-13):** Implemented #236 — replaced VocabularyScope enum with Path-based scope hierarchy. Five rounds of design review before implementation. The refactoring exposed and fixed: ownerId redundancy (compound key collapsed to single Path), findGlobalVocabulary multi-tenant bug (seeded row locked to default tenant), and a TOCTOU race in findOrCreateVocabulary (fixed with three-layer store SPI: findByScope + UNIQUE constraint + REQUIRES_NEW findOrCreate). Filed #263 (FilterScope — same violation in queues module). Closed #236.

## Immediate Next Step

No trailing work. Pick from What's Next.

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #263 | feat: replace FilterScope enum with Path-based hierarchy (queues) | M | Low | same pattern as #236 |
| #240 | design: human task lifecycle alignment | L | High | now critical |
| #253 | feat: MongoDB store — complete drop-in replacement | M | Med | |

## Key References

- Garden: GE-20260613-8845fa (flyway_scan.py cross-module V-number gotcha)
- Blog: `2026-06-13-mdp01-the-enum-that-was-always-a-path.md`
- Spec: `docs/specs/2026-06-12-vocabulary-scope-path-design.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
