# HANDOFF — 2026-06-29

## Last Session

Closed #276, #277, #278 — full api surface cleanup. Event hierarchy aligned:
WorkItemEvent promoted as primary typed interface, WorkLifecycleEvent deleted,
source():Object replaced with typed workItem():WorkItem. FilterAction typed
from Object to WorkItem. 14 SPI interfaces moved to api/spi/. Template creation
unified through createFromTemplate() — instantiate() and toCreateRequest() deleted.
3 missed source() calls found during full build verification (issue-tracker,
queues-dashboard, postgres-broadcaster) — fixed post-squash. Garden entries:
GE-20260521-3e030b revised (variant), GE-20260629-d6deca (new).

## Immediate Next Step

engine#585 — migrate WorkItemLifecycleAdapter from @ObservesAsync WorkLifecycleEvent
to @ObservesAsync WorkItemEvent. Run from the engine session. Prerequisite for
consuming the updated casehub-work.

## Cross-Module

**We're unblocking:**
- `engine#585` — observer type migration (WorkLifecycleEvent deleted)
- `casehub-desiredstate#43` — WorkItemCreator SPI for requiresHuman=true nodes

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#585 | Migrate work-adapter observer from WorkLifecycleEvent to WorkItemEvent | S | Low | engine session — mechanical |
| parent#314 | Update casehub-work deep-dive for SPI restructuring | S | Low | doc sync |
| #279 | Evaluate WorkItemGroupLifecycleEvent hierarchy integration | S | Med | design evaluation |
| #152 | Split casehub-work-examples into core and full variants | M | Low | standalone |

## Key References

- Spec: `docs/specs/2026-06-27-api-surface-cleanup-design.md`
- Previous refs: `git show HEAD~1:HANDOFF.md`
