# Product-Type Sub-Reference: Browser Extension

Audit a browser extension for Manifest V3 compliance, store policy, security,
and idiomatic fit.

---

## Detection

- `manifest.json` (or `manifest.firefox.json`, multiple manifests for cross-
  browser builds)
- `manifest_version`: 2 or 3
- Keys like `background`, `content_scripts`, `action`, `permissions`
- Build target via Vite/Webpack/Rollup configured for extension output

---

## Investigate

### Manifest version

- **MV3 required** for new submissions to Chrome Web Store (since June 2024
  for new extensions; June 2024 also marked the start of phased MV2
  deprecation). Firefox supports both; Edge follows Chrome.
- **MV2 in production** is a deadline-driven find: must migrate.
- **Cross-browser manifests** — projects often maintain a Chrome-MV3 + a
  Firefox-MV3 (with `browser_specific_settings`) + sometimes a legacy
  Firefox-MV2.

### Background script model

- **MV3:** service worker (event-driven, no persistent state). Cannot use
  `setInterval` reliably (worker terminates), DOM APIs, or persistent
  variables.
- **MV2:** persistent background page (deprecated). Flag if still in use.
- **Common MV3 migration bugs:**
  - `localStorage` in background — not available in service workers; use
    `chrome.storage.local`.
  - `setTimeout` for periodic work — use `chrome.alarms`.
  - DOM parsing in background — move to offscreen documents
    (`chrome.offscreen.createDocument`) or content scripts.
  - Stateful in-memory caches — gone after worker termination; use
    `chrome.storage.session`.

### Permissions

- **Principle of least privilege.** Each requested permission has a
  user-visible install warning. Over-permissioning hurts conversion and
  may fail review.
- **Host permissions.** `<all_urls>` triggers strong warnings — narrow if
  possible.
- **`activeTab`** is preferred over broad host permissions for one-shot
  interactions.
- **Optional permissions** (`optional_permissions` / `optional_host_permissions`)
  for features that aren't core — request at use time via
  `chrome.permissions.request`.

### Content scripts

- **Isolated worlds.** Content scripts run in an isolated world by default —
  cannot access page JS variables. Bridge via `window.postMessage` if
  needed; flag direct use of page globals.
- **CSP of host page.** Some pages have strict CSP that breaks injected
  scripts; declare `world: "MAIN"` if needed (MV3 supports this).
- **`document_idle` / `document_start`** — chosen appropriately for what the
  script does.
- **Match patterns.** `content_scripts.matches` precise enough that the
  script doesn't run on unintended sites.

### Storage

- **`chrome.storage.local`** for local-only state.
- **`chrome.storage.sync`** for synced settings (note 100KB total, 8KB per
  item limits).
- **`chrome.storage.session`** for non-persistent state in MV3.
- **`localStorage` / `sessionStorage`** in background script — broken in MV3.
- **Sensitive data.** Tokens / OAuth refresh — encrypt at rest if possible;
  scope storage so other extensions can't access.

### Messaging

- **`chrome.runtime.sendMessage`** / `onMessage` — typed message envelope
  (action + payload), validated.
- **External messaging.** `externally_connectable` keys — only your
  domains.

### UI surfaces

- **Action popup** (`action.default_popup`) — small UI, no long-running JS.
- **Options page** — full-page settings.
- **DevTools panel** — `devtools_page` + `chrome.devtools.panels`.
- **Side panel** (Chrome MV3) — `side_panel` API, replaces some popup use
  cases for persistent UI.
- **Offscreen documents** — for DOM access in MV3 (audio recording,
  clipboard, parsing).

### Security

- **CSP.** MV3 enforces `script-src 'self'` for extension pages — no remote
  scripts, no `eval`. Verify no `eval`-equivalents (`new Function`,
  `setTimeout(string)`).
- **Remote code.** MV3 forbids fetching and executing remote JS — code must
  ship in the package. Flag any pattern that would break this rule.
- **Token handling.** OAuth via `chrome.identity.launchWebAuthFlow` — never
  embed client secrets.
- **Cross-origin requests.** `host_permissions` declare what origins the
  extension can fetch.

### Distribution & policy

- **Chrome Web Store** review can take days; flag absence of pre-publish
  review automation (linting against MV3 rules).
- **Privacy Practices form** in CWS — declare data usage. Mismatch with
  actual data collection is a takedown risk.
- **Single Purpose policy** — Chrome rejects extensions that bundle unrelated
  features.
- **Use of permissions disclosure** in store listing.
- **Edge Add-ons / Firefox Add-ons / Safari** — separate review processes
  if cross-browser.

---

## Red flags

- `manifest_version: 2` in production (Chrome / Edge).
- `<all_urls>` host permission without clear justification.
- Background script using `setInterval` / `localStorage` (MV3 incompatible).
- Remote code loaded at runtime.
- `eval` / `new Function` / `setTimeout(string, ...)`.
- Tokens stored in `chrome.storage.sync` (limit + sync risk) or in plain
  `localStorage`.
- Content scripts on every page when `activeTab` would suffice.
- Single-purpose policy violation (e.g. password manager + price tracker +
  weather).

---

## Output section

```markdown
### Browser Extension

#### Manifest
- manifest_version: <N>
- Cross-browser: Chrome / Edge / Firefox / Safari status
- MV3 migration status (if relevant)

#### Background model
- Service worker / persistent / event page
- Migration bugs found (MV2 patterns surviving)

#### Permissions
- Declared: <list>
- Justification per high-impact permission
- Optional permissions used: yes/no

#### Content scripts
- match patterns precision
- world (isolated / MAIN)
- run_at choices

#### Storage
- Surfaces in use
- Sensitive data handling

#### UI surfaces
- [ ] popup
- [ ] options page
- [ ] devtools panel
- [ ] side panel (Chrome MV3)
- [ ] offscreen documents

#### Security
- CSP compliance
- Remote code: none confirmed?
- Token handling

#### Distribution
- Store: Chrome / Edge / Firefox / Safari
- Privacy Practices form alignment
- Single Purpose compliance

#### Findings
(table)
```
