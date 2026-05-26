# 17 — Frontend Modernization

Audit the frontend (web — and where applicable, native + RN/Flutter) for
**modernization opportunities** the codebase isn't using yet — modern CSS,
modern JS, current framework features, lean polyfill posture.

Output: `/docs/audits/frontend-modernization-audit.md`

---

## Cross-module relationship

- **Speed** (`04-performance/speed.md`) audits perceived performance;
  modernization wins often *also* deliver speed wins. This module focuses
  on the modernization angle (idiomatic, baseline-supported features the
  codebase isn't using yet).
- **UX** (`06-product/ux.md`) covers user-perceived heuristics; this module
  covers code-level modernization.
- **Architecture** (`03-quality/architecture.md`) covers structural
  patterns; this module is about platform-feature currency.
- **Volume CSS Monitor skill** — a sibling skill on baseline CSS features.
  Pair with this module when wiring in new CSS capabilities.

---

## Step 0: Live Discovery

Web platform features ship constantly. Baseline-supported features change
month over month. Fetch current truth before comparing.

### Baseline web platform features

Fetch https://web.dev/baseline — overview of Baseline status (Newly
available, Widely available, Limited availability).

For per-feature support: Fetch https://web-platform-dx.github.io/web-features/
or query the BCD-derived data on web.dev. caniuse.com is the canonical
human-readable view: Fetch https://caniuse.com/<feature-slug>.

Capture the current Baseline state for the modernization candidates listed
in this module:

- container queries
- `:has()`
- `@layer`
- `subgrid`
- `view-transition`
- `anchor-positioning`
- `text-wrap: balance` / `pretty`
- `<dialog>` element
- `color-mix()` / `oklch()` / `oklab()`
- `@property`
- iterator helpers (`Iterator.prototype.map` etc.)
- `Array.prototype.toSorted` and friends
- `Object.groupBy` / `Map.groupBy`
- `Promise.withResolvers`

Anything that has reached "Newly available" Baseline since the skill's last
update is fair game to recommend; anything still at "Limited availability"
should be flagged with a polyfill / progressive-enhancement caveat.

### Current framework stable versions

For each framework detected:

- **React, Next.js, Vue, Svelte, Astro, Remix, Solid, Nuxt, Angular:**
  `Bash` `npm view <pkg> version` returns latest stable. `npm view <pkg>
  versions --json` shows history.
