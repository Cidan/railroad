# 08 — Storage

This document covers how the application talks to Postgres and
Valkey: the connection topology, the schema, the migration
strategy, and the cache discipline. The data model itself is in
[03-data-model](./03-data-model.md); this is the runtime side of
those tables.

## Postgres

### Version

Postgres 16+ in production. We require:

- `uuid-ossp` (or `pgcrypto` for `gen_random_uuid()`)
- `citext` — for the `account.email` column

These ship as standard extensions; nothing exotic. We do
**not** require `pgvector` in MVP — there are no embeddings to
store. If a future phase re-introduces memory sets with
vector search, we add it then.

### Driver

`github.com/jackc/pgx/v5` via `pgxpool`. We use the native
protocol, prepared statements, and `COPY` where applicable. The
pool is configured at startup:

```go
cfg, _ := pgxpool.ParseConfig(dsn)
cfg.MaxConns = 25
cfg.MinConns = 2
cfg.MaxConnLifetime = 30 * time.Minute
cfg.MaxConnIdleTime = 5 * time.Minute
cfg.HealthCheckPeriod = 30 * time.Second
```

Defaults are conservative. They are environment-overridable
(`RAILROAD_PG_MAX_CONNS`, etc.).

### Schema

The schema is the set of tables from
[03-data-model](./03-data-model.md). It does not include ADK
session or event tables in MVP — there is no ADK in the
request path.

```sql
-- 000001_init.sql  (one migration, contains everything below)

CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS citext;

-- account, project, account_project
-- api_key, configuration, modelset, modelset_candidate
-- model, request_log
-- (all from 03-data-model.md, omitted here for brevity)

-- Optional: when RAILROAD_LOG_REQUEST_BODIES=true
CREATE TABLE request_log_body (
    request_log_id  uuid PRIMARY KEY REFERENCES request_log(id) ON DELETE CASCADE,
    request_body    jsonb,
    response_body   jsonb
);

-- 000002_seed_models.sql
INSERT INTO model (id, provider, provider_model_id, display_name, context_window, max_output_tokens, supports_tools, supports_vision, supports_streaming)
VALUES
    ('google/gemini-2.5-pro', 'gemini', 'gemini-2.5-pro', 'Gemini 2.5 Pro', 1048576, 65536, true, true, true),
    ('google/gemini-2.5-flash', 'gemini', 'gemini-2.5-flash', 'Gemini 2.5 Flash', 1048576, 65536, true, true, true)
ON CONFLICT (id) DO NOTHING;
```

**Migrations are append-only.** A migration that has been applied
to production is never edited. Corrections go in a new migration.

### Migrations tool

`github.com/pressly/goose/v3`. **DECIDE** — see
[13-decisions](./13-decisions.md). Other candidates:
`golang-migrate/migrate`, `ariga/atlas`. Goose is the simplest that
also has a Go API so we can embed it.

CLI subcommands:

```
railroad migrate up        # apply all pending
railroad migrate down      # roll back the last
railroad migrate status    # show applied / pending
railroad migrate redo      # down + up of the last
railroad migrate create NAME  # author a new migration file
```

In development, `RAILROAD_AUTO_MIGRATE=1` runs `migrate up` at
server start. In production, this is **off**; migrations are run
out-of-band (CI step, or a one-shot Job).

### Query conventions

- **Always use parameterized queries.** No `fmt.Sprintf` into SQL.
  Lint rule via `sqlclosecheck` + a custom analyzer (future).
- **Always scope to a project.** Every business query has
  `AND project_id = $N` as a defense in depth, even when the
  application has already filtered to a project_id in code.
- **Pagination uses keyset cursors, never `OFFSET`.** Every
  list endpoint accepts `?after=<opaque>`.
- **Time travel is via `created_at` only.** No
  `updated_at`-based incremental syncs; we don't need them in MVP.
- **Bulk operations are explicit.** No accidental N+1 from
  per-row queries inside a loop. The repository interface
  exposes `ListByProject(ctx, project, filter, cursor, limit)` and
  friends; if a caller needs bulk, the SQL is `WHERE id = ANY($1)`.

### Soft delete vs hard delete

For MVP we use **hard delete** with status columns. The
implementation:

- `account.status = 'deleted'` is a tombstone. The auth path
  rejects deleted accounts. Hard delete happens only via an
  admin command, never from the public API.
- `api_key.status = 'revoked'` is a tombstone. The auth path
  rejects revoked keys. Revoked keys are kept indefinitely
  (audit trail).
- `project.status = 'suspended' | 'deleted'` similarly.
- Rows are not physically removed.

This is simpler than `deleted_at` columns and works for our
operational needs.

### Connection pool sizing

A single railroad instance with default 25 connections can support
~250 in-flight requests at ~100ms per query, which is the
expected MVP load. If we grow past that, we either raise the
pool or scale horizontally. The pool is per-process, so
horizontal scaling is the natural answer.

### Read replicas

