# Helpdesk UI Upgrade — blocks-ui Components, Runtime State, Push

**Issue:** casehubio/parent#412
**Branch:** `issue-408-scenario-engine`
**Date:** 2026-08-13

## Summary

Replace the vanilla Lit ops dashboard in `casehub-examples/helpdesk` with platform components (blocks-ui, pages) and real-time push via the pages-push WebSocket wire protocol. Add a scenario controller panel for step-by-step demo pacing.

## Goals

1. Metrics use `blocks-kpi-metric-row`
2. Ticket table uses `pages-table`
3. Pipeline flow uses `blocks-timeline` for per-ticket stage visualization
4. Backend broadcasts events via pages-push protocol (WebSocket — see §Push Infrastructure for transport rationale)
5. Frontend receives push updates — no polling
6. Scenario controller provides step-by-step demo UX with explicit user pacing

## Non-Goals

- No pages DSL / `loadSite()` integration — the scenario controller requires imperative interactive UI (step pacing, form population, push observation) that doesn't fit the declarative pages model. The pages DSL is demonstrated in the pages-examples gallery.
- No new platform push infrastructure — uses existing `casehub-pages-push` and `casehub-pages-push-runtime` as-is.
- No persistent storage — the helpdesk remains in-memory for demo purposes.

---

## Backend Architecture

