# GLBA Safeguards Rule Readiness Mapping

Map findings to **GLBA Safeguards Rule** (16 CFR Part 314) requirements
for financial-services products.

Outputs:
- `/docs/audits/compliance/glba-readiness.md`
- `/docs/audits/compliance/glba-controls.csv`

---

## Live Discovery

- Fetch https://www.ftc.gov/business-guidance/privacy-security/gramm-leach-bliley-act — Gramm-Leach-Bliley Act current state.
- Fetch https://www.ftc.gov/legal-library/browse/rules/safeguards-rule — Safeguards Rule current text.
- Search "FTC Safeguards Rule update <current year>" — significant 2023 amendments expanded scope and added MFA + breach notification requirements; check current state.

---

## Scope

GLBA applies to **financial institutions** (broadly defined under FTC
jurisdiction — includes banks, lenders, mortgage brokers, tax prep,
collection agencies, financial advisors, payday lenders, motor vehicle
dealers, retailers offering credit, and many fintech products).

The Safeguards Rule requires implementing an **Information Security
Program** (ISP) to protect customer information.

Detection signals from Module 22 (Sector Compliance):
- Lending / loan origination flows
- Investment / brokerage features
- Tax preparation
- Credit reporting / scoring
- Mortgage servicing
- Banking / wallet features
- Customer financial data (account numbers, balances, credit data)

---

## Required practices (Safeguards Rule §314.4)

The 2023 amended rule requires:

| # | Practice | Code-auditable | Evidence module |
|---|---|---|---|
| (a) | **Designate a qualified individual** responsible for the ISP | OUT_OF_BAND | — |
| (b) | **Risk assessment** documented in writing | OUT_OF_BAND + Module 00.2 | — |
| (c) | **Design and implement safeguards** to control identified risks | High | All modules |
| (c)(1) | Access controls (authentication, authorization) | High | Module 02 |
| (c)(2) | Inventory of data, personnel, devices, systems, facilities | Medium | Module 01 + 03 |
| (c)(3) | **Encryption** of customer information in transit and at rest | High | Module 02 + 12 |
| (c)(4) | Secure development practices for in-house apps | Medium | Module 14 |
| (c)(5) | **Multi-factor authentication** for any individual accessing customer info | High | Module 02 |
| (c)(6) | Secure data disposal | Module 03 (retention + deletion) | — |
| (c)(7) | Change management procedures | Module 14 | — |
| (c)(8) | Monitor and log activity of authorized users; detect unauthorized access | Module 02 + 12 | — |
| (d) | **Regular testing** of safeguards (penetration testing or vuln assessment + continuous monitoring) | Medium | Module 06 + 08 |
| (e) | **Security awareness training** for all personnel | OUT_OF_BAND | — |
| (f) | **Service provider oversight** (vendor due diligence, contractual safeguards) | OUT_OF_BAND + Module 03 | — |
| (g) | **Adjust ISP** based on changes (testing, monitoring, threats) | OUT_OF_BAND | — |
| (h) | **Written incident response plan** | Module 14 + OUT_OF_BAND | — |
| (i) | **Annual reporting** to board of directors / senior officer | OUT_OF_BAND | — |
| §314.5 | **Breach notification** to FTC for incidents affecting ≥ 500 consumers (added 2024) | OUT_OF_BAND | — |

---

## Specific code-auditable items

- **MFA enforcement.** The 2023 amendments require MFA for any
  individual accessing customer information. Verify MFA is *enforced*
  (not just available) for staff access to financial-data systems.
- **Encryption at rest** for customer financial information stores.
- **Encryption in transit** with current TLS.
- **Audit log of customer-data access.** Every read of customer
  financial info logged with actor, timestamp, scope.
- **Customer financial data inventory.** What columns / files / queues
  hold financial data (cross-ref Module 03 PII inventory).
- **Vendor inventory.** Sub-processors handling customer financial data
  (cross-ref Module 03).
- **Secure disposal.** When customer relationships end, data deletion
  scheduled per retention policy.

---

## Output: markdown report

```markdown
# GLBA Safeguards Rule Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: 16 CFR Part 314 (Safeguards Rule, 2023 amendments) (fetched <date>)
Compliance automation: <Vanta / Drata / etc. / DIY>

## Scope assessment

- Financial institution under GLBA: yes / no / borderline
- Customer financial information categories: <inventory>

## Required practice posture

(table — IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND)

## 2023 amendments specific

- MFA enforced for all customer-info access: yes / no / partial
- Breach notification process for ≥ 500 consumers: documented / absent
- Encryption (in transit + at rest): yes / no / partial
- Annual reporting to board: OUT_OF_BAND

## Top gaps
(prioritized)

## Out-of-band checklist
- Designated Qualified Individual
- Written risk assessment
- Written incident response plan
- Annual board / management reporting
- Service provider oversight (DPAs, due diligence)
- Security awareness training records

## Confidence
H / M / L
```

## CSV matrix

Same schema. Rows for each Safeguards Rule §314.4 sub-element.

---

## Severity calibration

- **Critical:** unencrypted customer financial data at rest; MFA absent
  for staff access to customer financial info; missing audit log of
  customer data access.
- **High:** missing breach notification process; vendor without DPA
  handling customer financial data.
- **Medium:** OUT_OF_BAND policy gaps.
- **Low:** documentation refinements.

FTC enforcement of GLBA Safeguards has been increasing — recent consent
orders include significant monetary judgments. Calibrate accordingly.
