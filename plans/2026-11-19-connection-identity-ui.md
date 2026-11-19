# Connection & Identity UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #224 — Epic: Connection & Identity UI components
**Issue group:** #225, #226, #227, #228, #229, #230, #232, #328

**Goal:** Build composable Lit primitives for OAuth connection management and WebAuthn passkey enrollment, plus two pre-configured pages (connection management and login).

**Architecture:** Fine-grained primitives in a single `connection-manager` component package, composed into pre-configured pages. Types added to `mosaic-core`. Components are pure views (props in, `pages-event` out). `OAuthFlowController` handles popup+redirect OAuth with origin verification. `ProviderBrandRegistry` ships inline SVGs for Google/GitHub/Slack/Discord.

**Tech Stack:** Lit 3, TypeScript 5.8, Vitest, `@casehubio/pages-data` (emitPagesEvent), `@casehubio/pages-primitives` (a11y mixins)

## Global Constraints

- All components MUST have ARIA attributes (`role`, `aria-label`, state attrs)
- All components use `--pages-*` CSS custom properties with theme-agnostic fallbacks
- Custom element prefix: `mosaic-`
- Event emission via `emitPagesEvent(this, topic, payload)` from `@casehubio/pages-data`
- No hardcoded light-mode hex fallbacks
- Package naming: `@casehubio/mosaic-connection-manager`, `@casehubio/mosaic-login-page`
- Provider brand SVGs excluded from MIT license (TRADEMARK-NOTICE.md)

---

## Batch 1: Foundation — types + brand registry + package scaffold

After this batch: mosaic-core exports connection/credential types, the connection-manager package exists with the brand registry, and both compile and pass tests.

### Task 1: Add connection and credential types to mosaic-core

**Files:**
- Create: `packages/mosaic-core/src/types/connection.ts`
- Create: `packages/mosaic-core/src/types/credential.ts`
- Modify: `packages/mosaic-core/src/types/index.ts`
- Test: `packages/mosaic-core/src/types/connection.test.ts`
- Test: `packages/mosaic-core/src/types/credential.test.ts`

**Interfaces:**
- Produces: `ServiceConnectionStatus`, `ServiceConnection`, `ConsentRequest`, `ProviderBrand`, `AuthenticatorTransport`, `WebAuthnCredential` — all re-exported from `@casehubio/mosaic-core`

- [ ] **Step 1: Write connection.ts types**

```typescript
// packages/mosaic-core/src/types/connection.ts
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
  readonly icon: string;
  readonly brandColor: string;
}
```

- [ ] **Step 2: Write credential.ts types**

```typescript
// packages/mosaic-core/src/types/credential.ts
export type AuthenticatorTransport = 'usb' | 'ble' | 'nfc' | 'internal' | 'hybrid';

export interface WebAuthnCredential {
  readonly credentialId: string;
  readonly displayName: string;
  readonly transports: readonly AuthenticatorTransport[];
  readonly discoverable: boolean;
  readonly createdAt: string;
  readonly lastUsedAt: string | null;
}
```

- [ ] **Step 3: Re-export from types/index.ts**

Add to `packages/mosaic-core/src/types/index.ts`:
```typescript
export * from './connection.js';
export * from './credential.js';
```

- [ ] **Step 4: Write type tests**

```typescript
// packages/mosaic-core/src/types/connection.test.ts
import { describe, it, expectTypeOf } from 'vitest';
import type { ServiceConnection, ServiceConnectionStatus, ConsentRequest, ProviderBrand } from './connection.js';

describe('connection types', () => {
  it('ServiceConnectionStatus is a string union', () => {
    expectTypeOf<ServiceConnectionStatus>().toEqualTypeOf<'CONNECTED' | 'PARTIAL' | 'DISCONNECTED'>();
  });

  it('ServiceConnection has required fields', () => {
    const conn: ServiceConnection = {
      provider: 'google',
      status: 'CONNECTED',
      grantedScopes: ['email', 'profile'],
      missingScopes: [],
      connectedAt: '2026-01-01T00:00:00Z',
    };
    expectTypeOf(conn.provider).toBeString();
    expectTypeOf(conn.grantedScopes).toEqualTypeOf<readonly string[]>();
  });

  it('ConsentRequest has required fields', () => {
    const req: ConsentRequest = {
      authorizationUrl: 'https://accounts.google.com/o/oauth2/auth',
      provider: 'google',
      requestedScopes: ['calendar'],
      state: 'abc123',
    };
    expectTypeOf(req.authorizationUrl).toBeString();
  });

  it('ProviderBrand has required fields', () => {
    const brand: ProviderBrand = {
      providerId: 'google',
      displayName: 'Google',
      icon: '<svg>...</svg>',
      brandColor: '#4285F4',
    };
    expectTypeOf(brand.icon).toBeString();
  });
});
```

```typescript
// packages/mosaic-core/src/types/credential.test.ts
import { describe, it, expectTypeOf } from 'vitest';
import type { WebAuthnCredential, AuthenticatorTransport } from './credential.js';

describe('credential types', () => {
  it('AuthenticatorTransport is a string union', () => {
    expectTypeOf<AuthenticatorTransport>().toEqualTypeOf<'usb' | 'ble' | 'nfc' | 'internal' | 'hybrid'>();
  });

  it('WebAuthnCredential has required fields', () => {
    const cred: WebAuthnCredential = {
      credentialId: 'cred-1',
      displayName: 'MacBook Pro Touch ID',
      transports: ['internal'],
      discoverable: true,
      createdAt: '2026-01-01T00:00:00Z',
      lastUsedAt: null,
    };
    expectTypeOf(cred.transports).toEqualTypeOf<readonly AuthenticatorTransport[]>();
    expectTypeOf(cred.lastUsedAt).toEqualTypeOf<string | null>();
  });
});
```

- [ ] **Step 5: Run tests**

Run: `yarn --cwd packages/mosaic-core test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/mosaic-core/src/types/connection.ts packages/mosaic-core/src/types/credential.ts packages/mosaic-core/src/types/connection.test.ts packages/mosaic-core/src/types/credential.test.ts packages/mosaic-core/src/types/index.ts
git commit -m "feat(#224): add connection and credential types to mosaic-core"
```

