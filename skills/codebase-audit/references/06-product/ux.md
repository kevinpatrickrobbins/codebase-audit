# 15 — UX Audit

Perform a UX and interface audit of the application.

Create or update:

`/docs/audits/ux-audit.md`

---

## How to investigate

UX is harder to audit from code than security or performance — much of UX lives in
how the app *feels* in the browser, not in source files. Be honest about what code
review can and cannot tell you, and point out where a real-device walkthrough is
needed.

This module covers heuristic UX. Detailed WCAG/ADA/EAA compliance lives in
`references/02-compliance/accessibility.md` — when the work overlaps, do the
high-level heuristic check here and defer the legal-rigor check to Module 04.

### Step 1: Build a route map

Before evaluating UX, list the routes. The user journey lives in routes, not
components.

- Next.js App Router: walk `app/` directory; each `page.tsx` is a route.
- Next.js Pages: walk `pages/`.
- Remix / SvelteKit / Solid Start: `routes/` directory.
- Express/Hono/Fastify: grep for `app.get`, `app.post`, `router.get`, etc.
- Django: `urls.py`.

Group routes by user role / journey:
- Public (marketing, auth)
- First-run (onboarding, signup)
- Core product (the main jobs-to-be-done)
- Account & settings
- Admin

This grouping is the skeleton of every later UX finding.

### Step 2: Onboarding & first-run

- Find the signup flow. Walk it in code: signup form → email verify? → onboarding
  steps? → first product use.
- How many steps before the user reaches the core product value?
- Is there a tour, empty-state guidance, or sample data on first run? Search for
  components named `Welcome`, `Onboarding`, `EmptyState`, `GettingStarted`.
- Email verification: required before any use, or deferred? Both are valid; flag
  if the choice seems accidental.

### Step 3: Navigation & information architecture

- Find the primary nav component (`Nav`, `Sidebar`, `Header`, `MainNav`,
  `AppShell`). How many items? Is it flat or nested? What is hidden behind a "More"?
- Is there a search? Does it return results from across the app, or just one
  resource type?
- Are deep-linked URLs human-readable (`/projects/acme-redesign`) or opaque
  (`/p/8a3f9b2c`)?
- Breadcrumbs on nested pages?

### Step 4: Forms

Forms are where the most fixable UX issues live.

For each significant form (signup, login, payment, settings, item creation):
- Are labels visible (not just placeholders)?
- Are required fields marked visually?
- Is validation inline (on blur or on submit) or only after submission?
- Are error messages specific ("Password must be 8+ characters") or generic
  ("Invalid input")?
- Loading state during submit (button disabled + spinner)?
- Success state (toast? redirect? inline confirmation?)?
- Autofill-friendly: `autocomplete="email"`, `autocomplete="current-password"`,
  etc.? (Also a WCAG 1.3.5 requirement — flag for Module 04.)

### Step 5: Empty, loading, and error states

For each list/table view, check for:
- **Loading state.** Skeleton, spinner, or nothing? "Nothing" is jarring.
- **Empty state.** First-time empty (helpful: "Create your first project") vs
  filtered-empty ("No results for 'foo'") vs error-empty. Distinct messages?
