# MCP Server: stdio Transport Support

**Issue:** casehubio/connectors#326
**Date:** 2026-11-20

## Problem

The standalone MCP server (`mcp-server-connectors`) only supports HTTP transport. stdio is the primary transport for local MCP clients (Claude Desktop, Cursor, langchain4j CLI). Without it, the server can't integrate with process-based MCP clients.

The root cause is a module boundary problem: `casehub-platform-mcp` mixes reusable MCP infrastructure with `CaseHubMcpTools`, which declares `@McpServer("casehub")` — a named server that conflicts with `quarkus-mcp-server-stdio`'s single-server requirement. The standalone server currently hacks around this with `quarkus.arc.exclude-types`.

## Design

### Part 1: Platform module split (casehubio/platform)

**Move `CaseHubMcpTools` out of `casehub-platform-mcp`.**

After the move, `casehub-platform-mcp` contains only reusable infrastructure:

| Class | Purpose |
|-------|---------|
| `GraphQLModelScanner` | Discovers `@McpDomain` beans, builds domain models |
| `DynamicToolRegistrar` | Registers tools via `ToolManager` (dispatch or flat mode) |
| `ReflectiveOperationDispatcher` | Dispatches operations to domain beans |
| `DomainFilter` | Filters which domains are exposed |
| `LandscapeReportService` | Cross-domain report aggregation |
| `McpResourceRegistryBridge` | Bridges platform resource SPI to Quarkus MCP resource API |
| `DomainResourceRegistrar` | Registers domain-index MCP resources |

**Destination for `CaseHubMcpTools`:** the full platform app module (outside this slot) that runs in dispatch mode with `@McpServer("casehub")`. If no single app module is obvious, create a thin `casehub-platform-mcp-dispatch` module that the full app depends on. This is a platform-side decision.

### Part 2: Standalone server transport (casehubio/connectors)

**Add `quarkus-mcp-server-stdio` dependency to `mcp-server-connectors`.**

Changes to `mcp-server-connectors/pom.xml`:
- Add `quarkus-mcp-server-stdio` dependency (version `${quarkus-mcp-server.version}`, currently 1.11.1)
- Remove `quarkus-mcp-server-sse` if it's pulled in transitively (verify — it comes via `casehub-platform-mcp` which depends on it)

Changes to `mcp-server-connectors/src/main/resources/application.properties`:
- Remove `quarkus.arc.exclude-types` line (no longer needed after Part 1)
- Remove `quarkus.http.host` and `quarkus.http.port` (no HTTP listener by default)
- Keep `quarkus.banner.enabled=false` (clean stdio output)

For HTTP opt-in, add a Maven profile `http` that activates `quarkus-mcp-server-sse` and sets HTTP config. Or use a config property — TBD during implementation.

### Part 3: Remove exclusion of McpResourceRegistryBridge and DomainResourceRegistrar

The standalone server currently excludes these via `quarkus.arc.exclude-types` alongside `CaseHubMcpTools`. They are actually reusable — `McpResourceRegistryBridge` provides MCP resources (domain index, domain detail), and `DomainResourceRegistrar` registers them. After Part 1 removes the named-server conflict, these should work in the standalone server and provide useful MCP resources to clients.

Verify that both beans resolve their CDI dependencies in the standalone server context. `McpResourceRegistryBridge` injects `ResourceManager`, `ResourceTemplateManager`, and `ResourceTemplateCompletionManager` — these come from the Quarkus MCP server extension (available with either transport). `DomainResourceRegistrar` injects `McpResourceRegistry` (provided by the bridge) and `DomainModelRegistry` (already available).

## Scope

- **In scope:** Module split in platform, stdio dependency + config in connectors, removal of exclude-types hack
- **Out of scope:** Modular installable bundle architecture (filed as #333)
- **Cross-repo:** Part 1 is a platform change, Parts 2-3 are connectors changes. Platform change must land first.

## Testing

- Existing `McpServerStartupTest` verifies tool registration — should continue passing with stdio transport
- Add a test verifying stdio transport is active (Quarkus MCP server provides test infrastructure for this)
- Verify MCP resources are registered (domain-index, domains/{domain}) after removing the exclusion

## References

- `platform/mcp/src/main/java/io/casehub/platform/mcp/CaseHubMcpTools.java` — the class to move
- `connectors/mcp-server-connectors/src/main/resources/application.properties:6` — the exclude-types hack
- `platform/mcp/pom.xml:33` — existing `quarkus-mcp-server-sse` dependency
- casehubio/connectors#324 — MCP server packaging epic
- casehubio/connectors#333 — modular MCP architecture (future)
- casehubio/casehub-mosaic#328 — Connection homepage (needs HTTP transport)
