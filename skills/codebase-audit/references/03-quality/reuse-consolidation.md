# 23 — Reuse & Consolidation

Find code that solves the **same problem more than one way** — functions, hooks,
helpers, or query shapes that share an *intent* but differ in implementation or
name — and recommend collapsing each cluster to a single canonical version.

Create or update:

`/docs/audits/reuse-consolidation-audit.md`

---

This is the **deep** duplication pass. Module 05 (Architecture Integrity),
`references/03-quality/architecture.md` Step 6, does a fast architectural scan
keyed on *shared names* (`find references` on `Button`, `format`, `validate`).
That misses the case that matters most: three functions doing the same job under
three different names, sharing no tokens. Copy-paste detectors (`jscpd`,
`eslint-plugin-sonarjs` `no-identical-functions`) miss it too — they match text,
and these implementations differ textually. Catching "same job, different code"
is a *reading* problem, and that is what this module owns.

This is a **maintainability** concern, not a performance one. The goal is one
source of truth per behavior — fewer places to change, no silent drift between
variants. If consolidation also turns out to be faster or smaller, note it, but
do not confuse this with the optimization modules (09 Scalability, 10 Speed).

## How to investigate

### Step 1: Enumerate the verb surface

Inventory the code that *does something* — exported functions, class methods,
route/RPC handlers, React hooks, shared utilities. Skip pure types, constants,
and config; those are compared differently. Use the language server's symbol
index, `ctags`, or grep for declaration forms (`export function`,
`export const \w+ =`, `def `, `func `).

On a large codebase, scope to the highest-churn or highest-risk areas first
(auth, money, data access, formatting) rather than boiling the ocean.

### Step 2: Bucket by behavioral domain

Comparing every function to every other is O(n²) and hopeless. Group candidates
into behavioral buckets so comparison stays local. Derive the buckets from *this*
codebase; the usual suspects:

- date / time formatting and parsing
- currency / number / unit formatting
- string work: slugify, truncate, casing, sanitize
- validation (email, phone, IDs, schema-shaped input)
- fetch / HTTP client wrappers, retry / backoff
- auth / permission / entitlement checks
- data access: repeated query shapes, mappers, serializers
- error mapping, logging wrappers
- ID / token / hash generation
- pagination, sorting, filtering helpers

### Step 3: Write an intent signature per candidate

For each candidate, write one line describing *what it does* — input → output or
effect — deliberately ignoring its name and body. "Date → localized display
string." "Raw user record → safe public JSON." Two candidates with the same
intent signature but different names or implementations are a **consolidation
candidate**. This normalization to intent is the move that surfaces what
name-keyed and token-based scans cannot.

### Step 4: Confirm equivalence before flagging

This is the step that keeps the pass safe. Before recommending a merge, verify
the implementations actually serve the *same* intent across inputs and edge
cases. Two `formatDate` variants may differ in timezone handling, locale
default, or null behavior — and that difference may be **load-bearing**. If so,
the finding is *"divergent — reconcile intent deliberately,"* not *"duplicate —
delete one."* A blind merge that erases a real behavioral difference is worse
than the duplication it removes.

### Step 5: Choose the canonical version and its home

For each true cluster, recommend which implementation survives — the most
complete, correct, and best-tested — and where it belongs. In a monorepo that's
usually a shared package; cross-package reimplementations of one behavior are
the highest-value finds. Count call sites per variant (`find references`) to
size the migration, and rank the cluster by blast radius: a 30-call-site
duplication earns more attention than a 2-call-site one.

### Step 6: Tooling assist

Reading owns the semantic half; let tools own the mechanical half. Run `jscpd`
(or the project's existing duplication tooling) to catch copy-paste clones the
reading pass may skip, and fold its hits into the same clusters. State the split
explicitly so a reader knows the report covers both — textual clones (tool) and
divergent reimplementations (reading).

## Red flags

- The same behavior implemented under two or three different names.
- Per-feature reimplementations of a cross-cutting concern (each module ships
  its own `formatPrice` / `apiFetch` / `isValidEmail`).
- `utils2`, `helpers-new`, `lib/legacy` living beside the originals.
- The same `where` / `include` query shape pasted across many call sites.
- The same regex or magic constant redefined in several files.
- A hand-rolled validator next to a schema (`zod` / `valibot`) that already
  describes the same shape.

## Output structure

```markdown
# Reuse & Consolidation Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Summary
- <N> consolidation clusters across <M> behavioral domains
- <N> high-blast-radius (≥ <k> call sites), <N> divergent (needs reconciliation)
- Tooling: jscpd run? yes/no; clones folded in: <N>

## Clusters
For each cluster:

### <Intent — e.g. "Format a Date for display">
- **Implementations:**
  - `path/to/a.ts:LL` — `formatDate` (<call sites>)
  - `path/to/b.ts:LL` — `prettyDate` (<call sites>)
  - `path/to/c.tsx:LL` — inline `toLocaleDateString` (<call sites>)
- **Equivalence:** identical | equivalent | divergent (<what differs; load-bearing?>)
- **Recommended canonical:** `<which one>` in `<which package/module>` — why
- **Consolidation effort:** <low/med/high> (<N total call sites to migrate>)
- **Risk if left:** <drift, inconsistent behavior, repeated bugs>

## Final Recommendation
The single highest-leverage consolidation to do first. **One cluster**, with the
reason (usually highest call-site count × clearest correctness risk).

## Decision Summary
- Consolidation debt level: low / moderate / high
- Worst cluster: <intent + site count>
- Recommended posture: <consolidate now | consolidate opportunistically | leave>

## Confidence
High / Medium / Low (and why — e.g. equivalence judged by reading, not executing)

## Out of Scope / Inconclusive
Clusters where equivalence couldn't be confirmed without running the code or
domain knowledge the audit lacks.
```

## Severity calibration

- **Critical:** divergent implementations of one intent that actively cause bugs
  — two permission checks that disagree, two money-rounding helpers that produce
  different totals.
- **High:** a cluster whose consolidation would retire a recurring class of bugs,
  or with a large call-site count where drift is likely.
- **Medium:** clear duplication with no current harm (the classic
  `formatDate` / `prettyDate` drift).
- **Low:** near-duplicates arguably fine to keep separate; cosmetic.

## Evidence requirements

Every cluster lists at least two implementation sites with `file:line`. The
equivalence verdict is justified, not asserted — name what is the same and what
differs. A "consolidate" recommendation must name the canonical version and why
it wins. Avoid taste-based merges: if the only argument is "I'd write it
differently," it is not a finding.
