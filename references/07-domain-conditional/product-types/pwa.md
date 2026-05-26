# Product-Type Sub-Reference: PWA (Progressive Web App)

Audit a Progressive Web App — installable web app with offline capability and
native-like surfaces.

---

## Detection

- `manifest.webmanifest` or `manifest.json` declaring `display`, `start_url`,
  `icons`, etc. (web app manifest)
- Service worker registration in entry point
  (`navigator.serviceWorker.register('/sw.js')`)
- `next-pwa` / `vite-plugin-pwa` / `workbox-*` packages
- "Add to Home Screen" promotion code

If the project has just a `manifest.json` for an extension (not a PWA), the
`manifest_version` key disambiguates.

---

## Investigate

### Web app manifest

- **Required keys.** `name`, `short_name`, `start_url`, `display`, `icons`,
  `theme_color`, `background_color`. Missing any of these prevents
  installation.
- **`display` value.** `standalone` or `fullscreen` for installable; `browser`
  is not a real PWA experience.
- **`start_url`.** Often forgotten; should match the launch route.
- **Icons.** At minimum 192×192 and 512×512 PNG; ideally maskable variants
  (`purpose: "maskable"` for Android adaptive icons).
- **`scope`** — PWA scope (URL prefix); navigation outside opens the system
  browser.
- **Categories, screenshots, description** — for store-style listings on
  Android (Microsoft Store, Samsung Galaxy Store, Chrome PWA discovery).
- **Shortcuts** (`shortcuts` array) — for app-icon long-press menu.

### Service worker

- **Registration timing.** After page load, not blocking initial render.
- **Fetch strategy.** Workbox patterns are the standard:
  - `CacheFirst` — for hashed static assets (CSS, JS, fonts, images with
    versioned URLs).
  - `StaleWhileRevalidate` — for API responses where freshness is preferred
    but stale is acceptable.
  - `NetworkFirst` — for HTML / JSON where freshness matters more.
  - `NetworkOnly` — for non-cacheable (mutations).
- **Cache versioning.** A new SW must invalidate old caches; common bug is
  unbounded cache growth.
- **Update flow.** "New version available" prompts — `skipWaiting` /
  `clients.claim` strategy. Silent updates can trap users on old code.
- **Offline fallback.** A meaningful offline page or shell — not a generic
  browser error.

### Installability

- **Install prompt.** Listen for `beforeinstallprompt`, defer, show at the
  right moment (not on first paint).
- **Lighthouse PWA audit** results — installable, splash, themed.
- **iOS limitations.** iOS PWAs lose state on app-switch in older Safari;
  push notifications via Web Push only since iOS 16.4 (and only for
  installed PWAs). Flag iOS-specific gaps in the report.
- **Android.** Trusted Web Activity (TWA) / Bubblewrap if the project ships
  to Play Store as a PWA.

### Push & background

- **Web Push.** Service worker `push` handler + VAPID keys for backend.
- **Background sync.** `sync` and `periodicSync` events for queued mutations
  / refresh.
- **Background fetch.** For longer downloads.
- **Notifications API** — permission flow respectful (not on first load).

### Native-like UI

- **Theme color** matches OS chrome.
- **Splash screen** — auto-generated from manifest icons + background_color.
- **Status bar.** `apple-mobile-web-app-status-bar-style` for iOS.
- **Apple touch icons** (`<link rel="apple-touch-icon">`) — separate from
  manifest icons because of historical Safari behavior.
- **Display modes media queries** — `@media (display-mode: standalone)` for
  installed-vs-browser styling.
- **App shortcuts** in manifest.

### Performance

Most of Module 10 (Speed Audit) applies; specific to PWAs:

- **Critical asset preload** in SW.
- **App shell pattern** — minimal initial HTML cached aggressively.
- **Asset budget.** SW pre-caching with no budget can balloon storage.

### Storage

- **IndexedDB** for structured data — not localStorage for large/structured.
- **Cache API** for HTTP-shaped responses.
- **OPFS (Origin Private File System)** for file-like storage where
  appropriate.
- **Persistent storage.** `navigator.storage.persist()` to avoid eviction.

### Security

- **HTTPS required** — `localhost` is the only exception.
- **Service worker scope** — narrow as appropriate.
- **Origin trust** — service workers can intercept all requests in scope;
  treat the SW as a security boundary.

---

## Red flags

- Service worker that caches everything indefinitely with no version
  invalidation.
- `display: browser` (not a real PWA experience).
- Missing maskable icons (Android adaptive icon UX is poor without).
- Install prompt fired on first paint (annoying, low conversion).
- Push notification permission requested on first load.
- iOS gaps unacknowledged in projects targeting iOS users.
- Service worker logging large payloads (performance + privacy).
- No "new version available" handling — users can be stuck on stale code
  for weeks.

---

## Output section

```markdown
### PWA

#### Manifest
- Required keys present: <list of missing>
- display: <value>
- Icons (sizes, maskable): <status>
- Shortcuts: <count>

#### Service worker
- Registration timing
- Caching strategy by resource type
- Cache versioning / invalidation
- Update flow (skipWaiting / claim / prompt)
- Offline fallback

#### Installability
- Lighthouse PWA: <pass/fail>
- Install prompt UX
- iOS gaps
- TWA (Android) status

#### Push & background
- Web Push: configured?
- Background sync: configured?
- Notification permission UX

#### Native-like UI
- Theme color / splash / Apple touch icons
- display-mode media queries

#### Storage
- IndexedDB / Cache API / OPFS choices
- Persistent storage requested

#### Security
- HTTPS / scope / SW boundary discipline

#### Findings
(table)
```
