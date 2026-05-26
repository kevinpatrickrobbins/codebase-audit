# 04 — Accessibility & Compliance Audit

Evaluate the application against accessibility standards (WCAG 2.1/2.2 AA) and the
legal frameworks that reference them: ADA (US), AODA (Ontario), Section 508 (US
federal), EN 301 549 / EU EAA (European Union, enforceable since 28 June 2025).

This is the most legally-exposed of the audits. A failing accessibility posture is
not just a UX problem — it carries real fines, lawsuits, and procurement
disqualification risk in regulated jurisdictions.

Create or update:

`/docs/audits/accessibility-audit.md`

---

## When to apply

Run this module on any project that ships a user-facing UI — web app, web embed,
mobile web, PDF documents, or installable desktop/mobile clients.

If the project is purely a backend API or CLI with no human UI, note that in the
report and skip the manual checks. Still capture compliance posture for any
admin/dashboard surface, however small.

---

## Step 0: Live Discovery

Accessibility standards and enforcement evolve. Fetch current truth before
auditing.

### Current WCAG version

Fetch https://www.w3.org/TR/WCAG/ — returns the current published
recommendation. Note the version in your report.

Fetch https://www.w3.org/WAI/standards-guidelines/wcag/ — overview
including any version-in-progress (e.g., WCAG 3.0 working drafts).

If WCAG 3.0 has reached W3C Recommendation status, audit against it; the
posture table in Step 1 needs updating.

### Jurisdictional enforcement state

- **EU EAA / EN 301 549.** Fetch
  https://ec.europa.eu/social/main.jsp?catId=1202 — current enforcement
  state, member-state implementation status. The directive is in force from
  28 June 2025; per-country enforcement bodies and fine schedules may
  evolve.
- **ADA Title III (US).** Search "ADA Title III current DOJ rule
  WCAG" — DOJ issued a final rule April 2024 binding WCAG 2.1 AA for state
  and local government; private accommodation rules continue to evolve via
  litigation.
- **AODA (Ontario).** Fetch https://www.ontario.ca/laws/regulation/110191
  — current standard (currently WCAG 2.0 AA; potential update pending).
- **Section 508 (US federal).** Fetch https://www.section508.gov/ — current
  Refresh status (2017 Refresh references WCAG 2.0 AA; updates anticipated).

If the project serves a jurisdiction not in the above, Search
"<jurisdiction> accessibility law digital products" for current state.

### Required Reasons APIs (iOS) and platform-specific accessibility

If product-type detection (Module 19) flagged iOS / Android:

- Fetch https://developer.apple.com/documentation/bundleresources/privacy_manifest_files —
  current Required Reasons API list (Apple expands this list periodically).
- Fetch https://support.google.com/accessibility/android — current
  Android accessibility expectations.

### Output of discovery

Emit a short preamble:

```markdown
## Live Discovery (fetched YYYY-MM-DD)

| Source | Discovered |
|---|---|
| W3C WCAG (w3.org/TR/WCAG) | Current Recommendation: WCAG 2.2 (Oct 2023) |
| EU EAA enforcement (ec.europa.eu) | In force since 28 June 2025 |
| ADA Title III (DOJ) | April 2024 final rule for state/local gov; private context evolving |
| AODA (ontario.ca) | WCAG 2.0 AA (current, update pending) |
| Apple Required Reasons APIs | <list of currently-required APIs> |

Audit calibrated against this state.
```

If discovery fails: note in "Out of Scope / Inconclusive" and use the embedded
knowledge below with a "may be stale" caveat.

---

## Step 1: Identify the Compliance Surface and Jurisdictions

Before checking anything, determine *what* needs to comply and *under which laws*.

**Investigate:**

- Read README, marketing copy, and any `target-markets`-style config to identify the
  geographic markets the product serves.
- Check `package.json` `description`, `LICENSE` ownership, and any `Privacy Policy`
  page for the controlling entity's jurisdiction.
