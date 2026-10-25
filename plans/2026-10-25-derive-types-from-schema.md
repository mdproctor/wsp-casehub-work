# Derive Mosaic Types from Schema — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-mosaic#284 — Derive mosaic agency types from generated schema
**Issue group:** casehubio/casehub-mosaic#284

**Goal:** Replace mosaic's loose `{ type: string }` sub-spec types with
discriminated unions derived from agency's JSON Schema, and add a
staleness test that catches future drift.

**Architecture:** Copy agency's `pattern-spec-schema.json` into mosaic.
Upgrade hand-maintained TS types in `orchestration.ts` and pattern
category files to proper discriminated unions matching the schema's
`oneOf` variants. Add `schema-staleness.test.ts` that parses the schema
and validates TS variant lists against it.

**Tech Stack:** TypeScript, Vitest, JSON Schema (Draft 2020-12)

## Global Constraints

- All work is in `casehub-mosaic` repo at `/Users/mdproctor/claude/casehub/slots/219/casehub-mosaic`
- Build: `yarn build`, Test: `yarn test`, Typecheck: `yarn typecheck`
- Run mosaic-core tests only: `yarn workspace @casehubio/mosaic-core test`
- Schema discriminator values use camelCase (from Java), not kebab-case
- The schema is the source of truth for sub-spec variant names and required fields
- Runtime/audit types (ExecutionSnapshot, OrchestrationAuditEvent, etc.) are NOT changed

---

## Batch 1: Schema delivery + sub-spec type upgrades

### Task 1: Copy schema and upgrade sub-spec types in orchestration.ts

**Files:**
- Create: `packages/mosaic-core/src/types/patterns/pattern-spec-schema.json`
- Modify: `packages/mosaic-core/src/types/orchestration.ts`

**Interfaces:**
- Consumes: `pattern-spec-schema.json` from agency (`agentic-yaml/src/test/resources/schema/pattern-spec-schema.json`)
- Produces: `RoutingSpec`, `AggregationSpec`, `TerminationSpec`, `DecompositionSpec`, `ActivationSpec`, `AgentRefSpec`, `TaskNodeSpec`, `GovernanceSpec`, `AgreementSpec` as discriminated union types; upgraded `FailurePolicy` and `JudgmentSpec`

- [ ] **Step 1: Copy schema file**

Copy `pattern-spec-schema.json` from agency into mosaic:

```bash
cp /Users/mdproctor/claude/casehub/slots/219/agency/agentic-yaml/src/test/resources/schema/pattern-spec-schema.json /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic/packages/mosaic-core/src/types/patterns/pattern-spec-schema.json
```

- [ ] **Step 2: Write failing typecheck test**

Add a type-level test to `packages/mosaic-core/src/types/patterns/patterns.test.ts`
that constructs sub-spec values using the discriminated union variants.
This will fail because the current types are `{ type: string }`:

```typescript
it('RoutingSpec discriminates on type field', () => {
  const r: RoutingSpec = { type: 'first-match' };
  expect(r.type).toBe('first-match');
});

it('AggregationSpec discriminates on type field', () => {
  const a: AggregationSpec = { type: 'collect-all' };
  expect(a.type).toBe('collect-all');
});

it('TerminationSpec discriminates on type field', () => {
  const t: TerminationSpec = { type: 'goal-reached' };
  expect(t.type).toBe('goal-reached');
});

it('DecompositionSpec discriminates on type field', () => {
  const d: DecompositionSpec = { type: 'llm' };
  expect(d.type).toBe('llm');
});
```

Import `RoutingSpec`, `AggregationSpec`, `TerminationSpec`, `DecompositionSpec`
from `'../orchestration.js'` at the top of the test file.

- [ ] **Step 3: Run typecheck to verify it fails**

Run: `yarn workspace @casehubio/mosaic-core typecheck`
Expected: Errors — `RoutingSpec` and `DecompositionSpec` don't exist yet,
`AggregationSpec.type` is `string` not a literal union.

