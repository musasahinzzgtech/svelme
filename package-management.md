# Package Management

> **Svelme** · Bun 1.3.12 · No npm, no yarn

---

## Table of Contents

1. [Why Bun](#1-why-bun)
2. [Setup](#2-setup)
3. [Lockfile](#3-lockfile)
4. [Scripts Reference](#4-scripts-reference)
5. [Dependency Categories](#5-dependency-categories)
6. [Adding Dependencies](#6-adding-dependencies)
7. [Version Strategy](#7-version-strategy)
8. [CI / CD Considerations](#8-ci--cd-considerations)

---

## 1. Why Bun

| Feature | npm | yarn | bun |
|---|---|---|---|
| Install speed | Baseline | ~2× faster | **~10–25× faster** |
| Native TypeScript | ❌ | ❌ | ✅ (no ts-node needed) |
| Built-in test runner | ❌ | ❌ | ✅ |
| Native JSX | ❌ | ❌ | ✅ |
| Lockfile format | `package-lock.json` | `yarn.lock` | `bun.lock` |
| Drop-in npm replacement | — | Mostly | ✅ |

The project uses Bun as the **package manager only** — the runtime is still Node.js/Vite for the web build. The `"packageManager"` field in `package.json` pins the exact version:

```json
{
  "packageManager": "bun@1.3.12"
}
```

---

## 2. Setup

### Installing Bun

```bash
# macOS / Linux
curl -fsSL https://bun.sh/install | bash

# Or via Homebrew
brew install bun

# Verify
bun --version  # Should print 1.3.12 or compatible
```

### First-time project setup

```bash
cd btr/

# Install all dependencies (reads bun.lock)
bun install

# Start dev server
bun run dev
```

### Upgrading Bun

```bash
bun upgrade          # Upgrades to latest stable
bun upgrade --canary # Upgrades to canary (unstable)
```

---

## 3. Lockfile

The project uses **`bun.lock`** (not `package-lock.json` or `yarn.lock`).

### What's committed

```
✅ bun.lock         — committed, ensures reproducible installs
✅ package.json     — committed
❌ node_modules/    — gitignored, regenerated via `bun install`
❌ package-lock.json — deleted, not used
```

### Lockfile format

`bun.lock` is a binary lockfile (not human-readable JSON). It is deterministic — given the same `package.json`, `bun install` always produces the same `node_modules`.

### Why not `package-lock.json`?

- `bun.lock` is smaller and faster to parse
- Using both would cause conflicts — only one lockfile should exist per project
- `bun install` ignores `package-lock.json` (it reads `bun.lock` or `package.json`)

---

## 4. Scripts Reference

All scripts in `package.json` are run with `bun run <script>`:

```json
{
  "scripts": {
    "dev":             "vite dev",
    "build":           "vite build",
    "preview":         "vite preview",
    "check":           "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json",
    "storybook":       "storybook dev -p 6006",
    "build-storybook": "storybook build",
    "test":            "vitest --project=storybook"
  }
}
```

| Command | What it does |
|---|---|
| `bun run dev` | Start Vite dev server on `http://localhost:5173` with HMR |
| `bun run build` | Production build → `build/` directory |
| `bun run preview` | Serve the production build locally |
| `bun run check` | Type-check all Svelte + TypeScript files (no emit) |
| `bun run storybook` | Start Storybook on `http://localhost:6006` |
| `bun run build-storybook` | Export static Storybook to `storybook-static/` |
| `bun run test` | Run Storybook interaction tests via Playwright + Vitest |

### Shortcuts

Bun supports running scripts without the `run` keyword for known scripts:

```bash
bun dev        # Same as: bun run dev
bun build      # Same as: bun run build
```

---

## 5. Dependency Categories

### `dependencies` — shipped to the browser

```json
{
  "@reduxjs/toolkit":    "^2.12.0",   // Redux + RTK Query
  "@storybook/sveltekit":"^10.4.1",   // Storybook SvelteKit integration
  "@sveltejs/adapter-auto": "^7.0.1", // SvelteKit deployment adapter
  "@sveltejs/kit":       "^2.61.1",   // SvelteKit framework
  "@tailwindcss/vite":   "^4.3.0",    // TailwindCSS v4 Vite plugin
  "axios":               "^1.16.1",   // HTTP client
  "i18next":             "^26.3.0",   // Internationalization
  "redux-logger":        "^3.0.6",    // Redux console logger
  "redux-observable":    "^3.0.0-rc.3", // RxJS middleware for Redux
  "rxjs":                "^7.8.2",    // Reactive Extensions
  "tailwindcss":         "^4.3.0"     // CSS framework
}
```

### `devDependencies` — development tools only

```json
{
  "@chromatic-com/storybook":     "^5.2.1",    // Visual regression (Chromatic)
  "@storybook/addon-a11y":        "^10.4.1",   // Accessibility checks
  "@storybook/addon-docs":        "^10.4.1",   // Auto-generated docs
  "@storybook/addon-svelte-csf":  "^5.1.2",    // Native Svelte story format
  "@storybook/addon-vitest":      "^10.4.1",   // Run tests in Storybook
  "@storybook/svelte-vite":       "^10.4.1",   // Storybook Svelte+Vite (fallback)
  "@sveltejs/vite-plugin-svelte": "^7.1.2",    // Svelte Vite plugin
  "@tsconfig/svelte":             "^5.0.8",    // Base TypeScript config for Svelte
  "@types/node":                  "^24.12.3",  // Node.js type definitions
  "@types/redux-logger":          "^3.0.13",   // Redux logger types
  "@vitest/browser-playwright":   "^4.1.7",    // Playwright browser provider for Vitest
  "@vitest/coverage-v8":          "^4.1.7",    // V8 code coverage
  "playwright":                   "^1.60.0",   // Browser automation
  "storybook":                    "^10.4.1",   // Storybook core
  "svelte":                       "^5.55.5",   // Svelte compiler
  "svelte-check":                 "^4.4.8",    // Svelte type checker
  "typescript":                   "~6.0.2",    // TypeScript compiler
  "vite":                         "^8.0.12",   // Build tool
  "vitest":                       "^4.1.7"     // Unit/integration test runner
}
```

---

## 6. Adding Dependencies

### Runtime dependency

```bash
bun add axios                    # Adds to dependencies
bun add axios@1.16.1             # Exact version
```

### Dev dependency

```bash
bun add -d vitest                # Adds to devDependencies
bun add -d @types/node           # Type definitions
```

### Removing a dependency

```bash
bun remove redux-logger          # Removes from package.json + bun.lock
```

### Updating dependencies

```bash
bun update                       # Update all to latest within semver range
bun update axios                 # Update specific package
bun update --latest              # Update all, ignoring semver range (use carefully)
```

---

## 7. Version Strategy

| Prefix | Meaning | Example |
|---|---|---|
| `^` (caret) | Compatible releases (same major) | `^2.12.0` → any `2.x.x ≥ 2.12.0` |
| `~` (tilde) | Patch releases only | `~6.0.2` → any `6.0.x ≥ 6.0.2` |
| No prefix | Exact version | `1.3.12` → exactly this version |

### Current pinning choices

- **TypeScript** uses `~6.0.2` (tilde) — TypeScript minor versions can be breaking
- **Bun** is pinned exactly in `packageManager` field — runtime version must match
- Everything else uses `^` (caret) — allow minor + patch updates

---

## 8. CI / CD Considerations

### Recommended CI install command

```bash
bun install --frozen-lockfile   # Fails if bun.lock is out of sync with package.json
```

This ensures CI uses exactly the same versions as the `bun.lock` — no silent upgrades.

### Caching

Cache the `~/.bun` directory between CI runs for faster installs:

```yaml
# GitHub Actions example
- name: Cache bun modules
  uses: actions/cache@v4
  with:
    path: ~/.bun
    key: ${{ runner.os }}-bun-${{ hashFiles('bun.lock') }}
    restore-keys: ${{ runner.os }}-bun-

- name: Install dependencies
  run: bun install --frozen-lockfile
```

### Environment variable injection

Vite reads `.env` files automatically. For CI/production, inject `VITE_*` variables as environment variables in the build step:

```bash
VITE_API_BASE_URL=https://api.aloha.com bun run build
VITE_WS_URL=wss://api.aloha.com bun run build
```

### Playwright browsers

Playwright browser binaries must be installed separately:

```bash
bunx playwright install --with-deps chromium
bun run test
```
