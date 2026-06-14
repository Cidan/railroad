# 12 — Roadmap

This is the phased plan from an empty repo to a working MVP, with
the next round of features sketched out so we don't forget them.
Phases 0–5 land the MVP. Phase 6+ is post-MVP, listed in rough
priority order but not committed to in detail.

Each phase ends with a **done** criterion: something we can show,
test, or measure. The intent is to keep each phase small enough
to land in a few days of focused work, with a mergeable PR at
the end of each.

The MVP request path is a **clean passthrough with a modelset
resolver**. There is no ADK, no session persistence, no memory
injection, and no tool calling in the request path. We may add
ADK as a future modelset implementation — the seam is in place
— but it is not part of the MVP.

## Phase 0 — Repo skeleton (1 day)

**Goal**: a `go build` that produces a binary, a `make run` that
boots a process, and a `make test` that runs a single passing
test.

Tasks:
- `go mod init github.com/Cidan/railroad`
- `cmd/railroad/main.go` with `--version` and `--addr` flags, no
  routes yet
- `internal/config/config.go` loading from env
- `internal/observability/logger.go` (slog wrapper)
- `internal/server/server.go` with one route: `GET /healthz` that
  returns 200 OK
- `Makefile` with `build`, `test`, `run`, `lint`, `tidy`
- `docker-compose.yml` with Postgres + Valkey
- `.env.example`, `.gitignore`, `README.md`, `LICENSE`
- `.golangci.yml` with a reasonable baseline
- `Dockerfile` (multi-stage, scratch)

**Done**: `make run` brings up the binary, `curl /healthz` returns
200, `make test` is green, `make docker` produces a runnable image.

## Phase 1 — Data model and migrations (2 days)

**Goal**: the Postgres schema exists, migrations apply cleanly,
and the model registry has the two seed models.

Tasks:
- `migrations/000001_init.sql` with all tables from
  [03-data-model](./03-data-model.md)
- `migrations/000002_seed_models.sql` with `gemini-2.5-pro` and
  `gemini-2.5-flash`
- `internal/store/postgres/pool.go` — pgxpool constructor
- `internal/store/postgres/migrate.go` — goose-based migration
  runner, exposed as `railroad migrate ...`
- `internal/store/postgres/valkey.go` — valkey-go client
  constructor
- `internal/store/postgres/health.go` — `/readyz` checks

**Done**: `railroad migrate up` applies both migrations;
`psql` shows the tables; `/readyz` returns 200 with both deps
green.

## Phase 2 — API key resolution and stub auth (2 days)

**Goal**: an authenticated `/v1/chat/completions` that 401s on a
bad header and 200s on any well-formed key in stub mode, with the
full resolution chain in place.

Tasks:
- `internal/auth/key.go` — key generation and parsing
- `internal/auth/hash.go` — argon2id wrapper, plus the stub
  short-circuit
- `internal/auth/lookup.go` — Valkey-fronted lookup with PG
  fallback
- `internal/domain/*.go` — Go types for `Account`, `Project`,
  `APIKey`, `Configuration`, `Modelset`, `Candidate`, `Model`
- `internal/store/postgres/api_key.go` — repository
- `internal/store/postgres/project.go`, `configuration.go`,
  `modelset.go`, `model.go`
- `internal/api/middleware/auth.go` — bearer parsing, key
  resolution, `Resolved` injection
- `internal/api/middleware/request_id.go`, `access_log.go`,
  `recover.go`
- A `seed/dev.sql` that creates one account, one project, one
  modelset (with one candidate), one configuration, one
  api_key (prefix `rr_live_DEV0000000000`)

**Done**: `curl -H 'Authorization: Bearer rr_live_DEV0...'` is
accepted; missing or malformed header is 401; resolved `Project`,
`Configuration`, and `Modelset` are visible in logs at DEBUG.

## Phase 3 — Provider abstraction and Gemini (3 days)

**Goal**: a `providers.Provider` interface with a working Gemini
impl, called directly from a unit-test handler.