- [ ] **Step 4: Upgrade sub-spec types in orchestration.ts**

Replace the existing loose types and add new discriminated unions.
The full content of the type definitions to add/replace in
`packages/mosaic-core/src/types/orchestration.ts`:

**Replace `AggregationSpec`:**

```typescript
export type AggregationSpec =
  | { type: 'pass-through' }
  | { type: 'collect-all' }
  | { type: 'majority-vote' }
  | { type: 'auction' };
```

**Replace `TerminationSpec`:**

```typescript
export type TerminationSpec =
  | { type: 'max-iterations' }
  | { type: 'goal-reached' }
  | { type: 'judge-convergence' }
  | { type: 'all-agreed' }
  | { type: 'supervisor' }
  | { type: 'contested' }
  | { type: 'convergence' }
  | { type: 'single-pass' }
  | { type: 'agent-count' }
  | { type: 'accepted' }
  | { type: 'terminal-outcome' }
  | { type: 'deadline' };
```

**Add `RoutingSpec` (new type):**

```typescript
export type RoutingSpec =
  | { type: 'first-match'; guard?: string }
  | { type: 'round-robin' }
  | { type: 'sequential' }
  | { type: 'llm-selected' }
  | { type: 'select-all' };
```

**Add `DecompositionSpec` (new type):**

```typescript
export type DecompositionSpec =
  | { type: 'identity' }
  | { type: 'static' }
  | { type: 'llm' }
  | { type: 'hybrid' }
  | { type: 'heuristic' }
  | { type: 'goap' }
  | { type: 'capability-dependency' }
  | { type: 'forward-reasoning' };
```

**Add `ActivationSpec` (new type):**

```typescript
export type ActivationSpec =
  | { type: 'on-dispatch' }
  | { type: 'max-iterations' };
```

**Add `TaskNodeSpec` (new type):**

```typescript
export type TaskNodeSpec =
  | { type: 'primitive'; name?: string; description?: string; executor?: string }
  | { type: 'compound'; name?: string; subtasks?: TaskNodeSpec[] };
```

**Add `AgreementSpec` (new type):**

```typescript
export type AgreementSpec =
  | { type: 'unanimous' }
  | { type: 'majority' }
  | { type: 'threshold' };
```

**Add `GovernanceSpec` (new type):**

```typescript
export interface GovernanceSpec {
  oversight?: {
    classifier?: string;
    reversible?: boolean;
    candidateGroups?: string[];
  };
  trustRouting?: {
    threshold?: number;
    minimumObservations?: number;
    borderlineMargin?: number;
    blendFactor?: number;
  };
  cbrRouting?: {
    successWeight?: number;
    gateExpiredWeight?: number;
    gateRejectedWeight?: number;
    failureWeight?: number;
  };
  attestation?: {
    observer?: string;
    capabilityTag?: string;
  };
}
```

**Upgrade `AgentRef` to `AgentRefSpec`:**

Keep the existing `AgentRef` interface for backward compatibility (runtime
types use it). Add a new `AgentRefSpec` type for YAML pattern definitions:

```typescript
export interface BaseAgentRefSpec {
  readonly name?: string;
  readonly description?: string;
  readonly capabilities?: string[];
  readonly model?: string;
}

export type AgentRefSpec =
  | (BaseAgentRefSpec & { type: 'worker' })
  | (BaseAgentRefSpec & { type: 'channel'; channelId?: string })
  | (BaseAgentRefSpec & { type: 'human' })
  | (BaseAgentRefSpec & { type: 'external' })
  | (BaseAgentRefSpec & { type: 'composed'; pattern?: PatternDefinitionRef });

export interface PatternDefinitionRef {
  type: string;
  [key: string]: unknown;
}
```

**Upgrade `FailurePolicy` to match schema:**

