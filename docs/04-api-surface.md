# 04 — API surface

The public API is **OpenAI-compatible**. A client that works against
`api.openai.com` today should work against a railroad endpoint
unchanged, modulo the `Authorization` header and the `model` values.

The MVP exposes the minimum surface to make a chat completion work,
plus a small handful of operational endpoints.

## Versioning

- The public API is rooted at `/v1/...`. We follow OpenAI's
  `/v1/...` convention so the OpenAI SDK defaults work.
- Internal / admin endpoints are rooted at `/internal/...` (see
  [07-auth-and-tenancy](./07-auth-and-tenancy.md) for how those are
  gated).

## Endpoints (MVP)

| Method | Path | Purpose | Auth |
|---|---|---|---|
| `POST` | `/v1/chat/completions` | The main chat endpoint. Sync or streaming. | API key |
| `GET`  | `/v1/models` | List models the caller can use | API key |
| `GET`  | `/v1/models/{model_id}` | One model | API key |
| `GET`  | `/healthz` | Liveness. Returns `200 ok` if the process is up. | none |
| `GET`  | `/readyz` | Readiness. Verifies Postgres + Valkey reachable. | none |
| `GET`  | `/metrics` | Prometheus scrape endpoint. | none (deployer-gated) |

## `POST /v1/chat/completions`

### Request body

Standard OpenAI ChatCompletionRequest. The fields we honor for MVP:

| field | type | notes |
|---|---|---|
| `model` | `string` | required. Resolved against the `model` registry; if the key's configuration has a fixed model, this must match. |
| `messages` | `array` | required. OpenAI message shape: `{role, content, name, tool_call_id, tool_calls}`. |
| `stream` | `bool` | default `false`. |
| `temperature` | `number` | optional; falls back to `configuration.default_temperature` then the model default. |
| `top_p` | `number` | same fallback chain. |
| `max_tokens` | `integer` | falls back to `configuration.default_max_output_tokens` then model default. |
| `stop` | `string \| string[]` | optional. |
| `n` | `integer` | default `1`. **MVP**: must be `1`. We reject `n>1` with 400. |
| `user` | `string` | optional. Echoed back in the response and recorded in `request_log`. CEL expressions on the modelset can read this field to make routing decisions (e.g., "shard by user id"). |
| `tools` | `array` | optional. **MVP**: accepted but not used — Gemini tool support is wired but not exposed to clients in v1. We 501 if a client sends `tools`. **DECIDE**: ship tool support in MVP? See [13-decisions](./13-decisions.md). |
| `tool_choice` | `string \| object` | same as `tools`. |
| `response_format` | `object` | future. |
| `seed` | `integer` | future. |
| `frequency_penalty` | `number` | forward to provider if supported. |
| `presence_penalty` | `number` | forward to provider if supported. |
| `logit_bias` | `object` | forward to provider if supported. |
| `stream_options` | `object` | honor `include_usage` for streaming. |

We **forward but do not promise** fields OpenAI adds later. Unknown
fields in the body are 400'd, not silently dropped, to catch
client-side bugs early.

### Response (non-streaming)

Standard OpenAI `ChatCompletionResponse`:

```json
{
  "id": "chatcmpl-<uuid>",
  "object": "chat.completion",
  "created": 1749734400,
  "model": "google/gemini-2.5-pro",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you?",
        "refusal": null
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 12,
    "completion_tokens": 8,
    "total_tokens": 20
  },
  "system_fingerprint": "rr_<short_hash>"
}
```

- `system_fingerprint` is a short hash of the (model, configuration
  version). Stable for the same effective setup, changes when we
  materially change the response shape or translator. OpenAI
  includes it for cache-busting hints; we do the same.
- `usage.cached_tokens` is added when the upstream reports it.
  OpenAI added this in 2025; we follow.
- `usage.reasoning_tokens` is included for reasoning models (when
  Gemini exposes them).
- The `id` is `chatcmpl-<uuidv7>` so it sorts and is k-safe.

### Response (streaming)

`Content-Type: text/event-stream` (charset utf-8). Each event:

```
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":...,
       "model":"...","choices":[{"index":0,"delta":{"role":"assistant"},
       "finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":...,
       "model":"...","choices":[{"index":0,"delta":{"content":"Hello"},
       "finish_reason":null}]}

...

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":...,
       "model":"...","choices":[{"index":0,"delta":{},
       "finish_reason":"stop"}],
       "usage":{"prompt_tokens":12,"completion_tokens":8,"total_tokens":20}}

data: [DONE]
```

- We always send the leading `role:"assistant"` chunk so clients that
  only inspect the first delta still get the role.
