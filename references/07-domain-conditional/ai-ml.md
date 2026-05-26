# 18 — AI / ML Audit

Audit a codebase that uses LLMs or ML models — for prompt safety, output
validation, model versioning, cost guardrails, eval discipline, and
operational posture.

Output: `/docs/audits/ai-ml-audit.md`

---

## When to apply

Run this module **only when** the codebase uses LLMs or ML inference. Detect
by:

- `anthropic` / `@anthropic-ai/sdk` in deps
- `openai` in deps
- `@ai-sdk/*` (Vercel AI SDK)
- `langchain` / `langgraph` / `llamaindex`
- `cohere`, `replicate`, `mistral`, `groq`, `together`
- Cloudflare Workers AI bindings
- AWS Bedrock / Azure OpenAI / GCP Vertex AI clients
- `@huggingface/inference` / transformer-loading code
- Custom inference endpoints invoked from the app

If none of these are present, skip.

---

## Cross-module relationship

- **Cost** (`05-operations/cost.md`) audits LLM cost waste at the model-
  selection level (Opus where Haiku fits, etc.). This module goes deeper
  on the AI-specific concerns.
- **Security** (`02-compliance/security.md`) covers general security; this
  module covers AI-specific attacks (prompt injection).
- **Privacy** (`02-compliance/privacy.md`) — sub-processor relationship
  with LLM providers when prompts contain PII.

---

## Step 0: Live Discovery

Model lineups change monthly. Pricing and capabilities (context windows,
tool use, prompt caching, batch APIs) ship continuously. Fetch current
truth before comparing the codebase's choices.

### Current model lineup

- **Anthropic:** Fetch https://docs.claude.com/en/docs/about-claude/models/overview —
  current Claude model lineup, IDs, context windows, pricing. Or use
  `mcp__plugin_context7_context7__resolve-library-id` for the Anthropic
  TypeScript / Python SDK and `query-docs` for current model usage.
- **OpenAI:** Fetch https://platform.openai.com/docs/models — current
  GPT / o-series lineup, pricing, capabilities.
- **Google Gemini:** Fetch https://ai.google.dev/gemini-api/docs/models —
  current lineup.
- **Other providers** (Mistral, Cohere, Together, Groq, Bedrock, Vertex):
  Search "<provider> models pricing 2026" or fetch their official
  docs page.

For the models the codebase actually calls, capture: current name, current
context window, current pricing, status (current / legacy / deprecated /
sunset).

### Current capabilities

For each provider in use, fetch current support for:

- **Prompt caching** — Anthropic and OpenAI both have prompt caching now;
  capture the current cache hit discount and minimum-prefix size.
- **Batch API** — current pricing discount for non-realtime workloads.
- **Tool use / function calling** — current schema requirements.
- **Structured outputs** — JSON mode, schema-bound, response format.
- **Files API / multimodal** — current support and pricing.
- **Realtime / streaming** — current options.

For Anthropic specifically, the **`claude-api`** skill (in this user's
ecosystem) is the deeper guide — defer to it for SDK-level questions
including caching, thinking, compaction, tool use, batch, files,
citations, memory.

### Current SDK / library versions

- **`@anthropic-ai/sdk`:** `Bash` `npm view @anthropic-ai/sdk version`.
- **`openai`:** `Bash` `npm view openai version`.
- **`@ai-sdk/*` (Vercel AI SDK):** `Bash` `npm view ai version` and the
  individual provider packages.
- **`langchain` / `langgraph` / `llamaindex`:** moving fast — current
  best-practices via `query-docs` (context7).

If the codebase pins to a major version more than one behind current, that's
a finding (modules + providers ship breaking changes meaningfully often).

### Current LLM threat taxonomy

The OWASP Top 10 for LLM Applications is the canonical threat catalogue for
AI features and is refreshed yearly.

- Fetch https://genai.owasp.org/llm-top-10/ — the current list with
  descriptions and example mitigations.
- Or Search "OWASP LLM Top 10 <current year>".

Map findings in this audit to current taxonomy IDs (e.g. "LLM01 — Prompt
Injection", "LLM04 — Data and Model Poisoning", "LLM06 — Excessive Agency",
"LLM10 — Unbounded Consumption") so the report aligns with what security
teams expect to see.

