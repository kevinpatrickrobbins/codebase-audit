# 03 — Privacy & Data Protection

Audit the codebase against privacy regulations (GDPR, CCPA/CPRA, PIPEDA, PIPL,
LGPD) and the actual data-handling practices the code reveals. This module is
the legal-risk parallel to accessibility (`02-compliance/accessibility.md`).

Output: `/docs/audits/privacy-audit.md`

---

## When to apply

Run on any product that handles personally identifiable information (PII) —
which is almost every product with users. Skip only if the codebase clearly
processes no personal data (e.g., a math library).

---

## Cross-module relationship

- **Security** (`02-compliance/security.md`) covers the *security* of personal
  data handling (encryption, access control). This module covers the *legal
  posture* (consent, retention, rights).
- **Product Gap** (`06-product/gaps.md`) flags missing legal pages (Privacy
  Policy, ToS); this module audits whether the *code* matches what those
  pages claim.
- **Accessibility** (`02-compliance/accessibility.md`) is the parallel
  jurisdiction-aware compliance audit.

---

## Step 0: Live Discovery

Privacy regulations and enforcement state move continuously — adequacy
decisions, regulator guidance, fine schedules, and cross-border transfer
mechanisms change month-to-month. Fetch current truth before mapping the
codebase against any one regulation.

### Current GDPR / UK GDPR state

- **EU-US Data Privacy Framework (DPF):** Fetch
  https://commission.europa.eu/law/law-topic/data-protection/international-dimension-data-protection/eu-us-data-transfers_en —
  current adequacy decision state. The DPF has been challenged in court
  (Schrems-class litigation); if invalidated, Standard Contractual Clauses
  + Transfer Impact Assessment become mandatory again.
- **EDPB guidelines:** Fetch
  https://www.edpb.europa.eu/our-work-tools/general-guidance/guidelines-recommendations-best-practices_en —
  currently published guidelines (cookies, legitimate interest, AI
  training data, etc.).
- **Recent enforcement:** Search "GDPR major fines <current year>" —
  the largest recent fines reveal current enforcement priorities (often
  ad-tech, large-scale profiling, AI training data).

### Current US state privacy law landscape

- **CCPA / CPRA (California):** Fetch https://oag.ca.gov/privacy/ccpa —
  current rules; CPPA regulations evolve.
- **State law roster:** Search "US state privacy laws <current year>" —
  Virginia, Colorado, Connecticut, Utah, Texas, Oregon, Montana, Iowa,
  Tennessee, Indiana, Delaware, New Hampshire, New Jersey, Minnesota,
  Maryland have or are enacting comprehensive privacy laws. Each has its
  own DSR mechanism, opt-out, and business thresholds — capture the
  count of laws currently in effect.

### Other major jurisdictions

- **PIPEDA (Canada federal):** Fetch
  https://www.priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/ —
  current state; reform via Bill C-27 / CPPA may be in progress.
- **Quebec Law 25:** Fetch https://www.cai.gouv.qc.ca/ — current
  CAI guidance.
- **LGPD (Brazil):** Fetch https://www.gov.br/anpd/pt-br — ANPD
  current enforcement state.
- **PIPL (China):** Search "PIPL cross-border transfer <current year>" —
  CAC security assessments, standard contracts, certification routes.
- **DPDP Act (India):** Fetch
  https://www.meity.gov.in/data-protection-framework — current
  implementing rules status.

### Output of discovery

```markdown
## Live Discovery (fetched YYYY-MM-DD)

| Source | Discovered |
|---|---|
| EU-US DPF (commission.europa.eu) | <current adequacy state; any pending challenge> |
| EDPB guidelines | <recent guidelines relevant to the project's stack> |
| Recent GDPR enforcement | <top 3 recent fines + theme> |
| US state landscape | <count of states with laws in effect + new this year> |
| PIPEDA reform status | <Bill C-27 / CPPA current status> |
| PIPL transfer mechanisms | <current accepted routes> |

Audit calibrated against this state.
```

If discovery fails: note in "Out of Scope / Inconclusive". The embedded
regulation list below reflects this skill's last update and may be stale.

---

## Step 1: Compliance surface and jurisdictions

Determine which regulations apply:

| Regulation | Triggered by | Required posture |
|---|---|---|
| **GDPR** (EU) | EU residents using the product | Lawful basis, consent for non-essential, DSR rights, DPA, sub-processor list |
| **UK GDPR** | UK residents | Same as GDPR; ICO is the regulator |
| **CCPA / CPRA** (California) | CA residents; biz threshold | "Do Not Sell" / "Do Not Share" controls, DSR rights |
| **PIPEDA** (Canada federal) | Commercial activity in Canada | Consent, breach notification, DSR |
| **Quebec Law 25** | Quebec residents | Stricter than PIPEDA; explicit consent, DPIA |
| **PIPL** (China) | PRC residents; data export rules | Cross-border transfer requirements, separate consent |
| **LGPD** (Brazil) | Brazil residents | Mirrors GDPR closely |
| **Sector laws** (HIPAA, COPPA, GLBA, FERPA) | Sector | Out-of-scope for this module — flag for sector audit |

Output a posture table for each applicable regime.

---

## Step 2: PII inventory from code

Walk the codebase and inventory every place PII is collected, stored, or
transmitted:

- **Form inputs.** Grep for `<input>` fields collecting names, emails,
  addresses, phone, DOB, ID numbers, payment data, IP addresses, biometrics.
- **DB schemas.** Read schema files (`schema.prisma`, models) for PII
  columns. Flag any unencrypted column likely holding sensitive data.
- **API endpoints.** List endpoints accepting PII payloads. Each should be
  authenticated unless explicitly public (signup).
- **Logs.** Search for `console.log`, `logger.info` with user objects or
  request bodies — PII in logs is a common breach vector and a GDPR
  violation when not necessary.
- **Analytics events.** Search `track(`, `identify(`, `page(`, `mixpanel`,
  `amplitude`, `posthog`, `segment` calls — what user properties are
  shipped to third parties?
- **LLM prompts** — if the project uses LLMs, are user PII fields ever
  embedded in prompts that go to OpenAI/Anthropic/etc.? That's a sub-processor
  relationship requiring DPA.

Output the inventory as a table: PII type | Where collected | Where stored |
Where transmitted | Encrypted at rest? | In logs? | DPA required?

---

## Step 3: Consent mechanism

- **Cookie consent.** If the product uses non-essential cookies / trackers
  (analytics, ads, Hotjar, Intercom, Sentry session replay), a consent
  banner is required for EU users — and must actually *gate* the loading
  of those scripts. A banner that loads after the trackers fire is theatre.
- **Search the codebase** for: cookie consent libraries (`cookie-consent`,
  `klaro`, `cookiebot`, `osano`, `iubenda`, custom). Check whether scripts
  load before or after consent.
- **Defaults.** "All accept" by default fails GDPR; "all reject" or
  per-category granular consent passes.
- **Withdrawal.** Can the user change their mind? UI exists?
- **Records of consent.** Stored with timestamp, IP, version of the policy
  consented to?
- **Marketing consent** — separate from cookies; explicit opt-in for
  marketing emails (GDPR + CASL in Canada).

---

## Step 4: Data subject rights (DSR)

GDPR requires actionable paths for:

- **Access** (Article 15) — give the user a copy of their data.
- **Rectification** (Article 16) — let them correct it.
- **Erasure** (Article 17) — "right to be forgotten."
- **Portability** (Article 20) — machine-readable export.
- **Object** (Article 21) — opt out of marketing / profiling.

**Investigate:**

- Self-serve **export** path? (Search routes for `/export`, `/data-export`,
  `/account/data`.)
- Self-serve **delete account** path that actually deletes (not soft-delete
  for marketing)? Required by App Store policy 5.1.1(v) too.
- **Rectification** — can users edit their own data?
- **Marketing opt-out** — visible unsubscribe link, honors the choice,
  reflects in DB?

Each missing right is a Critical-tier finding for EU-serving products.

---

## Step 5: Retention

- **Retention policies** documented and enforced in code? Search for
  scheduled jobs that purge old records (`pg_cron`, `cron.daily`, queue
  jobs).
- **Inactive account deletion** — does the product delete data of users who
  haven't logged in for X years?
- **Backup retention** — separate from active data; documented?
- **Logs retention** — application logs, audit logs, server access logs each
  with their own clock?

Indefinite retention without a documented purpose is a GDPR violation.

---

## Step 5.5: Logging discipline (privacy lens)

(Cross-ref Module 02 Security Step 11 for the security lens on logging.)

From the privacy angle:

- **PII redaction at the logger level.** Allow-list / deny-list of fields
  in logger config (pino's `redact`, winston transforms, custom
  middleware). Manual per-call redaction rots; logger-level is durable.
