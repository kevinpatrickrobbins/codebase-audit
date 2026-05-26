# PCI DSS Readiness Mapping

Map findings to **PCI DSS 4.0** requirements and produce a readiness
report + CSV control matrix.

Outputs:
- `/docs/audits/compliance/pci-dss-readiness.md`
- `/docs/audits/compliance/pci-dss-controls.csv`

---

## Live Discovery

- Fetch https://www.pcisecuritystandards.org/document_library/ — current PCI DSS version.
- Fetch https://www.pcisecuritystandards.org/standards/pci_dss/ — current state of 4.0 enforcement.

PCI DSS 4.0 is fully effective since March 2025. The 12 high-level
requirements are stable across versions; specific testing procedures
evolve.

---

## Step 1: SAQ scope determination

This is the most important early step — determines the *size* of the
audit.

| SAQ | Scope | Typical use case |
|---|---|---|
| **SAQ A** | Card data fully outsourced (Stripe Elements, Adyen iframes, hosted checkout) | Stripe Checkout / Stripe Elements with card data never touching your servers |
| **SAQ A-EP** | Merchant website redirects but server processes some card data | Server-side Stripe.js with card data passing through your server |
| **SAQ B** | Card-present, terminal-only | Physical retailers with imprint or standalone terminals |
| **SAQ B-IP** | IP-connected POS only | Networked POS terminals |
| **SAQ C** | Payment app + internet, segmented | Standalone payment app on a segmented network |
| **SAQ C-VT** | Virtual terminal | Web-based virtual terminal with manual entry |
| **SAQ D-MER** | All other merchant scenarios | Storing or processing PAN; complex environments |
| **SAQ D-SP** | Service provider | Platform serving merchants, processes / transmits CHD |

Most SaaS that ever touches PAN should target **SAQ A** (smallest burden)
by using Stripe Elements / Adyen iframes / similar. SAQ A-EP and SAQ D
are dramatically more burdensome.

Detect in the codebase:
- `stripe-js` Elements integration → SAQ A candidate
- Server-side `req.body.card_number` patterns → SAQ A-EP at minimum, possibly SAQ D
- Stored `card_number` / `pan` columns → SAQ D + significant additional controls
- Hosted Checkout redirect → SAQ A

If detected scope > SAQ A, that itself is a finding ("payment integration
has unnecessary cardholder data scope; consider migrating to Stripe
Elements / Adyen iframes for SAQ A scope reduction").

---

## Step 2: Map findings to the 12 PCI DSS Requirements

| Req | Title | Evidence module(s) | Code-auditable? |
|---|---|---|---|
| **1** | Install and maintain network security controls | Module 12 + 02 | Partial — VPC, firewall, segmentation visible in IaC |
| **2** | Apply secure configurations | Module 12 + 08 | Partial — base image hardening, default-cred review |
| **3** | Protect stored account data | Module 02 + 03 | Yes — encryption at rest, tokenization, no CVV storage |
| **4** | Protect cardholder data with cryptography during transmission | Module 02 | Yes — TLS 1.2+, modern cipher suites |
| **5** | Protect all systems and networks from malicious software | OUT_OF_BAND mostly | Endpoint AV (out of code scope); container scanning (Module 08) |
| **6** | Develop and maintain secure systems | Module 02 + 06 + 14 | Yes — secure SDLC, change mgmt, vulnerability mgmt, code review |
| **7** | Restrict access by need to know | Module 02 + 19 SaaS | Yes — RBAC, role definitions |
| **8** | Identify users and authenticate access | Module 02 | Yes — unique IDs, MFA, password policy, session mgmt |
| **9** | Restrict physical access | OUT_OF_BAND | Inherited from cloud provider |
| **10** | Log and monitor all access | Module 02 + 12 + 14 | Yes — audit logs, log retention, monitoring |
| **11** | Test security regularly | Module 06 + 08 | Yes — vulnerability scanning, pen testing recommendation |
| **12** | Support information security with policies | OUT_OF_BAND | Policies, training, incident response |

---

## Step 3: Specific controls to verify in code

The headline checks for SaaS:

- **No PAN storage** unless tokenization platform requires it. Search for
  `card_number`, `pan`, `card_no` columns or fields.
- **CVV is NEVER stored** post-authorization. Search for `cvv`, `cvc`,
  `card_cvv` storage references.
- **Tokenization in use** for any persistent reference to a payment
  method (Stripe Customer / Payment Method ID, not the raw PAN).
- **TLS 1.2+** with modern cipher suite list.
- **Cardholder data scope** — what subset of the codebase touches CHD?
  Quarantine that to a minimal set of files / services for easier audit.
- **Network segmentation** — CHD-handling services isolated from rest of
  infrastructure if SAQ D scope.
- **Quarterly external scans** by an Approved Scanning Vendor (ASV) —
  evidence (scan reports) is OUT_OF_BAND but the team should know.
- **Annual penetration test** for SAQ D / service provider scope —
  evidence is OUT_OF_BAND.
- **PCI DSS 4.0 specific:** authenticated internal vulnerability scans,
  expanded encryption requirements, MFA for all access into cardholder
  data environment (not just admin).

---

## Output: markdown report

```markdown
# PCI DSS Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: PCI DSS 4.0 (fetched <date>)
SAQ scope detected: <A / A-EP / D / D-SP / etc.>
Compliance automation: <Vanta / Drata / etc. / DIY>

## SAQ scope assessment

Detected scope: <SAQ level>
Rationale: <evidence — Stripe Elements, server-side card handling, stored PAN, etc.>
Scope reduction recommendations: <if applicable — migrate to Stripe Elements, etc.>

## PAN / CVV inventory

- PAN storage: yes / no — locations
- CVV storage: NEVER acceptable post-auth — flagged if any
- Tokenization in use: yes / no

## Posture per requirement

(12 sections, one per req — IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND)

## Top gaps
(prioritized)

## Out-of-band checklist
- Quarterly ASV scans
- Annual penetration test (SAQ D / SP)
- Information security policy
- Security awareness training
- Incident response plan documentation

## Confidence
H / M / L
```

## Output: CSV control matrix

Same schema. One row per requirement (12 high-level + sub-requirements
where the team's scope expands them).

---

## Severity calibration

- **Critical:** any CVV storage; PAN storage without tokenization;
  unencrypted PAN transmission.
- **High:** SAQ scope larger than necessary (cost burden); MFA gaps for
  CHD-environment access; no quarterly ASV scan evidence.
- **Medium:** TLS cipher hygiene; non-critical control gaps.
- **Low:** documentation polish.

PCI DSS non-compliance can void merchant agreements with payment
processors; calibrate accordingly.