### Output of discovery

```markdown
## Live Discovery (fetched YYYY-MM-DD)

| Source | Discovered |
|---|---|
| Anthropic model lineup (docs.claude.com) | Opus 4.7, Sonnet 4.6, Haiku 4.5 currently active |
| OpenAI model lineup | gpt-4o, gpt-4o-mini, o1, o3-mini current |
| @anthropic-ai/sdk latest (npm) | <version> |
| Anthropic prompt caching | <current discount and min size> |
| Anthropic Batch API | <current discount> |

Calibrated audit follows.
```

If discovery fails: note explicitly. The embedded list of "Opus / GPT-4 /
Haiku / etc." in this skill reflects its last update and will fall behind.

---

## Step 1: Inventory

For each LLM call site:

- **Provider and model.** `claude-opus-4-7`, `gpt-4o-mini`, `claude-haiku-4-5-20251001`,
  `text-embedding-3-small`, etc.
- **Call type.** Chat completion, embedding, image, audio, batch, realtime.
- **Streaming or batch.** Realtime user-facing vs background batch.
- **Where the prompt comes from.** Hand-written constant, template with
  user input interpolated, retrieved from DB, dynamic from another LLM.

A useful output: a table of every LLM call site with model + cost-per-call
estimate.

---

## Step 2: Prompt injection defenses

User input flowing into prompts is the AI equivalent of SQL injection. The
attacker doesn't extract data from a query — they redirect the model to
follow attacker instructions.

- **Untrusted input segregation.** Are user-provided strings clearly
  delimited (XML tags, fenced sections) and the system prompt instructing
  the model to treat them as content, not instructions?
- **Indirect prompt injection.** When the prompt includes content fetched
  from URLs / docs / emails, that content is *also* untrusted. Models will
  follow instructions embedded in fetched content.
- **Tool-calling validation.** When the model can call tools (function
  calling), each tool's arguments must be validated server-side as if
  user-provided. Tool calls are not trusted just because the model emitted
  them.
- **Output never executed without validation.** Model output as a SQL
  query / shell command / file path is dangerous. Validate, sanitize, or
  use parametric forms (e.g., a structured tool call returning a query
  template + safe parameters).
- **Jailbreak resilience.** No silver bullet. Layered defenses:
  system-prompt instructions, output validation, content-policy filtering,
  rate-limited testing for new prompts.

---

## Step 3: Output validation

- **Structured outputs.** When the LLM should return JSON / a schema-bound
  object, use the provider's structured-output mode (OpenAI structured
  outputs, Anthropic tool use with JSON schema, Vercel AI SDK
  `generateObject`) — not "ask nicely and hope for valid JSON."
- **Schema validation** before consuming. Zod / Valibot / Pydantic on the
  output before any business logic uses it.
- **Refusal handling.** Models can refuse; the calling code should
  distinguish refusal from a malformed answer.
- **Hallucination handling.** Where factual accuracy matters (citations,
  numeric facts, code references), validate against ground truth before
  surfacing. RAG with citations is the common pattern; "trust the model"
  is the common bug.

---

## Step 4: Model versioning

- **Pinned model IDs.** `claude-haiku-4-5-20251001` (date-pinned) is
  reproducible; `claude-haiku-4-5` (alias) drifts as the alias updates.
  Either is OK, but the choice should be intentional and documented.
- **Behavior changes on model upgrade.** A test suite for the AI surface
  catches regressions when models update. Without tests, an "upgrade"
  silently breaks.
- **Model fallback.** When the primary model is unavailable / over rate
  limit, is there a fallback? `claude-opus-4-7` → `claude-sonnet-4-6` for
  graceful degradation, or hard fail with a user-visible error.

---

## Step 5: Cost guardrails

(Detail in `05-operations/cost.md`; AI-specific concerns here.)

- **Per-user / per-tenant rate limits.** A single abusive user driving
  $1000/day in Anthropic spend is a real failure mode. Track usage and cap
  it.
- **Token budgets per request.** `max_tokens` set sensibly per use case.
- **Prompt caching** (Anthropic, OpenAI) — for prompts with stable system
  prefixes > 1024 tokens, prompt caching gives ~90% off cached input.
  Verify it's enabled where the prefix is stable.
