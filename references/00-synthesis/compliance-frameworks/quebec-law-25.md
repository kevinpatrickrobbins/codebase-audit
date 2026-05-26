# Quebec Law 25 Readiness Mapping

Map findings to **Quebec Law 25** (formerly Bill 64; *Act to modernize
legislative provisions as regards the protection of personal information*).
Notably stricter than federal PIPEDA — closer to GDPR.

Outputs:
- `/docs/audits/compliance/quebec-law-25-readiness.md`
- `/docs/audits/compliance/quebec-law-25-controls.csv`

---

## Live Discovery

- Fetch https://www.cai.gouv.qc.ca/english — Commission d'accès à l'information du Québec (CAI), the regulator.
- Search "Quebec Law 25 enforcement actions <current year>" — recent CAI rulings.
- Search "Quebec Law 25 cross-border transfer assessment" — guidance evolves.

Phased in 2022 (governance), 2023 (consent + DSR), 2024 (data portability).
Mostly fully in force.

---

## Scope

Applies to any **enterprise carrying on activity in Québec** that collects,
holds, uses, or communicates personal information of Québec residents —
regardless of where the enterprise is established. US SaaS with Quebec
customers is in scope.

Detection signals from Module 01 jurisdictional landscape:
- Customers in Quebec (signals: French-Canadian locale `fr-CA`, Quebec
  references in marketing, billing addresses, currency CAD)
- Hosting in Canada (presumptive)
- Bilingual French/English UI (Charter of the French Language overlap)

---

## Required practices

| Requirement | Code-auditable | Evidence module |
|---|---|---|
| **Designate Person Responsible for Protection of Personal Information** | OUT_OF_BAND | — |
| **Privacy Impact Assessment** for new tech projects involving PI | OUT_OF_BAND | — |
| **Express consent** for collection / use / disclosure (opt-in, not buried) | Module 03 | Consent mechanism quality |
| **Granular consent** per purpose | Module 03 | Per-purpose consent flow |
| **Plain-language privacy policy** posted prominently, in French | Module 03 + OUT_OF_BAND | Policy content review |
| **Confidentiality incident notification** — to CAI and affected individuals if "risk of serious injury" | Module 02 + 14 | IR process + breach notification |
| **Confidentiality incident register** maintained | OUT_OF_BAND + Module 02 | — |
| **Cross-border transfer assessment** (PIA before transferring outside Quebec) | OUT_OF_BAND + Module 03 | Sub-processor docs |
| **Data minimization** | Module 03 | PII inventory |
| **Retention limits** (no longer than necessary) | Module 03 | Retention enforcement |
| **Right of access** | Module 03 | DSR access |
| **Right of rectification** | Module 03 | DSR rectification |
| **Right of erasure** | Module 03 | DSR erasure |
| **Right of data portability** (since Sept 2024) | Module 03 | DSR portability |
| **Right of de-indexation** ("right to be forgotten" against search engines) | Module 03 | DSR de-indexation |
| **Automated decision-making transparency** + right to be informed + human review | Module 18 | AI/ML disclosure |
| **Profiling restrictions** (must inform; opt-out for marketing) | Module 03 + 18 | — |
| **Confidentiality by default** (privacy-by-default settings) | Module 03 | Default settings |

---

## Specific code-auditable items

- **French-language posting.** Privacy policy, consent screens, account
  settings — all available in French. Charter of the French Language
  overlap; for Quebec consumers, French is required as the default.
- **Granular consent.** Single "I agree to everything" checkbox is
  insufficient. Each distinct purpose (marketing, analytics, profiling)
  needs its own consent.
- **Cross-border transfer flag.** When Quebec PI flows to US providers
  (AWS US regions, OpenAI, etc.), Law 25 requires a PIA before transfer.
  If automated, evidence of the assessment should exist; if not, flag.
- **Profiling / automated decisions.** Any algorithmic decision affecting
  the user (eligibility, risk scoring, ad targeting, recommendation
  ranking) requires disclosure + opt-out for marketing + right to human
  review.
- **De-indexation request handling.** Users can request that public-facing
  content about them be removed / de-indexed. Process must exist.

---

## Output: markdown report

```markdown
# Quebec Law 25 Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: Law 25 (RLRQ c P-39.1) (fetched <date>)
Compliance automation: <Vanta / Drata / etc. / DIY>

## Scope assessment

- Quebec residents in user base: yes / no / inferred from <evidence>
- French-language UI: present / absent
- Cross-border transfers from Quebec: yes / no — destinations

## DSR posture (compared to GDPR equivalents — typically aligned)

- Access / Rectification / Erasure / Portability / De-indexation

## Required practice posture

(table — IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND)

## Cross-border transfer assessment

- Sub-processors receiving Quebec PI
- PIA documented for each: yes / no
- Adequacy assessment

## French-language compliance

- Privacy policy in French: yes / no
- Consent screens: French default / English-only / parallel
- Account settings: localized

## Top gaps

## Out-of-band checklist
- Designated Person Responsible
- Privacy Impact Assessments documented
- Confidentiality incident register
- Cross-border transfer PIAs
- French-language privacy policy

## Confidentiality incident posture
(per Module 02 + 14)

## Confidence
H / M / L
```

## CSV matrix

Same schema as other framework sub-refs.

---

## Severity calibration

- **Critical:** no privacy policy in French for Quebec users; no DSR
  paths; cross-border transfer of Quebec PI without PIA; no incident
  notification process.
- **High:** consent mechanism is single-checkbox not granular; no
  designated Person Responsible.
- **Medium:** OUT_OF_BAND policy / PIA gaps; retention not enforced.
- **Low:** policy translation polish.

CAI penalties under Law 25: administrative monetary penalties up to
**$10M or 2% of worldwide turnover** (whichever greater) for the most
serious violations. Enforcement actions have been increasing since
phased-in completion.
