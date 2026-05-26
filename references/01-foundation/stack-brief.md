# 01.5 — Stack-Specific Brief

Compile a current, audit-time brief of vendor-specific gotchas,
configuration pitfalls, and operational best practices for every meaningful
third-party service the codebase uses. The brief is consumed by all
downstream finding-producing modules to give them stack-specific calibration
without the maintenance burden of static vendor sub-references.

Output: `/docs/audits/stack-brief.md`

---

## Why this exists

Generic audits miss vendor-specific failure modes ("WorkOS callbacks need
state-parameter CSRF defense", "Cloudflare KV writes are eventually
consistent", "Neon scale-to-zero charges per minute of activity, not per
request"). Static vendor reference files rot rapidly — vendor docs evolve
continuously, new advisories ship, SDK behaviors change.

This module's solution: **compile per-vendor briefs at audit-time from
authoritative live sources**. The brief is current as of the audit date,
not the skill's last update.

| Static vendor sub-references | Compiled brief (this approach) |
|---|---|
| Maintenance: many files to keep current | Maintenance: zero (live fetch each audit) |
| Stale by skill date | Current as of audit date |
| Limited to vendors we've pre-written for | Works for any vendor with discoverable docs |
| Skill grows with each new vendor | Skill stays the same size |
| Token cost: load only relevant sub-ref | Token cost: fetch latency per audit |

Trade-off: each audit pays fetch latency (~1–3 minutes for a typical SaaS
with 8–10 vendors). Worth it for current findings.

---

## When to apply

Always — every codebase has at least some third-party services. If
somehow none are present, output a minimal brief noting that and the
downstream modules continue with generic checks.

---

## Step 1: Read Module 01's vendor inventory

Module 01 (System Architecture) Step 7 detects external integrations.
Read that section's output as the vendor list.

If Module 01 hasn't run yet, do a quick parallel detection by category:

| Category | Detection signals |
|---|---|
| **Auth** | `workos`, `@clerk/`, `@auth0/`, `@supabase/auth-*`, `better-auth`, `lucia-auth`, `next-auth`/`@auth/*`, custom JWT lib |
| **Hosting / runtime** | `wrangler.toml` → Cloudflare Workers; `vercel.json` → Vercel; `netlify.toml` → Netlify; `fly.toml` → Fly.io; `railway.json` → Railway; `Dockerfile` → container; `serverless.yml` → AWS Lambda |
| **Database** | `@neondatabase/*` → Neon; `@supabase/supabase-js` → Supabase; `@planetscale/database` → PlanetScale; `@libsql/client` → Turso; `wrangler` D1 binding → Cloudflare D1; `prisma`/`@prisma/client` (look at provider in schema) |
| **Storage** | `@aws-sdk/client-s3` (check region/endpoint for R2/S3/B2/etc.); `cloudinary`; `uploadthing`; `@cloudflare/r2` |
| **Payments** | `stripe`, `@paddle/*`, `@lemonsqueezy/*`, `polar` |
| **Observability** | `@sentry/*`, `datadog`, `posthog-js`/`posthog-node`, `logrocket`, `@axiom/*`, `@logtail/*` |
| **Email / SMS** | `resend`, `postmark`, `@sendgrid/mail`, `mailgun.js`, `twilio`, `vonage` |
| **LLM** | `@anthropic-ai/sdk`, `openai`, `@ai-sdk/*`, `langchain`, `@cohere-ai/*`, `replicate` |
| **CDN / edge** | platform-implicit (Cloudflare, Vercel Edge, AWS CloudFront refs in config) |

Filter to vendors with operational, security, or cost implications. Skip
trivia (lodash, zod, framework deps unless platform).

---

## Step 2: For each vendor, fetch and compile

For each detected vendor, perform discovery in parallel where possible.

### Sources to consult (in priority order)

1. **Vendor's "production checklist" / "best practices" / "security" doc page.** Most vendors publish one. Examples:
   - WorkOS: Fetch https://workos.com/docs (look for /best-practices and /security)
   - Cloudflare Workers: Fetch https://developers.cloudflare.com/workers/platform/limits and /observability/security
   - Neon: Fetch https://neon.com/docs/manage/security and /connect/connection-pooling
   - Stripe: Fetch https://docs.stripe.com/security and /webhooks/best-practices
   - Sentry: Fetch https://docs.sentry.io/platforms/javascript/best-practices/
   - Vercel: Fetch https://vercel.com/docs/production-checklist
   - Anthropic: Fetch https://docs.claude.com/en/docs/about-claude/admin/security-compliance and /best-practices
   - (For unknown vendors: Search "<vendor> production checklist" or "<vendor> best practices documentation")

2. **Vendor's status / changelog / advisory page** for recent breaking changes, EOL notices, security advisories. Most have one (e.g. https://www.workos.com/changelog, https://developers.cloudflare.com/changelog).

3. **Community gotchas.** Vendor docs cover *capabilities*; community sources cover *pitfalls*. Both needed.
   - Search "<vendor> common production mistakes <current year>"
   - Search "<vendor> security pitfalls"
   - Search "<vendor> gotchas"

4. **SDK currentness.**
   - `Bash` `npm view <package> version` (or pip / cargo / go equivalents)
   - Compare against the version pinned in the project's lockfile
   - Flag if more than 1 major behind, or if marked deprecated

5. **Context7** for current SDK API documentation:
   - `mcp__plugin_context7_context7__resolve-library-id` to find the library
   - `mcp__plugin_context7_context7__query-docs` for current API + migration notes

### What to extract per vendor

For each vendor, capture:

- **Production checklist items** the vendor itself publishes (these are the things they wish more customers got right)
- **Common misconfigurations** (specific patterns to grep / verify in the codebase)
- **Required guardrails** (signature verification, scope limits, callback state, etc.)
- **Recent advisories** (security CVEs, breaking changes, EOL notices)
- **Pricing surprises** (free-tier limits, cost cliffs, scale-to-zero behavior)
- **Versioning posture** (current SDK vs pinned; deprecation status)

### Per-vendor brief format

```markdown
### <Vendor name>

**Role:** auth / hosting / db / payment / observability / etc.
**Detected SDK / version:** <package> <version> (from lockfile)
**Latest stable:** <version> (<N> majors behind, if any)
**Sources consulted:**
- <URL 1> (fetched YYYY-MM-DD)
- <URL 2> (fetched YYYY-MM-DD)
- Search: "<query>" (date)

**Production checklist (per vendor docs):**
- [ ] <item 1>
- [ ] <item 2>
- [ ] <item 3>

**Common misconfigurations to check in this codebase:**
- <pattern 1 — where to look in the repo>
- <pattern 2>

**Required guardrails:**
- <e.g. webhook signature verification with `<header-name>` header>
- <e.g. callback state-parameter CSRF defense>

**Recent advisories / breaking changes:**
- <if any: date and summary, with link>

**Pricing notes:**
- <if relevant: cost cliffs, scale-to-zero behavior, free-tier limits>

**Cross-vendor notes:**
- <interactions with other detected vendors, if any — see Step 3>
```

---

## Step 3: Cross-vendor interactions

Some gotchas only emerge at the seams between vendors. They aren't on
either vendor's docs alone.

Examples:

- **WorkOS + Cloudflare Workers:** cookie domain alignment for session;
  callback URL must match deployed Worker route
- **Neon + Cloudflare Workers:** `@neondatabase/serverless` (HTTP) vs
  traditional `pg` (TCP) — Workers can't use TCP pooling; must use HTTP
  driver
- **Stripe + Vercel Edge / Cloudflare Workers:** webhook signature
  verification needs raw request body; edge runtime body handling can
  break this
- **Anthropic SDK + Cloudflare Workers:** streaming response behavior in
  Workers runtime; `event.waitUntil` for background after response
- **Postgres + Vercel serverless functions:** connection pool exhaustion
  under cold-start fan-out (Module 11 has the depth)
- **Supabase Auth + Vercel App Router:** server component vs server
  action cookie handling differences

For each pair of detected vendors, run:
- Search "<vendor A> <vendor B> production" or "<vendor A>
  <vendor B> gotcha" — community sources are typically the best for these

Compile findings into a "Cross-Vendor Interactions" section in the brief.

---

## Step 4: Emit the brief

Write the compiled brief to `/docs/audits/stack-brief.md`.

The brief is then a **prerequisite document** for downstream finding-
producing modules. They should:

- Read `/docs/audits/stack-brief.md` if it exists at module start
- Apply per-vendor checklist items as additional investigation steps
- Cite the brief in findings (e.g. "Per Stack Brief WorkOS section:
  callback route must validate state parameter — `app/auth/callback/route.ts`
  does not")

---

## Output structure

```markdown
# Stack-Specific Brief

Date: YYYY-MM-DD (compiled from sources fetched today)
Repository commit: <git rev-parse HEAD>

## Compilation summary

- Vendors detected: <N>
- Sources consulted: <count of URLs + searches>
- Vendors with full vendor-doc coverage: <N>
- Vendors relying on community sources only: <N>
- Compilation took: <minutes>

## Detected vendors

| Vendor | Role | Detected SDK | Latest | Behind | Sources consulted |
|---|---|---|---|---|---|
| WorkOS | Auth | @workos-inc/node 7.x | 7.x | 0 | docs.workos.com (fetched 2026-05-09) |
| Cloudflare Workers | Hosting | wrangler 3.x | 3.x | 0 | developers.cloudflare.com (fetched 2026-05-09) |
| Neon | DB | @neondatabase/serverless 0.9.x | 0.9.x | 0 | neon.com/docs (fetched 2026-05-09) |
| ... | | | | | |

## Per-vendor briefs

### WorkOS
(per Step 2 template)

### Cloudflare Workers
(per template)

### Neon
(per template)

### ...

## Cross-vendor interactions
(per Step 3)

## Discovery gaps

- Vendors where authoritative docs were not findable: <list>
  → downstream modules use generic checks for these
- Vendors where compilation failed: <list, with reason>
- Cross-vendor pairs not investigated: <list>

## Confidence

- **High:** vendor's production-checklist doc was found, SDK version
  discoverable, recent advisories checked.
- **Medium:** relying on community sources only, or vendor docs found
  but unclear coverage.
- **Low:** neither vendor docs nor community sources usable — note the
  gap; downstream modules use generic checks for this vendor.
```

---

## Severity calibration

This module's "findings" are typically informational (the brief is input
to other modules' findings). However:

- **Recent security advisory** affecting a detected version → Critical
  finding in this brief (downstream modules will pick it up too).
- **EOL runtime / SDK** the project is on → High in this brief.
- **Major version behind** with breaking changes upstream → Medium.
- Outdated patterns the vendor's docs explicitly call out → context for
  downstream modules.

---

## Evidence requirements

Every per-vendor brief cites the URL fetched and the date. Every claim
("WorkOS requires X") cites its source. Hand-waving without source =
reject and re-fetch.

If a vendor's docs were not findable, say so — don't invent best
practices. Downstream modules will fall back to generic checks for that
vendor; that is the graceful degradation.

---

## Why this beats static vendor sub-references

We considered (and rejected) maintaining static `references/vendors/<vendor>.md`-style files. The compiled-brief approach wins because:

- **No skill rot.** Vendor docs change continuously; static files would need quarterly updates.
- **Open coverage.** Works for any vendor the project happens to use, not just the ones we pre-wrote for.
- **Cross-vendor coverage.** Static files can't capture A + B interactions; live community search can.
- **Smaller skill surface.** One mechanism vs many static files.

If a vendor turns out to have particularly tricky common pitfalls that
are *not* in its docs and *not* easily searchable, that's a candidate for
a small static supplement — but it would be the exception, not the rule.
