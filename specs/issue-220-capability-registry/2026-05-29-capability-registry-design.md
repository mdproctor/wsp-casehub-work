# Capability Registry Design

**Issue:** casehubio/work#220
**Branch:** issue-220-capability-registry
**Date:** 2026-05-29

## Problem

Engine `CaseDefinition` capability names and worker capability tags in `WorkerCandidate.capabilities` are both `String`, matched by `WorkBroker` using exact case-sensitive equality. There is no shared vocabulary, no validation, and no documentation of the matching contract. A mismatch (e.g. engine: `"legal-review"`, worker: `"legal_review"`) causes silent routing failure — `WorkOrchestrator` throws `IllegalStateException("No worker available for capability: legal-review")` with no diagnostic pointing at the vocabulary gap.

## Deliverables

Two deliverables in sequence:

1. **Short-term:** `WorkCapabilities` constants class in `casehub-work-api` — well-known string constants importable by engine case definitions and worker registrations.
2. **Long-term:** `CapabilityRegistry` SPI in `casehub-work-api` with creation-time validation wired into `WorkItemService`.

## SPI and Types (`casehub-work-api`)

### `CapabilityRegistry`

```java
public interface CapabilityRegistry {
    Set<String> capabilities();
    default ValidationMode validationMode() { return ValidationMode.PERMISSIVE; }
    default boolean isKnown(String tag) { return capabilities().contains(tag); }
}
```

`isKnown()` is a default interface method that always delegates to `capabilities()`. Implementors override `capabilities()` only — never `isKnown()` backed by a static field, which would silently bypass subclass overrides (GE-20260511-a5f47d).

### `ValidationMode`

```java
public enum ValidationMode { STRICT, WARN, PERMISSIVE }
```

- `PERMISSIVE` (default) — no check; identical behaviour to today
- `WARN` — logs unknown tags, proceeds
- `STRICT` — rejects creation with `UnknownCapabilityException` (maps to 400)

### `UnknownCapabilityException`

Unchecked. Carries the list of unknown tags. Mapped to `400` by `UnknownCapabilityExceptionMapper` in `runtime`.

### `WorkCapabilities`

```java
public final class WorkCapabilities {
    public static final String LEGAL_REVIEW      = "legal-review";
    public static final String CONTRACT_ANALYSIS = "contract-analysis";
    // ... populated from examples module usage
    private WorkCapabilities() {}
}
```

Populated from the capabilities already used in the `examples` module. Engine case definitions and worker registrations import from here rather than duplicating strings.

## Default Implementation (`casehub-work-core`)

```java
@DefaultBean
@ApplicationScoped
public class PermissiveCapabilityRegistry implements CapabilityRegistry {
    @Override public Set<String> capabilities() { return Set.of(); }
    @Override public ValidationMode validationMode() { return ValidationMode.PERMISSIVE; }
}
```

`@DefaultBean` means any application-scoped `CapabilityRegistry` in the deploying app displaces it automatically — no `@Alternative @Priority(1)` required. Existing deployments see no behaviour change.

Follows the same module placement pattern as `NoOpWorkerRegistry`: SPI interface in `api/`, CDI default in `core/`.

## Validator Service (`casehub-work-core`)

```java
@ApplicationScoped
public class CapabilityValidator {

    @Inject CapabilityRegistry registry;

    public void validate(String commaSeparatedCapabilities) {
        if (registry.validationMode() == ValidationMode.PERMISSIVE
                || commaSeparatedCapabilities == null
                || commaSeparatedCapabilities.isBlank()) {
            return;
        }
        List<String> unknown = Arrays.stream(commaSeparatedCapabilities.split(","))
                .map(String::trim).filter(s -> !s.isEmpty())
                .filter(tag -> !registry.isKnown(tag))
                .toList();
        if (unknown.isEmpty()) return;

        if (registry.validationMode() == ValidationMode.STRICT) {
            throw new UnknownCapabilityException(unknown);
        } else {
            Log.warnf("WorkItem references unregistered capabilities: %s", unknown);
        }
    }
}
```

PERMISSIVE short-circuits immediately — zero overhead on the default path.

## Creation-Path Wiring (`runtime`)

`CapabilityValidator` is injected into `WorkItemService` and called at the top of `create()` before any entity construction:

```java
@Inject CapabilityValidator capabilityValidator;

@Transactional
public WorkItem create(final WorkItemCreateRequest request) {
    capabilityValidator.validate(request.requiredCapabilities);
    // ... existing code unchanged
}
```

`WorkItemService.create()` is the single chokepoint — spawn (`WorkItemSpawnService`), multi-instance (`MultiInstanceSpawnService`), and template instantiation (`WorkItemTemplateService`) all build a `WorkItemCreateRequest` and pass through here.

If `requiredCapabilities` is also mutable via PATCH, the same `validate()` call guards that path.

`UnknownCapabilityExceptionMapper` in `runtime` maps `UnknownCapabilityException` to a `400` response with the unknown tag names in the body.

## Scoping

Scoping (per-domain policies, per-tenant registries) is out of scope. The SPI is a CDI `@Alternative` — teams can compose their own registry that delegates to sub-registries or scopes by prefix without changes to the SPI contract.

Startup validation of worker-declared capabilities is also out of scope. `WorkerCandidate.capabilities` is resolved lazily per-group by `WorkerRegistry` — there is no static worker capability declaration mechanism today. That requires a separate issue.

## Testing

| Test | Module | Type |
|------|--------|------|
| `CapabilityRegistryTest` | `api` | Unit — `isKnown()` delegates to `capabilities()`; GE-20260511-a5f47d guard |
| `PermissiveCapabilityRegistryTest` | `core` | Unit — PERMISSIVE, empty set, validate is no-op |
| `CapabilityValidatorTest` | `core` | Unit — all three modes; null/blank always pass; unknown tags named in exception |
| `WorkItemServiceCapabilityIT` | `runtime` | `@QuarkusTest` — STRICT registry via `@Alternative @Priority(1)` test profile; 201 for known, 400 for unknown |