- **Error state.** Generic "Something went wrong" vs specific ("Connection lost,
  retrying…").

For each form/mutation:
- Optimistic update or wait-and-show? (Optimistic is often nicer; rollback on
  error is the hard part.)

For routes:
- 404 page exists? Custom or framework default? Has a useful link out?
- 500 page exists?

### Step 6: Micro-interactions and feedback

- Toasts/snackbars for confirmations: `react-hot-toast`, `sonner`, custom.
- Confirmation modals for destructive actions (delete, archive, leave team)?
  Search for `AlertDialog`, `ConfirmDialog`, `confirm(`.
- Optimistic UI vs blocking spinners.
- Undo for destructive actions (gold standard for delete).

### Step 7: Visual & component consistency

- Is there a design system? (Look for `components/ui/`, Radix primitives,
  shadcn/ui, MUI, Chakra, Mantine, Ant Design, Tailwind UI patterns.)
- How many "Button" components exist? More than one suggests inconsistency.
- Are spacing and typography tokens centralized (Tailwind config, theme tokens,
  CSS variables) or ad hoc (`style={{ marginTop: 13 }}`)?
- Color: a theme palette, or inline hex codes everywhere?

### Step 8: Mobile and responsive

- Is the app responsive? Check breakpoints in CSS / Tailwind config.
- Any explicit "mobile" route group or component variants?
- Touch targets: minimum 44×44 px for primary actions on mobile (overlap with
  Module 04).
- Horizontal scroll on small screens? (Overlap with Module 04.)

### Step 9: Accessibility — heuristic only

Note the high-level state of:
- Visible focus rings (or `outline: none` everywhere — fail).
- Semantic HTML (or `<div onClick>` everywhere — fail).
- Alt text on images.

Defer detailed WCAG analysis to Module 04 — flag the bucket here, audit the
specifics there.

### Step 10: Copy & content

- Microcopy: button labels ("Submit" vs "Create project"), error messages, empty
  states. Boring/generic vs specific/branded.
- Inconsistent terminology: is the same thing called "project" in some places
  and "workspace" in others?
- Tone: formal vs casual — consistent across surfaces?

---

## Red flags

- Onboarding requires > 5 steps before the user sees product value.
- Generic error messages ("An error occurred") with no recovery path.
- `<div onClick>` used in place of `<button>` (accessibility AND UX issue).
- More than one Button component in the design system.
- Inline colors (`#3B82F6`) scattered through code rather than tokens.
- No 404 / no 500 page.
- Destructive actions without confirmation or undo.
- Forms without inline validation, only submit-and-pray.
- Modals that trap users (no escape, no cancel button).

---

## Output structure

```markdown
# User Experience Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Route Map
(brief, grouped by journey)

## Navigation and Information Architecture
- Clarity of navigation
- Logical grouping of features
- Discoverability of key actions

## User Flow Analysis
For each major flow (signup, primary task, settings change):
- Steps observed (cite components)
- Friction points
- Error/empty/loading state coverage

## Interface Design
- Visual hierarchy
- Component consistency (cite examples)
- Form usability
- Loading states
- Empty states
- Error messaging

## Accessibility — Heuristic
High-level state. Defer detailed audit to `accessibility-audit.md` (Module 04).

## Mobile and Responsive Design
Cite breakpoints, mobile variants, known issues.

## Copy & Content
Tone, terminology consistency, microcopy quality.

## UX Improvement Opportunities
Prioritized list. Each: severity (Critical / High / Medium / Low), effort
(S/M/L), evidence pointer.

## Final Recommendation
The single highest-impact UX action for this codebase right now.
**One thing.**

## Decision Summary

- Primary flows usable end-to-end: yes / no / partial
- Worst friction point: <description>
- Recommended posture: <keep | quick wins | bigger flow rework>

## Confidence
High / Medium / Low (and why)

## Out of Scope / Inconclusive
Things that need a real-device or real-user walkthrough — animations,
perceived speed, etc.
```

---

## Severity calibration

- **Critical:** users cannot complete a primary task (broken signup, broken
  checkout, broken core product flow).
- **High:** primary task is technically possible but high abandonment risk
  (confusing forms, missing error recovery, hostile defaults).
- **Medium:** annoyances on common paths (inconsistent components, missing empty
  states, awkward mobile layout).
- **Low:** polish issues that an experienced designer would notice but most users
  would not.

---

## Evidence requirements

Every finding cites a file path and (when relevant) the component or route name.
For findings that depend on real-device inspection (e.g., touch target feel,
animation jank), call them out in "Out of Scope / Inconclusive" rather than
guessing.
