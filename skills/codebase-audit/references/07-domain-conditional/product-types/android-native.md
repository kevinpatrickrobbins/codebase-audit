# Product-Type Sub-Reference: Android Native

Audit a native Android codebase (Kotlin / Java, sometimes via NDK) for
platform-idiom fit and under-utilized capabilities.

---

## Detection

- `build.gradle` / `build.gradle.kts` at root and `app/`
- `app/src/main/AndroidManifest.xml`
- `*.kt` or `*.java` under `app/src/main/java/` or `kotlin/`
- `gradle/wrapper/`

If both `ios/` and `android/` are present alongside RN or Flutter, this is a
wrapped app — defer to the RN/Flutter sub-reference; flag here only what is
Android-platform-bridge-specific.

---

## Live Discovery

Play Console requirements (target API level, foreground service types,
permission policies) and Android SDK features change yearly.

- **Target API level requirement:** Fetch
  https://support.google.com/googleplay/android-developer/answer/11926878 —
  current minimum `targetSdkVersion` for new and update submissions. (This
  bumps yearly — the embedded value in this sub-reference will rot.)
- **Play Console policies:** Fetch
  https://support.google.com/googleplay/android-developer/topic/9858052 —
  current policy area updates (data safety, foreground services, exact
  alarms, etc.).
- **Foreground service types:** Fetch
  https://developer.android.com/about/versions/14/changes/fgs-types-required —
  current required types and use cases.
- **Latest Android / Jetpack libraries:** Fetch
  https://developer.android.com/jetpack/androidx/versions for current
  AndroidX version channel; or `mcp__plugin_context7_context7__query-docs`
  for current Compose / Jetpack docs.
- **Latest Kotlin:** Fetch https://kotlinlang.org/docs/releases.html
  for current stable.
- **Data safety form requirements:** Fetch
  https://support.google.com/googleplay/android-developer/answer/10787469
  for current data-safety disclosure requirements.

Cite fetched URLs and dates. The embedded list of features (WorkManager,
Compose, DataStore, etc.) may be missing capabilities added since the
skill's last update.

If discovery fails: note explicitly; embedded list is the fallback.

---

## Investigate

### UI architecture

- **Jetpack Compose vs View XML.** For new code on minSdk 21+, Compose is the
  platform default. Pure Views in a new project warrants a "why?" note.
- **Single-Activity architecture.** Multiple `Activity` classes vs single
  Activity + Navigation Compose / Jetpack Navigation. Modern Android prefers
  single-Activity.
- **State management.** ViewModel + StateFlow / Compose state vs LiveData
  (legacy). Old projects often mix all three.

### Platform features under-utilized

- **WorkManager.** For deferred / periodic background work — replaces
  `AlarmManager` + `JobScheduler` + `BroadcastReceiver` patterns. Search for
  custom `AlarmManager.setRepeating` or scheduled `Service` — almost always
  should be WorkManager. (This is the user's specific example — common
  finding.)
- **DataStore (Preferences / Proto).** Replaces `SharedPreferences` —
  coroutine-based, type-safe. SharedPreferences in new code is legacy.
- **Room** for DB — vs raw `SQLiteOpenHelper` (legacy) or rolling your own.
- **Hilt** for DI vs manual or Dagger 2 boilerplate.
- **Coroutines + Flow** for async — vs RxJava (legacy in most new contexts)
  or callback chains.
- **Compose Animation APIs** for transitions — vs custom property animators.
- **Navigation Compose / Jetpack Navigation.**
- **Splash screen API** (Android 12+) — replaces custom splash Activity.
- **Predictive back gesture** support (Android 13+).
- **Per-app language preference** API (Android 13+) — `LocaleManager`.
- **App Widgets.** Glance API (modern Compose-based). Apps with daily
  glanceable info often benefit.
- **App Shortcuts** — static (`shortcuts.xml`) and dynamic. Cheap surface for
  primary actions.
- **App Links** — verified deep links via `assetlinks.json`.
- **Tile / Wear OS / Auto / TV.** If the domain fits.
- **Health Connect** if health-related.

### Platform compliance (Play Store)

- **Target API level.** Play Store requires recent target SDK (raised
  yearly — currently 34 / 35 for new and update submissions). Check
  `targetSdkVersion`.
