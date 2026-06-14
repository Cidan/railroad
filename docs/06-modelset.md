# 06 — Modelset

A **modelset** is the routing decision that picks which model serves
a chat completion. It is a small but central concept: an API key
binds to a configuration, and the configuration binds to a
modelset. The modelset holds an ordered list of model candidates
and a CEL expression that picks one in near real time.

For MVP, the resolver is pure data + CEL — no agent, no tool
plumbing, no memory. ADK is **not** in the request path. We
keep the option open to shim ADK *into* a modelset later for
advanced cases (judge/jury, multi-step agents), and this
document describes both the current shape and the future seam.

## What a modelset is

A `modelset` row is a named, project-scoped bundle of:

- An **ordered list of model candidates** with optional per-
  candidate weights and an optional eligibility predicate.
- A **CEL expression** that, given the request and the
  candidates, returns the position of the candidate to use. If
  the CEL is absent, the resolver uses the first eligible
  candidate in preference order.

```yaml
# conceptual shape
id: 9c5c...
project_id: 7d8f...
name: "Default routing"

cel_source: |
  size(request.messages[-1].content) > 4000 ? 0 : 1

candidates:
  - position: 0
    model_id: "google/gemini-2.5-pro"     # for long context
    weight: 1.0
    eligibility_cel: null
  - position: 1
    model_id: "google/gemini-2.5-flash"   # default
    weight: 1.0
    eligibility_cel: null
```

In this example, a request whose last message is longer than 4000
characters goes to `gemini-2.5-pro`; everything else goes to
`gemini-2.5-flash`. The CEL is the only place routing logic
lives; the candidates are pure data.

## Why CEL

We want routing decisions in **near real time** — every chat
completion evaluates the rule, and the rule has to be cheap.

CEL (Common Expression Language) is the right tool:

- **It's the standard.** Google uses it for IAM policy
  conditions, gRPC routing, and Kubernetes admission control.
  `github.com/google/cel-go` is the official Go implementation.
- **It's compiled to a native Go program.** A CEL expression is
  parsed once at modelset load time and evaluated many times
  per second with microsecond latency. The compiled program is
  cached in process memory.
- **It's sandboxed.** No I/O, no goroutines, no function
  declarations — the language is intentionally restricted. We
  declare the variables the expression can read; the rest is
  invisible.
- **It's debuggable.** CEL has a human-readable syntax, a
  parser error format, and tools like `cel-go`'s
  `cel.AstToCheckedExpr` for round-tripping.

CEL is not a general-purpose programming language. That's a
feature, not a bug — it forces routing logic to be small and
declarative, which is what we want for a per-request hot path.

## The data shape

Two tables (`modelset` and `modelset_candidate`); see
[03-data-model](./03-data-model.md) for the full schema.

```sql
CREATE TABLE modelset (
    id          uuid PRIMARY KEY,
    project_id  uuid NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name        text NOT NULL,
    description text,
    -- The CEL expression source. NULL = "use candidates[0]"
    -- (or the first eligible, if any candidate has an
    -- eligibility predicate).
    cel_source  text,
    metadata    jsonb NOT NULL DEFAULT '{}',
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE modelset_candidate (
    modelset_id     uuid NOT NULL REFERENCES modelset(id) ON DELETE CASCADE,
    position        integer NOT NULL CHECK (position >= 0),
    model_id        text NOT NULL REFERENCES model(id),
    weight          numeric(5,2) NOT NULL DEFAULT 1.0,
    -- Per-candidate CEL guard. Evaluated first; if false, the
    -- candidate is removed from the choice set before the
    -- modelset's main CEL runs. Useful for "skip pro if the
    -- project's budget is gone".
    eligibility_cel text,
    PRIMARY KEY (modelset_id, position)
);
```

The compiled CEL program is **not** persisted. We compile on
first use and cache in memory. A modelset with no `cel_source`
is a no-op resolver that always returns position 0.

## The CEL environment

The CEL expression sees a fixed, declared environment. We
deliberately do not let the expression call into arbitrary Go
functions; only a small whitelist of pure helpers.

