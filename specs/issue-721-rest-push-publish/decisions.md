## D1: Auth model

**Choice:** `@Authenticated` JWT — consistent with all other `/api/` resources
**Alternatives:**
- No auth (internal-only) — simpler, but assumes endpoint never exposed externally; inconsistent with other `/api/` resources
**Rationale:** Workers already have auth context from their session. Prevents accidental external exposure. Consistent with DataResource, LayoutResource, etc.
**Trade-offs:** Workers must present a valid JWT token when calling via `call: http`
**Sources:** `backend/data/src/main/java/io/casehub/pages/data/DataResource.java:25` — `@Authenticated` pattern
**Exploration:** quick
**Status:** captured

## D2: API surface

**Choice:** Raw JSON passthrough — `POST /api/push/publish/{topic}` with JSON body passed directly to `broadcast(topic, json)`, returns `{"seq": N}`
**Alternatives:**
- Typed envelope (`POST /api/push/publish` with `{"topic": "...", "payload": {...}}`) — topic in body instead of path; overengineered, inconsistent with REST conventions
**Rationale:** Thinnest wrapper over EventBroadcaster. Topic as path parameter follows REST conventions. Matches how `call: http` naturally works (POST a payload to a URL). Seq in response enables ordering/idempotency downstream.
**Trade-offs:** Cannot batch-publish to multiple topics in one request (not a current requirement)
**Sources:** `backend/push/src/main/java/io/casehub/pages/push/EventBroadcaster.java:19` — `broadcast(String topic, String payloadJson)` signature
**Exploration:** quick
**Status:** captured
