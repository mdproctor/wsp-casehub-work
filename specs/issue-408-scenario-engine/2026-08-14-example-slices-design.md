# Example Slices — Progressive Blocks Pattern Teaching

**Issue:** casehubio/parent#413 (slice map), casehubio/parent#412 (helpdesk redesign)
**Epic:** casehubio/parent#408
**Branch:** `issue-408-scenario-engine`
**Date:** 2026-08-14

## Summary

Redesign the example applications as thin teaching slices. Each slice demonstrates a specific Blocks agentic-AI pattern using real platform capabilities (engine, work, workers, qhorus). No hand-coded lifecycle logic — the platform does the work, the slice provides the thinnest domain skin.

This spec defines the slice map (all slices) and details slice 1 (helpdesk). Subsequent slices will get their own specs when work begins on them.

## Principles

- **Thinnest slice possible** — teaches one pattern grouping, not a whole application
- **Borrow, don't build** — reuse logic and UIs from full apps (devtown, aml, clinical)
- **Platform does the work** — CaseDefinition YAML drives lifecycle, engine orchestrates, work manages queues
- **Single embedded deployment** — one Quarkus app, all in-memory, fast startup
- **Two perspectives** — end-user view and ops view in every slice
- **Mini slide deck per slice** — each slice ships with a short teaching deck that explains the concept before the user runs the demo. Addresses the complexity criticism: one concept at a time, explained then demonstrated.

---

## Slice Map

Each slice is in a different domain — one capability focus per domain, never repeated. This keeps slices thin and avoids building out any single domain beyond its teaching purpose.

