# 00.1 — Audit Index

Create or update:

`/docs/audits/README.md`

This is the directory index for `/docs/audits/`. It lists every audit document
produced, with last-update date and a one-line summary. Run this module last
so it reflects what was actually produced in this audit run.

---

## Output structure

```markdown
# Audits

This directory contains the latest results of the codebase audit (per the
`codebase-audit` skill / `/audit` slash command).

Last full audit run: YYYY-MM-DD (commit: <hash>)

## Documents

### Synthesis (top-level deliverables)

| Module | Document | Last Updated | Summary |
|---|---|---|---|
| 00.2 | [Findings](./findings.md) | YYYY-MM-DD | Static catalog of risks across all audits — read this first for the high-level picture. |
| 00.3 | [Remediation Plan](./remediation-plan.md) | YYYY-MM-DD | Index of filed tickets (real GitHub Issues / TICKETS.md entries). |
| 00.4 | [Compliance Readiness](./compliance/) | YYYY-MM-DD or "N/A" | Conditional — per-framework readiness reports + CSV control matrices. **US:** SOC 2 / HIPAA / PCI DSS / FedRAMP / COPPA / FERPA / GLBA. **EU:** ISO 27001 / NIS2 / DORA / EU AI Act. **Canada:** Quebec Law 25 / CASL / Provincial Health (PHIPA / HIA / PHIA). |

### Foundation

| Module | Document | Last Updated | Summary |
|---|---|---|---|
| 01 | [System Architecture](../system-architecture.md) | YYYY-MM-DD | Snapshot of the system as currently implemented (overwritten each run). |
| 01.5 | [Stack-Specific Brief](./stack-brief.md) | YYYY-MM-DD | Compiled at audit-time from vendor docs + community sources. Per-vendor gotchas, advisories, pricing notes. Input to downstream modules. |

### Compliance

| Module | Document | Last Updated | Summary |
|---|---|---|---|
| 02 | [Security](./security-audit.md) | YYYY-MM-DD | Vulnerabilities, attack surface, data protection, secrets in git history. |
| 03 | [Privacy & Data Protection](./privacy-audit.md) | YYYY-MM-DD | GDPR, CCPA, PIPEDA, PIPL, LGPD posture; PII inventory; DSR rights; consent; logging discipline. |
| 04 | [Accessibility](./accessibility-audit.md) | YYYY-MM-DD | WCAG 2.1/2.2 AA, ADA, AODA, Section 508, EU EAA. |
| 22 | [Sector Compliance](./sector-compliance-audit.md) | YYYY-MM-DD or "N/A" | Conditional — HIPAA / PCI DSS / SOC 2 / ISO 27001 / FedRAMP / COPPA / FERPA / GLBA. |

### Quality

| Module | Document | Last Updated | Summary |
|---|---|---|---|
| 05 | [Architecture Integrity](./architecture-audit.md) | YYYY-MM-DD | Coupling, layers, naming, debt, type safety. |
| 06 | [Testing & Quality](./testing-audit.md) | YYYY-MM-DD | Coverage shape, pyramid balance, flakiness, contract testing. |
| 07 | [Documentation & Onboarding](./documentation-audit.md) | YYYY-MM-DD | README, ADRs, runbooks, dev-env reproducibility, onboarding. |
| 08 | [Dependencies & Supply Chain](./dependencies-audit.md) | YYYY-MM-DD | Outdated, EOL, CVEs, abandoned, license, SBOM, automation. |

### Performance

| Module | Document | Last Updated | Summary |
|---|---|---|---|
| 09 | [Performance & Scalability](./performance-audit.md) | YYYY-MM-DD | Query patterns, rendering, scalability under load. |
| 10 | [Speed (perceived)](./speed-audit.md) | YYYY-MM-DD | Core Web Vitals, hydration, asset perf, animation. |
| 11 | [Serverless DB & Cold-Start](./serverless-db-coldstart-audit.md) | YYYY-MM-DD or "N/A" | Conditional — only if stack is serverless/edge with connection-limited DB. |

### Operations

| Module | Document | Last Updated | Summary |
|---|---|---|---|
| 12 | [DevOps / Deployment](./devops-audit.md) | YYYY-MM-DD | Deployment strategy, CI/CD, observability, reliability. |
| 13 | [Cost Analysis & Right-Sizing](./cost-audit.md) | YYYY-MM-DD | Fitness of compute, DB, cache, queue, search, AI, monitoring, CDN. |
| 14 | [Engineering Practice](./engineering-practice-audit.md) | YYYY-MM-DD | PR / release / on-call / deprecation / repo health. |

### Product

| Module | Document | Last Updated | Summary |
|---|---|---|---|
| 15 | [UX Audit](./ux-audit.md) | YYYY-MM-DD | Navigation, flows, interface, mobile, copy. Heuristic only. |
| 16 | [Product Gap Analysis](./product-gap-analysis.md) | YYYY-MM-DD | Missing features for production readiness, edge cases, docs. |
| 17 | [Frontend Modernization](./frontend-modernization-audit.md) | YYYY-MM-DD | Modern CSS / JS / framework features under-utilized. |
| 20 | [i18n & l10n](./i18n-audit.md) | YYYY-MM-DD or "N/A" | Conditional — internationalization, localization, RTL, translation workflow. |
| 21 | [SEO & Discoverability](./seo-audit.md) | YYYY-MM-DD or "N/A" | Conditional — public-web only. Crawlability, structured data, social, monitoring. |

### Domain-conditional

| Module | Document | Last Updated | Summary |
|---|---|---|---|
| 18 | [AI / ML](./ai-ml-audit.md) | YYYY-MM-DD or "N/A" | Conditional — only if LLM SDKs detected. Prompt safety, output validation, evals, cost guardrails. |
| 19 | [Product-Type](./product-type-audit.md) | YYYY-MM-DD or "N/A" | Conditional — platform-idiom audit per detected type (iOS/Android/RN/Expo/Flutter/Electron/browser-ext/PWA/SaaS/marketplace/CLI/API). |

## How to read these documents

- **Stakeholders / non-engineers:** start with the [Findings](./findings.md).
  It gives the high-level picture without technical depth.
- **Engineers picking up work:** look at the [Remediation Plan](./remediation-plan.md)
  — it links to real tickets to claim.
- **System architects / new team members:** start with [System Architecture](../system-architecture.md).
- **Compliance / legal review:** [Security](./security-audit.md), [Privacy](./privacy-audit.md),
  and [Accessibility](./accessibility-audit.md). Risk register surfaces
  cross-cutting compliance issues.
- **Finance / leadership reviewing run-rate:** [Cost Analysis](./cost-audit.md) —
  fitness of infrastructure against workload, with estimated deltas.
- **Product / design reviewing user experience:** [UX](./ux-audit.md), [Speed](./speed-audit.md),
  and [Accessibility](./accessibility-audit.md) together form the user-facing
  picture.
- **Mobile / platform review:** [Product-Type Audit](./product-type-audit.md).
- **Operations / DevOps review:** [DevOps](./devops-audit.md), [Engineering Practice](./engineering-practice-audit.md),
  [Cost](./cost-audit.md).
- **AI engineering review (if applicable):** [AI/ML Audit](./ai-ml-audit.md).

## Conventions

- Audit documents are append-only — a new run adds a new dated section rather
  than overwriting (with the exception of System Architecture, which is always
  the current state, and the Audit Index itself).
- Findings cite specific file paths and (where relevant) line numbers.
- Severity scale: Critical / High / Medium / Low.
- Tickets emitted from these audits live in the project's tracking system (per
  `tickets-and-logbook` skill) — this directory documents the *audit*, the
  tickets are the *plan*.

## How to re-run

```
/audit                          # full suite
/audit security privacy         # selected modules
/audit --no-tickets             # full suite, draft remediation only
```

See the `codebase-audit` skill for the full module list and arguments.
```

