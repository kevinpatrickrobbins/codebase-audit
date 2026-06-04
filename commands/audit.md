# Codebase Audit

Run a comprehensive multi-dimensional audit of the current repository using the
`codebase-audit` skill. Produces structured documents under `/docs/audits/`
and, for the remediation step, files real tickets in the project's tracking
system.

## Arguments

$ARGUMENTS

Parse as zero or more module names (comma- or space-separated), plus optional
flags. Module names map to the 29-module catalog in the skill:

### Synthesis (run last; aggregates everything)

| Name | Module |
|---|---|
| `index` | 00.1 — Audit Index |
| `risk` | 00.2 — Findings |
| `remediation` / `tickets` | 00.3 — Remediation Plan (files real tickets) |
| `compliance` / `readiness` | 00.4 — Compliance Readiness Mapping (runs detected frameworks; see rows below for the full list) |
| `soc2` | 00.4 — SOC 2 readiness mapping only |
| `hipaa` | 00.4 — HIPAA readiness mapping only |
| `pci` / `pci-dss` | 00.4 — PCI DSS readiness mapping only |
| `iso27001` / `iso` | 00.4 — ISO 27001:2022 readiness mapping only |
| `fedramp` | 00.4 — FedRAMP readiness mapping only |
| `coppa` | 00.4 — COPPA readiness mapping only |
| `ferpa` | 00.4 — FERPA readiness mapping only |
| `glba` | 00.4 — GLBA Safeguards Rule readiness mapping only |
| `quebec-25` / `quebec-law-25` / `loi25` | 00.4 — Quebec Law 25 readiness only |
| `casl` | 00.4 — CASL readiness only |
| `nis2` | 00.4 — EU NIS2 Directive readiness only |
| `dora` | 00.4 — EU DORA readiness only |
| `eu-ai-act` / `ai-act` | 00.4 — EU AI Act readiness only |
| `phipa` / `provincial-health` | 00.4 — Canadian provincial health (PHIPA / HIA / PHIA) only |

### Foundation

| Name | Module |
|---|---|
| `architecture` | 01 — System Architecture (snapshot of current state) |
| `stack-brief` / `vendors` | 01.5 — Stack-Specific Brief (compiled vendor gotchas at audit-time) |

### Compliance

| Name | Module |
|---|---|
| `security` | 02 — Security |
| `privacy` | 03 — Privacy & Data Protection (GDPR/CCPA/PIPEDA/PIPL/LGPD) |
| `accessibility` / `a11y` | 04 — Accessibility (WCAG/ADA/AODA/EAA) |
| `sector` / `sector-compliance` | 22 — Sector Compliance (HIPAA/PCI/SOC 2/ISO 27001/FedRAMP/COPPA/FERPA/GLBA — conditional) |

### Quality

| Name | Module |
|---|---|
| `arch-integrity` | 05 — Architecture Integrity |
| `testing` / `tests` | 06 — Testing & Quality |
| `docs` / `documentation` | 07 — Documentation & Onboarding |
| `deps` / `dependencies` | 08 — Dependencies & Supply Chain |
| `reuse` / `consolidation` / `dedupe` | 23 — Reuse & Consolidation |
| `workarounds` / `root-cause` / `band-aids` | 24 — Workarounds & Root-Cause Gaps |

### Performance

| Name | Module |
|---|---|
| `scalability` / `perf` / `performance` | 09 — Performance & Scalability (DB, scale) |
| `speed` | 10 — Speed (perceived performance, Core Web Vitals) |
| `serverless` / `coldstart` | 11 — Serverless DB & Cold-Start (conditional) |

### Operations

| Name | Module |
|---|---|
| `devops` | 12 — DevOps / Deployment |
| `cost` | 13 — Cost Analysis & Right-Sizing |
| `engineering` / `practice` | 14 — Engineering Practice (PR / release / on-call / repo health) |

### Product

| Name | Module |
|---|---|
| `ux` | 15 — UX Audit (heuristic) |
| `gap` / `product-gap` | 16 — Product Gap Analysis |
| `frontend` / `modernization` | 17 — Frontend Modernization |
| `i18n` / `localization` | 20 — Internationalization & Localization (conditional) |
| `seo` / `discoverability` | 21 — SEO & Discoverability (conditional, public web) |

### Domain-conditional

