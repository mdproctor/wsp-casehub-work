# MCP Server: stdio Transport Support — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #326 — MCP server: stdio transport support
**Issue group:** #326

**Goal:** Enable stdio transport for the standalone MCP server by fixing the module boundary in platform (move `CaseHubMcpTools` out) and adding `quarkus-mcp-server-stdio` to `mcp-server-connectors`.

**Architecture:** Create a new `casehub-platform-mcp-dispatch` module in the platform repo to hold `CaseHubMcpTools` and its tests. This separates the named-server dispatch tools from the reusable MCP infrastructure in `casehub-platform-mcp`. Then update the connectors standalone server to use stdio transport and remove the `quarkus.arc.exclude-types` hack.

**Tech Stack:** Java 21, Quarkus 3.32.2, quarkus-mcp-server-stdio 1.11.1

## Global Constraints

- Quarkus version: 3.32.2
- quarkus-mcp-server extension version: 1.11.1
- All casehubio artifacts: 0.4-SNAPSHOT
- Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`

---

## Batch 1: Platform module split — extract CaseHubMcpTools

### Task 1: Create casehub-platform-mcp-dispatch module and move CaseHubMcpTools

**Files:**
- Create: `platform/mcp-dispatch/pom.xml`
- Move: `platform/mcp/src/main/java/io/casehub/platform/mcp/CaseHubMcpTools.java` → `platform/mcp-dispatch/src/main/java/io/casehub/platform/mcp/dispatch/CaseHubMcpTools.java` (use `ide_move_file`)
- Move: `platform/mcp/src/test/java/io/casehub/platform/mcp/CaseHubMcpToolsTest.java` → `platform/mcp-dispatch/src/test/java/io/casehub/platform/mcp/dispatch/CaseHubMcpToolsTest.java` (use `ide_move_file`)
- Move: `platform/mcp/src/test/java/io/casehub/platform/mcp/McpModelComprehensionIT.java` → `platform/mcp-dispatch/src/test/java/io/casehub/platform/mcp/dispatch/McpModelComprehensionIT.java` (use `ide_move_file`)
- Modify: `platform/pom.xml` — add `mcp-dispatch` to `<modules>`
- Test: moved test classes

**Interfaces:**
- Consumes: `DomainModelRegistry` (from `casehub-platform-mcp`), `DomainContentFormatter` (from `casehub-platform-mcp-core`), `SearchResult` (from `casehub-platform-mcp-core`)
- Produces: `CaseHubMcpTools` CDI bean with `@McpServer("casehub")`, `casehub_model` and `casehub_search` tools — consumed by the full platform app (outside this slot)

- [ ] **Step 1: Create the mcp-dispatch module POM**

Create `platform/mcp-dispatch/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-platform-mcp-dispatch</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform :: MCP Dispatch</name>
    <description>Named @McpServer("casehub") dispatch tools — casehub_model and casehub_search.
        Only needed by the full platform app running in dispatch mode.
        Not needed by standalone MCP servers using flat tool exposure.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-mcp-core</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.mcp</groupId>
            <artifactId>quarkus-mcp-server-sse</artifactId>
            <version>1.11.1</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform</artifactId>
            <version>${project.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-mcp</artifactId>
            <version>${project.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-api</artifactId>
            <version>${project.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-claude</artifactId>
            <version>${project.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-session-core</artifactId>
            <version>${project.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>**/*IT.java</exclude>
                    </excludes>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>generate-code</goal>
                            <goal>generate-code-tests</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add mcp-dispatch to the platform parent POM**

In `platform/pom.xml`, add `<module>mcp-dispatch</module>` after the `<module>mcp</module>` entry in the `<modules>` section.

- [ ] **Step 3: Create source directories**

```bash
mkdir -p platform/mcp-dispatch/src/main/java/io/casehub/platform/mcp/dispatch
mkdir -p platform/mcp-dispatch/src/test/java/io/casehub/platform/mcp/dispatch
```

- [ ] **Step 4: Move CaseHubMcpTools to the new module**

Use `ide_move_file` to move `CaseHubMcpTools.java` from `platform/mcp/src/main/java/io/casehub/platform/mcp/` to `platform/mcp-dispatch/src/main/java/io/casehub/platform/mcp/dispatch/`. This updates the package declaration and import references.

- [ ] **Step 5: Move CaseHubMcpToolsTest to the new module**

Use `ide_move_file` to move `CaseHubMcpToolsTest.java` from `platform/mcp/src/test/java/io/casehub/platform/mcp/` to `platform/mcp-dispatch/src/test/java/io/casehub/platform/mcp/dispatch/`.

- [ ] **Step 6: Move McpModelComprehensionIT to the new module**

Use `ide_move_file` to move `McpModelComprehensionIT.java` from `platform/mcp/src/test/java/io/casehub/platform/mcp/` to `platform/mcp-dispatch/src/test/java/io/casehub/platform/mcp/dispatch/`.

- [ ] **Step 7: Verify platform builds**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl mcp,mcp-dispatch -am`
(from the platform directory)

Expected: BUILD SUCCESS — `mcp` builds without `CaseHubMcpTools`, `mcp-dispatch` builds with it.

- [ ] **Step 8: Commit**

```bash
git add platform/mcp-dispatch/ platform/pom.xml platform/mcp/
git commit -m "refactor(mcp): extract CaseHubMcpTools to mcp-dispatch module

Move the named @McpServer(\"casehub\") dispatch tools to their own module
so casehub-platform-mcp is pure reusable MCP infrastructure. Standalone
MCP servers can depend on the infrastructure without inheriting the named
server that conflicts with stdio transport.

Refs casehubio/connectors#326"
```

---

## Batch 2: Standalone server — stdio transport and cleanup

### Task 2: Add stdio transport and remove exclude-types hack

**Files:**
- Modify: `connectors/mcp-server-connectors/pom.xml` — add `quarkus-mcp-server-stdio` dependency
- Modify: `connectors/mcp-server-connectors/src/main/resources/application.properties` — remove `exclude-types`, remove HTTP config, keep stdio-friendly settings
- Test: `connectors/mcp-server-connectors/src/test/java/io/casehub/connectors/mcp/McpServerStartupTest.java`

**Interfaces:**
- Consumes: `casehub-platform-mcp` (now without `CaseHubMcpTools`), `quarkus-mcp-server-stdio`
- Produces: standalone MCP server runnable via stdio

- [ ] **Step 1: Verify the existing test passes before changes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp-server-connectors`
Expected: BUILD SUCCESS — baseline green.

- [ ] **Step 2: Add quarkus-mcp-server-stdio dependency to POM**

In `connectors/mcp-server-connectors/pom.xml`, add after the `quarkus-security` dependency:

```xml
<!-- MCP stdio transport (default for local MCP clients) -->
<dependency>
    <groupId>io.quarkiverse.mcp</groupId>
    <artifactId>quarkus-mcp-server-stdio</artifactId>
    <version>${quarkus-mcp-server.version}</version>
</dependency>
```

- [ ] **Step 3: Update application.properties**

Replace the content of `connectors/mcp-server-connectors/src/main/resources/application.properties` with:

```properties
# CaseHub MCP Server — Connectors
# Run: java -jar target/quarkus-app/quarkus-run.jar

# Tool exposure: flat (individual tools) for external MCP clients
casehub.mcp.tool-exposure=flat

# Domain mode: discover (all on classpath) or explicit (filtered)
casehub.mcp.mode=discover
# casehub.mcp.domains=chat,calendar,document,location

# Prevent ArC from removing @McpDomain beans — they're discovered via BeanManager
# at runtime by GraphQLModelScanner, not via CDI injection.
quarkus.arc.unremovable-types=io.casehub.connectors.graphql.**

# Suppress @Blocking validation — domain API methods are entrypoints in the full
# platform (via generated REST endpoints) but here they're called reflectively by
# the MCP tool handler which already runs on a worker thread (blocking=true).
quarkus.execution-model-annotations.detection-mode=disabled

# Seed corpus directory — resolved relative to working directory
casehub.ref.seed-dir=corpus/

# Disable banner for clean stdio
quarkus.banner.enabled=false
```

Key changes:
- Removed `quarkus.arc.exclude-types` (no longer needed — `CaseHubMcpTools` is in a separate module)
- Removed `quarkus.http.host` and `quarkus.http.port` (no HTTP listener with stdio transport)

- [ ] **Step 4: Add an HTTP profile for opt-in HTTP transport**

Add to `connectors/mcp-server-connectors/pom.xml`, inside `<profiles>` after the existing `native` profile:

```xml
<profile>
    <id>http</id>
    <activation>
        <property><name>http</name></property>
    </activation>
    <dependencies>
        <dependency>
            <groupId>io.quarkiverse.mcp</groupId>
            <artifactId>quarkus-mcp-server-sse</artifactId>
            <version>${quarkus-mcp-server.version}</version>
        </dependency>
    </dependencies>
    <properties>
        <quarkus.http.host>127.0.0.1</quarkus.http.host>
        <quarkus.http.port>8080</quarkus.http.port>
    </properties>
</profile>
```

- [ ] **Step 5: Run existing tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp-server-connectors`
Expected: BUILD SUCCESS — `McpServerStartupTest` passes (tool registration works the same with stdio transport).

- [ ] **Step 6: Write a test verifying MCP resources are registered**

The resources were previously excluded via `quarkus.arc.exclude-types`. Now that the exclusion is removed, `McpResourceRegistryBridge` and `DomainResourceRegistrar` should be active and register domain resources.

Add to `McpServerStartupTest.java`:

```java
@Inject
io.casehub.platform.api.mcp.McpResourceRegistry resourceRegistry;

@Test
void domainResourcesRegistered() {
    assertThat(resourceRegistry.list()).isNotEmpty();
    assertThat(resourceRegistry.resolve("casehub-domain-index")).isPresent();
    assertThat(resourceRegistry.resolve("casehub-domains")).isPresent();
}
```

- [ ] **Step 7: Run the test to verify resources are registered**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl mcp-server-connectors`
Expected: BUILD SUCCESS — domain resources are registered.

If the test fails because `McpResourceRegistry` has no CDI producer in the standalone context, add a `@DefaultBean` producer in `McpServerBeans`:

```java
@Produces @DefaultBean @ApplicationScoped
public McpResourceRegistry mcpResourceRegistry(
        ResourceManager resourceManager,
        ResourceTemplateManager resourceTemplateManager,
        ResourceTemplateCompletionManager completionManager,
        Event<McpResourceRegistered> registeredEvent) {
    // McpResourceRegistryBridge is already @ApplicationScoped —
    // ArC will discover it directly. This fallback is only needed
    // if the bridge's dependencies aren't satisfied.
    return new io.casehub.platform.api.mcp.NoOpMcpResourceRegistry();
}
```

However, `McpResourceRegistryBridge` is `@ApplicationScoped` and its dependencies (`ResourceManager`, `ResourceTemplateManager`, `ResourceTemplateCompletionManager`) come from the Quarkus MCP server extension (available with stdio transport too), so it should resolve without a fallback. Verify empirically.

- [ ] **Step 8: Run the full module build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl mcp-server-connectors`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add mcp-server-connectors/
git commit -m "feat(mcp-server): add stdio transport, remove exclude-types hack

Add quarkus-mcp-server-stdio as default transport for the standalone MCP
server. Remove quarkus.arc.exclude-types — CaseHubMcpTools is now in a
separate platform module (mcp-dispatch), so the infrastructure module is
clean. MCP resources (domain-index, domains/{domain}) are now active.

HTTP transport available via -Dhttp Maven profile for the Connection
homepage use case.

Refs #326"
```

---

## References

- [2026-11-20-stdio-transport-design.md] — design spec this plan implements
- `platform/mcp/src/main/java/io/casehub/platform/mcp/CaseHubMcpTools.java` — the class to move
- `platform/mcp/src/test/java/io/casehub/platform/mcp/CaseHubMcpToolsTest.java` — test to move
- `platform/mcp/src/test/java/io/casehub/platform/mcp/McpModelComprehensionIT.java` — integration test to move
- `connectors/mcp-server-connectors/src/main/resources/application.properties:6` — the exclude-types hack to remove
- `connectors/mcp-server-connectors/pom.xml` — add stdio dependency
- `platform/mcp/pom.xml` — existing mcp module POM (reference for dep structure)
- GitHub #326 — focal issue
- GitHub #333 — modular MCP architecture (future, filed during brainstorming)
