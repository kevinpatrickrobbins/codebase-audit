# 00.4 — Compliance Readiness Mapping (Dispatcher)

Map findings from prior audit modules to **specific compliance-framework
controls** and emit auditor-ready output (markdown report + CSV control
matrix per framework). This is a **synthesis view**, not a finding-producing
module — it reads from work already done and projects it onto the framework's
control structure.

Outputs (per applicable framework):
- `/docs/audits/compliance/<framework>-readiness.md` (markdown report)
- `/docs/audits/compliance/<framework>-controls.csv` (auditor-friendly matrix)

---

## When to apply

Conditional — runs only frameworks the project is in scope for. Frameworks
in scope come from Module 22 (Sector Compliance) detection signals.

The user can also force a framework via `/audit compliance soc2` /
`/audit readiness hipaa` / etc. — useful when a team is preparing for a
specific audit even if the codebase doesn't have obvious signals.

---

## Cross-module relationship

This module is the *companion* to Module 22 (Sector Compliance), not a
duplicate:

| | Module 22 — Sector Compliance | Module 00.4 — Compliance Readiness (this) |
|---|---|---|
| **Lens** | Audit-time finding pass | Auditor-ready control mapping |
| **Driver** | Direct codebase scan | Framework control structure |
| **Output** | Severity-tiered findings | `<framework>-readiness.md` + `<framework>-controls.csv` |
| **Frameworks** | HIPAA, PCI, SOC 2, ISO 27001, FedRAMP, GLBA, COPPA, FERPA (8) | Same 8 + Quebec Law 25, CASL, NIS2, DORA, EU AI Act, Canadian provincial health (14) |
| **Question answered** | "What needs fixing?" | "Are we ready for the auditor?" |

Run Module 22 first; this module reads its detected sectors from
`sector-compliance-audit.md` (see Step 1 below). For the 8 overlapping
frameworks, Module 22 surfaces gaps as tickets and this module maps the
current state onto the auditor's control rubric — together they produce
both the fix list and the evidence packet.

---

## Framework sub-references

| Framework | Sub-reference | Common scope |
|---|---|---|
| **SOC 2 Type I / Type II** | `compliance-frameworks/soc2.md` | B2B SaaS selling to mid-market or enterprise |
| **HIPAA** | `compliance-frameworks/hipaa.md` | US health products handling PHI |
| **PCI DSS 4.0** | `compliance-frameworks/pci-dss.md` | Anyone touching cardholder data |
| **ISO/IEC 27001:2022** | `compliance-frameworks/iso27001.md` | EU enterprise sales; often parallel to SOC 2 |
| **FedRAMP** (Low / Moderate / High) | `compliance-frameworks/fedramp.md` | US federal procurement |
| **COPPA** | `compliance-frameworks/coppa.md` | US products serving children under 13 |
| **FERPA** | `compliance-frameworks/ferpa.md` | US K-12 / higher-ed student data |
| **GLBA Safeguards Rule** | `compliance-frameworks/glba.md` | US financial services |
| **Quebec Law 25** | `compliance-frameworks/quebec-law-25.md` | Anyone with Quebec residents in user base (often missed) |
| **CASL** | `compliance-frameworks/casl.md` | Anyone sending commercial email/SMS/push to Canadian recipients |
| **NIS2 Directive** | `compliance-frameworks/nis2.md` | EU "essential" / "important" entities — many SaaS in scope |
| **DORA** | `compliance-frameworks/dora.md` | EU financial entities + critical ICT third parties |
| **EU AI Act** | `compliance-frameworks/eu-ai-act.md` | EU-facing AI products (per risk-tier classification) |
| **Canadian Provincial Health** (PHIPA / HIA / PHIA / etc.) | `compliance-frameworks/canadian-provincial-health.md` | Canadian provincial health data |

When a new framework becomes relevant (state privacy laws, sector regs),
add a sub-reference following the same pattern.

---

## Step 1: Determine in-scope frameworks

From Module 22 (Sector Compliance):

```
detected_sectors = read /docs/audits/sector-compliance-audit.md → "Detected sector(s)"
```

If Module 22 hasn't run, do quick detection here per the table in
`references/02-compliance/sector-compliance.md`.