```typescript
export interface FailurePolicySpec {
  onRoutingFailure?: 'FAIL' | 'RETRY_BROADER' | 'ESCALATE';
  onDeadlock?: 'FAIL' | 'ESCALATE' | 'RETRY_DIFFERENT';
  agentRetry?: {
    maxRetries?: number;
    backoffStrategy?: 'FIXED' | 'EXPONENTIAL' | 'EXPONENTIAL_WITH_JITTER';
    backoff?: string;
    onExhausted?: 'FAIL' | 'ESCALATE' | 'SKIP';
  };
  replanPolicy?: {
    maxReplans?: number;
    fallbackAction?: 'FAIL' | 'RETRY_BROADER' | 'ESCALATE';
  };
}
```

Keep the existing `FailurePolicy` for runtime types. Add `FailurePolicySpec`
for YAML definitions.

**Upgrade `JudgmentSpec`:**

Replace the inline types in `patterns/deliberation.ts` with schema-aligned
versions. Add to `orchestration.ts`:

```typescript
export type JudgmentTriggerSpec =
  | { type: 'alwaysYield' }
  | { type: 'neverYield' }
  | { type: 'iterationBased'; every?: number }
  | { type: 'confidenceThreshold'; threshold?: number; extractor?: string };

export type JudgmentCallerSpec =
  | { type: 'single'; callerName?: string }
  | { type: 'fanOut'; callerNames?: string[] }
  | { type: 'escalationChain'; callerNames?: string[] };

export interface JudgmentSpecDef {
  trigger?: JudgmentTriggerSpec;
  caller?: JudgmentCallerSpec;
  agreement?: AgreementSpec;
}
```

**Export all new types** from `orchestration.ts`.

- [ ] **Step 5: Run typecheck to verify it passes**

Run: `yarn workspace @casehubio/mosaic-core typecheck`
Expected: PASS — new types compile.

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn workspace @casehubio/mosaic-core test`
Expected: PASS — existing tests still work (old types kept for backward
compat), new type-level tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic add packages/mosaic-core/src/types/patterns/pattern-spec-schema.json packages/mosaic-core/src/types/orchestration.ts packages/mosaic-core/src/types/patterns/patterns.test.ts
git -C /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic commit -m "feat: add schema and discriminated union sub-spec types Refs #284"
```

---

### Task 2: Update pattern category interfaces to use new sub-spec types

**Files:**
- Modify: `packages/mosaic-core/src/types/patterns/orchestration.ts`
- Modify: `packages/mosaic-core/src/types/patterns/deliberation.ts`
- Modify: `packages/mosaic-core/src/types/patterns/base.ts`
- Modify: `packages/mosaic-core/src/types/patterns/index.ts`
- Modify: `packages/mosaic-core/src/types/patterns/patterns.test.ts`

**Interfaces:**
- Consumes: `RoutingSpec`, `DecompositionSpec`, `ActivationSpec`, `GovernanceSpec`,
  `AgentRefSpec`, `FailurePolicySpec`, `JudgmentSpecDef`, `AgreementSpec` from Task 1
- Produces: Updated pattern category interfaces using schema-aligned types

- [ ] **Step 1: Write failing test for updated category interfaces**

Update `patterns.test.ts` to use the new sub-spec types in category
construction. The existing deliberation test uses `confidence-threshold`
(kebab-case) which must change to `confidenceThreshold` (camelCase):

```typescript
it('DeliberationPatternCategory has judgment fields', () => {
  const p: DeliberationPatternCategory = {
    category: 'deliberation',
    type: 'voting',
    name: 'risk-vote',
    maxRounds: 3,
    judgment: {
      trigger: { type: 'confidenceThreshold', threshold: 0.8 },
      caller: { type: 'fanOut' },
    },
  };
  expect(p.maxRounds).toBe(3);
  expect(p.judgment!.trigger!.type).toBe('confidenceThreshold');
});

it('OrchestrationPatternCategory uses typed routing and decomposition', () => {
  const p: OrchestrationPatternCategory = {
    category: 'orchestration',
    type: 'supervisor',
    name: 'risk-supervisor',
    routing: { type: 'first-match' },
    decomposition: { type: 'static' },
    aggregation: { type: 'collect-all' },
    termination: [{ type: 'max-iterations' }],
  };
  expect(p.routing!.type).toBe('first-match');
});
```

