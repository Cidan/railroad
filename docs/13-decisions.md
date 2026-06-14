# 13 — Decisions

This document records the decisions that shaped the plan, in the
form of short ADRs (Architecture Decision Records), plus a list
of decisions that are still open and need to be made before
code lands.

Each ADR captures the decision, the context, the alternatives
considered, and the consequences. When a decision is revisited
or reversed, edit the ADR in place — do not rewrite history.

---

## ADR-001 — Go 1.26.4

**Status**: Accepted.

**Context**: We need a Go version for railroad. The user
specified 1.26.4.

**Decision**: Pin to Go 1.26.4 in `go.mod` and CI. Use
`toolchain go1.26.4` to match.

**Consequences**:
- Standard library `net/http` pattern routing is available
  (Go 1.22+), reducing the need for a router.
- `iter.Seq2` is available (Go 1.23+), which we use for
  streaming.
- We get the standard library improvements in `slog`, `errors`,
  etc. without a third-party dependency.

---

## ADR-002 — ADK-go: future shim, not MVP agent

**Status**: Accepted (revised after initial draft).

**Context**: The user originally asked for "ADK for model
calls". The initial plan used ADK-go's `LLMAgent` and `Runner`
in every request, with sessions and memory services wired
through Postgres. The user then asked to rethink: a
passthrough, with a routing decision (the modelset) instead of
an agent.

**Alternatives considered**:
- **ADK-go v1.4.0 as the per-request agent runtime** (initial
  plan). Mature; gives us sessions, memory, tools, callbacks,
  plugins, traces. But: every chat completion runs through a
  full `LLMAgent` cycle, even when no agent behavior is
  wanted.
- **ADK-go v1.4.0 as a future modelset implementation** (chosen).
  ADK is no longer in the MVP request path. We may add it
  later as one `Resolver` implementation (alongside the
  CEL-routed default) for advanced cases — judge/jury,
  multi-step agents, structured tool use. The data model and
  handler contract are unchanged; the new implementation
  plugs into the existing `Resolver` interface.
- **Skip ADK entirely.** Use `go-genai` directly. Same
  outcome as the chosen path, with no future-shim option.

**Decision**: Use ADK-go v1.4.0 *only* as a future modelset
implementation. Isolate all ADK imports to
`internal/modelset/adk.go` (future) so a refactor stays local.
Do not depend on ADK-go for the MVP.

**Consequences**:
- The MVP request path is a clean passthrough: auth →
  modelset resolver → provider. No agent, no session, no
  memory in the critical path.
- The MVP binary is smaller and has fewer moving parts.
- When we need agent capabilities, the seam is in place; we
  add a new `Resolver` implementation and the data model
  gets an optional `implementation` field (or we keep it
  pure-data and dispatch in code).
- We do not need to depend on `google.golang.org/adk` for the
  MVP, but we can pull it in later without disturbing existing
  code.

---

## ADR-010 — Modelset is pure data + CEL

**Status**: Accepted.

**Context**: The user asked for a "modelset selector" with
ordered preferences and/or rules-based configuration, "in as
near real time as possible". This is the central design
question for the new shape.

**Alternatives considered**:
- **Pure data + CEL (chosen).** A modelset is an ordered list
  of model candidates and a CEL expression that picks one. The
  CEL is the only routing logic. The candidates are pure data.
  ADK (or judge/jury, or any other complex behavior) becomes
  a *kind of* modelset implementation; the data model is
  unchanged.
- **A `kind` enum: `simple`, `cel_routed`, `adk_agent`,
  `ensemble`, `judge_jury`.** Different kinds have different
  data shapes. More upfront structure; the data model carries
  a discriminator.
- **A `kind` enum in the resolver code, with a single data
  shape.** The data model is one shape, but the resolver
  dispatches on a kind field. Same surface as (B) but less
  data-model rigidity.

**Decision**: Pure data + CEL. The data model is one shape.
Future implementations (ADK, judge/jury) are new
`Resolver` types chosen by the operator (env var or admin
endpoint); the data is the same. If the implementation choice
ever needs to be stored in the data, we add a
`modelset.implementation` text field and treat it as
advisory.

