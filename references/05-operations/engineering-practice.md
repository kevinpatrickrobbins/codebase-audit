# 14 — Engineering Practice (Lifecycle Audit)

Audit the **development lifecycle process** the team follows — code review,
release engineering, incident response, deprecation, and the repo-health
signals that reveal engineering culture.

This module exists because much of "how the team works" lives outside any
single dimensional audit. PR process, release discipline, on-call posture,
and deprecation policy are scattered across DevOps and Architecture in
practice; this module brings them together.

Output: `/docs/audits/engineering-practice-audit.md`

---

## Cross-module relationship

- **DevOps** (`05-operations/devops.md`) covers infrastructure;
  this module covers process around it.
- **Documentation** (`03-quality/documentation.md`) covers content;
  this module covers the practices that generate / consume it.
- **Testing** (`03-quality/testing.md`) covers test discipline;
  this module covers PR-level quality discipline.

---

## Step 1: PR / code review process

- **Branch protection** on the default branch. Verifiable via `gh api
  repos/<owner>/<repo>/branches/main/protection` or settings doc:
  - Required PR before merge?
  - Required passing checks (which ones)?
  - Required reviews (count)?
  - Linear history / squash strategy?
  - Force-push allowed?
  - Admin enforcement?
- **PR template** present (`.github/pull_request_template.md`)?
  Includes context / testing notes / linked issue?
- **Issue templates** (`.github/ISSUE_TEMPLATE/`) — bug report / feature
  request / security report?
- **CODEOWNERS** (`.github/CODEOWNERS` or `docs/CODEOWNERS`) — coverage
  of the codebase? Sole owner = bus-factor risk; un-owned files don't get
  reviewed.
- **Review depth.** Look at recent PRs (`gh pr list --state merged
  --limit 20`). Review-comment count distribution. PRs merged with zero
  review comments and no requested changes is a smell on substantive
  PRs.
- **Review latency.** Time-to-first-review and time-to-merge medians.
  Long latency = team scaling pain; instant merges from author = absent
  review.
- **Self-merge** of substantive PRs — branch protection should prevent;
  if not enforced, flag.
- **Conventional Commits** or other commit-message convention. Drives
  changelog automation.
- **Squash vs rebase vs merge.** Consistency matters more than the choice.
- **PR size.** Large PRs (> 1000 LOC changed) are review-resistant; flag
  if median PR size is large.

---

## Step 2: Release engineering

- **Versioning scheme.** Semver, calver, sequential, or none?
- **Tag discipline.** Tags created at release? Annotated or lightweight?
  Signed?
- **CHANGELOG.md** maintained (cross-ref documentation module). Auto-
  generated from commits or hand-curated?
- **Release notes** at the GitHub Releases level, beyond CHANGELOG.
- **Release cadence.** Continuous deployment, weekly, monthly,
  ad-hoc-when-ready? Visible from tag history.
- **Release branch strategy** — trunk-based / `main` deploys, or
  `release/*` branches with backports?
- **Phased rollout.**
  - Web: canary / preview environments / feature flags.
  - Native: store phased rollout (App Store, Play Console).
  - Backend services: blue/green / rolling / one-instance-at-a-time.
- **Rollback story.** How is a bad release reverted? Documented?
- **Pre-release validation.** Staging environment that mirrors prod?
  Smoke tests after deploy?

---

## Step 3: Feature flags

- **Flag system.** LaunchDarkly / GrowthBook / ConfigCat / Statsig /
  Hypertune / Vercel Edge Config / Unleash / custom?
- **Flag hygiene.**
  - Stale flags — flags older than 60–90 days that are still around?
  - Owners — does each flag have a documented owner?
  - Expiry — does the flag system enforce expiry / cleanup?
  - Documentation — what does each flag gate?
- **Kill-switches.** Critical features behind a flag for emergency disable?
- **Code without flags.** Critical changes shipping without rollout
  protection?

---

## Step 4: Incident response

- **On-call setup.** PagerDuty / Opsgenie / Splunk On-Call / GitHub
  Action-based / none.
- **On-call runbook.** Cross-ref documentation module — but the linkage
  between alerts → runbooks is critical.
- **SLOs / SLIs.** Defined? Tracked? Error budget tracked?
- **Alert fatigue signals.** Repeated noisy alerts in commit history,
  alert-disabled comments, "ignore this alert it's broken" runbook items.
- **Postmortem culture.** Postmortem template? Public / internal? Recent
  postmortems in the repo (`docs/postmortems/`)? Action items tracked?
- **Status page.** Public status page (statuspage.io / Atlassian /
  Better Stack / custom)? Linked from app?
- **Communications template.** Customer comms drafts for major incident
  classes?

---

## Step 5: Deprecation & sunset

The most-skipped lifecycle stage.

- **Deprecation policy** documented (often in API docs)? Headers
  (`Sunset:`, `Deprecation:`)? Sunset dates announced before removal?
- **Migration guides** for deprecated → current.
- **Long-tail support windows** — how long old API / old client versions
  are supported.
- **Sunset of unused features.** Apps accumulate features; pruning policy
  exists?
- **Data sunset.** When features are removed, what happens to the data
  they generated? Retained? Migrated? Deleted with notice?

