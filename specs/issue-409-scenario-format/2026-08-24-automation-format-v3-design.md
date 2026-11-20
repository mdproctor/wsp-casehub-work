# CaseHub Automation & Scenario Format — v3 Design

**Issue:** casehubio/parent#409
**Date:** 2026-08-24
**Status:** Draft

## Overview

This spec defines the canonical YAML format for CaseHub automations and scenarios. An **automation** is a sequence of steps executed by one or more executors — seeding data, running smoke tests, driving workflows. A **scenario** is an automation with a narrative overlay (chapters and sections) for human-paced execution — demos, walkthroughs, training.

Both share the same execution engine. The format is the contract between YAML authors (humans and LLMs) and executors (browser, backend services, third-party integrations).

### Design principles

- **GraphQL-first.** Every CaseHub service exposes GraphQL via `@McpDomain` + `@PlatformQuery`/`@PlatformMutation`. The format uses GraphQL as the canonical server action type. (D1)
- **Three action categories.** ARIA for browser UI, GraphQL for CaseHub services, HTTP for third-party APIs. No "delivery modes." (D2)
- **ARIA by role + name.** Frontend elements are referenced by ARIA role and accessible name. No CSS selectors. (D3)
- **Distributed fragment execution.** The orchestrator partitions the YAML into fragments per executor. Each executor runs its fragment autonomously. (D4)
- **Steps are dispatch units.** All commands in a step run on one executor. Switching executors means a new step. (D8)

## Document Structure

A YAML document has a top-level `scenario` name and optional metadata, followed by the execution body at one of three entry points:

```yaml
scenario: <name>                    # required — unique identifier
description: <text>                 # optional — human-readable summary
speed: <number>                     # optional — inter-step delay multiplier (omit for no delay)
actor: <identity>                   # optional — default authentication identity (no header if omitted)
on-error: stop | continue | pause   # optional — error mode (default: stop)

# Entry point — exactly one of:
chapters: [...]                     # full narrative hierarchy
sections: [...]                     # mid-level grouping
steps: [...]                        # flat automation
```

The three entry points are mutually exclusive. Chapters contain sections; sections contain steps; steps contain commands. This layering is flexible — simple automations skip straight to steps, full demos use the whole hierarchy. (D6)

### Chapters

Optional narrative grouping for human-paced scenarios. Chapters have no execution semantics — they group sections for navigation ("run to chapter 3") and pacing.

```yaml
chapters:
  - label: "Customer Reports Issue"
    sections:
      - label: "Customer sends message"
        steps: [...]
      - label: "System classifies ticket"
        steps: [...]
  - label: "Agent Resolves Issue"
    sections: [...]
```

### Sections

Group steps within a chapter (or at the top level if no chapters). Like chapters, sections are presentation structure — the executor pauses between sections when a human is pacing.

### Steps

The execution unit. A step groups commands that run on a single executor. All commands in a step execute sequentially on the step's target.

```yaml
steps:
  - label: "Submit support request"    # required — human-readable description
    name: submit                        # optional — machine identifier for variable interpolation
    target: browser                     # required — which executor runs this step
    commands:                           # required — at least one command
      - action: fill
        element: {role: textbox, name: "Subject"}
        value: "Laptop won't boot"
      - action: click
        element: {role: button, name: "Submit"}
```

**Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `label` | yes | Human-readable description, unique within its parent |
| `name` | no | Machine identifier for variable interpolation (`${name.field}`). Required if later steps reference this step's results. Must be unique within the scenario. |
| `target` | yes | Executor that runs this step (`browser`, service name, etc.) |
| `actor` | no | Authentication identity for this step. Sent as `X-Scenario-Actor` header on service requests. Overrides the scenario-level `actor` if both are set. When omitted at both levels, no actor header is sent — the service's own authentication default applies. |
| `delay` | no | Milliseconds to wait before executing this step. When set, replaces the speed-based inter-step delay for this step. |
| `commands` | yes | Ordered list of commands to execute |

`target` routes the step to a named executor. The orchestrator validates that all targets have registered executors before starting. (D8, D9)

## Commands

The atomic action unit. Every command has an `action` field as the type discriminator and action-specific parameters. (D7)

