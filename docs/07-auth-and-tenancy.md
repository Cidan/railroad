# 07 — Auth and tenancy

The auth layer is the gatekeeper for every `/v1/...` request. It
parses the `Authorization` header, looks up the key, resolves the
full configuration chain (key → project → configuration →
modelset → candidates), and injects a `Resolved` struct into
the request context.

The MVP runs in **stub mode**: any well-formed key is accepted, no
cryptographic check is performed. The resolution chain is in place,
so the only change to enable real auth is one function.

## The API key format

```
rr_live_<32 base62 chars>
└┬┘ └─┬┘ └──────┬──────────┘
 │    │         └── random body (≈190 bits of entropy)
 │    └── env marker (`live` in MVP; future: `test`)
 └── literal prefix identifying the issuer
```

Total length: 41 characters. Always ASCII.

- **Prefix** `rr_live_` is the literal issuer tag. Future prefixes:
  `rr_test_` for test environment, `rr_int_` for an internal
  service-to-service flavor.
- **Body** 32 chars from `A-Za-z0-9`. We generate with
  `crypto/rand`. ~190 bits of entropy is plenty.
- **Lookup index** We do **not** store the full key. We store an
  argon2id hash. For lookup speed we also index by the **first 8
  characters of the body** (the `prefix` column in the
  `api_key` table) — globally unique. Lookup is
  `SELECT ... WHERE prefix = $1` then argon2id verify the body in
  constant time. This is the same approach GitHub, Stripe, and
  most key issuers use.

Why two lookups and not just hash-equal? Because the body is 32
chars, hashing the whole key to find a row in O(n) is untenable at
thousands of keys. The 8-char prefix gives us a small candidate
set (usually 1, occasionally a few collisions from random
selection) to verify in O(k) with the slow hash.

## Key creation

