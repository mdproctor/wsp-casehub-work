## D1: Nested YAML structure for escalation and skillMatch

**Choice:** Nested objects under `humanTask:` — `escalation:` and `skillMatch:` — mirroring the annotation grouping
**Alternatives:**
- Flat fields on HumanTask (`escalationOnExpiry`, `skillMatchStrategy`, etc.) — clutters the namespace, loses logical grouping
**Rationale:** `@Escalate` and `@SkillMatch` are separate annotations with their own attribute sets. Nested YAML mirrors this structure, reads naturally, and keeps HumanTask's top-level properties focused on core task config.
**Trade-offs:** Nested objects add one level of YAML indentation. Acceptable — the grouping provides clarity.
**Sources:** `@Escalate` annotation (4 attributes), `@SkillMatch` annotation (3 attributes), CaseDefinition.yaml HumanTask $defs
**Exploration:** quick
**Status:** captured

## D2: Inner records on HumanTaskTarget

**Choice:** Add `EscalationConfig` and `SkillMatchConfig` as inner records on `HumanTaskTarget`, with corresponding builder methods
**Alternatives:**
- Standalone records in `io.casehub.api.model` package — adds two new top-level files for small, tightly-coupled config
- Reuse work-annotations descriptors (`EscalationDescriptor`, `SkillMatchDescriptor`) — engine-api cannot depend on work-annotations; wrong dependency direction
**Rationale:** Inner records express that these configs are part of the HumanTask target definition, not standalone concepts. Matches `HumanTaskTarget`'s existing builder pattern. Engine-api has no dependency on work modules.
**Trade-offs:** Makes HumanTaskTarget slightly larger. Acceptable — the records are 4-5 fields each.
**Sources:** HumanTaskTarget.java (engine-api), EscalationDescriptor.java, SkillMatchDescriptor.java (work-annotations)
**Exploration:** quick
**Status:** captured

## D3: Escalation config threading via WorkItemCreateRequest string fields

**Choice:** Add `escalationOnExpiry` and `escalationOnClaimDeadline` as string fields on `WorkItemCreateRequest`, alongside the existing `routingStrategy`/`minimumScore` skill-match fields. `escalationDeadline` maps to existing `expiresAt` / `claimDeadline` fields depending on context. `generateSummary` passes as a boolean field.
**Alternatives:**
- Opaque JSON blob field (`escalationConfig`) — loses type safety, harder to validate
- Scope-only SlaBreachPolicy lookup — can't express per-WorkItem escalation targets in YAML
**Rationale:** Explicit fields match the existing pattern for skill-match data (`routingStrategy`, `requiredCapabilities`, `minimumScore`). SlaBreachPolicy reads these at breach time to determine escalation targets. Per-WorkItem config is more flexible than scope-only lookup.
**Trade-offs:** Adds 3 fields to WorkItemCreateRequest (already 30+ fields). Acceptable — WorkItemCreateRequest is a flat builder DTO by design.
**Depends on:** D2 (HumanTaskTarget carries the data that flows through here)
**Sources:** WorkItemCreateRequest.java (existing skill-match fields), SlaBreachPolicy SPI, garden GE-20260511-3e5a75
**Exploration:** quick
**Status:** captured
