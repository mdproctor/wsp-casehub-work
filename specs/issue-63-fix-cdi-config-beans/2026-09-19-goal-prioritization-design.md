# Goal Prioritization from Unmet Needs

**Issue:** casehubio/examples#69
**Branch:** issue-63-fix-cdi-config-beans
**Date:** 2026-09-19

## Problem

Goal revision currently evaluates goals based on static priority (PRIMARY/SECONDARY) and outcome metrics (success/failure counts). A goal that repeatedly succeeds gets refined; one that repeatedly fails gets abandoned. But nothing connects goal urgency to the character's unmet needs. Penelope's "solve-puzzles" goal stays at the same priority whether her Understanding need is critically neglected or fully satisfied.

The needs pyramid (#68) computes per-tier satisfaction levels and persists them as MindMap nodes. This issue wires those satisfaction levels into `ManorGoalRevisionStrategy` so the LLM can reprioritize goals based on which needs are most neglected.

## Architecture

```
MindMapStore                    ManorGoalRevisionStrategy
  ┌──────────────┐               ┌──────────────────┐
  │ need-satisfaction │ ──read──→ │ buildPrompt()     │
  │ nodes (from #68) │           │  + satisfaction   │
  └──────────────┘               │  levels (numeric) │
                                 │                   │
GoalRevisionContext              │ AgentProvider      │
  ┌──────────────┐               │  (LLM call)       │
  │ goals + counts│ ──read──→    │                   │
  └──────────────┘               │ parseResponse()   │
                                 │  + REPRIORITIZE   │
                                 └────────┬─────────┘
                                          │
                                 GoalRevisionProposal
                                   (includes REPRIORITIZE)
                                          │
                                 ManorGoalEvaluator
                                   handles REPRIORITIZE
                                   → updates goal priority
```

### Data Presentation (D43, GE-20260919-3f610d)

Satisfaction levels are presented as **raw numeric values** in the LLM prompt, not qualitative prose. ManorGoalRevisionStrategy is an LLM-as-reasoner consumer — precision matters for priority calibration. The qualitative rendering (NeedsPyramidPromptSection) serves the LLM-as-character consumer; the goal revision strategy serves the analytical consumer.

## Components

### 1. GoalRevisionAction.REPRIORITIZE (engine-api)

**Location:** `casehub-engine-api`, `io.casehub.api.spi.routing.GoalRevisionAction`

Add `REPRIORITIZE` to the existing enum: `REVISE, ABANDON, COMPLETE, REPRIORITIZE`.

REVISE changes what a goal means. REPRIORITIZE changes how urgent it is. These are semantically distinct — the LLM should be able to recommend either independently.

### 2. GoalRevisionProposal.RevisedGoal extension (engine-api)

**Location:** `casehub-engine-api`, `io.casehub.api.spi.routing.GoalRevisionProposal`

Add optional `GoalPriority newPriority` field to the `RevisedGoal` record. Used with `REPRIORITIZE` action. `null` for other actions.

GoalPriority is an enum in eidos-api (PRIMARY, SECONDARY). The LLM outputs the enum name as a string; `parseResponse()` converts via `GoalPriority.valueOf()`.

### 3. ManorGoalRevisionStrategy enhancement (wacky-manor)

**Location:** `wacky-manor`, `io.casehub.examples.manor.agent.ManorGoalRevisionStrategy`

**Constructor change:** Add `MindMapStore mindMapStore` parameter (D44). CDI-injected alongside existing `AgentProvider`.

**buildPrompt() change:** After formatting goals with success/failure counts, append a satisfaction section:

```
Need satisfaction levels:
  SAFETY=0.62, TASKS=0.28, SOCIAL=0.71, SELF_EXPRESSION=0.85, UNDERSTANDING=0.35

Consider these satisfaction levels when evaluating goals. Goals addressing
neglected needs (low satisfaction) should be elevated in priority. Goals
addressing already-satisfied needs may be deprioritized.
```

Satisfaction is read from MindMapStore by querying for `cognitiveKind: "need-satisfaction"` nodes matching the agent-id from `GoalRevisionContext.agentId()`.

**SYSTEM_PROMPT change:** Add REPRIORITIZE to the action list:

```
- REPRIORITIZE: change goal priority to address unmet needs
  (provide newPriority: "PRIMARY" or "SECONDARY")
```

**parseResponse() change:** Handle `newPriority` field in JSON output:

```java
GoalPriority priority = null;
if (node.has("newPriority") && !node.get("newPriority").isNull()) {
    priority = GoalPriority.valueOf(node.get("newPriority").asText());
}
```

**Graceful degradation:** If no need-satisfaction nodes exist (character has no drives, or #68 not enabled), the prompt omits the satisfaction section entirely. Goal revision continues with the existing success/failure-based logic.

### 4. ManorGoalEvaluator REPRIORITIZE handling (wacky-manor)

**Location:** `wacky-manor`, `io.casehub.examples.manor.agent.ManorGoalEvaluator`

Add `REPRIORITIZE` case to the switch in `evaluate()`:

```java
case REPRIORITIZE -> {
    if (revision.newPriority() != null) {
        for (int i = 0; i < finalGoals.size(); i++) {
            if (finalGoals.get(i).name().equals(revision.goalName())) {
                finalGoals.set(i, finalGoals.get(i).toBuilder()
                    .priority(revision.newPriority()).build());
                changed = true;
                break;
            }
        }
    }
}
```

## LLM Prompt Design

**System prompt (updated):**
```
You are a goal effectiveness analyst for an autonomous agent. Given the
agent's goals, their performance metrics, and their current need
satisfaction levels, evaluate each goal and recommend an action:
- REVISE: refine the goal description (provide revisedDescription)
- ABANDON: drop the goal — unachievable or no longer relevant
- COMPLETE: the goal has been achieved
- REPRIORITIZE: change priority to address unmet needs
  (provide newPriority: "PRIMARY" or "SECONDARY")
Only act on goals with clear signals. If a goal is fine as-is, omit it.
Return ONLY a JSON object: {"revisions": [{"goalName": "...",
"action": "REVISE"|"ABANDON"|"COMPLETE"|"REPRIORITIZE",
"revisedDescription": "..."|null, "newPriority": "PRIMARY"|"SECONDARY"|null,
"revisionReason": "..."}], "rationale": "..."}
```

**User prompt (example):**
```
Agent: penelope-pitstop

Goals:
- find-diamond: Find the legendary Doily Diamond (priority: PRIMARY, success: 2, failure: 0, rate: 100%)
- solve-puzzles: Solve the mansion's puzzles (priority: SECONDARY, success: 5, failure: 1, rate: 83%)

Need satisfaction levels:
  SAFETY=0.62, TASKS=0.28, SOCIAL=0.71, SELF_EXPRESSION=0.85, UNDERSTANDING=0.35

Respond with JSON only.
```

## Cross-Repo Scope

| Repo | Changes | Issue |
|------|---------|-------|
| **engine** | `GoalRevisionAction.REPRIORITIZE`, `RevisedGoal.newPriority` | Engine PR needed |
| **examples/wacky-manor** | `ManorGoalRevisionStrategy` enhancement, `ManorGoalEvaluator` REPRIORITIZE case | casehubio/examples#69 |

**Note:** Engine repo is not in this slot. The engine-api changes require cloning the engine repo, making the changes, installing the updated jar, and creating a separate PR.

## Empirical Validation

1. **Satisfaction in prompt:** Verify buildPrompt() includes satisfaction levels when need-satisfaction nodes exist. Verify omission when nodes are absent.

2. **REPRIORITIZE parsing:** LLM returns a REPRIORITIZE action with newPriority=PRIMARY. Verify parseResponse() produces a RevisedGoal with action=REPRIORITIZE and correct priority.

3. **Priority update:** ManorGoalEvaluator receives REPRIORITIZE. Verify the goal's priority changes from SECONDARY to PRIMARY.

4. **Graceful degradation:** No need-satisfaction nodes. Verify revision continues with existing logic (no errors, no satisfaction section in prompt).

5. **Cross-character divergence:** Same goals, different satisfaction levels. Verify different reprioritization recommendations.

## References

- `ManorGoalRevisionStrategy.java` — wacky-manor, enhanced in this issue
- `ManorGoalEvaluator.java` — wacky-manor, REPRIORITIZE case added
- `GoalRevisionAction` — engine-api jar, REPRIORITIZE added
- `GoalRevisionProposal.RevisedGoal` — engine-api jar, newPriority added
- `GoalRevisionContext` — engine-api jar, no changes
- `GoalPriority` — eidos-api jar (PRIMARY, SECONDARY)
- `PriorityAdjustment` — blocks-core, existing priority change record
- `NeedsPyramidPromptSection` — blocks-core, renders qualitative prose (#68)
- GE-20260919-3f610d — numeric values for LLM reasoning, qualitative for embodiment
- Decisions D43–D45 — specs/issue-63-fix-cdi-config-beans/decisions.md
- Issue #68 — Needs pyramid (upstream dependency)
- Issue #69 — Goal prioritization from unmet needs
