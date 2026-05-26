# 13 — Cost Analysis & Right-Sizing

Audit the codebase for **architectural-fitness mismatches that drive unnecessary
cost** — over- and under-engineered infrastructure relative to actual workload
and reliability requirements.

Create or update:

`/docs/audits/cost-audit.md`

---

## Posture: fitness, not minimization

This is not "make it cheap." It is "make the architecture commensurate with the
requirements." Over-engineering and under-engineering are both findings. A K8s
cluster running a single CRUD app is over-engineered; a single VM running a
mission-critical payments gateway is under-engineered.

**Anti-biases to internalize before starting:**

- **A boring "keep it" verdict is often the correct one.** If the architecture
  fits, do not invent a migration story. Reading code primed to find waste tends
  to find imaginary waste.
- **Do not recommend rewrites or platform migrations casually.** Targeted swaps
  (a single component, a deployment-target change) almost always beat full
  re-architectures.
- **Deployment fixes often beat code rewrites.** Before recommending application-
  level changes, ask: would changing the runtime, hosting model, or instance
  size solve this at lower risk?
- **Maintainer's knowledge is real.** A theoretically suboptimal stack the team
  knows cold beats a theoretically ideal stack they would have to learn.
- **Mark unknowns explicitly.** If you cannot estimate spend or scale from the
  code, say so and name what's missing.

### The solo-founder / small-team lens

Apply this lens unless the repo clearly shows a larger team (multiple
CODEOWNERS, dedicated infra folders, formal staging/preview environments,
ADRs, runbooks):

- Bias toward simplicity, fast iteration, and low operational overhead.
- Treat operational burden as a cost — engineering time is the most expensive
  resource a small team has.
- "Can one strong generalist run this?" is a core evaluation criterion, weighted
  alongside performance and reliability.
- If you find yourself recommending Kubernetes, a service mesh, event sourcing,
  CQRS, or a multi-service architecture for a project that looks solo-scale,
  stop and reconsider — the *current* setup may itself be the over-engineering.

If the repo shows clear larger-team signals, relax this lens proportionally,
but keep the bias toward simplicity.

This module shares posture with the standalone `stack-fit-audit` skill. For deep
language/runtime fit analysis, that skill goes further; this module focuses on
the cost-fitness slice.

---

## Step 0: Live Discovery

Cloud and SaaS pricing changes constantly: free-tier cliffs move, list prices
update, scale-to-zero behavior changes, new pricing models appear. Fetch
current truth before estimating spend deltas.

### Cited service pricing

For each meaningful vendor surfaced by Module 01.5 (Stack-Specific Brief),
fetch the current pricing page:

- **Vercel / Netlify / Cloudflare Workers / Fly.io / Railway / Render / Heroku** —
  Fetch the vendor's `/pricing` page. Capture: free-tier limits, paid-tier
  prices, scale-to-zero / always-on distinctions, egress fees.
- **AWS / GCP / Azure** — Fetch the specific service's pricing page
  (`/lambda/pricing`, `/run/pricing`, `/functions/pricing`, RDS pricing, etc.).
  For complex deltas (compute hours × region), recommend the user pull from
  AWS Cost Explorer / GCP Billing Export rather than estimate from list.
- **Database vendors** — Neon, Supabase, PlanetScale, Turso, Upstash, Cloudflare D1.
  Pricing for these often has scale-to-zero or per-request mechanics that
  shift quarterly: Fetch `<vendor>/pricing`.
- **Observability** — Datadog, Sentry, Honeycomb, Logtail, Better Stack: ingest
  pricing and per-host pricing both move.
- **LLM APIs** — Anthropic, OpenAI, Bedrock, Vertex per-token pricing.
  Cross-ref Module 18 Live Discovery (it fetches the same data).
- **Search / cache** — Algolia, Elasticsearch Cloud, Upstash Redis, Pinecone.

### Free-tier cliffs

Many fitness recommendations hinge on whether the workload fits a free tier.
The cliffs themselves move:

- Vercel Hobby vs Pro request limits
- Cloudflare Workers free tier request count
- Neon free DB hours
- Supabase free-project sleep behavior