- **Batch API** (Anthropic Batch, OpenAI Batch, Bedrock) for non-realtime —
  50% off list price.
- **Streaming for non-UX flows** is wasteful — no perceptual benefit and
  harder observability.
- **Spend monitoring.** Daily / weekly spend trended, alerted on anomaly.

---

## Step 6: Evals

A production AI feature without evals is a feature flying blind.

- **Eval set exists?** A representative dataset of inputs + expected /
  acceptable outputs.
- **Auto-graded evals.** Where possible — exact match, schema match,
  rubric-graded by another LLM (with calibration).
- **Eval coverage.** Edge cases, adversarial inputs, multilingual, unusual
  formats.
- **Run cadence.** Pre-release, on every prompt change, scheduled
  regression?
- **Comparative.** When model swap is considered, eval old vs new on the
  same set.
- **A/B testing in prod.** Feature flag the new prompt to a fraction of
  traffic and measure user-side outcomes.

---

## Step 7: RAG (if applicable)

For retrieval-augmented projects:

- **Indexing pipeline.** Document processing, chunking strategy, embedding
  model, vector store.
- **Chunk size and overlap** appropriate for the use case (small for
  factoids, larger for narrative).
- **Hybrid search.** Vector + keyword (BM25) hybrid often beats pure
  vector. Pure-vector RAG is the common naive setup.
- **Metadata filtering.** Vector search constrained by tenant, recency,
  permission?
- **Reranking.** Cohere Rerank, BGE Rerank, LLM-as-reranker — improves
  precision significantly over raw vector top-K.
- **Citations.** Every claim grounded in a retrievable source surfaced to
  the user (link to the source document).
- **Stale index.** When source content changes, index re-builds within
  what cadence?

---

## Step 8: Agent patterns (if applicable)

- **Tool definitions.** Each tool has clear name, description, parameter
  schema. The description is *prompt content* — model picks tools based on
  it.
- **Tool error handling.** Tool failures returned to the model with enough
  context to retry / handle, not as a hard exception.
- **Loop bounds.** Max iterations / max time / max tool calls — agents
  without bounds can spend unboundedly.
- **Side effects.** Tools that modify state (write to DB, send messages)
  require explicit user authorization or per-call confirmation, not
  silent execution.
- **Memory.** Conversation memory bounded; pruning strategy when context
  fills.

---

## Step 8.5: Agent and tool execution surface

Run this step when the codebase exposes an agent harness, registers MCP
servers, or otherwise lets the model invoke tools. Step 8 above covers the
agent *loop* (bounds, memory, side effects). This step covers the *system
surface* the agent operates within — the tools and harness wiring that
together define what the model can actually do.

### Tool / MCP server inventory

- **Which tools and MCP servers are wired in?** Configuration typically
  lives in `mcp.json`, `.mcp.json`, `claude_desktop_config.json`, code
  that calls `registerTool` / `tool({ ... })`, or framework-specific
  equivalents (LangGraph nodes, AI SDK tool definitions, Bedrock agent
  action groups).
- **Trust tier per server.** First-party (built and maintained in this
  repo) vs third-party (npm / pip package, prebuilt binary, fetched
  manifest). A third-party MCP server is arbitrary code executing with
  whatever permissions the harness grants — closer to installing a
  browser extension than to adding a library.
- **What does each tool actually access?** Filesystem (which paths),
  network (which hosts), secrets, the database, billing or payments,
  outbound messaging. A tool that can read `~/.ssh` is a critical-trust
  tool no matter how innocuous its description sounds.
- **Has the source code of third-party MCP servers been reviewed?** Tool
  registration is silent privilege escalation when reviewers don't read
  the implementation.

### Tool description as prompt surface

Tool descriptions are **part of the prompt the model sees** on every
turn. A malicious or compromised tool can include instructions in its
description that steer the model across the entire conversation, not
only when the tool is invoked.

- **Are tool descriptions auditable** — checked into source control,
  reviewed when servers are added, diffed when versions bump?
