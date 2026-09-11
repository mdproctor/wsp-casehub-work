# Decisions — #399 Remove TenantContextExecutor SPI

## D1: Cross-repo coordination approach

**Choice:** Add `createInTenantContext(String tenancyId, WorkItemCreateRequest request)` to `WorkItemOperations` SPI. Engine bridge drops `TenantContextExecutor` and calls the combined method. Work repo deletes `TenantContextExecutor`.

**Alternatives:**
- New `TenantAwareWorkItemCreator` SPI — cleaner separation but adds SPI count where we're trying to reduce it
- Route engine bridge through `InboundWorkItemScheduler` — semantic mismatch (scheduler is for engine-planning path, not message bridge), `InboundWorkItemRequest` is narrower than `WorkItemCreateRequest`

**Rationale:** Smallest change surface: one `default` method added to an existing SPI the engine bridge already injects, one SPI deleted. Boundary crossing remains explicitly visible (separate `tenancyId` parameter, not sourced from request fields — satisfying the compliance audit trail requirement). Follows the existing `WorkItemCreator.createMultiInstance()` pattern of `default` methods with `UnsupportedOperationException`.

**Trade-offs:** Mixes tenant infrastructure concern into the business operations SPI. Accepted because the alternative (keeping a separate SPI) is the exact problem we're solving.

**Sources:**
- `api/src/main/java/io/casehub/work/api/spi/TenantContextExecutor.java` — current SPI (single method)
- `api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java` — target SPI
- `runtime/src/main/java/io/casehub/work/runtime/service/TenantContextRunner.java` — implementation
- `engine-adapter/src/main/java/io/casehub/work/engine/InboundWorkItemSchedulerImpl.java` — only work-repo consumer
- Engine repo `InboundWorkItemBridge.java` commit `066878ef` (engine#974) — introduced engine dependency on `TenantContextExecutor`
- Protocol PP-20260609-fb6563 — confirms TenantContextRunner is canonical
- Issue #399 — rejected alternatives (auto-establish from request fields)
- Memory: `feedback_compile_time_over_wrappers` — if runtime always present, code against implementation directly

**Exploration:** quick
**Status:** captured
