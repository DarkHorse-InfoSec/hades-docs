# Docker Guide

This guide covers building, configuring, and running HADES in Docker containers for production deployments.

## Production Image

HADES provides a multi-stage Docker image based on `python:3.12-slim` with:

- Non-root user (UID 1000)
- YARA rules and web dashboard bundled
- Health checks built in
- Environment variable configuration
- OCI labels for container metadata

### Building the Image

```bash
docker build -f docker/hades-full.dockerfile -t hades-scanner:0.7.1 .
```

Or use the build script:

```bash
bash scripts/docker_build.sh
```

### Running Standalone

```bash
docker run -d \
  --name hades-api \
  -p 8666:8666 \
  -e HADES_API_KEY=your-secure-api-key \
  hades-scanner:0.7.1
```

Verify:

```bash
curl http://localhost:8666/api/v1/health
```

---

## Docker Compose Stack

### Basic Stack (SQLite)

```bash
docker compose -f docker/docker-compose.yml up -d
```

This starts the HADES API server and the secure file viewer.

### Enterprise Stack (PostgreSQL + Redis)

```bash
docker compose -f docker/docker-compose.yml --profile enterprise up -d
```

The enterprise profile adds:

- PostgreSQL 16 for persistent storage
- Redis 7 for rate limiting, caching, and pub/sub

---

## Environment Variable Reference

| Variable | Description | Default |
|---|---|---|
| `HADES_API_KEY` | API authentication key | `your-api-key-here` |
| `HADES_PORT` | API server port | `8666` |
| `HADES_WORKERS` | Uvicorn worker count | `1` |
| `HADES_LOG_LEVEL` | Logging level (debug, info, warning, error) | `info` |
| `HADES_DB_BACKEND` | Database backend (`sqlite` or `postgres`) | `sqlite` |
| `HADES_DB_DSN` | PostgreSQL connection string | None |
| `HADES_REDIS_URL` | Redis connection URL | None |
| `HADES_ENCRYPTION_KEY` | AES-256-GCM encryption key (base64) | None |
| `HADES_LICENSE_KEY` | License key for tier features | None |
| `VIRUSTOTAL_API_KEY` | VirusTotal threat intel API key | None |
| `ABUSEIPDB_API_KEY` | AbuseIPDB threat intel API key | None |
| `OTX_API_KEY` | OTX AlienVault threat intel API key | None |

!!! danger "Replace default API key"
    Never use `your-api-key-here` in production. Always set `HADES_API_KEY` to a strong, unique value.

---

## Volume Mounts

| Host Path | Container Path | Purpose |
|---|---|---|
| `./config/` | `/app/config/` | Configuration files |
| `./logs/` | `/app/logs/` | Application and alert logs |
| `./scan-data/` | `/app/scan-data/` | Files to scan or monitor |
| `./models/` | `/app/models/` | ML model files |
| `./rules/` | `/app/rules/` | Custom YARA rules (overlay) |

### Custom YARA Rules

Mount a directory of custom rules to overlay the built-in rules:

```bash
docker run -d \
  -v /path/to/custom/rules:/app/rules:ro \
  -p 8666:8666 \
  hades-scanner:0.7.1
```

### Persistent ML Models

Mount the models directory to persist trained ML models across container restarts:

```bash
docker run -d \
  -v hades-models:/app/models \
  -p 8666:8666 \
  hades-scanner:0.7.1
```

---

## Health Checks

The Docker image includes a built-in health check:

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:8666/api/v1/health || exit 1
```

Check container health:

```bash
docker inspect --format='{{.State.Health.Status}}' hades-api
```

---

## Scaling

### Workers

Increase API server workers for higher throughput:

```bash
docker run -d \
  -e HADES_WORKERS=4 \
  -p 8666:8666 \
  hades-scanner:0.7.1
```

!!! note "Worker considerations"
    Each worker runs an independent uvicorn process. With SQLite, each worker has its own database connection. For shared state (scan results, monitoring), use PostgreSQL with the enterprise stack.

### Load Balancing

Run multiple containers behind a reverse proxy:

```
                  +--> hades-api-1 (container)
Load Balancer --> +--> hades-api-2 (container)
                  +--> hades-api-3 (container)
```

Requirements:

- Use PostgreSQL for shared scan result storage
- Use Redis for distributed rate limiting and session state
- Run the file monitor on only one instance per watched directory
- Train ML models centrally and distribute to all instances

### Resource Limits

Recommended container resource limits:

| Configuration | CPU | Memory |
|---|---|---|
| Scanning only | 1 core | 1 GB |
| Scanning + ML | 2 cores | 2 GB |
| Scanning + monitoring | 2 cores | 2 GB |
| Full stack (all features) | 4 cores | 4 GB |

Set limits in Docker Compose:

```yaml
services:
  hades-api:
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 4G
        reservations:
          cpus: '2'
          memory: 2G
```

---

## Security

### Non-Root User

The Dockerfile creates a non-root `hades` user. The application runs as this user by default. Do not override with `--user root`.

### Read-Only Filesystem

For additional security, run with a read-only root filesystem:

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  -v ./logs:/app/logs \
  -v ./scan-data:/app/scan-data \
  -v ./config:/app/config:ro \
  hades-scanner:0.7.1
```

### Network Policies

In production, restrict outbound network access:

- Allow HTTPS (443) to threat intelligence APIs:
    - `www.virustotal.com`
    - `api.abuseipdb.com`
    - `otx.alienvault.com`
    - `mb-api.abuse.ch`
- Allow outbound to your SIEM target
- Block all other outbound traffic

### Secret Management

Use Docker secrets for sensitive values:

```bash
echo "your-secure-api-key" | docker secret create hades_api_key -
echo "your-encryption-key" | docker secret create hades_encryption_key -
```

Reference secrets in Docker Compose:

```yaml
services:
  hades-api:
    secrets:
      - hades_api_key
      - hades_encryption_key

secrets:
  hades_api_key:
    external: true
  hades_encryption_key:
    external: true
```

---

## Secure Viewer

HADES includes a separate Docker image for safely viewing potentially malicious media files in an isolated container:

```bash
docker build -f docker/secureviewer.dockerfile -t hades-viewer .
```

The secure viewer runs an Express.js server that renders files in a sandboxed environment with no access to the host filesystem.

---

## Troubleshooting

### Container Fails to Start

```bash
docker compose -f docker/docker-compose.yml logs hades-api
```

Common causes:

- Port 8666 already in use
- Missing or invalid configuration file
- Permission denied on volume mounts (ensure the `hades` user has write access)

### Health Check Fails

```bash
curl -v http://localhost:8666/api/v1/health
```

If an engine reports `degraded`:

- Check that YARA rules are present in the rules directory
- Verify Python dependencies are installed (`docker exec hades-api pip list`)

### Database Connection Issues

For PostgreSQL:

```bash
docker exec hades-api python -c "
import psycopg2
psycopg2.connect('postgresql://hades:password@hades-postgres:5432/hades')
print('Connection successful')
"
```

---

## Next Steps

- [Deployment Guide](deployment_guide.md) -- General deployment guidance
- [Enterprise Deployment](enterprise_deployment_guide.md) -- RBAC, SSO, PostgreSQL, Redis
- [CLI Reference](cli_reference.md) -- All CLI flags and options
