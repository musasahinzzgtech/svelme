# Svelme Documentation

> **Svelme** — Svelte 5 + SvelteKit + Redux Toolkit + TailwindCSS v4

---

## Documents

| Document | What it covers |
|---|---|
| [architecture.md](./architecture.md) | Overall system design, SvelteKit routing, store structure, boot sequence, aliases, environment variables, build pipeline |
| [design-systems.md](./design-systems.md) | TailwindCSS v4 tokens, color palette, typography, spacing, Atomic Design hierarchy, component API conventions, theming, Storybook, accessibility |
| [data-fetching-strategies.md](./data-fetching-strategies.md) | RTK Query, AbortController, cache TTLs, optimistic updates, polling, WebSocket streams, Redux Observable epics, error handling, token refresh |
| [state-management.md](./state-management.md) | Store structure, app slice, auth slice, RTK Query cache, action type conventions, selectors, TypeScript types |
| [package-management.md](./package-management.md) | Bun setup, lockfile strategy, scripts, dependency categories, versioning, CI/CD |

---

## Quick Start

```bash
# Install dependencies
bun install

# Start dev server
bun run dev               # http://localhost:5173

# Start Storybook
bun run storybook         # http://localhost:6006

# Type-check
bun run check

# Run tests
bun run test
```

---

## Technology Stack at a Glance

```
UI Layer         Svelte 5 (runes) + SvelteKit
Styling          TailwindCSS v4 (@theme tokens, no config file)
State            Redux Toolkit — slices for app + auth
Server State     RTK Query — HTTP caching, invalidation, optimistic updates
Async Effects    Redux Observable — RxJS epics for side effects
HTTP Client      Axios — interceptors, token injection, 401 refresh queue
Real-time        RxJS webSocket — patches RTK Query cache directly
i18n             i18next — en / tr, localStorage persistence
Component Docs   Storybook 10 — native Svelte CSF stories
Tests            Vitest + Playwright
Package Manager  Bun 1.3.12
Language         TypeScript 6 (strict mode)
```
