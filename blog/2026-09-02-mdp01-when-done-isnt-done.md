---
layout: post
title: "When Done Isn't Done — Designing Saga Compensation for CaseHub"
date: 2026-09-02
entry_type: article
subtype: diary
projects: [casehubio/work]
tags: [saga, compensation, bpmn, lifecycle, architecture, design]
series: issue-238-saga-compensation
---

# When Done Isn't Done — Designing Saga Compensation for CaseHub

A WorkItem marked COMPLETED should be done. That's the contract — seven terminal states, all final, no coming back. Every consumer in the platform relies on it: filter engines remove terminal items from queues, the engine marks PlanItems as finished, observers stop watching, dashboards close the card.

Except sometimes done work needs to be undone.

A clinical trial IRB approval gets reversed when the protocol is withdrawn. A financial decision gets unwound when fraud is detected downstream. A multi-step onboarding process needs to roll back step 7 because step 3's KYC check came back invalid after the fact. In regulated domains — EU AI Act Art.12, GDPR Art.17/22 — compensation isn't optional. The audit trail must capture both the original action and its structured reversal as immutable, causally-linked entries.

CaseHub had no model for this. The workarounds were ugly: create a manual WorkItem to undo previous work (no causal link), cancel the case (wrong semantics — stopping isn't compensating), or leave completed steps as-is and note the failure in the audit trail (inconsistent domain state). This is the Saga pattern gap.

## What a saga actually is

The term gets thrown around loosely. A saga is a specific thing: a sequence of local transactions where each step has a corresponding compensating transaction. If step N fails, the coordinator invokes compensation for steps N-1, N-2, ..., 1.

<svg viewBox="0 0 700 200" xmlns="http://www.w3.org/2000/svg" style="max-width:700px;font-family:monospace">
  <defs>
    <marker id="arr" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><path d="M0,0 L8,3 L0,6" fill="#333"/></marker>
    <marker id="arr-r" markerWidth="8" markerHeight="6" refX="0" refY="3" orient="auto"><path d="M8,0 L0,3 L8,6" fill="#c0392b"/></marker>
  </defs>
  <text x="20" y="30" font-size="13" fill="#666">Forward execution</text>
  <rect x="20" y="40" width="120" height="40" rx="6" fill="#2ecc71" opacity="0.2" stroke="#27ae60"/>
  <text x="45" y="65" font-size="12">Step A ✓</text>
  <line x1="140" y1="60" x2="180" y2="60" stroke="#333" marker-end="url(#arr)"/>
  <rect x="180" y="40" width="120" height="40" rx="6" fill="#2ecc71" opacity="0.2" stroke="#27ae60"/>
  <text x="205" y="65" font-size="12">Step B ✓</text>
  <line x1="300" y1="60" x2="340" y2="60" stroke="#333" marker-end="url(#arr)"/>
  <rect x="340" y="40" width="120" height="40" rx="6" fill="#2ecc71" opacity="0.2" stroke="#27ae60"/>
  <text x="365" y="65" font-size="12">Step C ✓</text>
  <line x1="460" y1="60" x2="500" y2="60" stroke="#333" marker-end="url(#arr)"/>
  <rect x="500" y="40" width="120" height="40" rx="6" fill="#e74c3c" opacity="0.2" stroke="#c0392b"/>
  <text x="525" y="65" font-size="12">Step D ✗</text>

  <line x1="560" y1="85" x2="560" y2="120" stroke="#c0392b" stroke-dasharray="4"/>
  <text x="570" y="115" font-size="11" fill="#c0392b">fault triggers compensation</text>

  <text x="20" y="150" font-size="13" fill="#c0392b">Compensation (reverse)</text>
  <rect x="340" y="160" width="120" height="35" rx="6" fill="#e74c3c" opacity="0.1" stroke="#c0392b" stroke-dasharray="3"/>
  <text x="355" y="182" font-size="11" fill="#c0392b">Undo C ✓</text>
  <line x1="340" y1="177" x2="300" y2="177" stroke="#c0392b" marker-end="url(#arr-r)"/>
  <rect x="180" y="160" width="120" height="35" rx="6" fill="#e74c3c" opacity="0.1" stroke="#c0392b" stroke-dasharray="3"/>
  <text x="195" y="182" font-size="11" fill="#c0392b">Undo B ▶</text>
  <line x1="180" y1="177" x2="140" y2="177" stroke="#c0392b" stroke-dasharray="4" marker-end="url(#arr-r)"/>
  <rect x="20" y="160" width="120" height="35" rx="6" fill="#fff" stroke="#c0392b" stroke-dasharray="3"/>
  <text x="40" y="182" font-size="11" fill="#999">Undo A ...</text>
</svg>

Key properties: each step declares its compensating action upfront. Compensation runs in reverse order. It's idempotent. And it can fail — which is its own problem.

BPMN 2.0 handles this with first-class compensation events: a completed activity can have a compensation boundary event that fires when a compensate event is thrown. The activity enters COMPENSATING, then COMPENSATED. Clean, well-specified, twenty years of production use behind it.

## The terminal invariant problem

CaseHub's WorkItemStatus has twelve values. Seven are terminal. The lifecycle alignment work (issue #240) had just finished auditing every `isTerminal()` consumer across the platform to add FAULTED and OBSOLETE. That audit found six locations with hardcoded status enumeration across four modules — the same fragility pattern at every site.

So when I started thinking about compensation, the obvious BPMN-aligned approach — add COMPENSATING and COMPENSATED to WorkItemStatus, let COMPLETED transition to COMPENSATING — immediately set off alarms. It would break the terminal invariant that the entire platform relies on. Every `isTerminal()` consumer would need re-auditing. Again.

The question became: is a compensated WorkItem the same entity in a new state, or is it new work?

## Compensation is new work

I think it's new work. Here's why.

A WorkItem represents a unit of work requiring human attention. When that work is completed, it's done — the human made their judgment, the decision is recorded. Compensation doesn't undo the judgment; it creates a new task asking someone (often the same person, sometimes a senior reviewer) to reverse the prior decision.

The ledger already models this correctly. `LedgerEntry` has `causedByEntryId` — a forward causal chain where new entries point back to what caused them. Compensation entries don't modify existing entries; they create new ones. The hash chain continues forward. The original decision remains in the tamper-evident record.

So the design: a completed WorkItem stays COMPLETED. A new compensating WorkItem is created with `compensatesWorkItemId` linking back. The original gets a denormalized `compensationStatus` field — NONE, COMPENSATING, COMPENSATED — for query convenience. The terminal invariant is preserved. The compensating WorkItem has its own full lifecycle: PENDING, claim, start, complete or fault.

<svg viewBox="0 0 700 280" xmlns="http://www.w3.org/2000/svg" style="max-width:700px;font-family:monospace">
  <text x="20" y="25" font-size="14" font-weight="bold">Separate-entity compensation model</text>

  <!-- Original -->
  <rect x="20" y="45" width="280" height="90" rx="8" fill="#f0f0f0" stroke="#999"/>
  <text x="30" y="65" font-size="12" font-weight="bold">Original WorkItem</text>
  <text x="30" y="85" font-size="11" fill="#555">status: COMPLETED (stays terminal)</text>
  <text x="30" y="105" font-size="11" fill="#c0392b">compensationStatus: COMPENSATING</text>
  <text x="30" y="125" font-size="11" fill="#666">id: abc-123</text>

  <!-- Arrow -->
  <line x1="300" y1="90" x2="380" y2="90" stroke="#333" stroke-width="2" marker-end="url(#arr)"/>
  <text x="310" y="80" font-size="10" fill="#666">creates</text>

  <!-- Compensating -->
  <rect x="380" y="45" width="280" height="90" rx="8" fill="#fef3e0" stroke="#e67e22"/>
  <text x="390" y="65" font-size="12" font-weight="bold" fill="#e67e22">Compensating WorkItem</text>
  <text x="390" y="85" font-size="11" fill="#555">status: PENDING → ... → COMPLETED</text>
  <text x="390" y="105" font-size="11" fill="#666">compensatesWorkItemId: abc-123</text>
  <text x="390" y="125" font-size="11" fill="#666">own full lifecycle</text>

  <!-- After completion -->
  <line x1="520" y1="135" x2="520" y2="170" stroke="#2ecc71" stroke-dasharray="4"/>
  <line x1="520" y1="170" x2="160" y2="170" stroke="#2ecc71" stroke-dasharray="4"/>
  <line x1="160" y1="170" x2="160" y2="195" stroke="#2ecc71" stroke-dasharray="4" marker-end="url(#arr)"/>
  <text x="250" y="165" font-size="10" fill="#27ae60">on completion, observer updates original</text>

  <rect x="20" y="200" width="280" height="55" rx="8" fill="#eafaf1" stroke="#27ae60"/>
  <text x="30" y="220" font-size="12" font-weight="bold" fill="#27ae60">Original — after compensation</text>
  <text x="30" y="240" font-size="11" fill="#27ae60">compensationStatus: COMPENSATED</text>
</svg>

## But cases ARE the thing being compensated

At the WorkItem level, separate entities make sense. At the case level, I reached a different conclusion.

A case is an orchestration. When it compensates, the case itself is the coordinator — it tracks which steps completed, fires compensating bindings in reverse order, handles failures. COMPENSATING is a phase the case goes through, not new work created alongside it.

So `CaseStatus` gains three values: COMPENSATING (active, not terminal), COMPENSATED (terminal), and COMPENSATION_FAULTED. That last one is important — it's not terminal either. It's analogous to SUSPENDED: the case needs operator attention and can be retried. Making it terminal would mean a partially-compensated case is "done" with no recovery path. Claude caught this during the decision review — I'd originally classified it as terminal, which contradicts the principle that terminal states have no outgoing transitions.

<svg viewBox="0 0 650 200" xmlns="http://www.w3.org/2000/svg" style="max-width:650px;font-family:monospace">
  <rect x="20" y="70" width="130" height="40" rx="6" fill="#ddd" stroke="#999"/>
  <text x="42" y="95" font-size="12">COMPLETED</text>

  <line x1="150" y1="90" x2="210" y2="90" stroke="#333" stroke-width="1.5" marker-end="url(#arr)"/>
  <text x="155" y="82" font-size="9" fill="#666">compensate</text>

  <rect x="210" y="70" width="150" height="40" rx="6" fill="#3498db" opacity="0.2" stroke="#2980b9"/>
  <text x="225" y="95" font-size="12" fill="#2980b9">COMPENSATING</text>

  <line x1="360" y1="80" x2="450" y2="50" stroke="#27ae60" marker-end="url(#arr)"/>
  <text x="380" y="55" font-size="9" fill="#27ae60">all done</text>
  <rect x="450" y="30" width="150" height="35" rx="6" fill="#2ecc71" opacity="0.2" stroke="#27ae60"/>
  <text x="470" y="52" font-size="12" fill="#27ae60">COMPENSATED</text>
  <text x="470" y="75" font-size="9" fill="#27ae60">(terminal)</text>

  <line x1="360" y1="100" x2="450" y2="140" stroke="#c0392b" marker-end="url(#arr)"/>
  <text x="370" y="135" font-size="9" fill="#c0392b">step faults</text>
  <rect x="450" y="120" width="180" height="35" rx="6" fill="#e74c3c" opacity="0.1" stroke="#c0392b"/>
  <text x="460" y="142" font-size="11" fill="#c0392b">COMPENSATION_FAULTED</text>
  <text x="460" y="165" font-size="9" fill="#c0392b">(active — retry allowed)</text>

  <path d="M 540 155 C 540 185, 285 185, 285 110" stroke="#c0392b" fill="none" stroke-dasharray="4" marker-end="url(#arr)"/>
  <text x="370" y="190" font-size="9" fill="#c0392b">operator retry</text>
</svg>

The asymmetry is deliberate. Cases are orchestrations — compensation is a phase they go through. WorkItems are work units — compensation creates new work.

## Not strict reverse — topological reverse

The original design said "strict reverse-completion order" for compensation. Step C before B before A. Claude's decision review pushed back: what about cases with parallel branches?

If steps B and C ran independently (no dependency between them), there's no ordering constraint in compensation either. Strict reverse would serialize them unnecessarily. The engine already has dependency information — Binding produces/consumes relationships, trigger conditions, the EventLog completion graph. A topological sort of the completion DAG gives the correct answer: dependent steps compensate in reverse dependency order, independent steps compensate concurrently.

For a linear chain, topological reverse IS strict reverse. For cases with parallel branches, it's more correct and faster.

## Worker-agnostic by design

CaseHub's engine is worker-agnostic. A Binding can target different worker types — JudgmentTarget (human tasks), CapabilityTarget (automated), SubCaseTarget, ExtensionTarget. Compensation follows the same principle.

Any binding can declare `compensate: <binding-name>` pointing to another binding. The compensating binding can be any worker type. A human approval can be compensated by an automated rollback. A workflow step can be compensated by a human review. The engine doesn't care — it fires the compensating binding, whatever type it is.

```yaml
casePlan:
  bindings:
    - name: irb-review
      target:
        type: judgment
        title: "IRB Protocol Review"
        scope: "casehubio/clinical/irb-review"
      compensate: irb-review-reversal

    - name: irb-review-reversal
      target:
        type: judgment
        title: "Reverse IRB Protocol Approval"
        scope: "casehubio/clinical/irb-reversal"
      compensation: true
```

Bindings marked `compensation: true` don't execute during forward flow. No PlanItems are created for them. They exist only as targets for compensation — activated on demand when the coordinator needs them.

## Where this goes

The foundation is in casehub-work: `CompensationStatus` enum, entity fields, Flyway migration, `compensate()` and `markCompensated()` service methods, a lifecycle observer that auto-marks originals when the compensating WorkItem completes, and a REST endpoint. The ledger gets a `CompensationSupplement` — a new supplement type alongside Compliance and Provenance, carrying the original entry ID, compensation reason, and regulatory basis. The engine gets the saga coordinator, CaseStatus states, and YAML schema. The bridge connects them. Qhorus notifies agents. Connectors notify the outside world.

Eight child issues, five repos, clear dependency graph. The ledger and the engine prerequisite can run in parallel. The bridge comes after the coordinator. Qhorus and connectors come last.

I'll update this entry as the implementation progresses across sessions.
