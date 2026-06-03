# 21 — SEO & Discoverability

Audit a public-facing web product for search-engine discoverability,
structured-data quality, social-share readiness, and crawlability.

Output: `/docs/audits/seo-audit.md`

---

## When to apply

Run on public-facing web products: marketing sites, content sites,
e-commerce, SaaS marketing pages, documentation sites. Skip for purely
authenticated B2B SaaS / internal tooling — SEO doesn't apply.

---

## Cross-module relationship

- **Speed** (Module 10) — Core Web Vitals are a ranking signal.
- **Frontend Modernization** (Module 17) — modern SSR helps
  crawlability.
- **Accessibility** (Module 04) — alt text and semantic HTML are SEO
  signals too.
- **i18n** (Module 20) — hreflang for multi-language sites.
- **Web Metadata skill** — sibling skill on Schema.org / JSON-LD; pair
  with this module for deep structured-data work.

---

## Step 0: Live Discovery

Search-engine guidance changes; structured-data vocabulary expands; AI
search reshapes discoverability assumptions.

- **Schema.org vocabulary:** Fetch https://schema.org/docs/full.html
  for current types.
- **Google Search guidance:** Fetch
  https://developers.google.com/search/docs — current crawler behavior,
  rich-result eligibility, indexing rules.
- **Core Web Vitals thresholds:** Fetch https://web.dev/articles/vitals
  for current targets (LCP, INP, CLS thresholds occasionally update).
- **AI search citation:** Search "AI search optimization 2026"
  for current best practice — Google AI Overviews, ChatGPT browsing,
  Perplexity, etc. treat content discoverability differently than
  classic search.
- **Sitemap protocol:** Fetch https://www.sitemaps.org/protocol.html
  for current spec.

---

## Step 1: Crawlability fundamentals

- **`robots.txt`** present, well-formed, not blocking what should be
  indexed. Cite the file.
- **`sitemap.xml`** present, current, referenced from `robots.txt`,
  ideally submitted to Search Console (often not verifiable from code;
  flag).
