# NIS2 Directive Readiness Mapping

Map findings to **NIS2** (EU Directive 2022/2555 — Network and Information
Security 2). Replaces NIS Directive (2016) and dramatically expands scope.

Outputs:
- `/docs/audits/compliance/nis2-readiness.md`
- `/docs/audits/compliance/nis2-controls.csv`

---

## Live Discovery

- Fetch https://digital-strategy.ec.europa.eu/en/policies/nis2-directive — Commission overview.
- Search "NIS2 national implementation <member-state> <current year>" — implementation status varies by country.
- Search "ENISA NIS2 implementing regulation" — ENISA technical guidance + implementing acts.

National-law transposition deadline: **17 October 2024**. Several member
states ran late; check current state per the specific country the project
operates in.

---

## Scope — much broader than NIS1

NIS2 applies to **essential** and **important entities** in 18 sectors.

### Essential entities (Annex I)

- Energy (electricity, district heating, gas, oil, hydrogen)
- Transport (air, rail, water, road)
- Banking
- Financial market infrastructures
- Health (healthcare providers, EU reference labs, R&D, manufacturers,
  pharma)
- Drinking water
- Wastewater
- **Digital infrastructure** — IXP, DNS service providers (excl. root),
  TLD registries, cloud computing service providers, data centre
  service providers, content delivery network providers, **trust
  service providers**, public electronic communications networks /
  services
- **ICT service management (B2B)** — managed service providers, managed
  security service providers
- Public administration
- Space

### Important entities (Annex II)

- Postal and courier services
- Waste management
- Manufacture, production, distribution of chemicals
- Production, processing, distribution of food
- Manufacturing (medical devices, computer/electronic, electrical
  equipment, machinery, motor vehicles, transport equipment)
- **Digital providers** — online marketplaces, online search engines,
  social networking platforms
- Research

### Size threshold

Default: **medium-sized or larger** (50+ employees or €10M+ turnover).
Some categories captured regardless of size (DNS, TLD registries, trust
service providers, electronic comms providers, public administration,
critical sector below threshold).

**Most cloud / SaaS / managed-service products at meaningful scale fall
in scope** — broader than many teams realize.

Detection signals from Module 01 jurisdictional landscape:
- EU customers / EU presence
- B2B SaaS providing managed services to other businesses
- Cloud / data-center / CDN offering
- Online marketplace / search / social

---

## Core obligations (Article 21)

NIS2 doesn't prescribe specific controls; it requires "appropriate and
proportionate" measures across these areas:

| # | Domain | Code-auditable | Evidence module |
|---|---|---|---|
| (a) | Risk analysis and information system security policies | OUT_OF_BAND + Module 00.2 | — |
| (b) | Incident handling | Module 14 | IR runbook + postmortems |
| (c) | Business continuity, backup management, disaster recovery, crisis management | Module 12 | Backup + restore drill |
| (d) | Supply chain security (incl. supplier and service provider security) | Module 03 + 08 | Sub-processor + dependency audits |
| (e) | Security in network / system acquisition, development, maintenance, vulnerability disclosure | Module 02 + 06 + 14 | SDLC, vuln management |
| (f) | Policies and procedures to assess effectiveness of cybersecurity measures | OUT_OF_BAND + Module 00.4 | — |
| (g) | Basic cyber hygiene practices and cybersecurity training | OUT_OF_BAND | — |
| (h) | Policies and procedures regarding the use of cryptography and encryption | Module 02 | Crypto inventory |
| (i) | Human resources security, access control policies, asset management | Module 02 + 14 | — |
| (j) | Multi-factor authentication or continuous authentication, secured voice/video/text comms, secured emergency comms | Module 02 | MFA enforcement |

---

## Incident reporting requirements (Article 23)

Significant incidents require staged reporting:

| Stage | Deadline | Content |
|---|---|---|
| **Early warning** | Within **24 hours** of awareness | Whether the incident is malicious, cross-border, etc. |
| **Incident notification** | Within **72 hours** of awareness | Initial assessment, severity, impact, IoCs |
| **Final report** | Within **1 month** of notification | Detailed description, type of threat, root cause, mitigation |

Reports go to the national CSIRT or competent authority.

Auditable in code: presence of an incident-reporting workflow that can
meet these timelines (process docs + automated severity classification +
contact with national CSIRT documented in runbook).

---

## Management body accountability (Article 20)

Senior management (board / executives) must:

- **Approve** the cybersecurity risk-management measures
- **Oversee** their implementation
- **Be trained** in cybersecurity (regular refresher)

Personally accountable for breaches of these obligations. OUT_OF_BAND from
code; flag for the team.

---

## Output: markdown report

```markdown
# NIS2 Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: Directive (EU) 2022/2555 + national transposition (fetched <date>)
Member state(s) of operation: <list>
National implementation status per member state: <link or note>
Compliance automation: <Vanta / Drata / etc. / DIY>

## Scope determination

- Essential / Important entity: <classification + sector>
- Size threshold met: yes / no / not applicable to category
- In scope under <member state> implementation: yes / no / borderline

## Article 21 measures posture

(table — IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND across the 10 domains)

## Article 23 incident reporting readiness

- 24h early warning capability: yes / no
- 72h incident notification capability: yes / no
- 1-month final report capability: yes / no
- National CSIRT contact documented in runbook: yes / no

## Article 20 management body accountability

- Cybersecurity training program for senior leadership: OUT_OF_BAND
- Risk-management measures approved by management body: OUT_OF_BAND

## Top gaps

## Out-of-band checklist
- Risk analysis documented
- Incident response policy
- Business continuity plan
- Supply chain security policy
- Cyber hygiene training records
- Management body training records

## Confidence
H / M / L
```

## CSV matrix

Same schema. Rows per Article 21 sub-domain + Article 23 + Article 20.

---

## Severity calibration

- **Critical:** in-scope entity with no incident response plan; no MFA for
  privileged access; no backup posture.
- **High:** Article 21 domain has GAP; incident reporting timelines not
  achievable.
- **Medium:** OUT_OF_BAND policy gaps; supply chain inventory absent.
- **Low:** documentation polish.

Penalties: **essential entities** up to €10M or 2% of global annual
turnover (whichever higher); **important entities** up to €7M or 1.4%.
National-level enforcement varies; consult the local DPA / cyber-security
authority for the specific member state.