```yaml
commands:
  - action: <action-name>
    # ... action-specific fields
```

### ARIA Actions

Browser UI automation using ARIA roles and accessible names. The `element` field identifies the target DOM element. (D3, D9)

| Action | Fields | Description |
|--------|--------|-------------|
| `navigate` | `value: <url-or-hash>` | Navigate to URL or hash route |
| `click` | `element: {role, name}` | Click an element |
| `fill` | `element: {role, name}`, `value: <text>` | Type text into an input |
| `select` | `element: {role, name}`, `value: <option>` | Select an option from a dropdown |
| `expand` | `element: {role, name}` | Expand a collapsible element |
| `collapse` | `element: {role, name}` | Collapse an expanded element |
| `assert` | `element: {role, name}`, `state: {<aria-attr>: <value>}` | Assert element state |
| `wait` | `element: {role, name}`, `state: {<aria-attr>: <value>}`, `timeout: <ms>` | Poll until element reaches state |

**Element reference:**

```yaml
element:
  role: button          # ARIA role
  name: "Submit"        # Accessible name
```

For scoped lookups (element within a container):

```yaml
element:
  role: button
  name: "Submit"
  within:
    role: dialog
    name: "Confirmation"
```

`within` is recursive — any depth of scoping is supported.

**Examples:**

```yaml
# Navigate to a page
- action: navigate
  value: "/helpdesk/intake"

# Fill a text field
- action: fill
  element: {role: textbox, name: "Subject"}
  value: "Laptop won't boot"

# Click a button inside a dialog
- action: click
  element:
    role: button
    name: "Confirm"
    within: {role: dialog, name: "Are you sure?"}

# Assert an alert is visible
- action: assert
  element: {role: alert, name: "Ticket created"}
  state: {aria-hidden: false}

# Wait for a spinner to finish
- action: wait
  element: {role: button, name: "Submit"}
  state: {aria-busy: false}
  timeout: 5000
```

### GraphQL Actions

CaseHub server operations via the platform's GraphQL API. The `domain` field routes to the correct `@McpDomain` resolver; `operation` names the `@PlatformQuery` or `@PlatformMutation` method. (D1)

```yaml
- action: graphql
  domain: <mcp-domain>        # @McpDomain value (e.g. "connectors")
  operation: <method-name>    # @PlatformQuery/@PlatformMutation method name
  params:                     # operation parameters (key-value)
    platform: "slack"
    sender: "Alice"
    message: "My laptop won't boot"
```

**With await** — poll a query until the result matches a condition:

```yaml
- action: graphql
  domain: helpdesk
  operation: getTicketStatus
  params:
    ticketId: "${inject.ticketId}"
  await:
    match:
      category: "HARDWARE"
    timeout: 30000
    interval: 500
```

**Await semantics:** when `await` is present, the executor calls the operation, then re-invokes it at `interval` ms until the result matches `match` or `timeout` ms elapse. `interval` defaults to 1000ms.

**`await` must only be used on query operations.** Each poll cycle re-invokes the operation. For mutations, this duplicates side effects (e.g., `injectChat` called 6 times if classification takes 3 seconds at 500ms interval). If you need to execute a mutation and then wait for a condition, use two steps: the mutation first (no `await`), then a query with `await`.

**Dispatch:** The executor builds an HTTP POST to the service's `/graphql` endpoint. The `GraphQLResolverProcessor` generates resolvers from SPI annotations — the YAML author writes `domain: connectors, operation: injectChat` and the platform routes to the right generated resolver.

**Domain-to-URL routing:** `ScenarioConfig` maps domain names to service URLs. Each domain resolves to a GraphQL endpoint via `ScenarioConfig.graphQLEndpoint(domain)`, falling back to a default endpoint if no domain-specific mapping exists. This is deployment configuration, not YAML — the YAML author writes the logical domain name and the runtime resolves the physical URL.

### HTTP Actions

Third-party REST calls for external integrations.

```yaml
- action: http
  method: POST                            # HTTP method
  url: "https://api.example.com/notify"   # target URL
  headers:                                # optional headers
    Authorization: "Bearer ${token}"
    Content-Type: "application/json"
  body:                                   # optional request body
    ticketId: "${create.ticketId}"
    status: "resolved"
```

