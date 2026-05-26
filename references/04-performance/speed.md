# 10 — Speed Audit (Perceived Performance)

Audit the codebase for **user-perceived speed** — how fast the running app
*feels* on a typical user's network and device.

Create or update:

`/docs/audits/speed-audit.md`

---

## Scope and distinction from Module 09

This module is **not** a duplicate of Module 09 (Performance & Scalability).
Split:

| Concern | Module |
|---|---|
| "Will it scale to 10k concurrent users?" | 09 (Performance & Scalability) |
| "Will the DB choke at 1M rows?" | 09 |
| "Are there N+1 queries?" | 09 |
| "Does it feel fast right now on a mid-range Android on 4G?" | **10 (this)** |
| "What's the LCP / INP / CLS?" | **10** |
| "Is the bundle bloated?" | **10** (with overlap to 09) |
| "Cold-start risk?" | 11 (Serverless) |

Module 09 owns scalability and DB; this module owns the user-side experience.
Cross-reference both ways when findings overlap.

---

## When to apply

Run this module on any project that ships a **user-facing UI** — web app, web
embed, mobile web, native mobile, desktop. For pure APIs / CLIs with no human
UI, skip — perceived speed is not the right framing; latency-and-scalability
analysis lives in Module 09.

---

## Step 0: Live Discovery

Web platform performance guidance shifts as browsers ship new capabilities
and Core Web Vitals thresholds update.

### Current Core Web Vitals thresholds

- Fetch https://web.dev/articles/vitals — current LCP / INP / CLS / FCP
  / TBT thresholds. INP replaced FID in March 2024; future updates may
  shift again. Capture today's "good" and "needs improvement" cutoffs.

### Current Baseline-supported platform features

For predictions about modern web APIs the project could lean on (View
Transitions, scroll-driven animations, `content-visibility`, native lazy
loading, native popover, etc.):

- Fetch https://web.dev/baseline — current Baseline state.
- Cross-ref Module 17 (Frontend Modernization) Live Discovery — they may
  already have fetched the relevant subset.

### Current framework perf primitives

For projects using React 18+, Next.js, Remix, SvelteKit, Astro, Qwik, etc.:

- `mcp__plugin_context7_context7__resolve-library-id` then `query-docs` for
  the framework — surfaces current streaming / RSC / islands / resumability
  capabilities the project may not be using yet.

### Output of discovery

```markdown
## Live Discovery (fetched YYYY-MM-DD)

| Source | Discovered |
|---|---|
| web.dev/vitals | LCP good ≤ <s>s; INP good ≤ <ms>ms; CLS good ≤ <n> |
| web.dev/baseline | Newly Baseline: <list of features the project could adopt> |
| Framework (<name>) | Latest stable: <ver>; current perf primitives: <list> |

Predictions below are calibrated against this state.
```

If discovery fails: note explicitly. The Core Web Vitals targets embedded
below reflect this skill's last update and may be stale.

---

## How to investigate

Speed audits live or die on calibration. The wrong frame: "how fast can it go
on my MacBook on Wi-Fi". The right frame: "how does it perform for the user
profile this product is built for?" Check the README and product positioning to
infer the user — global B2C is "mid-range Android on 4G in a region with high
RTT"; B2B internal tooling is "modern laptop on corporate Wi-Fi".

If real measurements are available (Lighthouse reports committed, RUM data in
analytics, Vercel Speed Insights, Cloudflare Web Analytics), use them. If not,
predict from code.

### Step 1: Identify the rendering target

The right perf model depends on what the app *is*:

| Target | Primary metrics | Critical concerns |
|---|---|---|
| Web (browser SPA / SSR / SSG) | LCP, INP, CLS, FCP, TBT, TTI | Bundle, hydration, asset perf, network |
| Mobile native (iOS/Android) | App start time, screen TTI, animation FPS | Cold start, asset weight, lifecycle |
| React Native / Flutter | App start, screen TTI, JS thread cost | Bridge cost (RN), build mode (release vs debug measurement) |
| Desktop (Electron) | Launch time, panel-switch latency | Bundle size, IPC roundtrips |
| PWA | Same as web + install / offline experience | Service worker fetch path, app shell |

Detect which by reading Module 01 (System Architecture) output, and confirm
specifics.

### Step 2: Core Web Vitals (web only)

