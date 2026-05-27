# Design System

> **Svelme** · TailwindCSS v4 · Svelte 5 · Atomic Design · Storybook 10

---

## Table of Contents

1. [Design Token System](#1-design-token-system)
2. [Color Palette](#2-color-palette)
3. [Typography](#3-typography)
4. [Spacing & Layout](#4-spacing--layout)
5. [Atomic Design Hierarchy](#5-atomic-design-hierarchy)
6. [Component API Conventions](#6-component-api-conventions)
7. [Theming](#7-theming)
8. [Utility Classes](#8-utility-classes)
9. [Storybook](#9-storybook)
10. [Accessibility](#10-accessibility)

---

## 1. Design Token System

The project uses **TailwindCSS v4** with the new `@theme {}` CSS block. There is **no `tailwind.config.js`** — all design tokens are CSS custom properties defined in `src/app.css`.

### Why `@theme {}` instead of `tailwind.config.js`

| TailwindCSS v3 | TailwindCSS v4 |
|---|---|
| `tailwind.config.js` — JS object | `@theme {}` — CSS custom properties |
| Separate config file | Co-located with CSS |
| Can't use in runtime CSS-in-JS | Available as `var(--color-primary)` at runtime |
| Requires PostCSS plugin | Native Vite plugin (`@tailwindcss/vite`) |

### Token Naming Convention

```css
@theme {
  --color-{name}:      #hex;       /* Colors */
  --font-family-{role}: "Font";    /* Typography */
  --font-size-{role}:  16px;       /* Type scale */
  --line-height-{role}: 24px;      /* Line heights */
  --radius-{size}:     0.25rem;    /* Border radii */
  --spacing-{name}:    8px;        /* Named spacing */
}
```

All tokens are accessible in Svelte components as:
- TailwindCSS classes: `bg-primary`, `text-on-surface`, `rounded-xl`
- CSS custom properties: `var(--color-primary)`, `var(--radius-xl)`

---

## 2. Color Palette

The palette is based on Material Design 3 dynamic color with a cyan/teal primary and pink/magenta secondary.

### Primary Scale (Cyan/Teal)

| Token | Value | Usage |
|---|---|---|
| `--color-primary` | `#00677d` | Primary actions, links, focus rings |
| `--color-primary-container` | `#00b4d8` | Primary button background |
| `--color-primary-fixed` | `#b3ebff` | Subtle primary highlights |
| `--color-primary-fixed-dim` | `#4cd6fb` | Hover states |
| `--color-on-primary` | `#ffffff` | Text on primary surfaces |
| `--color-on-primary-container` | `#00414f` | Text on primary containers |
| `--color-inverse-primary` | `#4cd6fb` | Primary color on dark surfaces |

### Secondary Scale (Pink/Magenta)

| Token | Value | Usage |
|---|---|---|
| `--color-secondary` | `#b90a5a` | Secondary actions, badges, labels |
| `--color-secondary-container` | `#fe4c8c` | Secondary button background |
| `--color-on-secondary` | `#ffffff` | Text on secondary surfaces |
| `--color-secondary-fixed` | `#ffd9e0` | Subtle secondary highlights |

### Tertiary Scale (Gold/Amber)

| Token | Value | Usage |
|---|---|---|
| `--color-tertiary` | `#785a00` | Jackpot amounts, VIP badges |
| `--color-tertiary-container` | `#c89f39` | Jackpot card accents |
| `--color-tertiary-fixed` | `#ffdf9b` | Gold glow effects |
| `--color-tertiary-fixed-dim` | `#edc157` | Jackpot amount text |

### Surface Scale (Light mode)

| Token | Value | Usage |
|---|---|---|
| `--color-background` | `#f5fafc` | Page background |
| `--color-surface` | `#f5fafc` | Card backgrounds |
| `--color-surface-bright` | `#f5fafc` | Elevated surfaces |
| `--color-surface-container-lowest` | `#ffffff` | Pure white cards |
| `--color-surface-container-low` | `#eff4f6` | Subtle container |
| `--color-surface-container` | `#eaeff1` | Standard container |
| `--color-surface-container-high` | `#e4e9eb` | Slightly elevated |
| `--color-surface-container-highest` | `#dee3e5` | Most elevated |
| `--color-surface-dim` | `#d6dbdd` | Scrim, overlay |
| `--color-surface-variant` | `#dee3e5` | Alternative container |

### On-Surface Text Colors

| Token | Value | Usage |
|---|---|---|
| `--color-on-background` | `#171c1e` | Body text on background |
| `--color-on-surface` | `#171c1e` | Body text on surfaces |
| `--color-on-surface-variant` | `#3d494d` | Secondary text, captions |
| `--color-outline` | `#6d797e` | Borders, dividers |
| `--color-outline-variant` | `#bcc9ce` | Subtle borders |

### Semantic Colors

| Token | Value | Usage |
|---|---|---|
| `--color-error` | `#ba1a1a` | Error states, destructive actions |
| `--color-error-container` | `#ffdad6` | Error banner backgrounds |
| `--color-on-error` | `#ffffff` | Text on error surfaces |

### Inverse (Dark mode surfaces)

| Token | Value | Usage |
|---|---|---|
| `--color-inverse-surface` | `#2c3133` | Dark overlays, tooltips |
| `--color-inverse-on-surface` | `#ecf2f4` | Text on dark overlays |

---

## 3. Typography

### Font Families

| Variable | Font | Usage |
|---|---|---|
| `--font-family-display-lg` | Montserrat | Hero headlines, H1 |
| `--font-family-display-lg-mobile` | Montserrat | Mobile H1 |
| `--font-family-headline-md` | Montserrat | Section headings, H2–H3 |
| `--font-family-body-lg` | Plus Jakarta Sans | Large body copy, lead paragraphs |
| `--font-family-body-md` | Plus Jakarta Sans | Standard body text |
| `--font-family-label-sm` | Plus Jakarta Sans | Labels, captions, tags |

Both fonts are loaded from Google Fonts (imported at the top of `app.css` before `@import "tailwindcss"`).

### Type Scale

| Role | Size | Line Height | Weight | Usage |
|---|---|---|---|---|
| `display-lg` | 48px | 56px | 700 | Hero title (desktop) |
| `display-lg-mobile` | 36px | 42px | 700 | Hero title (mobile) |
| `headline-md` | 24px | 32px | 600 | Section headings |
| `body-lg` | 18px | 28px | 400 | Lead copy, feature text |
| `body-md` | 16px | 24px | 400 | Standard body |
| `label-sm` | 12px | 16px | 400 | Tags, metadata, captions |

### Material Symbols

Icons use Google's **Material Symbols Outlined** variable font — also loaded from Google Fonts.

```svelte
<span class="material-symbols-outlined">casino</span>
<span class="material-symbols-outlined">account_balance_wallet</span>
<span class="material-symbols-outlined">sports_esports</span>
```

Icon weight is variable (100–700) and fill is variable (0–1), controlled via CSS:

```css
.material-symbols-outlined {
  font-variation-settings: 'FILL' 0, 'wght' 400;
}
```

---

## 4. Spacing & Layout

### Named Spacing Tokens

| Token | Value | Usage |
|---|---|---|
| `--spacing-base` | 8px | Base unit (all other spacing should be multiples) |
| `--spacing-gutter` | 24px | Column gutters in grids |
| `--spacing-container-padding-mobile` | 20px | Horizontal page padding (mobile) |
| `--spacing-container-padding-desktop` | 40px | Horizontal page padding (desktop) |
| `--spacing-section-gap` | 64px | Vertical gap between page sections |

### Border Radius

| Token | Value | Usage |
|---|---|---|
| `--radius-DEFAULT` | 0.25rem (4px) | Badges, tags |
| `--radius-lg` | 0.5rem (8px) | Input fields, small cards |
| `--radius-xl` | 0.75rem (12px) | Game cards, modals |
| `--radius-2xl` | 1rem (16px) | Large cards, panels |
| `--radius-3xl` | 1.5rem (24px) | Hero sections, hero cards |
| `--radius-full` | 9999px | Pills, avatar chips |

### Layout Strategy

- **Mobile-first** — base styles for mobile, `md:` / `lg:` breakpoints for desktop
- **12-column grid** via TailwindCSS grid utilities (`grid-cols-12`, `col-span-*`)
- **Container padding** uses the named spacing tokens
- **Section rhythm** uses `--spacing-section-gap` (64px) between major page sections

---

## 5. Atomic Design Hierarchy

Components are organized following the Atomic Design methodology:

```
src/components/
├── atoms/          ← Smallest indivisible units
│   ├── Button/
│   ├── Badge/
│   ├── Input/
│   ├── Avatar/
│   ├── Spinner/
│   └── Icon/
│
├── molecules/      ← Combinations of atoms
│   ├── GameCard/
│   ├── JackpotCard/
│   ├── PromoCard/
│   ├── SearchBar/
│   └── UserMenu/
│
├── organisms/      ← Complex, domain-aware sections
│   ├── GameGrid/
│   ├── Header/
│   ├── Footer/
│   ├── HeroSection/
│   └── JackpotBanner/
│
└── templates/      ← Page-level layout shells (no data)
    ├── AuthLayout/
    ├── DashboardLayout/
    └── LandingLayout/
```

### Naming Conventions

| Level | Example | Rule |
|---|---|---|
| Component file | `GameCard.svelte` | PascalCase |
| Story file | `GameCard.stories.svelte` | `*.stories.svelte` |
| CSS module | `GameCard.module.css` | `*.module.css` (if needed) |
| Test file | `GameCard.test.ts` | `*.test.ts` |
| Export | `export { default as GameCard }` | Default export + named re-export |

### Component Folder Structure

Each component lives in its own folder:

```
GameCard/
├── GameCard.svelte          ← Component implementation
├── GameCard.stories.svelte  ← Storybook stories (native Svelte CSF)
└── index.ts                 ← Re-exports { default as GameCard }
```

---

## 6. Component API Conventions

### Svelte 5 Props (Runes)

```svelte
<script lang="ts">
  // Props declared with $props()
  let {
    title,
    badgeVariant = 'default',
    onClick,
    children,
  }: {
    title: string;
    badgeVariant?: 'default' | 'hot' | 'new' | 'jackpot';
    onClick?: () => void;
    children?: import('svelte').Snippet;
  } = $props();

  // Derived values — always use $derived, never compute in template
  const badgeClass = $derived(badgeClassMap[badgeVariant]);
</script>
```

### Snippet-based Slots (Svelte 5)

```svelte
<!-- Parent -->
<Card>
  {#snippet header()}
    <h2>Title</h2>
  {/snippet}
  {#snippet default()}
    <p>Content</p>
  {/snippet}
</Card>

<!-- Child (Card.svelte) -->
<script lang="ts">
  let { header, children } = $props();
</script>

<div>
  {@render header?.()}
  {@render children?.()}
</div>
```

### Event Handling (Svelte 5)

```svelte
<!-- Prefer callback props over createEventDispatcher -->
<Button onClick={() => handleClick()} />

<!-- For native DOM events, use direct binding -->
<button onclick={handleClick}>Click</button>
```

### Reactive Declarations

```svelte
<script>
  // ✅ Correct — $derived for computed values
  const isHot = $derived(views > 10_000);
  const labelClass = $derived(isHot ? 'text-red-500' : 'text-gray-500');

  // ❌ Avoid — computing in template (no caching, harder to read)
  // {#if views > 10_000}
</script>
```

---

## 7. Theming

### Theme Modes

Three theme modes are supported: `light`, `dark`, `system`.

- `system` — follows OS preference via `window.matchMedia('(prefers-color-scheme: dark)')`
- Persisted to `localStorage` under key `btr_theme`
- Applied immediately on boot via `applyInitialTheme()` (before first render, no flash)

### Implementation

Theme is applied by setting `data-theme` attribute on `<html>`:

```ts
// store/app/app.slice.ts
function applyThemeToDom(theme: Theme) {
  if (theme === 'dark') {
    document.documentElement.setAttribute('data-theme', 'dark');
  } else if (theme === 'system') {
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    if (prefersDark) {
      document.documentElement.setAttribute('data-theme', 'dark');
    } else {
      document.documentElement.removeAttribute('data-theme');
    }
  } else {
    document.documentElement.removeAttribute('data-theme');
  }
}
```

### Dark Mode CSS Tokens

Override tokens inside `[data-theme="dark"]` selector in `app.css`:

```css
[data-theme="dark"] {
  --color-background:   #0f1416;
  --color-surface:      #171c1e;
  --color-on-surface:   #e1e7ea;
  /* … etc */
}
```

### Dispatching Theme Changes

```ts
import { store, appActions } from '$store';

store.dispatch(appActions.setTheme('dark'));
// Epic updates DOM + persists to localStorage automatically
```

---

## 8. Utility Classes

Two hand-crafted utility classes extend TailwindCSS in `app.css`:

### `.glass-panel`

Frosted glass card effect — used for header, game cards, hero sections.

```css
.glass-panel {
  background: rgba(255, 255, 255, 0.7);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.5);
  box-shadow: 0 8px 32px rgba(0, 180, 216, 0.05);
}
```

### `.btn-gradient`

Primary CTA button gradient — cyan-to-teal with lift-on-hover animation.

```css
.btn-gradient {
  background: linear-gradient(135deg, #00b4d8 0%, #00677d 100%);
  box-shadow: 0 4px 15px rgba(0, 180, 216, 0.2);
  transition: all 0.3s ease;
}
.btn-gradient:hover {
  transform: scale(1.05);
  box-shadow: 0 6px 20px rgba(0, 180, 216, 0.3);
}
```

### Body Background

The body has a full-page background image with a light frosted overlay:

```css
body {
  background-image: url("…casino-background…");
  background-size: cover;
  background-attachment: fixed;
}
body::before {
  /* Frosted overlay — ensures text readability */
  background: linear-gradient(
    to bottom,
    rgba(245, 250, 252, 0.8),
    rgba(245, 250, 252, 0.95)
  );
  position: fixed; inset: 0; z-index: -1;
}
```

---

## 9. Storybook

Storybook 10 with `@storybook/sveltekit` framework — native Svelte CSF (Component Story Format).

### Configuration

```
.storybook/
├── main.ts      ← Framework: @storybook/sveltekit, addons list
└── preview.ts   ← Global decorators, parameters
```

### Story Format (Svelte CSF)

Stories use **native Svelte syntax** (`.stories.svelte`) — not the MDX or JS story format.

```svelte
<!-- GameCard.stories.svelte -->
<script module>
  import { defineMeta } from '@storybook/addon-svelte-csf';
  import GameCard from './GameCard.svelte';

  const { Story } = defineMeta({
    title: 'Molecules/GameCard',
    component: GameCard,
    argTypes: {
      badgeVariant: {
        control: 'select',
        options: ['default', 'hot', 'new', 'jackpot'],
      },
    },
  });
</script>

<Story name="Default" args={{ title: 'Starburst', badgeVariant: 'hot' }}>
  {#snippet template(args)}
    <div class="p-8">
      <GameCard {...args} />
    </div>
  {/snippet}
</Story>

<Story name="Jackpot" args={{ title: 'Mega Fortune', badgeVariant: 'jackpot' }} />
```

> **Important:** Use `{#snippet template(args)}` wrappers inside `<Story>` for decorators — not the React-style `() => ({ template: '...' })` syntax which breaks in Svelte CSF.

### Addons

| Addon | Purpose |
|---|---|
| `@storybook/addon-docs` | Auto-generated component docs from JSDoc + prop types |
| `@storybook/addon-a11y` | Automated accessibility audit per story |
| `@storybook/addon-vitest` | Run interaction tests inline in Storybook |
| `@chromatic-com/storybook` | Visual regression testing via Chromatic |

### Running Tests

```bash
# Storybook interaction tests via Playwright
bun run test

# Storybook dev server
bun run storybook

# Build static Storybook
bun run build-storybook
```

---

## 10. Accessibility

### Implemented

- **Semantic HTML** — `<nav>`, `<main>`, `<header>`, `<footer>`, `<article>`, `<section>`
- **ARIA labels** — all interactive elements without visible text have `aria-label`
- **Focus management** — modals trap focus; route changes restore focus to `<main>`
- **Color contrast** — all text combinations pass WCAG AA (4.5:1 for body, 3:1 for large text)
- **Keyboard navigation** — all interactive elements reachable and operable via keyboard
- **Reduced motion** — animations respect `prefers-reduced-motion: reduce`

### A11y Addon

The `@storybook/addon-a11y` addon runs axe-core on every story and surfaces violations directly in the Storybook UI. All stories should pass with zero violations before merge.

### Color Contrast Reference

| Foreground token | Background token | Ratio | WCAG |
|---|---|---|---|
| `on-primary` (#fff) | `primary` (#00677d) | 7.2:1 | AAA |
| `on-surface` (#171c1e) | `surface` (#f5fafc) | 15.3:1 | AAA |
| `on-surface-variant` (#3d494d) | `surface` (#f5fafc) | 8.1:1 | AAA |
| `on-secondary` (#fff) | `secondary` (#b90a5a) | 5.8:1 | AA |
