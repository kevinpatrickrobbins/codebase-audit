# CASL Readiness Mapping

Map findings to **Canada's Anti-Spam Legislation** (CASL — S.C. 2010, c. 23).
Governs commercial electronic messages (CEMs), software installation, and
address harvesting.

Outputs:
- `/docs/audits/compliance/casl-readiness.md`
- `/docs/audits/compliance/casl-controls.csv`

---

## Live Discovery

- Fetch https://crtc.gc.ca/eng/internet/anti.htm — CRTC enforcement guidance.
- Fetch https://fightspam.gc.ca/eic/site/030.nsf/eng/home — official CASL site.
- Search "CASL enforcement <current year>" — recent CRTC penalties.

CASL has been fully in force since 2014. Private right of action (s. 51)
was indefinitely suspended in 2017; CRTC remains the primary enforcer.

---

## Scope

Applies to **commercial electronic messages (CEMs)** sent to or from
**any Canadian computer system**. CEM includes:

- Email
- SMS / MMS
- Social-media DMs
- App push notifications (when commercial in nature)
- Voice messages (some — overlaps with telemarketing rules)

Anyone sending CEMs to Canadian recipients is in scope, regardless of
sender location. US SaaS sending marketing emails to Canadian customers
is in scope.

Detection signals from Module 01 / Module 03:
- Email-sending integrations (Resend, Postmark, SendGrid, Mailgun, SES)
- Marketing automation tools (Customer.io, Mailchimp, HubSpot,
  ConvertKit)
- Push notification services (Expo Push, OneSignal, Firebase)
- SMS providers (Twilio, Vonage)
- Canadian residents in user base

---

## Three pillars

CASL has three core requirements for CEMs:

### 1. Consent

Sender must have **express** or **implied** consent before sending a CEM.

| Type | Definition | Validity |
|---|---|---|
| **Express consent** | Recipient affirmatively opts in (checkbox unchecked-by-default; signup including email-marketing checkbox where opt-in is voluntary; explicit form) | Indefinite until revoked |
| **Implied consent — existing business relationship** | Purchase, contract, written application, inquiry from recipient within last 6 months (or 24 months for purchase/contract) | Time-limited (6 / 24 months) |
| **Implied consent — non-business relationship** | Donation / volunteer for registered charity within 2 years; membership in club/association | Time-limited |
| **Conspicuous publication** | Recipient has prominently published their email without "no marketing" notice + message is relevant to their role | Narrow exception |

Pre-checked boxes are NOT express consent. Bundled consent (terms +
marketing in one click) is risky.

### 2. Identification

Every CEM must clearly identify:

- Sender (name of business sending; if on behalf of someone else, both)
- Mailing address
- Other contact info (phone, email, or website) valid for at least 60
  days

### 3. Unsubscribe

Every CEM must include a **functional unsubscribe mechanism**:

- Free of charge
- Accessible without additional information
- Processed within **10 business days**
- Valid for at least 60 days from message send

---

## Code-auditable items

- **Opt-in flow.** Email-marketing signup uses unchecked checkbox or
  explicit "Subscribe" action — not pre-checked, not bundled with terms
  acceptance.
- **Consent records.** When consent is obtained, store: timestamp, source
  (signup form, purchase, etc.), exact wording shown. Required for
  proof-of-consent in CASL enforcement.
- **Unsubscribe link in every CEM.** Search email templates for
  `{{unsubscribe_url}}` or equivalent. Missing in any CEM template = a
  finding.
- **Unsubscribe processing.** When a user unsubscribes, are they
  removed from sending lists within 10 business days? (Most ESPs handle
  this automatically; verify integration is wired up.)
- **Suppression list.** Unsubscribed addresses are added to a permanent
  suppression list and respected across all sending domains.
- **Sender identification in templates.** Footer of every CEM template
  includes business name, mailing address, contact info.
- **Implied-consent expiry tracking.** If relying on implied consent
  (purchase, inquiry), track the date and stop sending after the 6/24-month
  window.
- **Push notification opt-in.** Marketing push notifications need consent
  (typically captured via OS prompt + in-app toggle). Transactional pushes
  (order shipped, message received) are generally outside CASL scope.

---

## Output: markdown report

```markdown
# CASL Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: CASL (S.C. 2010, c. 23) (fetched <date>)
Compliance automation: ESP (Resend / Postmark / etc.) typically handles
mechanics; verify configuration.

## CEM-sending integrations detected

| Channel | Provider | Templates count | Templates audited |
|---|---|---|---|

## Three-pillar posture

### Consent
- Opt-in flow type: <express checkbox / implied / mixed>
- Consent records stored: yes / no
- Implied-consent expiry tracked: yes / no / not relying on it

### Identification
- All templates include business name, address, contact: yes / no / partial
- Templates missing identification: <list>

### Unsubscribe
- Unsubscribe link in every template: yes / no / partial
- Unsubscribe processing wired up: yes / no / depends on provider
- Suppression list maintained: yes / no
- Cross-domain suppression: yes / no

## Push notification compliance
(if applicable)

## Top gaps

## Out-of-band checklist
- Internal CASL compliance policy
- Training for marketing staff on consent
- Consent records system (CRM / database)
- Process for handling complaints

## Confidence
H / M / L
```

## CSV matrix

Same schema. Rows for each requirement.

---

## Severity calibration

- **Critical:** sending CEMs without consent (express or implied);
  unsubscribe link missing or non-functional; sending after unsubscribe
  request beyond 10 business days.
- **High:** pre-checked consent boxes; bundled consent with terms;
  identification missing from templates.
- **Medium:** consent records not stored (proof-of-consent gap);
  implied-consent expiry not tracked.
- **Low:** template polish.

CRTC penalties: up to **$1M per violation** (individuals) or **$10M per
violation** (organizations). Each CEM can be a separate violation;
volumes add up fast.