- Check robots/locale: are there `<link rel="alternate" hreflang="...">` tags? An
  i18n config (`next-intl`, `i18next`, etc.)? Multiple language files in
  `locales/`, `messages/`, or similar? These signal multi-jurisdiction reach.
- Identify B2B vs consumer-facing — EAA applies primarily to consumer-facing
  digital services in the EU. ADA Title III covers public accommodations.
- Search for `accessibility`, `wcag`, `vpat`, `acr` in the repo — any prior
  compliance statements, VPATs (Voluntary Product Accessibility Templates), or ACRs?

**Output a Compliance Posture Summary table:**

| Standard | Effective | Required Level | Likely In Scope? | Notes |
|---|---|---|---|---|
| WCAG 2.1 AA | 2018 | Industry baseline | Yes | International default |
| WCAG 2.2 AA | Oct 2023 | New SCs | Yes if recent claim | Adds 2.4.11 focus appearance, 2.5.7 dragging, 2.5.8 target size, 3.2.6 consistent help, 3.3.7 redundant entry, 3.3.8 accessible auth |
| ADA Title III (US) | DOJ rule April 2024 | WCAG 2.1 AA | If serving US public | Public accommodations; lawsuits common |
| Section 508 (US fed) | Refresh 2017 | WCAG 2.0 AA | If selling to US gov | Procurement gate |
| AODA (Ontario, CA) | Phased; 2021 deadline for orgs ≥50 | WCAG 2.0 AA (Level A and AA, ex 1.2.4 & 1.2.5) | If serving Ontario | Up to $100k/day fines for orgs |
| EN 301 549 / EU EAA | 28 June 2025 | WCAG 2.1 AA + EN 301 549 supplement | If serving EU consumers | New as of 2025; private-sector consumer products |

For each "Likely In Scope? Yes" row, note the concrete enforcement mechanism (lawsuit
risk, regulator fine, procurement disqualification). This frames the rest of the
audit as legal-risk-bearing, not optional polish.

---

## Step 2: Automated Tooling Check

Automated tools catch ~30–40% of accessibility issues. They are necessary but not
sufficient — a clean Lighthouse score does not mean compliant.

**Investigate:**

- Search `package.json` and CI workflows for: `axe-core`, `@axe-core/playwright`,
  `pa11y`, `jest-axe`, `eslint-plugin-jsx-a11y`, `lighthouse`, `playwright`'s
  accessibility tree, `react-axe`.
- Check whether automated a11y is wired into CI — `.github/workflows/*.yml` running
  Lighthouse CI or axe-core in PR checks.
- **Check the gate, not just the presence.** Does the CI step actually *fail* the
  build on new violations, or run-and-report without blocking? An axe-core step
  that emits a JSON report but exits 0 catches nothing. Find the threshold or
  assertion (e.g. `lhci autorun --assertions` config, axe `expect` assertions in
  Playwright tests, `pa11y --threshold`). Advisory-only is a finding.
- If `eslint-plugin-jsx-a11y` is present, check whether rules are at `error` or
  `warn` (warn is often ignored in practice). Ideally the lint job blocks merges.

**Red flags:**

- Zero automated a11y tooling in a project with consumer-facing UI.
- `jsx-a11y` rules disabled or set to `warn` with hundreds of warnings.
- Lighthouse a11y score is checked in CI but threshold is < 90.

**Note in report:** which automated tools are present, what they cover, what gaps
exist. If none are present, recommend axe-core in CI as the lowest-friction starting
point.

---

## Step 3: Manual Audit by WCAG Principle

Walk the codebase against each WCAG principle. For each finding, record: **WCAG SC
(success criterion) | Level | Issue | Location (file:line) | User impact | Fix**.

### 3.1 Perceivable

**Investigate:**

