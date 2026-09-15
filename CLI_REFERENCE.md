# Struct CLI (`struct`) — Operational Reference Manual

The `struct` CLI tool provides code generators, database migration runners, operational diagnostics, outbox relays, and project scaffolding.

```
struct [command] [flags]
```

---

## 1. Core Server & Operational Commands

### `struct serve`
Runs the HTTP API server. Equivalent to executing `cmd/api`.

```bash
struct serve
# Flags inherited from environment:
# PORT, APP_ENV, DB_DRIVER, DATABASE_URL, LOG_LEVEL, etc.
```

### `struct routes`
Prints the compiled HTTP route table without binding a socket. Useful for inspecting route paths, HTTP methods, and middleware registration during CI checks.

```bash
struct routes
```
**Example Output**:
```text
GET    /healthz
GET    /readyz
GET    /metrics
POST   /v1/users
GET    /v1/users/{id}
GET    /debug/pprof/
GET    /debug/pprof/profile
```

### `struct health`
Performs a local health probe against this service's own `/readyz` endpoint. Exits with code 0 on healthy status, 1 if unavailable.

```bash
struct health
```

### `struct version`
Displays the binary version, git commit hash, and Go runtime compiler version.

```bash
struct version
# Output: struct 1.0.0 (a1b2c3d) go1.27.1
```

### `struct doctor`
Runs automated diagnostic pre-flight checks:
- Verifies required environment configuration.
- Tests database connectivity if `DB_DRIVER` is configured.
- Validates i18n locale catalogs for missing message keys across all supported locales.

```bash
struct doctor
```

---

## 2. Code Generation (`struct make`)

All code generators detect the active module path from `go.mod` in the current working directory. Generated code is strictly formatted via `gofmt` and refuses to overwrite existing files without explicit user intervention.

### `struct make model <Name>`
Generates a domain model struct under `internal/mvc/models/`.

```bash
struct make model Invoice
# Creates: internal/mvc/models/invoice.go
```

### `struct make controller <Name>`
Generates an HTTP controller with dependency interfaces and route handlers under `internal/mvc/controllers/`.

```bash
struct make controller Invoice
# Creates: internal/mvc/controllers/invoice_controller.go
```

### `struct make service <Name>`
Generates a domain service interface and default implementation under `internal/mvc/services/`.

```bash
struct make service Invoice
# Creates: internal/mvc/services/invoice_service.go
```

### `struct make dto <Name>`
Generates request and response Data Transfer Objects under `internal/mvc/views/`.

```bash
struct make dto Invoice
# Creates: internal/mvc/views/invoice_dto.go
```

### `struct make enum <Name> --values=v1,v2,v3`
Generates a type-safe string enum with validation methods under `internal/mvc/models/`.

```bash
struct make enum PaymentStatus --values=pending,completed,failed,refunded
# Creates: internal/mvc/models/payment_status.go
```

### `struct make policy <Name>`
Generates a deny-by-default authorization policy under `internal/platform/security/authz/`.

```bash
struct make policy Invoice
# Creates: internal/platform/security/authz/invoice_policy.go
```

### `struct make event <Name>`
Generates a versioned domain event struct under `internal/platform/events/`.

```bash
struct make event InvoicePaidV1
# Creates: internal/platform/events/invoice_paid_v1.go
```

### `struct make job <Name>`
Generates a background job definition compatible with retry queues under `internal/support/queue/`.

```bash
struct make job SendInvoiceEmail
# Creates: internal/support/queue/send_invoice_email_job.go
```

### `struct make locale <Lang>`
Scaffolds a new translation catalog file under `internal/platform/i18n/locales/`.

```bash
struct make locale de
# Creates: internal/platform/i18n/locales/de.json
```

### `struct make rpc <Name>`
Generates a resilient internal RPC contract, client wrapper, and server handler stubs under `internal/platform/rpc/`.

```bash
struct make rpc BillingService
# Creates: internal/platform/rpc/billing_service.go
```

---

## 3. Database Migrations (`struct migrate`)

Migrations use dialect-specific SQL files located in `migrations/postgres/` and `migrations/mysql/`.

### `struct migrate create <name>`
Creates a timestamped `.up.sql` and `.down.sql` file pair for both PostgreSQL and MySQL dialects.

```bash
struct migrate create create_invoices_table
# Creates:
# migrations/postgres/20260915090000_create_invoices_table.up.sql
# migrations/postgres/20260915090000_create_invoices_table.down.sql
# migrations/mysql/20260915090000_create_invoices_table.up.sql
# migrations/mysql/20260915090000_create_invoices_table.down.sql
```

### `struct migrate up`
Applies all pending migrations against the database specified in `DATABASE_URL`.

```bash
struct migrate up
```

### `struct migrate down`
Rolls back the most recent migration batch.

```bash
struct migrate down
```

---

## 4. Operational Daemons & Reports

### `struct relay`
Runs the Transactional Outbox Relay daemon as a standalone worker process. Polls `outbox_events` and dispatches pending events to the configured message broker sink (`Kafka`, `NATS`, `Redis`, or `Log`).

```bash
struct relay
```

### `struct report signups`
Executes an aggregate dialect-aware database query to report user signups grouped by day over the last 30 days.

```bash
struct report signups
```

---

## 5. Scaffolding & Tooling

### `struct new <name> [--template=minimal|standard] [--module=PATH]`
Scaffolds a complete new microservice project from embedded starter templates.

```bash
# Minimal starter:
struct new payment-service --template=minimal

# Standard production-ready starter:
struct new order-service --template=standard --module=github.com/company/order-service
```

### `struct lint`
Executes internal style, formatting (`gofmt`), and correctness (`go vet`) verification checks.

```bash
struct lint
```

### `struct completion <bash|zsh|fish>`
Generates shell autocompletion scripts for CLI subcommands and flags.

```bash
# Bash:
source <(struct completion bash)

# Zsh:
source <(struct completion zsh)
```