**Consequences**:
- The data model is small and unambiguous. One table for
  modelsets, one for candidates.
- Routing is a string (the CEL source), easy to read, easy
  to log, easy to test.
- The CEL must be expressive enough to cover the cases we
  care about. CEL is a real language with maps, lists,
  strings, arithmetic, and conditionals. If we ever hit a
  limit, we add a *helper function* (declared in the env),
  not a new data shape.
- We commit to `cel-go` (the official Go SDK). Pin the
  version. CEL itself is a stable spec.

---

## ADR-003 — OpenAI-compatible API as the primary surface

**Status**: Accepted.

**Context**: The user said "openrouter-like clone" and
"openai API". OpenRouter's value prop is "drop-in OpenAI
replacement that also routes to many providers". Our minimum
version of that is "drop-in OpenAI replacement" — the
provider routing is internal (driven by the modelset) rather
than client-driven.

**Decision**: Expose `POST /v1/chat/completions`,
`GET /v1/models`, `GET /v1/models/{id}` with shapes
byte-compatible with the OpenAI Python and Node SDKs. Match
the error envelope, finish reasons, usage keys, and
streaming chunk format.

**Consequences**:
- Clients written against OpenAI work unchanged.
- We do not invent a new wire format. The benefit of being
  OpenAI-shaped is bigger than the cost of being
  OpenAI-constrained.
- We will keep up with OpenAI's surface as it evolves, on a
  best-effort basis. Unknown fields are 400'd (not silently
  dropped) so we don't drift.

---

## ADR-004 — Postgres + Valkey

**Status**: Accepted.

**Context**: The user specified Postgres + Valkey as the
backend. Both are excellent choices for the shape of data
railroad holds.

**Decision**:
- **Postgres 16+** for durable state: tenants, keys,
  configurations, modelsets, request logs.
- **Valkey 7.2+** for hot-path caching, rate-limit counters,
  and ephemeral state. The official `valkey-go` (rueidis)
  client, not `go-redis`, because of higher throughput and
  built-in server-assisted caching.

**Consequences**:
- We get durable ACID for business data and fast
  eventually-consistent caching. A well-understood split.
- We depend on a relational schema being the right shape
  for the tenants. We are confident it is.
- We do **not** need a separate queue or stream processor
  in MVP. If we add async work later, we add it then.
- We do **not** need `pgvector` in MVP. No embeddings to
  store; we drop the extension dependency. If a future
  phase adds memory sets, we add `pgvector` then.

---

## ADR-005 — `valkey-go` (rueidis) over `go-redis`

**Status**: Accepted.

**Context**: Both work with Valkey 7.2+ and Redis OSS
7.x. They're not API-compatible, so the choice is sticky.

**Alternatives considered**:
- **`valkey-io/valkey-go` (rueidis)** (chosen). Maintained
  by the Valkey project. Auto-pipelining, server-assisted
  client-side caching, OpenTelemetry instrumentation,
  distributed lock, cache-aside helpers.
- **`redis/go-redis` v9**. More popular, more tutorials.
  The benchmark in the valkey-go README shows rueidis ~14x
  faster on a high-parallelism local test.

**Decision**: Use `valkey-io/valkey-go`. If we need to
switch later, we wrap it behind a `Cache` interface in
`internal/store/cache` so the impact is local.

**Consequences**:
- Faster hot path under concurrency.
- Slightly less familiar API for new contributors.
  Mitigation: wrap the cache calls in our own interface;
  the rest of the code never sees rueidis specifics.
- We commit to the Valkey ecosystem. If we ever need to
  move to a different cache (DragonflyDB, KeyDB), the
  `internal/store/cache` interface is the seam.

---

## ADR-006 — Multi-tenant model: account → project → api key

**Status**: Accepted.

**Context**: The user specified:
- 1 account = 1 user
- An account must belong to at least one project
- An account can belong to many projects
- API keys are scoped to a single project

