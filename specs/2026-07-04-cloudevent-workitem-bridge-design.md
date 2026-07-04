# CloudEvent WorkItem Bridge — Design Spec

**Issue:** casehubio/work#172
**Date:** 2026-07-04
**Status:** Approved

## Problem

casehub-work is only addressable via REST API or in-JVM POJO calls (`WorkItemCreator`, `WorkItemService`). External systems that communicate via CloudEvents cannot request human work without coupling to casehub-work's Java API.

work#273 shipped the outbound direction — `WorkCloudEventAdapter` fires CDI `CloudEvent` on every WorkItem lifecycle transition. The inbound direction (CloudEvent → WorkItem creation) is missing.

## Design Principles

1. **CloudEvents are the inter-system boundary.** Within casehub, internal consumers use POJOs (`WorkItemLifecycleEvent`, `WorkItemCreateRequest`). CloudEvents are the wire format at the edge.
2. **Domain data flows through opaquely.** casehub-work never forces a schema on the caller's payload. Raw JSON passes from CloudEvent data to WorkItem `payload` untouched. Zero deserialization of domain content.
3. **Optional schema validation via template.** When a `WorkItemTemplate` declares an `inputDataSchema` (JSON Schema draft-07), casehub-work validates the payload at creation time. This is end-user configurable via the template API. No Java class coupling — validation is language-agnostic.

## CloudEvent Contract

### Inbound Type

`io.casehub.work.workitem.requested`

### Envelope Mapping

| CloudEvent field | Maps to |
|---|---|
| `type` | `io.casehub.work.workitem.requested` (constant) |
| `source` | Caller-defined URI (e.g. `/workflows/doc-approval`) — incorporated into `createdBy` for provenance |
| `id` | Caller-generated UUID — becomes WorkItem `callerRef` for correlation and idempotency |
| `tenancyid` ext | Tenant ID (required — missing rejects the event) |
| `templateid` ext | Template UUID or name (required) — resolved via `WorkItemTemplateService.findByRef()` |
| `data` | Domain payload — opaque JSON, stored as WorkItem `payload` |

### Outbound Types (formalise existing)

