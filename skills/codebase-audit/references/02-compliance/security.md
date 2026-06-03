# 02 — Security Audit

Perform a security audit of the repository.

Create or update the document:

`/docs/audits/security-audit.md`

If the file already exists, append a new dated section rather than replacing the
previous one — historical audits are useful for tracking regression.

---

## How to investigate

A security audit is most effective when grounded in concrete findings, not generic
checklists. Walk through the categories below in order. For each, do the actual
search; do not assume "they probably have it" or "they probably don't".

### Step 0: Live Discovery

Security advice doesn't rot as fast as some dimensions, but a few things move:

#### Current OWASP Top 10

Fetch https://owasp.org/www-project-top-ten/ — the current Top 10
(refreshed every 3–4 years; the most recent at this skill's writing was 2021).
A new edition would shift the categories below.

#### Current security-header best practice

Fetch https://owasp.org/www-project-secure-headers/ — current recommended
header set; new headers are added periodically (most recently `Permissions-Policy`).

#### Known-vulnerable patterns

Fetch https://github.com/advisories or per-ecosystem advisory feeds for
patterns that have emerged since this skill's last update. The lockfile-pattern
list in Step 8 reflects skill last-update; the live feed is authoritative.

#### Encryption recommendations

Cryptographic best practices evolve. Fetch https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html —
current recommendations for password hashing (currently argon2id
preferred, bcrypt acceptable, scrypt acceptable; PBKDF2 if FIPS-required).

#### Output of discovery

```markdown
## Live Discovery (fetched YYYY-MM-DD)

| Source | Discovered |
|---|---|
| OWASP Top 10 (owasp.org) | Current: <year> edition; categories: <list> |
| OWASP Secure Headers | Current header set: <list> |
| Password hashing (OWASP cheat sheet) | Currently recommended: <algorithm> |

Audit calibrated against this state.
```

If discovery fails: note explicitly. Embedded knowledge below reflects skill
last-update.

---

### Pass A: Trust Boundaries Enumeration

Before secret-hunting, enumerate every **trust boundary** in the system. A
trust boundary is any place where data crosses from one identity / privilege
context to another: HTTP entrypoints, RPC handlers, queue consumers, webhook
receivers, scheduled jobs, server-to-server calls, IPC, file-upload handlers.

For each boundary, document:

| Boundary (file:line) | Who can call it | Identity required | Authorization check | Data it can read/write | What happens on failure |
|---|---|---|---|---|---|

Walk every route file, server-action file, worker, queue consumer, cron
handler, and webhook receiver. List them. For each, fill the row.

This is the *foundation* for tenant-isolation, authorization, and secret-
leak findings — you cannot audit what you have not enumerated. A common
failure mode is an audit that grep-checks for known patterns but never
asks "what's the full set of entrypoints?"

### Step 1: Secrets in source

```
# Patterns to grep across the repo (case-insensitive):
- AWS:          AKIA[0-9A-Z]{16}
- Stripe:       sk_live_, pk_live_, rk_live_
- GitHub:       ghp_, ghs_, github_pat_
- Anthropic:    sk-ant-
- OpenAI:       sk-proj-, sk-[A-Za-z0-9]{40,}
- Generic:      api[_-]?key, secret, token, password, bearer (with literal values)
- JWT secrets:  jwt_secret, JWT_SECRET (with values)
- Connection strings: postgres://, mysql://, mongodb+srv:// (with creds)
```

Also check **non-obvious secret-leak vectors** beyond source code:

- **Browser bundles.** Any production secret reaching `window.*`, build-time
  inlined into client JS, or available via `process.env.NEXT_PUBLIC_*` /
  `VITE_*` / `EXPO_PUBLIC_*` / `PUBLIC_*` prefixes. Search: `grep -rn
  "NEXT_PUBLIC_\|VITE_\|EXPO_PUBLIC_\|PUBLIC_"` and verify each is
  legitimately public.
- **AI prompts.** When the codebase ships system prompts to LLMs that
  include API keys, internal URLs, or user PII inline — those become part
  of the provider's request log and (without proper opt-out) potentially
  training data. Search for prompt construction near LLM calls.
- **Logs.** Cross-ref Step 11 (Logging hygiene).
- **Error responses.** Stack traces leaking env values or paths.
- **Source maps in production** exposing file paths and credentials in
  comments.

Also check `.env`, `.env.local`, `.env.production` if present. In a healthy project
these are gitignored — verify by checking `.gitignore`. If `.env*` files are
committed, that is a Critical finding.

#### Secrets in git history

Removing a secret from current code is meaningless if it was ever committed
and pushed. The full git history must be scanned, not just the working tree.

- **Run a history scanner.** Recommend (or run, if available):
  - `gitleaks detect --source . --redact` — fast Go tool with default rules
  - `trufflehog git file://. --only-verified` — verifies whether the
    detected secret is actually live by calling the relevant API
  - `git secrets --scan-history` (older AWS Labs tool)
  - `detect-secrets scan --baseline .secrets.baseline` (Yelp)
- **GitHub native scanning:**
  - `gh api repos/{owner}/{repo}/secret-scanning/alerts` — if `gh` is
    authenticated, returns secret-scanning alerts (public repos and
    private with Advanced Security have this on automatically)
  - `gh api repos/{owner}/{repo}` returns `secret_scanning_push_protection`
    — verify push protection is on
- **Pre-commit hook for secrets.** Search for `.pre-commit-config.yaml`,
  `husky` config, lefthook, or similar — is `gitleaks-pre-commit`,
  `detect-secrets`, or `talisman` wired in? Pre-commit catches the next
  one before it lands.

**If a secret is found in history:**

- The secret must be **rotated** (assume compromised). Non-negotiable.
- History rewrite (`git filter-repo`, BFG Repo-Cleaner) is *optional* and
  contentious — it doesn't help once anyone has cloned, but it stops future
  readers. Document the choice.
- For public repos: assume the secret is in someone's training data,
  archive, or fork. Rotation alone is the right response.

**Severity calibration:**

- **Critical:** any live secret in git history of a public repo, or any
  unrotated secret in any repo.
- **High:** no scanning tooling configured at all on a project with > 6
  months of history.
- **Medium:** scanning runs only on PRs, no historical baseline scan
  ever performed; push protection off.

### Step 2: Authentication implementation

- **Password storage.** Find where passwords are persisted. Is hashing in use? Look
  for `bcrypt`, `argon2`, `scrypt`, `crypto.pbkdf2` calls. If you see `crypto.createHash('md5'|'sha1'|'sha256')`
  applied directly to passwords, that is broken (no salt, no work factor).
  Plaintext storage is Critical.
- **Session management.** Cookie config: `httpOnly: true`? `secure: true` in prod?
  `sameSite: 'lax' | 'strict'`? Search for `cookie.set` / `res.cookie` / NextAuth /
  Auth.js session config. Missing `httpOnly` allows XSS-driven session theft.
- **JWT handling.** If JWTs are used, check the algorithm: `none` is Critical;
  `HS256` with a weak/exposed secret is Critical; `RS256` is the safer default.
  Check whether tokens are validated (`jwt.verify`) or merely decoded (`jwt.decode`).
- **Reset / verification flows.** Are reset tokens single-use? Time-limited? Check
  the reset handler for token reuse and replay.

### Step 3: Authorization (the most missed category)

Authentication says "you are who you say you are." Authorization says "you are
allowed to do this." Most real-world breaches are authorization failures.

- For every API route or server action that returns or mutates data, ask: does it
  check that the *caller* is authorized to access *this specific resource*?
- Search for handlers that fetch by ID without ownership check:
  ```
  // Anti-pattern:
  const post = await db.post.findUnique({ where: { id: req.params.id } });
  return post;  // No check that req.user.id === post.authorId
  ```
- Look for admin-only routes — are they actually gated, or just hidden in the UI?
- For multi-tenant apps: is `tenantId` enforced server-side, or only filtered
  client-side?

### Step 4: Injection vulnerabilities

- **SQL injection.** Grep for raw SQL with string concatenation:
  `"SELECT * FROM " + tableName`, template literals with user input,
  `query(\`... ${input}\`)`. ORM use generally avoids this; raw queries reintroduce it.
- **Command injection.** Search for `child_process.exec`, `spawn`, `subprocess.call`
  with user input. `exec(userInput)` is Critical.
- **Prototype pollution.** `Object.assign({}, userInput)`, recursive merge libraries
  on untrusted input.
- **NoSQL injection.** Mongo queries built from `req.body` directly (e.g.,
  `User.find({ email: req.body.email })` — if body is `{ "$ne": null }`, returns all).
- **Path traversal.** File reads from user input — `fs.readFile(userPath)` without
  validation; `path.join(__dirname, userInput)` without `path.normalize` check.

### Step 5: XSS

- **React/Vue/Svelte:** grep for `dangerouslySetInnerHTML`, `v-html`, `{@html ...}`.
  Each instance must be reviewed — is the input sanitized (DOMPurify) or trusted?
- **Server-rendered:** raw template interpolation of user input.
- **Reflected XSS:** error messages that include unsanitized query parameters.

### Step 6: CSRF

- **Cookie-based auth + state-changing forms?** Then CSRF protection is required.
  Look for CSRF token middleware (`csurf`, NextAuth's built-in protection,
  Django CSRF middleware, Rails `protect_from_forgery`).
- **`SameSite=Lax` cookies** mitigate most CSRF for top-level navigations but not
  for embedded contexts. Check.
- **Pure JWT-in-Authorization-header** flows are not vulnerable to classic CSRF
  but are vulnerable to other attacks (XSS-driven token theft).

### Step 7: Headers

Search for security headers — usually in middleware, `next.config.js`,
`vercel.json`, or `helmet()` in Express:

- `Content-Security-Policy` — prevents most XSS exploitation
- `Strict-Transport-Security` — HSTS
- `X-Frame-Options: DENY` or CSP `frame-ancestors`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy`
- `Permissions-Policy`

### Step 8: Dependencies

- Read the lockfile date. If `package-lock.json`/`yarn.lock`/`pnpm-lock.yaml` was
  last modified > 6 months ago, dependencies are likely behind on security patches.
- Check for known-vulnerable patterns: very old `lodash` (< 4.17.21),
  `node-fetch` < 2.6.7, `axios` < 0.21.2, `next` major versions behind, etc.
- Note whether `npm audit` / `pnpm audit` would have output (you can recommend
  running it; you cannot run it without network).

### Step 9: File uploads

- Where does the app accept files? Look for `multer`, `formidable`, signed S3 URLs,
  Next.js API route accepting `multipart/form-data`.
- Are file types validated (and not just by client-side `accept=` attribute)?
- Are filenames sanitized (path traversal: `../../etc/passwd`)?
- Is the upload destination outside the web root, or behind authentication?
- Are uploads served back via the same domain (XSS risk via SVG, HTML files)?

### Step 10: Rate limiting and abuse

- Is there rate limiting on login, password reset, signup, API endpoints? Search
  for `rate-limit`, `upstash/ratelimit`, `@vercel/edge-config`, `express-rate-limit`.
- Is there CAPTCHA on signup or sensitive actions?
- Are error responses verbose enough to enable user enumeration? ("This email is
  already registered" is enumeration; "Check your email" is not.)

### Step 11: Logging hygiene

(Cross-ref: Module 03 Privacy covers the privacy / GDPR angle on logging.)

- **Stack traces in responses.** `5xx` responses returning stack traces leak
  file paths, library versions, internal function names. Verify production
  config returns sanitized errors (`app.set('env', 'production')` in Express,
  Next.js production builds, Django `DEBUG=False`, Rails production env).
- **Secrets in logs.** Search for `console.log(req.headers)`, `logger.info(user)`,
  `console.log(JSON.stringify(req.body))`. Anything logging the request
  body or full user object is suspect — `Authorization` headers, password
  fields, payment data leak this way.
- **PII in logs.** Any field that is PII (name, email, address, IP, payment
  data) — is it scrubbed before logging? Logger-level redaction (pino's
  `redact`, winston transforms) is durable; per-call redaction rots.
- **Log retention by purpose.** Application logs / audit logs / access logs
  / security event logs each have different retention needs. Indefinite
  retention is a privacy violation; 7-day retention on audit logs is a
  forensics gap.
- **Log access control.** Who can read prod logs? Engineer / on-call /
  all-staff / vendor (Datadog, Sentry)? Documented? Audit log of log
  access for sensitive systems.
- **Log injection.** User input flowing into log lines without escaping —
  attackers can inject CRLF to forge log entries. Less common in
  structured JSON logging.
- **Audit log of admin actions.** Distinct log stream for admin actions on
  user data — required for SOC 2, GDPR breach forensics, customer support
  trust. Records actor / timestamp / target / before-after values.
- **Centralized vs scattered.** Logs aggregated to a single ingest endpoint
  (Datadog, Better Stack, Loki, ELK) vs scattered across servers /
  containers? Aggregation is required for incident response.

---

### Pass B: Agentic-Code Risk

Codebases touched by AI coding agents accumulate a specific class of
hazards that pure-human codebases typically don't. Agents under deadline
pressure (or following permissive instructions) take shortcuts that read
fine in isolation but compromise security in aggregate.

**Search for:**

- **TODO security shortcuts** with no follow-up:
  ```
  grep -rn "TODO.*security\|FIXME.*auth\|TODO.*permission\|HACK.*bypass" src/
  ```
- **Temporary auth bypasses** — `if (process.env.NODE_ENV === 'development')
  return next()` patterns that ship to production. Or `// temporarily
  skip auth check`.
- **Demo / mock credentials.** Search: `admin@example.com`, `test123`,
  `password123`, `mockUser`, `fakeAdmin`, `debug-token`, `god-mode`.
- **Disabled middleware.** `app.use(auth)` → `// app.use(auth)` (commented
  out). `// @ts-ignore` over auth checks. `if (true) return next()`.