### Dependencies (new)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets-next</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-push</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-push-runtime</artifactId>
</dependency>
```

### Push Infrastructure

The pages-push runtime provides CDI beans for `TopicRegistry`, `EventStore` (in-memory, bounded), and `EventBroadcaster`. The consumer must provide a `SessionSender` bean that bridges the WebSocket transport.

**Transport rationale — WebSocket, not SSE:** Issue #412 references "SSE push" but the pages-push protocol is WebSocket-based. The issue's parenthetical "(pages-push protocol)" reveals the intent — use the existing pages-push infrastructure. The transport label was simply wrong. Pages-push uses WebSocket for bidirectional `listen`/`unlisten` subscription management with wildcard topic routing and sequence-numbered replay — capabilities SSE's unidirectional model cannot provide. The platform's existing SSE broadcasters (work's `WorkItemEventBroadcaster`, platform's `NotificationSseResource`) are a different pattern: server-driven, fixed-filter, no client subscription control. Issue #412's acceptance criteria should be updated to say "pages-push protocol (WebSocket)" to match the actual infrastructure.

#### HelpdeskPushEndpoint

A `@WebSocket` endpoint at `/push` that manages connections:

- **onOpen**: Generates a connection ID, stores the `WebSocketConnection` in a `ConcurrentHashMap`, registers with `TopicRegistry`.
- **onMessage**: Parses incoming text as `PushRequest`. Handles `listen` (register topics with optional `since` cursors for replay) and `unlisten` (deregister topics).
- **onClose**: Removes connection from the map and unregisters all topics from `TopicRegistry`.

#### HelpdeskSessionSender

A CDI `@ApplicationScoped` bean implementing `SessionSender`. Looks up the `WebSocketConnection` by connection ID from the shared map and calls `sendText()`. Handles closed connections gracefully (the event is persisted in `EventStore`; replay delivers on reconnect).

### CDI Event Bridge

#### TicketEvent

A record carrying the event type and ticket data:

```java
public record TicketEvent(Type type, Ticket ticket) {
    public enum Type { CREATED, CLASSIFIED, ASSIGNED, RESOLVED }
}
```

#### NotificationEvent

```java
public record NotificationEvent(String to, String message) {}
```

#### TicketPushObserver

An `@ApplicationScoped` bean that observes CDI events synchronously and bridges to `EventBroadcaster`:

- `@Observes TicketEvent` → `broadcaster.broadcast("helpdesk:tickets", payload)`
- `@Observes NotificationEvent` → `broadcaster.broadcast("helpdesk:notifications", payload)`

**Synchronous events guarantee ordering.** `TicketCreationHandler.onMessage()` calls `create()` → `classify()` → `assign()` sequentially. With `@Observes` + `Event.fire()`, each broadcast completes before the next `TicketService` method fires its event. This ensures the frontend receives CREATED before CLASSIFIED before ASSIGNED — critical for the pipeline timeline visualization. `@ObservesAsync` would process events on separate managed executor threads with no ordering guarantee.

After each ticket event, also broadcasts updated metrics to `helpdesk:metrics`:

```json
{"total": 3, "open": 1, "resolved": 2, "notified": 2}
```

The metrics topic is intentionally redundant — the frontend could derive these counts from ticket events. This is a deliberate design choice for demonstration purposes: it shows per-topic specialization where different consumers subscribe to different topics based on their needs.

### TicketService Changes

Add `@Inject Event<TicketEvent>` and fire synchronously (`Event.fire()`) from existing state-changing methods:

- `create()` → fires `TicketEvent(CREATED, ticket)`
- `classify()` → fires `TicketEvent(CLASSIFIED, ticket)`
- `assign()` → fires `TicketEvent(ASSIGNED, ticket)`
- `resolve()` → fires `TicketEvent(RESOLVED, ticket)`

Synchronous firing is deliberate — see §CDI Event Bridge for ordering rationale.

The `NotificationService` fires `NotificationEvent` synchronously after sending.

### Topic Structure

Three topics in a `helpdesk:` namespace:

| Topic | Payload | When |
|-------|---------|------|
| `helpdesk:tickets` | `{type, ticket}` | Any ticket state change |
| `helpdesk:notifications` | `{to, message}` | Notification sent |
| `helpdesk:metrics` | `{total, open, resolved, notified}` | After each ticket event |

The structured hierarchy demonstrates the platform's trie-based topic matching (`TopicRegistry`). The frontend uses per-topic `EventStreamController` instances — one per topic — providing type safety (`EventStreamController<TicketEvent>` vs `EventStreamController<unknown>`) and topic-scoped event histories. The `EventStreamPool` reuses the same underlying WebSocket connection across all controllers, so per-topic controllers add no wire overhead. The platform's wildcard subscription capability (`helpdesk:**`) is available but not used here — per-topic controllers are the cleaner pattern for typed consumption.

---

## Frontend Architecture

### Framework

Lit + Vite (continuation of existing helpdesk app). The shell component is a single `LitElement` that composes blocks-ui and pages components inline.

### Dependencies (new)

- `@casehubio/pages-component` — `EventStreamController` (Lit Reactive Controller for WebSocket push)
- `@casehubio/blocks-ui-kpi-metric-row` — KPI metric cards
- `@casehubio/pages-table` — interactive data table
- `@casehubio/blocks-ui-blocks-timeline` — strategy-driven timeline

Package names must be verified against each package's `package.json` `name` field — directory names don't always match (GE-20260803-17fc03).

### Push Connection

`EventStreamController` from `@casehubio/pages-component` manages the WebSocket lifecycle. One controller per topic — the `EventStreamPool` reuses the same underlying WebSocket connection (no overhead):

```typescript
private _ticketPush = new EventStreamController<TicketEvent>(this, '/push', 'helpdesk:tickets');
private _notifPush = new EventStreamController<NotificationEvent>(this, '/push', 'helpdesk:notifications');
private _metricsPush = new EventStreamController<MetricsSnapshot>(this, '/push', 'helpdesk:metrics');
```

Auto-connects on `hostConnected`, disconnects on `hostDisconnected`, triggers `requestUpdate()` on new events. The `all` property provides the full event history; `latest` provides the most recent event. Each controller filters to its own topic — `all` contains only payloads for that topic.

**Live-events-only model:** On first connect, `EventConnection.listen()` sends no `since` cursors — clients receive only events broadcast after they connect. There is no initial state hydration from the push layer. On reconnect (after a temporary disconnect), `EventConnection` tracks per-topic sequence numbers and sends `since` cursors for incremental replay, avoiding duplicates. The demo's sequential flow (start backend → open dashboard → run scenario) means the dashboard is open before any events are created, so the first-connect limitation does not affect the normal demo path. If late-connect support is needed (e.g., opening a second browser tab mid-demo), the existing `GET /tickets` REST endpoint provides a snapshot for initial hydration.

State updates read from each controller:

```typescript
get _metrics(): MetricsSnapshot {
  return this._metricsPush.latest ?? { total: 0, open: 0, resolved: 0, notified: 0 };
}
```

Ticket state is derived from `_ticketPush.all` via a computed getter (see §Ticket Table). No incremental accumulation — the getter rebuilds on each render, which is naturally idempotent and handles reconnection replay correctly.

### Component Layout

```
┌──────────────────────────────────────────────┬─────────────────┐
│  Header: "IT Help Desk"                      │  Scenario       │
├──────────────────────────────────────────────┤  Controller     │
│  <blocks-kpi-metric-row>                     │                 │
│  Total | Open | Resolved | Notified          │  Step 2/5       │
├──────────────────────────────────────────────┤  ┌───────────┐  │
│  <pages-table>                               │  │ Preview   │  │
│  Subject | Status | Category | Priority | …  │  │ text...   │  │
├──────────────────────────────────────────────┤  └───────────┘  │
│  <blocks-timeline>                           │  [Submit]       │
│  Per-ticket pipeline nodes                   │  [Next →]       │
├──────────────────────────────────────────────┤                 │
│  Notifications                               │                 │
└──────────────────────────────────────────────┴─────────────────┘
```

The dashboard and scenario controller are separated by a pages split container (`wireInteractivity("split", ...)`), making the controller panel resizable via drag handle.

### Metrics — blocks-kpi-metric-row

Bound to the `helpdesk:metrics` push topic via the `_metrics` getter defined in §Push Connection (`_metricsPush.latest` with zero-state fallback):

```typescript
get _metricDefs(): MetricDefinition[] {
  return [
    { key: 'total', value: this._metrics.total, label: 'Total' },
    { key: 'open', value: this._metrics.open, label: 'Open', status: 'warning' },
    { key: 'resolved', value: this._metrics.resolved, label: 'Resolved', status: 'normal' },
    { key: 'notified', value: this._metrics.notified, label: 'Notified' },
  ];
}
```

Rendered as:
```html
<blocks-kpi-metric-row .metrics=${this._metricDefs} columns="4" density="compact">
</blocks-kpi-metric-row>
```

### Ticket Table — pages-table

Ticket state is derived from the accumulated event history in `_ticketPush.all`. A computed getter rebuilds the ticket map on each render — simple, idempotent, and naturally handles reconnection replay (replayed events overwrite stale state):

```typescript
get _tickets(): Ticket[] {
  const map = new Map<string, Ticket>();
  for (const event of this._ticketPush.all) {
    map.set(event.ticket.id, event.ticket);
  }
  return [...map.values()];
}
```

No `@state()` field or imperative event handler is needed — `EventStreamController` triggers `requestUpdate()` when new events arrive, and the getter re-derives from the full `.all` history on each render cycle. For the helpdesk's event volume (tens of events per demo run), the O(n) scan is negligible.

Column config: Subject, Status (badge), Category, Priority (badge), Customer, Assignee, Actions (resolve button).

### Pipeline Timeline — blocks-timeline

A custom `HelpdeskPipelineStrategy` implements `TimelineStrategy`:

```typescript
const STAGES = ['created', 'classified', 'assigned', 'resolved'];

