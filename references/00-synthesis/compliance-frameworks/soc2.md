# SOC 2 Readiness Mapping

Map findings from finding-producing modules to **AICPA Trust Services
Criteria** and produce a Type I or Type II readiness report + CSV control
matrix.

Outputs:
- `/docs/audits/compliance/soc2-readiness.md`
- `/docs/audits/compliance/soc2-controls.csv`

---

## Live Discovery

Fetch https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2 —
confirm current Trust Services Criteria (TSC) and Points of Focus version.
The 2017 TSC with 2022 Points of Focus is the current standard at writing.

Search "SOC 2 common audit findings 2026" — surfaces what auditors
have been flagging in recent engagements; useful calibration.

---

## Step 1: Determine type and scope

**Type I** vs **Type II:**

| | Type I | Type II |
|---|---|---|
| What it attests | Controls *designed* appropriately | Controls *operated effectively over time* |
| Period | Point in time | 3–12 months (commonly 6) |
| Evidence requirement | Control exists | Control exists *and ran consistently* |
| Audit cost | Lower | Higher |
| Customer ask | Rare | Most common (enterprise procurement) |

**Trust Services Criteria categories** (5 total — most projects scope
Security + Availability + Confidentiality):

- **Security (CC1–CC9)** — always required
- **Availability (A1.1–A1.3)** — for products with uptime commitments
- **Processing Integrity (PI1.1–PI1.5)** — for products that process transactions
- **Confidentiality (C1.1–C1.2)** — for products handling confidential data
- **Privacy (P1.1–P8.1)** — only when needed (often skipped if GDPR-only)

Detect scope from the codebase:
- Stripe / payments / billing → Processing Integrity in scope
- Customer SLA / uptime page / contracts mentioning availability → Availability in scope
- Customer-confidential data handling → Confidentiality in scope
- B2B-only with no consumer privacy claims → Privacy probably out-of-scope (handled by separate Privacy module)

---

## Step 2: Map findings to TSC controls

For each TSC category, walk the controls and pull evidence from prior
modules.

### Common Criteria (Security)

| Control | Name | Evidence module(s) | What to look for |
|---|---|---|---|
| **CC1.1** | Demonstrates commitment to integrity and ethical values | OUT_OF_BAND | Code of conduct, ethics training |
| **CC1.2** | Board independence and oversight | OUT_OF_BAND | Board charter, advisor list |
| **CC1.3** | Management establishes structure | OUT_OF_BAND | Org chart, role definitions |
| **CC1.4** | Demonstrates commitment to competence | OUT_OF_BAND | Hiring criteria, training records |
| **CC1.5** | Holds individuals accountable | OUT_OF_BAND | Performance reviews, sanction policy |
| **CC2.1** | Quality of information | Modules 02, 12, 14 | Audit logs, monitoring quality |
| **CC2.2** | Internal communication | Module 14 | Incident comms, postmortems |
| **CC2.3** | External communication | OUT_OF_BAND | Customer notifications process |
| **CC3.1** | Specifies suitable objectives | OUT_OF_BAND | Risk assessment doc |
| **CC3.2** | Identifies risks | OUT_OF_BAND + Module 00.2 (Findings) | Annual risk assessment + this audit's register |
| **CC3.3** | Considers fraud potential | OUT_OF_BAND | Fraud-risk assessment |
| **CC3.4** | Identifies and analyzes change | Module 14 | Change management, PR review |
| **CC4.1** | Conducts ongoing monitoring | Module 12 | Observability, alerting |
| **CC4.2** | Evaluates and communicates deficiencies | Module 14 | Incident response, postmortems |
| **CC5.1** | Selects and develops control activities | Cross-module | Overall control set |
| **CC5.2** | Considers technology controls | Cross-module | Same |
| **CC5.3** | Deploys controls through policies | OUT_OF_BAND | Written policies |
| **CC6.1** | Logical and physical access (provisioning) | Module 02 | Access control implementation, RBAC |
| **CC6.2** | Removes access of inactive users | OUT_OF_BAND + Module 14 | Quarterly access review process; auto-deprovisioning |
| **CC6.3** | Authorizes changes to access | Module 02 + 14 | Approval workflow for permission grants |
| **CC6.4** | Restricts physical access | OUT_OF_BAND | Office / data center physical access |
| **CC6.5** | Disposes of physical assets | OUT_OF_BAND | Asset disposal policy |
| **CC6.6** | Implements logical access security measures | Module 02 | MFA, password policy, session management |
| **CC6.7** | Restricts the transmission of information | Module 02 + 12 | TLS, encryption in transit |
| **CC6.8** | Prevents or detects unauthorized software | Module 08 | Dependency scanning, supply-chain controls |
| **CC7.1** | Detects vulnerabilities (vulnerability mgmt) | Module 08 | Dep scanning + CVE patching cadence |
| **CC7.2** | Monitors system components and operations | Module 12 | Logging, monitoring, alerting |
| **CC7.3** | Evaluates security events | Module 14 | Incident response process |
| **CC7.4** | Responds to identified security events | Module 14 | IR runbooks, postmortems |
| **CC7.5** | Recovers from security events | Module 12 + 14 | DR plan, restore drills |
| **CC8.1** | Authorizes and manages changes (change mgmt) | Module 14 | PR review, branch protection, deployment audit |
| **CC9.1** | Identifies, selects, develops risk mitigation activities | Module 00.2 + 00.3 | Risk register, remediation plan |
| **CC9.2** | Vendor and business partner risk management | OUT_OF_BAND + Module 03 | Sub-processor list, DPAs |

