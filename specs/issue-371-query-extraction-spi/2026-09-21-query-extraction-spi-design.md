# Query Extraction SPI — Design Spec

**Issue:** casehubio/neocortex#371
**Date:** 2026-09-21
**Status:** Draft

## Summary

Add a `QueryExtractionStrategy` functional interface to `rag-api` that formalises the "case context → retrieval query" pattern. Wire it into `CaseContextRetriever` via a new method overload. This gives apps a standard SPI to implement for domain-specific query construction, while keeping the multi-corpus retrieval orchestration in `CaseContextRetriever`.

## Motivation

SOC shipped the first concrete implementation of this pattern (`SocRagRetrieveService` + `RuleRagRetrievalWorker`). The service extracts alert rules, ATT&CK techniques, and IOC types from case context and builds a query string. The retrieval mechanics (multi-corpus fan-out, dedup, score sorting) are delegated to `CaseContextRetriever`.

Every future app (AML, Clinical) will repeat this structure: extract domain-specific fields → build query → retrieve → map results. The extraction logic is the only part that varies per domain. Making it an SPI avoids each app reinventing the wiring.

## Scope

**In scope (this issue):**
- `QueryExtractionStrategy` — `@FunctionalInterface` in `rag-api`
- New `retrieve()` overload on `CaseContextRetriever` that accepts a strategy
- Tests

**Out of scope:**
- Worker factory template — tracked as casehubio/blocks#293
- Quarkus config-driven corpus selection — constructor parameter is sufficient
- Concrete strategy implementations — each app provides its own

## Design

### QueryExtractionStrategy

```java
package io.casehub.neocortex.rag;

@FunctionalInterface
public interface QueryExtractionStrategy {
    RetrievalQuery extractQuery(Map<String, Object> caseContext);
}
```

A pure function from unstructured case context to a `RetrievalQuery`. Returning `RetrievalQuery` (not `String`) gives strategies control over weight multipliers, BM25 boost, and query expansion. Simple strategies use `RetrievalQuery.of(text)`.

The strategy may return `null` to signal that the context does not contain enough information to form a query. `CaseContextRetriever` treats `null` as "skip retrieval" and returns an empty list.

### CaseContextRetriever changes

New overload:

```java
public List<RetrievedChunk> retrieve(
        Map<String, Object> caseContext,
        QueryExtractionStrategy strategy,
        List<CorpusRef> corpora,
        int maxResults) {
    RetrievalQuery query = strategy.extractQuery(caseContext);
    if (query == null) return List.of();
    return retrieve(query, corpora, maxResults);
}
```

The existing `retrieve(String queryText, List<CorpusRef>, int)` method is refactored internally to delegate to a new `retrieve(RetrievalQuery, List<CorpusRef>, int)`:

```java
public List<RetrievedChunk> retrieve(RetrievalQuery query, List<CorpusRef> corpora, int maxResults) {
    // existing multi-corpus fan-out, dedup, score-sort logic
}

public List<RetrievedChunk> retrieve(String queryText, List<CorpusRef> corpora, int maxResults) {
    if (queryText == null || queryText.isBlank() || corpora.isEmpty()) return List.of();
    return retrieve(RetrievalQuery.of(queryText), corpora, maxResults);
}
```

This gives three entry points at increasing levels of abstraction:
1. `retrieve(RetrievalQuery, corpora, maxResults)` — direct query, full control
2. `retrieve(String, corpora, maxResults)` — simple string query (existing API, preserved)
3. `retrieve(Map context, strategy, corpora, maxResults)` — strategy-driven extraction

### How SOC would migrate

Before (current SOC code):
```java
String queryText = buildQueryText(caseContext);
return contextRetriever.retrieve(queryText, CORPORA, MAX_RESULTS).stream()
    .map(CaseContextRetriever::toMap).toList();
```

After:
```java
return contextRetriever.retrieve(caseContext, this::buildQueryText, CORPORA, MAX_RESULTS).stream()
    .map(CaseContextRetriever::toMap).toList();
```

Where `buildQueryText` returns `RetrievalQuery` instead of `String`. Or, SOC can implement `QueryExtractionStrategy` as a separate `@ApplicationScoped` bean and inject it.

## Testing

- `QueryExtractionStrategy` null return → empty result
- `QueryExtractionStrategy` returning a valid query → delegates to retrieve correctly
- `CaseContextRetriever.retrieve(RetrievalQuery, ...)` — same behaviour as current string-based method
- Existing `CaseContextRetrieverTest` cases continue to pass (string overload delegates)

## References

- `rag-api/src/main/java/io/casehub/neocortex/rag/CaseContextRetriever.java` — existing retrieval utility
- `rag-api/src/main/java/io/casehub/neocortex/rag/RetrievalQuery.java` — query record
- `rag-api/src/main/java/io/casehub/neocortex/rag/CaseRetriever.java` — underlying SPI
- SOC `app/src/main/java/io/casehub/soc/engine/rag/SocRagRetrieveService.java` — reference implementation
- SOC `app/src/main/java/io/casehub/soc/worker/RuleRagRetrievalWorker.java` — worker template reference
- casehubio/blocks#293 — worker factory follow-up
- D1–D5 in `decisions.md` — design decisions
