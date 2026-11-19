# Connection & Identity UI — Design Spec

**Epic:** casehubio/casehub-mosaic#224
**Branch:** `issue-224-connection-identity-ui`
**Date:** 2026-11-19

## Problem

CaseHub apps need UI for managing service connections (OAuth providers like Google, GitHub, Slack) and identity credentials (WebAuthn passkeys). The platform has clean SPIs (`ServiceConnectionProvider`, `WebAuthnCredentialStore`, `IncrementalConsentHandler`, `ScopeRegistry`) but no frontend components. Two distinct consumers need this UI:

1. **Platform apps** — settings pages with full mosaic stack, `--pages-*` tokens, Lit components
2. **Standalone MCP server** (casehubio/casehub-mosaic#328) — lightweight self-contained connection management page

## Solution

Fine-grained Lit primitives composed into pre-configured pages. Components are pure views: data in via properties, user actions out via `pages-event`. Two ready-made pages serve the two primary contexts. Consumers can use a page directly or compose their own from primitives.

## Architecture

### Types (mosaic-core)

New files in `packages/mosaic-core/src/types/`:

**connection.ts:**
```typescript
export type ServiceConnectionStatus = 'CONNECTED' | 'PARTIAL' | 'DISCONNECTED';

export interface ServiceConnection {
  readonly provider: string;
  readonly status: ServiceConnectionStatus;
  readonly grantedScopes: readonly string[];
  readonly missingScopes: readonly string[];
  readonly connectedAt: string | null;
}

export interface ConsentRequest {
  readonly authorizationUrl: string;
  readonly provider: string;
  readonly requestedScopes: readonly string[];
  readonly state: string;
}

export interface ProviderBrand {
  readonly providerId: string;
  readonly displayName: string;
  readonly icon: string; // inline SVG markup
  readonly brandColor: string;
}
```

**credential.ts:**
```typescript
export interface WebAuthnCredential {
  readonly credentialId: string;
  readonly displayName: string;
  readonly transports: readonly string[];
  readonly discoverable: boolean;
  readonly createdAt: string;
  readonly lastUsedAt: string | null;
}
```

Both re-exported from `mosaic-core/src/types/index.ts`.

### Component Packages

#### `components/connection-manager/`

Single package exporting all primitives and the connection management page.

**File structure:**
```
components/connection-manager/
  src/
    primitives/
      provider-status-row.ts     # single provider: icon, name, status, scopes, action
      connect-button.ts          # branded "Connect with X" button
      scope-list.ts              # expandable granted/missing scope list
      passkey-row.ts             # single passkey: device, last used, transports, revoke
      passkey-register-button.ts # triggers WebAuthn registration ceremony
    controllers/
      oauth-flow-controller.ts   # popup + redirect fallback, state verification
      provider-brand-registry.ts # provider → { icon, color, displayName }
    connection-management-page.ts  # pre-configured page composing all primitives
    index.ts                       # re-exports everything
    types.ts                       # component-local types (event payloads, internal state)
  demo.html
```

**Exports:**
- All primitives (for custom composition)
- `OAuthFlowController` (for custom OAuth triggers)
- `ProviderBrandRegistry` (for brand lookup/override)
- `ConnectionManagementPage` (ready-made settings page)

#### `components/login-page/`

Separate package for the authentication context.

**File structure:**
```
components/login-page/
  src/
    login-page.ts           # social login buttons + passkey sign-in
    index.ts
  demo.html
```

Imports `connect-button` and `passkey-register-button` from `connection-manager`.

### Primitives — Detail

#### `provider-status-row`

Renders one row in the connection dashboard.

| Property | Type | Description |
|----------|------|-------------|
| `connection` | `ServiceConnection` | Provider connection data |
| `brand` | `ProviderBrand \| null` | Branding override (falls back to registry) |

| Event | Payload | When |
|-------|---------|------|
| `connection.connect` | `{ provider: string }` | User clicks Connect |
| `connection.disconnect` | `{ provider: string }` | User clicks Disconnect |
| `connection.consent` | `{ provider: string, missingScopes: string[] }` | User clicks Grant Access |

ARIA: `role="listitem"` within a `role="list"` container, `aria-label` with provider name and status.

**Layout:** Provider icon (from registry or brand prop) | Provider name | Status badge (Connected/Partial/Disconnected) | Expandable scope list | Action button (Connect/Grant Access/Disconnect).

#### `connect-button`

Branded OAuth button. Used in both management page (as action in provider-status-row) and login page (as "Sign in with X").

| Property | Type | Description |
|----------|------|-------------|
| `provider` | `string` | Provider ID |
| `mode` | `'connect' \| 'signin'` | Button label: "Connect with X" vs "Sign in with X" |
| `authorizationUrl` | `string \| null` | OAuth URL. When set, clicking triggers the OAuth flow |
| `brand` | `ProviderBrand \| null` | Branding override |
| `disabled` | `boolean` | Disabled state |

Internally instantiates `OAuthFlowController` when clicked with an `authorizationUrl`.

| Event | Payload | When |
|-------|---------|------|
| `connection.connect` | `{ provider: string }` | Clicked without authorizationUrl (host should fetch URL) |
| `oauth.complete` | `{ provider: string, state: string, code: string }` | OAuth flow completed successfully |
| `oauth.error` | `{ provider: string, error: string }` | OAuth flow failed or was cancelled |

ARIA: `role="button"`, `aria-label="Connect with {displayName}"` or `"Sign in with {displayName}"`.

**Brand styling:** Background uses `brandColor` from registry. Icon rendered inline. Follows provider guidelines (Google's specific "G" mark, GitHub's Invertocat, etc.).

#### `scope-list`

Expandable list showing granted and missing scopes.

| Property | Type | Description |
|----------|------|-------------|
| `grantedScopes` | `readonly string[]` | Scopes already granted |
| `missingScopes` | `readonly string[]` | Scopes not yet granted |
| `expanded` | `boolean` | Whether the list is expanded |

ARIA: `aria-expanded` on toggle, `role="list"` for scope items. Missing scopes visually distinguished (warning color).

#### `passkey-row`

One registered passkey credential.

| Property | Type | Description |
|----------|------|-------------|
| `credential` | `WebAuthnCredential` | Credential data |

| Event | Payload | When |
|-------|---------|------|
| `passkey.revoke` | `{ credentialId: string }` | User clicks Revoke |

ARIA: `role="listitem"`, `aria-label` with device name.

**Layout:** Device name | Transports (USB, BLE, NFC, internal) as icons | Created date | Last used | Revoke button.

#### `passkey-register-button`

Triggers the WebAuthn registration ceremony (browser-native prompt).

| Property | Type | Description |
|----------|------|-------------|
| `registrationOptions` | `object \| null` | PublicKeyCredentialCreationOptions from server. When set, clicking triggers navigator.credentials.create() |
| `disabled` | `boolean` | Disabled state |

| Event | Payload | When |
|-------|---------|------|
| `passkey.register-request` | `{}` | Clicked without registrationOptions (host should fetch options) |
| `passkey.registered` | `{ credential: PublicKeyCredential }` | Registration completed |
| `passkey.register-error` | `{ error: string }` | Registration failed or cancelled |

ARIA: `role="button"`, `aria-label="Register a passkey"`.

### Controllers — Detail

#### `OAuthFlowController`

Non-visual class managing the OAuth popup/redirect lifecycle.

```typescript
interface OAuthFlowOptions {
  authorizationUrl: string;
  state: string;
  popupWidth?: number;   // default 500
  popupHeight?: number;  // default 600
  redirectFallback?: boolean; // default true
}

interface OAuthFlowResult {
  code: string;
  state: string;
}

class OAuthFlowController {
  start(options: OAuthFlowOptions): Promise<OAuthFlowResult>;
  cancel(): void;
}
```

**Flow:**
1. Open popup window with `authorizationUrl`
2. If popup blocked and `redirectFallback` is true, navigate current page (save return URL in sessionStorage)
3. Listen for `postMessage` from popup with authorization code
4. Verify `state` matches
5. Resolve promise with `{ code, state }`
6. On error or cancel, reject with descriptive error

**Return from redirect:** On page load, check sessionStorage for pending OAuth state. If found, extract code from URL params, clean up, and emit result via event.

#### `ProviderBrandRegistry`

Static registry mapping provider IDs to branding.

```typescript
interface ProviderBrandEntry {
  displayName: string;
  icon: string;        // inline SVG
  brandColor: string;  // hex color
}

class ProviderBrandRegistry {
  static get(providerId: string): ProviderBrandEntry | null;
  static register(providerId: string, entry: ProviderBrandEntry): void;
  static has(providerId: string): boolean;
}
```

**Built-in providers:** google, github, slack, discord. Each with official SVG mark and brand color.

**Apple:** Not registered. Consumers add Apple via `register()` or use Apple's Sign in with Apple JS directly. A comment in the registry explains why.

**Trademark notice:** Package includes `TRADEMARK-NOTICE.md` stating logos are trademarks of their respective owners and excluded from the MIT license.

### Pre-configured Pages — Detail

#### `connection-management-page`

Composes all primitives into a complete connection management surface.

| Property | Type | Description |
|----------|------|-------------|
| `connections` | `readonly ServiceConnection[]` | All provider connections |
| `credentials` | `readonly WebAuthnCredential[]` | All passkey credentials |
| `providers` | `readonly string[]` | Available provider IDs (from ScopeRegistry) |

| Event | Payload | When |
|-------|---------|------|
| All primitive events | (see above) | Bubbled from composed primitives |

**Layout:**
```
┌─────────────────────────────────────────┐
│  Service Connections                     │
│  ┌─────────────────────────────────────┐│
│  │ 🔵 Google   Connected    [Manage ▾] ││
│  │ ⚫ GitHub   Partial      [Grant]    ││
│  │ 🟣 Slack    Disconnected [Connect]  ││
│  └─────────────────────────────────────┘│
│                                         │
│  Passkeys                               │
│  ┌─────────────────────────────────────┐│
│  │ MacBook Pro Touch ID  · 2d ago      ││
│  │ YubiKey 5C NFC        · 14d ago     ││
│  └─────────────────────────────────────┘│
│  [+ Register passkey]                   │
└─────────────────────────────────────────┘
```

ARIA: `role="region"` with `aria-label="Connection management"`. Service connections section uses `role="list"`. Passkeys section uses `role="list"`.

#### `login-page`

Authentication-context page with social login and passkey sign-in.

| Property | Type | Description |
|----------|------|-------------|
| `providers` | `readonly string[]` | Available provider IDs |
| `passkeyEnabled` | `boolean` | Whether to show passkey sign-in option |

| Event | Payload | When |
|-------|---------|------|
| `connection.connect` | `{ provider: string }` | Social login button clicked |
| `passkey.authenticate` | `{}` | Passkey sign-in initiated |

**Layout:**
```
┌─────────────────────────────┐
│         Sign in             │
│                             │
│  [🔵 Sign in with Google ]  │
│  [⚫ Sign in with GitHub ]  │
│  [🟣 Sign in with Slack  ]  │
│                             │
│  ─── or ───                 │
│                             │
│  [🔑 Sign in with passkey]  │
└─────────────────────────────┘
```

ARIA: `role="main"` with `aria-label="Sign in"`. Buttons in `role="group"` with `aria-label="Sign in options"`.

### Event Protocol

All events use `emitPagesEvent` from `@casehubio/pages-data`. Event topics:

| Topic | Payload | Source |
|-------|---------|--------|
| `connection.connect` | `{ provider: string }` | connect-button, provider-status-row |
| `connection.disconnect` | `{ provider: string }` | provider-status-row |
| `connection.consent` | `{ provider: string, missingScopes: string[] }` | provider-status-row |
| `oauth.complete` | `{ provider: string, state: string, code: string }` | connect-button (via OAuthFlowController) |
| `oauth.error` | `{ provider: string, error: string }` | connect-button (via OAuthFlowController) |
| `passkey.register-request` | `{}` | passkey-register-button |
| `passkey.registered` | `{ credential: PublicKeyCredential }` | passkey-register-button |
| `passkey.register-error` | `{ error: string }` | passkey-register-button |
| `passkey.revoke` | `{ credentialId: string }` | passkey-row |
| `passkey.authenticate` | `{}` | login-page |

### Styling

All components use `--pages-*` CSS custom properties. No hardcoded colors — theme-agnostic fallbacks per mosaic convention (D4 from #316).

Provider brand colors are used only on connect buttons and are applied via the `ProviderBrandRegistry`, not CSS tokens.

### Testing

Each primitive gets its own test file:
- ARIA assertions (role, aria-label, aria-expanded)
- Event emission on user interaction
- Property reflection
- Status badge rendering for each `ServiceConnectionStatus`
- Brand registry lookup and fallback

Pages get integration tests:
- Correct composition of primitives
- Event bubbling from primitives through page

`OAuthFlowController` gets unit tests:
- Popup open and postMessage handling
- State verification
- Redirect fallback detection
- Cancel behavior

### Dependency Graph

```
mosaic-core (types)
    ↑
connection-manager (primitives + controllers + management page)
    ↑
login-page (composes connect-button + passkey-register-button)
```

External dependencies: `lit`, `@casehubio/pages-data`, `@casehubio/pages-primitives` (a11y mixins).

### REST Endpoints (platform#898)

The platform needs REST endpoints to serve these components. Filed as casehubio/platform#898. The mosaic components are decoupled from the REST surface — host apps wire events to their own fetch layer.

### Issue Mapping

| Original issue | Covered by |
|----------------|------------|
| #225 Connect to Provider button | `connect-button` primitive |
| #226 OAuth redirect/popup flow | `OAuthFlowController` |
| #227 Connection dashboard | `connection-management-page` |
| #228 Passkey enrollment and management | `passkey-row` + `passkey-register-button` |
| #229 Social login button set | `connect-button` with `mode="signin"` |
| #230 Connection consent screen | `scope-list` + `connection.consent` event in `provider-status-row` |
| #232 Connection status component | `provider-status-row` primitive |
| #328 MCP server auth page | `connection-management-page` (standalone consumer) |

## References

- `platform-api/src/main/java/io/casehub/platform/api/authn/ServiceConnectionProvider.java` — connection SPI
- `platform-api/src/main/java/io/casehub/platform/api/authn/WebAuthnCredentialStore.java` — passkey SPI
- `platform-api/src/main/java/io/casehub/platform/api/authn/IncrementalConsentHandler.java` — consent SPI
- `platform-api/src/main/java/io/casehub/platform/api/authn/ScopeRegistry.java` — provider discovery
- `components/approval-gate/src/approval-gate.ts` — reference component (props+events pattern)
- `components/service-card/src/service-card.ts` — reference component (status rendering)
- `packages/mosaic-core/src/types/identity.ts` — existing identity types
- casehubio/platform#898 — REST endpoints (dependency)
- casehubio/casehub-mosaic#328 — MCP server auth page (consumer)
- Auth.js (next-auth) — provider brand SVG licensing precedent
- AI-Dico/social-login-buttons — trademark notice pattern
