# 02 — Architecture

## System diagram

```
            ┌─────────────────────────────────────────────────────────┐
            │                       Clients                            │
            │  (OpenAI SDK, curl, langchain, editor plugins, ...)    │
            └───────────────────────────┬─────────────────────────────┘
                                        │  HTTPS
                                        │  Authorization: Bearer rr_live_...
                                        ▼
            ┌─────────────────────────────────────────────────────────┐
            │                  railroad (single Go binary)            │
            │                                                         │
            │  ┌───────────────────────────────────────────────────┐  │
            │  │             HTTP layer (net/http + chi)            │  │
            │  │  /v1/chat/completions   /v1/models   /healthz      │  │
            │  └────────────────────┬──────────────────────────────┘  │
            │                       │                                  │
            │  ┌────────────────────▼──────────────────────────────┐  │
            │  │              Middleware stack                      │  │
            │  │  request-id  →  auth (api key)  →  rate  →  log    │  │
            │  └────────────────────┬──────────────────────────────┘  │
            │                       │                                  │
            │  ┌────────────────────▼──────────────────────────────┐  │
            │  │         Request context (resolved at auth)         │  │
            │  │  { api_key, project, account, configuration,       │  │
            │  │    modelset, model }                                │  │
            │  └────────────────────┬──────────────────────────────┘  │
            │                       │                                  │
            │  ┌────────────────────▼──────────────────────────────┐  │
            │  │              OpenAI handler                        │  │
            │  │  - decode ChatCompletionRequest                     │  │
            │  │  - modelset resolver picks the candidate           │  │
            │  │  - build provider request from OpenAI + config    │  │
            │  │  - call provider (sync or stream)                  │  │
            │  │  - translate provider response to OpenAI shape     │  │
            │  └────────────────────┬──────────────────────────────┘  │
            │                       │                                  │
            │  ┌────────────────────▼──────────────────────────────┐  │
            │  │           Modelset resolver (internal/modelset)    │  │
            │  │  - compile CEL expression (cached, per modelset)   │  │
            │  │  - evaluate CEL on (request, project, candidates)  │  │
            │  │  - return chosen candidate                          │  │
            │  │  - (future: ADK-backed resolver for advanced cases)│  │
            │  └────────────────────┬──────────────────────────────┘  │
            │                       │                                  │
            │  ┌────────────────────▼──────────────────────────────┐  │
            │  │             Provider registry                      │  │
            │  │  interface { Chat(ctx, req) (Stream, error) }       │  │
            │  │   ├─ gemini  (google.golang.org/genai)              │  │
            │  │   └─ (future: openai, anthropic, ...)              │  │
            │  └────────────────────┬──────────────────────────────┘  │
            │                       │                                  │
            │  ┌────────────────────▼──────────────────────────────┐  │
            │  │           Store layer                              │  │
            │  │  - postgres (pgx pool) — durable state              │  │
            │  │  - valkey (rueidis)  — cache, rate-limit counters   │  │
            │  └───────────────────────────────────────────────────┘  │
            │                                                         │
            └─────────────┬───────────────────────────┬───────────────┘
                          │                           │
                          ▼                           ▼
            ┌─────────────────────────┐   ┌─────────────────────────────┐
            │  Postgres (16+)         │   │  Valkey (7.2+)              │
            │  - tenants, keys, cfg   │   │  - api_key → project cache  │
            │  - modelsets, candidates│   │  - rate-limit counters      │
            │  - request logs         │   │  - response cache (future)  │
            └─────────────────────────┘   └─────────────────────────────┘
                          │
                          ▼
            ┌─────────────────────────────────────────────────────────┐
            │  Upstream providers                                      │
            │  - Gemini Developer API (generativelanguage.googleapis) │
            │  - (future: OpenAI, Anthropic, ...)                      │
            └─────────────────────────────────────────────────────────┘
```

Single binary, single process. No microservices. No message queue. The
async / streaming path is handled in-process by the provider client.

The request path is a clean passthrough: there is no agent, no
session, and no memory in the critical path. The only logic between
the OpenAI handler and the upstream provider is a single CEL
expression that picks which model to call.

## Request flow — `POST /v1/chat/completions`

```
Client                railroad                              Gemini
  │                      │                                    │
  │  POST /v1/chat/      │                                    │
  │  completions         │                                    │
  │  Authorization:      │                                    │
  │   Bearer rr_live_..  │                                    │
  │  {model, messages,   │                                    │
  │   stream: false}     │                                    │
  ├─────────────────────►│                                    │
  │                      │  1. middleware: request-id         │
  │                      │  2. middleware: auth               │
  │                      │     - parse "Bearer ..."           │
  │                      │     - look up key in valkey         │
  │                      │     - on miss, hit postgres         │
  │                      │     - resolve project + config +   │
  │                      │       modelset + candidates         │
  │                      │  3. handler: decode OpenAI req      │
  │                      │  4. handler: build Environment      │
  │                      │     - request, project, key,        │
  │                      │       candidates, now               │
  │                      │  5. modelset.Resolve(env)           │
  │                      │     - filter eligible candidates    │
  │                      │     - evaluate CEL (or no-op)       │
  │                      │     - return chosen candidate      │
  │                      │  6. provider lookup                 │
  │                      │     - candidate.Model.Provider      │
  │                      │  7. build provider request          │
  │                      │     - map OpenAI fields → provider  │
  │                      │     - apply config defaults         │
  │                      │     - prepend system_prompt         │
  │                      │  8. provider.Chat(ctx, req)         │
  │                      │     (or StreamChat)                 │
  │                      ├───────────────────────────────────►│
  │                      │                                    │
  │                      │◄───────────────────────────────────┤
  │                      │     genai.GenerateContentResponse  │
  │                      │  9. translate → OpenAI shape       │
  │                      │ 10. record request_log              │
  │                      │     - candidates chosen, latency,   │
  │                      │       tokens, trace_id              │
  │  HTTP 200            │                                    │
  │  {id, choices[],     │                                    │
  │   usage{...}}        │                                    │
  │◄─────────────────────┤                                    │
  │                      │                                    │
```

