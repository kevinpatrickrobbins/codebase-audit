# Canadian Provincial Health Privacy Readiness Mapping

Map findings to **provincial health-information privacy laws** in Canada:

- **PHIPA** (Ontario — Personal Health Information Protection Act, 2004)
- **HIA** (Alberta — Health Information Act)
- **PHIA** (Manitoba — Personal Health Information Act)
- **PIA** (Saskatchewan — Health Information Protection Act)
- **PHIA** (Nova Scotia — Personal Health Information Act)
- **PHIA** (Newfoundland and Labrador — Personal Health Information Act)
- **PHIA** (New Brunswick — Personal Health Information Privacy and Access Act)
- **PHIA** (Prince Edward Island — Health Information Act, expected)
- (Quebec health information falls primarily under Quebec Law 25 +
  *An Act respecting health and social services information*)

These are the Canadian analog to HIPAA, but provincial-level and varying
in detail. PIPEDA does not directly apply to health-information
*custodians* in provinces with substantially-similar legislation —
provincial law applies instead.

Outputs:
- `/docs/audits/compliance/canadian-provincial-health-readiness.md`
- `/docs/audits/compliance/canadian-provincial-health-controls.csv`

---

## Live Discovery

- Fetch https://www.ipc.on.ca/en — Ontario IPC (PHIPA regulator).
- Fetch https://oipc.ab.ca/ — Alberta OIPC (HIA regulator).
- Fetch https://www.ombudsman.mb.ca/ — Manitoba Ombudsman (PHIA regulator).
- Fetch similar provincial regulators per detected jurisdiction.
- Search "<province> health privacy enforcement <current year>".

Each province updates guidance independently; fetch per detected
jurisdiction.

---

## Scope

Applies to **health information custodians** (HICs) — most commonly
healthcare providers, hospitals, regulated health professionals, public
health authorities. Also **agents** of HICs (anyone authorized to act on
behalf of a HIC, including SaaS vendors processing health data on their
behalf).

EdTech / SaaS serving Canadian health providers typically fits as an
**agent / information manager** (terminology varies by province) and is
contractually bound through agreements similar to HIPAA BAAs.

Detection signals from Module 22:
- Canadian health customers (hospitals, clinics, EMR integrations)
- Personal health information fields
- Provincial EMR system integration (e.g., Ontario eHealth, Alberta
  Netcare, etc.)

---

## Cross-jurisdictional requirements (consistent themes)

| Requirement | Code-auditable | Evidence module |
|---|---|---|
| **Custodian / agent designation** in contract | OUT_OF_BAND | — |
| **Information manager agreement** (varies by province) | OUT_OF_BAND | — |
| **Limited collection** to what's necessary | Module 03 | PII inventory |
| **Limited use and disclosure** | Module 02 + 03 | Access controls |
| **Implied / express consent** (consent model varies by province; Ontario uses circle-of-care implied consent) | Module 03 | Consent flow |
| **Lockbox / withholding directive** (right to restrict access) | Module 03 | Patient consent management |
| **Right of access** | Module 03 | DSR access |
| **Right of correction** | Module 03 | DSR rectification |
| **Reasonable security safeguards** | Module 02 | Encryption, access control, audit log |
| **Breach notification to IPC / regulator + affected individuals** (thresholds vary) | Module 14 | IR process |
| **Audit log** of every access to personal health information | Module 02 + 12 | Audit log requirement |
| **Retention per provincial requirements** (vary; often years post-discharge) | Module 03 | Retention enforcement |
| **Cross-border transfer assessment** (some provinces restrict) | Module 03 | — |

---

## Province-specific notes

### Ontario (PHIPA)

- **Lockbox** (consent directive) is statutory — patients can restrict
  specific information from being shared even within the circle of care.
- **Audit log** requirement is explicit and detailed; access logs must be
  available to the IPC on request.
- **Mandatory breach notification** to IPC for "significant" breaches
  (volume, sensitivity, recurrence considerations).
- **EMR-specific obligations** for EHR systems contributing to provincial
  EHR (Ontario Laboratories Information System, ConnectingOntario).

### Alberta (HIA)

- **Information Manager Agreement** is a defined construct under HIA
  s. 66 — required for agents/custodians using third-party processors.
- **Privacy Impact Assessment** required before implementing any new
  practice that involves health information; submit to OIPC.
- **Mandatory breach notification** since Aug 2018 to OIPC + affected
  individuals.

### Manitoba (PHIA)

- **Information Manager Agreement** required.
- **Mandatory breach notification.**

### Other provinces

Generally similar pattern; consult specific provincial guidance via
Live Discovery.

---

## Code-auditable items

- **PHI inventory.** Same as HIPAA module — patient identifiers, MRNs,
  diagnoses, lab results, claim/billing data.
- **Encryption at rest + in transit** for PHI.
- **Access controls** with audit log of every PHI access.
- **Information Manager / Service Agreement** (similar to BAA) — should
  exist with each Canadian-health customer.
- **Lockbox enforcement** (Ontario specifically) — system can restrict
  specific records per patient request.
- **Cross-border data residency.** Several provinces have restrictions
  on storing health data outside Canada; check IaC for region pins. Some
  customers will require Canadian residency contractually.

---

## Output: markdown report

```markdown
# Canadian Provincial Health Privacy Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Frameworks consulted: PHIPA / HIA / PHIA / etc. (per detected provinces)
Detected provincial scope: <Ontario / Alberta / etc.>
Compliance automation: <Vanta / Drata / etc. / DIY>

## PHI inventory
(table — cross-ref Module 03)

## Custodian-agent posture
- Customer relationships: HIC / agent / information manager
- Information Manager Agreements (Alberta, Manitoba): in place / missing
- Service Agreements with Ontario customers: in place / missing

## Cross-jurisdictional posture

(table — IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND)

## Province-specific items

### Ontario (PHIPA)
- Lockbox / consent directive support: yes / no
- Audit log per IPC requirements: yes / no

### Alberta (HIA)
- PIA submission process: documented / informal
- Information Manager Agreement template: present

### Other (per detected scope)

## Cross-border data residency
- Where is PHI stored: <region(s)>
- Provincial restrictions met: yes / no / per-customer-contract

## Top gaps

## Out-of-band checklist
- Information Manager Agreements
- Privacy Impact Assessments
- Breach notification process per province
- Designated privacy officer
- Annual training records
- Provincial regulator engagement (notification of operations)

## Confidence
H / M / L
```

## CSV matrix

Same schema. Rows per cross-jurisdictional requirement + province-
specific items.

---

## Severity calibration

- **Critical:** PHI in non-Canadian regions for province with residency
  requirement; missing Information Manager Agreement; no audit log of PHI
  access.
- **High:** missing breach notification process; lockbox not supported
  (Ontario customers); no PIA process (Alberta).
- **Medium:** OUT_OF_BAND policy gaps.
- **Low:** documentation polish.

Provincial enforcement varies — Ontario IPC and Alberta OIPC are the most
active. Custodians have been fined / publicly reported; agents are
typically held accountable through contractual + regulatory chain.