- The trailing chunk with `usage` is sent only if
  `stream_options.include_usage=true` (OpenAI's behavior).
- A final `data: [DONE]` is always sent on a clean close.
- If the connection is closed mid-stream, we do **not** resend. The
  client must reconnect and start over (we don't keep server-side
  state for resumable streams in MVP).

### Errors

OpenAI error envelope:

```json
{
  "error": {
    "message": "human-readable, safe to surface to the user",
    "type": "invalid_request_error",
    "param": "messages[0].content",
    "code": "missing_required_field"
  }
}
```

| HTTP | `type` | when |
|---:|---|---|
| 400 | `invalid_request_error` | malformed body, missing field, unknown field, bad value |
| 401 | `invalid_request_error` (`code: "invalid_api_key"`) | no key, bad key, revoked key |
| 403 | `invalid_request_error` (`code: "project_disabled"`) | project suspended |
| 404 | `invalid_request_error` (`code: "model_not_found"`) | model id unknown or not in this key's allow list |
| 408 | `request_timeout` | client request took too long before headers completed |
| 409 | `invalid_request_error` (`code: "conflict"`) | rare, e.g. concurrent key rotation |
| 413 | `invalid_request_error` (`code: "payload_too_large"`) | request body too big |
| 422 | `invalid_request_error` (`code: "unprocessable"`) | semantically wrong but well-formed |
| 429 | `rate_limit_error` | rate limit hit |
| 500 | `server_error` (`code: "internal"`) | unexpected. trace_id is in the body and the `X-Request-ID` header |
| 502 | `upstream_error` | provider returned a malformed response |
| 503 | `server_error` (`code: "unavailable"`) | Postgres or Valkey down |
| 504 | `request_timeout` (`code: "upstream_timeout"`) | upstream call exceeded its deadline |

All error bodies include `request_id` (same value as `X-Request-ID`)
so support can correlate with logs.

### Request size limits

- Body cap: 10 MB (configurable). Returns 413 above.
- Number of messages: 1000 (configurable).
- Per-message content length: 1 MB after JSON normalization.

These are belt-and-suspenders. The model registry's `context_window`
is the real ceiling.

## `GET /v1/models`

Returns models the **caller's key** is allowed to use. For MVP, the
allow list is "all `active` models in the registry". In a future
phase this can be narrowed per project.

Response shape follows OpenAI's `ListModelsResponse`:

```json
{
  "object": "list",
  "data": [
    {
      "id": "google/gemini-2.5-pro",
      "object": "model",
      "created": 1749000000,
      "owned_by": "google",
      "display_name": "Gemini 2.5 Pro"
    }
  ]
}
```

Pagination: we use a `cursor` (an opaque base64 string encoding
`(created_at, id)`) rather than page numbers. **DECIDE**: do we
need pagination in MVP? The registry is small. If we keep it
unpaged and the registry grows, we add cursor pagination later.

## `GET /v1/models/{model_id}`

Returns one model. 404 if not found. Same shape as a single element
of `/v1/models`.

## `GET /healthz`

Returns `200 OK` with body `ok`. Used for liveness probes; the
process must be alive to respond. Does not check downstream
dependencies.

## `GET /readyz`

Returns `200 OK` with body `ready` only if:

- A `SELECT 1` to Postgres succeeds within 1 second.
- A `PING` to Valkey succeeds within 1 second.

Otherwise returns `503 Service Unavailable` with a JSON body listing
which dependency failed. Used for readiness probes; load balancers
should stop sending traffic when this fails.

## `GET /metrics`

Prometheus text format. See [10-observability](./10-observability.md)
for the metric list.

## Headers

Request headers we honor:

| header | behavior |
|---|---|
| `Authorization: Bearer <key>` | required on all `/v1/...` endpoints |
| `X-Request-ID` | if present, used as the trace id. We validate it's a printable ASCII string ≤128 chars. |
| `Accept: text/event-stream` | hint that the client wants streaming (the `stream: true` body field is authoritative) |
| `User-Agent` | recorded in the access log |

Response headers we always send:

| header | value |
|---|---|
| `X-Request-ID` | the request id (generated or echoed) |
| `X-Railroad-Version` | the build version (e.g. `v0.1.0-abc1234`) |
| `Content-Type` | per endpoint |

## Wire-compatibility targets

We target compatibility with:

- The OpenAI Python SDK ≥ 1.0
- The OpenAI Node SDK ≥ 4.0
- The LangChain OpenAI integration
- `curl` users

We do **not** currently target:

- The OpenAI Responses API (`/v1/responses`). Future.
- Realtime / WebSocket. Future.
- Function-calling response format details. **DECIDE** for MVP
  (see above).

## Future endpoints (not MVP)

- `POST /v1/completions` (legacy text completions)
- `POST /v1/embeddings`
- `POST /v1/responses` (OpenAI's newer shape)
- `POST /v1/audio/{speech,transcriptions}`
- `GET /v1/usage` (per-key usage rollup)
- `GET /internal/v1/projects`, `GET /internal/v1/keys` etc. (admin)
