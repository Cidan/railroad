# 10 — Observability

Railroad is meant to be debugged from production data. The MVP
invests heavily here: every request gets a trace id, structured
logs at the access and application layers, Prometheus metrics,
and OTel traces. We do not ship features that cannot be
understood post-hoc.

## Log lines (structured)

We use the standard library's `log/slog` with a JSON handler.
Every log line is a single JSON object on stdout.

### Required fields on every line

| field | type | source |
|---|---|---|
| `ts` | RFC 3339 with nanos | slog |
| `level` | `debug`/`info`/`warn`/`error` | slog |
| `msg` | string | slog |
| `service` | `railroad` | static |
| `version` | `v0.1.0-abc1234` | build flag |
| `request_id` | uuid | middleware (or "no-request" for startup lines) |
| `trace_id` | 32-char hex | OTel |

Optional context fields, added by the layer that has them:

- `project_id`, `api_key_id`, `account_id` (after auth)
- `model_id` (after config resolution)
- `latency_ms`, `status_code` (access log)
- `prompt_tokens`, `completion_tokens`, `cached_tokens` (after
  provider response)

### Access log

One line per request, on response:

```json
{
  "ts": "2026-06-13T10:21:33.412Z",
  "level": "info",
  "msg": "request",
  "service": "railroad",
  "version": "v0.1.0-abc1234",
  "request_id": "5e3a...",
  "trace_id": "1a2b...",
  "method": "POST",
  "path": "/v1/chat/completions",
  "status": 200,
  "bytes": 842,
  "latency_ms": 1240,
  "routing_latency_ms": 0.4,
  "client_ip": "203.0.113.42",
  "project_id": "prj_...",
  "api_key_id": "key_...",
  "modelset_id": "mset_...",
  "candidate_position": 1,
  "resolved_model_id": "google/gemini-2.5-flash",
  "stream": false,
  "prompt_tokens": 124,
  "completion_tokens": 76,
  "cached_tokens": 0
}
```

The access log is the only place that records token counts and
client IP. Application log lines are noise-filtered.

### Error log

Internal errors get an additional log line at ERROR with the
full stack:

```json
{
  "ts": "...", "level": "error", "msg": "upstream_error",
  "request_id": "...", "trace_id": "...",
  "error.kind": "*genai.APIError",
  "error.message": "rate limit exceeded",
  "error.stack": "..."
}
```

The `request_id` ties the access log to the error log.

### What we do **not** log

- Raw request and response bodies by default. The OpenAI spec
  has privacy implications (PII in prompts). Enable with
  `RAILROAD_LOG_REQUEST_BODIES=true` for debugging, which writes
  bodies to the `request_log_body` table.
- API keys, in any form. Never.
- Provider credentials. Never.
- Full message content, even metadata-only mode. We log token
  counts, not text.

## Metrics (Prometheus)

Exposed at `GET /metrics` in standard text format.

### HTTP metrics

| metric | type | labels |
|---|---|---|
| `http_requests_total` | counter | `method`, `path`, `status` |
| `http_request_duration_seconds` | histogram | `method`, `path`, `status` |
| `http_requests_in_flight` | gauge | `method`, `path` |
| `http_request_size_bytes` | histogram | `method`, `path` |
| `http_response_size_bytes` | histogram | `method`, `path` |

### Chat metrics

| metric | type | labels |
|---|---|---|
| `chat_completions_total` | counter | `modelset_id`, `provider`, `resolved_model_id`, `stream`, `finish_reason`, `status` |
| `chat_completion_duration_seconds` | histogram | `modelset_id`, `provider`, `resolved_model_id`, `stream`, `status` |
| `chat_completion_tokens_total` | counter | `modelset_id`, `provider`, `resolved_model_id`, `kind` (`prompt`/`completion`/`cached`/`reasoning`) |
| `chat_completion_time_to_first_token_seconds` | histogram | `modelset_id`, `provider`, `resolved_model_id`, `stream` |
| `chat_completion_stream_chunks_total` | counter | `modelset_id`, `provider`, `resolved_model_id` |

### Modelset metrics

| metric | type | labels |
|---|---|---|
| `modelset_resolution_total` | counter | `modelset_id`, `outcome` (`ok`/`no_cel`/`eval_error`/`bad_result`/`no_eligible`) |
| `modelset_resolution_duration_seconds` | histogram | `modelset_id`, `outcome` |
| `modelset_candidate_chosen_total` | counter | `modelset_id`, `candidate_position`, `resolved_model_id` |
| `modelset_compile_total` | counter | `modelset_id`, `outcome` (`ok`/`error`) |
| `modelset_cache_size` | gauge | number of compiled programs in the in-process `sync.Map` |

The `modelset_id` label is high-cardinality but bounded by the
number of modelsets across all projects in the system. We accept
this. If a deployment has tens of thousands of modelsets, we
add a `--max-cardinality` flag to drop the label.

### Auth metrics

