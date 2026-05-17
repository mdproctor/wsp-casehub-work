# Design Journal — epic-output-schema

### 2026-05-17 · §Build Roadmap

The `WorkItemFormSchema` entity (Phase 9, added in #98) was introduced when templates were simple default-value containers and schemas were treated as a separate category-level concern. With templates now serving as the canonical type definition — carrying outcomes, assignment strategies, deadlines, and routing rules — a separate schema registry became redundant and created a confusing two-schema system with unclear precedence.

Phase 22 consolidates type-level contracts onto the template: `WorkItemTemplate.inputDataSchema` and `outputDataSchema` (JSON Schema draft-07, TEXT columns) validate payload at instantiation and resolution at completion respectively. Both are snapshotted onto `WorkItem` at instantiation — the same pattern as `permittedOutcomes` (Phase 21) — so the WorkItem is self-governing after creation. Validation moved from the resource adapter layer into `WorkItemService`, making it consistent across all creation paths (REST, spawn, multi-instance).

`WorkItemFormSchema` entity, its CRUD REST resource, and all associated tests were deleted. `FormSchemaValidationService` (the JSON Schema library wrapper) is retained as a pure utility injected into `WorkItemService`. The UI schema-discovery path that previously called `GET /workitem-form-schemas?category=X` now calls `GET /workitem-templates/{id}` — no information is lost, just consolidated. Template versioning (#180) is deferred as the fuller reproducibility answer.

### 2026-05-17 · §Flyway Migration History

Three migrations were added: V23 and V24 add `input_data_schema` / `output_data_schema` TEXT columns to `work_item_template` and `work_item` respectively. V25 drops the `work_item_form_schema` table. Each uses separate `ALTER TABLE` statements — H2 in `MODE=PostgreSQL` does not support multi-column `ADD COLUMN` in a single statement.
