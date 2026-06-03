# Product-Type Sub-Reference: Electron

Audit an Electron desktop app for security posture, performance, and idiomatic
fit.

---

## Detection

- `electron` and / or `electron-builder` / `electron-forge` in
  `package.json`
- `main.{js,ts}` or similar with `BrowserWindow` import
- `electron-builder.yml` or `forge.config.{js,ts}`

---

## Investigate

### Architecture & isolation

Electron is the most consequential security area. Misconfigured Electron is
a remote-code-execution vector.

- **Context isolation.** `contextIsolation: true` in `BrowserWindow.webPreferences`
  — required for security. Off is a Critical.
- **Node integration in renderer.** `nodeIntegration: false` (default) —
  enabling it merges renderer + Node, exposing the OS to web content.
- **Sandbox.** `sandbox: true` for renderers when feasible.
- **Preload script + contextBridge.** All Node-flavored capabilities bridged
  via `contextBridge.exposeInMainWorld` with explicit, narrow surfaces — not
  `window.electron = require('electron')`.
- **CSP.** Content-Security-Policy header / meta in renderer pages — even for
  local content. Restricts what the renderer can fetch / eval.
- **Remote module.** `enableRemoteModule: false` — module deprecated and
  removed; flag any import of `@electron/remote`.
- **Web security.** `webSecurity: true` — disabling allows file:// access
  to do unsafe things.
- **Permissions handler.** `session.setPermissionRequestHandler` for media,
  geolocation, etc.

### IPC

- **IPC surface.** `ipcMain.handle` / `ipcRenderer.invoke` — clearly defined,
  validated channels. Not a wildcard channel that accepts arbitrary
  messages.
- **Input validation.** Treat IPC like an HTTP API — validate every payload,
  no `eval` of message content, no `require(messageContent)`.
- **Channel naming.** Versioned / namespaced channels rather than `'do-thing'`
  spread across the app.

### Security signing & distribution

- **Code signing.** Both macOS (Developer ID) and Windows (EV/OV cert).
  Unsigned builds trigger Gatekeeper / SmartScreen warnings.
- **Notarization.** macOS notarization step — if absent, downloads fail
  on modern macOS without intervention.
- **Auto-updater.** `electron-updater` (Squirrel) or custom — should pull
  signed delta updates.
- **Signed update channel.** Verify updater signature checks; an unsigned
  update channel is a malware delivery vector.

### Update strategy

- **Update frequency.** Stale Electron is a continuous CVE risk — Electron
  ships major versions every ~8 weeks; ride the line within ~2 majors.
- **Auto-update enabled?** Or do users manually re-download?
- **Phased rollout** for updates.

### Performance

- **App start time.** Many Electron apps take 3–5s to start; targets are
  < 2s for production. Hunt for:
  - Synchronous IPC on startup
  - Heavy bundling shipped to main process
  - Many windows / preload scripts
- **Memory.** Each `BrowserWindow` is a Chromium process. Three is fine; thirty
  is a problem.
- **Renderer JS bundle** — same concerns as a web app. Tree-shake, code split.
- **`will-change`** on always-animating elements — eats GPU memory.

### Native integrations under-utilized

- **System tray** for ambient apps.
- **Native menus** (`Menu.buildFromTemplate`) — Electron defaults are usually
  wrong on macOS without customization.
- **Shortcuts** (`globalShortcut`) for quick capture / activation features.
- **Notifications** (`new Notification(...)`) using OS-native.
- **Dock badges / progress bars** (`app.dock.setBadge`).
- **Touch Bar** (Macs that still have it; minor).
- **Auto-launch on login** (`app.setLoginItemSettings`).
- **Native dialogs** (`dialog.showOpenDialog`) — vs DOM-built modals for OS-
  level interactions.
- **Power management** (`powerSaveBlocker`) for media or sync apps that
  need to suppress sleep.
- **Native theme.** Respect `nativeTheme` for dark mode. Apps that ignore
  OS theme look out of place.

### Storage & data

- **`electron-store`** or `app.getPath('userData')` for app data — not
  alongside the binary.
- **Sensitive data via OS keychain** (`keytar` / native API) — not plaintext
  in user-data directory.

### Web vs Native parity

If the app has a web or mobile counterpart, check feature parity. Native
features that don't exist on web (file watchers, OS notifications, system
integrations) should be the value-add justifying Electron in the first place
— flag if the desktop app is a 1:1 port of the web app with no native
differentiation (the calculus may favor wrapping the web app rather than
maintaining the Electron app).

---

## Red flags

- `nodeIntegration: true` in any production `BrowserWindow`.
- `contextIsolation: false`.
- `webSecurity: false`.
- `@electron/remote` imported.
- Wildcard or unvalidated IPC handlers.
- No code signing / notarization.
- Electron version ≥ 4 majors behind current.
- App start > 5s on a typical machine.
- Native menus not customized on macOS.
- Tokens stored in plaintext in `userData`.
- Auto-updater disabled and no manual update prompt.

---

## Output section

```markdown
### Electron

#### Architecture & isolation
- contextIsolation: <on/off>
- nodeIntegration: <off/on>
- sandbox: <on/off>
- contextBridge usage: (cite preload file)
- CSP: (cite header / meta)

#### IPC
(Surface; validation; naming)

#### Security & distribution
- Code signing: macOS / Windows status
- Notarization (macOS): yes / no
- Auto-updater: configured? signed channel?

#### Update strategy
- Electron version: <N> (current: <M>)
- Update frequency / channel

#### Performance
- App start time
- Memory profile / window count
- Renderer bundle status

#### Native integrations under-utilized
- [ ] System tray
- [ ] Native menus
- [ ] Global shortcuts
- [ ] OS notifications
- [ ] Dock badges / progress
- [ ] Auto-launch
- [ ] Native dialogs
- [ ] Native theme respect

#### Storage
(userData; keychain for secrets)

#### Web/native parity
(differentiation justifying desktop app)

#### Findings
(table)
```