- **Data safety form alignment.** Play Console requires a Data Safety form;
  check the codebase's actual data collection (analytics SDKs, ads, etc.)
  matches what is likely declared.
- **Account deletion.** Play Console policy requires in-app account deletion
  for apps that create accounts. Verify.
- **Permissions.** Sensitive permissions (`READ_CONTACTS`, location,
  background location, exact alarm) — declared with justification, requested
  in-context.
- **Foreground service types.** Android 14+ requires declared types
  (`location`, `mediaPlayback`, `dataSync`, etc.). Check
  `<service android:foregroundServiceType=...>`.
- **Notification permission** — Android 13+ runtime permission for
  `POST_NOTIFICATIONS`. Apps that just `notify()` without requesting fail
  silently.

### Security

- **Android Keystore** for keys / tokens — not `SharedPreferences` or files.
  Search for plaintext token storage.
- **`EncryptedSharedPreferences`** as a midpoint for app secrets.
- **Network security config** (`network_security_config.xml`) — flag
  `cleartextTrafficPermitted="true"` outside dev builds.
- **Cert pinning** via `okhttp` `CertificatePinner` if domain-stable.
- **Biometric auth** via `BiometricPrompt`.
- **Tapjacking protection** — `setFilterTouchesWhenObscured`.
- **R8 / ProGuard** enabled in release builds (rule sets correct, app not
  obfuscation-broken).

### Performance

- **App start time.** Cold start < 2s on mid-range. Profile with Macrobenchmark.
  Hunt for heavy work in `Application.onCreate` and first-Activity
  `onCreate`.
- **Baseline Profiles** (`androidx.benchmark`) for hot paths — major perf win
  for app startup.
- **List performance.** `LazyColumn` (Compose) / `RecyclerView` with proper
  ViewHolder reuse — never raw `ListView` in new code.
- **Image loading.** Coil (Compose-friendly), Glide. Plain `BitmapFactory.decodeFile`
  in production code is suspect.
- **Memory.** Large bitmaps not downsampled per device density.
- **Battery.** WorkManager constraints; `JobScheduler` for legacy.

### Distribution & update

- **App Bundles** (`.aab`) — replaced APKs as Play Console requirement.
- **In-app updates** API for prompted updates.
- **Internal testing track** for beta.
- **Phased rollout** — flag if all releases go 100% immediately.

---

## Red flags

- `AlarmManager.setRepeating` for periodic work (use WorkManager).
- Tokens / secrets in `SharedPreferences` (use Keystore /
  EncryptedSharedPreferences).
- `targetSdkVersion` more than one year behind current (Play Store will
  reject submissions).
- `cleartextTrafficPermitted="true"` in production network config.
- Foreground service without `foregroundServiceType` on Android 14+.
- Notifications fired without requesting `POST_NOTIFICATIONS` (Android 13+).
- `Application.onCreate` doing heavy work blocking app start.
- Plain `Activity` per screen with manual back stack juggling (use
  Navigation).
- `RxJava` and `Coroutines` and `LiveData` and `StateFlow` all coexisting
  in a small project.
- No R8 / ProGuard in release builds.

---

## Output section

```markdown
### Android Native

#### UI architecture
(Compose / Views / hybrid — cite evidence)

#### Platform features under-utilized
- [ ] WorkManager (vs custom AlarmManager / Services)
- [ ] DataStore (vs SharedPreferences)
- [ ] Room (vs raw SQLite)
- [ ] Hilt
- [ ] Coroutines + Flow
- [ ] Splash Screen API
- [ ] Predictive back gesture
- [ ] App Widgets (Glance)
- [ ] App Shortcuts
- [ ] App Links (verified)
- [ ] Health Connect / domain frameworks

#### Compliance
- targetSdkVersion: <N> (current Play requirement: <M>) — status
- Account deletion in-app: present?
- Permissions justified and requested in-context: yes/no
- Foreground service types declared (Android 14+): yes/no
- POST_NOTIFICATIONS handled (Android 13+): yes/no

#### Security
(Keystore usage; network security config; cert pinning; R8.)

#### Performance
(Cold start; Baseline Profiles; list rendering; image loading.)

#### Distribution
(App Bundles; in-app updates; testing tracks.)

#### Findings
(table — severity / area / evidence / fix)
```