Fetch the current limits before recommending "stay on free" or "must upgrade."

### Output of discovery

```markdown
## Live Discovery (fetched YYYY-MM-DD)

| Vendor | Tier referenced | Source | Discovered |
|---|---|---|---|
| Vercel | Hobby vs Pro | vercel.com/pricing | Hobby: <limit>; Pro: $20/seat/mo |
| Neon | Free tier | neon.com/pricing | Free: <CU-hours>; Launch tier: $X/mo at <CU> |
| Anthropic | Opus 4.7 input | docs.claude.com | $X/MTok input, $Y/MTok output, prompt cache: $Z |

Cost-fitness recommendations below are calibrated against this state.
```

If discovery fails (no network), note explicitly and mark cost estimates as
"reflects skill last-update — recommend re-running with network access."

---

## How to investigate

Walk each spend category. For each, do three things:

1. **Detect** what the codebase is using (cite config files).
2. **Assess fitness** against observed requirements (one of: **strong fit** /
   **good enough** / **acceptable but strained** / **poor fit / over-** or
   **under-engineered**).
3. **Estimate the spend delta** to a better-fitting alternative, where the code
   gives enough signal to estimate. If it doesn't, mark "unknown — needs
   billing data."

### Step 1: Compute

**Detect.** Vercel / Netlify / Cloudflare Workers / Fly.io / Railway / Render /
Heroku / AWS Lambda / GCP Cloud Run / Azure Functions / k8s (any flavor) / EC2 /
self-hosted VPS. Cite the deploy config (`vercel.json`, `wrangler.toml`,
`Dockerfile`, `helm/`, `terraform/`, etc.).

**Common over-engineering patterns:**

- **K8s for a single app or small service.** $300–$2k+/month in control plane,
  node baseline, observability stack, and engineering time. For most CRUD apps
  with < 100 RPS sustained, Cloud Run / Workers / Fly Machines / a VPS does the
  same work for $5–$200/month.
- **Multi-region active/active** when a single region with managed failover
  would meet RPO/RTO. Multi-region doubles infra cost and triples operational
  complexity.
- **Always-on containers** for low-traffic workloads where serverless
  scale-to-zero would fit. The reverse — serverless for sustained high traffic —
  is *under-engineered*; flag both directions.
- **Premium instance types** without a profiled justification (a `c6.4xlarge`
  for a low-CPU app).
- **Separate staging environment running 24/7** when ephemeral preview
  environments per PR would fit (Vercel/Railway/Fly all support this).

**Common under-engineering patterns:**

- Single-VPS deploys for revenue-critical paths with no failover.
- No autoscaling on a workload that needs it.
- Manual deploys for an app with frequent changes.

### Step 2: Database

**Detect.** Postgres flavor (managed Aurora / RDS / Supabase / Neon / Railway /
self-hosted), MySQL/MariaDB, MongoDB, DynamoDB, SQLite, PlanetScale, Turso,
Cloudflare D1, Firebase. Cite the connection string env var or config.

**Common over-engineering patterns:**

- **Aurora / Spanner / CosmosDB** for workloads that fit comfortably on a single
  Postgres instance. Aurora is ~2× the cost of standard RDS for negligible
  gain at the small end.
- **MongoDB / DynamoDB** for relational data that would fit better in Postgres.
  NoSQL is a tooling choice, not a default; relational data forced into
  document stores produces application-layer joins that cost both compute and
  developer time.
- **Postgres + Redis** when Postgres alone (with proper indexes and judicious
  `pg_cron` or LISTEN/NOTIFY) would meet the workload. Redis adds an
  always-on monthly bill and a second source of truth that drifts.
- **Per-environment managed DBs** for tiny side projects (a $20/mo managed
  staging DB for an app with no users yet).

**Common under-engineering patterns:**

- SQLite or `*.db` files for multi-writer production workloads.
- No replicas for a high-read app where read replicas would offload by 50%+.
- Single-region DB with users worldwide, manifesting as 200ms+ query latency.

### Step 3: Caching

