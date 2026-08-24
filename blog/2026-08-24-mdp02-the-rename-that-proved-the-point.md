---
layout: post
title: "The Rename That Proved the Point"
date: 2026-08-24
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [refactoring, type-safety, llm-productivity, intellij, empirical]
series: issue-422-ts-programming-model
---

# The Rename That Proved the Point

Three fields needed renaming across two repositories. `Worker.capabilityNames` to `capabilities`. `Capability.inputSchema` to `inputProjection`. `Capability.outputSchema` to `outputProjection`. Mechanical work. The kind of thing a competent engineer does in thirty seconds with an IDE refactoring.

Claude tried it without the IDE first. The results are worth documenting because they're not anecdotal. Every tool call, every compile cycle, every false positive is in the session transcript. This is measured data from a real cross-repo rename on a codebase with 99 affected Java files across 30+ modules.

## The text-based attempt

The worker-api repo was renamed first via IntelliJ, but the engine repo resolved worker-api from a Maven jar, not the workspace source. IntelliJ's refactoring propagated within the worker repo but stopped at the project boundary. The engine still had the old names everywhere.

Claude fell back to `ide_replace_text_in_file` — individual file text replacement. The approach: search for `.capabilityName(`, replace with `.capability(`. Search for `.inputSchema(`, replace with `.inputProjection(`. Repeat per file.

The first false positive appeared on file four.

`CaseDefinitionRecorder` calls `wd.capabilityName()` on `WorkerDescriptor` and `bd.capabilityName()` on `BindingDescriptor`. These are different types with the same method name. The text replacement renamed them all. The compiler caught it — `WorkerDescriptor` has no `capability()` method. Two reversals, three extra tool calls.

`CandidateMatchingStrategyContractTest` calls `ctx.capabilityName()` on `CandidateMatchingContext`. Same pattern — different type, same text, wrong rename. Another reversal.

Maven compiles sequentially. Each module's test sources only compile after the previous module succeeds. This creates a whack-a-mole pattern: fix module A, recompile, discover module B is broken, fix B, recompile, discover module C. Six compile cycles before giving up, each exposing the next layer of failures.

At compile cycle six, the build reported 200 errors across 25 runtime test files. Modules further downstream — planning (26 test files), resilience (5), flow, a2a, mcp, react — hadn't been reached by the compiler yet.

Claude reported the situation as "tedious" and asked whether to continue or find a better approach. The text-based rename was abandoned incomplete.

## The measured cost

| Metric | At abandonment | Projected to completion |
|--------|---------------|------------------------|
| Tool calls | 69 | ~251 |
| Files fixed | 33 of 99 | 99 |
| Compile cycles | 6 | ~11 |
| False positives caught | 4 | ~16 (at observed 20% rate) |
| Wall time | ~25 min | ~45 min |

The projection method: 61 files remained from search results, averaging 2.3 replacement patterns per file, plus compile cycles for each remaining module. The false positive rate (4 out of 20 `.capabilityName(` replacements) is from a small sample — the true rate depends on how many types in the codebase share the renamed method name. In a platform with routing strategies, contexts, and plan steps all carrying a `capabilityName()` accessor, 20% is conservative.

## The type-safe attempt

After reverting all partial changes, the workspace was reopened with the worker repo included as a source module — not a jar dependency. IntelliJ resolved `io.casehub.worker.api.Worker` from the workspace source, giving it visibility into all callers across both repositories.

Three `ide_refactor_rename` calls. One per field.

| Rename | Files changed | False positives |
|--------|--------------|-----------------|
| `capabilityNames` to `capabilities` | 25 | 0 |
| `inputSchema` to `inputProjection` | 41 | 0 |
| `outputSchema` to `outputProjection` | 33 | 0 |

Total: 8 tool calls. One compile pass. Zero errors. Roughly two minutes.

## The comparison

| Metric | Text-based (projected) | IDE refactor (actual) | Ratio |
|--------|----------------------|----------------------|-------|
| Tool calls | ~251 | 8 | 31x |
| Wall time | ~45 min | ~2 min | 23x |
| Compile cycles | ~11 | 1 | 11x |
| False positives | ~16 + unknown silent | 0 | - |

The ratios are large but the qualitative gap is larger. The text-based approach is semantically unsound — it cannot distinguish `Worker.Builder.capabilityName()` from `CandidateMatchingContext.capabilityName()`. Both are `.capabilityName(` in text. Only one should be renamed. The four caught false positives were caught because the target type lacked a `capability()` method, producing a compile error. Any false positive where the target type *does* have a compatible method signature compiles silently and produces a runtime bug. There is no way to quantify how many of those exist without type analysis — which is precisely what the IDE refactoring provides.

## What this tells us about LLM productivity

An LLM coding agent without type-aware tooling is not slower at refactoring. It cannot safely do it at all.

The text-based approach was not a junior-engineer version of the same operation. It was a fundamentally different operation — string substitution with no semantic model. The 20% false positive rate on a 3-field rename across 99 files means the probability of at least one undetected semantic error approaches certainty as the codebase grows. The LLM has no in-loop type checker. It discovers errors only via compilation, and only for errors that produce compile failures. Silent type-compatible false positives are invisible.

The cost ratio — 31x in tool calls, 23x in time — understates the real difference because it compares a projected completion against an actual completion. The text-based approach was abandoned at 33% progress with an accelerating error rate. Projecting linear completion from a non-linear failure curve is generous.

Type-safe refactoring tools don't make LLMs faster. They make operations possible that are otherwise not safely achievable. That is a categorical difference, not a quantitative one.
