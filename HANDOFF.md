# HANDOFF — 2026-06-27

## Last Session

Closed #275 — extracted complete WorkItemCreator SPI into casehub-work-api.
Four value types moved to api (WorkItemPriority, WorkItemLabelRequest, WorkItemStatus,
WorkItemCreateRequest + tenancyId). Two SPI interfaces (WorkItemCreator, WorkItemLifecycle)
with hexagonal WorkItemSpiAdapter. Template creation unified via createFromTemplate() —
fixes post-mutation corruption where lifecycle events fired with stale values.
WorkItemLifecycleEvent implements WorkItemEvent for entity-free CDI observation.
Indexed findActiveByCallerRef + V38 caller_ref index. Garden entry GE-20260627-8d321f.

## Immediate Next Step

Engine work-adapter migration: engine#578 — shift compile dep from casehub-work to
casehub-work-api. All API types and SPIs are published. Run from the engine session.

## Cross-Module

**We're unblocking:**
- `casehub-desiredstate#43` — SimpleTransitionExecutor can now create WorkItems for requiresHuman=true nodes via WorkItemCreator SPI
- `engine#578` — work-adapter compile dep can shift to casehub-work-api

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#578 | Migrate work-adapter from casehub-work to casehub-work-api | M | Low | all types/SPIs ready, mechanical migration |
| #276 | Migrate existing SPIs to api/spi/ subpackage | S | Low | mechanical move |
| #277 | Consolidate WorkItemTemplateService.instantiate() through create() | M | Med | REST path alignment |
| #278 | Align WorkLifecycleEvent with WorkItemEvent interface | M | Med | filter engine impact |
| parent#314 | Update casehub-work deep-dive for SPI | S | Low | doc sync |
| #152 | Split casehub-work-examples into core and full variants | M | Low | standalone |

## Key References

- Spec: `docs/specs/2026-06-26-workitem-creator-spi-design.md` (revision 4)
- Previous refs: `git show HEAD~1:HANDOFF.md`
