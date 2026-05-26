# 09 — Performance & Scalability Audit

Analyze the system for performance and scalability risks.

Create or update:

`/docs/audits/performance-audit.md`

---

## How to investigate

Performance audits go wrong when they devolve into "use a CDN!" recommendations
without grounding. Tie every finding to a specific code location and a specific
load profile (current vs realistic future). Avoid recommending caching for routes
that aren't actually hot.

If the project uses a serverless or edge runtime with a connection-limited DB,
**Module 11 (`references/04-performance/serverless-coldstart.md`) covers the
cold-start and caching analysis in much more depth**. Defer to it for those
concerns; cover the rest here.

### Step 1: Identify the rendering model

This determines which perf class the app falls into.

- **Next.js App Router:** server components by default; check for `'use client'`
  pragmas; check for `force-dynamic`, `revalidate`, `generateStaticParams`.
- **Next.js Pages:** `getStaticProps` / `getServerSideProps` / `getStaticPaths`.
- **SvelteKit / Remix / Solid Start:** loaders run on server; client hydrates.
- **Pure SPA (CRA, Vite + React):** all rendering on client; data fetched
  client-side.
- **MPA / SSR (Rails, Django, Laravel):** server-rendered HTML; check for
  asset pipeline & caching.

The rendering model dictates where the perf bottleneck will live: SPA → JS bundle
size & client memory; SSR → server compute & DB queries; static → build time.

### Step 2: Database query patterns (usually the biggest win)

For each significant route or API handler:

- **Per-render query count.** How many DB queries fire per request? > 5 is
  suspicious; > 20 is almost always a problem.
- **N+1 patterns.** Search for `.map(async ... db.findUnique(...))`,
  `for ... await db.query(...)`. Loop + DB call = N+1.
- **Missing indexes.** Check `WHERE` clauses against schema indexes. ORM-generated
  queries often miss indexes that the schema does not declare.
- **Unbounded queries.** `findMany()` without `take`/`limit`. On a 1M-row table
  this is a request-killer.
- **Joins vs sequential fetches.** Prisma `include` and Drizzle relations are
  joins under the hood; manual sequential fetches multiply round trips.
- **Connection model.** New connection per request (bad for serverless), pool,
  or HTTP-based driver (e.g., Neon serverless). Defer detail to module 11.

### Step 3: API request patterns (client-side)

For SPAs and frameworks with significant client-side fetching:

- **Waterfall fetches.** Component A renders → fetches → renders B → fetches.
  React Suspense + parallel data libraries (React Query, SWR) or framework
  loaders eliminate these. Check for nested `useEffect(fetch)` chains.
- **Refetch storms.** SWR/React Query with aggressive `refetchOnWindowFocus` or
  `revalidateOnReconnect` — usually fine, but flag if the data is expensive.
- **Polling without cleanup.** `setInterval` in a component without cleanup
  in the return / `useEffect` cleanup. Memory leak + load multiplier.

### Step 4: Frontend rendering

- **Bundle size.** Search for `webpack-bundle-analyzer`, `next/bundle-analyzer`,
  `rollup-plugin-visualizer`. If never used, recommend running it.
- **Heavy dependencies.** Check `package.json` for known-heavy libraries:
  `moment` (vs `date-fns` / `dayjs`), full `lodash` (vs `lodash-es` cherry-pick),
  `xlsx`, `pdfkit`, `chart.js` + `chartjs-plugin-*`, `three`, `monaco-editor`.
  Are they dynamically imported, or shipped to every user?
- **Code splitting.** Routes split? Heavy modals dynamically imported?
- **Re-render hot spots.** Components that re-render too often:
  - Context with frequently-changing values used by many components.
  - `useMemo`/`useCallback` dependencies that change every render.
  - Object/array literals passed as props (cause child re-render).
- **List virtualization.** Long lists rendered without virtualization
  (`react-window`, `@tanstack/react-virtual`)? > 100 items = should virtualize.
- **Image optimization.** Raw `<img>` for user-uploaded images vs `next/image`,
  `<Image>` from `astro:assets`, etc. WebP / AVIF? Lazy loading?

### Step 5: Server-side compute

