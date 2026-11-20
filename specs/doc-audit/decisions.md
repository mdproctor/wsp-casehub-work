# Doc Audit & RAG Optimization — Decisions

## D1: Work-end gate scope

**Choice:** Change-type scoping using module tier classification, with protocol-change supplementary rule
**Alternatives:**
- Home repo only — catches direct drift but misses ripple effects from API/SPI changes
- Home repo + all direct dependents (original) — unbounded for foundation repos; platform-api dependents = entire ecosystem
- Flag dependents without auditing — cheap signal but no verification
- Topic-based impact detection — requires classifying changes by topic, which is itself an unsolved mapping problem
- Change-type scoping without module tier — under-specified without a classification discriminant
**Rationale:** The module tier structure (api/ = public SPI, runtime/ = internal, testing/ = test scope) already classifies change impact for code changes. Changes to api/ modules have cross-repo impact and check consumers via automated dependency analysis (D6). Changes to runtime/ modules check home repo guides only. Changes to testing/ modules skip doc checks. For non-code changes: when a protocol in casehub/garden changes, the PR-level GitHub Action (D2) detects the change and flags documentation sections across all repos that declare that protocol as a structural anchor (D4). Protocol references in documentation YAML frontmatter (e.g. `protocols: [flyway-version-range-allocation, module-tier-structure]`) enable this mapping — the same structural anchor mechanism that D4 uses for code references. This covers the 293 protocol files in casehub/garden without requiring module tier classification.
**Trade-offs:** Requires accurate module tier classification for code changes. Protocol-anchor coverage depends on documentation declaring protocol references during D3's audit. Mixed-tier changes (touching both api/ and runtime/) default to the higher impact tier.
**Sources:** docs/platform/dependency-map.md, module-tier-structure protocol (casehub/garden), coherence-protocol step 6, casehub/garden protocol count (293 files)
**Exploration:** quick → revised (R1-02, R2)
**Status:** revised

## D2: Enforcement model and enforcement point

**Choice:** Dual enforcement — hard gate at work-end for LLM sessions (primary), PR-level GitHub Action for all merge paths (complementary). Hard gate activates after D4 detection passes validation.
**Alternatives:**
- Advisory + issues for all — weaker enforcement, docs still fall behind
- Tiered (hard for API/SPI, advisory for internals) — complex to classify changes correctly
- Hard gate only at work-end (original) — misses human-only merge paths; risks enforcement theater if detector has low effectiveness
- PR-level Action only — covers all paths but loses work-end context (git log, blog history, session knowledge)
- Post-merge staleness dashboard — provides visibility but no enforcement; relies on voluntary action
- Work-end only enforcement point — misses human PR, manual git merge, and CI pipeline merge paths
**Rationale:** Work-end is the primary merge path (>95% of commits) and has the richest context (session memory, blog history, git log analysis). PR-level GitHub Action covers the gap for human merges, CI merges, and direct pushes — the blog entry "The Factory That Forgot" (2026-06-17) documents a manual git merge that bypassed work-end and created divergence with origin. The four merge paths to main — LLM session (work-end), human PR (GitHub merge), manual git merge, CI pipeline merge — are all covered by the PR-level Action. Work-end adds deeper analysis for the LLM path. Hard gate activation prerequisite: D4's structural anchor detection must demonstrate precision ≥ 80% (≤ 20% false positive rate) on D3's initial audit validation corpus. Validation sample: the results from D3's first-wave audit of 5 foundation repos (platform, worker, ledger, work, qhorus) — the audit produces known-stale sections (before fixes) and known-current sections (after fixes), forming a labeled corpus. Validation trigger: automatic after D3 first-wave completion; audit lead reviews detection precision against audit findings. If precision ≥ 80%, the hard gate activates. If not, the detection methodology is refined and re-validated against the same corpus. Cross-repo dependents get GitHub issues (unchanged).
**Trade-offs:** Dual enforcement is more complex — two detection surfaces to maintain (work-end skill + GitHub Action). PR-level Action has less context than work-end. Activation prerequisite delays hard-gate enforcement until detection is proven, but this prevents the "enforcement theater on a zero-output detector" failure mode. 80% precision threshold balances strictness against false-positive blocking risk.
**Sources:** Current work-end gate analysis (impl_doc_sync is advisory), blog 2026-06-17 (factory-that-forgot: manual merge bypass documented), close-progress logs (variable impl_doc_sync output across repos), work-end HARD-GATE convention
**Exploration:** quick → revised (R1-03, R1-10, R2)
**Status:** revised

## D3: Fix strategy for initial audit

