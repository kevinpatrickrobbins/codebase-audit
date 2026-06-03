# FERPA Readiness Mapping

Map findings to **FERPA** (Family Educational Rights and Privacy Act —
US, Department of Education) requirements.

Outputs:
- `/docs/audits/compliance/ferpa-readiness.md`
- `/docs/audits/compliance/ferpa-controls.csv`

---

## Live Discovery

- Fetch https://www2.ed.gov/policy/gen/guid/fpco/ferpa/index.html — current FERPA state.
- Fetch https://studentprivacy.ed.gov/ — Student Privacy Policy Office guidance.
- Search "FERPA edtech vendor school official exception <current year>" — current interpretive guidance.

---

## Scope

FERPA protects **education records** maintained by schools / districts
receiving federal funds, and by their service providers.

EdTech SaaS typically fits via the **School Official exception** —
school agrees in contract to treat the vendor as a "school official"
performing a service the school would otherwise perform itself.

Detection signals (from Module 22):
- LMS / SIS integration (Canvas, Blackboard, Schoology, PowerSchool, Clever)
- K-12 / higher-ed customer language
- Student data fields (grades, attendance, enrollment, behavior)
- "School District" / "University" in customer types

---

## Key requirements

| Requirement | Code-auditable | Evidence module |
|---|---|---|
| **Direct control by school** over the educational records the vendor holds | OUT_OF_BAND (contract) | — |
| **Use limited** to what the school authorized | Medium | Module 03 |
| **No re-disclosure** without school authorization | Module 03 | Sub-processor list |
| **Reasonable methods to identify and authenticate** access requests | Module 02 | Access control |
| **Records protected** with reasonable security | Module 02 + 12 | Encryption, access control |
| **Annual notification of rights** to parents / eligible students | OUT_OF_BAND | School handles |
| **Right to inspect and review** records | Module 03 | DSR access |
| **Right to seek amendment** of records | Module 03 | DSR rectification |
| **Directory information vs PII** distinction | Medium | Module 03 |

---

## Specific code checks

- **Educational records inventory.** Identify all "education records"
  the system stores (cross-ref Module 03 PII inventory): grades,
  transcripts, attendance, disciplinary records, behavioral records,
  enrollment status, demographic info.
- **Access controls per school / district.** Multi-tenant scoping
  (cross-ref Module 19 SaaS) — staff at School A cannot access School B's
  records.
- **Role-based within school.** Teacher sees only their classes; counselor
  sees more; admin sees most. RBAC depth.
- **Data sharing with third parties.** Any analytics / advertising /
  research integration must be authorized by the school under the
  contract terms (cross-ref Module 03 sub-processor inventory).
- **Audit log of record access.** Who viewed what record when. Required
  for breach forensics and potential parent/student-initiated audit.
- **Data retention.** Records retained per the school's policy, not
  vendor default. Schools have varied retention requirements (some
  permanent, some by years-since-graduation).
- **Data deletion on contract termination.** When the school ends the
  contract, vendor returns or destroys records per the agreement.
- **Surveillance / behavioral monitoring concerns.** AI features that
  surveil student behavior (proctoring, plagiarism detection,
  social-emotional analysis) have additional state-law overlay (e.g.
  several states have student-data privacy laws stricter than FERPA).

---

## Output: markdown report

```markdown
# FERPA Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: FERPA (20 USC § 1232g; 34 CFR Part 99) (fetched <date>)
Compliance automation: <Vanta / Drata / etc. / DIY>

## Scope assessment

- Vendor role: School Official (typical for SaaS)
- Customers: K-12 / higher-ed / both
- Education records held: <inventory>

## Required practice posture

(table — IMPLEMENTED / PARTIAL / GAP / OUT_OF_BAND)

## Multi-tenant isolation
(cross-ref Module 19 SaaS — required for FERPA when serving multiple districts)

## State-law overlay
- Student Online Personal Information Protection Act (SOPIPA — California
  and similar in many states)
- Specific state student-data privacy laws
- (Note: this audit focuses on federal FERPA; state laws may impose
  additional requirements)

## Top gaps

## Out-of-band checklist
- Data Privacy Agreement (DPA) with each district
- School Official contract language
- Annual security audit
- Breach notification process documentation

## Confidence
H / M / L
```

## CSV matrix

Same schema.

---

## Severity calibration

- **Critical:** cross-district / cross-school data exposure; missing
  access controls on educational records; analytics SDKs sharing student
  data without authorization.
- **High:** missing audit log of record access; data retention not
  configurable per school.
- **Medium:** documentation gaps (DPA boilerplate, breach process).
- **Low:** UI polish.

Note: FERPA enforcement is by Department of Education and is rare; loss
of federal funding is the technical penalty but more commonly the
practical risk is district contracts / state attorney general actions.
