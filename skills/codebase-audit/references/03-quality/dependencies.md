# 08 — Dependencies & Supply Chain

Audit the project's dependency graph for **outdated**, **end-of-life**,
**known-vulnerable**, **abandoned**, and **license-risky** packages, and
assess the project's supply-chain posture (lockfile hygiene, SBOM, signed
artifacts).

Output: `/docs/audits/dependencies-audit.md`

---

## Cross-module relationship

- **Security** (`02-compliance/security.md`) flags known CVEs at the high
  level and includes a "dependencies" sub-step. This module goes deeper
  on supply-chain hygiene.
- **DevOps** (`05-operations/devops.md`) covers build reproducibility and
  lockfile presence. This module audits the *currency and quality* of
  what's pinned.
- **Cost** (`05-operations/cost.md`) audits SaaS subscriptions; this module
  audits open-source dependencies.

---

## Step 0: Live Discovery

This module is one of the most rot-prone in the skill — runtime EOL dates,
latest dep versions, and CVE state change daily. Fetch current truth before
comparing.

### Runtime EOL schedules

For each runtime detected in the codebase, fetch the current EOL state:

- **Node.js:** Fetch https://nodejs.org/en/about/previous-releases — find
  the current Active LTS, Maintenance, and EOL versions. Or use the
  `endoflife.date` API: Fetch https://endoflife.date/api/nodejs.json
- **Python:** Fetch https://devguide.python.org/versions/ — current
  supported versions and EOL dates. Or https://endoflife.date/api/python.json
- **Ruby / Go / PHP / Java / .NET / etc.:** https://endoflife.date is the
  authoritative aggregator: Fetch https://endoflife.date/api/<lang>.json
- **Generic fallback:** Search "<runtime> EOL schedule"

Cite the fetched table date and the specific source URL in findings.

### Latest dep versions

For each direct dependency in the manifest:

- **npm / pnpm / yarn (Node):** `Bash` `npm view <pkg> version` returns
  latest stable. `npm view <pkg> versions --json` gives history.
  `npm view <pkg> deprecated` returns deprecation notice if any.
- **pip (Python):** `Bash` `pip index versions <pkg>` (or `python -m pip
  index versions <pkg>`).
- **Cargo (Rust):** `Bash` `cargo search <pkg> --limit 1` or query
  https://crates.io/api/v1/crates/<pkg>.
- **Go:** `Bash` `go list -m -versions <module>`.
- **gem (Ruby):** `Bash` `gem search ^<pkg>$ -e` or query rubygems.org API.

Capture the latest version per dep and compute "majors behind" relative to
what the lockfile pins.

### Known vulnerabilities

- **`npm audit --json`** if Node — captures everything in the lockfile against
  the current advisory database. Only works with network access; the result is
  a snapshot at fetch time.
