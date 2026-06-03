# Product-Type Sub-Reference: iOS Native

Audit a native iOS codebase (Swift / SwiftUI / UIKit, sometimes Objective-C)
for platform-idiom fit and under-utilized capabilities.

---

## Detection

- `*.xcodeproj` or `*.xcworkspace` at repo root
- `Package.swift` (Swift Package) or `Podfile` (CocoaPods) or `Cartfile`
- `Info.plist` declaring `CFBundleIdentifier`
- `*.swift` source under `Sources/` or app target

If both `ios/` and `android/` are present alongside React Native or Flutter
config, this is a wrapped app — defer detail to the RN / Flutter sub-reference,
flag here only what is iOS-platform-bridge-specific.

---

## Live Discovery

App Store policies, Required Reasons API list, and SDK/Xcode version
requirements change continuously.

- **App Store Review Guidelines:** Fetch
  https://developer.apple.com/app-store/review/guidelines/ — current
  guidelines.
- **Upcoming requirements:** Fetch
  https://developer.apple.com/news/upcoming-requirements/ — current
  deadlines (minimum SDK, Sign in with Apple, etc.).
- **Required Reasons APIs:** Fetch
  https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api —
  current list of APIs requiring a declared reason in
  `PrivacyInfo.xcprivacy`.
- **Privacy nutrition labels:** Fetch
  https://developer.apple.com/app-store/app-privacy-details/ — current
  required disclosures.
