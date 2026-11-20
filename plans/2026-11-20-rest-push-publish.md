# REST Push Publish Endpoint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #721 — REST endpoint for publishing to push topics
**Issue group:** #721

**Goal:** Add `POST /api/push/publish/{topic}` to `push-runtime` so YAML workers can broadcast events via `call: http`.

**Architecture:** Thin JAX-RS resource in `push-runtime` delegates to the existing `EventBroadcaster` CDI bean. Follows the same auth pattern as `DataResource` (`@Authenticated` + SmallRye JWT). Tests use REST-assured with signed JWT tokens, following `DataResourceRelayTest`.

**Tech Stack:** Quarkus (JAX-RS via quarkus-rest-jackson), SmallRye JWT, REST-assured

## Global Constraints

- Auth: `@Authenticated` at class level — all existing `/api/` resources use this
- Push topics use `:` separator, never `/` or wildcards — enforced by `EventBroadcaster.validateTopic`
- Follow existing push-runtime test patterns (`@QuarkusTest`, `TestPushConfig` for capturing sent messages)
- Follow existing data module auth test patterns (`Jwt.claims()...sign()`, `casehub-pages-auth` test dep)

---

## Batch 1: Publish endpoint with happy path

### Task 1: POM dependencies + PushPublishResource + happy-path test

**Files:**
- Modify: `backend/push-runtime/pom.xml`
- Create: `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushPublishResource.java`
- Test: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java`

**Interfaces:**
- Consumes: `EventBroadcaster.broadcast(String topic, String payloadJson)` → returns `long` seq
- Produces: `POST /api/push/publish/{topic}` → `{"seq": N}` (HTTP API consumed by YAML workers)

- [ ] **Step 1: Add dependencies to push-runtime pom.xml**

Add to `backend/push-runtime/pom.xml` `<dependencies>` section, before the `<!-- Test -->` comment:

```xml
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-jackson</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-smallrye-jwt</artifactId>
        </dependency>
```

Add to the test dependencies section (after the existing `assertj-core` entry):

```xml
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-auth</artifactId>
            <scope>test</scope>
        </dependency>
```

- [ ] **Step 2: Write the failing happy-path test**

Create `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java`:

```java
package io.casehub.pages.push.runtime;

import io.casehub.pages.push.TopicRegistry;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import io.smallrye.jwt.build.Jwt;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Set;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.equalTo;

@QuarkusTest
class PushPublishResourceTest {

    @Inject TestPushConfig testConfig;
    @Inject TopicRegistry topicRegistry;

    @BeforeEach
    void setUp() {
        testConfig.clear();
    }

    private String validToken() {
        return Jwt.claims()
                .subject("worker-1")
                .groups(Set.of("user"))
                .sign();
    }