API keys are created through an internal endpoint
(`POST /internal/v1/api-keys`), which is itself authenticated
differently (see [Internal endpoints](#internal-endpoints) below).

```go
// internal/auth/key.go (sketch)

func NewKey() (raw string, prefix string, hash []byte, err error) {
    body := make([]byte, 32)
    if _, err := rand.Read(body); err != nil { return "", "", nil, err }
    encoded := base62.Encode(body) // 32 base62 chars
    raw = "rr_live_" + encoded
    prefix = "rr_live_" + encoded[:8]
    hash, err = argon2idHash(raw) // see KeyHash below
    return raw, prefix, hash, err
}
```

The full raw key is returned in the response **exactly once**.
Nothing else ever sees the raw key. The server stores `prefix` and
`hash`.

## KeyHash

```go
// Argon2id parameters chosen for ~50ms verification on a modern
// server. Adjust per hardware.
var argonParams = &argon2id.Params{
    Memory:      64 * 1024, // 64 MiB
    Iterations:  3,
    Parallelism: 1,
    SaltLength:  16,
    KeyLength:   32,
}
```

For MVP stub mode, we use a sentinel value (`0x00` byte followed by
the raw key in plaintext, or even just `0x00` for "any key") and
the `Verify` function short-circuits to `true`. The full machinery
is in place; flipping to real auth is one env var:

```
RAILROAD_AUTH_MODE=stub    # MVP
RAILROAD_AUTH_MODE=verify  # production
```

## The auth middleware

```go
// internal/api/middleware/auth.go (sketch)

func Auth(lookup KeyLookup, cfg Config) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            raw, err := parseBearer(r.Header.Get("Authorization"))
            if err != nil {
                openaiError(w, 401, "invalid_api_key", "missing or malformed Authorization header")
                return
            }
            k, err := lookup.Resolve(r.Context(), raw)
            if err != nil {
                if errors.Is(err, ErrNotFound) || errors.Is(err, ErrRevoked) {
                    openaiError(w, 401, "invalid_api_key", "API key is invalid or revoked")
                    return
                }
                openaiError(w, 500, "internal", "auth backend unavailable")
                return
            }
            if k.ExpiresAt != nil && k.ExpiresAt.Before(time.Now()) {
                openaiError(w, 401, "invalid_api_key", "API key has expired")
                return
            }
            resolved := authctx.Resolved{
                APIKey:        k,
                Project:       k.Project,
                Account:       k.CreatedBy,
                Configuration: k.Configuration,
                Modelset:      k.Configuration.Modelset,
                Candidates:    k.Configuration.Modelset.Candidates, // ordered list
            }
            ctx := authctx.WithResolved(r.Context(), resolved)
            // touch last_used_at (fire-and-forget; throttled)
            go touchLastUsed(resolved.APIKey.ID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### KeyLookup

```go
// internal/auth/lookup.go (sketch)

type KeyLookup interface {
    Resolve(ctx context.Context, raw string) (domain.APIKey, error)
}

type CachedLookup struct {
    cache *valkey.Client
    repo  *pgRepo  // postgres fallback
}

func (l *CachedLookup) Resolve(ctx context.Context, raw string) (domain.APIKey, error) {
    // 1. parse out the prefix; cache key is the hash of (prefix + body) so we
    //    never store the raw key.
    if !strings.HasPrefix(raw, "rr_live_") || len(raw) != 41 {
        return domain.APIKey{}, ErrNotFound
    }
    prefix := raw[:len("rr_live_")+8]  // "rr_live_" + 8 chars
    cacheKey := "rr:apikey:" + prefix

    if cached, ok := l.cache.Get(ctx, cacheKey); ok {
        // cached value is {prefix, hash, ...metadata}. Verify raw against hash.
        if verifyHash(raw, cached.Hash) || l.cfg.AuthMode == "stub" {
            return materialize(cached, l.repo), nil  // may need to re-fetch on miss
        }
        return domain.APIKey{}, ErrNotFound
    }

    // 2. cache miss → postgres
    k, err := l.repo.GetByPrefix(ctx, prefix)
    if err != nil { return domain.APIKey{}, err }
    if !verifyHash(raw, k.Hash) && l.cfg.AuthMode != "stub" {
        return domain.APIKey{}, ErrNotFound
    }
    // 3. populate cache (60s TTL)
    l.cache.Set(ctx, cacheKey, k, 60*time.Second)
    return k, nil
}
```

In stub mode the cache hit is unconditional; we skip the
Postgres round trip in the common case. The first call still
warms the cache by hitting Postgres to look up a placeholder
row. **DECIDE**: should stub mode even require a row in
`api_key`? Two options:

- **(A) Always require a row.** Even in stub, an unknown prefix
  is 401. Tests can seed a row. This is more realistic and
  exercises the same code path.
- **(B) Stub mode bypasses the lookup entirely.** Any non-empty,
  well-formed key resolves to a synthetic default project +
  configuration. Simpler tests, but a bigger divergence between
  stub and prod.

We plan to ship **(A)**. Document the trade-off here so we can
revisit.

## Tenancy rules

Once the key is resolved, the rest of the request handler **must**
treat the resolved `Project` as the only project it can see. The
rules:

1. **Every repository method takes a `Project` (or `ProjectID`) as
   its first argument.** No "load by id" repository method exists
   without a project scope. The compiler enforces this for
   well-typed callers; reviewers enforce it for ad-hoc SQL.
2. **No global queries.** Code paths that look like
   `SELECT * FROM configuration WHERE id = $1` are forbidden in
   the request path. Always `WHERE id = $1 AND project_id = $2`.
3. **No foreign project writes.** A `PUT /v1/configurations/{id}`
   from a request scoped to project A cannot write to a row owned
   by project B. Verified by the auth path: the `Resolved`
   struct's `Configuration` is the one bound to the key's
   project. If a client passes a different `model`, the handler
   rejects it unless the configuration allows model override
   (future; MVP locks the model).
4. **Cache keys are tenant-scoped.** Every Valkey key starts with
   `rr:project:<id>:` or `rr:apikey:<prefix>:`. A bug that
   forgets the prefix will cause a key collision across tenants,
   which the integration tests catch.
5. **Postgres RLS is **not** MVP.** We considered row-level
   security as defense in depth, but it adds operational
   complexity (per-connection role switching) and the
   application-level scoping is sufficient for MVP. RLS can be
   added later as a defense-in-depth layer; the SQL schema is
   designed to be RLS-friendly (every business table has
   `project_id`).
6. **The internal admin endpoints run with `RAILROAD_ADMIN_TOKEN`
   or mTLS** and can read/write across projects. They are
   separately audited.

## Authorization model (MVP)

For MVP, the only check is "is the key valid and active?" If yes,
the request is allowed. There is no role-based access control on
chat completions (every key can do everything the model allows).

We do anticipate adding roles later, hence the `role` column on
`account_project`. The hooks to enforce it are:

- Wrap repository calls in a small `WithRole(ctx, role)` that
  checks the role is at least N.
- Add a `RequireProjectRole` middleware on internal endpoints.

## Internal endpoints

A second HTTP server (or a separate path prefix on the same
server, gated by a different auth check) exposes management
endpoints. These are **not** OpenAI-compatible. See the table
above for the full list.

| Method | Path | Purpose | Auth |
|---|---|---|---|
| `POST`   | `/internal/v1/projects` | create a project | admin token |
| `GET`    | `/internal/v1/projects` | list projects | admin token |
| `GET`    | `/internal/v1/projects/{id}` | one project | admin token |
| `POST`   | `/internal/v1/accounts` | create an account | admin token |
| `POST`   | `/internal/v1/accounts/{id}/projects` | add account to project | admin token |
| `POST`   | `/internal/v1/projects/{id}/api-keys` | create a key (returns raw key once) | admin token |
| `GET`    | `/internal/v1/projects/{id}/api-keys` | list keys (no raw values) | admin token |
| `DELETE` | `/internal/v1/api-keys/{id}` | revoke a key | admin token |
| `POST`   | `/internal/v1/projects/{id}/configurations` | create a configuration | admin token |
| `POST`   | `/internal/v1/projects/{id}/modelsets` | create a modelset (with candidates) | admin token |
| `PATCH`  | `/internal/v1/modelsets/{id}` | update a modelset (CEL, candidates) | admin token |
| `GET`    | `/internal/v1/projects/{id}/usage` | usage rollup | admin token |

Admin auth for MVP: a single static `RAILROAD_ADMIN_TOKEN` from the
env, sent as `Authorization: Bearer <token>`. The admin server
binds to a separate port (default `:8081`) so a deployer can
firewall it off from the public internet.

**Future** admin auth: per-account tokens, OIDC, mTLS. The
internal endpoints are shaped to be the only thing that changes.

## Error responses from auth

| case | HTTP | `error.code` |
|---|---|---|
| no `Authorization` header | 401 | `invalid_api_key` |
| `Authorization` not `Bearer ...` | 401 | `invalid_api_key` |
| bearer value not a valid key format | 401 | `invalid_api_key` |
| prefix not in `api_key` | 401 | `invalid_api_key` |
| prefix matches, hash mismatch | 401 | `invalid_api_key` |
| key is revoked | 401 | `invalid_api_key` |
| key is expired | 401 | `invalid_api_key` |
| project is suspended | 403 | `project_disabled` |
| project is deleted | 403 | `project_disabled` |
| Postgres / Valkey unavailable | 503 | `unavailable` |
| unexpected | 500 | `internal` |

We use a single `invalid_api_key` code for all 401 cases so we
don't leak which keys exist. Logging carries the precise reason
for operators.

## Operational notes

- **Key rotation.** A key can be replaced by creating a new one
  and revoking the old. We do not auto-rotate. Future: an
  optional expiry policy and a "rotate" endpoint that issues a
  new key and gives a grace period where both work.
- **Compromised key.** Revoke. The change propagates in seconds
  because the cache TTL is 60s; an operator can flush the cache
  prefix to revoke immediately.
- **Audit.** Every key creation and revocation is recorded in a
  separate `api_key_audit` table (future; not MVP). For MVP, the
  `created_at`, `last_used_at`, and `status` columns are
  sufficient.
- **Per-key rate limit.** The middleware checks a Valkey-backed
  token bucket keyed on the API key id. The default budget is
  generous (1000 RPM) and the global ceiling is in front of the
  per-key bucket. Per-key quotas per project are future.

## Open questions (DECIDE)

- Should stub mode accept any well-formed key, or require a row
  to exist in `api_key`? **Recommendation: require a row.**
  Realistic, exercises the same path, easy to seed in tests.
- Do we want a `RAILROAD_AUTH_MODE=disabled` mode for local
  development where no header is required? **Recommendation: no.**
  Tests should always set the header. Avoids the
  "works on my machine, broken in prod" footgun.
- Should we support `Authorization: <raw>` (no `Bearer` prefix)
  for clients that don't follow the standard? **Recommendation:
  no.** OpenAI requires `Bearer`. Stay compatible.