**Choice:** Dedicated audit slot with per-repo GitHub issues, prioritized by dependency blast radius, with explicit exit criteria
**Alternatives:**
- Batch by staleness tier — faster but less thorough per repo
- Central sweep from parent — loses per-repo git log and blog context, but can share platform context efficiently
- Automated skeleton generation — useful pre-pass (git log → draft change summary → LLM audit), but not sufficient alone
**Rationale:** Each repo needs access to its own git history, issues, and blogs to reconstitute accurate guides. Priority order: foundation repos first (platform, worker, ledger, connectors, iot, work, qhorus, eidos, neocortex, engine), then orchestration/integration (ras, desiredstate, blocks, blocks-ui, claudony, openclaw, workers, ops), then application (devtown, aml, clinical, life, drafthouse, quarkmind, soc, fsitrading). Foundation repos have the highest blast radius — their documentation errors cascade through all consumers. Exit criteria per repo: (1) all consumer guide sections verified against current code by LLM adversarial check, (2) all structural anchors (D4) validated via automated assertion, (3) per-repo GitHub issue closed with evidence commit.
**Trade-offs:** More sessions required. Foundation-first ordering means application repos wait longer. Automated skeleton pre-pass recommended to reduce per-repo session cost.
**Sources:** Staleness inventory (1,700+ commits across 28 repos), dependency map blast radius analysis
**Exploration:** quick → revised (R1-04)
**Status:** revised

## D4: Staleness detection methodology