- **Synchronous CPU-bound work in request path.** Image processing, PDF
  generation, ZIP unpacking, JSON parsing of huge payloads — these block the
  event loop in Node and serialize requests. Should be in a worker / queue.
- **Synchronous fs I/O.** `fs.readFileSync` in handlers (not at module load).
- **External API calls without timeouts.** `fetch(externalUrl)` with no `signal:
  AbortSignal.timeout(...)`. A slow upstream becomes app-wide latency.
- **Crypto in request path.** bcrypt/argon2 are intentionally slow (good for
  passwords, bad if called per request unnecessarily).

### Step 6: Caching layers

- **HTTP caching.** Are responses sending `Cache-Control` headers? `s-maxage` for
  CDN? `stale-while-revalidate`?
- **Application caching.** Redis / Upstash / Vercel KV / Cloudflare KV in use?
  What is cached? How is invalidation handled?
- **Framework caching.** Next.js fetch cache (App Router), `unstable_cache`,
  `revalidateTag`. SvelteKit load cache.
- **CDN.** Static assets on a CDN, or served from origin?

### Step 7: Memory & process model

- **Globals that grow.** Module-level `const cache = new Map()` without eviction.
  Memory leak in long-running processes; reset every cold-start in serverless.
- **Event listener leaks.** `useEffect` adding listeners without removing.
- **Streams without backpressure.** Reading a large file or response without
  pipe / streaming consumer.

### Step 8: Scalability profile

For each major user-facing route, walk through what happens at:

- 100 concurrent users
- 1,000 concurrent users
- 10,000 concurrent users
- 1M total records (if data scales with users)

Identify the *first* bottleneck that breaks. Often it is connection pool
exhaustion long before CPU. Cite the evidence: "at 1k concurrent, the unbounded
`findMany` in `app/dashboard/page.tsx:34` would scan the full `events` table —
currently small but grows linearly with usage."

---

### Step 9: Concurrency, idempotency, and race conditions

Most scalability bugs surface as concurrency bugs. Most concurrency bugs
surface only at scale. Audit for the patterns that hide:

#### Locking strategies

- **Optimistic locking.** Version columns (`updated_at` or `version`) on
  records, with `WHERE version = ?` in updates. Reasonable for low-contention
  resources; loses on hot rows.
- **Pessimistic locking.** `SELECT ... FOR UPDATE` — explicit row locks.
  Necessary for high-contention mutation flows.
- **Advisory locks** (Postgres `pg_advisory_lock`) — for cross-row /
  cross-table coordination (e.g., one worker per scheduled job).
- **Distributed locks** (Redis SETNX, Redlock) — for cross-process
  coordination. Each has known correctness caveats; use a battle-tested
  library, not hand-rolled.

Search for explicit locking and assess fit. Hand-rolled "if (busy) skip"
patterns are usually broken under contention.

#### Idempotency for mutations

- **Idempotency keys for non-API mutations.** Webhook receivers (Stripe,
  Shopify, etc.), queue jobs, retries — each invocation should be safe.
  Store the key + result; re-invocations return the stored result.
- **Database upsert vs insert-then-handle-conflict.** `ON CONFLICT DO
  UPDATE` is atomic; `INSERT` + `try/catch unique-violation` + `UPDATE` is
  a TOCTOU race.
- **Deduplication windows.** When two requests arrive simultaneously with
  the same logical operation, do both succeed-and-create-duplicates, or is
  the second blocked / merged?
- **Webhook retry semantics.** Stripe retries failed webhooks; receiver
  must be idempotent. Search for `event.id` deduplication checks before
  processing.

#### Race conditions in handlers

- **Read-modify-write atomicity.** Pattern:
  ```ts
  const x = await db.x.find(...);
  await db.x.update({ ...x, count: x.count + 1 });
  ```
  Two concurrent runs each read 5, both write 6 — should be 7. Fix:
  `UPDATE x SET count = count + 1` or transaction with row lock.
- **TOCTOU (time-of-check-time-of-use).** Check permission → act on
  resource — between check and act, permission may have changed.
- **Double-submit prevention.** Click-the-button-twice flows — front-end
  debounce *and* server-side idempotency.
