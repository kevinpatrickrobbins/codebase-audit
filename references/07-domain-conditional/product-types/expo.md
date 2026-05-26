# Product-Type Sub-Reference: Expo

Audit an Expo (managed or hybrid bare) React Native codebase.

---

## Detection

- `expo` in `package.json` dependencies
- `app.json` or `app.config.{js,ts}` with `expo:` key
- `eas.json` for EAS-managed builds
- Often no `ios/`/`android/` (managed) — if present, this is bare-workflow
  Expo (run `react-native.md` in addition)

---

## Investigate

### SDK currency

- **Expo SDK version.** Major Expo releases happen ~3× per year. Versions
  more than 2 majors behind lose access to current React Native versions and
  modern modules. Check `expo` version vs current (e.g., SDK 50, 51, 52, …).
- **Upgrade lag** — many production Expo projects skip the upgrade pain and
  fall behind on security patches. Flag if SDK is > 1 major behind current.

### Expo modules adoption

Expo's value over bare RN is the curated set of native modules. Common ones
to check the project is *using* (or has a clear reason not to):

- **`expo-router`** — file-based routing for native + web; new since SDK 49.
  Replaces React Navigation imperative setup.
- **`expo-image`** — performant image with caching, blurhash, content-fit.
- **`expo-secure-store`** — Keychain/Keystore-backed storage for secrets.
- **`expo-file-system`** — typed FS access.
- **`expo-notifications`** — unified push interface.
- **`expo-location`** — unified location with proper permission flow.
- **`expo-camera`** — camera + barcode scanning.
- **`expo-av` / `expo-video`** — media.
- **`expo-haptics`** — feedback.
- **`expo-linking` / `expo-router`** — deep links and Universal/App Links.
- **`expo-task-manager` + `expo-background-fetch`** — background work.
- **`expo-updates`** — OTA delivery (free for Hobby; EAS Update for prod).
- **`expo-auth-session`** — OAuth flows.
- **`expo-store-review`** — prompt for review at the right moment.
- **`expo-tracking-transparency`** — iOS ATT prompt (required if using IDFA).

### Build & deploy

- **EAS Build** for native binaries — vs local Xcode/Android Studio builds.
  EAS adds reproducibility and CI; not strictly required but a signal of
  process maturity.
- **EAS Update** for OTA JS deltas — usually the highest-leverage Expo
  capability that gets ignored.
- **Submission.** EAS Submit for store uploads vs manual Transporter / Play
  Console upload.
- **Build profiles.** `eas.json` should have at minimum `development`,
  `preview`, `production` — flag flat configs.

### Platform features (bridged)

Cross-reference `ios-native.md` and `android-native.md` for native-platform
under-utilization. Expo bridges most modern iOS/Android features but not all
(Live Activities require a config plugin or bare workflow). Flag asymmetries.

### Web target

Expo apps can target web via `react-native-web`. Check `app.json` for `"web":
{...}` config.

- If web is a target, check `expo-router` web rendering (SSG / SSR support).
- If web is *not* a target but bundles include `react-native-web`, that's bloat.

### Performance

- **Expo Go vs custom dev clients.** Expo Go (the public sandbox app) is for
  fast iteration but can't load custom native modules. Production apps
  using Expo Go for QA miss real device perf signals.
- **App start time** — Expo SDK upgrades typically improve this; staying
  current pays dividends.
- **Asset loading.** `expo-asset` for preloaded assets vs lazy-loaded.

### Security

- **Secrets** in `expo-secure-store`, not `AsyncStorage` or environment
  variables shipped to the bundle.
- **Environment variables.** Anything in `process.env.EXPO_PUBLIC_*` ships
  to the client — verify nothing sensitive is there.
- **`expo-tracking-transparency`** prompt if IDFA / cross-app tracking is
  used (iOS App Store requirement).

---

## Red flags

- Expo SDK > 2 majors behind current.
- No `eas.json` and no apparent CI for native builds — manual local builds
  in production.
- No `expo-updates` integration despite OTA being free.
- Sensitive data via `EXPO_PUBLIC_*` env vars (these ship to clients).
- `AsyncStorage` for tokens (use `expo-secure-store`).
- Missing `expo-tracking-transparency` while using IDFA / analytics that
  identify users across apps.
- React Navigation imperative setup when `expo-router` would simplify
  significantly.

---

## Output section

```markdown
### Expo

#### SDK currency
- Current: SDK <N>; Latest: SDK <M>
- Upgrade lag: <Strong fit / Good enough / Acceptable but strained / Poor fit>

#### Expo module adoption
- [ ] expo-router
- [ ] expo-image
- [ ] expo-secure-store
- [ ] expo-notifications
- [ ] expo-updates (OTA)
- [ ] expo-tracking-transparency (if IDFA)
- (... checklist of adopted vs missing relevant modules)

#### Build & deploy
- EAS Build: configured?
- EAS Update: configured? channels defined?
- EAS Submit: configured?
- Build profiles in eas.json: development / preview / production

#### Web target
(applicable? bloat?)

#### Security
(secret storage; env var hygiene; ATT prompt)

#### Findings
(table)
```

(Cross-reference `react-native.md`, `ios-native.md`, `android-native.md` for
non-Expo-specific findings.)