**Choice:** Structural anchors (primary) + diff-based triage (transition fallback for un-anchored sections) + LLM adversarial verification (semantic layer)
**Alternatives:**
- Diff-based triage then LLM adversarial (original) — unsolved mapping from code diffs to documentation sections
- Diff-based only — misses semantic drift (guide says X, code does Y now)
- LLM adversarial only — slow, token-heavy, may hallucinate findings
- Docs-as-tests — executable assertions for structural claims; a strict subset of structural anchors
**Rationale:** Documentation sections declare the code elements they describe via structural anchors (class names, SPI names, config keys, file paths, protocol references). When anchored elements change (detected via diff), the anchoring section is flagged as candidate-stale. LLM adversarial check reviews only flagged sections against actual code. Diff-based triage serves as a transition fallback: during and after D3's audit, structural anchors are being added to documentation. Until a section has anchors, diff-based triage fills the gap by scanning all un-anchored sections when any file in the repo changes, triggering LLM adversarial check on those sections. Sunset condition: once D3's exit criteria #2 is met across all repos (all structural anchors validated), diff-based triage is retired. Post-sunset, new documentation sections must include structural anchors at creation — enforced by D2's gate. Override mechanism: `verified-current: <date>` annotation suppresses re-flagging until the next anchor change.
**Trade-offs:** Adding structural anchors to existing documentation is upfront work (addressed by D3's audit slot). Anchors themselves can drift, but anchor-drift is detectable mechanically (missing class → broken anchor). Diff-based triage is intentionally coarse during the transition — it trades precision for coverage on un-anchored sections, and becomes unnecessary once anchoring is complete.
**Sources:** Garden entries on arc42stories quality gates (3-check sweep GE-20260601-85afd0, class name existence GE-20260601-b0eabf)
**Exploration:** quick → revised (R1-05, R2)
**Status:** revised

## D5: Document structure for session loading and RAG retrieval

**Choice:** Incremental, demand-driven decomposition following the PLATFORM.md precedent — extract largest sections as separate topic files with thin guide indexes. Add YAML frontmatter with per-section metadata for RAG-quality filtered retrieval.
**Alternatives:**
- Three-layer big-bang (capability index, per-capability chunks, thin guide indexes — original) — large upfront effort, circular dependency with D4, unnecessary given incremental path
- Ingest monolithic guides into neocortex RAG as-is — solves retrieval for existing content but yields poor chunk boundaries (recursive character splitting at 1000-char boundaries ignores section structure) and per-document metadata (all chunks share same tags)
- Capability lookup tables added to existing monolithic guides — still loads full file for one section
- Self-contained chunks only (no index) — good for retrieval, poor for session loading
- On-demand decomposition guided by retrieval failure data — valid refinement; the incremental approach subsumes this
**Rationale:** PLATFORM.md was organically decomposed into topic chunks (routing.md, persistence.md, auth.md, etc.) with a thin INDEX.md — this pattern proved effective for both session loading and retrieval quality. Consumer/contributor guides follow the same pattern: extract the 3-5 largest sections per guide into topic-scoped files, each with YAML frontmatter (capability tags, class anchors, config keys) enabling precise filtered retrieval via neocortex's PayloadFilter. The neocortex RAG pipeline's chunking is purely positional (LangChain4j recursive char splitter) — it cannot perform structure-aware decomposition or assign per-section metadata. Manual decomposition provides the pipeline with properly-bounded, individually-tagged documents that the automated chunking then handles effectively within each section.
**Trade-offs:** Incremental approach means some guides remain monolithic longer. Decomposition priorities are guided by guide size (526-line qhorus and 546-line devtown first) and retrieval failure data.
**Sources:** Current guide sizes (119-546 lines), PLATFORM.md decomposition precedent (topic files + thin index), neocortex rag pipeline analysis (recursive char splitting only, per-document metadata only)
**Exploration:** quick → revised (R1-06)
**Status:** revised
**Depends on:** D4 (structural anchors in YAML frontmatter serve as both staleness anchors and retrieval metadata)

## D6: Dependent repo detection

**Choice:** Automated POM analysis (CI-triggered) as primary source for dependency edges, with semantic Nature annotations maintained in docs/platform/dependency-map.md for impact classification
**Alternatives:**
- Static dependency map only (original) — subject to the same staleness problem the spec solves; human-maintained rows introduce non-determinism
- Dynamic Maven analysis at gate time — always accurate but requires all repos cloned, adds ~10s per work-end
- BOM-only analysis — simpler but misses transitive and optional dependencies
**Rationale:** A GitHub Action on casehub-parent runs dependency analysis on POM changes and writes the result to a generated dependency-graph.json. The work-end gate reads the cached graph — no per-close penalty, always accurate within one build cycle. The manual dependency map retains its Nature column (SPI signatures, runtime dep, compile scope) for change-type classification (feeds D1's module-tier scoping), but edge existence is automated. This eliminates the circularity: the dependency detection mechanism is no longer a manually-maintained document subject to the same staleness problem.
**Trade-offs:** CI job has a staleness window (triggered on POM changes, not nightly — latency is push-to-CI-completion, typically minutes). Nature annotations still require manual maintenance, but missing a Nature annotation degrades impact classification, not dependency detection — a safe failure mode.
**Sources:** docs/platform/dependency-map.md (~160 rows), Maven BOM in casehub-parent, coherence-protocol step 6
**Exploration:** quick → revised (R1-07)
**Status:** revised
**Depends on:** D1 (gate scope uses the generated graph for dependent repo resolution)

## D7: Arc42stories refresh approach

**Choice:** Epic-level refresh at epic close, using two-tier verification: 3-check sweep for structural assertions + lightweight LLM adversarial check for prose sections
**Alternatives:**
- Extend work-end gate per-branch (original) — cumulative work-end overload when combined with D1+D2+D4; wrong lifecycle granularity for arc42stories
- 3-check sweep only at epic close (round 1 revision) — covers structural assertions (key files, pattern anchors, issue status) but leaves prose sections (What it adds, Architectural decisions, Key wiring, Gotchas, Pattern to replicate) architecturally unverified
- Separate periodic refresh cycle — decoupled from feature work but drift accumulates between sweeps
- Both gate + periodic — belt and suspenders but more maintenance
**Rationale:** The arc42stories spec's §9.4 defines nine subsection types for layer entries. Four are structural (Key files, Pattern anchor, Pattern introduced, Accountability gaps closed) — mechanically checkable via the 3-check sweep (class name existence, file path validity, issue status). Five are prose (What it adds, Key wiring, Architectural decisions, Gotchas cause/fix, Pattern to replicate) — semantic content that can drift when behavior changes without renaming classes. The 3-check sweep alone leaves a known coverage gap on 5 of 9 subsection types. Adding a lightweight LLM adversarial check for prose sections at the same epic-close lifecycle point closes this gap. The LLM check is bounded — it runs once per epic (not per branch) against only the prose subsections of layers modified during the epic, making it affordable. Aligning to epic boundaries matches the arc42stories spec's own lifecycle: "at epic close, two things are distilled from the working document."
**Trade-offs:** Epic-level LLM check is heavier than 3-check sweep alone but runs infrequently. Drift in prose sections accumulates within an epic (bounded by epic duration, typically 1-2 weeks).
**Sources:** arc42stories spec §9.4 (9 subsection types: 4 structural, 5 prose), Garden entries GE-20260601-85afd0 (3-check sweep), GE-20260601-b0eabf (class name existence)
**Exploration:** quick → revised (R1-08, R1-16, R2)
**Status:** revised
**Depends on:** D4 (LLM adversarial methodology for prose verification)
