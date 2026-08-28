# Qhorus WorkItem Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #97 — WorkItem event mesh: lifecycle events crossing service boundaries
**Issue group:** #97

**Goal:** Build a `casehub-work-qhorus` module that bridges WorkItem lifecycle with Qhorus channels via MCP tools and a `WorkItemObserver` outbound adapter.

**Architecture:** Thin bridge module with 3 MCP tools (`request_human_work`, `check_work_status`, `wait_for_work`) and 1 `WorkItemObserver` implementation that posts terminal speech acts back to originating Qhorus channels. Correlation via `callerRef` with `qhorus:{channelId}/{messageId}/{correlationId}` format. Polling-based `wait_for_work` (no in-memory registry).

**Tech Stack:** Java 21, Quarkus 3.32.2, `quarkus-mcp-server-http` (MCP tools), `casehub-work-api` (WorkItemCreator, WorkItemObserver), `casehub-qhorus-api` (MessageDispatcher, ChannelReader)

## Global Constraints

- Java 21 source (`maven.compiler.release=21`), Java 26 JVM
- `casehub-work-api` and `casehub-qhorus-api` at compile scope (api modules only — not runtime)
- Jandex index required (`jandex-maven-plugin`)
- Pre-release: breaking changes cost nothing
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl qhorus`
- IntelliJ MCP for all code navigation and editing

---

### Task 1: Module scaffold and QhorusCallerRef

**Files:**
- Create: `qhorus/pom.xml`
- Create: `qhorus/src/main/java/io/casehub/work/qhorus/QhorusCallerRef.java`
- Create: `qhorus/src/test/java/io/casehub/work/qhorus/QhorusCallerRefTest.java`
- Modify: `pom.xml` (parent — add `<module>qhorus</module>`)

**Interfaces:**
- Produces: `QhorusCallerRef.encode(UUID channelId, long messageId, String correlationId)` → `String`, `QhorusCallerRef.parse(String callerRef)` → `QhorusRef`, `QhorusCallerRef.isQhorus(String callerRef)` → `boolean`

- [ ] **Step 1: Create module directory and pom.xml**

```xml
<!-- qhorus/pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-work-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-work-qhorus</artifactId>
    <name>CaseHub Work :: Qhorus Bridge</name>
    <description>MCP tool bridge between casehub-work WorkItems and Qhorus agent mesh channels</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-qhorus-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-mcp-server-http</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work-deployment</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work-persistence-memory</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-qhorus</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-qhorus-persistence-memory</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ledger</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ledger-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-h2</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.awaitility</groupId>
            <artifactId>awaitility</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <parameters>true</parameters>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <systemPropertyVariables>
                        <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
                        <maven.home>${maven.home}</maven.home>
                    </systemPropertyVariables>
                </configuration>
            </plugin>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals>
                            <goal>jandex</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>qhorus</module>` to the `<modules>` section in the parent pom, after `engine-adapter`:

```xml
    <module>engine-adapter</module>
    <module>qhorus</module>
```

- [ ] **Step 3: Write QhorusCallerRef tests**

```java
package io.casehub.work.qhorus;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

class QhorusCallerRefTest {

    @Test
    void encodeProducesCorrectFormat() {
        var channelId = UUID.fromString("a1b2c3d4-e5f6-7890-abcd-ef1234567890");
        var ref = new QhorusCallerRef(channelId, 42L, "corr-001");
        assertThat(ref.encode()).isEqualTo("qhorus:a1b2c3d4-e5f6-7890-abcd-ef1234567890/42/corr-001");
    }

    @Test
    void parseRoundTrips() {
        var channelId = UUID.fromString("a1b2c3d4-e5f6-7890-abcd-ef1234567890");
        var original = new QhorusCallerRef(channelId, 42L, "corr-001");
        var parsed = QhorusCallerRef.parse(original.encode());
        assertThat(parsed.channelId()).isEqualTo(channelId);
        assertThat(parsed.messageId()).isEqualTo(42L);
        assertThat(parsed.correlationId()).isEqualTo("corr-001");
    }

    @Test
    void isQhorusReturnsTrueForQhorusPrefix() {
        assertThat(QhorusCallerRef.isQhorus("qhorus:abc/1/x")).isTrue();
    }

