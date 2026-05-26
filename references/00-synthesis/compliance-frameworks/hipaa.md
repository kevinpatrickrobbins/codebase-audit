# HIPAA Readiness Mapping

Map findings to **HIPAA Security Rule** (45 CFR Part 164 Subpart C) safeguards
and produce a readiness report + CSV control matrix.

Outputs:
- `/docs/audits/compliance/hipaa-readiness.md`
- `/docs/audits/compliance/hipaa-controls.csv`

---

## Live Discovery

- Fetch https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html — Security Rule current state.
- Search "HIPAA enforcement actions <current year>" — recent OCR (Office for Civil Rights) precedent.
- Search "HIPAA Security Rule update <current year>" — pending rulemaking (HHS proposed updates in late 2024; check current state).

---

## Scope

HIPAA covers Covered Entities (CEs — providers, plans, clearinghouses)
and Business Associates (BAs — vendors handling PHI on a CE's behalf).
Most SaaS in scope are BAs.

Module 22 (Sector Compliance) detection signals: PHI fields, BAA
mentions, HL7/FHIR, ICD/CPT codes, health-domain language.

---

## Safeguards

The Security Rule has three categories. Each has Required (R) and
Addressable (A) implementation specifications. Addressable doesn't mean
optional — it means the entity may implement an equivalent if the spec
isn't reasonable; absence still requires documented justification.

### Administrative Safeguards (§164.308)

| Standard | Spec | R/A | Evidence module |
|---|---|---|---|
| **§164.308(a)(1)(i)** | Security Management Process | R | OUT_OF_BAND + Module 00.2 |
| §164.308(a)(1)(ii)(A) | Risk Analysis | R | OUT_OF_BAND |
| §164.308(a)(1)(ii)(B) | Risk Management | R | OUT_OF_BAND + Module 00.2/00.3 |
| §164.308(a)(1)(ii)(C) | Sanction Policy | R | OUT_OF_BAND |
| §164.308(a)(1)(ii)(D) | Information System Activity Review | R | Module 02 + 12 (audit logs) |
| **§164.308(a)(2)** | Assigned Security Responsibility | R | OUT_OF_BAND |
| **§164.308(a)(3)** | Workforce Security | R | OUT_OF_BAND |
| **§164.308(a)(4)** | Information Access Management | R | Module 02 (RBAC) |
| §164.308(a)(4)(ii)(A) | Isolating Health Care Clearinghouse Function | R | N/A typical |
| §164.308(a)(4)(ii)(B) | Access Authorization | A | Module 02 |
| §164.308(a)(4)(ii)(C) | Access Establishment and Modification | A | Module 02 + 14 |
| **§164.308(a)(5)** | Security Awareness and Training | R | OUT_OF_BAND |
| **§164.308(a)(6)** | Security Incident Procedures | R | Module 14 |
| **§164.308(a)(7)** | Contingency Plan | R | Module 12 |
| §164.308(a)(7)(ii)(A) | Data Backup Plan | R | Module 12 Step 8 |
| §164.308(a)(7)(ii)(B) | Disaster Recovery Plan | R | Module 12 |
| §164.308(a)(7)(ii)(C) | Emergency Mode Operation Plan | R | Module 12 + 14 |
| §164.308(a)(7)(ii)(D) | Testing and Revision Procedures | A | Module 12 (restore drill) |
| §164.308(a)(7)(ii)(E) | Applications and Data Criticality Analysis | A | Module 01 |
| **§164.308(a)(8)** | Evaluation | R | OUT_OF_BAND (annual security risk assessment) |
| **§164.308(b)** | Business Associate Contracts (BAA) | R | OUT_OF_BAND + Module 03 |

### Physical Safeguards (§164.310)

Mostly OUT_OF_BAND — facility access, workstation use, workstation
security, device and media controls. For a cloud-native SaaS, many of
these are inherited from the cloud provider's controls (and validated
via the cloud provider's own SOC 2 / HIPAA attestation). Document the
provider's compliance posture as inherited evidence.

### Technical Safeguards (§164.312)

Most code-auditable category.

| Standard | Spec | R/A | Evidence module |
|---|---|---|---|
| **§164.312(a)(1)** | Access Control | R | Module 02 |
| §164.312(a)(2)(i) | Unique User Identification | R | Module 02 (no shared accounts) |
| §164.312(a)(2)(ii) | Emergency Access Procedure | R | Module 14 (break-glass process) |
| §164.312(a)(2)(iii) | Automatic Logoff | A | Module 02 (session timeout) |
| §164.312(a)(2)(iv) | Encryption and Decryption | A | Module 02 + 12 (encryption at rest) |
| **§164.312(b)** | Audit Controls | R | Module 02 + 12 (audit log) |
| **§164.312(c)(1)** | Integrity | R | Module 02 + 09 (data integrity) |
| §164.312(c)(2) | Mechanism to Authenticate ePHI | A | Module 02 (data integrity checks) |
| **§164.312(d)** | Person or Entity Authentication | R | Module 02 (MFA, identity verification) |
| **§164.312(e)(1)** | Transmission Security | R | Module 02 + 12 (TLS) |
| §164.312(e)(2)(i) | Integrity Controls | A | Module 02 (signed payloads, HMAC) |
| §164.312(e)(2)(ii) | Encryption | A | Module 02 (TLS 1.2+) |

---

## Step 1: PHI inventory

Critical baseline. From Module 03 (Privacy) and direct codebase scan:

- Patient identifiers (names, MRNs, DOBs)
- Diagnoses, procedures, medications
- Lab results, imaging
- Claims / billing data
- Provider information

Document where PHI lives (DB columns, file paths, queue payloads, log
entries — *especially* logs).

If PHI is detected in logs without redaction → Critical.

---

## Step 2: BAA inventory

Every sub-processor handling PHI must have a Business Associate
Agreement. From Module 03:

- Stripe (HIPAA-eligible tier; BAA required)
- AWS (HIPAA-eligible services with BAA)
- Datadog, Sentry, etc. (HIPAA-eligible tiers vary)
- Anthropic / OpenAI (HIPAA-eligible tiers exist; specific terms apply)

Each detected sub-processor must be cross-referenced against the
project's BAA inventory. If any are on free tiers (which generally don't
include BAA) and PHI flows through them → Critical.