| variable | type | meaning |
|---|---|---|
| `request.model` | `string` | the `model` field from the OpenAI request |
| `request.stream` | `bool` | whether the client asked for SSE |
| `request.user` | `string` | the OpenAI `user` field (often used as a session id) |
| `request.messages` | `list<Message>` | the conversation |
| `request.tools` | `list<Tool>` | tool defs (empty for MVP) |
| `request.temperature` | `double` | may be 0 (means "unset") |
| `request.max_tokens` | `int` | may be 0 |
| `project.id` | `string` (uuid) | the calling project |
| `project.name` | `string` | for display in expressions |
| `key.id` | `string` (uuid) | the calling API key |
| `candidates` | `list<Candidate>` | `[{position, model_id, weight, eligible}]` |
| `now` | `Timestamp` | the current time, for time-of-day rules |

Where:

```cel
message Message {
    role: string,         // "system" | "user" | "assistant" | "tool"
    content: string,      // concatenated text
    tool_call_id: string  // empty for non-tool messages
}

tool Tool {
    name: string,
    has_params: bool
}

candidate Candidate {
    position: int,
    model_id: string,
    weight: double,
    eligible: bool         // pre-computed from eligibility_cel
}
```

The expression's return type is `int` — the position of the
chosen candidate. If the CEL returns a position outside the
candidates list, or returns the wrong type, the resolver falls
back to "first eligible candidate in preference order" and
logs a warning. The request still succeeds.

### Expression examples

Pick the cheapest model that supports tools:

```cel
size(request.tools) > 0
    ? candidates[0].model_id == "google/gemini-2.5-pro" ? 0 : 1
    : 1
```

(This isn't really expressing cost — it's a placeholder. CEL
doesn't have a built-in "cost" concept; we add cost as a
declared variable if we want it. See Future Work.)

Shard by user id modulo N candidates:

```cel
size(request.user) > 0
    ? int(hash(request.user)) % size(candidates)
    : 0
```

Block all but the first candidate outside business hours:

```cel
hour(now) >= 9 && hour(now) < 18 ? 0 : 1
```

The full CEL language reference is at
<https://github.com/google/cel-spec/blob/master/doc/langdef.md>.

## The resolver

```go
// internal/modelset/resolver.go (sketch)

package modelset

import (
    "context"
    "fmt"
    "sync"
    "time"

    "github.com/google/cel-go/cel"
    "github.com/google/cel-go/common/types/ref"

    "github.com/Cidan/railroad/internal/domain"
    "github.com/Cidan/railroad/internal/observability"
)

type Resolver struct {
    env      *cel.Env
    cache    sync.Map                  // modelset_id -> *compiled
    metrics  *observability.Metrics
}

type compiled struct {
    program cel.Program
    source  string                     // for diagnostics
}

type Environment struct {
    Request    openai.ChatRequest
    Project    domain.Project
    APIKey     domain.APIKey
    Candidates []domain.Candidate
    Now        time.Time
}

func NewResolver(metrics *observability.Metrics) (*Resolver, error) {
    env, err := cel.NewEnv(
        cel.Variable("request",     cel.DynType),
        cel.Variable("project",     cel.DynType),
        cel.Variable("key",         cel.DynType),
        cel.Variable("candidates",  cel.ListType(cel.DynType)),
        cel.Variable("now",         cel.TimestampType),
    )
    if err != nil { return nil, err }
    return &Resolver{env: env, metrics: metrics}, nil
}

// Resolve picks a candidate. Returns the candidate itself.
func (r *Resolver) Resolve(ctx context.Context, ms domain.Modelset, env Environment) (domain.Candidate, error) {
    start := time.Now()
    defer func() { r.metrics.RecordResolveLatency(time.Since(start)) }()

    // 1. Filter out ineligible candidates.
    eligible := make([]domain.Candidate, 0, len(env.Candidates))
    for _, c := range env.Candidates {
        eligible = append(eligible, c) // eligibility already computed in env
    }
    if len(eligible) == 0 {
        return domain.Candidate{}, ErrNoEligibleCandidate
    }

    // 2. If there's no CEL, return the first eligible.
    if ms.CELSource == "" {
        r.metrics.RecordResolution("no_cel", eligible[0].Position)
        return eligible[0], nil
    }

    // 3. Compile (or fetch from cache) and evaluate.
    prog, err := r.program(ctx, ms)
    if err != nil { return domain.Candidate{}, err }

    activation := map[string]any{
        "request":    requestAsCEL(env.Request),
        "project":    projectAsCEL(env.Project),
        "key":        keyAsCEL(env.APIKey),
        "candidates": candidatesAsCEL(eligible),
        "now":        env.Now,
    }
    out, _, err := prog.Eval(activation)
    if err != nil {
        r.metrics.RecordResolutionError("eval_error", ms.ID)
        return eligible[0], nil // fail open: return first eligible
    }

    pos, ok := out.Value().(int64)
    if !ok || pos < 0 || int(pos) >= len(eligible) {
        r.metrics.RecordResolutionError("bad_result", ms.ID)
        return eligible[0], nil
    }
    r.metrics.RecordResolution("ok", int(pos))
    return eligible[int(pos)], nil
}

func (r *Resolver) program(ctx context.Context, ms domain.Modelset) (cel.Program, error) {
    if v, ok := r.cache.Load(ms.ID); ok {
        return v.(*compiled).program, nil
    }
    ast, issues := r.env.Compile(ms.CELSource)
    if issues != nil && issues.Err() != nil {
        return nil, fmt.Errorf("compile %s: %w", ms.Name, issues.Err())
    }
    prog, err := r.env.Program(ast,
        cel.EvalOptions(cel.OptOptimize),
    )
    if err != nil { return nil, err }
    r.cache.Store(ms.ID, &compiled{program: prog, source: ms.CELSource})
    return prog, nil
}
```

