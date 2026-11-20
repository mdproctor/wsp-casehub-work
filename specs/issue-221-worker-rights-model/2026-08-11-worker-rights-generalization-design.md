# Worker Rights Model Generalization — Design Spec

**Issue:** casehubio/platform#221
**Date:** 2026-08-11
**Status:** Draft
**Depends on:** Original worker rights implementation (engine#833 Batch 3)

## 1. Problem Statement

The worker rights SPI types (`WorkerAction`, `WorkerCredential`, `WorkerCredentialStore`,
`WorkerAuthorizationPolicy`, `WorkerPermissionRequest`) are already in `platform-api` —
correctly placed for cross-repo consumption. However, the types are engine-flavored:

1. `WorkerAction` is a closed enum with engine-specific vocabulary (`READ_CONTEXT`,
   `SIGNAL_CASE`, `READ_PLAN_ITEMS`). Other domains (AML, connectors) cannot define
   their own actions without modifying platform-api.
2. `WorkerCredential` has `UUID caseId` — hard-coded to the engine's case concept.
   Non-engine services cannot scope credentials to their own resource types.
3. `WorkerPermissionRequest` has `caseDefinitionId` — engine-specific field that
   other domains would need to repurpose or ignore.
4. The `WorkerCredentialFilter` is in `engine-rest`, not reusable by other services
   that receive worker requests.

This spec generalizes these types so that any domain can use the worker rights
infrastructure without depending on engine concepts.

## 2. Design Decisions

| # | Decision | Choice |
|---|----------|--------|
| D1 | WorkerAction extensibility | Record-based value type |
| D2 | Credential resource scope | `ResourceId` value type replaces `UUID caseId` |
| D3 | Authorization context | Marker interface `WorkerAuthorizationContext` |
| D4 | Filter architecture | Single reusable filter + `WorkerScopeExtractor` SPI |
| D5 | Module naming | `acl-worker/` (capability-focused, consistent with `acl-inmem/`, `acl-jpa/`) |

## 3. Type Changes in platform-api

### 3.1 WorkerAction — enum to record

**Before:**
```java
public enum WorkerAction {
    READ_CONTEXT(AclAction.READ),
    WRITE_CONTEXT(AclAction.WRITE),
    SIGNAL_CASE(AclAction.WRITE),
    READ_EVENT_LOG(AclAction.READ),
    READ_PLAN_ITEMS(AclAction.READ),
    SPAWN_SUB_CASE(AclAction.WRITE),
    CLAIM_WORK_ITEM(AclAction.CLAIM),
    ADMIN(AclAction.ADMIN);
    // ...
}
```

**After:**
```java
public record WorkerAction(String name, AclAction aclAction) {

    public WorkerAction {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("name must not be blank");
        }
        if (aclAction == null) {
            throw new IllegalArgumentException("aclAction must not be null");
        }
    }
}
```

Equality is by both fields (record default). The constants pattern ensures
each domain defines consistent name-to-AclAction pairings. A `Set<WorkerAction>`
on a credential naturally deduplicates by value — two entries with the same
name but different AclAction are distinct and would indicate a domain bug.

Jackson serialization: records serialize naturally as
`{"name":"READ_CONTEXT","aclAction":"READ"}`. Deserialization reconstructs
the full record — no registry needed because the AclAction is stored alongside
the name.

Domains define their own constants:

```java
// In casehub-engine-api (engine-specific vocabulary)
public final class EngineWorkerActions {
    public static final WorkerAction READ_CONTEXT =
        new WorkerAction("READ_CONTEXT", AclAction.READ);
    public static final WorkerAction WRITE_CONTEXT =
        new WorkerAction("WRITE_CONTEXT", AclAction.WRITE);
    public static final WorkerAction SIGNAL_CASE =
        new WorkerAction("SIGNAL_CASE", AclAction.WRITE);
    public static final WorkerAction READ_EVENT_LOG =
        new WorkerAction("READ_EVENT_LOG", AclAction.READ);
    public static final WorkerAction READ_PLAN_ITEMS =
        new WorkerAction("READ_PLAN_ITEMS", AclAction.READ);
    public static final WorkerAction SPAWN_SUB_CASE =
        new WorkerAction("SPAWN_SUB_CASE", AclAction.WRITE);
    public static final WorkerAction CLAIM_WORK_ITEM =
        new WorkerAction("CLAIM_WORK_ITEM", AclAction.CLAIM);
    public static final WorkerAction ADMIN =
        new WorkerAction("ADMIN", AclAction.ADMIN);

    private EngineWorkerActions() {}
}
```

YAML parsing: `CaseDefinitionYamlMapper` maps kebab-case action names to
`EngineWorkerActions` constants by converting to upper-case and replacing
hyphens with underscores, then looking up via a static map.

### 3.2 ResourceId — new value type

```java
public record ResourceId(String type, String id) {

    public ResourceId {
        if (type == null || type.isBlank()) {
            throw new IllegalArgumentException("type must not be blank");
        }
        if (type.contains(":")) {
            throw new IllegalArgumentException("type must not contain ':'");
        }
        if (id == null || id.isBlank()) {
            throw new IllegalArgumentException("id must not be blank");
        }
    }

    @Override
    public String toString() {
        return type + ":" + id;
    }

    public static ResourceId parse(String value) {
        int colon = value.indexOf(':');
        if (colon <= 0 || colon == value.length() - 1) {
            throw new IllegalArgumentException(
                "ResourceId must be 'type:id', got: " + value);
        }
        return new ResourceId(value.substring(0, colon), value.substring(colon + 1));
    }
}
```

Jackson serialization: `@JsonValue` on `toString()`, `@JsonCreator` on `parse()`.
Serialized form: `"case:a1b2c3d4"`.

Package: `io.casehub.platform.api.acl`.

Follow-up (separate issue): retrofit `ResourceId` into `AccessControlProvider`
parameters to replace raw `String resourceId`.

### 3.3 WorkerCredential — generalized

**Before:**
```java
public record WorkerCredential(
    String token, String actorId, UUID caseId, String tenancyId,
    Set<WorkerAction> actions, Instant expiresAt, Instant createdAt) { ... }
```

**After:**
```java
public record WorkerCredential(
    String token,
    String actorId,
    ResourceId resourceId,
    String tenancyId,
    Set<WorkerAction> actions,
    Instant expiresAt,
    Instant createdAt) {

    public boolean isExpired() {
        return Instant.now().isAfter(expiresAt);
    }
}
```

Engine constructs: `new ResourceId(AclResourceType.CASE, caseId.toString())`

### 3.4 WorkerAuthorizationContext — new marker interface

```java
public interface WorkerAuthorizationContext {}
```

Package: `io.casehub.platform.api.acl`.

Engine defines:
```java
public record EngineAuthorizationContext(String caseDefinitionId)
    implements WorkerAuthorizationContext {}
```

### 3.5 WorkerPermissionRequest — generalized

**Before:**
```java
public record WorkerPermissionRequest(
    String actorId, String resourceType, Set<WorkerAction> workerActions,
    String caseDefinitionId, String tenancyId) {}
```

**After:**
```java
public record WorkerPermissionRequest(
    String actorId,
    String resourceType,
    Set<WorkerAction> actions,
    WorkerAuthorizationContext context,
    String tenancyId) {}
```

### 3.6 WorkerAuthorizationPolicy — simplified

**Before:**
```java
public interface WorkerAuthorizationPolicy {
    default AuthorizationDecision evaluate(WorkerPermissionRequest request) {
        return AuthorizationDecision.approve();
    }
}
```

**After:** Same signature — no change needed. The `WorkerPermissionRequest` now
carries `WorkerAuthorizationContext` (marker interface) instead of
`caseDefinitionId`. Domain-specific policies downcast the context:

```java
@ApplicationScoped
public class EngineWorkerAuthorizationPolicy implements WorkerAuthorizationPolicy {
    @Override
    public AuthorizationDecision evaluate(WorkerPermissionRequest request) {
        var ctx = (EngineAuthorizationContext) request.context();
        // use ctx.caseDefinitionId() for engine-specific policy logic
        return AuthorizationDecision.approve();
    }
}
```

### 3.7 Unchanged types

- `AuthorizationDecision` — no changes needed
- `WorkerAuthorizationDeniedException` — `caseDefinitionId` field renamed to
  `definitionId` (generic). The exception message uses whatever definition
  identifier the domain provides.
- `WorkerCredentialStore` — method signatures use `ResourceId` instead of
  `UUID caseId`:

```java
public interface WorkerCredentialStore {
    default void store(WorkerCredential credential) {}
    default Optional<WorkerCredential> lookup(String token) { return Optional.empty(); }
    default void revoke(String token) {}
    default List<WorkerCredential> revokeByResource(ResourceId resourceId) { return List.of(); }
    default List<WorkerCredential> revokeByActor(String actorId) { return List.of(); }
    default List<WorkerCredential> findActiveByActorAndResource(
        String actorId, ResourceId resourceId) { return List.of(); }
}
```

Note: `revokeByCase` → `revokeByResource`, `findActiveByActorAndCase` →
`findActiveByActorAndResource`.

## 4. New Module: acl-worker/

### 4.1 Module structure

```
acl-worker/
├── pom.xml
└── src/main/java/io/casehub/platform/acl/worker/
    ├── WorkerCredentialFilter.java
    ├── WorkerScopeExtractor.java
    └── NoOpWorkerScopeExtractor.java
```

Artifact: `casehub-platform-acl-worker`

Dependencies:
- `casehub-platform-api` (compile)
- `jakarta.ws.rs-api` (compile — for `ContainerRequestFilter`)
- `quarkus-arc` (compile — for CDI annotations)

### 4.2 WorkerScopeExtractor SPI

```java
public interface WorkerScopeExtractor {
    Optional<ResourceId> extractResourceId(ContainerRequestContext ctx);
}
```

Returns `Optional.empty()` when the request URL does not contain a
resource reference (scope validation skipped for that request).
Returns `Optional.of(resourceId)` when a resource is identified —
the filter compares it to the credential's `resourceId`.

### 4.3 FailClosedWorkerScopeExtractor (default)

```java
@DefaultBean
@ApplicationScoped
public class FailClosedWorkerScopeExtractor implements WorkerScopeExtractor {
    private static final ResourceId NEVER_MATCH =
        new ResourceId("__deny__", "__no_scope_extractor_configured__");

    @Override
    public Optional<ResourceId> extractResourceId(ContainerRequestContext ctx) {
        return Optional.of(NEVER_MATCH);
    }
}
```

Default: fail-closed. Any request with a worker credential is rejected (403)
unless a real `WorkerScopeExtractor` is provided. This prevents accidental
cross-resource access when a service adds `acl-worker` without configuring
scope extraction.

Services that genuinely want scope-free worker auth provide an explicit
`PassthroughWorkerScopeExtractor` that returns `Optional.empty()` — making
the insecure choice deliberate, not accidental.

### 4.4 WorkerCredentialFilter

```java
@Provider
@Priority(Priorities.AUTHENTICATION - 10)
public class WorkerCredentialFilter implements ContainerRequestFilter {

    private final WorkerCredentialStore credentialStore;
    private final WorkerScopeExtractor scopeExtractor;

    private final CurrentPrincipal currentPrincipal;

    @Inject
    public WorkerCredentialFilter(
        WorkerCredentialStore credentialStore,
        WorkerScopeExtractor scopeExtractor,
        CurrentPrincipal currentPrincipal) { ... }

    @Override
    public void filter(ContainerRequestContext ctx) {
        String token = ctx.getHeaderString("X-Worker-Credential");
        if (token == null) return;

        var credential = credentialStore.lookup(token);
        if (credential.isEmpty()) {
            ctx.abortWith(Response.status(401).entity("Invalid worker credential").build());
            return;
        }

        var cred = credential.get();
        if (cred.isExpired()) {
            ctx.abortWith(Response.status(401).entity("Worker credential expired").build());
            return;
        }

        String requestTenancy = currentPrincipal.tenancyId();
        if (!cred.tenancyId().equals(requestTenancy)) {
            ctx.abortWith(Response.status(403)
                .entity("Credential not scoped for this tenant").build());
            return;
        }

        var requestResource = scopeExtractor.extractResourceId(ctx);
        if (requestResource.isPresent()
            && !cred.resourceId().equals(requestResource.get())) {
            ctx.abortWith(Response.status(403)
                .entity("Credential not scoped for this resource").build());
            return;
        }

        ctx.setProperty("workerCredential", cred);
    }
}
```

Behavior:
- No `X-Worker-Credential` header → pass through (normal auth)
- Token not found → 401
- Token expired → 401
- Tenancy mismatch → 403
- Scope extractor returns a resource ID that doesn't match credential → 403
- Scope extractor returns empty → scope validation skipped (token-only mode)
- Valid → sets full `WorkerCredential` as a single typed request property

## 5. Implementation Changes in platform modules

### 5.1 NoOpWorkerCredentialStore (platform/)

Adapt method signatures: `revokeByCase(UUID)` → `revokeByResource(ResourceId)`,
`findActiveByActorAndCase` → `findActiveByActorAndResource`. No behavioral change —
still returns empty/no-op.

### 5.2 InMemoryWorkerCredentialStore (acl-inmem/)

Adapt to `ResourceId`:
- Internal map keyed by `String token` — unchanged
- `revokeByResource(ResourceId)` filters by `credential.resourceId().equals(resourceId)`
- `findActiveByActorAndResource` filters by actorId + resourceId

### 5.3 InMemoryWorkerCredentialStoreTest

Update test fixtures to construct `WorkerCredential` with `ResourceId` and
record-based `WorkerAction`.

## 6. Cross-repo impact (engine — separate branch)

The engine changes are not in this branch's scope. They are documented here
for completeness and should be tracked as a follow-up issue on `casehubio/engine`.

### 6.1 EngineWorkerActions constants class

New class in `casehub-engine-api` (or `casehub-engine-common`) — replaces
direct `WorkerAction` enum references throughout engine.

### 6.2 WorkerGrantOrchestrator

- Constructs `ResourceId` as `new ResourceId(AclResourceType.CASE, caseId.toString())`
- Constructs `EngineAuthorizationContext(caseDefinitionId)` for policy evaluation
- Uses `resourceId.toString()` when calling `AccessControlProvider` (until
  `AccessControlProvider` is retrofitted with `ResourceId`)
- `revokeForCase(UUID)` calls `revokeByResource(new ResourceId("case", caseId.toString()))`

### 6.3 WorkerCredentialFilter (engine-rest) — deleted

Replaced by the platform's `acl-worker` filter. Engine adds `acl-worker`
as a compile dependency and provides `CaseScopeExtractor`:

```java
@ApplicationScoped
public class CaseScopeExtractor implements WorkerScopeExtractor {
    private static final Pattern CASE_ID_PATTERN =
        Pattern.compile("cases/([0-9a-f-]{36})");

    @Override
    public Optional<ResourceId> extractResourceId(ContainerRequestContext ctx) {
        Matcher matcher = CASE_ID_PATTERN.matcher(ctx.getUriInfo().getPath());
        if (matcher.find()) {
            return Optional.of(new ResourceId(AclResourceType.CASE, matcher.group(1)));
        }
        return Optional.empty();
    }
}
```

### 6.4 YAML parsing

`CaseDefinitionYamlMapper` converts kebab-case `permissionIntent` values to
`WorkerAction` records via a lookup map keyed by name, populated from
`EngineWorkerActions` constants.

## 7. Why worker credentials are not a general credential framework

Worker credentials are a specific mechanism: runtime-issued, resource-scoped,
action-constrained, short-lived tokens. Other non-user auth mechanisms in the
platform serve different purposes:

| Mechanism | Purpose | Lifecycle |
|-----------|---------|-----------|
| Worker credentials | Case-scoped worker authorization | Runtime: mint at dispatch, revoke at completion |
| `CredentialResolver` | Static secret resolution (API keys, passwords) | Config-time: read from vault/config |
| OIDC / JWT (`CurrentPrincipal`) | User/service identity | Session: issued by IdP, validated per request |
| Webhook auth | Endpoint authentication | Registration-time: stored with endpoint descriptor |

A general credential validation framework would over-abstract these distinct
mechanisms. Worker credentials warrant their own infrastructure because their
lifecycle (mint → validate → revoke) and scope (resource-bound) are unique.

## 8. Scope Boundaries

### In scope (this branch — platform only)

- `WorkerAction` enum → record in platform-api
- `ResourceId` value type in platform-api
- `WorkerAuthorizationContext` marker interface in platform-api
- `WorkerCredential` generalized (ResourceId, record-based actions) in platform-api
- `WorkerPermissionRequest` generalized in platform-api
- `WorkerCredentialStore` method signature updates in platform-api
- `NoOpWorkerCredentialStore` adapted in platform/
- `InMemoryWorkerCredentialStore` adapted in acl-inmem/
- New `acl-worker/` module with filter + ScopeExtractor SPI
- Tests for all changed and new types
- CLAUDE.md updates (package structure, module list)
- capability-ownership.md correction (worker rights → platform, not engine)

### Not in scope (follow-up issues — GitHub issues created during implementation)

- Engine migration to new types (casehubio/engine issue)
- `ResourceId` retrofit into `AccessControlProvider` (casehubio/platform issue)
- `AclResourceType` engine-specific constants extraction (same problem as `WorkerAction`
  — constants like `CASE`, `PLAN_ITEM`, `EVENT_LOG` should move to engine-api, with
  platform-api retaining only the type or an empty class; casehubio/platform issue)
- Persistent `WorkerCredentialStore` implementations (consumer-provided)

## 9. Testing Strategy

| Layer | Approach |
|-------|----------|
| `WorkerAction` record | Unit — equality, serialization, domain constants |
| `ResourceId` | Unit — construction validation, parse, toString, Jackson round-trip |
| `WorkerCredential` | Unit — isExpired, construction with ResourceId |
| `WorkerCredentialStore` | Unit — method signature compatibility |
| `InMemoryWorkerCredentialStore` | Unit — store/lookup/revoke/revokeByResource lifecycle |
| `WorkerCredentialFilter` | Unit — token lookup, expiry, tenancy validation, scope enforcement via mock ScopeExtractor, fail-closed default, passthrough |
| `FailClosedWorkerScopeExtractor` | Unit — always returns sentinel ResourceId (never matches) |
| `WorkerPermissionRequest` | Unit — construction with WorkerAuthorizationContext |
