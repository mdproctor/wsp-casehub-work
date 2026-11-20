## D1: Module split strategy

**Choice:** Move `CaseHubMcpTools` out of `casehub-platform-mcp` into the platform app module
**Alternatives:**
- Split `casehub-platform-mcp` into `mcp-runtime` + `mcp-dispatch` — more structural but moves more code for a single-class problem
- `@IfBuildProperty` conditional on `CaseHubMcpTools` — works around the boundary instead of fixing it
- `quarkus.arc.exclude-types` (status quo) — fragile hack, breaks when classes are renamed
**Rationale:** `CaseHubMcpTools` is the only class with `@McpServer("casehub")`. Everything else in `casehub-platform-mcp` is reusable infrastructure. Moving one class fixes the boundary cleanly. Push reusable infrastructure down into platform for re-use; app-specific wiring stays in the app.
**Trade-offs:** Platform app module gains a dependency on quarkus-mcp-server (for `@McpServer` annotation), but it already has this transitively.
**Sources:** `platform/mcp/src/main/java/io/casehub/platform/mcp/CaseHubMcpTools.java`, `connectors/mcp-server-connectors/src/main/resources/application.properties` (exclude-types hack)
**Exploration:** quick
**Status:** captured

## D2: Default transport for standalone MCP server

**Choice:** stdio as default transport, HTTP opt-in via Maven profile or config override
**Alternatives:**
- Both active by default — unusual for stdio servers to also run HTTP; adds complexity
- Stdio only, no HTTP — loses the Connection homepage use case (#328)
**Rationale:** Claude Desktop, Cursor, and langchain4j CLI all expect process-based stdio MCP servers. HTTP is a secondary use case for the Connection homepage. Making stdio default and HTTP opt-in matches the primary audience.
**Trade-offs:** Users who want HTTP must explicitly enable it. The Connection homepage (#328) needs to know to activate the HTTP profile.
**Sources:** Issue #326 body, issue #328 (Connection homepage)
**Exploration:** quick
**Status:** captured

## D3: Modular MCP architecture (deferred)

**Choice:** Filed as casehubio/connectors#333 for future exploration — not in scope for this issue
**Alternatives:**
- Build the full modular bundle architecture now — too large, needs more thought on same-JVM value proposition
**Rationale:** The module split (D1) + stdio transport (D2) solves the immediate problem. The larger architectural direction needs design exploration around build-time vs runtime composition and whether same-JVM installs offer value.
**Trade-offs:** We're not solving the broader extensibility problem yet, but we're not creating barriers to it either — the split makes the infrastructure more composable.
**Sources:** Brainstorming conversation
**Exploration:** quick
**Status:** captured
