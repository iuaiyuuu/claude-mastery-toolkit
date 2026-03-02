# Production Resilience Guide

Guidance for making offline-first PWAs durable and robust in real-world deployments. Read this reference when building apps that store important user data locally, operate in unreliable network conditions, or run on iOS devices.

## Data Durability & Corruption Safety

IndexedDB is the primary local store for offline-first PWAs, but it is not immune to data loss. Abrupt app termination (OS kill, crash, battery death), storage eviction, and partial writes can all corrupt or lose data.

### Atomic Writes with Transaction Boundaries

IndexedDB operations are transactional, but only if you use transactions correctly. The key rule: group related writes into a single transaction so they either all succeed or all fail.

```javascript
// CORRECT: atomic write — both stores update together or neither does
async function saveWithSync(record, syncPayload) {
  const db = await getDB();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(['records', 'pending-sync'], 'readwrite');
    tx.objectStore('records').put(record);
    tx.objectStore('pending-sync').put(syncPayload);
    tx.oncomplete = () => resolve();
    tx.onerror = () => reject(tx.error);
    tx.onabort = () => reject(new Error('Transaction aborted'));
  });
}

// WRONG: two separate transactions — if the second fails, data is inconsistent
async function saveWithSyncBroken(record, syncPayload) {
  const db = await getDB();
  const tx1 = db.transaction('records', 'readwrite');
  tx1.objectStore('records').put(record);
  // If the app crashes here, the record is saved but never queued for sync
  const tx2 = db.transaction('pending-sync', 'readwrite');
  tx2.objectStore('pending-sync').put(syncPayload);
}
```

### Recovery After Abrupt Termination

When the app restarts after a crash or kill, run a consistency check:

```javascript
async function checkConsistency() {
  const db = await getDB();
  const tx = db.transaction(['records', 'pending-sync'], 'readonly');

  const allRecords = await getAllFromStore(tx.objectStore('records'));
  const allPending = await getAllFromStore(tx.objectStore('pending-sync'));

  const pendingIds = new Set(allPending.map(p => p.id));

  // Find records marked unsynced that are missing from the pending queue
  const orphans = allRecords.filter(r => !r.synced && !pendingIds.has(r.id));

  if (orphans.length > 0) {
    // Re-queue orphaned records for sync
    const fixTx = db.transaction('pending-sync', 'readwrite');
    for (const orphan of orphans) {
      fixTx.objectStore('pending-sync').put({ id: orphan.id, transaction: orphan });
    }
    console.log(`Re-queued ${orphans.length} orphaned records for sync`);
  }
}
```

Call this on app startup, before any user interaction.

### Storage Eviction Handling

Browsers can evict IndexedDB data under storage pressure (especially on iOS, which has a 50MB soft limit). Request persistent storage where supported, and handle eviction gracefully:

```javascript
async function requestPersistentStorage() {
  if (navigator.storage && navigator.storage.persist) {
    const granted = await navigator.storage.persist();
    if (!granted) {
      console.warn('Persistent storage not granted — data may be evicted under pressure');
      // Show a non-blocking warning to the user
    }
  }
}

// On app startup, verify data is intact
async function verifyDataIntegrity() {
  try {
    const db = await getDB();
    const tx = db.transaction('records', 'readonly');
    const count = await new Promise((res, rej) => {
      const req = tx.objectStore('records').count();
      req.onsuccess = () => res(req.result);
      req.onerror = () => rej(req.error);
    });
    console.log(`Data store intact: ${count} records`);
  } catch (err) {
    console.error('Data store may be corrupted or evicted:', err);
    // Trigger re-download from server if possible, or alert the user
  }
}
```

---

## Sync Robustness

Background Sync is powerful but needs guardrails for production use.

### Retry with Exponential Backoff

The browser retries Background Sync events automatically, but for manual sync (needed on iOS) or for controlling retry behavior, implement backoff:

```javascript
async function syncWithBackoff(record, attempt = 0) {
  const MAX_ATTEMPTS = 5;
  const BASE_DELAY_MS = 1000;

  if (attempt >= MAX_ATTEMPTS) {
    console.error(`Giving up on ${record.id} after ${MAX_ATTEMPTS} attempts`);
    await markSyncFailed(record.id);
    return;
  }

  try {
    await submitToServer(record);
    await removePending(record.id);
  } catch (err) {
    const delay = BASE_DELAY_MS * Math.pow(2, attempt) + Math.random() * 1000;
    console.log(`Retry ${attempt + 1} for ${record.id} in ${Math.round(delay)}ms`);
    await new Promise(resolve => setTimeout(resolve, delay));
    return syncWithBackoff(record, attempt + 1);
  }
}
```

### Queue Persistence Limits

Unbounded pending queues can exhaust storage, especially with large blobs. Set limits:

```javascript
const MAX_PENDING_ITEMS = 500;
const MAX_PENDING_SIZE_MB = 100;

async function canEnqueue() {
  const db = await getDB();
  const tx = db.transaction('pending-sync', 'readonly');
  const count = await storeCount(tx.objectStore('pending-sync'));

  if (count >= MAX_PENDING_ITEMS) {
    return { allowed: false, reason: `Queue full (${count}/${MAX_PENDING_ITEMS} items)` };
  }

  if (navigator.storage && navigator.storage.estimate) {
    const est = await navigator.storage.estimate();
    const usedMB = est.usage / (1024 * 1024);
    const quotaMB = est.quota / (1024 * 1024);
    if (usedMB > MAX_PENDING_SIZE_MB || usedMB / quotaMB > 0.8) {
      return { allowed: false, reason: `Storage pressure (${usedMB.toFixed(1)}MB used)` };
    }
  }

  return { allowed: true };
}
```

### Duplicate Prevention Across Reinstalls

If a user uninstalls and reinstalls the PWA (or clears data), the pending queue is lost but the server may have already received some records. Use server-side idempotency:

```javascript
// Client: always send a stable idempotency key
const response = await fetch('/api/records', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Idempotency-Key': record.id  // client-generated UUID
  },
  body: JSON.stringify(record)
});

// Server should:
// 1. Check if a record with this idempotency key already exists
// 2. If yes: return 200 (not 409) with the existing record — treat as success
// 3. If no: create the record normally
```

### Conflict Resolution Policies

When the same logical record can be edited offline on multiple devices, choose a policy:

- **Server wins (simplest)**: Server version always takes precedence. Client overwrites local data with server response after sync. Best for most apps.
- **Client wins**: Client version overwrites server. Only use when the client is the sole authority (e.g., single-user local-only with backup sync).
- **Last-write-wins with timestamps**: Compare `updatedAt` timestamps. Simple but can lose data if clocks are skewed.
- **Merge**: Field-level merge where non-conflicting changes are combined. Complex but preserves the most data. Only implement if the app genuinely needs concurrent editing.

For most offline-first PWAs, **server wins** is the right default. The client saves locally for offline use, syncs when online, and accepts the server response as authoritative.

---

## Storage Management

### Quota Awareness

Monitor storage usage and warn users before they hit limits:

```javascript
async function getStorageStatus() {
  if (!navigator.storage || !navigator.storage.estimate) return null;

  const est = await navigator.storage.estimate();
  const usedMB = est.usage / (1024 * 1024);
  const quotaMB = est.quota / (1024 * 1024);
  const pct = (est.usage / est.quota) * 100;

  return {
    usedMB: usedMB.toFixed(1),
    quotaMB: quotaMB.toFixed(0),
    percent: pct.toFixed(1),
    warning: pct > 70,   // show warning at 70%
    critical: pct > 90   // block new saves at 90%
  };
}
```

### Automatic Cleanup

Implement tiered cleanup to reclaim space:

```javascript
async function autoCleanup() {
  const status = await getStorageStatus();
  if (!status || !status.warning) return;

  // Tier 1: Clear old caches
  const cacheKeys = await caches.keys();
  for (const key of cacheKeys) {
    if (key.includes('old') || key.includes('v0')) {
      await caches.delete(key);
    }
  }

  // Tier 2: Remove synced records older than 30 days
  if (status.critical) {
    await clearSyncedOlderThan(30);
  }

  // Tier 3: Compress or remove large blobs from synced records
  if (status.critical) {
    await stripBlobsFromSyncedRecords();
  }
}
```

### Handling Large Blobs Efficiently

Photos and audio files are the biggest storage consumers. Strategies:

- **Compress images before storing**: Use canvas to resize photos to a maximum dimension (e.g., 1200px) before saving to IndexedDB. A 4MB phone photo becomes ~200KB.
- **Stream audio at low bitrate**: Configure MediaRecorder with `audioBitsPerSecond: 32000` for voice notes (speech doesn't need high fidelity).
- **Delete blobs after successful sync**: Once the server confirms receipt, remove the blob from IndexedDB and keep only the metadata.
- **Thumbnail-only local storage**: Store a small thumbnail locally, keep the full image only in the pending-sync queue.

```javascript
async function compressImage(file, maxDimension = 1200, quality = 0.7) {
  return new Promise((resolve) => {
    const img = new Image();
    img.onload = () => {
      const canvas = document.createElement('canvas');
      let { width, height } = img;
      if (width > maxDimension || height > maxDimension) {
        const ratio = Math.min(maxDimension / width, maxDimension / height);
        width *= ratio;
        height *= ratio;
      }
      canvas.width = width;
      canvas.height = height;
      canvas.getContext('2d').drawImage(img, 0, 0, width, height);
      canvas.toBlob(resolve, 'image/jpeg', quality);
    };
    img.src = URL.createObjectURL(file);
  });
}
```

---

## Privacy & Permissions UX

Camera and microphone access require user permission. Handle this responsibly.

### Progressive Permission Requests

Never request permissions on page load. Wait until the user takes an action that requires the permission, and explain why:

```javascript
async function requestCameraWithContext() {
  // Show explanation UI first
  const proceed = await showPermissionExplanation(
    'Camera Access',
    'We need camera access to photograph receipts. Photos are stored on your device and synced when you choose.'
  );

  if (!proceed) return null;

  try {
    const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
    return stream;
  } catch (err) {
    if (err.name === 'NotAllowedError') {
      showPermissionDeniedFallback('camera');
    }
    return null;
  }
}
```

### Degraded Modes When Permissions Are Denied

The app must remain functional even if the user denies camera or microphone access:

```javascript
function showPermissionDeniedFallback(permission) {
  if (permission === 'camera') {
    // Fall back to file picker (no camera, but user can select existing photos)
    document.getElementById('cameraInput').removeAttribute('capture');
    showMessage('Camera access denied. You can still attach photos from your gallery.');
  }
  if (permission === 'microphone') {
    // Hide voice note button, offer text notes instead
    document.getElementById('voiceBtn').style.display = 'none';
    showMessage('Microphone access denied. You can use text notes instead.');
  }
}
```

Check permission state proactively to show the right UI from the start:

```javascript
async function checkPermissions() {
  if (navigator.permissions) {
    const camera = await navigator.permissions.query({ name: 'camera' }).catch(() => null);
    const mic = await navigator.permissions.query({ name: 'microphone' }).catch(() => null);
    return {
      camera: camera ? camera.state : 'unknown',  // 'granted', 'denied', 'prompt'
      microphone: mic ? mic.state : 'unknown'
    };
  }
  return { camera: 'unknown', microphone: 'unknown' };
}
```

---

## Observability

For apps deployed in the field (factories, remote sites, warehouses), debugging issues requires observability built into the app itself.

### Diagnostic Log

Maintain a rolling log in IndexedDB that can be viewed or exported:

```javascript
const MAX_LOG_ENTRIES = 500;

async function logEvent(level, message, data = null) {
  const entry = {
    id: Date.now() + Math.random().toString(36).slice(2, 6),
    timestamp: new Date().toISOString(),
    level,  // 'info', 'warn', 'error'
    message,
    data: data ? JSON.stringify(data) : null
  };

  try {
    const db = await getDB();
    const tx = db.transaction('diagnostics', 'readwrite');
    const store = tx.objectStore('diagnostics');
    store.put(entry);

    // Trim old entries
    const countReq = store.count();
    countReq.onsuccess = () => {
      if (countReq.result > MAX_LOG_ENTRIES) {
        const cursor = store.openCursor();
        let toDelete = countReq.result - MAX_LOG_ENTRIES;
        cursor.onsuccess = (e) => {
          const c = e.target.result;
          if (c && toDelete > 0) { c.delete(); toDelete--; c.continue(); }
        };
      }
    };
  } catch { /* logging should never break the app */ }
}
```

### Debug Mode

Add a hidden debug panel accessible via a gesture (e.g., 5 taps on the header) or a URL parameter:

```javascript
let debugTapCount = 0;
document.querySelector('.app-header').addEventListener('click', () => {
  debugTapCount++;
  if (debugTapCount >= 5) {
    debugTapCount = 0;
    showDebugPanel();
  }
  setTimeout(() => { debugTapCount = 0; }, 3000);
});

async function showDebugPanel() {
  const status = await getStorageStatus();
  const pending = await getPendingCount();
  const logs = await getRecentLogs(50);

  // Render debug info (storage, pending count, SW status, recent logs)
  // Include an "Export Logs" button that downloads diagnostics as JSON
}
```

### Sync Status Inspection

Show sync history so users and support staff can see what synced and what didn't:

```javascript
async function getSyncHistory(limit = 20) {
  // Returns recent sync attempts with status, timestamp, and error if any
  const db = await getDB();
  const tx = db.transaction('sync-history', 'readonly');
  const all = await getAllFromStore(tx.objectStore('sync-history'));
  return all.sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp)).slice(0, limit);
}
```

---

## iOS-Specific Constraints

iOS/Safari has significant PWA limitations that affect architecture decisions. Test on real iOS devices — simulators don't always reproduce these behaviors.

### No Background Sync

Safari does not support the Background Sync API. For iOS, implement these fallbacks:

1. **Manual sync button**: Always provide a visible "Sync Now" button. This is the primary sync mechanism on iOS.
2. **Online event listener**: Trigger sync attempts when the `online` event fires.
3. **Visibility-based sync**: Sync when the app becomes visible (user switches back to it):

```javascript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible' && navigator.onLine) {
    triggerSync();
  }
});
```

### Storage Eviction

iOS aggressively evicts web app data:
- **50MB soft limit** per origin for IndexedDB + Cache API combined
- Data can be evicted after **7 days of non-use** in Safari (not home screen apps)
- Home screen apps get slightly more leeway but are still subject to storage pressure eviction
- `navigator.storage.persist()` is not supported on Safari

Mitigations:
- Keep total storage small by compressing blobs and cleaning up synced data
- On app startup, verify data integrity and warn the user if data appears missing
- Encourage users to add the app to their home screen (longer data retention than Safari tabs)

### Service Worker Lifecycle

Safari terminates service workers after approximately 48 hours of inactivity, and may not restart them reliably. Handle this:

```javascript
// On every page load, verify SW registration
async function ensureSWRegistered() {
  if (!('serviceWorker' in navigator)) return;

  const registration = await navigator.serviceWorker.getRegistration();
  if (!registration || !registration.active) {
    console.warn('SW not active, re-registering');
    await navigator.serviceWorker.register('/sw.js');
  }
}
```

### Install UX Differences

iOS has no `beforeinstallprompt` event. Users must manually use Share > Add to Home Screen. Help them:

```javascript
function isIOS() {
  return /iPad|iPhone|iPod/.test(navigator.userAgent) && !window.MSStream;
}

function isInStandaloneMode() {
  return window.matchMedia('(display-mode: standalone)').matches
      || window.navigator.standalone === true;
}

function showIOSInstallHint() {
  if (isIOS() && !isInStandaloneMode()) {
    // Show a dismissable banner with instructions:
    // "Install this app: tap the Share button, then 'Add to Home Screen'"
    // Include a visual diagram if possible
    // Store dismissal in localStorage so it doesn't re-appear
  }
}
```

### Other iOS Limitations

- **No Web Push before iOS 16.4**, and only for home screen apps after 16.4
- **No Badging API** on iOS
- **No File Handling API** on iOS
- **Limited MediaRecorder support**: audio/webm may not be supported; use audio/mp4 as fallback:

```javascript
function getAudioMimeType() {
  if (MediaRecorder.isTypeSupported('audio/webm')) return 'audio/webm';
  if (MediaRecorder.isTypeSupported('audio/mp4')) return 'audio/mp4';
  if (MediaRecorder.isTypeSupported('audio/aac')) return 'audio/aac';
  return ''; // let browser choose default
}
```

---

## Data Integrity for Offline-First Apps

When users create data offline that will be synced later, the data must be trustworthy. A corrupted or partial record synced to the server can cause downstream problems.

### Local Validation

Validate data at the point of entry, before saving to IndexedDB. Do not rely on server validation alone (the data may sit offline for days before syncing):

```javascript
function validateTransaction(txn) {
  const errors = [];
  if (!txn.amount || txn.amount <= 0) errors.push('Amount must be positive');
  if (!txn.description || txn.description.trim().length === 0) errors.push('Description required');
  if (!txn.date || isNaN(new Date(txn.date).getTime())) errors.push('Invalid date');
  if (!txn.category) errors.push('Category required');
  return { valid: errors.length === 0, errors };
}
```

### Checksums for Blob Integrity

Large blobs (photos, audio) can be silently corrupted during storage or retrieval. Compute a hash at capture time and verify before sync:

```javascript
async function computeHash(blob) {
  const buffer = await blob.arrayBuffer();
  const hashBuffer = await crypto.subtle.digest('SHA-256', buffer);
  return Array.from(new Uint8Array(hashBuffer))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}

// At capture time:
const photo = await compressImage(file);
const hash = await computeHash(photo);
// Store hash alongside blob in IndexedDB

// Before sync:
const storedBlob = record.receiptBlob;
const currentHash = await computeHash(storedBlob);
if (currentHash !== record.receiptHash) {
  console.error('Blob integrity check failed — data may be corrupted');
  await logEvent('error', 'Blob hash mismatch', { recordId: record.id });
  // Skip this blob during sync, or flag for user review
}
```

### Safeguards Against Partial Writes

IndexedDB transactions are atomic for a single transaction, but if the app writes across multiple transactions (or writes followed by a blob store), partial state can result. Defenses:

1. **Single transaction for related data** (covered in Durability section above)
2. **Write status field last**: Mark records as `status: 'complete'` only after all data is written. On startup, discard records with `status: 'partial'`.
3. **Tombstone pattern**: Instead of deleting records, mark them `deleted: true` and filter them out. This prevents ghost references and allows sync of deletions.

```javascript
async function saveRecord(record, blobs) {
  record.status = 'partial'; // start as partial

  const db = await getDB();
  const tx = db.transaction(['records', 'pending-sync'], 'readwrite');

  // Write everything in one transaction
  tx.objectStore('records').put(record);
  tx.objectStore('pending-sync').put({ id: record.id, record, ...blobs });

  await new Promise((resolve, reject) => {
    tx.oncomplete = resolve;
    tx.onerror = reject;
  });

  // Mark as complete in a second transaction
  const tx2 = db.transaction('records', 'readwrite');
  record.status = 'complete';
  tx2.objectStore('records').put(record);
}

// On startup: clean up partial records
async function cleanupPartials() {
  const db = await getDB();
  const tx = db.transaction('records', 'readwrite');
  const store = tx.objectStore('records');
  const all = await getAllFromStore(store);
  for (const record of all) {
    if (record.status === 'partial') {
      store.delete(record.id);
      console.warn(`Removed partial record: ${record.id}`);
    }
  }
}
```
