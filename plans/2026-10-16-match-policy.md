# MatchPolicy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #627 — Add MatchPolicy to KeyLookupStrategy for multi-record resolution

**Goal:** Enable KeyLookupStrategy to resolve multiple corpus records per key via configurable MatchPolicy (FIRST, RANDOM, SEQUENTIAL).

**Architecture:** Add MatchPolicy enum to simulation-api alongside ExhaustionPolicy. Enhance KeyLookupStrategy with per-key cursor state for SEQUENTIAL mode. Add CLAMP to ExhaustionPolicy. Wire through SimulationConfig → SimulationRuntime → YAML config.

**Tech Stack:** Java 21, JUnit 5, AssertJ

## Global Constraints

- simulation-api must remain zero-dependency (pure Java only)
- simulation-core depends on simulation-api — no reverse dependency
- simulation-config-core depends on simulation-api — not on simulation-core
- All new primitives must be thread-safe via j.u.c (no `synchronized`)
- Existing 3-arg `KeyLookupStrategy` constructor must be preserved

---

## Batch 1: API Foundation (simulation-api)

### Task 1: Add MatchPolicy enum, CLAMP to ExhaustionPolicy, lookupAllByKey default method

**Files:**
- Create: `simulation-api/src/main/java/io/casehub/platform/simulation/MatchPolicy.java`
- Modify: `simulation-api/src/main/java/io/casehub/platform/simulation/ExhaustionPolicy.java`
- Modify: `simulation-api/src/main/java/io/casehub/platform/simulation/SimulationCorpus.java`
- Test: `simulation-api/src/test/java/io/casehub/platform/simulation/MatchPolicyTest.java`

**Interfaces:**
- Produces: `MatchPolicy.FIRST`, `MatchPolicy.RANDOM`, `MatchPolicy.SEQUENTIAL`
- Produces: `ExhaustionPolicy.CLAMP`
- Produces: `SimulationCorpus.lookupAllByKey(String qualifiedName, String key) → List<O>`

- [ ] **Step 1: Create MatchPolicy enum**

```java
package io.casehub.platform.simulation;

public enum MatchPolicy {
    FIRST,
    RANDOM,
    SEQUENTIAL
}
```

- [ ] **Step 2: Add CLAMP to ExhaustionPolicy**

Modify `ExhaustionPolicy.java` — add `CLAMP` after `THROW`:

```java
public enum ExhaustionPolicy {
    WRAP,
    THROW,
    CLAMP
}
```

- [ ] **Step 3: Add lookupAllByKey default method to SimulationCorpus**

Add after the `lookupByKey` method (line 8):

```java
default List<O> lookupAllByKey(String qualifiedName, String key) {
    return list(qualifiedName).stream()
            .filter(r -> key.equals(r.key()))
            .map(InvocationRecord::output)
            .toList();
}
```

- [ ] **Step 4: Write tests for MatchPolicy values**

```java
package io.casehub.platform.simulation;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MatchPolicyTest {

    @Test
    void hasExpectedValues() {
        assertThat(MatchPolicy.values())
                .containsExactly(MatchPolicy.FIRST, MatchPolicy.RANDOM, MatchPolicy.SEQUENTIAL);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl simulation-api -Dtest=MatchPolicyTest`
Expected: PASS

- [ ] **Step 6: Verify existing ExhaustionPolicy tests still pass (if any) and compilation**

Run: `mvn --batch-mode test -pl simulation-api`
Expected: PASS (existing tests may fail if they assert exact enum values — fix by adding CLAMP to expected values)

- [ ] **Step 7: Commit**

```bash
git add simulation-api/src/main/java/io/casehub/platform/simulation/MatchPolicy.java
git add simulation-api/src/main/java/io/casehub/platform/simulation/ExhaustionPolicy.java
git add simulation-api/src/main/java/io/casehub/platform/simulation/SimulationCorpus.java
git add simulation-api/src/test/java/io/casehub/platform/simulation/MatchPolicyTest.java
git commit -m "feat(#627): add MatchPolicy enum, CLAMP exhaustion policy, lookupAllByKey default method"
```

---

## Batch 2: Corpus and Strategy Enhancement (simulation-inmem, simulation-core)

### Task 2: Override lookupAllByKey in InMemorySimulationCorpus

**Files:**
- Modify: `simulation-inmem/src/main/java/io/casehub/platform/simulation/inmem/InMemorySimulationCorpus.java`
- Modify: `simulation-inmem/src/test/java/io/casehub/platform/simulation/inmem/InMemorySimulationCorpusTest.java`

