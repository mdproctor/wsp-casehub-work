# HANDOFF — 2026-08-24

## Last Session

Built the `engine/generator/` module for model-canonical JSON Schema generation (engine#975). Uses victools/jsonschema-generator 4.38.0 with 7 custom modules to reflect on `CaseDefinition.class` and produce Draft 2020-12 schema. Key design decisions: reflection over JavaParser (D1), victools over custom walker (D2), one-off type alignment as prerequisite (D5). 4 commits, 19 tests green, 1 structural equivalence test @Disabled (217 differences remaining — iterative refinement).

## Immediate Next Step

Continue closing the structural equivalence gap. Three categories of work remain:
1. **More custom modules** (~7): GoalExpression, HumanTask, CloudEventTrigger, AgentModel, Cbr, Authorization, Capability schema overrides
2. **Enum handling**: victools emits `$ref` to enum types instead of inline `type: string, enum: [...]`
3. **@JsonPropertyDescription migration**: ~170 annotations to add across model types

After equivalence: YAML expansion (engine#976, Batch 3) — 13 CaseDefinition fields + 4 per-worker annotation fields.

## Cross-Module

**Blocked by:**
- worker-api — rename `Worker.capabilityNames` → `capabilities`, `Capability.inputSchema` → `inputProjection` (gates full structural equivalence, custom modules bridge for now) · S · Low

## References

| Doc | Path |
|-----|------|
| Design spec | `specs/issue-422-ts-programming-model/2026-08-24-schema-generator-design.md` |
| Decisions | `specs/issue-422-ts-programming-model/decisions.md` |
| Implementation plan | `plans/2026-08-24-schema-generator.md` |
| Design review | `~/reviews/casehub-slots/issue-422-schema-generator-20260824-044035/` |
