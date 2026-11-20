# Workbench Layout Migration: CSS Flexbox to Pages Layout Primitives

**Issue:** casehubio/chat-app#10
**Date:** 2026-08-10
**Status:** Draft

## Problem

The qhorus-workbench (~770 lines) implements all layout with bespoke CSS flexbox
and three hand-coded render methods (`_renderDesktop`, `_renderTablet`,
`_renderPhone`). Panel arrangement, dock strip rendering, responsive breakpoint
switching, drawer slide-in/out, and swipe gestures are all owned by this single
component. This creates:

- **Duplication across apps.** Any app needing a dockable panel layout must
  reimplement the same CSS, media queries, drawer transitions, and swipe handling.
- **No resize handles.** Desktop panels are fixed-width (240px nav, 220px member)
  with no drag-to-resize capability.
- **No layout persistence.** Panel open/close state resets on reload. Split ratios
  (once resize exists) would also need persistence.
- **Coupling.** Layout logic is interleaved with data management, WebSocket
  handling, and event coordination in one file.

## Solution

Migrate the workbench layout to pages-runtime. The workbench becomes a thin Lit
shell that delegates all panel arrangement to pages layout primitives
(`dockWorkbench()`, `split()`, `dockBar()`). Pages-runtime gaps are filled as
part of this work — improving pages benefits all apps.

Three vertical slices, each delivering a working layout mode:
1. Desktop — `dockWorkbench()` with split panes and drag handles
2. Tablet — exclusive-mode dock with sidebar switching
3. Phone — new drawer/overlay primitive with swipe gestures

## Non-Goals

- Changing the panel components themselves (task panel, correlation panel, etc.)
- Modifying the data flow (adapter, WebSocket, REST)
- Changing the event coordination pattern (panels communicate via workbench)
- Building a generic pages "responsive site" framework — this adds targeted
  primitives (shadow root rendering, drawer component type), not a full
  responsive engine

---

## 1. Architecture

### Before

```
QhorusWorkbenchElement (Lit, 770 lines)
├── CSS: flexbox layout, dock strip, drawers, backdrop, responsive
├── State: _dockState, _mode, _tabletTab, _drawerOpen
├── Layout: _renderDesktop(), _renderTablet(), _renderPhone()
├── Data: channels, messages, commitments, reactions, etc.
├── WebSocket: ConnectionController
├── Events: _onChatEvent dispatcher
└── Theme: injectTheme, applyThemeMode
```

### After

```
QhorusWorkbenchElement (Lit, thin shell — ~400 lines)
├── Breakpoint detection (MediaQueryList — existing)
├── Data state + adapter + WebSocket + events + theme (unchanged)
├── Builds pages component tree per breakpoint
└── Calls renderComponent() into shadow root

Pages-runtime (shared infrastructure)
├── renderComponent() — now supports ShadowRoot target
├── dockWorkbench() tree — split panes + dock bar + zones
├── Split: drag handles, ratio persistence, collapse
├── Dock: exclusive toggle, panel show/hide
├── Drawer: slide-in/out, backdrop, swipe (new primitive)
└── LayoutStore: localStorage persistence
```

### Workbench Shell Responsibilities

The Lit shell retains:
- **Data state** — channels, messages, commitments, reactions, members,
  presence, topics, view mode, selected message/channel/topic/artefact
- **Adapter** — `ChatDemoAdapter` with `onChange` callback
- **WebSocket** — `ConnectionController` for connection lifecycle
- **REST** — authenticated fetch for send/create/delete/react operations
- **Event coordination** — `_onChatEvent` dispatcher (panels → workbench → panels)
- **Theme** — dark/light toggle, token injection
- **Breakpoint detection** — `MediaQueryList` for desktop/tablet/phone
- **Tree building** — constructs the pages component tree for the current mode
- **Panel mounting** — creates Lit panel elements and mounts them into zones

