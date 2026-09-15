# Struct Framework — Complete HTTP API Reference

All requests and responses use `Content-Type: application/json; charset=utf-8` unless otherwise specified.
Routes marked with ✱ require an `Authorization: Bearer <access_token>` header.

---

## 1. System & Diagnostic Endpoints

### `GET /healthz`
Process liveness check. Always returns `200 OK` if the HTTP process is responsive.

**Response (`200 OK`)**:
```json
{
  "status": "ok",
  "app": "struct-framework",
  "version": "1.0.0",
  "uptime": "1h24m10s"
}
```

---

### `GET /readyz`
Comprehensive readiness probe. Verifies database and message broker availability. Returns `503 Service Unavailable` if any registered dependency fails.

**Response (`200 OK` - Ready)**:
```json
{
  "status": "ready",
  "checks": {
    "database": "healthy",
    "broker": "healthy"
  }
}
```

**Response (`503 Service Unavailable` - Degraded)**:
```json
{
  "status": "unavailable",
  "checks": {
    "database": "unavailable",
    "broker": "healthy"
  }
}
```

---

### `GET /metrics`
Prometheus text exposition format. Exposes standard HTTP metrics and Go runtime indicators.

**Sample Output**:
```text
# HELP go_goroutines Number of goroutines currently existing
# TYPE go_goroutines gauge
go_goroutines 14
# HELP http_requests_total Total number of HTTP requests processed
# TYPE http_requests_total counter
http_requests_total{method="GET",pattern="/v1/users/{id}",status="200"} 42
# HELP http_request_duration_seconds HTTP request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005"} 38
http_request_duration_seconds_bucket{le="+Inf"} 42
http_request_duration_seconds_sum 0.1245
http_request_duration_seconds_count 42
```

---

### `GET /debug/pprof/*`
Runtime profiling endpoints (available when `PPROF_ENABLED=true`).
- `GET /debug/pprof/`: HTML profiling index.
- `GET /debug/pprof/cmdline`: Command-line invocation string.
- `GET /debug/pprof/profile`: CPU profile (30s sample).
- `GET /debug/pprof/symbol`: Program counters to symbols lookup.
- `GET /debug/pprof/trace`: Execution trace.
- `GET /debug/pprof/goroutine`: Goroutine stack dump.
- `GET /debug/pprof/heap`: Memory allocation profile.

---

## 2. User Management Endpoints

### `POST /v1/users`
Creates a new user account.

**Request Body**:
```json
{
  "email": "developer@example.com",
  "password": "CorrectHorseBatteryStaple123!",
  "locale": "en"
}
```

**Response (`201 Created`)**:
```json
{
  "id": "usr_9c8b7a6d5e4f",
  "email": "developer@example.com",
  "locale": "en",
  "created_at": "2026-09-15T08:00:00Z"
}
```

---

### `GET /v1/users/{id}`
Retrieves user account by identifier.

**Response (`200 OK`)**:
```json
{
  "id": "usr_9c8b7a6d5e4f",
  "email": "developer@example.com",
  "locale": "en",
  "created_at": "2026-09-15T08:00:00Z"
}
```

**Response (`404 Not Found`)**:
```json
{
  "error": "user not found",
  "code": "NOT_FOUND",
  "request_id": "req-98765"
}
```

---

## 3. Authentication & Security Endpoints
*(Available when `DB_DRIVER=postgres` or `DB_DRIVER=mysql`)*

### `POST /v1/auth/login`
Password authentication. Returns token pair or MFA challenge ticket if 2FA is active.

**Request**:
```json
{
  "email": "developer@example.com",
  "password": "CorrectHorseBatteryStaple123!"
}
```

**Response (`200 OK` - Standard Login)**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "rft_8a7b6c5d4e3f2a1b0c9d8e7f"
}
```

**Response (`200 OK` - MFA Challenge Required)**:
```json
{
  "mfa_required": true,
  "mfa_ticket": "mfa_ticket_0a1b2c3d4e5f"
}
```

---

### `POST /v1/auth/refresh`
Rotates an existing refresh token for a fresh token pair.

**Request**:
```json
{
  "refresh_token": "rft_8a7b6c5d4e3f2a1b0c9d8e7f"
}
```

**Response (`200 OK`)**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "rft_new_fresh_token_hash"
}
```

