---
layout: post
title: "The SPI Nobody Injects — Part 2"
date: 2026-09-11
entry_type: note
subtype: diary
projects: [casehubio/work]
tags: [spi, multi-tenancy, cdi, cross-repo]
series: issue-399-remove-tenant-context-executor-spi
---

# The SPI Nobody Injects — Part 2

The previous entry documented the discovery: `TenantContextExecutor` exists
in `casehub-work-api` as an SPI, but zero production callers inject it. All
seven callers — timer jobs, CloudEvent adapters, coordinators, queue jobs,
dashboards, federation receivers — inject the concrete `TenantContextRunner`
directly. The SPI is an abstraction with no consumers.

So I filed #399 to remove it. Straightforward cleanup — delete the interface,
update the one caller that used it (`InboundWorkItemSchedulerImpl` from #397),
done.

Except it wasn't.

When I verified the engine repo before proceeding, commit `066878ef` told a
different story. Engine#974 — the circular dependency break — had moved
`InboundWorkItemBridge` *onto* `TenantContextExecutor`. The engine team
extracted the SPI specifically so `engine-inbound` could depend on `work-api`
instead of `work-runtime`. The commit message even says: "Requires casehubio/work
PR with TenantContextExecutor SPI interface."

The SPI that nobody injects in the work repo is actively consumed cross-repo.
Deleting it would break the engine build.

The fix was to absorb the tenant-context-plus-creation pattern into
`WorkItemOperations` — the SPI the engine bridge already injects. One new
`default` method: `createInTenantContext(String tenancyId, WorkItemCreateRequest request)`.
The engine drops `TenantContextExecutor` and calls the combined method. The
boundary crossing stays visible because `tenancyId` is a separate parameter,
not sourced from request fields — the compliance audit trail the original
design required.

Claude caught one more thing during review: `WorkItemService.createInTenantContext()`
was calling `this.create(request)` — a self-call that bypasses CDI interceptors.
`create()` is `@Transactional`. Without the interceptor, JPA operations would
throw `TransactionRequiredException` at runtime. The fix: inject
`WorkItemOperations self` and call through the CDI proxy. Standard pattern, easy
to miss when you're adding a delegation method to a bean that implements the
interface it needs to call through.

The lesson from this branch isn't technical. It's about the gap between
single-repo analysis and cross-repo reality. `find-references` in one repo
said the SPI was unused. `git log` in another repo said it was load-bearing.
Both were right within their scope.