The shell does NOT retain:
- Panel arrangement CSS (dock strip, nav-panel, member-panel widths)
- Drawer slide/backdrop/swipe CSS and logic
- Tablet tab switcher CSS and logic
- Phone header bar CSS
- Any `display: flex` layout rules for panel positioning

---

## 2. Desktop Layout (Slice 1)

### Pages Gaps to Fill

**Gap 1: Shadow root rendering.** `renderComponent()` in
`pages-component/src/renderer/render.ts` accepts `HTMLElement` only. Extend to
accept `ShadowRoot | HTMLElement`. The renderer creates child `<div>` elements
via `doc.createElement()` — this works identically in a shadow root since
`ShadowRoot` implements the `Node` interface with `appendChild()`.

**Gap 2: Host-panel component mounting.** The activation callback in
`pages-runtime/src/activation.ts` already creates Lit custom elements via
`document.createElement(tagName)` and sets properties. This is the mechanism
for mounting panel components (`<channel-nav>`, `<qhorus-task-panel>`, etc.)
into zones. Verify it works when the target is inside a shadow root.

### Component Tree

```typescript
dockWorkbench({
  centre: chatPanel,          // channel-feed + channel-input + topic-bar
  left: {
    panels: [
      { panelId: 'nav', icon: '💬', label: 'Channels', defaultOpen: true },
      { panelId: 'tasks', icon: '📋', label: 'Tasks', defaultOpen: false },
    ],
    exclusive: true,
  },
  right: {
    panels: [
      { panelId: 'members', icon: '👥', label: 'Members', defaultOpen: true },
      { panelId: 'correlation', icon: '🔗', label: 'Correlation', defaultOpen: false },
      { panelId: 'artifacts', icon: '📎', label: 'Artifacts', defaultOpen: false },
    ],
    exclusive: true,
  },
});
```

Both sides are exclusive — one panel visible per side at a time. Toggling the
active panel again collapses the zone entirely (split handle slides to edge).

### Split Behavior

The generated tree uses `split("horizontal", [...])` to arrange
dock-bar | left-zone | centre | right-zone. The existing `wireSplit()` in
`pages-component/src/renderer/interactive.ts` provides:
- Drag handles between panes (3px, `col-resize` cursor)
- Flex-based sizing with ratio persistence
- `pages-split-resize` events consumed by `LayoutStore`

Default ratios: `[0, 240, 1, 240]` (dock-bar auto-sized, left 240px,
centre flex-1, right 240px). User can drag to resize; ratios persist via
`LayoutStore`.

### Panel Mounting and Property Propagation

Each zone contains a `host-panel` component node. The activation callback
creates the Lit custom element (e.g., `<channel-nav>`, `<qhorus-task-panel>`)
and appends it to the zone div. The workbench shell keeps references to these
elements for reactive property updates:

```typescript
private _panelRefs = new Map<string, HTMLElement>();

private _buildDesktopTree() {
  // Build tree, register panel tag names
  // After renderComponent(), query and cache panel refs:
  for (const item of DOCK_ITEMS) {
    const el = (this.renderRoot as ShadowRoot)
      .querySelector(`[data-panel-id="${item.panelId}"]`);
    if (el) this._panelRefs.set(item.panelId, el as HTMLElement);
  }
}

override updated(changed: PropertyValues) {
  // Propagate data state to panel elements
  const nav = this._panelRefs.get('nav') as ChannelNavElement | undefined;
  if (nav) {
    nav.channels = this._channels;
    nav.selectedChannelId = this._selectedChannelId;
  }
  const tasks = this._panelRefs.get('tasks') as QhorusTaskPanelElement | undefined;
  if (tasks) {
    tasks.messages = this._filteredMessages();
    tasks.commitments = this._commitments;
    tasks.selectedMessageId = this._selectedMessageId;
  }
  // ... same pattern for members, correlation, artifacts
}
```

