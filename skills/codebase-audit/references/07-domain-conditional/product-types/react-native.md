# Product-Type Sub-Reference: React Native (bare workflow)

Audit a bare-workflow React Native codebase. For Expo-managed projects, run the
`expo.md` sub-reference in addition to or instead of this one.

---

## Detection

- `react-native` in `package.json` dependencies
- `ios/` and `android/` directories at repo root
- `metro.config.js` / `metro.config.ts`
- No `expo` config (or Expo config but bare-workflow ejected)

---

## Investigate

### Architecture

- **New Architecture (Fabric + TurboModules).** Enabled in `Podfile.properties.json`
  / `gradle.properties`? React Native 0.74+ has it stable; sticking with the old
  Bridge architecture is now the exception — flag if not enabled.
- **Hermes engine** for JS — should be on by default in 0.70+. Check
  `hermesEnabled=true`.
- **Native module strategy.** Custom native modules — TurboModule (new) vs
  legacy bridge. Mixing is fine but flag old patterns.

### Platform integration

- **Native lifecycle.** Background fetch, notifications, deep linking — handled
  via native APIs or RN bridge?
- **Permissions.** `react-native-permissions` for unified request UX, or
  per-platform?
- **Push.** FCM (Android) + APNs (iOS) — `notifee` / `@react-native-firebase/messaging`
  / `expo-notifications` (post-eject).
- **Storage.** `react-native-mmkv` (modern, fast) vs `AsyncStorage` (legacy,
  slow on large keys). Sensitive tokens go in Keychain/Keystore via
  `react-native-keychain`, not MMKV.
- **Images.** `expo-image` (modern, performant) or `react-native-fast-image`
  vs RN built-in `<Image>`. Built-in is OK for small images, slow for large
  galleries.
- **Lists.** `FlashList` (Shopify) for performant lists vs RN `FlatList`.
  `FlatList` is fine for small/static lists; `FlashList` is meaningfully
  faster for long/dynamic.
- **Gestures.** `react-native-gesture-handler` for complex gestures.
- **Animation.** `react-native-reanimated` (worklet-based, fast) vs `Animated`
  (JS thread, slow for complex). Reanimated 3+ for new work.
- **Navigation.** `@react-navigation/native` is the standard. Flag legacy
  `react-native-navigation` (Wix) if it's holding the project back.
- **Bottom sheets / modals.** `@gorhom/bottom-sheet` for native-feel sheets
  vs JS-only modals.

### Platform features (bridged)

Same idioms as the native sub-references — a React Native app that doesn't
expose iOS/Android platform features is leaving capability on the table:

- **iOS:** Live Activities, Widgets, App Shortcuts, App Clips, Sign in with
  Apple, Passkeys (see `ios-native.md`). RN bridges exist for most.
- **Android:** WorkManager (via `react-native-background-fetch` or custom),
  Widgets, App Shortcuts, Splash Screen API (see `android-native.md`).

If the project ships on both platforms but only implements features on one,
note the asymmetry.

### Compliance

- **Privacy declarations** for both stores (App Store + Play Console).
- **Account deletion in-app** required by both stores.
- **Required Reasons APIs** (iOS) — RN libraries may use these without
  declaring; flag `PrivacyInfo.xcprivacy` audit need.
- **targetSdkVersion** (Android) — Gradle file currency.

### Security

- **Tokens in Keychain/Keystore** via `react-native-keychain` — not
  AsyncStorage, not MMKV.
- **Cert pinning** if backend is single-purpose.
- **JS bundle in production** — Hermes bytecode is harder to inspect than
  raw JS, but neither is secure storage; treat all client code as public.
- **Cleartext traffic** disabled in `network_security_config.xml`
  (Android) and `Info.plist` ATS (iOS).

### Performance

- **App start time.** Cold start measured with `BundleStartupTimeMonitor`
  / native traces. Hermes precompilation, RAM bundles, Inline Requires.
- **JS thread cost.** Heavy work moved to Reanimated worklets, native
  modules, or background threads.
- **List performance.** `FlashList` over `FlatList` for long lists.
- **Image perf.** `expo-image` or `fast-image`; cached, sized correctly.
- **Bundle size.** Hermes bytecode + asset hygiene.

### Build & release

- **CI/CD.** EAS Build (Expo) or Bitrise / Codemagic / Fastlane / Xcode Cloud
  / GitHub Actions with native runners. Manual local builds for production
  release are a process risk.
- **OTA updates.** EAS Update / CodePush — for JS-only changes. Flag if
  the project never ships JS-only fixes between store releases (a high-leverage
  capability often missed).
- **Crash reporting** — Sentry (with `@sentry/react-native`) or Bugsnag with
  source maps uploaded.

---

## Red flags

- Old Architecture (no Fabric / TurboModules) on RN 0.74+ without a stated
  reason.
- Tokens in `AsyncStorage` or `MMKV` instead of Keychain/Keystore.
- Animations on the JS thread for anything complex (use Reanimated worklets).
- `FlatList` rendering 1000+ items.
- No OTA update channel — every JS-only fix waits for a store review.
- Asymmetric platform features (iOS-only widgets, Android-only WorkManager).
- React Native version 2+ majors behind without a stated reason.

---

## Output section

```markdown
### React Native

#### Architecture
- New Architecture (Fabric/TurboModules): enabled? cite Podfile/gradle
- Hermes: enabled?
- RN version: <N> (latest stable: <M>) — status

#### Platform integration
(Storage / images / lists / animation / navigation choices.)

#### Platform features under-utilized
- iOS: (cross-reference ios-native.md)
- Android: (cross-reference android-native.md)
- Asymmetries: (which platform is missing what)

#### Compliance
(stores' requirements)

#### Security
(Keychain usage; cleartext config; pinning; bundle posture)

#### Performance
(Cold start; list perf; image perf; animation thread)

#### Build & release
(CI; OTA channel; crash reporting; source maps)

#### Findings
(table — severity / area / evidence / fix)
```