### Availability (if in scope)

| Control | Name | Evidence module(s) | What to look for |
|---|---|---|---|
| **A1.1** | Maintains, monitors, and evaluates current processing capacity | Module 09 + 12 | Capacity planning, scalability analysis |
| **A1.2** | Authorizes, designs, develops, deploys backup, recovery procedures | Module 12 | Backup automation + restore drill |
| **A1.3** | Tests recovery procedures | Module 12 Step 8 | Restore drill performed and documented |

### Processing Integrity (if in scope)

| Control | Name | Evidence module(s) | What to look for |
|---|---|---|---|
| **PI1.1** | Obtains, generates, uses, communicates relevant information | Cross-module | Data flow documentation |
| **PI1.2** | Implements policies to maintain processing integrity | Module 06 | Test coverage on critical paths (auth, payments) |
| **PI1.3** | System processing is complete, valid, accurate, timely, authorized | Module 06 + 09 | Idempotency, transaction integrity, contract tests |
| **PI1.4** | Output is complete and accurate | Module 06 | Output validation tests |
| **PI1.5** | Stores inputs and outputs completely, accurately, timely | Module 12 + 09 | Data persistence patterns |

### Confidentiality (if in scope)

| Control | Name | Evidence module(s) | What to look for |
|---|---|---|---|
| **C1.1** | Identifies and maintains confidential information | Module 03 | PII / confidential-data inventory |
| **C1.2** | Disposes of confidential information | Module 03 | Retention enforcement, deletion |

### Privacy (if in scope — typically not for B2B SaaS that already runs Module 03)

Defer to Module 03 (Privacy & Data Protection); each P-control maps to a
GDPR-style obligation already covered there.

---

## Step 3: Type I vs Type II evaluation per control

For each control above, evaluate twice:

**Type I:** Is the control *designed* correctly?
- IMPLEMENTED: design exists in code / infrastructure
- PARTIAL: partial design
- GAP: design absent
- OUT_OF_BAND: design lives in policy, not code

**Type II:** Is there *evidence the control operated* over the audit
period?
- IMPLEMENTED: design + operation signal (e.g., 6 months of audit logs;
  3 quarterly access reviews; 2 restore drills)
- PARTIAL: design correct but operation evidence sparse
- GAP: no operation evidence

A control is "Type II-ready" only if BOTH design and operation are
attestable. Many startups are Type I-ready but discover gaps when going
for Type II.

Capture both columns in the CSV:
```csv
control_id,name,type1_status,type2_status,...
CC6.1,Logical Access,IMPLEMENTED,IMPLEMENTED,...
CC6.2,Removing inactive user access,GAP,GAP,...
CC7.5,Recovers from security events,IMPLEMENTED,PARTIAL,...
```

---

## Step 4: Common SOC 2 audit findings to check for

These are issues auditors flag often. Walk each before finalizing:

- **Access reviews rubber-stamped.** Quarterly access review is performed
  but the evidence is "approved by manager" with no actual review. Look
  for review logs / spreadsheets in the repo or referenced.
- **Audit log gaps.** "We log everything" but the log retention is 7 days
  or the destination is a personal Slack channel.
- **Vendor inventory drift.** Sub-processor list out of date relative to
  what the codebase actually integrates with (cross-ref Module 03).
- **Onboarding/offboarding inconsistency.** Manual process; evidence is
  ad-hoc emails. OUT_OF_BAND but commonly weak.
- **Risk assessment annual but performative.** Risk-register doc that's
  "annual" but never changes year-over-year.
- **Incident postmortem culture absent.** Incidents happen but no
  postmortem is written, or postmortems are written but action items
  never close.
