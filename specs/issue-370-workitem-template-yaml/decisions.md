## D1: Both standalone and inline YAML sources

**Choice:** Templates can be declared in standalone classpath files (`META-INF/work-templates.yaml`) AND inline in CaseDefinition.yaml under a top-level `workItemTemplates:` section. Both feed through the same resolution path.
**Alternatives:**
- Classpath only — simpler but forces case-specific templates into separate files
- Inline only — co-located but can't share templates across cases
**Rationale:** Standalone files enable reuse across cases and deployments. Inline enables self-contained CaseDefinition YAML for tutorials. Both converge at `findByRef(name)`.
**Trade-offs:** Two scan locations. Acceptable — the SPI abstracts the source.
**Sources:** WorkItemTemplateService.findByRef(), HumanTaskTarget.templateRef(), casehubio/parent#247 (yaml-core)
**Exploration:** quick
**Status:** captured

## D2: WorkItemTemplateProvider SPI with build-time default

**Choice:** New `WorkItemTemplateProvider` SPI in the api module. Default implementation is a Quarkus `@BuildStep` processor that scans classpath YAML at build time.
**Alternatives:**
- Runtime startup observer — simpler but misses Quarkus build-time optimization and native image pre-registration
- Direct store injection — no SPI, YAML module calls store.save() directly. Tighter coupling.
**Rationale:** SPI decouples the template source from the store. Build-time processor is consistent with Quarkus extension patterns and enables native image. Other providers (REST, programmatic, external config) can implement the same SPI.
**Trade-offs:** Adds an SPI for what's initially a single implementation. Acceptable — the SPI is small (one method) and the pattern is established in this codebase.
**Sources:** WorkItemTemplateStore interface, Quarkus @BuildStep pattern, casehub-work deployment module
**Exploration:** quick
**Status:** captured

## D3: Variable resolution via yaml-core at instantiation time

**Choice:** Templates support `${var.name}` expressions using yaml-core `VariableResolver`. Variables are resolved at template instantiation time (in `mergeRequestWithTemplate`), not at load time. Templates are stored with unexpanded expressions.
**Alternatives:**
- Static only — no variable support. Simpler but less powerful for tutorials showing parameterized templates.
- Load-time resolution — resolve variables when YAML is parsed. Loses the ability to parameterize per-instantiation.
**Rationale:** Instantiation-time resolution lets the same template produce different WorkItems based on case context. yaml-core `VariableResolver` is the platform-standard mechanism. Storing unexpanded expressions means templates are reusable across different contexts.
**Trade-offs:** `mergeRequestWithTemplate` gains a VariableResolver dependency. Template fields must be treated as potentially containing expressions, not literal values.
**Depends on:** D2 (templates loaded with raw expressions, not pre-resolved)
**Sources:** casehub-yaml-core VariableResolver, WorkItemTemplateService.mergeRequestWithTemplate()
**Exploration:** quick
**Status:** captured

## D4: Top-level workItemTemplates section in CaseDefinition

**Choice:** Inline templates go in a `workItemTemplates:` section at the CaseDefinition spec level (alongside `bindings:`, `goals:`, `capabilities:`). Templates declared once, referenced by name from `humanTask.templateRef:`.
**Alternatives:**
- Nested on humanTask — template config lives on the binding directly. No separate section. But no reuse across bindings within the same case.
**Rationale:** Separation enables one template referenced by multiple bindings. Matches the existing pattern where `templateRef:` is a name reference, not inline config. The humanTask binding already has inline config — this adds a reusable alternative.
**Trade-offs:** Adds a new top-level section to CaseDefinition schema. Acceptable — the schema already has many top-level sections.
**Depends on:** D1 (inline is one of two sources)
**Sources:** CaseDefinition.yaml schema, HumanTaskTarget.templateRef(), BindingDeserializer
**Exploration:** quick
**Status:** captured

## D5: Resolution priority — DB over YAML

**Choice:** `findByRef(name)` checks DB-persisted templates first (REST-created), then SPI-provided templates (YAML). DB wins on name collision.
**Alternatives:**
- YAML over DB — YAML is the "source of truth", DB overrides are ignored. Prevents runtime customization.
- Error on collision — reject ambiguous names. Too strict for the override use case.
**Rationale:** DB-first lets operators override YAML defaults at runtime without redeploying. The YAML template is the default; REST-created template with the same name is the override. This matches how Quarkus config works (runtime > build-time > default).
**Trade-offs:** Silent override — a REST-created template shadows a YAML template with no warning. Acceptable for pre-release; add a log warning if it becomes confusing.
**Sources:** WorkItemTemplateService.findByRef(), Quarkus config precedence model
**Exploration:** quick
**Status:** captured