For `stream: true`, the handler keeps an open SSE connection and
emits `data: {chunk}\n\n` for each provider event, ending with
`data: [DONE]\n\n`. The finish_reason and usage (in the final chunk)
match OpenAI's format.

The routing decision (step 5) is single-digit microseconds. The
provider call (step 8) dominates the latency.

## Process model

- One Go binary, `railroad`.
- It runs an HTTP server on a configurable port (default `:8080`).
- It owns two connection pools: pgxpool to Postgres, valkey client to
  Valkey.
- It does **not** run a separate background worker. Long-running
  work (request log rollups, model registry refresh) is deferred to
  a later phase. The MVP runs request-driven only.
- The CEL programs are compiled lazily on first use of each
  modelset. They live in a per-process `sync.Map` keyed on
  `modelset_id`. Restart = recompile (cheap).
- Graceful shutdown on SIGINT / SIGTERM: stop accepting new requests,
  drain in-flight up to a deadline, close pools.

## Configuration sources

Configuration is loaded in this order, with later sources overriding
earlier ones:

1. Compiled-in defaults (in `internal/config/defaults.go`).
2. Environment variables prefixed `RAILROAD_`.
3. A TOML or YAML config file at `$RAILROAD_CONFIG_FILE` if set.
4. The database — for *runtime* configuration (e.g. the model
   registry). The application process does not read tenant data from
   config files.

Sensitive values (Gemini API key, future per-project provider
credentials) are loaded from environment variables, not config files.
Future work may load them from a secrets backend.

## HTTP layer

- Standard library `net/http` with the Go 1.22+ pattern-routing
  syntax (`mux.Handle("POST /v1/chat/completions", h)`). This avoids
  pulling in a router dependency for the public API.
- `github.com/go-chi/chi/v5` for the admin / internal routes where we
  need middleware composition. **DECIDE:** we can also do everything
  with stdlib; chi is on the table only if middleware composition
  (e.g. per-route timeouts, body limits, audit logging) becomes
  awkward. If we don't add it, drop the dependency.
- One `http.Server` with explicit timeouts:
  - `ReadHeaderTimeout`: 5s
  - `ReadTimeout`: 60s
  - `WriteTimeout`: 0 (streaming); we set per-request deadlines
    instead
  - `IdleTimeout`: 120s
- TLS termination is the deployer's job. We bind plain HTTP.

## Middleware

| Order | Middleware | Responsibility |
|------:|-----------|----------------|
| 1 | `requestID` | Mint a `X-Request-ID` if absent, attach to context and logs |
| 2 | `realIP` | Resolve client IP from `X-Forwarded-For` if behind a proxy |
| 3 | `accessLog` | Structured log line on response |
| 4 | `auth` | Parse `Authorization: Bearer rr_...`, resolve key → project → configuration → modelset → candidates, inject into context. 401 on failure. |
| 5 | `rateLimit` | Per-key token-bucket counter in Valkey. 429 on exhaustion. **Future**: per-project or per-tenant tiers. |
| 6 | `recover` | Convert panics to 500 with a stable error code. |
| 7 | (handler) | The actual endpoint |

The auth middleware is the most important and is detailed in
[07-auth-and-tenancy](./07-auth-and-tenancy.md).

## Caching strategy

| Data | Where | TTL | Notes |
|---|---|---|---|
| `api_key` → `project_id, configuration_id` | Valkey | 60s | Hot path; saves a Postgres round trip |
| `project` | Valkey | 5 min | Rarely changes |
| `configuration` (with denormalized modelset_id) | Valkey | 5 min | Re-resolved on key lookup |
| `modelset` + candidates | Valkey | 5 min | The CEL source is in here; compiled program lives in process memory |
| `model` (registry row) | Valkey | 10 min | Static; refreshed on registry change via invalidation |
| Upstream `models.list` (Gemini) | Valkey | 1 hr | Avoid hitting the provider on every `GET /v1/models` |
| CEL compiled program | in-process | n/a | `sync.Map` keyed on `modelset_id`; cleared on process restart |
| Per-request response cache | Valkey | n/a | **Future**, not MVP |

All cache keys are namespaced: `rr:apikey:<sha256(key)[:16]>`,
`rr:project:<id>`, `rr:cfg:<id>`, `rr:ms:<id>`, etc.

## Concurrency model

- One goroutine per in-flight request. Go's net/http handles this.
- Streaming responses use the same goroutine and write to the
  `http.ResponseWriter` directly. Backpressure from a slow client
  naturally propagates.
- No goroutines are spawned per request beyond what the standard
  library does. The MVP does not need a worker pool.
- Database connections are pooled via pgx. The pool size is
  configurable, default 25.
- Valkey connections are managed by rueidis (auto-pipelining + a
  small connection pool per node).
- CEL programs are read-only after compile; the `sync.Map` is
  lock-free for reads. Compiles happen at most once per modelset
  per process.

## Out-of-process concerns

These are not part of the MVP binary, but the design leaves room for
them:

- A future **admin** binary (or subcommand) that talks directly to
  Postgres for tenant management.
- A future **evals** runner that replays recorded request/response
  pairs against new model versions to detect regressions.
- A future **reaper** that prunes old request logs and rotates
  API keys.
- A future **CEL playground** that lets operators test a CEL
  expression against a fake request without making a real call.
