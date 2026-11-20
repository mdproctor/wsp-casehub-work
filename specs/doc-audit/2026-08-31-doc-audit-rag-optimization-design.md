# Documentation Audit, RAG Optimization & Freshness Gate — Design Spec

**Date:** 2026-08-31
**Branch:** doc-audit (to be created)
**Covers:** Platform-wide documentation freshness, RAG retrieval quality, work-end enforcement
**Status:** Draft

---

## 1. Problem Statement

CaseHub has 28 repos with 1,700+ commits of drift since consumer/contributor guides were last updated (Aug 3-10, 2026). The documentation structure — INDEX.md → consumer/contributor indexes → per-repo guides + arc42stories — is comprehensive but stale. An LLM starting a session cannot trust that guides accurately describe current capabilities, SPIs, or module structures.

Three interconnected problems:
1. **Staleness** — guides don't reflect current code
2. **RAG inefficiency** — monolithic 200-550 line guides load unnecessary content and chunk poorly for retrieval
3. **No enforcement** — nothing prevents docs from falling behind as code evolves

---

## 2. Deliverables

| # | Deliverable | Lifecycle |
|---|-------------|-----------|
| D-A | Initial audit — identify all stale sections across 28 repos | One-time (audit slot) |
| D-B | Fix all identified staleness using git history, issues, and blogs as sources | One-time (audit slot, per-repo sessions) |
| D-C | RAG optimization — decompose guides, add structural anchors, create capability index | Incremental (during D-B, largest guides first) |
| D-D | Work-end freshness gate — adversarial enforcement preventing future drift | Permanent (skill modification) |

---

## 3. Architecture

### 3.1 Structural Anchors (D4) — the foundation

Documentation sections declare the code elements they describe via **structural anchors** — class names, SPI interfaces, config keys, file paths, protocol references. These serve dual purpose:

1. **Staleness detection:** When an anchored element changes (detected via git diff), the anchoring section is flagged as candidate-stale
2. **RAG metadata:** Anchors appear in YAML frontmatter, enabling precise filtered retrieval via neocortex's PayloadFilter

Anchor format in documentation YAML frontmatter:
```yaml
---
capability: notifications
audience: consumer
repo: casehub-platform
anchors:
  classes:
    - io.casehub.platform.notification.NotificationBridge
    - io.casehub.platform.notification.SubscriptionEngine
  spis:
    - io.casehub.platform.notification.spi.DeliveryChannel
  config-keys:
    - casehub.notification.digest.interval
  protocols:
    - flyway-version-range-allocation
    - notification-delivery-contract
---
```

When any anchored element is renamed, moved, or deleted, the diff-based triage detects the change and flags the section for adversarial verification.