- **Logs as data subject scope.** When a user requests deletion, logs
  containing their PII fall under the request. If logs are shipped to a
  third party (Datadog, Sentry, BetterStack), the deletion path must
  include those.
- **Sensitive-event logs.** "User signed in", "user changed password",
  "user accessed export" — are themselves a privacy-relevant data type.
  Treat as PII.
- **IP address logging.** IP is PII under GDPR. Indefinite IP retention
  in access logs is a violation; document the retention.
- **Logging in third-party SDKs.** Sentry, LogRocket, FullStory, Hotjar,
  Heap may capture session data including form input, URLs with query
  strings (potentially containing tokens / PII). Each is a sub-processor
  relationship requiring DPA + the same redaction discipline. PII
  scrubbing config in the SDK init?

## Step 6: Third-party data flow

- **Sub-processors** — list every third-party service that receives user
  data. Cross-check the project's published sub-processor list (if any).
- **DPA** with each sub-processor required for GDPR.
- **International transfers.** EU data going to US — Standard Contractual
  Clauses (SCCs), or under the EU-US Data Privacy Framework adequacy
  decision (since July 2023).
- **Trackers loaded.** Inventory every `<script>` from a third-party domain
  in the page output.
- **Pixel / fingerprinting** — Facebook Pixel, Google Tag, TikTok Pixel,
  LinkedIn Insight, etc. — each is a sub-processor relationship.

---

## Step 7: Data breach posture

- **Audit log** with actor / timestamp / target / change for sensitive
  data access. Required for breach forensics.
- **PII access logged** distinctly from general request logs.
- **Breach notification process** — documented? GDPR requires 72-hour
  notification; CCPA has its own clock.
- **Encryption at rest** for sensitive columns (`pgcrypto`, application-level
  AES, vault-managed keys).

---

## Step 8: Children's privacy (if applicable)

- **COPPA** (US) — under 13. Parental consent required.
- **GDPR Article 8** — under 16 (or member-state-set 13–16). Same.
- **Age gate** — does the signup ask for age? If absent, the product cannot
  claim it doesn't process children's data.

---

## Red flags

- Cookie banner that doesn't actually gate script loading.
- Default "all accept" consent.
- No DSR export or delete path on a product serving EU.
- Privacy Policy page references practices the code contradicts.
- PII logged to stdout / files in production.
- No DPA story for OpenAI/Anthropic/etc. when LLM prompts include user PII.
- Indefinite retention with no documented purpose.
- Sub-processor list absent or stale.
- Third-party trackers loading before consent.
- No breach-notification documentation.

---

## Output structure

```markdown
# Privacy & Data Protection Audit

Date: YYYY-MM-DD
Repository commit: <hash>

## Compliance surface
| Regulation | In scope? | Posture | Confidence |
|---|---|---|---|

## PII Inventory
(table: type / collected / stored / transmitted / encrypted / in logs / DPA)

## Consent mechanisms
- Cookie consent (gate or theatre?)
- Marketing consent (explicit opt-in?)
- Records of consent stored

## Data Subject Rights
- Access / Export
- Erasure
- Rectification
- Portability
- Object / Opt-out

## Retention
- Documented policies
- Active enforcement in code (cron jobs, scheduled deletes)
- Backup and log retention

## Third-party data flow
- Sub-processor inventory
- DPA status
- International transfer mechanism (SCCs / adequacy)
- Tracker inventory

## Breach posture
- Audit log quality
- Encryption at rest
- Notification process

## Children's privacy
(if applicable)

## Findings
(severity / category / evidence / fix)

## Final recommendation
Single highest-priority privacy action.

## Decision summary
- DSR rights all available: yes/no
- Cookie consent gates trackers: yes/no
- Sub-processor list current: yes/no
- Worst gap: <description>

## Confidence
High / Medium / Low

## Out of Scope / Inconclusive
```

---

## Severity calibration

- **Critical:** EU-serving product missing erasure/export, or PII logged to
  unsanitized destinations, or trackers firing before consent.
- **High:** missing DPAs, indefinite retention without documented purpose,
  audit log absent on PII-touching admin actions.
- **Medium:** consent UX flaws, missing sub-processor list, retention
  policies documented but not enforced in code.
- **Low:** style issues in the privacy policy, inconsistent terminology.

Every finding cites the file/line and (for legal claims) the relevant article
of the regulation.
