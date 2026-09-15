# Struct — Production-Ready Go Microservice Framework

[![Go Version](https://img.shields.io/badge/Go-1.27+-00ADD8?style=for-the-badge&logo=go)](https://go.dev/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](https://github.com/struct-kit/framework/blob/main/LICENSE)
[![Go Report Card](https://goreportcard.com/badge/github.com/struct-kit/framework?style=for-the-badge)](https://goreportcard.com/report/github.com/struct-kit/framework)
[![Documentation](https://img.shields.io/badge/Docs-struct--kit.github.io-2ea44f?style=for-the-badge&logo=read-the-docs)](https://struct-kit.github.io)
[![GitHub](https://img.shields.io/badge/GitHub-struct--kit%2Fframework-181717?style=for-the-badge&logo=github)](https://github.com/struct-kit/framework)

> **Struct** is a typed-first, production-ready Go framework for building independently deployable microservices. Every request, response, config value, and domain object is a concrete struct — no reflection-heavy magic, no hidden global state. Each service ships as a single static binary with its own database, migrations, and `struct` CLI — designed to run standalone or as one node in a larger service mesh.

---

## ✨ Design Principles

| Principle | Description |
|---|---|
| **Explicit over implicit** | No hidden global state, no magic DI containers |
| **Compile-time safety** | Generics for repositories/services; no `interface{}` where a concrete type will do |
| **Database-agnostic by contract** | Write once against `store.Driver`; PostgreSQL and MySQL interchangeable at deploy time |
| **Secure and private by default** | TLS, CSRF, rate limiting, strict headers, PII redaction — wired in from day one |
| **Observable by default** | Structured logs, metrics, traces, product analytics ship out of the box |
| **Global by default** | Every user-facing string flows through the i18n layer; zero hardcoded English |
| **Container-native** | Single static binary, distroless runtime, health/readiness probes, no local state |
| **Tooling included** | The `struct` CLI handles scaffolding, migrations, codegen, and operations |

---

## 🏗️ Architecture

### Project Structure

```text
service-name/
├── cmd/
│   ├── api/                        # HTTP server entrypoint (composition root)
│   │   └── main.go
│   └── struct/                     # CLI entrypoint (struct new, migrate, serve, ...)
│       └── main.go
├── internal/
│   ├── app/                        # dependency wiring, bootstrap, DI container
│   ├── platform/                   # infrastructure concerns
│   │   ├── config/ http/ logging/ security/ i18n/ store/
│   ├── mvc/                        # application/domain layer
│   │   ├── controllers/ models/ views/
│   ├── analytics/                  # typed event publisher (non-blocking)
│   ├── reports/                    # scheduled/on-demand aggregate reports
│   └── support/                    # cross-cutting helpers
│       ├── cache/ queue/ mailer/ scheduler/ crypto/ validation/
├── locales/                        # embedded translation catalogs
├── migrations/
│   ├── postgres/                   # golang-migrate-format SQL, Postgres dialect
│   └── mysql/                      # golang-migrate-format SQL, MySQL dialect
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── go.mod
└── go.sum
```

### Architecture Rules

- `cmd/api/main.go` only wires dependencies via `internal/app` — **no business logic, no route handlers.
- `internal/platform` is infrastructure only — it **never imports `internal/mvc`.
- `internal/mvc` is the only layer that knows HTTP shapes; controllers depend on service interfaces, never concrete stores or SQL dialects.
- A service picks exactly one active database backend at runtime (`DB_DRIVER=postgres|mysql`), but the codebase compiles and tests against **both**.

---

## 🚀 Quick Start

### 1. Install the CLI

```bash
# From source (Go):
go install github.com/struct-kit/framework/cmd/struct@latest

# Or download a prebuilt binary from releases
# Releases tab of struct-kit/framework
```

### 2. Scaffold a New Service

```bash
# Standard template + PostgreSQL + English/Arabic/Spanish locales
struct new my-service \
  --template=standard \
  --db=postgres \
  --locales=en,ar,es

cd my-service
```

Three template sizes are available:

- **`minimal`** — Single-file middleware chain, no MVC slice. For prototypes and CLI-only tools.
- **`standard`** (default) — Full layout, one `User` vertical slice, auth stack wired, both DB dialects scaffolded.
- **`full`** — Standard plus working examples of every `make` target (events, jobs, gRPC, policies, etc.).

### 3. Configure

Copy `.env.example` to `.env` and set your secrets:

```bash
cp .env.example .env
# Edit: JWT_SIGNING_KEY, ENCRYPTION_KEY, DATABASE_URL, REDIS_URL
```

### 4. Run Migrations & Start Developing

```bash
# Run database migrations
struct migrate up

# Start the dev server with hot-reload
struct serve --watch
```

### 5. Verify Everything Works

```bash
# Check environment and project health
struct doctor

# Run the test suite
go test -race -cover ./...

# List registered routes
struct routes
```

---

## ⚙️ Configuration

All configuration is loaded once at startup into a strict struct — no scattered `os.Getenv` calls.

| Variable | Default | Description |
|---|---|---|
| `APP_ENV` | `production` | Runtime environment |
| `SERVICE_NAME` | *(required)* | Service identifier |
| `PORT` | `8080` | HTTP port |
| `DB_DRIVER` | *(required)* | `postgres` or `mysql` |
| `DATABASE_URL` | *(required)* | Database connection string |
| `REDIS_URL` | *(required)* | Redis connection string |
| `JWT_SIGNING_KEY` | *(required in prod)* | JWT signing secret |
| `ENCRYPTION_KEY` | *(required in prod)* | AES-256-GCM field encryption key |
| `DEFAULT_LOCALE` | `en` | Fallback locale |
| `SUPPORTED_LOCALES` | `en` | Comma-separated list of locales |
| `READ_TIMEOUT` | `5s` | HTTP read timeout |
| `WRITE_TIMEOUT` | `10s` | HTTP write timeout |
| `SHUTDOWN_TIMEOUT` | `15s` | Graceful shutdown deadline |
| `RATE_LIMIT_RPS` | `20` | Default requests/second per IP |
| `LOG_LEVEL` | `info` | `debug` \| `info` \| `warn` \| `error` |

Secrets must be injected via your orchestrator's secret store (Docker Secrets, Vault, cloud secrets manager) — **never** committed to `.env` files in version control.

---

## 🧩 Core Components

### MVC Architecture

#### Models

Plain structs with validation tags, no framework or SQL-dialect coupling.

```go
// internal/mvc/models/user.go
type User struct {
    ID           string    `json:"id"`
    Email        string    `json:"email" validate:"required,email"`
    PasswordHash string    `json:"-"`
    Locale       string    `json:"locale" validate:"required,bcp47"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}
```

#### Controllers

Depend only on service interfaces, never concrete stores. Handlers return typed errors that middleware translates into localized HTTP responses.

```go
func (c *UserController) Create(w http.ResponseWriter, r *http.Request) {
    var req struct { Email, Password string }
    if err := json.NewDecoder(io.LimitReader(r.Body, 1<<20)).Decode(&req); err != nil {
        writeError(w, r, http.StatusBadRequest, i18n.KeyInvalidBody)
        return
    }
    user, err := c.svc.CreateUser(r.Context(), req.Email, req.Password, i18n.FromRequest(r))
    if err != nil { writeServiceError(w, r, err); return }
    writeJSON(w, http.StatusCreated, views.FromUser(user))
}
```

#### Views

Typed response DTOs — never raw model structs — so internal fields can never leak.

```go
type UserResponse struct {
    ID        string `json:"id"`
    Email     string `json:"email"`
    CreatedAt string `json:"created_at"`
}

func FromUser(u models.User) UserResponse { /* ... */ }
```

### Data Store Layer (PostgreSQL & MySQL)

`internal/platform/store` exposes one driver-agnostic `store.Driver` interface. Business logic in `internal/mvc/services` never imports either dialect package directly.

```go
// internal/platform/store/driver.go
type Driver interface {
    Exec(ctx context.Context, query string, args ...any) (Result, error)
    Query(ctx context.Context, query string, args ...any) (Rows, error)
    QueryRow(ctx context.Context, query string, args ...any) Row
    BeginTx(ctx context.Context) (Tx, error)
    Ping(ctx context.Context) error
    Close() error
}
```

Repositories are written once against `store.Driver` using portable SQL. Parameterized placeholders (`$1` for Postgres, `?` for MySQL) are handled by a `QueryBuilder` helper.

**Migrations** are maintained for both dialects in `migrations/postgres/` and `migrations/mysql/` with golang-migrate-style dirty-flag discipline.

### HTTP Layer

Built on Go 1.22+ `net/http.ServeMux` with hard timeouts.

#### Middleware Chain (order matters):

```
Recover → RequestID → RealIP → SecurityHeaders → RateLimit → CORS → Locale → AccessLog → Timeout → Auth
```

#### Service-to-Service Communication

- **Synchronous**: gRPC over mTLS with protobuf contracts, resilience middleware (deadline propagation, bounded retries, circuit breaker, bulkhead, client-side LB)
- **Asynchronous**: Transactional outbox pattern publishing typed domain events (Kafka or Redis Streams), idempotent consumers, schema versioning, dead-letter handling

### Security & Privacy

| Feature | Implementation |
|---|---|
| **Transport** | TLS 1.3 + HSTS, mTLS mandatory between internal services (SPIFFE/SPIRE-compatible) |
| **Headers** | HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy |
| **AuthN** | Argon2id password hashing, short-lived JWT + rotating refresh tokens, TOTP 2FA, WebAuthn/FIDO2 passkeys |
| **AuthZ** | Typed RBAC policies, deny-by-default, ownership-scoped at the repository layer |
| **CSRF** | Stateless double-submit tokens, `DisallowUnknownFields`, body size caps |
| **Rate Limiting** | Token-bucket per IP + per user, tiered limits by route sensitivity, exponential-backoff lockout |
| **Privacy** | PII redaction by allowlist (`LogSafe()`), `PurgeUser()` cascading deletion, AES-256-GCM column encryption |
| **Supply Chain** | SBOM generation, `cosign` image signing, SLSA build provenance |
| **Auditing** | Append-only audit log for privileged actions, authentication anomaly detection |

### Internationalization (i18n)

- Locale resolution: `?locale=` param → user preference → `Accept-Language` header → `DEFAULT_LOCALE`
- Embedded JSON catalogs (`//go:embed locales/*.json`)
- Message keys (`i18n.KeyInvalidBody`), never raw strings
- CLDR-compliant pluralization, number/date/currency formatting
- First-class RTL support in server-rendered HTML

---

## 🛠️ CLI (`struct`)

The `struct` CLI reuses the exact same services and config loading as the API — zero drift between CLI and server. Every scaffold command is deterministic, idempotent, and leaves the tree in a verified buildable state (`gofmt`, `go vet`, `go build` run automatically).

### Command Reference

| Command | Description |
|---|---|
| `struct new <name> [flags]` | Scaffold a new microservice |
| `struct workspace new <name> --services=a,b,c` | Scaffold a multi-service `go.work` monorepo |
| `struct doctor` | Environment preflight |
| `struct serve [--watch]` | Run the HTTP API (with hot-reload) |
| `struct migrate up / down` | Apply/roll back migrations for active `DB_DRIVER` |
| `struct migrate create NAME` | Generate migration pair under **both** dialect directories |
| `struct make model NAME` | Scaffold model struct with validation tags |
| `struct make controller NAME` | Scaffold controller + test file |
| `struct make service NAME` | Scaffold service interface + in-memory impl + test doubles |
| `struct make repository NAME` | Scaffold repository with Postgres and MySQL implementations |
| `struct make dto NAME` | Scaffold request/response DTO pair |
| `struct make enum NAME --values=A,B,C` | Scaffold typed Go 1.27 enum with String(), JSON (un)marshal, validation |
| `struct make policy NAME` | Scaffold deny-by-default authz policy struct + test |
| `struct make event NAME` | Scaffold versioned domain event, outbox Publish, consumer stub |
| `struct make job NAME` | Scaffold background worker with retry/backoff |
| `struct make rpc NAME` | Scaffold `.proto`, codegen, gRPC server + resilience client |
| `struct make middleware NAME` | Scaffold HTTP middleware |
| `struct make locale LANG` | Scaffold locale catalog stub, flag missing keys |
| `struct openapi generate` | Derive OpenAPI 3.1 spec from routes + DTOs |
| `struct routes` | Print registered route table |
| `struct lint` | `gofmt -l`, `go vet`, `golangci-lint` in one command |
| `struct bench` | Bench hot paths, diff against baseline, fail on regression |
| `struct health` | Hit `/readyz` locally (also Docker HEALTHCHECK) |
| `struct version` | Print build version, commit SHA, Go version |
| `struct about` | Print framework info |
| `struct db dump / restore` | Dump/restore schema + data (dialect-aware) |
| `struct db switch --to=mysql\|postgres` | Print migration/config migration steps between backends |
| `struct completion <bash\|zsh\|fish>` | Generate shell completion script |

---

## 📊 Observability

| Signal | Implementation |
|---|---|
| **Logging** | `log/slog` with JSON output in prod, text in dev. Request-scoped fields propagated via context. PII-redacted access logs. |
| **Metrics** | Prometheus `/metrics` on internal port: request counts, latency histograms, DB pool saturation, queue depth, analytics-drop counts |
| **Tracing** | OpenTelemetry SDK + OTLP exporter. Context propagated through middleware, DB/cache calls, and across service boundaries. |
| **Health** | `/healthz` (liveness) and `/readyz` (readiness — DB/cache reachable) mirrored by `struct healthcheck` in the CLI. |

---

## 🐳 Docker & Deployment

### Multi-Stage Dockerfile

Distroless, non-root, fully static (`CGO_ENABLED=0`), locales + migrations for both dialects baked in.

```dockerfile
# Stage 1: Build (golang:1.27-alpine)
#   - go build ./cmd/api → /out/service-api
#   - go build ./cmd/struct → /out/struct

# Stage 2: Runtime (gcr.io/distroless/static-debian12:nonroot)
#   - COPY binaries, migrations, locales, CA certs
#   - USER nonroot:nonroot
#   - ENTRYPOINT ["/app/service-api"]
```

### docker-compose.yml

Wires **both** database services; switch backends with one `DB_DRIVER` env var. Includes Postgres, MySQL, Redis, healthchecks, resource limits, read-only rootfs, `no-new-privileges`, and dropped capabilities.

```bash
docker compose -f docker/docker-compose.yml up --build -d
docker compose -f docker/docker-compose.yml exec api /app/struct migrate up
```

### CI/CD Checklist

1. `go vet ./...` + `staticcheck ./...`
2. `go test -race -cover ./...` against **both** PostgreSQL and MySQL
3. `govulncheck ./...`
4. i18n catalog-completeness check
5. Build + scan multi-stage Docker image (Trivy)
6. Push with immutable digest tag
7. Rolling update with readiness-gated traffic cutover
8. `struct migrate up` pre-deploy
9. Post-deploy smoke test `/readyz`

---

## 🧪 Testing Strategy

| Test Type | Scope |
|---|---|
| **Unit** | Controllers/services against mocked `store.Driver` |
| **Integration** | Run against real local PostgreSQL **and** real local MySQL — every repository, migration, and error mapping exercised against both |
| **i18n** | Catalog-completeness check fails the build if any supported locale is missing a key |
| **Security** | Fuzz input validation (`go test -fuzz`), `govulncheck`, container image scan (Trivy/Grype) |
| **Load** | `k6`/`vegeta` against staging to validate per-endpoint latency and throughput budget |
| **Bench** | `go test -bench` hot paths, `struct bench` fails build on configured regression threshold |
| **Contract** | Consumer-driven contract tests (Pact-style) in CI for every service-to-service pair |

---

## 📦 Support Packages

| Package | Responsibility |
|---|---|
| `cache` | Redis-backed cache with typed generics: `Get[T]`, `Set[T]`, read-through with single-flight dedup, negative caching |
| `queue` | Background job processing (Redis Streams / SQS adapter interface) |
| `mailer` | Transactional, localized email via SMTP/SES with templated, tested payloads |
| `scheduler` | Cron-style job scheduling with distributed locking |
| `crypto` | Argon2id password hashing, AES-GCM field encryption, token generation |
| `validation` | Struct-tag validation wrapper, i18n-integrated error formatting |

Each exposes a small interface so it can be mocked in unit tests.

---

## 📚 Learn More

- **Website & Documentation**: [https://struct-kit.github.io](https://struct-kit.github.io)
- **GitHub Repository**: [https://github.com/struct-kit/framework](https://github.com/struct-kit/framework)
- **Issues & Bug Reports**: [GitHub Issues](https://github.com/struct-kit/framework/issues)
- **Discussions**: [GitHub Discussions](https://github.com/struct-kit/framework/discussions)
- **Framework Specification**: See [STRUCT_FRAMEWORK.md](file:///Users/loaikanou/Documents/Projects/struct-kit/STRUCT_FRAMEWORK.md)

---

## 🤝 Contributing

Contributions are welcome! Please read through the framework specification in [STRUCT_FRAMEWORK.md](file:///Users/loaikanou/Documents/Projects/struct-kit/STRUCT_FRAMEWORK.md) before opening a PR.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'feat: add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

Make sure `struct lint`, `struct doctor`, and the full test suite pass locally before submitting.

---

## 📄 License

Struct is released under the **MIT License**.