**Override mechanism:** A `verified-current: 2026-08-31 | commit:<hash>` annotation suppresses re-flagging until the next anchor change. Used when an LLM adversarial check confirms the section is accurate despite an anchor change (e.g., internal refactoring that doesn't affect the documented behavior). The commit hash records which verification produced the annotation, enabling audit trail and staleness detection for ancient overrides.

**Anchor renewal:** When an anchored element is renamed or moved, Phase 3's adversarial verification (§3.3) is responsible for updating both the section prose AND the anchor declarations in YAML frontmatter. This is an explicit output of the adversarial check — the subagent returns corrected anchors alongside corrected prose. If the subagent cannot resolve the new name (element was deleted, not renamed), it removes the anchor and flags the section for human review. The CI anchor integrity step (§7) detects broken anchors between adversarial runs.

### 3.2 Per-Repo Guide Decomposition (D5)

The 2026-07-07 platform doc restructuring decomposed the monolithic PLATFORM.md (685 lines) into 16+ topic-scoped chunks under `docs/platform/` with a thin `INDEX.md` as the discovery entry point. That restructuring left per-repo guides (consumer-guide.md, contributor-guide.md) monolithic — ranging from 119 to 546 lines. This spec extends the same pattern to per-repo guides: extract the largest sections into standalone capability chunks, leaving guides as thin routing documents.

**Relationship to existing indexes:** `docs/consumer-index.md` currently routes by repo to monolithic per-repo guides. After decomposition, `docs/capabilities.md` (§5) replaces `consumer-index.md` as the primary capability discovery point, routing by capability to individual chunks instead of by repo to monolithic guides. `docs/INDEX.md` remains the universal entry point for cross-cutting topics and architecture docs. Both consumer and contributor guides are decomposed — consumer content into `capabilities/` chunks, contributor content into `internals/` chunks.

**Distinction from platform/ topic chunks:** `docs/platform/notifications.md` documents the cross-cutting notification architecture (subscription engine, delivery pipeline, digest batching) for platform builders. `docs/repos/casehub-platform/capabilities/notifications.md` documents what an app builder needs to USE the notification system (APIs to call, SPIs to implement, configuration). These serve different audiences at different abstraction levels and do not duplicate each other.

**Chunk ownership under subtree aggregation:** Capability and internals chunks originate in child repos under `docs/guides/capabilities/` and `docs/guides/internals/`, alongside the existing `docs/guides/consumer-guide.md` and `docs/guides/contributor-guide.md`. The subtree sync mechanism aggregates them into the parent repo at `docs/repos/<repo>/capabilities/` and `docs/repos/<repo>/internals/`. The child repo remains the source of truth — decomposition happens there, not in the parent. This preserves the existing ownership model established by commit `0751d804` (decentralised repo deep-dives).

Following this precedent, guides are incrementally decomposed:

**Layer 1 — Capability Index** (`docs/capabilities.md`)
A single cross-repo file mapping capabilities to locations. The RAG entry point for ad-hoc retrieval:

```markdown
| Capability | Repo | Audience | Chunk |
|------------|------|----------|-------|
| Send notifications | platform | consumer | repos/casehub-platform/capabilities/notifications.md |
| Case lifecycle | engine | consumer | repos/casehub-engine/capabilities/case-lifecycle.md |
| Trust routing | ledger | consumer | repos/casehub-ledger/capabilities/trust-routing.md |
```

**Layer 2 — Per-Capability Chunks** (`docs/repos/<repo>/capabilities/<topic>.md`)
Self-contained files with YAML frontmatter. Each chunk is independently loadable for RAG retrieval:

```
docs/repos/casehub-platform/
├── capabilities/
│   ├── notifications.md        (extracted from consumer-guide § Notifications)
│   ├── identity.md             (extracted from consumer-guide § Identity)
│   ├── expressions.md          (extracted from consumer-guide § Expressions)
│   └── ...
├── internals/
│   ├── notification-pipeline.md (extracted from contributor-guide § Notification Dispatch)
│   └── ...
├── consumer-guide.md           (thin index linking to capabilities/)
└── contributor-guide.md        (thin index linking to internals/)
```

**Layer 3 — Guide Indexes** (existing `consumer-guide.md` / `contributor-guide.md`)
Become thin routing documents: a brief repo overview + links to capability chunks. For session loading, an LLM reads the guide index and pulls in specific chunks as needed.

**Decomposition priority:** Largest guides first — devtown (546 lines), qhorus (526 lines), ops (474 lines), ledger (448 lines), platform (418 lines), engine (408 lines). Extract sections exceeding 40 lines into standalone capability chunks (typically 3-5 per guide). Smaller guides (fsitrading at 119 lines, quarkmind at 186 lines) stay monolithic unless retrieval failures justify decomposition.

### 3.3 Staleness Detection — Hybrid Methodology (D4)

Three-phase detection:

**Phase 1 — Structural anchor check (mechanical, fast)**
For each file changed in the branch diff, check whether any documentation section anchors that file's classes, SPIs, or config keys. If yes, flag the section as candidate-stale.

**Phase 2 — Diff-based triage (transition fallback)**
For documentation sections without structural anchors (during the audit transition period), scan all un-anchored sections when any file in the repo changes. This is intentionally coarse — it trades precision for coverage. Sunset condition: once all sections have anchors (D3 exit criteria), diff-based triage is retired.

**Phase 3 — LLM adversarial verification (semantic)**
For each flagged section, dispatch a subagent that reads the section content alongside the actual current code. The subagent has two responsibilities:

1. **Falsify claims:** compare section prose against current code — "This section says X, but the code now does Y." Report findings as specific line-level corrections.
2. **Renew anchors:** verify that all YAML frontmatter anchors (class names, SPIs, config keys) still resolve in the codebase. For renamed elements, update the anchor to the new name. For deleted elements, remove the anchor and flag the section for review.

The subagent has access to IntelliJ MCP tools (`ide_find_class`, `ide_find_symbol`, `ide_find_references`) for semantic code navigation. Success criteria: each finding must cite a specific code location (file + line) that contradicts the documented claim, or a specific anchor that fails to resolve. Uncertain findings (the subagent cannot confirm or deny a claim) are reported as `UNCERTAIN` and excluded from precision/recall metrics but included in the human review queue. Output is a structured list of `{section, claim, evidence, verdict: STALE|CURRENT|UNCERTAIN, corrected_anchors[]}`.

### 3.4 Work-End Freshness Gate (D1, D2)

**Scope:** Module-tier classification determines check breadth:
- Changes to `api/` modules → check home repo guides AND consumer guides of dependent repos
- Changes to `runtime/` modules → check home repo guides only
- Changes to `testing/` modules → skip doc checks

**Enforcement model — dual:**
1. **Work-end hard gate (primary, LLM sessions):** After code review and before squash, the hybrid detection runs. If candidate-stale sections are found, the gate blocks until they're updated. Hard gate activation prerequisite: D4's structural anchor detection must demonstrate precision ≥80% AND recall ≥60% on the initial audit validation corpus (equivalently, F1 ≥ 0.69). Precision alone is insufficient — a detector that conservatively flags nothing would have vacuous precision but zero recall. The recall threshold ensures the detector catches a meaningful fraction of actually-stale sections.
2. **PR-level GitHub Action (complementary, all merge paths):** Covers human PRs, manual merges, CI pipeline merges — paths that bypass work-end. Runs structural anchor check only (no LLM adversarial — too slow for CI).

**Dependent repo handling:** When api/ changes affect dependent repos, the gate creates GitHub issues on those repos with specific stale-section details. Does not block the home repo's close.

**Integration with work-end orchestrator:** New step `doc_freshness_gate` added between `impl_doc_sync` and `adr` in the orchestrator's step sequence. When hard gate is not yet activated (pre-validation), runs in advisory mode (reports findings but doesn't block).

