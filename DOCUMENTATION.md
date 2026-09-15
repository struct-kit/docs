# Struct — Technical Documentation

This is the technical reference for the Struct framework: every
configuration value, every HTTP route, every CLI command, and every
package, in one place. 

### Documentation Guides
- **[Framework Architecture Guide](FRAMEWORK_GUIDE.md)**: Deep dive into Clean Architecture, request lifecycle (`appctx`), error handling (`apperr`), message brokers, and security.
- **[API Reference](API_REFERENCE.md)**: Exhaustive reference for all HTTP endpoints, JSON schemas, headers, and status codes.
- **[CLI Reference](CLI_REFERENCE.md)**: Complete guide to the `struct` CLI tool, generators, and database migration commands.
- **[Interactive Web Documentation](index.html)**: Main responsive web portal with full tutorials, copyable snippets, and interactive API explorer ready for GitHub Pages (`https://struct-kit.github.io/docs/`).
- For a narrative walkthrough (how to run it, curl examples, what's built vs. not) see `README.md`. For build-by-build history, risk notes, and roadmap, see `PLAN.md`.

> **Build & Verification Status**: The codebase compiles cleanly on Go 1.27 (`go build ./...`), passes `go vet ./...` with zero warnings, and passes comprehensive unit tests with race detection enabled (`go test -p 1 -race -cover ./tests/...` or `make check`).
> Two areas in particular — the hand-rolled PostgreSQL/MySQL wire protocol clients and the hand-rolled WebAuthn/CBOR/COSE implementation — remain the highest-priority for integration testing against live infrastructure (PostgreSQL 18, MySQL 9, and physical WebAuthn authenticators). See `PLAN.md` and `tasks.md` for the optimization and hardening roadmap.

---

## 1. Architecture

```
cmd/
  api/            entrypoint — wires internal/app, binds a socket
  struct/         CLI entrypoint — wires internal/app, no socket needed
                  for most subcommands
internal/
  app/            composition root: Build(cfg) wires every dependency;
                  Routes(c) is the single source of truth for the route
                  table, shared by cmd/api and `struct routes`
  mvc/            domain layer: models, controllers, views, services,
                  apperr
  platform/       infrastructure: config, http, store, security,
                  i18n, rpc, events, metrics, tracing, procs, logging
  analytics/      non-blocking event publisher
  reports/        aggregate reporting queries
  support/        cross-cutting helpers: crypto, validation, queue
migrations/
  postgres/, mysql/   one pair of .up.sql/.down.sql files per table,
                      per dialect
docker/           Dockerfile + docker-compose.yml
```

**The one layering rule that's enforced everywhere:** `internal/platform`
never imports `internal/mvc`. Platform code (HTTP plumbing, database
drivers, security primitives, i18n, RPC, metrics) knows nothing about
domain types (`models.User`, `apperr.Error`). Where a platform-level
component needs a domain-level capability — `middleware.Auth` needing to
verify a token, `platformhttp.BuildHandler` needing to check database
health — it declares a small local interface (`TokenVerifier`, `Pinger`)
rather than importing the concrete implementation. `internal/mvc` is the
one layer allowed to import both `internal/platform` and `internal/app`'s
neighboring packages; `internal/app` is the only place a platform type
(`platformhttp.Route`) and an mvc type (`controllers.UserController`) are
allowed to meet.

Platform-to-platform imports (e.g. `internal/platform/store/postgres`
importing `internal/platform/events`) are unrestricted — the layering
rule is specifically about the platform→mvc direction.

---

## 2. Configuration

Every value is read once, in `internal/platform/config.Load()`, into a
single `Config` struct — no other package calls `os.Getenv` directly.

| Env var | Default | Notes |
|---|---|---|
| `APP_NAME` | `struct-framework` | Human-readable application name |
| `APP_DESCRIPTION` | `Typed-first secure Go microservice framework` | Application description |
| `APP_VERSION` | `1.0.0` | Semantic version string |
| `APP_ENV` | `production` | `config.Load` refuses to start in production without `JWT_SIGNING_KEY`/`ENCRYPTION_KEY` set |
| `SERVICE_NAME` | `struct-framework` | Identifier for tracing and logging |
| `PORT` | `8080` | |
| `DB_DRIVER` | `memory` | `memory`, `postgres`, or `mysql` — `memory` runs zero external infrastructure with in-memory auth |
| `DATABASE_URL` | *(empty)* | required when `DB_DRIVER` is `postgres` or `mysql` |
| `DATABASE_REPLICA_URL` | *(empty)* | optional read-replica URL for reporting queries |
| `DB_MAX_OPEN_CONNS` | `25` | Maximum open database connections |
| `DB_MAX_IDLE_CONNS` | `5` | Maximum idle database connections |
| `DB_CONN_MAX_LIFETIME` | `5m` | Maximum connection reuse duration |
| `MESSAGE_BROKER` | `memory` | Event broker driver: `memory`, `log`, `redis`, `nats`, `kafka` |
| `BROKER_URL` | *(empty)* | Connection URI for external brokers |
| `OUTBOX_RELAY_ENABLED` | `false` | When true, runs embedded outbox relay in API server background worker |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | *(empty)* | OTLP/HTTP JSON collector endpoint (e.g. `http://localhost:4318/v1/traces`) |
| `JWT_SIGNING_KEY` | *(empty)* | required in production; generated as a process-lifetime ephemeral key outside production if unset |
| `ENCRYPTION_KEY` | *(empty)* | same as above — encrypts TOTP secrets at rest; an ephemeral key means stored secrets become permanently undecryptable on restart |
| `DEFAULT_LOCALE` | `en` | must have a matching embedded catalog file |
| `SUPPORTED_LOCALES` | `en` | comma-separated; each entry must have a matching embedded catalog file or startup fails |
| `WEBAUTHN_RP_ID` | `localhost` | must match the real domain in any non-local deployment |
| `WEBAUTHN_ORIGIN` | `http://localhost:8080` | must match exactly what the browser reports |
| `READ_TIMEOUT` | `5s` | |
| `WRITE_TIMEOUT` | `10s` | |
| `SHUTDOWN_TIMEOUT` | `15s` | |
| `RATE_LIMIT_RPS` | `20` | per-IP; burst allowance is `2×` this value |
| `LOG_LEVEL` | `info` | Logging level (`debug`, `info`, `warn`, `error`) |
| `PPROF_ENABLED` | `false` | When true, exposes `/debug/pprof/*` endpoints |

---

## 3. HTTP API Reference

### Always available

| Method | Path | Notes |
|---|---|---|
| GET | `/healthz` | liveness endpoint returning JSON uptime, version, and app name |
| GET | `/readyz` | readiness endpoint checking DB and broker pings (503 if unhealthy) |
| GET | `/metrics` | Prometheus text exposition format — see §7 |
| GET | `/debug/pprof/*` | Go runtime profiling endpoints (only when `PPROF_ENABLED=true`) |
| POST | `/v1/users` | create a user |
| GET | `/v1/users/{id}` | fetch a user |

### Authentication (`/v1/auth/*`)

The auth stack provides durable persistence across PostgreSQL and MySQL, and includes a zero-dependency in-memory implementation (`MemoryAuthService`) for local development and CI testing. Routes marked ✱ require a valid `Authorization: Bearer <access_token>` header.

| Method | Path | What it does |
|---|---|---|
| POST | `/v1/auth/login` | password login → tokens, or an MFA ticket if 2FA is enabled |
| POST | `/v1/auth/refresh` | rotates a refresh token for a new pair |
| POST | `/v1/auth/logout` | revokes the session a refresh token belongs to |
| POST | `/v1/auth/logout-all` ✱ | revokes every active session for the caller |
| POST | `/v1/auth/mfa/totp` | redeems a login MFA ticket with a TOTP code or backup code |
| POST | `/v1/auth/totp/enroll` ✱ | generates a new (unconfirmed) TOTP secret + `otpauth://` URI |
| POST | `/v1/auth/totp/confirm` ✱ | confirms enrollment; returns one-time backup codes |
| DELETE | `/v1/auth/totp` ✱ | disables TOTP, deletes remaining backup codes |
| GET | `/v1/auth/totp/backup-codes` ✱ | returns count of remaining backup recovery codes |
| POST | `/v1/auth/passkeys/register/begin` ✱ | starts WebAuthn passkey registration |
| POST | `/v1/auth/passkeys/register/finish` ✱ | completes registration, persists the passkey |
| GET | `/v1/auth/passkeys` ✱ | lists the caller's registered passkeys |
| PATCH | `/v1/auth/passkeys/{id}` ✱ | renames a passkey (ownership-checked) |
| DELETE | `/v1/auth/passkeys/{id}` ✱ | removes a passkey (ownership-checked) |
| POST | `/v1/auth/mfa/passkey/begin` | starts passkey-as-second-factor for a login MFA ticket |
| POST | `/v1/auth/mfa/passkey/finish` | completes it → tokens |
| POST | `/v1/auth/passkeys/login/begin` | starts passwordless (discoverable or email-first) passkey login |
| POST | `/v1/auth/passkeys/login/finish` | completes it → tokens |

### Middleware chain (outer to inner)

```
Tracing → Recover → RequestID → RealIP → SecurityHeaders →
RateLimit → CORS → Locale → AccessLog → Metrics → Timeout → Auth
```

`Tracing` is outermost so its span covers every other middleware's own
overhead. `Auth` runs last: it validates Bearer tokens, rejecting invalid or
expired tokens with 401 Unauthorized, and allows unauthenticated requests
to pass through (attaching user ID to request context when a valid Bearer token
is present; enforcement for authenticated-only endpoints is verified via
`middleware.UserIDFrom`).
A consequence worth knowing: a rate-limited `429` or a panic `Recover`
catches happens *before* `Locale` runs, so neither response is ever
localized.

---

## 4. CLI Reference (`struct`)

| Command | What it does |
|---|---|
| `struct serve` | runs the HTTP API — equivalent to `cmd/api` |
| `struct routes` | prints the route table, no socket bound |
| `struct health` | hits this instance's own `/readyz` |
| `struct version` | prints version/commit/Go runtime version |
| `struct about` | prints framework information |
| `struct doctor` | environment/config/database/i18n-catalog checks |
| `struct make model\|controller\|service\|dto\|enum\|policy\|repository\|event\|job NAME` | scaffolds under `internal/mvc`/`internal/platform`/`internal/support` |
| `struct make locale LANG` | scaffolds a new locale catalog file, values empty pending translation |
| `struct make rpc NAME` | scaffolds an internal RPC contract + client + server handler stub |
| `struct migrate create NAME` | generates a timestamped `.up.sql`/`.down.sql` pair |
| `struct migrate up` / `down` | applies/rolls back against `DATABASE_URL` |
| `struct new NAME [--template=minimal\|standard] [--module=PATH]` | scaffolds a new project from the embedded starter template |
| `struct lint` | `gofmt`/`go vet`-style checks |
| `struct completion bash\|zsh\|fish` | shell completion scripts |
| `struct relay` | runs the outbox relay standalone, polling `outbox_events` |
| `struct report signups` | prints daily signup counts for the last 30 days (uses `DATABASE_REPLICA_URL` if set) |

---

## 5. Package Reference

### `internal/platform/config`
Single `Config` struct, loaded once via `Load()`. See §2.

### `internal/platform/http` + `.../middleware`
`Route`, `BuildMux`, `BuildHandler` (the composition root for the
middleware chain — see §3). `middleware` exposes `Chain`, `Recover`,
`RequestID`, `RealIP`, `SecurityHeaders`, `IPRateLimiter` (sharded across
16 lock/map shards — see `PLAN.md`'s Pass 8 entry), `CORS`, `Locale`,
`AccessLog`, `Metrics`, `Timeout`, `Auth`, and `Tracing`. Cross-cutting
dependencies are always declared as small local interfaces
(`TokenVerifier`, `Pinger`, `LocaleResolver`, `RequestMetrics`) rather
than importing the concrete implementation.

### `internal/platform/store` + `.../postgres` + `.../mysql`
`Driver`/`Tx`/`Result`/`Row`/`Rows` — the dialect-agnostic interfaces
every repository is written against. `postgres` and `mysql` each
implement a from-scratch wire-protocol client (framing, auth, query
execution, connection pooling, a migration runner) plus every
repository: `UserRepository`, `RefreshTokenRepository`,
`TOTPRepository`, `BackupCodeRepository`, `WebAuthnCredentialRepository`,
`OutboxPublisher`. **This is the highest-risk code in the project** — see
`PLAN.md`'s Pass 3a/3b entries.

### `internal/platform/security/{authn,totp,webauthn,authz}`
`authn`: HS256-only JWT (never dispatches on an attacker-controlled
`alg`), opaque refresh tokens. `totp`: RFC 6238 codes, `otpauth://` URIs,
backup codes. `webauthn`: a from-scratch CBOR decoder, COSE_Key parsing
(ES256/P-256 only), and registration/assertion ceremony verification,
scoped to attestation `"none"`. **The single highest-risk package in the
codebase** — never run against a real authenticator. `authz`:
deny-by-default `Policy` interface.

### `internal/platform/i18n`
Embedded `Catalog` (`//go:embed locales/*.json`, necessarily nested
under this package — embed can't reference a parent directory).
`T(locale, key, args...)` with fallback resolution; `IsSupported`,
`SupportedLocales`, `MissingKeys` for completeness checking. Only two
messages are actually routed through it in this codebase — see
`internal/mvc/controllers/i18n.go`'s doc comment.

### `internal/platform/rpc`
Resilient JSON-over-HTTP client (`Client.Call`): deadline propagation
with a safety-margin shrink, jittered-backoff retries (idempotent calls
only), a circuit breaker, a bulkhead, round-robin load balancing over a
`Resolver`, and W3C trace-context propagation on outbound calls.
`Handle[Req, Resp]` is the generic server-side counterpart. Stands in for
gRPC — see the package doc comment for why.

### `internal/platform/events`
`Event`/`Publisher` (the domain-event interface), `OutboxSource`/`Sink`/
`Relay` (the real transactional-outbox machinery `postgres.
OutboxPublisher`/`mysql.OutboxPublisher` implement).

### `internal/platform/metrics`
`Counter`/`CounterVec`/`Gauge`/`Histogram` and a `Registry` rendering
Prometheus's text exposition format (implements standard `io.WriterTo` via
`WriteTo(io.Writer) (int64, error)`). `HTTPMetrics` bundles the standard
`http_requests_total`/`http_request_duration_seconds` pair.

### `internal/platform/tracing`
W3C Trace Context (`traceparent` header) parsing/formatting, `Span`/
`StartSpan`, `LogExporter` (stands in for a real OTLP exporter).

### `internal/platform/procs`
Cgroup-aware `GOMAXPROCS` — reads a CPU quota from cgroup v1 or v2,
falling back to `runtime.NumCPU()` if none is imposed. Called once at
startup from `internal/app.Build`.

### `internal/analytics`
Non-blocking event `Publisher` — `Track` never blocks the caller; a full
buffer drops the event and increments a `DroppedEventsCounter`.

### `internal/reports`
`SignupReport` — a real, dialect-aware aggregate query.

### `internal/mvc/{models,controllers,views,apperr,services}`
The `User` vertical slice and the full auth stack's HTTP-facing layer.
`services.InMemoryUserService`/`PostgresUserService`/`MySQLUserService`
all satisfy one `UserService` interface; `PostgresAuthService`/
`MySQLAuthService` likewise satisfy one `AuthService` interface. This is
where each dialect's row type maps to `models.User` and each dialect's
error type maps to `apperr` — that translation lives here specifically
because `internal/platform/store/{postgres,mysql}` may not import
`internal/mvc`.

### `internal/support/{crypto,validation,queue}`
Password hashing (PBKDF2-HMAC-SHA256), AES-256-GCM field encryption,
request validation, and a minimal job queue interface.

---

## 6. Known Deviations (consolidated)

Every deviation below exists because the build environment had no
network access — no Go module proxy, no package manager. Each is
documented in full at its source package; this is the index.

| Framework guide spec | What's here instead | Where |
|---|---|---|
| Argon2id | PBKDF2-HMAC-SHA256 (RFC 8018) | `internal/support/crypto` |
| go-playground/validator | hand-rolled field-error accumulator | `internal/support/validation` |
| caarlos0/env | manual env parsing | `internal/platform/config` |
| x/time/rate + Redis | in-memory sharded token bucket | `internal/platform/http/middleware/rate_limit.go` |
| spf13/cobra | hand-rolled command tree | `internal/app/cli/command.go` |
| jackc/pgx | hand-rolled PostgreSQL wire protocol v3.0 client | `internal/platform/store/postgres/` |
| go-sql-driver/mysql | hand-rolled MySQL binary protocol client | `internal/platform/store/mysql/` |
| A JWT library | hand-rolled HS256-only JWT | `internal/platform/security/authn` |
| A WebAuthn/CBOR library | hand-rolled CBOR/COSE + ceremony verification | `internal/platform/security/webauthn` |
| gRPC + protobuf | JSON-over-HTTP with the same resilience patterns | `internal/platform/rpc` |
| Kafka/Redis Streams | `Sink`/`Publisher` interfaces, `LogSink` only | `internal/platform/events`, `internal/analytics` |
| Prometheus client library | hand-rolled (low-risk: plain text format) | `internal/platform/metrics` |
| OpenTelemetry SDK/OTLP | hand-rolled W3C trace context + `LogExporter` (low-risk: plain text format) | `internal/platform/tracing` |
| go.uber.org/automaxprocs | hand-rolled cgroup v1/v2 quota reader | `internal/platform/procs` |

---

## 7. Verification Status

The codebase compiles cleanly with `go build ./...` and `go vet ./...` under Go 1.27, and passes the comprehensive test suite with race detection (`go test -p 1 -race -cover ./tests/...` or `make check`).

In descending order of remaining integration verification:

1. **`internal/platform/security/webauthn`** and **`internal/platform/store/{postgres,mysql}`** — need live integration testing against physical authenticators and live PostgreSQL 18 / MySQL 9 database instances (see `docker/docker-compose.yml`).
2. **The auth stack built on top of those** (`services.*AuthService`, `AuthController`) — unit-tested with mocks/in-memory; needs end-to-end session and token lifecycle verification against live stores.
3. **`internal/platform/rpc`, `events`, `metrics`, `tracing`, `procs`** — covered by unit tests under `./tests/...` (`TestRegistry`, `TestCircuitBreaker`, etc.), verified with `-race`.
4. **Everything else** — standard library composition, verified by unit tests, `go vet`, and `gofmt`.

See `PLAN.md` and `tasks.md` for pass-by-pass test sequences and the master hardening roadmap.
