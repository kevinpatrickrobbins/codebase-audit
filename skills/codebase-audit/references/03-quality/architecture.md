# 05 — Architecture Integrity

Evaluate the structural architecture of the codebase.

Create or update:

`/docs/audits/architecture-audit.md`

---

## How to investigate

This is the most subjective audit. The trap: critiquing a codebase against an
imaginary "ideal architecture" that may not actually fit the project. A monolithic
Rails app can be a perfectly fine architecture for the team and product; layering
"hexagonal architecture" onto a 5kLOC side project is malpractice.

The right standard: **does the architecture support the kind of changes this team
needs to make?** Not "is it pretty" but "is it changeable safely."

### Step 1: Map the major components

Walk the directory tree and produce a one-line description per major module:

```
src/
├── app/         — Next.js App Router routes (pages + server actions)
├── components/  — React UI components
├── lib/         — Shared utilities + data access
├── server/      — Server-only code (DB, auth, queue)
├── workers/     — Background job handlers
├── ai/          — LLM prompt management
```

Note any non-obvious roots (e.g., `packages/` for monorepo, `apps/` for Turborepo,
`shared/` for code shared between client and server).

### Step 2: Identify implicit layers

Even codebases without explicit "layers" usually have implicit ones. Identify:

- **Presentation:** UI components, routes, pages
- **Application logic:** services, use-cases, controllers
- **Data access:** DB queries, ORM models, external API clients
- **Domain:** entity types, business rules
- **Infrastructure:** auth, queues, file storage clients

For each, ask: where is this *supposed* to live? And where does it actually live?

### Step 3: Coupling check

The classic indicators of bad coupling:

- **UI components calling DB directly.** Search components for ORM imports
  (`prisma`, `drizzle`, `@supabase/supabase-js` client). If a `<Button>`
  component imports `prisma`, that is a layering violation.
- **Domain logic in controllers/handlers.** Long route handler files (> 200 LOC)
  doing fetching + business logic + response shaping all together.
- **Cross-feature imports.** Feature A reaching into Feature B's internals
  (importing private helpers, not just public exports).
- **God modules.** A single file imported from 20+ places. `lib/utils.ts` /
  `helpers.ts` that has accumulated everything.
- **Circular imports.** A imports B imports A. Often indicates the wrong file
  carved the boundary.

How to find them:
- For each component: grep its imports — count direct DB / external service
  imports.
- For each handler: count LOC; > 100 is a flag, > 200 is a finding.
- Build a quick reverse-import map for the top 5 most-imported "utility" files —
  they are often god modules.

### Step 4: Module boundaries

- Are modules organized by **technical layer** (`controllers/`, `services/`,
  `repositories/`) or by **feature** (`features/billing/`, `features/auth/`)?
  Both are valid; mixing them inconsistently is a smell.
- Public vs private exports: are there `index.ts` barrel files that expose only
  the intended surface, or does everything `import { ... } from '../../private/file'`?
- Cross-feature dependencies: feature A → feature B is OK; mutual A ↔ B usually
  isn't.

### Step 5: Naming and conventions

- Casing: `camelCase`, `PascalCase`, `kebab-case` — consistently applied?
- File names: `UserProfile.tsx` vs `user-profile.tsx` vs `userProfile.tsx`. Pick
  one.
- Function names: are mutators named differently from queries (`getUser` vs
  `updateUser`)?
- Boolean variables: `isOpen`, `hasError` (clear) vs `open`, `error` (ambiguous).

### Step 6: Duplication

- Search for similar function bodies across the codebase. If `formatDate`,
  `formatTimestamp`, and `prettyDate` all exist and do similar things, that is
  drift.
- Multiple "Button" components in different folders.
- Repeated query shapes (the same `findMany({ where: ..., include: ... })`)
  in 5 places — usually a missing data access helper.

Use tooling thinking: an LSP "find references" on common names (`Button`,
`format`, `validate`) gives a quick duplication scan.

