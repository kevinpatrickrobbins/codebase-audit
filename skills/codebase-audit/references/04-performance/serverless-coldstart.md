# 11 — Serverless Database & Cold-Start Audit

Analyze the system for serverless database cold-start risk and compute cost exposure.

This module applies when the project uses a serverless or edge deployment platform
(Vercel, Netlify, Cloudflare Workers, Fly.io, etc.) combined with a serverless or
connection-limited database (Neon, PlanetScale, Supabase, Turso, Upstash, D1, etc.).
If neither is present, note that in the report and skip this module.

Create or update:

/docs/audits/serverless-db-coldstart-audit.md

Structure the report as follows.

## Report header

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Stack Identification

Identify:

- Deployment runtime (serverless functions, edge functions, containers, or hybrid)
- Database driver and connection model (HTTP-based, TCP pooled, WebSocket, embedded)
- Platforms detected (framework, hosting provider, database provider)
- Whether a connection pooler is in use (PgBouncer, Prisma Accelerate, Hyperdrive,
  built-in pooling, etc.)

## Route Rendering Classification

For each application route, identify:

- Rendering mode: static (SSG/prerendered), on-demand (SSR/server-rendered), or edge
- For static routes: whether static params are generated at build time
  (generateStaticParams, getStaticPaths, prerender, or equivalent)
- For dynamic routes: whether they are force-dynamic or conditionally dynamic
- Approximate entity count or data volume for each dynamic route (e.g., number of
  records that drive the route)
- Whether user-specific data prevents static generation

Produce a table: Route | Mode | Static Params | Entity Count | Notes

## Revalidation and Cache Strategy

For each route and API handler, document:

- Cache lifetime or revalidation interval (revalidate, Cache-Control,
  stale-while-revalidate, or equivalent)
- On-demand revalidation endpoints: what triggers them, what scope they invalidate
  (page vs layout vs full path tree)
- Framework-level data cache usage (Next.js fetch cache, SvelteKit load cache,
  Nuxt useAsyncData, Astro content collections, etc.)
- Whether cache scope mismatches exist (e.g., invalidating a layout when only a
  leaf page changed, causing unnecessary recomputation across many paths)

## Database Query Inventory

For each route type, count and classify queries:

- Total queries fired per page render
- Queries that return shared/global data (same result for all users/requests) vs.
  user-specific or request-specific data
- Queries inside loops or conditionals that may multiply at runtime (N+1 patterns)
- Queries that cross multiple tables (joins vs. sequential round trips)
- Connection acquisition overhead: does the driver open a new connection per request
  or reuse a pool?

Produce a table: Route | Queries/Render | Shared Queries | User-Specific Queries | N+1 Risk | Connection Model

## API Handler Analysis

For each API route or server action:

- Is the handler stateless and cold-startable, or does it rely on persistent
  in-memory state?
- What Cache-Control or CDN directives are set, if any?
- Is there any caching layer between the handler and the database (in-memory, KV,
  CDN)?
- Does the handler perform mutations that must bypass cache?
- Is it deployed as an edge function or a regional serverless function?

## Cold-Start and Compute Cost Exposure

Evaluate the overall cold-start risk profile:

- Frequency of cold starts: how often do dynamic routes force a new compute instance
  and database connection?
- Connection saturation risk: can concurrent cold starts exhaust database connection
  limits?
- Estimated query volume under representative load (requests/day × queries/request)
- Idle cost: does the database pause/scale-to-zero between requests, and what is the
  wake latency?
- Regions: are functions and database co-located? Is there cross-region latency?

## Caching Opportunity Analysis

Identify queries and responses that are candidates for caching:

- Shared data queries: list each one, the data it returns, and how frequently it
  changes
- Recommended cache layer for each: framework data cache, platform KV (Vercel KV,
  Cloudflare KV, Upstash), self-managed Redis, or CDN edge caching
- Suggested TTL for each candidate, based on data volatility
- Cache invalidation strategy: time-based expiry, on-demand purge, or event-driven
- Estimated reduction in database connections and compute cost if caching is applied
- For managed KV/Redis: estimate daily command volume (GET + SET per cached query ×
  estimated requests/day); compare against free-tier limits; conclude whether the
  operational complexity is justified

## Cache invalidation correctness

Beyond "is caching configured?" — does it behave correctly under load.

- **Cache stampede / dogpile.** When a cached value expires, multiple
  concurrent requests miss the cache, all hit origin, all write the new
  value. At scale this can saturate the DB. Mitigations:
  - Probabilistic early refresh (`stale-while-revalidate` semantics)
  - Lock around regeneration (one request regenerates; others wait or
    return stale)
  - Background refresh on a schedule
- **Race conditions on warm.** First request "warming" the cache while a
  second request also warms it — both write, last write wins, no
  consistency guarantee.
- **Stale-while-revalidate semantics.** Verify the chosen platform's
  behavior matches expectations: stale content served while background
  refresh runs (not blocking first user). Check Next.js `revalidate`,
  Vercel ISR, Cloudflare cache rules.
- **Tag-based vs path-based invalidation precision.** Invalidating
  too-broad a scope (e.g., `revalidatePath('/')` when only one page
  changed) re-renders many pages unnecessarily — multiplies origin load
  on update. Use tags (`revalidateTag`) for granular invalidation where
  available.
- **Cross-region cache consistency.** Multi-region edge caches may show
  different values to users in different regions during invalidation
  rollout — document the eventual-consistency window.
- **Cache key collisions.** Cache keys derived from URLs may collide if
  query params or headers vary in ways the key doesn't capture. Search
  for cache-key construction; verify it includes all variance dimensions
  (auth state, locale, A/B variant).

---

## Prioritized Recommendations

List recommendations in descending order of impact-to-effort ratio:

- Static generation opportunities: routes that could be prerendered or ISR-revalidated
  instead of fully on-demand
- Shared query caching: highest-traffic queries returning non-user-specific data
- Revalidation scope fixes: overly broad invalidation paths that trigger unnecessary
  recomputation
- Connection pooling: whether a pooler is needed, and which option fits the stack
- KV or Redis adoption: whether connection/compute volume justifies a managed cache,
  with estimated command volume and conclusion on ROI
- Region alignment: co-locating compute and database to minimize cold-start latency

For each recommendation: Effort (Low / Medium / High) | Impact (Low / Medium / High) | Change type (code / config / platform)

## Final Recommendation

The single highest-impact serverless / cold-start action for this codebase right
now. **One thing.**

## Decision Summary

- Stack actually exposed to cold-start risk: yes / no / partial
- Worst exposure: <connection storm | per-request driver init | over-broad invalidation | other>
- Recommended posture: <keep | targeted pooler / cache add | bigger rendering shift>

## Confidence

High / Medium / Low (and why)

## Out of Scope / Inconclusive

Items that need real production traces, vendor billing data, or load testing to
confirm.
