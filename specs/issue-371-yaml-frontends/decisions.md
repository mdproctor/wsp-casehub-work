## D1: Integration approach — SlaBreachPolicy implementation

**Choice:** Pure `SlaBreachPolicy` implementation (`DeclarativeSlaBreachPolicy`, id: `"declarative"`)
**Alternatives:**
- Extend `resolveBreachDecision` in `ExpiryLifecycleService` — embeds YAML logic in the service, breaking SPI separation
- PreferenceProvider-based — conflates deployment-level SLA declarations with runtime-configurable preferences
**Rationale:** Preserves the SPI contract, follows the established NamedStrategy + StrategyResolver pattern, doesn't touch ExpiryLifecycleService. The YAML policy is just another policy implementation — clean, testable, composable with per-item config.
**Trade-offs:** Needs `Instance<SlaBreachPolicy>` injection for fallback chaining (slightly more wiring). Requires a new config property to activate (`casehub.work.sla.breach-policy=declarative`).
**Sources:** ExpiryLifecycleService.java:75 (StrategyResolver resolution), NoOpSlaBreachPolicy.java, GE-20260810-724b82 (StrategyResolver is config-driven)
**Exploration:** quick
**Status:** captured

## D2: YAML action syntax — full BreachDecision coverage

**Choice:** Support all `BreachDecision` variants: `fail`, `extend`, `escalateTo` (with optional `deadline`), `exhausted`. Both string shorthand and object form. Skip `Chained` (YAML shouldn't express atomic same-event fallback — it's a programmatic concern).
**Alternatives:**
- Minimal (fail, extend, escalateTo only) — simpler parser but omits `exhausted`, which is a natural terminal action for deployment-wide config
- Minimal + exhausted — arbitrary line; if we support exhausted, there's no principled reason to omit it from "full"
**Rationale:** The YAML surface should map 1:1 to the `BreachDecision` sealed interface (minus `Chained`). This prevents users from hitting a wall where their deployment-wide policy can express some outcomes but not others. String shorthand (`fail`, `extend`) keeps simple cases readable.
**Trade-offs:** Slightly larger parser surface. `Exhausted` is rarely used in deployment-wide defaults (more common in per-scope overrides).
**Sources:** BreachDecision.java, GE-20260810-793376 (Chained is atomic same-event fallback, not suitable for YAML)
**Exploration:** quick
**Depends on:** D1 (integration approach)
**Status:** captured

## D4: Config property override surface — defaults only

**Choice:** Config properties for default actions only: `casehub.work.sla.defaults.on-completion-expiry`, `.on-claim-expiry`, `.claim-extension-hours`, `.extension-hours`. Scopes always come from classpath YAML. When both config properties and classpath YAML exist, config properties win for the defaults section.
**Alternatives:**
- No config overrides — all config from classpath YAML; tutorials must create a resource file even for basic defaults
- Full mirror — mirror entire YAML structure in config properties; SmallRye can't handle polymorphic values (string vs object) without custom converters
**Rationale:** The simplest tutorials need "if unclaimed after 4 hours, escalate to team-leads" without writing a YAML file. Config properties for defaults achieve this. Scopes with compound actions require the full YAML expressiveness and belong in the classpath resource.
**Trade-offs:** Two config surfaces for the same concern (defaults). Config properties are limited to string shorthands only — object-form actions (escalateTo with deadline) require the classpath YAML.
**Sources:** WorkItemsConfig.java, issue #372 acceptance criteria ("Tutorial-ready example")
**Exploration:** quick
**Depends on:** D1, D2 (action syntax)
**Status:** captured

## D3: Fallback chain mechanism — config property

**Choice:** New config property `casehub.work.sla.declarative-fallback=no-op`. The declarative policy looks up this id via `Instance<SlaBreachPolicy>` and delegates when no YAML scope or default matches. Default fallback: `"no-op"` (Fail).
**Alternatives:**
- Hardcoded Fail — simpler but blocks hybrid YAML + Java deployments where custom policies handle exotic scopes
- Inject-all + iterate — auto-discovers policies but ordering is undefined; implicit behavior is worse than explicit config
**Rationale:** Explicit config for the fallback preserves the NamedStrategy selection model. Hybrid deployments set `declarative-fallback=my-custom-policy` and get YAML for most scopes with a CDI policy for the rest. The default `no-op` means standalone YAML works without additional config.
**Trade-offs:** One more config property. The fallback policy must be a valid `SlaBreachPolicy` bean in CDI.
**Sources:** GE-20260810-724b82 (StrategyResolver is config-driven), WorkItemsConfig.SlaConfig
**Exploration:** quick
**Depends on:** D1 (integration approach)
**Status:** captured

## D5: Module placement — runtime

**Choice:** `DeclarativeSlaBreachPolicy` and `SlaDefaultsYamlLoader` live in the runtime module alongside `NoOpSlaBreachPolicy` and `ExpiryLifecycleService`.
**Alternatives:**
- New sla-yaml module — cleaner dependency separation but adds a module for ~3 classes
- api module — wrong; api is for interfaces and records, not implementations
**Rationale:** The declarative policy is a `SlaBreachPolicy` implementation, same as `NoOpSlaBreachPolicy`. Same module, same package (`io.casehub.work.runtime.service`). Jackson YAML is already a transitive dependency (used by `WorkItemTemplateYamlLoader`). No new module needed.
**Trade-offs:** None significant. The runtime module grows by ~3 classes.
**Sources:** NoOpSlaBreachPolicy.java, WorkItemTemplateYamlLoader.java, docs/MODULES.md
**Exploration:** quick
**Depends on:** D1 (integration approach)
**Status:** captured
