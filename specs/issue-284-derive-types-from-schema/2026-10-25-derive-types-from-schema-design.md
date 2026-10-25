# Design: Derive mosaic agency types from generated schema

**Issue:** casehubio/casehub-mosaic#284
**Epic:** casehubio/casehub-mosaic#283 (Unified agency capability taxonomy)
**Dependency:** casehubio/agency#353 (closed)

## Problem

Mosaic's TS types for agency patterns are hand-maintained with loose shapes.
Sub-specs like `AggregationSpec`, `TerminationSpec`, and `RoutingSpec` use
`{ type: string }` — losing all variant-level type safety that exists in
Java's sealed hierarchies. There is no mechanism to detect structural drift
between Java and TS.

The existing `catalog-staleness.test.ts` validates only the capability name
list (28 names x 8 categories). It does not check sub-spec shapes.

## Solution

Expand the staleness testing to validate TS type *shapes* against agency's
generated `pattern-spec-schema.json`. Upgrade the hand-maintained TS sub-spec
types from loose `{ type: string }` to proper discriminated unions. No codegen —
mosaic keeps editorial control over its types; the test ensures they stay
aligned with Java.

## Scope boundaries

**In scope:**
- YAML document shape types (`PatternDefinition` structure + sub-specs)
- Sub-spec types imported by pattern categories from `orchestration.ts`
- New `schema-staleness.test.ts` with structural validation

**Out of scope:**
- Runtime/audit types (`ExecutionSnapshot`, `OrchestrationAuditEvent`,
  `OrchestrationPayload`, etc.) — these are mosaic's own domain
- Code generation tooling
- Renaming `decomposition` or `channel-execution` (already aligned in catalog)

## Architecture

### Schema delivery

Copy `pattern-spec-schema.json` from agency into
`packages/mosaic-core/src/types/patterns/pattern-spec-schema.json`
alongside the existing `capability-catalog.json`. Same cross-repo
update pattern when agency schema changes.

### TS type upgrades

Upgrade sub-spec types from `{ type: string }` to discriminated unions.
The types to upgrade, derived from the schema's `$defs`:

| Type | Current (mosaic) | Target (from schema) |
|------|-----------------|---------------------|
| `RoutingSpec` | `{ type: string; guard?: string }` | `{ type: 'first-match'; guard?: string } \| { type: 'round-robin' } \| { type: 'sequential' } \| { type: 'llm-selected' } \| { type: 'select-all' }` |
| `AggregationSpec` | `{ type: string }` | `{ type: 'pass-through' } \| { type: 'collect-all' } \| { type: 'majority-vote' } \| { type: 'auction' }` |
| `TerminationSpec` | `{ type: string; iterations?; resolvedStatuses?; ... }` | 12-variant discriminated union (see below) |
| `DecompositionSpec` | `{ strategy: string }` | 8-variant discriminated union: `identity \| static \| llm \| hybrid \| heuristic \| goap \| capability-dependency \| forward-reasoning` |
| `ActivationSpec` | not present | `{ type: 'on-dispatch' } \| { type: 'max-iterations' }` |
| `AgentRefSpec` | `AgentRef` with `type: AgentRefType` | 5-variant union with per-variant fields (e.g., Channel adds `channelId`, Composed adds `pattern`) |
| `TaskNodeSpec` | not present | `{ type: 'primitive'; ... } \| { type: 'compound'; ... }` |
| `JudgmentSpec` | `{ trigger: { type: string }; caller: { type: string } }` | Nested discriminated unions: trigger (4 variants), caller (3 variants), agreement (3 variants) |
| `FailurePolicy` | mosaic-shaped (routing/aggregation actions) | Schema-shaped (onRoutingFailure, onDeadlock, agentRetry, replanPolicy) |
| `GovernanceSpec` | not present | `{ oversight?; trustRouting?; cbrRouting?; attestation? }` |

**TerminationSpec 12 variants:** `max-iterations`, `goal-reached`,
`judge-convergence`, `all-agreed`, `supervisor`, `contested`,
`convergence`, `single-pass`, `agent-count`, `accepted`,
`terminal-outcome`, `deadline`.

**FailurePolicy divergence note:** Mosaic's current `FailurePolicy` uses
different field names (`routingFailureAction`, `aggregationFailureAction`,
`agentRetryPolicy`) vs the schema (`onRoutingFailure`, `onDeadlock`,
`agentRetry`, `replanPolicy`). The schema also has `replanPolicy` which
mosaic lacks. The upgrade aligns mosaic to the schema's field names.

### Where types live

The sub-spec types currently live in `packages/mosaic-core/src/types/orchestration.ts`.
This file also contains runtime/audit types that are out of scope.

**Approach:** Keep the upgraded sub-spec types in `orchestration.ts` (same location,
same imports). Add new types (`ActivationSpec`, `DecompositionSpec`, `TaskNodeSpec`,
`GovernanceSpec`) to `orchestration.ts` as well. The runtime/audit types in the
same file are untouched.

The pattern category interfaces (`OrchestrationPatternCategory`,
`DeliberationPatternCategory`, etc.) in `patterns/` import from `orchestration.ts`
and will pick up the upgraded types automatically.

### Schema staleness test