HTTP actions support variable interpolation in `url`, `headers`, and `body` fields.

**Dispatch:** `http` is a well-known action handled by the executor framework, not by `@ScenarioAction` handlers. Service executors dispatch via `java.net.http.HttpClient`; the browser executor dispatches via `fetch`. For server-side webhooks (Slack, PagerDuty, etc.), target a service executor to avoid browser CORS restrictions.

### Custom Actions

Any action name not in the well-known set (ARIA actions, `graphql`, `http`) is a **custom action** handled by the target executor. Custom actions map to `@ScenarioAction` CDI handlers on the service.

```yaml
# YAML
- action: create-ticket
  data:
    subject: "Laptop won't boot"
    category: "HARDWARE"

# Java handler on the target service
@ScenarioAction("create-ticket")
Map<String, Object> createTicket(ActionContext ctx) {
    var ticket = ticketService.create(ctx.data("subject"), ctx.data("category"));
    return Map.of("ticketId", ticket.id().toString());
}
```

Custom actions use a `data` field for key-value parameters. The executor's `@ScenarioAction` registry maps the action name to a handler method. The handler's return value is available for variable interpolation.

**Handler contract:**

| Aspect | Rule |
|--------|------|
| **Parameter access** | `ActionContext` provides `data(String key)` for accessing `data` fields, `actor()` for the step's authentication identity, and `stepName()` for the step's machine name. |
| **Return type** | `Map<String, Object>` — the returned map becomes the command's contribution to the step result. `void` handlers contribute an empty map. |
| **Error handling** | If a handler throws, the step fails with the exception message as the error. Checked and unchecked exceptions are both caught. No default execution timeout — handlers must manage their own timeouts. |
| **Discovery** | `ActionRegistry` scans CDI beans for `@ScenarioAction` annotations at startup. The annotation lives on the declaring class, not on CDI proxy subclasses — the registry traverses the superclass chain to find annotated methods. |
| **Registration** | On executor connect, the `executor-register` message's `actions` list is populated from the `ActionRegistry`'s discovered action names. |

**Well-known action names** (reserved — cannot be used as custom actions):
`navigate`, `click`, `fill`, `select`, `expand`, `collapse`, `assert`, `wait`, `graphql`, `http`

## Variable Interpolation

Steps that produce results can be referenced by later steps using `${stepName.field.path}` syntax. (D11)

```yaml
steps:
  - label: "Create ticket"
    name: create
    target: helpdesk
    commands:
      - action: create-ticket
        data: {subject: "Laptop won't boot"}
        # handler returns: {ticketId: "T-001", status: "OPEN"}

  - label: "Verify classification"
    name: verify
    target: helpdesk
    commands:
      - action: check-status
        data:
          ticketId: "${create.ticketId}"
```

**Rules:**
- Step `name` is the namespace — only named steps can be referenced
- Dot-path navigates nested result objects: `${create.details.category}`
- Interpolation happens in `value`, `data`, `params`, `url`, `headers`, `body`, and `await.match` fields
- Unknown step reference throws with available step names listed
- Step names must be unique within the scenario — duplicate names are a parse error
- Regex: `\$\{([^}]+)}`

**Multi-command result aggregation:**
When a step has multiple commands, their results merge into a single map with last-write-wins semantics. Each command's result keys are added to the step result; if two commands produce the same key, the later command's value overwrites. The merged map is what subsequent steps see via `${stepName.field}`.

**Intra-step references are prohibited.** A command within step `X` cannot reference `${X.field}` — the step's result namespace is only available after all commands in the step complete. If command 2 needs data from command 1's result, split them into separate steps with the same target (they will batch into one `dispatch-sequence`).

## Error Handling

The `on-error` field at the scenario level controls failure behaviour. (D10)

| Mode | Behaviour | Use case |
|------|-----------|----------|
| `stop` (default) | Abort all executors immediately | Unattended automations — fail-fast |
| `continue` | Skip the failed step's dependents transitively | Resilient pipelines — run what you can |
| `pause` | Freeze all executors for operator intervention | Human-paced demos — let the operator decide |

