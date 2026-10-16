# Decisions — Issue #627: MatchPolicy for KeyLookupStrategy

## D1: Where the queue behavior belongs

**Choice:** In the strategy's resolution of corpus records (MatchPolicy on KeyLookupStrategy)
**Alternatives:**
- Queue in KeyExtractor (stateful iterator) — breaks the extractor contract; extractors derive keys FROM input, making them stateful iterators conflates two concerns
- New QueuedKeyLookupStrategy class — unnecessary; the existing class can handle all three modes with a single enum field
**Rationale:** The corpus already supports multiple records per key (they accumulate but only the first was returned). The strategy is the natural place to decide how to select among multiple matches. No SPI or corpus changes needed beyond a default method.
**Trade-offs:** KeyLookupStrategy gains state (cursor map) when using SEQUENTIAL, making it no longer purely stateless. Acceptable since SequentialStrategy already uses this pattern. The cursor map is bounded by the corpus key space (only keys that exist in the corpus produce entries) and is garbage-collected with the strategy instance on overlay pop or runtime teardown. Note: RecordedReplayStrategy's sequential fallback operates on key MISS (falls through to index-based iteration); MatchPolicy SEQUENTIAL iterates on key MATCH (multiple records for the same key). These are orthogonal capabilities.
**Sources:** simulation-core KeyLookupStrategy.java, simulation-core SequentialStrategy.java, simulation-inmem InMemorySimulationCorpus.java
**Exploration:** deep-analysis
**Status:** captured

## D2: Exhaustion policy for SEQUENTIAL mode

**Choice:** Add CLAMP to existing ExhaustionPolicy enum; default to CLAMP when match=sequential
**Alternatives:**
- Hardcode clamp behavior — inflexible; some users may want WRAP (cycling) or THROW (strict scripting)
- Separate exhaustion concept for MatchPolicy — creates parallel concept when ExhaustionPolicy already models this exactly
- THROW as default — fail-fast surfaces misconfiguration immediately, but breaks the primary conversation use case where exceeding scripted depth should degrade gracefully, not crash
**Rationale:** ExhaustionPolicy (WRAP, THROW) already models "what to do when records run out." CLAMP is a third natural option: clamp the index to [0, size-1], repeating the last record indefinitely. Reusing the existing enum avoids concept proliferation. SequentialStrategy benefits from CLAMP too, but requires explicit code changes — the current `if`-chain pattern must be converted to an exhaustive `switch` (no default) so that adding enum values produces a compile error rather than silent behavioral degradation.
**Trade-offs:** Adding to an existing API-module enum — but it's additive and backward-compatible. CLAMP as default for match=sequential masks insufficient corpus data (last response repeats silently); mitigated by emitting a log statement (INFO level) when CLAMP activates (cursor exceeds corpus size), making exhaustion visible without breaking the flow. SimulationApiTest.exhaustionPolicyHasExpectedValues() will break — this is the awareness gate working as designed.
**D2 depends on D1:** CLAMP is independently useful for SequentialStrategy regardless of D1. However, the default-selection behavior (CLAMP when match=sequential) requires D1's MatchPolicy.
**Sources:** simulation-api ExhaustionPolicy.java, simulation-core SequentialStrategy.java
**Exploration:** quick
**Status:** revised
**Revised:** R1 review — renamed STICKY_LAST → CLAMP (Java 21 Math.clamp precedent, consistent with WRAP/THROW mechanism-focused naming); acknowledged SequentialStrategy requires explicit code changes with exhaustive switch; added logging on CLAMP activation; documented CLAMP-vs-THROW default trade-off; added dependency note

## D3: SPI expansion — lookupAllByKey default method on SimulationCorpus

**Choice:** Add `lookupAllByKey` as a default method on `SimulationCorpus` (simulation-api SPI)
**Alternatives:**
- No SPI change, filter in strategy only — forces every strategy needing all-by-key to independently call `list()` + filter; duplicates logic, prevents corpus implementations from providing optimized lookups
- Abstract method (no default) — breaks all existing SimulationCorpus implementations; disproportionate breakage for an additive capability
- Utility method in simulation-core — avoids SPI expansion but prevents corpus implementations from overriding with optimized lookups
**Rationale:** The default implementation delegates to `list()` + filter (O(n) in total records), making it non-breaking for all existing implementors including third-party ones. InMemorySimulationCorpus overrides with a direct stream filter on its internal store. The SPI expansion is deliberate: it signals to corpus implementors that multi-record-per-key lookup is a first-class operation they may want to optimize.
**Trade-offs:** Third-party SimulationCorpus implementations get the O(n) default silently — correct but not optimal. Implementors who care about performance can override.
**Sources:** simulation-api SimulationCorpus.java, simulation-inmem InMemorySimulationCorpus.java
**Exploration:** surfaced during R1 review (was implicit)
**Status:** captured

## D4: MatchPolicy module placement

**Choice:** MatchPolicy enum in simulation-api module
**Alternatives:**
- simulation-core — smaller API surface, but forces simulation-config-core to depend on simulation-core for parsing, inverting the dependency direction
- String constants instead of enum — avoids module placement question but loses type safety and IDE support
**Rationale:** Follows the established pattern: ExhaustionPolicy is in simulation-api for the same reason — config parsers (YamlSimulationConfig in simulation-config-core) need to resolve configuration strings to typed enum values. simulation-config-core depends on simulation-api, not on simulation-core. Placing MatchPolicy in simulation-core would either require simulation-config-core to take a dependency on simulation-core (circular risk) or force config parsing to use raw strings validated later.
**Sources:** simulation-api ExhaustionPolicy.java, simulation-config-core YamlSimulationConfig.java, consumer-guide.md module dependency table
**Exploration:** surfaced during R1 review (was implicit)
**Status:** captured