**Detect.** Search for `redis`, `@upstash/redis`, `ioredis`, Vercel KV,
Cloudflare KV, in-memory `LRU`/`Map`, framework caches (Next.js `unstable_cache`,
SvelteKit hooks), CDN cache headers.

**Common over-engineering patterns:**

- **Redis (managed, always-on)** for caches that would fit in a per-process
  in-memory LRU with a few hundred MB. A small Upstash plan is $0–$10/month
  and reasonable; a managed Redis at $50–$200/month for a side project is
  not.
- **Cache layers stacked** (CDN → Redis → in-memory) without measured hit-rate
  data. Each layer adds invalidation complexity.
- **Cache for never-hot data.** A cache for queries that fire once per user
  per session has near-zero hit rate — net cost is positive.

**Common under-engineering patterns:**

- No caching anywhere on a high-RPS read-heavy app.
- Whole-page server rendering for content that's identical across users (should
  be ISR/SSG/edge-cached).

### Step 4: Queue / background jobs

**Detect.** BullMQ, Inngest, Trigger.dev, Hatchet, Temporal, AWS SQS+Lambda,
Cloud Tasks, Kafka, RabbitMQ, Vercel cron, GitHub Actions scheduled, custom
`setInterval`.

**Common over-engineering patterns:**

- **Kafka** when Redis Streams, SQS, or even Postgres-as-queue would do.
  Self-hosted Kafka is a part-time job in itself.
- **Temporal / Step Functions** for simple sequential jobs that don't need
  durable workflows.
- **Multiple queue systems coexisting** (BullMQ + Inngest + Vercel cron) where
  one would suffice — common as projects accrete features.

**Common under-engineering patterns:**

- Long-running tasks blocking request handlers (sending email, generating PDF
  in the request path).
- `setInterval` in a Node process pretending to be a scheduler (lost on
  restart, no observability).

### Step 5: Search

**Detect.** Algolia, Elasticsearch / OpenSearch, Meilisearch, Typesense,
Postgres FTS (`tsvector`, `to_tsquery`), pgvector, Pinecone, Weaviate.

**Common over-engineering patterns:**

- **Algolia / Elasticsearch** for product catalogs of < 100k items where
  Postgres FTS or pg_trgm would satisfy the requirement at $0 marginal cost.
  Algolia entry tier is ~$50/mo; Elasticsearch managed clusters start ~$100/mo.
- **Vector DB (Pinecone, Weaviate)** for tiny embedding sets (< 100k vectors)
  that fit in pgvector or a flat in-process index.

**Common under-engineering patterns:**

- `LIKE '%term%'` queries on large tables with no index — table scans.
- No relevance ranking on user-facing search.

### Step 6: AI / LLM

**Detect.** Search `package.json` / imports for `anthropic`, `openai`, `@ai-sdk/*`,
`langchain`, `cohere`, custom inference endpoints. Find which models are actually
called (`claude-opus-4-7`, `gpt-4o`, `claude-haiku-4-5-20251001`, etc.).

**Common over-engineering patterns:**

- **Opus / GPT-4** in a hot path where Haiku / GPT-4o-mini would be sufficient.
  10–60× the per-token cost.
- **Self-hosted inference** (a $500/mo GPU) for a workload doing < 1M tokens/day
  that an API would serve at < $50.
- **Large embedding models** (`text-embedding-3-large`, 3072-dim) when
  `text-embedding-3-small` (1536-dim) suffices — half the storage, similar
  retrieval quality for most use cases.
- **No prompt caching** when prompts have a stable system prefix > 1024 tokens.
  Anthropic prompt caching gives ~90% off cached input — usually a single
  config flip.
- **No batch API usage** for non-realtime workloads (50% off list price for
  Anthropic Batch / OpenAI Batch).
- **Streaming for batch / non-UX workflows.** No latency benefit, harder
  observability.

**Common under-engineering patterns:**

- Haiku where Opus is genuinely needed (complex reasoning, long context). The
  cheap model that fails the user costs more in the end.
- No retry / fallback when calls fail.

### Step 7: Monitoring & observability