---

## Completeness gate

Before writing the index, scan `/docs/audits/` for what files were actually
produced in this run. Then apply the following checks:

**Must-run modules** — if any of these output files are absent, the audit
is incomplete. Surface a warning block at the top of the index:

| Module | Expected file | Consequence if absent |
|---|---|---|
| 02 Security | `security-audit.md` | Critical risk surface unexamined |
| 08 Dependencies | `dependencies-audit.md` | CVE and EOL exposure unknown |
| 00.2 Findings | `findings.md` | No consolidated risk picture |
| 00.3 Remediation | `remediation-plan.md` | No actionable next steps |

**Conditional modules** — absent only if the module auto-skipped (serverless,
AI/ML, product-type, i18n, SEO). Do not flag these as missing without
confirming the skip reason.

**Warning format** (insert at top of index if any must-run files are absent):

```markdown
> **Incomplete audit** — the following modules did not produce output in
> this run: <list>. Findings and the remediation plan should not be treated
> as complete until these modules are run.
```

**Minimum finding threshold** — if the codebase has > 500 files and
`security-audit.md` contains zero findings, surface a note: "Zero security
findings on a project of this size is unusual — verify that Pass A
(Trust Boundaries Enumeration) was completed."

---

## Implementation notes

- Conditionally include or omit conditional-module rows (11, 18, 19) based on
  whether the audit produced those files.
- "Last Updated" date for each document is the date of the most recent
  appended section in that document (or the file's mtime, as a fallback).
- Keep the table tight. Detailed summaries belong in the documents themselves.
- The categorical grouping (Synthesis / Foundation / Compliance / Quality /
  Performance / Operations / Product / Domain-conditional) mirrors the skill's
  reference layout — readers who learn one structure can navigate the other.
