# REST Endpoint for Publishing to Push Topics

**Issue:** casehubio/casehub-pages#721
**Date:** 2026-11-20

## Summary

Add `POST /api/push/publish/{topic}` to `casehub-pages-push-runtime` so that YAML workers can publish events to push topics via `call: http`. Currently, publishing is only available from server-side Java code via `EventBroadcaster`.

## Architecture

The endpoint is a thin JAX-RS resource in `push-runtime` that delegates to the existing `EventBroadcaster` CDI bean.

```
Worker (do: block)
  → call: http POST /api/push/publish/{topic}
    → PushPublishResource
      → EventBroadcaster.broadcast(topic, json)
        → EventStore.append (persistence)
        → TopicRegistry.connections → SessionSender.send (delivery)
```

No new abstractions. The resource is a REST adapter over an existing capability.

## Endpoint

```
POST /api/push/publish/{topic}
Content-Type: application/json
Authorization: Bearer <jwt>

<arbitrary JSON payload>

→ 200 {"seq": 42}
```

- **Path parameter `{topic}`** — the push topic to broadcast to (e.g. `debate:abc`, `metrics:server1:cpu`). Must not contain wildcards (`*`) or `/` separators (enforced by `EventBroadcaster.validateTopic`).
- **Request body** — arbitrary JSON, passed as-is to `EventBroadcaster.broadcast(topic, payloadJson)`.
- **Response** — JSON object with the sequence number assigned by `EventStore.append`. The seq enables downstream ordering and idempotency checks.
- **Auth** — `@Authenticated` at class level, consistent with all other `/api/` resources.

### Error responses

| Condition | Status | Body |
|-----------|--------|------|
| Invalid topic (wildcard or `/`) | 400 | `{"error": "broadcast topic must not contain wildcards: ..."}` |
| Missing/invalid JWT | 401 | Standard Quarkus security response |
| Empty body | 400 | `{"error": "request body must not be empty"}` |

`EventBroadcaster.validateTopic` throws `IllegalArgumentException` for bad topics. The resource catches `IllegalArgumentException` and returns 400. No custom exception mapper needed — a local try/catch is sufficient for a single method.

## Changes

### 1. `PushPublishResource.java` (new)

`backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushPublishResource.java`

```java
@Path("/api/push/publish")
@Authenticated
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class PushPublishResource {

    @Inject EventBroadcaster broadcaster;

    @POST
    @Path("/{topic}")
    public Response publish(@PathParam("topic") String topic, String body) {
        // validate non-empty body, delegate to broadcaster, return seq
    }
}
```

### 2. `PushPublishResourceTest.java` (new)

`backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java`

`@QuarkusTest` integration test using `@TestSecurity` to simulate authenticated requests:
- Publish to a topic → 200 with seq
- Publish with wildcard topic → 400
- Publish with empty body → 400
- Publish without auth → 401
- Publish and verify event delivered to subscribed connection

### 3. `push-runtime/pom.xml` (modified)

Add dependencies:
- `quarkus-rest` — JAX-RS runtime
- `quarkus-security` — `@Authenticated` annotation support
- `quarkus-rest-jackson` — JSON serialisation for request/response bodies
- `quarkus-test-security` (test scope) — `@TestSecurity` for integration tests

## Topic encoding

Push topics use `:` as separator (e.g. `debate:abc`). Since `{topic}` is a single path segment, topics with `:` work naturally — colons are valid in URL path segments. No special encoding needed.

## Trade-offs

- Workers must present a valid JWT token. This is consistent with the platform auth model but means internal-only use cases still pay the auth cost.
- No batch publish (single topic per request). Not a current requirement; can be added later if needed.

## References

- `backend/push/src/main/java/io/casehub/pages/push/EventBroadcaster.java` — broadcast logic and topic validation
- `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushProducers.java` — CDI wiring for EventBroadcaster
- `backend/data/src/main/java/io/casehub/pages/data/DataResource.java` — auth pattern precedent
- `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/BroadcastIntegrationTest.java` — existing push test patterns
- casehubio/examples#212 — YAML-only helpdesk initiative (parent context)
