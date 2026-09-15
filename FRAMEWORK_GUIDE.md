# Struct Framework — Comprehensive Architecture & Engineering Guide

## 1. Architectural Philosophy & Clean Design

Struct is a typed-first, production-grade Go microservice framework designed around **Clean Architecture**, **Explicit Dependency Injection**, and **Zero Reflection at Runtime**.

Every request, response, configuration value, and domain entity is represented by concrete Go structs. Struct services compile to single static binaries capable of operating standalone or as cooperative nodes in a distributed microservice cluster.

```
+-------------------------------------------------------------------------+
|                              Clean Architecture                         |
+-------------------------------------------------------------------------+
|  Cmd Layer              cmd/api (HTTP server)   cmd/struct (CLI tool)   |
+-------------------------------------------------------------------------+
|  Application Layer      internal/app (Composition Root, Wiring, Routes) |
+-------------------------------------------------------------------------+
|  MVC / Domain Layer     internal/mvc                                    |
|                         models/     services/   controllers/   views/   |
|                         apperr/ (Typed Domain Error Hierarchy)         |
+-------------------------------------------------------------------------+
|  Platform Layer         internal/platform                               |
|                         http/ (router, middleware, appctx)             |
|                         security/ (authn, totp, webauthn, authz)       |
|                         store/ (postgres, mysql, migrations)           |
|                         events/ (publisher, broker, outbox relay)      |
|                         metrics/    tracing/    logging/    procs/      |
+-------------------------------------------------------------------------+
|  Support Layer          internal/support (crypto, validation, queue)   |
+-------------------------------------------------------------------------+
```

### The Strict Layering Rule
A foundational invariant is enforced across the codebase:
> **`internal/platform` never imports `internal/mvc`.**

- **Platform components** (HTTP engine, security primitives, database drivers, metrics, tracing, events) have zero knowledge of business models (`models.User`) or application error wrappers (`apperr.Error`).
- When a platform component needs domain capability (such as `middleware.Auth` needing token verification, or `/readyz` needing a database ping), it defines a **local single-method interface** (`TokenVerifier`, `Pinger`, `LocaleResolver`).
- **`internal/app`** is the Composition Root where platform types and controller types meet.

---

## 2. Request Lifecycle & `AppContext`

Struct provides a unified request lifecycle context via `internal/platform/http/appctx`. As a request traverses the middleware pipeline, contextual metadata is progressively attached to a thread-safe `RequestContext`:

```
Incoming Request
       │
       ▼
 1. Tracing          ──> Parses traceparent or generates new TraceID & SpanID
       │
 2. Recover          ──> Guards against panics, formats JSON 500 error
       │
 3. RequestID        ──> Reuses X-Request-ID or generates unique correlation ID
       │
 4. RealIP           ──> Resolves client IP from trusted reverse proxy headers
       │
 5. SecurityHeaders  ──> Injects HSTS, X-Content-Type-Options, Frame-Options
       │
 6. RateLimit        ──> 16-shard FNV-hash token bucket per client IP
       │
 7. CORS             ──> Deny-by-default cross-origin validation
       │
 8. Locale           ──> Resolves language (?locale= -> Accept-Language -> default)
       │
 9. AccessLog        ──> Structured slog audit log with RequestID and TraceID
       │
10. Metrics          ──> Latency histogram & request counter
       │
11. Timeout          ──> Context deadline propagation (10s default)
       │
12. Auth             ──> Validates Bearer JWT; rejects invalid tokens with 401;
       │                 injects UserID to AppContext when valid
       ▼
  Mux & Controller   ──> Handlers access appctx.FromContext(ctx)
```

### Accessing AppContext in Handlers
Handlers and services retrieve request metadata with zero type assertions:

```go
func (c *UserController) GetProfile(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Direct typed accessors
    reqID := appctx.RequestID(ctx)
    traceID := appctx.TraceID(ctx)
    locale := appctx.Locale(ctx)
    
    // User authentication verification
    userID, ok := appctx.UserID(ctx)
    if !ok {
        writeError(w, http.StatusUnauthorized, "authentication required", nil)
        return
    }
    
    // Structured logging with context automatically includes request_id & trace_id
    c.logger.InfoContext(ctx, "fetching user profile", "user_id", userID)
}
```

