# 19 — Product-Type Audit (Dispatcher)

Detect the product type and run the matching platform-idiom audit. The goal is
to surface platform-specific opportunities the codebase isn't taking advantage
of — e.g., an Android app reaching for cron-style polling instead of
WorkManager, a Next.js app that should be a PWA but has no manifest, a
multi-tenant SaaS without proper tenant scoping.

Create or update:

`/docs/audits/product-type-audit.md`

This module is a **dispatcher**. It detects, classifies, and routes to one or
more product-type sub-references in `references/07-domain-conditional/product-types/`.

---

## How to investigate

### Step 1: Detect the product type(s)

A codebase often has more than one type — e.g., an Expo app + an admin web
dashboard + a CLI is three types in one repo. Detect each and run each
applicable sub-reference.

| Signal | Product type | Sub-reference |
|---|---|---|
| `*.xcodeproj`, `Package.swift`, `Info.plist`, `*.swift` | iOS native | `references/07-domain-conditional/product-types/ios-native.md` |
| `build.gradle*`, `AndroidManifest.xml`, `*.kt`/`*.java` under `app/src/main/` | Android native | `references/07-domain-conditional/product-types/android-native.md` |
| `react-native` in `package.json`; `ios/` + `android/` siblings; `metro.config.js` | React Native (bare) | `references/07-domain-conditional/product-types/react-native.md` |
| `expo` in `package.json`; `app.json` / `app.config.{js,ts}` with `expo:` key | Expo | `references/07-domain-conditional/product-types/expo.md` |
| `pubspec.yaml`; `*.dart`; `flutter` SDK references | Flutter | `references/07-domain-conditional/product-types/flutter.md` |
| `electron` / `electron-builder` in `package.json`; `main.{js,ts}` with `BrowserWindow` | Electron | `references/07-domain-conditional/product-types/electron.md` |
| `manifest.json` with `manifest_version` (2 or 3); `background`/`content_scripts` keys | Browser extension | `references/07-domain-conditional/product-types/browser-extension.md` |
| `manifest.webmanifest` / `manifest.json` (web app type); service worker registration | PWA | `references/07-domain-conditional/product-types/pwa.md` |
| Multi-tenant data model (tenant_id columns; org/team primitives in auth); pricing/billing routes | SaaS multi-tenant | `references/07-domain-conditional/product-types/saas-multi-tenant.md` |
| Two-sided signup; payment splitting (Stripe Connect, Adyen MarketPay); ratings/reviews on actors | Marketplace | `references/07-domain-conditional/product-types/marketplace.md` |
| `bin` field in `package.json`; `cli.{ts,js}`; `commander`/`yargs`/`clipanion`/`oclif` import; no UI code | CLI tool | `references/07-domain-conditional/product-types/cli-tool.md` |
| API only (no frontend bundle, no static-output rendering) — Express/Fastify/Hono/NestJS/FastAPI server with no view layer | API-only service | `references/07-domain-conditional/product-types/api-only.md` |

If no signal matches strongly, the product is likely a generic web app (Module
1 covers that) — no type-specific audit, just note it and skip.

### Step 2: For each detected type, load the sub-reference

Each sub-reference is self-contained: detection details, platform idioms to
check, common under-utilization patterns, store/distribution policy if
applicable, platform-specific security and perf considerations, output
template.

Run each sub-reference and produce a section in `product-type-audit.md` per
detected type.

### Step 3: Cross-cutting platform-agnostic checks

Some checks apply to *any* product that has end users:

- **Versioning & update strategy.** How does a user move from version A to B?
  - Web: every visit pulls latest; cache-busting works.
  - Native: store update; OTA where available (Expo Updates, CodePush).
  - Desktop: auto-updater (`electron-updater`, Squirrel, Sparkle).
  - Browser extension: store auto-update.
  - CLI: how does a user upgrade? (npm install, brew, package manager.)

- **Crash & error reporting.** Sentry / Bugsnag / Crashlytics integrated?
  Source maps uploaded? PII filtered?

- **Telemetry / analytics.** Any usage tracking? Privacy-respecting? Opt-in
  required for jurisdictions that demand it?

- **Onboarding / first-run.** Distinct from web onboarding (covered in Module
  7) — native apps often have permission grants, push opt-ins, etc.

- **Backwards compatibility.** Old versions in the wild that the backend must
  still support — is there a deprecation policy?

---

## Output structure

```markdown
# Product-Type Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Detected product types

| Type | Confidence | Evidence |
|---|---|---|
| Expo | High | package.json:dependencies has `expo`; app.config.ts present |
| SaaS multi-tenant | Medium | tenant_id on `events`, `users.organization_id` FK |
| API-only | (skipped — no signal) | — |

## Per-type audit findings

### Expo
(Findings from `references/07-domain-conditional/product-types/expo.md`)

### SaaS multi-tenant
(Findings from `references/07-domain-conditional/product-types/saas-multi-tenant.md`)

## Cross-cutting findings
- Versioning & update strategy
- Crash & error reporting
- Telemetry & analytics
- Onboarding / first-run
- Backwards compatibility

## Improvement Tiers

### Tier 1 — Low-cost
### Tier 2 — Medium-cost
### Tier 3 — High-cost (architectural)

## Final Recommendation
Single best product-type-related action for this codebase right now.

## Decision Summary

- Product type clearly fits the codebase: yes / no
- Major platform capabilities under-utilized: yes / no — list
- Worth acting on now: yes / no

## Confidence
High / Medium / Low

## Out of Scope / Inconclusive
What requires running the app on real devices or against real platforms to
confirm.
```

---

## Notes

- A finding here often crosses with other modules. Examples: WorkManager
  vs custom AlarmManager (this module + Module 09), Keychain vs UserDefaults
  for tokens (this module + Module 02). Cross-reference both ways.
- "Not using X" is only a finding if X would meaningfully improve the project.
  Don't recommend platform features for the sake of completeness — calibrate
  to whether the workload benefits.
- If a sub-reference doesn't exist for a detected type, note it in the output
  ("Detected: Tauri desktop app — no sub-reference; recommended manual
  platform-idiom review against Tauri docs"). Do not invent guidance.

## Sub-reference inheritance

Sub-references under `references/07-domain-conditional/product-types/*.md`
are platform-specific investigation playbooks. They **inherit from this
dispatcher**:

- **Severity calibration** — sub-refs use this dispatcher's severity scale
  (Critical / High / Medium / Low) and the Module 00.2 likelihood scale
  (Likely / Possible / Unlikely). They do not redefine severity.
- **Evidence requirements** — sub-refs follow the top-level rule: every
  finding cites a file path and (where relevant) a line number. Absence-
  based findings name what was searched for and where.
- **Output discipline** — each sub-ref provides its own "Output section"
  template; that template feeds into this dispatcher's "Per-type audit
  findings" section per the output structure above.

If a sub-reference omits a severity or evidence section, that is by design
— the dispatcher owns those concerns. Do not re-add them to the sub-ref;
update the dispatcher if calibration needs to change.
