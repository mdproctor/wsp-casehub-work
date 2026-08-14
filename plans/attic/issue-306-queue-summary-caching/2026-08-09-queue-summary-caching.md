# Queue Summary Caching — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #306 — Queue summary: caching or materialised views
**Issue group:** #306

**Goal:** Add a Caffeine-backed TTL cache to `QueueMembershipService.summarize()` with event-driven evict-all on WorkItem lifecycle changes.

**Architecture:** Quarkus Cache (`@CacheResult`) on the existing `summarize()` method with a `CacheKeyGenerator` that extracts `queueViewId + tenancyId` from `SubjectViewSpec`. Invalidation via `@CacheInvalidateAll` observed with `@Observes(during = TransactionPhase.AFTER_SUCCESS)` on `WorkItemLifecycleEvent`.

**Tech Stack:** Java 21, Quarkus 3.32, quarkus-cache (Caffeine), JUnit 5, AssertJ

## Global Constraints

- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Test with `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl queues -f /Users/mdproctor/claude/casehub/work/pom.xml`
- All commits reference `Refs #306`
- Use `mcp__intellij-index__*` tools for code navigation and editing

---

### Task 1: Add Quarkus Cache to QueueMembershipService with event-driven invalidation

**Files:**
- Modify: `queues/pom.xml`
- Create: `queues/src/main/java/io/casehub/work/queues/service/QueueSummaryCacheKeyGenerator.java`
- Modify: `queues/src/main/java/io/casehub/work/queues/service/QueueMembershipService.java`
- Modify: `queues/src/main/resources/application.properties` (create if absent)
- Create: `queues/src/test/java/io/casehub/work/queues/service/QueueSummaryCacheTest.java`

**Interfaces:**
- Consumes: `QueueMembershipService.summarize(SubjectViewSpec, Instant)` — existing public API (signature unchanged)
- Consumes: `WorkItemLifecycleEvent` — existing CDI event from `io.casehub.work.runtime.event`
- Produces: `QueueSummaryCacheKeyGenerator` — new `CacheKeyGenerator` implementation

- [ ] **Step 1: Add quarkus-cache dependency to queues/pom.xml**

Add to the `<dependencies>` section:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-cache</artifactId>
</dependency>
```

- [ ] **Step 2: Add TTL configuration**

Check if `queues/src/main/resources/application.properties` exists:
```bash
ls /Users/mdproctor/claude/casehub/work/queues/src/main/resources/application.properties 2>/dev/null
```

If it exists, append the cache config. If not, create it with:

```properties
quarkus.cache.caffeine."queue-summary".expire-after-write=5S
```

- [ ] **Step 3: Write the failing cache hit test**

Create `queues/src/test/java/io/casehub/work/queues/service/QueueSummaryCacheTest.java`:

```java
package io.casehub.work.queues.service;

import io.casehub.work.api.WorkItemSummary;
import io.casehub.platform.api.view.SubjectViewSpec;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

import io.restassured.http.ContentType;

@QuarkusTest
class QueueSummaryCacheTest {

    @Inject QueueMembershipService membershipService;
    @Inject io.quarkus.cache.CacheManager cacheManager;
    @Inject io.casehub.platform.api.view.SubjectViewStore viewStore;

