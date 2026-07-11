---
name: sveltekit-tauri
description: "SvelteKit frontend patterns for Tauri desktop apps. Use when writing Svelte components, pages, layouts, load functions, or routing in a Tauri project. Triggers on .svelte files, +page, +layout, +server, load function, SvelteKit, adapter, SSR, prerender. Complements the tauri-v2 skill (Rust backend); this skill owns the frontend boundary."
version: 1.0.0
---

# SvelteKit in Tauri — Frontend Constraints

> Tauri renders SvelteKit as a **static SPA** inside a webview — there is no Node.js server. Every SvelteKit feature that assumes a server is unavailable.

## Rendering Mode

Tauri loads the built frontend from disk via the asset protocol. The app runs as a **Single-Page Application** using `@sveltejs/adapter-static` with `fallback: 'index.html'`. SSR is globally disabled at the root layout:

```typescript
// src/routes/+layout.ts
export const ssr = false;
```

This single export shapes everything below — keep it. Removing it breaks Tauri API access and causes hydration mismatches.

## Forbidden Patterns

These SvelteKit features require a running Node.js server. Tauri has none.

| Feature | Why it fails | Alternative |
|---------|-------------|-------------|
| `+server.ts` / `+server.js` (API routes) | No server to handle requests | Tauri commands via `invoke()` |
| `+page.server.ts` / `+layout.server.ts` | Server-only load functions need Node runtime | Use `+page.ts` / `+layout.ts` (universal load) |
| `actions` (form actions) | Requires server endpoints | Handle form submission client-side, call `invoke()` |
| `adapter-node` / `adapter-auto` | Produce server bundles Tauri cannot serve | `adapter-static` only |
| `cookies` / `locals` / `platform` in load | Server-only RequestEvent properties | Store state client-side or in Tauri managed state |
| `fetch` to own API routes | No server to receive the request | `invoke()` for Rust commands, `fetch` for external URLs |
| `$env/dynamic/private` | Server-only env vars | `$env/static/public` or Tauri env plugin |
| `event.setHeaders()` | Server-side response manipulation | Tauri HTTP headers config in `tauri.conf.json` |

**When you see a `+server.ts` file or server load function in a Tauri project, it is a bug.** Replace it with client-side logic + Tauri IPC.

## Universal Load Functions

All data loading happens in universal load functions (`+page.ts` / `+layout.ts`), which run in the browser. Tauri APIs are available here because SSR is off:

```typescript
// src/routes/dashboard/+page.ts
import { invoke } from '@tauri-apps/api/core';
import type { PageLoad } from './$types';

export const load: PageLoad = async () => {
  const stats = await invoke<DashboardStats>('get_dashboard_stats');
  return { stats };
};
```

If you use SSG with prerendering on specific routes (`export const prerender = true`), know that load functions **will not** have access to Tauri APIs during the build — `window.__TAURI__` does not exist at build time. SPA mode (no prerendering) is the safe default.

## Calling Tauri from Svelte Components

All Tauri API calls must run **client-side only**. With `ssr = false` globally, every component is client-only by default. Use `invoke` for Rust commands, `listen`/`emit` for events:

```svelte
<script lang="ts">
  import { invoke } from '@tauri-apps/api/core';
  import { listen } from '@tauri-apps/api/event';
  import { onMount, onDestroy } from 'svelte';

  let data = $state<string>('');
  let unlisten: (() => void) | undefined;

  onMount(async () => {
    data = await invoke<string>('greet', { name: 'World' });
    unlisten = await listen('backend-event', (e) => {
      console.log('Received:', e.payload);
    });
  });

  onDestroy(() => unlisten?.());
</script>

<p>{data}</p>
```

**Cleanup is mandatory.** Always call `unlisten()` in `onDestroy` for event listeners to prevent memory leaks across navigation.

## Routing

SvelteKit's client-side router works normally — `goto()`, `<a href>`, `+layout.svelte` nesting all function. The static adapter with `fallback: 'index.html'` handles deep-link fallbacks.

One caveat: **avoid `window.location` for navigation.** It triggers a full page reload inside the webview. Use SvelteKit's `goto()` instead:

```typescript
import { goto } from '$app/navigation';

// Correct — SPA navigation
goto('/settings');

// Wrong — full reload, loses app state
window.location.href = '/settings';
```

## Configuration Invariants

These files form the Tauri + SvelteKit contract. Do not alter without understanding the consequences.

**`svelte.config.js`** — must use `adapter-static` with `fallback: 'index.html'`:
```javascript
import adapter from '@sveltejs/adapter-static';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

const config = {
  preprocess: vitePreprocess(),
  kit: {
    adapter: adapter({
      fallback: 'index.html',
    }),
  },
};

export default config;
```

**`tauri.conf.json`** — `frontendDist` points to `../build` (SvelteKit default output):
```json
{
  "build": {
    "beforeDevCommand": "pnpm dev",
    "beforeBuildCommand": "pnpm build",
    "devUrl": "http://localhost:5173",
    "frontendDist": "../build"
  }
}
```

**`src/routes/+layout.ts`** — SSR disabled globally:
```typescript
export const ssr = false;
```

## Environment Variables

`$env/dynamic/private` and `$env/static/private` are server-only — do not use them.

For public build-time values, use `$env/static/public` (prefixed `PUBLIC_` in `.env`):
```typescript
import { PUBLIC_API_URL } from '$env/static/public';
```

For runtime secrets or native-level configuration, use Tauri's managed state or the `tauri-plugin-store` for key-value persistence.

## Prerendering vs SPA

| Mode | `prerender` | `ssr` | Tauri API in load? | Use case |
|------|------------|-------|-------------------|----------|
| **SPA** (default) | `false` | `false` | ✅ Yes | Most Tauri apps — all rendering client-side |
| **SSG** | `true` | `false` | ❌ No (build-time only) | Static marketing pages within the app |
| **SSG + selective** | per-route | `false` | ⚠️ Only in non-prerendered routes | Hybrid — static shells with dynamic content |

When in doubt, use **SPA mode** — it is the path of least surprise with Tauri.

## Known Issues

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `ReferenceError: window is not defined` | SSR enabled, code runs on "server" (build) | Ensure `export const ssr = false` in root `+layout.ts` |
| `__TAURI__` is undefined | Prerendering runs load at build time | Disable prerender for routes using Tauri APIs |
| 404 on deep-link refresh | Missing `fallback` in adapter-static config | Set `fallback: 'index.html'` in `svelte.config.js` |
| Form action error | Using `+page.server.ts` with actions | Move to client-side handler with `invoke()` |
| Blank page after navigation | `window.location` used instead of `goto()` | Use `goto()` from `$app/navigation` |
