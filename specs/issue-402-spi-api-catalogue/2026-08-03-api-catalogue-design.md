# API Catalogue — Design Spec

**Issue:** casehubio/parent#402
**Date:** 2026-08-03
**Status:** Draft (post-review revision)

---

## Problem

CaseHub's documentation has three layers, two of which are covered:

| Layer | What it answers | Status |
|-------|----------------|--------|
| **Contextual** (consumer/contributor guides) | When to use this? How does it fit? | Done (#377) |
| **Mechanical** (API reference) | What are the exact signatures, types, annotations? | **Missing** |
| **Live** (IntelliJ MCP) | Who implements this? Who calls this? | Available in-session |

The contextual layer (guides) documents intent and usage. The live layer (IntelliJ) answers cross-repo questions on demand. What's missing is a mechanical, always-fresh, RAG-indexable API reference — the precise type signatures, method contracts, and parameter types that an LLM needs when implementing against the platform.

### Why not just use IntelliJ MCP?

IntelliJ is session-bound. RAG pipelines, docs sites, and offline contexts can't query it. A committed markdown reference is loadable anywhere.

### Why not extend the consumer guides?

The guides are hand-written contextual documentation — they explain when and why. Mixing in machine-generated type signatures creates maintenance conflict: the generated content goes stale independently of the hand-written content, and regeneration would overwrite manual edits. Separate files, same directory.

### Design evolution from #359

The original #359 design proposed per-SPI markdown chunks (~50-100 lines each, ~50+ files) focused on interfaces with @DefaultBean implementations. Through brainstorming, this evolved in two ways:

1. **Scope broadened** from SPI-only to full public API — an LLM needs APIs it calls and types it constructs, not just SPIs it implements.
2. **Shape changed** from hand-written/LLM-generated chunks to machine-generated doclet output — existing tools (jmarkdoc, TypeDoc) produce accurate API references without custom extraction code.

The cross-repo implementation matrix (which apps implement which SPIs) remains as parent's unique contribution, but the base documentation is now mechanically generated rather than manually curated.

---

## Design Decisions

### 1. Use existing tools, don't build custom extractors

**Java:** Javadoc markdown doclet — either [jmarkdoc](https://github.com/AdamBien/jmarkdoc) (Adam Bien, designed for agent/RAG contexts, requires JDK 25+ to run) or [neuhalje/markdown-doclet](https://github.com/neuhalje/markdown-doclet) (integrates with maven-javadoc-plugin, broader JDK support). Both generate markdown from Java source using the javadoc API — full type resolution, generics, annotations, javadoc comments.

**TypeScript:** [TypeDoc](https://typedoc.org/) + [typedoc-plugin-markdown](https://typedoc-plugin-markdown.org/) (v4.12+). Uses the TypeScript compiler internally — full type resolution. Generates markdown directly. Respects barrel exports, so only exported types are documented.

**Why not a Python/regex approach:** Cross-module type resolution (generics, indirect implementations, type hierarchies) requires an AST. Regex parsing misses indirect `implements`, generic type parameters, and cross-package references. The existing tools use the language's own compiler/parser — they're correct by construction.

### 2. Scope: full public API, not just SPIs

The catalogue covers every public type in the consumer-facing API surface — interfaces, classes, records, enums. Not just SPIs (interfaces with @DefaultBean). An LLM implementing an app needs:

- **SPIs** it implements (AgentRoutingStrategy, WorkerSelectionStrategy)
- **APIs** it calls (CaseHubRuntime, MessageDispatcher)
- **Types** it constructs (CaseDefinition, AgentDescriptor)
- **Enums/constants** it references (WorkItemStatus, Priority)

### 3. Scope filtering is per-repo, not centralised

Each repo owns its own filtering configuration as part of its build config:

- **Java repos with `-api` modules:** the maven-javadoc-plugin is configured on the `-api` module only — the Maven module boundary IS the filter
- **Java repos with `*.api` packages:** the javadoc plugin's `<sourceFileIncludes>` or package filter specifies which packages to scan
- **Java repos with explicit config (e.g., blocks):** the javadoc plugin config lists specific packages
- **TypeScript repos:** TypeDoc follows barrel exports from `index.ts` — the module's export declarations ARE the filter

No centralised filtering config in parent. Each repo decides what's public. Parent only aggregates whatever each repo generates.

### 4. Decentralised generation, centralised aggregation

Same pattern as the #377 consumer/contributor guides:

- **Each repo** generates its own API docs into `docs/guides/api/` using its own CI
- **Parent** aggregates via the existing git subtree sync (`scripts/sync-guides.sh` — already syncs `docs/guides/`)
- **Parent** adds the cross-repo layer that individual repos can't produce

No new sync infrastructure. The subtree split on `docs/guides/` picks up the `api/` subdirectory automatically.

**Note:** `sync-guides.sh` currently gates on `consumer-guide.md` or `contributor-guide.md` existing. The gate condition must be updated to also trigger when `docs/guides/api/` exists — otherwise repos that only have generated API docs (no hand-written guides) would be silently skipped.

### 5. Cross-repo implementation matrix is parent's unique contribution

Individual repos can document their own types. Only parent can correlate across repos: "who implements AgentRoutingStrategy across the ecosystem?" This is the one custom piece — a `cross-repo-implementations.md` generated by parent's CI after aggregation.

**How it works:** The per-repo doclet output already includes type hierarchy information (which interfaces a class implements, which classes it extends). The cross-repo overlay script reads the generated markdown from all aggregated repos, extracts `implements`/`extends` declarations from the structured output, and correlates them. No source code scanning needed — the doclets already did the hard AST work. This avoids the regex-on-source contradiction: the overlay reads structured doclet output, not raw Java source.

### 6. Generated output is never hand-edited

All files in `docs/guides/api/` carry a `<!-- Generated — do not edit -->` header. Regeneration deletes the entire `docs/guides/api/` directory and recreates it from scratch — this prevents stale files from persisting when types are removed from the API surface. Contextual documentation stays in the hand-written guides.

### 7. Diff-gated commits — no timestamps, no noise

Generated files must NOT include timestamps, build IDs, or any non-deterministic content. The doclet/TypeDoc output must be identical for identical input — same source → same markdown, byte-for-byte.

CI commits only when `git diff --quiet docs/guides/api/` fails (content actually changed). This prevents endless identical commits on every build. If the API source didn't change, the generated output is identical, the diff is empty, no commit happens.

**Deterministic output must be verified** during tool evaluation (implementation step 1). If the chosen tool includes non-deterministic elements (member ordering, timestamps), configure or post-process to make output stable.

### 8. Consumer/caller data is out of scope

The original #359 design included "Used By" sections showing who calls each SPI. This requires IntelliJ-level cross-repo reference analysis which the doclet tools cannot provide. The cross-repo-implementations.md covers the "who implements" dimension. The "who calls" dimension remains available via IntelliJ MCP in-session (`ide_find_references`). If a static "Used By" capability is needed later, it would require a separate IntelliJ-driven generation step.

---

## Architecture

### Per-repo structure (generated)

```
<repo>/
  docs/guides/
    consumer-guide.md          (existing, hand-written)
    contributor-guide.md       (existing, hand-written)
    api/                       (NEW, generated — deleted and recreated each run)
      index.md                 (module-level overview)
      AgentRoutingStrategy.md  (per-type file)
      CaseDefinition.md
      ...
```

### Parent structure (aggregated + parent-owned)

```
parent/
  docs/
    repos/                             (aggregated via subtree sync)
      casehub-engine/
        consumer-guide.md
        contributor-guide.md
        api/                           (arrives via subtree sync)
          index.md
          AgentRoutingStrategy.md
          ...
      casehub-work/
        ...
    api/                               (parent-owned, NOT aggregated)
      INDEX.md                         (platform-wide API discovery — links to docs/repos/*/api/)
      cross-repo-implementations.md    (SPI overlay — cross-repo correlation)
```

`docs/repos/*/api/` contains per-repo API docs (aggregated from child repos). `docs/api/` contains parent-authored cross-repo content. The path distinction makes ownership clear: `repos/` is aggregated, `api/` is parent-owned.

### Per-type file format (Java example)

The exact format depends on the chosen doclet tool. The output must include at minimum:

```markdown
<!-- Generated — do not edit -->
# AgentRoutingStrategy

> `io.casehub.api.spi.routing` · casehub-engine-api

Single-responsibility strategy for selecting an agent implementation
for a given capability request.

## Methods

### route

```java
RoutingResult route(AgentRoutingContext context)
```

Select an agent for the given routing context.

**Parameters:**
- `context` — `AgentRoutingContext` — the routing input including
  case context, capability, candidate agents, and routing signals

**Returns:** `RoutingResult` — the selected agent with confidence
score and routing metadata

## Implements / Extends

(doclet-generated type hierarchy information — used by the cross-repo overlay)

## See Also

- `AgentRoutingContext` — [AgentRoutingContext.md](AgentRoutingContext.md)
- `RoutingResult` — [RoutingResult.md](RoutingResult.md)
```

The "Implements / Extends" section is what the cross-repo overlay reads to build the implementation matrix. If the chosen doclet doesn't include this, the overlay script would need to add a lightweight structured metadata section during generation.

### Cross-repo implementations file format

```markdown
<!-- Generated — do not edit -->
# Cross-Repo SPI Implementations

SPIs with implementations across multiple repos. For each SPI:
the interface location, default bean, and every implementation
found across the platform.

## AgentRoutingStrategy

> `io.casehub.api.spi.routing` · casehub-engine-api
> Default: `ComposableAgentRoutingStrategy` (casehub-engine)

| Repo | Implementation | Summary |
|------|---------------|---------|
| blocks | `LlmAgentRoutingStrategy` | LLM-reasoned agent selection |
| blocks | `CbrAgentRoutingStrategy` | CBR-evidence based routing |
| engine | `ComposableAgentRoutingStrategy` | Composes signal providers (default) |

...
```

### Platform-wide INDEX.md

Links to each repo's aggregated `api/index.md` with a brief description of what that repo's API surface covers. Same role as `consumer-index.md` but for the mechanical API reference.

---

## Generation Pipeline

### Step 1: Per-repo generation (each repo's CI)

**Java repos:**

Add a maven-javadoc-plugin execution (or standalone jmarkdoc invocation) to the repo's build config. The plugin configuration in each repo's `pom.xml` specifies:
- Source paths (scoped to the `-api` module, or specific packages)
- Output directory: `docs/guides/api/`
- Format: markdown (via the doclet)

The generation step:
1. Delete `docs/guides/api/` entirely (clean slate — prevents stale files)
2. Run the doclet
3. `git diff --quiet docs/guides/api/` — only commit if content changed
4. If changed: `git commit` with `[skip ci]` in the message (prevents re-triggering)

**TypeScript repos (pages, blocks-ui):**

Add a `typedoc` npm script with `typedoc-plugin-markdown`. Configuration in `typedoc.json` within each repo:
- Entry points (barrel exports from `index.ts`)
- Output directory: `docs/guides/api/`
- Format options: table layout for properties, GFM-compatible

Same clean-slate + diff-gate + `[skip ci]` pattern as Java.

**TypeScript-specific considerations:**
- blocks-ui is a monorepo with workspaces (`packages/*/src/index.ts`). TypeDoc supports multiple entry points — configure one per workspace package.
- TypeDoc resolves types from `tsconfig.json`. Ensure `tsconfig.json` includes all packages that should be documented.
- Web component metadata (attributes, events, slots) may require a TypeDoc plugin or custom theme to extract from decorators/static fields.

**CI trigger:** Per-repo workflows trigger on changes to source files (`src/`), NOT on changes to `docs/`. This prevents commit loops where generated docs trigger another generation run.

**CI permissions:** Workflows that commit generated docs back to the repo need `contents: write` permission. Use `permissions: contents: write` in the workflow YAML, or use a PAT with repo scope.

### Step 2: Aggregation (parent CI)

`scripts/sync-guides.sh` syncs `docs/guides/` from each repo into `docs/repos/<name>/`. The `api/` subdirectory is included automatically.

**Required change:** Update the gate condition in sync-guides.sh (currently line ~63) to also accept repos that have `docs/guides/api/` but no hand-written guides:

```bash
# Current gate (too narrow):
if [ ! -f "consumer-guide.md" ] && [ ! -f "contributor-guide.md" ]; then

# Updated gate:
if [ ! -f "consumer-guide.md" ] && [ ! -f "contributor-guide.md" ] && [ ! -d "api" ]; then
```

### Step 3: Cross-repo overlay (parent CI)

After aggregation, a script reads the aggregated `docs/repos/*/api/` files to build `cross-repo-implementations.md`:

1. Walk all `docs/repos/*/api/*.md` files
2. Parse each file for "Implements" / "Extends" sections (structured data from doclet output)
3. For each interface found, collect all implementing classes across repos
4. Generate the implementation matrix grouped by SPI interface

This reads structured doclet output, not raw source code — the doclets already resolved type hierarchies correctly. The overlay is a correlation step, not an analysis step.

### Step 4: Commit and push

If any generated files changed (`git diff --quiet`), commit to parent main with `[skip ci]`.

---

## CI Workflow

### Per-repo CI (added to each repo's existing workflow)

**Trigger:** Push to main, filtered to source file paths only (`src/**`, `api/src/**`). Does NOT trigger on `docs/` changes.

**Steps:**
1. Delete `docs/guides/api/`
2. Run jmarkdoc/TypeDoc on API sources
3. If `docs/guides/api/` changed → commit with `[skip ci]`

### Parent CI: `generate-api-catalogue.yml`

**Triggers:**
- Push to main (when `docs/repos/` changes — i.e., after a sync-guides run)
- Weekly schedule (catches accumulated changes)
- Manual dispatch

**Steps:**
1. Run sync-guides.sh --local (aggregates latest guides including api/)
2. Run cross-repo overlay script → `docs/api/cross-repo-implementations.md`
3. Update `docs/api/INDEX.md`
4. If diff exists → commit with `[skip ci]` and push

**Loop prevention:** All automated commits use `[skip ci]` in the commit message. CI triggers are path-filtered to source directories only. This breaks the potential loop: source change → per-repo generation → sync to parent → overlay generation → done (no re-trigger).

---

## Tool Selection: jmarkdoc vs neuhalje

Decision deferred to implementation step 1. Evaluate both against engine-api:

| Criterion | jmarkdoc | neuhalje/markdown-doclet |
|-----------|----------|--------------------------|
| JDK requirement | JDK 25+ (to run, not to analyze) | Standard javadoc API |
| Maven integration | Standalone JAR | maven-javadoc-plugin |
| RAG-optimized output | Yes (designed for it) | Generic markdown |
| Maintenance | New, active (Adam Bien) | Older, less active |
| Output quality | Evaluate | Evaluate |
| Deterministic output | Verify | Verify |
| Type hierarchy in output | Verify | Verify |

**Evaluation criteria (implementation step 1):**
1. Output quality — readable, well-structured markdown
2. Deterministic — identical output for identical input (no timestamps, stable ordering)
3. Type hierarchy — includes implements/extends information (needed for cross-repo overlay)
4. JDK compatibility — if jmarkdoc requires JDK 25+, verify CI runners support it (the analyzed code is still JDK 21)
5. Integration simplicity — Maven plugin vs standalone JAR

---

## Relationship to Existing Documentation

| Layer | What | Authored by | Where |
|-------|------|-------------|-------|
| Consumer/contributor guides | When to use, how it fits, patterns | Hand-written (LLM-assisted) per repo | `docs/guides/*.md` |
| API catalogue (this spec) | Exact signatures, types, annotations | Machine-generated per repo | `docs/guides/api/*.md` |
| Cross-repo implementations | Who implements what, across repos | Machine-generated in parent | `docs/api/cross-repo-implementations.md` |
| Platform indexes | Discovery entry points | Hand-maintained in parent | `docs/consumer-index.md`, `docs/api/INDEX.md` |
| Live queries | Ad-hoc cross-repo lookups | IntelliJ MCP in-session | Not persisted |

No duplication between layers. Guides explain intent. API catalogue documents contracts. Cross-repo overlay correlates implementations. Indexes enable discovery. IntelliJ answers one-off questions.

---

## What's NOT in Scope

- **Restructuring repos** to add `-api` modules or `api` packages — the tool adapts to current structure via per-repo config
- **Consumer/caller documentation** ("Used By" sections) — requires IntelliJ-level analysis the doclets can't provide. Available via `ide_find_references` in-session.
- **Audit tooling** (#359 part 1) — separate concern, separate issue
- **Implementation pattern catalogue** (#359 addendum) — separate concern
- **Human-facing docs site** — the markdown is the deliverable; rendering to HTML is a future concern
- **Javadoc replacement** — the API catalogue complements javadoc, doesn't replace it

---

## Implementation Sequence

1. **Evaluate Java doclet** — run jmarkdoc and neuhalje against engine-api. Check output quality, determinism, type hierarchy inclusion, JDK compatibility.
2. **Evaluate TypeDoc** — run typedoc + typedoc-plugin-markdown against pages. Check output quality, determinism, monorepo support (blocks-ui workspaces), web component metadata extraction.
3. **Configure one Java repo** (engine) — add doclet config to engine-api pom.xml, generate docs/guides/api/, verify output, commit
4. **Configure one TS repo** (pages) — add typedoc config, generate docs/guides/api/, verify output, commit
5. **Update sync-guides.sh gate** — add `docs/guides/api/` directory check to the gate condition
6. **Sync to parent** — run sync-guides.sh --local, verify aggregation picks up api/ subdirectory
7. **Build cross-repo overlay** — Python script reading aggregated doclet output, generating cross-repo-implementations.md
8. **Write docs/api/INDEX.md** — platform-wide API discovery index linking to docs/repos/*/api/
9. **Roll out to remaining Java repos** — add doclet config per repo. Verify each repo's `-api` module or package filter is correct before configuring.
10. **Roll out to remaining TS repos** (blocks-ui) — add typedoc config
11. **Per-repo CI** — add generation step to each repo's workflow (source-path-filtered trigger, `[skip ci]` commits, `contents: write` permission)
12. **Parent CI workflow** — add generate-api-catalogue.yml (sync + overlay + index)
13. **Update docs/INDEX.md and consumer-index.md** — link to new API catalogue

---

## Review Findings Addressed

| # | Finding | Resolution |
|---|---------|------------|
| 1 | Regex/AST contradiction in cross-repo overlay | Overlay reads structured doclet output, not source. Doclets do the AST work. |
| 2 | Centralised config consumed by decentralised CI | Eliminated centralised config. Each repo owns its filtering via build config. |
| 3 | CI commit loop | Path-filtered triggers (source only) + `[skip ci]` commit messages. |
| 4 | sync-guides.sh gate skips api-only repos | Gate updated to also check for `docs/guides/api/` directory. |
| 5 | Scope divergence from #359/#402 | Added "Design Evolution" section explaining the broadening. |
| 6 | Non-existent modules in config | Removed centralised config. Per-repo build config is always correct for that repo. |
| 7 | No consumer/caller data | Acknowledged as out of scope. Added Decision 8. |
| 8 | Two `docs/api/` directories confusing | Clarified: `docs/repos/*/api/` is aggregated, `docs/api/` is parent-owned. |
| 9 | Stale file cleanup | Generation deletes `docs/guides/api/` before regenerating. |
| 10 | TypeScript pipeline unexamined | Added TS-specific considerations (monorepo, web components, tsconfig). |
| 11 | CI write permissions | Documented `contents: write` requirement. |
| 12 | Deterministic output unverified | Added as evaluation criterion in tool selection. |
