# 09 — Configuration

This document describes the runtime configuration system: how
API keys resolve to model + system prompt + sampling defaults,
and how **modelsets** carry the routing logic.

The configuration system is the single most important abstraction
in railroad. It is what makes "swap the model", "route by prompt
size", or "add a new candidate" a database change rather than a
code change.

## What a configuration is

A `configuration` row is a named bundle of runtime settings
attached to one or more API keys. It carries the per-request
defaults that should apply when a client doesn't override them in
the chat completion body, and it points at a **modelset** for
model selection.

```yaml
# the shape of a configuration, conceptually
id: 9c5c...
project_id: 7d8f...
name: "Customer Support Bot"
modelset_id: "mset_a3b4..."     # see "What a modelset is" below
system_prompt: |
  You are a customer support agent for ACME.
  Always be polite. Never promise refunds over $50.
  If you don't know, escalate to a human.
default_temperature: 0.2
default_top_p: 1.0
default_max_output_tokens: 1024
metadata:
  team: "support"
  contact: "support@acme.example"
```

## Resolution order for chat completion fields

For each request, the model + sampling params + system prompt are
resolved in this order (later overrides earlier):

1. **The modelset resolver picks the model.** The `model` field
   in the request is *not* authoritative — the modelset is.
   See "What the request's `model` field means" below.
2. **Compile-time defaults** for the chosen model (from the
   `model` table's `default_*` columns — not in MVP schema; if
   absent, the provider's own defaults are used).
3. **Configuration defaults** — the `configuration`'s
   `default_*` columns.
4. **OpenAI request body fields** — `temperature`, `top_p`,
   `max_tokens`, etc.
5. **System prompt** is similarly layered. The OpenAI request may
   include a system message in `messages[]`; we **append** the
   configuration's system prompt to it (config prompt last so it
   has the last word). MVP always appends.

## What the request's `model` field means

The OpenAI request body's `model` field is *informational*, not
authoritative. The actual model used is whatever the modelset
resolver returns. Concretely:

- If the request's `model` matches a candidate in the resolved
  modelset, the resolver may use the CEL to confirm or override.
  The resolver's choice wins.