**Detect.** Sentry, Datadog, New Relic, Grafana Cloud, Honeycomb, Logtail,
Better Stack, Axiom, OpenTelemetry, console.log.

**Common over-engineering patterns:**

- **Multiple overlapping observability stacks** (Datadog APM + Sentry + Grafana
  + custom Loki) ingesting the same events. Easy to accrete; rarely pruned.
- **Datadog at the small end** ($31/host/month + $0.10/GB log ingest scales
  alarmingly with no-op verbose logging).
- **Verbose application logging** in `info` mode for non-billable events
  driving observability bills.

**Common under-engineering patterns:**

- No error tracking at all on a production app.
- No structured logging (every grep is an ordeal).
- Health endpoint that always returns 200 (= no health check).

### Step 8: CDN & edge

**Detect.** Cloudflare, Fastly, Vercel Edge, AWS CloudFront, Bunny.net,
self-rolled.

**Common over-engineering patterns:**

- **CloudFront / Fastly** for a small site where Cloudflare's free tier covers
  the use case.
- **Edge functions for everything**, including endpoints that are slower at the
  edge (anything that talks to a single-region DB without a regional pooler).
  Module 11 covers this in depth — cross-reference there.

**Common under-engineering patterns:**

- No CDN at all in front of a high-traffic site.
- Static assets served from origin.

### Step 9: Email / SMS / comms

**Detect.** Postmark, Resend, SendGrid, SES, Mailgun, Twilio, Vonage, custom
SMTP.

**Common over-engineering patterns:**

- **SendGrid Premium / dedicated IPs** at low volumes. Resend or Postmark at
  the same volumes is cheaper and has better deliverability defaults.
- **Twilio** for transactional SMS at low volume in markets where it's
  expensive (some countries are $0.30+/SMS via Twilio).

### Step 10: Storage

**Detect.** S3, R2, B2, GCS, Cloudinary, UploadThing, blob storage in DB.

**Common over-engineering patterns:**

- **S3** when **R2** would fit (R2 has zero egress fees — significant savings
  for any user-facing media app).
- **Cloudinary / Imgix** at small scales when Next.js `<Image>` + R2/S3 with a
  CDN would do.

**Common under-engineering patterns:**

- Files stored as `bytea` / blob columns in Postgres at scale.
- No CDN in front of media.

### Step 11: SaaS subscriptions audit

Search the repo (and `package.json`) for SaaS API integrations. List them with
estimated monthly cost where the tier is documented or inferable. Look for:

- Multiple tools doing the same job (Mixpanel + PostHog + Amplitude).
- Tools that haven't been touched (no recent code referencing them).
- Tools paid annually mid-tier when free/Hobby would fit current scale.

---

## Red flags

- K8s manifests in a repo with one app and < 5 pods total.
- Postgres + Redis with negligible Redis hit-rate data.
- Aurora cluster with min 2× standby for an app at < 1 RPS sustained.
- Elasticsearch holding < 50k documents.
- Datadog APM enabled across all services with no `info` log discipline.
- Opus or GPT-4 used in a high-frequency path with no caching.
- Multi-region replication configured but only one region has traffic.
- Three or more queue systems coexisting.
- "Staging" Postgres at the same instance class as prod, running idle 24/7.
- A single-tenant container per user (massive overprovisioning for a SaaS).

---

## Output structure

