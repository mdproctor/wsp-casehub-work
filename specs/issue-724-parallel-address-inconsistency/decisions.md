## D1: Align scanner to step-click's address convention for parallel lanes

**Choice:** Add unnamed-branch handling to `StepPositionScanner.scanNamedBranches` so it produces `0.lane.N.0` addresses for array-format parallel steps, matching step-click's existing convention.
**Alternatives:**
- Drop unnamed parallel support from step-click — breaks valid array-format YAML
- Extract shared address utility — over-engineering for a one-line scanner fix
**Rationale:** The scanner is the incomplete side. step-click already handles both named (`0.lane.agent.0`) and unnamed (`0.lane.0.0`) correctly. The scanner only handles named keys, skipping list items entirely. Fixing the scanner is minimal and non-breaking.
**Trade-offs:** Scanner gains slightly more complexity in `scanNamedBranches` (or a parallel-specific path).
**Sources:** `packages/yaml-core/src/annotation/step-position-scanner.ts:170-213`, `packages/pages-step-viewer/src/step-click.ts:54-66`, `packages/yaml-core/src/step/walker.ts:474-488`
**Exploration:** quick
**Status:** captured