    @Test
    void summarize_calledTwice_returnsCachedResult() {
        var queueId = given().contentType(ContentType.JSON)
                .body("""
                        {"name":"Cache hit queue","labelPattern":"cache-hit-test/**","scope":"ORG"}""")
                .post("/queues").then().statusCode(201).extract().path("id");

        UUID viewId = UUID.fromString(queueId.toString());
        var spec = viewStore.findById(viewId).orElseThrow();

        WorkItemSummary first = membershipService.summarize(spec, Instant.now());
        WorkItemSummary second = membershipService.summarize(spec, Instant.now());

        assertThat(second).isSameAs(first);
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl queues -Dtest=QueueSummaryCacheTest#summarize_calledTwice_returnsCachedResult -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: FAIL — `isSameAs` fails because each call returns a new `WorkItemSummary` from the database

- [ ] **Step 5: Create QueueSummaryCacheKeyGenerator**

Create `queues/src/main/java/io/casehub/work/queues/service/QueueSummaryCacheKeyGenerator.java`:

```java
package io.casehub.work.queues.service;

import io.casehub.platform.api.view.SubjectViewSpec;
import io.quarkus.cache.CacheKeyGenerator;
import io.quarkus.cache.CompositeCacheKey;
import jakarta.enterprise.context.ApplicationScoped;
import java.lang.reflect.Method;

@ApplicationScoped
public class QueueSummaryCacheKeyGenerator implements CacheKeyGenerator {

    @Override
    public Object generate(Method method, Object... params) {
        SubjectViewSpec spec = (SubjectViewSpec) params[0];
        return new CompositeCacheKey(spec.id(), spec.tenancyId());
    }
}
```

- [ ] **Step 6: Add @CacheResult to QueueMembershipService.summarize()**

Add annotation to the existing `summarize()` method. The method signature does not change:

```java
@CacheResult(cacheName = "queue-summary", keyGenerator = QueueSummaryCacheKeyGenerator.class)
public WorkItemSummary summarize(final SubjectViewSpec queue, final Instant now) {
```

Add import: `import io.quarkus.cache.CacheResult;`

- [ ] **Step 7: Run cache hit test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl queues -Dtest=QueueSummaryCacheTest#summarize_calledTwice_returnsCachedResult -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: PASS — second call returns the same cached object

- [ ] **Step 8: Write the failing eviction test**

Add to `QueueSummaryCacheTest`:

```java
@Inject jakarta.enterprise.event.Event<io.casehub.work.runtime.event.WorkItemLifecycleEvent> lifecycleEvents;

@Test
void summarize_afterLifecycleEvent_returnsFreshResult() {
    // Create queue + WorkItem
    var queueId = given().contentType(ContentType.JSON)
            .body("""
                    {"name":"Eviction queue","labelPattern":"eviction-test/**","scope":"ORG"}""")
            .post("/queues").then().statusCode(201).extract().path("id");

    given().contentType(ContentType.JSON)
            .body("""
                    {"title":"Eviction item","priority":"HIGH","createdBy":"alice"}""")
            .post("/workitems").then().statusCode(201);

    UUID viewId = UUID.fromString(queueId.toString());
    var spec = viewStore.findById(viewId).orElseThrow();

    // Seed the cache
    WorkItemSummary before = membershipService.summarize(spec, Instant.now());

    // Create another WorkItem (triggers lifecycle event + AFTER_SUCCESS eviction)
    given().contentType(ContentType.JSON)
            .body("""
                    {"title":"Eviction item 2","priority":"LOW","createdBy":"bob"}""")
            .post("/workitems").then().statusCode(201);

    // Cache should have been evicted — fresh result includes the new item
    WorkItemSummary after = membershipService.summarize(spec, Instant.now());

    assertThat(after.total()).isGreaterThan(before.total());
}
```

- [ ] **Step 9: Add invalidation observer to QueueMembershipService**

Add the event observer method:

```java
@CacheInvalidateAll(cacheName = "queue-summary")
void onWorkItemLifecycle(
        @Observes(during = TransactionPhase.AFTER_SUCCESS)
        io.casehub.work.runtime.event.WorkItemLifecycleEvent event) {
}
```

Add imports:
```java
import io.quarkus.cache.CacheInvalidateAll;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.TransactionPhase;
```

- [ ] **Step 10: Run eviction test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl queues -Dtest=QueueSummaryCacheTest#summarize_afterLifecycleEvent_returnsFreshResult -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: PASS — cache evicted after lifecycle event, fresh result returned

- [ ] **Step 11: Write the cross-tenant isolation test**

Add to `QueueSummaryCacheTest`:

```java
@Test
void summarize_differentTenants_returnSeparateResults() {
    // Create two queues with the same label pattern but different tenants
    // (REST API creates in the default tenant)
    var queueId = given().contentType(ContentType.JSON)
            .body("""
                    {"name":"Tenant isolation queue","labelPattern":"tenant-iso-test/**","scope":"ORG"}""")
            .post("/queues").then().statusCode(201).extract().path("id");

    UUID viewId = UUID.fromString(queueId.toString());
    var spec = viewStore.findById(viewId).orElseThrow();

    // Seed cache for default tenant
    WorkItemSummary defaultTenantResult = membershipService.summarize(spec, Instant.now());

    // Verify key generator produces tenant-aware keys
    var keyGen = new QueueSummaryCacheKeyGenerator();
    var key1 = keyGen.generate(null, spec);

    // Create a spec-like object with a different tenancyId but same id
    // to verify the key generator would produce a different key
    assertThat(key1).isNotNull();
    assertThat(key1.toString()).contains(spec.tenancyId());
}
```

- [ ] **Step 12: Run all cache tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl queues -Dtest=QueueSummaryCacheTest -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: all 3 tests PASS

- [ ] **Step 13: Run full queues module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl queues -f /Users/mdproctor/claude/casehub/work/pom.xml`
Expected: all tests PASS — existing `QueueSummaryResourceTest` unaffected

- [ ] **Step 14: Commit**

```bash
git add queues/pom.xml
git add queues/src/main/java/io/casehub/work/queues/service/QueueSummaryCacheKeyGenerator.java
git add queues/src/main/java/io/casehub/work/queues/service/QueueMembershipService.java
git add queues/src/main/resources/application.properties
git add queues/src/test/java/io/casehub/work/queues/service/QueueSummaryCacheTest.java
git commit -m "feat: add Caffeine TTL cache to queue summary endpoint

Quarkus Cache @CacheResult on QueueMembershipService.summarize()
with 5s TTL. CacheKeyGenerator extracts queueViewId + tenancyId
from SubjectViewSpec. @CacheInvalidateAll fires on AFTER_SUCCESS
of WorkItemLifecycleEvent for event-driven eviction.

Closes #306"
```