This replaces the current approach of inline Lit template bindings
(`.channels=${this._channels}`) with imperative property assignment. The
trade-off is explicit ref management, but it cleanly separates layout (pages)
from data flow (shell).

### LayoutStore Integration

```typescript
private _layoutStore = createLocalLayoutStore('qhorus-workbench:');
```

On `connectedCallback`: load saved `LayoutState` (split ratios + dock states).
On split-resize or dock-toggle events: save updated state. The existing
`pages-split-resize` and `pages-dock-toggle` event handlers in
`pages-runtime/src/site.ts` handle the persistence loop.

### Theme Toggle

The theme toggle button remains in the dock strip — it's a `DockItem` entry
but with no associated panel. On click, it toggles `_darkMode` and calls
`applyThemeMode()`. This is app-level behavior, not layout.

---

## 3. Tablet Layout (Slice 2)

### Pages Gaps to Fill

**Gap 3: Exclusive dock mode for constrained widths.** The dock bar already
supports `exclusive: true` per side. On tablet, the layout changes to:
dock-bar | sidebar (one panel) | centre. The sidebar shows whichever panel
is active in the dock — this is the same exclusive toggle as desktop, but
with a single combined zone instead of left/right split.

This may require no pages changes if the workbench shell simply builds a
different tree at the tablet breakpoint:

```typescript
dockWorkbench({
  centre: chatPanel,
  left: {
    panels: ALL_DOCK_ITEMS,   // all 5 panels in one exclusive zone
    exclusive: true,
  },
});
```

### Tree Structure

```
dock-bar | sidebar (exclusive, 280px) | centre (flex-1)
```

The sidebar shows the currently selected panel. The tab switcher UI from the
current implementation is replaced by the dock strip — each dock button
selects which panel appears in the sidebar. Simpler, consistent with desktop.

---

## 4. Phone Layout (Slice 3)

### Pages Gaps to Fill

**Gap 4: Drawer/overlay primitive.** New component type `drawer` in pages:
- Slides in from left or right edge
- Semi-transparent backdrop
- Swipe-to-dismiss gesture
- Only one drawer open at a time
- `prefers-reduced-motion` support

The drawer primitive encapsulates the CSS transitions, backdrop management,
and touch event handling currently in the workbench (`.drawer`, `.backdrop`,
`SwipeController`).

**Gap 5: Phone header bar.** A simple horizontal bar with icon buttons.
Could be a `columns()` layout or a dedicated `app-bar` primitive. Likely
`columns()` is sufficient.

### Tree Structure

```
app-bar (header: hamburger | channel-name | spacer | members | more)
centre (chat — full width)
drawer-left (nav panel)
drawer-right (members | correlation | artifacts | tasks — exclusive)
```

### SwipeController Migration

The existing `SwipeController` reactive controller handles edge-swipe
detection, velocity/distance thresholds, and `prefers-reduced-motion`. This
logic moves into the pages drawer primitive so all apps get swipe-to-dismiss
behavior.

---

## 5. Migration Path

### What Changes in the Workbench

| Current | After |
|---------|-------|
| `static styles = css\`...\`` (100+ lines of layout CSS) | Minimal CSS: `:host { display: block; height: 100% }` |
| `_renderDesktop()` (15 lines of flexbox template) | `_buildDesktopTree()` → `renderComponent()` |
| `_renderTablet()` (20 lines of tab switcher template) | `_buildTabletTree()` → `renderComponent()` |
| `_renderPhone()` (20 lines of drawer template) | `_buildPhoneTree()` → `renderComponent()` |
| `_dockState: Record<string, boolean>` | `LayoutState.docks` via `LayoutStore` |
| `_tabletTab: string` | Eliminated — exclusive dock handles this |
| `_drawerOpen: string \| null` | Drawer primitive state |
| `SwipeController` | Moves to pages drawer primitive |
| `_toggleDock()` with mode branching | `pages-dock-toggle` events — pages handles the mode |
| Manual dock strip HTML | `dockBar()` component in the tree |
| Fixed panel widths (240px, 220px) | Resizable splits with ratio persistence |