| # | Domain | Capability Focus | Key Concepts |
|---|--------|-----------------|--------------|
| 1 | **IT Helpdesk** | Case, Worker, WorkItem, SEQUENCE, bindings | CaseDefinition YAML, AI-as-worker, work queues |
| 2 | **Code Review** (software) | DEBATE, conversation protocol, convergence, epistemic ground | Speech acts, obligations, common ground |
| 3 | **Incident Response** (security) | SUPERVISOR, oversight gates, trust scoring, routing rationale | Graduated autonomy, risk classification |
| 4 | **Construction Planning** (engineering) | HTN, GOAP, decomposition strategies, compound plans | Sub-cases, CompletionSemantics (All/MofN/FirstWins), task trees |
| 5 | **Procurement** (supply chain) | Negotiation, contract net, commitment lifecycle | NegotiationProjection (casehubio/blocks#104), speech acts |
| 6 | **Factory Floor** (manufacturing) | Agent Mesh, channel semantics, affordances, disposition routing | Work/observe/oversight channels, ACTIVE/REACTIVE/SILENT |
| 7 | **Shipment Tracking** (logistics) | Tiered summarisation, observation pipeline, real-time tracking | Per-shipment → per-route → fleet-wide summarisation tiers |
| 8 | **Loan Application** (financial) | Pipeline with gates, M-of-N approval, oversight | Multi-approver sign-off, regulated stage transitions |
| 9 | **Patient Referral** (healthcare) | Resilient execution, escalation chains, DLQ/retry | Fault tolerance, can't-drop semantics, progressive recovery |
| 10 | **Insurance Claim** (regulatory) | Accountability chain, Art.12 ledger, compliance audit | Tamper-evident audit, every decision traceable |
| 11 | **Recruitment Pipeline** (HR) | Peer review, maker-checker, competitive selection | Multiple independent assessors, two-stage validation |
| 12 | **Ops Monitoring** (platform) | Watchdog monitoring, system summarisation, queue health | The platform watching itself, 11 watchdog conditions |
| 13 | **Kubernetes Deployment** (infrastructure) | Desired state reconciliation, drift detection, ops coordination | Declare desired state, detect drift, reconcile, report |

Each slice is standalone — runnable independently with `mvn quarkus:dev`. Later slices may reference concepts from earlier ones but don't depend on them at build time.

---

## Slice 1: Helpdesk — Case + Worker + Work Item

### What the user learns

1. **A Case is a defined unit of work** — write a CaseDefinition YAML, the engine runs it
2. **AI is just another Worker** — LLM classifier and auto-resolver are worker functions, dispatched the same way as human tasks
3. **Bindings connect triggers to capabilities** — declarative wiring, not imperative code
4. **Work Items are the human handoff** — engine creates a WorkItem, it appears in a queue, human claims and resolves it
5. **Routing is a platform decision** — the case definition declares capabilities and candidates, the engine picks the worker
6. **Two perspectives, one system** — end-user sees ticket status, ops sees queues/cases/workers

### Architecture

```
Chat message (scenario inject)
  → InboundSignalBridge (casehub-engine-inbound)
    → Creates case from registered CaseDefinition
  → Binding fires: triage capability (contextChange filter: ".message != null")
    → Worker: classifies ticket (keyword matching worker function)
    → contextWrite merges classification into case context
  → Binding fires: auto-resolve (contextChange filter + when guard: ".category == 'ACCESS'")
    → Worker: auto-resolves → case context updated → goal met → case completes
  → Binding fires: human-resolve (contextChange filter + when guard: ".category != 'ACCESS'")
    → humanTask binding → WorkItem created in casehub-work
      → Appears in ops queue (blocks-work-item-inbox)
      → Human claims → resolves → outputMapping writes to case context → goal met → case completes
  → Engine lifecycle events → CDI events → TicketPushObserver → WebSocket push → Dashboard updates
```

### CaseDefinition YAML

```yaml
dsl: "0.1"
version: "1.0.0"
name: helpdesk-ticket
namespace: io.casehub.examples.helpdesk
title: IT Help Desk Ticket

spec:
  capabilities:
    - name: triage
      description: Classify and prioritize the ticket
      inputSchema: "{ message: .message, from: .from }"
      outputSchema: "{ category: .category, priority: .priority }"
    - name: resolve
      description: Auto-resolve simple tickets
      inputSchema: "{ message: .message, category: .category }"
      outputSchema: "{ resolution: .resolution }"

  workers:
    - name: keyword-classifier
      description: Classifies tickets by keyword matching
      capabilities: [triage]
      do:
        - classify:
            # Worker function provided by CDI — see KeywordClassifierFunction
            call: io.casehub.examples.helpdesk.worker.classify

    - name: auto-resolver
      description: Auto-resolves simple access tickets
      capabilities: [resolve]
      do:
        - resolve:
            call: io.casehub.examples.helpdesk.worker.autoResolve

  bindings:
    - name: triage-on-message
      capability: triage
      on:
        contextChange:
          filter: ".message != null and .category == null"
      contextWrite: ". + { category: .triage.category, priority: .triage.priority }"

    - name: auto-resolve-simple
      capability: resolve
      on:
        contextChange:
          filter: ".category != null and .resolution == null"
      when: ".category == 'ACCESS'"
      contextWrite: ". + { resolution: .resolve.resolution, status: 'RESOLVED' }"

    - name: human-resolve-complex
      on:
        contextChange:
          filter: ".category != null and .resolution == null"
      when: ".category != 'ACCESS'"
      humanTask:
        title: "Resolve: hardware/software ticket"
        candidateGroups: [helpdesk-specialists]
        expiresIn: PT4H
        inputMapping: "{ subject: .message, category: .category, priority: .priority, from: .from }"
        outputMapping: "{ resolution: .resolution, status: 'RESOLVED' }"

  goals:
    - name: ticket-resolved
      kind: success
      condition: ".status == 'RESOLVED'"

  completion:
    success:
      allOf: [ticket-resolved]
```

### Backend Changes

**Remove:** Hand-coded `TicketService` lifecycle (classify, assign, resolve methods), `DemoTicketClassifier`, `TicketCreationHandler` bespoke orchestration.

**Add:**
- `helpdesk-ticket.yaml` — CaseDefinition (above)
- `KeywordClassifierFunction` — implements `CallableDispatcher` (registered via `CallableDispatchRegistry`). Reads classification entries loaded at bootstrap, matches keywords, returns category + priority.
- `AutoResolverFunction` — implements `CallableDispatcher`. Returns a canned resolution for ACCESS tickets.
- `HelpdeskInboundPolicy` — implements `InboundWorkItemPolicy` SPI to bridge `ReceivedMessage` CDI events to case creation via `InboundSignalBridge`.

**Keep (adapted):**
- `TicketPushObserver` — adapted to observe engine lifecycle CDI events (`CaseContextChangedEvent`, `PlanItemStatusChangedEvent`) instead of the removed `TicketEvent`. Bridges to `EventBroadcaster` for WebSocket push.
- `HelpdeskPushEndpoint` + `HelpdeskSessionSender` + `ConnectionRegistry` — unchanged WebSocket transport.
- Scenario bootstrap/inject endpoints — adapted to create cases via `CaseHubRuntime.startCase()` instead of calling `TicketService` directly.

**Dependencies to add to helpdesk POM:**
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine</artifactId>
    <version>${casehub.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-schema</artifactId>
    <version>${casehub.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-rest</artifactId>
    <version>${casehub.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-inbound</artifactId>
    <version>${casehub.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-persistence-memory</artifactId>
    <version>${casehub.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-work-runtime</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

### Frontend Changes

**End-user tab** (simplified from current):
- Ticket submission form (scenario-driven or manual)
- Ticket status card (shows case milestones: created → classified → assigned → resolved)
- Notification feed

**Ops tab** (new — existing blocks-ui components):
- `<blocks-work-item-workbench>` — inbox + detail for the specialist queue
- `<blocks-case-explorer>` — browse active/completed cases, view plan items
- Case detail showing milestones, workers dispatched, work item status

Both tabs share the same push connection. Push topics adapted to carry engine lifecycle events.

### Scenario Flow

4 steps:

1. **Bootstrap** — load case definition + keyword classification data
2. **Submit: password reset** — AI worker auto-resolves (case completes without human) → end-user tab shows resolved
3. **Submit: laptop issue** — AI classifies → WorkItem created → visible in ops tab queue
4. **Resolve (ops tab)** — switch to ops, claim WorkItem in `blocks-work-item-workbench`, complete it → case completes → end-user tab shows resolved

Step 2 shows the fully automated path. Steps 3-4 show the human-in-the-loop path. The contrast teaches: same case definition, same engine, different routing based on classification.

### Info Overlay

The `?` button explains:
- **Case** — what it is, how the YAML defines it
- **Worker** — AI classification and auto-resolution as worker functions
- **Work Item** — the human handoff mechanism
- **Engine** — orchestrates the case lifecycle from the definition
- **Blocks** — composable case management building blocks

### Mini Slide Deck

Ships with the slice. 5-6 slides:
1. The problem: integrating AI into business processes
2. What is a Case? (CaseDefinition YAML, engine runs it)
3. What is a Worker? (AI and human are peers)
4. What is a Work Item? (human handoff, queues, claim lifecycle)
5. The demo: watch it work (screenshot of each step)
6. What's next: trust, oversight, negotiation (teaser for later slices)

---

## Testing Strategy

- **CaseDefinition parsing** — verify the YAML loads and bindings resolve
- **Worker functions** — unit test KeywordClassifierFunction and AutoResolverFunction
- **Engine integration** — submit a message, verify the case progresses through milestones to completion
- **WorkItem creation** — verify the humanTask binding creates a WorkItem with correct candidateGroups
- **Push events** — verify engine lifecycle events broadcast to connected WebSocket clients
- **E2E (Playwright)** — full scenario walkthrough: bootstrap → auto-resolve → human-resolve

---

## Decisions

See [slice-decisions.md](slice-decisions.md) for the full decision log.

| # | Decision | Choice |
|---|----------|--------|
| SD1 | Slice structure | 6+ progressive slices, each teaching one pattern grouping |
| SD2 | Slice map | Helpdesk → Review → Incident → Planning → Negotiation → Collaboration → Summarisation |
| SD3 | Slice 1 scope | Case + Worker + WorkItem + bindings + basic routing |
| SD4 | Deployment | Single embedded Quarkus, all in-memory |
| SD5 | Dependencies | Maven deps from existing platform repos |
| SD6 | Build principle | Borrow from full apps, don't build bespoke |
| SD7 | Trust deferred | Slice 3, not slice 1 |
| SD8 | Teaching materials | Mini slide deck per slice |
