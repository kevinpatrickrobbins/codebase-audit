# 24 — Workarounds & Root-Cause Gaps

Find **band-aids** — code that sidesteps a problem instead of fixing its root
cause — and, for each, name the real fix that was skipped. This is the
codebase-facing version of the solution-vs-workaround question: not "is there a
bug" but "was a problem *worked around* rather than solved."

Create or update:

`/docs/audits/workarounds-audit.md`

---

This supersedes the shallow end of Module 05 (Architecture Integrity) Step 10,
which only *counts* `TODO` / `FIXME` / `HACK` markers. This module does the
analysis those markers hint at: what problem is being avoided, whether a real
fix exists, and what it is. It is distinct from Module 23 (Reuse &
Consolidation), though they overlap — sometimes the workaround *is* duplication
(copy the code rather than refactor the shared path).

The discipline that makes this useful is restraint: a defensive pattern with a
good reason is correct engineering, not a workaround. The job is to separate *"a
real fix was skipped"* from *"this is the right call"* — and to make the
**silent** workarounds visible, because those are the ones nobody decided to
keep.

## How to investigate

### Step 1: Signal scan

Grep for the patterns that usually mark avoidance. Each is a *candidate*, not a
verdict.

- **Type / lint suppressions:** `@ts-ignore`, `@ts-expect-error` (especially with
  no explanation), `eslint-disable`, `as any`, `as unknown as`, non-null `!`,
  `// @ts-nocheck`.
- **"Temporary" language:** `HACK`, `FIXME`, `XXX`, `TEMP`, `KLUDGE`, and prose
  like "for now," "until we," "remove this once," "revisit."
- **Swallowed failures:** empty `catch {}`, `catch (e) { return null|[]|{} }`,
  broad catches with no rethrow or report.
- **Timing band-aids:** `setTimeout(fn, 0)` or arbitrary delays to dodge a race;
  `sleep` / fixed waits in tests; polling where an event or callback exists.
- **Special-casing:** hardcoded `if (id === '...')` / tenant / env checks that
  paper over a missing general mechanism.
- **Downstream patching:** fixing a symptom in the consumer or in rendered output
  instead of the producer; client-side patching of a server bug; CSS
  `!important` fighting the cascade.
- **Disabled safeguards:** `.skip` / `.only` / `xit` tests, commented-out
  validation or assertions, file-scope disabled lint rules.
- **Dependency dodges:** a version pinned to avoid a breaking upgrade; a vendored
  copy patched in place; a shim where configuration would do.
- **Retry-as-cure:** retry / backoff wrapping a call that fails for a fixable,
  deterministic reason.

### Step 2: Infer the root cause

For each candidate, answer: *what problem is this avoiding?* `as any` on a
response → the type doesn't match reality (schema drift). `setTimeout(_, 0)`
before reading the DOM → render ordering not actually awaited. A swallowed catch
→ an error path nobody handled. The root cause is the real finding; the
workaround is just where it surfaced.

### Step 3: Classify

- **Band-aid** — a real fix exists and is feasible. Recommend it, with effort.
- **Justified** — a genuine constraint (upstream / vendor bug, platform quirk,
  deliberate trade-off) where the workaround is the right call. Keep it; note
  why, and whether it's documented.
- **Unknown** — needs the author's intent or context the audit lacks. Say so;
  don't guess.

### Step 4: Documented vs silent

A workaround commented with its reason and a ticket (`HACK: remove when upstream
#123 ships`) is *managed* debt. A silent, undocumented one is *invisible* debt —
nobody chose to keep it and nobody is tracking it. Flag silent workarounds
higher; the fix for a justified-but-silent one may simply be to document and
ticket it.

### Step 5: Don't cry wolf

Not every `try/catch`, retry, or feature flag is a workaround. Require evidence
that a real fix was *skipped* — the suppression with no explanation, the
"temporary" that `git blame` dates to two years ago, the special-case that
should be config. A report that flags every defensive line is noise and gets
ignored. A short list of genuine band-aids with the root cause named is worth
more than a hundred suppression hits.

## Red flags

- A cluster of suppressions (`as any`, `@ts-ignore`) concentrated in one file or
  module — type lies quarantined where the real model was never fixed.
- "Temporary" code that `git blame` dates to months or years ago.
- Empty catches or catch-and-default that hide failures from monitoring.
- `setTimeout` / `sleep` used as a synchronization mechanism.
- Hardcoded special-cases for specific IDs, tenants, or environments.
- Output post-processing that patches a bug the generator should fix.
- Skipped or `.only` tests with no linked ticket.

## Output structure

```markdown
# Workarounds & Root-Cause Gaps Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Summary
- <N> workarounds reviewed; <N> band-aids, <N> justified, <N> unknown
- <N> silent / undocumented, <N> documented-and-ticketed
- Highest-risk theme: <e.g. swallowed errors in the payment path>

## Findings
For each workaround:

### <Short description — e.g. "as any on the Stripe webhook payload">
- **Location:** `path/to/file.ts:LL`
- **Root cause it avoids:** <the real problem being sidestepped>
- **Classification:** band-aid | justified | unknown
- **Documented?** silent | comment-only | comment + ticket #N
- **Recommended real fix:** <what to do instead> — effort <low/med/high>
- **Risk if left:** <what breaks, when>

## Themes
Clusters where many workarounds share one root cause (e.g. "no shared error type
→ every handler swallows differently").

## Final Recommendation
The single root cause whose fix would retire the most band-aids — or the one
masking the most dangerous failure. **One thing.**

## Decision Summary
- Workaround debt level: low / moderate / high
- Most dangerous band-aid: <description + location>
- Recommended posture: <fix root causes now | document & ticket | accept>

## Confidence
High / Medium / Low (and why — classification often needs author intent)

## Out of Scope / Inconclusive
Candidates that need the author's context to classify, or that depend on runtime
behavior the audit can't observe.
```

## Severity calibration

- **Critical:** a workaround masking a security or correctness root cause — a
  swallowed auth / permission error, a disabled tenant-isolation test, an
  `as any` hiding an unvalidated external payload on a money path.
- **High:** a band-aid likely to break under load or change, or one that hides
  real failures from monitoring; silent and undocumented.
- **Medium:** a documented workaround that should still be fixed; a "temporary"
  that has outlived its excuse.
- **Low:** justified-and-documented trade-offs; cosmetic suppressions in
  non-critical paths.

## Evidence requirements

Every finding cites the workaround's `file:line` and names the root cause it
avoids. Classification is justified, not asserted — a "band-aid" verdict must say
what the real fix is; a "justified" verdict must name the constraint. The
distinction between *evidence that a fix was skipped* and *a reviewer's stylistic
preference* must be explicit; the latter is not a finding.