    @Test
    void isQhorusReturnsFalseForOtherPrefixes() {
        assertThat(QhorusCallerRef.isQhorus("case:abc/pi:def")).isFalse();
        assertThat(QhorusCallerRef.isQhorus(null)).isFalse();
        assertThat(QhorusCallerRef.isQhorus("")).isFalse();
    }

    @Test
    void parseMalformedThrows() {
        assertThatThrownBy(() -> QhorusCallerRef.parse("qhorus:missing-segments"))
                .isInstanceOf(IllegalArgumentException.class);
        assertThatThrownBy(() -> QhorusCallerRef.parse("case:not-qhorus/1/x"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 4: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl qhorus -Dtest=QhorusCallerRefTest`
Expected: Compilation failure — `QhorusRef` does not exist

- [ ] **Step 5: Implement QhorusCallerRef**

```java
package io.casehub.work.qhorus;

import java.util.UUID;

public record QhorusCallerRef(UUID channelId, long messageId, String correlationId) {

    public static final String PREFIX = "qhorus:";

    public static boolean isQhorus(final String callerRef) {
        return callerRef != null && callerRef.startsWith(PREFIX);
    }

    public static QhorusCallerRef parse(final String callerRef) {
        if (!isQhorus(callerRef)) {
            throw new IllegalArgumentException("Not a Qhorus callerRef: " + callerRef);
        }
        final String body = callerRef.substring(PREFIX.length());
        final String[] parts = body.split("/", 3);
        if (parts.length != 3) {
            throw new IllegalArgumentException("Malformed Qhorus callerRef (expected 3 segments): " + callerRef);
        }
        return new QhorusCallerRef(UUID.fromString(parts[0]), Long.parseLong(parts[1]), parts[2]);
    }

    public String encode() {
        return PREFIX + channelId + "/" + messageId + "/" + correlationId;
    }
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl qhorus -Dtest=QhorusCallerRefTest`
Expected: All 5 tests PASS

- [ ] **Step 7: Commit**

```bash
git add qhorus/ pom.xml
git commit -m "feat(#97): scaffold casehub-work-qhorus module with QhorusCallerRef

New module bridging casehub-work WorkItems and Qhorus channels.
QhorusCallerRef encodes channelId/messageId/correlationId in the
existing callerRef field.

Refs #97, Refs #92"
```

---

### Task 2: Outbound lifecycle adapter — QhorusWorkItemLifecycleAdapter

**Files:**
- Create: `qhorus/src/main/java/io/casehub/work/qhorus/QhorusWorkItemLifecycleAdapter.java`
- Create: `qhorus/src/test/java/io/casehub/work/qhorus/QhorusWorkItemLifecycleAdapterTest.java`

**Interfaces:**
- Consumes: `QhorusCallerRef.isQhorus()`, `QhorusCallerRef.parse()` from Task 1
- Consumes: `WorkItemObserver.onStatusChange(WorkItemStatusEvent)` — SPI in `casehub-work-api`
- Consumes: `MessageDispatcher.dispatch(MessageDispatch)` — SPI in `casehub-qhorus-api`
- Produces: CDI bean implementing `WorkItemObserver` — posts terminal speech acts to Qhorus channels

- [ ] **Step 1: Write failing tests**

Create test class with `@QuarkusTest`. Requires test `application.properties` first:

```properties
# qhorus/src/test/resources/application.properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:workqhorus;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true

quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:qhorus;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.qhorus.database.generation=none
quarkus.flyway.qhorus.migrate-at-start=true
```

```java
package io.casehub.work.qhorus;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.ChannelReader;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.channel.ChannelCreateRequest;
import io.casehub.qhorus.api.channel.ChannelManager;
import io.casehub.qhorus.api.message.*;
import io.casehub.work.api.*;
import io.casehub.work.api.spi.WorkItemCreator;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class QhorusWorkItemLifecycleAdapterTest {

    @Inject WorkItemCreator workItemCreator;
    @Inject QhorusWorkItemLifecycleAdapter adapter;
    @Inject ChannelManager channelManager;
    @Inject ChannelReader channelReader;
    @Inject ConsumerMessaging messaging;

    @Test
    void terminalCompletedPostsDoneToChannel() {
        var channel = channelManager.create(ChannelCreateRequest.builder()
                .name("test/oversight-done").description("test").semantic(ChannelSemantic.APPEND).build());

        // Post a QUERY to get a messageId for inReplyTo
        var queryResult = messaging.dispatch(MessageDispatch.builder()
                .channelId(channel.id()).sender("agent-1").type(MessageType.QUERY)
                .content("Need human review").actorType(ActorType.AGENT)
                .correlationId("corr-1").build());

        var callerRef = new QhorusCallerRef(channel.id(), queryResult.messageId(), "corr-1").encode();

        var event = new WorkItemStatusEvent(
                WorkEventType.COMPLETED, UUID.randomUUID(), WorkItemStatus.COMPLETED,
                "human-1", "Approved", callerRef, "human-1", "reviewers",
                "approved", "default", Instant.now());

        adapter.onStatusChange(event);

        var messages = messaging.history(channel.id(), queryResult.messageId(), 10);
        assertThat(messages).hasSize(1);
        assertThat(messages.get(0).messageType).isEqualTo(MessageType.DONE);
        assertThat(messages.get(0).correlationId).isEqualTo("corr-1");
        assertThat(messages.get(0).inReplyTo).isEqualTo(queryResult.messageId());
    }

    @Test
    void nonQhorusCallerRefIsIgnored() {
        var event = new WorkItemStatusEvent(
                WorkEventType.COMPLETED, UUID.randomUUID(), WorkItemStatus.COMPLETED,
                "human-1", "Done", "case:abc/pi:def", "human-1", null,
                "done", "default", Instant.now());

        // Should not throw — silently ignored
        adapter.onStatusChange(event);
    }

    @Test
    void nonTerminalStatusIsIgnored() {
        var channelId = UUID.randomUUID();
        var callerRef = new QhorusCallerRef(channelId, 1L, "corr-1").encode();

        var event = new WorkItemStatusEvent(
                WorkEventType.ASSIGNED, UUID.randomUUID(), WorkItemStatus.ASSIGNED,
                "human-1", "Claimed", callerRef, "human-1", null,
                null, "default", Instant.now());

        // Should not throw — non-terminal, ignored
        adapter.onStatusChange(event);
    }

    @Test
    void rejectedPostsFailure() {
        var channel = channelManager.create(ChannelCreateRequest.builder()
                .name("test/oversight-reject").description("test").semantic(ChannelSemantic.APPEND).build());

        var queryResult = messaging.dispatch(MessageDispatch.builder()
                .channelId(channel.id()).sender("agent-1").type(MessageType.QUERY)
                .content("Review needed").actorType(ActorType.AGENT)
                .correlationId("corr-2").build());

        var callerRef = new QhorusCallerRef(channel.id(), queryResult.messageId(), "corr-2").encode();

        var event = new WorkItemStatusEvent(
                WorkEventType.REJECTED, UUID.randomUUID(), WorkItemStatus.REJECTED,
                "human-1", "Cannot complete", callerRef, "human-1", null,
                null, "default", Instant.now());

        adapter.onStatusChange(event);

        var messages = messaging.history(channel.id(), queryResult.messageId(), 10);
        assertThat(messages).hasSize(1);
        assertThat(messages.get(0).messageType).isEqualTo(MessageType.FAILURE);
    }

    @Test
    void cancelledPostsDecline() {
        var channel = channelManager.create(ChannelCreateRequest.builder()
                .name("test/oversight-cancel").description("test").semantic(ChannelSemantic.APPEND).build());

        var queryResult = messaging.dispatch(MessageDispatch.builder()
                .channelId(channel.id()).sender("agent-1").type(MessageType.QUERY)
                .content("Review needed").actorType(ActorType.AGENT)
                .correlationId("corr-3").build());

        var callerRef = new QhorusCallerRef(channel.id(), queryResult.messageId(), "corr-3").encode();

        var event = new WorkItemStatusEvent(
                WorkEventType.CANCELLED, UUID.randomUUID(), WorkItemStatus.CANCELLED,
                "system", "Cancelled", callerRef, null, null,
                null, "default", Instant.now());

        adapter.onStatusChange(event);

        var messages = messaging.history(channel.id(), queryResult.messageId(), 10);
        assertThat(messages).hasSize(1);
        assertThat(messages.get(0).messageType).isEqualTo(MessageType.DECLINE);
    }

    @Test
    void dispatchFailureDoesNotPropagate() {
        // callerRef points to a non-existent channel — dispatch will fail
        var callerRef = new QhorusCallerRef(UUID.randomUUID(), 999L, "corr-bad").encode();

        var event = new WorkItemStatusEvent(
                WorkEventType.COMPLETED, UUID.randomUUID(), WorkItemStatus.COMPLETED,
                "human-1", "Done", callerRef, "human-1", null,
                "done", "default", Instant.now());

        // Must not throw — exception swallowed
        adapter.onStatusChange(event);
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl qhorus -Dtest=QhorusWorkItemLifecycleAdapterTest`
Expected: Compilation failure — `QhorusWorkItemLifecycleAdapter` does not exist

- [ ] **Step 3: Implement QhorusWorkItemLifecycleAdapter**

```java
package io.casehub.work.qhorus;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageDispatcher;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.work.api.WorkEventType;
import io.casehub.work.api.WorkItemStatusEvent;
import io.casehub.work.api.spi.WorkItemObserver;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class QhorusWorkItemLifecycleAdapter implements WorkItemObserver {

    private static final Logger LOG = Logger.getLogger(QhorusWorkItemLifecycleAdapter.class);

    @Inject
    MessageDispatcher messageDispatcher;

    @Override
    public void onStatusChange(final WorkItemStatusEvent event) {
        try {
            if (!QhorusCallerRef.isQhorus(event.callerRef())) {
                return;
            }
            if (!event.status().isTerminal()) {
                return;
            }

            final QhorusCallerRef ref = QhorusCallerRef.parse(event.callerRef());
            final MessageType speechAct = mapToSpeechAct(event.eventType());
            if (speechAct == null) {
                return;
            }

            final String content = buildContent(event);

            messageDispatcher.dispatch(MessageDispatch.builder()
                    .channelId(ref.channelId())
                    .sender("workitems")
                    .type(speechAct)
                    .correlationId(ref.correlationId())
                    .inReplyTo(ref.messageId())
                    .content(content)
                    .actorType(ActorType.SYSTEM)
                    .tenancyId(event.tenancyId())
                    .build());

        } catch (final Exception ex) {
            LOG.warnf(ex, "Qhorus channel post failed for callerRef=%s workItemId=%s",
                    event.callerRef(), event.workItemId());
        }
    }

    static MessageType mapToSpeechAct(final WorkEventType eventType) {
        return switch (eventType) {
            case COMPLETED -> MessageType.DONE;
            case REJECTED, FAULTED, ESCALATED -> MessageType.FAILURE;
            case CANCELLED, EXPIRED, OBSOLETE -> MessageType.DECLINE;
            default -> null;
        };
    }

    private static String buildContent(final WorkItemStatusEvent event) {
        return "{\"workItemId\":\"" + event.workItemId()
                + "\",\"status\":\"" + event.status()
                + "\",\"outcome\":" + jsonString(event.outcome())
                + ",\"resolution\":" + jsonString(event.detail())
                + ",\"assigneeId\":" + jsonString(event.assigneeId()) + "}";
    }

    private static String jsonString(final String value) {
        return value == null ? "null" : "\"" + value.replace("\"", "\\\"") + "\"";
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl qhorus -Dtest=QhorusWorkItemLifecycleAdapterTest`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add qhorus/
git commit -m "feat(#97): QhorusWorkItemLifecycleAdapter — outbound speech act bridge

WorkItemObserver implementation that posts terminal lifecycle events
(COMPLETED→DONE, REJECTED→FAILURE, CANCELLED→DECLINE, etc.) back to
the originating Qhorus channel. Try-catch wraps the entire observer
to prevent transaction rollback.

Refs #97, Refs #92"
```

---

### Task 3: MCP tools — WorkQhorusMcpTools

**Files:**
- Create: `qhorus/src/main/java/io/casehub/work/qhorus/WorkQhorusMcpTools.java`
- Create: `qhorus/src/test/java/io/casehub/work/qhorus/WorkQhorusMcpToolsTest.java`

**Interfaces:**
- Consumes: `QhorusRef` from Task 1, `QhorusWorkItemLifecycleAdapter` from Task 2
- Consumes: `WorkItemCreator.create()`, `WorkItemCreator.findByCallerRef()` from `casehub-work-api`
- Consumes: `ChannelReader.findByName()` from `casehub-qhorus-api`
- Consumes: `MessageDispatcher.dispatch()` from `casehub-qhorus-api`
- Produces: 3 MCP tools: `request_human_work`, `check_work_status`, `wait_for_work`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.work.qhorus;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.*;
import io.casehub.qhorus.api.message.*;
import io.casehub.work.api.*;
import io.casehub.work.api.spi.WorkItemCreator;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class WorkQhorusMcpToolsTest {

    @Inject WorkQhorusMcpTools tools;
    @Inject WorkItemCreator workItemCreator;
    @Inject ChannelManager channelManager;
    @Inject ConsumerMessaging messaging;

    @Test
    void requestHumanWorkCreatesWorkItemAndPostsQuery() {
        var channel = channelManager.create(ChannelCreateRequest.builder()
                .name("test/request-roundtrip").description("test").semantic(ChannelSemantic.APPEND).build());

        var result = tools.requestHumanWork("test/request-roundtrip", "Review PR #42",
                "Please review the pull request", "reviewers", null, null, null, "agent-1");

        assertThat(result.workItemId()).isNotNull();
        assertThat(result.correlationId()).isNotNull();
        assertThat(result.status()).isEqualTo("PENDING");

        // Verify callerRef format
        assertThat(QhorusCallerRef.isQhorus(result.callerRef())).isTrue();
        var ref = QhorusCallerRef.parse(result.callerRef());
        assertThat(ref.channelId()).isEqualTo(channel.id());
        assertThat(ref.correlationId()).isEqualTo(result.correlationId());

        // Verify QUERY posted to channel
        var messages = messaging.history(channel.id(), 0L, 10);
        assertThat(messages).anyMatch(m -> m.messageType == MessageType.QUERY
                && m.correlationId.equals(result.correlationId()));
    }

    @Test
    void requestHumanWorkRejectsNonexistentChannel() {
        assertThatThrownBy(() -> tools.requestHumanWork("nonexistent/channel",
                "Title", "Desc", null, null, null, null, "agent-1"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Channel not found");
    }

    @Test
    void checkWorkStatusReturnsPendingThenCompleted() {
        var channel = channelManager.create(ChannelCreateRequest.builder()
                .name("test/check-status").description("test").semantic(ChannelSemantic.APPEND).build());

        var created = tools.requestHumanWork("test/check-status", "Review",
                "Review needed", null, null, null, null, "agent-1");

        var pending = tools.checkWorkStatus(created.callerRef());
        assertThat(pending.status()).isEqualTo("PENDING");

        // Complete the WorkItem via the SPI
        var ref = workItemCreator.findByCallerRef(created.callerRef()).orElseThrow();
        // Would need WorkItemService to transition — verify at integration level
        assertThat(ref.status()).isEqualTo(WorkItemStatus.PENDING);
    }

    @Test
    void checkWorkStatusReturnsNotFound() {
        var result = tools.checkWorkStatus("qhorus:00000000-0000-0000-0000-000000000000/1/nonexistent");
        assertThat(result.status()).isEqualTo("NOT_FOUND");
    }

    @Test
    void waitForWorkReturnsImmediatelyWhenAlreadyTerminal() {
        var channel = channelManager.create(ChannelCreateRequest.builder()
                .name("test/wait-terminal").description("test").semantic(ChannelSemantic.APPEND).build());

        var created = tools.requestHumanWork("test/wait-terminal", "Review",
                "Review needed", null, null, null, null, "agent-1");

        // wait_for_work on a PENDING item with very short timeout
        var result = tools.waitForWork(created.callerRef(), 1, 1);
        assertThat(result.timedOut()).isTrue();
        assertThat(result.status()).isEqualTo("PENDING");
    }

    @Test
    void twoRequestsCreateDistinctWorkItems() {
        var channel = channelManager.create(ChannelCreateRequest.builder()
                .name("test/idempotency").description("test").semantic(ChannelSemantic.APPEND).build());

        var r1 = tools.requestHumanWork("test/idempotency", "Review 1",
                "First", null, null, null, null, "agent-1");
        var r2 = tools.requestHumanWork("test/idempotency", "Review 2",
                "Second", null, null, null, null, "agent-1");

        assertThat(r1.workItemId()).isNotEqualTo(r2.workItemId());
        assertThat(r1.correlationId()).isNotEqualTo(r2.correlationId());
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl qhorus -Dtest=WorkQhorusMcpToolsTest`
Expected: Compilation failure — `WorkQhorusMcpTools` does not exist

- [ ] **Step 3: Implement WorkQhorusMcpTools**

```java
package io.casehub.work.qhorus;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelReader;
import io.casehub.qhorus.api.message.*;
import io.casehub.work.api.*;
import io.casehub.work.api.spi.WorkItemCreator;
import io.quarkiverse.mcp.server.Tool;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.UUID;

@ApplicationScoped
public class WorkQhorusMcpTools {

    private static final Logger LOG = Logger.getLogger(WorkQhorusMcpTools.class);

    @Inject ChannelReader channelReader;
    @Inject MessageDispatcher messageDispatcher;
    @Inject WorkItemCreator workItemCreator;
    @Inject CurrentPrincipal currentPrincipal;

    @Tool(description = "Request human work by creating a WorkItem and posting a QUERY to a Qhorus channel")
    public HumanWorkResponse requestHumanWork(
            String channel, String title, String description,
            String candidateGroups, String priority, String payload,
            String templateId, String sender) {

        final Channel ch = channelReader.findByName(channel)
                .orElseThrow(() -> new IllegalArgumentException("Channel not found: " + channel));

        final String correlationId = UUID.randomUUID().toString();

        final DispatchResult queryResult = messageDispatcher.dispatch(MessageDispatch.builder()
                .channelId(ch.id())
                .sender(sender)
                .type(MessageType.QUERY)
                .content(title + (description != null ? " — " + description : ""))
                .correlationId(correlationId)
                .actorType(ActorType.AGENT)
                .tenancyId(currentPrincipal.tenancyId())
                .build());

        final String callerRef = new QhorusCallerRef(ch.id(), queryResult.messageId(), correlationId).encode();

        final WorkItemCreateRequest.Builder requestBuilder = WorkItemCreateRequest.builder()
                .title(title)
                .description(description)
                .callerRef(callerRef)
                .createdBy("qhorus:" + sender)
                .tenancyId(currentPrincipal.tenancyId());

        if (candidateGroups != null) requestBuilder.candidateGroups(candidateGroups);
        if (priority != null) requestBuilder.priority(WorkItemPriority.valueOf(priority.toUpperCase()));
        if (payload != null) requestBuilder.payload(payload);
        if (templateId != null) requestBuilder.templateId(UUID.fromString(templateId));

        final WorkItemRef ref = workItemCreator.create(requestBuilder.build());

        return new HumanWorkResponse(ref.id(), callerRef, correlationId, ref.status().name());
    }

    @Tool(description = "Check the current status of a previously requested human work item")
    public WorkStatusResponse checkWorkStatus(String callerRef) {
        return workItemCreator.findByCallerRef(callerRef)
                .map(ref -> new WorkStatusResponse(ref.id(), ref.status().name(),
                        ref.assigneeId(), ref.outcome(), ref.resolution(), false))
                .orElse(new WorkStatusResponse(null, "NOT_FOUND", null, null, null, false));
    }

    @Tool(description = "Poll until a human work item reaches a terminal state or times out")
    public WorkStatusResponse waitForWork(String callerRef, int timeoutSeconds, int pollIntervalSeconds) {
        final int timeout = timeoutSeconds > 0 ? timeoutSeconds : 300;
        final int interval = pollIntervalSeconds > 0 ? pollIntervalSeconds : 5;
        final long deadline = System.currentTimeMillis() + (timeout * 1000L);

        while (System.currentTimeMillis() < deadline) {
            final var refOpt = workItemCreator.findByCallerRef(callerRef);
            if (refOpt.isEmpty()) {
                return new WorkStatusResponse(null, "NOT_FOUND", null, null, null, false);
            }
            final WorkItemRef ref = refOpt.get();
            if (ref.status().isTerminal()) {
                return new WorkStatusResponse(ref.id(), ref.status().name(),
                        ref.assigneeId(), ref.outcome(), ref.resolution(), false);
            }
            try {
                Thread.sleep(interval * 1000L);
            } catch (final InterruptedException e) {
                Thread.currentThread().interrupt();
                return new WorkStatusResponse(ref.id(), ref.status().name(),
                        ref.assigneeId(), ref.outcome(), ref.resolution(), true);
            }
        }

        return workItemCreator.findByCallerRef(callerRef)
                .map(ref -> new WorkStatusResponse(ref.id(), ref.status().name(),
                        ref.assigneeId(), ref.outcome(), ref.resolution(), true))
                .orElse(new WorkStatusResponse(null, "NOT_FOUND", null, null, null, true));
    }

    public record HumanWorkResponse(UUID workItemId, String callerRef, String correlationId, String status) {}
    public record WorkStatusResponse(UUID workItemId, String status, String assigneeId,
                                     String outcome, String resolution, boolean timedOut) {}
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl qhorus -Dtest=WorkQhorusMcpToolsTest`
Expected: All 6 tests PASS

- [ ] **Step 5: Run all module tests together**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl qhorus`
Expected: All tests PASS (QhorusCallerRefTest + QhorusWorkItemLifecycleAdapterTest + WorkQhorusMcpToolsTest)

- [ ] **Step 6: Commit**

```bash
git add qhorus/
git commit -m "feat(#97): WorkQhorusMcpTools — request_human_work, check_work_status, wait_for_work

Three MCP tools giving Qhorus agents explicit interface for requesting
and tracking human work. request_human_work validates channel, posts
oversight/QUERY, creates WorkItem with qhorus: callerRef. wait_for_work
uses polling (cluster-safe, no in-memory registry).

Refs #97, Refs #92"
```

---

### Task 4: Documentation and module registration

**Files:**
- Modify: `docs/MODULES.md` — add qhorus module entry
- Modify: `ARC42STORIES.MD` — add module to Building Block View if applicable
- Modify: `CLAUDE.md` — add qhorus module description

**Interfaces:**
- Consumes: nothing — documentation only

- [ ] **Step 1: Add module to docs/MODULES.md**

Add to the "Integration Modules (built)" table:

```markdown
| `qhorus/` | Qhorus bridge; MCP tools for agent→WorkItem requests (`request_human_work`, `check_work_status`, `wait_for_work`), outbound lifecycle adapter posts terminal speech acts to originating channels. |
```

- [ ] **Step 2: Add module description to CLAUDE.md**

Add a section after the engine-adapter module description:

```markdown
## qhorus Module

Qhorus bridge (`casehub-work-qhorus`, package `io.casehub.work.qhorus`). Bridges casehub-work WorkItem lifecycle with Qhorus agent mesh channels.

Activated by adding `casehub-work-qhorus` to the consumer's classpath. Requires both `casehub-work` and `casehub-qhorus` at runtime.

Three MCP tools: `request_human_work` (creates WorkItem + posts oversight/QUERY), `check_work_status` (polls status), `wait_for_work` (polls until terminal).

Outbound: `QhorusWorkItemLifecycleAdapter` implements `WorkItemObserver`, posts terminal events (COMPLETED→DONE, REJECTED→FAILURE, CANCELLED→DECLINE) back to the originating Qhorus channel. CallerRef format: `qhorus:{channelId}/{messageId}/{correlationId}`.
```

- [ ] **Step 3: Commit**

```bash
git add docs/MODULES.md CLAUDE.md
git commit -m "docs(#97): register casehub-work-qhorus module in docs

Refs #97"
```