For each significant route:

#### Largest Contentful Paint (LCP) — target ≤ 2.5s

Predict from code:

- What is the LCP element likely to be? Hero image, hero heading, video poster?
  Inspect the page-level component for the most likely candidate.
- Is the LCP image served via `next/image` / `<Image>` / responsive `srcset`?
  Or a raw `<img src="...big.png">`?
- Is `priority` / `fetchpriority="high"` set on the LCP image?
- Are render-blocking resources in front of it? Look for non-async `<script>`,
  blocking `<link rel="stylesheet">`, font requests without `font-display`.
- Is the LCP blocked behind client-side data fetches? Anti-pattern: SSR shell
  shows skeleton, real content fetches client-side after hydration — LCP fires
  on skeleton, real content arrives much later (poor INP/UX even if LCP looks OK).

#### Interaction to Next Paint (INP) — target ≤ 200ms

INP replaced FID in March 2024. It measures the *worst* interaction delay on
the page, not just the first.

- Hunt for long tasks on user interaction:
  - Heavy state updates (large list re-renders without `React.memo` or
    virtualization)
  - Synchronous validation on input change (regex over large text on each
    keystroke)
  - Layout thrashing (read-write-read-write DOM in event handlers)
  - Listeners that do too much before yielding (`scroll` without debounce)
- React 18+ projects: are `startTransition` / `useDeferredValue` used for
  non-urgent updates?
- Check for blocking JS during typical interactions: opening menus, switching
  tabs, sorting tables, opening modals.

#### Cumulative Layout Shift (CLS) — target ≤ 0.1

- Images and embeds without explicit `width`/`height` (or aspect-ratio CSS).
- Web fonts without `size-adjust` or fallback metrics, causing reflow on font
  swap.
- Late-loaded ads / embeds / banners pushing content down.
- Content swapping mid-render (skeleton → real content with different
  dimensions).

### Step 3: First Contentful Paint and Time-to-Interactive

- **FCP:** time to first paint of any content. Driven by HTML response time,
  render-blocking resources, and font policy.
- **TBT (Total Blocking Time):** sum of long-task blocking time. Drives TTI.
- **TTI:** when the page becomes reliably interactive. For SPAs, this is
  usually after main bundle hydration.

### Step 4: Critical rendering path

- **Render-blocking CSS.** Search for `<link rel="stylesheet">` without
  `media="print"` swap or critical-path inlining. Frameworks vary: Next.js
  inlines critical CSS by default; Vite + custom apps often don't.
- **Render-blocking JS.** Any `<script>` without `async`/`defer` in the head.
- **Inline `<script>` blocks** at the top of the document that do non-trivial
  work.
- **`@import` cascades in CSS** — each `@import` is a serial round trip.
- **Web fonts.**
  - Self-hosted via `next/font`, `@fontsource`, or system stack? Or hot-linked
    from Google Fonts / Adobe Fonts (extra DNS + connection)?
  - `font-display: swap` (or `optional`) set?
  - Variable fonts vs multiple weight files?
  - Subsetting (latin only vs full Unicode)?
  - Preloading critical fonts with `<link rel="preload" as="font">`?

### Step 5: Bundle and JS execution

(Some overlap with Module 09 — the focus here is *runtime* execution cost on
the user's device, not bundle size in isolation.)

- **Bundle size.** Run `next build --analyze` mentally — what's in the main
  bundle that doesn't need to be?
- **Code splitting.** Are routes lazily loaded? Heavy modals dynamically
  imported (`import()` / `dynamic()`)?
- **Tree shaking.** Imports of full libraries (`import _ from 'lodash'`) vs
  cherry-picked (`import debounce from 'lodash/debounce'`).
- **Dependency weight.** Known-heavy libraries shipped to client:
  - `moment` (~270 KB min) → use `date-fns` (~10 KB cherry-picked) or `dayjs`
    (~7 KB) or native `Intl.DateTimeFormat`.
  - Full `lodash` → cherry-pick or use ES alternatives.
  - `xlsx`, `pdfkit`, `chart.js` + plugins, `three`, `monaco-editor` — should
    be dynamically imported, not in main bundle.
  - Older versions of Material UI / styled-components shipped to every page.
- **Polyfill bloat.** A core-js bundle for browsers that don't need it (check
  `browserslist` — `defaults` includes browsers that need polyfills; `>0.5%,
  not dead, last 2 versions` excludes most of them).