**Interfaces:**
- Consumes: `SimulationCorpus.lookupAllByKey(String, String) → List<O>` (from Task 1)
- Produces: Optimized `InMemorySimulationCorpus.lookupAllByKey` override

- [ ] **Step 1: Write failing test for lookupAllByKey**

Add to `InMemorySimulationCorpusTest.java`:

```java
@Test
void lookupAllByKeyReturnsAllMatchingOutputsInInsertionOrder() {
    corpus.record(QN, TENANT, "conv", "input1", "response-1");
    corpus.record(QN, TENANT, "conv", "input2", "response-2");
    corpus.record(QN, TENANT, "conv", "input3", "response-3");
    corpus.record(QN, TENANT, "other", "input4", "other-response");

    List<String> results = corpus.lookupAllByKey(QN, "conv");

    assertThat(results).containsExactly("response-1", "response-2", "response-3");
}

@Test
void lookupAllByKeyReturnsEmptyListForMissingKey() {
    corpus.record(QN, TENANT, "existing", "in", "out");

    List<String> results = corpus.lookupAllByKey(QN, "missing");

    assertThat(results).isEmpty();
}

@Test
void lookupAllByKeyReturnsEmptyListForMissingQualifiedName() {
    List<String> results = corpus.lookupAllByKey("nonexistent.method", "any");

    assertThat(results).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl simulation-inmem -Dtest=InMemorySimulationCorpusTest#lookupAllByKey*`
Expected: FAIL — default method works but returns unoptimized results. Actually these may pass via default method. Verify compilation first.

Run: `mvn --batch-mode test -pl simulation-inmem -Dtest=InMemorySimulationCorpusTest`
Expected: PASS (default method provides correct behavior)

- [ ] **Step 3: Override lookupAllByKey in InMemorySimulationCorpus**

Add after `lookupByKey` method (after line 26):

```java
@Override
public List<O> lookupAllByKey(final String qualifiedName, final String key) {
    final var records = store.get(qualifiedName);
    if (records == null) {
        return List.of();
    }
    return records.stream()
            .filter(r -> key.equals(r.key()))
            .map(InvocationRecord::output)
            .toList();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl simulation-inmem`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add simulation-inmem/src/main/java/io/casehub/platform/simulation/inmem/InMemorySimulationCorpus.java
git add simulation-inmem/src/test/java/io/casehub/platform/simulation/inmem/InMemorySimulationCorpusTest.java
git commit -m "feat(#627): override lookupAllByKey in InMemorySimulationCorpus"
```

### Task 3: Enhance KeyLookupStrategy with MatchPolicy

**Files:**
- Modify: `simulation-core/src/main/java/io/casehub/platform/simulation/strategy/KeyLookupStrategy.java`
- Modify: `simulation-core/src/test/java/io/casehub/platform/simulation/strategy/KeyLookupStrategyTest.java`

**Interfaces:**
- Consumes: `MatchPolicy` enum (from Task 1), `SimulationCorpus.lookupAllByKey` (from Task 1)
- Produces: `KeyLookupStrategy(corpus, qualifiedName, keyExtractor, matchPolicy, exhaustionPolicy)` constructor

- [ ] **Step 1: Write failing tests for SEQUENTIAL match policy**

Add to `KeyLookupStrategyTest.java`:

```java
@Test
void sequentialMatchIteratesThroughRecordsInOrder() {
    final var corpus = ListBackedCorpus.of(
            entry("conv", "in1", "first"),
            entry("conv", "in2", "second"),
            entry("conv", "in3", "third"));

    final var strategy = new KeyLookupStrategy<String, String>(
            corpus, QN, input -> "conv",
            MatchPolicy.SEQUENTIAL, ExhaustionPolicy.CLAMP);

    assertThat(strategy.resolve("any")).isEqualTo("first");
    assertThat(strategy.resolve("any")).isEqualTo("second");
    assertThat(strategy.resolve("any")).isEqualTo("third");
}

@Test
void sequentialClampRepeatsLastRecord() {
    final var corpus = ListBackedCorpus.of(
            entry("conv", "in1", "first"),
            entry("conv", "in2", "last"));

    final var strategy = new KeyLookupStrategy<String, String>(
            corpus, QN, input -> "conv",
            MatchPolicy.SEQUENTIAL, ExhaustionPolicy.CLAMP);

    strategy.resolve("any"); // first
    strategy.resolve("any"); // last
    assertThat(strategy.resolve("any")).isEqualTo("last");
    assertThat(strategy.resolve("any")).isEqualTo("last");
}

