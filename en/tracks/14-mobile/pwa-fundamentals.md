# PWA Fundamentals

## Overview

Progressive Web Apps (PWAs) bridge the gap between websites and native apps. They are regular web applications enhanced with modern APIs that enable installation on the home screen, offline functionality, background sync, and push notifications — without requiring an app store. A PWA is not a separate codebase; it is your existing web app with a service worker, a web app manifest, and an HTTPS deployment.

PWAs are the right choice when you need broad reach without the friction of app store review, when your team is pure web, or when your content-first app can run fully in the browser. They are not the right choice when you need deep hardware integration (Bluetooth, NFC, camera RAW access) or when platform-specific UI consistency is non-negotiable.

---

## Prerequisites

- Solid understanding of JavaScript and the browser event loop
- Familiarity with the Fetch API and Promises
- Basic understanding of HTTP caching headers
- Track 02 — Frontend (React or equivalent framework)

---

## Core Concepts

### The Service Worker

A service worker is a JavaScript file that the browser runs in a background thread, separate from the main page. It acts as a programmable network proxy: every fetch request the page makes can be intercepted, modified, responded to from cache, or forwarded to the network.

Key characteristics:
- Runs in a separate worker context (no DOM access)
- Communicates with pages via `postMessage`
- Persists between page loads and browser restarts
- Has its own lifecycle: install → activate → fetch

**Service Worker Lifecycle**

```
Browser parses SW script
       ↓
  install event fires
  (cache static assets here)
       ↓
  waiting (old SW still active)
       ↓
  activate event fires
  (clean up old caches here)
       ↓
  fetch events intercepted
```

**Registration**

```javascript
// main.js — register from the page
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const registration = await navigator.serviceWorker.register('/sw.js', {
        scope: '/', // controls which URLs the SW manages
      });
      console.log('SW registered:', registration.scope);
    } catch (error) {
      console.error('SW registration failed:', error);
    }
  });
}
```

**Minimal service worker**

```javascript
// sw.js
const CACHE_NAME = 'my-app-v1';
const STATIC_ASSETS = ['/', '/index.html', '/app.js', '/styles.css'];

self.addEventListener('install', (event) => {
  // Cache static assets during install
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(STATIC_ASSETS))
  );
  // Skip waiting so new SW activates immediately
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  // Delete old caches
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((key) => key !== CACHE_NAME)
          .map((key) => caches.delete(key))
      )
    )
  );
  // Take control of existing clients
  self.clients.claim();
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => cached ?? fetch(event.request))
  );
});
```

### Caching Strategies

The caching strategy you choose depends on the type of content.

**Cache First (static assets)**
Serve from cache if available; fall back to network. Best for versioned assets (JS bundles with content hashes).

```javascript
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  const cache = await caches.open(CACHE_NAME);
  cache.put(request, response.clone()); // clone because response is a stream
  return response;
}
```

**Network First (API responses)**
Try network; fall back to cache if offline. Best for frequently updated data.

```javascript
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
    return response;
  } catch {
    return caches.match(request);
  }
}
```

**Stale While Revalidate (feed data)**
Return cached version immediately; fetch a fresh version in the background for next time.

```javascript
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  const fetchPromise = fetch(request).then((response) => {
    cache.put(request, response.clone());
    return response;
  });

  return cached ?? fetchPromise;
}
```

### Web App Manifest

The manifest is a JSON file that tells the browser how to present your app when installed.

```json
{
  "name": "My Awesome App",
  "short_name": "AwesomeApp",
  "description": "A progressive web application",
  "start_url": "/?utm_source=pwa",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#6366f1",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide"
    }
  ],
  "categories": ["productivity", "utilities"],
  "shortcuts": [
    {
      "name": "New Entry",
      "short_name": "New",
      "description": "Create a new entry",
      "url": "/new",
      "icons": [{ "src": "/icons/new-96.png", "sizes": "96x96" }]
    }
  ]
}
```

Link the manifest from your HTML:

```html
<link rel="manifest" href="/manifest.json" />
<!-- iOS requires separate meta tags -->
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="default" />
<meta name="apple-mobile-web-app-title" content="AwesomeApp" />
<link rel="apple-touch-icon" href="/icons/icon-192.png" />
```

**`display` values:**
- `standalone` — looks like a native app (no browser chrome)
- `minimal-ui` — minimal browser controls shown
- `fullscreen` — hides all browser UI (games)
- `browser` — opens in the regular browser tab

---

## Hands-On Examples

### Offline Page Fallback

```javascript
// sw.js
const OFFLINE_URL = '/offline.html';

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) =>
      cache.addAll([OFFLINE_URL, ...STATIC_ASSETS])
    )
  );
});

self.addEventListener('fetch', (event) => {
  // Only intercept navigation requests for offline fallback
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match(OFFLINE_URL))
    );
    return;
  }
  // Other requests use cache-first
  event.respondWith(cacheFirst(event.request));
});
```