- **Hydration cost.** For SSR/SSG apps, large hydration payloads block INP.
  React Server Components / Astro Islands / Qwik resumability dramatically
  reduce this — flag if not used in a project where hydration is the
  bottleneck.

### Step 6: Image performance

- Format: WebP or AVIF with fallback, or stuck on JPEG/PNG?
- Sizing: served at intrinsic resolution or scaled down? `<Image>` / `srcset`
  with multiple sizes?
- Lazy loading: native `loading="lazy"` on below-the-fold images?
- Above-the-fold: **NOT** lazy-loaded; ideally `priority` / `fetchpriority="high"`.
- Decoding: `decoding="async"` on non-critical images.
- LQIP / blur placeholder for hero images.
- Sprites / icon fonts vs SVG components — context-dependent; flag bloat.

### Step 7: Network

- **Total request count** for a typical route render. > 50 is a smell;
  > 100 is an issue.
- **DNS prefetch / preconnect / preload** for critical third-party domains.
- **HTTP/2 / HTTP/3** — usually handled by the host; flag if origin doesn't
  support it.
- **Third-party scripts.** Analytics, chat widgets, A/B test SDKs, marketing
  pixels — list every one and its perf cost. Often the biggest single drag on
  a marketing site.
- **Self-host vs CDN.** Self-hosted Google Tag Manager / fonts can be faster
  than the canonical CDN due to connection reuse.
- **Cache headers.** Static assets with long `max-age`? HTML with sensible
  `s-maxage`?

### Step 8: Animation and motion

- **Animations using `transform` and `opacity` only** (compositor-only, fast)
  vs animations on `width`, `height`, `top`, `left` (layout-thrashing, slow).
- **`will-change` overuse** — promotes elements to layers permanently, eats GPU
  memory.
- **`requestAnimationFrame` loops** without unsubscribe on unmount.
- **Heavy CSS** (`box-shadow`, `filter: blur`) on scroll-triggered elements.
- **`prefers-reduced-motion`** respected? (Overlap with Module 04 — flag here
  too if missing for *perf* reasons; OS reduce-motion users often have
  lower-end devices.)
- **Frame budget for hero animations** — running 60fps on a low-end Android?

### Step 9: Streaming and progressive rendering (web)

- **Streaming SSR.** Next.js App Router, Remix, SvelteKit, Nuxt 3 all support
  it. Is the app streaming, or rendering whole-page-then-flush?
- **Suspense boundaries.** Used to stream slow data while critical content
  paints? Or absent, blocking the whole render?
- **PPR (Partial Pre-Rendering)** if Next.js — flag opportunities.
- **Islands** (Astro, Fresh) where applicable — flag if the project uses a
  heavy framework for content that's mostly static.

### Step 10: Mobile-app-specific (iOS / Android / RN / Flutter)

- **App start time (cold).** Should be < 2s on mid-range hardware. Hunt for:
  - Synchronous network calls in `applicationDidFinishLaunching` /
    `MainActivity.onCreate` / `App` constructor.
  - Heavy library imports at startup.
  - Auth initialization that blocks first paint.
- **Time-to-interactive per screen.** Tap → screen-rendered ≤ 1s on
  mid-range.
- **Animation FPS during navigation.** 60fps target; 30fps drops are visible.
- **List performance.** Are large lists virtualized (`FlashList`, `RecyclerView`,
  `LazyColumn`)?
- **Image caching.** `expo-image`, Glide, SDWebImage configured properly?
- **Bundle size (RN/Flutter).** Hermes (RN) / split APKs / asset trimming?
- **Memory footprint.** Large bitmaps not downsampled.

(Detailed mobile guidance also lives in Module 19 sub-references for
`ios-native`, `android-native`, `react-native`, etc. — flag findings here, run
Module 19 for deeper platform-idiom analysis.)

### Step 11: Real measurement integration

Note the state of the team's perf measurement:

- **Lighthouse CI** in CI? Threshold for failing builds?
- **Vercel Speed Insights / Cloudflare Web Analytics** integrated?
- **Custom RUM** (`web-vitals` package, Sentry Performance, Datadog RUM)?
- **Synthetic monitoring** (Checkly, BetterStack, custom Playwright scheduled
  runs)?

