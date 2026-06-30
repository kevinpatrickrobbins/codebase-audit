---
name: codebase-audit
description: >
  Audits a software codebase across 30 dimensions — security, privacy,
  accessibility, sector compliance (HIPAA/PCI/SOC 2/ISO 27001/FedRAMP/COPPA/
  FERPA/GLBA/EU AI Act/NIS2/DORA/Quebec Law 25/etc.), architecture, testing,
  dependencies, code reuse / consolidation, workaround / root-cause detection,
  dead code / unused-surface detection, performance, speed, DevOps, cost,
  engineering practice, UX, product gaps, frontend modernization, i18n, SEO,
  AI/ML, and product-type platform idioms. Produces structured docs under
  `/docs/audits/` and files real remediation tickets. Use for any audit /
  review / analysis request, including narrow asks like "security review",
  "is this accessible", "are we GDPR/HIPAA/PCI/SOC 2 compliant", "is the app
  slow", "is this ready to ship", "audit our deployment", "find product
  gaps", "is our SEO any good", "are our deps current", "are we overpaying
  for infra", "can a new dev onboard", "find dead code", "what can we
  delete", "is there bloat in this codebase".
  The `/audit` command is the user-facing entry point.
---

# Codebase Audit Skill

This skill guides Claude through a systematic, multi-dimensional audit of a
software repository, producing structured documents under `/docs/audits/` and
filing real remediation tickets in the project's tracking system.

The `/audit` command is the user-facing entry point. The skill itself
contains the playbooks for each module.

---

## Reference layout

The `references/` directory is organized by category — synthesis at the top
(easy to find; these are the deliverables stakeholders look at first), then
foundation, then audit dimensions grouped by theme:

```
references/
├── 00-synthesis/                       (run last, listed first — top-level deliverables)
│   ├── audit-index.md                  Module 00.1 — directory README
│   ├── findings.md                Module 00.2 — static catalog of risks
│   ├── remediation-plan.md             Module 00.3 — files real tickets
│   ├── compliance-readiness.md         Module 00.4 — framework readiness mapping (dispatcher)
│   └── compliance-frameworks/          Module 00.4 sub-references
│       ├── soc2.md                     SOC 2 Type I / Type II
│       ├── hipaa.md                    HIPAA Security Rule
│       ├── pci-dss.md                  PCI DSS 4.0
│       ├── iso27001.md                 ISO/IEC 27001:2022
│       ├── fedramp.md                  FedRAMP (NIST 800-53 Rev 5)
│       ├── coppa.md                    COPPA (US children's privacy)
│       ├── ferpa.md                    FERPA (US student records)
│       ├── glba.md                     GLBA Safeguards Rule
│       ├── quebec-law-25.md            Quebec Law 25 (Canada — GDPR-strict)
│       ├── casl.md                     Canada's Anti-Spam Legislation
│       ├── nis2.md                     EU NIS2 Directive
│       ├── dora.md                     EU Digital Operational Resilience Act
│       ├── eu-ai-act.md                EU AI Act
│       └── canadian-provincial-health.md  PHIPA / HIA / PHIA (Canada)
├── 01-foundation/
│   ├── system-architecture.md          Module 01 — current-state snapshot
│   └── stack-brief.md                  Module 01.5 — vendor-specific gotchas, compiled at audit-time
├── 02-compliance/                      (legal-risk-bearing dimensions)
│   ├── security.md                     Module 02 — vulnerabilities, attack surface
│   ├── privacy.md                      Module 03 — GDPR/CCPA/PIPEDA/PIPL/LGPD
│   ├── accessibility.md                Module 04 — WCAG 2.1/2.2 AA, ADA, AODA, EAA
│   └── sector-compliance.md            Module 22 — HIPAA, PCI DSS, SOC 2, ISO 27001, FedRAMP, COPPA, FERPA, GLBA (conditional)
├── 03-quality/                         (engineering integrity)
│   ├── architecture.md                 Module 05 — coupling, layering, debt
│   ├── testing.md                      Module 06 — coverage, pyramid, flakiness
│   ├── documentation.md                Module 07 — README, ADR, runbooks, onboarding
│   └── dependencies.md                 Module 08 — supply chain, CVEs, EOL, SBOM
├── 04-performance/
│   ├── scalability.md                  Module 09 — DB, query patterns, scale
│   ├── speed.md                        Module 10 — perceived perf, Core Web Vitals
│   └── serverless-coldstart.md         Module 11 — conditional, edge + serverless DB
├── 05-operations/
│   ├── devops.md                       Module 12 — deployment, CI/CD, observability
│   ├── cost.md                         Module 13 — fitness vs spend
│   └── engineering-practice.md         Module 14 — PR / release / on-call / repo health
├── 06-product/
│   ├── ux.md                           Module 15 — heuristic UX
│   ├── gaps.md                         Module 16 — production-readiness gaps
│   ├── frontend-modernization.md       Module 17 — modern CSS/JS/framework features
│   ├── i18n.md                         Module 20 — internationalization & localization
│   └── seo.md                          Module 21 — SEO & discoverability
└── 07-domain-conditional/              (run only when applicable)
    ├── ai-ml.md                        Module 18 — when LLM SDKs detected
    ├── product-type.md                 Module 19 — dispatcher; detects product type(s)
    └── product-types/                  Module 19 sub-references
        ├── ios-native.md
        ├── android-native.md
        ├── react-native.md
        ├── expo.md
        ├── flutter.md
        ├── electron.md
        ├── browser-extension.md
        ├── pwa.md
        ├── saas-multi-tenant.md
        ├── marketplace.md
        ├── cli-tool.md
        └── api-only.md
```