**Decision**: Encode this in the data model with an m2m
join `account_project` and a `project_id` on every business
table. Enforce the "at least one project" rule in application
code (in the account and account-repository layer). API keys
are project-scoped; the resolved `Project` and
`Configuration` are injected into the request context at
auth time.

**Consequences**:
- The data model is clear and matches the spec.
- Cross-tenant access is structurally hard: a handler
  receives a `Project` and has no way to ask for data
  outside it.
- The "at least one project" rule is an application-level
  invariant, not a DB constraint. A future migration could
  add a check constraint or trigger if we want defense in
  depth.

---

## ADR-007 — API key format and hashing

**Status**: Accepted.

**Context**: API keys are the only authentication the public
API uses. The format and storage matter for security and
operability.

**Decision**:
- Format: `rr_live_<32 base62 chars>`. ~190 bits of
  entropy in the body.
- Storage: argon2id hash of the full key. The first 8 chars
  of the body (joined with the prefix) are stored as a
  `prefix` column, indexed, and used as the lookup key.
  Verification is constant-time.
- The raw key is shown **once**, at creation. The server
  never logs the raw key.

**Consequences**:
- Brute force is bounded by argon2id cost (~50ms/verify
  on modern hardware).
- A leaked DB doesn't yield usable keys.
- Operators can identify a key by its prefix in logs
  (`rr_live_AbCdEfGh...`) without exposing the secret.

---

## ADR-008 — Gemini as the first provider

**Status**: Accepted.

**Context**: The user said "let's just start with gemini".

**Decision**: Implement Gemini via
`github.com/googleapis/go-genai` v1.60.0. Put it behind the
`providers.Provider` interface. The interface is the only
thing the rest of the code knows about; adding OpenAI,
Anthropic, etc. later is a new
`internal/providers/<name>/` directory and a registry entry.

**Consequences**:
- We validate the provider abstraction against a real
  implementation before adding the second one.
