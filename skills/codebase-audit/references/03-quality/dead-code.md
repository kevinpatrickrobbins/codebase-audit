# 25 — Dead Code & Unused Surface

Find code that is **never executed and never referenced** — unused exports,
orphaned files, unreachable branches, dead dependencies, and abandoned
feature surface — and recommend deletion.

Create or update:

`/docs/audits/dead-code-audit.md`

---

## Relationship to neighboring modules

- **Module 23 (Reuse & Consolidation)** finds code that does the same job
  multiple ways and is still **live** — called from more than one place.
  This module finds code that is called from **nowhere**.
- **Module 24 (Workarounds & Root-Cause Gaps)** finds code that actively
  executes but sidesteps a problem. This module finds code that doesn't
  execute at all.
- This module **deepens** Module 05 (Architecture Integrity) Step 6
  (duplication scan) and Step 10 (`"Legacy"`/`"v2"`/`"new-"` directories,
  commented-out code red flags), which only spot the *pattern* at a
  glance. This module confirms whether the candidate is actually
  unreferenced and sizes the deletion — the same relationship Modules 23
  and 24 already have to Module 05.

This is a **mechanical-first** pass — the inverse of Module 23's
reading-first approach. Whether code is referenced anywhere in the
reachable graph is a property a tool can prove, not a judgment call.
Reading still matters for what static analysis structurally can't see:
dynamic dispatch, string-based routing, reflection, and entry points a
tool doesn't know about.

## Step 0: Live Discovery

The unused-code tooling landscape moves — a tool considered current can
become maintenance-only within a year. Verify before recommending.

- **JS/TS:** confirm whether `knip` is still the maintained leader, or
  check for a successor. As of this skill's last update, `ts-prune` is in
  maintenance mode (no further development), superseded by `knip` — which
  additionally finds unused files, dependencies, and class/enum members,
  and ships 150+ framework plugins (Next.js, Vite, Nx, Astro, etc.) so it
  understands framework-specific entry points out of the box. Fetch the
  current `knip` repo/docs or search "knip vs ts-prune `<year>`" to
  confirm currency before citing either by name.
- **Python:** `vulture` is the long-standing tool; `deadcode` is a newer
  alternative with auto-fix and improved scope/namespace tracking. Check
  release activity on both before recommending one over the other.
- **Go:** `golang.org/x/tools/cmd/deadcode`, or `staticcheck`'s `U1000`
  check.
- **Rust:** the compiler's own `dead_code` lint (on by default) catches
  unreferenced items within a crate; `cargo-udeps` (nightly) finds unused
  *Cargo dependencies* — a different scope, both relevant.
- **Ruby:** `debride`, or a custom `rubocop` cop.
- **Ecosystem-agnostic file-level fallback:** a reverse-import graph
  (every file, who imports it) built from the language server, or
  `madge` / `dependency-cruiser` (JS), finds orphaned files even without
  a language-specific dead-code tool.

If discovery is unavailable (no network), fall back to the tool names
above with a `may be stale, last verified: <skill update date>` caveat —
never assert a tool is current without checking.

## How to investigate

### Step 1: Run the mechanical sweep

This is the part a tool does better than reading. Prefer the project's
already-configured tool if one exists (check `package.json`
devDependencies/scripts, `pyproject.toml`, CI config); otherwise run the
appropriate tool from Step 0, read-only — no `--fix` / auto-remove flags
during an audit. Flag candidates; do not delete.

For a JS/TS monorepo, run `knip` from the workspace root so it sees
cross-package references — a per-package run produces false positives for
anything imported only by a sibling workspace package.

Capture:

- Unused files (never imported by any reachable entry point)
- Unused exports (exported but never imported elsewhere)
- Unused dependencies (declared in the manifest, never imported)
- Unused class members / enum members (language-server-assisted tools only)

### Step 2: Triage tool output before trusting it

Unused-code tools produce false positives for code reached through paths
their static analysis can't see:

- **Entry points the tool doesn't know about** — serverless/edge handler
  exports (`fetch`/`scheduled` on Cloudflare Workers, Lambda handlers),
  CLI bin scripts, dynamically-`import()`ed code, code referenced only
  from a config file (`next.config.js` plugin lists, ESLint plugin
  registration).
- **String-based / reflective dispatch** — a route registered by string
  name and looked up at runtime, a class instantiated via a
  dependency-injection container, a job handler dispatched by a `type`
  field in a queue payload, a block/component registry keyed by a string
  type.
- **Public package API.** For a published package (vs. an app), "unused
  within this repo" isn't the same as "unused" — external consumers may
  import it. Check whether the candidate is part of the package's
  declared public exports (`package.json` `exports` field, `index.ts`
  barrel) before flagging.
- **Test-only usage.** Code only referenced from tests is a judgment
  call: dead production code kept alive only by its test, or a
  legitimately test-only helper? Note which.

Most unused-code tools (knip especially) support config to declare entry
points and ignore patterns — check whether the project's config
(`knip.json`, a `.vulture-whitelist`) already encodes these exceptions,
since an unconfigured first run over-reports.

### Step 3: Confirm each candidate by reading

For each candidate the tool surfaces, after Step 2 triage removes the
structural false positives, confirm with a targeted search:

- Find-references for the exact symbol name across the whole repo,
  including non-source files (config, docs that reference it as an API
  name, `*.test.*`).
