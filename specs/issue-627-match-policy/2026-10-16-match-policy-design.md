# Design: MatchPolicy for KeyLookupStrategy

**Issue:** #627
**Date:** 2026-10-16
**Status:** Draft

## Problem

`KeyLookupStrategy` returns the first matching record when multiple corpus
entries share the same key (`findFirst()` on the stream). This is accidental
behavior — no test or contract protects it. Multiple records per key
accumulate in the corpus but only the first is ever retrieved, making the
rest dead data.

This prevents simulating multi-turn conversations where each invocation
should return the next scripted response. `SequentialStrategy` exists for
input-agnostic sequential iteration, but there is no key-scoped sequential
resolution.

## Solution

Add a `MatchPolicy` enum to control how `KeyLookupStrategy` resolves
multiple records sharing the same key. Add `STICKY_LAST` to
`ExhaustionPolicy` for conversation-safe exhaustion.

## Strategy Landscape (after change)

| Strategy | Key? | Multi-record behavior |
|----------|------|-----------------------|
| `SequentialStrategy` | No | Iterates all records in order, ignores input |
| `KeyLookupStrategy` + `FIRST` | Yes | First match per key (default, backward-compat) |
| `KeyLookupStrategy` + `RANDOM` | Yes | Random pick from matching records |
| `KeyLookupStrategy` + `SEQUENTIAL` | Yes | Per-key cursor, configurable exhaustion |

## Module Changes

### simulation-api

**New enum: `MatchPolicy`**
Package: `io.casehub.platform.simulation`

```java
public enum MatchPolicy {
    FIRST,      // return first matching record (default)
    RANDOM,     // random selection from matching records
    SEQUENTIAL  // cursor-based iteration, per-key state
}
```

**ExhaustionPolicy: add `STICKY_LAST`**

```java
public enum ExhaustionPolicy {
    WRAP,        // cycle back to first record
    THROW,       // throw when exhausted
    STICKY_LAST  // repeat last record forever
}
```

`STICKY_LAST` is available to both `KeyLookupStrategy` (SEQUENTIAL mode)
and `SequentialStrategy`. Safe default for conversations.

**SimulationCorpus: add `lookupAllByKey` default method**

```java
default List<O> lookupAllByKey(String qualifiedName, String key) {
    return list(qualifiedName).stream()
            .filter(r -> key.equals(r.key()))
            .map(InvocationRecord::output)
            .toList();
}
```

Default implementation filters `list()` by key. Non-breaking — existing
implementations inherit it. `InMemorySimulationCorpus` overrides for
efficiency.

### simulation-core

**KeyLookupStrategy enhancement**

New fields:
- `MatchPolicy matchPolicy` (default: `FIRST`)
- `ExhaustionPolicy exhaustionPolicy` (default: `STICKY_LAST`, only used for SEQUENTIAL)
- `ConcurrentHashMap<String, AtomicInteger> cursors` (lazy, only created for SEQUENTIAL)

Constructor changes:
- Existing 3-arg constructor `(corpus, qualifiedName, keyExtractor)` preserved — defaults to `FIRST`
- New 5-arg constructor adds `matchPolicy` and `exhaustionPolicy`

`resolve()` behavior by MatchPolicy:

- **FIRST:** `corpus.lookupByKey(qn, key)` → first match or throw (unchanged)
- **RANDOM:** `corpus.lookupAllByKey(qn, key)` → `ThreadLocalRandom` pick, throw if empty
- **SEQUENTIAL:** `corpus.lookupAllByKey(qn, key)` → index by cursor, advance cursor
  - Cursor key: `qualifiedName + ":" + extractedKey`
  - STICKY_LAST: clamp index to `size - 1`
  - WRAP: `index % size`
  - THROW: throw `SimulationKeyNotFoundException` when `index >= size`

`canResolve()`: unchanged — checks `lookupByKey` presence (at least one record exists).

`dataRealism()`: unchanged — `DOMAIN_PLAUSIBLE`.

**SequentialStrategy: add STICKY_LAST support**

Current code uses `index % size` for WRAP and throws for THROW. Add
STICKY_LAST: clamp to `size - 1` when `index >= size`. Small addition
to existing switch/if.

### simulation-inmem

**InMemorySimulationCorpus: override `lookupAllByKey`**

```java
@Override
public List<O> lookupAllByKey(String qualifiedName, String key) {
    var records = data.get(qualifiedName);
    if (records == null) return List.of();
    return records.stream()
            .filter(r -> key.equals(r.key()))
            .map(InvocationRecord::output)
            .toList();
}
```

Direct stream filter on the internal `CopyOnWriteArrayList` — avoids
the default method's `list()` which copies all records.

### simulation-config-core

**YamlSimulationConfig: add `match` field**

Add `match` to `MethodConfig` record. Parsed from YAML as string,
mapped to `MatchPolicy` enum (case-insensitive).

When creating `KeyLookupStrategy` from config:
- `match` absent or `first` → `MatchPolicy.FIRST`
- `match: random` → `MatchPolicy.RANDOM`
- `match: sequential` → `MatchPolicy.SEQUENTIAL`
  - If `exhaustion-policy` not set, default to `STICKY_LAST`
  - If `exhaustion-policy` set, use that value

YAML example:

```yaml
strategies:
  io.casehub.platform.agent.AgentSession.query:
    strategy: key-lookup
    key-extractor: sessionId
    match: sequential
    exhaustion-policy: sticky-last
    corpus:
      - key: "order-help"
        output: "What's your order number?"
      - key: "order-help"
        output: "Found it. Here's the status."
      - key: "order-help"
        output: "Anything else I can help with?"
```

## Thread Safety

- `ConcurrentHashMap<String, AtomicInteger>` for cursors — lock-free,
  virtual-thread safe
- `AtomicInteger.getAndIncrement()` for cursor advancement — atomic
- `ThreadLocalRandom` for RANDOM — no contention
- Cursor map only created when `matchPolicy == SEQUENTIAL` — no overhead
  for FIRST/RANDOM

## Backward Compatibility

- All existing configs unchanged — `match` defaults to `FIRST`
- Existing 3-arg `KeyLookupStrategy` constructor preserved
- `ExhaustionPolicy.STICKY_LAST` is additive — no existing code uses it
- `lookupAllByKey` is a default method — no implementor changes required
- `SequentialStrategy` gains STICKY_LAST support but WRAP/THROW unchanged

## Testing

- `KeyLookupStrategyTest`: FIRST returns same record always, RANDOM returns
  varying records, SEQUENTIAL advances through records in order
- SEQUENTIAL exhaustion: STICKY_LAST repeats last, WRAP cycles, THROW throws
- Single-record per key: all three policies behave identically
- Empty key: throws `SimulationKeyNotFoundException` regardless of policy
- Thread-safety: concurrent SEQUENTIAL resolve produces monotonic advancement
- `SequentialStrategy` STICKY_LAST: existing tests + new sticky-last case
- `InMemorySimulationCorpus`: lookupAllByKey returns all records in
  insertion order, empty list for missing key
- YAML config parsing: `match` field maps correctly to MatchPolicy

## References

- `simulation-core/.../KeyLookupStrategy.java` — current first-match impl
- `simulation-core/.../SequentialStrategy.java` — AtomicInteger cursor pattern
- `simulation-api/.../ExhaustionPolicy.java` — WRAP/THROW enum
- `simulation-api/.../SimulationCorpus.java` — lookupByKey SPI
- `simulation-inmem/.../InMemorySimulationCorpus.java` — CopyOnWriteArrayList storage
- `simulation-config-core/.../YamlSimulationConfig.java` — MethodConfig parsing