- **Text alternatives (1.1.1).** Grep for `<img` (or framework equivalent like
  `<Image`). For each, check `alt`. Missing or empty `alt=""` on informative images
  is a fail. Decorative images need explicit `alt=""` (empty), not omission.
- **Time-based media (1.2.x).** Search for `<video`, `<audio`, embeds. Check for
  captions (`<track kind="captions">`), transcripts, or descriptive alternatives.
- **Adaptable (1.3.x).** Check semantic HTML — search for divs that should be
  buttons/links/headings. Grep for `onClick=` on `<div>` (anti-pattern). Check
  heading hierarchy: do pages start at `h1`? No skipped levels?
- **Tables (1.3.1, 1.3.2).** Search for `<table>`. For each:
  - Is it a *data* table or a *layout* table? Layout tables fail (use CSS grid
    or flex instead).
  - Does it have a `<caption>` describing its purpose?
  - Are header cells `<th>` with a `scope` attribute (`row`, `col`)?
  - For complex tables (merged cells, multi-level headers), are `headers`/`id`
    associations in place?
  - Lists rendered as styled `<div>` stacks instead of `<ul>`/`<ol>`/`<li>` are
    a common related failure — flag them here too.
- **Distinguishable (1.4.x).**
  - **Color contrast (1.4.3, 1.4.11).** Open the design tokens (CSS variables,
    theme files, `tailwind.config.{js,ts}`, `theme.ts`). Compute contrast ratios
    for foreground/background pairs. Required: 4.5:1 normal text, 3:1 large text
    (≥18pt or 14pt bold), 3:1 non-text UI. Flag any combo below threshold.
  - **Color-only signaling (1.4.1).** Grep for "red"/"green" CSS classes used as
    the *only* signal of state (e.g. error vs success indicated only by color).
    Look for icon-less status indicators.
  - **Resize (1.4.4).** Check whether base font sizes are in `rem`/`em` (good) or
    `px` (often blocks browser zoom). Check for `user-scalable=no` in viewport
    meta tag (a fail).
  - **Reflow (1.4.10).** Check responsive layout at 320 CSS pixels — note any
    horizontal scroll on small screens.

### 3.2 Operable

**Investigate:**

- **Keyboard (2.1.x).** Grep for custom interactive components: `onClick` without
  `onKeyDown`, `role="button"` without `tabindex`. Drag-and-drop without
  keyboard alternative (2.5.7 in WCAG 2.2).
- **Modal focus management (2.4.3, 3.2.1).** For every modal/dialog component
  (`<Dialog>`, `<Modal>`, Radix `Dialog`, custom): does focus move *into* the
  modal on open? Is focus *trapped* inside while open? Does it *return* to the
  triggering element on close? Search for state hooks managing focus refs or
  use of headless libraries (Radix, Reach, Headless UI) that handle this for
  you. Hand-rolled modals without explicit focus management almost always fail
  at least one of the three.
- **Icon-only controls.** Grep for `<button>` whose only child is an `<svg>`,
  `<Icon>`, or emoji. Each must have an `aria-label`, visually hidden text
  child, or `aria-labelledby`. Icon buttons without an accessible name are a
  4.1.2 fail.
- **Enough time (2.2.x).** Search for session timeouts, auto-logout, auto-refresh.
  Check for warnings/extensions per 2.2.1.
- **Seizures (2.3.1).** Search for animations: `@keyframes`, `framer-motion`,
  `gsap`, video autoplay. Anything flashing >3 times/second is a fail.
