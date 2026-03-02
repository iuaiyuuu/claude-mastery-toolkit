---
name: production-frontend
description: >
  Generate clean, structured, maintainable frontend code following real-world production
  practices — not demo or tutorial output. Trigger when the user asks to build frontend
  code for deployment, production use, or real projects. Trigger when mentioning
  production-ready code, clean architecture, separation of concerns, or code "like a
  real project." Trigger for websites, web apps, dashboards, or any frontend deliverable
  emphasizing quality, maintainability, or scalability. Trigger when avoiding placeholder
  text, lorem ipsum, or demo patterns. Do NOT trigger for quick prototypes or learning
  exercises. Use alongside frontend-design for aesthetics and pwa-builder for offline
  capabilities.
---

# Production Frontend

Generate frontend code that resembles the output of a well-run engineering team's build
pipeline — not the output of a tutorial, demo, or AI chatbot. Every file produced by
this skill should be deployable with minimal modification and maintainable by a
professional team.

## Core Principles

These principles govern every decision. When in tension, resolve in this priority order:
correctness → accessibility → performance → maintainability → brevity.

### 1. No Demo Artifacts

Never include:
- Placeholder text (lorem ipsum, "Your Company Name Here", "TODO: replace this")
- Tutorial or explanatory comments ("This function does X because Y")
- Console.log statements or debugging code
- Hardcoded development URLs, API keys, secrets, or credentials
- Disabled security headers or CORS wildcards
- Commented-out code blocks
- Unused imports, variables, or dead code paths

Comments are allowed only when they explain **why** — not what. Example:
```
// Debounce resize to avoid layout thrashing on older Safari versions
```

### 2. Separation of Concerns

Structure code so that each file or module has one clear responsibility.

**For single-file deliverables** (HTML artifacts, single-page apps):
- Group CSS in a `<style>` block at the top, logic in a `<script>` block at the bottom
- Use CSS custom properties for theming, not inline styles scattered throughout markup
- Keep event handlers as named functions, not anonymous inline expressions

**For multi-file deliverables**:
- Structure follows the conventions of the target framework or build tool
- Styles live in dedicated files (CSS modules, scoped styles, or a design-tokens file)
- Business logic, data fetching, and UI rendering are separate layers
- Configuration lives in dedicated files, not buried in component code

Read `references/file-structure.md` for framework-specific layouts.

### 3. Semantic HTML and Accessibility

Every element serves a structural purpose. This is non-negotiable.

- Use semantic elements: `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<header>`, `<footer>`, `<figure>`, `<figcaption>`, `<details>`, `<summary>`, `<time>`
- Every `<img>` has a meaningful `alt` attribute (empty string `alt=""` only for decorative images, with `role="presentation"`)
- Every interactive element is keyboard-accessible and has a visible focus indicator
- Forms use `<label>` elements associated via `for`/`id`, not placeholder-as-label
- Color contrast meets WCAG 2.1 AA (4.5:1 for normal text, 3:1 for large text)
- Use `aria-` attributes only when native semantics are insufficient — prefer native HTML
- Skip-to-content link for pages with navigation
- Language attribute on `<html>` element
- Page titles are descriptive and unique

### 4. Performance by Default

Every generated file should be fast without requiring a separate optimization pass.

**CSS**:
- Prefer `transform` and `opacity` for animations (compositor-only properties)
- Avoid `@import` chains — use a single stylesheet or CSS custom properties
- Use `will-change` sparingly and only on elements that will animate
- Prefer `clamp()` over media query breakpoints for fluid typography and spacing
- No `!important` unless overriding third-party styles

**JavaScript**:
- Lazy-load non-critical modules with dynamic `import()`
- Use `loading="lazy"` and `decoding="async"` on below-fold images
- Prefer event delegation over per-element listeners for repeated elements
- Use `requestAnimationFrame` for visual updates, not `setTimeout`
- Avoid synchronous layout reads followed by writes (forced reflow)
- Debounce or throttle scroll/resize handlers

**Assets**:
- Specify explicit `width` and `height` on images to prevent layout shift
- Use `<picture>` with `srcset` for responsive images when multiple sizes are provided
- Preload critical fonts with `<link rel="preload" as="font" crossorigin>`
- Prefer system font stacks or `font-display: swap` for web fonts

**Loading strategy**:
- Critical CSS inlined or loaded first
- Non-critical JS deferred (`defer` attribute) or loaded asynchronously
- Preconnect to required origins: `<link rel="preconnect" href="...">`

### 5. Responsive Design

