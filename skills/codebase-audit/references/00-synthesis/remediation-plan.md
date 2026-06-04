# 00.3 — Remediation Plan (Ticket Emitter)

Convert all findings from the prior audit modules into discrete, atomic, actionable
tickets in the project's tracking system.

This module is **not** a duplicate of the Findings (Module 00.2). The Risk
Register is a static catalog of risks for reference. This module emits real tickets
that the team can pick up, sequence, and ship.

The expected output is two artifacts:

1. **Real tickets in the project's tracking system.** GitHub Issues (preferred) or
   `docs/TICKETS.md` (legacy projects), filed in the project's existing format per
   the `tickets-and-logbook` skill.
2. **A summary index at `/docs/audits/remediation-plan.md`** — a brief markdown
   file that lists every ticket filed (with link/ID), grouped by audit dimension,
   and notes the recommended sequence.

---

## Step 0: Compare with previous remediation plan

If a prior `/docs/audits/remediation-plan.md` exists, compute the **ticket
delta** before filing new ones — to avoid duplicate filings and to surface
ticket lifecycle progress.

**Process:**

1. Read the prior remediation-plan.md index (lists every ticket previously
   filed with issue number and source finding).
2. For each previously-filed ticket, query its current state:
   - GitHub: `gh issue view #N --json state,labels,closedAt,title`
   - TICKETS.md: search for the ticket entry
3. For each current finding, check whether a prior ticket already covers it
   (same source module + section, similar title).

**Output a "Delta" section in the new remediation-plan.md:**

```markdown
## Delta from previous run (YYYY-MM-DD → YYYY-MM-DD)

### Closed tickets since last audit (count)
- #101 — Hash passwords with argon2id — closed YYYY-MM-DD
- #102 — Add CSP header — closed YYYY-MM-DD

### Tickets still open (count)
- #105 — Add rate limit on /api/auth — status: ready (1 month old)
- #110 — Cross-tenant export check — status: in-progress (2 weeks old)
- #115 — Backup restore drill — status: proposed (3 months old, stale)

### New tickets to file this run (count)
- (full list of new ones generated from this audit)

### Resolved without ticket
- Findings from prior audit that are no longer present this audit but
  weren't closed via a tracked ticket — note for the team's awareness.

### Stale / blocked tickets requiring attention
- Tickets older than 90 days in `proposed` state — escalate or de-prioritize
- Tickets in `blocked` state — note the dependency
```

**Suppression of duplicate ticket filing:**

If a current finding matches a prior open ticket (same source + similar
title), DO NOT file a new ticket. Instead:
- Add a comment to the existing ticket: "Confirmed still present in audit
  run YYYY-MM-DD"
- Note in the remediation-plan delta that the finding "still tracked by
  prior ticket #N"

This keeps tickets durable across audit runs rather than accumulating
duplicates.

---

## Step 1: Detect the project's ticket system

Run, in order:

```bash
gh auth status      # is GitHub CLI authenticated?
git remote -v       # does the project have a github.com remote?
ls docs/TICKETS.md  # does a flat-file tracker exist?
ls docs/LOGBOOK.md  # does a logbook exist?
```

Decision tree:

- **Has `github.com` remote AND `gh` is authenticated** → file as GitHub Issues
  (preferred). Use the format from the `tickets-and-logbook` skill.
- **Has `docs/TICKETS.md` but no GitHub** → append to TICKETS.md in the project's
  existing format. Recommend running the `tickets-migration` skill to move to
  GitHub Issues going forward.
- **Has both** → recommend running `tickets-migration` first. If user declines,
  file to GitHub Issues (the canonical destination per the skill).
- **Has neither** → write all proposed tickets to `/docs/audits/remediation-tickets.md`
  as ready-to-file drafts in the standard issue body format. Note in the
  remediation-plan.md index that no tracking system was detected and the team
  should either set up GitHub Issues or adopt the tickets workflow.

If the user invoked `/audit --no-tickets`, skip filing entirely — produce the
draft markdown file regardless of what tracking system exists.

---

## Step 2: Convert findings to ticket bodies

Walk every finding from every finding-producing module that ran this audit:

- **Compliance:** 02 (security), 03 (privacy), 04 (accessibility), 22 (sector — if applicable)
- **Quality:** 05 (architecture integrity), 06 (testing), 07 (documentation), 08 (dependencies), 23 (reuse & consolidation), 24 (workarounds & root-cause gaps)
- **Performance:** 09 (scalability), 10 (speed), 11 (serverless cold-start — if applicable)
- **Operations:** 12 (devops), 13 (cost), 14 (engineering practice)
- **Product:** 15 (UX), 16 (product gaps), 17 (frontend modernization), 20 (i18n — if applicable), 21 (SEO — if applicable)
- **Domain-conditional:** 18 (AI/ML — if applicable), 19 (product-type — if applicable)

Module 01 (System Architecture) typically produces no tickets — it is a
snapshot, not a findings-bearing audit.

**One ticket = one atomic fix.** Do not bundle "fix all auth issues" into one
ticket. Do bundle a finding that requires multiple file edits as one ticket if
they all serve a single coherent fix.