---

### `POST /v1/auth/logout`
Revokes the session corresponding to the submitted refresh token.

**Request**:
```json
{
  "refresh_token": "rft_8a7b6c5d4e3f2a1b0c9d8e7f"
}
```

**Response (`204 No Content`)**

---

### `POST /v1/auth/logout-all` ✱
Revokes all active sessions for the authenticated caller.

**Headers**:
`Authorization: Bearer <access_token>`

**Response (`204 No Content`)**

---

### `POST /v1/auth/totp/enroll` ✱
Generates a new TOTP enrollment secret and `otpauth://` URI.

**Response (`200 OK`)**:
```json
{
  "secret": "JBSWY3DPEHPK3PXP",
  "otpauth_uri": "otpauth://totp/Struct:developer@example.com?secret=JBSWY3DPEHPK3PXP&issuer=Struct"
}
```

---

### `POST /v1/auth/totp/confirm` ✱
Confirms TOTP enrollment with a valid 6-digit code. Returns single-use backup recovery codes.

**Request**:
```json
{
  "code": "583921"
}
```

**Response (`200 OK`)**:
```json
{
  "backup_codes": [
    "A1B2C-D3E4F",
    "G5H6I-J7K8L",
    "M9N0O-P1Q2R",
    "S3T4U-V5W6X",
    "Y7Z8A-B9C0D",
    "E1F2G-H3I4J",
    "K5L6M-N7O8P",
    "Q9R0S-T1U2V"
  ]
}
```

---

### `DELETE /v1/auth/totp` ✱
Disables TOTP 2FA and revokes all remaining backup codes.

**Response (`204 No Content`)**

---

### `GET /v1/auth/totp/backup-codes` ✱
Returns count of remaining unused backup codes.

**Response (`200 OK`)**:
```json
{
  "remaining": 7
}
```

---

### `POST /v1/auth/mfa/totp`
Completes login by redeeming an `mfa_ticket` with a 6-digit TOTP code or an unused backup code.

**Request**:
```json
{
  "ticket": "mfa_ticket_0a1b2c3d4e5f",
  "code": "583921"
}
```

**Response (`200 OK`)**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "rft_new_session_token"
}
```

---

### `POST /v1/auth/passkeys/register/begin` ✱
Initiates WebAuthn passkey registration ceremony.

**Response (`200 OK`)**:
```json
{
  "ceremony_id": "ceremony_12345",
  "challenge": "dGhpcy1pcy1hLXZhbGlkLWNoYWxsZW5nZQ",
  "rp_name": "Struct",
  "user_name": "developer@example.com"
}
```

---

### `POST /v1/auth/passkeys/register/finish` ✱
Completes WebAuthn passkey registration ceremony.

**Request**:
```json
{
  "ceremony_id": "ceremony_12345",
  "client_data_json": "base64url_encoded_client_data",
  "attestation_object": "base64url_encoded_cbor_attestation",
  "nickname": "MacBook Touch ID"
}
```

**Response (`201 Created`)**

---

### `GET /v1/auth/passkeys` ✱
Lists all registered passkeys for the caller.

**Response (`200 OK`)**:
```json
[
  {
    "id": "cred_a1b2c3d4",
    "nickname": "MacBook Touch ID",
    "created_at": "2026-09-15T08:30:00Z"
  }
]
```

---

### `POST /v1/auth/passkeys/login/begin`
Starts passwordless primary login using a WebAuthn passkey.

**Request**:
```json
{
  "email": "developer@example.com"
}
```

**Response (`200 OK`)**:
```json
{
  "ceremony_id": "ceremony_login_67890",
  "challenge": "base64url_challenge_bytes",
  "allow_credential_ids": ["base64url_credential_id"]
}
```

---

### `POST /v1/auth/passkeys/login/finish`
Completes passwordless WebAuthn login and issues session tokens.

**Request**:
```json
{
  "ceremony_id": "ceremony_login_67890",
  "credential_id": "base64url_credential_id",
  "client_data_json": "base64url_client_data",
  "authenticator_data": "base64url_auth_data",
  "signature": "base64url_signature"
}
```

**Response (`200 OK`)**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "rft_new_session_token"
}
```