A few design notes:

- **Fail open.** If the CEL fails to compile or evaluate, we
  return the first eligible candidate and log. A misconfigured
  routing rule should degrade to "the default" — not break all
  traffic.
- **Compile errors are caught at modelset write time.** The
  admin endpoint that creates or updates a modelset compiles
  the CEL once and rejects the write on compile error. A bad
  rule never reaches the request path.
- **The compiled program is per-modelset and process-local.**
  Restarting the process recompiles. The compile is fast
  (single-digit ms); we don't bother caching in Valkey.
- **Eligibility is pre-computed.** Each candidate's
  `eligibility_cel` is evaluated once when the
  `Environment` is built. The main CEL sees the pre-filtered
  list with `candidate.eligible = true` for all entries.
- **The metrics are deliberately coarse.** Per-CEL-expression
  cardinality would explode. We aggregate by outcome
  (`ok`, `no_cel`, `eval_error`, `bad_result`).

## The fallback chain (future)

The MVP does not implement automatic fallback. If the chosen
model returns 429, we surface that 429 to the client. The
operator can configure fallback in the CEL itself
(`candidates[0]` if eligible, else `candidates[1]`) or write a
guard that returns a different position.

A later phase adds a transparent fallback layer in the
provider: if the chosen candidate returns a transient error,
try the next eligible candidate. This is independent of the
CEL — it's a resilience feature on top of the routing
decision. Tracked in [12-roadmap](./12-roadmap.md) Future Work.

## How the OpenAI handler drives a request

```go
// internal/api/openai/chat.go (sketch, revised)

func (h *Handler) ChatCompletions(w http.ResponseWriter, r *http.Request) {
    resolved := authctx.FromContext(r.Context())  // Resolved struct

    var req openai.ChatCompletionRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil { ... }

    // 1. Resolve model via modelset.
    env := modelset.Environment{
        Request:    req,
        Project:    resolved.Project,
        APIKey:     resolved.APIKey,
        Candidates: resolved.Modelset.Candidates,
        Now:        time.Now().UTC(),
    }
    candidate, err := h.resolver.Resolve(r.Context(), resolved.Modelset, env)
    if err != nil { /* 503 or 500 */ }

    // 2. Build the provider request from the OpenAI request
    //    and the resolved model's provider_model_id.
    preq := buildProviderRequest(candidate, resolved.Configuration, req)

    // 3. Get the provider and call.
    provider, err := h.registry.Get(candidate.Model.Provider)
    if err != nil { /* 500 */ }

    if req.Stream {
        h.streamRun(w, r, provider, candidate, preq)
    } else {
        h.blockingRun(w, r, provider, candidate, preq)
    }
}
```