All output is mobile-first and works across viewport sizes without horizontal scrolling.

- Use `min-width` media queries (mobile-first progression)
- Prefer CSS Grid and Flexbox over absolute positioning or float layouts
- Touch targets are at least 44×44 CSS pixels
- Text is readable without zooming (minimum 16px base font size)
- Viewport meta tag is always present: `<meta name="viewport" content="width=device-width, initial-scale=1">`
- Test mental model: content should reflow gracefully from 320px to 2560px

### 6. Security Defaults

- Never output hardcoded secrets, API keys, tokens, or credentials
- Use `rel="noopener noreferrer"` on external links with `target="_blank"`
- Set `Content-Security-Policy` meta tags when generating full HTML pages
- Sanitize any user-provided content before rendering (no raw `innerHTML` with user data)
- Use `SameSite` and `Secure` flags conceptually when discussing cookies
- Environment variables for all configuration that varies between environments
- Avoid `eval()`, `new Function()`, and `document.write()`

## Output Conventions

### Naming

- CSS classes: kebab-case (`user-profile-card`, not `UserProfileCard` or `userProfileCard`)
- CSS custom properties: `--color-primary`, `--spacing-md`, `--font-heading`
- JS/TS variables and functions: camelCase
- JS/TS components (React, Vue, etc.): PascalCase
- Files: kebab-case for plain files, PascalCase for component files when framework convention dictates
- IDs: kebab-case, used sparingly (prefer classes for styling, IDs for anchors and label association)

### CSS Architecture

Use a design-tokens approach for consistency:

```css
:root {
  --color-surface: #ffffff;
  --color-on-surface: #1a1a1a;
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 2rem;
  --spacing-xl: 4rem;
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --font-body: system-ui, -apple-system, sans-serif;
  --font-heading: var(--font-body);
  --transition-fast: 150ms ease;
  --transition-normal: 250ms ease;
}
```

This is a starting point. Adjust tokens to match the project's design system. The
frontend-design skill may override font and color choices — defer to it for aesthetics.

### Meta Tags

Every full HTML page includes:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="description" content="...">
  <title>Descriptive Page Title</title>
</head>
```

Add Open Graph and Twitter Card meta tags when the page is publicly accessible.

### Error States and Edge Cases

Production code handles the unexpected:
- Empty states (no data, no results, no content)
- Loading states (skeleton screens or spinners, not blank pages)
- Error states (network failures, invalid data, permission denied)
- Overflow (long text, many items, large numbers)
- Missing images (fallback or graceful degradation)

Never leave a component that only works with the happy path.

## Framework-Specific Guidance

When the user specifies a framework, read `references/file-structure.md` for the
canonical project layout. When no framework is specified, default to vanilla HTML/CSS/JS
with semantic structure and progressive enhancement.

**React**: Functional components with hooks. No class components. Props validated with
TypeScript or PropTypes. State management at the lowest necessary level. Custom hooks
for reusable logic. Error boundaries around async boundaries.

**Vue**: Composition API preferred. `<script setup>` syntax. Props defined with
TypeScript or `defineProps`. Composables for shared logic.

**Vanilla**: Progressive enhancement. Core content works without JS. Enhance with JS
for interactivity. Use native APIs (`<dialog>`, `<details>`, `popover`) before
reaching for JS solutions.

## Interaction with Other Skills

- **frontend-design**: Governs visual aesthetics (fonts, colors, layout creativity, motion). This skill governs code quality, structure, and deployability. When both apply, frontend-design makes the aesthetic choices and this skill enforces the engineering standards. If frontend-design suggests inline styles or non-semantic markup for visual effect, restructure to achieve the same visual result with clean code.
- **pwa-builder**: Governs offline, installability, and service workers. This skill ensures the underlying code quality of whatever pwa-builder architects.
- **db-architect**: When the frontend connects to a backend, db-architect governs data design. This skill governs how the frontend consumes that data (fetch patterns, error handling, caching strategy).

## Pre-Delivery Checklist

Before presenting any output, verify:

1. No placeholder text, TODO comments, or demo content
2. No hardcoded secrets, credentials, or development-only URLs
3. All images have alt text
4. All interactive elements are keyboard-accessible
5. CSS custom properties used for theming values
6. No unused code, imports, or dead paths
7. Responsive from 320px to 2560px
8. Error, loading, and empty states handled
9. No console.log, debugger, or development artifacts
10. External links have rel="noopener noreferrer"
11. Semantic HTML elements used throughout
12. Performance: images lazy-loaded, scripts deferred, critical CSS prioritized
