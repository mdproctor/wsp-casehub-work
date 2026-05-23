# Design Journal — issue-222-close-epic-212-workspace

<!-- Merged from issue-212-sla-breach-policy workspace branch -->

### 2026-05-22 · §Flyway Migration History

V31 adds two nullable `scope VARCHAR(255)` columns — one on `work_item`, one on `work_item_template`.
Scope carries a slash-separated hierarchical path (e.g. `casehubio/devtown/pr-review`) that anchors
a WorkItem's SLA breach policy resolution to the correct `Preferences` scope. Null means root scope;
the expiry service uses `Path.root()` as the fallback. Engine-side propagation of scope to
`HumanTaskTarget` is tracked in engine#330.

### 2026-05-22 · §Build Roadmap

SlaBreachPolicy SPI (#213/#212): `BreachDecision` sealed interface (Fail / EscalateTo / Extend /
Chained), `SlaBreachPolicy` SPI replacing `EscalationPolicy`, `SlaBreachContext` (BreachType,
BreachedTask, Path scope, Preferences), `NoOpSlaBreachPolicy` `@DefaultBean`. `ExpiryLifecycleService`
rewritten: policy-driven decision execution, `SlaBreachEvent` CDI event carrying the leaf decision.
`EscalationPolicy` and qualifiers deprecated; dead implementations marked `@Deprecated` (removal in
work#215). `scanRoots` three-param fix (#205). Queues cleanup: `quarkus-junit`, `@Inject ObjectMapper`,
JQ non-boolean test (#208/#209/#210). `casehub-platform-api` added as compile dep to `casehub-work-api`.

### 2026-05-22 · §Testing Strategy

api: 60 tests including `SlaBreachPolicyTest` (factory, chaining, pattern-match, devtown two-tier
acceptance test). runtime: 746 tests; `ExpiryLifecycleServiceTest` rewritten with 31 tests covering
all four `BreachDecision` variants, `SlaBreachEvent` leaf capture, and `CLAIM_EXPIRED` lifecycle event
unconditional firing. Template→item and create→item scope propagation covered. queues: 84 tests;
JQ non-boolean output path added. `casehub-platform` (Jandex-indexed) added as test dep to runtime and
queues poms so `MockPreferenceProvider @DefaultBean` is discoverable in `@QuarkusTest` contexts.