@Test
void sequentialWrapCyclesBackToFirst() {
    final var corpus = ListBackedCorpus.of(
            entry("conv", "in1", "first"),
            entry("conv", "in2", "second"));

    final var strategy = new KeyLookupStrategy<String, String>(
            corpus, QN, input -> "conv",
            MatchPolicy.SEQUENTIAL, ExhaustionPolicy.WRAP);

    assertThat(strategy.resolve("any")).isEqualTo("first");
    assertThat(strategy.resolve("any")).isEqualTo("second");
    assertThat(strategy.resolve("any")).isEqualTo("first");
}

@Test
void sequentialThrowThrowsWhenExhausted() {
    final var corpus = ListBackedCorpus.of(
            entry("conv", "in1", "only"));

    final var strategy = new KeyLookupStrategy<String, String>(
            corpus, QN, input -> "conv",
            MatchPolicy.SEQUENTIAL, ExhaustionPolicy.THROW);

    assertThat(strategy.resolve("any")).isEqualTo("only");
    assertThatThrownBy(() -> strategy.resolve("any"))
            .isInstanceOf(SimulationKeyNotFoundException.class);
}

@Test
void randomMatchReturnsFromMatchingRecords() {
    final var corpus = ListBackedCorpus.of(
            entry("conv", "in1", "a"),
            entry("conv", "in2", "b"),
            entry("other", "in3", "c"));

    final var strategy = new KeyLookupStrategy<String, String>(
            corpus, QN, input -> "conv",
            MatchPolicy.RANDOM, ExhaustionPolicy.CLAMP);

    var results = new java.util.HashSet<String>();
    for (int i = 0; i < 100; i++) {
        results.add(strategy.resolve("any"));
    }
    assertThat(results).containsExactlyInAnyOrder("a", "b");
}

@Test
void firstMatchWithMultipleRecordsAlwaysReturnsFirst() {
    final var corpus = ListBackedCorpus.of(
            entry("conv", "in1", "first"),
            entry("conv", "in2", "second"));

    final var strategy = new KeyLookupStrategy<String, String>(
            corpus, QN, input -> "conv",
            MatchPolicy.FIRST, ExhaustionPolicy.CLAMP);

    assertThat(strategy.resolve("any")).isEqualTo("first");
    assertThat(strategy.resolve("any")).isEqualTo("first");
}

