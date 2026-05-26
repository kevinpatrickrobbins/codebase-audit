# DORA Readiness Mapping

Map findings to **DORA** (EU Regulation 2022/2554 — Digital Operational
Resilience Act). Effective **17 January 2025** for financial entities and
critical ICT third-party providers.

Outputs:
- `/docs/audits/compliance/dora-readiness.md`
- `/docs/audits/compliance/dora-controls.csv`

---

## Live Discovery

- Fetch https://www.eiopa.europa.eu/digital-operational-resilience-act-dora_en — current state.
- Fetch https://www.esma.europa.eu/policy-activities/digital-finance-and-innovation/digital-operational-resilience-act-dora — ESMA guidance.
- Search "DORA RTS ITS <current year>" — Regulatory Technical Standards / Implementing Technical Standards.
- Search "DORA enforcement <current year>" — recent enforcement actions.

DORA has multiple Level 2 measures (RTS / ITS) that have been published
in tranches; check current set.

---

## Scope

Applies to:

1. **Financial entities** operating in the EU:
   - Credit institutions
   - Payment institutions, e-money institutions
   - Investment firms
   - Crypto-asset service providers (under MiCA)
   - Central securities depositories, central counterparties
   - Trading venues, trade repositories
   - Insurance and reinsurance undertakings
   - Insurance / pension fund intermediaries
   - Credit rating agencies
   - Audit firms (ICT auditing)
   - Crowdfunding service providers
   - Securitisation repositories

2. **ICT third-party service providers** that serve financial entities,
   particularly **critical ICT third-party service providers (CTPPs)**
   designated by the European Supervisory Authorities — directly
   supervised under the EU Oversight Framework.

If the project is a SaaS serving financial-sector customers, DORA
flow-down through customer contracts likely affects the project even if
not directly in scope.

Detection signals from Module 22 / Module 01:
- Banking / fintech / insurance / investment customers
- Direct registration as a financial entity in any EU member state
- MiCA crypto-asset offering
- Existing customer contracts with DORA-related provisions

---

## Five pillars

### Pillar 1: ICT risk management (Articles 5–16)

Comprehensive ICT risk management framework. Code-auditable elements:

| Domain | Module |
|---|---|
| Information security policy | OUT_OF_BAND + Module 02 |
| Identification of ICT-supported business functions | Module 01 |
| Continuous risk identification | Module 00.2 |
| Protection and prevention | Module 02 + 12 |
| Detection (anomalous activities, single points of failure) | Module 12 + 14 |
| Response and recovery | Module 12 + 14 |
| Backup, restoration, recovery procedures | Module 12 Step 8 |
| Learning and evolving | Module 14 (postmortems) |
| Communication | Module 14 |

### Pillar 2: ICT incident reporting (Articles 17–23)

Major ICT-related incidents require classification + reporting to
competent authorities:

| Stage | Deadline | Content |
|---|---|---|
| **Initial notification** | 24h after classification as major | Type, area affected, business impact |
| **Intermediate report** | 72h after initial OR upon material change | Updated info, mitigations underway |
| **Final report** | **1 month** after final root-cause analysis | Detailed description, root cause, action plan |

Significant cyber threats may also require reporting (voluntary or
mandatory depending on classification).

### Pillar 3: Digital operational resilience testing (Articles 24–27)

- **Basic testing** for all in-scope entities: vulnerability assessments,
  open-source assessments, network security assessments, gap analyses,
  physical reviews, source code reviews where feasible, scenario-based
  tests, compatibility tests, performance tests, end-to-end tests,
  penetration tests.
- **Threat-Led Penetration Testing (TLPT)** for significant entities:
  every 3 years; conducted by external testers; live production systems.
  Aligns with TIBER-EU framework where adopted.

### Pillar 4: ICT third-party risk management (Articles 28–44)

Pre-contract assessment + ongoing monitoring + concentration risk
analysis + exit strategies for ICT third-party service providers.
Contract requirements (Article 30) are extensive and prescriptive.

### Pillar 5: Information sharing arrangements (Article 45)

Voluntary cyber-threat-information sharing with peer financial entities,
ESAs, ENISA, etc.

---

## Code-auditable specifics

- **ICT asset inventory** including criticality classification.
- **MFA** for privileged + remote access.
- **Encryption** in transit and at rest.
- **Network segmentation** isolating critical systems.
- **Logging** with sufficient retention to support root-cause analysis.
- **Incident classification logic** in monitoring stack — major incidents
  trigger appropriate workflows within DORA timelines.
- **Backup + restore drill** evidence (cross-ref Module 12 Step 8).
- **Vulnerability management cadence** (cross-ref Module 08).
- **Third-party SBOM / inventory** for ICT services used (cross-ref
  Module 03 + Module 01.5 Stack Brief).

---

## Output: markdown report

```markdown
# DORA Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: Regulation (EU) 2022/2554 + Level 2 measures (fetched <date>)
Entity classification: Financial entity / ICT third-party / out of scope
Compliance automation: <Vanta / Drata / etc. / DIY>

## Scope determination

- Direct financial entity classification: <type or N/A>
- ICT services provided to financial-sector customers: yes / no
- Critical ICT third-party designation likely: yes / no / unclear

## Five-pillar posture

### Pillar 1: ICT risk management
(table — IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND)

### Pillar 2: Incident reporting
- Major incident classification logic: present / absent
- 24h / 72h / 1-month timeline achievable: yes / no
- Authority contact documented: yes / no

### Pillar 3: Resilience testing
- Basic testing program: present / absent
- TLPT applicable + scheduled: yes / no / not significant
- Last test date: <date or never>

### Pillar 4: Third-party risk management
- ICT third-party inventory: present / absent
- Pre-contract assessment process: documented / informal / absent
- Critical ICT third parties identified: <list or "none">
- Contract requirements (Article 30): met / partial / unknown

### Pillar 5: Information sharing
- Threat-intelligence sharing membership: yes / no / N/A

## Top gaps

## Out-of-band checklist
- ICT risk management policy
- Information security policy
- Incident classification & response plan
- Resilience testing program plan
- Third-party risk management policy
- Exit strategies for critical ICT third parties

## Confidence
H / M / L
```

## CSV matrix

Same schema. Rows per Article + sub-domain.

---

## Severity calibration

- **Critical:** financial entity in scope with no incident classification
  process; missing MFA for privileged access; no backup posture; no ICT
  third-party inventory.
- **High:** Pillar 1 domain GAP; reporting timelines not achievable;
  TLPT applicable but never scheduled.
- **Medium:** OUT_OF_BAND policy gaps; basic testing not formalized.
- **Low:** documentation polish.

Penalties: **financial entities** — up to **2% of total annual worldwide
turnover** (administrative). **ICT third parties** designated as critical —
up to **1% of average daily worldwide turnover** (daily fines, capped at
6 months). Penalties stack across violations.
