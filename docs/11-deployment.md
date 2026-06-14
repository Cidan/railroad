# 11 — Deployment

This document covers how to build, run, configure, and ship
railroad. The MVP is a single Go binary; everything else
(Postgres, Valkey) is a dependency you run separately or as
part of a docker compose for local dev.

## Build

### From source

```bash
git clone https://github.com/Cidan/railroad
cd railroad
go build -o ./bin/railroad ./cmd/railroad
```

The binary is statically linked by default (Go's default for
`go build`). The Dockerfile below produces a
`FROM scratch` image containing just the binary and a CA
cert bundle.

### Make targets

```
make build          # go build → ./bin/railroad
make test           # go test ./...
make lint           # golangci-lint run
make migrate-up     # apply pending migrations
make migrate-status # show migration state
make run            # build + start
make compose-up     # docker compose up (Postgres + Valkey)
make compose-down   # docker compose down
make docker        # build container image
make tidy          # go mod tidy
```

### Dockerfile

Multi-stage. The final image is `scratch` with ca-certificates
and tzdata:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.26.4-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/railroad ./cmd/railroad

FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=build /out/railroad /railroad
EXPOSE 8080 8081
ENTRYPOINT ["/railroad"]
```

The image is ~20 MB.

## Runtime modes

```
railroad serve      # default; starts the HTTP servers
railroad migrate    # runs the migrations CLI (up/down/status/...)
railroad version    # prints version info
railroad config     # prints the resolved configuration (with secrets redacted)
```

The `serve` command is the default if no subcommand is given.

## Configuration

All configuration is environment variables. No config file is
required for MVP. The prefix is `RAILROAD_` for app config; we
also read standard `OTEL_*` and `GO_*` env vars for
observability and runtime tuning.

### Server

| var | default | meaning |
|---|---|---|
| `RAILROAD_HTTP_ADDR` | `:8080` | bind address for the public API |
| `RAILROAD_INTERNAL_ADDR` | `:8081` | bind address for the internal/admin API |
| `RAILROAD_INTERNAL_ENABLED` | `true` | whether to start the internal server at all |
| `RAILROAD_TLS_CERT` / `RAILROAD_TLS_KEY` | unset | if set, serves HTTPS |
| `RAILROAD_BODY_LIMIT_BYTES` | `10485760` | max request body size (10 MB) |
| `RAILROAD_REQUEST_TIMEOUT` | `60s` | per-request deadline for sync handlers |
| `RAILROAD_STREAM_TIMEOUT` | `5m` | per-request deadline for streaming handlers |
| `RAILROAD_LOG_LEVEL` | `info` | `debug`/`info`/`warn`/`error` |
| `RAILROAD_LOG_FORMAT` | `json` | `json` or `text` (text is for local dev) |
| `RAILROAD_VERSION` | build flag | version string for the `X-Railroad-Version` header |

### Postgres

| var | default | meaning |
|---|---|---|
| `RAILROAD_PG_DSN` | required | libpq-style DSN, e.g. `postgres://railroad:...@db:5432/railroad?sslmode=require` |
| `RAILROAD_PG_MAX_CONNS` | `25` | pgx pool max |
| `RAILROAD_PG_MIN_CONNS` | `2` | pgx pool min |
| `RAILROAD_PG_STATEMENT_TIMEOUT` | `30s` | per-statement timeout |
| `RAILROAD_AUTO_MIGRATE` | `false` | run migrations on `serve` start (dev only) |

### Valkey

| var | default | meaning |
|---|---|---|
| `RAILROAD_VALKEY_ADDRS` | required | comma-separated `host:port` list |
| `RAILROAD_VALKEY_PASSWORD` | unset | |
| `RAILROAD_VALKEY_DB` | `0` | |
| `RAILROAD_VALKEY_TLS` | `false` | enable TLS |
| `RAILROAD_VALKEY_DISABLE_CACHE` | `false` | disable server-assisted client-side caching |

### Auth

| var | default | meaning |
|---|---|---|
| `RAILROAD_AUTH_MODE` | `stub` | `stub` or `verify` |
| `RAILROAD_ADMIN_TOKEN` | required for internal API | bearer token for `/internal/...` |
| `RAILROAD_API_KEY_CACHE_TTL` | `60s` | API key cache TTL |
| `RAILROAD_RATE_LIMIT_RPM` | `1000` | per-key requests per minute |

### Provider (Gemini)

| var | default | meaning |
|---|---|---|
| `GEMINI_API_KEY` | required | the upstream API key |
| `RAILROAD_GEMINI_BASE_URL` | unset | override (mostly for testing against a mock) |
| `RAILROAD_GEMINI_TIMEOUT` | `60s` | per-call timeout |
| `RAILROAD_GEMINI_RETRY_MAX` | `1` | retries on 5xx / 429 |

### Modelset / CEL