- **Xcode / iOS SDK versions:** Fetch the App Store Connect news for
  current minimum-SDK requirements (e.g., "must be built with Xcode 16+
  starting Apr 2025").
- **Latest Swift / SwiftUI:** Fetch https://www.swift.org/ for current
  Swift version; `mcp__plugin_context7_context7__query-docs` for current
  SwiftUI APIs.

Cite fetched URLs and dates in the report. The embedded list of "Live
Activities, App Clips, Sign in with Apple, Passkeys, etc." may be missing
features added since the skill's last update — check the developer.apple.com
What's New page for the most recent OS release.

If discovery fails: note explicitly; embedded list is the fallback.

---

## Investigate

### UI architecture

- **SwiftUI vs UIKit.** Pure SwiftUI, pure UIKit, or hybrid (`UIHostingController`)?
  For new code in iOS 16+ targets, SwiftUI is the platform default. Hybrid is
  fine; pure UIKit on a new project warrants a "why?" note.
- **Navigation.** `NavigationStack` (iOS 16+) vs `NavigationView` (deprecated)
  vs custom navigation stack. Flag deprecated APIs.
- **Async patterns.** `async`/`await` (Swift 5.5+) vs Combine vs callbacks.
  Mixing all three in new code is a smell.

### Platform features under-utilized

- **Live Activities + Dynamic Island.** For apps with ongoing tasks (deliveries,
  timers, sport scores, builds), Live Activities surface in the Dynamic Island
  and Lock Screen — major engagement lift. Search for `ActivityKit`. Flag
  apps that have obvious Live-Activity-shaped features but no integration.
- **Widgets.** `WidgetKit` for home/lock screen widgets. Apps with daily
  glanceable info (next event, count, status) usually benefit.
- **App Shortcuts.** `AppIntents` framework — surfaces actions in Spotlight,
  Siri, Shortcuts app. Cheap to add for any app with primary actions.
- **App Clips.** For consumer apps with a "first interaction without install"
  use case (events, payments, parking). `AppClip` target.
- **Universal Links.** Deep linking with `apple-app-site-association`. Search
  for that file in `public/` or backend serving config; check `Associated
  Domains` capability.
- **Sign in with Apple.** Required by App Store guidelines if other social
  logins are offered. Check for `ASAuthorizationAppleIDProvider`.
- **Passkeys (`AuthenticationServices`).** For login flows in iOS 16+,
  passkeys offer better UX than passwords. Check for `ASAuthorizationPasskey
  Credential`.
- **Background tasks.** `BGTaskScheduler` for periodic background work. Apps
  using `Timer` or sleeping web requests for "background" work are doing it
  wrong.
- **`async let` and `TaskGroup`** for parallel work — vs serial `await` chains.
- **StoreKit 2** for purchases (subscriptions, in-app). StoreKit 1 is legacy.
- **HealthKit / HomeKit / WeatherKit / WalletKit** if the product domain fits.

### Platform compliance (App Store)

- **Privacy nutrition labels.** App Store requires declarations for data
  collection. Check that the codebase's actual data collection matches what's
  expected (e.g., analytics SDK present but app's privacy declaration says no
  data collected = audit risk).
- **App Tracking Transparency.** If the app uses IDFA / cross-site tracking,
  `requestTrackingAuthorization` must be implemented and the prompt shown.
- **Required reasons APIs.** Apple now requires a declared reason for using
  certain APIs (UserDefaults, file timestamps, system boot time). Check
  `PrivacyInfo.xcprivacy` is present and accurate.
- **Sign in with Apple** requirement when other social logins exist.
- **Account deletion.** App Store guideline 5.1.1(v) — apps that allow
  account creation must allow deletion in-app. Verify.

### Security

- **Keychain** for tokens / secrets — not `UserDefaults`. Search for
  `UserDefaults.set` of anything that looks like a token.
- **Secure Enclave** / FaceID-gated secrets where appropriate (`LAContext`,
  `kSecAttrAccessControl`).
- **App Transport Security (ATS).** Check `Info.plist` for
  `NSAllowsArbitraryLoads = true` — disabling ATS globally is almost always
  wrong.
- **Pinning.** Cert pinning if the app talks to a single-purpose backend.
- **Jailbreak detection.** Only if threat model warrants it.

### Performance

- **App start time.** Storyboard launch screen vs SwiftUI launch — minimize
  blocking init.
- **List performance.** `LazyVStack` / `List` with stable identifiers vs
  `VStack` with everything rendered.
- **Image loading.** `AsyncImage` is convenient but uncached; `Kingfisher` /
  `Nuke` / `SDWebImage` for production.
- **Memory.** Profile heaviest images; downsample per device.

### Distribution & update

- **TestFlight** for beta — flag absence as a process gap.
- **Phased release** — flag if all releases go 100% immediately.

---

## Red flags

- All async work via callbacks or Combine in a Swift 5.5+ project (use
  `async`/`await`).
- Tokens stored in `UserDefaults`.
- No `PrivacyInfo.xcprivacy` in a 2024+ app.
- `NSAllowsArbitraryLoads = true` without per-domain exception.
- Custom navigation stack reinventing `NavigationStack`.
- Heavy work on the main thread on app launch.
- IAP / subscriptions implemented with custom HTTP calls instead of StoreKit.

---

## Output section

```markdown
### iOS Native

#### UI architecture
(SwiftUI / UIKit / hybrid — cite evidence)

#### Platform features under-utilized
- [ ] Live Activities — applicable? (yes/no, why)
- [ ] Widgets
- [ ] App Shortcuts (AppIntents)
- [ ] App Clips
- [ ] Universal Links
- [ ] Sign in with Apple
- [ ] Passkeys
- [ ] Background tasks (BGTaskScheduler)
- [ ] StoreKit 2
- [ ] Domain frameworks (HealthKit/HomeKit/etc.)

#### Compliance
- Privacy nutrition labels accurate: yes/no/unverifiable
- App Tracking Transparency: required? implemented?
- PrivacyInfo.xcprivacy: present? accurate?
- Sign in with Apple: required? implemented?
- Account deletion in-app: present?

#### Security
(Keychain usage; ATS config; pinning; etc.)

#### Performance
(App start; list perf; image loading.)

#### Distribution
(TestFlight; phased rollout.)

#### Findings
(table — severity / area / evidence / fix)
```
