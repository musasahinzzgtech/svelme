# State Management

> **Svelme** · Redux Toolkit · RTK Query · Redux Observable · RxJS

---

## Table of Contents

1. [Store Structure](#1-store-structure)
2. [App Slice](#2-app-slice)
3. [Auth Slice](#3-auth-slice)
4. [RTK Query Cache Slice](#4-rtk-query-cache-slice)
5. [Action Type Conventions](#5-action-type-conventions)
6. [Selector Patterns](#6-selector-patterns)
7. [Dispatch Patterns in Svelte 5](#7-dispatch-patterns-in-svelte-5)
8. [Advanced Logger](#8-advanced-logger)
9. [TypeScript Types](#9-typescript-types)

---

## 1. Store Structure

```ts
// store/index.ts
export const store = configureStore({
  reducer: rootReducer,         // app + auth + rtkApi
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({ serializableCheck: false })
      .concat(
        epicMiddleware,          // RxJS epics — async side effects
        baseApi.middleware,      // RTK Query — cache lifecycle
        advancedLogger,          // Dev-only console groups
      ),
  devTools: import.meta.env.DEV,
});

epicMiddleware.run(rootEpic);   // Must run AFTER store creation

export type RootState   = ReturnType<typeof rootReducer>;
export type AppDispatch = typeof store.dispatch;
```

### Root Reducer

```ts
// store/root.reducer.ts
export const rootReducer = combineReducers({
  app:                   appReducer,      // Theme, locale, init status
  auth:                  authReducer,     // User session
  [baseApi.reducerPath]: baseApi.reducer, // RTK Query cache ('rtkApi')
});
```

### Single-Import Convenience

The store `index.ts` re-exports all slices, actions, and selectors:

```ts
export * from './app';    // appActions, appSelectors, AppState
export * from './auth';   // authActions, authSelectors, AuthState
export * from './api';    // RTK Query hooks, CACHE_TAGS
```

Components and epics import from `$store`:

```ts
import { store, appActions, selectTheme, authActions, selectIsAuthenticated } from '$store';
```

---

## 2. App Slice

Manages global application state: initialization, theme, and locale.

### State Shape

```ts
interface AppState {
  status: 'idle' | 'initializing' | 'ready' | 'error';
  theme:  'light' | 'dark' | 'system';
  locale: 'en' | 'tr';
  error:  string | null;
}
```

### Action Types

```ts
// store/app/app.actionTypes.ts
export const APP_ACTION_TYPES = {
  INIT_APP:         'app/initApp',
  INIT_APP_SUCCESS: 'app/initAppSuccess',
  INIT_APP_FAILURE: 'app/initAppFailure',
  SET_THEME:        'app/setTheme',
  SET_LOCALE:           'app/setLocale',
  SET_LOCALE_SUCCESS:   'app/setLocaleSuccess',
  SET_LOCALE_FAILURE:   'app/setLocaleFailure',
} as const;
```

### Key Behaviors

**`initApp` sequence:**

```
dispatch(appActions.initApp())
  → app.status = 'initializing'
  → initAppEpic:
      applyInitialTheme()   — reads localStorage, sets data-theme on <html>
      initI18n()            — reads localStorage, initializes i18next
  → dispatch(appActions.initAppSuccess())
  → app.status = 'ready'
```

**`setTheme`:**

```ts
dispatch(appActions.setTheme('dark'))
// Slice: state.theme = 'dark'
// DOM side effect: document.documentElement.setAttribute('data-theme', 'dark')
// Persistence: localStorage.setItem('btr_theme', 'dark')
```

**`setLocale`:**

```ts
dispatch(appActions.setLocale('tr'))
// setLocaleEpic fires:
//   await changeLocale('tr')    — i18next.changeLanguage()
//   dispatch(setLocaleSuccess('tr'))
// Slice: state.locale = 'tr'
// Persistence: localStorage.setItem('btr_lang', 'tr')
```

### Selectors

```ts
import { selectAppStatus, selectTheme, selectLocale } from '$store';

const status = selectAppStatus(store.getState()); // 'ready'
const theme  = selectTheme(store.getState());     // 'dark'
const locale = selectLocale(store.getState());    // 'tr'
```

---

## 3. Auth Slice

Manages authentication state: user session, login/logout status, errors.

### State Shape

```ts
interface AuthState {
  user:            UserDto | null;
  isAuthenticated: boolean;
  status:          'idle' | 'loading' | 'success' | 'failure';
  error:           string | null;
}
```

### Action Types

```ts
export const AUTH_ACTION_TYPES = {
  LOGIN:                    'auth/login',
  LOGIN_SUCCESS:            'auth/loginSuccess',
  LOGIN_FAILURE:            'auth/loginFailure',
  REGISTER:                 'auth/register',
  REGISTER_SUCCESS:         'auth/registerSuccess',
  REGISTER_FAILURE:         'auth/registerFailure',
  LOGOUT:                   'auth/logout',
  LOGOUT_SUCCESS:           'auth/logoutSuccess',
  CHECK_SESSION:            'auth/checkSession',
  CHECK_SESSION_SUCCESS:    'auth/checkSessionSuccess',
  CHECK_SESSION_FAILURE:    'auth/checkSessionFailure',
  REFRESH_TOKEN:            'auth/refreshToken',
  REFRESH_TOKEN_SUCCESS:    'auth/refreshTokenSuccess',
  REFRESH_TOKEN_FAILURE:    'auth/refreshTokenFailure',
  SET_USER:                 'auth/setUser',
} as const;
```

### Auth Flow

**Login (via RTK Query + Epics):**

```
Component calls: authApi.endpoints.login.initiate({ email, password })
  → RTK Query fires HTTP POST /auth/login
  → onQueryStarted: tokenStore.set(accessToken), tokenStore.setRefresh(refreshToken)
  → matchFulfilled → loginFulfilledEpic:
      window.history.pushState({}, '', '/dashboard')
      dispatch(authActions.loginSuccess({ user }))
  → authSlice: user = payload.user, isAuthenticated = true

  OR

  → matchRejected → loginRejectedEpic:
      dispatch(authActions.loginFailure({ message }))
  → authSlice: error = message, status = 'failure'
```

**Logout:**

```
dispatch(authActions.logout())
  → logoutEpic: RTK Query authApi.logout.initiate()
  → dispatch(authActions.logoutSuccess())
  → logoutSuccessEpic:
      tokenStore.clear()
      window.history.pushState({}, '', '/login')
      RTK Query invalidates ALL tags (cache cleared)
```

**Session check on boot:**

```
INIT_APP_SUCCESS → checkSessionEpic:
  tokenStore.get()
  → No token: dispatch(checkSessionFailure())
  → Token exists: RTK Query authApi.getMe.initiate()
    → Success: dispatch(checkSessionSuccess({ user }))
               authSlice: user = payload.user, isAuthenticated = true
    → Failure: dispatch(checkSessionFailure())
               authSlice: user = null, isAuthenticated = false
```

### Selectors

```ts
import { selectUser, selectIsAuthenticated, selectAuthStatus, selectAuthError } from '$store';

const user             = selectUser(store.getState());            // UserDto | null
const isAuthenticated  = selectIsAuthenticated(store.getState()); // boolean
const status           = selectAuthStatus(store.getState());      // 'loading' | 'success' | ...
const error            = selectAuthError(store.getState());       // string | null
```

---

## 4. RTK Query Cache Slice

The RTK Query cache lives at `state.rtkApi`. It is managed entirely by RTK Query — never mutated directly from application code.

### Cache Key Structure

```
state.rtkApi
  queries:
    getMe(undefined)                      → { data: UserDto, status: 'fulfilled' }
    getGames({ page: 1, category: null }) → { data: GamesListResponse }
    getGames({ page: 2, category: null }) → { data: GamesListResponse }  ← separate key
    getGame('game-123')                   → { data: GameDto }
    getJackpots(undefined)                → { data: JackpotDto[] }
    getProfile(undefined)                 → { data: UserDto }
    getWallet(undefined)                  → { data: WalletDto }
  mutations:
    login                                 → { status: 'fulfilled' }
    logout                                → { status: 'pending' }
```

### Manually Patching Cache

The WebSocket epic and optimistic update pattern both use `util.updateQueryData`:

```ts
// Immer-based draft mutation
dispatch(
  jackpotsApi.util.updateQueryData('getJackpots', undefined, (draft) => {
    const idx = draft.findIndex((j) => j.id === updatedJackpot.id);
    if (idx !== -1) draft[idx] = { ...draft[idx], ...updatedJackpot };
  }),
);
```

### Manually Invalidating Tags

```ts
dispatch(baseApi.util.invalidateTags(['User', 'Wallet']));
// RTK Query will refetch all subscribed queries that provide these tags
```

---

## 5. Action Type Conventions

### Naming

| Pattern | Example | Usage |
|---|---|---|
| `{slice}/{verb}` | `app/setTheme` | Synchronous slice actions |
| `{slice}/{verb}Success` | `auth/loginSuccess` | Successful async result |
| `{slice}/{verb}Failure` | `auth/loginFailure` | Failed async result |
| `{slice}/{verb}Pending` | RTK Query generates these | (Not used manually) |
| `api/{verb}` | `api/searchGames` | Custom actions for epics |

### Action Creators

```ts
// store/auth/auth.actions.ts
import { createAction } from '@reduxjs/toolkit';
import { AUTH_ACTION_TYPES } from './auth.actionTypes';
import type { UserDto } from '$services/app';

export const authActions = {
  login:               createAction<{ email: string; password: string }>(AUTH_ACTION_TYPES.LOGIN),
  loginSuccess:        createAction<{ user: UserDto }>(AUTH_ACTION_TYPES.LOGIN_SUCCESS),
  loginFailure:        createAction<{ message: string }>(AUTH_ACTION_TYPES.LOGIN_FAILURE),
  logout:              createAction(AUTH_ACTION_TYPES.LOGOUT),
  logoutSuccess:       createAction(AUTH_ACTION_TYPES.LOGOUT_SUCCESS),
  checkSessionSuccess: createAction<{ user: UserDto }>(AUTH_ACTION_TYPES.CHECK_SESSION_SUCCESS),
  checkSessionFailure: createAction(AUTH_ACTION_TYPES.CHECK_SESSION_FAILURE),
  // …
};
```

---

## 6. Selector Patterns

### Simple Selectors

```ts
// store/app/app.selectors.ts
import type { RootState } from '../index';

export const selectAppStatus = (state: RootState) => state.app.status;
export const selectTheme     = (state: RootState) => state.app.theme;
export const selectLocale    = (state: RootState) => state.app.locale;
```

### Memoized Selectors

For derived data, use `createSelector` from RTK:

```ts
import { createSelector } from '@reduxjs/toolkit';

export const selectIsAppReady = createSelector(
  selectAppStatus,
  (status) => status === 'ready',
);

export const selectIsAdmin = createSelector(
  selectUser,
  (user) => user?.role === 'admin',
);
```

### RTK Query Selectors

RTK Query generates selectors for every endpoint:

```ts
// Generated automatically — access cache data without hooks
const result = gamesApi.endpoints.getGames.select({ page: 1 })(store.getState());
const { data, isLoading, isError, error } = result;
```

---

## 7. Dispatch Patterns in Svelte 5

### In Components

```svelte
<script lang="ts">
  import { store, authActions } from '$store';

  function handleLogout() {
    store.dispatch(authActions.logout());
  }
</script>

<button onclick={handleLogout}>Log out</button>
```

### Subscribing to State in Components

```svelte
<script lang="ts">
  import { store, selectIsAuthenticated, selectUser } from '$store';

  // $state reactive to store changes
  let isAuthenticated = $state(selectIsAuthenticated(store.getState()));
  let user = $state(selectUser(store.getState()));

  // Subscribe to store updates
  const unsubscribe = store.subscribe(() => {
    isAuthenticated = selectIsAuthenticated(store.getState());
    user = selectUser(store.getState());
  });

  import { onDestroy } from 'svelte';
  onDestroy(unsubscribe);
</script>

{#if isAuthenticated}
  <p>Welcome, {user?.name}!</p>
{/if}
```

### RTK Query Hooks in Svelte

RTK Query hooks aren't React hooks — in Svelte, use the dispatch + subscribe pattern or a wrapper:

```ts
// Option A: Direct initiate (for one-shot calls in epics)
dispatch(gamesApi.endpoints.getGames.initiate({ page: 1 }))

// Option B: Select from cache (in components)
const result = gamesApi.endpoints.getGames.select({ page: 1 })(store.getState())
```

---

## 8. Advanced Logger

The custom dev logger (`store/middleware/logger.middleware.ts`) provides color-coded console groups for every dispatched action.

### Color Scheme

| Action pattern | Color | Hex |
|---|---|---|
| `app/*` | Teal | `#1abc9c` |
| `auth/*` | Burnt orange | `#e67e22` |
| `*/fulfilled` | Green | `#2ecc71` |
| `*/rejected` | Red | `#e74c3c` |
| `*/pending` | Orange | `#f39c12` |
| Everything else | Purple | `#9b59b6` |

### Output Format

```
▶ [auth/loginSuccess]  +42ms
  Action:  { type: 'auth/loginSuccess', payload: { user: { … } } }
  Changed: { auth: { user: [Object], isAuthenticated: true } }
  ─────────────────────────────────────────────────────────────
```

### Configuration

```ts
// Pre-configured export
export const advancedLogger = createAdvancedLogger({
  enabled: import.meta.env.DEV,    // Silent in production
  collapsed: true,                  // console.groupCollapsed
  diff: true,                       // Show only changed state keys
  noiseFilter: ['@@redux/', '@@INIT'],
});
```

---

## 9. TypeScript Types

### Accessing State Type

```ts
import type { RootState, AppDispatch } from '$store';

// Typed dispatch (knows all action creators)
const dispatch: AppDispatch = store.dispatch;

// Typed state selector
function mySelector(state: RootState) {
  return state.auth.user;
}
```

### DTO Types

All API response shapes are defined in `src/services/app/app.dto.ts`:

```ts
export interface UserDto {
  id:       string;
  email:    string;
  name:     string;
  role:     'user' | 'admin' | 'vip';
  avatar?:  string;
  balance?: number;
}

export interface GameDto {
  id:           string;
  title:        string;
  category:     string;
  thumbnail:    string;
  rtp:          number;
  isHot?:       boolean;
  isFeatured?:  boolean;
  jackpotAmount?: number;
}

export interface JackpotDto {
  id:     string;
  name:   string;
  amount: number;
  currency: string;
}

export interface WalletDto {
  balance:  number;
  currency: string;
  bonus:    number;
}

export interface TransactionDto {
  id:        string;
  type:      'deposit' | 'withdrawal' | 'bet' | 'win' | 'bonus';
  amount:    number;
  currency:  string;
  timestamp: string;
  status:    'pending' | 'completed' | 'failed';
}
```

### Epic Type Helper

```ts
import type { Epic } from 'redux-observable';
import type { Action } from '@reduxjs/toolkit';
import type { RootState } from '$store';

// Shorthand for all epics in this project
type AppEpic = Epic<Action, Action, RootState>;
```