| var | default | meaning |
|---|---|---|
| `RAILROAD_CEL_MAX_EXPR_LENGTH` | `4096` | reject CEL expressions longer than this at write time |
| `RAILROAD_CEL_MAX_EVAL_DURATION` | `5ms` | per-request budget for CEL evaluation; expressions that exceed it are treated as `eval_error` and the resolver falls back to the first eligible candidate |
| `RAILROAD_CEL_MAX_PROGRAMS` | `10000` | soft cap on the in-process compiled-program cache; oldest entries are evicted past this |

### Observability

Standard OTel env vars:

- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_PROTOCOL` (`grpc` or `http/protobuf`)
- `OTEL_SERVICE_NAME` (defaults to `railroad`)
- `OTEL_RESOURCE_ATTRIBUTES`
- `OTEL_TRACES_SAMPLER` / `OTEL_TRACES_SAMPLER_ARG`

App-specific:

- `RAILROAD_LOG_REQUEST_BODIES` (`false`) — when `true`, writes
  bodies to `request_log_body`. Off by default for privacy.

## Local development

### With docker compose

`docker-compose.yml` at the repo root:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: railroad
      POSTGRES_PASSWORD: railroad
      POSTGRES_DB: railroad
    ports: ["5432:5432"]
    volumes: ["./.data/pg:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U railroad"]
      interval: 5s

  valkey:
    image: valkey/valkey:7.2
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 5s

  # railroad is run from `make run` in dev, not via compose
```

`.env.development` (gitignored):

```
RAILROAD_PG_DSN=postgres://railroad:railroad@localhost:5432/railroad?sslmode=disable
RAILROAD_VALKEY_ADDRS=localhost:6379
RAILROAD_AUTH_MODE=stub
RAILROAD_ADMIN_TOKEN=devtoken
GEMINI_API_KEY=<your key>
```

Then:

```bash
make compose-up        # postgres + valkey
cp .env.example .env.development
$EDITOR .env.development
make migrate-up        # apply schema
make run               # starts railroad on :8080 and :8081
```

The `make run` target sources `.env.development` and runs the
binary with `RAILROAD_AUTO_MIGRATE=true` so you can iterate
on migrations.

### Smoke test

```bash
# Create a project, modelset, configuration, and key (in real usage, via
# the internal API; in dev, seed directly):
psql $RAILROAD_PG_DSN -c "INSERT INTO project (id, name) VALUES ('11111111-1111-1111-1111-111111111111', 'dev');"
psql $RAILROAD_PG_DSN -c "INSERT INTO model (id, provider, provider_model_id, display_name, context_window, max_output_tokens) VALUES ('google/gemini-2.5-pro','gemini','gemini-2.5-pro','Gemini 2.5 Pro', 1048576, 65536), ('google/gemini-2.5-flash','gemini','gemini-2.5-flash','Gemini 2.5 Flash', 1048576, 65536);"

psql $RAILROAD_PG_DSN -c "INSERT INTO modelset (id, project_id, name, cel_source) VALUES ('44444444-4444-4444-4444-444444444444', '11111111-1111-1111-1111-111111111111', 'default', NULL);"
psql $RAILROAD_PG_DSN -c "INSERT INTO modelset_candidate (modelset_id, position, model_id) VALUES ('44444444-4444-4444-4444-444444444444', 0, 'google/gemini-2.5-pro'), ('44444444-4444-4444-4444-444444444444', 1, 'google/gemini-2.5-flash');"

psql $RAILROAD_PG_DSN -c "INSERT INTO configuration (id, project_id, name, modelset_id) VALUES ('22222222-2222-2222-2222-222222222222', '11111111-1111-1111-1111-111111111111', 'dev', '44444444-4444-4444-4444-444444444444');"
psql $RAILROAD_PG_DSN -c "INSERT INTO api_key (id, project_id, name, prefix, key_hash) VALUES ('33333333-3333-3333-3333-333333333333', '11111111-1111-1111-1111-111111111111', 'dev', 'rr_live_AAAAAAAA', '\\x00');"

# Use it
curl -sS http://localhost:8080/v1/chat/completions \
    -H "Authorization: Bearer rr_live_AAAAAAAABBBBBBBBCCCCCCCCDDDDDDDD" \
    -H "Content-Type: application/json" \
    -d '{
      "model": "google/gemini-2.5-pro",
      "messages": [{"role":"user","content":"Hello, who are you?"}]
    }'
```

In stub mode, the bearer value can be anything well-formed. The
`model` field in the request is informational; the modelset's
CEL (or first-candidate default) decides which model is actually
called. The response's `model` field will show the resolved
model.

In stub mode, the bearer value can be anything well-formed.

## Production deployment

### Single binary, behind a load balancer

A typical setup:

- 1+ railroad instances (stateless except for the Postgres
  connection and Valkey connection)
