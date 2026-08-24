## D1: Scope includes both PlanItem and ActionGate

**Choice:** Both HumanTask PlanItem dispatch/lifecycle and ActionGate approval/rejection are in scope for the distributed CloudEvent bridge.
**Alternatives:**
- PlanItem only, ActionGate deferred — simpler first pass but incomplete distributed story
**Rationale:** Both use the same callerRef pattern (`case:{caseId}/pi:{planItemId}` vs `case:{caseId}/gate:{gateId}`), the same CloudEvent transport, and the same module. Handling both in one module avoids a second pass through the same architecture.
**Trade-offs:** Larger scope for an already L-sized issue. Mitigated by the shared infrastructure — the ActionGate path is incremental once PlanItem works.
**Sources:** `WorkItemLifecycleAdapter` (decompiled — routes on CallerRef type), `ActionGateCompletionApplier` (decompiled), `EventBusAddresses.java` (action gate addresses)
**Exploration:** quick
**Status:** captured

## D2: Engine-only changes — no work repo modifications

**Choice:** All changes stay within the engine repo. The new engine module reimplements PlanItem/ActionGate completion logic using engine SPIs rather than extracting from `casehub-work-engine-adapter`.
**Alternatives:**
- Extract `PlanItemCompletionApplier` to engine, update work-engine-adapter to delegate — cleaner but requires cross-repo coordination
- Shared SPI in engine-common — adds abstraction for a single consumer
**Rationale:** Avoids cross-repo PR coordination. The completion logic is straightforward (~100 lines) and uses engine SPIs directly (`BlackboardRegistry`, `PlanItem`, `CrossTenantCaseInstanceRepository`, `BridgeResolver`). Two implementations are acceptable at pre-release — consolidation can follow (tracked in engine#974).
**Trade-offs:** Duplication of completion logic between work-engine-adapter and the new engine module. Both use the same underlying SPIs so they naturally stay aligned.
**Sources:** `PlanItemCompletionApplier.class` (decompiled), `HumanTaskScheduler.java:25`, `HumanTaskScheduleRequest.java:26`
**Exploration:** deep-analysis
**Status:** captured

## D3: Depend on casehub-work-api for CloudEvent type constants

**Choice:** The new engine module depends on `casehub-work-api` (Tier 1) for `WorkCloudEventTypes` and `WorkItemStatus`.
**Alternatives:**
- Hardcode CloudEvent type strings — zero coupling but risk of drift
**Rationale:** `casehub-work-api` is a pure Tier 1 artifact with no engine dependency. One-way dependency, same pattern as `engine-api → casehub-worker-api`. Doesn't worsen the existing engine ↔ work repo-level cycle (which is at the runtime/adapter level, tracked in engine#974).
**Trade-offs:** Adds one more edge from engine → work-api. Acceptable because the edge is one-way and the cycle is already present via other modules.
**Sources:** `WorkCloudEventTypes.class` (decompiled), engine root `pom.xml` (already declares `casehub-work-api` in dependencyManagement)
**Exploration:** deep-analysis
**Status:** captured

## D4: New engine module for distributed CloudEvent bridge

**Choice:** New module `casehub-engine-work-cloudevent` (or similar short name per maven-submodule-folder-naming protocol). Optional, activated by classpath presence.
**Alternatives:**
- Engine runtime — couples CloudEvent SDK to core even when not needed
- Engine common SPI + module impl — over-abstracted for a single implementation
**Rationale:** Follows the established pattern of optional engine modules (`engine-a2a`, `engine-mcp`, `engine-react`, `engine-flow`). Co-located mode (work-engine-adapter) and distributed mode (this module) are mutually exclusive deployment choices.
**Trade-offs:** One more module to maintain. Offset by clean separation — consumers that don't need distributed work don't pay for CloudEvent SDK on the classpath.
**Sources:** `a2a/pom.xml`, `mcp/pom.xml`, `react/pom.xml` (module patterns)
**Exploration:** quick
**Status:** captured