- **`pip-audit`** for Python (https://pypi.org/project/pip-audit/) if
  installed.
- **`cargo audit`** for Rust if installed.
- **GitHub advisories:** `gh api` `/repos/{owner}/{repo}/dependabot/alerts`
  if `gh` is authenticated and the repo has Dependabot.
- **Known-bad-version table:** Fetch https://github.com/advisories or
  per-ecosystem advisory feeds (npm, PyPI, etc.) for live patterns.

If audit tools cannot run (no network / no install), fall back to the embedded
known-vulnerable patterns in Step 4 below — and clearly note that those
patterns reflect the skill's last update, not live state.

### Currency of frameworks

For major frontend frameworks the project uses (Next.js, React, Vue, Svelte,
Angular, etc.), fetch current stable version:

- `mcp__plugin_context7_context7__resolve-library-id` then `query-docs` for the
  framework — returns current docs which surface current version.
- Or `Bash` `npm view <framework> version`.
- Or Fetch the framework's official site.

Report each framework's "current vs latest stable" in findings.

### Output of discovery

Before diving into the inventory, emit a short discovery preamble in the
audit output:

```markdown
## Live Discovery (fetched YYYY-MM-DD)

| Source | Discovered |
|---|---|
| Node.js LTS schedule (nodejs.org) | Active LTS: 22.x; Maintenance: 20.x; EOL: ≤ 18.x |
| Python EOL (devguide) | EOL: ≤ 3.9; Security-only: 3.10; Bugfix: 3.11+ |
| `next` latest (npm) | 15.5.0 |
| `react` latest (npm) | 19.0.0 |
| `npm audit` summary | 0 critical, 2 high, 12 moderate |

Findings below are calibrated against this state.
```

If discovery fails (no network), note explicitly:

```markdown
## Live Discovery — UNAVAILABLE

Network access not available in this audit run. Findings rely on the skill's
embedded knowledge as of <skill last-updated date> and may be stale.
Recommend re-running this module with network access.
```

---

## Step 1: Dependency manifest inventory

Identify package manifests:

- **Node:** `package.json` + lockfile (`package-lock.json` /
  `pnpm-lock.yaml` / `yarn.lock` / `bun.lockb`)
- **Python:** `pyproject.toml` / `requirements*.txt` / `Pipfile.lock` /
  `poetry.lock` / `uv.lock`
- **Rust:** `Cargo.toml` + `Cargo.lock`
- **Go:** `go.mod` + `go.sum`
- **Ruby:** `Gemfile` + `Gemfile.lock`
- **Java/Kotlin:** `pom.xml` / `build.gradle*` + lockfile
- **PHP:** `composer.json` + `composer.lock`
- **iOS:** `Podfile.lock` / `Package.resolved`
- **Android:** `gradle.lockfile` if used

Note any project missing a lockfile — non-reproducible builds.

---

## Step 1.5: Vendored / embedded third-party code

Declared dependency manifests are only one supply-chain surface. Many projects
ship third-party code **vendored directly into the source tree** — copied in
rather than installed from a registry. This code is invisible to `npm audit`,
`pip-audit`, Dependabot, and Renovate, can carry years-old CVEs, and often
diverges silently from upstream.

This step matters especially for:

- Legacy / monorepo takeovers where the boundary between project code and
  vendored deps has eroded.
- Native / mobile codebases (C, C++, iOS, Android) where vendoring is
  conventional and package managers cover only part of the surface.
- Drivers, middleware, or SDKs that historically vendor upstream libraries
  (FFmpeg, OpenSSL, Boost, zlib, libjpeg, sqlite, etc.).
- Hardware / firmware projects with multiple language toolchains.

If none of these apply and a quick scan reveals no candidate directories,
skip this step and note "no vendored code detected" in the output.

### Detection heuristics

Directory names that signal vendored code:

- Generic: `vendor/`, `vendored/`, `third_party/`, `3rdparty/`, `external/`,
  `externals/`, `libs/`, `Libraries/`, `deps/`, `dependencies/`, `extern/`,
  `packages/local/`, `submodules/` (when not actual git submodules)
- Named after the upstream directly: `ffmpeg/`, `openssl/`, `boost/`,
  `sqlite/`, `zlib/`, `libpng/`, `curl/`, `lua/`, `libsodium/`, etc.

Other in-file signals:

- A `LICENSE`, `COPYING`, `COPYRIGHT`, or `NOTICE` file inside a
  subdirectory whose terms differ from the project's own license.
- Per-subdirectory `README`, `CHANGELOG`, or `VERSION` files identifying
  an upstream project and pinned version.
- Source files whose copyright headers name a different organization.
- Build artifacts committed to source: `build/`, `dist/`, `Debug/`,
  `Release/`, `obj/`, `bin/`, `target/`, `.ipch/`, plus compiled binaries
  (`*.o`, `*.lib`, `*.a`, `*.exe`, `*.dll`, `*.so`, `*.dylib`) and
  checked-in minified bundles (`*.min.js`, committed `node_modules/`).

### What to record per vendored module

For each detected vendored library:

- **Name and version.** From `VERSION` / `version.h` / `package.json`
  inside the dir, header copyright lines, or `git log` of the original
  project.
- **Last sync date.** `git log -- <vendored-path>` reveals when the
  vendored copy was last refreshed.
- **Upstream current version.** `gh api repos/<owner>/<repo>/releases/latest`
  or the upstream registry.
- **Known CVEs for the pinned version.** Search "<library> <version>
  CVE", or query the GitHub Advisory Database.
- **Divergence.** Has the project modified the vendored code? `git log
  -- <vendored-path>` shows project-side commits; diffing against the
  upstream tag at the recorded sync point shows the actual delta.
- **Why vendored.** Is there a documented reason (license, patch needed,
  no registry support, build-system reasons)? Absent justification, the
  vendoring is likely accidental drift.

### Four-level verdict per vendored module

| Verdict | Meaning |
|---|---|
| **Core asset** | The project owns this code, even if it forked from upstream years ago. Keep it, document the origin in `THIRD_PARTY.md`, treat it as project code from here on. |
| **Extract & merge** | Vendored only because the registry couldn't cover it at the time. Today there's a published version — switch to declared dep and delete the vendored copy. |
| **Rebuild** | Vendored library is stale, has CVEs, and may carry local modifications. Refresh from current upstream and port any required patches forward. |
| **Deprecate** | Vendored library is no longer used (dead code) or has been replaced by another dep. Delete it. |

### Committed build artifacts

A subset of vendored findings — committed output that should be ignored:

- `node_modules/` committed
- Compiled binaries (`*.exe`, `*.dll`, `*.so`, `*.dylib`, `*.a`, `*.o`)
  committed alongside source
- Minified bundles checked in when source is also present
- Multi-megabyte `Debug/`, `obj/`, `bin/`, `.ipch/` directories

Symptoms: bloated clones, broken developer onboarding ("just regenerate it"
doesn't work because nothing regenerates), CI building on top of committed
artifacts. List size per directory and recommend `.gitignore` + (when the
historical bloat is significant) `git filter-repo` to scrub history.

### Output additions

Append to the audit document:

```markdown
## Vendored / embedded third-party code

| Path | Library | Pinned version | Upstream current | Last sync | CVEs in pinned | Modified? | Verdict |
|------|---------|----------------|------------------|-----------|----------------|-----------|---------|
| `third_party/sqlite` | sqlite-amalgamation | 3.36.0 | 3.46.1 | 2021-08 | CVE-2023-7104 | no | Rebuild |
| `vendor/ffmpeg` | FFmpeg | 2.8.x | 7.1 | 2015-11 | many | yes | Rebuild |
| `libs/internal-utils` | (forked from `utils-js`) | n/a | n/a | 2019 | n/a | yes | Core asset |

### Committed build artifacts

| Path | Size | Recommendation |
|------|------|----------------|
| `node_modules/` | 412 MB | gitignore + history rewrite |
| `Debug/` | 89 MB | gitignore + delete from HEAD |
```

---

## Step 2: Outdated packages

For each direct dependency:

- **Major versions behind.** A dep on version 4.x when 7.x is stable is
  "behind"; how far is it?
- **Last published.** When was the version in use last released? Old
  releases of currently-active projects are normal; old releases of
  inactive projects are abandonment risk.
- **Last project commit.** When did upstream last commit to the repo? More
  than 12 months without commits and no recent release = abandoned.

Heuristics (from code, not from running tools):

- The lockfile timestamp tells you when deps were last updated. If the
  lockfile was last touched > 6 months ago, the project is behind on
  patches.
- Explicit version pins (`"^4.17.21"`) vs floating (`"*"`) — the latter
  makes builds non-reproducible.

Recommend running `npm outdated` / `pnpm outdated` / `pip list --outdated`
and capturing output, since live network access is needed for the
authoritative answer.

---

## Step 3: End-of-life runtimes

- **Node.js** versions go end-of-life on a published schedule (Active LTS,
  Maintenance, EOL). Check `engines.node` in `package.json`, `.nvmrc`,
  `Dockerfile FROM node:` lines.
- **Python** EOL: 3.8 EOL Oct 2024; 3.9 EOL Oct 2025. Check
  `python_requires`, `Dockerfile FROM python:` lines, CI matrix.
- **Ruby** — ~3 years from release.
- **Go** — N-2 supported; older versions get no security patches.
- **Java** — LTS schedule (8, 11, 17, 21).
- **PHP** — short cycle; check `php` directive in `composer.json`.

Running on an EOL runtime is a Critical security finding regardless of
whether other dependencies are current.

---

## Step 4: Known vulnerabilities (CVEs)

You typically can't run `npm audit` from a static audit, but you can flag:

- **Notoriously vulnerable old versions** present in the lockfile:
  - `lodash` < 4.17.21 (prototype pollution)
  - `axios` < 0.21.2 (SSRF), < 1.6.0 (CSRF token)
  - `node-fetch` < 2.6.7 (DNS rebinding)
  - `minimist` < 1.2.6
  - `tar`, `node-ipc`, `event-stream` historical incidents
  - `xz-utils` < 5.6.2 (CVE-2024-3094 backdoor) — for any project pulling
    upstream Linux base images
- **Dependabot / Renovate** integration — if the project has either, it's
  likely staying current; if neither, manual currency is the failure mode.

Recommend the actual fresh `audit` run and capture output as evidence.

---

## Step 5: Abandoned & deprecated

- **Abandoned packages.** Search for known-abandoned: `request` (Node), Vue
  2 in active use post-EOL, AngularJS, packages with fewer than 10 commits
  ever and last update > 2 years ago.
- **Deprecated by author.** `npm view <pkg> deprecated` returns deprecation
  notices.
- **Renamed packages.** e.g. `node-uuid` → `uuid`, `babel-*` → `@babel/*`.

A long tail of small abandoned utility packages is normal noise; a
business-critical dep that's abandoned is a finding.

---

## Step 6: Transitive bloat

- **Dependency count.** A `package.json` with 30 deps and a `node_modules`
  with 1500 packages is one of those projects. Run `du -sh node_modules`
  mentally — > 500MB on a small project signals deep transitive trees.
- **Bundle weight on client side** (overlap with module 10 — speed). A
  client bundle pulling all of `lodash`, all of `moment`, all of `aws-sdk`
  v2, etc.
- **Duplicate packages** — same package at multiple versions in the tree
  due to peer-dep mismatches. `npm dedupe` can sometimes help; the
  underlying cause is what matters.

---

## Step 7: License compliance

- **License field** in `package.json` reviewed for compatibility with the
  project's distribution license.
- **GPL / AGPL** transitively included can force the project's own license.
- **Non-permissive** licenses sneaked into deps (BSL, SSPL — not OSI-approved
  in some cases).
- **License audit tools** — `license-checker` (Node), `pip-licenses`,
  `cargo-license`. Recommend running.

A startup taking on enterprise customers will be asked for license
inventory; absence is a sales-cycle drag.

---

## Step 8: Supply-chain hardening

- **Lockfile committed and verified in CI.** `npm ci` (not `npm install`)
  to enforce.
- **Integrity hashes** in lockfile (default for npm/pnpm/yarn lockfiles
  since v2+).
- **Dependency confusion / typosquatting.** Search for very-recent additions
  with names suspiciously close to popular packages.
- **Direct git dependencies** (`"some-pkg": "github:user/repo"`) — pinned
  to a SHA, not a branch?
- **Private registries.** If the project has private packages, `.npmrc` /
  `.pip.conf` correct?
- **SBOM (Software Bill of Materials).** SPDX or CycloneDX format generated?
  Any procurement-conscious customer will eventually ask. Tools: `syft`,
  `cyclonedx-bom`.
- **Signed artifacts.** Releases signed (Sigstore, GPG)?
- **Provenance.** SLSA level — for supply-chain-aware customers, having a
  SLSA Level 3 build process is increasingly expected.

---

## Step 9: Automation

- **Dependabot** (GitHub) or **Renovate** configured? Cadence reasonable
  (weekly minor, immediate security)?
- **Auto-merge for low-risk updates** (patch versions of known-safe deps,
  with passing CI)?
- **Stale dep PRs** sitting open — sign of automation without follow-through.
- **CI security audit step.** `npm audit --audit-level=high` failing the
  build, or advisory-only?

---

## Red flags

- Lockfile last modified > 6 months ago.
- Node 16.x, Python 3.8/3.9, or other EOL runtime in production.
- Direct dep > 3 majors behind upstream.
- `package.json` floating versions (`"*"`).
- `npm install` (not `npm ci`) in CI.
- No Dependabot / Renovate config.
- GPL/AGPL transitive deps in a closed-source product.
- Direct git dependencies pinned to a branch (not SHA).
- > 50 stale dependency PRs open.
- No SBOM despite enterprise procurement context.
- Vendored third-party libraries with no documented sync cadence.
- Vendored libraries pinned to a version with known CVEs.
- `node_modules/` or other build output committed to source.

---

## Output structure

```markdown
# Dependencies & Supply Chain Audit

Date: YYYY-MM-DD
Repository commit: <hash>

## Manifest inventory
- Package manager: <npm/pnpm/yarn/bun/pip/cargo/etc.>
- Lockfile: present? committed? last modified?

## Runtime currency
- Node / Python / Ruby / Go / etc. — version vs current support matrix
- EOL status

## Direct dependency currency
| Package | Current | Latest | Behind | Last published | Risk |

## Known-vulnerable patterns detected
(table — package, version, CVE / known issue, severity, fix)

## Abandoned / deprecated
(list)

## Transitive shape
- Dependency count
- node_modules size (if Node)
- Duplicate-version count
- Bundle weight (cross-ref module 10 — speed)

## License posture
- License inventory tool present?
- Incompatible licenses detected (GPL/AGPL in closed-source)
- License field hygiene

## Supply-chain hardening
- Lockfile integrity hashes
- CI uses install-from-lockfile (npm ci / equivalent)
- Direct git deps pinned to SHA
- Private registry config
- SBOM generation
- Signed artifacts
- SLSA level (if relevant)

## Automation
- Dependabot / Renovate config
- Cadence and auto-merge policy
- CI audit gate (blocking vs advisory)

## Findings
(severity / category / evidence / fix)

## Final recommendation
Single highest-priority dependency action.

## Decision summary
- Runtime EOL: yes/no
- Critical CVEs detected: yes/no
- Automation in place: yes/no
- Worst gap: <description>

## Confidence
High / Medium / Low
- Live `npm audit` / `pip-audit` not run from this static analysis;
  recommend capturing output for definitive CVE list.

## Out of Scope / Inconclusive
```

---

## Severity calibration

- **Critical:** runtime EOL; high-severity CVE in production-reachable code;
  abandoned business-critical dep with no replacement plan; vendored
  library with known critical CVE in production code path.
- **High:** several deps multiple majors behind; no Dependabot/Renovate;
  GPL/AGPL contamination of closed-source; vendored library years stale
  with no documented sync plan.
- **Medium:** lockfile freshness drift; minor outdatedness; missing SBOM
  for B2B-targeting product; committed build artifacts bloating the repo.
- **Low:** version-pin formatting, deprecation warnings without exploit.

Every finding cites the lockfile / manifest line, or — for vendored
code — the directory path and the upstream-identifying file (`VERSION`,
`LICENSE`, copyright header).