- For deeper version-aware features (what's new in React 19? Next.js 15?),
  use `mcp__plugin_context7_context7__resolve-library-id` then
  `query-docs` to fetch current docs.

### Browser support matrix

Fetch the current `defaults` of browserslist if relevant:
https://browsersl.ist/#q=defaults — see what `defaults` actually expands to
*today*. Targeting `defaults` ships polyfills for browsers that may have
been pruned since the codebase last updated browserslist.

### Output of discovery

```markdown
## Live Discovery (fetched YYYY-MM-DD)

| Source | Discovered |
|---|---|
| Baseline (web.dev) | Newly available since last skill update: <list> |
| `next` latest (npm) | 15.5.0 |
| `react` latest (npm) | 19.0.0 |
| browserslist `defaults` (browsersl.ist) | Currently expands to: <browser list> |

Modernization recommendations calibrated against this state.
```

If discovery fails: note explicitly and fall back to embedded list (which
reflects the skill's last update and may understate current support).

---

## Step 1: Identify the frontend stack

- **Framework:** Next.js (App vs Pages Router) / Remix / SvelteKit / Nuxt /
  Solid Start / Astro / Qwik / Angular / Vue / Ember.
- **Major version.** vs current stable.
- **CSS strategy.** Tailwind / CSS Modules / styled-components / Emotion /
  vanilla-extract / CSS-in-JS / plain CSS.
- **Build target.** browserslist config, baseline support claims.

---

## Step 2: Modern CSS opportunities

CSS has shipped major capabilities in the last 2 years that most codebases
haven't adopted. Hunt for:

- **Container queries.** `@container` for component-level responsive vs
  viewport-only `@media`. Search for components that have media-query-based
  responsive sizing — many would be cleaner as container queries.
- **`:has()` selector.** Parent-state styling without JS. Search for cases
  where a wrapper class is toggled in JS to style a parent based on child
  state — `:has()` removes that JS.
- **Logical properties.** `margin-inline-start` vs `margin-left`,
  `padding-block` vs `padding-top/bottom`. Critical for RTL support.
- **`@layer` cascade layers.** Manage specificity without `!important`.
- **`subgrid`.** Aligned grid items across nested containers — Firefox and
  Safari long, Chrome since 117 (Sep 2023).
- **`color-mix()` / `oklch()` / `oklab()`.** Modern color spaces and
  computed colors.
- **`:focus-visible`.** Keyboard-only focus rings (cross-ref module
  `02-compliance/accessibility.md`).
- **`accent-color`** for native form controls.
- **`scroll-behavior: smooth` / scroll-snap** properties.
- **`aspect-ratio`** vs the old padding-bottom hack.
- **Custom properties (CSS variables)** used as the design-token layer
  vs Sass variables (compile-time only).
- **`@property`** for typed custom properties — animatable, fallback-able.
- **`text-wrap: balance` / `text-wrap: pretty`** for headlines and prose.
- **`view-transition`** API (where supported) for native page transitions.
- **`anchor-positioning`** for tooltip / popover positioning (rolling out
  in Chrome).
- **Dialog element** + `::backdrop` for modals — replaces div-based modal
  scaffolding.

For each opportunity: is the project on a `browserslist` that supports it?
Which components would benefit?

---

## Step 3: Modern JS opportunities

- **`async`/`await`** consistently — vs callback chains or raw `.then()`
  pyramids.
- **Top-level `await`** in modules where supported.
- **`Promise.allSettled` / `Promise.any`** vs hand-rolled equivalents.
- **`structuredClone()`** vs `JSON.parse(JSON.stringify(...))` (loses
  Date, Map, Set, etc.).
- **`Array.prototype.toSorted` / `toReversed` / `toSpliced` / `with`** —
  immutable array methods.
- **`Object.groupBy` / `Map.groupBy`** — replaces `reduce` group-by
  patterns.
- **Optional chaining `?.` / nullish coalescing `??`** — `obj && obj.x &&
  obj.x.y` patterns are legacy.
- **Logical assignment `??=` / `||=` / `&&=`.**
- **`AbortController` / `AbortSignal.timeout()`** for cancelable fetches —
  vs racing-with-setTimeout-and-Promise.race.
- **`Intl.*`** for date/number/list/relative-time formatting — vs
  `moment` / `dayjs`.
- **Native `fetch`** in Node 18+ — vs `node-fetch` / `axios` for trivial
  uses.
- **`crypto.randomUUID()`** vs the `uuid` package (in Node and modern
  browsers).
- **Iterator helpers (`map`/`filter`/`take` on iterators)** — landed 2024+.
- **Pipe operator** (Stage 2/3 — won't be widely available, but flag if
  codebase has shimmed it).
- **Decorators** (Stage 3 in TC39, TS 5+).
- **`using` declarations** (`Symbol.dispose`) for resource management.

---

## Step 4: Framework-version utilization

For the major frameworks, are the *current* idioms in use?

### React 18 / 19

- **Server Components** (RSC) where the framework supports them (Next.js
  App Router). A project on Next.js 13+ App Router but with `'use client'`
  on every component is missing the point.
- **Suspense boundaries** for async data — actually used, or only React
  18 in name?
- **`useTransition` / `useDeferredValue`** for non-urgent updates.
- **`use()` hook** for unwrapping promises (React 19).
- **`useOptimistic`** for optimistic UI (React 19).
- **`useActionState`** + form actions (React 19, Next.js 14+).
- **Concurrent rendering** properly leveraged.
- **`React.memo` / `useMemo` discipline** — overuse is a smell, but
  zero-use on heavy components is also a finding (or React Compiler used).
- **React Compiler** — auto-memoization. If on React 19 but not using
  the compiler, flag.

### Next.js

- **App Router vs Pages Router.** App Router is the future; new code on
  Pages Router needs a reason. Migration status if mixed.
- **Server Actions** for form submission vs API routes.
- **`generateMetadata`** for dynamic metadata vs old `<Head>`.
- **`<Image>`, `<Script>`, `next/font`** — using built-in optimization vs
  raw equivalents.
- **PPR (Partial Pre-Rendering)** if on canary / 15+.
- **Middleware** — minimal, edge-compatible.
- **`unstable_cache` / `revalidateTag`** for granular cache control.

### Vue 3

- **Composition API** — vs Options API for new code.
- **`<script setup>`** syntax.
- **`defineModel`** (Vue 3.4+).
- **Reactivity Transform** (deprecated, but flag if still used).

### Svelte 5

- **Runes** (`$state`, `$derived`, `$effect`) — vs reactive declarations.
- Check for migration progress if it's a Svelte 4→5 project.

### Astro

- **Content Collections** for typed content.
- **View Transitions** baked in.
- **Server Islands** (Astro 5+).

### Angular

- **Standalone components** (Angular 14+) — vs NgModules.
- **Signals** (Angular 16+).
- **New control flow** (`@if` / `@for`, Angular 17+) — vs `*ngIf` /
  `*ngFor`.

---

## Step 5: Bundle / build modernization

- **Build tool.** Vite / Turbopack / esbuild / Rollup / Webpack. Webpack
  isn't a finding by itself, but a slow Webpack project on a big codebase
  often benefits from migration.
- **ES module output.** ESM-only (`"type": "module"`) vs CJS legacy.
- **Browserslist tightness.** A `defaults` config pulls in IE11-era
  polyfills nobody needs. Consider tightening (`> 0.5%, not dead, not IE
  11, last 2 versions`) — impact on bundle size is real.
- **Polyfill bloat.** core-js shipping for browsers that have native
  support — measure.
- **Babel runtime.** Still needed if all targets support modern JS? Often
  not.
- **Code splitting at route + component level.**
- **Tree-shaking effectiveness** — barrel imports breaking it (cross-ref
  React best-practices skill).

---

## Step 6: Form patterns

- **Modern form libraries** — react-hook-form, formik (legacy now),
  TanStack Form, Conform, vanilla forms via Server Actions. The
  uncontrolled-components + Server-Action pattern is the new default for
  Next.js projects; controlled-state-everywhere is legacy.
- **Validation library** — Zod / Valibot / Yup / Joi. Schema-driven vs
  hand-rolled.
- **HTML form attributes** — modern `autocomplete` values
  (cross-ref module `02-compliance/accessibility.md`), `inputmode`,
  `enterkeyhint`.

---

## Step 7: Image / asset modernization

- **Image formats.** AVIF / WebP with fallback vs JPEG/PNG only.
- **Responsive images.** `srcset` / `sizes` / `<picture>` (or framework
  equivalent).
- **Lazy loading.** `loading="lazy"` (vs IntersectionObserver shims).
- **`fetchpriority="high"`** on LCP image.
- **Variable fonts.** Single file, multiple weights / styles vs many
  static font files.
- **`font-display: swap`** (or `optional`) — never `block`.
- **Subsetting.** Latin-only vs full Unicode for an English-only site.
- **`@font-face` size-adjust** to reduce CLS.

---

## Step 8: PWA / offline capabilities

If the project would benefit from PWA features (cross-ref module
`07-domain-conditional/product-types/pwa.md`):

- **Manifest** present?
- **Service worker** with sensible caching strategy?
- **`fetch` interception** for offline fallback?
- **Background sync** for queued mutations?

If the project clearly should be a PWA but isn't, flag.

---

## Red flags

- React project on 17.x in 2025+.
- Next.js Pages Router with no App Router migration plan on a project
  shipping new features.
- `moment` in client bundle on a new project.
- `axios` for trivial fetches in Node 18+ environments.
- `outline: none` everywhere (cross-ref accessibility).
- Browserslist `defaults` shipping IE11 polyfills.
- All-controlled forms with manual `useState` per field on a Next.js 14+
  project (Server Actions exist).
- `media (max-width)` queries everywhere when component-level container
  queries would be cleaner.
- `JSON.parse(JSON.stringify(obj))` for deep clone (`structuredClone`
  exists).
- `obj && obj.x && obj.x.y` (optional chaining exists since 2020).
- Self-rolled spinner / skeleton instead of `useTransition` for loading
  state.
- Custom modal scaffolding instead of `<dialog>` element.
- `padding-top: 56.25%` aspect-ratio hack instead of `aspect-ratio: 16/9`.

---

## Output structure

```markdown
# Frontend Modernization Audit

Date: YYYY-MM-DD
Repository commit: <hash>

## Stack
- Framework + version vs current stable
- CSS strategy
- Build target / browserslist

## Modern CSS opportunities
(checklist with specific component locations)
- Container queries
- :has()
- Logical properties
- @layer
- subgrid
- color-mix() / oklch
- :focus-visible
- aspect-ratio
- text-wrap: balance / pretty
- <dialog>
- view-transition / anchor-positioning (where supported)

## Modern JS opportunities
(checklist with specific code locations)
- async/await consistency
- structuredClone
- toSorted / toReversed / etc.
- Object.groupBy
- Optional chaining / nullish coalescing
- AbortSignal.timeout
- Intl.* over moment
- Native fetch / crypto.randomUUID

## Framework utilization
(per-framework checklist of current idioms — used / not used)

## Bundle modernization
- Build tool currency
- Browserslist tightness
- Polyfill bloat
- Code splitting

## Forms
- Library currency
- Validation library
- Modern form-element attributes

## Images / fonts / assets
- Format adoption
- Responsive images
- Variable fonts
- font-display

## PWA / offline (if applicable)

## Findings

## Final recommendation
Single highest-priority modernization action with expected impact.

## Decision summary
- Stack on current major: yes/no
- Major capabilities under-utilized: <count> with examples
- Worth acting on now: yes/no

## Confidence
High / Medium / Low

## Out of Scope / Inconclusive
```

---

## Severity calibration

Modernization findings rarely rise to Critical — they're improvements, not
defects. Calibrate accordingly:

- **High:** capabilities the project explicitly markets (e.g., RTL claimed
  but logical properties not used; PWA claimed but no service worker).
- **Medium:** clear quality-of-life wins (container queries, `:has()`,
  modern JS array methods, framework-current patterns).
- **Low:** polish, taste, micro-improvements.

The exception: a project on an EOL framework version (React 16, Vue 2)
escalates to High or Critical due to security-patch availability — but
those overlap with module `03-quality/dependencies.md`.

Every finding cites a file/line and the modern alternative.