- **Broad `any` permissions.** `roles: ['*']`, `scopes: '*'`,
  `permissions: 'all'` — sometimes legitimate, often not.
- **Unsafe fallback behaviour.** `try { authCheck() } catch { /* allow */ }`
  — failing open instead of failing closed.
- **"Temporary" comments around security:**
  ```
  grep -rn "TEMP\|temporary\|HACK\|TODO\|FIXME" src/ --include="*.ts" \
    | grep -iE "(auth|admin|permission|tenant|secret|password|api[_-]?key)"
  ```
- **Test code shipped to prod.** `if (req.headers['x-test-bypass'])` /
  `if (user.email.endsWith('@test.com'))` — backdoors created for testing
  that survived to production.
- **`// AI: ...` or `// Claude: ...` annotations** indicating AI
  generation that may not have been reviewed for security.

**Severity calibration:** any of these in a production-shipped path is
**Critical**. Evidence-based — cite the exact file:line. AI agents are not
malicious, but they're undisciplined about safety nets when humans don't
review.

This pass is genuinely novel (most security audits don't look for it) and
becomes more important the more LLM-augmented coding the team does.

### Pass C: Chained-Risk Analysis

Real breaches rarely come from a single weakness. They come from chains:
minor issue A + minor issue B + missing guardrail C = serious business
risk. Audit checklists that report each issue in isolation miss the most
dangerous patterns.

**For each weakness identified above, ask:**

1. **What would this enable?** (immediate consequence)
2. **What second weakness would compound it?** (escalation step)
3. **Could this expose tenant data?**
4. **Could this create admin access?**
5. **Could this destroy data?**
6. **Could this leak secrets?**
7. **Could this bypass billing, ownership, or organization boundaries?**

Then **construct chain narratives** for the most dangerous combinations.
Format:

```markdown
### Chain: <Short scenario name>

**Severity:** Critical / High

**Steps:**
1. <Weakness 1 — cite finding>
2. <Weakness 2 — cite finding>
3. <Missing guardrail — cite>

**Potential impact:** <concrete consequence — "cross-tenant data export",
"silent admin escalation", "ransomware where backups also destroyed">

**Why each step alone wasn't flagged Critical:** <each step is medium /
high in isolation, but combination is lethal>

**Mitigation (any one step breaks the chain):**
- <fix at step 1>
- <fix at step 2>
- <add the missing guardrail>
```

**Common chain patterns to look for:**

- **Cross-tenant export exposure:** missing tenant check on export route
  + admin middleware trusting client-supplied org ID + shared worker with
  broad DB access = tenant A exports tenant B's data
- **Silent admin escalation:** weak role-assignment endpoint + missing
  audit log + no alerting = attacker grants self admin without trace
- **Ransomware-survivable destruction:** app admin can delete records +
  app admin can delete backups (or backups stored in same account) +
  no immutability = compromised admin destroys everything
- **Secret cascade:** committed secret (rotated but in history) +
  rotation didn't include all consumers + one consumer still uses old
  secret = old secret is still live somewhere
- **Webhook-driven privilege:** webhook signature check missing + webhook
  handler trusts payload `userId` field + handler grants permissions =
  attacker forges webhook to grant self access

The chain analysis is what elevates an audit from "list of issues" to "do
we actually know what could go wrong?". Run it explicitly; document at
least 2–3 chains even on a clean codebase.

### Pass D: Attack Path Priority Scoring

After the vulnerability tables are populated from Passes A–C, score every
**Critical** and **High** finding on four exploitability axes. This produces
a P-level that cross-cuts severity: a Critical finding that requires a
zero-day and admin compromise is real but theoretical (P3); a Medium finding
with a public PoC and no auth required is being exploited right now (P0).
**Severity describes impact. Priority describes urgency.**

**Score each finding on four axes (0 = easiest for attacker, 3 = hardest):**

| Axis | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| **Access required** | None — unauthenticated | Authenticated user | Privileged / admin | Internal network only |
| **Complexity** | Single HTTP request | Multi-step sequence | Requires specific app state | Race condition / timing-dependent |
| **Social engineering** | None — attacker acts alone | Attacker targets only themselves | A user must be tricked | An admin must be tricked |
| **Exploitation barrier** | Public PoC exists | Known technique, manual execution | Custom code required | Novel research / zero-day |

**Sum the four axis scores (0–12) and assign a priority tier:**

| Score | Priority | Interpretation |
|---|---|---|
| 0–3 | **P0** | Ship blocker — do not deploy until fixed |
| 4–6 | **P1** | Fix within 48 hours |
| 7–9 | **P2** | Fix within current sprint |
| 10–12 | **P3** | Document and schedule — currently theoretical |

**How to apply:**

1. For every Critical and High row in the vulnerability tables, add a
   `Priority` column with the P-level and the raw score in parentheses
   — e.g., `P0 (2)`.
2. Apply judgment, not only arithmetic: a score of 4 that can be trivially
   automated by a script can be bumped to P0. Document the reason.
3. After the tables, emit a "P0 / P1 Attack Paths" block (see output
   structure below) listing only P0 and P1 findings sorted by score ascending.
   If none: write "No immediately weaponizable paths identified (all findings
   P2 or P3)."