Not in MVP. The plan is to add a `RAILROAD_PG_READ_DSN` later and
route `SELECT`-only queries through it. The repository interface
is shaped to make this a thin wrapper change.

## Valkey

### Version

Valkey 7.2+. We use the OSS Valkey, not a hosted Redis service,
unless the deployer wires one up.

### Client

`github.com/valkey-io/valkey-go` (rueidis). Rationale:

- Auto-pipelining gives higher throughput under concurrency than
  go-redis.
- Server-assisted client-side caching is useful for the
  `api_key` lookup hot path.
- First-class OpenTelemetry instrumentation.
- Maintained by the Valkey project itself.

### Topology

A single Valkey node is fine for MVP. The rueidis client
auto-rediscovers the cluster topology if we later point it at a
cluster.

### What we store in Valkey

| Key pattern | Value | TTL | Source of truth |
|---|---|---|---|
| `rr:apikey:<prefix>` | JSON: `{key_id, project_id, configuration_id, hash_algo, hash, status, expires_at}` | 60s | `api_key` table |
| `rr:project:<id>` | JSON: `{name, status}` | 5 min | `project` table |
| `rr:cfg:<id>` | JSON: `{name, modelset_id, system_prompt, defaults...}` | 5 min | `configuration` table |
| `rr:ms:<id>` | JSON: `{name, cel_source, candidates: [...]}` | 5 min | `modelset` + `modelset_candidate` |
| `rr:model:<id>` | JSON: full `model` row | 10 min | `model` table |
| `rr:ratelimit:<key_id>:<window>` | integer (token count) | 60s | ephemeral |
| `rr:provider:gemini:models` | JSON list of Gemini model ids | 1 hr | `gemini` provider |
| `rr:apikey:idx:prefix:<prefix>` | `1` (a tombstone for negative caching) | 10s | ephemeral |
| compiled CEL programs | in-process `sync.Map` keyed on `modelset_id` | n/a | derived from `rr:ms:<id>.cel_source` |

### Cache discipline

- **Reads are best-effort.** A cache miss falls through to
  Postgres; a cache that lies (e.g. wrong data after a write) is
  OK because every cache entry has a TTL and the source of truth
  is Postgres.
- **Writes do not invalidate the cache eagerly.** A change to a
  project's configuration will be picked up within 5 minutes
  (the TTL). For actions that need immediate effect (key
  revocation), the API key prefix cache is bypassed (we go
  straight to Postgres) or the operator can
  `DEL rr:apikey:<prefix>` via a debug endpoint.
- **The cache is not authoritative for security decisions.** Even
  if a cache entry says "valid", the auth middleware re-validates
  `status = 'active'` and `expires_at > now()`.

### Negative caching

For unknown prefixes (an invalid key), we cache a tombstone for
10s. This protects against brute-force scans: 10s of retries per
prefix is bounded, and we still hit Postgres for the actual
verification, so a stolen-but-revoked key cannot be brute-forced
faster than the cache TTL.

### Connection pool

Rueidis manages its own pool. Defaults are fine; we override only
if we hit bottlenecks. Notable options:

- `ClientTrackingOptions`: enable for the model and configuration
  caches (server-assisted invalidation).
- `DisableAutoPipelining`: not used; we want the throughput.
- `RingScaleEachConn`: leave at default.

### Memory budget

For a small deployment, ~50 MB of Valkey is enough. We do not
cap Valkey memory in the design; it's the deployer's job to set
`maxmemory` and an eviction policy (recommendation:
`allkeys-lru`).

## Backups

- **Postgres**: WAL archiving + daily base backup. The deployer
  is responsible (we do not own the database). We do not write
  database backup logic.
- **Valkey**: RDB snapshot + AOF. Same — deployer's job.
- **request_log body table** if enabled is excluded from backup
  retention after 30 days (future cleanup job; not MVP).

## Disaster recovery

- **Postgres unreachable**: `/readyz` returns 503; load balancer
  removes us from the pool. No traffic = no data loss. When it
  comes back, Valkey caches warm back up over the next minute.
- **Valkey unreachable**: Auth and rate limiting degrade. We
  fail-closed on rate limiting (deny new requests for 30s) and
  fail-open on auth (fall through to Postgres for every request,
  which is slower but correct). Logged at ERROR.
- **Both down**: We serve 503 from `/readyz`. Better to be honest
  than to fail open.

## Capacity planning (a sketch)

For an MVP-sized deployment (single instance, 1k active keys,
~50 RPS sustained, ~500 RPS peak):

- Postgres: 1 small instance, 25 connections, 5 GB disk. Plenty.
- Valkey: 1 small instance, 100 MB `maxmemory`. Plenty.
- railroad: 1 instance, 512 MB RAM, 0.5 vCPU. Plenty.
- Bandwidth: ~5 MB/s out at peak (a few streaming completions
  each ~10 KB/s).

This is intentionally a small box. The architecture supports
horizontal scale-out when we get there.