Already shipped via `WorkCloudEventAdapter` (#273). The full `WorkEventType` enum maps to CloudEvent types:

`io.casehub.work.workitem.created`, `.completed`, `.rejected`, `.cancelled`, `.expired`, `.assigned`, `.started`, `.delegated`, `.escalated`, etc.

### Correlation

The CloudEvent `id` on the request becomes the WorkItem's `callerRef`. Outbound lifecycle CloudEvents carry `callerRef` in the data payload, so the original requester can match request → response.

## Type Constants — `WorkCloudEventTypes`

Location: `casehub-work-api` (pure Java, no framework deps).

```java
public final class WorkCloudEventTypes {
    public static final String PREFIX = "io.casehub.work.workitem.";
    public static final String REQUESTED = PREFIX + "requested";
    public static final String CREATED = PREFIX + "created";
    public static final String COMPLETED = PREFIX + "completed";
    public static final String REJECTED = PREFIX + "rejected";
    // ... one per WorkEventType enum value

    public static final String GROUP_PREFIX = "io.casehub.work.group.";

    public static final String EXT_TENANCY_ID = "tenancyid";
    public static final String EXT_TEMPLATE_ID = "templateid";
}
```

The outbound `WorkCloudEventAdapter` and `WorkItemLifecycleEvent.of()` migrate to use these constants instead of hardcoded strings.

**Drift prevention:** A test in `casehub-work-api` asserts that `WorkCloudEventTypes` has a `PREFIX + name().toLowerCase()` constant for every `WorkEventType` enum value. New enum values without a matching constant fail the build.

## Inbound Adapter — `WorkCloudEventInboundAdapter`

Location: `runtime` (`io.casehub.work.runtime.event`), alongside the outbound `WorkCloudEventAdapter`.

`@ApplicationScoped` CDI bean.

### Behavior

1. `@ObservesAsync CloudEvent` — filters on type `io.casehub.work.workitem.requested`
2. Extracts `tenancyid` and `templateid` from CloudEvent extensions
3. **Validates required extensions:** missing `tenancyid` or `templateid` → log at ERROR with CloudEvent `id` and `source`, return without processing
4. Wraps execution in `TenantContextRunner.runInTenantContext()` (per protocol `async-event-tenant-context-propagation`)
5. **Idempotency check:** `workItemStore.findActiveByCallerRef(ce.getId())` — if an active WorkItem with this `callerRef` already exists, skip creation (log at DEBUG)
6. Builds `WorkItemCreateRequest`:
   - `.templateId(resolvedId).payload(rawDataAsString).callerRef(ce.getId()).createdBy("cloudevent:" + ce.getSource())`
   - → `WorkItemTemplateService.createFromTemplate()`

All inbound creation goes through the template path. There is no direct (template-less) path — external systems define a `WorkItemTemplate` once during integration setup, then every CloudEvent from that system references it. This keeps domain data opaque and avoids coupling CloudEvent producers to casehub-work's internal DTOs.

### Error Handling

**Client/configuration errors** (missing extensions, template not found, payload schema validation failure) are caught, logged at ERROR with structured context (CloudEvent `id`, `source`, `type`), and do not propagate. Retrying the same malformed or misconfigured event would produce the same failure.

**Infrastructure errors** (database, transaction, CDI runtime) propagate to the caller. This allows transport-level retry — e.g., Kafka does not commit the offset, HTTP returns 500.

**System-level at-least-once delivery** is achieved by combining:
- **Idempotency guard** (`findActiveByCallerRef`) — makes reprocessing safe
- **Transport-level retry** — redelivers events that failed due to transient infrastructure errors

This mirrors the `InboundWorkItemBridge` pattern in casehub-engine: policy exceptions are caught and logged; infrastructure exceptions propagate to the transport's safety net.

### Observability

OTel trace propagation is platform-provided — Quarkus OpenTelemetry instruments CDI observers and `@Transactional` methods automatically. The `WorkItemService.create()` call participates in the active trace context.

Extraction of CloudEvent `traceparent`/`tracestate` distributed tracing extensions for cross-system trace linking is a future enhancement.

## Payload Validation

`WorkItemTemplate` already has an `inputDataSchema` field (JSON Schema draft-07) that validates the WorkItem payload at creation time. `WorkItemService.create()` runs the validation when `inputDataSchema` is non-null — violations reject creation with a clear error message.

This provides everything the spec needs for optional payload validation:
- **Language-agnostic** — JSON Schema, no Java class loading
- **Template-configurable** — set via the template API, null means no validation
- **Already implemented** — no new column, no new migration

**Consumption-time typed access:** consumers handle their own deserialization — `objectMapper.readValue(ref.payload(), MyDomainType.class)`. One line, one deserialize on demand. No convenience method on `WorkItemRef` — that would require adding Jackson to `casehub-work-api` (pure Java, Tier 1, no framework deps).

## Event Pattern Discipline

Events are for **notifications** ("this happened") — direct calls are for **requests** ("do this").

All existing `WorkItemLifecycleEvent` observers in casehub-work are notifications — correct usage. The inbound CloudEvent adapter is a **boundary adapter** for external systems — also correct (cross-system request at the edge).

Internal callers (`casehub-engine`, `casehub-work-flow`) must continue using direct SPI calls (`WorkItemCreator.create()`, `CaseSignalSink.signal()`). No new event-as-request patterns within casehub.

**Follow-up:** casehubio/work#290 — relocate `HumanTaskScheduleHandler` from casehub-engine to casehub-work. Fixes the existing event-as-request anti-pattern (`HumanTaskScheduleEvent`) by replacing it with a direct `WorkItemCreator.create()` call. Domain coherence: HumanTask is Work's responsibility.

## What We Don't Touch

- `HumanTaskFlowBridge` — CDI bridge in the `flow` module (`io.casehub.work.flow.HumanTaskFlowBridge`) for Quarkus Flow workflow integration. Creates WorkItems via `WorkItemService.create()` for same-JVM workflow orchestration. This adapter and `HumanTaskFlowBridge` serve different integration points: CloudEvents for cross-system boundary requests, Flow bridge for same-JVM workflow tasks. Both coexist.
- `WorkCloudEventAdapter` — only change is migrating hardcoded type strings to constants
- `WorkItemLifecycleEvent` — same, use constants
- No quarkus-flow integration — this is casehub-work only

## Round-Trip Integration Test

In `runtime` module:

1. Create a `WorkItemTemplate` with title/groups/priority configured
2. Fire CDI `Event<CloudEvent>.fireAsync()` with type `REQUESTED`, templateId extension, arbitrary JSON data
3. Assert: WorkItem created with correct template fields, payload contains raw domain JSON, `callerRef` matches CloudEvent `id`, `createdBy` is `"cloudevent:" + source`
4. Complete the WorkItem via `WorkItemService.complete()`
5. Assert: CDI `CloudEvent` with type `COMPLETED` was fired, containing the `callerRef` for correlation
6. **Idempotency:** fire the same CloudEvent again (same `id`) → assert no duplicate WorkItem created