- For a file-level candidate, check whether anything dynamically
  constructs its path (`import(\`./blocks/${type}\`)` patterns — common
  in plugin/block-registry architectures).
- Check the file's commit history. A file with no commits in over a year
  and zero references is a stronger candidate than one just refactored
  last week, which might be mid-migration with the old call sites about
  to be removed in a follow-up commit.

### Step 4: Classify and size

For each confirmed-dead candidate:

- **Verdict:** dead (delete) | grandfathered (kept for a documented
  reason — name it) | needs-author-context (can't confirm without
  history the audit lacks).
- **Size:** LOC for files; note how many *other* dead items depend on
  this one — a dead file that only dead files import is a bigger single
  deletion than it first looks.
- **Cascade.** Does removing this candidate orphan something else (the
  last caller of a util makes the util itself dead)? Re-running the
  sweep after a deletion round typically surfaces a second wave.

### Step 5: Look beyond exports — unreachable code and abandoned surface

The mechanical sweep catches unreferenced *symbols*. These adjacent
dead-code shapes need reading:

- **Unreachable branches.** Code after an unconditional
  `return`/`throw`/`break`; an `if` whose condition is a constant; a
  `switch` case that can never match.
- **Disabled-and-orphaned feature flags.** A flag permanently off in
  every environment, with the code path behind it never reachable —
  confirm against the flag's actual runtime values (env config, a
  `feature_flags` table), not just its default.
- **Parallel "v2"/"legacy" implementations from an incomplete
  migration**, where the old path is provably unreachable (the router
  never routes to it, no remaining caller) rather than just *suspiciously
  named*. Module 05 flags the directory pattern; this step confirms
  whether it's actually dead or still load-bearing.
- **Commented-out code blocks.** Module 05 Step 10 counts these; this
  module's verdict is binary — does a live equivalent already cover what
  the comment did? If yes, delete. If the comment is the only record of
  intent, that's a documentation gap, not a dead-code finding.
- **Dead migrations/schema for removed features** — a DB column, table,
  or API field with zero application code reading or writing it.
  Cross-reference Module 09 (Performance & Scalability) if removing it
  requires a migration.

## Red flags

- A tool-detected unused-export count from a codebase that's never run
  the sweep before (no `knip.json` / equivalent config present) — first
  runs on a mature codebase often surface dozens to hundreds of hits;
  that volume is itself a finding about hygiene practice, not just the
  individual items.
- Files with zero inbound references and no commits in the last year.
- A `legacy/`, `v2/`, `old-*`, `*-deprecated` directory still present
  after the migration it names has shipped.
- Feature-flag-gated code where the flag has held one fixed value in
  every environment for months.
- Exported symbols from a barrel `index.ts` that nothing outside the
  package imports.
- Dependencies in the manifest with zero import statements anywhere in
  source.

## Output structure

```markdown
# Dead Code & Unused Surface Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Summary
- Tooling run: <knip/vulture/deadcode/etc.> v<version>, <N> raw hits
- <N> confirmed dead after triage + reading, <N> false positives explained,
  <N> needs-author-context
- Largest single deletion opportunity: <file/module, LOC>

## Findings
For each confirmed candidate:

### <Symbol or file path>
- **Location:** `path/to/file.ts:LL`
- **Kind:** unused file | unused export | unused dependency | unreachable
  branch | dead feature flag | orphaned migration
- **Tool/method:** <knip hit | manual grep | git log evidence>
- **Verdict:** dead — delete | grandfathered — keep (reason) |
  needs-author-context
- **Cascade:** deleting this also orphans: <list, if any>
- **Risk if left:** <onboarding confusion, accidental edits to dead code,
  bundle/binary size, false sense of coverage>

## Tooling false positives explained
What the sweep flagged that is NOT dead, and why (dynamic dispatch,
public API, config-referenced) — this builds the ignore-list for the
next run.

## Final Recommendation
The single largest-blast-radius deletion to do first. **One thing.**

## Decision Summary
- Dead-code volume: low / moderate / high
- Worst offender: <path + size>
- Recommended posture: <delete now | delete opportunistically as touched |
  needs author confirmation first>

## Confidence
High / Medium / Low (and why — confidence is lower wherever dynamic
dispatch or reflection makes static analysis unreliable)

## Out of Scope / Inconclusive
Candidates that need the author's intent, or that the tooling couldn't
reach (e.g., no tool installed for this ecosystem; recommend installing
and re-running).
```

## Severity calibration

- **Critical:** dead code on a security-sensitive path that's been
  silently disabled — e.g., a feature-flagged auth check that's actually
  unreachable, masking a security gap as "covered" when it isn't.
- **High:** large-blast-radius dead surface (a whole unused module, an
  abandoned migration path) that actively confuses maintainers or bloats
  the deployed bundle/binary.
- **Medium:** confirmed-dead individual files/exports with no immediate
  harm beyond clutter.
- **Low:** small unused exports, commented-out snippets, cosmetic.

## Evidence requirements

Every finding names the tool or method that confirmed deadness (not just
"looks unused") and cites `file:line`. A "dead" verdict on a candidate
that turns out to be a false positive (dynamic dispatch, public API) is a
defect in the audit — Step 2/3 triage exists specifically to prevent
this. "Grandfathered" verdicts must name the actual reason it's kept, not
just assert one.
