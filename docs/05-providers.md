# 05 — Providers

The provider layer is the seam between railroad's internal model of
"send a chat completion, get a stream of events back" and each
upstream vendor's specific API. Every provider implements the same
interface; the agent runtime talks only to the interface.

For MVP, the registry has one provider: **Gemini**. Additional
providers plug in via the registry, with no changes to the agent
runtime.

## The abstraction

The interface is **not** a direct mirror of any single
vendor. It's shaped to be easy to implement against any
chat-completion API (Gemini, OpenAI, Anthropic Messages,
Mistral, vLLM, etc.) and to be the only thing the OpenAI
handler needs to know about.

```go
// internal/providers/provider.go (sketch)

package providers

import (
    "context"
    "iter"
)

// Provider is the contract every upstream integration must satisfy.
type Provider interface {
    // Name returns the registry name (e.g. "gemini"). Stable.
    Name() string

    // Chat runs a non-streaming chat completion and returns the
    // full result. Implementations should map provider-specific
    // errors to ChatError so the handler can render the right
    // OpenAI error code.
    Chat(ctx context.Context, req ChatRequest) (ChatResponse, error)

    // StreamChat returns a pull-based iterator of ChatEvents.
    // The iterator MUST close cleanly on context cancel, and the
    // last event (if any) must be a Finish event with a non-empty
    // FinishReason.
    StreamChat(ctx context.Context, req ChatRequest) (iter.Seq2[ChatEvent, error], error)
}

// ChatRequest is the provider-agnostic request shape. It is
// constructed by the OpenAI handler from the resolved
// configuration and the user-visible OpenAI request.
type ChatRequest struct {
    Model              string         // the upstream model id (e.g. "gemini-2.5-pro")
    SystemInstruction  string         // resolved system prompt
    Messages           []ChatMessage  // user/assistant/tool turns
    Tools              []ToolDef      // optional; empty for MVP
    Temperature        *float64       // nil = provider default
    TopP               *float64
    MaxOutputTokens    *int
    Stop               []string
    // Metadata is opaque to the provider but can be used for
    // routing hints (e.g. a regional preference). Providers may
    // ignore it.
    Metadata           map[string]string
}

type ChatMessage struct {
    Role       string         // "user" | "assistant" | "system" | "tool"
    Content    []ContentPart  // text, image, audio, ...
    ToolCallID string         // for role="tool"
    ToolCalls  []ToolCall     // for role="assistant"
}

type ContentPart struct {
    Kind    string // "text" | "image_url" | "input_audio" | ...
    Text    string
    // For binary parts:
    URL     string  // http(s) URL or data: URL
    Bytes   []byte  // if already in memory
    Mime    string
}

type ToolDef struct {
    Name        string
    Description string
    // JSON Schema for parameters, encoded as a json.RawMessage
    // so providers can pass it through verbatim.
    Parameters  json.RawMessage
}

type ToolCall struct {
    ID        string
    Name      string
    Arguments json.RawMessage
}

type ChatResponse struct {
    Message         ChatMessage
    FinishReason    string  // "stop" | "length" | "tool_calls" | "content_filter" | "error"
    Usage           Usage
    ProviderRequestID string  // upstream's request id, for debugging
}

type Usage struct {
    PromptTokens     int
    CompletionTokens int
    CachedTokens     int
    ReasoningTokens  int
    TotalTokens      int
}

// ChatEvent is a single event in a streamed chat. The event
// sequence is:
//   1. zero or more Content events
//   2. zero or more ToolCall events (function calling)
//   3. exactly one Finish event
type ChatEvent interface {
    chatEvent()
}

type ContentEvent struct{ Delta string }
type ToolCallEvent struct{ Call ToolCall }
type FinishEvent struct {
    Reason  string  // see ChatResponse.FinishReason
    Usage   Usage
}

func (ContentEvent) chatEvent()  {}
func (ToolCallEvent) chatEvent() {}
func (FinishEvent) chatEvent()   {}

// ChatError carries an OpenAI-style error code so handlers can
// map to status codes without knowing the provider.
type ChatError struct {
    StatusCode int    // suggested HTTP status (400/429/500/502/504)
    Code       string // "rate_limit" | "context_length" | "model_not_found" | "internal" | ...
    Message    string // human-readable, safe to surface
    Cause      error
}
```

Why `iter.Seq2[ChatEvent, error]` for streaming? It is the Go 1.23+
range-over-func primitive, gives us pull-based backpressure for free,
and forces the provider to handle context cancellation correctly (the
caller stops pulling).

## The Gemini implementation

The Gemini provider lives in `internal/providers/gemini/`. It is
**not** a wrapper around ADK — it is a thin wrapper around
`github.com/googleapis/go-genai` that satisfies `providers.Provider`.