### Step 7: Error handling discipline

Beyond "is there try/catch?":

- **Try/catch density.** Too many: paranoid, swallows real errors. Too few:
  one thrown error nukes the request.
- **Errors swallowed.** `catch (e) { /* ignore */ }` or `catch (e) {
  console.log(e) }` with no rethrow / no monitoring is a recipe for silent
  failure.
- **Error classification.** Are errors categorized — *transient* (retry
  appropriate), *permanent* (don't retry), *business* (return to user as
  validation), *programmer* (alert)? Without classification, retry logic
  and alerting are uniform when they shouldn't be.
- **Custom error types.** Search for error class definitions (`class
  ValidationError extends Error`). Classes with `code: string` properties
  are useful; stringly-typed `throw new Error('not found')` parsed by
  downstream string-matching is a fragility magnet.
- **Circuit breakers.** For calls to external services, is there a circuit
  breaker (`opossum`, custom)? When upstream is failing, circuit-open
  prevents hammering and lets it recover. Absent on a project with
  multiple upstream dependencies = a finding.
- **Retry budgets with jitter.** Naive exponential backoff (`2^n` ms) on
  every caller causes thundering herd. Jitter (`exp + random(0, exp)`)
  and per-process budgets prevent.
- **Fallback strategies.** When upstream is down, what does the user see?
  Stale cache, degraded mode, error page? "Crash" is usually wrong for
  user-facing flows.
- **Boundary handling.** Errors caught once at the request boundary
  (middleware) and logged + sanitized > caught everywhere ad-hoc.
- **Unhandled rejections / uncaughtException.** Listeners installed in app
  entry to log + crash gracefully? Process supervisor (systemd, pm2, k8s)
  configured to restart?
- **Error messages to users.** Distinguish operator errors (logged with
  detail) from user errors (clear actionable message, no internal detail).

### Step 8: Type safety & domain modeling (TypeScript projects)

A type-safe codebase shapes itself. Look for the depth of type discipline,
not just whether `strict: true` is on.

#### tsconfig flags beyond `strict`

`strict: true` is table stakes. The flags often missed that catch real bugs:

- **`noUncheckedIndexedAccess`** — `arr[i]` becomes `T | undefined`, forcing
  out-of-bounds handling. Single biggest type-safety win not enabled by
  `strict`.
- **`exactOptionalPropertyTypes`** — distinguishes `{ x?: T }` (key may be
  absent) from `{ x: T | undefined }` (key present, value possibly undefined).
- **`noImplicitOverride`** — child class methods overriding parent must
  explicitly declare `override`.
- **`verbatimModuleSyntax`** — disambiguates type-only vs value imports;
  forces `import type` where appropriate.
- **`noPropertyAccessFromIndexSignature`** — forces `obj["key"]` for
  index-signature access vs `obj.key` for declared properties.
- **`noFallthroughCasesInSwitch`** — catches missing `break`.

#### Type-coverage as a metric