### 3.5 Arc42Stories Refresh (D7)

Arc42stories refresh runs at **epic close** (not per-branch), using two-tier verification.

**Trigger:** The `.plan` metadata includes an `arc42stories: true` flag, set at plan creation when the work involves architectural changes (new layers, module restructuring, SPI additions). When this flag is present and the `advance` command marks the last issue as done (no remaining issues in the plan), the arc42stories refresh runs as a post-advance hook. Plans without the `arc42stories` flag (e.g., a plan with 2 bug fixes) skip the refresh entirely. This prevents the refresh from firing on every plan completion — only architecturally-scoped plans trigger it.

**Tier 1 — 3-check sweep (structural assertions):**
1. Issue status: `gh issue view N` for every §12 reference — remove COMPLETED issues from Active Risks
2. Class name existence: `git ls-files` with qualified name resolution — for every §9.4 Key files entry, search for `<SimpleName>.java` and `<SimpleName>.kt` (covering both Java and Kotlin sources), then verify the file contains the expected package declaration matching the fully qualified name. For inner classes, search the outer class file. This replaces the fragile `find . -name "ClassName.java"` approach.
3. File path validity: verify all referenced file paths still exist

**Tier 2 — LLM adversarial check (prose sections):**
For the 5 prose subsection types in §9.4 (What it adds, Key wiring, Architectural decisions, Gotchas, Pattern to replicate), dispatch an LLM agent that reads the prose alongside current code and flags inaccuracies.

Only layers modified during the epic are checked — bounded scope.

### 3.6 Dependent Repo Detection (D6)

**Primary:** Automated POM analysis via scheduled GitHub Action on casehub-parent. The Action runs on a daily schedule (not triggered by child repo POM changes, since GitHub Actions in one repo cannot trigger on events in another repo). It clones all child repos, analyzes their POMs, and writes `dependency-graph.json` to the parent repo. The work-end gate reads this cached graph — no per-close penalty. Staleness window: up to 24 hours for newly added dependencies; acceptable because dependency changes are infrequent (typically 1-2 per week across all repos).

**On-demand refresh:** A `workflow_dispatch` trigger on the same Action allows manual refresh when a dependency change is known to have occurred. The work-end `doc_freshness_gate` step can invoke this via `gh workflow run` if the cached graph is older than the branch's POM changes.

**Supplementary:** `docs/platform/dependency-map.md` retains its Nature column (SPI signatures, runtime dep, compile scope) for change-type classification. Edge existence is automated; Nature annotations are manually maintained.