- **Does the harness pass tool descriptions through verbatim**, or
  sanitize / quote them before injecting? Verbatim injection means any
  upstream string is a potential instruction.
- **Can tool descriptions change dynamically** (e.g., fetched from a
  remote registry at runtime)? If yes, the threat surface includes
  whoever controls that remote.

### Indirect injection escalation through tools

The high-impact escalation path: the model fetches untrusted content
(web page, email, user-uploaded file, search result), that content
contains instructions, and the model emits a tool call following those
instructions — exfiltrating data, sending a message, making a payment.

- **Which tools feed fetched content back into the loop?** `fetch_url`,
  `read_file` on user-uploaded paths, `search_email`, `read_pdf`,
  `browse_page` — any tool whose output can carry attacker text.
- **Are tool-call arguments derived from fetched content validated**
  before execution? `send_email({to: <attacker-controlled>, body:
  <stolen-secret>})` should be caught by a policy layer, not silently
  executed.
- **Is there a confused-deputy boundary** between the model and the
  user? When the model emits a destructive tool call, does the user
  confirm — or does the harness rubber-stamp it?

### Tool-call policy and sandboxing

- **Permission scoping per tool.** Read-only vs read-write. Allow-listed
  paths vs filesystem-wide. Allow-listed hosts vs arbitrary network.
- **Sandboxing.** Do tools run in the harness's own process, or in an
  isolated context (subprocess, container, separate user)? A shell tool
  in the harness process effectively grants the model the harness's
  full privileges.
- **Audit log of tool calls.** Every call recorded with arguments,
  return value, and the conversation turn that triggered it. Without
  this, a successful injection attack is invisible after the fact.
- **Preflight hooks.** Pre-tool-call hooks that can inspect and block
  suspicious calls — writes to `~/.ssh`, sends to external domains,
  payments above a threshold, deletes outside a workspace path.

### Lateral surface via tools

- **Tools that spawn subagents.** `delegate_to_subagent` extends the
  trust boundary — what authority does the child run with?
- **Tools that register more tools at runtime.** Runtime expansion of
  the trust surface based on model output is one of the highest-risk
  patterns.
- **Tools that execute arbitrary shell / code.** A `bash` or `python`
  tool is functionally `eval()` for the model.

### Model-emitted code paths

(Cross-ref Step 2 — prompt injection defenses — specifically about
tool-bypassing execution.)

- **Model output executed as code, SQL, or shell** without traversing
  the tool layer (`exec(model_output)`, `bash -c "$LLM_OUTPUT"`).
- **Model-emitted URLs / file paths followed without validation.**
- **Eval-style JSON-as-code** that uses `eval` or `Function()` on
  parsed model output.

### Harness configuration drift

- **Is MCP / tool config diffed in code review?** Adding a server is a
  security-relevant change but often a one-line edit. CODEOWNERS rule
  or PR template forcing review?
- **Permission allow-lists in source vs dotfile only.** Source-controlled
  allow-lists are auditable; dotfile-only configs drift between
  developers and machines.
- **Hooks committed to source vs configured locally.** Hooks are the
  primary policy layer in many harnesses — they need to be reviewable
  artifacts.

### Red flags (agent execution surface)

- Third-party MCP servers wired in with no documented review.
- Tool descriptions fetched dynamically from a remote source.
- Fetched-content tools feeding directly into state-mutating tools with
  no policy layer between.
- `eval(model_output)` or shell execution of model output anywhere in
  the loop.
- No audit log of tool calls.
- No preflight hook capable of blocking suspicious calls.
- Wide filesystem / network permissions granted to all tools by default.
- Subagent / delegation tools inheriting full parent authority with no
  scoping.
- Tool config stored only in a developer dotfile, not source-controlled.

---

## Step 9: Privacy & data handling

- **PII in prompts.** Are user PII fields ever embedded in prompts that go
  to LLM providers? If yes:
  - DPA with the provider in place?
  - Provider's data-retention setting (Anthropic / OpenAI offer "no
    training" options for API customers)?
  - Disclosure to end users in privacy policy?
- **Cross-tenant prompt leakage.** Tenant A's data should never appear in
  Tenant B's RAG retrieval. Verify metadata filtering is enforced
  server-side.
