# 01 — System Architecture

You have completed a comprehension pass of the repository.

Create or update the file:

`/docs/system-architecture.md`

This document describes the current system as implemented in the repository. It is
overwritten on each audit run (unlike the dated audit reports), because it is meant
to reflect the system *now*.

---

## How to investigate

This module is the foundation for every later audit. Get this wrong and the security
audit will miss attack surfaces, the perf audit will misread the rendering model, and
the architecture audit will critique boundaries that were never the actual design.

**Read in this order:**

1. **Stack identification.** Open the package manifest first:
   - JavaScript/TypeScript: `package.json` (note `dependencies`, `scripts`, framework
     in `next`/`vite`/`remix`/`astro`/`@nestjs/*`/etc.)
   - Python: `pyproject.toml`, `requirements*.txt`, `Pipfile`
   - Rust: `Cargo.toml`
   - Go: `go.mod`
   - Ruby: `Gemfile`
   - Java/Kotlin: `pom.xml`, `build.gradle*`
   - .NET: `*.csproj`, `*.sln`
   - PHP: `composer.json`

2. **Stated purpose.** Read the top-level `README.md`. Note what the project says it
   does, who it is for. Cross-check this against what you find in the code; flag
   any divergence in "Known Limitations".

3. **Directory map.** Walk the top-level structure with `Glob *` or `ls`. For each
   non-trivial directory, note its apparent role in 1 line. If it is a monorepo
   (Nx, Turborepo, pnpm workspaces, lerna), list each package.

4. **Entry points.** Find where requests enter the system:
   - Web: `app/` (Next.js App Router), `pages/` (Next.js Pages or Nuxt), `src/routes/`
     (SvelteKit, Solid, React Router), `urls.py` + `views.py` (Django),
     `routes.rb` (Rails), `app.js`/`server.ts`/`main.ts`.
   - Background: `worker.ts`, `cron.ts`, `queue/*.ts`, Celery tasks, BullMQ jobs.
   - CLI: `bin/`, `cli.ts`, entry in `package.json` `"bin"`.

5. **Data layer.** Find schema, models, and migrations:
   - ORMs: `prisma/schema.prisma`, `drizzle/schema.ts`, `models/*.py`,
     `db/schema.rb`, `entities/*.ts` (TypeORM).
   - Migrations: `prisma/migrations/`, `db/migrate/`, `supabase/migrations/`,
     `migrations/`.
   - Raw SQL: any `.sql` files committed.

6. **Auth.** Find the authentication and authorization layer:
   - Common patterns: `lib/auth*`, `auth.config.ts`, `middleware.ts`, NextAuth /
     Auth.js, Clerk, Supabase Auth, Better Auth, Lucia, custom JWT.
   - Look for the *check*, not just the *login* — where is "is this user allowed?"
     decided per request?

7. **External integrations.** Search for SDK imports:
   - Payments: `stripe`, `paddle`, `lemon-squeezy`.
   - Email/SMS: `resend`, `postmark`, `twilio`, `sendgrid`, `mailgun`.
   - Analytics: `posthog`, `mixpanel`, `segment`, `ga4`, `plausible`.
   - Errors: `@sentry`, `bugsnag`, `rollbar`.
   - File storage: `@aws-sdk/client-s3`, `cloudinary`, `uploadthing`.
   - LLM: `anthropic`, `openai`, `@ai-sdk/*`, `langchain`.

8. **Deployment.** Identify hosting from config files:
   - `vercel.json` → Vercel; `netlify.toml` → Netlify; `fly.toml` → Fly.io;
     `wrangler.toml` → Cloudflare Workers; `Dockerfile` + `docker-compose.yml` →
     container deploy; `.github/workflows/*.yml` → CI-driven; `railway.json` →
     Railway; `serverless.yml` → AWS Lambda via Serverless Framework.
   - Note the runtime: edge, serverless, container, traditional VM.

9. **Environment.** Read `.env.example` (do *not* read `.env` or `.env.local` — those
   may contain secrets in some projects). Note required variables and what they
   tell you about external services.

---

## Step 10: Jurisdictional Landscape

The single most-missed input to compliance scope. Many teams discover late
that they're in scope for NIS2, EU AI Act, Quebec Law 25, or sector
regulations because they never explicitly mapped where their users live
and where their data flows.