### Install Prompt

Browsers fire a `beforeinstallprompt` event when they decide your app is installable. You must save this event and trigger it on user gesture.

```javascript
// installPrompt.js
let deferredPrompt = null;

window.addEventListener('beforeinstallprompt', (event) => {
  event.preventDefault(); // prevent automatic mini-infobar
  deferredPrompt = event;
  showInstallButton(); // show your custom install button
});

async function handleInstallClick() {
  if (!deferredPrompt) return;

  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice;

  if (outcome === 'accepted') {
    console.log('User accepted the install prompt');
  } else {
    console.log('User dismissed the install prompt');
  }

  deferredPrompt = null;
  hideInstallButton();
}

// React component example
function InstallButton() {
  const [canInstall, setCanInstall] = useState(false);
  const promptRef = useRef(null);

  useEffect(() => {
    const handler = (e) => {
      e.preventDefault();
      promptRef.current = e;
      setCanInstall(true);
    };
    window.addEventListener('beforeinstallprompt', handler);
    return () => window.removeEventListener('beforeinstallprompt', handler);
  }, []);

  const install = async () => {
    if (!promptRef.current) return;
    promptRef.current.prompt();
    const { outcome } = await promptRef.current.userChoice;
    if (outcome === 'accepted') setCanInstall(false);
    promptRef.current = null;
  };

  if (!canInstall) return null;
  return <button onClick={install}>Install App</button>;
}
```

### Push Notifications

Push notifications require three parties: your server, a push service (run by browser vendors), and the browser.

```javascript
// 1. Request permission
async function requestNotificationPermission() {
  const permission = await Notification.requestPermission();
  if (permission !== 'granted') {
    throw new Error('Notification permission denied');
  }
}

// 2. Subscribe the browser to push
async function subscribeToPush() {
  const registration = await navigator.serviceWorker.ready;

  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true, // required: must show notification for every push
    applicationServerKey: urlBase64ToUint8Array(process.env.VAPID_PUBLIC_KEY),
  });

  // 3. Send subscription to your server
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  });

  return subscription;
}

// Helper: convert VAPID key
function urlBase64ToUint8Array(base64String) {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
  const rawData = atob(base64);
  return Uint8Array.from([...rawData].map((char) => char.charCodeAt(0)));
}

// 4. Handle push in service worker
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {};
  event.waitUntil(
    self.registration.showNotification(data.title ?? 'Notification', {
      body: data.body,
      icon: '/icons/icon-192.png',
      badge: '/icons/badge-96.png',
      data: { url: data.url },
      actions: [
        { action: 'open', title: 'Open' },
        { action: 'dismiss', title: 'Dismiss' },
      ],
    })
  );
});

// 5. Handle notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'dismiss') return;

  const url = event.notification.data?.url ?? '/';
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then((windowClients) => {
      const existing = windowClients.find((c) => c.url === url && 'focus' in c);
      if (existing) return existing.focus();
      return clients.openWindow(url);
    })
  );
});
```

**Server-side push (Node.js with web-push)**

```javascript
import webpush from 'web-push';

webpush.setVapidDetails(
  'mailto:admin@example.com',
  process.env.VAPID_PUBLIC_KEY,
  process.env.VAPID_PRIVATE_KEY
);

// Generate VAPID keys once: webpush.generateVAPIDKeys()

async function sendPushNotification(subscription, payload) {
  try {
    await webpush.sendNotification(subscription, JSON.stringify(payload));
  } catch (error) {
    if (error.statusCode === 410) {
      // Subscription expired — remove from database
      await db.pushSubscriptions.delete(subscription.endpoint);
    }
    throw error;
  }
}
```

---

## Common Patterns & Best Practices

### Background Sync

Queue failed requests and replay them when connectivity is restored.

```javascript
// In your fetch code
async function saveData(data) {
  try {
    await fetch('/api/save', { method: 'POST', body: JSON.stringify(data) });
  } catch {
    // Store in IndexedDB for later
    await idb.put('sync-queue', { id: crypto.randomUUID(), data, timestamp: Date.now() });

    const registration = await navigator.serviceWorker.ready;
    await registration.sync.register('sync-saves');
  }
}

// In service worker
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-saves') {
    event.waitUntil(syncPendingSaves());
  }
});

async function syncPendingSaves() {
  const queue = await idb.getAll('sync-queue');
  for (const item of queue) {
    await fetch('/api/save', { method: 'POST', body: JSON.stringify(item.data) });
    await idb.delete('sync-queue', item.id);
  }
}
```

### Update Detection

```javascript
// Notify users when a new version is available
function registerSW() {
  navigator.serviceWorker.register('/sw.js').then((registration) => {
    registration.addEventListener('updatefound', () => {
      const newWorker = registration.installing;

      newWorker.addEventListener('statechange', () => {
        if (
          newWorker.state === 'installed' &&
          navigator.serviceWorker.controller
        ) {
          // New version is ready — show update banner
          showUpdateBanner(() => {
            newWorker.postMessage({ type: 'SKIP_WAITING' });
            window.location.reload();
          });
        }
      });
    });
  });
}

// In sw.js
self.addEventListener('message', (event) => {
  if (event.data?.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});
```

