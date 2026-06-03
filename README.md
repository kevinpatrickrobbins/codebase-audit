# codebase-audit

An audit playbook that examines a software codebase across 27 dimensions and
produces structured documentation under `docs/audits/` plus filed remediation
tickets in the project's tracking system.

Covers security, privacy, accessibility, sector compliance (HIPAA / PCI DSS /
SOC 2 / ISO 27001 / FedRAMP / GLBA / COPPA / FERPA / EU AI Act / NIS2 / DORA /
Quebec Law 25 / etc.), architecture, testing, dependencies, performance, speed,
DevOps, cost, engineering practice, UX, product gaps, frontend modernization,
i18n, SEO, AI/ML, and product-type platform idioms (iOS / Android / RN / Expo /
Flutter / Electron / browser-extension / PWA / SaaS multi-tenant / marketplace /
CLI / API).

The entry point is the `/audit` command.

---

## Install

This is a Claude Code **plugin**.

**From GitHub:**

```
/plugin marketplace add kevinpatrickrobbins/codebase-audit
/plugin install codebase-audit@codebase-audit
```

**From a local clone:**

```
/plugin marketplace add /absolute/path/to/codebase-audit
/plugin install codebase-audit@codebase-audit
```

**For local development (no install):**

```
claude --plugin-dir /absolute/path/to/codebase-audit
```

**Verify.** In a project directory, run `/audit` — Claude loads the skill and
begins the comprehension pass.

---

## Usage

```
/audit                          # full suite (27 modules, conditional ones auto-skip)
/audit ship                     # pre-deploy readiness verdict
/audit security                 # security posture review
/audit security privacy a11y    # specific modules
/audit launch-compliance        # pre-launch regulatory readiness
/audit takeover                 # new-maintainer / legacy walkthrough
/audit --no-tickets             # draft remediation only, do not file tickets
```

Full module catalog and named-mode list: see
[SKILL.md](./skills/codebase-audit/SKILL.md) Step 2 and Step 5, or
[commands/audit.md](./commands/audit.md).

---

## Layout

```
.claude-plugin/
  plugin.json            Plugin manifest
  marketplace.json       Marketplace entry (installable via /plugin marketplace add)
commands/audit.md        The /audit slash command body (plugin entry point)
skills/codebase-audit/
  SKILL.md               Methodology + module catalog (read this first)
  MAINTENANCE.md         Maintainer guide (numbering policy, linter)
  VERSION                Current release date
  modules.json           Authoritative module catalog (source of truth)
  references/            Per-module investigation playbooks (8 categories)
```

Maintainer-only tooling (`tools/`, `.github/`, `CHANGELOG.md`) is gitignored and
lives outside the published plugin.

---

## Quality gates

The skill ships with a drift linter (maintainer-only tooling, kept outside the
published plugin) that validates six invariants:

1. Every backticked `references/...` path resolves to a real file.
2. Each reference file's H1 module number matches the SKILL.md catalog.
3. Modules listed under "Live Discovery" in SKILL.md have a `## Step 0: Live
   Discovery` section, and vice versa.
4. Every `/audit` argument name maps to a real catalog module, and every
   catalog module has at least one `/audit` name.
5. Every finding-producing module's output template contains a
   `Repository commit:` header, a `## Final Recommendation` section, and a
   `## Decision Summary` section.
6. The SKILL.md catalog block (between `<!-- BEGIN: modules-catalog -->`
   markers) matches what `tools/render-from-json.ps1` would render from
   `modules.json`.

Run locally against the skill root:

```
node tools/lint-drift.mjs --skill-root skills/codebase-audit
```

---

## Maintenance

Maintainer notes: see [MAINTENANCE.md](./skills/codebase-audit/MAINTENANCE.md)
for the numbering policy, the renderer workflow, the renumbering checklist, and
CHANGELOG conventions.

---

## License

Source-available under PolyForm Shield 1.0.0.

Use is free for any purpose, commercial or not — including running the skill
on commercial codebases. You may modify and redistribute.

What is NOT permitted: using this skill in or as part of a competing product
or service. Specifically, you may not repackage, rebrand, or resell it as
your own auditing product.

For licensing exceptions, open an issue at
<https://github.com/kevinpatrickrobbins/codebase-audit/issues>.

Full text: [`LICENSE`](./LICENSE.md).