**Dependency tracking for `continue` mode:** A step B depends on step A if B references A's results via variable interpolation (`${A.field}`). When A fails with `on-error: continue`, B is skipped. If C depends on B, C is also skipped (transitive).

**Step-level results on failure:**
- Failed step: `{ok: false, error: "<message>"}`
- Remaining commands in the step after the failure: skipped
- Partial command results: discarded — the step result contains only the error, not results from commands that succeeded before the failure. This is intentional: partial results are unreliable (side effects may be incomplete), and dependent steps should not execute against a half-finished state.
- Await timeout: `{ok: false, error: "await timed out"}`
- In `continue` mode: a step that references a failed step via `${failedStep.field}` is skipped transitively. The failed step's result has no usable fields — only `ok` and `error`.

## Speed and Pacing

The `speed` field controls inter-step delay for human-paced scenarios.

```yaml
scenario: helpdesk-demo
speed: 1          # 1 = normal (1000ms between steps), 2 = double speed, 0.5 = half speed
on-error: pause
chapters: [...]
```

- When `speed` is omitted: no inter-step delay — steps execute as fast as possible (the default for automations)
- When `speed` is set: delay between steps is `1000 / speed` ms (opt-in pacing for demos and walkthroughs)
- `speed: 0` is invalid (would mean infinite delay)
- Executors apply the delay between steps, not between commands within a step
- Speed can be adjusted at runtime via control messages (pause, resume, step, speed)
- Step-level `delay` replaces the speed-based inter-step delay for that step — the author's explicit wait overrides global pacing

Automations omit `speed` for fastest execution. Human-paced scenarios set `speed` to opt into inter-step pacing.

## Dispatch Protocol

The orchestrator partitions the scenario into fragments per executor and distributes them over WebSocket. (D4)

### Fragment partitioning

1. Walk the step list (flattened from chapters/sections)
2. Group consecutive steps with the same `target` into a fragment
3. Send each fragment to its target executor via `dispatch-sequence`

### Wire protocol

```
Executor → Orchestrator: executor-register
  {op: "executor-register", name: "helpdesk", actions: ["create-ticket", "resolve-ticket"]}

Orchestrator → Executor: dispatch-sequence
  {op: "dispatch-sequence", sessionId: "s-001", executorId: "helpdesk",
   steps: [...], speed: 1.0, paused: false}

Executor → Orchestrator: step-result
  {op: "step-result", sessionId: "s-001", stepName: "create", ok: true,
   result: {ticketId: "T-001"}}

Orchestrator → Executor: executor-control
  {op: "executor-control", sessionId: "s-001", command: "pause|resume|step|speed",
   value: <number for speed>}
```

**Batching rules** (in precedence order):
1. **Variable dependencies break batches:** if a step references any prior step's results via `${...}` interpolation, it cannot be batched with steps before that dependency. The orchestrator dispatches it individually after resolving the variable from the dependency's `step-result`.
2. **Same-target consecutive steps batch:** consecutive steps with the same target and no variable dependencies are batched into one `dispatch-sequence`.
3. **New sequences queue:** sequences arriving while one is running are queued and appended when the current sequence completes.

**Dependency detection:** the orchestrator scans all interpolatable fields (`value`, `data`, `params`, `url`, `headers`, `body`, `await.match`) for `${stepName.field}` patterns. Any match breaks the batch — variable resolution is centralized in the orchestrator, not in executors. Executors receive fully resolved step data and never see `${...}` patterns.

### Browser-only mode

When no backend orchestrator is present, the browser executor runs the full YAML directly — parsing it locally, managing its own step sequencing, and executing commands against the DOM and services.

| Action type | Browser-only behavior |
|---|---|
| ARIA | Execute against DOM |
| GraphQL | HTTP POST via `fetch` to the service's `/graphql` endpoint |
| HTTP | Execute via `fetch` |
| Custom | Skip with diagnostic warning — no `@ScenarioAction` handler registry exists in the browser |

This is the same execution model without the distribution layer. Steps targeting specific service executors (e.g., `target: helpdesk`) are executed by the browser — target names are ignored in browser-only mode since there is only one executor.

## Reconciliation

### Delete (dead code)

Format A's parser and execution code is orphaned — no production code path references it. Delete:

