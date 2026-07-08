---
layout: post
title: "The Preference That Wasn't Single"
date: 2026-07-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [queues, preferences, notifications, types]
---

I wanted to clean up the aftermath of the types/labels convention (#291) and then build queue trend data for sparklines. Both landed in one session — one a mechanical sweep, the other a proper feature with design review.

## The sweep that grew

The #291 review had flagged five minor findings. What I expected to be a quick fix-and-move-on turned wider: five REST tests were silently creating WorkItems without types. They sent `typePaths` (the template field) to the WorkItem endpoint, Jackson ignored the unknown property, and the tests passed because they never asserted on types. The fix was straightforward — `"typePaths", "[\"test\"]"` becomes `"types", List.of("test")` — but the scope surprised me.

The notification matching was the more interesting finding. Notification rules did flat `List.contains()` for type matching while the query layer used `Path.isAncestorOf()` for hierarchical matching. A rule configured for type `"legal"` wouldn't fire for a WorkItem with type `"legal/contract"`. We added ancestor matching — `t.equals(ruleType) || t.startsWith(ruleType + "/")` — with four tests covering the ancestor, deeper-than, and prefix-but-not-ancestor cases.

## Queue trend data

The real work was #289. Queue dashboard cards need sparkline trend data, and casehub-work has no historical queue membership data — every queue query evaluates membership live.

The design is simple: a Quartz `@Scheduled` heartbeat ticks every five minutes, checks the preference-configured snapshot interval (default hourly), and persists a `QueueSnapshot` row per queue. A `GET /queues/{id}/trend?period=24h` endpoint queries the snapshots.

Three design decisions I wanted to get right:

**Preferences, not @ConfigProperty.** The platform has a typed `PreferenceKey<T>` system with `DurationPreference`, scoped by tenant, backed by YAML or JPA. The snapshot interval and retention period belong there, not as scattered config properties. I hit a gotcha immediately: `DurationPreference` implements `MultiValuePreference`, so `prefs.get(KEY)` doesn't compile — you need `prefs.getOrDefault(KEY, "")` with an empty subKey. Not obvious from the protocol.

**Shared membership evaluation.** `QueueResource.query()` had inline label-pattern + JEXL evaluation. The snapshot job needs the same logic. We extracted `QueueMembershipService` with two methods: `evaluateMembers()` returns full entities (for the REST response), `countMembers()` uses a new `WorkItemStore.countByQuery()` to avoid hydrating entities when all we need is a count.

**Cross-tenant discovery.** The snapshot job needs to iterate all tenants. We followed the existing `CrossTenantWorkItemStore` pattern — a `CrossTenantQueueViewStore` with a single `findDistinctTenancyIds()` method, `REQUIRES_NEW` transaction, `@WorkSystem` CDI producer guard.

The adversarial design review raised 18 issues across three rounds. The most valuable catches: the preference type mismatch, the missing batch query for last-snapshot-times (N+1 in the original design), the FK `ON DELETE CASCADE` from `queue_snapshot` to `queue_view`, and the transaction boundaries on the job's tick method.

The type hierarchy question — `DurationPreference` being `MultiValuePreference` when you're using it as a single value — feels like a gap in the platform API. A `SingleValueDurationPreference` or a change to make `DurationPreference` implement both interfaces would eliminate the gotcha.
