# 06 — Testing & Quality

Audit the project's test posture: coverage shape, pyramid balance, flakiness
signals, contract testing, mutation testing, and the broader "tests as
engineering culture signal."

Output: `/docs/audits/testing-audit.md`

---

## Cross-module relationship

- **Architecture** (`03-quality/architecture.md`) treats tests as an
  architecture signal — "if tests are hard to write, the architecture has
  hidden coupling." This module is the focused deep-dive.
- **DevOps** (`05-operations/devops.md`) covers CI gating; this module
  audits what runs *inside* the CI test job.

---

## Step 1: Test framework inventory

Identify what's in use:

- **Unit:** Jest / Vitest / Mocha / Pytest / RSpec / Go test / xUnit etc.
- **Integration:** same frameworks, often with a separate config.
- **E2E:** Playwright / Cypress / WebdriverIO / Selenium / Detox (RN) /
  Maestro / Espresso (Android) / XCUITest (iOS).
- **Component:** Storybook + Testing Library / Cypress Component Testing.
- **API contract:** Pact / Schemathesis / Dredd.
- **Visual regression:** Percy / Chromatic / Argos / Loki / Backstop.
- **Load:** k6 / Artillery / JMeter / Gatling.
- **Mutation:** Stryker (Node) / mutmut (Python) / pitest (Java).
- **Property-based:** fast-check (Node) / Hypothesis (Python) /
  proptest (Rust).

Multiple frameworks for the same layer is sometimes a smell of unfinished
migration.

---

## Step 2: Coverage shape (not just %)

Raw coverage % is a weak signal. **Coverage shape** is what matters.

- **Where is coverage dense?** Unit tests over pure utilities, business
  logic, validators?
- **Where is coverage thin?** Critical paths — auth, payments, data
  mutations? Authorization checks? Error branches?
- **Coverage of new code.** Many projects have high overall coverage from
  early greenfield tests but write nothing for new code. Check git history /
  recent additions.
- **Coverage tool.** `c8` / `nyc` / `vitest --coverage` / `coverage.py` —
  report format (text vs lcov vs cobertura).

Recommend running coverage with branch (not just line) tracking.

---

## Step 3: Test pyramid balance

A healthy project has a pyramid: many fast unit tests, fewer integration
tests, few e2e tests. Inverted pyramids ("ice cream cone") — many e2e, few
unit — are slow, flaky, and a culture signal.

For each layer count the tests:

- **Unit count vs e2e count.** Ratio ~70:20:10 healthy; 10:30:60 unhealthy.
- **Test runtime distribution.** A unit test should run < 10ms; integration
  < 1s; e2e < 30s. Aggregate runtime budgets per layer matter.
- **Layer separation enforced?** Unit tests not hitting DB/network. Often
  drifts — `import { db }` in a "unit" test is a smell.

---

## Step 4: What's tested vs what should be

For the project's most critical paths (from Module 01 — System Architecture):

- **Auth.** Login, logout, session, password reset, 2FA — covered?
- **Authorization.** Per-role / per-tenant access checks — covered? (Most-missed.)
- **Payments / billing.** Subscription create, cancel, upgrade, failed
  payment, webhook delivery — covered?
- **Data mutations.** Create, update, delete on primary entities — covered
  with fixtures that exercise edge cases?
- **Webhooks (inbound).** Signature validation tested? Replay protection?
- **Background jobs.** Idempotency, retry, failure modes — tested?
- **Migrations.** Each migration's up + down tested? On real schema?

A 90%-coverage project that doesn't test its authorization checks has
worthless coverage.

---

## Step 5: Test reliability

- **Flaky tests.** Search for `.skip`, `.todo`, `.only`, `xit`, `xdescribe`,
  retry decorators (`retry: 3`). Each is a signal of accepting flake.
- **Quarantine / flaky-test tracking.** Some projects have a `flaky/` test
  dir or a CI label. Better than nothing; ideal is zero flakes.
- **Snapshot tests.** Snapshots that update on every run signal weak
  assertions. Snapshot count vs assertion count balance.
- **Time / random / network in tests.** Mocked `Date.now`, frozen UUIDs,
  mocked fetch — or are tests racing real time / network?

---

## Step 6: Contract testing (for APIs / services)

