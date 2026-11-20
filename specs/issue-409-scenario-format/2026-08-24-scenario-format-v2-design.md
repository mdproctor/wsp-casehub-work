# Scenario Format v2 — GraphQL-First, ARIA, Distributed Execution

> **Issue:** casehubio/parent#409
> **Scope:** Revise the scenario format spec to replace delivery modes with
> typed action blocks (GraphQL, ARIA, HTTP), add a translation grammar for
> LLM-driven scenario authoring, and define a JSON Schema for build-time
> validation.
> **Audience:** Platform builders (executor implementation), app builders
> (scenario authoring), LLMs (automated scenario generation)

## 1. Motivation

The existing scenario format (`docs/platform/scenario-format.md`) uses three
delivery modes (`rest`, `ui-form`, `simulated`) that conflate action semantics
with transport mechanism. This creates three problems:

1. **Discoverability** — an LLM has no way to know which REST endpoints or
   connector targets are available without a separate registry.
2. **Simulated is a runtime concern** — whether an event is real or injected
   depends on the build profile (`demo`), not the YAML.
3. **REST is untyped** — endpoints, request bodies, and response shapes are
   undocumented in the format itself.

GraphQL solves all three: the schema is self-describing, type-safe, and
queryable. Every CaseHub service already exposes GraphQL via the
`GraphQLResolverProcessor` — the SPI interface IS the API contract.

## 2. Action Model

A step has exactly one action block. The key determines the type:

| Key | Type | Executor |
|-----|------|----------|
| `mutation` | GraphQL mutation | Backend (Java executor) |
| `query` | GraphQL query | Backend (Java executor) |
| `aria` | ARIA UI automation | Browser (JS executor) |
| `http` | HTTP call | Either (backend or browser via fetch) |

No `delivery`, `action`, `endpoint`, or `target` fields. The action block
replaces all of them.

### 2.1 GraphQL actions

```yaml
- name: start-case
  mutation: startCase
  data:
    namespace: "helpdesk"
    name: "ticket"
    context: { priority: "HIGH" }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mutation` | string | one of mutation/query | GraphQL mutation name |
| `query` | string | one of mutation/query | GraphQL query name |
| `data` | object | no | Variables passed to the operation |

The mutation/query name must match an operation in the target service's
GraphQL schema. Build-time validation cross-references scenario files
against the schema (see §8).

`data` supports the same sourcing as v1: inline objects, external file
references via `source`, and data shapes (`bulk`, `stepped`, `stream`,
`single`). See existing spec §5 — unchanged.

#### What happened to `delivery: simulated`?

Injection of external events is now a regular GraphQL mutation on the
connector's demo SPI. The demo profile exposes an `injectEvent` mutation
(or domain-specific equivalents like `injectMessage`). The YAML calls it
like any other mutation:

```yaml
- name: plumber-message
  mutation: injectMessage
  data:
    connector: chat
    from: "Bob's Plumbing"
    text: "Thursday 2pm works for us"
```

Whether this fires a real CDI event or a simulated one depends on the
active build profile — the YAML doesn't know or care.

### 2.2 ARIA actions

```yaml
- name: create-task
  aria:
    - navigate: "#home"
    - click: "New Task"
    - fill: { Title: "Chase Bob", Domain: "CONTRACTOR_COORDINATION" }
    - click: "Submit"
    - await: { event: "work-item-created" }
```

ARIA actions target elements by accessible name. When the accessible name
is ambiguous, add the ARIA role:

```yaml
- click: { role: button, name: "Submit" }
- fill: { role: textbox, name: "Title", value: "Chase Bob" }
```

#### 2.2.1 ARIA action vocabulary

| YAML | Behaviour |
|------|-----------|
| `navigate: "#path"` | Hash navigation to a page/view |
| `click: "name"` | Click element with matching accessible name |
| `click: { role, name }` | Click element matching role + accessible name |
| `fill: { Name: value, ... }` | Set fields by accessible name |
| `fill: { role, name, value }` | Set a specific field by role + accessible name |
| `select: { name, value }` | Select dropdown option by accessible name |
| `hover: "name"` | Hover over element by accessible name |
| `scroll: { name, to }` | Scroll element (`top`, `bottom`) |
| `await: { ... }` | Wait for a condition (see §5) |