### What Stays Unchanged

- `ChatDemoAdapter` and all data state management
- `ConnectionController` for WebSocket lifecycle
- All REST methods (`_sendMessage`, `_createChannel`, etc.)
- `_onChatEvent` dispatcher and all event handling
- Panel components (`qhorus-task-panel`, `qhorus-correlation-panel`, etc.)
- Theme toggle logic
- `responsive.ts` media query constants

### File Changes

| File | Change |
|------|--------|
| `qhorus-workbench.ts` | Rewrite render methods → tree builders, remove layout CSS |
| `swipe-controller.ts` | Eventually removed (logic moves to pages) |
| `responsive.ts` | Unchanged (breakpoint constants still used) |
| All panel components | Unchanged |
| `chat-demo-adapter.ts` | Unchanged |
| `connection-controller.ts` | Unchanged |

### Pages Changes (Separate Repo)

| Module | Change |
|--------|--------|
| `pages-component/renderer/render.ts` | Accept `ShadowRoot \| HTMLElement` as target |
| `pages-component/renderer/interactive.ts` | Verify split/dock wiring works in shadow DOM |
| `pages-runtime/src/activation.ts` | Verify host-panel mounting in shadow DOM |
| `pages-ui/src/dsl/builders.ts` | Possibly extend `dockWorkbench()` for exclusive sides |
| `pages-component/src/model/` | New `DrawerProps` type |
| `pages-component/renderer/` | New `wireDrawer()` interactive handler |
| `pages-runtime/src/activation.ts` | Drawer activation (slide, backdrop, swipe) |

---

## 6. Testing Strategy

### Pages-Level Tests

- `renderComponent()` renders into a `ShadowRoot` (jsdom or happy-dom)
- Split drag updates ratios and emits `pages-split-resize`
- Dock toggle in exclusive mode hides previous panel, shows new one
- Drawer opens/closes with correct CSS transitions
- LayoutStore round-trips split ratios and dock states

### Workbench-Level Tests

- Tree builder produces correct structure for each breakpoint
- Panel components receive correct props after data state changes
- Dock toggle changes visible panel (integration with pages events)
- Split resize persists via LayoutStore
- Phone drawer opens on header button click
- Theme toggle still works

### Regression

- All existing workbench tests continue to pass
- All existing pages tests continue to pass

---

## 7. Build Order

### Slice 1: Desktop (prerequisite: pages in slot)

1. **Pages: shadow root rendering** — extend `renderComponent()` target type
2. **Pages: verify split + dock in shadow DOM** — integration test
3. **Pages: exclusive dock mode** — if not already supported by
   `dockWorkbench()` config
4. **Chat-app: desktop tree builder** — `_buildDesktopTree()` using
   `dockWorkbench()`
5. **Chat-app: shell integration** — `renderComponent()` in `render()`,
   property propagation in `updated()`
6. **Chat-app: remove desktop layout CSS** — verify resizable splits work
7. **Chat-app: LayoutStore wiring** — persist split ratios and dock state

### Slice 2: Tablet

1. **Chat-app: tablet tree builder** — `_buildTabletTree()` with single
   exclusive sidebar
2. **Chat-app: remove tablet layout CSS** — tab switcher eliminated
3. **Verify** dock strip drives sidebar panel switching

### Slice 3: Phone

1. **Pages: drawer primitive** — component type, CSS transitions, backdrop
2. **Pages: swipe gesture handler** — port from `SwipeController`
3. **Chat-app: phone tree builder** — `_buildPhoneTree()` with drawers
4. **Chat-app: remove phone layout CSS** — drawers, backdrop, phone header
5. **Chat-app: remove SwipeController** — logic now in pages
6. **Verify** swipe gestures and drawer behavior
