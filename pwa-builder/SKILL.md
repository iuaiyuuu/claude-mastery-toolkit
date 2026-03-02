---
name: pwa-builder
description: >
  Build production-ready Progressive Web Apps (PWAs) with offline-first architecture,
  installability, and mobile-optimized performance. Use this skill whenever the user asks
  to build an app that should work offline, be installable on a device, or function as a
  PWA. Also trigger when the user mentions service workers, web app manifests, cache
  strategies, offline-first design, or wants to make an existing web app installable or
  available without network. Trigger for data-centric apps (dashboards, field tools,
  inventory trackers, forms) that must work in low-connectivity or offline environments.
  Do NOT trigger for simple static websites, blogs, or landing pages unless the user
  explicitly requests offline support or installability.
---

# PWA Builder

Build Progressive Web Apps that are installable, offline-capable, and performant on mobile devices. This skill focuses on frontend architecture and does not prescribe a backend, framework, or build tool.

## When This Skill Applies

Use this skill when the user wants an app that works offline, should be installable, mentions service workers or cache strategies, is building a data-centric tool for mobile or low-connectivity use, or wants to convert an existing web app into a PWA.

Do NOT use when the request is a simple website without offline or installability needs, when the user just wants a pretty UI (use the frontend-design skill), or for native app development.

If the user needs both strong visual design AND PWA capabilities, use this skill for architecture and the frontend-design skill for aesthetics.

---

## Mandatory PWA Components

Every PWA must include these four components. Without all four, the app is not a PWA.

### 1. Web App Manifest (manifest.json)

The manifest tells the browser the app is installable and how it should behave when launched.

Example manifest:
```json
{
  "name": "App Full Name",
  "short_name": "App",
  "description": "What this app does",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#1a1a2e",
  "orientation": "any",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-512-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

Key decisions:
- display standalone removes browser chrome. Use minimal-ui if the user needs a back button.
- orientation: portrait for phone-first apps, any for tablets/mixed.
- Always include a 192px and a 512px icon. Include a maskable icon for Android adaptive icons.
- Link manifest in head: <link rel="manifest" href="/manifest.json">
- Add <meta name="theme-color"> matching theme_color.

### 2. Service Worker

The service worker is the engine of offline capability. It intercepts network requests and decides whether to serve from cache or network.

**Development vs. Production**: During development, aggressive caching causes stale content issues. Structure the service worker so caching behavior can be toggled. A simple approach: check a DEV_MODE constant at the top of sw.js, or use a versioned cache name that changes on each build.

Registration (in main JS):
```javascript
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js')
      .then(reg => console.log('SW registered:', reg.scope))
      .catch(err => console.error('SW registration failed:', err));
  });
}
```

Register on load to avoid competing with critical resources during first paint.

Lifecycle - install, activate, fetch:

```javascript
const CACHE_VERSION = 'v1';
const APP_SHELL_CACHE = `app-shell-${CACHE_VERSION}`;
const DATA_CACHE = `data-${CACHE_VERSION}`;

const APP_SHELL_FILES = [
  '/',
  '/index.html',
  '/styles.css',
  '/app.js',
  '/offline.html'
];

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(APP_SHELL_CACHE)
      .then(cache => cache.addAll(APP_SHELL_FILES))
      .then(() => self.skipWaiting())
  );
});

self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys.filter(key => key !== APP_SHELL_CACHE && key !== DATA_CACHE)
            .map(key => caches.delete(key))
      )
    ).then(() => self.clients.claim())
  );
});
```

### 3. Caching Strategy

This is the most important architectural decision in a PWA. Choose based on app type:

| App Type | Strategy | Why |
|---|---|---|
| Content site / blog | Network First | Freshness matters; cache is fallback |
| Dashboard / analytics | Stale While Revalidate | Show cached data fast, update silently |
| Offline-first field tool | Cache First | Must work without network; sync when online |
| Forms / data entry | Cache First + Background Sync | Never lose user input |
| Media-heavy app | Cache First (assets) + Network First (metadata) | Large assets expensive to re-fetch |

Read references/caching-strategies.md for full implementations of each strategy pattern.

The fetch handler routes requests to the appropriate strategy:

```javascript
self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // Navigation requests: network first with offline fallback
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match('/offline.html'))
    );
    return;
  }

  // API requests: strategy depends on app type
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirstWithCache(event.request, DATA_CACHE));
    return;
  }

  // Static assets: cache first
  event.respondWith(
    caches.match(event.request).then(cached => cached || fetch(event.request))
  );
});
```

Navigation requests always try network first and fall back to offline.html to prevent blank screens. Asset requests use cache-first because they are versioned and change infrequently.

### 4. Offline Fallback Page (offline.html)

Shown when the user is offline and the requested page is not cached. Prevents blank white screens.

Requirements:
- Pre-cached during service worker install (part of the app shell)
- Fully self-contained: inline all styles, no external requests
- Communicates offline state and what still works
- Has a retry mechanism

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Offline</title>
  <style>
    body { font-family: system-ui; display: flex; align-items: center;
           justify-content: center; min-height: 100vh; margin: 0;
           background: #f5f5f5; color: #333; text-align: center; }
    .offline-container { max-width: 400px; padding: 2rem; }
    button { padding: 0.75rem 1.5rem; border: none; border-radius: 8px;
             background: #1a1a2e; color: white; cursor: pointer; font-size: 1rem; }
  </style>
</head>
<body>
  <div class="offline-container">
    <h1>You are offline</h1>
    <p>Check your connection and try again.</p>
    <button onclick="window.location.reload()">Retry</button>
  </div>
</body>
</html>
```

