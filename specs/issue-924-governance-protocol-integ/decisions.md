## D1: Integration approach — template-first evaluator

**Choice:** The protocol evaluator's primary job IS template matching. `register()` takes protocol definitions with templates, `evaluate()` takes agent response text, returns matched template name + extracted params + advisory.
**Alternatives:**
- Event handler — template matching as one event type among many. Overcomplicates the API for what is fundamentally a response-matching concern.
- Separate concern — new `ProtocolResponseMatcher` alongside evaluator. Creates unnecessary indirection when the evaluator skeleton exists and is hollow.
**Rationale:** The existing `PlaybookProtocolEvaluator` has empty methods — it was scaffolded for exactly this purpose. Template matching IS the protocol evaluation for governance responses.
**Trade-offs:** Locks the evaluator to template-based matching; if protocols later need non-template evaluation (e.g. semantic/embedding-based), the evaluator API would need extension.
**Sources:** `coordination-core/src/main/java/.../protocol/PlaybookProtocolEvaluator.java`, issue #924, issue #921
**Exploration:** quick
**Status:** captured

## D2: Protocol definition shape — ProtocolDefinition record

**Choice:** New `ProtocolDefinition` record (name, `Map<String, String>` templates, unmatched handling config) in `coordination-core/protocol/`. The evaluator's `register()` takes this instead of raw `Map<String, Object>`.
**Alternatives:**
- YAML schema extension — requires parser and schema work beyond the integration scope.
- Builder API only — no reusable definition object; callers must rebuild TemplateRegistry each time.
**Rationale:** A typed record makes protocol definitions first-class, inspectable, and serializable. YAML support can wrap it later.
**Trade-offs:** Commits to a specific record shape; changes require a new version of the record.
**Sources:** `TemplateRegistry.Builder` API, `PlaybookProtocolEvaluator.register()` signature
**Exploration:** quick
**Depends on:** D1 (template-first evaluator)
**Status:** captured

## D3: Unmatched response handling — sealed result type

**Choice:** Return a sealed interface: `Matched(name, params)` | `Unmatched(response)`. Callers decide the action (rephrase, escalate, fail).
**Alternatives:**
- Advisory with REROUTE — overloads the advisory type with match semantics. Conflates structural match failure with governance-level concerns.
- Optional.empty() — loses the original response text, making retry/rephrase logic harder for callers.
**Rationale:** Sealed types make the two outcomes explicit in the type system. The caller is forced to handle both cases. No opinions about what to do with an unmatched response are baked in.
**Trade-offs:** Callers must pattern-match instead of checking Optional. Slightly more ceremony.
**Sources:** Java sealed interface pattern, `TemplateRegistry.match()` returns `Optional<Match>`
**Exploration:** quick
**Depends on:** D1 (template-first evaluator)
**Status:** captured

## D4: Module placement — coordination-core/protocol/

**Choice:** Keep everything in `coordination-core/protocol/` where the skeleton already lives. Replace the hollow stubs.
**Alternatives:**
- New protocol-core module — unnecessary module proliferation for what is currently 4-5 classes.
**Rationale:** `coordination-core` already depends on `yaml-core`. The `protocol/` package was created for this purpose. The existing types (`ProtocolAdvisory`, `ProtocolInstanceKey`, `AdvisorySeverity`, `SuggestedAction`) remain relevant.
**Trade-offs:** Couples protocol semantics to the coordination module. Extraction to a separate module remains straightforward later.
**Sources:** `coordination-core/pom.xml` (yaml-core dependency), `coordination-core/protocol/` package
**Exploration:** quick
**Status:** captured
