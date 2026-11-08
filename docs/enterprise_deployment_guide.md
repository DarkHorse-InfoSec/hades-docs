# Enterprise Deployment Guide

> **Updated 2026-11-08:** HADES Enterprise is the same `hades` and `hades-server` binaries as every tier; the Enterprise features are unlocked by a license key with Enterprise entitlements, bought or upgraded in the customer portal. There is no pip package and no container image: install with [Path A or Path B](installation.md), then return here for the enterprise configuration. Containerized and distributed deployments (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap and are not shipped; see the [Kubernetes Guide](kubernetes_guide.md). SSO and the PostgreSQL backend were removed; see [Breaking Changes](#breaking-changes). Run `hades doctor` to see which optional components (for example `bcrypt`, `cryptography`, `redis`, `prometheus_client`) are present in your build.

HADES enterprise features: RBAC, Redis, encryption at rest, multi-tenancy, license management, Prometheus observability, and Kubernetes deployment. All enterprise features are optional and backward compatible with previous deployments.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [RBAC Setup](#rbac-setup)
4. [Database](#database)
5. [Redis Setup](#redis-setup)
6. [Encryption at Rest](#encryption-at-rest)
7. [Multi-Tenant Setup](#multi-tenant-setup)
8. [License Management](#license-management)
9. [Docker Compose Deployment](#docker-compose-deployment)
10. [Observability](#observability)
11. [Kubernetes Deployment](#kubernetes-deployment)
12. [Migration Guide](#migration-guide)
13. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- The `hades` and `hades-server` binaries ([Installation](installation.md))
- A license key with Enterprise entitlements
- (Optional) Redis 7+

## Installation

HADES ships as two self-contained binaries, `hades` and `hades-server`; there are no pip packages or extras to install. Install them as described in [Installation](installation.md), then check which optional components your build includes:

```bash
hades doctor
```

The enterprise features use bcrypt, PyJWT, cryptography and redis; `hades doctor` reports bcrypt, cryptography and redis.

---

## RBAC Setup

Role-Based Access Control manages users, API keys, and permissions.

### Roles and Permissions

| Permission | Admin | Analyst | Viewer |
|---|:---:|:---:|:---:|
| scan, sanitize | Y | Y | - |
| scan_read, case_read | Y | Y | Y |
| case_create, case_update | Y | Y | - |
| case_delete | Y | - | - |
| user_manage, config_manage | Y | - | - |
| monitor_manage, plugin_manage | Y | - | - |
| evidence_read, evidence_export | Y | Y | Y |
| audit_read, health | Y | Y | Y |

### Create Admin User

```bash
hades admin create-admin admin_username
```

> **Known issue in v1.7.4:** in the compiled `hades` binary, `hades admin create-admin`, `create-user`, `list-users`, `create-tenant` and `list-tenants` exit with an internal error (`NameError`) instead of running. The fix is tracked for the next release. Until then these commands cannot be used from the shipped binary.

You will be prompted for a password. The command outputs the API key.

### Create Users

```bash
# Create analyst
hades admin create-user analyst1 --role analyst

# Create viewer
hades admin create-user viewer1 --role viewer
```

### List Users

```bash
hades admin list-users
```

### Enable RBAC in Config

Edit `cli/hades_config.json`:

```json
{
  "auth": {
    "rbac_enabled": true
  }
}
```

When RBAC is enabled, API keys are validated against the user database instead of the flat `api_keys` list. Legacy flat API keys continue to work when RBAC is disabled.

### API Key Rotation

Via the REST API (admin only):

```bash
curl -X POST http://localhost:8666/api/v1/auth/users/{user_id}/rotate-key \
  -H "X-API-Key: admin-api-key"
```

---

## Database

HADES stores scan results, audit logs and cases in SQLite (WAL mode, a single file, no setup). SQLite is the only database backend: PostgreSQL support was removed in v1.7.3. `--db-backend postgres` and `--db-dsn` are command-line errors, and a leftover `HADES_DB_BACKEND=postgres` or `HADES_DB_DSN` makes `hades serve --health-check` report unhealthy and `hades admin migrate` refuse to run.

---

## Redis Setup

Redis provides distributed rate limiting, session management, pub/sub, and caching. Falls back to in-memory when unavailable.

### Configure

```json
{
  "redis": {
    "enabled": true,
    "url": "redis://localhost:6379/0",
    "password": null,
    "ssl": false
  }
}
```

Or via environment:

```bash
export HADES_REDIS_URL="redis://localhost:6379/0"
```

### Features Backed by Redis

- Sliding-window rate limiting (distributed across workers)
- Session storage with TTL
- Pub/Sub for real-time event distribution
- Distributed locks for coordinated operations
- Result caching

---

## Encryption at Rest

AES-256-GCM encryption for sensitive database fields.

### 1. Generate Key

```bash
hades serve --generate-encryption-key
```

> **Known issue in v1.7.4:** in the compiled binary, `hades serve --generate-encryption-key` exits with "Enterprise storage module not available": the storage module is not included in the v1.7.4 build. The fix is tracked for the next release.

This outputs a base64-encoded key. Store securely.

### 2. Set Key

Via environment variable:

```bash
export HADES_ENCRYPTION_KEY="base64-encoded-key-here"
```

### 3. Enable in Config

```json
{
  "encryption": {
    "enabled": true,
    "encrypted_fields": {
      "scans": ["results_json"],
      "audit_log": ["details"],
      "cases": ["notes"]
    }
  }
}
```

### What Gets Encrypted

- Scan result JSON (findings, threat details)
- Audit log details
- Case investigation notes
- Any custom fields configured in `encrypted_fields`

Encryption is transparent: queries work normally; data is encrypted on write and decrypted on read.

---

## Multi-Tenant Setup

Isolate data across organizational boundaries using column-based tenant isolation.

### Enable Tenants

```json
{
  "tenants": {
    "enabled": true,
    "isolation_mode": "column"
  }
}
```

### Create Tenants

```bash
hades admin create-tenant 'Acme Corp'
hades admin create-tenant 'Beta Inc'
```

> **Known issue in v1.7.4:** in the compiled `hades` binary, `hades admin create-admin`, `create-user`, `list-users`, `create-tenant` and `list-tenants` exit with an internal error (`NameError`) instead of running. The fix is tracked for the next release. Until then these commands cannot be used from the shipped binary.

### List Tenants

```bash
hades admin list-tenants
```

### Assign Users to Tenants

Via the API (admin only):

```bash
curl -X PUT http://localhost:8666/api/v1/auth/users/{user_id} \
  -H "X-API-Key: admin-key" \
  -H "Content-Type: application/json" \
  -d '{"tenant_id": "tenant-uuid"}'
```

### Tenant Administration

Via the API:

```bash
# List tenants
curl http://localhost:8666/api/v1/admin/tenants

# Create tenant
curl -X POST http://localhost:8666/api/v1/admin/tenants \
  -H "Content-Type: application/json" \
  -d '{"name": "New Tenant", "config": {"plan": "enterprise"}}'
```

---

## License Management

### View License Info

```bash
hades admin license
```

### Set License Key

Via environment variable:

```bash
export HADES_LICENSE_KEY="your-license-key-here"
```

Via config:

```json
{
  "auth": {
    "license_key": "your-license-key-here"
  }
}
```

Via CLI flag:

```bash
hades admin license --key "your-key"

# Show the active licence
hades admin license
```

### Tiers

| Feature | Free | Professional | Enterprise |
|---|:---:|:---:|:---:|
| CLI scanning | Y | Y | Y |
| Basic API | Y | Y | Y |
| YARA rules | Y | Y | Y |
| SIEM export | - | Y | Y |
| Threat intel | - | Y | Y |
| ML detection | - | Y | Y |
| Evidence chain | - | Y | Y |
| Monitoring | - | Y | Y |
| Plugins | - | Y | Y |
| RBAC | - | - | Y |
| Cloud scanning | - | - | Y |
| CI/CD integration | - | - | Y |
| Chat bots | - | - | Y |
| Multi-tenant | - | - | Y |
| Encrypted storage | - | - | Y |

---

## Docker Compose Deployment

Containerized and distributed deployments (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap and are not shipped; see the [Kubernetes Guide](kubernetes_guide.md). The compose files, profiles and scaled or high-availability stacks described in earlier versions of this guide are not distributed with the binaries. A current deployment is one `hades-server` per host:

```bash
export HADES_API_KEY="your-production-api-key"
hades-server --host 0.0.0.0 --port 8666
```

### Production Checklist

1. Set `HADES_API_KEY` to a strong, unique key
2. Enable encryption: set `HADES_ENCRYPTION_KEY`
3. Configure resource limits for your workload
4. Set up monitoring (health endpoint: `/api/v1/health`, metrics: `/metrics`)
5. Enable RBAC and create admin user
6. Confirm Prometheus metrics are available: `hades doctor` lists `prometheus_client`, and `curl http://localhost:8666/metrics` returns metrics
7. Point your own Prometheus (and Grafana, if you use it) at `/metrics` (see [Observability](#observability))

---

## Observability

The API server includes a Prometheus metrics exporter. Metrics are on by default and switch off automatically when the `prometheus_client` component is not present in your build.

```bash
# Check that prometheus_client is present in your build
hades doctor

# Scrape endpoint
curl http://localhost:8666/metrics

# Run a quick health probe
hades serve --health-check
```

For full setup details including Prometheus scrape configuration, Grafana dashboard walkthrough, PromQL query examples, and alerting configuration, see the **[Observability Guide](observability_guide.md)**.

---

## Kubernetes Deployment

Kubernetes deployment (Helm chart, Kustomize overlays) is on the Enterprise roadmap and is not shipped. The [Kubernetes Guide](kubernetes_guide.md) describes the planned architecture.

---

## Migration Guide

Upgrade by installing the new release the same way you installed it; see [Upgrading](installation.md#upgrading). Then run `hades admin migrate` if the release notes mention schema changes. The pip-based migration steps for v0.5.0 to v0.7.1 described in earlier versions of this guide do not apply to the binaries.

### Verify After Any Migration

```bash
# Check the new version and its components
hades --version
hades doctor

# Check the API starts and answers
hades serve --port 8666
curl http://localhost:8666/api/v1/health
```

### Breaking Changes

- **v1.7.4:** SSO (the OIDC and SAML endpoints and the `auth.sso` config section) was removed. Use RBAC users with per-user API keys.
- **v1.7.3:** PostgreSQL support was removed; SQLite is the only database backend. `--db-backend postgres` and `--db-dsn` are rejected (see [Database](#database)). The ML ensemble (`--ml-ensemble`, `GET /api/v1/ml/ensemble/status`) was removed; the signed XGBoost classifier is the ML stage.

---

## Troubleshooting

### Enterprise auth module not available

Run `hades doctor`. If it reports `bcrypt` or `cryptography` as missing, your build does not include them and the auth module cannot load; contact support@darkhorseinfosec.com.

### Redis connection failed

HADES falls back to in-memory state automatically. To troubleshoot:

```bash
redis-cli -u redis://localhost:6379/0 PING
```

### Encryption key errors

Key must be exactly 32 bytes (256 bits). Generate with:

```bash
hades serve --generate-encryption-key
```

> **Known issue in v1.7.4:** in the compiled binary, `hades serve --generate-encryption-key` exits with "Enterprise storage module not available": the storage module is not included in the v1.7.4 build. The fix is tracked for the next release.

### License validation failed

Check that the key is correctly set (no trailing whitespace). View current status:

```bash
hades admin license
```
