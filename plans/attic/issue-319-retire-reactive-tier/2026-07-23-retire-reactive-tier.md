# Retire Reactive Tier — casehub-work (#319)

**Focal issue:** #319

## Scope Assessment

casehub-work was **already blocking-first** — it uses `quarkus-hibernate-orm-panache` (blocking JPA),
`quarkus-jdbc-postgresql`, and has zero `Reactive*` source files. All `Reactive*` SPIs, implementations,
and handler classes found during initial analysis were in the **engine** repo (indexed by IntelliJ
because both projects share an IDE instance).

The actual work-repo scope is confined to `engine-adapter/`:

## Changes Made

### WorkItemLifecycleAdapter (inbound bridge: WorkItem events → PlanItem transitions)
- `ReactiveCrossTenantCaseInstanceRepository` → `CrossTenantCaseInstanceRepository` (blocking variant already exists in engine JAR)
- Removed `.await().atMost(TIMEOUT)` from two `findByUuid()` call sites — now direct blocking calls
- Removed unused `TIMEOUT` field and `Duration` import

### 3 event handlers (outbound bridge: engine events → WorkItem creation)
- `ActionGateWorkItemHandler`: `blocking = true` → `@RunOnVirtualThread`
- `HumanTaskScheduleHandler`: `blocking = true` → `@RunOnVirtualThread`
- `ActionGateCancelledHandler`: `blocking = true` → `@RunOnVirtualThread`

All three already had `void` return types and `@Transactional` — the only change is the thread dispatch model.

## What Stays Reactive (by design)
- `postgres-broadcaster/` — LISTEN/NOTIFY, genuinely event-driven SSE
- `queues-postgres-broadcaster/` — same
- `flow/` module — `Uni<String>` return in `HumanTaskFlowBridge` for Quarkus-Flow async workflow suspension

## Pre-existing Issues
- engine-adapter tests have 21 CDI deployment failures (unsatisfied `WorkItemCreator` dependency) — confirmed identical on main before this branch