    @Test
    void publish_broadcasts_and_returns_seq() {
        topicRegistry.listen("conn-1", List.of("orders:new"));

        given()
                .auth().oauth2(validToken())
                .contentType(ContentType.JSON)
                .body("{\"item\":\"widget\",\"qty\":3}")
                .when()
                .post("/api/push/publish/orders:new")
                .then()
                .statusCode(200)
                .body("seq", equalTo(1));

        assertThat(testConfig.sent()).hasSize(1);
        assertThat(testConfig.sent().getFirst().message()).contains("\"topic\":\"orders:new\"");
        assertThat(testConfig.sent().getFirst().message()).contains("\"item\":\"widget\"");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl backend/push-runtime -Dtest=PushPublishResourceTest#publish_broadcasts_and_returns_seq test -f /Users/mdproctor/claude/casehub/pages/pom.xml`

Expected: FAIL — `PushPublishResource` does not exist, so `POST /api/push/publish/orders:new` returns 404.

- [ ] **Step 4: Implement PushPublishResource**

Create `backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushPublishResource.java`:

```java
package io.casehub.pages.push.runtime;

import io.casehub.pages.push.EventBroadcaster;
import io.quarkus.security.Authenticated;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.BadRequestException;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

@Path("/api/push/publish")
@Authenticated
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class PushPublishResource {

    public record PublishResponse(long seq) {}

    @Inject
    EventBroadcaster broadcaster;

    @POST
    @Path("/{topic}")
    public PublishResponse publish(@PathParam("topic") String topic, String body) {
        if (body == null || body.isBlank()) {
            throw new BadRequestException("request body must not be empty");
        }
        try {
            long seq = broadcaster.broadcast(topic, body);
            return new PublishResponse(seq);
        } catch (IllegalArgumentException e) {
            throw new BadRequestException(e.getMessage());
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl backend/push-runtime -Dtest=PushPublishResourceTest#publish_broadcasts_and_returns_seq test -f /Users/mdproctor/claude/casehub/pages/pom.xml`

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add backend/push-runtime/pom.xml \
       backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushPublishResource.java \
       backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java
git commit -m "feat(#721): add PushPublishResource with happy-path test

POST /api/push/publish/{topic} broadcasts JSON payload to push topic
subscribers via EventBroadcaster. Returns {\"seq\": N} for ordering.

Refs #721"
```

## Batch 2: Error handling and edge cases

### Task 2: Error and auth tests

**Files:**
- Modify: `backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java`

**Interfaces:**
- Consumes: `PushPublishResource` from Task 1
- Produces: test coverage for error paths (no new production code expected)

- [ ] **Step 1: Write failing test — no auth returns 401**

Add to `PushPublishResourceTest.java`:

```java
    @Test
    void publish_without_auth_returns_401() {
        given()
                .contentType(ContentType.JSON)
                .body("{\"x\":1}")
                .when()
                .post("/api/push/publish/test:topic")
                .then()
                .statusCode(401);
    }
```

- [ ] **Step 2: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl backend/push-runtime -Dtest=PushPublishResourceTest#publish_without_auth_returns_401 test -f /Users/mdproctor/claude/casehub/pages/pom.xml`

Expected: PASS — `@Authenticated` already handles this.

- [ ] **Step 3: Write failing test — wildcard topic returns 400**

Add to `PushPublishResourceTest.java`:

```java
    @Test
    void publish_wildcard_topic_returns_400() {
        given()
                .auth().oauth2(validToken())
                .contentType(ContentType.JSON)
                .body("{\"x\":1}")
                .when()
                .post("/api/push/publish/metrics:*:cpu")
                .then()
                .statusCode(400);
    }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl backend/push-runtime -Dtest=PushPublishResourceTest#publish_wildcard_topic_returns_400 test -f /Users/mdproctor/claude/casehub/pages/pom.xml`

Expected: PASS — `EventBroadcaster.validateTopic` throws `IllegalArgumentException`, caught and mapped to `BadRequestException` in the resource.

- [ ] **Step 5: Write failing test — empty body returns 400**

Add to `PushPublishResourceTest.java`:

```java
    @Test
    void publish_empty_body_returns_400() {
        given()
                .auth().oauth2(validToken())
                .contentType(ContentType.JSON)
                .body("")
                .when()
                .post("/api/push/publish/test:topic")
                .then()
                .statusCode(400);
    }
```

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl backend/push-runtime -Dtest=PushPublishResourceTest#publish_empty_body_returns_400 test -f /Users/mdproctor/claude/casehub/pages/pom.xml`

Expected: PASS — the `body.isBlank()` guard in the resource handles this.

- [ ] **Step 7: Write test — publish with no subscribers succeeds**

Add to `PushPublishResourceTest.java`:

```java
    @Test
    void publish_with_no_subscribers_returns_seq() {
        given()
                .auth().oauth2(validToken())
                .contentType(ContentType.JSON)
                .body("{\"alert\":true}")
                .when()
                .post("/api/push/publish/nobody:listens")
                .then()
                .statusCode(200)
                .body("seq", equalTo(1));

        assertThat(testConfig.sent()).isEmpty();
    }
```

- [ ] **Step 8: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl backend/push-runtime -Dtest=PushPublishResourceTest#publish_with_no_subscribers_returns_seq test -f /Users/mdproctor/claude/casehub/pages/pom.xml`

Expected: PASS — `EventBroadcaster.broadcast` handles no-subscriber case gracefully.

- [ ] **Step 9: Run all tests in push-runtime**

Run: `/opt/homebrew/bin/mvn -pl backend/push-runtime test -f /Users/mdproctor/claude/casehub/pages/pom.xml`

Expected: ALL PASS — verify no regressions in existing `BroadcastIntegrationTest`, `CdiCommandResultHandlerTest`, etc.

- [ ] **Step 10: Commit**

```bash
git add backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/PushPublishResourceTest.java
git commit -m "test(#721): add error and edge case tests for PushPublishResource

Covers: 401 without auth, 400 for wildcard topic, 400 for empty body,
200 with no subscribers.

Refs #721"
```

## References

- [2026-11-20-rest-push-publish-design.md] — design spec this plan implements
- [backend/push/src/main/java/io/casehub/pages/push/EventBroadcaster.java] — broadcast logic
- [backend/push-runtime/src/main/java/io/casehub/pages/push/runtime/PushProducers.java] — CDI wiring
- [backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/BroadcastIntegrationTest.java] — existing test patterns
- [backend/push-runtime/src/test/java/io/casehub/pages/push/runtime/TestPushConfig.java] — test SessionSender
- [backend/data/src/main/java/io/casehub/pages/data/DataResource.java] — `@Authenticated` pattern
- [backend/data/src/test/java/io/casehub/pages/data/DataResourceRelayTest.java] — REST-assured + JWT auth test pattern
- [backend/data/pom.xml] — dependency pattern for auth + REST modules
- [GitHub #721] — focal issue