- `pages/backend/scenario/` — `ScenarioParser.java`, `Scenario.java`, `ScenarioStep.java` (sealed interface with AriaStep/GraphQLStep/SimulatedStep), `AriaTarget.java`, `AwaitCondition.java`
- `pages/backend/scenario/src/test/` — `ScenarioParserTest.java` and all test YAML files
- `pages/backend/cross-parser-test/` — cross-parser round-trip tests for the old format

### Keep and adapt

- `pages/backend/scenario-runtime/VariableContext.java` — variable interpolation engine. Proven, matches D11's syntax. Adapt to the new command structure.
- `pages/backend/scenario-runtime/GraphQLDispatcher.java` — HTTP-to-GraphQL dispatch. Adapt to use `domain` + `operation` from the new format.
- `pages/backend/scenario-runtime/ScenarioExecutor.java` — sequential execution with variable context. Refactor for the new step/command hierarchy.
- `pages/backend/scenario-runtime/AriaDispatcher.java` — ARIA command dispatch to browser via push wire. Adapt field names (`element` instead of `target`).

### Distributed executor protocol branch (issue #408)

The implementation described in the "From Protocol to Proof" blog post (2026-08-21) exists on an unmerged feature branch. These classes are not in the current project index. Their relationship to v3:

- `HierarchicalParser` — superseded by the v3 parser. The chapter/section/step/command hierarchy is the same; the command structure differs (v3 uses `action` as type discriminator with uniform command objects).
- `ScenarioOrchestrator` + `SequencePartitioner` — adapt for v3. The orchestration logic (partitioning, dispatching, control messages) is correct. Variable dependency tracking needs updating for the multi-command step model and the result aggregation semantics defined here.
- `@ScenarioAction` + `ActionRegistry` — keep as-is. The annotation and registry are the custom action mechanism described in this spec's Custom Actions section. The CDI proxy superclass-chain traversal is the same.
- `ScenarioExecutorClient` — keep as-is. The service executor library's WebSocket message handling, `dispatch-sequence` reception, and `step-result` reporting match this spec's wire protocol.
- Enhanced `scenario-handler.ts` — adapt for v3. The step queue, pause state, and speed-paced delays are the foundation for the browser executor. Extend with HTTP action support via `fetch`.

### Supersede

- `pages/packages/pages-aria/src/scenario/types.ts` — TypeScript types still reference Format A's flat structure with `delivery` field. Rewrite to match this spec (see Vocabulary Mapping section for target types).
- `parent/docs/platform/scenario-format.md` — v1 spec. Replace with this spec once landed.
- `specs/issue-409-scenario-format/2026-08-24-scenario-format-v2-design.md` — v2 spec (discarded after review). Delete.

## Complete Examples

### Minimal automation — seed data

```yaml
scenario: seed-tickets
on-error: stop
steps:
  - label: "Create hardware ticket"
    name: hw
    target: helpdesk
    commands:
      - action: create-ticket
        data:
          subject: "Laptop won't boot"
          category: "HARDWARE"
          priority: "HIGH"

  - label: "Create software ticket"
    name: sw
    target: helpdesk
    commands:
      - action: create-ticket
        data:
          subject: "Excel crashes on startup"
          category: "SOFTWARE"
          priority: "MEDIUM"
```

### Full scenario — helpdesk demo

