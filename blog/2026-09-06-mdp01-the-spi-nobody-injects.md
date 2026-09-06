---
layout: post
title: "The SPI Nobody Injects"
date: 2026-09-06
entry_type: note
subtype: diary
projects: [casehubio/work]
tags: [dependency-design, bridge-modules, spi, tenant-context]
---

# The SPI Nobody Injects

I was implementing `InboundWorkItemScheduler` in the work engine-adapter — part of engine#974, breaking the circular engine→work dependency. Straightforward SPI implementation: convert `InboundWorkItemRequest` to `WorkItemCreateRequest`, wrap in tenant context, call `WorkItemCreator.create()`. The interesting part was the tenant context question.

The `InboundWorkItemBridge` in the engine fires from a qhorus `afterCompletion(STATUS_COMMITTED)` callback — outside the originating request scope. No `CurrentPrincipal`, no tenant context. The request carries `tenancyId`, so the information exists. The question was how to establish it.

The engine-adapter's POM has `casehub-work-api` at compile scope and `casehub-work` (the runtime) at test scope only. The SPI layer — `TenantContextExecutor` in work-api — exists so the adapter codes against the interface, not the implementation. I initially presented three options: use the SPI, set tenancyId on the request and hope the service handles it, or add work-runtime as a compile dependency and inject `TenantContextRunner` directly.

The third option felt wrong: "tighter coupling." But I traced it, and the premise didn't hold.

The engine-adapter is a bridge module. It connects casehub-engine and casehub-work. No application adds it without both sides on the classpath — that's the entire point of a bridge. There is no deployment where `casehub-work-engine-adapter` is present but `casehub-work` runtime is absent.

So the compile-time separation between work-api and work-runtime in this module isn't about classpath flexibility. It's about compilation isolation — changes to `WorkItemService` internals don't force recompilation of the adapter. That's a build concern, not an architectural boundary.

Then Claude traced who actually injects `TenantContextExecutor`. The answer: nobody. Zero callers. Every production caller in the work repo — `ClaimDeadlineTimerJob`, `ExpiryTimerJob`, `WorkCloudEventInboundAdapter`, `MultiInstanceCoordinator`, `QueueSnapshotJob`, `QueueDashboard`, `FederationReceiver` — injects `TenantContextRunner` directly. The SPI exists in work-api. It has an implementation. It has no consumers.

That's a pure abstraction artifact. A type created for a dependency inversion principle that nobody exercises.

The deeper question: why does every out-of-scope caller need to establish tenant context at all? The `WorkItemCreateRequest` already carries `tenancyId`. The service copies it to the entity. The entity persists it. The problem is that `WorkItemService.create()` internally calls `workItemStore.get()` — a read that hits Hibernate tenant filters, which need `CurrentPrincipal.tenancyId()` from the request scope. The service has the tenant information but can't use it because the read path looks elsewhere.

A self-sufficient service — one that checks whether `CurrentPrincipal` is available and establishes its own context from `request.tenancyId` when it isn't — would eliminate the wrapping for every caller. The service already knows the answer; forcing seven callers to provide it externally is ceremony, not architecture.

I used `TenantContextExecutor` for this branch — the existing pattern, minimal disruption. But I filed #399 for the self-sufficient refactoring, with the full migration surface mapped: all seven callers, which are write-path and which are read-path, and the protocol that would change.

Bridge modules are a special case for dependency design. When a module exists only because two other modules need to talk, both sides are always present. SPI indirection in that context buys compilation isolation at best. When the SPI has zero injection callers, even that is theoretical.