#### 2.2.2 Fill resolution

`fill: { Title: "Chase Bob", Domain: "CONTRACTOR" }` resolves each key
against the accessible names of form fields in the current context:

1. Find the element whose accessible name matches the key.
2. Dispatch based on element role:

| Role | Action |
|------|--------|
| `textbox`, `searchbox` | Type — sets value via keyboard simulation |
| `checkbox`, `switch` | Click — toggles if current state differs |
| `combobox`, `listbox` | Select — selects matching option |
| `spinbutton` | Type — sets numeric value |

3. An accessible name not found in the DOM produces a step warning
   (non-fatal). A DOM element with no matching key is ignored.

### 2.3 HTTP actions

```yaml
# Simple
- name: fetch-weather
  http: GET https://api.weather.com/forecast

# With body
- name: notify-external
  http:
    method: POST
    url: https://hooks.example.com/webhook
    headers:
      Authorization: "Bearer ${env.WEBHOOK_TOKEN}"
    data:
      event: "case-completed"
      caseId: "${steps.start-case.result.id}"
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `http` | string or object | yes | Short form: `METHOD URL`. Long form: object with fields below. |
| `method` | string | yes (long form) | HTTP method |
| `url` | string | yes (long form) | Full URL |
| `headers` | map | no | HTTP headers |
| `data` | object | no | Request body (JSON) |

HTTP actions work in both backend and browser-only executors. The
browser executor uses `fetch`; the backend uses an HTTP client.

## 3. Step Schema (revised)

```yaml
- name: create-task-visible
  trigger: { after: seed-actors, delay: 3000 }
  mutation: createTask
  data:
    title: "Chase Bob for quote"
    domain: "CONTRACTOR_COORDINATION"
  actor: household-admin
  await: { event: "work-item-created" }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Unique identifier within the scenario |
| `mutation` | string | one action required | GraphQL mutation name |
| `query` | string | one action required | GraphQL query name |
| `aria` | Action[] | one action required | ARIA action sequence |
| `http` | string or object | one action required | HTTP call |
| `trigger` | Trigger | no | When to start (fires at T=0 if absent) |
| `data` | object or DataRef | no | Variables / payload / external file |
| `actor` | string | no | Actor identity for `X-Scenario-Actor` header |
| `await` | Await | no | Completion condition |
| `fast-fallback` | object | no | Alternative action for fast speed mode |

**Validation rules:**
- Exactly one action block per step (`mutation`, `query`, `aria`, or `http`).
- `name` must be unique across all steps.
- `actor` is sent as the `X-Scenario-Actor` header on GraphQL and HTTP calls.
- `fast-fallback` provides an alternative action block for fast speed mode
  (e.g., a GraphQL mutation instead of an ARIA sequence).

## 4. Trigger Types

Unchanged from v1 for TimeTrigger and AfterTrigger. DataTrigger gains
GraphQL query support.

### 4.1 TimeTrigger

```yaml
trigger: { at: 10000 }
```

### 4.2 AfterTrigger

```yaml
trigger: { after: "seed-actors", delay: 5000 }
```

### 4.3 DataTrigger

Supports both GraphQL query polling and HTTP polling:

```yaml
# GraphQL query polling
trigger:
  when:
    query: cases
    variables: { filter: { status: "COMPLETED" } }
    match: { totalCount: { gt: 0 } }
    poll: 500

# HTTP polling (third-party or browser-only)
trigger:
  when:
    http: GET https://api.example.com/status
    match: { ready: true }
    poll: 1000
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | one of query/http | GraphQL query name to poll |
| `variables` | object | no | Query variables |
| `http` | string | one of query/http | HTTP endpoint to poll |
| `match` | object | yes | Predicate — fires when response matches |
| `poll` | number (ms) | no | Polling interval (default: 500) |

## 5. Await and Verification

Unchanged semantics from v1. Await confirms the backend processed the
action and can appear as a step-level field or within an `aria` sequence.

| Type | Syntax | Behaviour |
|------|--------|-----------|
| SSE event | `{ event: "work-item-created" }` | Wait for matching SSE event |
| GraphQL query poll | `{ query: "cases", match: { ... } }` | Poll query until match |
| HTTP poll | `{ http: "GET /status", match: { ... } }` | Poll endpoint until match |
| Delay | `{ delay: 2000 }` | Simple wait (ms) |

Timeout rules unchanged from v1 §8.2.

## 6. GraphQL Translation Grammar

Rules for deriving scenario YAML from a service's GraphQL schema. An LLM
reads the schema and applies these rules mechanically.

### 6.1 Schema → Steps

| GraphQL concept | Scenario YAML | Example |
|----------------|---------------|---------|
| Mutation | Step with `mutation:` | `mutation: startCase` |
| Mutation input type | `data:` block with matching field names | `data: { namespace: "x" }` |
| Query | `trigger.when.query` or `await.query` | `query: cases` |
| Query filter input | `trigger.when.variables` or `await.variables` | `variables: { filter: ... }` |
| Query result fields | `match:` predicate | `match: { totalCount: { gt: 0 } }` |
| Subscription | `await: { event: "..." }` | `await: { event: "caseCreated" }` |

### 6.2 Translation procedure

Given a `.graphqls` schema (or the runtime schema from `/graphql/schema.graphql`):

1. **List mutations** — each is a candidate step action.
2. **Read input types** — the mutation's input type defines the `data:` shape.
   Optional fields can be omitted; required fields must be present.
3. **Read return types** — the mutation's return type defines what's available
   for `match:` predicates in downstream `await` or `trigger.when` blocks.
4. **List queries** — each is a candidate for `trigger.when.query` or
   `await.query` polling.
5. **List subscriptions** — each maps to `await: { event: "..." }` where
   the event name is the subscription field name.

### 6.3 Example: engine schema → YAML

Given the engine's GraphQL schema:

```graphql
type Mutation {
  startCase(input: StartCaseInput!): CaseInstanceType!
  signalCase(caseId: ID!, path: String!, value: String!): SignalResult!
  suspendCase(caseId: ID!): CaseControl!
}

input StartCaseInput {
  namespace: String!
  name: String!
  version: String
  context: JSON
}

type Query {
  cases(filter: CaseFilterInput, page: PageInput): CasePage!
}
```

An LLM derives:

```yaml
steps:
  - name: start-case
    mutation: startCase
    data:
      namespace: "helpdesk"
      name: "ticket"
      context: { priority: "HIGH" }
    await:
      query: cases
      variables: { filter: { status: "ACTIVE" } }
      match: { totalCount: { gt: 0 } }

  - name: signal-case
    trigger: { after: start-case, delay: 2000 }
    mutation: signalCase
    data:
      caseId: "${steps.start-case.result.id}"
      path: "/approval"
      value: "APPROVED"