- **Backup never restored.** Cross-ref Module 12 Step 8.
- **DR plan exists but never tested.** Same.
- **Change management = "we use GitHub."** Without branch protection +
  required review + audit log of protected-branch changes.
- **MFA "available" not "enforced."** Optional MFA is not the same as
  required MFA.
- **Vulnerability management cadence.** "We patch when we notice." SLA?
  Tracking? Cross-ref Module 08.

Each detected weakness from this list should map to a specific control
and surface in the CSV.

---

## Step 5: Compliance automation tool integration

If Vanta / Drata / Secureframe / Sprinto / Hyperproof / Thoropass detected
in the codebase or as a GitHub App:

- Note the tool and likely evidence destination
- Note that this audit does NOT pull from the tool's API (per scope)
- Recommend the team verify the corresponding controls show "passing" in
  the tool's dashboard
- For controls the tool typically auto-collects (cloud config, GitHub
  branch protection, employee state from HRIS), expect the tool's evidence
  rather than direct codebase findings

---

## Output: markdown report

```markdown
# SOC 2 Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: SOC 2 TSC 2017 with 2022 Points of Focus (fetched <date>)
Compliance automation: <Vanta / Drata / etc. / DIY>
Type targeted: Type I / Type II

## Scope assessed

- Common Criteria (Security): yes (always)
- Availability: yes / no — rationale
- Processing Integrity: yes / no — rationale
- Confidentiality: yes / no — rationale
- Privacy: yes / no — rationale (typically defer to Module 03)

## Posture

**Type I readiness:** Ready / Mostly ready (N gaps) / Not yet ready (N+ gaps)
**Type II readiness:** Ready / Operation evidence sparse / Not yet attemptable

**Critical gaps (block audit):** <count>
**Out-of-band gaps (must be gathered separately):** <count>

## Top gaps to close before audit
(prioritized list)

## Out-of-band checklist
(controls flagged OUT_OF_BAND — what to gather from outside the codebase
and bring to the auditor)

## Compliance automation status
(detected tool + recommendations)

## Cross-references to other audit findings
(this list of controls overlaps with findings in security-audit.md,
devops-audit.md, etc. — link to specific findings)

## Confidence
H / M / L — based on how much of the framework's controls were auditable
from code (typically 50-60% of TSC controls have a codebase signal; the
rest are policy/process)
```

## Output: CSV control matrix

```csv
framework,control_id,control_name,category,type1_status,type2_status,evidence_module,evidence_file,evidence_summary,gap_notes,out_of_band,recommendation
SOC2,CC1.1,Demonstrates commitment to integrity and ethical values,Control Environment,OUT_OF_BAND,OUT_OF_BAND,—,—,—,—,Yes,Document and disseminate code of conduct
SOC2,CC1.2,Board independence and oversight,Control Environment,OUT_OF_BAND,OUT_OF_BAND,—,—,—,—,Yes,Establish board charter (or note for early-stage exemption)
SOC2,CC6.1,Logical and physical access (provisioning),Common Criteria - Access Control,IMPLEMENTED,IMPLEMENTED,02-security,security-audit.md,WorkOS auth + middleware enforced,—,No,—
SOC2,CC6.2,Removes access of inactive users,Common Criteria - Access Control,GAP,GAP,—,—,—,No automated deprovisioning detected; no quarterly access review process found,No,Implement quarterly access review process or auto-deprovisioning via WorkOS+Vanta
SOC2,CC7.5,Recovers from security events,Common Criteria - System Operations,IMPLEMENTED,PARTIAL,12-devops,devops-audit.md,DR plan exists; restore not drilled in last 6 months,Type II requires evidence of restore drill within audit period,No,Conduct restore drill within next 30 days; document outcome
...
```

The CSV header schema is consistent across all framework sub-references —
auditors and compliance teams can join across frameworks if they pursue
multiple.

---

## Severity calibration (gap-level)

- **Critical:** in-scope control has Type I GAP and the team is targeting
  Type I attestation in <90 days
- **High:** in-scope control has Type II GAP and team targets Type II in
  <12 months
- **Medium:** OUT_OF_BAND control with no evidence team is gathering it
  (will surface in auditor interview as a finding)
- **Low:** PARTIAL controls with clear strengthening path, or
  documentation polish

---

## Evidence requirements

Every IMPLEMENTED claim cites the source audit doc and finding. Every GAP
cites the absence-evidence (what was searched for and not found). Every
OUT_OF_BAND row notes the type of evidence that lives outside the
codebase, so the team knows what to bring to the auditor.
