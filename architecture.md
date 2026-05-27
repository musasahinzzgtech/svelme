# Architecture

> **Svelme** · Svelte 5 + SvelteKit · RTK Query + Redux Observable · TailwindCSS v4

---

## Table of Contents

1. [High-Level Overview](#1-high-level-overview)
2. [Technology Stack](#2-technology-stack)
3. [Directory Structure](#3-directory-structure)
4. [SvelteKit Routing](#4-sveltekit-routing)
5. [Application Boot Sequence](#5-application-boot-sequence)
6. [Store Architecture](#6-store-architecture)
7. [Middleware Stack](#7-middleware-stack)
8. [Alias Map](#8-alias-map)
9. [Environment Variables](#9-environment-variables)
10. [Build Pipeline](#10-build-pipeline)

---

## 1. High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser                              │
│                                                             │
│  ┌──────────┐   SvelteKit   ┌──────────────────────────┐   │
│  │  Routes  │ ────────────► │   Svelte 5 Components    │   │
│  │(public)  │               │  (Atomic Design)         │   │
│  │(protected│               └───────────┬──────────────┘   │
│  │(admin)   │                           │ $store/*          │
│  └──────────┘               ┌───────────▼──────────────┐   │
│                             │      Redux Store          │   │
│                             │  ┌──────────────────────┐│   │
│                             │  │   app  │  auth  │ api ││   │
│                             │  └──────────────────────┘│   │
│                             │  ┌──────────────────────┐│   │
│                             │  │   RTK Query Cache    ││   │
│                             │  └──────────────────────┘│   │
│                             │  ┌──────────────────────┐│   │
│                             │  │  Redux Observable    ││   │
│                             │  │  (RxJS Epics)        ││   │
│                             │  └──────────────────────┘│   │
│                             └───────────┬──────────────┘   │
│                                         │                   │
│                             ┌───────────▼──────────────┐   │
│                             │  Axios + Interceptors     │   │
│                             │  (token injection,        │   │
│                             │   silent refresh, queue)  │   │
│                             └───────────┬──────────────┘   │
└─────────────────────────────────────────┼─────────────────┘
                                          │
                              ┌───────────▼───────────┐
                              │    Backend REST API    │
                              │    + WebSocket server  │
                              └───────────────────────┘
```

### Key Design Decisions

| Decision | Rationale |
|---|---|
| RTK Query for HTTP | Caching, deduplication, polling, AbortController, invalidation — all free |
| Redux Observable for orchestration | Cross-feature side effects, WebSocket streams, navigation after async actions |
| Epics react to RTK Query lifecycle | `matchFulfilled`/`matchRejected` bridge the two systems cleanly |
| Axios with interceptors | Single place for auth token injection, 401 refresh queue, error normalization |
| Svelte 5 runes | Fine-grained reactivity without virtual DOM overhead |

---

## 2. Technology Stack

| Layer | Technology | Version |
|---|---|---|
| UI Framework | Svelte | 5.x (runes) |
| Meta-framework | SvelteKit | 2.x |
| Build tool | Vite | 8.x |
| CSS | TailwindCSS | 4.x (`@theme {}`) |
| State management | Redux Toolkit | 2.x |
| Server state | RTK Query | (bundled in RTK) |
| Async middleware | Redux Observable | 3.x-rc |
| Reactive streams | RxJS | 7.x |
| HTTP client | Axios | 1.x |
| Internationalization | i18next | 26.x |
| Component docs | Storybook | 10.x |
| Testing | Vitest + Playwright | 4.x + 1.x |
| Package manager | Bun | 1.3.12 |
| Language | TypeScript | 6.x (strict) |

---

## 3. Directory Structure

```
btr/
├── src/
│   ├── api/                    # HTTP transport layer
│   │   ├── index.ts            # Axios instance, interceptors, tokenStore
│   │   └── library/
│   │       └── app/            # Endpoint constants (AUTH, GAMES, …)
│   │           └── index.ts
│   │
│   ├── services/               # Singleton service classes
│   │   └── app/
│   │       ├── app.service.ts  # AppService — wraps all domain calls
│   │       └── app.dto.ts      # TypeScript DTOs
│   │
│   ├── store/                  # Redux store
│   │   ├── index.ts            # configureStore, exports
│   │   ├── root.reducer.ts     # combineReducers
│   │   ├── root.epic.ts        # combineEpics
│   │   ├── middleware/
│   │   │   └── logger.middleware.ts   # Color-coded dev logger
│   │   ├── app/                # App slice (theme, locale, init)
│   │   │   ├── app.actionTypes.ts
│   │   │   ├── app.actions.ts
│   │   │   ├── app.slice.ts
│   │   │   ├── app.epics.ts
│   │   │   ├── app.selectors.ts
│   │   │   └── index.ts
│   │   ├── auth/               # Auth slice (session, login, logout)
│   │   │   ├── auth.actionTypes.ts
│   │   │   ├── auth.actions.ts
│   │   │   ├── auth.slice.ts
│   │   │   ├── auth.epics.ts
│   │   │   ├── auth.selectors.ts
│   │   │   └── index.ts
│   │   └── api/                # RTK Query APIs
│   │       ├── base.api.ts     # createApi, axiosBaseQuery, CACHE_TAGS
│   │       ├── auth.api.ts     # login, logout, getMe
│   │       ├── games.api.ts    # getGames, getFeaturedGames, getGame
│   │       ├── jackpots.api.ts # getJackpots, getJackpot
│   │       ├── jackpots.epic.ts# WebSocket stream + gameSearch debounce
│   │       ├── user.api.ts     # getProfile, updateProfile, getWallet
│   │       └── index.ts
│   │
│   ├── i18n/                   # Internationalization
│   │   ├── index.ts            # i18next setup, initI18n(), changeLocale()
│   │   └── locales/
│   │       ├── en.json
│   │       └── tr.json
│   │
│   ├── components/             # Atomic Design component library
│   │   ├── atoms/
│   │   ├── molecules/
│   │   ├── organisms/
│   │   └── templates/
│   │
│   ├── routes/                 # SvelteKit file-based routing
│   │   ├── +layout.svelte      # Root layout (dispatches initApp)
│   │   ├── (public)/           # No auth required
│   │   ├── (protected)/        # Auth guard
│   │   └── (admin)/            # Auth + admin role guard
│   │
│   ├── app.html                # SvelteKit HTML template
│   ├── app.css                 # TailwindCSS v4 + @theme tokens
│   └── app.d.ts                # Global TypeScript ambient declarations
│
├── .storybook/                 # Storybook configuration
├── docs/                       # Architect documentation (you are here)
├── static/                     # Static assets
├── svelte.config.js            # SvelteKit adapter + path aliases
├── vite.config.ts              # Vite + Vitest config
├── tailwind.config.js          # Not used — TailwindCSS v4 uses @theme
├── tsconfig.json               # Extends .svelte-kit/tsconfig.json
└── package.json
```

---

## 4. SvelteKit Routing

### Route Groups

SvelteKit route groups (parenthesized directories) share layouts without affecting URLs.

```
routes/
├── +layout.svelte              ← Root layout: dispatches store.dispatch(initApp())
│
├── (public)/                   ← Public pages — no auth required
│   ├── +layout.svelte          ← Wraps content in AppLayout
│   ├── +page.svelte            ← / (Landing page)
│   ├── login/
│   │   └── +page.svelte        ← /login
│   └── register/
│       └── +page.svelte        ← /register
│
├── (protected)/                ← Requires authentication
│   ├── +layout.ts              ← Auth guard — redirects to /login if not authed
│   ├── +layout.svelte          ← Loading spinner during app init
│   ├── dashboard/
│   │   └── +page.svelte        ← /dashboard
│   ├── games/
│   │   └── +page.svelte        ← /games
│   └── profile/
│       └── +page.svelte        ← /profile
│
└── (admin)/                    ← Requires auth + admin role
    ├── +layout.ts              ← Role guard — redirects non-admins to /dashboard
    ├── +layout.svelte          ← Red admin banner strip
    └── admin/
        └── dashboard/
            └── +page.svelte    ← /admin/dashboard (URL includes /admin/ prefix)
```

### Auth Guard (`(protected)/+layout.ts`)

```ts
import { redirect } from '@sveltejs/kit';
import { store, selectIsAuthenticated, selectAppStatus } from '$store';

export const load = () => {
  const state = store.getState();

  // Allow through while app is still initialising (session check pending)
  if (selectAppStatus(state) === 'initializing') return {};

  if (!selectIsAuthenticated(state)) {
    const redirectTo = encodeURIComponent(window.location.pathname);
    throw redirect(302, `/login?redirectTo=${redirectTo}`);
  }
};
```

### Admin Guard (`(admin)/+layout.ts`)

```ts
// Auth check → Role check → Pass or redirect
if (!selectIsAuthenticated(state)) throw redirect(302, '/login');
if (selectUser(state)?.role !== 'admin') throw redirect(302, '/dashboard');
```

### URL vs File Resolution

| File path | URL |
|---|---|
| `routes/(public)/+page.svelte` | `/` |
| `routes/(public)/login/+page.svelte` | `/login` |
| `routes/(protected)/dashboard/+page.svelte` | `/dashboard` |
| `routes/(admin)/admin/dashboard/+page.svelte` | `/admin/dashboard` |

> **Important:** `(protected)/dashboard` and `(admin)/dashboard` would both resolve to `/dashboard` — this is why the admin route is under `admin/dashboard/`.

---

## 5. Application Boot Sequence

```
Browser loads page
      │
      ▼
+layout.svelte (root)
  onMount → store.dispatch(appActions.initApp())
      │
      ▼
initAppEpic                           [store/app/app.epics.ts]
  bootstrapApp():
    ├── applyInitialTheme()           reads localStorage btr_theme
    └── initI18n()                    reads localStorage btr_lang
                                      → i18next.init()
      │
      ▼
dispatch(initAppSuccess)
      │
      ├──► checkSessionEpic           [store/auth/auth.epics.ts]
      │      tokenStore.get()
      │      if token → RTK Query authApi.getMe.initiate()
      │        ├── success → dispatch(checkSessionSuccess({ user }))
      │        └── failure → dispatch(checkSessionFailure())
      │
      └──► jackpotWsEpic              [store/api/jackpots.epic.ts]
             opens WebSocket /jackpots
             patches RTK Query cache on each message
             exponential back-off reconnect
             tears down on LOGOUT_SUCCESS
```

### Theme Resolution Order

1. `localStorage.getItem('btr_theme')` → `'light'` | `'dark'` | `'system'`
2. If `'system'` → `window.matchMedia('(prefers-color-scheme: dark)')`
3. Default → `'light'`

### Locale Resolution Order

1. `localStorage.getItem('btr_lang')` → `'en'` | `'tr'`
2. `navigator.language` → match against `SUPPORTED_LOCALES`
3. Default → `'en'`

---

## 6. Store Architecture

```
Redux Store
├── app         (AppState)
│   ├── status: 'idle' | 'initializing' | 'ready' | 'error'
│   ├── theme:  'light' | 'dark' | 'system'
│   └── locale: 'en' | 'tr'
│
├── auth        (AuthState)
│   ├── user:          UserDto | null
│   ├── isAuthenticated: boolean
│   ├── status:        'idle' | 'loading' | 'success' | 'failure'
│   └── error:         string | null
│
└── rtkApi      (RTK Query cache)
    ├── auth:         { getMe, login, logout, register }
    ├── games:        { getGames, getFeaturedGames, getGame, getCategories }
    ├── jackpots:     { getJackpots, getJackpot }
    └── user:         { getProfile, updateProfile, getWallet, getTransactions }
```

### Slice vs RTK Query — when to use each

| Concern | Use |
|---|---|
| App-level state (theme, locale, boot status) | Slice |
| Auth session state (user object, isAuthenticated) | Slice |
| Server data (games, jackpots, wallet) | RTK Query |
| Cross-feature side effects (redirect after login) | Epic |
| HTTP transport (caching, dedup, abort) | RTK Query |
| Real-time streams (WebSocket) | Epic → patches RTK Query cache |

---

## 7. Middleware Stack

Middleware executes left-to-right on every dispatched action:

```
dispatch(action)
      │
      ▼
1. epicMiddleware       — intercepts actions, runs RxJS epics
      │
      ▼
2. baseApi.middleware   — RTK Query cache subscriptions, polling, invalidation
      │
      ▼
3. advancedLogger       — color-coded console groups (DEV only)
      │
      ▼
4. Redux core reducer   — state update
```

**Order matters:**
- `epicMiddleware` must come first so it sees actions before state updates
- RTK Query middleware must follow epics (epics can dispatch RTK Query actions)
- Logger must be last to capture final state after all middleware processed

### Advanced Logger

Color scheme by action type:

| Action prefix | Color |
|---|---|
| `app/` | Teal |
| `auth/` | Burnt orange (#e67e22) |
| `*/fulfilled` | Green |
| `*/rejected` | Red |
| `*/pending` | Orange |
| Everything else | Purple |

Noise filters: `@@redux/INIT` and `@@INIT` are silently skipped.

---

## 8. Alias Map

Defined in both `svelte.config.js` (SvelteKit resolution) and `vite.config.ts` (Vite resolution):

| Alias | Resolves to | Notes |
|---|---|---|
| `$components` | `src/components` | Atomic Design components |
| `$pages` | `src/pages` | Page-level Svelte files |
| `$store` | `src/store` | Redux store, actions, selectors |
| `$hooks` | `src/hooks` | Custom Svelte hooks |
| `$utils` | `src/utils` | Pure utility functions |
| `$services` | `src/services` | AppService singleton + DTOs |
| `$api` | `src/api` | Axios instance, tokenStore, endpoints |
| `$routes` | `src/routes` | Route constants |
| `$assets` | `src/assets` | Images, icons, fonts |
| `$i18n` | `src/i18n` | i18next setup, translation helpers |

> **Directory vs file aliases:** All aliases point to directories (not `index.ts` files). This allows sub-path imports like `$api/library/app` to resolve correctly.

---

## 9. Environment Variables

| Variable | Description | Example |
|---|---|---|
| `VITE_API_BASE_URL` | Backend REST API base URL | `https://api.aloha.com` |
| `VITE_WS_URL` | WebSocket server base URL | `wss://api.aloha.com` |

> All `VITE_` prefixed variables are inlined at build time by Vite and accessible via `import.meta.env.*`.

---

## 10. Build Pipeline

```
bun run build
      │
      ├── svelte-kit sync         — generates .svelte-kit/ tsconfig + types
      ├── vite build              — bundles with Rollup
      │   ├── TailwindCSS v4      — JIT CSS from @theme tokens
      │   ├── Svelte compiler     — runes → vanilla JS
      │   └── TypeScript 6        — strict type checking
      └── adapter-auto            — outputs for detected platform (Node/Vercel/CF)
```

### Scripts Reference

| Script | Command | Purpose |
|---|---|---|
| `dev` | `vite dev` | Local dev server with HMR |
| `build` | `vite build` | Production build |
| `preview` | `vite preview` | Preview production build locally |
| `check` | `svelte-kit sync && svelte-check` | Type-check all Svelte + TS files |
| `storybook` | `storybook dev -p 6006` | Component explorer |
| `build-storybook` | `storybook build` | Static Storybook export |
| `test` | `vitest --project=storybook` | Run Storybook interaction tests via Playwright |
