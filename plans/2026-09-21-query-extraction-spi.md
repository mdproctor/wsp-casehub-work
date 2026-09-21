# Query Extraction SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #371 — Case-context retrieval pattern — generic query extraction from case context for RAG retrieval

**Goal:** Add a `QueryExtractionStrategy` functional interface to `rag-api` and wire it into `CaseContextRetriever` so apps can implement domain-specific query construction against a standard SPI.

**Architecture:** One new `@FunctionalInterface` (`QueryExtractionStrategy`) in `rag-api`. `CaseContextRetriever` gains a `retrieve(RetrievalQuery, ...)` overload (core logic) and a `retrieve(Map, strategy, ...)` overload (strategy-driven). The existing `retrieve(String, ...)` method delegates to the new `RetrievalQuery` overload. No new modules, no new dependencies.

**Tech Stack:** Java 21, JUnit 5

## Global Constraints

- All code in `rag-api` module — zero new dependencies
- `QueryExtractionStrategy` must be `@FunctionalInterface` (lambda-friendly)
- Existing `CaseContextRetrieverTest` must continue to pass unchanged
- Follow existing code style in `rag-api` (no Javadoc, `java.util.logging`)

---

## Batch 1: QueryExtractionStrategy SPI + CaseContextRetriever refactor

### Task 1: Create QueryExtractionStrategy and refactor CaseContextRetriever

**Files:**
- Create: `rag-api/src/main/java/io/casehub/neocortex/rag/QueryExtractionStrategy.java`
- Modify: `rag-api/src/main/java/io/casehub/neocortex/rag/CaseContextRetriever.java`
- Modify: `rag-api/src/test/java/io/casehub/neocortex/rag/CaseContextRetrieverTest.java`

**Interfaces:**
- Consumes: `RetrievalQuery`, `CaseRetriever`, `CorpusRef`, `RetrievedChunk` (all existing in `rag-api`)
- Produces: `QueryExtractionStrategy` — `@FunctionalInterface` with `RetrievalQuery extractQuery(Map<String, Object> caseContext)`

- [ ] **Step 1: Write failing test — strategy with null return yields empty list**

Add to `CaseContextRetrieverTest.java`:

```java
@Test void strategyReturningNullYieldsEmpty() {
    CaseRetriever delegate = (q, c, max, f) -> fail("should not be called");
    var retriever = new CaseContextRetriever(delegate);
    QueryExtractionStrategy strategy = ctx -> null;
    var results = retriever.retrieve(Map.of("alert", "test"), strategy, List.of(CORPUS_A), 10);
    assertTrue(results.isEmpty());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rag-api -Dtest=CaseContextRetrieverTest#strategyReturningNullYieldsEmpty -DfailIfNoTests=false`
Expected: FAIL — `QueryExtractionStrategy` does not exist, `retrieve` overload does not exist

- [ ] **Step 3: Create QueryExtractionStrategy interface**

Create `rag-api/src/main/java/io/casehub/neocortex/rag/QueryExtractionStrategy.java`:

```java
package io.casehub.neocortex.rag;

import java.util.Map;

@FunctionalInterface
public interface QueryExtractionStrategy {
    RetrievalQuery extractQuery(Map<String, Object> caseContext);
}
```

- [ ] **Step 4: Refactor CaseContextRetriever — extract RetrievalQuery overload and add strategy overload**

Refactor the existing `retrieve(String, List<CorpusRef>, int)` method to delegate to a new `retrieve(RetrievalQuery, List<CorpusRef>, int)` overload. Add the strategy-driven overload. The result:

