# Decisions — mosaic#284

## D1: Derivation scope

**Choice:** YAML document shapes only — PatternDefinition structure + sub-specs (RoutingSpec, TerminationSpec, AggregationSpec, AgentRefSpec, etc.)
**Alternatives:**
- Everything (YAML + runtime/audit types) — much larger scope, requires a second schema or manual traceability for types agency doesn't own
**Rationale:** The JSON Schema covers the YAML document shape (PatternDefinition). Runtime types like ExecutionSnapshot and OrchestrationAuditEvent are mosaic's own domain — agency doesn't generate schemas for them and shouldn't constrain them.
**Trade-offs:** Runtime types can drift from Java without detection, but they're mosaic-owned and intentionally different.
**Sources:** pattern-spec-schema.json (agency), orchestration.ts (mosaic), patterns/ directory (mosaic)
**Exploration:** quick
**Status:** captured

## D2: Consumption mechanism

**Choice:** Staleness test — hand-maintained TS types validated against the committed JSON Schema
**Alternatives:**
- Codegen from schema — perfect fidelity, zero drift, but adds a build step, generated types may be awkward, and mosaic loses editorial control over type naming/structure
- Hybrid (generate sub-specs, hand-maintain categories) — middle ground but two maintenance modes increases cognitive load
**Rationale:** Mosaic needs editorial control over its type shapes for UI consumption. A staleness test catches drift without forcing generated type ergonomics. Same pattern as the existing catalog staleness test.
**Trade-offs:** Manual sync required when Java types change. Test failure is the signal, not automatic update.
**Sources:** catalog-staleness.test.ts (mosaic), pattern-spec-schema.json (agency)
**Exploration:** quick
**Status:** captured

## D3: Validation depth

**Choice:** Variant names + required fields per variant
**Alternatives:**
- Variant names only — catches new/removed variants but misses field-level drift
- Full structural match — maximum fidelity but brittle; any optional Java field addition breaks mosaic CI
**Rationale:** Required fields represent the contract — if Java says a field is required, mosaic must know about it. Optional fields can drift without breaking consumers.
**Trade-offs:** Optional field additions in Java won't be detected. Mosaic may lag behind on optional features.
**Sources:** pattern-spec-schema.json oneOf structure with required arrays per variant
**Exploration:** quick
**Status:** captured

## D4: Schema delivery

**Choice:** Commit pattern-spec-schema.json directly into mosaic alongside capability-catalog.json
**Alternatives:**
- Fetch from agency at test time — always up-to-date but adds network dependency to CI
- npm package — clean dependency but heavy infrastructure for two JSON files
**Rationale:** Same pattern already established for capability-catalog.json. Cross-repo PRs update both files when agency schema changes. No build coupling.
**Trade-offs:** Manual copy step when agency schema changes. Agency's SchemaDriftTest catches schema changes on the agency side; mosaic's staleness test catches mismatches on the mosaic side.
**Sources:** capability-catalog.json (already committed in mosaic)
**Exploration:** quick
**Status:** captured

## D5: Name alignment

**Choice:** Verify and close — the catalog is the contract, names already match
**Alternatives:**
- Investigate renaming channel-execution to match qhorus terminology — would require catalog regeneration and cross-repo coordination
**Rationale:** The capability-catalog.json is the contract between agency and mosaic. Both `decomposition` and `channel-execution` already match between catalog and mosaic TS types. The PatternType.HTN ↔ CapabilityType.DECOMPOSITION mapping is internal to Java and doesn't affect the contract.
**Trade-offs:** If qhorus renames its channel terminology, a future catalog update will propagate the change via the staleness test.
**Sources:** capability-catalog.json, CapabilityType.java, catalog-staleness.test.ts
**Exploration:** quick
**Status:** captured

## D6: Test file organisation

**Choice:** Separate schema-staleness.test.ts alongside existing catalog-staleness.test.ts
**Alternatives:**
- Expand catalog-staleness.test.ts to include schema checks — single file but mixes two concerns (name list vs structural shape)
**Rationale:** Catalog staleness checks names/categories. Schema staleness checks structural shapes. Separate files, separate concerns, clearer failure messages.
**Trade-offs:** Two test files to maintain instead of one. Minor overhead.
**Sources:** catalog-staleness.test.ts (existing pattern)
**Exploration:** quick
**Status:** captured