---

## Responsive Design for PWAs

PWAs in standalone mode have no browser URL bar and no back button. This changes the design constraints.

**Mobile-first rules:**
- Start with mobile layout, use min-width media queries to scale up
- Touch targets: minimum 44x44px (Apple HIG) or 48x48dp (Material)
- Viewport meta: width=device-width, initial-scale=1.0

**Standalone display mode considerations:**
- No browser back button: provide in-app navigation
- No URL bar: status bar area is part of your app visual space
- Handle safe area insets for notched / dynamic island devices:

```css
body {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}

@media (display-mode: standalone) {
  .browser-only-nav { display: none; }
  .app-header { padding-top: env(safe-area-inset-top); }
}
```

---

## Performance Targets

Default targets. Adjust based on app complexity: a simple checklist should exceed these; a data-heavy dashboard may relax them.

| Metric | Target | Why |
|---|---|---|
| App shell size | < 200KB gzipped | Fast first load, fits in cache budget |
| First Contentful Paint | < 1.5s on 3G | Users see content quickly |
| Time to Interactive | < 3s on 3G | App responds to input quickly |
| Lighthouse PWA score | 100 | Validates all PWA requirements met |
| Offline start | Works | Core app loads without network |

**Achieving small app shells:** inline critical CSS and defer the rest, code-split if using a framework, compress and minify all assets, use system fonts or limit web fonts to one weight of one family, lazy-load non-critical images with loading="lazy".

---

## Local Data and Security

For apps that store data locally (common for offline-first PWAs):

**Storage options:**
- IndexedDB: primary choice for structured data. Use a wrapper like idb for cleaner async API.
- Cache API: for HTTP responses, managed by the service worker.
- localStorage: only for small key-value config. Not for app data (synchronous, 5MB limit).

**Security considerations:**
- Always serve over HTTPS (service workers require it)
- Set appropriate Content-Security-Policy headers
- Never store secrets in localStorage or IndexedDB without encryption
- For sensitive local data, consider the Web Crypto API for encryption at rest
- Clear cached credentials on logout
- Use SameSite cookie attributes and CSRF tokens for API communication

**Data durability, sync robustness, storage management, permissions UX, observability, iOS constraints, and data integrity**: Read `references/production-resilience.md` for detailed guidance on these production-critical topics. Consult this reference whenever building apps that store important user data locally, operate in unreliable network conditions, or must work on iOS.

---

## Optional Capabilities

Include only when the user requirements call for them. Each adds complexity and may trigger permission prompts.

- **Push Notifications**: requires push service + VAPID keys. Only if explicitly needed.
- **Background Sync**: defers actions until online. Great for field data entry apps.
- **Periodic Background Sync**: periodic data refresh. Chrome only; mention the limitation.
- **Share Target**: receive shared content via share_target in manifest.
- **File Handling**: open specific file types via file_handlers in manifest.
- **Badging API**: count badge on app icon for unread items or pending actions.

---

## Testing Checklist

Before considering a PWA complete:

1. **Installability**: Chrome DevTools Application tab shows no manifest errors; install prompt fires
2. **Offline**: DevTools Network set to Offline; app shell loads, offline page shows for uncached routes
3. **Service worker**: Application tab shows active registration, no console errors
4. **Lighthouse**: Run PWA audit, target score 100
5. **iOS/Safari** (critical, many PWA features are limited):
   - No Background Sync — implement manual sync button and online/visibility event fallbacks
   - No push notifications before iOS 16.4, and only for home-screen apps after
   - 50MB storage quota, can be evicted under storage pressure or after 7 days of non-use
   - Service worker terminated after ~48 hours of inactivity — verify registration on each load
   - No install prompt: users must manually Add to Home Screen; guide with UI hints
   - MediaRecorder audio/webm may not be supported; provide audio/mp4 fallback
   - See `references/production-resilience.md` iOS section for full constraints and workarounds
6. **Cache invalidation**: deploy a change, verify new version loads after SW update cycle
7. **Navigation**: in standalone mode, all navigation works without browser back button

---

## Non-Goals

This skill does NOT cover:
- Backend architecture, APIs, or database design
- Native app development or hybrid frameworks (Capacitor, Cordova)
- Build tool configuration (Webpack, Vite, Rollup)
- App store submission (Google Play TWA, Microsoft Store PWA)
- Framework-specific PWA plugins (next-pwa, vite-plugin-pwa, etc.)

This skill produces frontend architecture and PWA infrastructure. Pair with other skills for backend, framework specifics, and visual design.