### Task 2: Scaffold connection-manager package + ProviderBrandRegistry + Topics

**Files:**
- Create: `components/connection-manager/package.json`
- Create: `components/connection-manager/tsconfig.json`
- Create: `components/connection-manager/tsconfig.build.json`
- Create: `components/connection-manager/vitest.config.ts`
- Create: `components/connection-manager/src/index.ts`
- Create: `components/connection-manager/src/topics.ts`
- Create: `components/connection-manager/src/controllers/provider-brand-registry.ts`
- Create: `components/connection-manager/src/controllers/provider-brand-registry.test.ts`
- Create: `components/connection-manager/TRADEMARK-NOTICE.md`
- Modify: `tsconfig.json` (root — add project reference)

**Interfaces:**
- Consumes: `ProviderBrand` from `@casehubio/mosaic-core`
- Produces: `ConnectionManagerTopics`, `ProviderBrandRegistry`, `defaultBrandRegistry`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/mosaic-connection-manager",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/mosaic-core": "workspace:*",
    "@casehubio/pages-data": "*",
    "@casehubio/pages-primitives": "*",
    "lit": "^3.0.0"
  },
  "devDependencies": {
    "@open-wc/testing": "^4.0.0",
    "jsdom": "^25.0.0",
    "rimraf": "^6.1.0",
    "typescript": "~5.8.0",
    "vitest": "^3.0.0"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "files": ["dist", "package.json"]
}
```

- [ ] **Step 2: Create tsconfig.json and tsconfig.build.json**

tsconfig.json:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"],
  "references": [
    { "path": "../../packages/mosaic-core" }
  ]
}
```

tsconfig.build.json — same as tsconfig.json (copy the pattern from approval-gate if it has a separate build config, otherwise use the same file).

- [ ] **Step 3: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';
import { existsSync } from 'fs';