- A Postgres primary + at least one replica
- A Valkey primary + replicas (or a managed service)
- An HTTPS-terminating load balancer in front
- A Postgres + Valkey network-isolated subnet

### Kubernetes (sketch)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: railroad }
spec:
  replicas: 3
  selector: { matchLabels: { app: railroad } }
  template:
    metadata: { labels: { app: railroad } }
    spec:
      containers:
        - name: railroad
          image: ghcr.io/cidan/railroad:v0.1.0
          ports:
            - { containerPort: 8080, name: http }
            - { containerPort: 8081, name: internal }
          env:
            - { name: RAILROAD_PG_DSN, valueFrom: { secretKeyRef: { name: railroad-pg, key: dsn } } }
            - { name: RAILROAD_VALKEY_ADDRS, value: valkey:6379 }
            - { name: RAILROAD_AUTH_MODE, value: verify }
            - { name: RAILROAD_ADMIN_TOKEN, valueFrom: { secretKeyRef: { name: railroad-admin, key: token } } }
            - { name: GEMINI_API_KEY, valueFrom: { secretKeyRef: { name: railroad-gemini, key: key } } }
            - { name: OTEL_EXPORTER_OTLP_ENDPOINT, value: http://otel-collector:4317 }
          readinessProbe:
            httpGet: { path: /readyz, port: http }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { cpu: 2,    memory: 1Gi }
---
apiVersion: v1
kind: Service
metadata: { name: railroad }
spec:
  selector: { app: railroad }
  ports:
    - { name: http,     port: 443, targetPort: 8080 }
    - { name: internal, port: 8081, targetPort: 8081 }
```

The `internal` port should be exposed only on a private network
or firewalled off entirely if you don't need a remote admin.

### Cloud Run / Fly.io / etc.

The same container works on any platform that runs a long-lived
container. Notable tweaks:

- The container must allow long-lived streaming responses;
  Cloud Run defaults are 5 minutes; that's our default too.
- The internal port should not be reachable from the public
  internet. Cloud Run can do this by binding services to a
  separate internal-only service.

## Configuration as deployed

The MVP reads everything from env vars. There is no config file
in production. This is intentional: env vars are the most
12-factor-correct option and avoid the "is the file mounted?"
class of bug.

If we need runtime config (e.g. toggling a provider without a
deploy), the right answer is a small admin endpoint that writes
to a `config_overrides` table read at the start of every
request. Not MVP.

## Secrets

The MVP expects:

- `GEMINI_API_KEY` — from a secret store (GCP Secret Manager,
  Vault, AWS SSM, etc.) injected at deploy time.
- `RAILROAD_PG_DSN` — same.
- `RAILROAD_ADMIN_TOKEN` — same.

For local dev, a `.env.development` file is fine. The
`.env.development` file is gitignored. The `.env.example`
checked in shows the shape without values.

## Versioning and releases

- Git tag = release. `v0.1.0`, `v0.1.1`, etc.
- Container image: `ghcr.io/cidan/railroad:<tag>` and
  `ghcr.io/cidan/railroad:latest` (for the default branch).
- `cmd/railroad/main.go` sets `var Version = "dev"` at build
  time; the Dockerfile passes `-ldflags="-X main.Version=$(git describe)"`.
- The version is exposed in the `X-Railroad-Version` response
  header and in the log `version` field.

## CI (sketch)

A future PR will add `.github/workflows/ci.yml`:

- On PR: `go test ./...`, `golangci-lint run`, `go mod tidy` check.
- On merge to `main`: build + push the container image, run
  integration tests against a real Postgres + Valkey.

Not MVP for the planning phase; the Makefile targets make
local iteration straightforward.

## Upgrades

Because migrations are append-only and the binary reads
schema-permissioned queries, the upgrade procedure is:

1. Build the new image.
2. Run `railroad migrate up` (a one-shot Job).
3. Roll the new image out.

The MVP does not require downtime. Migrations are
forward-compatible with the old binary: a new column is
ignored by the old code, and we never rename or drop columns
in a single migration.

## Operational notes

- **First-run seeding.** The `model` table is seeded by
  migration 000002. No other data is auto-created. The first
  account + project + key must be created via the internal
  API.
- **Time source.** All timestamps are UTC. The container's
  `TZ` is irrelevant; the binary uses `time.Now().UTC()`.
- **Backpressure.** Streaming responses use the kernel TCP
  buffer. A slow client will eventually cause the writer to
  block, which propagates as latency to the provider call. We
  do not cancel upstream calls because of slow clients in MVP.
  **DECIDE** — see [13-decisions](./13-decisions.md).
- **Memory leaks.** The process is expected to be restarted
  occasionally (every few days) to reclaim RSS from long-lived
  goroutine state. We do not have any known leaks. A weekly
  cron-driven rolling restart is a cheap insurance policy.
