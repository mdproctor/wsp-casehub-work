# Parallel Lane Address Inconsistency Fix

Refs #724, related to #722, parent #720

## Problem

`StepPositionScanner` and `assignStepAddresses` produce different addresses for parallel step children, breaking bidirectional click-sync in the split-view playbook editor.

**Scanner** (`scanNamedBranches`): Only handles named YAML keys (object-format parallel). Skips list items entirely — produces no child addresses for array-format parallel steps.

**step-click** (`assignStepAddresses`): Handles both formats:
- Named branches: `0.lane.agent.0` (via `branch.name`)
- Unnamed array: `0.lane.0.0` (via numeric index)

## YAML Formats

Named (object — both paths handle correctly):
```yaml
- parallel:
    agent:
      - action: agent-step
    reviewer:
      - action: review-step
```

Unnamed (array — scanner produces nothing):
```yaml
- parallel:
    - action: step-1
    - action: step-2
```

The walker (`walker.ts:474-488`) resolves array-format parallel into `ParallelStep.steps` (no `branches`). Object-format resolves into both `branches` (with `name` from YAML key) and `steps` (flattened).

## Fix

In `scanNamedBranches`, after the existing named-key scan finds no keys, detect list items at the expected indent and assign numeric lane addresses (`${address}.lane.${index}.0`), matching step-click's unnamed-branch convention.

The scanner already skips list items (line 201: `if (stripped.startsWith('- ')) { i++; continue; }`). For `parallel` specifically, those list items ARE the branches. The fix detects this case and scans them as individual lanes with numeric identifiers.

## Scope

- Modify `scanNamedBranches` in `step-position-scanner.ts` to handle list items for `parallel` label
- Add test case for unnamed parallel in `step-position-scanner.test.ts`
- Verify existing named-branch tests still pass

## References

- `packages/yaml-core/src/annotation/step-position-scanner.ts:170-213` — scanNamedBranches
- `packages/pages-step-viewer/src/step-click.ts:54-66` — parallel address assignment
- `packages/yaml-core/src/step/walker.ts:474-488` — parallel YAML resolution
- `packages/pages-step-viewer/src/step-click.test.ts:107-121` — unnamed parallel test
