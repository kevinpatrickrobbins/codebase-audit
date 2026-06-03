# Product-Type Sub-Reference: Marketplace

Audit a two-sided marketplace (sellers + buyers, hosts + guests, drivers +
riders, freelancers + clients). Marketplaces add risk surfaces that single-
sided SaaS doesn't have: payment splitting, dispute handling, trust signals,
two-sided onboarding.

---

## Detection

- Two distinct user roles with separate signup / onboarding flows
- Payment splitting: Stripe Connect, Adyen MarketPay, PayPal Marketplaces
- Reviews / ratings tied to one or both actors
- Listing / inventory model (one side creates listings the other side
  consumes)
- Booking / transaction lifecycle (request → accept → complete → review)

---

## Investigate

### Two-sided onboarding

- **Distinct flows.** Separate signup / verification / KYC for each side;
  trying to share a single signup usually fails one side or the other.
- **Verification depth.** Sellers/hosts/providers usually need more verification
  (identity, payout details, tax info) than buyers.
- **Empty side / cold start.** First-100 sellers gating buyer experience —
  documented or hand-curated? (This is product, not code, but flag if the
  app assumes both sides are populated and falls over otherwise.)
- **Profile completeness.** Required fields per side; review/approval
  workflow for new sellers.

### Payments

- **Connected accounts.** Stripe Connect (Standard / Express / Custom),
  Adyen MarketPay, etc. Implementation in `lib/payments/` or similar.
- **Payment intent flow.** Direct, Destination, or Separate Charges &
  Transfers? Determines who is responsible for disputes and refunds.
- **Application fee.** Platform fee deducted at charge time (clean) vs
  separate transfer (double-bookkeeping).
- **Payout schedule.** Per-seller cadence; min payout thresholds; bank
  verification status checks before scheduling.
- **Tax.** Stripe Tax / TaxJar integration for marketplace tax obligations.
  US sales tax for marketplaces is increasingly the platform's burden
  (marketplace facilitator laws); EU OSS / IOSS for cross-border.
- **1099-K reporting (US).** For US marketplaces, required reporting on
  sellers above thresholds.
- **Refunds.** Buyer-initiated, dispute-driven, manual. Who eats the platform
  fee on refund?
- **Dispute / chargeback handling.** Webhook handlers for `charge.dispute.*`
  events; evidence submission flow.

### Trust & safety

- **Reviews.** Two-sided review (host reviews guest + guest reviews host)
  released simultaneously to prevent retaliation. Single-sided review systems
  are weaker.
- **Review fraud prevention.** Limits on reviewing without a transaction;
  velocity checks; vote-rigging detection.
- **Fraud detection.** Stripe Radar for payments; in-app signals for fake
  listings (image hashes, duplicate text, suspicious pricing).
- **Reporting / flagging.** Buyers can report listings; sellers can report
  buyers. Triage workflow.
- **KYC / AML.** Stripe Identity, Persona, Onfido for high-value
  marketplaces.
- **Account age / reputation surfaces** (badges, verified, top-rated).

### Listings & inventory

- **Listing creation.** Validation, image hosting, content moderation
  (manual or AI-driven).
- **Search & discovery.** Module 09 for perf; here, the relevance ranking and
  filter taxonomy. Algolia / Typesense / Postgres FTS — chosen appropriately
  for catalog size?
- **Geolocation.** Latitude/longitude indexing (PostGIS, geohash) for
  location-aware search.
- **Availability / inventory model.** Calendars, bookings, capacity.
  Concurrency: two buyers booking the same slot — locking or optimistic with
  rollback?

### Communication

- **In-app messaging** between sides — not just email, to keep PII off-platform.
- **PII redaction.** Phone numbers, email addresses, off-platform payment
  links blocked or warned in messages (if business model depends on
  on-platform transactions).
- **Block / mute** functionality.

### Cancellations & disputes

- **Cancellation policy** machine-readable (so it can be enforced and
  refunded automatically).
- **Dispute escalation path** to human support; SLA documented.
- **Mediation** capability for partial refunds.

### Compliance

- **Marketplace facilitator tax laws** (US, EU, Canada, etc.) — platform
  liability for tax collection.
- **Consumer protection.** EU Modernization Directive, UK CMA — required
  pre-contract disclosures.
- **GDPR.** Two-sided data sharing — what does each side see about the
  other? Documented in privacy policy.
- **AML.** Above thresholds.
- **Accessibility.** Cross-reference Module 04.

### Operational

- **Two-sided rate limits.** Spamming sellers via DMs vs spamming listings
  via low-effort buyers.
- **Anti-abuse bans.** Account-level vs payment-method-level vs IP-level.
- **Per-region rollout.** Marketplaces are very local — feature flags by
  region.

---

## Red flags

- Single signup / role flag instead of distinct flows.
- Direct charge + manual transfer instead of Connect / MarketPay.
- One-sided reviews (only buyers review sellers, or vice versa).
- No PII-redaction in in-app messaging despite on-platform-only business
  model.
- Stripe webhooks for `charge.dispute.*` not handled.
- No tax integration in jurisdictions with marketplace facilitator laws.
- No cancellation policy enforced server-side.
- KYC absent for high-value transactions.
- Search by `LIKE '%term%'` on a 100k+ listing table.

---

## Output section

```markdown
### Marketplace

#### Two-sided onboarding
- Distinct flows: yes/no
- Verification depth per side
- Profile review/approval

#### Payments
- Splitting: <Connect Standard/Express/Custom | MarketPay | other>
- Payment intent flow
- Application fee model
- Payout schedule
- Tax integration
- 1099-K (US) handling
- Refund and chargeback paths

#### Trust & safety
- Two-sided reviews
- Fraud detection (Radar / custom)
- Reporting / flagging
- KYC / AML

#### Listings & inventory
- Search / discovery (perf cross-ref: Module 09)
- Geolocation / PostGIS
- Availability concurrency

#### Communication
- In-app messaging
- PII redaction (if relevant)
- Block / mute

#### Cancellations & disputes
- Machine-readable policies
- Dispute SLA

#### Compliance
- Marketplace facilitator tax laws
- Consumer protection (jurisdiction-specific)
- GDPR posture
- AML / KYC scope

#### Operations
- Two-sided rate limits
- Anti-abuse strategy
- Per-region rollout

#### Findings
(table)
```
