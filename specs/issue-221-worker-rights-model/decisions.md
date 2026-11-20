## D1: WorkerAction extensibility model

**Choice:** Record-based value type — `record WorkerAction(String name, AclAction aclAction)`
**Alternatives:**
- Interface-based with domain enums — Jackson polymorphic serialization adds complexity for no gain
- String-based with separate mapping — loses semantic pairing of action-name-to-AclAction in the credential
- Sealed interface across modules — sealed can't name implementations in other modules; doesn't work for cross-repo extensibility
**Rationale:** Simplest generalization that preserves the semantic pairing. `name` serves audit, `aclAction` drives enforcement. Domains define their own constants without extending an enum. No serialization complexity — value equality by content.
**Trade-offs:** No compile-time exhaustive switch on actions. Acceptable because current code never switches on actions — it only calls `.aclAction()`. Invalid name-to-AclAction pairings are possible but mitigated: actions are constructed only by orchestrators via domain constants classes, not by arbitrary callers.
**Exploration:** quick
**Status:** captured

## D2: Credential resource scope

**Choice:** Replace `UUID caseId` with `ResourceId` value type — `record ResourceId(String type, String id)`
**Alternatives:**
- Raw `String resourceId` — loses structural validation; aligns with `AccessControlProvider`'s existing weakness rather than fixing it
- Keep `UUID caseId` and add alongside — transitional state that never gets cleaned up
**Rationale:** Pre-release platform — breaking changes cost nothing, so do the right thing. `ResourceId` enforces format at construction, provides structured access to type and id components. Serializes as `"type:id"` via Jackson. Follow-up: retrofit `ResourceId` into `AccessControlProvider` (separate issue — broader ecosystem impact).
**Trade-offs:** New value type. Negligible cost — records are trivial.
**Depends on:** D1 (WorkerAction extensibility)
**Exploration:** quick
**Status:** revised (R1-02: ResourceId value type replaces raw String)

## D3: WorkerPermissionRequest authorization context

**Choice:** Marker interface `WorkerAuthorizationContext` — domains implement with their own fields, policy does one safe downcast
**Alternatives:**
- Generic `<C>` type parameter — creates type-erasure gap at the filter-to-policy boundary (D4's single reusable filter can't be parameterized)
- Replace `caseDefinitionId` with `String definitionId` — still hardcodes a single field
- Generic `Map<String, String> context` — stringly-typed, no discoverability
**Rationale:** Eliminates the generic/filter tension (R1-03, R1-08). The policy is always domain-specific, so the one downcast is safe and happens in exactly one place. Same pluggability as generics, no type-erasure gap. The `ContextBridge<T>` pattern uses generics because the bridge itself needs type-safe serialization/deserialization — the authorization policy doesn't.
**Trade-offs:** One unchecked downcast in each domain's policy implementation. Acceptable — domain-specific policy knows its own context type.
**Depends on:** D1 (WorkerAction extensibility), D2 (Credential resource scope)
**Exploration:** quick
**Status:** revised (R1-03: marker interface replaces generic <C>)

## D4: Platform filter architecture

**Choice:** Single reusable filter + `WorkerScopeExtractor` SPI in a new platform module
**Alternatives:**
- Abstract filter base class — every domain writes a subclass; can't compose multiple extractors; less flexible
**Rationale:** Consistent with platform's SPI + `@DefaultBean` pattern. One filter for all services. Scope validation is opt-in via `WorkerScopeExtractor`; `NoOpWorkerScopeExtractor @DefaultBean` skips scope validation (token-only mode). JAX-RS dependency means it can't go in `platform-api` — gets its own module.
**Trade-offs:** Adds a new platform module. Acceptable because the module is thin (filter + SPI) and follows the established classpath-activation pattern. Default NoOp means "no scope enforcement" — consistent with platform's opt-in security model (NoOpAccessControlProvider, AutoApprovePolicy). Services that need scope enforcement must provide an extractor.
**Depends on:** D2 (Credential resource scope)
**Exploration:** quick
**Status:** captured

## D5: Filter module naming and placement

**Choice:** New `acl-worker/` module — capability-focused name consistent with `acl-inmem/`, `acl-jpa/`
**Alternatives:**
- `worker-credential-filter/` — names the mechanism (JAX-RS filter), not the capability
- Put in `platform/` (universal dependency) — auto-activates filter everywhere
**Rationale:** JAX-RS filters activate by classpath presence. Must be opt-in. Module naming follows capability convention (`acl-*`), not implementation mechanism. Wouldn't need renaming if implementation changes (e.g., Vert.x handler).
**Trade-offs:** Adds a module. Acceptable — module is thin.
**Depends on:** D4 (Platform filter architecture)
**Exploration:** quick
**Status:** revised (R1-05: capability-focused naming)