Always offer **SOC 2** as a candidate for B2B SaaS — it's almost always
applicable even if not yet pursued.

---

## Step 2: Live Discovery

Compliance frameworks change less frequently than tech, but they do
change. Fetch current state for each framework before mapping:

- **SOC 2 / TSC:** Fetch https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2 — current Trust Services Criteria.
- **HIPAA:** Fetch https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html — Security Rule current state.
- **PCI DSS:** Fetch https://www.pcisecuritystandards.org/document_library/ — current version (4.0+).
- **ISO 27001:** Fetch https://www.iso.org/standard/27001 — version (currently 27001:2022 with 93 Annex A controls vs older 114).
- **FedRAMP:** Fetch https://www.fedramp.gov/baselines/ — current baselines (NIST SP 800-53 Rev 5).
- **COPPA:** Fetch https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa — current rule (potentially in flux).
- **FERPA:** Fetch https://www2.ed.gov/policy/gen/guid/fpco/ferpa/index.html
- **GLBA:** Fetch https://www.ftc.gov/business-guidance/privacy-security/gramm-leach-bliley-act — Safeguards Rule current state.

Note the versions you fetched at the top of each report.

---

## Step 3: Detect compliance-automation tools

Many small-mid SaaS projects use compliance automation platforms to
collect evidence. Detect whether one is installed:

| Tool | Detection signal |
|---|---|
| **Vanta** | `@vanta/sdk` in package.json; `.vanta/`; `vanta` GitHub App on the repo (check `gh api repos/{owner}/{repo}/installations`) |
| **Drata** | `@drata/*` packages; Drata GitHub App; `drata-config.*` |
| **Secureframe** | `@secureframe/*`; Secureframe GitHub App |
| **Sprinto** | `@sprinto/*`; `.sprinto/` |
| **Hyperproof** | `@hyperproof/*`; Hyperproof references |
| **Thoropass** | `@thoropass/*`; references |

**Detect-only.** Do not call any of these tools' APIs (the user explicitly
opted out). Note the integration in the report so the team / auditor knows
where to look for evidence.

If a tool is detected, surface in each framework report:

```markdown
## Compliance automation
Detected: Vanta (via GitHub App + @vanta/sdk in lib/compliance/vanta.ts)
Likely evidence destination: Vanta dashboard for this org.
This audit does NOT pull current control state from Vanta — verify there
that controls map to passing.
```

If no tool detected, note: "DIY compliance posture — evidence will need
to be collected manually for the auditor."

---

## Step 4: For each in-scope framework, run the sub-reference

Each sub-reference produces:

1. **Control mapping table** — every framework control with status, evidence, and gap notes.
2. **Markdown report** — narrative summary, posture, top gaps, recommendations.
3. **CSV control matrix** — auditor-friendly, importable to Excel / Google Sheets.

Status taxonomy (consistent across frameworks):

| Status | Meaning |
|---|---|
| **IMPLEMENTED** | Control is present + evidence exists in repo / linked tools |
| **PARTIAL** | Control is partially in place; specific gaps identified |
| **GAP** | Control should be present but isn't |
| **OUT_OF_BAND** | Control exists outside auditable code (HR policy, training, physical security, etc.) — flagged for the team to gather |
| **N/A** | Control not applicable to this project's scope |

---

## Step 5: Emit outputs

For each in-scope framework, write two files:

### Markdown report

```markdown
# <Framework> Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: <e.g. SOC 2 TSC 2017 with 2022 Points of Focus> (fetched <date>)
Compliance automation: <Vanta / Drata / etc. / DIY>

## Posture

**Readiness level:** Type I-ready / Type II-ready / Not yet ready
**Critical gaps:** <count>
**Out-of-band gaps:** <count> (must be gathered separately)

## Control coverage

| Control ID | Name | Status | Evidence | Gap notes |
|---|---|---|---|---|

(short form per framework — full table in CSV)

## Top gaps to close

(prioritized list of GAP-status controls)

## Out-of-band checklist

(controls flagged OUT_OF_BAND — what to collect from outside the codebase)

## Compliance automation status

(detected tool + what audit-time signals tell us about its operation)

## Recommendations

(prioritized actions before the auditor walks in)

## Confidence

High / Medium / Low — based on how much of the framework's controls are
auditable from code vs out-of-band.
```