`type-coverage` (https://github.com/plantain-00/type-coverage) measures the
percentage of code with non-`any` types. > 99% is the goal; < 95% suggests
systemic any-leakage. Look for `type-coverage` in `package.json`,
`devDependencies`, or CI.

#### Primitive obsession

Search for repeated `string` and `number` types where domain types would
catch real bugs:

- `userId: string` vs `userId: UserId` (branded type — can't accidentally
  pass a `productId` where `userId` was expected)
- `email: string` vs `email: Email`
- `cents: number` vs `cents: Cents` (so `Cents` and `Dollars` don't mix)
- `urlString: string` vs `url: URL`
- `Date` vs `string` for timestamps — pick one, don't mix

Branded / nominal types in TS:

```ts
type Brand<K, T> = K & { __brand: T };
type UserId = Brand<string, 'UserId'>;
const asUserId = (s: string): UserId => s as UserId;
```

Look for whether the codebase has a `domain/types/` or similar with brand
definitions, vs raw primitives everywhere.

#### Schema-as-types at boundaries

External-data boundaries (HTTP requests, env vars, DB query results, LLM
outputs) are where types tend to be lies. Schema validation libraries
provide one definition that yields both compile-time types and runtime
parsing:

- **Zod** (most common in TS), **Valibot** (smaller bundle), **Effect Schema**
- **Pydantic** (Python)

The pattern at every boundary:

```ts
const UserSchema = z.object({ id: z.string(), email: z.string().email() });
type User = z.infer<typeof UserSchema>;
const user = UserSchema.parse(rawData); // throws if invalid
```

The anti-pattern: `JSON.parse(rawData) as User`. The cast says it's a User;
nothing actually checks.

#### Discriminated unions vs flat optionals

Catches the most bugs:

```ts
// Bad: flat optionals — must check each combination
type Result = { data?: T; error?: Error };

// Good: discriminated union — TS narrows automatically
type Result = { ok: true; data: T } | { ok: false; error: Error };
```

Search for types with multiple optional fields whose presence is correlated;
usually a missing-discriminated-union smell.

#### Result / Either for expected failures

For *expected* failures (invalid input, not-found, business-rule fail),
returning `Result<T, E>` is more honest than throwing. Reserve `throw` for
programmer errors and unexpected conditions.

Libraries: `neverthrow`, `effect`, or hand-rolled discriminated union.

#### `as unknown as T` smell

`as` is an escape hatch. Concentrated in 1–2 files = quarantined; scattered
across handlers = pervasive type lies.

`grep -rn "as unknown as" src/` and count + locate. Note the distribution.

#### Boolean parameters

Multiple `boolean` parameters at call sites are unreadable:

```ts
sendEmail(user, true, false); // what do those mean?
sendEmail(user, { html: true, urgent: false }); // better
```

Functions with > 1 `boolean` parameter — flag as candidates for
discriminated unions or option objects.

#### TS enums vs `as const`

TS `enum` has subtle issues (numeric pitfalls, duplicate-value bugs,
non-tree-shakeable, doesn't play well with `verbatimModuleSyntax`). Prefer:

```ts
const Status = { Active: 'active', Archived: 'archived' } as const;
type Status = (typeof Status)[keyof typeof Status];
```

Search for `enum ` usage and flag for migration if the codebase prefers
the modern style.

### Step 9: Tests as architecture signal

Tests reflect architecture. If tests are very hard to write, the architecture
probably has hidden coupling.

- Test directory layout: mirrors source? next to source? all in `tests/`?
- Test types: unit only? integration? end-to-end?
- Test coverage of business logic vs controllers vs UI?
- Are tests mostly mocking everything, or do they hit real (test) DBs?
- Quick sanity check: is the test count proportional to the source size?

### Step 9.5: Time, date, and timezone correctness

The most common silent-bug class. Almost every codebase has at least one
of these:

- **Mixing UTC and local.** Server stores `created_at` in UTC; some code
  paths convert to local for display, others compare UTC vs local strings,
  off by `(local offset)` hours.
- **String dates in DB or API.** ISO 8601 strings vs unix timestamps —
  pick one and enforce. Mixed formats accumulate.
- **`new Date(year, month, day)`** — `month` is 0-indexed in JS. `new
  Date(2026, 1, 1)` is February 1, not January 1. Frequent bug source.
- **`Date#getMonth` 0-indexed** — same trap from the other side.
- **Naive month / year arithmetic.** Adding 30 days isn't "next month".
  Adding 365 days isn't "next year" in leap years. Use `date-fns` /
  `dayjs` / `Temporal` (when available) helpers.
- **Daylight Saving boundaries.** Cron jobs scheduled at 2 AM local fire
  twice or zero times on DST changes. Twice-per-day jobs may suddenly
  triple or skip. Check schedules around DST-relevant boundaries.
- **Leap years (Feb 29).** Edge cases bug business logic that assumes year
  length.
- **Locale formatting.** `toLocaleDateString()` without explicit locale
  defaults to user-system — different per user. Use `Intl.DateTimeFormat`
  with explicit locale.
- **Persisted Date columns.** Postgres `timestamp` (no timezone) vs
  `timestamptz` (UTC-stored, displayed in client TZ). Pick `timestamptz`
  unless there's a clear reason. Mixing is a debugging nightmare.
- **`Date.now()` in tests.** Frozen-time discipline (Sinon, Jest fake
  timers, Vitest `vi.useFakeTimers()`) — non-deterministic time = flaky
  tests.
- **Timezone-naive scheduling.** "Run at 9 AM" — for whom? UTC, server TZ,
  user TZ, business HQ TZ?
- **Date input validation.** `parseDate(userInput)` falling through to
  `new Date(userInput)` — accepts almost anything as valid, silently
  produces `Invalid Date`. Use a proper parser.

Check for the `Temporal` API (TC39 Stage 3) once browsers / Node ship it
widely — it solves several of the above by design.

### Step 10: Technical debt markers

Search for:
- `TODO`, `FIXME`, `HACK`, `XXX`, `WTF` comments.
- Commented-out code (often old code "kept just in case").
- Files dated as old by `git blame` but still linked into the active build.
- "Temporary" workarounds with no removal date.

Count them, group them, and call out the worst clusters.

---

## Red flags

- A single file > 1000 lines (god file).
- Handler files > 300 lines.
- `lib/utils.ts` with > 30 unrelated exports.
- Components that import the ORM directly.
- More than one Button component (or any other primitive).
- `any` count growing over time.
- Test count flat as source grows.
- TODOs that are dated > 1 year ago.
- Multiple state management libraries coexisting (Zustand + Redux + Context).
- "Legacy" / "v2" / "new-" directories alongside the originals (incomplete migrations).

---

## Output structure

```markdown
# Architecture Review

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Architectural Overview
- Major structural components (cite directories)
- Implicit / explicit layers
- Module organization style (technical-layer vs feature)

## Coupling and Separation of Concerns
For each violation found: location (file path), issue, impact, fix.
- Tightly coupled modules
- Violations of modular design
- Missing abstraction layers
- Cross-feature reach-ins

## Codebase Maintainability
- Naming consistency
- Code duplication (cite top duplicates)
- Complexity (cite god files)
- Error handling patterns
- Type safety (for TS projects)

## Technical Debt
- TODO/FIXME inventory (count, oldest, worst clusters)
- Commented-out code
- Incomplete migrations
- "Temporary" workarounds

## Test Architecture
- Test layout and types
- Coverage shape (where tests are dense vs sparse)

## Architectural Improvements
Prioritized list. Each: severity, effort, expected benefit, evidence.

## Final Recommendation
The single highest-impact architectural action for this codebase right
now. **One thing.**

## Decision Summary

- Architecture supports the changes the team needs to make: yes / no / partial
- Worst structural issue: <description>
- Recommended posture: <keep | targeted refactor | bigger structural shift>

## Confidence
High / Medium / Low (and why)

## Out of Scope / Inconclusive
Things that need team discussion or product context to evaluate (e.g., "should
this be a microservice" requires product knowledge this audit lacks).
```

---

## Severity calibration

- **Critical:** an architecture issue that actively breaks production or blocks
  the team from shipping (circular import causing runtime errors, missing
  abstraction making every change a multi-file edit).
- **High:** changes are slow, error-prone, or repeatedly introduce bugs in
  unrelated areas.
- **Medium:** clear smells (duplication, god files) without immediate harm.
- **Low:** style and consistency issues.

---

## Evidence requirements

Every finding cites a file path. For coupling claims, show the actual import or
call site. For duplication claims, list at least two of the duplicate sites.
Avoid taste-based critiques without evidence.