- **Server rendering vs client-only.** SPAs that render content
  client-side may not be crawled by all engines (Google handles it; many
  smaller engines and some AI agents don't). Server-render content meant
  to be indexed.
- **Static generation** for content pages where applicable — fastest +
  most cacheable.
- **Indexability** — search HTML for `<meta name="robots"
  content="noindex">` on pages that *should* be indexed. Common bug:
  `noindex` left from staging.
- **Rendered HTML quality.** Test by viewing the HTML response (not the
  hydrated DOM) — does it contain the content? Or just the JS shell?

## Step 2: Meta tags

For each significant page type:

- **`<title>`** — descriptive, unique per page, < 60 chars typical.
- **`<meta name="description">`** — 150–160 chars, distinct per page.
  Long-form pages benefit from custom descriptions; many sites repeat the
  site-wide tagline (anti-pattern).
- **Canonical URL** — `<link rel="canonical">` to prevent duplicate-content
  issues.
- **Robots meta** appropriate (index / noindex, follow / nofollow).
- **Viewport meta** correct (cross-ref accessibility — `user-scalable=no`
  is also an SEO signal of poor mobile experience).

## Step 3: Open Graph & Twitter Cards

- **OG tags** — `og:title`, `og:description`, `og:image`, `og:type`,
  `og:url`. Missing OG = social shares look broken.
- **Twitter Card** tags — `twitter:card`, `twitter:title`, etc.
- **Image dimensions** — Twitter / OpenGraph have specific recommended
  sizes (1200×630 typical, with separate 1200×600 for Twitter summary).
- **Per-page customization** vs site-wide defaults — content pages
  benefit from per-page OG image; settings page can fall back to default.
- **Generated OG images** — Vercel `next/og` / similar for dynamic OG
  per content piece.

## Step 4: Structured data (Schema.org / JSON-LD)

- **JSON-LD blocks** in HTML — `<script type="application/ld+json">`.
- **Type appropriateness.** Article / Product / Organization / Person /
  WebSite / BreadcrumbList / Event / Recipe / FAQPage / HowTo /
  SoftwareApplication / etc.
- **Required + recommended properties** populated per type.
- **Validation** — Google Rich Results Test (cite if used in CI),
  schema.org validator.
- **Sibling skill: `web-metadata`** has the depth here — defer to it for
  detailed Schema.org work; use this module to flag presence/absence and
  high-level alignment.

## Step 5: URLs

- **Human-readable slugs** (`/blog/how-to-x`) vs opaque IDs (`/p/8a3f9b2c`).
- **Trailing slash consistency.** One way, redirect the other.
- **Lowercase consistency.**
- **301 redirects from old URLs** if migration history exists — old URLs
  shouldn't 404 if they were ever indexed. Cite the redirect config.
- **Hreflang** for multi-language sites (cross-ref i18n module).
- **URL parameters** for tracking (`utm_*`) — canonical excludes them?
- **Pagination URLs** — `?page=2` indexable or `noindex`?

## Step 6: Content

- **`<h1>`** unique per page, descriptive.
- **Heading hierarchy** (cross-ref accessibility module).
- **Image alt text** — SEO signal *and* accessibility (cross-ref).
- **Internal linking** — content cross-linked appropriately? Orphan pages
  (no internal links pointing in) hurt indexing.
- **Anchor text descriptive** vs "click here".
- **Content depth** — too-thin pages (< 200 words) tend to underperform.
- **Duplicate content** within the site — boilerplate-heavy pages with
  small unique content blocks.
- **Author / date metadata** for articles (E-E-A-T signals).

## Step 7: Technical

- **Core Web Vitals** (cross-ref Module 10) — major ranking factor.
- **Mobile-friendly** rendering (cross-ref Module 04, 10, 15).
- **HTTPS** everywhere — non-HTTPS resources block ranking.
- **HTTP/2 or HTTP/3** at edge.
- **Image optimization** for crawl budget — large images slow crawl
  beyond hurting users.
- **JS bundle size** — large bundles slow first-meaningful-paint, hurts
  rankings.
- **404 handling** — proper `404 Not Found` status (not a soft 200 with
  "Not found" content). Soft-404 hurts crawl budget.

## Step 8: Discoverability beyond classic search

Modern discoverability isn't only Google ranking:

- **AI search citation eligibility** — clean structured data, clear
  authority signals, RSS / atom feeds for content sites. Google AI
  Overviews / ChatGPT browsing / Perplexity preferentially cite
  well-structured sources.
- **Social previews** — OG image quality, accurate descriptions.
- **App search** — App Store / Play Store metadata if also a mobile app
  (cross-ref Module 19).
- **Voice search** — long-tail conversational queries; FAQ / How-To
  schema helps.
- **In-product search** — site search (Algolia DocSearch, Pagefind,
  Postgres FTS) ≠ external search but affects on-site discoverability.

## Step 9: Monitoring

- **Search Console / Bing Webmaster Tools** verification meta tag in
  HTML, or DNS-verified.
- **Analytics** with search-traffic segmentation (organic vs direct vs
  referral).
- **Indexed page count** monitored over time.
- **Sitemap submission status** monitored.
- **Rank tracking** — Ahrefs, Semrush, etc. — outside the codebase, but
  if referenced in docs/runbooks, note the integration.

---

## Red flags

- `noindex` on production pages (left from staging or wrong default).
- `robots.txt: Disallow: /` in production.
- All pages share the same `<title>`.
- Missing OG tags — social shares look broken.
- No JSON-LD on a content site / product page / event page.
- SPA renders all content client-side with no SSR or SSG fallback.
- No 301 redirects from old URLs after a site rebuild.
- Hreflang missing on multi-language site.
- All H1s the same site-wide (often bad theme defaults).
- 404 page returns 200 status (soft 404).
- Heavy JS bundle on marketing pages that should be near-static.

---

## Output structure

```markdown
# SEO & Discoverability Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Live Discovery (fetched YYYY-MM-DD)
(table)

## Crawlability
- robots.txt / sitemap.xml
- Indexability per page type
- Render strategy (SSR / SSG / CSR) — rendered HTML quality

## Meta tags
- title / description / canonical / robots / viewport

## Social
- OG / Twitter Card / image dimensions / generated OG images

## Structured data
- JSON-LD coverage per content type
- Validation status

## URLs
- Slug quality / trailing-slash / case / 301s / hreflang / pagination

## Content
- Heading hierarchy / alt text / internal linking / depth / E-E-A-T

## Technical
- Core Web Vitals (cross-ref Module 10)
- Mobile / HTTPS / asset optimization / 404 status

## Beyond search
- AI search / social / app stores / voice / in-product

## Monitoring
- Search Console / analytics / indexed-pages tracking

## Findings

## Final recommendation

## Decision summary
- Site indexable: yes/no
- Major gaps: <list>

## Confidence: H / M / L

## Out of Scope / Inconclusive
(off-page SEO — backlinks, brand authority — not auditable from the
codebase)
```

---

## Severity calibration

- **Critical:** site is not indexable (`noindex` global, `robots.txt`
  blocks all, SPA with no rendered HTML); 301 chains broken from major
  prior URLs.
- **High:** missing OG / structured data on content meant to be
  shareable; duplicate titles / descriptions across pages; soft-404 on
  not-found.
- **Medium:** thin content; weak internal linking; missing hreflang on
  multi-language site.
- **Low:** style polish; minor metadata issues.

Every finding cites a URL or file path.