---

## 3. Typed Error Architecture (`apperr`)

Services never return raw strings or ad-hoc HTTP status codes. They return typed domain errors defined in `internal/mvc/apperr`.

### Error Kinds & Canonical Status Codes
| Kind | HTTP Status | Code | Typical Scenario |
|---|---|---|---|
| `KindBadRequest` | `400 Bad Request` | `BAD_REQUEST` | Malformed syntax or non-JSON body |
| `KindUnauthorized` | `401 Unauthorized` | `UNAUTHORIZED` | Missing or invalid authentication |
| `KindForbidden` | `403 Forbidden` | `FORBIDDEN` | Access denied by policy / ownership check |
| `KindNotFound` | `404 Not Found` | `NOT_FOUND` | Resource does not exist |
| `KindConflict` | `409 Conflict` | `RESOURCE_CONFLICT` | Unique key violation (e.g. email in use) |
| `KindValidation` | `422 Unprocessable` | `VALIDATION_FAILED` | Field-level business validation error |
| `KindLocked` | `423 Locked` | `ACCOUNT_LOCKED` | Account locked due to repeated auth failures |
| `KindTooManyRequests` | `429 Too Many Req` | `RATE_LIMIT_EXCEEDED` | Rate limit burst exhausted |
| `KindInternal` | `500 Server Error` | `INTERNAL_SERVER_ERROR` | Unhandled database or system fault |
| `KindUnavailable` | `503 Unavailable` | `SERVICE_UNAVAILABLE` | Database or dependency unreachable |
| `KindTimeout` | `504 Gateway Timeout`| `REQUEST_TIMEOUT` | Upstream or downstream deadline expired |

### Unified RFC 7807-Style Error Responses
All error responses generated by `internal/mvc/controllers/render.go` share a deterministic JSON layout:

```json
{
  "error": "The specified user account is locked due to repeated failed logins",
  "code": "ACCOUNT_LOCKED",
  "request_id": "c7f91a08b3e64f20",
  "fields": {
    "password": "too many attempts"
  }
}
```

---

## 4. Multi-Database & Message Broker Architecture

Struct abstracts data storage and event publishing behind declarative contracts, allowing swapping between in-memory mock implementations and live distributed systems without altering domain services.

### Database Dialects
- **`memory`**: In-process atomic map store. Requires zero external containers; ideal for tests and bootstrap evaluation.
- **`postgres`**: Wire protocol v3.0 driver supporting cleartext & MD5 authentication, parameter binding, transactions, and migration runner.
- **`mysql`**: Binary protocol client supporting `mysql_native_password` authentication, binary statement execution, transactions, and migration runner.

### Message Broker Options
Configure via `MESSAGE_BROKER` environment variable:
- **`memory`**: Synchronous in-memory event bus with subscriber callbacks.
- **`log`**: Logs every dispatched event through structured logging without external network dependencies.
- **`redis`**: Streams backend for distributed pub/sub.
- **`nats`**: High-throughput distributed messaging.
- **`kafka`**: Distributed commit log with partition ordering.

### Transactional Outbox Pattern
Domain mutations write events into an `outbox_events` table inside the same atomic database transaction. A decoupled relay daemon (`struct relay`) polls undispatched rows and guarantees at-least-once delivery to downstream consumers.

---

## 5. Security & Authentication Deep Dive

1. **Defensive JWT Token Stack**:
   - Strictly HS256-enforced. Header `alg` is ignored during algorithm verification to prevent `alg: none` confusion attacks.
   - Constant-time verification using `crypto/hmac.Equal`.
2. **Two-Factor Authentication (TOTP)**:
   - RFC 6238 implementation with 30-second time steps, ±1 drift window tolerance, and encrypted secret storage at rest using AES-256-GCM.
   - Single-use 10-character backup codes hashed with SHA-256.
3. **WebAuthn / FIDO2 Passkeys**:
   - Native CBOR and COSE ES256 parser supporting passkeys both as MFA and passwordless primary authentication.
4. **Brute-Force Protection**:
   - Exponential login lockout: 5 failed attempts locks an account for 15 minutes, with failure counts reset upon successful login.
5. **PII Masking**:
   - Structured logging automatically redacts passwords, tokens, API keys, and authorization headers.