For each finding, compose a ticket using the `tickets-and-logbook` body
structure:

```markdown
## Problem
<concrete, evidence-cited description from the audit finding>

Source: docs/audits/<audit-file>.md (Module <N>)
Severity: <Critical|High|Medium|Low>
Evidence: <file/path/to/source.ts:LL>

## Approach
- <step 1>
- <step 2>
- <step 3>

## Dependencies
Depends on: #N — <reason>   (or "—" if none)

## Acceptance Criteria
- [ ] <testable criterion 1>
- [ ] <testable criterion 2>
- [ ] <evidence of regression-safety, e.g., new test, manual check>

## Testing
- <how to verify criterion 1>
- <how to verify criterion 2>

## Verification Test
- <a single concrete verification that proves the fix is in place AND that
  the original weakness is closed — e.g., "POST /api/export with org_id
  belonging to a different tenant returns 403 (not 200), validated via
  integration test in test/security/cross-tenant.spec.ts">
- This is distinct from "Testing" (which proves the criteria) — this
  proves the *threat* is no longer present. For security tickets, this
  field is required.

## Logbook
—
```

**Title format:** Short imperative phrase (≤ 70 chars). Examples:

- ✅ `Hash passwords with argon2id in lib/auth/hash.ts`
- ✅ `Add CSP header to vercel.json`
- ✅ `Pin Node base image to 20.11.0-alpine in Dockerfile`
- ✅ `Move account deletion behind GDPR data export gate`
- ❌ `Security` (too vague)
- ❌ `Various security improvements` (not atomic)

---

## Step 3: Assign labels

Use the `tickets-and-logbook` two-label scheme: **one thematic + one status**.

**Thematic label per audit dimension:**

| Audit module | Thematic label |
|---|---|
| 01 — System Architecture | (no tickets typically) |
| 02 — Security | `quality` (or project-specific `security` if it exists) |
| 03 — Privacy | `quality` (and `privacy` if it exists) |
| 04 — Accessibility | `quality` (and `accessibility` / `a11y` if it exists) |
| 05 — Architecture Integrity | `quality` |
| 06 — Testing | `quality` |
| 07 — Documentation | `quality` |
| 08 — Dependencies | `quality` or `infrastructure` |
| 09 — Performance & Scalability | `quality` |
| 10 — Speed | `quality` |
| 11 — Serverless cold-start | `infrastructure` or `quality` |
| 12 — DevOps | `infrastructure` |
| 13 — Cost | `infrastructure` |
| 14 — Engineering Practice | `quality` |
| 15 — UX | `feature` (if new behavior) or `quality` (if fix) |
| 16 — Product Gap | `feature` |
| 17 — Frontend Modernization | `quality` |
| 18 — AI / ML | `quality` or `feature` |
| 19 — Product-Type | varies by finding |
| 20 — i18n / l10n | `feature` (and `i18n` if it exists) |
| 21 — SEO | `quality` |
| 22 — Sector Compliance | `quality` (and the framework label if it exists — `hipaa`, `pci`, etc.) |
| 23 — Reuse & Consolidation | `quality` |
| 24 — Workarounds & Root-Cause Gaps | `quality` |

If the project has audit-specific labels (e.g., `security`, `accessibility`,
`a11y`), prefer those. Otherwise fall back to the standard set: `feature`,
`planning`, `quality`, `infrastructure`.

**Status label:** every audit-emitted ticket starts as `proposed`. They are not
ready until the team scopes and sequences them.

```bash
gh issue create \
  --title "Hash passwords with argon2id in lib/auth/hash.ts" \
  --label "quality,proposed" \
  --body "$(cat <<'EOF'
## Problem
...
EOF
)"
```

---

## Step 4: Sequence by dependency and severity

Before filing, sort the tickets:

1. **Dependencies first.** If ticket B says "depends on A," file A first so its
   `#N` is known. Update B's body to reference the real `#N`.
2. **Within independent tickets, file in severity order** (Critical → High →
   Medium → Low). This makes browsing the issue list useful — most-severe at
   bottom of "recently created" view.

Group commonly-related tickets so they read as a coherent set when listed.

---

## Step 5: Confirmation gate (always)

Before bulk-creating issues, **show the user the list and ask for confirmation.**
Filing N issues is a high-blast-radius action that creates external state. The
confirmation step is non-optional even in auto modes.

Output the list as:

```
About to file <N> tickets:

1. [Critical] Hash passwords with argon2id in lib/auth/hash.ts (security)
   Depends on: —
   AC: 3 criteria
   Source: docs/audits/security-audit.md §3.1

2. [Critical] Add CSP header to vercel.json (security)
   Depends on: —
   AC: 2 criteria
   Source: docs/audits/security-audit.md §3.7

... (full list)

Proceed? [y/N]
```

On confirmation, file each ticket via `gh issue create`. On each successful
filing, capture the issue number and update any dependent ticket bodies before
filing them.

If a ticket fails to file, stop and report — do not silently skip. The user can
fix and re-run.

---

## Step 6: Write the index file

