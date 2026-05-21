# casehub-work — Session Handover
**Date:** 2026-05-21

## What Was Done This Session

**#182 — WorkItemCreateRequest builder refactor (merged to main):**
- 24-param positional record → `final class` with enforced builder; no positional constructor exists
- `CreateWorkItemRequest` (HTTP DTO record) gained inner Builder
- 60+ call sites across 7 modules migrated via subagent-driven development
- Drift-protection test guards future field/setter mismatches

**Small fixes merged:**
- #197 — `SelectionContext` 8th arg (`excludedUsers`) in 3 casehub-work-core tests
- #196 — `WorkItemQueueEventTest` using JSON body for claim endpoint; fix is `?claimant=alice` query param (body is silently ignored by JAX-RS `@QueryParam`)

**DESIGN.md exhaustive audit:**
- Fixed stale test counts across all modules; added V28 Flyway entry; split examples/queues-examples in test table; added #182 roadmap entry
- Surfaced 2 pre-existing test failures: #196 (now fixed) and #197 (now fixed)

**Branch hygiene:**
- `epic-excluded-users` formally closed via work-end; EPIC-CLOSED.md on branch, deletion due 2026-06-03
- `epic-exclusion-audit` and `epic-output-schema` still open — no EPIC-CLOSED.md, work-end not yet run

**Prompt snippet refined:**
- Clarified cost vs complexity distinction in design philosophy item #5; updated parent docs/prompt-snippets.md

## Current State

- Both repos on `main`, 693 runtime tests, 82 queues tests (all passing)
- `epic-exclusion-audit` branch: retained until 2026-06-02 (already merged)
- `epic-output-schema` branch: needs work-end

## Immediate Next Step

Run `work-end` on `epic-exclusion-audit` and `epic-output-schema` — both merged to main but never formally closed.

## What's Left

- `epic-exclusion-audit` — work-end not run, no EPIC-CLOSED.md · XS · Low
- `epic-output-schema` — work-end not run, no EPIC-CLOSED.md · XS · Low
- #193 — `instantiate()` exclusion gap (assigneeId override vs template excludedUsers) · XS · Low
- #194 — `WorkItemLabelResponse` type on `WorkItemCreateRequest.labels` (response type in service command) · S · Med
- #195 — `WorkItemCreateRequest.toString()` scope and `formKey` omission comments · XS · Low
- casehub-engine #187 — SelectionContext 8th arg in engine (tracked here, fix is in engine repo) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | work-end on epic-exclusion-audit + epic-output-schema | XS | Low | Do first |
| — | casehub-clinical Epic 4 — adverse event escalation | L | Med | — |
| #193 | instantiate() exclusion gap | XS | Low | — |
| #182 debt | #194 label type, #195 toString comments | S | Low | Low priority |

## Key References

- Blog: `blog/2026-05-21-mdp01-record-cant-say-no.md`
- Garden: GE-20260521-6f257b (JAX-RS body ignored), GE-20260521-340888 (git cherry-pick not-merged), GE-20260521-cb1eea (git three-dot diff misleading)
- Protocol: PP-20260521-9d4988 (journal migration number sync)