export default defineConfig({
  resolve: {
    alias: [
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-primitives/src')) ? [{ find: '@casehubio/pages-primitives', replacement: path.resolve(__dirname, '../../../pages/packages/pages-primitives/src') }] : []),
      { find: '@casehubio/mosaic-core', replacement: path.resolve(__dirname, '../../packages/mosaic-core/src') },
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-data/src')) ? [{ find: '@casehubio/pages-data', replacement: path.resolve(__dirname, '../../../pages/packages/pages-data/src') }] : []),
    ],
  },
  esbuild: {
    target: 'es2022',
    tsconfigRaw: {
      compilerOptions: {
        experimentalDecorators: true,
        useDefineForClassFields: false,
      },
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
  },
});
```

- [ ] **Step 4: Write Topics constant**

```typescript
// src/topics.ts
export const ConnectionManagerTopics = {
  CONNECT: 'connection.connect',
  DISCONNECT: 'connection.disconnect',
  CONSENT: 'connection.consent',
  OAUTH_COMPLETE: 'oauth.complete',
  OAUTH_ERROR: 'oauth.error',
  PASSKEY_REGISTER_REQUEST: 'passkey.register-request',
  PASSKEY_REGISTERED: 'passkey.registered',
  PASSKEY_REGISTER_ERROR: 'passkey.register-error',
  PASSKEY_AUTHENTICATE_REQUEST: 'passkey.authenticate-request',
  PASSKEY_AUTHENTICATED: 'passkey.authenticated',
  PASSKEY_AUTHENTICATE_ERROR: 'passkey.authenticate-error',
  PASSKEY_REVOKE: 'passkey.revoke',
} as const;
```

- [ ] **Step 5: Write ProviderBrandRegistry test**

```typescript
// src/controllers/provider-brand-registry.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { ProviderBrandRegistry, defaultBrandRegistry } from './provider-brand-registry.js';

describe('ProviderBrandRegistry', () => {
  it('defaultBrandRegistry has google', () => {
    const brand = defaultBrandRegistry.get('google');
    expect(brand).not.toBeNull();
    expect(brand!.displayName).toBe('Google');
    expect(brand!.brandColor).toMatch(/^#/);
    expect(brand!.icon).toContain('<svg');
  });

  it('defaultBrandRegistry has github', () => {
    expect(defaultBrandRegistry.has('github')).toBe(true);
  });

  it('defaultBrandRegistry has slack', () => {
    expect(defaultBrandRegistry.has('slack')).toBe(true);
  });

  it('defaultBrandRegistry has discord', () => {
    expect(defaultBrandRegistry.has('discord')).toBe(true);
  });

  it('returns null for unknown provider', () => {
    expect(defaultBrandRegistry.get('unknown')).toBeNull();
  });

  it('custom instance allows registration', () => {
    const registry = new ProviderBrandRegistry();
    registry.register('custom', {
      providerId: 'custom',
      displayName: 'Custom',
      icon: '<svg></svg>',
      brandColor: '#000',
    });
    expect(registry.get('custom')).not.toBeNull();
    expect(registry.get('custom')!.displayName).toBe('Custom');
  });

  it('custom registration overrides built-in', () => {
    const registry = new ProviderBrandRegistry();
    registry.register('google', {
      providerId: 'google',
      displayName: 'Custom Google',
      icon: '<svg>custom</svg>',
      brandColor: '#000',
    });
    expect(registry.get('google')!.displayName).toBe('Custom Google');
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `yarn --cwd components/connection-manager test`
Expected: FAIL (provider-brand-registry.ts doesn't exist yet)

- [ ] **Step 7: Implement ProviderBrandRegistry**

```typescript
// src/controllers/provider-brand-registry.ts
import type { ProviderBrand } from '@casehubio/mosaic-core';

const GOOGLE_ICON = `<svg viewBox="0 0 24 24" width="18" height="18"><path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92a5.06 5.06 0 0 1-2.2 3.32v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.1z"/><path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/><path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/><path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/></svg>`;

const GITHUB_ICON = `<svg viewBox="0 0 24 24" width="18" height="18"><path fill="currentColor" d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>`;

const SLACK_ICON = `<svg viewBox="0 0 24 24" width="18" height="18"><path fill="#E01E5A" d="M5.042 15.165a2.528 2.528 0 0 1-2.52 2.523A2.528 2.528 0 0 1 0 15.165a2.527 2.527 0 0 1 2.522-2.52h2.52v2.52zm1.271 0a2.527 2.527 0 0 1 2.521-2.52 2.527 2.527 0 0 1 2.521 2.52v6.313A2.528 2.528 0 0 1 8.834 24a2.528 2.528 0 0 1-2.521-2.522v-6.313z"/><path fill="#36C5F0" d="M8.834 5.042a2.528 2.528 0 0 1-2.521-2.52A2.528 2.528 0 0 1 8.834 0a2.528 2.528 0 0 1 2.521 2.522v2.52H8.834zm0 1.271a2.528 2.528 0 0 1 2.521 2.521 2.528 2.528 0 0 1-2.521 2.521H2.522A2.528 2.528 0 0 1 0 8.834a2.528 2.528 0 0 1 2.522-2.521h6.312z"/><path fill="#2EB67D" d="M18.956 8.834a2.528 2.528 0 0 1 2.522-2.521A2.528 2.528 0 0 1 24 8.834a2.528 2.528 0 0 1-2.522 2.521h-2.522V8.834zm-1.27 0a2.528 2.528 0 0 1-2.523 2.521 2.527 2.527 0 0 1-2.52-2.521V2.522A2.527 2.527 0 0 1 15.163 0a2.528 2.528 0 0 1 2.523 2.522v6.312z"/><path fill="#ECB22E" d="M15.163 18.956a2.528 2.528 0 0 1 2.523 2.522A2.528 2.528 0 0 1 15.163 24a2.527 2.527 0 0 1-2.52-2.522v-2.522h2.52zm0-1.27a2.527 2.527 0 0 1-2.52-2.523 2.526 2.526 0 0 1 2.52-2.52h6.315A2.528 2.528 0 0 1 24 15.163a2.528 2.528 0 0 1-2.522 2.523h-6.315z"/></svg>`;

const DISCORD_ICON = `<svg viewBox="0 0 24 24" width="18" height="18"><path fill="#5865F2" d="M20.317 4.37a19.791 19.791 0 0 0-4.885-1.515.074.074 0 0 0-.079.037c-.21.375-.444.864-.608 1.25a18.27 18.27 0 0 0-5.487 0 12.64 12.64 0 0 0-.617-1.25.077.077 0 0 0-.079-.037A19.736 19.736 0 0 0 3.677 4.37a.07.07 0 0 0-.032.027C.533 9.046-.32 13.58.099 18.057a.082.082 0 0 0 .031.057 19.9 19.9 0 0 0 5.993 3.03.078.078 0 0 0 .084-.028 14.09 14.09 0 0 0 1.226-1.994.076.076 0 0 0-.041-.106 13.107 13.107 0 0 1-1.872-.892.077.077 0 0 1-.008-.128 10.2 10.2 0 0 0 .372-.292.074.074 0 0 1 .077-.01c3.928 1.793 8.18 1.793 12.062 0a.074.074 0 0 1 .078.01c.12.098.246.198.373.292a.077.077 0 0 1-.006.127 12.299 12.299 0 0 1-1.873.892.077.077 0 0 0-.041.107c.36.698.772 1.362 1.225 1.993a.076.076 0 0 0 .084.028 19.839 19.839 0 0 0 6.002-3.03.077.077 0 0 0 .032-.054c.5-5.177-.838-9.674-3.549-13.66a.061.061 0 0 0-.031-.03zM8.02 15.33c-1.183 0-2.157-1.085-2.157-2.419 0-1.333.956-2.419 2.157-2.419 1.21 0 2.176 1.096 2.157 2.42 0 1.333-.956 2.418-2.157 2.418zm7.975 0c-1.183 0-2.157-1.085-2.157-2.419 0-1.333.955-2.419 2.157-2.419 1.21 0 2.176 1.096 2.157 2.42 0 1.333-.946 2.418-2.157 2.418z"/></svg>`;

const BUILT_IN_PROVIDERS: ReadonlyMap<string, ProviderBrand> = new Map([
  ['google', { providerId: 'google', displayName: 'Google', icon: GOOGLE_ICON, brandColor: '#4285F4' }],
  ['github', { providerId: 'github', displayName: 'GitHub', icon: GITHUB_ICON, brandColor: '#24292e' }],
  ['slack', { providerId: 'slack', displayName: 'Slack', icon: SLACK_ICON, brandColor: '#4A154B' }],
  ['discord', { providerId: 'discord', displayName: 'Discord', icon: DISCORD_ICON, brandColor: '#5865F2' }],
]);

export class ProviderBrandRegistry {
  private readonly brands: Map<string, ProviderBrand>;

  constructor(defaults?: Map<string, ProviderBrand>) {
    this.brands = new Map(defaults ?? BUILT_IN_PROVIDERS);
  }

  get(providerId: string): ProviderBrand | null {
    return this.brands.get(providerId) ?? null;
  }

  register(providerId: string, brand: ProviderBrand): void {
    this.brands.set(providerId, brand);
  }

  has(providerId: string): boolean {
    return this.brands.has(providerId);
  }
}

export const defaultBrandRegistry = new ProviderBrandRegistry();
```

- [ ] **Step 8: Write TRADEMARK-NOTICE.md**

```markdown
# Trademark Notice

The provider logos (SVG icons) included in this package are trademarks
of their respective owners:

- Google: Google LLC
- GitHub: GitHub, Inc.
- Slack: Slack Technologies, LLC
- Discord: Discord Inc.

These logos are included for nominative use in "Sign in with" and
"Connect with" UI buttons, as permitted by each provider's brand
guidelines. They are NOT covered by this package's MIT license.

Apple's logo is not included. Use Apple's Sign in with Apple JS or
provide your own via `ProviderBrandRegistry.register()`.
```

- [ ] **Step 9: Write initial index.ts**

```typescript
// src/index.ts
export { ConnectionManagerTopics } from './topics.js';
export { ProviderBrandRegistry, defaultBrandRegistry } from './controllers/provider-brand-registry.js';
```

- [ ] **Step 10: Run tests and typecheck**

Run: `yarn --cwd components/connection-manager test`
Expected: PASS

Run: `yarn --cwd components/connection-manager typecheck`
Expected: PASS (after `yarn install` to link workspace deps)

- [ ] **Step 11: Commit**

```bash
git add components/connection-manager/ tsconfig.json
git commit -m "feat(#224): scaffold connection-manager package with ProviderBrandRegistry and Topics"
```

---

## Batch 2: OAuth primitives — connect-button + OAuthFlowController + scope-list

After this batch: the OAuth connection primitives work — branded buttons trigger OAuth flows, scope lists expand/collapse, all with ARIA and events.

### Task 3: OAuthFlowController

**Files:**
- Create: `components/connection-manager/src/controllers/oauth-flow-controller.ts`
- Create: `components/connection-manager/src/controllers/oauth-flow-controller.test.ts`

**Interfaces:**
- Produces: `OAuthFlowController`, `OAuthFlowOptions`, `OAuthFlowResult`

- [ ] **Step 1: Write failing tests**

```typescript
// src/controllers/oauth-flow-controller.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { OAuthFlowController } from './oauth-flow-controller.js';

describe('OAuthFlowController', () => {
  let controller: OAuthFlowController;
  let openSpy: ReturnType<typeof vi.spyOn>;

  beforeEach(() => {
    controller = new OAuthFlowController();
    openSpy = vi.spyOn(window, 'open');
  });

  afterEach(() => {
    controller.cancel();
    vi.restoreAllMocks();
  });

  it('opens popup with authorization URL', async () => {
    const fakePopup = { closed: false, close: vi.fn() } as unknown as Window;
    openSpy.mockReturnValue(fakePopup);

    const promise = controller.start({
      authorizationUrl: 'https://accounts.google.com/o/oauth2/auth?state=abc',
      state: 'abc',
      expectedOrigin: 'http://localhost:8080',
    });

    // Simulate postMessage from popup
    window.dispatchEvent(new MessageEvent('message', {
      data: { type: 'oauth-callback', code: 'auth-code-123', state: 'abc' },
      origin: 'http://localhost:8080',
    }));

    const result = await promise;
    expect(result.code).toBe('auth-code-123');
    expect(result.state).toBe('abc');
    expect(openSpy).toHaveBeenCalled();
  });

  it('rejects messages from wrong origin', async () => {
    const fakePopup = { closed: false, close: vi.fn() } as unknown as Window;
    openSpy.mockReturnValue(fakePopup);

    const promise = controller.start({
      authorizationUrl: 'https://accounts.google.com/o/oauth2/auth?state=abc',
      state: 'abc',
      expectedOrigin: 'http://localhost:8080',
    });

    // Wrong origin — should be ignored
    window.dispatchEvent(new MessageEvent('message', {
      data: { type: 'oauth-callback', code: 'stolen', state: 'abc' },
      origin: 'https://evil.com',
    }));

    // Correct origin
    window.dispatchEvent(new MessageEvent('message', {
      data: { type: 'oauth-callback', code: 'real-code', state: 'abc' },
      origin: 'http://localhost:8080',
    }));

    const result = await promise;
    expect(result.code).toBe('real-code');
  });

  it('rejects on state mismatch', async () => {
    const fakePopup = { closed: false, close: vi.fn() } as unknown as Window;
    openSpy.mockReturnValue(fakePopup);

    const promise = controller.start({
      authorizationUrl: 'https://accounts.google.com/o/oauth2/auth?state=abc',
      state: 'abc',
      expectedOrigin: 'http://localhost:8080',
    });

    window.dispatchEvent(new MessageEvent('message', {
      data: { type: 'oauth-callback', code: 'code', state: 'wrong-state' },
      origin: 'http://localhost:8080',
    }));

    // Should still be pending — wrong state ignored
    // Send correct state
    window.dispatchEvent(new MessageEvent('message', {
      data: { type: 'oauth-callback', code: 'code', state: 'abc' },
      origin: 'http://localhost:8080',
    }));

    const result = await promise;
    expect(result.state).toBe('abc');
  });

  it('cancel resolves with error', async () => {
    openSpy.mockReturnValue(null); // popup blocked
    // With redirectFallback: false, should reject
    await expect(controller.start({
      authorizationUrl: 'https://example.com',
      state: 'abc',
      expectedOrigin: 'http://localhost:8080',
      redirectFallback: false,
    })).rejects.toThrow('popup blocked');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd components/connection-manager test`
Expected: FAIL

- [ ] **Step 3: Implement OAuthFlowController**

```typescript
// src/controllers/oauth-flow-controller.ts
export interface OAuthFlowOptions {
  authorizationUrl: string;
  state: string;
  expectedOrigin: string;
  popupWidth?: number;
  popupHeight?: number;
  redirectFallback?: boolean;
}

export interface OAuthFlowResult {
  code: string;
  state: string;
}

export class OAuthFlowController {
  private cleanup: (() => void) | null = null;

  async start(options: OAuthFlowOptions): Promise<OAuthFlowResult> {
    const {
      authorizationUrl,
      state,
      expectedOrigin,
      popupWidth = 500,
      popupHeight = 600,
      redirectFallback = true,
    } = options;

    const left = (screen.width - popupWidth) / 2;
    const top = (screen.height - popupHeight) / 2;
    const features = `width=${popupWidth},height=${popupHeight},left=${left},top=${top},toolbar=no,menubar=no`;

    const popup = window.open(authorizationUrl, 'oauth-popup', features);

    if (!popup) {
      if (redirectFallback) {
        sessionStorage.setItem('oauth-pending-state', state);
        sessionStorage.setItem('oauth-return-url', window.location.href);
        window.location.href = authorizationUrl;
        return new Promise(() => {}); // never resolves — page navigates away
      }
      throw new Error('popup blocked');
    }

    return new Promise<OAuthFlowResult>((resolve, reject) => {
      const onMessage = (event: MessageEvent) => {
        if (event.origin !== expectedOrigin) return;
        const data = event.data;
        if (data?.type !== 'oauth-callback') return;
        if (data.state !== state) return;

        this.doCleanup();
        popup.close();
        resolve({ code: data.code, state: data.state });
      };

      const pollTimer = window.setInterval(() => {
        if (popup.closed) {
          this.doCleanup();
          reject(new Error('OAuth popup closed by user'));
        }
      }, 500);

      this.cleanup = () => {
        window.removeEventListener('message', onMessage);
        window.clearInterval(pollTimer);
        this.cleanup = null;
      };

      window.addEventListener('message', onMessage);
    });
  }

  cancel(): void {
    this.doCleanup();
  }

  private doCleanup(): void {
    this.cleanup?.();
  }

  static checkRedirectReturn(): OAuthFlowResult | null {
    const state = sessionStorage.getItem('oauth-pending-state');
    if (!state) return null;

    const params = new URLSearchParams(window.location.search);
    const code = params.get('code');
    if (!code) return null;

    sessionStorage.removeItem('oauth-pending-state');
    sessionStorage.removeItem('oauth-return-url');
    return { code, state };
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn --cwd components/connection-manager test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add components/connection-manager/src/controllers/oauth-flow-controller.ts components/connection-manager/src/controllers/oauth-flow-controller.test.ts
git commit -m "feat(#226): add OAuthFlowController with popup, redirect fallback, and origin verification"
```

### Task 4: scope-list primitive

**Files:**
- Create: `components/connection-manager/src/primitives/scope-list.ts`
- Create: `components/connection-manager/src/primitives/scope-list.test.ts`

**Interfaces:**
- Produces: `ScopeList` (`mosaic-scope-list` custom element)

- [ ] **Step 1: Write failing test**

```typescript
// src/primitives/scope-list.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './scope-list.js';

type ScopeListEl = HTMLElement & {
  grantedScopes: readonly string[];
  missingScopes: readonly string[];
  expanded: boolean;
  updateComplete: Promise<boolean>;
};

describe('mosaic-scope-list', () => {
  let el: ScopeListEl;

  beforeEach(async () => {
    el = document.createElement('mosaic-scope-list') as ScopeListEl;
    el.grantedScopes = ['email', 'profile'];
    el.missingScopes = ['calendar'];
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => { el.remove(); });

  it('has role="list"', () => {
    const list = el.shadowRoot!.querySelector('[role="list"]');
    expect(list).not.toBeNull();
  });

  it('renders granted scopes', () => {
    expect(el.shadowRoot!.textContent).toContain('email');
    expect(el.shadowRoot!.textContent).toContain('profile');
  });

  it('renders missing scopes with warning styling', () => {
    const missing = el.shadowRoot!.querySelectorAll('.scope-missing');
    expect(missing.length).toBe(1);
  });

  it('supports aria-expanded toggle', async () => {
    el.expanded = false;
    await el.updateComplete;
    const toggle = el.shadowRoot!.querySelector('[aria-expanded]');
    expect(toggle?.getAttribute('aria-expanded')).toBe('false');

    el.expanded = true;
    await el.updateComplete;
    expect(toggle?.getAttribute('aria-expanded')).toBe('true');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd components/connection-manager test -- scope-list`
Expected: FAIL

- [ ] **Step 3: Implement scope-list**

```typescript
// src/primitives/scope-list.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('mosaic-scope-list')
export class ScopeList extends LitElement {
  @property({ attribute: false }) grantedScopes: readonly string[] = [];
  @property({ attribute: false }) missingScopes: readonly string[] = [];
  @property({ type: Boolean }) expanded = false;

  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui); font-size: 0.85rem; }
    .toggle { background: none; border: none; cursor: pointer; padding: 0.25rem 0; color: inherit; font-size: inherit; }
    .toggle:hover { text-decoration: underline; }
    .scope-item { padding: 0.125rem 0; display: flex; align-items: center; gap: 0.25rem; }
    .scope-granted { color: var(--pages-success-11, #155724); }
    .scope-missing { color: var(--pages-warning-11, #856404); }
    .dot { width: 6px; height: 6px; border-radius: 50%; display: inline-block; }
    .dot-granted { background: var(--pages-success-9, #28a745); }
    .dot-missing { background: var(--pages-warning-9, #ffc107); }
  `;

  override render() {
    const total = this.grantedScopes.length + this.missingScopes.length;
    if (total === 0) return nothing;

    return html`
      <button class="toggle"
        aria-expanded="${this.expanded}"
        @click=${() => { this.expanded = !this.expanded; }}>
        ${this.expanded ? '▾' : '▸'} ${total} scope${total !== 1 ? 's' : ''}
        ${this.missingScopes.length > 0 ? html` (${this.missingScopes.length} missing)` : nothing}
      </button>
      ${this.expanded ? html`
        <div role="list" aria-label="Scopes">
          ${this.grantedScopes.map(s => html`
            <div role="listitem" class="scope-item scope-granted">
              <span class="dot dot-granted"></span> ${s}
            </div>
          `)}
          ${this.missingScopes.map(s => html`
            <div role="listitem" class="scope-item scope-missing">
              <span class="dot dot-missing"></span> ${s}
            </div>
          `)}
        </div>
      ` : nothing}
    `;
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn --cwd components/connection-manager test -- scope-list`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add components/connection-manager/src/primitives/scope-list.ts components/connection-manager/src/primitives/scope-list.test.ts
git commit -m "feat(#230): add mosaic-scope-list primitive with expandable granted/missing scopes"
```

### Task 5: connect-button primitive

**Files:**
- Create: `components/connection-manager/src/primitives/connect-button.ts`
- Create: `components/connection-manager/src/primitives/connect-button.test.ts`
- Modify: `components/connection-manager/src/index.ts`

**Interfaces:**
- Consumes: `ProviderBrand` from mosaic-core, `OAuthFlowController`, `defaultBrandRegistry`, `ConnectionManagerTopics`
- Produces: `ConnectButton` (`mosaic-connect-button` custom element)

- [ ] **Step 1: Write failing test**

```typescript
// src/primitives/connect-button.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import './connect-button.js';

type ConnectButtonEl = HTMLElement & {
  provider: string;
  mode: 'connect' | 'signin';
  authorizationUrl: string | null;
  loading: boolean;
  error: string | null;
  disabled: boolean;
  updateComplete: Promise<boolean>;
};

describe('mosaic-connect-button', () => {
  let el: ConnectButtonEl;

  beforeEach(async () => {
    el = document.createElement('mosaic-connect-button') as ConnectButtonEl;
    el.provider = 'google';
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => { el.remove(); });

  it('has role="button" and aria-label', () => {
    const btn = el.shadowRoot!.querySelector('button');
    expect(btn).not.toBeNull();
    expect(btn!.getAttribute('aria-label')).toContain('Google');
  });

  it('renders provider brand from registry', () => {
    expect(el.shadowRoot!.innerHTML).toContain('<svg');
    expect(el.shadowRoot!.textContent).toContain('Google');
  });

  it('renders "Connect with" in connect mode', () => {
    expect(el.shadowRoot!.textContent).toContain('Connect with');
  });

  it('renders "Sign in with" in signin mode', async () => {
    el.mode = 'signin';
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Sign in with');
  });

  it('emits connection.connect when clicked without authorizationUrl', async () => {
    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    el.shadowRoot!.querySelector('button')!.click();
    expect(handler).toHaveBeenCalledOnce();
    const detail = handler.mock.calls[0][0].detail;
    expect(detail.topic).toBe('connection.connect');
    expect(detail.payload.provider).toBe('google');
  });

  it('shows loading spinner when loading=true', async () => {
    el.loading = true;
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.spinner')).not.toBeNull();
    expect(el.shadowRoot!.querySelector('button')!.disabled).toBe(true);
  });

  it('shows error message when error is set', async () => {
    el.error = 'Connection failed';
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Connection failed');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd components/connection-manager test -- connect-button`
Expected: FAIL

- [ ] **Step 3: Implement connect-button**

```typescript
// src/primitives/connect-button.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { unsafeHTML } from 'lit/directives/unsafe-html.js';
import { emitPagesEvent } from '@casehubio/pages-data';
import type { ProviderBrand } from '@casehubio/mosaic-core';
import { defaultBrandRegistry } from '../controllers/provider-brand-registry.js';
import { OAuthFlowController } from '../controllers/oauth-flow-controller.js';
import { ConnectionManagerTopics } from '../topics.js';

@customElement('mosaic-connect-button')
export class ConnectButton extends LitElement {
  @property() provider = '';
  @property() mode: 'connect' | 'signin' = 'connect';
  @property({ attribute: 'authorization-url' }) authorizationUrl: string | null = null;
  @property({ attribute: 'expected-origin' }) expectedOrigin: string | null = null;
  @property({ attribute: false }) brand: ProviderBrand | null = null;
  @property({ type: Boolean }) loading = false;
  @property() error: string | null = null;
  @property({ type: Boolean }) disabled = false;

  private flowController = new OAuthFlowController();

  private get resolvedBrand(): ProviderBrand | null {
    return this.brand ?? defaultBrandRegistry.get(this.provider);
  }

  static override styles = css`
    :host { display: inline-block; }
    button {
      display: inline-flex; align-items: center; gap: 0.5rem;
      padding: 0.625rem 1rem; border: 1px solid var(--pages-neutral-6, #dee2e6);
      border-radius: var(--pages-radius-2, 6px); cursor: pointer;
      font-family: var(--pages-font-family, system-ui); font-size: 0.875rem;
      color: #fff; transition: opacity 0.15s;
    }
    button:hover:not(:disabled) { opacity: 0.9; }
    button:disabled { opacity: 0.6; cursor: not-allowed; }
    .icon { display: flex; align-items: center; }
    .spinner { width: 16px; height: 16px; border: 2px solid rgba(255,255,255,0.3); border-top-color: #fff; border-radius: 50%; animation: spin 0.6s linear infinite; }
    @keyframes spin { to { transform: rotate(360deg); } }
    .error { color: var(--pages-danger-11, #721c24); font-size: 0.75rem; margin-top: 0.25rem; }
  `;

  override render() {
    const brand = this.resolvedBrand;
    const label = this.mode === 'signin' ? 'Sign in with' : 'Connect with';
    const displayName = brand?.displayName ?? this.provider;
    const bgColor = brand?.brandColor ?? 'var(--pages-neutral-9, #333)';

    return html`
      <button
        style="background: ${bgColor}"
        aria-label="${label} ${displayName}"
        ?disabled=${this.disabled || this.loading}
        @click=${this.handleClick}>
        ${this.loading
          ? html`<span class="spinner"></span>`
          : brand?.icon ? html`<span class="icon">${unsafeHTML(brand.icon)}</span>` : nothing}
        ${label} ${displayName}
      </button>
      ${this.error ? html`<div class="error" role="alert">${this.error}</div>` : nothing}
    `;
  }

  private async handleClick() {
    if (this.authorizationUrl && this.expectedOrigin) {
      this.loading = true;
      this.error = null;
      try {
        const result = await this.flowController.start({
          authorizationUrl: this.authorizationUrl,
          state: new URL(this.authorizationUrl).searchParams.get('state') ?? '',
          expectedOrigin: this.expectedOrigin,
        });
        emitPagesEvent(this, ConnectionManagerTopics.OAUTH_COMPLETE, {
          provider: this.provider, ...result,
        });
      } catch (err) {
        const message = err instanceof Error ? err.message : 'OAuth flow failed';
        this.error = message;
        emitPagesEvent(this, ConnectionManagerTopics.OAUTH_ERROR, {
          provider: this.provider, error: message,
        });
      } finally {
        this.loading = false;
      }
    } else {
      emitPagesEvent(this, ConnectionManagerTopics.CONNECT, { provider: this.provider });
    }
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this.flowController.cancel();
  }
}
```

- [ ] **Step 4: Update index.ts**

Add to `src/index.ts`:
```typescript
export { ScopeList } from './primitives/scope-list.js';
export { ConnectButton } from './primitives/connect-button.js';
export { OAuthFlowController } from './controllers/oauth-flow-controller.js';
export type { OAuthFlowOptions, OAuthFlowResult } from './controllers/oauth-flow-controller.js';
```

- [ ] **Step 5: Run tests**

Run: `yarn --cwd components/connection-manager test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add components/connection-manager/src/primitives/connect-button.ts components/connection-manager/src/primitives/connect-button.test.ts components/connection-manager/src/index.ts
git commit -m "feat(#225): add mosaic-connect-button with branded OAuth, loading/error states, and ARIA"
```

---

## Batch 3: Provider status row + passkey primitives

After this batch: all primitives exist — provider rows with status/scopes/actions, passkey rows with revoke, register button, and authenticate button.

### Task 6: provider-status-row primitive

**Files:**
- Create: `components/connection-manager/src/primitives/provider-status-row.ts`
- Create: `components/connection-manager/src/primitives/provider-status-row.test.ts`

**Interfaces:**
- Consumes: `ServiceConnection`, `ProviderBrand` from mosaic-core, `defaultBrandRegistry`, `ConnectionManagerTopics`, `ScopeList`, `ConnectButton`
- Produces: `ProviderStatusRow` (`mosaic-provider-status-row`)

- [ ] **Step 1: Write failing test**

```typescript
// src/primitives/provider-status-row.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import type { ServiceConnection } from '@casehubio/mosaic-core';
import './provider-status-row.js';

type ProviderStatusRowEl = HTMLElement & {
  connection: ServiceConnection;
  loading: boolean;
  error: string | null;
  updateComplete: Promise<boolean>;
};

describe('mosaic-provider-status-row', () => {
  let el: ProviderStatusRowEl;

  beforeEach(async () => {
    el = document.createElement('mosaic-provider-status-row') as ProviderStatusRowEl;
    el.connection = {
      provider: 'google',
      status: 'CONNECTED',
      grantedScopes: ['email', 'profile'],
      missingScopes: [],
      connectedAt: '2026-01-01T00:00:00Z',
    };
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => { el.remove(); });

  it('has aria-label with provider name and status', () => {
    expect(el.getAttribute('aria-label')).toContain('Google');
    expect(el.getAttribute('aria-label')).toContain('Connected');
  });

  it('renders status badge', () => {
    const badge = el.shadowRoot!.querySelector('.status-badge');
    expect(badge).not.toBeNull();
    expect(badge!.textContent!.trim()).toBe('Connected');
  });

  it('shows Disconnect button for CONNECTED status', () => {
    const btn = el.shadowRoot!.querySelector('.action-btn');
    expect(btn!.textContent!.trim()).toBe('Disconnect');
  });

  it('shows Grant Access button for PARTIAL status', async () => {
    el.connection = { ...el.connection, status: 'PARTIAL', missingScopes: ['calendar'] };
    await el.updateComplete;
    const btn = el.shadowRoot!.querySelector('.action-btn');
    expect(btn!.textContent!.trim()).toBe('Grant Access');
  });

  it('emits connection.disconnect on Disconnect click', () => {
    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
    const detail = handler.mock.calls[0][0].detail;
    expect(detail.topic).toBe('connection.disconnect');
    expect(detail.payload.provider).toBe('google');
  });

  it('emits connection.consent for PARTIAL with missing scopes', async () => {
    el.connection = { ...el.connection, status: 'PARTIAL', missingScopes: ['calendar'] };
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('pages-event', handler);
    el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
    const detail = handler.mock.calls[0][0].detail;
    expect(detail.topic).toBe('connection.consent');
    expect(detail.payload.missingScopes).toEqual(['calendar']);
  });
});
```

- [ ] **Step 2: Run test to verify it fails, implement, run tests again, commit**

Follow the same TDD cycle. Implementation renders provider icon + name + status badge + expandable scope-list + action button. Sets `aria-label` on the host in `connectedCallback` / `updated`. Emits `connection.connect`, `connection.disconnect`, or `connection.consent` via `emitPagesEvent` depending on `connection.status`.

- [ ] **Step 3: Commit**

```bash
git add components/connection-manager/src/primitives/provider-status-row.ts components/connection-manager/src/primitives/provider-status-row.test.ts
git commit -m "feat(#232): add mosaic-provider-status-row with status badge, scopes, and action events"
```

### Task 7: passkey-row, passkey-register-button, passkey-authenticate-button

**Files:**
- Create: `components/connection-manager/src/primitives/passkey-row.ts`
- Create: `components/connection-manager/src/primitives/passkey-row.test.ts`
- Create: `components/connection-manager/src/primitives/passkey-register-button.ts`
- Create: `components/connection-manager/src/primitives/passkey-register-button.test.ts`
- Create: `components/connection-manager/src/primitives/passkey-authenticate-button.ts`
- Create: `components/connection-manager/src/primitives/passkey-authenticate-button.test.ts`
- Modify: `components/connection-manager/src/index.ts`

**Interfaces:**
- Consumes: `WebAuthnCredential` from mosaic-core, `ConnectionManagerTopics`
- Produces: `PasskeyRow` (`mosaic-passkey-row`), `PasskeyRegisterButton` (`mosaic-passkey-register`), `PasskeyAuthenticateButton` (`mosaic-passkey-authenticate`)

- [ ] **Step 1: Write passkey-row test**

Test covers: ARIA (`role="listitem"`, `aria-label` with device name), renders device name and transports as icons, renders last-used relative time, emits `passkey.revoke` on revoke button click.

- [ ] **Step 2: Implement passkey-row, verify tests pass**

Simple Lit component. Renders credential info. Revoke button emits `emitPagesEvent(this, ConnectionManagerTopics.PASSKEY_REVOKE, { credentialId })`.

- [ ] **Step 3: Write passkey-register-button test**

Test covers: ARIA (`role="button"`, `aria-label="Register a passkey"`), emits `passkey.register-request` when clicked without `registrationOptions`, loading/error states.

- [ ] **Step 4: Implement passkey-register-button, verify tests pass**

When `registrationOptions` is set, calls `navigator.credentials.create()`. On success emits `passkey.registered`. On failure emits `passkey.register-error`. When no options, emits `passkey.register-request` for host to fetch.

- [ ] **Step 5: Write passkey-authenticate-button test**

Test covers: ARIA, emits `passkey.authenticate-request` without `challengeOptions`, loading/error states.

- [ ] **Step 6: Implement passkey-authenticate-button, verify tests pass**

Same pattern as register — calls `navigator.credentials.get()` when `challengeOptions` set.

- [ ] **Step 7: Update index.ts with all passkey exports**

- [ ] **Step 8: Run all tests**

Run: `yarn --cwd components/connection-manager test`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add components/connection-manager/src/primitives/passkey-*.ts components/connection-manager/src/index.ts
git commit -m "feat(#228): add passkey primitives — row, register, and authenticate buttons"
```

---

## Batch 4: Pre-configured pages — connection-management-page + login-page

After this batch: both pages exist, compose primitives, and work in demo.html.

### Task 8: connection-management-page

**Files:**
- Create: `components/connection-manager/src/connection-management-page.ts`
- Create: `components/connection-manager/src/connection-management-page.test.ts`
- Create: `components/connection-manager/demo.html`

**Interfaces:**
- Consumes: all primitives, `ServiceConnection[]`, `WebAuthnCredential[]`
- Produces: `ConnectionManagementPage` (`mosaic-connection-management`)

- [ ] **Step 1: Write failing test**

Test covers: ARIA (`role="region"`, `aria-label="Connection management"`), renders provider rows for each connection, renders passkey rows for each credential, renders register-passkey button, bubbles events from primitives.

- [ ] **Step 2: Implement connection-management-page, verify tests pass**

Composes `provider-status-row` in a `role="list"` for connections section, `passkey-row` in a `role="list"` for passkeys section, and a `passkey-register-button` at the bottom.

- [ ] **Step 3: Create demo.html with static fixture data**

HTML page that imports the component and renders it with sample `ServiceConnection[]` and `WebAuthnCredential[]`.

- [ ] **Step 4: Run all tests**

Run: `yarn --cwd components/connection-manager test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add components/connection-manager/src/connection-management-page.ts components/connection-manager/src/connection-management-page.test.ts components/connection-manager/demo.html
git commit -m "feat(#227): add mosaic-connection-management page composing all primitives"
```

### Task 9: login-page package

**Files:**
- Create: `components/login-page/package.json`
- Create: `components/login-page/tsconfig.json`
- Create: `components/login-page/tsconfig.build.json`
- Create: `components/login-page/vitest.config.ts`
- Create: `components/login-page/src/login-page.ts`
- Create: `components/login-page/src/login-page.test.ts`
- Create: `components/login-page/src/index.ts`
- Create: `components/login-page/demo.html`
- Modify: `tsconfig.json` (root — add project reference)

**Interfaces:**
- Consumes: `ConnectButton`, `PasskeyAuthenticateButton` from connection-manager
- Produces: `LoginPage` (`mosaic-login-page`)

- [ ] **Step 1: Scaffold package** (package.json, tsconfig, vitest.config — same patterns as connection-manager, plus reference to connection-manager)

- [ ] **Step 2: Write failing test**

Test covers: ARIA (`role="main"`, `aria-label="Sign in"`, buttons in `role="group"`), renders connect-buttons for each provider in `mode="signin"`, renders passkey-authenticate-button when `passkeyEnabled=true`, hides passkey when `passkeyEnabled=false`.

- [ ] **Step 3: Implement login-page, verify tests pass**

Renders social login buttons vertically with "or" divider before passkey option.

- [ ] **Step 4: Create demo.html, run all tests**

- [ ] **Step 5: Commit**

```bash
git add components/login-page/
git commit -m "feat(#229): add mosaic-login-page with social login buttons and passkey sign-in"
```

---

## Batch 5: Integration — final exports, build verification, doc update

After this batch: everything builds, all tests pass, docs updated.

### Task 10: Final integration and documentation

**Files:**
- Modify: `components/connection-manager/src/index.ts` (ensure all exports)
- Modify: `docs/guides/consumer-guide.md` (add connection-manager and login-page to module table)
- Modify: `docs/guides/contributor-guide.md` (add connection-manager architecture notes)

- [ ] **Step 1: Verify all exports from connection-manager index.ts**

Ensure index.ts re-exports: all primitives, `OAuthFlowController`, `ProviderBrandRegistry`, `defaultBrandRegistry`, `ConnectionManagerTopics`, `ConnectionManagementPage`.

- [ ] **Step 2: Run full build**

Run: `yarn build`
Expected: PASS (all packages compile)

- [ ] **Step 3: Run full test suite**

Run: `yarn test`
Expected: PASS

- [ ] **Step 4: Run typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 5: Run aria-check**

Run: `yarn aria-check`
Expected: PASS (all new components have ARIA)

- [ ] **Step 6: Update consumer-guide.md**

Add connection-manager and login-page to the module table with maturity "Alpha".

- [ ] **Step 7: Update contributor-guide.md**

Add brief architecture note about the connection-manager package structure (primitives, controllers, pages pattern).

- [ ] **Step 8: Commit**

```bash
git add components/connection-manager/src/index.ts docs/guides/consumer-guide.md docs/guides/contributor-guide.md
git commit -m "docs(#224): add connection-manager and login-page to guides, finalize exports"
```

## References

- [2026-11-19-connection-identity-ui-design.md] — design spec this plan implements
- `platform-api/src/main/java/io/casehub/platform/api/authn/ServiceConnectionProvider.java:6-19` — connection SPI
- `platform-api/src/main/java/io/casehub/platform/api/authn/WebAuthnCredentialStore.java:1-13` — credential SPI
- `components/approval-gate/src/approval-gate.ts` — reference component (props+events+ARIA pattern)
- `components/service-card/src/service-card.ts` — reference component (status rendering)
- `packages/mosaic-core/src/types/identity.ts` — existing identity types
- casehubio/platform#898 — REST endpoints (dependency)
- casehubio/casehub-mosaic#328 — MCP server auth page (consumer)
- casehubio/casehub-mosaic#224-#232 — epic and child issues
