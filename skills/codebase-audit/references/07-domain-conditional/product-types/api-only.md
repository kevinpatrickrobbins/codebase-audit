# Product-Type Sub-Reference: API-Only Service

Audit a backend API service with no frontend (REST, GraphQL, RPC, gRPC).

---

## Detection

- Server framework: Express / Fastify / Hono / NestJS / Koa / FastAPI / Flask
  / Django REST Framework / Rails API / Spring Boot / ASP.NET Core / Gin /
  Echo / Axum / Actix
- No view layer / no static-output rendering / no client bundle
- Routes that return JSON / Protobuf, not HTML
- Often: OpenAPI spec, Postman collection, Swagger UI

---

## Investigate

### API style

- **REST** — most common. Verify resource-oriented URLs, correct verb usage
  (GET safe & idempotent; POST creates; PUT/PATCH/DELETE).
- **GraphQL** — schema-first or code-first; typed; single endpoint.
- **gRPC** — Protobuf schema; binary; service-to-service typical.
- **tRPC** — TS-only, typed end-to-end with client.
- **JSON-RPC** — older but still seen.

Mixing styles in one service usually warrants a "why?" note.

### Schema & contract

- **OpenAPI / Swagger spec** maintained alongside code? Generated from code
  (TypeBox, Zod-to-OpenAPI, FastAPI auto-gen, NestJS swagger module) or
  hand-written (drifts)?
- **GraphQL schema** versioned and validated in CI?
- **gRPC `.proto`** files in a shared `proto/` repo or vendored?
- **Breaking-change detection** in CI — `swagger-diff`, `openapi-diff`,
  `buf breaking`.

### Versioning

- **URL versioning** (`/v1/`, `/v2/`) — easy to reason about; explicit.
- **Header versioning** (`Accept: application/vnd.api+json;version=2`) —
  cleaner URLs; harder for clients.
- **No versioning** — fine for an internal API or pre-launch; risky for a
  published API.
- **Deprecation policy** documented (timeline, headers, sunset dates).

### Authentication

- **API keys** — per-customer, rotatable, scoped; not shared globally.
- **OAuth 2.0 / OIDC** — for user-on-behalf flows; PKCE for public clients.
- **JWT / opaque tokens** — verification on every request.
- **mTLS** — for service-to-service in regulated contexts.
- **Webhook signatures** (HMAC) for inbound webhooks from third parties.

### Authorization

Same concerns as Module 02 + SaaS sub-reference if multi-tenant. Specific to
API-only:

- **Per-endpoint scope check.** Tokens carry scopes (`read:users`,
  `write:billing`); endpoints assert required scope.
- **Object-level authorization on every fetch-by-id.** No IDOR.
- **Field-level authorization.** Some fields visible only to higher tiers.

### Rate limiting

- **Per-key / per-user / per-endpoint** — not just global.
- **Standard headers** (`X-RateLimit-Limit`, `X-RateLimit-Remaining`,
  `X-RateLimit-Reset`, or RFC 9239 `RateLimit` header).
- **`429 Too Many Requests`** with `Retry-After`.
- **Burst vs sustained** — token bucket vs fixed window.
- **Cost-weighted** — expensive endpoints cost more "tokens" than cheap.

### Idempotency

- **`Idempotency-Key` header** support on POST / PATCH endpoints that
  create or mutate non-idempotent resources (Stripe-style).
- **Storage** of keys (often Redis with TTL); collision behavior documented.
- **DELETE** is idempotent by spec — verify second DELETE returns 404 or
  204, not 500.

### Errors

