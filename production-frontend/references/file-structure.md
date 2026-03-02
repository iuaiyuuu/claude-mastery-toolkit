# File Structure Reference

Framework-specific project layouts for production frontend code. Use the layout that
matches the user's specified framework. When no framework is specified, use the Vanilla
layout.

## Vanilla HTML/CSS/JS

For single-page deliverables, a single HTML file with grouped `<style>` and `<script>`
blocks is acceptable. For multi-page sites:

```
project/
├── index.html
├── css/
│   ├── tokens.css          # Design tokens (custom properties)
│   ├── reset.css            # Minimal reset (box-sizing, margin normalization)
│   └── main.css             # Component and layout styles
├── js/
│   ├── main.js              # Entry point, event delegation, initialization
│   └── modules/             # Feature-specific modules (ES module imports)
│       ├── navigation.js
│       └── form-validation.js
├── assets/
│   ├── images/
│   └── fonts/
└── favicon.ico
```

Key conventions:
- `tokens.css` is imported first, providing all custom properties
- `reset.css` is minimal: box-sizing border-box, zero margins on body, img max-width 100%
- JS entry point uses `defer` attribute and initializes after DOM ready
- ES modules with `type="module"` for modern browsers; no bundler required for simple sites

## React (Vite + TypeScript)

```
src/
├── main.tsx                  # Entry point, renders App
├── App.tsx                   # Root component, routing
├── components/
│   ├── ui/                   # Reusable, generic components
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   └── Modal.tsx
│   └── layout/               # Structural components
│       ├── Header.tsx
│       ├── Footer.tsx
│       └── Sidebar.tsx
├── features/                  # Domain-specific feature modules
│   └── dashboard/
│       ├── DashboardPage.tsx
│       ├── DashboardChart.tsx
│       ├── useDashboardData.ts
│       └── dashboard.module.css
├── hooks/                     # Shared custom hooks
│   ├── useMediaQuery.ts
│   └── useDebounce.ts
├── lib/                       # Utilities, API client, constants
│   ├── api.ts
│   ├── constants.ts
│   └── utils.ts
├── styles/
│   ├── tokens.css             # CSS custom properties
│   ├── reset.css              # Minimal reset
│   └── global.css             # Global styles (typography, base elements)
└── types/
    └── index.ts               # Shared TypeScript types
```

Key conventions:
- Feature folders contain everything related to that feature (component, hook, styles, types)
- `ui/` components are stateless, generic, and reusable across features
- Custom hooks start with `use` and live in `hooks/` if shared, or in the feature folder if local
- CSS Modules for component-scoped styles; `tokens.css` imported globally
- No barrel exports (`index.ts` re-exporting everything) unless the folder has 5+ exports
- TypeScript strict mode enabled

## Vue (Vite + TypeScript)

```
src/
├── main.ts                    # Entry point
├── App.vue                    # Root component
├── components/
│   ├── ui/                    # Reusable components
│   │   ├── BaseButton.vue
│   │   ├── BaseInput.vue
│   │   └── BaseModal.vue
│   └── layout/
│       ├── AppHeader.vue
│       ├── AppFooter.vue
│       └── AppSidebar.vue
├── features/
│   └── dashboard/
│       ├── DashboardPage.vue
│       ├── DashboardChart.vue
│       └── useDashboardData.ts
├── composables/               # Shared composables
│   ├── useMediaQuery.ts
│   └── useDebounce.ts
├── lib/
│   ├── api.ts
│   └── utils.ts
├── styles/
│   ├── tokens.css
│   ├── reset.css
│   └── global.css
├── types/
│   └── index.ts
└── router/
    └── index.ts               # Vue Router configuration
```

Key conventions:
- `<script setup lang="ts">` syntax for all components
- Base components prefixed with `Base` (BaseButton, BaseInput)
- Scoped styles with `<style scoped>` referencing CSS custom properties
- Composables start with `use` and follow the same locality rule as React hooks
- Props defined with `defineProps<{ ... }>()` for type safety

## Next.js (App Router + TypeScript)

```
app/
├── layout.tsx                 # Root layout
├── page.tsx                   # Home page
├── globals.css                # Global styles + tokens
├── dashboard/
│   ├── page.tsx               # Dashboard route
│   ├── loading.tsx            # Loading UI
│   ├── error.tsx              # Error boundary
│   └── components/
│       ├── DashboardChart.tsx
│       └── DashboardStats.tsx
└── api/                       # API routes (if needed)
    └── health/
        └── route.ts
components/
├── ui/                        # Shared UI components
│   ├── Button.tsx
│   └── Modal.tsx
└── layout/
    ├── Header.tsx
    └── Footer.tsx
lib/
├── api.ts                     # Data fetching utilities
├── constants.ts
└── utils.ts
```

Key conventions:
- Colocate route-specific components in the route folder
- Use `loading.tsx` and `error.tsx` for each route segment
- Server Components by default; add `'use client'` only when needed (event handlers, hooks, browser APIs)
- Shared components live outside `app/` directory
- Data fetching happens in Server Components or Route Handlers, not in client components

## Astro

```
src/
├── layouts/
│   └── BaseLayout.astro       # Shared HTML shell
├── pages/
│   ├── index.astro            # Home page
│   └── about.astro
├── components/
│   ├── Header.astro           # Static components
│   ├── Footer.astro
│   └── interactive/           # Client-hydrated components
│       └── SearchDialog.tsx   # React/Vue/Svelte island
├── styles/
│   ├── tokens.css
│   └── global.css
├── lib/
│   └── utils.ts
└── content/                   # Content collections (if CMS-like)
    └── blog/
        └── first-post.md
```

Key conventions:
- Astro components for static content; framework components only for interactive islands
- Explicit `client:` directives (`client:load`, `client:visible`, `client:idle`)
- Content collections for structured content with type-safe schemas
- Zero JS shipped by default; JS only where interactivity is required

## General Rules (All Frameworks)

- Environment variables: `.env.example` with all required keys (no values) checked into version control
- No `any` types in TypeScript — use `unknown` and narrow, or define proper types
- Exports: named exports preferred over default exports (except for page/route components where the framework requires it)
- Error boundaries or error handling at async boundaries (data fetching, lazy imports)
- Tests colocated with the code they test (ComponentName.test.tsx next to ComponentName.tsx) — but only generate tests if the user requests them