The handler is **the only place that knows about both the
OpenAI shape and the provider shape**. Everything below it
is decoupled.

## The Resolved struct (after auth)

```go
// internal/authctx/context.go (sketch)

type Resolved struct {
    APIKey        domain.APIKey
    Project       domain.Project
    Account       domain.Account
    Configuration domain.Configuration
    Modelset      domain.Modelset        // new
    Model         domain.Model           // the "primary" model, for
                                         // headers and default model id;
                                         // not necessarily the one we use
}
```

`Resolved.Modelset` is fully materialized at auth time. The
handler does not re-fetch.

## What we deliberately do **not** have

For MVP, the request path is a clean passthrough with one
routing decision. We do not have:

- **ADK agents per request.** No `LLMAgent`, no `Runner`, no
  per-request session creation in the request path.
- **Session persistence.** OpenAI chat completions are
  stateless; we mirror that. The Postgres `adk_session` and
  `adk_event` tables are not in MVP.
- **Memory sets / memory injection.** Dropped from MVP per the
  design decision; can be added later as a modelset-time
  feature.
- **Tools / function calling.** Accepted in the OpenAI
  request body for forward compatibility, but not exposed
  yet. A future phase adds them via a `tool_modelset`
  implementation.
- **Bidirectional streaming / live mode.** OpenAI's realtime
  endpoints are out of scope.

## Future: ADK as a modelset implementation

We can add ADK without changing the data model or the handler
contract. The seam is the `Resolver` interface in
`internal/modelset/resolver.go`.

A future `ADKResolver` would:

- Build an `LLMAgent` from the first candidate (the model) and
  the rest (sub-agents or tools).
- Optionally use a `SequentialAgent` or `LoopAgent` to
  orchestrate.
- Run via the ADK `Runner`, mapping events to `ChatEvent`s.
- Plug into the same `Resolver` interface so the handler
  doesn't change.

The data shape stays the same. The kind of resolver chosen is
either driven by a future `modelset.implementation` field
(`cel` (default), `adk_agent` (future), `judge_jury` (future))
or by the kind of CEL expression (a `kind()` builtin or a
declared discriminator). The current MVP ships with one
implementation; we add others when there's a concrete need.

Specifically, "judge/jury" would look like:

```yaml
kind: judge_jury
cel_source: |
  # Run the request against candidates 0 and 1, then have
  # candidate 2 (a strong model) pick the best. The resolver
  # implementation handles the multi-call orchestration.
  2
```

The data is the same. The implementation knows how to do
multi-model calls. The handler is unchanged.

## Future: memory sets as a modelset-time feature

If we want memory sets back, the natural shape is:

- A `memory_set_id` on the `configuration` (as before).
- A `MemoryAwareCEL` extension: a declared variable
  `memories` in the CEL environment, populated by a top-K
  pgvector lookup at request time.
- The handler (or a helper) prepends the memories to the
  system prompt before calling the model.

This is a clean addition because the resolver contract doesn't
change — the CEL just gets more inputs. We do not need to
re-architect for it. Not MVP per the current decision; tracked
in [12-roadmap](./12-roadmap.md) Future Work.

## Future: cost-aware routing

The `model` table will eventually have
`input_cost_per_million` and `output_cost_per_million` columns.
We can then add a `cost` declared variable to the CEL
environment:

```cel
cost: map<string, double>  # model_id -> estimated cost for this request
```

Expressions can then prefer cheaper models for cheap tasks
and reserve expensive models for hard ones:

```cel
request.estimated_output_tokens < 500 && cost[candidates[0].model_id] > 0.01
    ? 1  # use flash
    : 0  # use pro
```

`request.estimated_output_tokens` requires a token-counting
step (a fast heuristic on the request, not a real count) which
is a separate feature. Tracked in Future Work.

## Versioning policy

`cel-go` is on a 0.x version line that we pin to a minor
version. We do not let it auto-upgrade. CEL itself is a
stable spec; the Go SDK is the moving part.
