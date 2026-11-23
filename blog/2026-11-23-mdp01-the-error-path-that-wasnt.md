---
layout: post
title: "The Error Path That Wasn't"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/ledger]
tags: [trust-scoring, testing, defensive-programming, jackson, edge-cases]
---

# The Error Path That Wasn't

The ledger's trust scoring pipeline parses JSON from `relevanceSnapshot` — a string field on each attestation that carries contextual data about the attestor's relationship to the work being reviewed. Org distance, team membership, capability overlap. The parsing code had a try/catch that swallowed malformed JSON and returned an empty map. Defensive and correct. But nobody had ever tested what happens when you actually feed it garbage.

The code review note from a few days ago said as much: "defensive and correct, but untested." That's the kind of statement that ages badly. So I wrote the tests.

## The obvious cases were fine

Null input, blank strings, truncated JSON, plain text, HTML fragments — `SnapshotParser.parse()` handled all of them exactly as designed. The catch block fires, logs at FINE, returns `Map.of()`. The pipeline continues with NEUTRAL relevance. Ten test cases, all green.

Then Claude ran the full suite and one case tripped: the JSON literal `"null"`.

## Jackson's quiet corner case

Jackson's `readValue("null", Map.class)` doesn't throw. It parses `"null"` as valid JSON — because it is valid JSON — and returns Java `null`. The try/catch never fires because there's no exception. The method returns `null` instead of `Map.of()`, and downstream code that calls `.containsKey()` on the result gets a `NullPointerException`.

This is the kind of bug that's hard to find by reading code. The defensive pattern *looks* complete: null/blank check at the top, try/catch around the parse, `Map.of()` on failure. Three layers of protection, and all three miss the same case — a parse that succeeds but produces `null`.

The fix is one line:

```java
Map<String, Object> result = MAPPER.readValue(att.relevanceSnapshot, Map.class);
return result != null ? result : Map.of();
```

## Why this matters beyond the fix

Trust scoring is the pipeline that decides whether an agent's attestations carry weight. If a single malformed snapshot can cause a `NullPointerException` in the relevance computation, it doesn't just fail that one attestation — it can take down the entire per-actor trust recomputation pass. One bad row in the database, and an actor's trust score stops updating.

The `SnapshotParser` extraction earlier this week consolidated the parsing logic into one place, which means this fix covers both the Beta path in `TrustScoreCalculator` and the EigenTrust path in `EigenTrustEdgeWeights`. Before the extraction, the same bug existed in two separate locations with two separate try/catch blocks, neither of which caught it.

The broader point: "defensive and correct" is a claim about the author's intent, not about the code's behaviour. The only way to verify it is to throw the worst inputs you can think of at it and see what comes out. Sometimes the worst input isn't the obviously malformed one — it's the edge case that's technically valid but semantically wrong.