---

## Red flags

- `eval()`, `new Function()`, `vm.runInThisContext()` on user input.
- Disabled SSL verification (`rejectUnauthorized: false`, `verify=False`).
- Wildcard CORS (`Access-Control-Allow-Origin: *`) on authenticated endpoints.
- `process.env.JWT_SECRET || "dev-secret"` (fallback secret = secret in prod).
- Custom crypto: rolling your own hash, encryption, or signing is almost always wrong.

---

## Output structure

```markdown
# Security Audit

Date: YYYY-MM-DD
Repository commit: <git rev-parse HEAD>

## Scope
What areas of the system were analyzed (and what was out of scope — e.g., no live
penetration testing, no infrastructure access).

## Security Architecture Overview
Authentication model, API protections, data handling — 3–5 paragraph summary.

## Vulnerabilities

### Critical
| Issue | Location | Risk | Attack Scenario | Mitigation | Priority |
|---|---|---|---|---|---|

### High
| Issue | Location | Risk | Attack Scenario | Mitigation | Priority |
|---|---|---|---|---|---|

### Medium
| Issue | Location | Risk | Attack Scenario | Mitigation |
|---|---|---|---|---|

### Low
| Issue | Location | Risk | Attack Scenario | Mitigation |
|---|---|---|---|---|

## P0 / P1 Attack Paths

*Findings from the tables above scored P0 or P1 by Pass D, sorted by score
ascending (most immediately weaponizable first).*

| Priority (score) | Finding | Severity | Why immediately actionable |
|---|---|---|---|

> If no P0 or P1 findings: write "No immediately weaponizable paths identified
> (all findings P2 or P3)."

## Attack Surface Analysis

Cover each of:
- Authentication and session management
- Password storage
- API authentication
- Authorization (most-missed)
- Secrets management
- Environment variables
- Injection vulnerabilities (SQL, command, NoSQL, prototype, path)
- XSS risks
- CSRF protections
- Input validation
- Dependency vulnerabilities
- File upload risks
- Headers
- Rate limiting

## Data Protection
- Handling of sensitive data (PII, payment data, credentials)
- Database access patterns (least privilege?)
- Logging of secrets
- Exposure of internal errors
- Encryption at rest and in transit

## Recommended Remediation Priorities
List the top 5 fixes with estimated effort. (Detailed tickets are filed in
Module 00.3 — Remediation Plan.)

## Final Recommendation
The single highest-impact security action for this codebase right now.
**One thing.**

## Decision Summary

- Authentication / authorization currently sound: yes / no / partial
- Worst category: <secrets | authz | injection | other>
- Single biggest exposure: <description>
- Recommended posture: <ship-blockers fixed | Tier 1 hardening | deeper review needed>

## Confidence
High / Medium / Low (and why)

## Out of Scope / Inconclusive
What requires live testing, infrastructure access, or production data to
confirm.
```

---

## Severity calibration

- **Critical:** RCE, persistent unauthorized data access, payment fraud vector,
  exposed secrets that grant production access, broken authentication.
- **High:** privilege escalation, persistent XSS, unauthenticated PII read,
  business-logic bypass, SSRF.
- **Medium:** information disclosure, weak rate limiting, missing security headers
  on sensitive routes, reflected XSS with mitigations, weak password policy.
- **Low:** best-practice deviations with no clear exploit path, verbose error
  messages with no useful info to attacker, missing security headers on non-sensitive
  pages.

When in doubt about severity, document the attack chain — if you cannot construct
one, drop the severity by a level.

---

## Evidence requirements

Every Critical or High finding must cite the file path and line. If a finding is
based on absence of something (e.g., "no CSRF protection"), describe what you
searched for and where you searched.