```

### 6.4 Conventions

- **Mutation names** are camelCase, matching the GraphQL operation name
  exactly.
- **Data field names** match the GraphQL input type field names exactly —
  no case conversion.
- **Result references** use `${steps.<name>.result.<field>}` to pass
  outputs from one step to another.

## 7. Distributed Execution Model

### 7.1 Submission

Scenario YAML is submitted to the execution server via a GraphQL mutation:

```graphql
mutation submitScenario(yaml: String!, options: ScenarioOptions): ScenarioRun!
```

The browser-only executor loads YAML directly (file, URL, or embedded).

### 7.2 Fragment distribution

The backend executor partitions the scenario into fragments by action type
and target:

1. **GraphQL steps** — retained by the backend executor.
2. **ARIA steps** — sent to the browser executor via ControlChannel.
3. **HTTP steps** — retained by backend, or sent to browser if browser-only.

Each fragment is a valid YAML subset: it contains the steps assigned to
that executor plus their triggers, data, and awaits. The executor runs its
fragment autonomously — parsing the YAML, building a local trigger graph,
and sequencing steps.

### 7.3 Cross-executor triggers

When an AfterTrigger references a step in a different fragment, the
distributing backend bridges the completion signal:

1. Fragment A completes step `seed-actors`.
2. Backend receives the completion event.
3. Backend sends a trigger signal to Fragment B (which has a step with
   `trigger: { after: seed-actors }`).

The fragment executor treats external trigger signals identically to
local step completions.

### 7.4 Browser-only mode

No backend — the browser executor has the full YAML and runs everything:

- `aria` steps execute directly against the DOM.
- `http` steps execute via `fetch`.
- `mutation`/`query` steps are invalid — the executor skips them with a
  diagnostic warning.

## 8. Build-Time Validation

### 8.1 JSON Schema

A JSON Schema (`scenario-schema.json`) validates the YAML format
mechanically. It enforces:

- Top-level structure (`scenario`, `description`, `speed`, `loop`,
  `on-error`, `data`, `steps`)
- Step schema (exactly one action block, valid trigger/await shapes)
- Action-specific field requirements (e.g., `http` long form requires
  `method` + `url`)
- Data shape constraints (`mode` only with `source`)

The schema is published as a platform artifact. IDE plugins and CI
pipelines validate scenario files against it.

### 8.2 GraphQL cross-reference validation

Beyond format validation, a build-time check can verify that GraphQL
operations named in scenario files actually exist in the target service's
schema:

1. Extract all `mutation:` and `query:` values from the scenario YAML.
2. Load the target service's generated schema
   (`target/generated/schema.graphql`).
3. Verify each operation name exists in the schema.
4. Optionally verify `data:` fields match the operation's input type.

This catches typos and stale scenario files after API changes.

### 8.3 GraphQL enforcement

Every CaseHub service must expose a GraphQL API. Enforcement:

- The `GraphQLResolverProcessor` already generates resolvers from
  `@PlatformQuery`/`@PlatformMutation` on SPI interfaces.
- A build-time check verifies: if a module has an `-api` artifact with
  SPI interfaces, a corresponding `graphql` module must exist.
- CI fails the build if a service has SPIs without GraphQL coverage.

## 9. Top-Level Structure (revised)

```yaml
scenario: life-household-demo
description: "A week in the life of a family"
speed: 1
loop: false
on-error: continue

data:
  transactions: "data/bank-transactions-6mo.json"
  messages: "data/whatsapp-messages.json"

steps:
  - name: seed-actors
    mutation: loadActors
    data: { source: "data/demo-actors.json", mode: bulk }

  - name: plumber-confirms
    trigger: { after: seed-actors, delay: 5000 }
    mutation: injectMessage
    data:
      connector: chat
      from: "Bob's Plumbing"
      text: "Thursday 2pm confirmed"

  - name: approve-invoice
    trigger: { after: plumber-confirms, delay: 8000 }
    actor: household-admin
    aria:
      - navigate: "#inbox"
      - click: "Overdue"
      - click: "Approve"
      - fill: { Amount: "450" }
      - click: "Confirm"
      - await: { event: "oversight-gate-resolved" }

  - name: check-weather
    trigger: { at: 0 }
    http: GET https://api.weather.com/forecast?location=london
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `scenario` | string | yes | — | Unique identifier |
| `description` | string | no | — | Human-readable description |
| `speed` | number | no | `1` | Playback speed multiplier |
| `loop` | boolean | no | `false` | Restart when all steps complete |
| `on-error` | enum | no | `continue` | `continue`, `stop`, or `pause` |
| `data` | map | no | — | Named external data file references |
| `steps` | Step[] | yes | — | Ordered list of scenario steps |

## 10. Migration from v1

| v1 concept | v2 equivalent |
|------------|---------------|
| `delivery: rest` + `endpoint:` | `mutation:` or `query:` |
| `delivery: ui-form` + `ui-actions:` | `aria:` |
| `delivery: simulated` + `target:` | `mutation:` (on connector's demo SPI) |
| `action:` field | Removed — the mutation/query name IS the action |
| `endpoint:` field | Removed — GraphQL operation name replaces it |
| `target:` field | Moved into `data.connector` where needed |
| CSS/data-* selectors | ARIA accessible names and roles |
| `fill: { from: data }` | `fill: { Name: value }` with explicit values |
| DataTrigger with `endpoint:` | `trigger.when.query` or `trigger.when.http` |

## References

- `docs/platform/scenario-format.md` — v1 spec being revised
- `platform/graphql-generator/` — `GraphQLResolverProcessor` source
- `engine/graphql/` — engine's GraphQL resolvers (example domain)
- `specs/issue-408-scenario-engine/decisions.md` — parent epic decisions
- WAI-ARIA 1.2 spec — role and accessible name definitions
