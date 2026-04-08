# HADES REST API Reference

## Overview

The HADES REST API exposes file scanning, metadata sanitization, and threat detection capabilities as HTTP endpoints. Built on FastAPI, it wraps the core `ExifScanner` and `EnhancedDetectionEngine` into a stateless service with SQLite-backed result persistence.

**Base URL:** `http://<host>:<port>/api/v1`

**Default:** `http://127.0.0.1:8666/api/v1`

**Content Types:**
- Requests: `multipart/form-data` (file uploads), `application/json` (JSON bodies), standard query parameters
- Responses: `application/json` (all endpoints except `/sanitize` which returns the file)

## Authentication

All endpoints except `/api/v1/health` require authentication. HADES supports two authentication methods:

### API Key Authentication

Pass an API key via the `X-API-Key` header:

```
X-API-Key: <your-api-key>
```

**Default development key:** `your-api-key-here`

API keys are configured in `cli/hades_config.json` under the `api.api_keys` array. For production, replace the default key with strong, unique keys.

```json
{
  "api": {
    "api_keys": ["your-production-key-here"]
  }
}
```

### JWT / RBAC Authentication (Enterprise)

When RBAC is enabled (`auth.rbac_enabled: true` in config), users can authenticate via the login endpoint to receive a per-user API key. User management, role-based access control, and SSO are available through the `/api/v1/auth/` endpoints documented below.

---

## Core Endpoints

### GET /api/v1/health

Health check endpoint. Does **not** require authentication.

Returns server status, version, uptime, and engine availability.

**Response (200):**

```json
{
  "status": "healthy",
  "version": "0.7.1",
  "uptime_seconds": 142.57,
  "engines": {
    "exif_scanner": "ok",
    "enhanced_detection": "ok"
  }
}
```

Engine status is `"ok"` when the engine initializes successfully, or `"degraded"` if initialization fails (e.g., missing YARA library).

**Example:**

```bash
curl http://localhost:8666/api/v1/health
```

---

### GET /api/v1/rules/status

Returns information about loaded YARA rule files.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "rules_directory": "/path/to/rules",
  "rule_count": 5,
  "rule_files": [
    "enhanced_detection.yara",
    "advanced_threats.yar",
    "enterprise_threats.yar",
    "steganography.yar",
    "metadata_threats.yar"
  ],
  "last_updated": "2026-02-17T10:30:00+00:00"
}
```

`last_updated` is `null` if no rule files are found.

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/rules/status
```

---

### POST /api/v1/scan/file

Upload and scan a single file. Runs both the basic `ExifScanner` and the `EnhancedDetectionEngine`, then persists the result for later retrieval.

**Headers:** `X-API-Key` (required)

**Request:** `multipart/form-data` with a field named `file`.

**Response (200):**

```json
{
  "scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "file_name": "suspicious.jpg",
  "file_size": 245760,
  "status": "suspicious",
  "threat_score": 72.5,
  "threat_level": "high",
  "basic_scan": {
    "status": "suspicious",
    "scan_time": 0.34,
    "findings": [
      "Embedded executable detected in EXIF comment"
    ]
  },
  "enhanced_detection": {
    "overall_threat_score": 72.5,
    "threat_level": "high",
    "processing_time": 1.23,
    "detection_summary": "Multiple threat indicators found",
    "yara_matches": [],
    "ioc_matches": [],
    "heuristic_findings": []
  },
  "metadata_summary": {
    "fields_extracted": 15
  },
  "scanned_at": "2026-02-17T10:35:00+00:00"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -F "file=@suspicious.jpg" \
     http://localhost:8666/api/v1/scan/file
```

---

### POST /api/v1/scan/batch

Upload and scan multiple files in a single request.

**Headers:** `X-API-Key` (required)

**Request:** `multipart/form-data` with one or more fields named `files`.

**Response (200):**

```json
{
  "total_files": 3,
  "scanned": 2,
  "errors": [
    {
      "file_name": "huge_file.bin",
      "error": "File exceeds size limit"
    }
  ],
  "results": [
    {
      "scan_id": "...",
      "file_name": "image1.jpg",
      "threat_score": 0.0,
      "threat_level": "safe"
    },
    {
      "scan_id": "...",
      "file_name": "document.pdf",
      "threat_score": 45.0,
      "threat_level": "medium"
    }
  ]
}
```

Each entry in `results` has the same structure as the single-file scan response.

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -F "files=@image1.jpg" \
     -F "files=@document.pdf" \
     -F "files=@archive.zip" \
     http://localhost:8666/api/v1/scan/batch
