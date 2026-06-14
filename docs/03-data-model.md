# 03 — Data model

This document describes the entities in Postgres that back railroad.
The Go types in `internal/domain` mirror these one-to-one, with
validation that Postgres cannot express (e.g. "at least one project
per account").

## ERD

```
   account ──< account_project >── project ──< api_key ──> configuration ──> modelset ──< modelset_candidate ──> model
                                                                                │
                                                                                └── (CEL expression lives on modelset)
                                       │
                                       └──< request_log
```

Relationships:

- `account 1..N — N..1 project` (via `account_project`)
- `project 1..N api_key`
- `project 1..N configuration`
- `project 1..N modelset`
- `api_key N..1 configuration` (each key points at exactly one config;
  a config can be pointed at by many keys)
- `configuration N..1 modelset` (each config references one modelset;
  a modelset can be referenced by many configs)
- `modelset 1..N modelset_candidate` (ordered list of candidates)
- `modelset_candidate N..1 model`
- `api_key 1..N request_log`
- `project 1..N request_log`

The asymmetry — "account must belong to at least one project" — is
enforced in application code, not by a Postgres constraint, because
m2m membership with a min-cardinality constraint is awkward in SQL.

## Tables

Times are stored as `timestamptz` (UTC). All IDs are `uuid v7` so
they sort by creation time and are k-safe for distributed generation.

### `account`

| column | type | notes |
|---|---|---|
| `id` | `uuid` PK | |
| `email` | `citext` UNIQUE NOT NULL | the user's contact |
| `display_name` | `text` NOT NULL | |
| `status` | `text` NOT NULL DEFAULT 'active' | `active`, `suspended`, `deleted` |
| `created_at` | `timestamptz` NOT NULL DEFAULT now() | |
| `updated_at` | `timestamptz` NOT NULL DEFAULT now() | trigger-maintained |

Index: `account_email_idx` on `lower(email)`.

### `project`

| column | type | notes |
|---|---|---|
| `id` | `uuid` PK | |
| `name` | `text` NOT NULL | |
| `status` | `text` NOT NULL DEFAULT 'active' | |
| `created_at` | `timestamptz` NOT NULL | |
| `updated_at` | `timestamptz` NOT NULL | |

### `account_project`

The m2m join. Carries a role for future use.

| column | type | notes |
|---|---|---|
| `account_id` | `uuid` FK → `account(id)` ON DELETE CASCADE | |
| `project_id` | `uuid` FK → `project(id)` ON DELETE CASCADE | |
| `role` | `text` NOT NULL DEFAULT 'member' | `owner`, `member`, `viewer` (future) |
| `joined_at` | `timestamptz` NOT NULL DEFAULT now() | |

PK: `(account_id, project_id)`.

**Invariant** (application-enforced): an `account` with zero rows in
`account_project` is a violation. Enforced on insert/delete in the
repository.

### `api_key`

The credential presented by clients.

| column | type | notes |
|---|---|---|
| `id` | `uuid` PK | internal id, never exposed |
| `project_id` | `uuid` FK → `project(id)` NOT NULL | scope |
| `created_by_account_id` | `uuid` FK → `account(id)` NOT NULL | audit trail |
| `name` | `text` NOT NULL | user-given label |
| `prefix` | `text` NOT NULL | public, e.g. `rr_live_AbCdEfGh`. UNIQUE. |
| `key_hash` | `bytea` NOT NULL | argon2id of the full key. For MVP stub mode, store a sentinel (`0x00...`) and skip verification. |
| `key_hash_algo` | `text` NOT NULL DEFAULT 'argon2id' | future-proofing |
| `configuration_id` | `uuid` FK → `configuration(id)` | which config to use; can be NULL to mean "use project default" (future) |
| `status` | `text` NOT NULL DEFAULT 'active' | `active`, `revoked` |
| `created_at` | `timestamptz` NOT NULL | |
| `last_used_at` | `timestamptz` | updated on each successful auth |
| `expires_at` | `timestamptz` | NULL = no expiry |

Indexes:
- `api_key_prefix_idx` on `prefix` (the lookup path)
- `api_key_project_idx` on `(project_id, status)`

**Never** store the raw key. The plain key is returned **once** at
creation time and never again.

### `configuration`

A named set of runtime settings attached to one or more API keys.
Owns the system prompt and default sampling params; delegates
model selection to a `modelset`.

| column | type | notes |
|---|---|---|
| `id` | `uuid` PK | |
| `project_id` | `uuid` FK → `project(id)` NOT NULL | |
| `name` | `text` NOT NULL | |
| `modelset_id` | `uuid` FK → `modelset(id)` NOT NULL | which modelset to route through |
| `system_prompt` | `text` | NULL = no override |
| `default_temperature` | `numeric(3,2)` | NULL = pass through to provider default |
| `default_top_p` | `numeric(3,2)` | |
| `default_max_output_tokens` | `integer` | |
| `tool_allowlist` | `jsonb` | future: list of tool names this config may call |
| `metadata` | `jsonb` NOT NULL DEFAULT '{}' | free-form, opaque to the app |
| `created_at` | `timestamptz` NOT NULL | |
| `updated_at` | `timestamptz` NOT NULL | |

Index: `(project_id, name)` unique partial where status active.

The `modelset_id` is required; we don't ship a per-key model
override (yet). If a key needs a different model, the operator
creates a new configuration.

### `modelset`

A named, project-scoped routing decision. Holds an ordered list
of model candidates and a CEL expression that picks one. The full
shape of the resolver is in [06-modelset](./06-modelset.md).

| column | type | notes |
|---|---|---|
| `id` | `uuid` PK | |
| `project_id` | `uuid` FK → `project(id)` NOT NULL | |
| `name` | `text` NOT NULL | |
| `description` | `text` | |
| `cel_source` | `text` | the CEL expression source. NULL = "always use candidates[0]" (the first candidate, after eligibility filtering). |
| `metadata` | `jsonb` NOT NULL DEFAULT '{}' | free-form |
| `created_at` | `timestamptz` NOT NULL | |
| `updated_at` | `timestamptz` NOT NULL | |

We do **not** persist the compiled CEL program. We compile from
`cel_source` on first use and cache the program in process memory
per `modelset_id`. Compile is single-digit ms.

### `modelset_candidate`

The ordered list of model candidates belonging to a modelset.

| column | type | notes |
|---|---|---|
| `modelset_id` | `uuid` FK → `modelset(id)` ON DELETE CASCADE NOT NULL | |
| `position` | `integer` NOT NULL CHECK (position >= 0) | 0-based preference order; lower = preferred |
| `model_id` | `text` FK → `model(id)` NOT NULL | |
| `weight` | `numeric(5,2)` NOT NULL DEFAULT 1.0 | for CEL expressions that consult weight; not used by the default resolver |
| `eligibility_cel` | `text` | per-candidate guard; if present and evaluates to false, the candidate is filtered out before the main CEL runs |

PK: `(modelset_id, position)`.

A modelset must have at least one candidate; enforced in the
application (a `CHECK` constraint with a subquery is awkward and
we'd repeat it for every insert path).

### `model` (the registry)

Static-ish metadata about the models we expose to clients. The
`id` is the public identifier (e.g. `google/gemini-2.5-pro`,
`anthropic/claude-3-5-sonnet-latest`, etc.) that clients put in
`model=`.

| column | type | notes |
|---|---|---|
| `id` | `text` PK | the OpenAI-style public id |
| `provider` | `text` NOT NULL | `gemini`, `openai`, `anthropic`, ... |
| `provider_model_id` | `text` NOT NULL | the id the upstream API uses |
| `display_name` | `text` NOT NULL | |
| `context_window` | `integer` NOT NULL | |
| `max_output_tokens` | `integer` NOT NULL | |
| `supports_tools` | `boolean` NOT NULL DEFAULT false | |
| `supports_vision` | `boolean` NOT NULL DEFAULT false | |
| `supports_streaming` | `boolean` NOT NULL DEFAULT true | |
| `input_cost_per_million` | `numeric(12,6)` | future billing / cost-aware routing |
| `output_cost_per_million` | `numeric(12,6)` | future billing / cost-aware routing |
| `status` | `text` NOT NULL DEFAULT 'active' | `active`, `deprecated`, `disabled` |
| `created_at` | `timestamptz` NOT NULL | |
| `updated_at` | `timestamptz` NOT NULL | |

Seeded by a migration (idempotent `INSERT ... ON CONFLICT DO NOTHING`)
and editable via an admin endpoint. New providers = new rows.

### `request_log`

One row per inbound request. Used for usage tracking, debugging, and
audit. We log metadata only by default; opt-in for content.

| column | type | notes |
|---|---|---|
| `id` | `uuid` PK | |
| `trace_id` | `text` | matches OTel trace id |
| `api_key_id` | `uuid` FK → `api_key(id)` NOT NULL | |
| `project_id` | `uuid` FK → `project(id)` NOT NULL | denormalized for fast project-scoped queries |
| `account_id` | `uuid` FK → `account(id)` NOT NULL | the account that owns the key |
| `modelset_id` | `uuid` FK → `modelset(id)` | which modelset routed this request |
| `requested_model` | `text` | the public id the client asked for |
| `resolved_model_id` | `text` | the public id we actually called (after CEL routing) |
| `resolved_provider` | `text` | `gemini`, etc. |
| `resolved_provider_model_id` | `text` | the upstream id |
| `candidate_position` | `integer` | which position in the modelset was chosen (0-based) |
| `stream` | `boolean` NOT NULL | |
| `status_code` | `integer` NOT NULL | |
| `error_code` | `text` | railroad's error code, NULL on success |
| `prompt_tokens` | `integer` | from upstream usage metadata |
| `completion_tokens` | `integer` | |
| `cached_tokens` | `integer` | upstream-reported cached |
| `routing_latency_ms` | `integer` | time spent in the modelset resolver (CEL eval) |
| `latency_ms` | `integer` NOT NULL | wall clock from auth-complete to response-complete |
| `created_at` | `timestamptz` NOT NULL DEFAULT now() | |

Index: `(project_id, created_at DESC)` for the project usage view.
Index: `(modelset_id, created_at DESC)` for per-modelset analytics.

The body of the request and response is **not** stored in MVP. It
can be turned on with an env var (`RAILROAD_LOG_REQUEST_BODIES=true`)
for debugging, which writes to a separate `request_log_body` table
to keep `request_log` cheap.

### `provider_credential`

Future. For MVP, we use a single global Gemini API key from the
env and ignore this table. The shape is here so the schema doesn't
need to migrate when we wire per-project credentials.

| column | type | notes |
|---|---|---|
| `id` | `uuid` PK | |
| `project_id` | `uuid` FK → `project(id)` | NULL = global |
| `provider` | `text` NOT NULL | |
| `credential_type` | `text` NOT NULL | `api_key`, `service_account_json`, `oauth_token` |
| `encrypted_value` | `bytea` NOT NULL | AES-256-GCM with a KEK from the env |
| `expires_at` | `timestamptz` | |
| `created_at` | `timestamptz` NOT NULL | |
| `updated_at` | `timestamptz` NOT NULL | |

## Validation rules the database can't enforce

These live in `internal/store/...` and surface as Go errors:

1. `account` cannot have zero rows in `account_project` after any
   write. Implemented in `AccountRepository.DeleteMembership` (the
   last membership delete returns an error).
2. `api_key.prefix` is globally unique. The `UNIQUE` constraint
   handles this.
3. `api_key.key_hash` is required even in stub mode (use a sentinel
   value).
4. `configuration.project_id` must match the project that owns the
   `api_key` that references it. Checked in the application when an
   API key is created or its configuration is changed.
5. `modelset.project_id` must match the `configuration.project_id`
   that references it. Same check.
6. `modelset` must have at least one `modelset_candidate`. Enforced
   in the candidate repository (the last candidate delete returns
   an error, and modelset creation takes candidates in the same
   transaction).
7. `model.status` must be `active` for any model referenced by a
   `modelset_candidate`. Enforced on write and on read in the
   auth middleware (a deprecated model is treated as a 400 from
   the client, not a 500).
8. The CEL `cel_source` on `modelset` must compile. Enforced at
   write time in the modelset repository (we compile it; a compile
   error rejects the write).

## What we are **not** shipping in MVP

- **Memory sets / memory items.** Removed from MVP per design
  decision. The schema is clean to re-add later as a
  modelset-time feature (see [06-modelset](./06-modelset.md)
  "Future: memory sets as a modelset-time feature").
- **`pgvector` extension.** No embeddings in MVP, so no pgvector.
  This removes one extension dependency.
- **`adk_session` / `adk_event` tables.** No ADK in the request
  path. If we add ADK as a future modelset implementation, the
  tables come back at that time.

## Migrations

- Stored in `migrations/` as plain SQL files named
  `NNNNNN_description.sql` (zero-padded sequence number).
- Applied with `github.com/pressly/goose/v3` (**DECIDE** — see
  [13-decisions](./13-decisions.md)). Goose is simple, has a Go API
  we can call at startup or in a CLI subcommand, and works without
  a separate tool.
- Each migration is one `.sql` file with `-- +goose Up` /
  `-- +goose Down` directives.
- The startup sequence is: `railroad migrate up` then
  `railroad serve`. Migrations are not auto-run by `serve` in
  production but **are** in dev (gated by `RAILROAD_AUTO_MIGRATE=1`).
- The seed for the `model` table is itself a migration
  (`000002_seed_models.sql`).

The MVP migration set:

1. `000001_init.sql` — all tables above, plus the required
   Postgres extensions (`pgcrypto`, `citext`).
2. `000002_seed_models.sql` — Gemini 2.5 Pro and Flash.
3. (later, as needed) `000003_*.sql` — any schema change after
   MVP. Future migrations may re-introduce memory sets or
   `provider_credential` as separate, clearly-bounded changes.
