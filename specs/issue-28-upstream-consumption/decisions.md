## D1: Integration model — pages-runtime hosted

**Choice:** Migrate the workbench to be rendered by pages-runtime, with Lit panel components mounted into generated DOM zones. The workbench becomes a thin Lit shell delegating layout to pages.
**Alternatives:**
- Lit-hosted, pages-data only — keep bespoke CSS layout, use only DockItem/LayoutState for state. Moves away from the goal of pages as the universal layout engine.
- Hybrid Lit wrappers — build `<pages-split-pane>` etc. as Lit components wrapping imperative rendering. Creates a wrapper layer that pages should eventually own.
**Rationale:** Pages is the strategic layout engine. Making it capable enough for the workbench benefits all apps, not just chat-app.
**Trade-offs:** Requires filling gaps in pages-runtime (shadow root support, responsive layout). More upfront work, but no throwaway code.
**Exploration:** quick
**Status:** captured

## D2: Gap-filling philosophy — improve pages, don't work around

**Choice:** When pages lacks a capability the workbench needs, add it to pages rather than building a bespoke workaround in chat-app.
**Alternatives:**
- Workaround in chat-app — faster for this project but creates tech debt and doesn't benefit other apps.
**Rationale:** User principle: "always look to improve pages, as it means all apps benefit."
**Trade-offs:** Slower initial delivery. Requires changes to the pages repo (peer repo — needs separate slot or slot expansion).
**Exploration:** quick
**Status:** captured

## D3: Phone layout — unified via pages drawer primitive

**Choice:** All three layout modes (desktop, tablet, phone) use pages layout primitives. Phone layout uses a new drawer/overlay primitive in pages rather than bespoke CSS in the workbench.
**Alternatives:**
- Split boundary — desktop/tablet use pages, phone keeps existing drawer/swipe CSS. Simpler but two layout systems coexist.
**Rationale:** Same principle as D2. A pages drawer primitive benefits any app that needs mobile-responsive layout.
**Trade-offs:** Requires designing and building a drawer primitive in pages (new component type).
**Depends on:** D1 (pages-runtime hosted), D2 (improve pages)
**Exploration:** quick
**Status:** captured

## D4: Sequencing — co-evolve in vertical slices (Approach B)

**Choice:** Build pages capabilities and migrate the workbench incrementally, one layout mode at a time: desktop first (shadow root + dockWorkbench), tablet second (exclusive-mode dock), phone last (drawer primitive).
**Alternatives:**
- Pages-first, then migrate — build all pages capabilities upfront, then rewrite the workbench. Clean separation but blocks chat-app progress and delays validation.
**Rationale:** Each slice delivers a working layout mode and validates the pages additions with a real consumer. The temporary coexistence is acceptable since each mode already renders independently.
**Trade-offs:** Temporary coexistence of two layout systems during migration. Each slice must leave the workbench fully functional.
**Depends on:** D1, D2, D3
**Exploration:** quick
**Status:** captured

## D5: Panel exclusivity — both sides exclusive

**Choice:** Both left and right panel zones use exclusive switching — one panel visible per side at a time. Left: nav or tasks. Right: members, correlation, or artifacts. Toggling the active panel again hides the zone entirely.
**Alternatives:**
- Simultaneous stacking — allow multiple panels open per side with split stacking. Creates cramped layout (240px left, 220px×N right) with no cross-panel workflow that justifies it.
- Right exclusive, left simultaneous — keep nav always visible while tasks opens. But nav is transient (pick channel, done) and doesn't need persistent space.
**Rationale:** First-principles analysis: nav is transient, tasks is monitoring (both for current channel only), correlation and artifacts are message-context panels triggered on demand. No workflow requires seeing two panels on the same side simultaneously. Exclusive switching maps directly to dockWorkbench() with exclusive: true per side — no sub-zones, no API changes.
**Trade-offs:** Cannot see channel list and task list simultaneously. Acceptable because switching channels already updates the task list.
**Depends on:** D1
**Exploration:** quick
**Status:** captured
