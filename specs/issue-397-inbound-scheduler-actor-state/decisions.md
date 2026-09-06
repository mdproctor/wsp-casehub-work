## D1: Module placement

**Choice:** Both `InboundWorkItemScheduler` implementation and `WorkActorStateContributor` in `casehub-work-engine-adapter`, package `io.casehub.work.engine`
**Alternatives:**
- Dedicated new module — unnecessary; both are engine-integration concerns and the adapter already bridges engine and work
- work-runtime — wrong ownership; these implement engine-defined SPIs, not work-internal concerns
**Rationale:** The adapter already contains `HumanTaskScheduleHandler` (same pattern as the scheduler) and `CallerRef.parse()` (needed by the contributor). Both SPIs are defined in engine repos (engine-common, platform-api). The adapter is the natural home.
**Trade-offs:** None significant — the adapter already depends on all required modules.
**Sources:** engine-adapter/pom.xml (existing deps), HumanTaskScheduleHandler.java (pattern), CallerRef.java (caseId parsing)
**Exploration:** quick
**Status:** captured

## D2: InboundWorkItemScheduler dependencies and tenant context

**Choice:** Inject `WorkItemCreator` (work-api SPI) + `TenantContextExecutor` (work-api SPI). Wrap creation in `tenantContextExecutor.runInTenantContext(request.tenancyId(), ...)`.
**Alternatives:**
- Direct `WorkItemService` + `TenantContextRunner` — adds casehub-work as compile dep; valid (runtime always present) but not needed for this issue
- Self-sufficient service (no wrapping) — better long-term design but bigger scope; filed as #399
**Rationale:** Consistent with `HumanTaskScheduleHandler` pattern (injects WorkItemCreator). TenantContextExecutor wrapping follows async-event-tenant-context-propagation protocol. Filed #399 for the self-sufficient refactoring.
**Trade-offs:** Caller still wraps in TenantContextExecutor — accepted debt, tracked by #399.
**Sources:** HumanTaskScheduleHandler.java (pattern), TenantContextRunner.java (TenantContextExecutor SPI), InboundWorkItemBridge.java (caller context — afterCompletion callback, outside request scope)
**Exploration:** deep-analysis
**Status:** captured

## D3: Priority string-to-enum mapping

**Choice:** Parse `InboundWorkItemRequest.priority()` (String) to `WorkItemPriority` via `WorkItemPriority.valueOf()`, null-safe. Invalid values propagate `IllegalArgumentException`.
**Alternatives:**
- Default to MEDIUM on parse failure — masks caller bugs silently
- String passthrough — WorkItemCreateRequest.priority is WorkItemPriority enum, not String; type mismatch
**Rationale:** Fail-fast on invalid priority is consistent with WorkItemCreateRequest validation. Null priority is valid (defaults to MEDIUM in WorkItemService.create() at line 176).
**Trade-offs:** Callers sending an unrecognised priority string get an exception — this is correct behaviour.
**Sources:** WorkItemCreateRequest.java:15 (priority field type), WorkItemService.java:176 (MEDIUM default), InboundWorkItemRequest.java:41 (String priority)
**Exploration:** quick
**Status:** captured

## D4: WorkActorStateContributor query and caseId extraction

**Choice:** Query via `WorkItemStore.scan()` with `WorkItemQuery.builder().assigneeId(actorId).statusIn(List.of(ASSIGNED, IN_PROGRESS, SUSPENDED)).build()`. Extract caseId via `CallerRef.parse(wi.callerRef())` — returns sealed type with `caseId()` method, or null for non-engine callerRefs. `sourceName()` returns `"work"`.
**Alternatives:**
- Use `WorkItemQuery.inbox()` factory — adds candidateGroups/candidateUserId params we don't need (pass null); builder is cleaner
- Skip caseId extraction — accumulator.workItem() requires caseId parameter; null is valid for non-engine items but engine items should have it
**Rationale:** Matches the original actor-state-view spec's intent. ASSIGNED/IN_PROGRESS/SUSPENDED captures active work (excludes PENDING — actor hasn't claimed yet). CallerRef.parse() is already in the same module — no new dependency, type-safe, handles both PlanItemRef and GateRef formats.
**Trade-offs:** PENDING items excluded from actor state — actor is a candidate but hasn't accepted. This is intentional per the original spec.
**Sources:** actor-state-view design spec (WorkActorStateContributor section), CallerRef.java (sealed interface with caseId()), WorkItemQuery.java:42-83 (builder and inbox factory)
**Exploration:** quick
**Status:** captured