```markdown
# Cost Analysis & Right-Sizing Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Posture

Solo-founder lens applied: yes / no — with rationale.
This audit may flag both over- and under-engineering.

## Spend Shape Inventory

| Category | Detected | Source (file) | Fitness | Est. monthly cost | Est. delta to better fit |
|---|---|---|---|---|---|
| Compute | Vercel + Hobby | vercel.json | Strong fit | ~$0 | — |
| Database | Aurora cluster (RDS) | terraform/db.tf:14 | Acceptable but strained | ~$280 | -$200 (single Postgres or Supabase Pro) |
| Cache | Redis (Upstash) | lib/redis.ts:1 | Poor fit (over) | ~$60 | -$60 (in-process LRU sufficient) |
| Queue | BullMQ + Vercel cron + Inngest | (3 files) | Poor fit (over) | ~$80 | -$60 (consolidate to one) |
| Search | Algolia | lib/search.ts:5 | Acceptable but strained | ~$50 | -$50 (Postgres FTS suffices for 12k items) |
| AI/LLM | Opus 4.7 in hot path | lib/agent.ts:42 | Poor fit (over) | ~unknown | -75–90% (Haiku 4.5 + prompt caching) |
| Monitoring | Sentry + Datadog APM | (2 files) | Acceptable but strained | ~$200 | -$150 (Sentry alone) |
| CDN | Cloudflare (free) | DNS | Strong fit | $0 | — |

(One row per detected category. Mark unknowns "—".)

## Findings

For each finding worth acting on:

### [Finding title — imperative phrasing]
- Severity: <Critical | High | Medium | Low>
- Category: <compute / db / cache / etc.>
- Evidence: <file:line>
- Current cost shape: <estimate>
- Recommended fit: <description>
- Estimated delta: <$X/month, with assumptions>
- Implementation risk: <Low / Moderate / High>
- Rollback story: <how to revert if it doesn't work>

## Improvement Tiers

### Tier 1 — Low-cost (this week)
Configuration flips, dependency removals, env-var changes. List each with
estimated delta.

### Tier 2 — Medium-cost (sprint or two)
Single-component swaps (Algolia → Postgres FTS; Redis → in-memory; Opus → Haiku
with prompt caching). List each with estimated delta and risk.

### Tier 3 — High-cost (architectural)
Platform migrations, deployment-model changes, database engine changes. Be
willing to say "none of these are worth doing right now."

## Final Recommendation

The single best cost-related action for this repository right now. **One thing.**

## Decision Summary

```
- Over-engineered components identified: yes / no
- Under-engineered components identified: yes / no
- Worth acting on now: yes / no
- Single biggest delta: $X/month from <action>
- Recommended posture: <keep / target Tier 1 / target Tier 2 / consider Tier 3>
```

## Confidence

- **High** — I had clear evidence of usage patterns, scale, and current costs.
- **Medium** — Some inference from code shape, billing data not available.
- **Low** — Estimates rely on assumptions about traffic and scale that I could
  not verify from the repo.

## Out of Scope / Inconclusive

What this audit could not determine without billing data, real traffic
profiles, or production logs. Be explicit.
```

---

## Severity & fitness calibration

**Severity** here is about money + risk:

- **Critical:** clear runaway spend on infra the workload doesn't justify
  ($1k+/month delta vs a fit-for-purpose alternative); or under-engineering
  that risks outage on revenue-critical paths.
- **High:** $200–$1k/month delta; or under-engineering that imposes meaningful
  ops burden.
- **Medium:** $50–$200/month delta, or quality-of-life cost wins.
- **Low:** small wins, polish, or defer-able.

**Fitness buckets** per spend category:

- **Strong fit:** the choice actively suits the workload.
- **Good enough:** no real friction; keeping it is correct.
- **Acceptable but strained:** works, but creates cost or operational friction.
- **Poor fit / over-engineered:** complexity and cost exceed the workload's
  needs.
- **Poor fit / under-engineered:** the workload is straining the choice.

---

## Evidence requirements

Every finding cites a config file or import. Every cost estimate names its
assumptions ("assumes 100 RPS sustained, 50/50 read/write, current Aurora
db.r6g.large at on-demand pricing"). Where you cannot estimate from code,
mark the cost as "unknown — needs billing data" and recommend the first step
to get that data (e.g., "request 30 days of AWS Cost Explorer per-service
breakdown").

A finding without a $ estimate (or a clearly-marked "unknown") is not
actionable. Reject it and gather more evidence or escalate to the maintainer
with what's missing.

---

## What this module is *not*

- Not a contract review (procurement / negotiation lives elsewhere).
- Not a performance audit (module 09 covers DB/scaling; module 10 covers
  perceived speed; module 11 covers cold-start risk).
- Not a stack-fit verdict (the standalone `stack-fit-audit` skill goes
  deeper on language/runtime fit). If a finding here is really a stack-fit
  question, note it and recommend running that skill.