const STATUS_TO_STAGE: Record<string, number> = {
  OPEN: 0,      // created
  TRIAGED: 1,   // classified
  ASSIGNED: 2,  // assigned
  RESOLVED: 3,  // resolved
  CLOSED: 3,    // same as resolved for pipeline purposes
};

class HelpdeskPipelineStrategy implements TimelineStrategy<Ticket[]> {
  defaultLayout: Layout = 'vertical';

  toNodes(tickets: Ticket[]): TimelineNode[] {
    return tickets.flatMap(ticket => {
      const completedIdx = STATUS_TO_STAGE[ticket.status] ?? 0;
      return STAGES.map((stage, i) => ({
        key: `${ticket.id}:${stage}`,
        label: `${ticket.subject} — ${stage}`,
        status: i < completedIdx ? 'completed'
              : i === completedIdx ? 'active'
              : 'pending',
        timestamp: stage === 'created' ? ticket.createdAt
                 : stage === 'resolved' ? ticket.resolvedAt
                 : undefined,
        actor: stage === 'assigned' ? ticket.assigneeId : undefined,
        category: ticket.id,
      }));
    });
  }
}
```

Four stages map 1:1 to `TicketEvent.Type` values. The earlier draft included `received` (pre-ticket message intake) and `notified` (from `NotificationEvent`), but these cross domain boundaries — `received` precedes ticket creation and `notified` belongs to the notification domain, not the ticket lifecycle. Keeping the pipeline to ticket-lifecycle stages ensures the strategy can derive all state from `Ticket` data alone.

`STATUS_TO_STAGE` maps `TicketStatus` enum values to stage indices. The `Ticket` model has `createdAt` and `resolvedAt` timestamps; intermediate stages (`classified`, `assigned`) have no per-stage timestamps in the model, so `timestamp` is `undefined` for those nodes — the timeline displays progression without exact times.

Layout: `vertical`. Nodes grouped by ticket (via `category`), showing each ticket's progression through the pipeline stages. As push events arrive, node statuses update from `pending` → `active` → `completed`.

Data binding — the timeline receives push-driven ticket state via the `.data` property:

```html
<blocks-timeline .data=${this._tickets} .strategy=${this._pipelineStrategy}></blocks-timeline>
```

The `.data` binding triggers `willUpdate` → `strategy.toNodes(data)` on every state change, re-rendering the pipeline nodes. The `.endpoint` property (REST-based data loading via `DataSourceMixin`) is not used — all data arrives through push.

### Scenario Controller Panel

The scenario controller is a resizable side panel that drives the demo step-by-step. It communicates with the backend via REST API and observes push events to track automated stage completions.

#### Data Flow

1. Controller loads scenario steps from an embedded TypeScript module (`src/scenarios/help-desk-basic.ts`), imported at build time — no runtime fetch or YAML parsing
2. Each step has: description, action type (bootstrap/submit/resolve), and parameters
3. User reads the step description and preview text
4. User clicks "Submit" or "Next" to execute the action
5. For actions with automated follow-up (submit → classify → assign), the controller subscribes to `helpdesk:tickets` push events and waits for the automated stages to complete before enabling the next step
6. Step indicator updates: "Step 2/5 — Classify (waiting...)" → "Step 2/5 — Classify (done)" → "Step 3/5"

#### Push Observation

The controller creates its own `EventStreamController<TicketEvent>('/push', 'helpdesk:tickets')`. The `EventStreamPool` reuses the same underlying WebSocket connection as the dashboard's controllers — no additional connection overhead. In standalone mode, the controller's controller is the only one, establishing its own pool connection.

When automated stages complete:

- Ticket CREATED → controller knows the submission was processed
- Ticket CLASSIFIED → classification stage complete
- Ticket ASSIGNED → assignment stage complete

**Correlation mechanism:** The `POST /scenario/inject/chat` endpoint is fire-and-forget — it fires a CDI event and returns before the ticket is created. The controller uses temporal correlation: after submit, it records the length of `_ticketPush.all` and watches for the next CREATED event beyond that index. The CREATED event's `ticket.id` becomes the correlation key for subsequent CLASSIFIED and ASSIGNED events. For a single-user demo, this is reliable — no concurrent ticket creation can interleave.

#### UX Requirements

- **Click feedback**: All buttons and interactive elements have visible press states (`:active` transforms, brief color transitions)
- **Text preview**: Before submitting a chat message, the controller shows the exact text that will be sent in a preview area
- **Explicit pacing**: No auto-progression. User clicks "Next" / "Submit" to advance
- **Step indicator**: Shows current position ("Step 2/5") and stage status
- **Standalone capability**: Communicates via REST API only — works in the same browser window as the dashboard, in a separate tab, or on a different device entirely

#### Standalone Mode

Standalone mode is activated via the `?standalone` query parameter. The app shell checks `new URLSearchParams(location.search).has('standalone')` on startup:

- **Default (no parameter):** Full dashboard with scenario controller in a split layout
- **`?standalone`:** Scenario controller only, no dashboard panel, no split container

When opened standalone (separate window/device), the scenario controller renders without the dashboard panel. It still drives the backend via REST and observes events via its own WebSocket push subscription. The dashboard (if open elsewhere) updates in real time from the same push events.

---

## Data Flow Summary

```
User clicks "Submit" in Scenario Controller
  → POST /scenario/inject/chat {from, channelId, text}
  → TicketCreationHandler @ObservesAsync ReceivedMessage
    → TicketService.create() → fires TicketEvent(CREATED)
    → TicketService.classify() → fires TicketEvent(CLASSIFIED)
    → TicketService.assign() → fires TicketEvent(ASSIGNED)
  → TicketPushObserver @Observes TicketEvent (synchronous — ordered)
    → EventBroadcaster.broadcast("helpdesk:tickets", {type, ticket})
    → EventBroadcaster.broadcast("helpdesk:metrics", {total, open, ...})
  → WebSocket → all connected clients
    → Dashboard: updates metrics, table, timeline
    → Scenario Controller: advances step indicator