- [ ] **Step 2: Run typecheck to verify failures**

Run: `yarn workspace @casehubio/mosaic-core typecheck`
Expected: Errors — `OrchestrationPatternCategory` doesn't have `routing`
as `RoutingSpec` type yet.

- [ ] **Step 3: Update OrchestrationPatternCategory**

In `packages/mosaic-core/src/types/patterns/orchestration.ts`:

```typescript
import type {
  AgentRefSpec, FailurePolicySpec, AggregationSpec, TerminationSpec,
  RoutingSpec, DecompositionSpec, ActivationSpec, GovernanceSpec,
  JudgmentSpecDef,
} from '../orchestration.js';

export type OrchestrationPatternType =
  'supervisor' | 'choreography' | 'saga' | 'decomposition';

export interface OrchestrationPatternCategory {
  category: 'orchestration';
  type: OrchestrationPatternType;
  name?: string;
  description?: string;
  task?: string;
  agents?: AgentRefSpec[];
  routing?: RoutingSpec;
  decomposition?: DecompositionSpec;
  activation?: ActivationSpec;
  aggregation?: AggregationSpec;
  termination?: TerminationSpec[];
  judgment?: JudgmentSpecDef;
  governance?: GovernanceSpec;
  failurePolicy?: FailurePolicySpec;
}
```

- [ ] **Step 4: Update DeliberationPatternCategory**

In `packages/mosaic-core/src/types/patterns/deliberation.ts`, remove the
inline `JudgmentTrigger`, `JudgmentCaller`, and `JudgmentSpec` types.
Import `JudgmentSpecDef` from `../orchestration.js`:

```typescript
import type {
  AgentRefSpec, FailurePolicySpec, AggregationSpec, TerminationSpec,
  RoutingSpec, DecompositionSpec, ActivationSpec, GovernanceSpec,
  JudgmentSpecDef,
} from '../orchestration.js';

export type DeliberationPatternType =
  'voting' | 'debate' | 'negotiation' | 'coalition' | 'normative';

export interface DeliberationPatternCategory {
  category: 'deliberation';
  type: DeliberationPatternType;
  name?: string;
  description?: string;
  task?: string;
  agents?: AgentRefSpec[];
  judgment?: JudgmentSpecDef;
  maxRounds?: number;
  judge?: AgentRefSpec;
  routing?: RoutingSpec;
  decomposition?: DecompositionSpec;
  activation?: ActivationSpec;
  aggregation?: AggregationSpec;
  termination?: TerminationSpec[];
  governance?: GovernanceSpec;
  failurePolicy?: FailurePolicySpec;
}
```

- [ ] **Step 5: Update BasePatternCategory**

In `packages/mosaic-core/src/types/patterns/base.ts`, add optional
sub-spec fields shared by the simpler categories:

```typescript
import type { AgencyCategoryName } from './categories.js';
import type { AgentRefSpec, FailurePolicySpec } from '../orchestration.js';

export interface BasePatternCategory {
  category: AgencyCategoryName;
  type: string;
  name?: string;
  description?: string;
  task?: string;
  agents?: AgentRefSpec[];
  failurePolicy?: FailurePolicySpec;
}
```

Update the specific category interfaces to use the correct agent/failure
types. The type unions (`PerceptionPatternCategory`, etc.) stay the same —
they just inherit the updated base types.

- [ ] **Step 6: Update index.ts re-exports**

In `packages/mosaic-core/src/types/patterns/index.ts`, ensure all new
types from `orchestration.ts` are re-exported. The `export * from './orchestration.js'`
already covers types exported from `patterns/orchestration.ts`. Verify
that `../orchestration.ts` exports are also accessible. Add explicit
re-export if needed:

```typescript
export type {
  RoutingSpec, DecompositionSpec, ActivationSpec, GovernanceSpec,
  AgentRefSpec, FailurePolicySpec, JudgmentSpecDef, JudgmentTriggerSpec,
  JudgmentCallerSpec, AgreementSpec, TaskNodeSpec,
} from '../orchestration.js';
```

- [ ] **Step 7: Fix any remaining compilation errors**

Search for any other files that import the old types (`JudgmentTrigger`,
`JudgmentCaller`, `JudgmentSpec`, or use `AgentRef[]` in pattern contexts).
Update imports. Use `ide_find_references` or grep:

```bash
grep -rn "JudgmentTrigger\|JudgmentCaller\|JudgmentSpec" /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic/packages/ --include='*.ts' --exclude-dir=node_modules
```

- [ ] **Step 8: Run typecheck and tests**

Run: `yarn workspace @casehubio/mosaic-core typecheck`
Expected: PASS

Run: `yarn workspace @casehubio/mosaic-core test`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic add packages/mosaic-core/src/types/
git -C /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic commit -m "feat: update pattern categories to use schema-aligned sub-spec types Refs #284"
```

---

## Batch 2: Staleness test + cleanup

### Task 3: Write schema-staleness.test.ts

**Files:**
- Create: `packages/mosaic-core/src/types/patterns/schema-staleness.test.ts`

**Interfaces:**
- Consumes: `pattern-spec-schema.json` (committed in Task 1)
- Produces: Staleness test that CI runs on every commit

- [ ] **Step 1: Write the staleness test**

Create `packages/mosaic-core/src/types/patterns/schema-staleness.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import schema from './pattern-spec-schema.json';

type SchemaObject = {
  oneOf?: Array<{
    properties?: Record<string, { const?: string }>;
    required?: string[];
    $ref?: string;
  }>;
  properties?: Record<string, unknown>;
};

const defs: Record<string, SchemaObject> = (schema as Record<string, unknown>)['$defs'] as Record<string, SchemaObject>;

function getSchemaVariants(defName: string): string[] {
  const def = defs[defName];
  if (!def?.oneOf) return [];
  return def.oneOf
    .map(v => v.properties?.['type']?.const)
    .filter((v): v is string => v !== undefined)
    .sort();
}

function getSchemaTopologies(): string[] {
  const root = schema as SchemaObject;
  if (!root.oneOf) return [];
  return root.oneOf
    .map(v => v.properties?.['type']?.const)
    .filter((v): v is string => v !== undefined)
    .sort();
}

// -- TS variant lists: must match schema exactly --

const TS_TOPOLOGIES = [
  'conditional', 'debate', 'htn', 'loop',
  'parallel', 'sequence', 'supervisor', 'voting',
].sort();

const TS_ROUTING_VARIANTS = [
  'first-match', 'llm-selected', 'round-robin', 'select-all', 'sequential',
].sort();

const TS_AGGREGATION_VARIANTS = [
  'auction', 'collect-all', 'majority-vote', 'pass-through',
].sort();

const TS_TERMINATION_VARIANTS = [
  'accepted', 'agent-count', 'all-agreed', 'contested', 'convergence',
  'deadline', 'goal-reached', 'judge-convergence', 'max-iterations',
  'single-pass', 'supervisor', 'terminal-outcome',
].sort();

const TS_DECOMPOSITION_VARIANTS = [
  'capability-dependency', 'forward-reasoning', 'goap', 'heuristic',
  'hybrid', 'identity', 'llm', 'static',
].sort();

const TS_ACTIVATION_VARIANTS = [
  'max-iterations', 'on-dispatch',
].sort();

const TS_AGENT_REF_VARIANTS = [
  'channel', 'composed', 'external', 'human', 'worker',
].sort();

const TS_TASK_NODE_VARIANTS = [
  'compound', 'primitive',
].sort();

const TS_AGREEMENT_VARIANTS = [
  'majority', 'threshold', 'unanimous',
].sort();