### 3.7 RAG Ingestion Pipeline (D5)

The YAML frontmatter on capability chunks enables RAG-quality filtered retrieval via neocortex's existing `CorpusIngestionService` infrastructure — the same framework that Hortora's knowledge garden uses for YAML-frontmatter Markdown ingestion.

**Architecture:** A new `CorpusIngestionBinding` for CaseHub documentation, registered via CDI:

| Component | Implementation | Role |
|-----------|---------------|------|
| `ChangeSource` | `FlatChangeSource` pointed at `docs/repos/*/capabilities/*.md` + `docs/platform/*.md` | Filesystem scanning with cursor-based delta detection |
| `MetadataExtractor` | New `DocMetadataExtractor` (implements `MetadataExtractor`) | Parses YAML frontmatter → `ExtractionResult(body, metadata, listMetadata)` where metadata includes `capability`, `audience`, `repo` and listMetadata includes `anchors` |
| `CorpusReader` | Existing filesystem reader | Reads file content as `byte[]` |
| `CorpusRef` | `new CorpusRef("<tenantId>", "casehub-docs")` | Per-tenant corpus isolation |

Neocortex already provides `YamlFrontmatterExtractor` — a `@DefaultBean` `MetadataExtractor` that parses flat key:value YAML frontmatter. `DocMetadataExtractor` extends this base pattern with full YAML parsing (via SnakeYAML, already a Quarkus transitive dependency) to handle the nested anchor structure (`anchors.classes`, `anchors.spis`, `anchors.config-keys`), mapping nested lists to `ExtractionResult.listMetadata`. The `CorpusIngestionService` handles everything else: LangChain4j document splitting, metadata propagation to all chunks (via `chunkDocument()` which creates `ChunkInput` records with the same metadata for every segment), dedup via `DedupEmbeddingIngestor`, cursor persistence via `CursorStore`, and reconciliation.

**Metadata propagation:** Each chunk inherits all frontmatter metadata from its source document. A 300-line capability chunk split into 3 RAG chunks produces 3 `ChunkInput` records, each carrying the same `metadata: {capability: "notifications", audience: "consumer", repo: "casehub-platform"}` and `listMetadata: {anchors: ["NotificationBridge", "SubscriptionEngine"]}`. `PayloadFilter` queries on any metadata field match all chunks from the document, not just the first.

**Retrieval integration:** Callers construct `PayloadFilter` queries from session context — e.g., `PayloadFilter.and(PayloadFilter.eq("repo", "casehub-platform"), PayloadFilter.eq("audience", "consumer"))` to retrieve consumer-facing platform documentation.

**Hosting model:** casehub-parent is a Maven BOM project with no Quarkus runtime — the `CorpusIngestionBinding` cannot run inside it. Following the `example-rag-pipeline` precedent in neocortex, a new `neocortex/doc-ingestion/` module provides a Quarkus CLI tool (`@QuarkusMain`) that:
1. Bootstraps CDI to auto-wire `EmbeddingIngestor` (Qdrant), `CursorStore`, and `DocumentSplitter`
2. Constructs a `CorpusIngestionBinding` with `FlatChangeSource` + `DocMetadataExtractor` pointed at the docs directory
3. Calls `service.reconcile("casehub-docs", binding, splitter)` and exits

This follows the same standalone pattern as `FlatCorpusIngestDemo`: construct the binding, call `processBinding()` or `reconcile()`, exit. No long-running application required.

**Trigger:** The CI daily scheduled Action (§3.6) runs `doc-ingestion-cli reconcile --docs <path>` after cloning casehub-parent. Local development invokes the same CLI manually. The documentation binding does NOT use `AUTO` mode (filesystem watching) — documentation ingestion is a batch operation, not a continuous process.

---

## 4. Initial Audit Execution (D3)

### 4.1 Slot Setup

Create a dedicated audit slot that pulls in all 28 repos. The slot enables cross-repo audit work without blocking feature branches.

### 4.2 Priority Order

Foundation repos first (highest blast radius → most downstream consumers):