---

## Output: markdown report

```markdown
# HIPAA Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: HIPAA Security Rule (45 CFR Part 164 Subpart C) (fetched <date>)
Pending rulemaking: <note any in-flight HHS NPRMs>
Role: Business Associate / Covered Entity / both
Compliance automation: <Vanta / Drata / etc. / DIY>

## PHI inventory
(table: data type / location / encryption / in logs)

## BAA inventory
(table: sub-processor / handles PHI / BAA in place / tier with BAA support)

## Posture

**Critical gaps (block compliance):** <count>
**OUT_OF_BAND policies / procedures still needed:** <count>

## Top gaps
(prioritized)

## Out-of-band checklist
- Annual security risk assessment
- Workforce training records
- Sanction policy
- BAAs with all sub-processors
- Designated Security Officer
- Contingency plan documentation

## Confidence
H / M / L
```

## Output: CSV control matrix

Same schema as SOC 2 sub-ref. One row per Security Rule standard with
status, evidence pointers, gap notes, OUT_OF_BAND flag.

---

## Severity calibration

- **Critical:** PHI in non-BAA-covered service; PHI in unredacted logs;
  encryption-at-rest gap on PHI store; missing audit controls.
- **High:** missing access-control specifics, no breach-notification
  process, no contingency plan testing.
- **Medium:** policy gaps the team can fill quickly.
- **Low:** documentation polish.

HIPAA fines are tiered (Tier 1 unknowing through Tier 4 willful neglect)
and recently the maximum has trended upward — calibrate severity with
actual enforcement risk in mind.
