## D1: Strategy location

**Choice:** Test sources — `TestAppraisalStrategy` in `cognition/src/test/java/.../eval/`
**Alternatives:**
- Production sources alongside `NoOpAppraisalStrategy` — available for offline/fallback but production uses LLM-based strategy; unnecessary API surface
**Rationale:** This is a test fixture for divergence testing, not a production capability
**Trade-offs:** Not reusable outside test code; if a production rule-based fallback is ever needed, it would be a separate implementation
**Sources:** `NoOpAppraisalStrategy.java`, `LlmAppraisalStrategy.java`, `AppraisalStrategy.java`
**Exploration:** quick
**Status:** captured