| Wave | Repos | Rationale |
|------|-------|-----------|
| 1 (foundation) | platform, worker, ledger, connectors, work, qhorus, eidos, neocortex, engine, iot | API/SPI changes cascade to all consumers |
| 2 (orchestration) | ras, desiredstate, blocks, blocks-ui, claudony, openclaw, workers, ops, pages | Integration layer — consumes foundation, consumed by apps |
| 3 (application) | devtown, aml, clinical, life, drafthouse, quarkmind, soc, fsitrading, chat-app | Leaf nodes — consume platform, not consumed |

### 4.3 Per-Repo Audit Process

For each repo:

1. **Delta analysis:** `git log --since=<guide-last-updated> --oneline` — classify changes by impact (new modules, renamed types, new SPIs, removed features, architectural shifts)
2. **Source mining:** Read blog entries (workspace `blog/` + published `casehubio.github.io/_articles/`), GitHub issues, and git commit messages to reconstruct what changed and why
3. **Guide update:** Fix consumer guide, contributor guide sections affected by the delta
4. **Structural anchor insertion:** Add YAML frontmatter with anchors to each updated section (or new capability chunk if decomposing)
5. **Arc42stories check:** Run 3-check sweep on ARC42STORIES.MD if it exists
6. **Commit + close issue:** Commit updates, close the per-repo audit issue with evidence

### 4.4 Exit Criteria Per Repo

1. All consumer guide sections verified against current code by LLM adversarial check
2. All structural anchors validated via automated assertion (anchored class/SPI exists in codebase)
3. Per-repo GitHub issue closed with evidence commit

### 4.5 Validation Corpus