- **Logging.** Prompts + completions logged to your own systems. Logs
  contain everything sent to the model — treat them like database PII.
- **Right to erasure.** When a user requests data deletion, are their
  prompt/completion logs also deleted?

---

## Step 10: Operational

- **Latency monitoring.** P50 / P95 / P99 per endpoint. LLMs vary; track
  the distribution.
- **Error rate monitoring.** Provider 5xx, 429, content-policy refusals.
- **Token usage monitoring.** Input / output / cached tokens per call,
  trended.
- **Provider outage handling.** Fallback or graceful degradation when
  Anthropic / OpenAI / Bedrock has an outage.
- **`.env` for API keys.** Never committed.

---

## Red flags

- User input directly concatenated into prompts with no delimiter
  discipline.
- Model output executed as code / SQL / shell command without validation.
- `JSON.parse(modelOutput)` with no schema validation.
- Prompts sent to LLM containing user PII without DPA / provider opt-out
  config.
- Opus/GPT-4 in a hot path where Haiku/4o-mini would suffice
  (cross-ref module `05-operations/cost.md`).
- No prompt caching despite stable system prompts > 1024 tokens.
- No eval set; "we'll know if it's broken when users complain."
- Model alias used without commitment to track behavior changes.
- Agent loops with no max-iteration cap.
- Cross-tenant RAG retrieval without metadata filtering.

---

## Output structure

```markdown
# AI / ML Audit

Date: YYYY-MM-DD
Repository commit: <hash>

## Inventory
| Call site | Model | Type | Stream/Batch | Source of prompt | Est cost / call |

## Prompt injection defenses
- Input segregation
- Indirect-injection awareness (untrusted fetched content)
- Tool-call argument validation
- Output execution discipline

## Output validation
- Structured-output mode used
- Schema validation
- Refusal handling
- Hallucination guards (RAG citations, ground-truth checks)

## Model versioning
- Pinned IDs vs aliases
- Behavior-change test suite
- Fallback chain

## Cost guardrails
(cross-ref cost audit)
- Per-user/tenant rate limits
- Token budgets
- Prompt caching enabled
- Batch API used where applicable
- Spend monitoring

## Evals
- Eval set
- Auto-grading
- Coverage of edge cases
- Run cadence
- Comparative across model swaps

## RAG (if applicable)
- Indexing pipeline
- Hybrid search
- Metadata filtering (tenant isolation)
- Reranking
- Citations
- Index freshness

## Agents (if applicable)
- Tool definitions
- Loop bounds
- Side-effect authorization
- Memory bounds

## Agent execution surface (if applicable)
- Tool / MCP server inventory (trust tier per server)
- Tool description audit + sanitization
- Indirect-injection escalation paths (fetched content → state-mutating tool)
- Per-tool permission scoping + sandboxing
- Audit log of tool calls
- Preflight hook coverage
- Lateral surface (subagents, runtime tool registration, shell tools)
- Model-emitted code execution paths
- Harness config under code review
- Mapped to current OWASP LLM Top 10 IDs

## Privacy & data handling
(cross-ref privacy audit)
- PII in prompts (DPA / opt-out)
- Cross-tenant isolation
- Logging discipline
- Erasure including LLM logs

## Operations
- Latency / error / token monitoring
- Provider outage fallback

## Findings

## Final recommendation
Single highest-priority AI/ML action.

## Decision summary
- Prompt injection defenses present: yes/no
- Output validation present: yes/no
- Evals present: yes/no
- Worst gap: <description>

## Confidence
High / Medium / Low

## Out of Scope / Inconclusive
```

---

## Severity calibration

- **Critical:** prompt-injection-vulnerable user-facing flow that touches
  data or tools; PII shipped to provider with no DPA / opt-out; cross-
  tenant RAG leakage.
- **High:** no evals on a customer-facing AI feature; no output validation
  before consuming; runaway-cost potential (no rate limits / token caps).
- **Medium:** model aliases without behavior-tracking; no prompt caching
  on stable prefixes; verbose logging without scrubbing.
- **Low:** model overkill for the task (cross-ref cost); minor prompt
  hygiene.

Every finding cites the call site or absent-evidence pointer.
