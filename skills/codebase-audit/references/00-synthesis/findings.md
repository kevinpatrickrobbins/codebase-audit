# 00.2 — Findings

Create or update the project findings.

Location:

`/docs/audits/findings.md`

---

## Step 0: Compare with previous audit run

If a prior findings exists at `/docs/audits/findings.md`, compute
a **delta** before writing the new section. This converts the audit from
one-shot to a continuous-improvement signal — stakeholders care more about
"what changed since last audit" than "raw current state."

**Process:**

1. Read prior findings (most recent dated section).
2. Match prior risks to current findings using:
   - Title similarity
   - Source audit module + section
   - File:line if cited in both
3. For each matched risk, classify:
   - **Closed** — risk no longer present in current findings
   - **Mitigated** — risk reduced (severity dropped or scope narrowed)
   - **Persistent** — same risk, same severity, still open
   - **Worsened** — same risk, severity increased
4. For unmatched current findings: **New** since last audit
5. For unmatched prior risks not appearing now: **Closed** (or Re-verify if
   the relevant module didn't run this time)

**Output a "Delta from previous audit" section before the new full
register:**

```markdown
## Delta from previous audit (YYYY-MM-DD → YYYY-MM-DD)

### Closed since last audit (count)
- Risk: <title> — was <severity>; closed by <evidence: ticket #N or
  observation>

### Mitigated (count)
- Risk: <title> — severity <prior> → <current>; reason

### Persistent (count)
- Risk: <title> — open since <first audit date>; ticket #N

### Worsened (count)
- Risk: <title> — severity <prior> → <current>; reason

### New since last audit (count)
- Risk: <title> — first appeared this run; severity <current>

### Net change
- Critical: <prior count> → <current count>
- High: <prior count> → <current count>
- Medium: <prior count> → <current count>
- Low: <prior count> → <current count>
- Trend: improving / steady / regressing
```

Persistent-Critical risks across multiple audit runs are themselves a
findings entry: "Risk-management discipline failing — Critical
items not closing across audit cycles."

If no prior audit exists, omit the Delta section and note "first audit
run — establishing baseline."

---

## Output a Non-Negotiable Invariants section

At the top of the findings, surface the project's **non-negotiable
security invariants** — rules the application must never violate. These are
posture commitments, not findings; they frame what the team agrees is
inviolable so subsequent findings can be measured against them.

**Default invariants** (use as baseline; teams may add domain-specific):

```markdown
## Non-Negotiable Invariants

- Authentication is not authorization.
- Never trust tenant / org / user IDs from the client.
- Every tenant-scoped query must enforce tenant isolation server-side.
- No production secret may appear in source code, logs, browser bundles,
  or AI prompts.
- No admin route may rely only on hidden UI for protection.
- No destructive operation may be possible without authorization,
  logging, and recovery.
- Backups must be restorable without using the compromised application.
- Production and development environments must not share credentials.
- AI agents must not weaken security to satisfy tests or speed
  implementation.
- Bus factor of one is unacceptable for production-critical knowledge.
- Every audit-eligible action (admin / billing / privilege / data
  destruction) must be logged with actor, timestamp, target, and outcome.
```

If the audit detects a project context that warrants more (financial
processing → "no card data persisted in app DB", health → "PHI never
crosses BAA boundary"), add domain-specific invariants based on findings
from Module 22 (Sector Compliance).

Findings that *violate* an invariant are automatically Critical regardless
of likelihood; the team committed not to allow them.

## Output an Overall Risk Level rollup

After the per-severity tables, surface a single **Overall Risk Level**
sentence with reasoning:

```markdown
## Overall Risk Level

**LOW / MODERATE / HIGH / CRITICAL**

Reasoning: <one paragraph — what's working, what's not, what's the
biggest exposure, what's the recommended next action>.
```

Calibration:

- **CRITICAL** — at least one Critical finding unmitigated, or an
  invariant violated in production code.
- **HIGH** — multiple High findings, or one Critical with active
  mitigation in progress.
- **MODERATE** — pattern of Medium findings, no active Critical, posture
  generally sound.
- **LOW** — only Low findings; foundations are solid; remaining items are
  hardening rather than fixes.

This rollup gives non-engineering stakeholders (founders, board,
investors, customers running security questionnaires) the one-line
summary they need.

## Purpose

The findings is a **static catalog of risks** synthesized from all prior
audit reports. It is a reference document — useful for board-level reviews,
launch readiness gates, security questionnaires, and onboarding new team members.

This is **not** a backlog or task list. The Remediation Plan (Module 00.3) handles
that — Module 00.3 emits real tickets in the project's tracking system. Module 00.2
(this one) catalogs.

| | Module 00.2 — Findings | Module 00.3 — Remediation Plan |
|---|---|---|
| **Type** | Static catalog | Action artifact |
| **Output** | Markdown table | Filed GitHub Issues / TICKETS.md entries |
| **Audience** | Stakeholders, auditors, board | Engineering team |
| **Purpose** | Visibility | Execution |
| **Updates** | Each audit run | When ticket status changes |

---

## How to build it

After all other audit modules have run, walk every finding and capture it as a
single risk row.

For each finding, decide:

- **Risk** — short statement of what could go wrong (impact framing).
- **Severity** — Critical / High / Medium / Low (from the source audit).
- **Likelihood** — Likely / Possible / Unlikely. Calibrate against this product's
  exposure: a public B2C app makes "Likely" exploitation more probable than an
  internal tool.
- **Impact** — concrete consequence: "user data leak", "regulatory fine",
  "outage", "team velocity loss".
- **Source** — which audit document and section.
- **Mitigation Strategy** — short summary; full plan lives in the corresponding
  Module 00.3 ticket.
- **Status** — Open / Tracked (i.e., a ticket exists in Module 00.3) / Mitigated /
  Accepted (the team has consciously chosen to accept the risk).

---

## Output structure

```markdown
# Findings

Date: YYYY-MM-DD
Audit run commit: <git rev-parse HEAD>

## Summary

- <N> Critical, <N> High, <N> Medium, <N> Low risks identified
- <N> tracked in remediation tickets (see remediation-plan.md)
- <N> accepted, <N> mitigated since last audit (if applicable)

## Risks by Severity

### Critical

| Risk | Likelihood | Impact | Source | Mitigation | Status |
|---|---|---|---|---|---|
| Plaintext password storage | Likely | Mass credential compromise → reputational + legal | security-audit.md §3.1 | Hash with argon2id; rotate existing | Tracked (#101) |
| ... | | | | | |

### High
(same table)

### Medium
(same table)

### Low
(same table)

## Risks by Domain
(Cross-cut view: same risks, regrouped.)

### Security
- (rows from above, security-only)

### DevOps / Infrastructure
- ...

### UX
- ...

### Performance
- ...

### Architecture
- ...

### Product Gaps
- ...

### Accessibility & Compliance
- ...

### Serverless / Cold-Start (if applicable)
- ...

## Accepted Risks
Risks the team has consciously chosen to accept. Each row includes the
rationale and a review date.

| Risk | Rationale | Reviewed | Next Review |
|---|---|---|---|

## Mitigation History (since previous audit)
| Risk | Status change | Date | Evidence |
|---|---|---|---|
```

---

## Severity & likelihood calibration

**Severity** is inherited from the source audit (see modules 02–21, plus 22 if the
project is in scope for sector compliance).

**Likelihood** considers exposure:

- **Likely:** exploitable by an unauthenticated attacker with public knowledge
  of the vulnerability class; reachable via normal user behavior; or already
  observed (e.g., in logs).
- **Possible:** requires authenticated user, specific conditions, or sustained
  effort.
- **Unlikely:** requires insider access, complex chained exploit, or unrealistic
  conditions.

For accessibility risks, "Likely" is appropriate when the product serves a
jurisdiction with active enforcement (EU after 28 June 2025; US ADA Title III
public accommodations).

---

## Evidence requirements

Every row cites a source audit document. A findings entry without a source
is unverifiable and must be rejected.

For "Tracked" status entries, include the issue number in the Status column —
this gives readers a one-click path from the catalog into the actionable plan.

---

## What this module is *not*

- Not a list of tasks. (Module 00.3.)
- Not a project plan. (No estimates, owners, or deadlines.)
- Not exhaustive. The register captures *risks worth flagging*, not every
  minor lint issue. Use judgment: a typo in error copy is a finding for module
  15 (UX) but not a findings entry.