New file: `packages/mosaic-core/src/types/patterns/schema-staleness.test.ts`

The test parses `pattern-spec-schema.json` and validates:

1. **Top-level topology discriminators** — the `oneOf` at the schema root
   lists 8 type values (`supervisor`, `debate`, `loop`, `parallel`, `voting`,
   `conditional`, `sequence`, `htn`). The test extracts these and compares
   against a TS list. This catches new/removed pattern topologies.

2. **Sub-spec variant names** — for each `$defs` entry that uses `oneOf`
   with a `type` discriminator const, extract the variant names. Compare
   against the corresponding TS type's union members. This catches
   new/removed variants in any sub-spec.

3. **Required fields per variant** — for each variant in a `oneOf`,
   check the `required` array (beyond the universal `type` field).
   Verify the corresponding TS variant type declares those fields as
   non-optional. This catches required field additions in Java.

**Test structure:**

```typescript
import schema from './pattern-spec-schema.json';

// Extract all oneOf-discriminated defs from $defs
function extractVariantNames(defName: string): string[] { ... }
function extractRequiredFields(defName: string, variantType: string): string[] { ... }

describe('schema staleness', () => {
  // Top-level topologies
  it('pattern topologies match schema', () => { ... });

  // Per sub-spec: variant names
  it('RoutingSpec variants match schema', () => { ... });
  it('AggregationSpec variants match schema', () => { ... });
  // ... etc for each sub-spec

  // Per sub-spec: required fields per variant
  it('RoutingSpec required fields match schema', () => { ... });
  // ... etc
});
```

**What the test does NOT check:** optional fields, field types, nested
object shapes beyond the top level. This is intentional — mosaic may
choose different optional field granularity for its UI needs.

### Orphan removal

After upgrading the types, remove any TS type members that don't trace
to the schema. Concrete candidates based on current analysis:

- `TerminationSpec.resolvedStatuses` — not in schema (mosaic-specific)
- `TerminationSpec.maxDisputeRounds` — not in schema
- `TerminationSpec.when` — not in schema
- `RoutingSpec.guard` — present in mosaic's `routing: { type: string; guard?: string }`
  but not in the schema's `RoutingSpec` (moved to `first-match` or removed)

These should be verified during implementation. If they serve a
mosaic-specific purpose not covered by the YAML spec, they can stay
but must be documented as mosaic extensions.

### Name alignment verification

Add assertions to the existing `catalog-staleness.test.ts` (or the new
`schema-staleness.test.ts`) confirming:
- `decomposition` in catalog matches `decomposition` in TS types
- `channel-execution` in catalog matches `channel-execution` in TS types
- All schema topology type values match a subset of catalog capability names

This closes the name alignment scope item with a test rather than a rename.

## Testing strategy

| Test | What it catches | Failure mode |
|------|----------------|-------------|
| `catalog-staleness.test.ts` (existing) | Capability name/category drift | New CapabilityType in Java not reflected in TS |
| `schema-staleness.test.ts` (new) | Sub-spec structural drift | New variant or required field in Java not reflected in TS |
| TS compilation | Type errors in consumers | Upgraded discriminated unions catch invalid pattern configs at compile time |

## Data flow

```
agency: Java sealed hierarchies
    ↓ (AgencySchemaGenerator + SchemaDriftTest)
agency: pattern-spec-schema.json (committed baseline)
    ↓ (manual copy via cross-repo PR)
mosaic: pattern-spec-schema.json (committed copy)
    ↓ (schema-staleness.test.ts reads at test time)
mosaic: TS types validated against schema
```

When agency's Java types change:
1. `SchemaDriftTest` in agency fails → developer regenerates schema with `-Dschema.update`
2. Developer copies updated schema to mosaic
3. `schema-staleness.test.ts` in mosaic fails → developer updates TS types
4. Both repos updated in a coordinated PR

## Implementation order

1. Copy `pattern-spec-schema.json` into mosaic
2. Write `schema-staleness.test.ts` — initially it will fail because TS types are loose
3. Upgrade sub-spec types in `orchestration.ts` to discriminated unions
4. Update pattern category interfaces to use new sub-spec types
5. Add new types (`ActivationSpec`, `DecompositionSpec`, `TaskNodeSpec`, `GovernanceSpec`)
6. Fix `schema-staleness.test.ts` to pass
7. Remove orphaned type members
8. Add name alignment assertions
9. Verify all existing tests still pass

## References

- `agency/agentic-yaml/src/test/resources/schema/pattern-spec-schema.json` — source schema
- `agency/agentic-yaml/src/main/java/io/casehub/agency/agentic/yaml/AgencySchemaGenerator.java` — schema generator
- `agency/agentic-yaml/src/main/java/io/casehub/agency/agentic/yaml/CapabilityCatalogGenerator.java` — catalog generator
- `mosaic/packages/mosaic-core/src/types/patterns/catalog-staleness.test.ts` — existing staleness test
- `mosaic/packages/mosaic-core/src/types/orchestration.ts` — sub-spec types to upgrade
- `mosaic/packages/mosaic-core/src/types/patterns/` — category interfaces
- casehubio/agency#353 — taxonomy enum and catalog generator (closed)
- casehubio/casehub-mosaic#283 — parent epic
