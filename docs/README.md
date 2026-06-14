# railroad — design plan

This folder contains the design plan for **railroad**, a multi-tenant AI
gateway written in Go. Railroad exposes an OpenAI-compatible HTTP API on
the front side and routes requests to upstream model providers (Gemini
first, then more) on the back side. Routing is driven by a **modelset**
— a small data structure with an ordered list of model candidates and a
CEL expression that picks one per request. The MVP request path is a
clean passthrough: no ADK, no agent, no memory in the critical path.

These documents describe the **target architecture**. They are not yet
implementation: we are still aligning on scope, terminology, and the
shape of the data model before code is written. Anything marked
**MVP** is in scope for the first shippable cut. Anything marked
**Future** is intentionally out of scope and tracked so we do not
re-litigate it later.

## Reading order

| # | Document | What it covers |
|---|----------|----------------|
| 01 | [Overview](./01-overview.md) | What railroad is, goals, non-goals, principles |
| 02 | [Architecture](./02-architecture.md) | High-level system diagram, request flow, process model |
| 03 | [Data model](./03-data-model.md) | Accounts, projects, API keys, configurations, modelsets, models |
| 04 | [API surface](./04-api-surface.md) | Public OpenAI-compatible endpoints, error shape, streaming |
| 05 | [Providers](./05-providers.md) | The provider abstraction and the Gemini implementation |
| 06 | [Modelset](./06-modelset.md) | The modelset resolver: candidates, CEL, eligibility, future ADK shim |
| 07 | [Auth and tenancy](./07-auth-and-tenancy.md) | API key format, stub auth, multi-tenant isolation rules |
| 08 | [Storage](./08-storage.md) | Postgres tables, Valkey keys, connection pooling, migrations |
| 09 | [Configuration](./09-configuration.md) | Configurations, modelsets, attachment semantics |
| 10 | [Observability](./10-observability.md) | Structured logs, metrics, distributed tracing, cost tracking |
| 11 | [Deployment](./11-deployment.md) | Build, env vars, Docker, local dev with docker-compose |
| 12 | [Roadmap](./12-roadmap.md) | Phased delivery plan from empty repo to first usable release |
| 13 | [Decisions](./13-decisions.md) | Architecture decision records and open questions |

## Status

This plan is **draft**. Sections are stable enough to start writing
code against, but expect revisions as we discover constraints. Anything
ambiguous is called out explicitly with **DECIDE** markers; please
raise them rather than guess.

## Module and target

- Go module: `github.com/Cidan/railroad`
- Go toolchain: 1.26.4 (current stable as of 2026-06-02)
- Gemini Go SDK: v1.60.0 (`github.com/googleapis/go-genai`) — used
  directly in the provider layer for model calls
- CEL: `github.com/google/cel-go` — compiled to a native Go program
  per modelset, evaluated per request in microseconds
- Valkey client: `github.com/valkey-io/valkey-go` (rueidis)
- Postgres driver: `github.com/jackc/pgx/v5`
- Migrations: `github.com/pressly/goose/v3` (TBD — see [13-decisions](./13-decisions.md))
- HTTP router: `net/http` (Go 1.22+ pattern routing) for the public API;
  `chi` for admin/internal routes if we need middleware composition
  (TBD — see [02-architecture](./02-architecture.md))
- ADK: **not** an MVP dependency. ADK-go v1.4.0 is reserved as a
  future modelset implementation, behind the `Resolver` interface
  in [06-modelset](./06-modelset.md).
