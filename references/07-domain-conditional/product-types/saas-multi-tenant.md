# Product-Type Sub-Reference: SaaS (Multi-Tenant)

Audit a multi-tenant SaaS product. The defining concern is tenant isolation —
the difference between a small SaaS and a big breach is usually one missed
authorization check.

---

## Detection

- Data model with `tenant_id`, `organization_id`, `team_id`, `workspace_id`
  columns on most user-data tables
- Auth/data model with `Organization` / `Team` / `Workspace` primitives
- Pricing/billing routes (`/pricing`, `/billing`, `/checkout`)
- Subscription / seat semantics in code (Stripe Subscriptions, Lago, Orb,
  Polar)
- Org/team invite flows

A B2C product with public signup and no team primitive is *not* multi-tenant
by this definition — flag and skip the tenant-isolation depth, run the
B2C-relevant checks only.

---

## Investigate

### Tenant isolation (the critical category)

This is where serious SaaS breaches happen.

- **Row-level enforcement on every query.** For every API/handler that reads
  tenant-scoped data, is `tenant_id = current.tenant.id` enforced
  *server-side*? Client-side filtering is never enough.
  - Best-case patterns: Postgres RLS (Row-Level Security) policies; an ORM
    middleware that auto-injects the tenant clause; a dedicated
    data-access layer that takes the tenant context as required input.
  - Worst-case: ad-hoc `where: { tenantId }` sprinkled in handlers — easy
    to forget on new endpoints. Flag as systemic risk even if all current
    handlers are correct.
- **IDOR (Insecure Direct Object Reference) checks.** Pick 5 random
  resource-fetch handlers and verify they cross-check ownership:
  ```
  // GOOD: ownership filter
  const post = await db.post.findFirst({
    where: { id, tenantId: ctx.tenant.id }
  });

  // BAD: globally addressable by id
  const post = await db.post.findUnique({ where: { id } });
  ```
- **Admin / impersonation paths.** Internal admin tooling that can read any
  tenant's data — is access logged? Audit log tied to a real human?
- **Tenant deletion.** When a tenant cancels, what happens to their data?
  Hard delete, soft delete with retention, archive? Document the policy or
  flag absence.
- **Tenant export.** GDPR-required for EU tenants — verify a path exists.

### Authentication & SSO

- **Org-scoped sessions.** A user signed into Org A should not have implicit
  access to Org B (even if the user has both); switching orgs is an explicit
  action.
- **SAML SSO** for enterprise tenants — `passport-saml`, WorkOS, Auth0,
  Clerk, Better Auth. Flag if the product targets enterprise but has no SSO.
- **SCIM** for user provisioning — same enterprise tier.
- **Domain-bound auto-join** ("anyone with @acme.com auto-joins Acme org") —
  fine if intentional, dangerous if accidental for fragmented domains
  (gmail.com, etc.).
- **Service accounts / API keys.** Per-org, per-user, both?
- **MFA / 2FA.** Optional or enforceable by org admins?

### Authorization (RBAC)

- **Role primitives.** Owner / Admin / Member is the typical floor; some
  products need fine-grained permissions per resource.
- **Permission checks consistent across surfaces.** Admin UI hides a button
  but the backend doesn't enforce — frontend-only authorization is a common
  fail.
- **Audit log.** Every admin action recorded with actor, timestamp, target,
  before/after values. Required for SOC 2.

### Billing & subscription

- **Plan enforcement server-side.** Free-tier limits enforced by code, not
  just UI. Search for `if (plan === 'pro')` in handlers vs only in UI
  components.
- **Seat enforcement.** Seat-based plans — adding a member without paid
  seats triggers either an upgrade prompt, a hard block, or a usage alert;
  not silent overage.
- **Usage-based billing accuracy.** If usage-billed: the metering pipeline
  has reconciliation, idempotent emit, and replay capability. (Common
  source of revenue leak.)
- **Failed payment handling.** Dunning flow, suspension, downgrade,
  not silent app failure.
- **Refunds / cancellation.** Self-service cancel; refund pathway documented.

### Onboarding

- **Org creation vs user signup.** First user creates an org; subsequent
  users are invited. Self-serve org creation with subdomain or path-scoped
  URL.
- **Invitation flow.** Email-verified invites, expiry, revocability.
- **Onboarding checklist.** Empty-state guidance — connecting integrations,
  inviting team, completing profile.
- **Sample data / templates** for first-run.

### Compliance & enterprise readiness

- **SOC 2.** Most B2B SaaS targets SOC 2 Type 2 by some scale. Audit log,
  least-privilege access, vendor management — flag gaps if the product is
  pitching enterprise.
- **GDPR.** DPA available; sub-processor list maintained; data-export and
  deletion paths.
- **HIPAA / PCI / FedRAMP** — only if product domain demands it. Flag the
  obligation.
- **Data residency.** EU customers may demand EU-region data; multi-region
  story present?
- **Trust page** at `/trust` or `/security` — uptime status link, security
  practices summary, compliance badges.
- **Privacy policy + ToS + DPA** linked from app and marketing site.

### Operational concerns specific to SaaS

- **Tenant-isolated rate limits** — abuse from one tenant doesn't affect
  others.
- **Per-tenant feature flags** — beta features rolled out per tenant.
- **Per-tenant observability.** Logs/metrics tagged with `tenant_id` for
  triage. (Mind PII spillage in logs.)
- **Backup & restore at the tenant level.** Per-tenant point-in-time
  restore is much harder than full-DB restore — flag if required by
  contracts.

---

## Red flags

- Resource fetches by ID without tenant scope check.
- Admin actions without audit log.
- Plan limits enforced only in UI.
- No SSO / SCIM in a product pitched at enterprise.
- Frontend-only RBAC (button hidden, endpoint open).
- No tenant export path (GDPR exposure).
- Logs containing PII without redaction.
- Single shared rate limit across all tenants.
- Tenant deletion not documented.

---

## Output section

```markdown
### SaaS Multi-Tenant

#### Tenant isolation
- Enforcement pattern: <RLS / ORM middleware / DAL / ad-hoc>
- Sampled IDOR checks: <pass/fail with evidence>
- Admin / impersonation logged: yes/no
- Tenant deletion documented: yes/no
- Tenant export available: yes/no

#### Authentication
- Org-scoped sessions
- SSO (SAML / OIDC / WorkOS): present? required?
- SCIM: present?
- MFA enforceable by org admin: yes/no

#### Authorization
- Roles defined
- Server-side enforcement on every gated endpoint
- Audit log: present? completeness?

#### Billing & subscription
- Plan enforcement server-side: yes/no
- Seat enforcement
- Usage metering accuracy (if applicable)
- Failed payment / dunning flow
- Self-serve cancel

#### Onboarding
- Org creation flow
- Invitation flow (verification, expiry, revoke)
- Onboarding state / checklist

#### Compliance
- SOC 2 readiness: <state>
- GDPR DPA / sub-processor list / export+delete
- Sector compliance (HIPAA/PCI/etc.) if relevant
- Data residency story
- Trust / security page

#### Operations
- Tenant-isolated rate limits
- Per-tenant feature flags
- Tenant-scoped observability
- Tenant-level backup/restore (if required)

#### Findings
(table — severity / area / evidence / fix)
```
