# Handover — 2026-05-29

## Last Session

Closed `issue-220-capability-registry` — `Capability` value type with kebab-case enforcement,
`CapabilityRegistry` SPI with config-driven validation mode (STRICT/WARN/PERMISSIVE),
`CapabilityParser` with strict and lenient parse modes, wired into `WorkItemService` +
`WorkItemAssignmentService`. 757 tests pass. ADRs 0003+0004 promoted. Pushed to upstream.

## Immediate Next Step

**`issue-223-provenance-supplement`** — branch exists locally, no PR filed.
Four consecutive handovers. Decide: raise the PR or close the branch.

## Cross-Module

*Unchanged — `git show HEAD~2:HANDOFF.md`*

## What's Left

- `issue-223-provenance-supplement` — open branch, no PR, deferred four times · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| parent#66 | Apply CLAUDE.md size discipline to remaining casehubio repos | L | Low | casehub-work is reference impl |

## Key References

- Garden: GE-20260529-636a36 (JAX-RS mapper bypassed by generic catch — new this session)
- Garden: GE-20260529-6eccfe (strict-on-write / lenient-on-read parse technique — new this session)
- Garden: GE-20260529-5a8158 (ConfigMapping prefix claims @ConfigProperty keys — new this session)
- Blog: `2026-05-29-mdp08-string-that-looked-like-a-string.md`
- ADRs: `docs/adr/0003-capability-vocabulary-as-validated-value-type.md`, `0004-capability-validation-mode-as-deployment-config.md`