The first-wave audit produces a labeled corpus from the first 5 foundation repos audited (platform, worker, ledger, work, qhorus — a subset of Wave 1's 10 repos):
- **Known-stale sections** (before fixes) — true positives for staleness detection
- **Known-current sections** (after fixes) — true negatives

This corpus validates D4's structural anchor detection against both precision and recall:
- **Precision ≥80%** — of sections flagged as stale, ≤20% are false positives
- **Recall ≥60%** — of actually-stale sections, ≥60% are detected
- **F1 ≥0.69** — harmonic mean ensures neither metric is sacrificed for the other

If thresholds are met, the work-end hard gate activates. If not, the detection methodology is refined and re-validated against the same corpus.

---

## 5. Capability Index Design

The cross-repo capability index (`docs/capabilities.md`) replaces the current `docs/consumer-index.md` as the primary capability discovery entry point. Where consumer-index.md routes by repo to monolithic per-repo guides, capabilities.md routes by capability to individual chunks — enabling precise, topic-scoped loading:

```markdown
# CaseHub Capability Index

> What can I do with CaseHub? Find the capability, follow the link.

## Orchestration
| Capability | What it does | Consumer chunk | Contributor chunk |
|------------|-------------|----------------|-------------------|
| Case lifecycle | Define and execute multi-step case plans | engine/capabilities/case-lifecycle.md | engine/internals/case-execution.md |
| Work items | Human task inbox with SLA and delegation | work/capabilities/work-items.md | work/internals/work-item-store.md |
| Worker dispatch | Automated task execution and routing | worker/capabilities/worker-api.md | worker/internals/executor-pipeline.md |

## Communication
| Capability | What it does | Consumer chunk | Contributor chunk |
|...

## AI & Knowledge
...
```

Organized by domain (orchestration, communication, identity, audit, AI, UI, infrastructure, operations) rather than by repo. An LLM searching "how do I send a notification" scans the table, finds the row, and loads the specific chunk.

---

## 6. Skill Modifications

### 6.1 work-end orchestrator

New step `doc_freshness_gate`:
- **Position:** After `impl_doc_sync`, before `adr`
- **Phase:** `closing:review`
- **Type:** `judgment` (LLM decides whether findings are genuine)
- **Skip condition:** No code changes in the branch (docs-only branches skip)
- **Activation:** Advisory mode until validation corpus confirms ≥80% precision AND ≥60% recall (F1 ≥0.69), then hard gate

### 6.2 implementation-doc-sync

Extend to check structural anchors in addition to session-scoped analysis. When a changed file is anchored by a documentation section, flag that section even if the session didn't explicitly touch it.

### 6.3 New: doc-freshness-check script

**Location:** `soredium/doc-freshness/doc-freshness-check.py` — the skill package for the doc freshness gate. Installed to `~/.claude/skills/doc-freshness/` via `sync-local`, matching the existing skill distribution pattern.

**Work-end integration:** The `doc_freshness_gate` step (§6.1) is a `judgment` handler in the work-end orchestrator. It invokes `doc-freshness-check.py` via `python3 ~/.claude/skills/doc-freshness/doc-freshness-check.py` with arguments:
- `--diff <branch-diff-path>` — the branch diff (already available from work-end context)
- `--graph <dependency-graph-path>` — path to `dependency-graph.json` in casehub-parent
- `--docs <docs-root>` — path to the docs directory

**GitHub Action integration:** The Action checks out soredium to access the script, or the script is vendored into casehub-parent's `.github/scripts/` directory.

**Output:** JSON list of candidate-stale sections with evidence (which anchor changed, what the diff shows). The work-end handler reads this JSON and dispatches Phase 3 adversarial verification (§3.3) on flagged sections. Shared implementation ensures work-end and CI use the same detection logic.

---

## 7. GitHub Action (PR-level enforcement)

**Deployment:** A reusable workflow (`.github/workflows/doc-freshness.yml`) is defined in casehub-parent and called from each child repo's CI via `uses: casehubio/parent/.github/workflows/doc-freshness.yml@main`. Each child repo adds a thin caller workflow (3 lines). The reusable workflow pattern ensures consistent detection logic across all 28 repos without duplicating the Action definition.

**Per-repo caller workflow** (deployed to each child repo):
```yaml
on: pull_request
jobs:
  doc-freshness:
    uses: casehubio/parent/.github/workflows/doc-freshness.yml@main
```

**Reusable workflow steps:**
1. Check out parent repo to access `dependency-graph.json` and `doc-freshness-check.py`
2. **Doc freshness check:** Run `doc-freshness-check.py` with the PR diff. If candidate-stale sections found: post PR comment listing them, set check to "action required"
3. **Anchor integrity check:** Walk all documentation files with YAML frontmatter, extract anchor class/SPI names, verify each resolves in the codebase via `git ls-files` + package verification (same approach as §3.5 Tier 1 step 2). **Cross-repo filtering:** only verify anchors whose frontmatter `repo` field matches the current repo — anchors referencing upstream classes (e.g., `CurrentPrincipal` from `casehub-platform-api` in an engine consumer guide) are skipped by CI and caught instead by Phase 3's adversarial check, which has full cross-repo code navigation via IntelliJ MCP. Report broken same-repo anchors as PR comments. This catches anchors that broke outside the current branch's diff — e.g., a class deleted in a prior merge that no branch diff matched to Phase 1's structural check.
4. If no sections flagged and no broken anchors: pass

Lightweight — structural anchor check + anchor integrity only, no LLM adversarial. Covers the merge paths that work-end misses.

---

## 8. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Structural anchors themselves drift (class renamed but anchor not updated) | False negatives — stale section not detected | Broken anchors are mechanically detectable (class not found in codebase). The reusable workflow's anchor integrity step (§7 step 3) validates all anchors on every PR. Phase 3 adversarial verification (§3.3) explicitly renews anchors — updating renamed references and removing deleted ones. |
| Work-end becomes too slow with doc gate | Developer friction, gate bypass | Structural anchor check is O(diff size), not O(repo size). LLM adversarial runs only on flagged sections. |
| Audit slot conflicts with feature work | Merge conflicts on guides | Slot is dedicated — no feature work shares the audit branches. Guide updates are additive (new content), not conflicting. |
| Monolithic guides resist decomposition | RAG quality stays poor for un-decomposed guides | Demand-driven — decompose when retrieval failures occur or guide exceeds size threshold. Not all-or-nothing. |
| Arc42stories prose drift between epics | Inaccurate architecture docs | Bounded by epic duration (1-2 weeks). Acceptable trade-off vs per-branch overhead. |

---

## References

- decisions.md — 7 validated decisions (D1-D7)
- docs/platform/dependency-map.md — current manual dependency map
- docs/INDEX.md — current documentation entry point
- docs/consumer-index.md / docs/contributor-index.md — current guide indexes
- GE-20260601-85afd0 — 3-check quality sweep for arc42stories
- GE-20260601-b0eabf — class name existence verification
- GE-20260623-e02ce2 — GitHub issue stateReason technique
- arc42stories spec §9.4 — 9 subsection types (4 structural, 5 prose)
- PLATFORM.md decomposition precedent — topic files + thin index pattern
- Blog "The Factory That Forgot" (2026-06-17) — manual merge bypass incident