---

## Step 6: Repo-health git signals

These are derivable from `git log` / `gh api` and tell you a lot about
team scale, focus, and pain points that no static-analysis audit can.

- **Commit cadence.** `git log --oneline | wc -l` / time since first
  commit. Daily commits? Weekly? Sporadic?
- **Active contributors.** `git shortlog -sn --since="6 months ago"` —
  distribution. Solo? Long tail? Healthy plurality?
- **Bus factor.** Top contributor's % of commits in last 6 months. > 80%
  is a single-point-of-failure signal.
- **Hot files.** `git log --pretty=format: --name-only --since="6 months
  ago" | sort | uniq -c | sort -rg | head -10` — top-changed files.
  Files with disproportionate churn often indicate pain points or god
  modules (cross-ref architecture module).
- **Branch lifetime.** Long-lived feature branches accumulate merge
  conflicts. `gh pr list --state open` showing PRs open > 30 days.
- **Stale issues / PRs.** `gh issue list --state open --search
  "updated:<2024-11-09"` count.
- **Velocity trend.** PRs merged per week trending up, flat, or down?
- **Author distribution per file.** Files only ever touched by one person
  = knowledge silo.

---

## Step 7: Telemetry / instrumentation hygiene

- **Event taxonomy.** Are analytics events defined in a typed registry
  (vs string literals scattered in code)?
- **Event schema** versioned?
- **PII in events** — scrubbed? Cross-ref privacy module.
- **Event dictionary** documented somewhere developers can find?
- **Naming convention** consistent (`object_action` vs `actionedObject`
  vs random)?
- **Test coverage of analytics.** Are tracked events asserted in tests?

---

## Step 8: Migration discipline

- **DB migrations.** Cross-ref DevOps. Each migration's up/down tested,
  rollback story clear.
- **Data backfills.** Long-running backfill procedures — idempotent? Can
  be paused/resumed? Progress tracked?
- **Feature-flag migrations.** Removing a flag without leaving stub code.
  Cleanup PRs after rollout.
- **API version migrations.** When a v1 → v2 migration happens, are
  callers proactively notified?
- **Breaking-change communication.** Changelog entry, in-app notice,
  email, deprecation period?

---

## Red flags

- Default branch with no protection rules.
- CODEOWNERS absent in a project with > 3 contributors.
- Median PR has zero review comments before merge.
- One contributor holds > 80% of recent commits with no documented
  succession.
- Releases are "whenever someone remembers."
- No phased rollout for production deploys.
- No on-call rotation despite production users.
- "ignore this alert" comments in code.
- Postmortems absent on a project with known incidents.
- Feature flags older than a year still around.
- API has no deprecation policy.
- Stale-PRs count rising over time.
- Hot files match closely with bug-fix PRs (where the fires keep happening).

---

## Output structure

```markdown
# Engineering Practice Audit

Date: YYYY-MM-DD
Repository commit: <hash>

## PR / code review
- Branch protection rules
- PR template / issue template
- CODEOWNERS coverage
- Review depth signals
- Review latency
- Commit-message convention
- PR-size discipline

## Release engineering
- Versioning scheme
- Tag / changelog discipline
- Release cadence
- Phased rollout
- Rollback story

## Feature flags
- Flag system
- Hygiene (stale / owned / documented / expiring)
- Kill-switch coverage

## Incident response
- On-call setup
- SLOs / SLIs
- Alert hygiene
- Postmortem culture
- Status page
- Customer comms templates

## Deprecation & sunset
- Documented policy
- Migration guides
- Support windows
- Feature pruning
- Data sunset

## Repo-health signals (git-derived)
- Commit cadence
- Contributor distribution / bus factor
- Hot files (cross-ref architecture)
- Branch lifetime / stale PRs
- Velocity trend

## Telemetry hygiene
- Event taxonomy / registry
- Schema versioning
- PII scrubbing (cross-ref privacy)
- Event dictionary
- Test coverage of analytics

## Migration discipline
- DB migrations
- Data backfills
- Feature-flag cleanup
- API version migrations
- Breaking-change communication

## Findings

## Final recommendation
Single highest-priority practice action.

## Decision summary
- PR review enforced: yes/no
- Releases reproducible and reversible: yes/no
- On-call coverage adequate for production: yes/no
- Bus factor: <N> (>1 healthy)
- Worst gap: <description>

## Confidence
High / Medium / Low

## Out of Scope / Inconclusive
(things requiring repo settings access via `gh` or talking to the team —
e.g. branch protection rules, real on-call rotations, alert fatigue).
```

---

## Severity calibration

- **Critical:** no branch protection on a production codebase; no on-call
  on a production system with users; bus factor of one.
- **High:** zero-review merges of substantive PRs; no rollback story; no
  postmortem culture on a system with incidents.
- **Medium:** stale feature flags; missing deprecation policy; sparse
  PR templates.
- **Low:** style / convention / hygiene.

Some findings here require running `gh` commands or talking to the team —
note clearly when a finding is from-code-only and when it would need
verification. Branch protection rules can be checked via
`gh api repos/<owner>/<repo>/branches/main/protection` if `gh` is
authenticated.

Every finding cites a file path or a `gh`-derivable evidence path.