@Test
void sequentialMaintainsIndependentCursorsPerKey() {
    final var corpus = ListBackedCorpus.of(
            entry("a", "in1", "a-first"),
            entry("a", "in2", "a-second"),
            entry("b", "in3", "b-first"),
            entry("b", "in4", "b-second"));

    final var strategy = new KeyLookupStrategy<String, String>(
            corpus, QN, input -> input,
            MatchPolicy.SEQUENTIAL, ExhaustionPolicy.CLAMP);

    assertThat(strategy.resolve("a")).isEqualTo("a-first");
    assertThat(strategy.resolve("b")).isEqualTo("b-first");
    assertThat(strategy.resolve("a")).isEqualTo("a-second");
    assertThat(strategy.resolve("b")).isEqualTo("b-second");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl simulation-core -Dtest=KeyLookupStrategyTest`
Expected: FAIL — compilation error, 5-arg constructor doesn't exist yet

- [ ] **Step 3: Implement KeyLookupStrategy with MatchPolicy**

Replace the full `KeyLookupStrategy.java` content:

```java
package io.casehub.platform.simulation.strategy;

import io.casehub.platform.simulation.DataRealism;
import io.casehub.platform.simulation.ExhaustionPolicy;
import io.casehub.platform.simulation.KeyExtractor;
import io.casehub.platform.simulation.MatchPolicy;
import io.casehub.platform.simulation.SimulationCorpus;
import io.casehub.platform.simulation.SimulationKeyNotFoundException;

import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.logging.Logger;

public final class KeyLookupStrategy<I, O> implements io.casehub.platform.simulation.SimulationStrategy<I, O> {

    private static final Logger LOG = Logger.getLogger(KeyLookupStrategy.class.getName());

    private final SimulationCorpus<I, O> corpus;
    private final String qualifiedName;
    private final KeyExtractor<I> keyExtractor;
    private final MatchPolicy matchPolicy;
    private final ExhaustionPolicy exhaustionPolicy;
    private final ConcurrentHashMap<String, AtomicInteger> cursors;

    public KeyLookupStrategy(final SimulationCorpus<I, O> corpus,
                             final String qualifiedName,
                             final KeyExtractor<I> keyExtractor) {
        this(corpus, qualifiedName, keyExtractor, MatchPolicy.FIRST, ExhaustionPolicy.CLAMP);
    }

    public KeyLookupStrategy(final SimulationCorpus<I, O> corpus,
                             final String qualifiedName,
                             final KeyExtractor<I> keyExtractor,
                             final MatchPolicy matchPolicy,
                             final ExhaustionPolicy exhaustionPolicy) {
        this.corpus = corpus;
        this.qualifiedName = qualifiedName;
        this.keyExtractor = keyExtractor;
        this.matchPolicy = matchPolicy;
        this.exhaustionPolicy = exhaustionPolicy;
        this.cursors = matchPolicy == MatchPolicy.SEQUENTIAL
                ? new ConcurrentHashMap<>() : null;
    }

    @Override
    public O resolve(final I input) {
        final String key = keyExtractor.extract(input);
        return switch (matchPolicy) {
            case FIRST -> resolveFirst(key);
            case RANDOM -> resolveRandom(key);
            case SEQUENTIAL -> resolveSequential(key);
        };
    }

    private O resolveFirst(final String key) {
        return corpus.lookupByKey(qualifiedName, key)
                .orElseThrow(() -> new SimulationKeyNotFoundException(key));
    }

    private O resolveRandom(final String key) {
        final List<O> all = corpus.lookupAllByKey(qualifiedName, key);
        if (all.isEmpty()) {
            throw new SimulationKeyNotFoundException(key);
        }
        return all.get(ThreadLocalRandom.current().nextInt(all.size()));
    }

    private O resolveSequential(final String key) {
        final List<O> all = corpus.lookupAllByKey(qualifiedName, key);
        if (all.isEmpty()) {
            throw new SimulationKeyNotFoundException(key);
        }
        final String cursorKey = qualifiedName + ":" + key;
        final int i = cursors.computeIfAbsent(cursorKey, k -> new AtomicInteger(0))
                .getAndIncrement();
        final int size = all.size();
        return switch (exhaustionPolicy) {
            case CLAMP -> {
                final int idx = Math.clamp(i, 0, size - 1);
                if (i >= size) {
                    LOG.info(() -> "CLAMP activated for " + cursorKey
                            + " at index " + i + " (size: " + size + ")");
                }
                yield all.get(idx);
            }
            case WRAP -> all.get(i % size);
            case THROW -> {
                if (i >= size) {
                    throw new SimulationKeyNotFoundException(key);
                }
                yield all.get(i);
            }
        };
    }

    @Override
    public boolean canResolve(final I input) {
        final String key = keyExtractor.extract(input);
        return corpus.lookupByKey(qualifiedName, key).isPresent();
    }

    @Override
    public DataRealism dataRealism() {
        return DataRealism.DOMAIN_PLAUSIBLE;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl simulation-core -Dtest=KeyLookupStrategyTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add simulation-core/src/main/java/io/casehub/platform/simulation/strategy/KeyLookupStrategy.java
git add simulation-core/src/test/java/io/casehub/platform/simulation/strategy/KeyLookupStrategyTest.java
git commit -m "feat(#627): enhance KeyLookupStrategy with MatchPolicy support"
```

### Task 4: Add CLAMP support to SequentialStrategy with exhaustive switch

**Files:**
- Modify: `simulation-core/src/main/java/io/casehub/platform/simulation/strategy/SequentialStrategy.java`
- Modify: `simulation-core/src/test/java/io/casehub/platform/simulation/strategy/SequentialStrategyTest.java`

**Interfaces:**
- Consumes: `ExhaustionPolicy.CLAMP` (from Task 1)
- Produces: Updated `SequentialStrategy` handling all three exhaustion policies

- [ ] **Step 1: Write failing tests for CLAMP behavior**

Add to `SequentialStrategyTest.java`:

```java
@Test
void clampRepeatsLastRecordAfterExhaustion() {
    final var corpus = ListBackedCorpus.of(
            entry("k1", "in1", "first"),
            entry("k2", "in2", "last"));

    final var strategy = new SequentialStrategy<String, String>(
            corpus, QN, ExhaustionPolicy.CLAMP);

    assertThat(strategy.resolve("any")).isEqualTo("first");
    assertThat(strategy.resolve("any")).isEqualTo("last");
    assertThat(strategy.resolve("any")).isEqualTo("last");
    assertThat(strategy.resolve("any")).isEqualTo("last");
}

@Test
void clampCanResolveAlwaysReturnsTrueWhenCorpusNotEmpty() {
    final var corpus = ListBackedCorpus.of(entry("k1", "in1", "out"));

    final var strategy = new SequentialStrategy<String, String>(
            corpus, QN, ExhaustionPolicy.CLAMP);

    strategy.resolve("any"); // consume the only record
    assertThat(strategy.canResolve("any")).isTrue();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl simulation-core -Dtest=SequentialStrategyTest#clamp*`
Expected: FAIL — CLAMP falls through to WRAP behavior (no explicit handling)

- [ ] **Step 3: Convert SequentialStrategy to exhaustive switch**

Replace the `resolve` and `canResolve` methods in `SequentialStrategy.java`:

```java
@Override
public O resolve(final I input) {
    final int size = corpus.size(qualifiedName);
    if (size == 0) {
        throw new SimulationExhaustedException(
                "Corpus is empty for " + qualifiedName);
    }
    final int i = index.getAndIncrement();
    final int resolvedIndex = switch (exhaustionPolicy) {
        case THROW -> {
            if (i >= size) {
                throw new SimulationExhaustedException(
                        "Corpus exhausted at index " + i + " (size: " + size + ") for " + qualifiedName);
            }
            yield i;
        }
        case WRAP -> i % size;
        case CLAMP -> {
            final int idx = Math.clamp(i, 0, size - 1);
            if (i >= size) {
                LOG.info(() -> "CLAMP activated for " + qualifiedName
                        + " at index " + i + " (size: " + size + ")");
            }
            yield idx;
        }
    };
    return corpus.lookupByIndex(qualifiedName, resolvedIndex)
            .orElseThrow(() -> new SimulationExhaustedException(
                    "No entry at index " + resolvedIndex + " for " + qualifiedName));
}

@Override
public boolean canResolve(final I input) {
    final int size = corpus.size(qualifiedName);
    if (size == 0) {
        return false;
    }
    return switch (exhaustionPolicy) {
        case THROW -> index.get() < size;
        case WRAP, CLAMP -> true;
    };
}
```

Add the logger field at the top of the class (after the class declaration):

```java
private static final java.util.logging.Logger LOG =
        java.util.logging.Logger.getLogger(SequentialStrategy.class.getName());
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl simulation-core -Dtest=SequentialStrategyTest`
Expected: PASS (all existing tests + new CLAMP tests)

- [ ] **Step 5: Commit**

```bash
git add simulation-core/src/main/java/io/casehub/platform/simulation/strategy/SequentialStrategy.java
git add simulation-core/src/test/java/io/casehub/platform/simulation/strategy/SequentialStrategyTest.java
git commit -m "feat(#627): add CLAMP to SequentialStrategy with exhaustive switch"
```

---

## Batch 3: Config Wiring (simulation-config-core, simulation-core)

### Task 5: Wire MatchPolicy through config and runtime

**Files:**
- Modify: `simulation-core/src/main/java/io/casehub/platform/simulation/SimulationConfig.java`
- Modify: `simulation-core/src/main/java/io/casehub/platform/simulation/SimulationRuntime.java:176-178`
- Modify: `simulation-config-core/src/main/java/io/casehub/platform/simulation/config/YamlSimulationConfig.java:277-316,558`
- Test: `simulation-config-core/src/test/java/io/casehub/platform/simulation/config/YamlSimulationConfigTest.java`

**Interfaces:**
- Consumes: `MatchPolicy` (from Task 1), `KeyLookupStrategy` 5-arg constructor (from Task 3)
- Produces: `SimulationConfig.matchPolicy(String qualifiedName) → Optional<MatchPolicy>`, YAML `match:` field parsing

- [ ] **Step 1: Add matchPolicy to SimulationConfig interface**

Add to `SimulationConfig.java` after `exhaustionPolicy` method (line 11):

```java
default Optional<MatchPolicy> matchPolicy(String qualifiedName) {
    return Optional.empty();
}
```

Add import for `MatchPolicy` at the top.

- [ ] **Step 2: Add match field to YamlSimulationConfig.MethodConfig**

Modify the `MethodConfig` record (line 558) — add `matchPolicy` field:

```java
record MethodConfig(
        String strategy,
        Boolean capture,
        ExhaustionPolicy exhaustionPolicy,
        MatchPolicy matchPolicy,
        String keyExtractor,
        String scorer,
        Double threshold,
        List<CorpusEntry> corpus,
        List<String> corpusFiles,
        DataRealism fallthroughRealism) {}
```

- [ ] **Step 3: Parse match field in parseMethodConfig**

In `parseMethodConfig` (line 277), add after the `ep` parsing (after line 284):

```java
MatchPolicy mp = props.containsKey("match")
        ? MatchPolicy.valueOf(
        ((String) props.get("match")).toUpperCase().replace("-", "_"))
        : null;
```

Update the `return new MethodConfig(...)` call (line 314) to include `mp`:

```java
return new MethodConfig(strategy, capture, ep, mp, keyExtractor, scorer,
        threshold, corpus, corpusFiles, fallthroughRealism);
```

- [ ] **Step 4: Implement matchPolicy accessor in YamlSimulationConfig**

Add after `exhaustionPolicy` method (after line 103):

```java
@Override
public Optional<MatchPolicy> matchPolicy(String qualifiedName) {
    return Optional.ofNullable(methods.get(qualifiedName))
            .map(MethodConfig::matchPolicy);
}
```

Add import for `MatchPolicy` at the top of the file.

- [ ] **Step 5: Wire MatchPolicy in SimulationRuntime.createStrategy**

Modify the `"key-lookup"` case in `createStrategy` (lines 176-179):

```java
case "key-lookup" -> {
    final KeyExtractor extractor = requireExtractor(qualifiedName);
    final MatchPolicy mp = config.matchPolicy(qualifiedName).orElse(MatchPolicy.FIRST);
    final ExhaustionPolicy ep = config.exhaustionPolicy(qualifiedName)
            .orElse(mp == MatchPolicy.SEQUENTIAL ? ExhaustionPolicy.CLAMP : ExhaustionPolicy.CLAMP);
    yield new KeyLookupStrategy<>(corpus, qualifiedName, extractor, mp, ep);
}
```

Add import for `MatchPolicy` at the top of the file.

- [ ] **Step 6: Write config parsing test**

Add to `YamlSimulationConfigTest.java`:

```java
@Test
void matchPolicyParsedFromYaml() {
    String yaml = """
            strategies:
              test.method:
                strategy: key-lookup
                key-extractor: identity
                match: sequential
                exhaustion-policy: clamp
                corpus:
                  - key: conv
                    output: first
                  - key: conv
                    output: second
            """;
    var config = YamlSimulationConfig.parse(yaml);

    assertThat(config.matchPolicy("test.method"))
            .isPresent()
            .contains(MatchPolicy.SEQUENTIAL);
    assertThat(config.exhaustionPolicy("test.method"))
            .isPresent()
            .contains(ExhaustionPolicy.CLAMP);
}

@Test
void matchPolicyDefaultsToEmptyWhenAbsent() {
    String yaml = """
            strategies:
              test.method:
                strategy: key-lookup
                key-extractor: identity
                corpus:
                  - key: k1
                    output: out
            """;
    var config = YamlSimulationConfig.parse(yaml);

    assertThat(config.matchPolicy("test.method")).isEmpty();
}
```

- [ ] **Step 7: Run all tests across affected modules**

Run: `mvn --batch-mode test -pl simulation-api,simulation-core,simulation-inmem,simulation-config-core`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add simulation-core/src/main/java/io/casehub/platform/simulation/SimulationConfig.java
git add simulation-core/src/main/java/io/casehub/platform/simulation/SimulationRuntime.java
git add simulation-config-core/src/main/java/io/casehub/platform/simulation/config/YamlSimulationConfig.java
git add simulation-config-core/src/test/java/io/casehub/platform/simulation/config/YamlSimulationConfigTest.java
git commit -m "feat(#627): wire MatchPolicy through config and runtime"
```

---

## References

- [2026-10-16-match-policy-design.md] — design spec this plan implements
- `simulation-api/.../ExhaustionPolicy.java` — existing enum extended with CLAMP
- `simulation-api/.../SimulationCorpus.java` — SPI extended with lookupAllByKey default
- `simulation-core/.../KeyLookupStrategy.java` — enhanced with MatchPolicy
- `simulation-core/.../SequentialStrategy.java` — CLAMP support + exhaustive switch
- `simulation-core/.../SimulationConfig.java` — matchPolicy accessor added
- `simulation-core/.../SimulationRuntime.java:171-193` — strategy factory wiring
- `simulation-config-core/.../YamlSimulationConfig.java` — YAML match field parsing
- [GitHub #627] — focal issue
