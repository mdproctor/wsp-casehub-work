## D1: GraphQL as primary server action type

**Choice:** GraphQL operations replace REST as the canonical server-side action in scenario YAML. Every CaseHub service must expose GraphQL — enforced at build time.
**Alternatives:**
- REST endpoints as primary delivery — loosely typed, requires separate documentation, LLMs can't discover available operations from the contract alone
- GraphQL with REST fallback — unnecessary complexity since every CaseHub service will have GraphQL
**Rationale:** GraphQL is self-describing (schema is the contract), type-safe (server validates inputs), and composable (queries select specific fields for matching). LLMs read the schema and know exactly what operations exist, what inputs they take, and what they return. The `GraphQLResolverProcessor` already generates resolvers from `@PlatformQuery`/`@PlatformMutation` on SPI interfaces, so any repo with an SPI gets GraphQL automatically.
**Trade-offs:** External platforms still integrate via REST — but REST is auto-exposed alongside GraphQL by SmallRye. The YAML format doesn't use REST; external integrators do.
**Exploration:** quick
**Status:** captured

## D2: Three action types — GraphQL, ARIA, HTTP

**Choice:** Scenario steps have three action types: GraphQL (CaseHub server operations), ARIA (frontend UI automation), HTTP (third-party REST calls). No "delivery modes" — the action type is implicit from what's being called.
**Alternatives:**
- Two types (GraphQL + ARIA only) — fails for browser-only scenarios that need to call third-party APIs
- Existing three delivery modes (rest, ui-form, simulated) — conflates delivery mechanism with action semantics; "simulated" is a runtime profile concern, not a YAML concern
**Rationale:** Clean separation of concerns. GraphQL covers all CaseHub backend operations. ARIA covers all frontend interactions using standardised accessibility vocabulary. HTTP covers external integrations and works in both backend and browser-only executors.
**Trade-offs:** Three vocabularies for an LLM to learn — but each is well-defined and standardised.
**Exploration:** quick
**Status:** captured

## D3: Direct ARIA references for frontend actions

**Choice:** Frontend steps reference ARIA roles and accessible names directly (e.g. `{ role: 'button', name: 'Submit' }`). No CSS selectors, no data-* attributes.
**Alternatives:**
- Natural language selectors resolved against ARIA tree — more readable but ambiguous, requires fuzzy matching
- CSS / data-* attribute selectors (existing spec) — coupled to implementation, not standardised, LLMs don't inherently understand them
**Rationale:** ARIA is a standardised vocabulary that LLMs already understand. Mechanically validatable against the rendered DOM. Makes scenarios accessibility-correct by design.
**Trade-offs:** Requires applications to have proper ARIA markup — but they should anyway.
**Exploration:** quick
**Status:** captured

## D4: Distributed fragment execution model

**Choice:** The backend executor partitions the YAML into fragments and distributes them to target executors (browser, other services). Each executor runs its fragment locally and autonomously — parsing the YAML, managing its own trigger graph, sequencing its own steps. Browser-only mode is the same model without a distributing backend.
**Alternatives:**
- Central command stream — backend sends one command at a time to each executor. Higher coordination overhead, single point of sequencing failure.
- Full script broadcast — every executor gets the full YAML and filters to its own steps. Wasteful, leaks information about other executors.
**Rationale:** YAML becomes a portable execution contract. Any executor (Java, browser JS, future distributed services) runs the same format. Self-contained fragments enable autonomous execution with local trigger graphs.
**Trade-offs:** Cross-executor triggers require coordination protocol between the distributing backend and fragment executors.
**Depends on:** D2 (action types determine fragment partitioning)
**Exploration:** quick
**Status:** captured

## D5: YAML submission via GraphQL

**Choice:** Scenario YAML scripts are submitted to the execution server via a GraphQL mutation. The executor is a CaseHub service with a GraphQL API like any other.
**Alternatives:**
- REST upload endpoint — inconsistent with GraphQL-first platform rule
- File-based loading — doesn't support remote/distributed submission
**Rationale:** Consistent with D1 — the executor follows the same platform rules as every other service.
**Trade-offs:** None significant — GraphQL handles string/text payloads fine.
**Depends on:** D1 (GraphQL-first platform rule)
**Exploration:** quick
**Status:** captured

## D8: Step-level target — steps are dispatch units

**Choice:** The `target` field lives on the step, not on individual commands. All commands in a step execute on one executor. Switching executors means starting a new step.
**Alternatives:**
- Command-level target — each command specifies its own executor. More flexible for interleaved browser/server sequences, but the orchestrator must decompose steps into sub-fragments per executor, which is functionally identical to having smaller steps. Creates partial failure ambiguity (browser command succeeds, server command fails — is the step failed?). Conflicts with D4's fragment model where steps are the dispatch unit.
**Rationale:** If you're switching executors, that IS a new step. The step boundary communicates the executor switch to the reader. The orchestrator sends each step to one executor wholesale — no decomposition needed. Batching consecutive same-target steps is trivial. Error model is clean: a step succeeds or fails on one executor. Command-level targets would require the orchestrator to create implicit step boundaries at every executor switch, which means the format pretends steps span executors when the runtime splits them anyway.
**Trade-offs:** Interleaved browser/server sequences produce more steps. This is a feature — each executor switch is visible in the YAML, not hidden inside a mixed step.
**Depends on:** D4 (distributed fragment execution — steps are the natural fragment boundary)
**Exploration:** quick
**Status:** captured

