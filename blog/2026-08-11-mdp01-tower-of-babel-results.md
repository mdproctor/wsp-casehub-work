---
layout: post
title: "Three Ways to Coordinate an Incident Response"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [multi-agent-coordination, casehub-example, incident-response, speech-acts]
series: issue-189-normative-interop-experiment
---

# Three Ways to Coordinate an Incident Response

The [design entry](/blog/2026-08-10-normative-interop-experiment-design) set up an experiment. What we actually built is a tutorial — three orchestration patterns running the same scenario with deterministic workers, no LLMs in the loop. The experiment that would test the normative layer thesis (do LLMs produce incompatible semantics under pressure?) requires LLM-backed agents producing non-deterministic responses across multiple runs. We're not there yet.

What we do have is a worked example showing the structural differences between three approaches. That's worth documenting on its own terms.

## The scenario

A migration script deployed to production corrupts 1000 user records. Five agents coordinate the response: detection, diagnosis, rollback, recovery, and communication. The rollback succeeds at the schema level but fails at the data level — the UPDATE statement destroyed information that dropping the column can't recover. The communication agent can't notify users until the scope of affected records is known, so it declines the first request. Recovery restores from backup. Communication completes after recovery verifies.

The scenario is designed so that FAILED and DECLINED both look like "the agent didn't complete its task" from the outside. How each variant handles that distinction is the interesting part.

## Variant A — imperative orchestrator

A switch statement interprets string-typed action codes. The orchestrator knows the scenario by name:

```java
private String handleFailure(String agentId) {
  failedAgents.add(agentId);
  if ("rollback".equals(agentId)) {
    return "activate_recovery";
  }
  return "record_failure";
}
```

Three agent names hardcoded into routing logic. Three sets of mutable state manually tracking what happened. Replace the agents and you rewrite the routing.

## Variant B — CaseHub engine

Declarative bindings, typed outcomes, goal-based completion. The rollback binding declares an outcome policy; the recovery binding triggers on diagnostic state:

```yaml
- name: recovery
  capability: data-recovery
  on:
    contextChange:
      filter: '(._diagnostics.rollback.status == "FAILED"
              or ._diagnostics.rollback.status == "REROUTES_EXHAUSTED")
              and .recovery == null'
```

The worker returns `WorkerResult.failed()` — a sealed type, not a string. The engine routes FAILED and DECLINED through different outcome policies. Bindings reference capabilities, not workers.

The integration test boots the full engine (`@QuarkusTest` with CDI, Vert.x, Quartz, in-memory persistence) and drives the case from `incident_started = true` to COMPLETED.

## Variant C — Qhorus channel protocol

Workers are external — no in-process functions. The CaseHub definition is four lines:

```java
public class IncidentCaseHubC extends YamlCaseHub {
  public IncidentCaseHubC() {
    super("casehub/incident-case-c.yaml");
  }
}
```

A `ChannelDrivenIncidentSimulator` implements `CaseChannelProvider`, intercepts COMMAND messages posted by the engine, and responds with typed speech acts (DONE, DECLINE, FAILURE). The `QhorusMessageSignalBridge` routes these through the engine's outcome pipeline — the same handler that processes in-process WorkerResult outcomes.

Two engine changes made this possible: `WorkerScheduleEventHandler` now posts the COMMAND before calling `submit()` (catching `ProvisioningException` for external-only workers), and the bridge now routes DONE messages through `handleWorkerOutcome()` so external workers get the same output application and goal evaluation as in-process workers.

## What the binding filter bugs taught us

Three bugs surfaced in Variant B when the engine ran the scenario end-to-end:

**Infinite re-firing.** `.rollback == null` stays true forever when the worker fails — the engine writes to `_diagnostics.rollback`, not `.rollback`. The binding creates new PlanItems on every context change. Fix: add a diagnostic guard.

**Wrong status string.** The recovery binding triggered on `"FAILED"`. The engine writes `"REROUTES_EXHAUSTED"`. These are different concepts — worker failure vs capability exhaustion.

**Deduplication collision.** Two notification bindings targeting the same worker and capability produced identical input hashes. Fix: include recovery state in the projection.

These bugs are real costs of declarative orchestration. Variant A can't have any of them because the orchestrator controls dispatch imperatively. The question is whether the decoupling and auditability that declarative bindings provide are worth the cost of thinking about what makes a binding *stop* firing, not just what makes it *start*.

## What we didn't build

The normative layer thesis — that LLMs produce incompatible semantics when coordinating without governance — requires LLM agents in the loop. Everything here uses deterministic workers. The scenario infrastructure (H2 database, tools, YAML definitions, channel simulator) is scaffolding for that experiment. The integration tests validate the engine wiring. Neither is evidence for or against the thesis.

The experiment that matters: run this scenario with Claude and GPT-4 as the agents. Give both the same tools and vocabulary. Measure whether they agree on which outcomes are FAILURE vs DECLINE across multiple runs. Measure whether the commitment lifecycle catches the disagreements. That's a future session.
