## D1: Backend Push Architecture (revised after R1-04, then restored)

**Choice:** Quarkus WebSocket endpoint + CDI event bridge
**Alternatives:**
- Direct EventBroadcaster injection in TicketService — fewer classes, demonstrates the platform push contract directly, but leaks push concerns into the domain service and isn't how production code should be structured
- SSE bridge over pages-push — simpler frontend but doesn't demonstrate the actual wire protocol, loses replay-on-reconnect and topic subscription
**Rationale:** Example apps are skeleton starter apps — production code quality by default. CDI events decouple TicketService from push infrastructure: the service fires domain events, a separate observer bridges to EventBroadcaster. This teaches both the CDI decoupling pattern AND the platform push contract (visible in the observer). Extensible — audit, analytics, webhooks added via new observers without touching TicketService.
**Trade-offs:** More classes (TicketEvent, TicketPushObserver). Justified — this is the production pattern people should copy. @ObservesAsync virtual thread gotchas apply but are documented in the garden (GE-20260613-6527d0, GE-20260605-494ed0).
**Exploration:** quick → R1-04 revised to direct broadcast → user restored CDI as production pattern
**Status:** captured

## D3: Push Topic Structure (revised after R1-03)

**Choice:** Structured topic hierarchy with wildcard subscription — `helpdesk:tickets`, `helpdesk:notifications`, `helpdesk:metrics` consumed via `helpdesk:**`
**Alternatives:**
- Single flat topic `helpdesk:events` — simpler but misses demonstrating the platform's trie-based wildcard topic matching
- Per-ticket topics (helpdesk:tickets:{id}) — enables per-ticket filtering but dashboard shows everything, over-engineered
**Rationale:** Structured topics with wildcard subscription demonstrates the platform's TopicRegistry matching primitive (R1-03). Still one subscription on the frontend (`helpdesk:**`), but the topic hierarchy shows the naming convention real apps will follow. Also enables the scenario controller to subscribe selectively if needed.
**Trade-offs:** Backend broadcasts to three topic names instead of one. Negligible complexity increase.
**Exploration:** quick
**Status:** revised (R1-03: structured topics demonstrate wildcard matching)

## D4: Pipeline Timeline Strategy

**Choice:** Per-ticket pipeline nodes — each ticket gets timeline nodes for its journey through fixed stages (message → ticket → classify → assign → resolve → notify)
**Alternatives:**
- Global event stream (one node per push event, no per-ticket grouping) — simpler strategy but doesn't show pipeline progression per ticket
- Per-stage aggregation / Kanban (ticket counts at each stage) — useful for operator overview but doesn't show individual ticket progression
**Rationale:** The issue asks for "pipeline flow as events progress." Per-ticket nodes show each ticket moving through stages in real time. Strategy's `toNodes()` maps ticket state to stage completion. Fixed stages are specific to the helpdesk example; the TimelineStrategy pattern itself provides the extension point.
**Trade-offs:** More complex strategy implementation. Worth it — this is the primary visualization feature.
**Exploration:** quick
**Status:** captured (R1-06: added per-stage alternative, kept original choice)

## D5: Scenario Controller Placement

**Choice:** Collapsible side panel using pages split container / sidebar interactivity, communicating via backend REST API
**Alternatives:**
- Separate route (/helpdesk vs /helpdesk/scenario) — navigational separation loses side-by-side demo experience
- Floating overlay — visually cluttered, obscures dashboard content
**Rationale:** Side panel gives side-by-side demo experience. Backend API communication means it also works standalone (separate browser window, different device). Pages dock infrastructure provides the panel chrome.
**Trade-offs:** Dashboard has less horizontal space when panel is open. Acceptable — panel is collapsible.
**Exploration:** quick
**Status:** captured

## D5a: Scenario Controller UX Constraints (revised after R1-05)

**Choice:** Step-by-step pacing with explicit user control + push event observation
**Details:**
- Click feedback on all interactive elements (buttons, tabs — press states, visual response)
- Text entry is visible before submission — user sees what the scenario will submit
- Explicit "Next" / "Submit" button to advance — no auto-progression
- Step indicator (e.g. "Step 2/5") showing position in scenario
- Controller subscribes to push events (helpdesk:**) to observe automated stage completions (classify, assign) and advance the step indicator accordingly
**Rationale:** Demo UX — the viewer needs time to read and understand what's happening before the next action fires. Push observation solves the gap where automated backend stages (classify, assign) complete asynchronously — the controller watches for these events to update its state and enable the next user action.
**Exploration:** quick
**Status:** revised (R1-05: added push event observation for automated stage tracking)

## D6: Frontend Push Connection

**Choice:** EventStreamController from @casehubio/pages-component (Lit Reactive Controller wrapping EventStream/EventConnection)
**Alternatives:**
- Manual EventConnection from pages-data — more boilerplate, duplicates what EventStreamController provides
- loadSite() with push data sources — the pages DSL pipeline handles push natively, but the scenario controller requires imperative interactive UI (step pacing, text preview, form population) that doesn't fit the declarative pages model
**Rationale:** EventStreamController auto-manages WebSocket lifecycle (connect on hostConnected, disconnect on hostDisconnected), triggers requestUpdate() on events, exposes `latest`, `all`, and `status`. Designed for exactly this use case. Now lives in pages-component (moved from blocks-ui-core).
**Trade-offs:** Commits to Lit as the component framework. This is a continuation of the existing helpdesk app (already Lit/Vite) and is explicit here.
**Exploration:** quick
**Status:** revised (R1-01/R1-02: loadSite() bypass made explicit with rationale; Lit commitment documented)

## D2: Frontend Architecture

**Choice:** Single Lit shell component composing blocks-ui/pages components inline
**Alternatives:**
- loadSite() with pages DSL — the platform's primary rendering API, but the scenario controller panel requires imperative interactive elements (step-by-step pacing, form population, push event observation) that don't fit the declarative pages component tree model
- Split into sub-components — more modular but adds indirection that obscures learning
- Controller pattern — extracts push logic but overhead without benefit at this scope
**Rationale:** Example app prioritises readability. One file shows the full picture: push subscription, data flow, component wiring. A learner sees everything in context. Lit is the existing framework choice from the first helpdesk iteration. The loadSite() bypass is justified by the scenario controller's interactive requirements (R1-01).
**Trade-offs:** File will be larger (~300-400 lines). The example teaches Lit + platform-components composition, not the pages DSL pipeline. This is an intentional scope choice — the pages DSL is demonstrated elsewhere (pages-examples gallery).
**Exploration:** quick
**Status:** revised (R1-01/R1-02/R1-07: loadSite() alternative made explicit, Lit commitment documented, monolith-as-pedagogy noted as contested but chosen)
