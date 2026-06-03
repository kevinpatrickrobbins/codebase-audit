# 22 — Sector Compliance

Detect the project's regulatory sector and audit against the applicable
framework: HIPAA (US health), PCI DSS (payment), SOC 2, ISO 27001,
FedRAMP (US gov), GLBA (US financial), COPPA (children), FERPA (US
education).

Output: `/docs/audits/sector-compliance-audit.md`

This module is **conditional** — only run when sector signals are present.

---

## When to apply

| Signal | Sector |
|---|---|
| PHI fields, BAA mentions, HL7/FHIR, ICD/CPT codes | **HIPAA** |
| Card data, Stripe/Adyen/Braintree integration with `card_number` storage, PAN | **PCI DSS** |
| B2B enterprise selling, security questionnaire mentions, SOC 2 marketing | **SOC 2 / ISO 27001** |
| FedRAMP marketplace mention, .gov customers | **FedRAMP** |
| Banking, lending, insurance integration | **GLBA** |
| Children-facing under 13 | **COPPA** |
| Student-data, LMS integration, K-12 / higher-ed customers | **FERPA** |

If no sector signal is present and the project is general-purpose, skip
this module. Note in the audit run summary that sector compliance was
not run and why.

---

## Cross-module relationship

- **Privacy** (Module 03) covers GDPR / CCPA / general privacy law — not
  sector-specific frameworks.
- **Security** (Module 02) covers technical controls — sector compliance
  often requires *processes* and *documentation* alongside technical
  controls.
- **Compliance Readiness Mapping** (Module 00.4) is the *companion* to
  this module. **This module produces audit-time findings** (severity-
  tiered, ticket-ready) from direct codebase scan. **Module 00.4 produces
  auditor-ready control mappings** (per-framework readiness reports + CSV
  control matrices) and additionally covers 6 frameworks not in scope here:
  Quebec Law 25, CASL, NIS2, DORA, EU AI Act, Canadian provincial health.
  Run this module for "what needs fixing"; run 00.4 for "are we ready
  for the auditor". Findings here should cross-reference the relevant
  Module 00.4 sub-reference (e.g. `compliance-frameworks/hipaa.md`) when
  the control language matters for the team's evidence packet.

---

## Step 0: Live Discovery

Sector frameworks update less often than tech (HIPAA / PCI on multi-year
cycles) but still — fetch authoritative current state.

- **HIPAA:** Fetch https://www.hhs.gov/hipaa/index.html — current
  rule state. Search "HIPAA enforcement actions 2026" for recent
  precedent.
- **PCI DSS:** Fetch https://www.pcisecuritystandards.org/ — current
  PCI DSS version. (4.0 fully effective since March 2025; check for
  updates.)
- **SOC 2:** Fetch
  https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2 —
  current Trust Services Criteria.
- **ISO 27001:** Fetch https://www.iso.org/standard/82875.html —
  current standard version (27001:2022 most recent).
- **FedRAMP:** Fetch https://www.fedramp.gov/ — current baselines
  (Low / Moderate / High) and authorization process.
- **COPPA:** Fetch
  https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa
  for current FTC rule (potentially in flux given recent rulemaking).
- **FERPA:** Fetch
  https://www2.ed.gov/policy/gen/guid/fpco/ferpa/index.html

---

## Step 1: HIPAA (US health)

If health-related signals detected:

- **PHI inventory.** Patient names, MRNs, DOBs, diagnosis, procedures,
  lab results, images, claim/billing data — locate every column / form /
  log that touches PHI.
- **Encryption at rest + in transit** for all PHI.
- **Access controls** with **audit log** of every PHI access (who, when,
  what record).
- **BAA (Business Associate Agreement)** with every sub-processor that
  handles PHI — Stripe, AWS, Datadog, Anthropic / OpenAI, etc. all have
  HIPAA-eligible tiers with BAAs. Free tiers usually do not.
- **Minimum-necessary access** enforced (role-based, just-in-time access
  for support).
- **Data retention** per HIPAA (six-year minimum for some records).
- **Breach notification** process documented (60-day rule for HHS
  notification of breaches affecting > 500 individuals).
- **Workforce sanction policy** documented.
- **Annual security risk assessment** done.
- **Backup + disaster recovery + emergency mode** plans.
- **Workstation security** (out of code-audit scope; flag).

## Step 2: PCI DSS (payment cardholder data)

If card data signals detected:

- **Scope minimization.** Use **Stripe Elements / Adyen iframes** /
  similar to keep card data out of your servers entirely (SAQ-A scope,
  smallest burden) vs receiving and forwarding (SAQ-D, much larger).
- **No PAN storage** unless absolutely required. If stored: encrypted +
  tokenized (preferably tokenized via the payment processor).
- **CVV never stored** post-authorization (any storage = automatic fail).
- **TLS 1.2+** for all card transmission.
- **Network segmentation** of cardholder data environment from rest of
  infrastructure.
- **PCI DSS version compliance** (4.0+ fully required since March 2025).
- **Quarterly external scan** by an Approved Scanning Vendor (ASV).
- **Annual SAQ** (Self-Assessment Questionnaire) filed at the right level
  (A / A-EP / B / B-IP / C / D).

## Step 3: SOC 2 (Trust Services Criteria)

For a platform pursuing SOC 2 Type II (the typical B2B target):

The five Trust Services Criteria — Security (always), Availability,
Processing Integrity, Confidentiality, Privacy. Most B2B targets
Security + Availability + Confidentiality.