Tasks:
- `internal/providers/provider.go` — the interface from
  [05-providers](./05-providers.md)
- `internal/providers/registry.go` — `NewRegistry`
- `internal/providers/gemini/client.go` — `genai` wrapper
- `internal/providers/gemini/translate.go` — request/response
  translation
- `internal/providers/gemini/translate_test.go` — table-driven
  tests with recorded fixtures
- `internal/providers/gemini/errors.go` — `mapErr`

**Done**: a unit test can call
`geminiClient.Chat(ctx, ChatRequest{...})` against the live API
with a real `GEMINI_API_KEY` and assert on the translated
response. Translation tests pass with golden files.

## Phase 4 — Chat completions endpoint (4 days)

**Goal**: `POST /v1/chat/completions` works end-to-end against
Gemini, both sync and streaming, with the OpenAI wire format.

Tasks:
- `internal/api/openai/types.go` — request/response structs
  (from [04-api-surface](./04-api-surface.md))
- `internal/api/openai/chat.go` — sync handler
- `internal/api/openai/chat_stream.go` — SSE handler
- `internal/api/openai/models.go` — `/v1/models`
- `internal/api/openai/translate.go` — OpenAI ↔ provider
  translation (no modelset yet; pick the first candidate)
- Integration test: a recorded HTTP exchange through the
  handler, asserting the OpenAI response shape

**Done**: an OpenAI SDK call against `http://localhost:8080/v1/`
returns a correct response. The streaming path produces
byte-identical SSE output to a reference client. The integration
test runs in CI against a recorded fixture.

## Phase 5 — Modelset resolver + CEL (3 days)

**Goal**: a configuration can reference a modelset with multiple
candidates and a CEL expression, and the resolver picks the
right candidate per request.

Tasks:
- `internal/modelset/resolver.go` — the `Resolver` interface and
  the default CEL implementation
- `internal/modelset/cel.go` — env, type declarations, helpers
- `internal/modelset/cache.go` — `sync.Map` of compiled programs
  keyed on `modelset_id`
- `internal/store/postgres/modelset.go` — repository
- Update the chat handler to call `resolver.Resolve(env)`
- Update `request_log` to record `modelset_id` and
  `candidate_position`
- A test fixture: two candidates (Pro, Flash), a CEL rule
  that picks Pro for long messages, Flash otherwise
- A test that the rule is enforced end-to-end
- Admin endpoint to create / update modelsets, with CEL
  compile at write time

**Done**: with the seeded modelset, a request whose last
message is >4000 chars routes to Pro, others to Flash. The
`request_log` row records which one was chosen. Updating the
CEL in the admin endpoint is rejected on a compile error and
accepted on success; the next request uses the new rule.

## Phase 6 — Observability hardening (2 days)

**Goal**: traces, metrics, and access logs are correct and
useful.

Tasks:
- `internal/observability/tracer.go` — OTel tracer provider
- `internal/observability/metrics.go` — Prometheus registry
- `internal/observability/middleware.go` — access log,
  request id, modelset-resolve-span emission
- `internal/observability/middleware_test.go`
- A Grafana dashboard JSON in `deploy/observability/`
- A runbook in `RUNBOOK.md` with the suggested alerts

**Done**: a load test produces traces with the expected
hierarchy (auth → handler → modelset.resolve →
provider.<name>.chat), metrics with the expected labels
(including `modelset_id`), and access logs with the
expected fields. The runbook is reviewable.

## Phase 7 — Hardening (3 days, after MVP)

**Goal**: the MVP is safe to run in a semi-public environment.

Tasks:
- Rate limiting per API key (Valkey token bucket)
- Request body size limits
- Per-request deadlines (sync and streaming)
- Panic recovery
- Slowloris protection (read header timeouts)
- Upstream retries with backoff
- Graceful shutdown (drain in-flight, close pools)
- A small set of property-based tests for the request ↔
  response translation
- A load test script that hits `/v1/chat/completions` at
  100 RPS for 1 minute

**Done**: the load test runs without errors; a forced panic in
a handler returns 500; an upstream timeout returns 504.

