# Enterprise Deployment Guide

> **NOTICE (2026-05-06):** This guide describes Enterprise features that remain in HADES, but its install instructions reference the pre-v1.4 distribution model (`pip install "hades-scanner[enterprise]"` etc.). Those commands no longer apply. HADES Enterprise tier is now activated by purchasing/upgrading your license key in the customer portal -- the same Nuitka binary that ships to all tiers becomes the Enterprise binary when the license key has Enterprise entitlements. RBAC, SSO, PostgreSQL backend, Redis caching, encryption at rest, multi-tenancy, and Kubernetes deployment manifests are all available; the application code and operator-facing configuration described below remain the right setup pattern. See [Installation Guide](installation.md) for the binary-install paths (Path A portal direct download or Path B Homebrew tap), then return here for the enterprise-specific configuration.

HADES v0.7.1 enterprise features: RBAC, SSO, PostgreSQL, Redis, encryption at rest, multi-tenancy, license management, Prometheus observability, and Kubernetes deployment. All enterprise features are optional and backward compatible with previous deployments.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [RBAC Setup](#rbac-setup)
4. [SSO Configuration](#sso-configuration)
5. [PostgreSQL Setup](#postgresql-setup)
6. [Redis Setup](#redis-setup)
7. [Encryption at Rest](#encryption-at-rest)
8. [Multi-Tenant Setup](#multi-tenant-setup)
9. [License Management](#license-management)
10. [Docker Compose Deployment](#docker-compose-deployment)
11. [Observability](#observability)
12. [Kubernetes Deployment](#kubernetes-deployment)
13. [Migration Guide](#migration-guide)
14. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Python 3.8+
- HADES v0.7.1
- (Optional) PostgreSQL 14+
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

Enterprise dependencies include: bcrypt, PyJWT, cryptography, psycopg2-binary, redis.

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
python core/hades_enhanced_cli.py --create-admin admin_username
```

You will be prompted for a password. The command outputs the API key.

### Create Users

```bash
# Create analyst
python core/hades_enhanced_cli.py --create-user analyst1 --user-role analyst

# Create viewer
python core/hades_enhanced_cli.py --create-user viewer1 --user-role viewer
```

### List Users

```bash
python core/hades_enhanced_cli.py --list-users
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

## SSO Configuration

### OIDC (OpenID Connect)

Configure in `cli/hades_config.json`:

```json
{
  "auth": {
    "sso": {
      "oidc": {
        "enabled": true,
        "issuer_url": "https://your-idp.example.com",
        "client_id": "hades-client-id",
        "client_secret": "your-client-secret",
        "redirect_uri": "https://hades.example.com/api/v1/auth/sso/oidc/callback"
      },
      "role_mapping": {
        "security-admins": "admin",
        "soc-analysts": "analyst",
        "default": "viewer"
      }
    }
  }
}
```

Supported OIDC providers: Okta, Azure AD, Keycloak, Auth0, Google Workspace.

### SAML 2.0

```json
{
  "auth": {
    "sso": {
      "saml": {
        "enabled": true,
        "idp_metadata_url": "https://idp.example.com/metadata",
        "sp_entity_id": "hades-scanner",
        "sp_acs_url": "https://hades.example.com/api/v1/auth/sso/saml/acs"
      }
    }
  }
}
```

SSO users are auto-provisioned (JIT) on first login with roles mapped from IdP groups.

---

## PostgreSQL Setup

### 1. Create Database

```sql
CREATE DATABASE hades;
CREATE USER hades WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE hades TO hades;
```

### 2. Configure

Edit `cli/hades_config.json`:

```json
{
  "database": {
    "backend": "postgres",
    "postgres": {
      "dsn": "postgresql://hades:your_secure_password@localhost:5432/hades",
      "min_connections": 2,
      "max_connections": 10
    }
  }
}
```

Or set environment variables:

```bash
export HADES_DB_BACKEND=postgres
export HADES_DB_DSN="postgresql://hades:your_secure_password@localhost:5432/hades"
```

### 3. Run Migration

```bash
python core/hades_enhanced_cli.py --migrate-db --db-backend postgres \
  --db-dsn "postgresql://hades:password@localhost:5432/hades"
```

### Environment Override Precedence

1. `HADES_DB_BACKEND` and `HADES_DB_DSN` environment variables
2. `database` section in `hades_config.json`
3. Default: SQLite at `core/scan_results.db`

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
python core/hades_enhanced_cli.py --generate-encryption-key
```

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
python core/hades_enhanced_cli.py --create-tenant "Acme Corp"
python core/hades_enhanced_cli.py --create-tenant "Beta Inc"
```

### List Tenants

```bash
python core/hades_enhanced_cli.py --list-tenants
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
python core/hades_enhanced_cli.py --license-info
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
python core/hades_enhanced_cli.py --license-key "your-key" --license-info
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
| SSO | - | - | Y |
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

### Enterprise Stack (PostgreSQL + Redis)

```bash
cd docker

# Set environment variables
export HADES_DB_BACKEND=postgres
export HADES_DB_DSN="postgresql://hades:hades_secure_password@hades-postgres:5432/hades"
export HADES_REDIS_URL="redis://hades-redis:6379/0"
export HADES_API_KEY="your-production-api-key"
export HADES_DB_PASSWORD="your-postgres-password"

# Start with enterprise profile
docker compose --profile enterprise up -d
```

### Scaled Deployment (PostgreSQL + Redis + Workers)

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

Includes PostgreSQL, Redis, API server, workers, Prometheus, Grafana, and Alertmanager. See [Observability](#observability) below.

### Production Checklist

1. Set `HADES_API_KEY` to a strong, unique key
2. Change default PostgreSQL password
3. Enable encryption: set `HADES_ENCRYPTION_KEY`
4. Configure resource limits for your workload
5. Set up monitoring (health endpoint: `/api/v1/health`, metrics: `/metrics`)
6. Enable RBAC and create admin user
7. Configure SSO if using corporate identity provider
8. Enable Prometheus metrics: set `HADES_METRICS_ENABLED=true` or use `--metrics` flag
9. Deploy Grafana dashboards for operational visibility (see [Observability](#observability))
10. For Kubernetes deployments, configure HPA and resource limits (see [Kubernetes Deployment](#kubernetes-deployment))

---

## Observability

HADES v0.7.1 includes built-in Prometheus metrics, Grafana dashboards, and alerting rules. Observability is opt-in and has zero overhead when disabled.

```bash
# Install observability support
pip install "hades-scanner[observability]"

# Enable metrics on the API server
hades-enhanced --serve --metrics

# Run a quick health probe
hades-enhanced --health-check
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

3. Run database migration if switching to PostgreSQL:

```bash
python core/hades_enhanced_cli.py --migrate-db --db-backend postgres \
  --db-dsn "postgresql://hades:password@localhost:5432/hades"
```

### Verify After Any Migration

```bash
# Check CLI works
python core/hades_enhanced_cli.py --help

# Check API starts
python core/hades_enhanced_cli.py --serve --port 8666

# Run tests
python -m pytest core/ cli/ config/ plugins/ tests/ demo/ grafana/ kubernetes/ -v --tb=short
```

### Breaking Changes

None across all versions. Each release is fully backward compatible with prior deployments. All new features are opt-in.

---

## Troubleshooting

### Enterprise auth module not available

Install enterprise dependencies:

```bash
pip install "hades-scanner[enterprise]"
```

### PostgreSQL connection failed

Verify DSN, ensure PostgreSQL is running, check network/firewall:

```bash
psql "postgresql://hades:password@localhost:5432/hades" -c "SELECT 1"
```

### Redis connection failed

HADES falls back to in-memory state automatically. To troubleshoot:

```bash
redis-cli -u redis://localhost:6379/0 PING
```

### Encryption key errors

Key must be exactly 32 bytes (256 bits). Generate with:

```bash
python core/hades_enhanced_cli.py --generate-encryption-key
```

### License validation failed

Check that the key is correctly set (no trailing whitespace). View current status:

```bash
python core/hades_enhanced_cli.py --license-info
```