- **Navigable (2.4.x).**
  - **Skip links (2.4.1).** Search for `<a href="#main">Skip to content</a>`,
    `<SkipNav>`, `SkipLink`, or framework equivalent. Should render before the
    `<header>` and become visible on focus. Missing skip nav on a page with
    substantial repeated navigation is a Level A failure.
  - **Page titles (2.4.2).** Grep `<title>`, Next.js `metadata.title`,
    `useDocumentTitle`, framework-specific `<Head>` / `<svelte:head>`. Each
    route must have a *unique, descriptive, non-empty* title. Empty titles,
    identical titles across routes, or titles set only via client-side JS
    without an SSR fallback are failures.
  - **Focus order (2.4.3).** `tabindex` values > 0 (anti-pattern — breaks DOM
    order). Visible visual order should match DOM tab order.
  - **Visible focus (2.4.7, 2.4.11).** Grep for `outline: none` / `outline: 0`
    — almost always wrong unless paired with a custom `:focus-visible` style.
    WCAG 2.2 SC 2.4.11 (focus appearance) requires a minimum 2px solid outline
    with at least 3:1 contrast against adjacent colors.
- **Input modalities (2.5.x).** Touch target size (2.5.5/2.5.8): minimum 24×24
  CSS px (WCAG 2.2 AA), recommended 44×44. Check button/link sizes in mobile
  layouts.

### 3.3 Understandable

**Investigate:**

- **Readable (3.1.x).** Check `<html lang="...">` is set on every page. For multi-
  language content, check `lang` attribute on the language-switched portion.
- **Predictable (3.2.x).** Check that focus or input changes do not cause
  unexpected context changes (auto-submit on focus, auto-redirect on input).
- **Input Assistance (3.3.x).** Forms — every input has a `<label>` (not just
  placeholder); error messages are programmatically associated via
  `aria-describedby`; required fields marked via `aria-required` AND visually;
  errors identifiable by more than color.
- **Error summary pattern (3.3.1).** On submit failure for forms with multiple
  fields, is there an *error summary* at the top of the form listing all errors
  with anchor links to each invalid field? Is focus moved to the summary on
  submit failure (so screen reader users hear the error count, not silence)?
  Per-field error messages alone are insufficient on long forms — users cannot
  find what failed without scrolling.
- **Accessible authentication (3.3.8 — WCAG 2.2 AA).** Cognitive function tests
  (CAPTCHA, memorize-this-token) need an alternative — passkeys, OAuth, password
  managers, or copy-paste enabled.

### 3.4 Robust

**Investigate:**

- **Parsing (4.1.1 — removed in WCAG 2.2 but still relevant).** Run an HTML
  validator mentally: duplicate IDs, unclosed tags, unescaped attributes.
- **Name, Role, Value (4.1.2).** Custom components — does every interactive
  element expose its accessible name (via `aria-label`, `aria-labelledby`, or
  child text)? Role appropriate? Value/state exposed via `aria-pressed`,
  `aria-expanded`, etc.?