**File numbering vs run order:** the directory prefix numbers (`00-`, `01-`,
`02-`, …) signal *category importance for navigation* (synthesis at top,
domain-conditional last). They do **not** dictate run order. Run order is
defined explicitly in [Step 4: Sequencing](#step-4-sequencing) below.

---

## Step 1: Comprehension Pass

Before running any audit, build a working mental model of the system. The
audits that follow are only as good as this first pass — confused first reads
produce thorough-sounding reports that miss the actual issues.

Read in this order:

0. **Read project context docs FIRST if they exist at the repo root** —
   `AGENTS.md`, `CLAUDE.md`, `MAINTENANCE.md`, and any `.cursor/rules/*` or
   similar agent-targeted instructions. These declare project-specific context
   that every audit recommendation must respect:

   - **Workflow conventions.** Is this a solo-dev project (commits to `main`,
     no PRs) or a team project (PR-based, code review, GitHub Issues)?
   - **Tracking system.** GitHub Issues, Linear, Jira, in-repo `docs/TICKETS.md`?
     Module 00.3 (Remediation Plan) files tickets — knowing the system matters.
   - **Locked tech-stack decisions.** What is non-negotiable vs. open to
     recommendation?
   - **Workflow taboos.** Some projects forbid feature branches, some require
     them. Some forbid GitHub Issues, some require them. Don't recommend a
     workflow that contradicts what the project explicitly rejects.
   - **Domain-specific rules.** Multi-tenancy isolation, security boundaries,
     RBAC models, billing-related don'ts.

   If `AGENTS.md` says "solo dev — no PR workflow," audit recommendations MUST
   NOT propose PR-based CI gates, code review processes, or multi-contributor
   workflows. If `CLAUDE.md` says "use TICKETS.md not GitHub Issues," Module
   00.3 MUST file tickets to `TICKETS.md`, not via `gh issue create`.

   Surface what you read from these files in your Output Summary (Step 6) so
   the user can confirm you respected the context.

1. **Identify the stack.** Read package manifest (`package.json`,
   `pyproject.toml`, `Cargo.toml`, `go.mod`, `Gemfile`, etc.). Note framework,
   runtime, package manager, major dependencies.
2. **Read the README.** Note stated purpose and intended users.
3. **Walk the directory structure.** Use Glob or `ls` to map top-level layout.
4. **Find entry points.** Routes, server entry, worker entry.
5. **Find data layer.** Schema, models, migrations.
6. **Find deployment config.** Identify the deploy target.
7. **Find environment config.** `.env.example`, secret manager refs.

Only describe what can be inferred from the codebase. Do not speculate beyond
what the code shows. If something is unclear, say so explicitly.

If the project context docs from step 0 explicitly forbid a workflow pattern
(team review, PRs, GitHub Issues, etc.), every audit recommendation throughout
the run MUST conform. Recommendations that violate declared project conventions
are a defect of the audit, not a "best practice that should override the
project's choice."

---

## Step 1.5: Live Discovery (anti-rot)

Several audit dimensions depend on fast-moving facts: WCAG version, runtime
EOL dates, latest framework versions, current LLM models, store policies,
CVE state, current Baseline-supported web platform features. Hardcoding these
in skill content makes findings rot — a 2026 audit run citing 2025 facts is
unreliable.

For modules with **"Live Discovery"** sections, **fetch the current
authoritative source at audit-time** before comparing the codebase against
it. The skill contains the *methodology* (how to investigate); the *current
truth* comes from live sources.

### How to discover

| Tool | Use for |
|---|---|
| Fetch | known authoritative URLs (W3C WCAG, MDN, Apple/Google policy pages, runtime EOL pages) |
| Search | "current X" / "latest Y" queries when URL is unknown |
| `Bash` | tooling: `npm view <pkg> version`, `npm outdated --json`, `pip index versions`, `cargo search`, `gh api` |
| `mcp__plugin_context7_context7__query-docs` | up-to-date library docs (React, Next.js, Anthropic SDK, etc.) |
| `mcp__plugin_context7_context7__resolve-library-id` | resolve a library name to a context7 ID |

### Reporting discovered facts

Every module that performs Live Discovery should:

1. Cite the **source URL** of the discovered fact.
2. Cite the **date** the source was fetched (today's date).
3. Cite the **version / value** that was discovered.
4. Compare codebase findings against the discovered current state.

Example output line:

> Discovered: WCAG 2.2 is the current published recommendation
> (https://www.w3.org/TR/WCAG22/, fetched 2026-05-09). The codebase appears
> to target WCAG 2.1 (no SC 2.4.11 focus appearance, no SC 2.5.8 target
> size). Closing the 2.2 gap is the next compliance step.

### When discovery is unavailable

- No network access in this environment
- Source has moved / is paywalled
- Tool failed for a transient reason

Then:

- **Note the gap explicitly** in "Out of Scope / Inconclusive"
- **Fall back to embedded knowledge** with a `may be stale, last-known: <date>` caveat
- **Never silently fabricate** currency claims

### Modules with Live Discovery

- **02 — Security** — current OWASP Top 10
- **03 — Privacy** — current adequacy decisions, recent enforcement actions, cross-border transfer mechanisms
- **04 — Accessibility** — current WCAG version, jurisdiction enforcement state
- **08 — Dependencies** — latest versions, runtime EOL schedules, CVE state
- **10 — Speed** — current Core Web Vitals thresholds, Baseline-supported features, framework perf primitives
- **13 — Cost** — current pricing for cited services (where useful)
- **17 — Frontend Modernization** — Baseline state, current framework versions
- **18 — AI / ML** — current model lineup and pricing
- **19 — Product-Type sub-references** — platform policy currency (App Store, Play Console, etc.)
- **20 — i18n & l10n** — current CLDR / Intl API / ICU MessageFormat state, framework i18n library versions
- **21 — SEO & Discoverability** — current Schema.org vocabulary, Google Search guidance, AI search citation practice
- **25 — Dead Code & Unused Surface** — current best-in-class unused-code tooling per ecosystem (e.g. knip vs. ts-prune, vulture vs. deadcode)

Other modules can adopt the pattern as their content rots; flag candidates
for the next maintenance pass.

---

## Step 2: Module catalog

> **Module 22 vs Module 00.4 — the two compliance lanes.** Both touch the
> 8 sector frameworks (HIPAA, PCI DSS, SOC 2, ISO 27001, FedRAMP, GLBA,
> COPPA, FERPA). They are **complementary, not redundant**:
>
> - **Module 22 (Sector Compliance)** — *audit-time finding pass*, driven
>   by direct codebase scan. Output: `sector-compliance-audit.md` with
>   severity-tiered findings ("PHI in unredacted logs — Critical, fix
>   ticket").
> - **Module 00.4 (Compliance Readiness Mapping)** — *auditor-ready
>   control mapping*, driven by control structure. Output: per-framework
>   `<framework>-readiness.md` + `<framework>-controls.csv` matrix
>   (IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND / N/A per control).
>
> Module 00.4 also covers 6 frameworks Module 22 does not: Quebec Law 25,
> CASL, NIS2, DORA, EU AI Act, Canadian provincial health. Run Module 22
> for "what needs fixing", Module 00.4 for "are we ready for the
> auditor".

| # | Module | Output File | Reference |
|---|---|---|---|
<!-- BEGIN: modules-catalog (generated from modules.json; run tools/render-from-json.ps1 to regenerate) -->
| **Synthesis** | | | |
| 00.1 | Audit Index | `/docs/audits/README.md` | `references/00-synthesis/audit-index.md` |
| 00.2 | Findings | `/docs/audits/findings.md` | `references/00-synthesis/findings.md` |
| 00.3 | Remediation Plan (tickets) | `/docs/audits/remediation-plan.md` + filed tickets | `references/00-synthesis/remediation-plan.md` |
| 00.4 | Compliance Readiness Mapping | `/docs/audits/compliance/<framework>-readiness.md` + `<framework>-controls.csv` per framework | `references/00-synthesis/compliance-readiness.md` (+ `compliance-frameworks/*.md`) |
| **Foundation** | | | |
| 01 | System Architecture | `/docs/system-architecture.md` | `references/01-foundation/system-architecture.md` |
| 01.5 | Stack-Specific Brief | `/docs/audits/stack-brief.md` | `references/01-foundation/stack-brief.md` |
| **Compliance** | | | |
| 02 | Security | `/docs/audits/security-audit.md` | `references/02-compliance/security.md` |
| 03 | Privacy & Data Protection | `/docs/audits/privacy-audit.md` | `references/02-compliance/privacy.md` |
| 04 | Accessibility & Compliance | `/docs/audits/accessibility-audit.md` | `references/02-compliance/accessibility.md` |
| 22 | Sector Compliance (conditional) | `/docs/audits/sector-compliance-audit.md` | `references/02-compliance/sector-compliance.md` |
| **Quality** | | | |
| 05 | Architecture Integrity | `/docs/audits/architecture-audit.md` | `references/03-quality/architecture.md` |
| 06 | Testing & Quality | `/docs/audits/testing-audit.md` | `references/03-quality/testing.md` |
| 07 | Documentation & Onboarding | `/docs/audits/documentation-audit.md` | `references/03-quality/documentation.md` |
| 08 | Dependencies & Supply Chain | `/docs/audits/dependencies-audit.md` | `references/03-quality/dependencies.md` |
| 23 | Reuse & Consolidation | `/docs/audits/reuse-consolidation-audit.md` | `references/03-quality/reuse-consolidation.md` |
| 24 | Workarounds & Root-Cause Gaps | `/docs/audits/workarounds-audit.md` | `references/03-quality/workarounds.md` |
| 25 | Dead Code & Unused Surface | `/docs/audits/dead-code-audit.md` | `references/03-quality/dead-code.md` |
| **Performance** | | | |
| 09 | Performance & Scalability | `/docs/audits/performance-audit.md` | `references/04-performance/scalability.md` |
| 10 | Speed (perceived) | `/docs/audits/speed-audit.md` | `references/04-performance/speed.md` |
| 11 | Serverless DB & Cold-Start (conditional) | `/docs/audits/serverless-db-coldstart-audit.md` | `references/04-performance/serverless-coldstart.md` |
| **Operations** | | | |
| 12 | DevOps / Deployment | `/docs/audits/devops-audit.md` | `references/05-operations/devops.md` |
| 13 | Cost Analysis & Right-Sizing | `/docs/audits/cost-audit.md` | `references/05-operations/cost.md` |
| 14 | Engineering Practice | `/docs/audits/engineering-practice-audit.md` | `references/05-operations/engineering-practice.md` |
| **Product** | | | |
| 15 | UX Audit | `/docs/audits/ux-audit.md` | `references/06-product/ux.md` |
| 16 | Product Gap Analysis | `/docs/audits/product-gap-analysis.md` | `references/06-product/gaps.md` |
| 17 | Frontend Modernization | `/docs/audits/frontend-modernization-audit.md` | `references/06-product/frontend-modernization.md` |
| 20 | i18n & l10n (conditional) | `/docs/audits/i18n-audit.md` | `references/06-product/i18n.md` |
| 21 | SEO & Discoverability (conditional) | `/docs/audits/seo-audit.md` | `references/06-product/seo.md` |
| **Domain-conditional** | | | |
| 18 | AI / ML (conditional) | `/docs/audits/ai-ml-audit.md` | `references/07-domain-conditional/ai-ml.md` |
| 19 | Product-Type (conditional) | `/docs/audits/product-type-audit.md` | `references/07-domain-conditional/product-type.md` (+ `product-types/*.md`) |
<!-- END: modules-catalog -->

---

## Step 3: Document Creation Rules

- **Create or update.** If a file already exists, append a new dated section
  rather than replacing it — except `system-architecture.md`, which is always
  the current state.
- **Date format:** `YYYY-MM-DD`.
- **Commit hash:** run `git rev-parse HEAD` if in a git repo; otherwise `N/A`.
- **Be concrete.** Every finding cites a file path and (where relevant) line
  number. Vague claims are worthless.
- **No speculation.** Only report what is observable in the codebase. If a
  check is inconclusive, say so in "Out of Scope / Inconclusive".
- **Calibrate severity** consistently: Critical (data loss, RCE, exploitable
  PII leak, legal exposure) / High / Medium / Low.
- **Mark unknowns explicitly.** When billing data, real-device verification,
  or production traffic data is needed, name the gap rather than guessing.

---

## Step 4: Sequencing

Modules feed each other; some are conditional. Run in this order:

```
Foundation:        01 → 01.5 (stack brief — input to downstream)
Detection:         19 (product-type detection)
Compliance:        02 → 03 → 04 → 22 (sector — conditional)
Quality:           08 → 05 → 23 → 24 → 25 → 06 → 07
Performance:       09 → 10 → 11 (conditional)
Operations:        12 → 13 → 14
Product:           15 → 16 → 17 → 20 (conditional) → 21 (conditional)
Domain:            18 (conditional)
Synthesis (last):  00.2 → 00.3 → 00.4 (conditional, per detected sectors) → 00.1
```

**Modules that consume the Stack Brief:**

After Module 01.5 produces `/docs/audits/stack-brief.md`, the following
finding-producing modules should read it at module start and apply
vendor-specific calibration to their checks:

- **02 Security** — vendor-specific auth/secret/webhook gotchas
- **03 Privacy** — sub-processor relationships, vendor data handling
- **11 Serverless cold-start** — runtime + DB driver interactions
- **12 DevOps** — vendor-specific deployment / observability gotchas
- **13 Cost** — current pricing, free-tier cliffs, scale-to-zero specifics
- **18 AI / ML** — model lineup, current capabilities, batch / caching status
- **19 Product-Type sub-refs** — platform + vendor specifics

Each of these modules cites the Stack Brief in findings where vendor-
specific items apply. The brief is the *input*; the modules produce the
*findings*.

Dependency rationale:

- **Module 01** (System Architecture) is consumed by every later module — run first.
- **Module 19** (Product-Type detection) runs early so later modules can adapt
  (e.g., accessibility audit knows the platform; cost audit knows native vs web).
- **Module 08** (Dependencies) runs before quality modules because EOL runtime
  findings cascade into other quality concerns.
- **Modules 23 + 24** (Reuse & Consolidation, Workarounds & Root-Cause Gaps)
  run right after **Module 05** (Architecture Integrity): both deepen analysis
  that Module 05 only scans at the surface (Step 6 duplication → 23; Step 10
  tech-debt markers → 24), so they reuse its component map and findings.
- **Module 25** (Dead Code & Unused Surface) runs right after **Modules 23 +
  24**: it deepens the same Module 05 surface scan (Step 6 duplication, Step
  10 "legacy"/"v2" directories) from a different angle — confirming whether
  flagged code is actually unreferenced, rather than merely duplicated (23)
  or band-aided (24). Running last in the trio lets it benefit from 23/24
  having already established the component map and the canonical
  implementations, so it isn't re-deriving that context.
- **Module 11** (Serverless cold-start) is conditional — only when stack is
  serverless/edge with a connection-limited DB.
- **Module 18** (AI/ML) is conditional — only when LLM SDKs are present.
- **Module 13** (Cost) follows Module 12 (DevOps) — DevOps documents the
  current shape; cost analysis assesses fitness against workload.
- **Module 00.2** (Findings) draws from all prior audits — run after
  every other finding-producing module.
- **Module 00.3** (Remediation Plan) converts findings into discrete, atomic
  tickets. Run after 00.2 so the findings informs prioritization.
  **Module 00.3 is not a duplicate of Module 00.2** — 00.2 is a static catalog;
  00.3 emits actionable, dependency-sequenced tickets.
- **Module 00.1** (Audit Index) catalogs the produced documents — runs last.

---

## Step 5: Partial Audits

If the user only wants one or a few modules (e.g. `/audit security`,
`/audit cost speed`), run only the requested modules. List which other modules
are available in your response and offer to run the full suite.

Do not silently expand or shrink scope.

### Named modes

Some partial-audit invocations bundle a recurring module set behind a
named shortcut. These are conventional groupings for common questions
— mechanically equivalent to a manual multi-module selection, just
documented so the same question yields the same answer shape every time.

#### `/audit ship` — Ship readiness

**When to use.** The user asks "is this ready to ship", "what would
break in production", "can we launch", "ready for the demo / customer
rollout", or any pre-deploy / pre-release risk question. The full
25-module audit is too broad for this — the user wants a fast
ship / block verdict, not a regulatory deliverable.

**Modules run, and why each is in the ship set:**

| # | Module | Why it gates shipping |
|---|---|---|
| 01 | System Architecture | Snapshot of what's actually shipping |
| 02 | Security | Any critical vulnerability is a block |
| 08 | Dependencies | Runtime EOL or known CVE in prod path is a block |
| 06 | Testing | Coverage of the critical paths going live |
| 12 | DevOps | Deploy plan, rollback, observability |
| 16 | Product Gaps | Known-incomplete features users will hit |
| 00.2 | Findings (top 5 only) | Synthesizes the above into a single list |

**Output.** Instead of the standard `/docs/audits/` artifacts plus
remediation plan, emit a single file
`/docs/audits/ship-readiness-<YYYY-MM-DD>.md` with:

- **Verdict:** `SHIP` / `SHIP-WITH-CAVEATS` / `BLOCK`
- **Blockers** — Critical findings from any module. Must fix before
  deploy.
- **Caveats** — High findings. Should fix soon; not blockers.
- **Watch list** — Medium findings. Track post-launch.
- **Recommended actions** — ranked by severity, with file paths.

**What is intentionally not in ship mode:**

- Compliance modules (03 Privacy, 04 Accessibility, 22 Sector) — these
  are regulatory readiness, separate from technical shipping. Use
  `/audit launch-compliance` when that's the question.
- UX, frontend modernization, i18n, SEO — quality improvements, not
  ship blockers.
- Cost — overspending in production is recoverable; not a ship gate.
- AI/ML (18) — run separately when applicable.

If a compliance, AI, or product-type concern surfaces during the ship
audit and looks like a blocker, **name it explicitly and recommend the
corresponding deeper module** — do not silently expand scope. The whole
point of ship mode is keeping the answer fast and focused.

#### Other recurring groupings (recipes, not formal modes)

| Invocation | Modules | When |
|---|---|---|
| `/audit security` | 02, 03, 22 | Security posture review |
| `/audit perf` | 09, 10, 11 | Performance / scalability questions |
| `/audit launch-compliance` | 02, 03, 04, 22 | Pre-launch regulatory readiness |
| `/audit takeover` | 01, 05, 07, 08, 14 | New maintainer / legacy walkthrough |
| `/audit ai` | 18 (+ 02, 03 as cross-refs) | LLM-feature deep dive |
| `/audit hygiene` | 23, 24, 25 → 00.2 (findings); 00.3 draft | Code-hygiene pass — duplication + workarounds + dead code folded into the findings catalog, **not** auto-filed (≡ `reuse workarounds dead-code risk --no-tickets`). |

These are conventions only — the user types the invocation, the skill
runs the listed modules. Don't invent new named modes ad hoc; if a new
recurring grouping emerges, document it here first.

---

## Step 6: Output Summary

After completing all modules, provide:

1. The most critical findings across all reports (top 5).
2. The list of files created or updated (with paths).
3. For Module 00.3: number of tickets filed (or drafted), and where they live.
4. Any modules skipped and why (conditional modules, partial-audit selection).
5. **Live-discovery health.** For each module that ran with a Live Discovery
   step (see the list in Step 1.5), report whether it successfully fetched
   current authoritative state or fell back to embedded knowledge — and the
   date of the fetched data. A run that fell back across many modules is
   higher-risk for stale findings; surface that explicitly so the user can
   decide whether to re-run with network access.
6. Top 3–5 recommended next actions, ranked by severity.

Keep this summary tight — the audit documents themselves contain the detail.

---

## Step 7: Module-numbering policy

Every reference file's H1 includes its catalog module number (e.g.
`# 09 — Performance & Scalability Audit`). The catalog table in Step 2 is the
authoritative source of truth for module numbers.

When numbers change:

- Update the catalog table in `SKILL.md` (Step 2).
- Update the sequencing diagram (Step 4).
- Update the H1 in the affected reference file(s).
- Search `references/` for every cross-reference of the old number and update
  it. Prefer the form `Module NN (Name)` so a reader can find the target
  even if the number drifts again — or use the file path form
  (`references/<category>/<file>.md`) which is rename-stable.
- Update `/audit` command file (module-name table and examples).
- Add a CHANGELOG entry describing the renumbering and the files touched.

The directory-prefix numbers (`00-`, `01-`, …) are a category-navigation
aid; they are not module numbers. Module numbers come from the catalog.

See `MAINTENANCE.md` for the broader maintainer playbook.

---

## A note on style

These audits exist to produce *honest, actionable* assessments. Avoid the trap
of checklist theater: filling every section with vague observations to look
thorough. A short, sharp report with five well-evidenced findings is worth
more than thirty pages of generic platitudes. If a section genuinely has
nothing to flag, say so and move on.

A boring "keep it" verdict is often the correct one. Don't invent migration
stories where none are needed. Calibrate to the project's actual scale and
maintainer team size — recommendations that fit a 50-person engineering org
are usually wrong for a solo founder.
