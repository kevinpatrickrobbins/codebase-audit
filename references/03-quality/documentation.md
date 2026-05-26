# 07 — Documentation & Onboarding

Audit the project's documentation, contributor experience, and onboarding
path. The acid test: **can a new developer get the dev environment running
and ship a small change in their first day?**

Output: `/docs/audits/documentation-audit.md`

---

## Cross-module relationship

- **Product Gap** (`06-product/gaps.md`) flags missing user-facing docs
  (help center, FAQ). This module is developer-facing docs.
- **Architecture** (`03-quality/architecture.md`) covers code-comment
  hygiene at a high level; this module is more comprehensive.

---

## Step 1: README

The single highest-leverage doc.

- **Exists?** And is it the canonical entry point, not a stub?
- **Quickstart.** Does it actually work? Mentally walk every step:
  - Required runtimes / tools listed (Node version, Python version, Docker)
  - Install commands accurate
  - Env-var setup (`.env.example` referenced or instructions inline)
  - Run command produces the expected output
  - Common failure modes documented
- **What it is.** First paragraph clearly states what the project does.
- **What it is not.** Optional but valuable — manages expectations.
- **Status / maturity.** "Alpha / Beta / Stable / Maintenance" labeled?
- **Links.** Docs site, contributing guide, license, code of conduct.
- **Currency.** Last updated when? Compared to recent code changes —
  likely accurate or rotted?

---

## Step 2: Dev environment reproducibility

Can a new dev get a working environment without tribal knowledge?

- **Runtime version pinning.** `.nvmrc` / `.node-version` / `python_requires`
  / `rust-toolchain.toml` / etc.
- **`.env.example`** present and actually covers every var the app reads.
- **Local services.** `docker-compose.yml` for DB / Redis / queue / mock
  services? Is `docker compose up` enough to run the app?
- **`.devcontainer/`** for VS Code Dev Containers / Codespaces — opt-in
  but reduces "works on my machine" issues.
- **Bootstrap script.** A single `./scripts/setup` or `make bootstrap` that
  installs everything and seeds initial data. Top tier.
- **Sample data / seed.** Database seed for first-run usable state. Apps
  that require manual data creation before they're usable have a real
  onboarding tax.

---

## Step 3: CONTRIBUTING and code of conduct

- **CONTRIBUTING.md** present? Covers:
  - How to run tests
  - Coding style / linter / formatter (or links)
  - PR process — branch naming, commit format, review expectations
  - Commit message convention (Conventional Commits, plain, etc.)
  - How to file an issue
- **CODE_OF_CONDUCT.md** for OSS / public projects. Many enterprises
  require it for procurement.
- **`.github/PULL_REQUEST_TEMPLATE.md`** — guides PR authors to provide
  context.
- **`.github/ISSUE_TEMPLATE/`** — bug report / feature request forms.

---

## Step 4: Architecture decision records (ADRs)

ADRs (Architecture Decision Records) capture *why* a decision was made
when it was made — invaluable when a future developer asks "why did we
pick X?"

- `docs/adr/` or `docs/decisions/` directory.
- **Format** — usually MADR or Nygard format.
- **Currency** — recent decisions captured? Or stopped after the first
  five?
- **Search** — is there an index?

Absent ADRs aren't a Critical finding for small projects but a strong
signal for ones that have grown.

---

## Step 5: Runbooks

For services that run in production:

- `docs/runbooks/` or similar.
- Coverage of common incidents — "DB connection exhausted", "queue
  backed up", "deployment failed midway", "OAuth provider down".
- Each runbook: symptoms, diagnostic commands, mitigations, escalation
  path.
- Last reviewed timestamp — runbooks rot fastest.
- Linked from alerts (PagerDuty / Opsgenie / Grafana alert links).

---

## Step 6: API / SDK documentation

If the project exposes an API:

- **OpenAPI / GraphQL schema** committed and published?
- **Hosted docs** — Swagger UI, Stoplight, Redoc, Mintlify, custom?
- **Per-endpoint examples** — request and response bodies?
- **Authentication setup** explained?
- **SDK generation** — auto-generated, hand-written, both?
- **Versioning / deprecation policy** documented?

If the project ships a public SDK:

- **README of the SDK package** itself (often forgotten).
- **Migration guides** between major versions.

---

## Step 7: Code-level documentation

- **Function / module comments** where they add real value (non-obvious
  behavior, invariants, why-not-the-obvious-approach).