const TS_JUDGMENT_TRIGGER_VARIANTS = [
  'alwaysYield', 'confidenceThreshold', 'iterationBased', 'neverYield',
].sort();

const TS_JUDGMENT_CALLER_VARIANTS = [
  'escalationChain', 'fanOut', 'single',
].sort();

describe('schema staleness — variant names', () => {
  it('pattern topologies match schema', () => {
    expect(TS_TOPOLOGIES).toEqual(getSchemaTopologies());
  });

  it('RoutingSpec variants match schema', () => {
    expect(TS_ROUTING_VARIANTS).toEqual(getSchemaVariants('RoutingSpec'));
  });

  it('AggregationSpec variants match schema', () => {
    expect(TS_AGGREGATION_VARIANTS).toEqual(getSchemaVariants('AggregationSpec'));
  });

  it('TerminationSpec variants match schema', () => {
    expect(TS_TERMINATION_VARIANTS).toEqual(getSchemaVariants('TerminationSpec'));
  });

  it('DecompositionSpec variants match schema', () => {
    expect(TS_DECOMPOSITION_VARIANTS).toEqual(getSchemaVariants('DecompositionSpec'));
  });

  it('ActivationSpec variants match schema', () => {
    expect(TS_ACTIVATION_VARIANTS).toEqual(getSchemaVariants('ActivationSpec'));
  });

  it('AgentRefSpec variants match schema', () => {
    expect(TS_AGENT_REF_VARIANTS).toEqual(getSchemaVariants('AgentRefSpec'));
  });

  it('TaskNodeSpec variants match schema', () => {
    expect(TS_TASK_NODE_VARIANTS).toEqual(getSchemaVariants('TaskNodeSpec'));
  });

  it('AgreementSpec variants match schema', () => {
    expect(TS_AGREEMENT_VARIANTS).toEqual(getSchemaVariants('AgreementSpec'));
  });

  it('JudgmentSpec trigger variants match schema', () => {
    const triggerDef = defs['JudgmentSpec']?.properties?.['trigger'] as SchemaObject | undefined;
    if (!triggerDef?.oneOf) { expect.fail('JudgmentSpec.trigger missing oneOf'); return; }
    const schemaVariants = triggerDef.oneOf
      .map(v => v.properties?.['type']?.const)
      .filter((v): v is string => v !== undefined)
      .sort();
    expect(TS_JUDGMENT_TRIGGER_VARIANTS).toEqual(schemaVariants);
  });

  it('JudgmentSpec caller variants match schema', () => {
    const callerDef = defs['JudgmentSpec']?.properties?.['caller'] as SchemaObject | undefined;
    if (!callerDef?.oneOf) { expect.fail('JudgmentSpec.caller missing oneOf'); return; }
    const schemaVariants = callerDef.oneOf
      .map(v => v.properties?.['type']?.const)
      .filter((v): v is string => v !== undefined)
      .sort();
    expect(TS_JUDGMENT_CALLER_VARIANTS).toEqual(schemaVariants);
  });
});

describe('schema staleness — required fields', () => {
  function getSchemaRequiredFields(defName: string, variantType: string): string[] {
    const def = defs[defName];
    if (!def?.oneOf) return [];
    const variant = def.oneOf.find(v => v.properties?.['type']?.const === variantType);
    if (!variant?.required) return [];
    return variant.required.filter(f => f !== 'type').sort();
  }

  it('all sub-spec variants have type as only required field', () => {
    const specs = [
      'RoutingSpec', 'AggregationSpec', 'TerminationSpec',
      'DecompositionSpec', 'ActivationSpec', 'AgentRefSpec',
      'TaskNodeSpec', 'AgreementSpec',
    ];
    for (const spec of specs) {
      const variants = getSchemaVariants(spec);
      for (const variant of variants) {
        const extraRequired = getSchemaRequiredFields(spec, variant);
        expect(extraRequired, `${spec}.${variant} has unexpected required fields beyond 'type'`).toEqual([]);
      }
    }
  });
});

