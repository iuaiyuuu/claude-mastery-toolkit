# Caching Strategy Implementations

Full implementation patterns for each caching strategy referenced in the main skill.

## Network First

Try the network. If it fails or times out, serve from cache. Best for content that changes frequently where freshness matters more than speed.

```javascript
async function networkFirst(request, cacheName, timeoutMs = 3000) {
  const cache = await caches.open(cacheName);
  try {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeoutMs);
    const response = await fetch(request, { signal: controller.signal });
    clearTimeout(timeoutId);
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  } catch (err) {
    const cached = await cache.match(request);
    return cached || new Response('Network error', { status: 503 });
  }
}
```

The timeout prevents slow networks from blocking the user indefinitely. 3 seconds is a good default; reduce for real-time data, increase for large payloads.

## Cache First

Serve from cache immediately. Only go to the network if the cache misses. Best for static assets and offline-first apps where the cached version is acceptable.

```javascript
async function cacheFirst(request, cacheName) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const cache = await caches.open(cacheName);
  try {
    const response = await fetch(request);
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  } catch (err) {
    return new Response('Offline', { status: 503 });
  }
}
```

For versioned assets (app.v2.js, styles.abc123.css), cache-first is ideal because the URL changes when the content changes.

## Stale While Revalidate

Serve from cache immediately for speed, then fetch from network in the background to update the cache. The user sees stale data now but fresh data next time.

```javascript
async function staleWhileRevalidate(request, cacheName) {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);

  const fetchPromise = fetch(request).then(response => {
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  }).catch(() => null);

  return cached || await fetchPromise || new Response('Offline', { status: 503 });
}
```

Best for dashboards, analytics, and data that updates periodically but where showing slightly stale data is acceptable.

## Cache First with Background Sync (for form data)

Store user input locally first. Sync to the server when connectivity is available. This pattern is critical for field data entry apps where losing user input is unacceptable.

**In the app (not the service worker):**
```javascript
async function saveAndSync(data) {
  // Always save locally first
  await saveToIndexedDB('pending-submissions', data);

  if (navigator.onLine) {
    try {
      await submitToServer(data);
      await removeFromIndexedDB('pending-submissions', data.id);
    } catch (err) {
      // Will be retried by background sync or manual retry
      console.log('Server unavailable, queued for sync');
    }
  } else if ('serviceWorker' in navigator && 'SyncManager' in window) {
    const reg = await navigator.serviceWorker.ready;
    await reg.sync.register('sync-submissions');
  }
}
```

**In the service worker:**
```javascript
self.addEventListener('sync', event => {
  if (event.tag === 'sync-submissions') {
    event.waitUntil(syncPendingSubmissions());
  }
});

async function syncPendingSubmissions() {
  const pending = await getAllFromIndexedDB('pending-submissions');
  for (const item of pending) {
    try {
      await submitToServer(item);
      await removeFromIndexedDB('pending-submissions', item.id);
    } catch (err) {
      // Will be retried on next sync event
      throw err;
    }
  }
}
```

Note: Background Sync is not supported on iOS/Safari. For cross-platform compatibility, also implement a manual sync button and an online event listener as fallbacks.

## Mixed Strategy (common for real apps)

Most production apps combine strategies. Route requests based on URL pattern:

```javascript
self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // Navigation: network first with offline fallback
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match('/offline.html'))
    );
    return;
  }

  // API data: stale while revalidate (or network first for critical data)
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(staleWhileRevalidate(event.request, 'data-v1'));
    return;
  }

  // Images: cache first (large, expensive to re-fetch)
  if (url.pathname.match(/\.(png|jpg|jpeg|webp|gif|svg)$/)) {
    event.respondWith(cacheFirst(event.request, 'images-v1'));
    return;
  }

  // App shell (JS, CSS, HTML): cache first with versioned URLs
  event.respondWith(
    caches.match(event.request).then(cached => cached || fetch(event.request))
  );
});
```

This is the pattern most apps should start with. Adjust the routing logic based on the specific app requirements.