A speed audit without RUM data is a prediction; with RUM it's a diagnosis.
If absent, recommend the lowest-friction RUM addition (`web-vitals` package
posting to whatever logging stack already exists).

---

## Red flags

- LCP image is a raw `<img src="hero.png">` (no priority, no responsive
  variants, no preload).
- Main bundle > 200 KB compressed for a page that doesn't need it.
- `import _ from 'lodash'` (or moment) in client code.
- Images without explicit dimensions causing CLS.
- Non-async third-party scripts in `<head>`.
- `setInterval` running expensive work on every tick.
- React `useEffect(() => fetch...)` in the LCP component when SSR/loader would
  serve it pre-rendered.
- `font-display: block` (the default for unconfigured custom fonts).
- Long lists rendered without virtualization.
- No `priority` / `fetchpriority="high"` on the obvious LCP candidate.
- Marketing site with 5+ third-party tracking pixels in `<head>`.

---

## Output structure

```markdown
# Speed Audit (Perceived Performance)

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Calibration
- User profile assumed: <e.g. mid-range Android, 4G, EU>
- Real measurement available: <yes (cite source) | no, predicted from code>

## Rendering target
<Web SPA / SSR / SSG / Native iOS / Native Android / RN / Flutter / Electron>

## Core Web Vitals (or platform equivalents)

| Metric | Target | Predicted | Real (if RUM available) | Status |
|---|---|---|---|---|
| LCP | ≤ 2.5s | ~3.2s | — | At risk |
| INP | ≤ 200ms | ~280ms | — | At risk |
| CLS | ≤ 0.1 | ~0.04 | — | OK |
| FCP | ≤ 1.8s | ~1.6s | — | OK |
| TBT | ≤ 200ms | ~340ms | — | At risk |

(For mobile native: App start time, screen TTI, animation FPS instead.)

## Findings by area

### LCP / hero
### INP / interactivity
### CLS / layout stability
### Critical rendering path
### Bundle & JS execution
### Images
### Fonts
### Network
### Animation
### Streaming / progressive rendering
### Mobile-specific (if applicable)

For each finding: severity, evidence (file:line), expected impact, fix,
implementation effort.

## Real Measurement Status
- Lighthouse CI: present / absent / configured-but-advisory
- RUM: present / absent — recommendation if absent
- Synthetic monitoring: present / absent

## Improvement Tiers

### Tier 1 — Low-cost (this week)
Config flips, attribute additions, small dependency swaps. Each with
estimated impact (e.g. "expected -300ms LCP").

### Tier 2 — Medium-cost (sprint or two)
Bundle restructure, hydration boundary changes, image-pipeline upgrade.

### Tier 3 — High-cost (architectural)
Streaming SSR adoption, RSC migration, switch to islands architecture, native
view layer rewrite. Be willing to say "none of these are worth doing right now."

## Final Recommendation
The single highest-impact speed action for this codebase right now. **One
thing.**

## Decision Summary

```
- App is currently fast for its target user: yes / no / unknown
- Worst metric: <LCP | INP | CLS | TTI | other>
- Single biggest win: <description, expected delta>
- Recommended posture: <keep / Tier 1 / Tier 2 / Tier 3 worth investigating>
```

## Confidence
- High / Medium / Low (and why)

## Out of Scope / Inconclusive
What needs real-device or production-traffic measurement to confirm.
```

---

## Severity calibration

- **Critical:** primary user-facing route fails Core Web Vitals targets in a
  way that materially harms business outcomes (e.g. LCP > 4s on the main
  marketing page; INP > 500ms on the primary interaction; mobile cold start
  > 5s on mid-range hardware).
- **High:** noticeable user-perceptible lag on common paths.
- **Medium:** suboptimal patterns that will degrade under content growth or
  scale (unbounded lists, missing image optimization, poor cache headers).
- **Low:** micro-optimizations, polish, and best-practice deviations with
  little real-world impact.

---

## Evidence requirements

Every finding cites a file path. Every recommendation includes an expected
impact estimate where possible ("removing the synchronous JSON.parse on app
launch in `App.tsx:14` should reduce cold start by ~200ms based on bundle
size").

For predictions made without real measurement, mark them as predictions and
recommend the team capture the actual baseline. Speed work without measurement
is guessing — surface that gap explicitly.
