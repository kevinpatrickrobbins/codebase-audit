# FedRAMP Readiness Mapping

Map findings to **FedRAMP** baselines (Low / Moderate / High) which
implement **NIST SP 800-53 Rev 5** controls.

Outputs:
- `/docs/audits/compliance/fedramp-readiness.md`
- `/docs/audits/compliance/fedramp-controls.csv`

---

## Live Discovery

- Fetch https://www.fedramp.gov/baselines/ — current baselines
  (the catalog is large; reference, don't fully embed).
- Fetch https://www.fedramp.gov/program-basics/ — current process state.
- Fetch https://nvd.nist.gov/800-53 — NIST 800-53 Rev 5 control catalog.
- Fetch https://www.fedramp.gov/marketplace/ — what authorization stage
  the team's offering is at (if listed).

FedRAMP is in active modernization (FedRAMP 20x program); fetch current state.

---

## Scope

FedRAMP applies when selling SaaS to US federal agencies. Three baselines:

| Baseline | Use case | Approximate control count |
|---|---|---|
| **Low** | Public information; minimal sensitivity | ~125 controls |
| **Moderate** | Most agency data | ~325 controls |
| **High** | Sensitive law enforcement / health / finance / national security | ~425 controls |

Most commercial SaaS targeting fed: **Moderate**.

---

## Audit posture

A code-audit covers a meaningful subset of the technical controls. The
authorization process itself requires:

- A 3PAO (Third Party Assessment Organization) assessment
- A Plan of Action & Milestones (POA&M)
- An Authorization to Operate (ATO) — sponsored by an agency or via the
  FedRAMP Joint Authorization Board

This module produces a *readiness signal*, not the authorization package.

---

## NIST 800-53 Rev 5 control families (at-a-glance)

NIST 800-53 controls are grouped by family. Code-auditable subset:

| Family | Description | Code-auditable | Evidence module |
|---|---|---|---|
| **AC** | Access Control | High | Module 02 |
| **AT** | Awareness and Training | OUT_OF_BAND | — |
| **AU** | Audit and Accountability | High | Module 02 + 12 |
| **CA** | Assessment, Authorization, Monitoring | OUT_OF_BAND | — |
| **CM** | Configuration Management | High | Module 12 + 14 |
| **CP** | Contingency Planning | Medium | Module 12 |
| **IA** | Identification and Authentication | High | Module 02 |
| **IR** | Incident Response | Medium | Module 14 |
| **MA** | Maintenance | OUT_OF_BAND mostly | — |
| **MP** | Media Protection | OUT_OF_BAND mostly | — |
| **PE** | Physical and Environmental | OUT_OF_BAND (cloud-inherited) | — |
| **PL** | Planning | OUT_OF_BAND | — |
| **PM** | Program Management | OUT_OF_BAND | — |
| **PS** | Personnel Security | OUT_OF_BAND | — |
| **PT** | PII Processing and Transparency | Medium | Module 03 |
| **RA** | Risk Assessment | Module 00.2 + OUT_OF_BAND | — |
| **SA** | System and Services Acquisition | Medium | Module 14 |
| **SC** | System and Communications Protection | High | Module 02 + 12 |
| **SI** | System and Information Integrity | High | Module 02 + 06 + 12 |
| **SR** | Supply Chain Risk Management | Medium | Module 08 |

For the readiness CSV, map each in-baseline control (Low/Mod/High) to a
status. Use the high-leverage families (AC, AU, CM, IA, SC, SI) as the
focus — these are most code-auditable.

---

## FedRAMP-specific code-auditable items

- **FIPS 140-2/3 cryptography** — verify cryptographic libraries used
  meet FIPS validation. Some cloud providers offer FIPS-validated modes
  (AWS KMS FIPS endpoints, etc.). Look for explicit configuration.
- **US-only data residency** — all data and control plane in US regions.
  Check IaC for region pins.
- **Continuous monitoring** — automated security scanning + log shipping
  to a federally-acceptable destination.
- **Boundary diagram** — required artifact; OUT_OF_BAND.
- **System Security Plan (SSP)** — required document; OUT_OF_BAND.
- **POA&M (Plan of Action & Milestones)** — tracks open findings.
  Cross-reference Module 00.3 (Remediation Plan).

---

## Output: markdown report

```markdown
# FedRAMP Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: NIST SP 800-53 Rev 5 + FedRAMP baselines (fetched <date>)
Target baseline: Low / Moderate / High
Authorization status: <none / in progress / sponsored by agency X / on marketplace>
Compliance automation: <Vanta / Drata / etc. / DIY>

## Readiness summary

- Code-auditable controls in baseline: <N>
- IMPLEMENTED: <count>
- PARTIAL: <count>
- GAP: <count>
- OUT_OF_BAND: <count>

## FedRAMP-specific items

- FIPS-validated cryptography: yes / no / partial
- US-only data residency: yes / no
- Continuous monitoring: configured / partial / absent
- Boundary diagram: present (OUT_OF_BAND) / missing
- System Security Plan: present (OUT_OF_BAND) / missing
- POA&M tracking: present / use Module 00.3 output as basis

## Top gaps

(prioritized — typically Access Control, Audit, System Communications)

## Out-of-band checklist

- 3PAO engagement
- Boundary diagram
- System Security Plan (SSP)
- Personnel security (background checks, US persons clearance if needed)
- Configuration management plan documentation
- Contingency plan documentation
- Continuous monitoring strategy

## Confidence
H / M / L
```

## CSV matrix

Same schema. ~125 / ~325 / ~425 rows depending on baseline.

---

## Severity calibration

- **Critical:** non-FIPS cryptography in cryptographic-control path; data
  residency outside US; missing access control baseline.
- **High:** audit logging gaps; missing continuous monitoring.
- **Medium:** OUT_OF_BAND policy/document gaps.
- **Low:** documentation refinements.

FedRAMP is a multi-month / multi-year process; this readiness view is for
preparing the engagement, not substituting for it.
