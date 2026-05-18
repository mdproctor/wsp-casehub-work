# Design Journal — epic-excluded-users

### 2026-05-18 · §Build Roadmap

Conflict-of-interest enforcement is a policy concern, not a data concern. Rather than hardcoding a comma-separated field check throughout the codebase, we introduced `ExclusionPolicy` as a first-class SPI in `casehub-work-api`, consistent with `EscalationPolicy`, `ClaimSlaPolicy`, and `WorkerSelectionStrategy`. The `CommaSeparatedExclusionPolicy` `@DefaultBean` covers the immediate OHT use case (excluding the task initiator by user ID). Custom implementations (LDAP groups, role-based, time-window) are pluggable without touching core code.

Enforcement spans all five assignment paths — manual claim, direct assignee at creation, delegation, auto-assignment candidate filter, and `SelectionContext` for external strategies. The `SelectionContext` addition is architecturally significant: it ensures the exclusion constraint travels with the selection context, so any external `WorkerSelectionStrategy` implementation has access to it. Omitting it would have left a semantic hole where custom strategies could bypass exclusions. `clone()` deliberately carries `excludedUsers` forward — cloning a WorkItem that has conflict-of-interest rules should inherit those rules, not silently drop them.

### 2026-05-18 · §Flyway Migration History

V23 and V24 add `excluded_users TEXT` to `work_item_template` and `work_item` respectively. Both use separate `ALTER TABLE` statements — H2 in `MODE=PostgreSQL` does not support multi-column `ADD COLUMN` in a single statement (same constraint as V23/V24 from #170, which will need renumbering when both epics merge).
