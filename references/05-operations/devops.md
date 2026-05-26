# 12 — DevOps / Deployment Audit

Evaluate the deployment and infrastructure architecture of the project.

Create or update:

`/docs/audits/devops-audit.md`

If the file already exists, append a new dated section.

---

## How to investigate

DevOps audits often degenerate into "doesn't have CI/CD therefore bad". Avoid that.
The goal is to assess whether the deployment story is *production-ready for this
project's risk profile* — a side project does not need blue/green; a payments
platform does.

### Step 1: Identify the deploy target

Check (in this order):

| File | Implies |
|---|---|
| `vercel.json` / `now.json` | Vercel |
| `netlify.toml` | Netlify |
| `wrangler.toml` / `wrangler.jsonc` | Cloudflare Workers / Pages |
| `fly.toml` | Fly.io |
| `railway.json` / `nixpacks.toml` | Railway |
| `app.yaml` (top-level) | Google App Engine |
| `Dockerfile` + `docker-compose.yml` | Container, often self-hosted or container PaaS |
| `serverless.yml` | AWS Lambda via Serverless Framework |
| `cdk.json` / `*.ts` with `aws-cdk-lib` import | AWS CDK |
| `*.tf` | Terraform-managed infra |
| `helm/` / `*.yaml` with `apiVersion: apps/v1` | Kubernetes |
| `Procfile` | Heroku-style PaaS |
| None of the above | Manual deploy or undocumented |

If multiple are present, note the conflict and try to determine the canonical one
(often the most recently modified, or the one referenced in CI workflow).

### Step 2: CI/CD pipeline

Read `.github/workflows/*.yml`, `.gitlab-ci.yml`, `bitbucket-pipelines.yml`,
`.circleci/config.yml`, `Jenkinsfile`, etc.

Per workflow:
- **Triggers:** push to which branches? PRs? tags? schedule?
- **Stages:** install → typecheck → lint → test → build → deploy?
- **Secrets:** which secrets does the workflow consume? Are they referenced from
  the platform's secret store, or hardcoded?
- **Branch protection:** can production be deployed from a feature branch? From a
  PR that has not been merged? (You may not be able to verify branch-protection
  rules from the repo, but the workflow itself can hint.)
- **Test gating:** does deploy run *before* tests pass, or only after? If
  `deploy` does not `needs:` the test job, deploys can ship broken code.

### Step 3: Build and reproducibility

- Are dependencies pinned? (Lockfile present, committed.)
- Does the build use a specific Node/Python/Go version, or "latest"? Look for
  `engines` in `package.json`, `.nvmrc`, `.node-version`, `python_requires` in
  `pyproject.toml`, `go` directive in `go.mod`.
- Is there a `Dockerfile`? Does it use `latest` tags or pinned digests? Pinning
  to `node:20.11.0-alpine` is reproducible; `node:latest` is not.
- Is the build cacheable? (Multi-stage Dockerfile; CI cache config.)

### Step 4: Secrets management

- Where do production secrets live? Platform-native (Vercel env, AWS SSM, Doppler,
  Infisical, 1Password Secrets Automation, GitHub Secrets) or in a `.env` on a
  server (worse)?
- Is `.env*` in `.gitignore`?
- Is there a `.env.example` documenting required variables?
- Are secrets rotated? (You may not be able to tell from code; flag if there's no
  evidence of a rotation process.)

### Step 5: Database migrations & schema-change safety

Critical and often badly handled.

- **Find the migration directory** (`prisma/migrations`, `db/migrate`,
  `supabase/migrations`, `drizzle/migrations`, etc.).
- **How are migrations applied to production?** Look in CI for
  `prisma migrate deploy`, `npx drizzle-kit push`, `supabase db push`, etc.
  `prisma db push` (no history) is fine for dev, dangerous for prod.
- **Atomic, reversible, tested?** `down` migrations exist? Tested on a
  staging copy of prod? Some migrations are inherently irreversible (data
  loss); document those.

#### Schema-change mechanics

The category of bugs that crash production at deploy time:

- **Long-locking operations.** `ALTER TABLE ADD COLUMN NOT NULL DEFAULT
  ...` on Postgres ≥ 11 is fast (metadata-only); on older versions it
  rewrites the table. Audit recent migrations against the actual DB
  version.
- **Concurrent index creation.** Postgres `CREATE INDEX` blocks writes;
  `CREATE INDEX CONCURRENTLY` does not. Production migrations should use
  CONCURRENTLY for non-trivial tables.
- **Online schema change tools.** For MySQL projects with large tables:
  `pt-online-schema-change`, `gh-ost` — avoid the hour-long lock.
- **Pre-deploy vs post-deploy.** Schema-additive migrations (add column,
  add index) ship *before* code deploy; schema-removing migrations (drop
  column, drop table) ship *after* the code that no longer uses them
  rolls out. Search the deploy pipeline for this discipline.
- **Migration rollback story.** "Just run `down`" is naive — once data is
  written to a new column, dropping the column loses data. Roll-forward
  is usually safer than roll-back.
- **CI guard for slow migrations.** Some teams add a check that warns
  when a migration runs > N seconds on a sample dataset.
- **Backfills are not migrations.** A migration that adds a column AND
  populates it AND adds NOT NULL is three operations; should be three
  deploys (add column nullable → backfill → enforce NOT NULL).

### Step 6: Observability

- **Logging:** structured (JSON) or `console.log`? Cite the logger:
  `pino`, `winston`, `bunyan`, `loglevel`, or none.
- **Error tracking:** Sentry, Bugsnag, Rollbar SDK initialized? Where?
- **Metrics:** Prometheus, Datadog, New Relic, OpenTelemetry instrumented?
- **Tracing:** distributed tracing across services, or none?
- **Uptime monitoring:** any external pinger configured? (UptimeRobot, BetterStack,
  Cronitor — usually external, but referenced in README or runbooks.)
- **Health endpoints:** `/health`, `/api/health`, `/readyz`, `/livez` present?
  What do they check? (A health endpoint that always returns 200 is theatre.)

### Step 7: Reliability

- **Rollback strategy:** Vercel/Netlify allow instant rollback to previous
  deployment — assume yes if those are the platform. For container/k8s deploys,
  is there an explicit rollback step in CI?
- **Crash recovery:** for long-running services, is there a process supervisor
  (systemd, pm2, k8s)? Restart policy?
- **Database backups:** any reference to backup automation, or backup verification
  cron? Many databases (Supabase, Neon, RDS) have managed backups but recovery is
  rarely tested.
- **Scaling:** does the platform autoscale? Concurrency limits configured? Cold
  start vs warm pool?

---

### Step 8: Backup & restore drill

The category most teams say is fine but never test.

- **Backup tested by actually restoring**, not just "backups run nightly."
  When was the last successful restore drill? Documented?
- **Compromise-survivable backups.** *Critical:* backups must be
  restorable **without using the compromised application**. Failure modes:
  - App admin can delete backups via the same console used to manage the
    app — an attacker with admin access destroys both data and recovery
    path.
  - Backups stored in the same cloud account as production with the same
    IAM credentials — root credential compromise = total loss.
  - Restore process requires the app's auth system to log in to the
    backup tool — if the app is down or compromised, you can't restore.
  - Mitigations: separate cloud account / project for backups; immutable
    storage (S3 Object Lock, GCS Bucket Lock); offline copy for the
    most-critical data; restore tooling that doesn't depend on the app's
    auth.
- **Point-in-time recovery (PITR)** configured for managed DBs (RDS,
  Supabase, Neon, PlanetScale)?