```

---

### GET /api/v1/scan/{scan_id}

Retrieve a previously stored scan result by its ID.

**Headers:** `X-API-Key` (required)

**Path Parameters:**
- `scan_id` (string, required) — UUID returned from a prior scan.

**Response (200):**

```json
{
  "scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "file_name": "suspicious.jpg",
  "file_size": 245760,
  "status": "suspicious",
  "threat_score": 72.5,
  "threat_level": "high",
  "created_at": "2026-02-17T10:35:00+00:00",
  "results": { }
}
```

The `results` field contains the full scan result object as stored.

**Response (404):**

```json
{
  "error": "Not Found",
  "message": "Scan ID 'invalid-id' not found"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/scan/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

### POST /api/v1/sanitize

Upload a file, strip its metadata, and download the sanitized version.

**Headers:** `X-API-Key` (required)

**Request:** `multipart/form-data` with:
- `file` (required) — The file to sanitize.
- `remove_all` (optional, string) — Set to `"true"` to remove all metadata fields. Default: `"false"`.
- `fields_to_remove` (optional, repeated string) — Specific metadata field names to remove. Ignored when `remove_all` is `"true"`.

**Response (200):** The sanitized file is returned as a binary download with filename `sanitized_<original_name>`.

**Response (500) on sanitization failure:**

```json
{
  "error": "Sanitization Failed",
  "message": "Description of the error"
}
```

**Example — remove all metadata:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -F "file=@photo.jpg" \
     -F "remove_all=true" \
     -o sanitized_photo.jpg \
     http://localhost:8666/api/v1/sanitize
```

**Example — remove specific fields:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -F "file=@photo.jpg" \
     -F "fields_to_remove=GPS:GPSLatitude" \
     -F "fields_to_remove=GPS:GPSLongitude" \
     -o sanitized_photo.jpg \
     http://localhost:8666/api/v1/sanitize
```

---

## Authentication & User Management

The authentication endpoints manage user login, RBAC user administration, API key rotation, license information, and SSO flows. These endpoints are registered at `/api/v1/auth/` and require the enterprise auth module (`core/auth/`).

**Requirements:** RBAC must be enabled in `cli/hades_config.json` with `auth.rbac_enabled: true`. SSO endpoints require either OIDC or SAML provider configuration.

---

### POST /api/v1/auth/login

Authenticate with username and password. Returns the user's API key, role, and tenant information.

**Headers:** None required (this is the authentication entry point).

**Request body (JSON):**

| Field      | Type   | Required | Description       |
|------------|--------|----------|-------------------|
| `username` | string | Yes      | User's username   |
| `password` | string | Yes      | User's password   |

**Response (200):**

```json
{
  "api_key": "hades-key-a1b2c3d4e5f6...",
  "role": "analyst",
  "tenant_id": "tenant-001",
  "user_id": "usr-12345678",
  "username": "jdoe"
}
```

**Response (401) — invalid credentials:**

```json
{
  "error": "Unauthorized",
  "message": "Invalid credentials"
}
```

**Response (501) — RBAC not configured:**

```json
{
  "error": "Not Implemented",
  "message": "RBAC not configured"
}
```

**Example:**

```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"username": "analyst1", "password": "s3cur3pass"}' \
     http://localhost:8666/api/v1/auth/login
```

---

### GET /api/v1/auth/users

List all users. Requires admin role.

**Headers:** `X-API-Key` (required, admin user key)

**Query Parameters:**

| Parameter   | Type   | Required | Description                          |
|-------------|--------|----------|--------------------------------------|
| `tenant_id` | string | No       | Filter users by tenant ID            |

**Response (200):**

```json
{
  "users": [
    {
      "user_id": "usr-12345678",
      "username": "admin",
      "role": "admin",
      "tenant_id": null,
      "is_active": true,
      "created_at": "2026-02-17T08:00:00+00:00"
    },
    {
      "user_id": "usr-87654321",
      "username": "analyst1",
      "role": "analyst",
      "tenant_id": "tenant-001",
      "is_active": true,
      "created_at": "2026-02-18T10:15:00+00:00"
    }
  ]
}
```

**Example:**

```bash
curl -H "X-API-Key: <admin-api-key>" \
     http://localhost:8666/api/v1/auth/users
```

---

### POST /api/v1/auth/users

Create a new user. Requires admin role.

**Headers:** `X-API-Key` (required, admin user key)

**Request body (JSON):**

| Field       | Type   | Required | Description                                        |
|-------------|--------|----------|----------------------------------------------------|
| `username`  | string | Yes      | Unique username                                    |
| `password`  | string | Yes      | User password                                      |
| `role`      | string | No       | Role: `admin`, `analyst`, or `viewer` (default: `viewer`) |
| `tenant_id` | string | No       | Assign user to a tenant                            |

**Response (201):**

```json
{
  "user_id": "usr-a1b2c3d4",
  "username": "new_analyst",
  "role": "analyst",
  "api_key": "hades-key-f6e5d4c3b2a1...",
  "tenant_id": "tenant-001"
}
```

**Response (400) — missing fields:**

```json
{
  "error": "Bad Request",
  "message": "username and password required"
}
```

**Response (409) — duplicate username:**

```json
{
  "error": "Conflict",
  "message": "Username 'analyst1' already exists"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: <admin-api-key>" \
     -H "Content-Type: application/json" \
     -d '{"username": "new_analyst", "password": "str0ngP@ss!", "role": "analyst"}' \
     http://localhost:8666/api/v1/auth/users
```

---

### PUT /api/v1/auth/users/{user_id}

Update a user's role, active status, or tenant assignment. Requires admin role.

**Headers:** `X-API-Key` (required, admin user key)

**Path Parameters:**
- `user_id` (string, required) — The ID of the user to update.

**Request body (JSON):**

| Field       | Type    | Required | Description                         |
|-------------|---------|----------|-------------------------------------|
| `role`      | string  | No       | New role: `admin`, `analyst`, `viewer` |
| `is_active` | boolean | No       | Enable or disable the user          |
| `tenant_id` | string  | No       | New tenant assignment               |

**Response (200):**

```json
{
  "user_id": "usr-87654321",
  "username": "analyst1",
  "role": "admin",
  "is_active": true,
  "tenant_id": "tenant-001"
}
```

**Response (404) — user not found:**

```json
{
  "error": "User not found"
}
```

**Example:**

```bash
curl -X PUT \
     -H "X-API-Key: <admin-api-key>" \
     -H "Content-Type: application/json" \
     -d '{"role": "admin"}' \
     http://localhost:8666/api/v1/auth/users/usr-87654321
```

---

### POST /api/v1/auth/users/{user_id}/rotate-key

Rotate a user's API key. Admin users can rotate any key; non-admin users can only rotate their own.

**Headers:** `X-API-Key` (required)

**Path Parameters:**
- `user_id` (string, required) — The ID of the user whose key to rotate.

**Response (200):**

```json
{
  "user_id": "usr-87654321",
  "api_key": "hades-key-new-rotated-value...",
  "message": "API key rotated successfully"
}
```

**Response (403) — insufficient permissions:**

```json
{
  "error": "Forbidden",
  "message": "Can only rotate your own key or be admin"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: <your-api-key>" \
     http://localhost:8666/api/v1/auth/users/usr-87654321/rotate-key
```

---

### GET /api/v1/auth/license

Return current license information. Returns free tier defaults when no license key is configured.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "tier": "enterprise",
  "tenant_name": "Acme Corp",
  "max_users": 50,
  "features": [
    "rbac",
    "sso",
    "multi_tenant",
    "siem_forwarding",
    "threat_intel",
    "ml_detection"
  ],
  "expires_at": "2027-02-17T00:00:00+00:00",
  "issued_at": "2026-02-17T00:00:00+00:00",
  "valid": true
}
```

**Response (200) — free tier (no license key):**

```json
{
  "tier": "free",
  "tenant_name": "",
  "max_users": 1,
  "features": ["basic_scanning", "yara_detection"],
  "expires_at": null,
  "issued_at": "",
  "valid": true
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/auth/license
```

---

### GET /api/v1/auth/sso/oidc/authorize

Start an OIDC single sign-on authorization flow. Returns the authorization URL to redirect the user to the identity provider.

**Headers:** `X-API-Key` (required)

**Requirements:** OIDC provider must be configured in `cli/hades_config.json` under `auth.sso.oidc`.

**Response (200):**

```json
{
  "authorization_url": "https://idp.example.com/oauth2/authorize?client_id=hades&...",
  "state": "rAnDoM_StAtE_tOkEn",
  "nonce": "rAnDoM_nOnCe_vAlUe"
}
```

**Response (501) — OIDC not configured:**

```json
{
  "error": "Not Implemented",
  "message": "OIDC not configured"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/auth/sso/oidc/authorize
```

---

### POST /api/v1/auth/sso/oidc/callback

Handle the OIDC callback after the user authenticates with the identity provider. Validates the ID token and provisions the user via JIT (just-in-time) provisioning if RBAC is enabled.

**Headers:** `X-API-Key` (required)

**Request body (JSON):**

| Field      | Type   | Required | Description                           |
|------------|--------|----------|---------------------------------------|
| `id_token` | string | Yes      | The ID token received from the IdP    |

**Response (200) — with RBAC:**

```json
{
  "api_key": "hades-key-sso-provisioned...",
  "role": "analyst",
  "user_id": "usr-sso-12345",
  "username": "jdoe@example.com",
  "sso_provider": "oidc"
}
```

**Response (200) — without RBAC:**

```json
{
  "sso_user": {
    "sub": "auth0|12345",
    "email": "jdoe@example.com",
    "name": "Jane Doe",
    "groups": ["analysts", "incident-response"]
  },
  "sso_provider": "oidc"
}
```

**Response (401) — invalid token:**

```json
{
  "error": "Unauthorized",
  "message": "Token validation failed: expired"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6..."}' \
     http://localhost:8666/api/v1/auth/sso/oidc/callback
```

---

## Storage Administration

The storage admin endpoints provide operational visibility and management for the enterprise storage backend, including database status, Redis monitoring, encryption state, and multi-tenant administration. These endpoints are registered at `/api/v1/admin/` and require the enterprise storage module (`core/storage/`).

**Requirements:** Enterprise storage must be available (`core/storage/` modules installed and configured in `cli/hades_config.json`).

---

### GET /api/v1/admin/database/status

Return the database backend type, connection status, and table row counts.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "backend": "sqlite",
  "status": "connected",
  "tables": {
    "scan_results": 142,
    "evidence_log": 58,
    "cases": 3
  }
}
```

**Response (503) — no backend configured:**

```json
{
  "detail": "No database backend configured"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/admin/database/status
```

---

### POST /api/v1/admin/database/migrate

Run database schema migrations. Accepts a list of SQL migration statements.

**Headers:** `X-API-Key` (required)

**Request body (JSON):** An array of SQL migration strings.

```json
[
  "ALTER TABLE scan_results ADD COLUMN tenant_id TEXT",
  "CREATE INDEX idx_tenant ON scan_results(tenant_id)"
]
```

**Response (200):**

```json
{
  "status": "ok",
  "migrations_applied": "2"
}
```

**Response (503) — no backend configured:**

```json
{
  "detail": "No database backend configured"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '["ALTER TABLE scan_results ADD COLUMN tenant_id TEXT"]' \
     http://localhost:8666/api/v1/admin/database/migrate
```

---

### GET /api/v1/admin/redis/status

Return the Redis connection status. Returns a not-configured response when Redis is not enabled.

**Headers:** `X-API-Key` (required)

**Response (200) — connected:**

```json
{
  "enabled": true,
  "status": "connected"
}
```

**Response (200) — not configured:**

```json
{
  "enabled": false,
  "status": "not_configured"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/admin/redis/status
```

---

### GET /api/v1/admin/encryption/status

Return the encryption subsystem status. Never exposes the actual encryption key.

**Headers:** `X-API-Key` (required)

**Response (200) — enabled:**

```json
{
  "enabled": true,
  "encryption_available": true
}
```

**Response (200) — not configured:**

```json
{
  "enabled": false
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/admin/encryption/status
```

---

### GET /api/v1/admin/tenants

List all tenants. Requires the tenant management subsystem to be enabled.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
[
  {
    "tenant_id": "tenant-001",
    "name": "Acme Corp",
    "config": {"max_scans_per_day": 1000},
    "is_active": true,
    "created_at": "2026-02-17T08:00:00+00:00"
  },
  {
    "tenant_id": "tenant-002",
    "name": "Globex Inc",
    "config": {},
    "is_active": true,
    "created_at": "2026-02-18T09:30:00+00:00"
  }
]
```

**Response (503) — tenant management not configured:**

```json
{
  "detail": "Tenant management not configured"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/admin/tenants
```

---

### POST /api/v1/admin/tenants

Create a new tenant.

**Headers:** `X-API-Key` (required)

**Request body (JSON):**

| Field    | Type   | Required | Description                        |
|----------|--------|----------|------------------------------------|
| `name`   | string | Yes      | Tenant display name                |
| `config` | object | No       | Tenant-specific configuration map  |

**Response (200):**

```json
{
  "tenant_id": "tenant-003",
  "name": "New Organization",
  "config": {"max_scans_per_day": 500},
  "is_active": true,
  "created_at": "2026-02-21T12:00:00+00:00"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"name": "New Organization", "config": {"max_scans_per_day": 500}}' \
     http://localhost:8666/api/v1/admin/tenants
```

---

### PUT /api/v1/admin/tenants/{tenant_id}

Update a tenant's name, configuration, or active status.

**Headers:** `X-API-Key` (required)

**Path Parameters:**
- `tenant_id` (string, required) — The ID of the tenant to update.

**Request body (JSON):**

| Field       | Type    | Required | Description                        |
|-------------|---------|----------|------------------------------------|
| `name`      | string  | No       | New tenant display name            |
| `config`    | object  | No       | Updated configuration map          |
| `is_active` | boolean | No       | Enable or disable the tenant       |

**Response (200):**

```json
{
  "tenant_id": "tenant-003",
  "name": "New Organization",
  "config": {"max_scans_per_day": 1000},
  "is_active": true,
  "created_at": "2026-02-21T12:00:00+00:00"
}
```

**Response (404) — tenant not found:**

```json
{
  "detail": "Tenant not found"
}
```

**Example:**

```bash
curl -X PUT \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"is_active": false}' \
     http://localhost:8666/api/v1/admin/tenants/tenant-003
```

---

## MITRE ATT&CK Mapping

The MITRE ATT&CK endpoints provide technique lookups and the full mapping matrix used by HADES to contextualize detection findings. Powered by `core/mitre_mapping.py`.

**Requirements:** The `mitre_mapping` module must be available (included in the standard installation).

---

### GET /api/v1/mitre/matrix

Return the full MITRE ATT&CK technique matrix organized by tactic. Each tactic maps to a list of technique objects with ID, name, description, URL, and severity.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "matrix": {
    "Execution": [
      {
        "technique_id": "T1059",
        "tactic": "Execution",
        "technique_name": "Command and Scripting Interpreter",
        "description": "Adversaries may abuse command and script interpreters to execute commands, scripts, or binaries.",
        "url": "https://attack.mitre.org/techniques/T1059/",
        "severity": "high"
      },
      {
        "technique_id": "T1059.001",
        "tactic": "Execution",
        "technique_name": "PowerShell",
        "description": "Adversaries may abuse PowerShell commands and scripts for execution.",
        "url": "https://attack.mitre.org/techniques/T1059/001/",
        "severity": "high"
      }
    ],
    "Initial Access": [
      {
        "technique_id": "T1190",
        "tactic": "Initial Access",
        "technique_name": "Exploit Public-Facing Application",
        "description": "Adversaries may attempt to exploit a weakness in an Internet-facing host or program.",
        "url": "https://attack.mitre.org/techniques/T1190/",
        "severity": "critical"
      }
    ],
    "Defense Evasion": [
      {
        "technique_id": "T1027",
        "tactic": "Defense Evasion",
        "technique_name": "Obfuscated Files or Information",
        "description": "Adversaries may attempt to make payloads difficult to discover or analyze.",
        "url": "https://attack.mitre.org/techniques/T1027/",
        "severity": "high"
      }
    ]
  }
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/mitre/matrix
```

---

### GET /api/v1/mitre/lookup

Look up a specific MITRE ATT&CK technique by its ID.

**Headers:** `X-API-Key` (required)

**Query Parameters:**

| Parameter   | Type   | Required | Description                                  |
|-------------|--------|----------|----------------------------------------------|
| `technique` | string | Yes      | MITRE technique ID (e.g., `T1059`, `T1027.003`) |

**Response (200):**

```json
{
  "technique_id": "T1059",
  "tactic": "Execution",
  "technique_name": "Command and Scripting Interpreter",
  "description": "Adversaries may abuse command and script interpreters to execute commands, scripts, or binaries.",
  "url": "https://attack.mitre.org/techniques/T1059/",
  "severity": "high"
}
```

**Response (400) — missing parameter:**

```json
{
  "error": "Bad Request",
  "message": "Missing 'technique' query parameter"
}
```

**Response (404) — technique not found:**

```json
{
  "error": "Not Found",
  "message": "Technique 'T9999' not found"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/mitre/lookup?technique=T1059"
```

---

## Threat Intelligence

The threat intelligence endpoints provide cloud-based IOC lookups, cache management, enrichment, and feed scheduling powered by `core/cloud_threat_intel.py` and `core/threat_intel_scheduler.py`.

**Requirements:** The `requests` Python package must be installed (`pip install requests`). Provider API keys must be configured in `cli/hades_config.json` under `threat_intel.providers`.

---

### GET /api/v1/threat-intel/lookup

Look up an indicator of compromise (hash, IP, URL, or domain) against configured threat intelligence providers.

**Headers:** `X-API-Key` (required)

**Query Parameters:**

| Parameter   | Type   | Required | Description                                                    |
|-------------|--------|----------|----------------------------------------------------------------|
| `ioc`       | string | Yes      | The indicator to look up (file hash, IP address, URL, domain)  |
| `providers` | string | No       | Comma-separated provider names to query (default: all configured) |

**Response (200):**

```json
{
  "ioc": "d41d8cd98f00b204e9800998ecf8427e",
  "ioc_type": "hash",
  "consensus": {
    "verdict": "malicious",
    "confidence": 85,
    "provider_count": 3,
    "verdicts": {
      "malicious": 2,
      "suspicious": 1,
      "clean": 0
    }
  },
  "results": [
    {
      "source": "virustotal",
      "verdict": "malicious",
      "confidence": 92,
      "details": "Detected by 45/70 engines",
      "raw_response": {}
    },
    {
      "source": "malwarebazaar",
      "verdict": "malicious",
      "confidence": 100,
      "details": "Known malware sample: Emotet",
      "raw_response": {}
    },
    {
      "source": "otx",
      "verdict": "suspicious",
      "confidence": 60,
      "details": "Found in 3 OTX pulses",
      "raw_response": {}
    }
  ]
}
```

**Response (400) — missing IOC parameter:**

```json
{
  "error": "Bad Request",
  "message": "'ioc' query parameter is required"
}
```

**Example — hash lookup:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/threat-intel/lookup?ioc=d41d8cd98f00b204e9800998ecf8427e"
```

**Example — IP lookup with specific providers:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/threat-intel/lookup?ioc=8.8.8.8&providers=abuseipdb,otx"
```

---

### GET /api/v1/threat-intel/cache/stats

Retrieve statistics about the threat intelligence result cache.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "total_entries": 142,
  "hit_count": 580,
  "miss_count": 142,
  "hit_rate": 0.803,
  "expired_entries": 12,
  "cache_size_bytes": 45056,
  "oldest_entry": "2026-02-17T08:00:00+00:00",
  "newest_entry": "2026-02-18T14:30:00+00:00"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/threat-intel/cache/stats
```

---

### DELETE /api/v1/threat-intel/cache

Flush all cached threat intelligence results. Use this after rotating API keys or when cached data is suspected to be stale.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "message": "Threat intel cache cleared",
  "entries_removed": 142
}
```

**Example:**

```bash
curl -X DELETE -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/threat-intel/cache
```

---

### GET /api/v1/threat-intel/scheduler

Return the status of the threat intelligence feed scheduler, including per-feed configuration, last pull times, and IOC counts.

**Headers:** `X-API-Key` (required)

**Requirements:** The `threat_intel_scheduler` module must be available and feeds must be configured in `cli/hades_config.json` under `threat_intel_scheduler.feeds`.

**Response (200):**

```json
{
  "status": {
    "running": true,
    "feeds": {
      "otx-malware": {
        "enabled": true,
        "feed_type": "otx_pulse",
        "interval_hours": 6.0,
        "last_pulled": "2026-02-21T08:00:00+00:00",
        "next_pull": "2026-02-21T14:00:00+00:00",
        "ioc_count": 1247
      },
      "abuse-ch-urlhaus": {
        "enabled": true,
        "feed_type": "csv",
        "interval_hours": 12.0,
        "last_pulled": "2026-02-21T06:00:00+00:00",
        "next_pull": "2026-02-21T18:00:00+00:00",
        "ioc_count": 583
      }
    },
    "total_iocs": 1830
  }
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/threat-intel/scheduler
```

---

### POST /api/v1/threat-intel/scheduler

Force an immediate pull of all configured threat intelligence feeds.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "message": "Feed update triggered",
  "results": {
    "otx-malware": true,
    "abuse-ch-urlhaus": true
  }
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/threat-intel/scheduler
```

---

## YARA Rule Builder

The YARA rule builder endpoints allow generating YARA rules from templates with customizable parameters. Powered by `core/yara_rule_builder.py`.

**Requirements:** The `yara_rule_builder` module must be available (included in the standard installation).

---

### GET /api/v1/rules/templates

List all available YARA rule templates with their name, description, category, and severity.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "templates": [
    {
      "name": "suspicious_metadata",
      "description": "Detect suspicious metadata patterns in file headers",
      "category": "metadata",
      "severity": "medium"
    },
    {
      "name": "embedded_executable",
      "description": "Detect executable content embedded in non-executable files",
      "category": "payload",
      "severity": "high"
    },
    {
      "name": "obfuscated_content",
      "description": "Detect Base64 or encoded payloads in metadata fields",
      "category": "evasion",
      "severity": "high"
    }
  ]
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/rules/templates
```

---

### POST /api/v1/rules/build

Build a YARA rule from a named template with the provided parameters. Returns the generated rule source and validation status.

**Headers:** `X-API-Key` (required)

**Request body (JSON):**

| Field      | Type   | Required | Description                                      |
|------------|--------|----------|--------------------------------------------------|
| `template` | string | Yes      | Template name (from `/rules/templates`)           |
| `params`   | object | No       | Template-specific parameters (varies by template) |

**Response (200):**

```json
{
  "rule": "rule suspicious_metadata_custom {\n    meta:\n        description = \"Custom metadata detection rule\"\n        severity = \"medium\"\n    strings:\n        $s1 = \"eval(\" nocase\n    condition:\n        $s1\n}",
  "valid": true,
  "error": null
}
```

**Response (400) — missing template:**

```json
{
  "error": "Bad Request",
  "message": "Missing 'template' field"
}
```

**Response (400) — invalid template name:**

```json
{
  "error": "Bad Request",
  "message": "Template 'nonexistent' not found"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"template": "suspicious_metadata", "params": {"pattern": "eval("}}' \
     http://localhost:8666/api/v1/rules/build
```

---

## ML Anomaly Detection

The ML anomaly detection endpoints expose the machine-learning-based metadata anomaly detection engine powered by `core/ml_detection.py`. These endpoints require `scikit-learn` and `numpy` to be installed.

---

### GET /api/v1/ml/status

Returns the current state of the ML detection engine, including whether a model is loaded and training statistics.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "model_loaded": true,
  "is_trained": true,
  "feature_count": 15,
  "training_samples": 500,
  "contamination": 0.1,
  "model_path": "models/baseline_model.joblib"
}
```

When the ML module is not available:

```json
{
  "available": false,
  "message": "ML detection module not available. Install scikit-learn and numpy."
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/ml/status
```

---

### POST /api/v1/ml/retrain

Trigger model retraining using accumulated training samples. The model is retrained in place and the updated version is saved.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "message": "Model retrained successfully",
  "training_samples": 750,
  "model_path": "models/baseline_model.joblib"
}
```

**Response (400) if no training samples are available:**

```json
{
  "error": "Bad Request",
  "message": "No training samples available. Run scans or use --ml-train first."
}
```

**Response (503) if the ML module is not available:**

```json
{
  "error": "Service Unavailable",
  "message": "ML detection module not available"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/ml/retrain
```

---

### GET /api/v1/ml/features

Extract the metadata feature vector for a specific file without scoring it. Useful for debugging and understanding what the model sees.

**Headers:** `X-API-Key` (required)

**Query Parameters:**

| Parameter   | Type   | Required | Description                     |
|-------------|--------|----------|---------------------------------|
| `file_path` | string | Yes      | Absolute path to the file to extract features from |

**Response (200):**

```json
{
  "file_path": "/evidence/suspicious.jpg",
  "features": {
    "file_size": 245760,
    "file_size_log": 12.41,
    "entropy": 7.92,
    "metadata_field_count": 15,
    "metadata_total_length": 342,
    "has_gps": 0,
    "has_thumbnail": 1,
    "suspicious_field_count": 0,
    "creation_modify_delta": 3600.0,
    "extension_mimetype_mismatch": 0,
    "header_magic_valid": 1,
    "embedded_file_count": 0,
    "null_byte_ratio": 0.02,
    "printable_ratio": 0.15,
    "longest_run_length": 128
  },
  "feature_vector": [245760, 12.41, 7.92, 15, 342, 0, 1, 0, 3600.0, 0, 1, 0, 0.02, 0.15, 128]
}
```

**Response (400) — missing file_path:**

```json
{
  "error": "Bad Request",
  "message": "'file_path' query parameter is required"
}
```

**Response (404) — file not found:**

```json
{
  "error": "Not Found",
  "message": "File not found: /nonexistent/file.jpg"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/ml/features?file_path=/evidence/suspicious.jpg"
```

---

### GET /api/v1/ml/ensemble/status

Return the status of the ML ensemble detector, including the number of features and feature names used by the extended feature extractor.

**Headers:** `X-API-Key` (required)

**Requirements:** The `ml_ensemble` module must be available (`scikit-learn` and `numpy` installed).

**Response (200):**

```json
{
  "ensemble_available": true,
  "feature_count": 25,
  "feature_names": [
    "file_size",
    "file_size_log",
    "entropy",
    "metadata_field_count",
    "metadata_total_length",
    "has_gps",
    "has_thumbnail",
    "suspicious_field_count",
    "creation_modify_delta",
    "extension_mimetype_mismatch",
    "header_magic_valid",
    "embedded_file_count",
    "null_byte_ratio",
    "printable_ratio",
    "longest_run_length",
    "script_tag_count",
    "php_pattern_count",
    "shell_command_count",
    "iframe_count",
    "encoding_layer_depth",
    "suspicious_url_count",
    "sql_pattern_count",
    "binary_in_text_ratio",
    "metadata_entropy",
    "field_name_anomaly_score"
  ]
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/ml/ensemble/status
```

---

## Behavioral Analysis

The behavioral analysis endpoints expose cross-file campaign detection powered by `core/behavioral_analysis.py`. The system correlates scan results across multiple files to identify shared IOCs, similar metadata patterns, and temporal clustering.

**Requirements:** The `behavioral_analysis` module must be available (included in the standard installation).

---

### GET /api/v1/behavioral/campaigns

List detected threat campaigns with related files, shared indicators, and confidence scores.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "campaigns": [
    {
      "campaign_id": "camp-a1b2c3d4",
      "related_files": [
        "/evidence/invoice_jan.docx",
        "/evidence/invoice_feb.docx",
        "/evidence/macro_helper.xlsm"
      ],
      "shared_indicators": [
        {"type": "domain", "value": "c2.malicious-example.com"},
        {"type": "hash", "value": "d41d8cd98f00b204e9800998ecf8427e"}
      ],
      "confidence": 0.85,
      "first_seen": "2026-02-17T08:00:00+00:00",
      "last_seen": "2026-02-20T14:30:00+00:00",
      "tactic_pattern": "Execution -> Defense Evasion -> Exfiltration"
    }
  ],
  "file_count": 42,
  "edge_count": 18
}
```

**Response (200) — no campaigns detected:**

```json
{
  "campaigns": [],
  "file_count": 5,
  "edge_count": 0
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/behavioral/campaigns
```

---

## File Monitoring

The monitoring endpoints manage a real-time file watcher powered by `core/file_monitor.py`. When running, the monitor watches directories for new or modified files, automatically scans them through both `ExifScanner` and `EnhancedDetectionEngine`, and generates alerts (JSON Lines log + optional webhook) when threat scores exceed a configurable threshold.

**Requirements:** The `watchdog` Python package must be installed (`pip install watchdog>=3.0.0`). The `requests` package is optional and only needed for webhook delivery.

---

### GET /api/v1/monitor/status

Returns the current state of the file monitor.

**Headers:** `X-API-Key` (required)

**Response (200) when a monitor is running:**

```json
{
  "running": true,
  "watched_directories": ["/evidence/incoming"],
  "recursive": true,
  "files_scanned": 42,
  "alerts_generated": 3,
  "last_alert": {
    "timestamp": "2026-02-17T14:22:01+00:00",
    "file_path": "/evidence/incoming/suspicious.jpg",
    "file_hash": "a1b2c3...",
    "threat_score": 85.0,
    "threat_level": "critical",
    "event_type": "created",
    "findings_summary": "Embedded executable detected in EXIF comment",
    "basic_status": "suspicious",
    "enhanced_summary": {}
  },
  "started_at": "2026-02-17T14:00:00+00:00",
  "alert_threshold": 7,
  "alerts_log": "/path/to/logs/alerts.json"
}
```

**Response (200) when no monitor has been started:**

```json
{
  "running": false,
  "message": "No monitor has been started"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/monitor/status
```

---

### POST /api/v1/monitor/start

Start the file monitor on one or more directories.

**Headers:** `X-API-Key` (required)

**Request body (JSON):**

| Field              | Type     | Required | Description                                       |
|--------------------|----------|----------|---------------------------------------------------|
| `directories`      | string[] | Yes      | Directories to watch                              |
| `recursive`        | boolean  | No       | Watch subdirectories (default: `true`)            |
| `alert_threshold`  | integer  | No       | Minimum threat score for alerts (default: `7`)    |
| `webhook_url`      | string   | No       | URL to POST alert payloads to                     |
| `scan_delay_seconds` | integer | No      | Debounce delay before scanning (default: `2`)     |

**Response (200):**

```json
{
  "message": "Monitor started",
  "status": {
    "running": true,
    "watched_directories": ["/evidence/incoming"],
    "recursive": true,
    "files_scanned": 0,
    "alerts_generated": 0,
    "last_alert": null,
    "started_at": "2026-02-17T14:00:00+00:00",
    "alert_threshold": 7,
    "alerts_log": "/path/to/logs/alerts.json"
  }
}
```

**Response (409) if monitor is already running:**

```json
{
  "error": "Conflict",
  "message": "Monitor is already running. Stop it first.",
  "status": { }
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"directories": ["/evidence/incoming"], "alert_threshold": 5}' \
     http://localhost:8666/api/v1/monitor/start
```

---

### POST /api/v1/monitor/stop

Stop the currently running file monitor.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "message": "Monitor stopped",
  "status": {
    "running": false,
    "watched_directories": ["/evidence/incoming"],
    "files_scanned": 42,
    "alerts_generated": 3,
    "last_alert": { },
    "started_at": "2026-02-17T14:00:00+00:00",
    "alert_threshold": 7,
    "alerts_log": "/path/to/logs/alerts.json"
  }
}
```

**Response (409) if no monitor is running:**

```json
{
  "error": "Conflict",
  "message": "No monitor is currently running"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/monitor/stop
```

---

## Plugins

The plugin endpoints manage the HADES detection plugin system powered by `core/plugin_api.py`.

---

### GET /api/v1/plugins

List all loaded plugins with their name, version, author, and supported file types.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "plugins": [
    {
      "name": "Known-Bad Hash Checker",
      "version": "1.0.0",
      "author": "DarkHorse Information Security LLC",
      "supported_types": [],
      "status": "loaded"
    }
  ]
}
```

If the plugin system is not available, the response returns an empty list with a message:

```json
{
  "plugins": [],
  "message": "Plugin system is not available"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/plugins
```

---

### POST /api/v1/plugins/reload

Unload all plugins, re-discover, and re-load them. Useful after adding or modifying plugin files.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "message": "Plugins reloaded",
  "results": {
    "Known-Bad Hash Checker": "ok",
    "Entropy Analyzer": "ok"
  },
  "plugins": [
    {
      "name": "Known-Bad Hash Checker",
      "version": "1.0.0",
      "author": "DarkHorse Information Security LLC",
      "supported_types": [],
      "status": "loaded"
    }
  ]
}
```

**Response (503) if the plugin system is not available:**

```json
{
  "error": "Service Unavailable",
  "message": "Plugin system is not available"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/plugins/reload
```

---

## Plugin Marketplace

The plugin marketplace endpoints provide discovery, installation, removal, and update capabilities for HADES plugins. Powered by `plugins/registry.py`.

**Requirements:** The `plugins/registry.py` module must be available (included in the standard installation).

---

### GET /api/v1/plugins/marketplace/search

Search the plugin marketplace by keyword, category, or supported file type.

**Headers:** `X-API-Key` (required)

**Query Parameters:**

| Parameter   | Type   | Required | Description                                                    |
|-------------|--------|----------|----------------------------------------------------------------|
| `query`     | string | No       | Search keyword to match against plugin name and description    |
| `category`  | string | No       | Filter by category: `detection`, `enrichment`, `reporting`, `integration` |
| `file_type` | string | No       | Filter by supported file type (e.g., `.jpg`, `.pdf`)           |

**Response (200):**

```json
{
  "plugins": [
    {
      "name": "steganography_detector",
      "version": "1.2.0",
      "author": "DarkHorse Information Security LLC",
      "description": "Detect steganographic content hidden in image files",
      "homepage_url": "",
      "download_url": "",
      "min_hades_version": "0.5.0",
      "dependencies": [],
      "file_types": [".jpg", ".png", ".bmp"],
      "category": "detection",
      "sha256": "a1b2c3d4...",
      "license": "Proprietary"
    }
  ],
  "total": 1
}
```

**Example — keyword search:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/plugins/marketplace/search?query=steganography"
```

**Example — filter by category:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/plugins/marketplace/search?category=detection"
```

---

### POST /api/v1/plugins/marketplace/install

Install a plugin from the marketplace by name. The plugin is downloaded, SHA-256 verified, and loaded into the plugin directory.

**Headers:** `X-API-Key` (required)

**Request body (JSON):**

| Field  | Type   | Required | Description              |
|--------|--------|----------|--------------------------|
| `name` | string | Yes      | Plugin name to install   |

**Response (200):**

```json
{
  "status": "installed",
  "plugin": "steganography_detector"
}
```

**Response (400) — missing name:**

```json
{
  "error": "Bad Request",
  "message": "Missing 'name' field"
}
```

**Response (404) — plugin not found:**

```json
{
  "error": "Not Found",
  "message": "Plugin 'nonexistent' not found or install failed"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"name": "steganography_detector"}' \
     http://localhost:8666/api/v1/plugins/marketplace/install
```

---

### DELETE /api/v1/plugins/marketplace/{name}

Uninstall a plugin by name. Removes the plugin from the plugins directory.

**Headers:** `X-API-Key` (required)

**Path Parameters:**
- `name` (string, required) — The name of the plugin to uninstall.

**Response (200):**

```json
{
  "status": "uninstalled",
  "plugin": "steganography_detector"
}
```

**Response (404) — plugin not found:**

```json
{
  "error": "Not Found",
  "message": "Plugin 'nonexistent' not found"
}
```

**Example:**

```bash
curl -X DELETE \
     -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/plugins/marketplace/steganography_detector
```

---

### GET /api/v1/plugins/marketplace/updates

Check for available updates across all installed plugins.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "updates": [
    {
      "name": "steganography_detector",
      "current": "1.1.0",
      "latest": "1.2.0"
    }
  ],
  "total": 1
}
```

**Response (200) — no updates available:**

```json
{
  "updates": [],
  "total": 0
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/plugins/marketplace/updates
```

---

### POST /api/v1/plugins/marketplace/{name}/update

Update a specific installed plugin to its latest version.

**Headers:** `X-API-Key` (required)

**Path Parameters:**
- `name` (string, required) — The name of the plugin to update.

**Response (200):**

```json
{
  "status": "updated",
  "plugin": "steganography_detector"
}
```

**Response (404) — plugin not found or update failed:**

```json
{
  "error": "Not Found",
  "message": "Plugin 'nonexistent' not found or update failed"
}
```

**Example:**

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/plugins/marketplace/steganography_detector/update
```

---

## SIEM Integration

The SIEM endpoints allow exporting scan results in industry-standard log formats and managing forwarding configuration.

---

### POST /api/v1/export

Export a stored scan result in a SIEM-compatible format.

**Headers:** `X-API-Key` (required)

**Request body (JSON):**

| Field       | Type   | Required | Description                                              |
|-------------|--------|----------|----------------------------------------------------------|
| `scan_id`   | string | Yes      | UUID of a previously stored scan result                  |
| `format`    | string | Yes      | Output format: `cef`, `syslog`, `stix`, `leef`, or `ecs`|

**Response (200):**

```json
{
  "scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "format": "stix",
  "exported_at": "2026-02-17T14:30:00+00:00",
  "data": {
    "type": "bundle",
    "id": "bundle--a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "objects": [
      {
        "type": "identity",
        "id": "identity--hades-scanner",
        "name": "HADES Metadata Forensics Engine",
        "identity_class": "system"
      },
      {
        "type": "observed-data",
        "id": "observed-data--...",
        "first_observed": "2026-02-17T14:22:01Z",
        "last_observed": "2026-02-17T14:22:01Z",
        "number_observed": 1,
        "object_refs": ["file--..."]
      }
    ]
  }
}
```

For `cef`, `syslog`, and `leef` formats, the `data` field contains a string instead of a JSON object:

```json
{
  "scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "format": "cef",
  "exported_at": "2026-02-17T14:30:00+00:00",
  "data": "CEF:0|DarkHorse InfoSec|HADES|1.0.0|HADES-SCAN|Metadata scan result|7|fname=suspicious.jpg cn1=72 cn1Label=ThreatScore cs3=high cs3Label=ThreatLevel"
}
```

**Response (400) — missing fields:**

```json
{
  "error": "Bad Request",
  "message": "Both 'scan_id' and 'format' are required"
}
```

**Response (400) — invalid format:**

```json
{
  "error": "Bad Request",
  "message": "Invalid format 'xml'. Supported formats: cef, syslog, stix, leef, ecs"
}
```

**Response (404) — scan not found:**

```json
{
  "error": "Not Found",
  "message": "Scan ID 'invalid-id' not found"
}
```

**Example:**

```bash
curl -X POST http://localhost:8666/api/v1/export \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890", "format": "stix"}'
```

---

### GET /api/v1/siem/config

Retrieve the current SIEM forwarding configuration.

**Headers:** `X-API-Key` (required)

**Response (200):**

```json
{
  "enabled": true,
  "default_format": "cef",
  "targets": [
    {
      "type": "syslog",
      "host": "10.0.0.50",
      "port": 514,
      "protocol": "udp",
      "format": "cef"
    }
  ],
  "auto_forward_scans": true,
  "auto_forward_monitor_alerts": true,
  "min_severity_to_forward": "medium"
}
```

When no SIEM configuration exists, returns defaults:

```json
{
  "enabled": false,
  "default_format": "cef",
  "targets": [],
  "auto_forward_scans": false,
  "auto_forward_monitor_alerts": false,
  "min_severity_to_forward": "low"
}
```

**Example:**

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/siem/config
```

---

### PUT /api/v1/siem/config

Update the SIEM forwarding configuration. The new configuration replaces the existing one entirely.

**Headers:** `X-API-Key` (required)

**Request body (JSON):**

| Field                         | Type     | Required | Description                                              |
|-------------------------------|----------|----------|----------------------------------------------------------|
| `enabled`                     | boolean  | No       | Enable or disable SIEM forwarding (default: `false`)     |
| `default_format`              | string   | No       | Default output format: `cef`, `syslog`, `stix`, `leef`, `ecs` |
| `targets`                     | array    | No       | List of forwarding targets (see below)                   |
| `auto_forward_scans`          | boolean  | No       | Automatically forward scan results (default: `false`)    |
| `auto_forward_monitor_alerts` | boolean  | No       | Automatically forward monitor alerts (default: `false`)  |
| `min_severity_to_forward`     | string   | No       | Minimum severity: `low`, `medium`, `high`, `critical`    |

**Target object fields:**

| Field      | Type   | Required | Description                                     |
|------------|--------|----------|-------------------------------------------------|
| `type`     | string | Yes      | `syslog`, `file`, or `http`                     |
| `host`     | string | Syslog   | Syslog destination hostname or IP               |
| `port`     | integer| Syslog   | Syslog destination port                         |
| `protocol` | string | No       | `udp` (default) or `tcp` — syslog only          |
| `format`   | string | No       | Override format for this target                 |
| `path`     | string | File     | File path for file-type targets                 |
| `url`      | string | HTTP     | URL for HTTP-type targets                       |
| `headers`  | object | No       | Additional HTTP headers — http only             |

**Response (200):**

```json
{
  "message": "SIEM configuration updated",
  "config": {
    "enabled": true,
    "default_format": "cef",
    "targets": [
      {
        "type": "syslog",
        "host": "10.0.0.50",
        "port": 514,
        "protocol": "udp",
        "format": "cef"
      }
    ],
    "auto_forward_scans": true,
    "auto_forward_monitor_alerts": true,
    "min_severity_to_forward": "medium"
  }
}
```

**Response (400) — invalid target type:**

```json
{
  "error": "Bad Request",
  "message": "Invalid target type 'kafka'. Supported types: syslog, file, http"
}
```

**Example:**

```bash
curl -X PUT http://localhost:8666/api/v1/siem/config \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{
       "enabled": true,
       "default_format": "cef",
       "targets": [
         {
           "type": "syslog",
           "host": "10.0.0.50",
           "port": 514,
           "protocol": "udp",
           "format": "cef"
         }
       ],
       "auto_forward_scans": true,
       "auto_forward_monitor_alerts": true,
       "min_severity_to_forward": "medium"
     }'
```

---

## WebSocket

Real-time event streaming is available via WebSocket at `/ws/{client_id}`.

**Authentication:** Pass the API key as a query parameter: `?api_key=<your-key>`.

**Connection URL:** `ws://localhost:8666/ws/<client_id>?api_key=your-api-key-here`

**Subscriptions:** After connecting, send a JSON message to subscribe to event types:

```json
{
  "type": "subscribe",
  "subscription": "scan_results"
}
```

The server confirms each subscription:

```json
{
  "type": "subscription_confirmed",
  "subscription": "scan_results"
}
```

---

## Error Responses

All error responses follow a consistent JSON format:

```json
{
  "error": "<Error Type>",
  "message": "<Human-readable description>"
}
```

| Status Code | Error Type             | Common Causes                                    |
|-------------|------------------------|--------------------------------------------------|
| 400         | Bad Request            | Missing required `file` field, empty filename    |
| 401         | Unauthorized           | Missing or invalid `X-API-Key` header            |
| 403         | Forbidden              | Insufficient role permissions (RBAC)             |
| 404         | Not Found              | Invalid scan ID, unknown endpoint                |
| 409         | Conflict               | Duplicate username, monitor already running      |
| 413         | Payload Too Large      | File exceeds configured `max_file_size_mb` limit |
| 422         | Validation Error       | FastAPI request validation failure               |
| 429         | Rate Limit Exceeded    | Too many requests within the sliding window      |
| 500         | Internal Server Error  | Unhandled exception during processing            |
| 501         | Not Implemented        | Feature not configured (e.g., RBAC disabled)     |
| 503         | Service Unavailable    | Required module not installed (e.g., watchdog)   |

## Configuration

The API reads configuration from `cli/hades_config.json`. Key settings:

| Setting Path                     | Default | Description                              |
|----------------------------------|---------|------------------------------------------|
| `scanning.max_file_size_mb`      | 50      | Maximum upload file size in megabytes    |
| `api.api_keys`                   | `["your-api-key-here"]` | List of valid API keys    |
| `scanning.custom_yara_directory` | `rules/`| Directory containing YARA rule files     |
| `version`                        | `0.7.1` | Version string returned by health check  |
| `auth.rbac_enabled`              | `false` | Enable RBAC user management              |
| `auth.license_key`               | `null`  | Enterprise license key (or `HADES_LICENSE_KEY` env var) |
| `auth.sso.oidc`                  | `{}`    | OIDC provider configuration              |
| `auth.sso.saml`                  | `{}`    | SAML provider configuration              |
| `database`                       | `{}`    | Database backend configuration           |
| `encryption.enabled`             | `false` | Enable field-level encryption            |
| `redis`                          | `{}`    | Redis state manager configuration        |
| `tenants.enabled`                | `false` | Enable multi-tenant support              |
| `threat_intel_scheduler.feeds`   | `[]`    | Threat intelligence feed configurations  |

## Starting the Server

**Via the enhanced CLI (recommended):**

```bash
python core/hades_enhanced_cli.py --serve --port 8666
```

**Via the API module directly:**

```bash
python core/hades_api.py --host 127.0.0.1 --port 5000 --debug
```

**Programmatically:**

```python
from core.hades_api import create_app
import uvicorn

app = create_app(config_override={"api": {"api_keys": ["my-key"]}})
uvicorn.run(app, host="0.0.0.0", port=8666)
```

**Via Docker:**

```bash
docker run -p 8666:8666 hades-scanner:0.7.1
```

## Playbook Engine Endpoints

The playbook engine provides event-driven response automation. All endpoints require API key authentication.

### GET /api/v1/playbooks/

List all configured playbooks.

**Query Parameters:**
- `enabled_only` (bool, optional): If `true`, return only enabled playbooks.

**Response (200):**

```json
{
  "playbooks": [
    {
      "playbook_id": "builtin-malware-analysis",
      "name": "Malware Analysis Workflow",
      "description": "High-threat scan results with YARA matches trigger full analysis workflow.",
      "enabled": true,
      "trigger": {
        "event_type": "scan_result",
        "conditions": {"threat_score_gte": 8.0, "has_yara_match": true}
      },
      "action_count": 5,
      "tags": ["malware", "yara"],
      "version": "1.0.0"
    }
  ],
  "count": 5
}
```

### GET /api/v1/playbooks/{playbook_id}

Get a single playbook by ID, including full action definitions.

### POST /api/v1/playbooks/

Create a custom playbook.

**Request Body:**

```json
{
  "playbook_id": "custom-example",
  "name": "Custom Playbook",
  "description": "Custom response automation",
  "trigger": {
    "event_type": "scan_result",
    "conditions": {"threat_score_gte": 5.0}
  },
  "actions": [
    {"action_type": "create_case", "params": {"tags": ["custom"]}, "required": false},
    {"action_type": "audit_log", "params": {}}
  ],
  "tags": ["custom"]
}
```

### PATCH /api/v1/playbooks/{playbook_id}

Update a playbook's mutable fields (name, description, enabled, trigger, actions, tags).

### DELETE /api/v1/playbooks/{playbook_id}

Delete a playbook.

### POST /api/v1/playbooks/{playbook_id}/enable

Enable a playbook.

### POST /api/v1/playbooks/{playbook_id}/disable

Disable a playbook.

### POST /api/v1/playbooks/{playbook_id}/trigger

Manually trigger a playbook with a test event.

**Request Body:**

```json
{
  "event_type": "scan_result",
  "data": {"file_name": "test.jpg", "threat_score": 9.0}
}
```

**Response (200):**

```json
{
  "execution_id": "uuid",
  "playbook_id": "builtin-malware-analysis",
  "status": "completed",
  "action_results": [
    {"action_type": "create_case", "success": true, "message": "Case created: CASE-001"},
    {"action_type": "siem_forward", "success": false, "message": "SIEM manager not available"},
    {"action_type": "audit_log", "success": true, "message": "Audit entry recorded"}
  ]
}
```

### GET /api/v1/playbooks/executions/

List playbook execution history.

**Query Parameters:**
- `playbook_id` (str, optional): Filter by playbook ID.
- `limit` (int, optional): Max results (default 50, max 1000).

### GET /api/v1/playbooks/executions/{execution_id}

Get full details of a specific execution, including trigger event and action results.

---

### Built-in Playbooks

| ID | Trigger | Conditions | Actions |
|----|---------|------------|---------|
| `builtin-phishing-triage` | email_scan | threat_score >= 7.0 | create_case, siem_forward, slack_notify, audit_log |
| `builtin-bec-detection` | email_scan | threat_score >= 5.0, has_finding_type: impersonation | create_case, siem_forward, webhook, audit_log |
| `builtin-malware-analysis` | scan_result | threat_score >= 8.0, has_yara_match | create_case, siem_forward, evidence_export, slack_notify, audit_log |
| `builtin-credential-leak` | scan_result | has_finding_type: pii | create_case, siem_forward, webhook, audit_log |
| `builtin-suspicious-activity` | monitor_alert | threat_score >= 5.0 | create_case, siem_forward, webhook, audit_log |
| `builtin-dlp-exfiltration` | scan_result | has_finding_type: exfiltration | quarantine, create_case, siem_forward (CEF), webhook, audit_log |
| `builtin-dlp-pii-exposure` | scan_result | has_finding_type: pii, threat_score >= 3.0 | create_case, siem_forward (CEF), slack_notify, audit_log |
| `builtin-ransomware-response` | scan_result | has_finding_type: ransomware, threat_score >= 7.0 | quarantine (required), create_case, siem_forward (STIX), evidence_export, slack_notify, teams_notify, webhook, audit_log |
| `builtin-insider-threat` | monitor_alert | threat_score >= 4.0, has_finding_type: credential | create_case, siem_forward (CEF), audit_log, evidence_export |
| `builtin-campaign-escalation` | campaign_detected | (none) | create_case, siem_forward (STIX), slack_notify, teams_notify, evidence_export, audit_log |

---

## Scan Result Persistence

Scan results are stored in a SQLite database at `core/scan_results.db`. Each scan is assigned a UUID (`scan_id`) that can be used to retrieve results later via `GET /api/v1/scan/{scan_id}`. The database is created automatically on first run. When enterprise storage is enabled, results are additionally stored via the configured database backend (SQLite or PostgreSQL).

## Interactive API Documentation

FastAPI provides automatic interactive API documentation:

- **Swagger UI:** `http://localhost:8666/docs`
- **ReDoc:** `http://localhost:8666/redoc`

These endpoints are available without authentication and reflect all registered routes.
