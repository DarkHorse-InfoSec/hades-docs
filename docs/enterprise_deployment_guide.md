# Enterprise Deployment Guide

> **NOTICE (2026-05-06):** This guide describes Enterprise features that remain in HADES, but its install instructions reference the pre-v1.4 distribution model (`pip install "hades-scanner[enterprise]"` etc.). Those commands no longer apply. HADES Enterprise tier is now activated by purchasing/upgrading your license key in the customer portal -- the same Nuitka binary that ships to all tiers becomes the Enterprise binary when the license key has Enterprise entitlements. RBAC, Redis caching, encryption at rest, multi-tenancy, and Kubernetes deployment manifests are all available (SSO and the PostgreSQL backend were removed; see [Breaking Changes](#breaking-changes)); the application code and operator-facing configuration described below remain the right setup pattern. See [Installation Guide](installation.md) for the binary-install paths (Path A portal direct download or Path B Homebrew tap), then return here for the enterprise-specific configuration.

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

- Python 3.8+
- HADES v0.7.1
- (Optional) Redis 7+
- (Optional) Docker and Docker Compose
- (Optional) Kubernetes 1.24+ with Helm 3 (for Kubernetes deployments)
- (Optional) prometheus_client (for metrics collection)

## Installation

Install with enterprise dependencies:

```bash
pip install "hades-scanner[enterprise]"
```

Or install all features:

```bash
pip install "hades-scanner[full]"
```

Enterprise dependencies include: bcrypt, PyJWT, cryptography, redis.

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

### Basic (SQLite)

```bash
cd docker
docker compose up -d
```

### Enterprise Stack (Redis)

```bash
cd docker

# Set environment variables
export HADES_REDIS_URL="redis://hades-redis:6379/0"
export HADES_API_KEY="your-production-api-key"

# Start with enterprise profile
docker compose --profile enterprise up -d
```

### Scaled Deployment (Redis + Workers)

```bash
docker compose -f docker/docker-compose.scale.yml up -d --scale hades-worker=4
```

### High-Availability Deployment (nginx + Multi-Instance)

```bash
docker compose -f docker/docker-compose.ha.yml up -d --scale hades-api=4
```

### Full Stack with Observability

```bash
docker compose -f docker/docker-compose.full-stack.yml up -d
```

Includes Redis, API server, workers, Prometheus, Grafana, and Alertmanager. See [Observability](#observability) below.

### Production Checklist

1. Set `HADES_API_KEY` to a strong, unique key
2. Enable encryption: set `HADES_ENCRYPTION_KEY`
3. Configure resource limits for your workload
4. Set up monitoring (health endpoint: `/api/v1/health`, metrics: `/metrics`)
5. Enable RBAC and create admin user
6. Enable Prometheus metrics: set `HADES_METRICS_ENABLED=true` or use `--metrics` flag
7. Deploy Grafana dashboards for operational visibility (see [Observability](#observability))
8. For Kubernetes deployments, configure HPA and resource limits (see [Kubernetes Deployment](#kubernetes-deployment))

---

## Observability

HADES v0.7.1 includes built-in Prometheus metrics, Grafana dashboards, and alerting rules. Observability is opt-in and has zero overhead when disabled.

```bash
# Install observability support
pip install "hades-scanner[observability]"

# Enable metrics on the API server
hades serve --metrics

# Run a quick health probe
hades serve --health-check
```

Key environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `HADES_METRICS_ENABLED` | `true` | Enable/disable Prometheus metrics |

For full setup details including Prometheus scrape configuration, Grafana dashboard walkthrough, PromQL query examples, and alerting configuration, see the **[Observability Guide](observability_guide.md)**.

---

## Kubernetes Deployment

HADES provides a Helm chart and Kustomize overlays for Kubernetes deployments with horizontal pod autoscaling, ingress, and ServiceMonitor integration.

```bash
# Install with Helm
helm install hades kubernetes/helm/hades \
  --set api.apiKey=your-secret-key \
  --set redis.enabled=true

# Kustomize (production overlay)
kubectl apply -k kubernetes/kustomize/overlays/production
```

For the full Kubernetes deployment guide including values reference, secrets management, HPA tuning, and troubleshooting, see the **[Kubernetes Guide](kubernetes_guide.md)**.

---

## Migration Guide

### From v0.7.0 to v0.7.1

1. Update the package:

```bash
pip install --upgrade "hades-scanner[full]"
```

2. (Optional) Install observability extras:

```bash
pip install "hades-scanner[observability]"
```

3. Add the `metrics` section to `cli/hades_config.json` (optional, defaults to enabled):

```json
{
  "metrics": {
    "enabled": true,
    "endpoint": "/metrics"
  }
}
```

4. No database migrations required. No breaking changes.

### From v0.6.0 to v0.7.x

1. Update the package:

```bash
pip install --upgrade "hades-scanner[full]"
```

2. Add the `distributed` config section to `cli/hades_config.json` for worker pool features (optional).

3. No database migrations required. All v0.7.x features are opt-in.

### From v0.5.0 to v0.6.0+

1. Update the package:

```bash
pip install --upgrade "hades-scanner[full]"
```

2. Add the enterprise config sections to `cli/hades_config.json` (database, redis, encryption, auth, tenants). All default to disabled.

### Verify After Any Migration

```bash
# Check CLI works
hades --help

# Check API starts
hades serve --port 8666

# Run tests
python -m pytest core/ cli/ config/ plugins/ tests/ demo/ grafana/ kubernetes/ -v --tb=short
```

### Breaking Changes

- **v1.7.4:** SSO (the OIDC and SAML endpoints and the `auth.sso` config section) was removed. Use RBAC users with per-user API keys.
- **v1.7.3:** PostgreSQL support was removed; SQLite is the only database backend. `--db-backend postgres` and `--db-dsn` are rejected (see [Database](#database)). The ML ensemble (`--ml-ensemble`, `GET /api/v1/ml/ensemble/status`) was removed; the signed XGBoost classifier is the ML stage.

---

## Troubleshooting

### Enterprise auth module not available

Install enterprise dependencies:

```bash
pip install "hades-scanner[enterprise]"
```

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
