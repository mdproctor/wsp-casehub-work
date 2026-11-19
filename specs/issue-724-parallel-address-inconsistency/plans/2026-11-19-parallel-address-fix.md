# Parallel Address Inconsistency Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #724 — parallel lane address inconsistency
**Issue group:** #724 (related: #722, parent #720)

**Goal:** Align `StepPositionScanner` with `assignStepAddresses` so both produce identical addresses for unnamed parallel step children.

**Architecture:** Modify `scanNamedBranches` in the scanner to detect list items at `nameIndent` when `label === 'lane'` (unique to parallel). For each list item, call `scanBlock` with prefix `${address}.lane.${laneIndex}`, which produces `lane.N.0` addresses and handles nested control flow automatically.

**Tech Stack:** TypeScript, Vitest

## Global Constraints

- Address format for unnamed lanes: `${address}.lane.${index}.0` — matches step-click convention
- `label === 'lane'` guard ensures only parallel is affected; race/select/match unchanged
- `scanBlock` reuse for nested control-flow descent — no new scanning logic

---

## Batch 1: Scanner unnamed-lane support

### Task 1: Add unnamed parallel branch scanning to StepPositionScanner

**Files:**
- Modify: `packages/yaml-core/src/annotation/step-position-scanner.ts:170-213`
- Test: `packages/yaml-core/src/annotation/step-position-scanner.test.ts`

**Interfaces:**
- Consumes: existing `scanBlock`, `findNextSibling`, `scanNamedBranches` internals
- Produces: `StepLineInfo[]` entries with `address: '0.lane.N.0'` for unnamed parallel children

- [ ] **Step 1: Write failing test for unnamed parallel**

Add this test to the `nested parallel / swim-lane` describe block in `step-position-scanner.test.ts`:

```typescript
it('scans parallel steps with unnamed lanes', () => {
  const yaml = [
    '- parallel:',
    '    - action: step-1',
    '    - action: step-2',
  ].join('\n');
  const result = StepPositionScanner.scan(yaml);
  expect(result).toEqual([
    { line: 0, endLine: 3, address: '0' },
    { line: 1, endLine: 2, address: '0.lane.0.0' },
    { line: 2, endLine: 3, address: '0.lane.1.0' },
  ]);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/yaml-core/src/annotation/step-position-scanner.test.ts`
Expected: FAIL — the test expects 3 entries but gets only 1 (the `'0'` parallel step itself; no children because `scanNamedBranches` skips list items).

- [ ] **Step 3: Implement unnamed-lane handling in scanNamedBranches**

In `step-position-scanner.ts`, modify the `scanNamedBranches` function. Add a `laneIndex` counter before the while loop, and replace the list-item skip at line 200-201 with lane-aware handling:

Replace:
```typescript
    const stripped = line.trimStart();
    if (stripped.startsWith('- ')) { i++; continue; }
```

With:
```typescript
    const stripped = line.trimStart();
    if (stripped.startsWith('- ')) {
      if (label === 'lane') {
        const laneEnd = findNextSibling(lines, i, nameIndent, stepEnd);
        scanBlock(lines, i, laneEnd, nameIndent, `${address}.${label}.${laneIndex}`, results);
        laneIndex++;
        i = laneEnd;
        continue;
      }
      i++; continue;
    }
```

And add `let laneIndex = 0;` just before the `let i = scanStart;` line (around line 191).

- [ ] **Step 4: Run tests to verify pass**

Run: `yarn vitest run packages/yaml-core/src/annotation/step-position-scanner.test.ts`
Expected: ALL PASS — both the new unnamed test and all existing named-branch tests.

- [ ] **Step 5: Run step-click tests to verify cross-package consistency**

Run: `yarn vitest run packages/pages-step-viewer/src/step-click.test.ts`
Expected: ALL PASS — no changes to step-click, just confirming address formats still align.

- [ ] **Step 6: Commit**

```bash
git add packages/yaml-core/src/annotation/step-position-scanner.ts packages/yaml-core/src/annotation/step-position-scanner.test.ts
git commit -m "fix(#724): scan unnamed parallel lanes in StepPositionScanner

scanNamedBranches now detects list items when label is 'lane' and
assigns lane.N.0 addresses matching assignStepAddresses convention.

Refs #724"
```

## References

- [2026-11-19-parallel-address-fix-design.md] — design spec this plan implements
- [packages/yaml-core/src/annotation/step-position-scanner.ts:170-213] — scanNamedBranches function
- [packages/pages-step-viewer/src/step-click.ts:54-66] — parallel address assignment in step-click
- [packages/yaml-core/src/step/walker.ts:474-488] — parallel YAML resolution (array vs object)
- [packages/pages-step-viewer/src/step-click.test.ts:107-121] — unnamed parallel test reference
- [GitHub #724] — focal issue
- [GitHub #722] — related match/race address mismatch (separate fix)
- [GitHub #720] — parent split-view playbook editor feature