```go
// internal/providers/gemini/client.go (sketch)

package gemini

import (
    "context"
    "iter"

    "github.com/googleapis/go-genai"

    "github.com/Cidan/railroad/internal/providers"
)

type Client struct {
    genai *genai.Client
    creds genai.Credential
}

func New(ctx context.Context, apiKey string) (*Client, error) { ... }

func (c *Client) Name() string { return "gemini" }

func (c *Client) Chat(ctx context.Context, req providers.ChatRequest) (providers.ChatResponse, error) {
    cfg := c.toGenaiConfig(req)
    contents := c.toGenaiContents(req.Messages)
    resp, err := c.genai.Models.GenerateContent(ctx, req.Model, contents, cfg)
    if err != nil { return providers.ChatResponse{}, mapErr(err) }
    return c.fromGenaiResponse(resp), nil
}

func (c *Client) StreamChat(ctx context.Context, req providers.ChatRequest) (iter.Seq2[providers.ChatEvent, error], error) {
    cfg := c.toGenaiConfig(req)
    contents := c.toGenaiContents(req.Messages)
    stream := c.genai.Models.GenerateContentStream(ctx, req.Model, contents, cfg)
    return func(yield func(providers.ChatEvent, error) bool) {
        var usage providers.Usage
        for chunk, err := range stream {
            if err != nil { yield(nil, mapErr(err)); return }
            // translate chunk → events
            for _, ev := range c.toChatEvents(chunk, &usage) {
                if !yield(ev, nil) { return }
            }
        }
        yield(providers.FinishEvent{Reason: "stop", Usage: usage}, nil)
    }, nil
}
```

The translation tables — `toGenaiConfig`, `toGenaiContents`,
`fromGenaiResponse`, `toChatEvents` — are the only pieces with
Gemini-specific knowledge. Everything else is provider-agnostic.

### Mapping finish reasons

| Gemini | OpenAI |
|---|---|
| `STOP` | `stop` |
| `MAX_TOKENS` | `length` |
| `SAFETY` | `content_filter` |
| `RECITATION` | `content_filter` |
| `BLOCKLIST` | `content_filter` |
| `PROHIBITED_CONTENT` | `content_filter` |
| `SPII` | `content_filter` |
| `MALFORMED_FUNCTION_CALL` | `stop` (with a `tool_calls` partial) |
| `OTHER` | `stop` |

### Mapping errors

| Gemini error | `ChatError.Code` | HTTP |
|---|---|---|
| 400 invalid argument | `invalid_request` | 400 |
| 404 not found (model) | `model_not_found` | 404 |
| 429 resource exhausted | `rate_limit` | 429 |
| 500/502/503/504 | `upstream_error` | 502 |
| timeout | `upstream_timeout` | 504 |
| cancelled | `cancelled` | 499 (no body) |

The `mapErr` helper inspects the `*genai.APIError` and produces a
`providers.ChatError`. The HTTP layer in the OpenAI handler reads
`StatusCode` and `Code` to build the response.

### Auth to Gemini

- For MVP: a single `GEMINI_API_KEY` env var. The Gemini client uses
  the Developer API (`generativelanguage.googleapis.com`), not
  Vertex.
- Future: per-project `provider_credential` rows
  (see [03-data-model](./03-data-model.md)). The provider factory
  picks credentials based on the API key's project.

## The provider registry

```go
// internal/providers/registry.go (sketch)

package providers

type Registry interface {
    Get(name string) (Provider, error)
    List() []string
}

func NewRegistry(creds Credentials) Registry {
    return &registry{
        providers: map[string]Provider{
            "gemini": geminiClientForCreds(creds.Gemini),
            // future: "openai", "anthropic", ...
        },
    }
}
```

The registry is constructed once at startup. It is immutable.
If we need to add a provider without a restart, we make it a
`*Registry` behind a mutex and reload from a config file or
DB event (future).

The OpenAI handler looks up a provider by name (the
`model.provider` field on the resolved candidate, returned by
the modelset resolver). The provider is the only thing the
handler needs to know about to dispatch.

## Why a registry at all if there's one provider?

The point isn't the current count. It's the seam. The OpenAI
handler must never import `internal/providers/gemini`
directly — only `internal/providers`. This keeps:

- the handler testable with a fake provider
- new providers additive (no changes to the handler)
- the dependency graph acyclic

## Adding a new provider

1. Create `internal/providers/<name>/`. Implement `Provider`.
2. Add a `New<Name>Client` constructor.
3. Register it in `providers.NewRegistry` behind a feature flag (so
   we can disable in prod without removing the code).
4. Add rows to the `model` registry migration for each new public
   model id.
5. Add integration tests under `internal/providers/<name>/testdata/`
   that hit the live provider with recorded fixtures (httptest).
6. Update [04-api-surface](./04-api-surface.md) and this doc to
   reflect the new model ids.

## What we do **not** put in the provider interface

- Token counting. Providers return usage in their `FinishEvent`
  or `ChatResponse`. The handler tallies.
- Retry. The handler has a single retry policy (one retry on
  transient errors, see [10-observability](./10-observability.md)).
  Providers do not retry internally.
- Caching. The handler or a higher layer can cache responses
  (future), not the provider.
- Auth. The provider gets pre-resolved credentials.

## Out-of-process provider calls

We are deliberately not putting the provider call in a separate
worker. Reasons:

- Streaming latency is dominated by time-to-first-byte. A separate
  worker adds a hop.
- We want a single, simple failure domain.
- We can extract the provider into a sidecar later if profiling
  says we need to (e.g. for OOM isolation of a runaway provider).