- **Access control.** Documented user provisioning, just-in-time access,
  quarterly reviews, deprovisioning on offboarding.
- **Change management.** PR review + approval + deployment audit log.
  Cross-ref Module 14 (Engineering Practice).
- **Risk assessment** annual + on major change.
- **Vendor management.** Sub-processor inventory + due diligence + DPA.
  Cross-ref Module 03 (Privacy).
- **Audit logs.** Comprehensive, centralized, tamper-evident retention.
- **Incident response.** Documented playbook + tested + postmortems.
  Cross-ref Module 14.
- **Backups + DR drill.** Cross-ref Module 12 (DevOps Step 8).
- **Encryption at rest + in transit.**
- **Vulnerability management.** Regular scans, patching cadence, SLA.
  Cross-ref Module 08 (Dependencies).
- **Security awareness training** for staff.
- **Compliance automation tools** (Vanta, Drata, Secureframe, Sprinto)?
  Codebase often has integration scripts.

A SOC 2 audit reviews evidence over a 6–12 month observation window;
this module flags whether the *evidence-generating mechanisms* are in
place in the codebase / repo.

## Step 4: ISO 27001

Information Security Management System (ISMS) certification.

- **ISMS scope statement** documented.
- **Statement of Applicability** mapping ISO 27001:2022 Annex A controls
  (93 controls in 2022 version vs 114 in 2013).
- **Risk treatment plan.**
- **Management review** cadence documented.
- **Internal audit** scheduled.
- **Tooling overlap with SOC 2** — Vanta, Drata, etc. support both.

## Step 5: FedRAMP (US federal gov procurement)

If selling to US federal gov:

- **Authorized boundary** documented.
- **Baseline:** Low / Moderate / High.
- **NIST SP 800-53 controls** implementation.
- **3PAO assessment** completed or in progress.
- **Marketplace listing** status.
- **FIPS 140-2/3 cryptography** required.
- **US-only data residency** typically required.
- **US persons clearance** for support staff (often required at higher
  baselines).

## Step 6: GLBA (US financial)

If financial services:

- **Safeguards Rule** (16 CFR Part 314) controls.
- **Privacy notices** to consumers (initial + annual).
- **Information security program** with designated qualified individual.
- **Multi-factor authentication** for access to consumer info (2023
  Safeguards Rule update).
- **Breach notification** to FTC for incidents affecting ≥ 500 consumers.

## Step 7: COPPA (US children under 13)

If serving kids:

- **Verifiable parental consent** before collecting any personal info.
- **Age gating** to detect children's signups (and not gathering DOB
  before parental consent).
- **Data minimization** — collect only what's needed for the activity.
- **No targeted advertising** to children.
- **Parental access** to review / delete child's info.

## Step 8: FERPA (US education)

If serving K-12 or higher-ed handling student records:

- **Educational records** are protected under FERPA.
- **School Official exception** for SaaS handling student data —
  agreement with the school treats vendor as a school official subject
  to FERPA.
- **Directory information vs PII** distinction.
- **Parental / student access** to records.

---

## Cross-sector red flags

- Sector-applicable framework named in marketing but no evidence of
  implementation in the codebase (no audit log, no encryption-at-rest
  config, no BAA mentions in vendor docs).
- "We're SOC 2 compliant" without an actual report.
- Children-facing product with no age gate.
- Cardholder data in a regular DB without tokenization.
- Health data in non-BAA-covered services (free Datadog tier, etc.).
- FedRAMP-marketed without authorization status.
- Sector signal present but the team is unaware (the signal may be
  evidence the product has accidentally entered a regulated context).

---

## Output structure

```markdown
# Sector Compliance Audit

Date: YYYY-MM-DD

## Live Discovery (fetched YYYY-MM-DD)
(table)

## Detected sector(s)
| Sector | Confidence | Evidence (file:line) |
|---|---|---|

## Per-sector findings

### HIPAA (if applicable)
(checklist with evidence)

### PCI DSS (if applicable)
(checklist with evidence)

### SOC 2 (if applicable)
(checklist + state of compliance program — automation tools, prior audit)

### ISO 27001 (if applicable)

### FedRAMP (if applicable)

### COPPA / FERPA / GLBA (if applicable)

## Gap analysis
What's required by detected sector(s) vs what's in place.

## Findings

## Final recommendation
Single highest-priority compliance action.

## Decision summary
- Sector(s) detected: <list>
- Material gaps: <list>
- Compliance posture: <state>

## Confidence: H / M / L
- Audit-only evidence; full compliance assessment requires policies,
  procedures, and external audit beyond code review.

## Out of Scope / Inconclusive
- Out-of-repo policies / procedures / training / facilities
- External audit reports (cited if linked from codebase, otherwise out
  of scope)
- Workforce-level controls
```

---

## Severity calibration

- **Critical:** sector applies and a regulated data type is mishandled
  (PHI in non-BAA service, PAN in plain DB, children's PII without
  parental consent).
- **High:** sector applies but compliance program is minimal (no audit
  log, no DR plan, no incident response).
- **Medium:** sector compliance partially in place; specific controls
  missing.
- **Low:** documentation polish, naming consistency.

Sector compliance is one area where "Low" severity is rare — the
framework either applies or doesn't, and gaps tend to be material.

Every finding cites the file/line evidence and the relevant section of
the framework (e.g., "HIPAA Security Rule §164.312(a)(1) Access
Control").