This step produces a **jurisdictional inventory** that downstream modules
(02 Security, 03 Privacy, 04 Accessibility, 22 Sector Compliance, 00.4
Compliance Readiness) consume for scope determination.

**Investigate:**

- **Locale config.** `next.config.js` `i18n.locales`, similar — what
  locales does the product target? `fr-CA` ⇒ Quebec exposure; `de-DE` ⇒
  Germany exposure; `pt-BR` ⇒ Brazil (LGPD).
- **Marketing copy.** Search README, marketing pages, marketing site for
  region claims: "EU customers", "available in Canada", "global", country
  lists, "GDPR-compliant" / "AODA-compliant" / "FedRAMP" claims.
- **Pricing / billing.** Currencies supported (Stripe / Paddle config) —
  EUR/GBP ⇒ EU/UK; CAD ⇒ Canada; INR ⇒ India (DPDP Act); BRL ⇒ Brazil.
  Country lists in checkout.
- **Hosting regions.** Deploy config for what regions infrastructure
  runs in (`vercel.json` regions, AWS region pins in IaC, Cloudflare
  Workers — global by default).
- **Data residency.** Where customer data is stored — explicit regions in
  DB / R2 / S3 config.
- **Customer-facing legal pages.** Privacy Policy, Terms — language
  enumerating jurisdictions (e.g. "We comply with GDPR, CCPA, AODA").
- **Existing compliance claims.** Any badges or attestations referenced
  (SOC 2, HIPAA, ISO 27001, FedRAMP, EAA).
- **Analytics regions** if accessible — actual user geo distribution.
- **Customer types.** B2B SaaS selling to specific sectors (gov →
  FedRAMP; banks → DORA / GLBA; health → HIPAA / provincial Canadian
  health).

**Output a Jurisdictional Inventory section in `system-architecture.md`:**

```markdown
## 9. Jurisdictional Landscape

### Markets served (evidence-based)

| Region | Evidence | Confidence |
|---|---|---|
| United States | Stripe Connect US accounts; en-US default locale | High |
| European Union | Pricing in EUR; GDPR mention in privacy policy; de-DE/fr-FR locales | High |
| Canada | CAD currency in Stripe; en-CA locale; Toronto pricing | High |
| Quebec specifically | fr-CA locale with QC-specific copy; "Loi 25" mention | Medium |
| United Kingdom | GBP currency; UK-specific subdomain | High |
| Brazil | pt-BR locale; LGPD mention | Medium |

### Likely-applicable frameworks (for downstream modules)

| Framework | In scope? | Triggered by |
|---|---|---|
| GDPR | Yes | EU users |
| UK GDPR | Yes | UK users |
| CCPA / CPRA | Yes | US California users |
| PIPEDA | Yes | Canadian users (federal) |
| Quebec Law 25 | Yes | Quebec users specifically |
| AODA | Yes (if Ontario business) | Ontario operations |
| EU EAA | Yes from June 2025 | EU consumer-facing |
| EU AI Act | Yes (per AI risk tier) | EU + AI features |
| NIS2 | Possibly | Cloud / SaaS at scale + EU operations |
| DORA | Possibly | Financial-sector customers |
| CASL | Yes if email-marketing to Canadian recipients | Email integration + Canadian users |
| HIPAA | No / Yes | US health data |
| Provincial Canadian health | No / Yes | Canadian health data |
| Sector regs (PCI / SOC 2 / ISO 27001 / FedRAMP) | per Module 22 detection | per detection |

### Data flow geography

| Data type | Stored in | Transferred to | Cross-border issues? |
|---|---|---|---|
| Customer PII | EU (Frankfurt) | US (analytics provider) | EU→US transfer — SCCs / DPF? |
| Payment data | Stripe (US) | — | Tokenized; outside scope |
| LLM prompts | Anthropic (US) | — | If contains EU PII: sub-processor relationship + DPA |

### Notes

- (Anything inconclusive)
- (Recommendations to clarify scope explicitly: ask product team to
  confirm intended markets, gather actual GA4/PostHog geo data, etc.)
```

This jurisdictional inventory is the **foundation** for compliance
readiness. Module 22 (Sector Compliance) and Module 00.4 (Compliance
Readiness Mapping) read it to determine which framework sub-references
to load.

---

## Step 11: Git-Churn Hotspots