User clicks "Resolve" on ticket (dashboard or controller)
  → PUT /tickets/{id}/resolve {resolution}
  → TicketService.resolve() → fires TicketEvent(RESOLVED)
  → NotificationService → fires NotificationEvent
  → TicketPushObserver broadcasts to helpdesk:tickets, helpdesk:notifications, helpdesk:metrics
  → WebSocket → all connected clients update
```

---

## Testing Strategy

### Backend

- **TicketPushObserver**: Verify that CDI events result in `EventBroadcaster.broadcast()` calls with correct topics and payloads
- **HelpdeskPushEndpoint**: Integration test — WebSocket client connects, sends `listen`, receives events after ticket state changes
- **HelpdeskSessionSender**: Verify connection lookup and send, verify graceful handling of closed connections

### Frontend

- **Push connection**: Mock WebSocket, verify state updates on event arrival
- **Component binding**: Verify metrics, table, and timeline update from state changes
- **Scenario controller**: Verify step progression, push event observation, REST API calls
- **E2E (Playwright)**: Bootstrap → submit ticket → verify push-driven updates appear in dashboard (no polling)

---

## Files Changed

### Backend (casehub-examples/helpdesk)

| File | Change |
|------|--------|
| `pom.xml` | Add quarkus-websockets-next, casehub-pages-push, casehub-pages-push-runtime |
| `event/TicketEvent.java` | New — CDI event record (`io.casehub.examples.helpdesk.event`) |
| `event/NotificationEvent.java` | New — CDI event record (`io.casehub.examples.helpdesk.event`) |
| `push/HelpdeskPushEndpoint.java` | New — WebSocket endpoint (`io.casehub.examples.helpdesk.push`) |
| `push/HelpdeskSessionSender.java` | New — SessionSender CDI bean (`io.casehub.examples.helpdesk.push`) |
| `push/TicketPushObserver.java` | New — CDI event → EventBroadcaster bridge (`io.casehub.examples.helpdesk.push`) |
| `TicketService.java` | Modified — inject Event<TicketEvent>, fire on state changes |
| `NotificationService.java` | Modified — inject Event<NotificationEvent>, fire after send |
| `TicketResource.java` | Modified — resolve endpoint fires through TicketService (which fires CDI event) |

### Frontend (casehub-examples/helpdesk/src/main/webui)

| File | Change |
|------|--------|
| `package.json` | Add blocks-ui, pages-component, pages-table dependencies |
| `src/helpdesk-app.ts` | Rewrite — push connection, blocks-ui/pages component composition, scenario controller |
| `src/pipeline-strategy.ts` | New — HelpdeskPipelineStrategy for blocks-timeline |
| `vite.config.ts` | Add Vite aliases for @casehubio/* packages (resolve from source trees, following blocks-ui-examples pattern) |

---

## Garden Gotchas (applicable)

| ID | Summary | Mitigation |
|----|---------|------------|
| GE-20260806-10d369 | EventStreamController is WebSocket, not SSE | Using it correctly — pages-push IS WebSocket |
| GE-20260812-5cd146 | EventConnection drops non-event wire messages | Backend wraps all messages in PushMessage.event() |
| GE-20260806-1f881e | SSEManager eventNames filters on protocol-level event field | Not using SSEManager — using EventStreamController/WebSocket |
| GE-20260704-73bebb | ~~Event op skips lastSeq tracking~~ | **Stale — `EventConnection` now tracks per-topic seq numbers and uses incremental replay on reconnect.** Garden entry should be revised or retired. |
| GE-20260705-ab2230 | SseEventSink has no onClose | Not using SSE — WebSocket has proper lifecycle |
| GE-20260613-6527d0 | CDI events fire before transaction commits | In-memory store, no transactions — not applicable here |
| GE-20260803-17fc03 | Package directory names don't match npm names | Verify package.json name field before adding dependencies |

---

## Decisions

See [decisions.md](decisions.md) for the full decision log with rationale and alternatives.

| # | Decision | Choice |
|---|----------|--------|
| D1 | Backend push | WebSocket + CDI events + EventBroadcaster |
| D2 | Frontend arch | Single Lit shell component, inline composition |
| D3 | Topics | Structured hierarchy (helpdesk:tickets/notifications/metrics) with per-topic controllers — no wildcard |
| D4 | Timeline | Per-ticket pipeline nodes via custom TimelineStrategy |
| D5 | Scenario controller | Resizable side panel via pages split, REST + push observation |
| D5a | Scenario UX | Step-by-step pacing, visible text, explicit Next, push observation |
| D6 | Push connection | EventStreamController from @casehubio/pages-component |
