# HANDOFF — 2026-07-07

## Last Session

Implemented #291 (types/labels convention on WorkItem/WorkItemTemplate). Added `types: Set<Path>` to both entities, removed `category` (49+ call sites across 12 modules). WorkItemType embeddable with `@ElementCollection` join table + unique constraint. Path validation via `Path.parse()` at template instantiation and direct creation. Ancestor matching in all three store implementations (JPA LIKE, InMemory Path.isAncestorOf, MongoDB $regex). Types promoted to direct `@JsonProperty` on WorkItemLifecycleEvent. Flyway V42. Design review (13 issues, all verified). Code review (10 findings, all Critical/Important fixed). 1,493 tests passing across 9 modules.

## Immediate Next Step

File the cross-repo issues that were created: garden#4 (protocol file), parent#354 (PLATFORM.md cross-ref). Update engine#653 work-adapter to use types instead of category.

## What's Left

- casehubio/garden#4 — create `definable-entity-types-labels.md` protocol file · S · Low
- casehubio/parent#354 — PLATFORM.md protocol cross-reference · XS · Low
- casehubio/work#293 — minor review findings: SpawnCallerRefTest field name, KeywordSkillMatcher null guard, notification type matching · XS · Low
- casehubio/work#294 — sync api-reference.md and integration-guide.md for category→types · S · Low
- casehubio/engine#653 — engine work-adapter update for types (source-breaking: WorkItemCreateRequest.category removed) · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #293 | Minor review findings from #291 | XS | Low | |
| #294 | Doc sync for category→types | S | Low | |
| #288 | Queue summary REST endpoint for blocks-ui dashboard cards | — | — | Enhancement |
| #289 | Historical queue trend data for sparklines | — | — | Enhancement |
| #290 | Relocate HumanTask adapter from engine to work | L | Med | |
| #287 | Retrofit work SPIs to extend NamedStrategy | M | Med | engine#634 dep |
| #152 | Split examples into core and full variants | M | Low | |

## References

- Blog: `2026-07-07-mdp07-the-second-adopter.md`
- Spec: `specs/2026-07-06-workitem-types-labels-design.md` (workspace)
- Plan: `plans/attic/issue-291-workitem-types/2026-07-06-workitem-types-labels.md`
- Cross-repo: garden#4, parent#354, engine#653, work#293, work#294