```java
package io.casehub.neocortex.rag;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.logging.Level;
import java.util.logging.Logger;

public class CaseContextRetriever {

    private static final Logger LOG = Logger.getLogger(CaseContextRetriever.class.getName());

    private final CaseRetriever caseRetriever;

    public CaseContextRetriever(CaseRetriever caseRetriever) {
        this.caseRetriever = caseRetriever;
    }

    public List<RetrievedChunk> retrieve(RetrievalQuery query, List<CorpusRef> corpora, int maxResults) {
        if (corpora.isEmpty()) return List.of();

        var byDocId = new LinkedHashMap<String, RetrievedChunk>();

        for (var corpus : corpora) {
            try {
                for (var chunk : caseRetriever.retrieve(query, corpus, maxResults)) {
                    byDocId.merge(chunk.sourceDocumentId(), chunk,
                            (existing, incoming) -> incoming.relevanceScore() > existing.relevanceScore() ? incoming : existing);
                }
            } catch (Exception e) {
                LOG.log(Level.WARNING, "Retrieval from corpus " + corpus.corpusName() + " failed — skipping", e);
            }
        }

        return byDocId.values().stream()
                .sorted(Comparator.comparingDouble(RetrievedChunk::relevanceScore).reversed())
                .limit(maxResults)
                .toList();
    }

    public List<RetrievedChunk> retrieve(String queryText, List<CorpusRef> corpora, int maxResults) {
        if (queryText == null || queryText.isBlank()) return List.of();
        return retrieve(RetrievalQuery.of(queryText), corpora, maxResults);
    }

    public List<RetrievedChunk> retrieve(
            Map<String, Object> caseContext,
            QueryExtractionStrategy strategy,
            List<CorpusRef> corpora,
            int maxResults) {
        if (corpora.isEmpty()) return List.of();
        RetrievalQuery query = strategy.extractQuery(caseContext);
        if (query == null) return List.of();
        return retrieve(query, corpora, maxResults);
    }

    public static Map<String, Object> toMap(RetrievedChunk chunk) {
        var map = new LinkedHashMap<String, Object>();
        map.putAll(chunk.metadata());
        map.put("content", chunk.content());
        map.put("sourceDocumentId", chunk.sourceDocumentId());
        map.put("relevanceScore", chunk.relevanceScore());
        return Map.copyOf(map);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rag-api -Dtest=CaseContextRetrieverTest#strategyReturningNullYieldsEmpty -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 6: Write test — strategy extracts query and retrieves results**

Add to `CaseContextRetrieverTest.java`:

```java
@Test void strategyExtractsQueryAndRetrievesResults() {
    CaseRetriever delegate = (q, c, max, f) -> {
        assertEquals("alert-rule technique-1", q.text());
        return List.of(chunk("d1", "threat intel", 0.9));
    };
    var retriever = new CaseContextRetriever(delegate);
    QueryExtractionStrategy strategy = ctx -> {
        String alert = (String) ctx.getOrDefault("alert", "");
        String technique = (String) ctx.getOrDefault("technique", "");
        return RetrievalQuery.of(alert + " " + technique);
    };
    var context = Map.<String, Object>of("alert", "alert-rule", "technique", "technique-1");
    var results = retriever.retrieve(context, strategy, List.of(CORPUS_A), 10);
    assertEquals(1, results.size());
    assertEquals("d1", results.getFirst().sourceDocumentId());
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rag-api -Dtest=CaseContextRetrieverTest#strategyExtractsQueryAndRetrievesResults -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 8: Write test — strategy overload with empty corpora returns empty**

Add to `CaseContextRetrieverTest.java`:

```java
@Test void strategyWithEmptyCorporaReturnsEmpty() {
    CaseRetriever delegate = (q, c, max, f) -> fail("should not be called");
    var retriever = new CaseContextRetriever(delegate);
    QueryExtractionStrategy strategy = ctx -> fail("should not be called");
    var results = retriever.retrieve(Map.of(), strategy, List.of(), 10);
    assertTrue(results.isEmpty());
}
```

- [ ] **Step 9: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rag-api -Dtest=CaseContextRetrieverTest#strategyWithEmptyCorporaReturnsEmpty -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 10: Write test — RetrievalQuery overload behaves same as string overload**

Add to `CaseContextRetrieverTest.java`:

```java
@Test void retrievalQueryOverloadBehavesSameAsStringOverload() {
    CaseRetriever delegate = (q, c, max, f) -> List.of(chunk("d1", "text", 0.9));
    var retriever = new CaseContextRetriever(delegate);
    var stringResults = retriever.retrieve("query", List.of(CORPUS_A), 10);
    var queryResults = retriever.retrieve(RetrievalQuery.of("query"), List.of(CORPUS_A), 10);
    assertEquals(stringResults.size(), queryResults.size());
    assertEquals(stringResults.getFirst().sourceDocumentId(), queryResults.getFirst().sourceDocumentId());
}
```

- [ ] **Step 11: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rag-api -Dtest=CaseContextRetrieverTest#retrievalQueryOverloadBehavesSameAsStringOverload -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 12: Write test — strategy can set weight multipliers**

Add to `CaseContextRetrieverTest.java`:

```java
@Test void strategyCanSetWeightMultipliers() {
    CaseRetriever delegate = (q, c, max, f) -> {
        assertEquals(2.0, q.weightMultipliers().get("bm25"), 0.001);
        return List.of(chunk("d1", "text", 0.9));
    };
    var retriever = new CaseContextRetriever(delegate);
    QueryExtractionStrategy strategy = ctx ->
            RetrievalQuery.of("query").withBm25Boost(2.0);
    var results = retriever.retrieve(Map.of(), strategy, List.of(CORPUS_A), 10);
    assertEquals(1, results.size());
}
```

- [ ] **Step 13: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rag-api -Dtest=CaseContextRetrieverTest#strategyCanSetWeightMultipliers -DfailIfNoTests=false`
Expected: PASS

- [ ] **Step 14: Run full existing test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rag-api -Dtest=CaseContextRetrieverTest -DfailIfNoTests=false`
Expected: All tests PASS (original 11 + 5 new = 16 total)

- [ ] **Step 15: Commit**

```bash
git add rag-api/src/main/java/io/casehub/neocortex/rag/QueryExtractionStrategy.java rag-api/src/main/java/io/casehub/neocortex/rag/CaseContextRetriever.java rag-api/src/test/java/io/casehub/neocortex/rag/CaseContextRetrieverTest.java
git commit -m "feat(#371): add QueryExtractionStrategy SPI and CaseContextRetriever overloads

Adds @FunctionalInterface QueryExtractionStrategy to rag-api for
domain-specific query extraction from case context. Refactors
CaseContextRetriever to accept RetrievalQuery directly and adds
strategy-driven retrieve() overload.

Closes #371"
```

## References

- [2026-09-21-query-extraction-spi-design.md] — design spec this plan implements
- [rag-api/src/main/java/io/casehub/neocortex/rag/CaseContextRetriever.java] — existing retrieval utility being refactored
- [rag-api/src/main/java/io/casehub/neocortex/rag/RetrievalQuery.java] — query record (return type of SPI)
- [rag-api/src/test/java/io/casehub/neocortex/rag/CaseContextRetrieverTest.java] — existing tests (11 tests, must not break)
- [casehubio/blocks#293] — follow-up worker factory issue
- [GitHub #371] — focal issue