High-churn files — those touched most frequently in recent git history — are a
reliable leading indicator of instability, technical debt, and future bugs.
Intersecting churn with file size surfaces the riskiest surface area for
downstream modules to prioritize.

**Investigate:**

Run in the repo root:

```shell
# Files changed most often in the last 6 months
git log --since="6 months ago" --name-only --format="" | grep -v '^$' | sort | uniq -c | sort -rn | head -20
```

For each file in the top 20, note its approximate line count.

**Signals to escalate:**

- **> 20 changes in 6 months AND > 300 lines** — double-risk: pass to
  Security audit as a priority surface and to Architecture audit as a
  decomposition candidate.
- **Migration files in the churn list** — schema still settling; flag in
  the data-layer section and for the DevOps audit.
- **Config / deploy files** (`vercel.json`, `.env.example`, Dockerfile) —
  operational turbulence; flag for the DevOps audit.
- **Test files** dominating the list — possible flakiness or constantly-shifting
  requirements; flag for the Testing audit.
- **Auth / middleware files** in the churn list — elevated security attention;
  note in Pass A of the Security audit.

**Output:** populate "High-Churn Hotspots" (section 9 in the output template
below). If the repository has fewer than 50 commits or is under 6 months old,
mark the section "N/A — insufficient git history".

---

## Red flags to call out

- README says X but the code does Y (or doesn't do X at all).
- Multiple competing frameworks coexisting (Next.js + CRA, Vite + Webpack, etc.).
- Auth referenced but not enforced: middleware exists but does not actually gate
  protected routes.
- Hardcoded credentials in committed source (escalate to security audit; flag here).
- Critical features without an obvious code home.
- "It's a monolith" / "It's microservices" claimed but the code shows the opposite.
- Deploy target unclear or contradictory (Dockerfile but also `vercel.json`?).

---

## Output structure

Use these section headings exactly. Empty sections may be marked "N/A — not present
in this codebase".

```markdown
# System Architecture

Date: YYYY-MM-DD

## 1. System Purpose
- What the application does (1–3 sentences, evidence-based)
- Who the intended users are (cite README, marketing copy, or auth/role enums)

## 2. Core User Flows
- Primary workflows users follow (cite the route files / page components)
- Entry points into the system (URLs, page components, API endpoints)

## 3. Application Architecture
- Frontend framework and structure (cite `package.json` and entry points)
- Backend services and API layers (cite handler files)
- Data flow between components (RSC vs CSR, REST vs RPC vs GraphQL, etc.)

## 4. Database Design
- Database technology (cite driver / ORM)
- Major tables / models (link to schema file)
- Relationships between entities (1–2 sentence summary, not a full ERD)

## 5. Authentication and Authorization
- Login methods (cite auth config)
- Session handling (cookie / JWT / DB session)
- Access control model (RBAC, ABAC, none, ad hoc)

## 6. External Integrations
- APIs, third-party services, payment systems, analytics (cite SDK imports)

## 7. Deployment Architecture
- Hosting platform (cite the deploy config file)
- Build and deployment process (cite CI workflow file)
- Environment configuration (cite `.env.example`)

## 8. Known Limitations
- Incomplete features (cite TODO/FIXME or stub files)
- Areas that appear fragile (cite the evidence — e.g., "no error handling around
  external API X in `path/to/file.ts:LL`")

## 9. High-Churn Hotspots

*Files changed most frequently in the last 6 months, cross-referenced with
size. Downstream modules use this list as a prioritization input.*

| File | Changes (6 mo) | Est. lines | Downstream signal |
|---|---|---|---|
| `path/to/file.ts` | 34 | ~820 | Security (priority surface) · Architecture (decompose) |
| `path/to/schema.prisma` | 22 | ~310 | Data layer settling — DevOps flag |
| `path/to/auth.ts` | 18 | ~195 | Auth-surface churn — Security flag |

> Files appearing here should be cross-referenced in the Security audit's
> Pass A (Trust Boundaries) and the Architecture audit's coupling analysis.
```

---

## Evidence requirements

Every section should cite at least one file path. If you cannot find evidence for a
section (e.g., no obvious deployment config), say so explicitly: "No deploy config
file found in the repository — deploy mechanism unclear." Do not fabricate.

Only describe what can be inferred from the codebase. Do not speculate.
