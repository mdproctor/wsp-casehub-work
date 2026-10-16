# Decisions — Issue #627: MatchPolicy for KeyLookupStrategy

## D1: Where the queue behavior belongs

**Choice:** In the strategy's resolution of corpus records (MatchPolicy on KeyLookupStrategy)
**Alternatives:**
- Queue in KeyExtractor (stateful iterator) — breaks the extractor contract; extractors derive keys FROM input, making them stateful iterators conflates two concerns
- New QueuedKeyLookupStrategy class — unnecessary; the existing class can handle all three modes with a single enum field
**Rationale:** The corpus already supports multiple records per key (they accumulate but only the first was returned). The strategy is the natural place to decide how to select among multiple matches. No SPI or corpus changes needed beyond a default method.
**Trade-offs:** KeyLookupStrategy gains state (cursor map) when using SEQUENTIAL, making it no longer purely stateless. Acceptable since SequentialStrategy already uses this pattern.
**Sources:** simulation-core KeyLookupStrategy.java, simulation-core SequentialStrategy.java, simulation-inmem InMemorySimulationCorpus.java
**Exploration:** deep-analysis
**Status:** captured

## D2: Exhaustion policy for SEQUENTIAL mode

**Choice:** Add STICKY_LAST to existing ExhaustionPolicy enum; default to STICKY_LAST when match=sequential
**Alternatives:**
- Hardcode sticky-last behavior — inflexible; some users may want WRAP (cycling) or THROW (strict scripting)
- Separate exhaustion concept for MatchPolicy — creates parallel concept when ExhaustionPolicy already models this exactly
**Rationale:** ExhaustionPolicy (WRAP, THROW) already models "what to do when records run out." STICKY_LAST is a third natural option. Reusing the existing enum avoids concept proliferation and lets SequentialStrategy benefit from it too.
**Trade-offs:** Adding to an existing API-module enum — but it's additive and backward-compatible.
**Sources:** simulation-api ExhaustionPolicy.java, simulation-core SequentialStrategy.java
**Exploration:** quick
**Status:** captured
