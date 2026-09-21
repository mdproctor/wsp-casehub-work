# Decisions — #371 Query Extraction SPI

## D1: SPI module location

**Choice:** neocortex `rag-api` — alongside `CaseContextRetriever` and `RetrievalQuery`
**Alternatives:**
- blocks (engine layer) — closer to Worker consumers, but creates reverse dependency for neocortex usage
- New `rag-context` module — avoids bloating rag-api, but adds module management overhead for a single interface
**Rationale:** The SPI is a retrieval concern (context → query), not a worker concern. Placing it in rag-api means any consumer (blocks, engine, standalone) can use it without depending on the worker framework. rag-api already has zero heavyweight deps.
**Trade-offs:** rag-api gains a `Map<String, Object>` parameter type in its API surface, which is less typed than a domain-specific context object. Acceptable because case context is inherently unstructured across domains.
**Sources:** `rag-api/src/main/java/io/casehub/neocortex/rag/CaseContextRetriever.java`, SOC `SocRagRetrieveService.java`
**Exploration:** quick
**Status:** captured

## D2: Corpus configuration mechanism

**Choice:** Constructor/method parameter — `List<CorpusRef>` passed explicitly
**Alternatives:**
- Quarkus `@ConfigMapping` — declarative but couples to Quarkus and adds config complexity for typically static data
- SPI method on the extraction strategy — bundles extraction + corpus selection, tight coupling between two concerns
**Rationale:** Simple, explicit, no framework dependency. SOC already hardcodes its corpus list as a field — moving it to a parameter is the minimal extraction. Each app provides its own list at wiring time.
**Trade-offs:** No declarative config — apps must wire corpora programmatically. Fine for 1-3 corpora per app; would need revisiting if corpus lists became dynamic or large.
**Sources:** SOC `SocRagRetrieveService.java` (hardcoded `CORPORA` field)
**Exploration:** quick
**Status:** captured

## D3: SPI return type

**Choice:** `RetrievalQuery` — the strategy returns a full `RetrievalQuery`, not just a `String`
**Alternatives:**
- Return `String` — simpler contract, but domain-specific tuning (BM25 boost, weight multipliers, expansion) requires separate configuration paths
**Rationale:** `RetrievalQuery` gives the strategy full control over retrieval behaviour (weight multipliers, BM25 boost, expanded text). Apps that just want a string use `RetrievalQuery.of(text)` — zero overhead. CaseContextRetriever already works with `RetrievalQuery` internally.
**Trade-offs:** Slightly higher surface area for simple strategies. Mitigated by `RetrievalQuery.of()` factory.
**Sources:** `rag-api/src/main/java/io/casehub/neocortex/rag/RetrievalQuery.java`
**Exploration:** quick
**Status:** captured

## D4: Composition approach

**Choice:** Standalone `@FunctionalInterface` SPI + new overload on `CaseContextRetriever`
**Alternatives:**
- New `ContextAwareRetriever` composite class — pre-configured with strategy + corpora, but adds a class + CDI producer for syntactic sugar that saves one parameter at the call site
**Rationale:** One interface, one new method, zero new classes. The functional interface is composable, testable, and lambda-friendly. `ContextAwareRetriever` would only save passing strategy + corpora at the single call site per app — not worth the abstraction.
**Trade-offs:** Call sites pass strategy + corpora explicitly each time. Acceptable because there's typically one call site per app.
**Sources:** SOC `SocRagRetrieveService.java` (single call site pattern), `CaseContextRetriever.java`
**Exploration:** quick
**Status:** captured

## D5: Worker template ownership

**Choice:** Out of scope for neocortex — tracked as casehubio/blocks#293
**Alternatives:**
- Include worker factory in neocortex — would introduce engine/worker API dependency into neocortex, violating its role as a standalone inference/retrieval library
**Rationale:** Worker is an engine/blocks API. Neocortex provides the retrieval SPI. The worker template is a thin shell (~15 lines) that each app can write, or blocks#293 can provide generically.
**Trade-offs:** Until blocks#293 ships, each app writes its own thin worker factory. SOC already has one that works.
**Sources:** SOC `RuleRagRetrievalWorker.java`, casehubio/blocks#293
**Exploration:** quick
**Status:** captured
