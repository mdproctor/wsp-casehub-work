## D1: Prompt restructuring strategy

**Choice:** Formation-anchored prompt structure — restructure system prompt so formation memories are the organizing anchor, demote contrast from primary differentiator
**Alternatives:**
- Dual-pass with exemplar injection — less disruptive but exemplars fight against existing emphasis hierarchy rather than reshaping it
**Rationale:** Root cause is prompt structure weighting, not missing content. The emphasis hierarchy puts contrast first and formation memories last. Exemplars alone can't overcome that.
**Trade-offs:** Might occasionally reduce body-territory specificity, but body territory remains required as the assigned location — it just stops being the dominant signal.
**Sources:** DriveDescriptionGenerationPhase.java:55-85 (SYSTEM_PROMPT), DriveDescriptionGenerationPhase.java:398-403 (user prompt formation memory section), neocortex#735 issue body
**Exploration:** quick
**Status:** captured

## D2: Sentence budget

**Choice:** Keep 3-5 sentences, force integration over appending
**Alternatives:**
- Increase to 4-6 sentences — more room but risks bloating observation context passed to character agent loop
**Rationale:** The LLM should weave formation memory body echoes into the existing structure, not append a separate formation memory paragraph. Constraint drives quality.
**Trade-offs:** Tighter budget means each sentence must do more work — formation echo, body territory, and channel texture all in 3-5 sentences.
**Sources:** neocortex#735 acceptance criteria, CognitionObservationBuilder.java:164-206 (observation assembly)
**Exploration:** quick
**Depends on:** D1 (prompt structure determines how budget is used)
**Status:** captured