```yaml
scenario: helpdesk-demo
description: "IT help desk — customer reports issue through to resolution"
speed: 1
on-error: pause

chapters:
  - label: "Customer Reports Issue"
    sections:
      - label: "Customer submits support request"
        steps:
          - label: "Navigate to intake form"
            target: browser
            commands:
              - action: navigate
                value: "/helpdesk/intake"

          - label: "Fill out and submit the form"
            target: browser
            commands:
              - action: fill
                element: {role: textbox, name: "Subject"}
                value: "Laptop won't boot after update"
              - action: fill
                element: {role: textbox, name: "Description"}
                value: "Updated Windows last night, now stuck on blue screen"
              - action: select
                element: {role: combobox, name: "Category"}
                value: "HARDWARE"
              - action: click
                element: {role: button, name: "Submit"}

          - label: "Verify submission confirmation"
            target: browser
            commands:
              - action: assert
                element: {role: alert, name: "Ticket created"}
                state: {aria-hidden: false}

      - label: "System processes the ticket"
        steps:
          - label: "Inject chat message"
            name: inject
            target: helpdesk
            commands:
              - action: graphql
                domain: connectors
                operation: injectChat
                params:
                  platform: "web-form"
                  sender: "alice.chen@example.com"
                  message: "Laptop won't boot after update"

          - label: "Wait for classification"
            name: classified
            target: helpdesk
            commands:
              - action: graphql
                domain: helpdesk
                operation: getTicketStatus
                params:
                  ticketId: "${inject.ticketId}"
                await:
                  match:
                    category: "HARDWARE"
                  timeout: 30000

  - label: "Agent Resolves Issue"
    sections:
      - label: "Agent picks up the ticket"
        steps:
          - label: "Navigate to agent dashboard"
            target: browser
            commands:
              - action: navigate
                value: "/helpdesk/dashboard"
              - action: click
                element:
                  role: row
                  name: "Laptop won't boot after update"

          - label: "Assign to current agent"
            name: assign
            target: helpdesk
            commands:
              - action: graphql
                domain: helpdesk
                operation: assignTicket
                params:
                  ticketId: "${inject.ticketId}"
                  agentId: "agent-001"

      - label: "Agent resolves and closes"
        steps:
          - label: "Add resolution notes"
            target: browser
            commands:
              - action: fill
                element: {role: textbox, name: "Resolution"}
                value: "Rolled back Windows update, laptop boots normally"
              - action: click
                element: {role: button, name: "Resolve"}

          - label: "Verify resolution"
            target: browser
            commands:
              - action: assert
                element: {role: status, name: "Ticket status"}
                state: {aria-current: "true"}
              - action: wait
                element: {role: button, name: "Resolve"}
                state: {aria-disabled: "true"}
                timeout: 3000
```

### HTTP integration — notify external system

```yaml
scenario: notify-slack
on-error: stop
steps:
  - label: "Create ticket"
    name: create
    target: helpdesk
    commands:
      - action: create-ticket
        data:
          subject: "Server disk full"
          priority: "CRITICAL"

  - label: "Notify Slack channel"
    target: helpdesk
    commands:
      - action: http
        method: POST
        url: "https://hooks.slack.com/services/T00/B00/xxx"
        headers:
          Content-Type: "application/json"
        body:
          text: "Critical ticket created: ${create.ticketId} — Server disk full"
```

## Vocabulary Mapping

### YAML → TypeScript

| YAML concept | TypeScript type | Description |
|---|---|---|
| scenario document | `Scenario` | Top-level parsed scenario |
| chapter | `Chapter` | Narrative grouping |
| section | `Section` | Step grouping |
| step | `Step` | Execution unit with target and commands |
| ARIA command | `AriaCommand` | Browser UI action (`click`, `fill`, etc.) |
| GraphQL command | `GraphQLCommand` | CaseHub service operation |
| HTTP command | `HttpCommand` | Third-party REST call |
| custom command | `CustomCommand` | `@ScenarioAction` handler invocation |
| element reference | `ElementRef` | ARIA role + accessible name |
| await condition | `AwaitCondition` | Poll-until-match condition |
| step result | `StepResult` | `{ok, result?, error?}` |

**Target TypeScript types** (replace `pages/packages/pages-aria/src/scenario/types.ts`):

```typescript
interface Scenario {
  scenario: string;
  description?: string;
  speed?: number;
  actor?: string;
  onError?: 'stop' | 'continue' | 'pause';
  chapters?: Chapter[];
  sections?: Section[];
  steps?: Step[];
}

interface Chapter { label: string; sections: Section[]; }
interface Section { label: string; steps: Step[]; }

interface Step {
  label: string;
  name?: string;
  target: string;
  actor?: string;
  delay?: number;
  commands: Command[];
}

type Command = AriaCommand | GraphQLCommand | HttpCommand | CustomCommand;

interface AriaCommand {
  action: 'navigate' | 'click' | 'fill' | 'select'
        | 'expand' | 'collapse' | 'assert' | 'wait';
  element?: ElementRef;
  value?: string;
  state?: Record<string, unknown>;
  timeout?: number;
}

interface GraphQLCommand {
  action: 'graphql';
  domain: string;
  operation: string;
  params?: Record<string, unknown>;
  await?: AwaitCondition;
}

interface HttpCommand {
  action: 'http';
  method: string;
  url: string;
  headers?: Record<string, string>;
  body?: unknown;
}

interface CustomCommand {
  action: string;
  data?: Record<string, unknown>;
}

interface ElementRef {
  role: string;
  name: string;
  within?: ElementRef;
}

interface AwaitCondition {
  match: Record<string, unknown>;
  timeout?: number;
  interval?: number;
}

interface StepResult {
  ok: boolean;
  result?: Record<string, unknown>;
  error?: string;
}
```