- **Duplicate signups / votes / purchases.** UNIQUE constraints + idempotent
  insert.

#### Eventual consistency awareness

- **Read-your-writes.** After a write to a primary, an immediate read may
  hit a replica that hasn't replicated. UI patterns: optimistic update,
  cache-busting, "stickier" routing post-write.
- **Cross-system consistency.** "Wrote to DB, then enqueued job; what if
  the enqueue fails?" — outbox pattern (write event to DB in same
  transaction; separate worker publishes).

#### Long-running operations

- **Backfill jobs.** Idempotent so they can be paused / resumed. Progress
  tracking (last-processed-id). Bounded batch size to avoid lock storms.
- **Bulk operations.** UI flows that "delete N items" — atomic transaction
  vs partial-success reporting.

#### Detection patterns

```
# Flag the read-modify-write anti-pattern:
grep -rn "find.*then.*update" src/ # rough; investigate hits

# Look for explicit locking usage:
grep -rn "FOR UPDATE\|advisory_lock\|SETNX" src/

# Look for idempotency-key handling:
grep -rn "Idempotency-Key\|idempotency_key\|event\.id" src/
```

A project handling money or sending notifications without explicit
idempotency / locking is one outage away from a customer-impact bug.

---

## Red flags

- `findMany()` / `SELECT *` without limits.
- N+1 loops (`for ... await db.find...`).
- `setInterval` in components without cleanup.
- Polling at < 5s intervals with full payload.
- Synchronous `fs.readFileSync` / `crypto.pbkdf2Sync` in request handlers.
- `useMemo` over a primitive expression (no benefit, adds work).
- Large lists without virtualization.
- Raw `<img>` for user-uploaded images.
- Heavy libraries (`moment`, full `lodash`) in client bundle.
- No DB indexes on foreign keys used in `WHERE` / `ORDER BY`.

---

## Output structure

```markdown
# Performance and Scalability Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Current Performance Characteristics
- Rendering strategy (SSR / SSG / SPA / hybrid — cite evidence)
- API request patterns
- Database query patterns

## Performance Risks
For each: severity, location (file:line), impact, fix.
- Inefficient database queries
- N+1 query patterns
- Large payload responses
- Unnecessary re-renders
- Blocking operations
- Large memory allocations

## Scalability Analysis
Walk through projected behavior at 100 / 1k / 10k concurrent users. Identify
first bottleneck.

## Frontend Performance
- Bundle size (recommend an analyzer run if never done)
- Code splitting
- Heavy dependencies
- Image optimization
- List virtualization
- Re-render hotspots

## Server-Side Compute
- CPU-bound work in request path
- External API timeouts
- Sync I/O

## Caching Strategy
- HTTP caching
- Application caching (Redis / KV)
- CDN
- Framework caches

## Optimization Opportunities
Prioritized list — each item: severity, expected impact, effort, evidence
pointer. Rank by impact-to-effort ratio.

## Final Recommendation
The single highest-impact scalability action for this codebase right now.
**One thing.**

## Decision Summary

- Current scale headroom: <comfortable | tight | already-stressed>
- First bottleneck under projected load: <DB | compute | network | other>
- Recommended posture: <keep | targeted fix | bigger architectural shift>

## Confidence
High / Medium / Low (and why)

## Out of Scope / Inconclusive
Items that need real load testing or production traces — note here, do not
guess.
```

---

## Severity calibration

- **Critical:** route is unusable at current scale, or will be at expected
  near-term scale (broken pagination on a growing table; unbounded query that
  will OOM).
- **High:** noticeable degradation at typical use (Time-to-Interactive > 5s on
  3G; queries > 1s on common pages).
- **Medium:** suboptimal patterns that will matter at scale (missing indexes
  on tables likely to grow, unmemoized expensive contexts).
- **Low:** micro-optimizations with little real-world impact at current scale.

---

## Evidence requirements

Every finding cites a file path and line. Every recommendation includes a
plausible impact estimate (e.g., "removing the N+1 in `app/feed/page.tsx:28`
would reduce queries from O(N) to O(1) for feed loads"). Avoid generic
recommendations like "add caching" without saying *what* to cache and *why
that data is hot*.