### CSV matrix

A flat file with one row per control. Schema:

```csv
framework,control_id,control_name,category,status,evidence_module,evidence_file,evidence_summary,gap_notes,out_of_band,recommendation
SOC2,CC6.1,Logical Access,Common Criteria - Access Control,IMPLEMENTED,02-security,security-audit.md,WorkOS auth + middleware enforced,—,No,—
SOC2,CC6.2,Removing access of inactive users,Common Criteria - Access Control,GAP,—,—,—,No automated deprovisioning detected,No,Implement quarterly access review or auto-deprovisioning
SOC2,CC1.1,Demonstrates commitment to integrity,Control Environment,OUT_OF_BAND,—,—,—,Code of conduct + ethics training is policy/process,Yes,Document and distribute
```

The CSV is the auditor-friendly artifact — many auditors prefer a
spreadsheet they can annotate during fieldwork.

---

## SOC 2 sidebar: Type I vs Type II distinction

(Applies to SOC 2 only; other frameworks have analogous distinctions handled
in their own sub-references.)

Type I evaluates **control design** at a point in time. Type II evaluates
**control operation** over a period (typically 6–12 months).

Type II requires **evidence over time**, not just "the control exists":

- Access reviews performed quarterly (and not rubber-stamped)
- Vulnerability scans run + tickets closed
- Incident postmortems written + action items tracked
- Backup restore drills performed + outcomes documented
- Change management evidence (PR reviews, branch protection enforcement)

The SOC 2 sub-reference distinguishes these explicitly. For Type II:
- "IMPLEMENTED" requires both *design* (the control exists) AND *operation
  signal* (evidence the control runs).
- A control with "design correct but no operation evidence" is PARTIAL for
  Type II readiness even though it might be IMPLEMENTED for Type I.

For the other frameworks (HIPAA, PCI, ISO 27001, FedRAMP), the operation
question is similar but framing differs — each sub-reference handles its
own conventions.

---

## Step 6: Output structure (top-level summary)

After all in-scope frameworks run, emit a top-level summary at
`/docs/audits/compliance/README.md`:

```markdown
# Compliance Readiness Summary

Date: YYYY-MM-DD

## Frameworks assessed

| Framework | Type | Posture | Critical gaps | Out-of-band | Report |
|---|---|---|---|---|---|
| SOC 2 | Type II | Not yet ready | 4 | 7 | [link](./soc2-readiness.md) / [CSV](./soc2-controls.csv) |
| HIPAA | n/a | Partially ready | 2 | 9 | [link](./hipaa-readiness.md) / [CSV](./hipaa-controls.csv) |
| ISO 27001 | n/a | Type II-equivalent ready | 3 | 5 | [link](./iso27001-readiness.md) / [CSV](./iso27001-controls.csv) |

## Cross-framework observations

- Controls that are gaps in *multiple* frameworks (high leverage to fix)
- Compliance automation status

## Top recommendations
```

This top-level summary is the entry point for stakeholders; the per-
framework reports are the depth.

---

## Severity calibration

This module's "findings" are gap-level — control gaps, not security
vulnerabilities. Severity translates to:

- **Critical:** control is required for in-scope framework + is GAP +
  blocks the audit (e.g. no audit log on a SOC 2 audit).
- **High:** control is in scope + PARTIAL.
- **Medium:** OUT_OF_BAND control with no evidence the team is collecting
  it (likely will surface during auditor interview).
- **Low:** documentation polish, control naming consistency.

---

## What this module is NOT

- **Not a substitute for an actual auditor.** This produces a *readiness
  view*. The actual SOC 2 / HIPAA / etc. audit is performed by an
  independent firm against evidence, not just code.
- **Not exhaustive.** Some controls are in-scope for the framework but
  out-of-scope for this module (HR, training, physical security). Those
  appear as OUT_OF_BAND with a reminder.
- **Not legal advice.** Severity and applicability decisions still require
  human judgment + legal review for sector-specific contexts.