## Phase 8 — Internal admin API (2 days, after MVP)

**Goal**: an operator can create accounts, projects,
configurations, modelsets, and keys via HTTP rather than SQL.

Tasks:
- `internal/api/internal/projects.go`, `accounts.go`,
  `api-keys.go`, `configurations.go`, `modelsets.go`, `usage.go`
- Admin auth middleware (token check)
- Internal server bound to a separate port

**Done**: the smoke test in [11-deployment](./11-deployment.md)
runs end to end via curl against the internal API.

## Phase 9 — Real auth (1 day, after MVP)

**Goal**: switch `RAILROAD_AUTH_MODE` to `verify` and the
existing flow works.

Tasks:
- Fix any edge cases in the argon2id implementation
- Add timing-attack-safe comparisons
- Document the env var to flip
- An integration test that exercises both stub and verify modes

**Done**: `RAILROAD_AUTH_MODE=verify` in a fresh deployment
accepts a real key and rejects a fake one.

## Phase 10+ — Future work (not committed)

Listed in rough order. The details for each will get their own
plan when we get to them.

- **ADK-backed modelset implementation.** A future
  `internal/modelset/adk.go` builds an `LLMAgent` from the
  candidates and runs it. Used for judge/jury, multi-step
  agents, structured tool use. Same `Resolver` interface, so
  the handler is unchanged. The data model is unchanged. New
  `modelset.implementation` enum if we want to dispatch on
  data, or just a different resolver chosen at startup.
- **Additional providers.** OpenAI, Anthropic, Mistral, vLLM.
  The interface is in place; the work is per-provider
  translation tests and live calls.
- **Tool calling.** Surface OpenAI `tools` / `tool_choice` via
  the provider abstraction. Requires careful finish-reason
  and tool-call-id translation. Probably a
  `tool_aware_modelset` implementation.
- **Memory sets as a modelset-time feature.** Re-introduce
  `memory_set` as a config field and a CEL variable
  `memories`; the handler injects them as a system-prompt
  prefix. Schema change is contained; the resolver interface
  is unchanged.
- **Embeddings endpoint.** `POST /v1/embeddings`. Reuses the
  pgvector plumbing if memory is re-introduced; otherwise
  straight through the provider.
- **Responses API.** `POST /v1/responses` (OpenAI's newer
  shape).
- **Per-tenant rate limits.** Currently global per-key; add
  per-project tiers.
- **Cost budgets.** Per-configuration monthly token cap.
- **Response caching.** Identical requests within a TTL return
  the cached response. Privacy implications: opt-in per
  configuration.
- **Audit log.** Admin operations get a separate audit trail.
- **Per-tenant encryption at rest** for the
  `provider_credential` table.
- **Multi-region active-active.** Read replicas, Valkey
  cluster, careful session handling.
- **A small web UI** for managing tenants, keys, and
  configurations. (Out of product scope; not MVP.)
- **A CEL playground** for testing routing rules against
  fake requests without making real calls.

## Risks and how we handle them

- **CEL correctness.** Routing is now driven by a user-provided
  string. A buggy expression could send traffic to the wrong
  model, or fail open in surprising ways. Mitigation: compile
  at write time (reject on syntax/type errors), fail open to
  the first eligible candidate at runtime, and ship a
  playground for safe testing.
- **Provider credit cost.** A bug that loops requests could
  cost real money. Mitigation: per-key rate limits (phase 7),
  per-configuration cost budgets (future), alerting on
  `chat_completion_tokens_total` rate.
- **Privacy / PII in prompts.** Customers will pass sensitive
  data. We default to **not** storing bodies; we offer an
  opt-in for debugging.
- **Streaming wire-format drift.** OpenAI changes shape. We
  maintain a conformance test that diffs our SSE output
  against OpenAI's spec on every release.
- **ADK shim drift (future).** When we add ADK as a modelset
  implementation, ADK-go will still be a fast-moving API.
  Same mitigation as the previous plan: pin the minor version,
  read the changelog on every bump, keep ADK imports local.
