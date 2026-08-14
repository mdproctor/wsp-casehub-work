# HANDOFF — 2026-08-14

## Last Session

SPI extraction (#337) merged to main. JPQL entity names fixed, assignment strategies migrated to record API, OCC version restored in REST responses, spawn service updated to persist strategy modifications. Work CI still red — 5 queues-examples scenario tests fail (inferred labels empty). Peer repos: devtown GREEN, engine GREEN, aml/clinical/iot still RED (need more WorkItemEntity→WorkItem fixes). life upstream push blocked by Flyway rebase conflicts.

## The Remaining Bug — Inferred Labels Empty

### Symptom
5 queues-examples scenario tests fail. All test label filter rules that should produce INFERRED labels on WorkItem creation. The response shows `inferredLabels: []`.

### What Changed
Last green CI: `5591128f` (Aug 8). Compare production code:
```bash
git diff 5591128f..HEAD -- runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java
```

The `create()` method changed from entity-mutation to record-builder:

**Before (entity, working):**
```java
public WorkItem create(WorkItemCreateRequest request) {
    final WorkItem item = new WorkItem();  // JPA entity
    item.status = PENDING;
    item.title = request.title;
    // ... set 30+ fields directly ...
    item = assignmentService.assign(item, CREATED);
    final WorkItem saved = workItemStore.put(item);
    lifecycleEmitter.emit(WorkItemLifecycleEvent.of("CREATED", saved, ...));
    return saved;  // <-- same Hibernate-managed object the observer mutated
}
```

**After (record, broken):**
```java
public WorkItem create(WorkItemCreateRequest request) {
    WorkItem item = WorkItem.builder()
        .status(PENDING).title(request.title)
        // ... builder fields ...
        .build();
    item = assignmentService.assign(item, CREATED);
    final WorkItem saved = workItemStore.put(item);
    lifecycleEmitter.emit(WorkItemLifecycleEvent.of("CREATED", saved, ...));
    return workItemStore.get(saved.id()).orElse(saved);  // <-- re-read attempt
}
```

### The Core Tension: Shared Mutable State vs Immutable Snapshots

The old code relied on Hibernate's L1 cache sharing. The entity returned by `create()` was the SAME Java object as the one in the Hibernate session. When `FilterEvaluationObserver` (synchronous `@Observes`) loaded and modified the entity via `workItemStore.put()`, it modified the same managed object. The caller's reference automatically reflected the labels.

With immutable records, each `put()` returns a frozen snapshot. The observer creates a separate snapshot with labels. The caller's snapshot is stale. The re-read attempt (`workItemStore.get()`) should fix this, but the test still fails.

### Theory 1: Re-read hits stale L1 cache
`JpaWorkItemStore.get()` calls `WorkItemEntity.findById(id)` which returns the entity from Hibernate's L1 cache. The observer's `put()` called `updateEntity()` on that same managed entity (adding labels). So `get()` should return the entity WITH labels, and `toDomain()` should produce a record with labels. If this theory is wrong, the labels should be there. **Test: add a breakpoint in `get()` and check the entity's labels list.**

### Theory 2: Observer isn't firing
The `FilterEvaluationObserver` uses `@Observes` (synchronous). The `WorkItemLifecycleEmitter` calls `delegate.fire(event)` (synchronous) then `delegate.fireAsync(event)`. The synchronous fire should invoke the observer before `create()` returns. **Test: add `LOG.info("observer fired")` at the top of `FilterEvaluationObserver.onLifecycleEvent()` and check test output.**

### Theory 3: Observer fires but label rules return empty
The observer calls `labelRuleEngine.evaluate()` which queries `LabelRuleStore` for rules. If the rules aren't visible (wrong tenancy, not committed yet, test setup timing), the engine finds no rules and adds no labels. **Test: add logging inside `LabelRuleEngine.evaluate()` to print the number of rules found.**

### Theory 4: Observer's put() creates a detached entity
The observer does:
```java
WorkItemEntity entity = WorkItemEntityMapper.toEntity(wi);  // NEW entity, not managed
labelRuleEngine.evaluate(entity, context, eventType);        // adds labels to new entity
WorkItem updated = WorkItemEntityMapper.toDomain(entity);    // record with labels
workItemStore.put(updated);                                   // should update managed entity
```
The `put()` calls `findById(id)` → gets MANAGED entity from L1. Calls `updateEntity(managed, updated)` → copies labels from record to managed entity. Entity is dirty. BUT: does `updateEntity()` correctly handle the labels collection? It replaces `entity.labels = new ArrayList<>(...)`. Hibernate sees a collection replacement. For `@ElementCollection`, replacing the collection is valid but Hibernate tracks by identity — if the old collection was a PersistentBag, replacing it might cause issues. **Test: check if `updateEntity()` replaces or modifies the existing labels collection.**

### Recommended Debugging Approach
1. Run `LegalRoutingScenarioTest` with a debugger
2. Set breakpoints at:
   - `FilterEvaluationObserver.onLifecycleEvent()` line 38 — does it fire?
   - `LabelRuleEngine.evaluate()` — how many rules found?
   - `WorkItemEntityMapper.updateEntity()` — are labels being copied to the managed entity?
   - `WorkItemService.create()` return — does the re-read include labels?
3. The first breakpoint that doesn't hit reveals where the chain breaks

### Files to Compare
```bash
# The full production diff since last green
git diff 5591128f..HEAD -- runtime/src/main/java/

# Key files
git diff 5591128f..HEAD -- runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java
git diff 5591128f..HEAD -- runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java
git diff 5591128f..HEAD -- runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemEntityMapper.java
git diff 5591128f..HEAD -- queues/src/main/java/io/casehub/work/queues/service/FilterEvaluationObserver.java
```

## Peer Repo CI Status

| Repo | CI | Issue |
|------|-----|-------|
| work | RED | 5 label-filter scenario tests |
| devtown | GREEN | — |
| engine | GREEN | — |
| aml | RED | ~20 WorkItemEntity→WorkItem type mismatches in production code |
| clinical | RED | ~8 similar |
| iot | RED | ~4 similar |
| life | not pushed to upstream | Flyway migration rebase conflict |

## Stale State to Clean Up

- **engine repo**: branch `issue-903-goal-revision-action-enum` has a stale commit `0ceb4a26` (duplicate of a fix already on main). 6 stashes, 2 from this session.
- **work repo**: branch `issue-510-case-level-sla-timer` had a stale commit (removed via `reset --hard`). 3 old stashes (not from this session).

## References

| Artifact | Path |
|----------|------|
| Last green SHA | `5591128f` |
| Design spec | `specs/issue-333-progress-api-docs-spi-fix/2026-08-13-progress-docs-spi-extraction-design.md` |
| WorkItemEntityMapper | `runtime/src/main/java/io/casehub/work/runtime/repository/WorkItemEntityMapper.java` |
| FilterEvaluationObserver | `queues/src/main/java/io/casehub/work/queues/service/FilterEvaluationObserver.java` |
| LabelRuleEngine | `runtime/src/main/java/io/casehub/work/runtime/filter/LabelRuleEngine.java` |
| Forage: IntelliJ collateral | `~/.claude/garden/entries/GE-20260813-intellij-rename-collateral.md` |