### Workbox

Workbox is Google's library for service worker patterns. It eliminates boilerplate and handles edge cases.

```javascript
// vite.config.ts with vite-plugin-pwa
import { defineConfig } from 'vite';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\.example\.com\/.*/i,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              expiration: { maxEntries: 50, maxAgeSeconds: 300 },
              networkTimeoutSeconds: 10,
            },
          },
          {
            urlPattern: /^https:\/\/fonts\.googleapis\.com\/.*/i,
            handler: 'StaleWhileRevalidate',
            options: { cacheName: 'google-fonts-stylesheets' },
          },
        ],
      },
      manifest: {
        name: 'My App',
        short_name: 'App',
        theme_color: '#6366f1',
        icons: [
          { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
        ],
      },
    }),
  ],
});
```

---

## Anti-Patterns to Avoid

**Caching everything without a strategy.** Stale data silently served from cache causes mysterious bugs. Each resource type needs an explicit strategy chosen based on freshness requirements.

**Requesting notification permission on page load.** Browsers block sites that do this. Always request permission in response to a user action (a button click).

**Forgetting to clone responses before caching.** `Response` objects are single-use streams. Call `response.clone()` before putting in cache and returning.

**Missing the `waitUntil` call.** Service worker event handlers are terminated as soon as the main function returns. Wrap async work in `event.waitUntil()` to extend the lifetime.

**Deploying without HTTPS.** Service workers require HTTPS (localhost is the only exception). Plan your deployment accordingly.

**Calling `skipWaiting()` unconditionally in the install handler.** This can break clients mid-session when they receive broken partial asset sets. Gate it on a user-acknowledged update.

**Ignoring iOS limitations.** Safari has significant PWA gaps: no push notifications until iOS 16.4, limited storage quota, no background sync. Always test on actual iOS devices and document what features are unavailable.

---

## Debugging & Troubleshooting

### Chrome DevTools

- **Application tab > Service Workers** — see registration status, lifecycle state, inspect/update
- **Application tab > Cache Storage** — browse cached responses
- **Application tab > Manifest** — validate manifest and installability
- **Network tab > Offline checkbox** — simulate offline mode
- `chrome://serviceworker-internals` — low-level view of all SWs

### Common Issues

**SW not updating:** Browser caches sw.js. Use `updateViaCache: 'none'` in the register call or add a cache-busting query param during development.

```javascript
navigator.serviceWorker.register('/sw.js?v=' + BUILD_ID, {
  updateViaCache: 'none',
});
```

**Opaque responses polluting cache:** Cross-origin requests without CORS return opaque responses (status 0). Do not cache them — one opaque response can fill your entire cache quota. Add a check:

```javascript
if (response.type === 'opaque') return response; // don't cache
```

**Install not triggering on iOS:** iOS requires: HTTPS, valid manifest with `start_url`, at least one 192x192 icon, `display: standalone`. Missing any of these silently prevents the Add to Home Screen option.

**Push subscription lost after browser update:** Always check if a subscription still exists before assuming it is valid. Re-subscribe silently if needed.

---

## Real-World Scenarios

### Offline-First Note Taking App

Architecture:
1. All writes go to IndexedDB first (synchronous from user's perspective)
2. A background sync queue holds pending server writes
3. SW intercepts API calls: GET uses stale-while-revalidate, POST/PUT/DELETE use network-first with background sync fallback
4. On reconnect, sync queue replays in order with conflict resolution (last-write-wins or CRDT)

### PWA for a News Site

Architecture:
1. Shell (HTML, CSS, JS) is cached on first install — loads instantly
2. Article HTML pages are cached on visit (cache-first with 24h expiration)
3. API feed endpoint uses network-first with 5-minute cache
4. Images use stale-while-revalidate with a 100-entry LRU cache
5. Push notifications for breaking news; user configures topics in settings

---

## Further Reading

- [web.dev — Progressive Web Apps](https://web.dev/progressive-web-apps/)
- [MDN — Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [Workbox documentation](https://developer.chrome.com/docs/workbox/)
- [What PWA Can Do Today](https://whatpwacando.today/) — current browser capability matrix
- [vite-plugin-pwa](https://vite-pwa-org.netlify.app/)

---

## Summary

A PWA is built on three pillars: a service worker (network proxy + cache), a web manifest (installability metadata), and HTTPS. Caching strategies (cache-first, network-first, stale-while-revalidate) must be chosen per resource type. Push notifications require VAPID keys, a push subscription stored server-side, and a `push` event handler in the SW. Workbox removes boilerplate and handles edge cases like opaque response caching. Test on real iOS hardware early — Safari's PWA support lags Chrome significantly.