- **JSDoc / docstrings / Rust-doc / Godoc.** Used consistently? Public
  APIs documented?
- **Inline `// TODO` / `// FIXME`** with author and date — or anonymous
  rotting markers?
- **Auto-generated reference** (TypeDoc, JSDoc, Sphinx, rustdoc, godoc)
  exists? Hosted?

The bar for inline comments isn't "comment everything" — it's "comment
where the *why* is non-obvious".

---

## Step 8: Changelog

- **`CHANGELOG.md`** present? Or release notes on GitHub Releases?
- **Format** — Keep a Changelog format, semantic categories
  (Added / Changed / Deprecated / Removed / Fixed / Security)?
- **Auto-generated** from commits, or hand-curated?
- **User-facing vs developer-facing.** A library has dev-facing changelogs;
  a SaaS app has user-facing release notes (different doc).

---

## Step 9: Diagrams

- **Architecture diagrams** — at least one. ASCII / Mermaid / draw.io /
  Excalidraw.
- **Sequence diagrams** for non-trivial flows (auth, checkout, multi-step
  process). Mermaid supports these in markdown.
- **ERD / data model** picture for the DB.
- **Deployment diagram.**
- **Currency** — diagrams rot fastest of all docs; check whether the
  one labeled "v2 architecture" still matches the codebase.

---

## Step 10: Onboarding test (the acid test)

The clearest signal: walk the README quickstart end-to-end as if new.

Imagine a new developer with the listed runtimes installed:

1. `git clone` ... → works?
2. Install deps as documented → works without errors?
3. Set up env vars per `.env.example` → all vars documented and clear?
4. Run dev command → app runs and is reachable?
5. Run tests → all pass?
6. Make a small change → see it reflected?

Each step that requires undocumented tribal knowledge is a finding.

---

## Red flags

- README is a stub or last meaningfully edited > 1 year ago.
- Quickstart references commands that don't exist.
- `.env.example` missing or stale.
- No `docker-compose.yml` / `.devcontainer/` for projects with non-trivial
  service dependencies.
- "Ask <person> for the seed file" / private Slack reference for setup.
- No CONTRIBUTING or PR template.
- ADRs absent on a project with > 1 year history and clear major
  decisions.
- Runbooks absent on production services.
- API / SDK with no docs site.
- "v2 architecture diagram" that's clearly v1.
- Hundreds of `// TODO` with no owner / date.

---

## Output structure

```markdown
# Documentation & Onboarding Audit

Date: YYYY-MM-DD
Repository commit: <hash>

## README
- Present and canonical: yes/no
- Quickstart accuracy (mentally walked): pass/fail
- What-it-is clarity
- Currency

## Dev environment reproducibility
- Runtime pinning (.nvmrc / etc.)
- .env.example completeness
- docker-compose / devcontainer
- Bootstrap script
- Seed data

## Contributor docs
- CONTRIBUTING.md
- CODE_OF_CONDUCT.md
- PR / issue templates
- Commit convention

## ADRs
- Present? Format? Currency?

## Runbooks
- Present? Coverage of common incidents? Linked from alerts?

## API / SDK docs
- Schema published?
- Hosted docs site?
- Examples? Auth setup? Versioning policy?
- SDK README and migration guides

## Code-level docs
- Comment hygiene (where non-obvious)
- JSDoc / docstrings on public APIs
- Auto-generated reference

## Changelog
- Present? Format? Currency?

## Diagrams
- Architecture / sequence / ERD / deployment
- Currency

## Onboarding acid test
Step-by-step result of mentally walking the new-dev path. Each step:
pass / fail / undocumented friction.

## Findings

## Final recommendation
Single highest-priority docs action.

## Decision summary
- New dev can ship in day-one: yes/no
- Critical decisions documented: yes/no
- Worst gap: <description>

## Confidence
High / Medium / Low

## Out of Scope / Inconclusive
(items requiring actually running through the steps with a fresh machine)
```

---

## Severity calibration

- **Critical:** quickstart is broken; production runbooks absent; new dev
  cannot bootstrap the project.
- **High:** README is stub-quality; ADR absence on major decisions; SDK
  with no docs.
- **Medium:** stale diagrams; thin changelog; ad-hoc PR template.
- **Low:** comment-style consistency; minor README polish.

Every finding cites the relevant doc file (or its absence).