### YAML → Java

| YAML concept | Java type | Module |
|---|---|---|
| scenario document | `Scenario` (record) | scenario-runtime |
| chapter | `Chapter` (record) | scenario-runtime |
| section | `Section` (record) | scenario-runtime |
| step | `Step` (record) | scenario-runtime |
| command | `Command` (sealed interface) | scenario-runtime |
| element reference | `AriaTarget` (record) | scenario (existing, adapted) |
| await condition | `AwaitCondition` (record) | scenario (existing) |
| step result | `ExecutionResult` (record) | scenario-runtime (existing) |
| variable context | `VariableContext` | scenario-runtime (existing) |
| custom action handler | `@ScenarioAction` annotation | scenario-client |
| action parameter access | `ActionContext` interface | scenario-client |

## Deferred Capabilities

The following v1 features are intentionally deferred from v3. Each is tracked as a separate GitHub issue.

| Capability | v1 location | Rationale for deferral | Issue |
|---|---|---|---|
| Trigger model (TimeTrigger, AfterTrigger, DataTrigger) | §3 | v3's sequential model covers the majority of use cases. Triggers add significant orchestrator complexity. | [#424](https://github.com/casehubio/parent/issues/424) |
| Data shapes (bulk/stepped/stream) | §5.1 | Requires pacing integration with the speed model. Designed on the protocol branch but not yet reconciled with v3's command structure. | [#425](https://github.com/casehubio/parent/issues/425) |
| Loop field | §1 | Continuous restart is a controller/runtime concern. The format supports it trivially once the controller is built. | [#426](https://github.com/casehubio/parent/issues/426) |
| SSE event-based await | §8.1 | v3's GraphQL polling covers the primary server-side completion detection case. Push-event await requires push wire subscription management. | [#427](https://github.com/casehubio/parent/issues/427) |
| External data file references | §5.2 | Deferred with data shapes — same design scope. | [#425](https://github.com/casehubio/parent/issues/425) |

**Not restored** (intentional removals, not deferrals):
- `fast-fallback` — v1 workaround for slow UI automation. v3's clean action type separation (GraphQL vs ARIA) makes this unnecessary. Fast execution uses `speed` or writes a GraphQL-only scenario.
- Verification mode — runtime/executor configuration, not a format concern. The same YAML runs in normal or verification mode; the executor decides whether to assert or execute. No format change needed.

## References

- `pages/backend/scenario-runtime/src/main/java/.../VariableContext.java` — variable interpolation implementation
- `pages/backend/scenario-runtime/src/main/java/.../GraphQLDispatcher.java` — GraphQL HTTP dispatch
- `pages/backend/scenario-runtime/src/main/java/.../ScenarioExecutor.java` — step execution engine
- `pages/backend/scenario/src/test/resources/scenarios/*.yaml` — Format A test YAML (to be deleted)
- `pages/wksp/specs/issue-408-scenario-engine/2026-08-20-distributed-executor-protocol-design.md` — Format B distributed executor protocol spec
- `platform/graphql-generator/.../GraphQLResolverProcessor.java` — annotation processor generating GraphQL resolvers
- `platform/platform-api/.../McpDomain.java`, `PlatformQuery.java`, `PlatformMutation.java` — SPI annotations
- `pages/packages/pages-aria/src/scenario/types.ts` — TypeScript types (to be rewritten)
- D1–D11 decisions: `specs/issue-409-scenario-format/decisions.md`
- Hybrid shorthand evaluation: casehubio/parent#423
