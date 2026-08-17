## D1: Enriched list response shape

**Choice:** Nest the full `WorkItemSummary` under a `summary` field on each queue entry in the list response
**Alternatives:**
- Flatten to explicit fields (`pendingCount`, `activeCount`, `completedCount`) — simpler but loses `overdue`, `claimDeadlineBreached`, `byPriority`, `oldestCreatedAt`; future fields require API changes
**Rationale:** UI already consumes `WorkItemSummary` from `/queues/{id}/summary`. Same shape avoids a second DTO. Summary is 6 fields — negligible response size increase. Already cached per queue.
**Trade-offs:** Slightly larger list response payload than flat fields. Acceptable for typical queue counts (<50).
**Sources:** `QueueResource.java:61` (current list), `WorkItemSummary.java`, `QueueMembershipService.java:33` (cached summarize)
**Exploration:** quick
**Status:** captured