describe('schema staleness — name alignment', () => {
  it('schema topologies are a subset of catalog capability names', () => {
    // eslint-disable-next-line @typescript-eslint/no-var-requires
    const catalog = require('./capability-catalog.json');
    const catalogNames: string[] = catalog.capabilities.map((c: { name: string }) => c.name);
    const topologies = getSchemaTopologies();
    for (const t of topologies) {
      expect(catalogNames, `Schema topology '${t}' not in catalog`).toContain(t);
    }
  });
});
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `yarn workspace @casehubio/mosaic-core test`
Expected: PASS — all staleness checks green (TS lists match schema).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic add packages/mosaic-core/src/types/patterns/schema-staleness.test.ts
git -C /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic commit -m "feat: add schema-staleness test for sub-spec drift detection Refs #284"
```

---

### Task 4: Remove orphaned type members and verify green

**Files:**
- Modify: `packages/mosaic-core/src/types/orchestration.ts`
- Modify: `packages/mosaic-core/src/types/patterns/patterns.test.ts`

**Interfaces:**
- Consumes: All types from Tasks 1-3
- Produces: Clean type surface with no orphaned members

- [ ] **Step 1: Identify orphaned type members**

Search for references to the old `TerminationSpec` fields that were
mosaic-specific and not in the schema:

```bash
grep -rn "iterations\?\|resolvedStatuses\|maxDisputeRounds" /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic/packages/ --include='*.ts' --exclude-dir=node_modules
```

Also check old `JudgmentTrigger`, `JudgmentCaller`, `JudgmentSpec` types:

```bash
grep -rn "JudgmentTrigger\b\|JudgmentCaller\b" /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic/packages/ --include='*.ts' --exclude-dir=node_modules
```

- [ ] **Step 2: Remove orphaned fields from old TerminationSpec**

If the old `TerminationSpec` interface (with `iterations?`,
`resolvedStatuses?`, `maxDisputeRounds?`, `when?`) is still in
`orchestration.ts`, remove it — the new discriminated union
replaces it. Verify no runtime code references those fields.

- [ ] **Step 3: Remove inline judgment types from deliberation.ts**

If `JudgmentTrigger`, `JudgmentCaller`, `JudgmentSpec` are still
in `patterns/deliberation.ts` after Task 2, remove them.
They are replaced by `JudgmentTriggerSpec`, `JudgmentCallerSpec`,
`JudgmentSpecDef` from `orchestration.ts`.

- [ ] **Step 4: Verify no remaining references to old types**

```bash
grep -rn "confidence-threshold\|fan-out" /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic/packages/ --include='*.ts' --exclude-dir=node_modules
```

If found, update to schema-aligned camelCase (`confidenceThreshold`, `fanOut`).

- [ ] **Step 5: Run full build and test suite**

Run: `yarn typecheck && yarn test`
Expected: PASS — all typechecks and tests green.

- [ ] **Step 6: Run full build**

Run: `yarn build`
Expected: PASS — project builds cleanly.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic add packages/mosaic-core/src/types/
git -C /Users/mdproctor/claude/casehub/slots/219/casehub-mosaic commit -m "chore: remove orphaned type members and align naming Refs #284"
```

---

## References

- [2026-10-25-derive-types-from-schema-design.md] — design spec this plan implements
- `agency/agentic-yaml/src/test/resources/schema/pattern-spec-schema.json` — source schema (1191 lines)
- `mosaic/packages/mosaic-core/src/types/orchestration.ts` — sub-spec types to upgrade
- `mosaic/packages/mosaic-core/src/types/patterns/` — category interfaces
- `mosaic/packages/mosaic-core/src/types/patterns/catalog-staleness.test.ts` — existing staleness test pattern
- `mosaic/packages/mosaic-core/src/types/patterns/patterns.test.ts` — existing type-level tests
- casehubio/casehub-mosaic#284 — focal issue
- casehubio/casehub-mosaic#283 — parent epic
- casehubio/agency#353 — taxonomy enum and catalog generator (closed dependency)
