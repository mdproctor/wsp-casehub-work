# Decisions — #946 Playbook Template Declaration and Executor Wiring

## D1: Event pipeline location

**Choice:** New TemplateMatchingFilter in PlaybookRunnerCore
**Alternatives:**
- Step-level matching — matching inside individual steps (await, signal). Scatters matching logic, harder to test uniformly.
- Compile-time only — defers runtime wiring. Doesn't deliver the executor integration the issue requires.
**Rationale:** Keeps extraction in the executor where the issue says it belongs. Single interception point wraps channel input, matching before steps see the message.
**Trade-offs:** Adds a new abstraction to the runner's input path.
**Sources:** PlaybookRunnerCore.java, issue #946 design section
**Exploration:** quick
**Status:** captured

## D2: Variable constraint enforcement

**Choice:** Enforce at extraction time
**Alternatives:**
- Declaration only — parse and store but don't validate. Useful for tooling but no runtime safety.
**Rationale:** VariableConstraint from #944 already supports enum/string validation in TemplateExtractor. Enforcing at extraction prevents invalid variable values from reaching playbook steps.
**Trade-offs:** Slightly stricter — a template match with an unexpected value returns no match instead of a match with an invalid value.
**Sources:** VariableConstraint.java (yaml-core), TemplateExtractor.extract(input, constraints), issue #944
**Exploration:** quick
**Depends on:** D1 (matching happens in the runner)

## D3: Match result type

**Choice:** New TemplateMatchEvent record
**Alternatives:**
- Reuse ProtocolMatchResult — simpler but lacks score and original input fields.
**Rationale:** The issue specifies matched events carry type, fields, score, and original input. ProtocolMatchResult.Matched only has templateName and params. A richer record wraps the ProtocolEvaluator result with the additional fields.
**Trade-offs:** New type to maintain, but it's a simple record.
**Sources:** ProtocolMatchResult.java, issue #946 needs section
**Exploration:** quick
**Depends on:** D1 (the runner produces these events)
