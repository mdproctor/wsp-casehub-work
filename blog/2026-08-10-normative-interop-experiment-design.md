---
title: "Designing the Normative Layer Experiment"
date: 2026-08-10
type: diary
status: draft
tags: [normative-layer, experiment-design, multi-agent-coordination, speech-acts]
entry_type: note
subtype: diary
projects: [casehubio/work]
---

# Designing the Normative Layer Experiment

The engine has grown substantially since issue #189 was written. Compound PlanItems, DAG execution, lifecycle scopes, composable routing signals, HTN decomposition — none of this existed when the experiment was first conceived as "five LangChain4j agents vs five agents with Qhorus speech acts." The experiment needed revising, and the revision turned out to be the interesting part.

## The experiment tests enforcement, not vocabulary

The original framing positioned Variant A (no normative layer) as agents coordinating via free-form natural language, and Variant B as agents using typed speech act tools. A review caught the flaw: if Variant A agents have no structured vocabulary, we're testing whether having the right words helps — not whether *enforcing* the words helps. Those are different questions.

The fix: Variant A agents get the same vocabulary tools as Variant B. `report_failure`, `report_decline`, `report_completion` — mirrors of every exercised speech act type. The difference is that nothing enforces them. An agent can call `report_completion` after calling `report_failure` and nobody stops it. No commitment store, no state machine, no obligation ledger. Same words, no governance.

This makes the comparison tighter and the result more interesting. If Variant A achieves high consistency despite lacking enforcement, that weakens the case for the normative layer's overhead. The experiment is designed to detect that outcome honestly.

## A correction that sharpened the design

I had the Variant A agents built with LangChain4j's `@AiService` and Variant B with CaseHub's `Agent.builder()`. Mark pointed out the obvious: both should use `ChatModelProvider`. The experiment tests the normative layer, not LLM frameworks. CaseHub already integrates LangChain4j — framing Variant A as "LangChain4j standalone" introduces a confound that has nothing to do with coordination semantics.

Both variants now use identical agent construction: `Agent.builder().systemPrompt(role).model(chatModel).build()`. Same `ChatModelProvider`, same model, same prompt. The only difference is what wraps their execution. In Variant A, a deterministic `FreeFormIncidentOrchestrator` routes based on the agent's JSON response. In Variant B, the CaseHub engine routes via compound PlanItems, binding triggers, outcome policies, and the Qhorus commitment lifecycle.

## What the full governance stack actually exercises

The revised Variant B is not just "typed messages." It exercises:

- **Four-phase compound hierarchy** — Triage, Remediation, Early Notification, Confirmed Communication. Each with `All` completion semantics. Recovery gates on Rollback reaching FAULTED via conditional entry conditions.
- **Goal-based completion** — `SUCCESS = AllOf(records_recovered, users_notified)`. `FAILURE = AnyOf(sla_breach_unrecovered)`. The engine evaluates JQ conditions against the case context and fires `GoalReachedEvent` when satisfied.
- **OutcomePolicy** — Communication DECLINEs the initial notification (scope unknown). `onDecline: REROUTE` with `maxRerouteAttempts: 1` exhausts immediately. The DECLINE is the data point, not a failure to handle.
- **WorkerOutcome pipeline** — `WorkerResult.failed()` flows through `QhorusMessageSignalBridge` as `CommitmentState.FAILED`. `WorkerResult.declined()` flows as `CommitmentState.DECLINED`. The engine distinguishes them structurally — they're different commitment terminal states with different downstream behaviour.
- **CASE-scoped PERSISTENT COMPANION supervisor** — observational, not routing. The engine handles routing deterministically through its infrastructure.

The comparison becomes: "coordination by conversation" (Variant A, where the orchestrator interprets vocabulary tool calls) vs "coordination by governance" (Variant B, where the engine enforces obligation lifecycles, compound completion, and goal evaluation).

## The seeded state matters

Both variants query the same H2 database with a real corruption scenario: 1000 user records with corrupted preferences from a flawed migration, a daily backup with clean copies, an irreversible data change masked by a reversible schema change. The agents reason about actual state. When the rollback agent calls `attemptSchemaRollback`, it succeeds. When it calls `checkDataIntegrity`, it fails. The agent discovers the data corruption is irreversible — it doesn't read a script saying so.

This is what makes the FAILURE vs DECLINE distinction load-bearing. The rollback agent tried and failed. The communication agent didn't try — it can't act without scope. Both "didn't complete their work." Without governance, a post-incident reviewer reading the transcript sees two agents that produced no output. With governance, the commitment ledger shows FAILED on one and DECLINED on the other. Different words, same outcome, completely different meaning.

## What's next

Tasks 1-3 are done — the shared scenario infrastructure, the ungoverned orchestrator (Variant A), and the CaseHub case definition (Variant B) are all implemented and tested. The remaining work is the measurement framework (three-layer metrics: domain-neutral outcomes, structural verification, cost metrics) and the test harness (a demo test mirroring the existing normative benchmark's narrated output, plus the full 20-run matrix with Fisher's exact test for statistical comparison).

The Zone mapping from the existing normative benchmark applies directly but with a subtlety: Variant A is not Zone 1 (free-form text). It's Zone 2 (typed vocabulary without enforcement). The question isn't whether structure helps — the existing benchmark proved that. The question is whether enforcement adds auditability beyond what vocabulary alone provides.
