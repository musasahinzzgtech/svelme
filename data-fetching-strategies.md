# Data Fetching Strategies

> **Aloha Casino — btr** · RTK Query · Redux Observable · Axios · WebSocket · RxJS

---

## Table of Contents

1. [Overview](#1-overview)
2. [Axios Transport Layer](#2-axios-transport-layer)
3. [RTK Query](#3-rtk-query)
4. [AbortController Integration](#4-abortcontroller-integration)
5. [Cache Strategy by Domain](#5-cache-strategy-by-domain)
6. [Optimistic Updates](#6-optimistic-updates)
7. [Polling](#7-polling)
8. [WebSocket Streams](#8-websocket-streams)
9. [Search Debounce](#9-search-debounce)
10. [Redux Observable Epics](#10-redux-observable-epics)
11. [Error Handling](#11-error-handling)
12. [Token Refresh Flow](#12-token-refresh-flow)
13. [Decision Guide](#13-decision-guide)

---

## 1. Overview

The project uses a **dual-layer async pattern**:

```
┌─────────────────────────────────────────────────────────────┐
│                    Dual-Layer Async                         │
│                                                             │
│  RTK Query                    Redux Observable (Epics)      │
│  ─────────────────────────    ────────────────────────────  │
│  • HTTP transport             • Cross-feature orchestration │
│  • Request deduplication      • Auth slice sync after login │
│  • Automatic caching          • Navigation (pushState)      │
│  • Cache invalidation         • WebSocket streams           │
│  • AbortController            • Search debounce             │
│  • Retry on 5xx/network       • React to matchFulfilled     │
│  • Polling                    • React to matchRejected      │
│  • Optimistic updates         • Long-running coordination   │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │   Axios Instance    │
               │  • Bearer token     │
               │  • 401 → refresh   │
               │  • Error normalize  │
               └─────────────────────┘
                          │
                          ▼
                    Backend REST API
```

**RTK Query owns HTTP** — it handles caching, deduplication, AbortController, retry, polling.  
**Epics own orchestration** — they react to RTK Query lifecycle actions to update other slices, navigate, and manage WebSocket streams.

---

## 2. Axios Transport Layer

All HTTP goes through a single shared Axios instance (`src/api/index.ts`).

### Instance Configuration

```ts
export const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  headers: { 'Content-Type': 'application/json' },
  timeout: 15_000,
});
```

### Request Interceptor — Token Injection

```ts
api.interceptors.request.use((config) => {
  const token = tokenStore.get();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

### Response Interceptor — Envelope Unwrapping

The backend wraps all responses in `{ data, success, status }`. The interceptor unwraps this:

```ts
api.interceptors.response.use(
  (response) => {
    // Unwrap { data, success, status } envelope
    if (response.data?.data !== undefined) {
      response.data = response.data.data;
    }
    return response;
  },
  (error) => Promise.reject(buildApiError(error)),
);
```

### Token Store

```ts
export const tokenStore = {
  get:        () => localStorage.getItem('btr_access_token'),
  set:        (t: string) => localStorage.setItem('btr_access_token', t),
  clear:      () => { localStorage.removeItem('btr_access_token'); localStorage.removeItem('btr_refresh_token'); },
  getRefresh: () => localStorage.getItem('btr_refresh_token'),
  setRefresh: (t: string) => localStorage.setItem('btr_refresh_token', t),
};
```

### Error Normalization

`buildApiError()` normalizes all errors into a consistent `ApiError` shape:

```ts
interface ApiError {
  message: string;
  status:  number;
  code?:   string;
  errors?: Record<string, string[]>;  // Field-level validation errors
}
```

---

## 3. RTK Query

### Base API (`store/api/base.api.ts`)

```ts
export const baseApi = createApi({
  reducerPath: 'rtkApi',
  baseQuery: axiosBaseQuery,      // Our custom axios-backed base query
  tagTypes: CACHE_TAGS,           // Domain tag universe
  keepUnusedDataFor: 60,          // Global TTL: 60 seconds
  endpoints: () => ({}),          // Endpoints injected per-domain
});
```

### Injected APIs

Rather than one monolithic API object, endpoints are injected per domain:

```ts
// games.api.ts
export const gamesApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({ ... }),
  overrideExisting: false,
});

// auth.api.ts
export const authApi = baseApi.injectEndpoints({ ... });

// user.api.ts
export const userApi = baseApi.injectEndpoints({ ... });

// jackpots.api.ts
export const jackpotsApi = baseApi.injectEndpoints({ ... });
```

All share the same `baseApi` instance → same cache, same tag universe, same reducerPath (`rtkApi`).

### Cache Tag Universe

```ts
export const CACHE_TAGS = [
  'Auth',         // Auth session / getMe
  'User',         // Profile data
  'Games',        // Games list
  'Game',         // Individual game
  'Jackpots',     // Jackpot list
  'Promotions',   // Promo list
  'Wallet',       // Wallet balance
  'Transactions', // Transaction history
] as const;
```

### Tag-Based Cache Invalidation

When a mutation fires, it `invalidatesTags` — RTK Query automatically refetches any subscribed queries that provide those tags.

**Login flow:**

```ts
login: builder.mutation({
  // ...
  invalidatesTags: ['Auth', 'User', 'Wallet', 'Transactions'],
  // After login, stale user/wallet data is automatically re-fetched
})
```

**Logout flow:**

```ts
logout: builder.mutation({
  // ...
  invalidatesTags: ['Auth', 'User', 'Games', 'Jackpots', 'Promotions', 'Wallet', 'Transactions'],
  // Clears entire cache on logout — no stale authenticated data
})
```

**Profile update:**

```ts
updateProfile: builder.mutation({
  // ...
  invalidatesTags: ['User', 'Auth'],
  // After update, profile and auth session re-fetched
})
```

### Deduplication

RTK Query automatically deduplicates concurrent requests with the same cache key. If two components both call `useGetGamesQuery()` simultaneously, only **one** HTTP request is made — both components receive the same cached data.

---

## 4. AbortController Integration

RTK Query passes an `AbortSignal` via the `signal` argument to the base query. The custom `axiosBaseQuery` attaches it to the axios config:

```ts
const rawAxiosBaseQuery: BaseQueryFn<AxiosRequestConfig, unknown, ApiError> =
  async (config, _api, _extraOptions, { signal }) => {
    try {
      const result = await api({
        ...config,
        signal,   // ← Axios 1.x accepts AbortSignal natively
      });
      return { data: result.data };
    } catch (err) {
      // CanceledError = axios abort; AbortError = fetch abort
      if (err.name === 'CanceledError' || err.name === 'AbortError') {
        return { error: { message: 'Request cancelled', status: 0 } };
      }
      return { error: buildApiError(err) };
    }
  };
```

### When RTK Query fires the abort signal

| Scenario | Behavior |
|---|---|
| Component unmounts before response | Signal fired → axios request cancelled |
| New query supersedes previous (e.g. search) | Previous signal fired → no stale response processed |
| User navigates away mid-request | SvelteKit unmounts component → signal fired |
| `refetch()` called with request in-flight | Previous request aborted, new one started |

### Search-as-you-type cancellation

In `jackpots.epic.ts`, the `gameSearchEpic` uses `switchMap` — which **automatically unsubscribes** the previous inner observable when a new value arrives. This triggers the AbortController signal via the RTK Query subscription:

```ts
switchMap((query) =>
  from(
    dispatch(gamesApi.endpoints.getGames.initiate(params)),
  ),
)
// Each new keystroke cancels the previous in-flight request
```

---

## 5. Cache Strategy by Domain

| Endpoint | `keepUnusedDataFor` | Tag | Rationale |
|---|---|---|---|
| `getMe` | 60s (global) | `['Auth']` | Session rarely changes; re-fetched on login |
| `getGames` (list) | 60s (global) | `['Games', 'LIST']` + per-item `['Game', id]` | Moderate freshness |
| `getFeaturedGames` | **30s** | `['Games', 'FEATURED']` | Featured games rotate quickly (lobby freshness) |
| `getGame` (detail) | 60s (global) | `['Game', id]` | Individual game rarely changes |
| `getCategories` | **300s (5min)** | `['Games', 'CATEGORIES']` | Categories almost never change |
| `getJackpots` | **30s** | `['Jackpots', 'LIST']` | Augmented by WebSocket stream |
| `getProfile` | 60s (global) | `['User']` | Updated optimistically on edit |
| `getWallet` | 60s (global) | `['Wallet']` | Balance changes after bets/deposits |
| `getTransactions` | 60s (global) | `['Transactions']` | History grows with each transaction |
| `getNotifications` | **30s** | none | Notifications appear quickly |

### Per-item Game Tags

`getGames` provides both a list tag and per-item tags:

```ts
providesTags: (result) =>
  result
    ? [
        ...result.items.map(({ id }) => ({ type: 'Game' as const, id })),
        { type: 'Games', id: 'LIST' },
      ]
    : [{ type: 'Games', id: 'LIST' }],
```

This allows granular invalidation: updating a single game (e.g. via admin panel) can invalidate only `['Game', specificId]` without busting the entire games list cache.

---

## 6. Optimistic Updates

`updateProfile` uses RTK Query's `onQueryStarted` pattern for immediate UI feedback:

```ts
updateProfile: builder.mutation<UserDto, UpdateProfileRequest>({
  query: (body) => ({ url: APP_ENDPOINTS.USER.UPDATE_PROFILE, method: 'PATCH', data: body }),

  onQueryStarted: async (patch, { dispatch, queryFulfilled }) => {
    // Step 1: Immediately patch the cache (user sees change instantly)
    const patchResult = dispatch(
      userApi.util.updateQueryData('getProfile', undefined, (draft) => {
        Object.assign(draft, patch);  // Immer draft mutation
      }),
    );

    try {
      // Step 2: Wait for server — replace with authoritative data
      const { data } = await queryFulfilled;
      dispatch(
        userApi.util.updateQueryData('getProfile', undefined, (draft) => {
          Object.assign(draft, data);
        }),
      );
    } catch {
      // Step 3: Server rejected — undo the optimistic patch
      patchResult.undo();
    }
  },

  invalidatesTags: ['User', 'Auth'],
}),
```

**UX result:** Profile changes appear instantly. If the server rejects (e.g. validation error), the UI reverts automatically.

---

## 7. Polling

RTK Query supports automatic polling via `pollingInterval` on the query subscription. Wallet balance is a candidate for polling:

```ts
// In a Svelte component
const wallet = useGetWalletQuery(undefined, {
  pollingInterval: 30_000,  // Refetch every 30 seconds
});
```

The `keepUnusedDataFor` on `getWallet` (60s) ensures the cache isn't prematurely evicted between polls.

---

## 8. WebSocket Streams

Live jackpot amounts are streamed via WebSocket and used to patch the RTK Query cache directly — keeping RTK Query as the **single source of truth** while avoiding HTTP polling for real-time data.

### Architecture

```
jackpotWsEpic (RxJS)
      │
      ├── Fires on: INIT_APP_SUCCESS
      ├── Opens: webSocket<JackpotDto>('wss://.../jackpots')
      │
      ├── Each message:
      │   dispatch(jackpotsApi.util.updateQueryData('getJackpots', undefined, draft => {
      │     const idx = draft.findIndex(j => j.id === msg.id);
      │     if (idx !== -1) draft[idx] = { ...draft[idx], ...msg };
      │   }))
      │
      ├── On error:
      │   retryWhen → exponential backoff (1s → 2s → 4s → … → 30s max)
      │
      └── On LOGOUT_SUCCESS:
          takeUntil → socket closed, subscription cleaned up
```

### Implementation

```ts
export const jackpotWsEpic: AppEpic = (action$, _state$, { dispatch }) => {
  let socket$: WebSocketSubject<JackpotDto> | null = null;

  return action$.pipe(
    ofType(APP_ACTION_TYPES.INIT_APP_SUCCESS),
    switchMap(() => {
      socket$ = webSocket<JackpotDto>({
        url: `${WS_URL}/jackpots`,
        openObserver:  { next: () => console.debug('[WS] connected') },
        closeObserver: { next: () => console.debug('[WS] disconnected') },
      });

      return socket$.pipe(
        tap((updatedJackpot) => {
          dispatch(jackpotsApi.util.updateQueryData('getJackpots', undefined, (draft) => {
            const idx = draft.findIndex((j) => j.id === updatedJackpot.id);
            if (idx !== -1) draft[idx] = { ...draft[idx], ...updatedJackpot };
          }));
        }),
        mergeMap(() => EMPTY),   // Epics must return Actions — side effects in tap

        retryWhen((errors$) =>   // Exponential backoff reconnect
          errors$.pipe(
            mergeMap((err, attempt) => {
              const backoff = Math.min(1000 * 2 ** attempt, 30_000);
              console.warn(`[WS] reconnecting in ${backoff}ms`, err);
              return timer(backoff);
            }),
          ),
        ),

        takeUntil(action$.pipe(ofType(AUTH_ACTION_TYPES.LOGOUT_SUCCESS))),
      );
    }),
  );
};
```

### Reconnect Backoff Schedule

| Attempt | Delay |
|---|---|
| 1st disconnect | 1,000ms |
| 2nd disconnect | 2,000ms |
| 3rd disconnect | 4,000ms |
| 4th disconnect | 8,000ms |
| 5th+ disconnect | 30,000ms (capped) |

---

## 9. Search Debounce

Search-as-you-type is coordinated via a custom RxJS epic rather than component-level `setTimeout`:

```ts
// Dispatch this action from the search input's oninput handler
export const searchGamesAction = (query: string) =>
  ({ type: 'api/searchGames', payload: query }) as const;

export const gameSearchEpic: AppEpic = (action$, _state$, { dispatch }) =>
  action$.pipe(
    filter((a) => a.type === 'api/searchGames'),
    map((a) => a.payload),
    debounceTime(350),          // Wait 350ms after last keystroke
    distinctUntilChanged(),     // Skip if query didn't actually change
    switchMap((query) =>        // Cancel previous request on new keystroke
      from(dispatch(gamesApi.endpoints.getGames.initiate({ q: query }))).pipe(
        mergeMap(() => EMPTY),
      ),
    ),
  );
```

**Behaviour:**

1. User types "star" → action dispatched for each letter
2. `debounceTime(350)` — only "star" (after 350ms pause) fires the HTTP call
3. User types "starburst" while "star" request is in-flight → `switchMap` cancels it
4. `distinctUntilChanged` — deleting and re-typing the same query doesn't re-fetch

---

## 10. Redux Observable Epics

### Epic Layers

**Layer A — Direct epics** (non-RTK-Query flows)

| Epic | Trigger | What it does |
|---|---|---|
| `initAppEpic` | `INIT_APP` | Runs `bootstrapApp()`, dispatches `INIT_APP_SUCCESS` |
| `setLocaleEpic` | `SET_LOCALE` | Calls `changeLocale()`, dispatches `SET_LOCALE_SUCCESS` |
| `checkSessionEpic` | `INIT_APP_SUCCESS` | Calls `authApi.getMe.initiate()`, dispatches session result |
| `logoutEpic` | `LOGOUT` | Calls `authApi.logout.initiate()`, dispatches `LOGOUT_SUCCESS` |

**Layer B — RTK Query reaction epics** (react to RTK Query lifecycle)

| Epic | Trigger | What it does |
|---|---|---|
| `loginFulfilledEpic` | `authApi.login.matchFulfilled` | `pushState('/dashboard')`, dispatches `loginSuccess` |
| `loginRejectedEpic` | `authApi.login.matchRejected` | Dispatches `loginFailure` with error message |
| `registerFulfilledEpic` | `authApi.register.matchFulfilled` | `pushState('/dashboard')`, dispatches `registerSuccess` |
| `registerRejectedEpic` | `authApi.register.matchRejected` | Dispatches `registerFailure` |
| `logoutSuccessEpic` | `LOGOUT_SUCCESS` | `tokenStore.clear()`, `pushState('/login')` |

**Layer C — API epics** (streaming + search)

| Epic | Trigger | What it does |
|---|---|---|
| `jackpotWsEpic` | `INIT_APP_SUCCESS` | Opens WebSocket, patches RTK Query cache, reconnects |
| `gameSearchEpic` | `api/searchGames` action | Debounces, cancels previous, fires RTK Query |

### Epic Composition

```ts
// root.epic.ts
export const rootEpic = combineEpics(
  ...appEpics,    // [initAppEpic, setLocaleEpic]
  ...authEpics,   // [checkSessionEpic, logoutEpic, loginFulfilledEpic, ...]
  ...apiEpics,    // [jackpotWsEpic, gameSearchEpic]
);
```

### Epic Type

All epics are typed as:

```ts
type AuthEpic = Epic<Action, Action, RootState>;
//                   ^input  ^output ^state
```

---

## 11. Error Handling

### Error Shape

All errors are normalized to `ApiError`:

```ts
interface ApiError {
  message: string;                              // Human-readable message
  status:  number;                              // HTTP status (0 = network)
  code?:   string;                              // App-level error code
  errors?: Record<string, string[]>;            // Field-level validation errors
}
```

### Retry Strategy

`axiosBaseQuery` is wrapped with RTK Query's `retry` utility:

```ts
const axiosBaseQuery = retry(rawAxiosBaseQuery, {
  maxRetries: 3,
  retryCondition: (error: unknown) => {
    const status = (error as ApiError)?.status ?? 0;
    return status === 0 || status >= 500;  // Network errors + server errors
    // 4xx errors are NOT retried — bad input won't fix itself
  },
});
```

### Epic Error Handling

Epics use `catchError` to prevent the epic stream from dying on errors:

```ts
checkSessionEpic: (action$, _state$, { dispatch }) =>
  action$.pipe(
    ofType(APP_ACTION_TYPES.INIT_APP_SUCCESS),
    mergeMap(() => {
      return from(dispatch(authApi.endpoints.getMe.initiate())).pipe(
        map((result) =>
          result.data
            ? authActions.checkSessionSuccess({ user: result.data })
            : authActions.checkSessionFailure(),
        ),
        catchError(() => of(authActions.checkSessionFailure())),
        // ↑ Epic survives errors — never lets the stream die
      );
    }),
  );
```

---

## 12. Token Refresh Flow

When the API returns `401 Unauthorized`, the axios response interceptor attempts a silent token refresh before failing:

```
Request made with access token
          │
          ▼
Server returns 401
          │
          ▼
Interceptor: is refresh already in progress?
  ├── Yes → queue this request, wait for refresh to complete, then retry
  └── No  → set isRefreshing = true
              │
              ▼
         POST /auth/refresh with refreshToken
              │
         ├── Success → store new tokens
         │             retry all queued requests
         │             resolve original request
         └── Failure → clear all tokens
                       redirect to /login
                       reject all queued requests
```

### Key Properties

- **Only one refresh** in-flight at a time (guarded by `isRefreshing` flag)
- **Queue draining** — all requests that arrived during the refresh are retried in order
- **Always-succeed logout** — `logoutEpic` catches errors and dispatches `logoutSuccess` anyway (local state always clears)

---

## 13. Decision Guide

Use this to decide which fetching strategy to use for a new feature:

```
Is the data server-side and cacheable?
  YES → RTK Query (query endpoint)
    │
    ├── Does it change on mutations? → Add invalidatesTags
    ├── Does it change in real-time? → WebSocket epic patches cache
    ├── Does it need polling?        → pollingInterval on subscription
    └── Does it need immediate UI feedback? → onQueryStarted optimistic update

  NO (client-side state) → Redux slice

Is it a write operation?
  YES → RTK Query (mutation endpoint)
    └── Need side effect after? → Epic reacts to matchFulfilled/matchRejected

Is it a long-running side effect?
  YES → Redux Observable Epic
    └── Examples: navigation after auth, WebSocket lifecycle, search debounce

Is it app initialization?
  YES → initApp action → initAppEpic
    └── Downstream: checkSessionEpic, jackpotWsEpic react to INIT_APP_SUCCESS
```
