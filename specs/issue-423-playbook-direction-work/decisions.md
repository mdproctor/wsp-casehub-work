## D1: How to derive actor/claimant from principal

**Choice:** Remove actor/claimant params from user-facing SPIs (Approach A)
**Alternatives:**
- @ContextParam("actorId") annotation — requires platform-api code generator change, misleading SPI params remain
- Override in implementation only — dishonest contract, generated REST still accepts useless query params
**Rationale:** SPIs are internal to the work module (zero external implementors confirmed across all casehub repos). Removing the params makes the contract honest — identity comes from exactly one place. Test updates are mechanical.
**Trade-offs:** Larger diff across 16 method signatures in WorkItemLifecycleApi; every direct SPI test caller needs updating
**Sources:** WorkItemLifecycleApi (api/src/main/java), DefaultWorkItemLifecycleApi (rest/src/main/java), WorkLifecycleResource (generated), grep across ~/claude/casehub/* for external implementors
**Exploration:** quick
**Status:** captured

## D2: BulkOperationRequest.actorId field

**Choice:** Remove the field entirely
**Alternatives:**
- Deprecate (keep field, ignore at runtime) — softer migration but adds ambiguity
**Rationale:** Pre-release stage, no backward compat needed. Clean break avoids confusion about whether the field does anything.
**Trade-offs:** Any existing client sending actorId in bulk requests will get a deserialization warning or error (depending on Jackson config). Acceptable at pre-release.
**Sources:** BulkOperationRequest (api/src/main/java), DefaultWorkItemBulkApi (rest/src/main/java)
**Exploration:** quick
**Depends on:** D1 (actor from principal)
**Status:** captured