- **OpenAPI / GraphQL schema** validated as a test artifact?
  `openapi-validator`, `swagger-cli`, schema-diff in CI.
- **Consumer-driven contracts.** Pact / Spring Cloud Contract — required
  for multi-service projects to prevent integration regressions.
- **Mock servers** matched to the schema (Prism, MSW seeded from
  OpenAPI).
- **gRPC / Protobuf** breaking-change detection (`buf breaking`).

---

## Step 7: Mutation & property-based (advanced)

- **Mutation testing** (Stryker, mutmut, pitest) — measures whether tests
  actually catch *bugs*, not just whether they execute the code. Mutation
  score > 80% is strong; absent is a maturity gap, not a Critical.
- **Property-based tests** (fast-check, Hypothesis) for parsers, serializers,
  invariant-bearing logic. Often missing where most useful.

---

## Step 8: Test data and fixtures

- **Factories** (`fishery`, `factory_bot`, `factory_boy`) vs hand-written
  fixtures. Factories scale; hand-written fixtures rot.
- **Snapshot of production data** in fixtures — a privacy violation if
  unsanitized real PII.
- **Database seeding** for integration tests — transactional rollback or
  cleanup-and-recreate?

---

## Step 9: Speed

- **Total test suite runtime.** A 30-minute test suite is a productivity
  drag and a CI cost. Shard, parallelize, prune.
- **Slowest 10 tests.** Frameworks usually surface this. Often disproportionately
  expensive tests can be rewritten.
- **CI test time.** Ideally < 10 minutes for PR feedback.

---

## Step 10: CI gating

- **Test job blocks merges?** Required check on the protected branch.
- **Parallelism.** Sharding (`--shard`) or matrix?
- **Caching.** Build cache reused across runs?
- **Coverage gating.** Threshold enforced (e.g., < 80% fails) — or just
  reported?
- **Flaky retry policy.** Auto-retry once is sometimes pragmatic;
  unbounded retries hide bugs.

---

## Red flags

- "Ice cream cone" pyramid (mostly e2e).
- Authorization paths not tested.
- > 5% of tests skipped / todo / quarantined.
- Snapshots updated mechanically each run.
- Test suite > 30 minutes for PR feedback.
- No fixtures factory; hand-written copy-pasted test data.
- Real PII in test fixtures.
- Migration up/down not tested.
- Webhook signature validation not tested.
- New code shipping without new tests (git diff vs test diff).
- Flaky tests retried indefinitely.

---

## Output structure

```markdown
# Testing & Quality Audit

Date: YYYY-MM-DD
Repository commit: <hash>

## Test framework inventory
(unit / integration / e2e / contract / visual / load / mutation / property)

## Coverage shape
- Overall %: <N>
- Branch coverage tracked: yes/no
- Critical paths covered: auth / authz / payments / mutations / webhooks /
  migrations (per-area assessment)
- New-code coverage trend

## Pyramid balance
- Unit / integration / e2e counts
- Runtime budget per layer
- Layer separation enforced

## Reliability
- Skipped / todo / quarantined count
- Flake tolerance signals
- Snapshot discipline
- Time / random / network mocking

## Contract testing
- Schema validation in CI
- Consumer-driven contracts (if multi-service)

## Mutation / property-based
- Mutation score (if measured)
- Property-based test presence

## Test data
- Factories vs hand-written
- Production-data risk

## Speed
- Total runtime
- Slowest tests
- PR feedback time

## CI gating
- Required check
- Parallelism / sharding
- Coverage threshold (blocking?)
- Retry policy

## Findings

## Final recommendation
Single highest-priority testing action.

## Decision summary
- Authorization tested: yes/no
- Pyramid healthy: yes/no
- Suite blocks merges: yes/no
- Worst gap: <description>

## Confidence
High / Medium / Low

## Out of Scope / Inconclusive
```

---

## Severity calibration

- **Critical:** untested authorization on revenue-critical paths; no test
  coverage of payment flows; migration without tested rollback.
- **High:** > 10% test suite skipped/quarantined; ice-cream-cone pyramid
  blocking CI for hours; no contract tests in a multi-service project.
- **Medium:** fixtures hand-rolled; mutation testing absent; flaky retries.
- **Low:** style / framework hygiene.

Every finding cites the test file or the absence-evidence (what was
searched for and not found).
