# 01 — Overview

## What is railroad?

Railroad is a **multi-tenant AI gateway**. It accepts requests that
look exactly like OpenAI Chat Completions API calls, looks up which
configuration the caller's API key points at, asks a **modelset**
which model to use for this particular request, and proxies the
call to that upstream provider.

The modelset is the routing layer: a small data structure with an
ordered list of model candidates and a CEL expression that picks
one in near real time. For a request whose last message is short,
the modelset might pick Gemini 2.5 Flash; for a long context, it
might pick Gemini 2.5 Pro. The routing logic is data, not code,
and changes don't require a deploy.

The first upstream is **Google Gemini**. Additional providers
(OpenAI, Anthropic, etc.) plug into the same provider abstraction.

It is intentionally not trying to be a UI, a model trainer, a
fine-tuner, a billing system, or a model router in the OpenRouter
"lowest-latency provider" sense. Those can come later or be left
to upstream layers.

## What is *not* in the request path

Railroad is a **passthrough with routing**, not an agent platform.
The request path looks like:

```
OpenAI-shaped request
   → auth (api key → project → configuration → modelset)
   → modelset resolver (CEL picks a candidate)
   → chosen provider
   → OpenAI-shaped response
```

There is no agent, no session persistence, no memory injection,
no tool calling, and no multi-step orchestration in the MVP. The
handler is a thin OpenAI ↔ provider translator with a routing
decision in the middle.

We may add a more structured **ADK-backed modelset** later for
cases that need it (judge/jury, multi-step agents, structured tool
use). ADK becomes a *kind of* modelset implementation, not a
layer that wraps every call. The seam for this is preserved in
the data model and the resolver interface — see
[06-modelset](./06-modelset.md) for the shape.

## Why build it?

We want a single front door for LLM calls across our own apps that:

1. Looks like OpenAI to the client, so the OpenAI SDK and the
   wider ecosystem of OpenAI-shaped tools (langchain, llamaindex,
   editor plugins, etc.) work unchanged.
2. Lets us route between multiple models per API key — for
   example, "use the cheap flash model for short prompts, the
   expensive pro model for long ones" — without changing client
   code or rebuilding the binary.
3. Lets us swap the upstream provider per API key without
   redeploying clients.
4. Is multi-tenant from day one, with proper isolation, so we can
   hand API keys to other people and teams without writing a
   permission system later.

## What "multi-tenant" means here

- An **account** is one human (or one service identity). 1
  account = 1 user, no exceptions.
- A **project** is a workspace. It owns configurations, modelsets,
  and API keys.
- Every account must belong to **at least one** project. An account
  can belong to **many** projects.
- **API keys** are scoped to a single project. The key carries a
  reference to a **configuration** (system prompt, defaults) which
  carries a reference to a **modelset** (candidates + CEL rule).
- The full chain is:
  `Authorization: Bearer <key>` →
  `api_key → project → configuration → modelset → candidates`
  → CEL picks one → upstream provider.

The data model in [03-data-model](./03-data-model.md) and the auth
flow in [07-auth-and-tenancy](./07-auth-and-tenancy.md) make this
concrete.

## Goals (MVP)

1. A single Go binary that runs the public API.
2. **OpenAI-compatible** `POST /v1/chat/completions` (sync +
   streaming via SSE) and `GET /v1/models`, against Gemini.
3. **Multi-tenant data model** live in Postgres: accounts, projects,
   API keys, configurations, modelsets, models.
4. **API key reading** in middleware. For the MVP, the middleware
   accepts any non-empty, well-formed key (stub), but the
   resolution chain (key → project → configuration → modelset) is
   in place so swapping in real validation later is a one-file
   change.
5. **Modelset resolver** using CEL. The same expression is
   compiled once per modelset and evaluated per request in
   microseconds.
6. **Provider abstraction** with Gemini as the first impl.
7. **Observability**: structured logs, Prometheus metrics, OTel
   traces. Routing decisions show up in the access log and the
   trace.
8. **Local dev** with `docker compose` (Postgres + Valkey) and a
   single `make run`.

## Non-goals (MVP)

These are deliberately out of scope. They are listed so we do not
drift into building them.

- A web UI / admin console. Admin operations are CLI-only or HTTP
  endpoints behind admin auth in MVP; a UI is a later project.
- Billing, invoicing, payments. We log cost-relevant data; we do
  not charge.
- Per-tenant rate limiting beyond a global ceiling. Per-tenant
  limits are a config and enforcement layer we add in a later
  phase.
- Model routing that picks the cheapest / fastest provider per
  request. We route to a single configured model per request
  (chosen by CEL); we do not score providers.
- ADK / agents in the request path. Deferred to a future
  modelset implementation, not removed — the seam is in place.
- Memory sets. Removed from MVP; can be re-added later as a
  modelset-time feature.
- Fine-tuning, dataset management, eval pipelines. Out of scope.
- Realtime / audio / video endpoints. Chat completions only for
  MVP.
- Embeddings endpoint. Future, after the abstractions settle.
- Multi-region / active-active deployment. Single-region for MVP.

## Core principles

- **Correctness before features.** A wrong answer is worse than no
  answer. Token accounting, finish reasons, error mapping, and
  streaming chunks must match OpenAI's wire format exactly.
- **Multi-tenant from line one.** No code path may leak data
  across projects. Every repository method takes a project (or
  key) scope as its first argument.
- **Routing is data, not code.** The modelset is a row in the
  database, the CEL expression is a string, the candidates are a
  list. Adding a new routing rule does not require a rebuild.
- **Stable abstractions over concrete providers.** The provider
  interface, the modelset interface, the configuration interface,
  and the model registry are all things we expect to live for
  years.
- **Observability is a product feature.** Every request gets a
  trace ID, structured logs, and token usage recorded. Every
  routing decision gets a metric. We do not ship changes that
  cannot be debugged from production data.
- **Stub the auth, don't skip it.** Even when "any key works", the
  middleware still parses, validates the prefix, and walks the
  resolution chain. We just skip the cryptographic check.
- **Passthrough by default, structure when asked for.** The MVP
  request path is a clean passthrough. We add structure (ADK
  agents, memory, tools) only when a concrete use case shows up,
  and we add it *as a modelset implementation* — not by wrapping
  every call.

## How this compares to OpenRouter

OpenRouter is the closest public reference. We copy the parts of
its shape that matter (OpenAI compatibility, project + key model,
provider routing) and diverge where we have a reason:

| Concern | OpenRouter | railroad |
|---|---|---|
| Public API | OpenAI-compatible | OpenAI-compatible |
| Auth | Account + key | Project + key, with explicit per-key config |
| Provider routing | Auto-route to cheapest / fastest | Explicit CEL rules per modelset, evaluated per request |
| Memory / state | None first-class | Deferred (memory sets can return as a modelset-time feature) |
| Agents | None | Pluggable as a future modelset kind (ADK) |
| Hosting | SaaS | Self-hostable; one binary, one docker compose |
| Provider creds | Their own | Per-project credentials (future) or single shared creds (MVP) |
| Open source | No | Yes (intent) |