- If the request's `model` does **not** match any candidate in
  the modelset, we still proceed (CEL might be dynamic, and the
  client doesn't have visibility into modelset internals). The
  `model` field is recorded in the access log and in
  `request_log.requested_model`.
- Future: we may add a per-configuration "lock to a single
  model" mode that 400s if the request's `model` doesn't match
  the locked model. Not in MVP.

This shape lets us change routing without clients noticing. A
client sending `model: "google/gemini-2.5-pro"` may get routed
to `google/gemini-2.5-flash` by the modelset. The response
includes the actual model used in the `model` field, so the
client can see what happened.

## What a modelset is

A `modelset` row is a named, project-scoped bundle of:

- An **ordered list of model candidates**, each referencing a row
  in the `model` registry. Candidates are tried in position order
  by default, but the CEL can pick any position.
- A **CEL expression** that returns the position of the candidate
  to use. If absent, the resolver returns the first eligible
  candidate in preference order.

```yaml
# the shape of a modelset, conceptually
id: mset_a3b4...
project_id: 7d8f...
name: "Default routing"

cel_source: |
  size(request.messages[-1].content) > 4000 ? 0 : 1

candidates:
  - position: 0
    model_id: "google/gemini-2.5-pro"
    eligibility_cel: null
  - position: 1
    model_id: "google/gemini-2.5-flash"
    eligibility_cel: null
```

For full details on the resolver, the CEL environment, the
eligibility predicates, and the future ADK shim, see
[06-modelset](./06-modelset.md).

## The model registry

The `model` table is the source of truth for what model ids the
public API accepts. A few rules:

- The public `id` is what clients put in `model=`. Format:
  `<provider>/<upstream-id>`, e.g. `google/gemini-2.5-pro`.
- The `provider` is one of the registered provider names.
- The `provider_model_id` is the id passed to the upstream API.
- `status` is `active` for usable models, `deprecated` for
  models that should warn but still work, `disabled` for models
  that return 404. We default to `active`.
- New models are added by inserting rows (admin endpoint or
  migration). They are picked up within the cache TTL
  (10 minutes for the model cache) or on next process restart.

The `GET /v1/models` endpoint returns all `active` rows. We do
not (yet) filter by project.

## API key → configuration

The chain is:

```
api_key.configuration_id   →  configuration.id
configuration.modelset_id  →  modelset.id
modelset_candidate.model_id → model.id
```

The chain is **eagerly resolved** at auth time. The `Resolved`
struct (see [07-auth-and-tenancy](./07-auth-and-tenancy.md)) has
the `APIKey`, `Project`, `Account`, `Configuration`, `Modelset`,
and the list of `Candidate`s materialized. The handler never
re-resolves during the request.

We also store a denormalized cache entry on `api_key` (the prefix
lookup hits) so we can invalidate just this chain without
flushing the whole `api_key` cache.

## Changing a configuration

Configurations are mutable. When you change a configuration's
`modelset_id`:

- New requests with the same API key will use the new modelset
  within the cache TTL (5 minutes for the `rr:cfg:<id>` cache).
- Existing in-flight requests are unaffected (they already have
  the modelset in their `Resolved` struct).

When you change a modelset's `cel_source` or its candidates:

- Same TTL behavior.
- The new CEL compiles lazily on the next request that uses the
  modelset; the compile error (if any) is rejected at write
  time, so the request path never sees a bad CEL.

There is no "version" on a configuration or modelset for MVP. If
we need versioning later, we add a `version` integer and a join
table that pins API keys to a specific version.

## Defaults if a configuration is missing

The auth path returns an error if the API key has no
configuration. There is no implicit fallback to a project
default in MVP. If we want one, we add a
`project.default_configuration_id` column and use it when
`api_key.configuration_id IS NULL`. **DECIDE**.

## Example: setting up a project

End-to-end admin flow for a new project (all via internal
endpoints; for MVP this is curl + admin token):

```bash
# 1. Create the account
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{"email":"alice@example.com","display_name":"Alice"}' \
    http://localhost:8081/internal/v1/accounts
# → {"id": "acc_..."}

# 2. Create the project
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{"name":"ACME Support"}' \
    http://localhost:8081/internal/v1/projects
# → {"id": "prj_..."}

# 3. Add the account to the project
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{"account_id":"acc_...","role":"owner"}' \
    http://localhost:8081/internal/v1/accounts/acc_.../projects

# 4. Create a modelset
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{
      "name": "Default routing",
      "cel_source": "size(request.messages[-1].content) > 4000 ? 0 : 1",
      "candidates": [
        {"position": 0, "model_id": "google/gemini-2.5-pro"},
        {"position": 1, "model_id": "google/gemini-2.5-flash"}
      ]
    }' \
    http://localhost:8081/internal/v1/projects/prj_.../modelsets
# → {"id": "mset_..."}

# 5. Create a configuration referencing the modelset
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{
      "name":"Support Bot",
      "modelset_id":"mset_...",
      "system_prompt":"You are a customer support agent for ACME."
    }' \
    http://localhost:8081/internal/v1/projects/prj_.../configurations
# → {"id": "cfg_..."}

# 6. Create an API key bound to the configuration
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{"name":"web-app","configuration_id":"cfg_..."}' \
    http://localhost:8081/internal/v1/projects/prj_.../api-keys
# → {"id": "key_...", "raw_key": "rr_live_<32 chars>"}  ← only time the raw key is shown
```

The user (Alice) now uses `rr_live_...` against
`POST /v1/chat/completions` and gets responses from the support
bot configuration. The modelset picks Pro for long prompts,
Flash for short ones, all without the client knowing.

## What's intentionally missing

- **Memory sets.** Removed from MVP per the design decision.
  Adding memories back is a future modelset-time feature
  (see [06-modelset](./06-modelset.md) "Future: memory sets as
  a modelset-time feature") and does not require changing the
  configuration shape.
- **Tool calling.** Not surfaced in MVP. The provider
  abstraction and the `tool_allowlist` column on `configuration`
  exist for the day we do; today, clients sending `tools` get a
  501. (See [04-api-surface](./04-api-surface.md).)
- **Per-tenant encryption of the system prompt.** Not needed
  for MVP; the `system_prompt` column is plaintext. A future
  per-project "secrets in config" feature would add encryption.

## Future

- **Per-config tool allowlist.** Today, `configuration.tool_allowlist`
  is a jsonb blob in the schema but not honored. Future: parse
  the list, expose to the provider as allowed tool names, reject
  unknown tools with 400.
- **Per-config streaming override.** Some configurations might
  force non-streaming for log/audit reasons.
- **Configuration inheritance.** A project could have a base
  configuration; child configurations override specific fields.
  Useful for org-wide model policies.
- **Cost budgets per configuration.** A monthly token cap; the
  auth middleware checks before admitting a request.
- **Model aliasing.** `production-safe → google/gemini-2.5-pro`
  indirection so we can rename models without breaking clients.
- **CEL variables: cost, latency, model capability.** As the
  data shape grows, the CEL environment grows too — the
  resolver declares what variables exist; the expressions
  read them.
- **Memory sets as a modelset-time feature.** Re-introduce
  `memory_set` as a config field and a CEL variable
  `memories`; the handler injects them as a system-prompt
  prefix. Not in MVP.
