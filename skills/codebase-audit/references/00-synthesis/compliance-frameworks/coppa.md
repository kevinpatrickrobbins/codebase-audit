# COPPA Readiness Mapping

Map findings to **COPPA** (Children's Online Privacy Protection Rule —
US, FTC) requirements.

Outputs:
- `/docs/audits/compliance/coppa-readiness.md`
- `/docs/audits/compliance/coppa-controls.csv`

---

## Live Discovery

- Fetch https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa — current rule.
- Search "COPPA rule update <current year>" — FTC has been actively
  rulemaking; check for changes (e.g. updated definitions, broader scope,
  edtech-specific provisions).

---

## Scope

COPPA applies to **operators of online services directed to children
under 13** OR services with **actual knowledge** that they collect
personal information from children under 13.

Detect signals from Module 22 (Sector Compliance):
- Marketing copy referencing children, kids, teens-13-and-under
- Features designed for children (age-appropriate UI)
- Education products serving K-6
- Games rated for children
- Signup that doesn't ask for age (creates "actual knowledge" risk)

---

## Required practices

| # | Practice | Code-auditable | Evidence module |
|---|---|---|---|
| 1 | **Notice on website / service** about info-collection practices | OUT_OF_BAND (Privacy Policy) | Module 03 |
| 2 | **Verifiable parental consent** before collecting personal info from children | High | Module 02 + 03 |
| 3 | **Parents can review** what's collected | Module 03 | DSR rights |
| 4 | **Parents can revoke consent** and require deletion | Module 03 | Erasure |
| 5 | **Limit collection** to what's necessary for the activity | Module 03 | PII inventory minimization |
| 6 | **Reasonable security measures** for kids' info | Module 02 | Encryption, access control |
| 7 | **Data retention** only as long as necessary | Module 03 | Retention enforcement |
| 8 | **No targeted advertising** to children | Module 03 + 02 | Ad SDK presence + targeting config |
| 9 | **Age gating** to detect children's signups (recommended) | Module 02 + 03 | Signup flow age check |

---

## Specific code checks

- **Age gate at signup.** Search for date-of-birth / age field in signup
  components. If absent + product targets / could attract children →
  Critical (creates "actual knowledge" exposure).
- **Parental consent flow.** If under-13 signup is allowed, where's the
  verifiable parental consent (email, payment-card verification, signed
  consent form)?
- **Ad SDK presence.** Search for AdMob, Google Ads, IronSource, Unity
  Ads, etc. Targeted advertising to children is prohibited; even
  contextual ads in children's products require careful configuration.
- **Third-party tracking.** Analytics SDKs that may collect persistent
  identifiers from children (Google Analytics, Mixpanel, Facebook Pixel,
  TikTok Pixel) — for kids' products these typically require COPPA-safe
  configuration or removal.
- **Geolocation collection.** "Geolocation information sufficient to
  identify street name and city" is COPPA-protected for under-13s.
- **Photo/video/audio collection** containing a child's image or voice =
  COPPA-protected.
- **Chat / community features.** "Mechanism to allow children to share
  personally identifiable information with others" requires either
  monitoring or explicit parental consent.

---

## Output: markdown report

```markdown
# COPPA Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: COPPA Rule (16 CFR Part 312) (fetched <date>)
Pending FTC rulemaking: <note any in-flight changes>
Compliance automation: <Vanta / Drata / etc. / DIY>

## Scope assessment

- Product directed to children under 13: yes / no / inferred from <evidence>
- Actual knowledge of under-13 users: yes / no / risk (no age gate)
- COPPA in scope: yes / no / risk

## Children's data inventory

- PII fields collected from children (cross-ref Module 03)
- Geolocation: yes / no
- Photo / audio: yes / no
- Persistent identifiers (analytics, ads): list

## Required practice posture

(table — IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND for each of 9 practices)

## Top gaps
(prioritized)

## Out-of-band checklist
- Privacy notice tailored to COPPA
- Parental consent process documentation
- FTC Safe Harbor program participation (optional)

## Confidence
H / M / L
```

## CSV matrix

Same schema. ~9 high-level practice rows + sub-items.

---

## Severity calibration

- **Critical:** product clearly targets children + no parental consent
  flow; targeted advertising to children's product; geolocation
  collected from minors without parental consent.
- **High:** no age gate on a product that's likely to attract minors;
  third-party tracking SDKs without COPPA-safe configuration.
- **Medium:** retention policies not enforced; privacy notice not
  COPPA-tailored.
- **Low:** documentation polish.

FTC enforcement: per-violation civil penalties scale (currently
$50,120/violation or higher per recent enforcement). Calibrate accordingly.
