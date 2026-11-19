## D1: Primary embedding context

**Choice:** Settings page as primary context
**Alternatives:**
- Onboarding flow — guides rather than manages; different layout but same primitives
- All three equally — over-engineers the initial design
**Rationale:** Settings page is the most common embedding. Onboarding and admin follow the same component patterns with different composition.
**Trade-offs:** Onboarding-specific affordances (progress, required-before-proceed) are deferred.
**Sources:** casehubio/casehub-mosaic#224 body, casehubio/casehub-mosaic#328
**Exploration:** quick
**Status:** captured

## D2: Unified OAuth + passkeys dashboard

**Choice:** One connection dashboard shows OAuth providers AND passkeys as different credential types
**Alternatives:**
- Split — separate components for OAuth and passkeys; different mental models displayed in separate sections
- OAuth only — defer passkey UI entirely
**Rationale:** User sees everything in one place. Both are "ways I'm connected to this system." The primitives are different (redirect vs browser prompt) but the management UX is the same.
**Trade-offs:** Dashboard component is more complex; needs to handle two distinct interaction patterns.
**Sources:** platform-api ServiceConnectionProvider, WebAuthnCredentialStore
**Exploration:** quick
**Status:** captured

## D3: OAuth popup with redirect fallback

**Choice:** Try popup window first, fall back to full-page redirect if blocked
**Alternatives:**
- Full redirect only — simpler but navigates away, loses page state
- Popup only — rejects browsers that block popups
**Rationale:** Popup keeps user on page (better UX for settings context). Redirect fallback handles popup blockers gracefully.
**Trade-offs:** More implementation complexity — popup lifecycle, postMessage listener, blocker detection.
**Sources:** casehubio/casehub-mosaic#226 body
**Exploration:** quick
**Status:** captured

## D4: Types in mosaic-core

**Choice:** Add connection.ts and credential.ts to mosaic-core/src/types/ alongside identity.ts
**Alternatives:**
- New mosaic-connection package — more isolation but unnecessary overhead for a few interfaces
- Inline in components — duplication across 6+ components
**Rationale:** These are platform domain types shared by multiple components. mosaic-core already has types/identity.ts with WorkIdentity — same pattern.
**Trade-offs:** mosaic-core gains more type surface, but these types are stable (mirror Java records).
**Sources:** mosaic-core/src/types/identity.ts, platform-api ServiceConnection.java, WebAuthnCredential.java
**Exploration:** quick
**Status:** captured

## D5: Props + events data layer

**Choice:** Components are pure views — receive data as properties, emit events for mutations
**Alternatives:**
- Endpoint URL + self-fetch — more self-contained but couples to REST shape, handles loading/error/auth internally
**Rationale:** Matches approval-gate pattern. No CORS/auth concerns inside components. Host app owns fetch lifecycle. Testable with static data.
**Trade-offs:** Every consumer must wire up data fetching. Mitigated by pre-configured pages providing a ready-made composition.
**Depends on:** D8 (pre-configured pages provide the wired-up experience)
**Sources:** approval-gate component pattern, platform#898 (REST endpoints)
**Exploration:** quick
**Status:** captured

## D6: Fine-grained primitives + pre-configured pages

**Choice:** Build small composable primitives (provider-status-row, connect-button, scope-list, passkey-row, passkey-register-button). Ship pre-configured pages that compose them. Consumers reuse pages or compose their own.
**Alternatives:**
- Keep all 6 original issues as separate components — maximum granularity but some are too thin
- Consolidate to 2 large components — simpler but less reusable, doesn't serve MCP server use case
**Rationale:** Two consumers: full platform apps and standalone MCP server (#328). Primitives serve both. Pages reduce boilerplate for common cases.
**Trade-offs:** More components to build and test. Justified by the two distinct consumer contexts.
**Sources:** casehubio/casehub-mosaic#328 (MCP server auth page), casehubio/casehub-mosaic#224 (platform epic)
**Exploration:** quick
**Status:** captured

## D6a: Packaging — single component package

**Choice:** Primitives are internal modules within a single `components/connection-manager/` package. Pre-configured pages are peer exports. Login page is a separate `components/login-page/` package.
**Alternatives:**
- Separate packages per primitive — 7+ new packages for one epic, inconsistent with mosaic convention
**Rationale:** Existing mosaic components are each a standalone package under `components/`. Primitives are internal building blocks, not independently consumable. Consumers import from `connection-manager` and get both primitives and the management page.
**Trade-offs:** Primitives can't be tree-shaken independently. Acceptable — they're small.
**Depends on:** D6 (fine-grained primitives + pages)
**Sources:** mosaic component structure (approval-gate, service-card patterns)
**Exploration:** quick (decision-review refinement)
**Status:** captured

## D7: Standalone OAuthFlowController

**Choice:** OAuthFlowController as a standalone non-visual class. connect-button uses it internally. Any component can instantiate it directly.
**Alternatives:**
- Embedded in connect button only — simpler API but couples flow logic to one UI element
**Rationale:** The flow logic (popup, postMessage, state verification, redirect fallback) is reusable. connect-button is just one consumer — login-page needs it too.
**Trade-offs:** None significant — option 2 is just option 1 composed.
**Sources:** casehubio/casehub-mosaic#226 body
**Exploration:** quick
**Status:** captured

## D8: Two pre-configured pages

**Choice:** connection-management-page (manage existing connections, settings/MCP context) and login-page (authenticate, sign-in context)
**Alternatives:**
- One adaptive page — conflates management and authentication intents
- Three pages (add standalone consent page) — consent is a flow within management, not its own page
**Rationale:** Different user intents: "manage my connections" vs "sign in." Different layouts, same primitives.
**Trade-offs:** Two page components to maintain.
**Depends on:** D6 (pages compose primitives)
**Sources:** casehubio/casehub-mosaic#328, casehubio/casehub-mosaic#224
**Exploration:** quick
**Status:** captured

## D9: Built-in ProviderBrandRegistry with trademark notice

**Choice:** Ship inline SVGs for Google, GitHub, Slack, Discord with trademark exclusion in LICENSE. Apple as a slot only (they provide their own JS-rendered button). Consumers can override or add providers.
**Alternatives:**
- Consumer-supplied only — zero trademark concern but more work for every consumer
**Rationale:** Standard open-source approach (Auth.js, social-login-buttons). Nominative use for login buttons is well-established. Trademark notice makes licensing clear.
**Trade-offs:** Must maintain SVGs and stay current with brand guidelines. Apple requires special handling.
**Sources:** Auth.js (next-auth) repo, AI-Dico/social-login-buttons LICENSE pattern, provider brand guidelines
**Exploration:** deep-analysis (web research on licensing)
**Status:** captured
