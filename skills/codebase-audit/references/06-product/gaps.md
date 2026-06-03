# 16 — Product Gap Analysis

Evaluate the application as if it were preparing for public release.

Create or update:

`/docs/audits/product-gap-analysis.md`

---

## How to investigate

Product gap analysis asks: "what would a reasonable user expect to be in this app
that isn't?" The risk is filling in *what you would build* rather than what is
actually missing for *this* product. Calibrate to the project's actual scope: a
B2B internal tool does not need consumer-grade onboarding; a payments platform
must.

### Step 1: Anchor on the product type

Before listing gaps, classify the product. The gaps differ.

- **Consumer SaaS** (signup → use): needs frictionless signup, password reset,
  email verification, account settings, billing, account deletion, support.
- **B2B SaaS** (seat-based, often invite-only): needs team management, roles,
  audit logs, SSO, billing tied to seats, data export.
- **Internal tooling** (auth via SSO, no public signup): can skip much of the
  consumer onboarding; needs strong audit logs and admin-only paths.
- **Marketplace** (two-sided): needs onboarding for each side, payouts, dispute
  flows, ratings.
- **Agentic / AI tool** (LLM-backed): needs cost guardrails, content filtering,
  rate limits, output history.
- **Mobile app + backend**: needs offline support, app-store assets, push
  notifications, deep links.

If the product type is unclear, look at signup flow + monetization model + auth
config to infer it. State the inferred type explicitly in the report — later
sections depend on it.

### Step 2: Walk the standard production checklist

For each item, check the codebase for evidence. Mark **Present**, **Partial**, or
**Missing**, with file pointers.

#### Authentication & accounts
- Signup flow (cite the route)
- Login flow
- Email verification
- Password reset / forgot password
- 2FA / MFA option
- Social login / SSO
- Profile editing (name, email, avatar)
- Password change (separate from reset)
- Email change with re-verification
- Account deletion (legally required in many jurisdictions — GDPR, CCPA)
- Data export (legally required — GDPR Article 20)
- Session management (see active sessions, revoke)

#### Billing (if monetized)
- Pricing page
- Checkout flow
- Subscription management (upgrade, downgrade, cancel)
- Invoice download
- Payment method update
- Failed payment handling
- Refund flow (or documented policy)
- Tax handling (Stripe Tax, Paddle, manual)
- Receipts / billing emails

#### Communication
- Transactional emails (signup, reset, billing)
- Notifications (in-app, email, push)
- Notification preferences
- Marketing emails (with proper opt-out)

#### Admin & ops
- Admin dashboard (user list, force-reset, impersonate, suspend)
- Audit log (who did what when)
- Customer support contact path (ticket form / email / chat link)
- Status page (or link to one)
- Feature flags (LaunchDarkly, GrowthBook, ConfigCat, custom)

#### Data & content
- Search
- Filters / sort on lists
- Pagination on growing lists
- Bulk actions
- Empty states
- Onboarding state (sample data, getting-started checklist)

#### Errors & resilience
- Custom 404 page
- Custom 500 page
- Network error states (offline indicator, retry)
- Form errors (per-field, summary)
- Optimistic updates with rollback

#### Legal & compliance
- Terms of Service link
- Privacy Policy link
- Cookie consent (if EU-relevant — overlap with Module 04)
- GDPR data subject rights surfacing (export, delete)
- Imprint / Impressum (German market)

#### Discoverability (B2C / public web)
- SEO basics: `<title>`, `<meta description>`, Open Graph, Twitter Cards
- Sitemap, robots.txt
- Structured data (JSON-LD) for rich results
- 301 redirects from old URLs (if migrated)

#### Operability
- Health endpoint (overlap with Module 12)
- Status page (overlap with Module 12)
- Monitoring & alerting (overlap with Module 12)
- Backup & restore tested (overlap with Module 12)

### Step 3: Edge cases

- Failed states: what happens if the third-party API the app depends on goes
  down? Stripe is down — can users still log in?
- Concurrent edits: two users editing the same resource — last-write-wins?
  optimistic locking? collaborative?
- Data migrations: how does the app handle a user with stale data from an old
  schema?
- Internationalization: hardcoded English strings limit market reach. Check for
  `next-intl`, `react-i18next`, locale files.
- Time zones: stored as UTC and displayed in user TZ? Or stored as local and
  ambiguous?

### Step 4: Documentation gaps

User-facing docs:
- Help center / FAQ
- Onboarding guide
- API docs (if API is public)
- Changelog / release notes

Developer-facing docs:
- README quickstart accurate? (Try to mentally follow it.)
- Architecture / contributing docs?
- ADRs (architectural decision records)?
- Runbooks for ops scenarios?

### Step 5: Pre-launch quality gates

If the product is preparing for launch:

- Load testing run? (Grep for `k6`, `artillery`, JMeter scripts.)
- Penetration testing planned or done? (Module 02 is a code audit, not a pentest.)
- Beta / early access program?
- Rollout plan (gradual? all-at-once? feature-flagged?)
- Customer support staffed for launch?

---

## Red flags

- No way to delete an account.
- No password reset.
- No email verification on signup.
- Marketing site exists but no Terms / Privacy.
- App stores PII but has no GDPR data export path.
- Admin actions taken silently with no audit log.
- Billing exists but no way to cancel without contacting support.
- 404 / 500 pages are framework defaults with no useful info.
- Customer support contact is missing or buried.
- Hardcoded English copy in a product targeting non-English markets.

---

## Output structure

```markdown
# Product Gap Analysis

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Product Type
Inferred type (consumer SaaS / B2B / etc.) and the evidence that led to that
inference. Subsequent sections calibrate to this type.

## Current Feature Set
Summary of implemented capabilities (cite key route files).

## Missing Core Features
Categorized:
- Authentication & accounts
- Billing (if applicable)
- Communication
- Admin & ops
- Data & content
- Errors & resilience
- Legal & compliance
- Discoverability
- Operability

For each missing item: severity, expected effort, why it matters for THIS
product type.

## Edge Cases
Scenarios that will hit production but appear unhandled. Cite the absent
handlers.

## Documentation Gaps
User-facing and developer-facing.

## Pre-Launch Quality Gates
What launch-readiness work has been done; what is missing.

## Recommended Product Roadmap
Phased: P0 (blocks launch), P1 (first 30 days), P2 (first 90 days). Each item
references a finding above.

## Final Recommendation
The single highest-impact product action for this codebase right now.
**One thing.**

## Decision Summary

- Production-ready for stated user type: yes / no / partial
- Worst gap: <description>
- Recommended posture: <ship-blockers fixed | launch-readiness work | bigger product re-scope>

## Confidence
High / Medium / Low (and why)

## Out of Scope / Inconclusive
Items that depend on product context this audit doesn't have (e.g., target
markets, monetization plans).
```

---

## Severity calibration

- **Critical:** legal/regulatory non-compliance (GDPR delete missing, no privacy
  policy on a public product), or core path missing (no password reset).
- **High:** common-expectation gaps that erode trust (no 404, no support contact,
  no account settings).
- **Medium:** quality-of-life gaps (no bulk actions, no search filters).
- **Low:** nice-to-haves and discoverability polish.

---

## Evidence requirements

Every "Missing" claim is grounded in *not finding* something. Describe what you
searched for and where. "No password reset route — searched for `/forgot-password`,
`/reset`, `password.reset`, and route files containing 'password' — found only
`/login` and `/signup`."