- **Consistent error envelope.** Either bare HTTP status + JSON `{ error,
  message, details }`, or [RFC 9457 Problem Details](https://datatracker.ietf.org/doc/html/rfc9457).
  Mixed shapes across endpoints make client error handling miserable.
- **Status codes used correctly.** 200 for success, 400 for client error
  with body, 401 for unauthenticated, 403 for unauthorized, 404 for
  not-found, 409 for conflict, 422 for validation, 429 for rate-limited,
  500 for server. Avoid `200 { "error": "..." }` (a common JSON-API
  anti-pattern).
- **Error codes** machine-readable (`USER_NOT_FOUND`) in addition to
  human-readable messages.

### Pagination

- **Cursor-based** preferred for large datasets / mutable order — avoids
  duplicate / skipped items on insert.
- **Offset-based** OK for small / static datasets.
- **`Link` header** (RFC 5988) or pagination object in body — consistent
  across endpoints.
- **Default + max page size** enforced server-side.

### Filtering, sorting, sparse fieldsets

- **Filter syntax** consistent — query string parameters or GraphQL args.
- **Sort syntax** consistent (`?sort=-created_at,name`).
- **Sparse fieldsets** (`?fields=id,name`) for bandwidth-sensitive clients.
- **Embedded resources** (`?expand=user,billing`) vs N+1 round trips.

### Webhooks (outbound)

- **Signed webhooks** — HMAC with shared secret; `X-Signature` header;
  documented verification process.
- **Retries** — exponential backoff with cap; document max-attempts.
- **Idempotency on receiver side** — webhook ID for dedup.
- **Replay capability** — admin-triggered webhook redelivery for failed
  events.

### Documentation

- **OpenAPI / GraphQL playground / docs site.** Hosted Swagger UI,
  Stoplight, Redoc, Mintlify, etc.
- **Examples in docs** — request + response for every endpoint.
- **Authentication setup** explained.
- **SDK** — generated or hand-written, kept in sync.

### Performance

(See Module 09 for DB / scaling.)

- **N+1 queries** in expand / include flows.
- **Streaming responses** (`Transfer-Encoding: chunked`) for large
  payloads.
- **Compression** (gzip / br) — enabled at framework or proxy level.
- **HTTP/2 or HTTP/3** support at the edge / reverse proxy.

### Operability

- **Health endpoint** that actually checks dependencies (DB ping, queue
  ping, downstream API reachability) — not unconditional 200.
- **Readiness vs liveness.** `/healthz` (alive) vs `/readyz` (ready to
  serve traffic) distinct in container deploys.
- **Request ID** propagated through logs (`X-Request-Id` header).
- **Structured logging** with the request ID, route, status, duration.
- **Tracing** — OpenTelemetry / Datadog / Honeycomb spans across services.

### Security

(See Module 02 in detail.)

- **CORS** — only origins that should access the API; not `*` for credentialed
  endpoints.
- **CSRF** — token-bearing API can be CSRF-immune; cookie-bearing API needs
  CSRF defenses.
- **Input validation** — schema validation library on every input
  (Zod, Joi, Pydantic, JSON Schema).
- **Output filtering** — response shaping to never leak fields the caller
  shouldn't see (per-tier auth).

---

## Red flags

- No OpenAPI / GraphQL schema in code or docs.
- Versioning absent for a published API.
- Inconsistent error envelopes across endpoints.
- `200 { "error": "..." }` patterns.
- Global rate limit only (one abusive customer crashes everyone).
- No idempotency-key support on POST endpoints that create.
- No request ID in logs.
- Health endpoint always returns 200.
- Webhook signing absent for an API that delivers webhooks.
- `LIMIT 100` hardcoded as max page size in some endpoints, missing in
  others.
- Default page size of 1000 (sets a poor expectation that every consumer
  pays for).

---

## Output section

```markdown
### API-Only Service

#### API style
- REST / GraphQL / gRPC / tRPC / RPC
- Mixing: yes/no — rationale

#### Schema & contract
- OpenAPI / GraphQL / .proto: present? generated from code? versioned?
- Breaking-change detection in CI: yes/no

#### Versioning
- Strategy: URL / header / none
- Deprecation policy

#### Authentication
- API keys / OAuth / JWT / mTLS / signed webhooks (inbound)

#### Authorization
- Per-endpoint scope checks
- Object-level (IDOR safety)
- Field-level (per-tier filtering)

#### Rate limiting
- Per-key / per-endpoint / global
- Headers
- Cost-weighted

#### Idempotency
- Idempotency-Key support

#### Errors
- Envelope consistency
- Status code discipline
- Machine-readable codes

#### Pagination, filtering, sorting
- Cursor / offset
- Default + max page size enforced

#### Webhooks (outbound)
- Signing / retries / replay

#### Documentation
- Hosted docs
- Examples per endpoint
- SDK status

#### Performance
- N+1 in expand flows
- Compression / streaming
- HTTP/2 or 3

#### Operability
- Health / readiness / liveness
- Request ID propagation
- Structured logging
- Tracing

#### Security
- CORS / CSRF / input validation / output filtering

#### Findings
(table)
```