| Name | Module |
|---|---|
| `ai` / `ml` | 18 — AI / ML Audit (auto-skipped if no LLM SDKs detected) |
| `product-type` / `platform` | 19 — Product-Type Audit (iOS / Android / RN / Expo / Flutter / Electron / browser-ext / PWA / SaaS multi-tenant / marketplace / CLI / API-only) |

### Named modes (recurring partial-audit shortcuts)

These bundle a recurring module set behind a named shortcut — mechanically
equivalent to a manual multi-module selection, but documented so the same
question yields the same answer shape every time.

| Name | Modules | When to use |
|---|---|---|
| `ship` | 01, 02, 08, 06, 12, 16, 00.2 (top-5 only) | Pre-deploy / pre-release risk question. Emits a single `/docs/audits/ship-readiness-<YYYY-MM-DD>.md` with `SHIP` / `SHIP-WITH-CAVEATS` / `BLOCK` verdict. See SKILL.md Step 5 for details. |
| `security` | 02, 03, 22 | Security posture review |
| `perf` | 09, 10, 11 | Performance / scalability questions |
| `launch-compliance` | 02, 03, 04, 22 | Pre-launch regulatory readiness |
| `takeover` | 01, 05, 07, 08, 14 | New maintainer / legacy walkthrough |
| `ai` | 18 (+ 02, 03 as cross-refs) | LLM-feature deep dive |
| `hygiene` | 23, 24, 00.2 (findings); 00.3 draft only | Code-hygiene pass — duplication + workarounds added to the findings catalog without filing tickets (≡ `reuse workarounds risk --no-tickets`) |

If the user types a name not in this table, treat it as a module-name list
and parse per the tables above.

### Flags

- `--no-tickets` — run remediation as a draft markdown list only; do not
  file GitHub Issues or write to TICKETS.md
- `--dry-run` — produce all reports but do not write any audit files; print
  a summary instead

### Examples

- `/audit` → full suite (all applicable modules)
- `/audit ship` → pre-deploy ship-readiness verdict with the ship-set modules
- `/audit security privacy accessibility` → all three compliance audits
- `/audit deps testing docs` → quality-focused pass
- `/audit hygiene` → duplication + workaround scan, folded into the findings catalog (no tickets filed)
- `/audit cost speed` → infra and perf-focused pass
- `/audit launch-compliance` → pre-launch regulatory readiness
- `/audit takeover` → new-maintainer walkthrough
- `/audit --no-tickets` → full suite, but module 00.3 produces a draft only

---

## Your Task

Invoke the `codebase-audit` skill and follow its workflow exactly. The skill
auto-loads from this plugin. If it is not available in this session, read
`${CLAUDE_PLUGIN_ROOT}/skills/codebase-audit/SKILL.md` directly.

1. **Comprehension pass.** Walk the directory tree, read entry points, identify
   the stack.

2. **Module selection.** If the user passed module names as arguments, restrict
   the run to those modules. Otherwise run the full suite in the skill's
   prescribed order. Conditional modules (11 — serverless cold-start, 18 —
   AI/ML, 19 — product-type) auto-skip when not applicable.

3. **Run each module.** For each module, read its reference file (under
   `${CLAUDE_PLUGIN_ROOT}/skills/codebase-audit/references/<category>/<module>.md`)
   for the investigation playbook and required output structure. Cite file
   paths and line numbers as evidence. Do not speculate beyond what the code shows.

4. **Module 00.3 — Remediation Plan.** This module files **real tickets**, not a
   duplicate of the findings (Module 00.2). Follow the project's
   `tickets-and-logbook` conventions:
   - If the project has a GitHub remote and `gh auth status` is OK: each
     finding becomes one `gh issue create` with `proposed` status.
   - If the project has a `docs/TICKETS.md`: append entries in the project's
     existing format.
   - If neither: write `/docs/audits/remediation-tickets.md` containing draft
     issue bodies, ready to be filed manually.
   - **Always** preview the ticket list and ask for confirmation before bulk-
     creating issues. The `--no-tickets` flag forces draft-only mode.

5. **Output summary.** After all modules complete, output:
   - Files created or updated (with paths)
   - Top 5 cross-module findings, ranked by severity
   - Number of tickets filed (or drafted)
   - Modules skipped and the reason

---

## Why this command exists

A full codebase audit is one of those things that benefits from a fixed
playbook — it is too easy to miss a dimension or to write reports that look
thorough but lack evidence. The `codebase-audit` skill is the playbook; this
slash command is the deterministic invocation.