- **Backup encryption** at rest enabled.
- **Cross-region** backup replication for disaster recovery (single-region
  backup of single-region DB doesn't help if the region itself is down).
- **Backup retention** schedule (daily / weekly / monthly with different
  retentions). One day of backups = lose a week of work to a delayed
  bug discovery.
- **Tenant-level restore** capability if multi-tenant SaaS — restoring
  one tenant from yesterday without rolling back everyone is much harder
  than full-DB restore. Document if absent.
- **Backup *and storage*** — DB backups + application data (S3 buckets,
  uploaded files, etc.) — both protected?
- **RTO / RPO targets** documented? (Recovery Time / Point Objectives —
  how long to restore, and how much data loss is acceptable.)

### Step 9: Environment parity

- **Dev / staging / prod config drift.** Same env-var schema across all?
  `.env.example` reflects every required var?
- **Same DB engine + version** across environments. Dev on SQLite, prod
  on Postgres = bugs that don't reproduce.
- **Same image / runtime.** Local Docker build matches prod image?
- **Feature flag exposure** — staging has every flag on, prod has the
  intended subset?
- **Secrets parity.** Same secret names across envs (with different
  values).
- **External-service test doubles.** Local dev has a mock Stripe, prod
  hits real Stripe — flag pricing, webhook signature, and behavior
  differences explicitly.

### Step 10: Network security (infra layer)

(App-layer security is in Module 02; this is the infra layer.)

- **WAF** — Cloudflare, AWS WAF, Vercel Firewall configured? Custom
  rules for common attack patterns? OWASP Core Rule Set tier?
- **DDoS protection** — Cloudflare proxy, AWS Shield, etc.?
- **TLS termination.** Where does TLS end? Cipher suites up to date?
  HSTS preload list?
- **mTLS for service-to-service** in regulated contexts.
- **VPC / subnet posture.** Public subnets only for what needs to be
  public; DB in private subnet.
- **Firewall / security group rules.** Ingress restricted to expected
  sources; egress limited where possible (e.g., LLM API allowlist for
  AI workloads — prevents data exfil if app gets compromised).
- **Bastion / VPN access** for ops vs direct internet access to prod
  infra.
- **Cloud audit log enabled** — AWS CloudTrail, GCP Cloud Audit Logs,
  Azure Activity Log. Covers control-plane changes; required for SOC 2.

---

## Red flags

- Production deploy step lacks `needs: test` — deploys can ship broken code.
- `:latest` Docker tags in production.
- Secrets committed in plaintext (`.env` checked in, secrets in YAML).
- No rollback plan documented and platform is not Vercel/Netlify/etc.
- Migrations applied via `prisma db push` in production (drift, no history).
- Health endpoint returns 200 unconditionally without checking dependencies.
- `npm install` (not `npm ci`) in CI — non-reproducible builds.
- Build uses `node:latest` or unpinned base image.
- `console.log` in production with no log shipper.

---

## Output structure

```markdown
# DevOps Architecture Review

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Current Deployment Strategy
- Hosting platform (cite config file)
- CI/CD pipeline (cite workflow file)
- Containerization (if present)
- Build process (cite Dockerfile or build script)
- Environment configuration (cite where secrets live)

## Infrastructure Risks
For each: severity, evidence (file path), recommendation.
- Secrets management
- Environment configuration
- Build reproducibility
- Dependency pinning
- Infrastructure security
- Production vs development configuration drift

## Deployment Reliability
- Rollback strategy
- Monitoring and logging
- Crash recovery
- Database migrations (the high-risk category)
- Scaling capability

## Observability
- Structured logging
- Error tracking
- Performance monitoring
- Uptime monitoring
- Distributed tracing
- Health endpoints

## Recommendations
Prioritized list of improvements. Each item: severity, effort, change type
(code / config / platform), evidence pointer.

## Final Recommendation
The single highest-impact DevOps action for this codebase right now.
**One thing.**

## Decision Summary

- Deploy posture production-ready for this project's risk profile: yes / no / partial
- Worst gap: <secrets | migrations | observability | rollback | backup | other>
- Recommended posture: <keep | targeted hardening | bigger platform / process change>

## Confidence
High / Medium / Low (and why)

## Out of Scope / Inconclusive
Items that need infrastructure access, runtime traces, or external service
verification to confirm.
```

---

## Evidence requirements

Cite specific config files (`vercel.json`, `.github/workflows/deploy.yml:25`).
Quote the relevant snippet in findings where useful. If a category is not
applicable (e.g., no CI exists at all), state that and recommend the simplest
viable starting point — do not pad the report with generic CI/CD platitudes.