- **ARIA Authoring Practices Guide (APG) conformance.** For each custom widget
  — tabs, accordion, combobox, dialog, disclosure, listbox, menu, menubar,
  radio group, slider, spinbutton, switch, treeview, toolbar, etc. — verify
  conformance to the W3C APG pattern for that widget
  (https://www.w3.org/WAI/ARIA/apg/patterns/). Specifically: required keyboard
  interactions (e.g. arrow keys for tab list, Esc to close menu), required
  ARIA states/properties, required focus model. Custom widgets that "look like"
  a native control but fail APG keyboard or state requirements are one of the
  most common categories of audit findings.
- **Status messages (4.1.3).** Toast/snackbar/error notifications — is the
  notification region marked `aria-live="polite"` (or `assertive`)? If updates
  fire without `aria-live`, screen readers miss them.

---

## Step 4: Forms — Deep Dive

Forms are the most common place for serious a11y failures. Audit every form:

| Field | Has `<label>`? | Required visible & programmatic? | Error associated via `aria-describedby`? | Error text not color-only? | Autocomplete attribute (1.3.5)? |

For payment, signup, login, password reset, account deletion forms specifically:
add a column for `autocomplete` attributes — these are required by WCAG 1.3.5
(Identify Input Purpose) and dramatically improve password manager and autofill
support. Common values to verify, by field type:

- **Auth:** `username`, `current-password`, `new-password`, `one-time-code`
- **Identity:** `name`, `given-name`, `family-name`, `email`, `tel`,
  `tel-national`, `bday`
- **Address:** `street-address`, `address-line1`, `address-line2`,
  `address-level1` (state/province), `address-level2` (city), `postal-code`,
  `country`, `country-name`
- **Payment:** `cc-name`, `cc-number`, `cc-exp`, `cc-exp-month`, `cc-exp-year`,
  `cc-csc`

Note: `autocomplete="off"` is generally ignored by modern browsers — to suppress
autofill of an existing password on a *create new password* field, use
`autocomplete="new-password"` instead.

---

## Step 5: Mobile, Responsive, and Native

If the project ships mobile web or a React Native / Flutter / Swift / Kotlin app:

- Touch targets ≥ 44×44 (WCAG 2.5.5 AAA, 2.5.8 AA in 2.2).
- Pinch-zoom not disabled.
- Native: VoiceOver / TalkBack labels on every interactive element.
- Reduced motion (prefers-reduced-motion / OS setting) respected.
- Dynamic Type / system font scaling supported.

---

## Step 6: Documents and Media

If the app generates or serves PDFs, presentations, or videos:

- PDFs: tagged structure, alt text on images, reading order, language declared.
- Videos: captions (open or closed), transcripts, audio description for
  significant visual-only content (1.2.5 AA).
- Charts/data viz: text alternatives — not just `alt="chart"` but the data the
  chart conveys.

---

## Step 7: Compliance Posture & Legal Risk

Combine the findings into a posture summary.

### Public-facing posture artifacts

Beyond the in-product compliance checks, audit whether the product **publicly
demonstrates** its accessibility posture — a growing expectation for procurement
and an EU EAA conformity-assessment input.

- **Accessibility statement page.** Does the product publish a statement
  (commonly at `/accessibility`, linked from the footer)? Check footer
  components, sitemap, and `app/accessibility`, `pages/accessibility`,
  `routes/accessibility`. The statement should declare conformance level,
  known limitations, contact for accessibility feedback, date last reviewed,
  and (for EU products) the relevant national enforcement body. Required by
  EAA for EU consumer-facing products; expected for AODA-covered organizations
  and US federal procurement.
- **VPAT / Accessibility Conformance Report (ACR).** For products selling to
  enterprise or government, a VPAT (Voluntary Product Accessibility Template)
  is almost always requested in procurement. Search for `VPAT`, `ACR`,
  `conformance-report`, `accessibility-conformance` in the repo and `/public/`.
  "No VPAT" is a procurement risk, not a WCAG failure — flag separately from
  technical findings.
- **Feedback contact.** Is there a documented channel (email, form) for users
  to report accessibility issues? AODA, EAA, and Section 508 all expect this.

**Output sections:**

```markdown
## Compliance Posture

| Standard | Status | Confidence | Blocking Issues |
|---|---|---|---|
| WCAG 2.1 AA | Partial | High | <count> A-level failures, <count> AA failures |
| WCAG 2.2 AA | Failing | High | New 2.2 SCs not addressed: <list> |
| EU EAA (28 June 2025) | At risk | Medium | Depends on EU-consumer scope |
| ADA Title III (US) | At risk if serving US public | Medium | Form labels, contrast |
| AODA (if Ontario) | ... | ... | ... |
| Section 508 (if US gov) | ... | ... | ... |

## Top Blocking Issues

(5–10 highest-impact failures, with WCAG SC, location, fix.)

## Quick Wins

(Issues fixable in <1 day with high impact: alt text, lang attribute, focus
visible, form labels.)

## Long-term Roadmap

(Issues requiring design or architectural change: keyboard-trap-free modals,
form rebuild, color system overhaul.)

## Recommended Tooling

(Concrete CI integrations: axe-core in Playwright, Lighthouse CI thresholds,
eslint-plugin-jsx-a11y at error.)
```

---

## Step 8: Operational Accessibility

Beyond audit-time findings, accessibility posture depends on whether the team
can detect and prevent regressions. This section captures the operational shape
of the product's a11y practice.

- **CI gating.** Does the build *fail* on new accessibility regressions? See
  Step 2 — confirm the assertion is binding, not advisory. Advisory-only CI
  rots: violations accumulate and the report becomes background noise.
- **Pre-merge a11y review.** Is accessibility a checklist item in the PR
  template (`.github/pull_request_template.md`)? In larger orgs, PRs touching
  user-facing UI route through a designated reviewer.
- **Scheduled production scans.** For products with frequent content updates
  (CMS, builder platforms, marketing sites with editable pages), is there a
  scheduled crawl that scans deployed pages with axe-core / Pa11y / Lighthouse
  and surfaces regressions? Search `.github/workflows/` for `cron:` schedules
  running a11y tools, or look for external monitors (BetterStack a11y,
  AccessLint, Deque axe Monitor) referenced in README/runbooks.
- **Component-library a11y baseline.** If the product uses a design system, is
  the component library itself audited and documented for accessibility? Check
  Storybook for `@storybook/addon-a11y`. A non-accessible primitive (`Button`,
  `Input`, `Modal`) makes every page that uses it non-accessible; auditing the
  primitives once is far higher leverage than auditing each page.
- **User feedback loop.** Where do reports of accessibility issues land? Is
  there an email or form (per Step 7 — feedback contact)? Are reports triaged
  into the same backlog as other bugs, or do they vanish?
- **Training & ownership.** Is there a designated a11y owner? Are designers and
  engineers given any onboarding on accessible patterns? This is hard to
  evaluate from code alone — note in "Out of Scope / Inconclusive" if unclear.

Output a brief Operational Accessibility section in the report citing the
specific evidence found (workflow files, PR template, Storybook config, etc.)
or noting that nothing was found.

---

## Output structure

```markdown
# Accessibility & Compliance Audit

Date: YYYY-MM-DD
Repository commit: <hash>

## Compliance Surface
(jurisdictions, standards in scope)

## Compliance Posture Summary
(table)

## Public-Facing Posture
- Accessibility statement page presence and completeness
- VPAT / ACR status
- Feedback contact channel

## Automated Tooling Status
(what is in place, what gaps exist)

## Findings by WCAG Principle
### Perceivable
### Operable
### Understandable
### Robust

## Forms Deep-Dive
(table)

## Mobile / Native
(if applicable)

## Documents & Media
(if applicable)

## Operational Accessibility
- CI gating status (advisory vs blocking)
- Scheduled production scans
- Component-library a11y baseline
- Feedback / triage loop
- Designated owner / training

## Top Blocking Issues
## Quick Wins
## Long-term Roadmap
## Recommended Tooling

## Final Recommendation
The single highest-impact accessibility action for this codebase right
now. **One thing.**

## Decision Summary

- WCAG 2.2 AA reachable from current state: yes / no / partial
- Worst category: <perceivable | operable | understandable | robust>
- Single biggest blocker: <description>
- Recommended posture: <ship-blockers fixed | Tier 1 quick wins | deeper review needed>

## Confidence
High / Medium / Low (and why)

## Out of Scope / Inconclusive
(things you could not check from code alone — e.g. screen-reader actual behavior,
real-device touch target verification)
```

---

## Evidence requirements

Every finding must cite a file path and (where relevant) a line number. Vague
claims like "the forms are not accessible" are useless. Concrete claims like
"`app/components/SignupForm.tsx:42` — email input lacks `<label>` association,
relies on placeholder only (WCAG 1.3.1, 3.3.2, Level A)" are actionable and become
clean ticket bodies in Module 00.3.

If a check is genuinely impossible from code alone (e.g. testing real screen
reader output), call that out in the "Out of Scope / Inconclusive" section rather
than guessing or skipping silently.