| metric | type | labels |
|---|---|---|
| `auth_lookup_total` | counter | `result` (`hit`/`miss`/`negative_hit`/`error`) |
| `auth_lookup_duration_seconds` | histogram | `result` |
| `auth_failures_total` | counter | `reason` (`no_header`/`bad_format`/`not_found`/`revoked`/`expired`/`hash_mismatch`) |

### Provider metrics

| metric | type | labels |
|---|---|---|
| `provider_request_total` | counter | `provider`, `model_id`, `outcome` (`ok`/`error`/`timeout`) |
| `provider_request_duration_seconds` | histogram | `provider`, `model_id`, `outcome` |
| `provider_upstream_status` | counter | `provider`, `model_id`, `status_code` |

### Storage metrics

| metric | type | labels |
|---|---|---|
| `pgpool_acquire_total` | counter | `result` |
| `pgpool_acquire_duration_seconds` | histogram | `result` |
| `pgpool_connections_in_use` | gauge | |
| `valkey_op_total` | counter | `op`, `result` |
| `valkey_op_duration_seconds` | histogram | `op`, `result` |

### Process metrics

Standard Go runtime metrics via `promhttp.Handler()` with the
Go collector enabled.

## Distributed traces (OpenTelemetry)

We export OTLP traces via gRPC, configurable via
`OTEL_EXPORTER_OTLP_ENDPOINT` (and the standard OTel env vars).
The default exporter is a no-op if no endpoint is configured —
the spans are still created, just not exported.

### Span hierarchy

The hierarchy is shallow: auth → handler → modelset.resolve →
provider.<name>.chat. The provider span is the one that records
TTFT, finish reason, and token usage. There are no agent /
session / memory spans in the request path.

```
HTTP POST /v1/chat/completions
└─ middleware.auth                              (rr.* attributes)
   └─ auth.lookup                               (rr.* attributes)
   └─ auth.cache_get                            (rr.* attributes)
      └─ (valkey op)
      └─ (postgres query, on miss)

   └─ handler.openai_chat
      └─ modelset.resolve                      (CEL evaluation)
      └─ provider.<name>.chat                  (the upstream HTTP)
```

The `modelset.resolve` span records the CEL evaluation latency,
the resolved candidate, and (on error) the reason. The
`provider.<name>.chat` span wraps the actual upstream call.

We add our own `rr.*` attributes to every span:

| attribute | source |
|---|---|
| `rr.request_id` | middleware |
| `rr.project_id` | auth |
| `rr.account_id` | auth |
| `rr.api_key_id` | auth |
| `rr.configuration_id` | auth |
| `rr.modelset_id` | auth |
| `rr.candidate_position` | resolver |
| `rr.resolved_model_id` | resolver |
| `rr.provider` | resolver |
| `rr.stream` | handler |
| `rr.prompt_tokens` | after response |
| `rr.completion_tokens` | after response |
| `rr.cached_tokens` | after response |

We respect the W3C trace-context standard: incoming
`traceparent` headers are honored. If absent, we mint a new
trace id.

### Sampling

Default: `parent_based(always_on)`. In production, switch to
`parent_based(rate(0.1))` for 10% sampling. The OTel SDK is
configured by env vars; we don't bake the rate in.

## Cost tracking (future)

The `request_log` table already has `prompt_tokens` and
`completion_tokens`. Once we have model pricing in the `model`
table, we can compute cost per request:

```
cost = (prompt_tokens / 1_000_000) * input_cost_per_million
      + (completion_tokens / 1_000_000) * output_cost_per_million
```

We will add a `cost_microcents` column to `request_log` (1
microcent = $0.00001) and a Prometheus counter
`chat_completion_cost_microcents_total` with labels
`model_id, project_id`. **Not MVP.**

## Audit log (future)

For now, the only audit trail is `request_log` (one row per
request). When we add admin operations (key creation, key
revocation, configuration changes, model registry changes), we
will add a separate `audit_log` table with
`(actor_account_id, action, target_type, target_id, before,
after, created_at)`. **Not MVP.**

## Health probes

- `/healthz` — liveness, 200 OK. Always returns 200 if the
  process is up. Used by orchestrators to decide "kill and
  restart".
- `/readyz` — readiness, 200 if Postgres + Valkey reachable
  within 1s, 503 otherwise. Body has a JSON map of dependency
  status. Used by load balancers to decide "send traffic".

## Alerting (suggested, future)

These are not built in; they are dashboards the operator wires
up against Prometheus.

- `rate(chat_completions_total{status="500"}[5m]) > 0.1` —
  5xx rate
- `histogram_quantile(0.99, chat_completion_duration_seconds) > 10`
  — p99 latency over 10s
- `pgpool_connections_in_use / pgpool_max_connections > 0.8` —
  pool exhaustion
- `auth_failures_total > 100` for 1m — possible credential
  stuffing

## Tooling

- **Logs**: ship to whatever the deployer uses (Loki, CloudWatch,
  Datadog). JSON makes this easy.
- **Metrics**: Prometheus scrape. Standard alerting rules in
  `deploy/observability/`.
- **Traces**: OTLP gRPC to a collector (Tempo, Jaeger, Cloud
  Trace). The OTel SDK handles batching and retries.