After filing, write `/docs/audits/remediation-plan.md`:

```markdown
# Remediation Plan

Date: YYYY-MM-DD
Audit run commit: <git rev-parse HEAD>

## Summary

- <N> tickets filed across <K> audit dimensions
- Tracking system: <GitHub Issues | TICKETS.md | draft markdown>
- <N_critical> Critical, <N_high> High, <N_medium> Medium, <N_low> Low

## Tickets by audit dimension

### Security (Module 02)
- #101 [Critical] Hash passwords with argon2id in lib/auth/hash.ts
- #102 [Critical] Add CSP header to vercel.json
- #103 [High] ...

### Privacy (Module 03)
- ...

### Accessibility (Module 04)
- ...

### Architecture Integrity (Module 05)
- ...

### Testing (Module 06)
- ...

### Documentation (Module 07)
- ...

### Dependencies (Module 08)
- ...

### Reuse & Consolidation (Module 23)
- ...

### Workarounds & Root-Cause Gaps (Module 24)
- ...

### Performance & Scalability (Module 09)
- ...

### Speed (Module 10)
- ...

### Serverless / Cold-Start (Module 11, if applicable)
- ...

### DevOps (Module 12)
- #110 [High] Pin Node base image to 20.11.0-alpine in Dockerfile
- ...

### Cost (Module 13)
- ...

### Engineering Practice (Module 14)
- ...

### UX (Module 15)
- ...

### Product Gap (Module 16)
- ...

### Frontend Modernization (Module 17)
- ...

### AI / ML (Module 18, if applicable)
- ...

### Product-Type (Module 19, if applicable)
- ...

### i18n / l10n (Module 20, if applicable)
- ...

### SEO (Module 21, if applicable)
- ...

### Sector Compliance (Module 22, if applicable)
- ...

## Recommended Sequence

Choose the framing that matches the project's stage:

### Option A: Effort-tier framing (post-launch / continuous)

Phase 0 — Pre-launch blockers (Critical + legal exposure):
- #101 (auth)
- #102 (CSP)
- #1XX (a11y blocker if EU EAA in scope)
- ...

Phase 1 — First 30 days:
- (High-severity items, parallelizable)

Phase 2 — First 90 days:
- (Medium and lower)

### Option B: Launch-stage framing (pre-launch / pre-paid-launch)

For products preparing for major release milestones, four launch-stage
buckets read more naturally than effort tiers:

- **Fix immediately** — security invariant violations, Critical issues
  that block any further development cleanly.
- **Fix before beta** — High-severity issues that would damage trust
  with early users; major UX gaps; data-handling correctness.
- **Fix before paid launch** — issues that become legal / billing /
  procurement risks once money changes hands; compliance posture; SLA
  / DR readiness.
- **Monitor continuously** — Medium / Low issues; ongoing posture work;
  things that are fine now but deserve a watcher.

Use Option A for established products iterating; Option B for
pre-launch / pre-monetization projects. Either works; pick one and be
consistent in the index file.

## Notes
- Status: every ticket is `proposed`. Promote to `ready` after scoping; to
  `in-progress` when picked up; close on completion per the tickets-and-logbook
  skill.
- Dependencies: see individual issue bodies.
- Acceptance criteria are starting points — refine as needed during scoping.
```

---

## Special cases

### Project uses TICKETS.md flat file

Append to `docs/TICKETS.md` using the project's existing format. Recommend (in
the index file) running the `tickets-migration` skill to move to GitHub Issues.

### Project has no tracking system

Write `/docs/audits/remediation-tickets.md` containing every ticket body in
sequence, separated by `---` rules. Each body is in the standard format from
the tickets-and-logbook skill, ready to be `gh issue create`d when the team
sets up tracking.

In the index file, recommend setting up GitHub Issues and pasting the bodies.

### `--no-tickets` flag

Always produce the markdown draft. Do not file. The index file should still be
written, with a note that ticket filing was deferred and how to file later.

### `--dry-run` flag

Do not create files at all. Print the index file content to chat instead, with
a summary of how many tickets *would* have been filed.

---

## Severity calibration (inherited from source modules)

Severity comes from the source audit. If the source audit didn't assign severity
to a finding, use:

- **Critical:** legal/regulatory exposure, data loss, RCE, broken auth, complete
  feature outage.
- **High:** privilege escalation, persistent UX failures, scaling cliffs near
  current load.
- **Medium:** clear issues with no immediate exploit/break, performance
  degradation patterns.
- **Low:** polish, deviations from best practice, future-proofing.

---

## What this module is *not*

- Not a duplicate of Module 00.2 (Findings). Module 00.2 is a catalog; this is
  ticket creation.
- Not a project plan. We file tickets and recommend a sequence; we do not
  estimate days, assign owners, or set deadlines (those are team decisions).
- Not a code review. We file *what to do*, not *how exactly to write the fix*.
  Approach bullets are guidance, not prescription.

---

## Evidence requirements

Every ticket body cites the source audit document and the file/line evidence
from the original finding. A ticket that says "fix the auth issue" with no
pointer to the audit or code is a malformed ticket and must be rejected before
filing.
