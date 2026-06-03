# Product-Type Sub-Reference: Flutter

Audit a Flutter codebase (Dart, targeting iOS / Android / web / desktop).

---

## Detection

- `pubspec.yaml` referencing `flutter` SDK
- `*.dart` source under `lib/`
- `ios/`, `android/`, optionally `web/`, `windows/`, `macos/`, `linux/`

---

## Investigate

### Flutter & Dart version

- **Flutter SDK version.** Flutter ships frequent minor releases — being more
  than ~6 months behind starts to bite (compatibility with new packages,
  store requirements). Check `flutter --version` evidence in CI configs or
  `.fvmrc` / `fvm_config.json`.
- **Dart version** — implied by Flutter SDK; flag if pinned to an old SDK
  preventing pkg upgrades.
- **Null safety.** Mandatory since Dart 2.12 — verify no opt-out files.

### State management

Flutter has no canonical state solution; the project's choice is a defining
architectural decision.

- **Riverpod** (modern, type-safe, recommended for new projects)
- **Provider** (legacy from same author; still common; transition to Riverpod
  often beneficial)
- **Bloc / flutter_bloc** (event-driven; popular at scale)
- **GetX** (controversial; mixes routing/DI/state; flag if used unintentionally)
- **MobX**, **Redux** — less common
- **`setState` only** — fine for very small apps, struggles at scale.

Flag if multiple solutions coexist without clear separation, or if a project
that has outgrown `setState` hasn't adopted one.

### Routing

- **`go_router`** (declarative, recommended for new) vs imperative
  `Navigator.push`. Deep linking and web URLs benefit from declarative.
- **`auto_route`** (code-gen alternative).

### Platform integration

- **Platform channels** for native APIs not covered by plugins.
- **FFI** for native libraries (rare, usually not needed).
- **Plugin currency.** Major plugins (camera, location, biometric, sqflite,
  shared_preferences, secure_storage) — pinned to old majors miss platform
  updates.

### Cross-reference

For platform-specific concerns, run the sibling sub-references:

- **iOS:** see `ios-native.md` (Live Activities, App Shortcuts, etc. — Flutter
  has limited bridging here; flag opportunities lost).
- **Android:** see `android-native.md` (WorkManager, Widgets — same).

### Performance

- **Build modes.** Profile mode for perf testing — debug mode is **not** a
  perf signal. Verify any profiling cited used profile or release mode.
- **Widget tree depth.** Deeply nested widgets without `const` constructors
  are cheap in Dart but rebuild more than necessary.
- **`const` constructors.** Use widely; the analyzer can flag missing ones
  with `prefer_const_constructors` lint.
- **Image caching.** `cached_network_image` for remote assets vs raw
  `Image.network` (uncached).
- **List performance.** `ListView.builder` over `ListView` with children list.
- **`RepaintBoundary`** for complex animated widgets.
- **Skia / Impeller renderer.** Impeller is now default on iOS and stable on
  Android — flag if explicitly opted out.
- **Tree shaking.** Release builds with `--split-debug-info` and proper
  obfuscation.

### App size

- **AAB (Android App Bundle)** for store delivery — Play Console requirement.
- **App Store thinning** — `--obfuscate` for release.
- **Flutter assets** trimmed for actual usage; no bundled fonts/images
  unused.

### Security

- **`flutter_secure_storage`** for tokens — not `shared_preferences`.
- **HTTPS only** — Android `cleartextTrafficPermitted="true"` disabled in
  prod.
- **Cert pinning** via `dio` `CertificatePinner` or platform-level config.
- **`flutter_jailbreak_detection`** if threat model warrants.

### Compliance

- **Privacy declarations** in both store consoles match actual data
  collection (analytics SDKs, ads, etc.).
- **iOS Required Reasons APIs** — Flutter plugins may use `UserDefaults`,
  file timestamps, etc. without declaring. Generate / verify
  `PrivacyInfo.xcprivacy`.
- **Android targetSdkVersion** currency.
- **Account deletion in-app.**
- **iOS ATT** if cross-app tracking via IDFA — uses `app_tracking_transparency`
  package.

### Web target

If the project enables web:

- Flutter Web outputs are heavy; flag if web is a target but no clear
  justification (often a regular React app fits better).
- SEO is poor on Flutter Web (canvas rendering); not appropriate for
  marketing sites or content sites.
- Check renderer (`html` vs `canvaskit`) — `html` is smaller, `canvaskit`
  matches mobile rendering exactly.

### Build & deploy

- **CI/CD.** Codemagic, Bitrise, GitHub Actions with Flutter setup, or
  manual local builds.
- **Crash reporting.** Sentry / Firebase Crashlytics — symbol files
  uploaded.

---

## Red flags

- Flutter SDK > 6 months behind current.
- Multiple state-management solutions tangled in one project.
- Tokens in `shared_preferences`.
- Imperative `Navigator.push` deep into the app for a project with web/deep-
  link requirements.
- Flutter Web for content / SEO-dependent surfaces.
- Manual local builds in production with no CI artifact.
- Plugins pinned ≥ 1 major behind upstream.

---

## Output section

```markdown
### Flutter

#### Flutter & Dart
- Flutter SDK: <N> (current stable: <M>) — status
- Dart null safety: enforced

#### State management
(Solution in use; appropriateness for scale)

#### Routing
(Declarative vs imperative; deep-link support)

#### Platform integration
- Plugin currency: top 5 plugins and their versions vs current
- Native bridges (channels / FFI) used for: (list)

#### Platform features under-utilized
- iOS: (cross-reference ios-native.md)
- Android: (cross-reference android-native.md)

#### Performance
(Build mode discipline; const constructor coverage; image caching; list perf;
renderer choice)

#### Compliance
(Privacy declarations; PrivacyInfo.xcprivacy; targetSdkVersion; ATT; account
deletion)

#### Web target (if applicable)
(Justification; renderer; SEO concerns)

#### Findings
(table)
```