- The Gemini SDK is small and well-maintained; we add it
  directly. We do **not** add ADK-go as a transitive dep
  in MVP (that's only for the future shim path).
- We commit to the Gemini Developer API for MVP, not
  Vertex. Vertex is a credential-swap change when we need
  it.

---

## ADR-009 — Project-scoped API keys

**Status**: Accepted (confirmed with the user before the
plan was written).

**Context**: Two options:
- (A) Project-scoped: the key belongs to one project; the
  key carries a single configuration.
- (B) Account-scoped, project-aware: the key is owned by
  an account and can target any of that account's projects.

**Decision**: (A). Project-scoped.

**Consequences**:
- Simpler mental model.
- Matches OpenRouter's model.
- Multi-project use means creating one key per project,
  which is the right granularity for per-project policies
  (rate limits, budgets, audit) anyway.

---

## ADR-011 — Fail-open on CEL errors

**Status**: Accepted.

**Context**: When the CEL expression on a modelset fails to
compile (rejected at write time, so should never reach the
request path) or fails to evaluate at runtime, the resolver
has three options: fail closed (return 500), fail open to a
default, or fail open to the first eligible candidate.

**Decision**: Fail open to the first eligible candidate in
preference order. Log a warning. The request continues
normally.

**Consequences**:
- A misconfigured modelset degrades to "the default", not
  "all traffic broken". This is the right failure mode for
  a routing layer — a bad rule should not take down the
  API.
- We log loudly so operators see the issue. The
  `modelset_resolution_total{outcome="eval_error"}` metric
  spikes.
- Compile-time errors are still rejected at write time.
  The fail-open is only for runtime surprises (e.g., a
  declared variable the CEL author didn't anticipate).

---

## Open decisions (DECIDE)

These are recorded explicitly so we do not silently make the
call when we get to implementation. Each lists the question,
the recommended default, and the trigger to revisit.

### DECIDE-1 — Stub auth: any key vs seeded key

**Question**: In `RAILROAD_AUTH_MODE=stub`, should we accept
any well-formed `rr_live_...` key, or require a row to
exist in `api_key`?

**Recommended**: require a row. More realistic, exercises
the same code path, easy to seed in tests.

**Revisit if**: tests become annoying to write because
every test needs to seed a key first.

### DECIDE-2 — Migrations tool: goose vs migrate vs atlas

**Question**: Which migrations library?

**Recommended**: `github.com/pressly/goose/v3`. Has a Go
API we can call from a CLI subcommand, simple SQL files,
no need for a separate tool.

**Revisit if**: we want declarative schema diffing (atlas)
or a more featureful CLI (migrate).

### DECIDE-3 — HTTP router: stdlib only vs stdlib + chi

**Question**: Use stdlib's `net/http` pattern routing only,
or also include `chi` for internal routes?

**Recommended**: stdlib only for the public API. Add `chi`
only if middleware composition on the internal API becomes
awkward.

**Revisit if**: the internal API grows to need per-route
middleware (audit logging on mutating routes, etc.).

### DECIDE-4 — Tool calling in MVP

**Question**: Ship tool/function calling in MVP, or 501
clients that send `tools`?

**Recommended**: 501 in MVP. The translation is
non-trivial (finish reasons, tool call ids, parallel tool
calls). The provider interface is in place so we can turn
it on later.

**Revisit if**: a concrete use case appears before phase 5.

### DECIDE-5 — Default configuration fallback

**Question**: If an `api_key.configuration_id` is NULL, do
we fall back to a project default, or reject the request?

**Recommended**: reject for MVP. Force every key to have a
configuration. The schema allows NULL (for future), but
the auth path treats NULL as a 500.

**Revisit if**: project-level defaults become a real need.

### DECIDE-6 — Slow client backpressure on streaming

**Question**: When a streaming client reads slowly, do we
cancel the upstream call, or just let the writer block?

**Recommended**: let it block. The client's pace sets the
pace. We do not cancel upstream calls because of slow
reads.

**Revisit if**: a slow client ties up a provider connection
and the cost becomes significant.

### DECIDE-7 — Response caching

**Question**: Do we cache identical requests and return the
cached response?

**Recommended**: not in MVP. Privacy implications are real
(identically-prompted requests may be from different users
with different authorization). Future feature, opt-in per
configuration.

**Revisit if**: customers ask for it.

### DECIDE-8 — Model id allowlist per project

**Question**: Should a project be able to restrict which
models its modelsets can call?

**Recommended**: not in MVP. The model registry is global.
Per-project allowlists are a clean future addition.

**Revisit if**: a multi-tenant customer wants to lock
their keys to a specific model.

### DECIDE-9 — `n>1` in chat completions

**Question**: Reject `n>1` (we plan to) or implement it?

**Recommended**: reject. Gemini's equivalent
(`candidateCount`) exists but matching OpenAI's exact
semantics (n independent samples, each with their own
`usage`) is fiddly.

**Revisit if**: a concrete need appears.

### DECIDE-12 — CEL provider variables (cost, latency, model
capability)

**Question**: As the data shape grows, the CEL environment
can grow too. Which variables do we add in MVP, and which
do we defer?

**Recommended**: MVP declares `request.*`, `project.*`,
`key.*`, `candidates`, and `now`. Future phases can add
`cost` (when we track costs), `latency` (when we have
provider latency data), `model.supports_tools`, etc. Each
addition is a small, deliberate change to the CEL env.

**Revisit if**: a concrete need appears for cost-aware or
capability-aware routing in MVP.

### DECIDE-13 — Modelset `implementation` field on the row

**Question**: If the data shape is one, but we have
multiple resolver implementations (CEL today, ADK later),
do we add an `implementation` column to `modelset`?

**Recommended**: not in MVP. Today there is one
implementation. When we add ADK, we either pick the
implementation at startup (env var) or add a
`modelset.implementation` text field then. Adding the
field now is over-engineering for a single value.

**Revisit if**: we ship two resolver implementations in
the same binary and the operator wants to mix.

---

## How to add a new ADR

When you make a non-trivial decision during implementation,
add an ADR to this file. The format is short — context,
alternatives, decision, consequences — and the numbering is
sequential. Do not edit a past ADR; add a new one that
supersedes it if needed.