## D7: Explicit command objects — action as value, not key

**Choice:** Every command uses an explicit `action` field as the type discriminator: `{action: click, target: {role: button, name: Submit}}`. All three action types (ARIA, GraphQL, HTTP) share the same structure.
**Alternatives:**
- ARIA shorthand keys — action name as the YAML key (`click: {role: button, name: Submit}`). More compact for ARIA-heavy sequences, reads like prose. But only helps ARIA — GraphQL and HTTP commands are the same length either way. Creates two syntactic styles in one file. Key collision risk between action names and metadata fields. Harder to schema-validate (polymorphic keys vs uniform discriminator).
- Hybrid — allow both forms. Doubles parser complexity for style flexibility without a clear win.
**Rationale:** The format serves humans, LLMs, and parsers. Uniform structure means one pattern to learn, generate, and validate. The compactness savings from shorthand are real but marginal (one field per command) and only apply to ARIA actions while introducing inconsistency with GraphQL/HTTP commands in the same file.
**Trade-offs:** ARIA sequences are slightly more verbose than shorthand. Acceptable — the consistency across all action types outweighs per-command brevity.
**Depends on:** D2 (three action types must use the same command structure)
**Exploration:** quick
**Status:** captured

## D9: Field naming — `target` for executor, `element` for ARIA element

**Choice:** Step-level `target` means which executor runs the step (`target: browser`). Command-level `element` means which ARIA element to interact with (`element: {role: button, name: Submit}`). No collision — each field name matches what it IS.
**Alternatives:**
- Both called `target` — disambiguated by context (step vs command) and type (string vs object), but readers must infer meaning from position. Same word, different semantics at different levels is a readability trap.
- Step-level called `executor` — more precise, but `target` is the natural word for "where this runs" and is already established in Format B.
**Rationale:** ARIA targets are elements — role + accessible name identifies a DOM element. Calling the field `element` isn't a rename, it's using the correct name. `target` at the step level means the execution target (which executor). Two distinct concepts, two distinct names.
**Trade-offs:** None — this is just correct naming.
**Depends on:** D3 (ARIA element references), D8 (step-level target routing)
**Exploration:** quick
**Status:** captured

## D10: Three error modes — stop, continue, pause

**Choice:** Scenario-level `on-error` field with three modes: `stop` (abort all executors — default), `continue` (skip failed step's dependents transitively), `pause` (freeze for operator intervention). Default is `stop`.
**Alternatives:**
- Stop only — error recovery is the executor's problem. But the format already captures execution semantics (target routing, variable dependencies), and error policy is part of execution semantics.
- Two modes (stop + continue, drop pause) — pause is a runtime/UI concern. But pause is the natural mode for human-paced demos (D6) — when something fails during a live walkthrough, freezing for intervention is the right behaviour.
**Rationale:** The error mode is a property of the use case. Unattended automations use `stop` (fail-fast, safe default). Resilient pipelines use `continue` (skip failures, run what you can). Human-paced scenarios use `pause` (freeze, let the operator decide). All three are legitimate use cases that the format should support declaratively.
**Trade-offs:** `pause` blocks forever without a human — but that's the point. An automation that declares `pause` is declaring it expects human oversight.
**Depends on:** D6 (automations vs scenarios — error mode aligns with use case)
**Exploration:** quick
**Status:** captured

## D11: Variable interpolation — ${stepName.field.path}

**Choice:** Keep the existing `${stepName.field.path}` syntax from `VariableContext.java`. Step names are the namespace, dot-paths navigate the result object. Regex: `\$\{([^}]+)}`. Steps that produce results need a `name` field; unnamed steps can't be referenced.
**Alternatives:**
- Scoped prefix syntax `${steps.name.result.field}` — more explicit about where variables come from, but more verbose and doesn't match the existing implementation. The v2 spec proposed this and the review correctly flagged it as divergent from working code.
**Rationale:** Proven implementation in `VariableContext.java`. Simple, compact, well-understood. No reason to change what works.
**Trade-offs:** Step names must be unique within the scenario for unambiguous variable resolution. This is already enforced by the existing implementation.
**Sources:** `pages/backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/VariableContext.java`
**Exploration:** quick
**Status:** captured

## D6: Flexible hierarchy — automations as core, scenarios as overlay

**Choice:** The base format is an automation (steps + commands, no human oversight needed). Chapters and sections are a presentation overlay for when humans pace through the execution (demos, walkthroughs). Top level can be chapters, sections, or steps directly — mutually exclusive.
**Alternatives:**
- Always full hierarchy — every automation must have chapters/sections even when running unattended, adding verbosity for no benefit
- Two-level only (steps + commands) — drops the narrative structure that makes helpdesk-style demos self-documenting and navigable
**Rationale:** Automations and scenarios are different use cases sharing the same execution engine. Simple automations (seed data, smoke tests) just need steps. Human-paced demos need narrative structure. The format shouldn't force one shape on both.
**Trade-offs:** Parser must handle three entry points (chapters, sections, steps). Worth it — the alternative is forcing demo chrome on headless automations.
**Depends on:** D4 (distributed execution model applies to both automations and scenarios)
**Exploration:** quick
**Status:** captured
