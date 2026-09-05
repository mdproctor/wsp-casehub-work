---
layout: post
title: "Three Ways to Undo — Teaching Compensation by Example"
date: 2026-09-05
entry_type: article
subtype: diary
projects: [casehubio/work]
tags: [saga, compensation, examples, api-design, teaching]
series: issue-238-saga-compensation
---

# Three Ways to Undo — Teaching Compensation by Example

The saga compensation model [landed last week](2026-09-02-mdp01-when-done-isnt-done.md) — `CompensationStatus`, the `compensate()` API, `CompensationLifecycleObserver` auto-completing the cycle, three guards to prevent impossible states. Twenty-six design decisions across five repos. The model works, the tests pass, and nobody knows how to use it.

That's the gap. The examples module has seventeen scenarios covering every other capability — ledger chains, queues, AI routing, exclusion policies, escalation — but zero for compensation. A developer adding compensation to their Quarkus app has the javadoc and nothing else. The javadoc tells you `compensate()` takes four arguments. It doesn't tell you that the compensating WorkItem is real work with its own lifecycle, that different actors typically handle the reversal, or that you'll hit a 409 if you use `candidateUsers` instead of `candidateGroups`.

I wanted examples that teach at three different levels.

## The first thing that goes wrong

The expense approval scenario is the starter. A finance analyst creates a $50K payment approval, a senior officer approves it, then internal audit discovers the invoice was for a cancelled project. A compliance officer — not the original approver — handles the reversal.

The point isn't the workflow. The point is showing that `compensate()` creates a *new WorkItem* with its own lifecycle. The compliance officer claims it, starts it, completes it with their own resolution. The original automatically transitions to COMPENSATED via `CompensationLifecycleObserver`. Two independent audit trails, one causal link.

Claude caught something I'd have missed in review: `candidateUsers` with a single user auto-assigns the WorkItem to ASSIGNED status, making a subsequent `claim()` call fail with a 409. The fix is `candidateGroups` when you want the claim step to be explicit. It's a one-character distinction in the builder that changes the entire lifecycle flow. The kind of thing you only discover by running it.

## The saga without an engine

The second scenario is the interesting one. A multi-step loan application — credit check, property valuation, final approval — all linked by `callerRef`. After all three complete, a regulatory audit invalidates the credit check. All three must be reversed in dependency order: approval first, then valuation, then credit check.

casehub-work runs standalone here. No engine driving the ordering. The application code handles it — and that's the teaching point. When you're using casehub-work without the engine, *you* are the saga coordinator. The example shows how: compensate in reverse, observe the intermediate COMPENSATING state between steps (split transactions via `QuarkusTransaction.requiringNew()` make this visible), query by `callerRef` to verify the final state.

The split-transaction pattern matters. In the expense scenario, everything resolves in one `@Transactional` call — the compensating WorkItem is created, claimed, completed, and the original auto-transitions to COMPENSATED before the method returns. Clean, simple, and completely unrealistic for production. In the loan scenario, each compensation step commits independently. The developer sees COMPENSATING as a real intermediate state that persists between requests — the state their UI code needs to handle, their queue filters need to account for, their polling logic needs to wait on.

## When it says no

The third scenario exercises the three guards. Cannot compensate a non-COMPLETED WorkItem — you can't reverse effects that haven't been produced. Cannot compensate twice — compensation is a pair, not a chain. Cannot compensate a compensating WorkItem — meta-compensation is a new forward operation, not recursion.

These aren't error handling. They're the compensation model's invariants, and a developer who understands them understands the model. The scenario also puts the compensating WorkItem through suspend and resume — a doctor goes on leave mid-reversal, another doctor picks it up. While the compensating WorkItem is suspended, the original stays COMPENSATING. The system reflects reality.

## The dependency direction lesson

I'd initially planned engine-integrated examples in the same branch — saga orchestration, the notification pipeline, the full BPMN-style compensation flow. Claude pointed out the obvious: engine depends on work, not the reverse. Putting engine examples in the work repo creates a circular dependency. They belong in the engine repo, built there, tested there.

The engine examples are now separate issues in `casehubio/engine`. The work examples teach the API; the engine examples will teach the orchestration. Same dependency direction as the code.
