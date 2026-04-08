# HADES Production Deployment Checklist
Version: 1.0.0

DarkHorse Information Security LLC

---

This checklist covers the critical steps required to deploy HADES in a production environment. Each section contains actionable steps with example commands. Complete every section before exposing the service to production traffic.

---

## 1. TLS Setup

HADES does not terminate TLS natively. Use nginx as a reverse proxy in front of the API server to handle HTTPS termination.

### 1.1 Obtain a TLS Certificate

Use a trusted CA or Let's Encrypt. For Let's Encrypt with certbot:

```bash
sudo apt install certbot
sudo certbot certonly --standalone -d hades.example.com
```

Certificate files will be at:
- `/etc/letsencrypt/live/hades.example.com/fullchain.pem`
- `/etc/letsencrypt/live/hades.example.com/privkey.pem`

### 1.2 Nginx HTTPS Configuration

Create `/etc/nginx/conf.d/hades-tls.conf`:

```nginx
upstream hades_api {
    least_conn;
    server 127.0.0.1:8666;
    keepalive 32;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name hades.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name hades.example.com;

    # -- TLS Certificates -------------------------------------------------
    ssl_certificate     /etc/letsencrypt/live/hades.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hades.example.com/privkey.pem;

    # -- TLS Hardening ----------------------------------------------------
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # -- Security Headers -------------------------------------------------
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;

    # -- File Upload Limit ------------------------------------------------
    client_max_body_size 100m;

    # -- Rate Limiting ----------------------------------------------------
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;
    limit_req_zone $binary_remote_addr zone=scan_limit:10m rate=30r/m;

    # -- Health Check (no rate limit) -------------------------------------
    location = /api/v1/health {
        proxy_pass http://hades_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # -- WebSocket --------------------------------------------------------
    location = /ws {
        proxy_pass http://hades_api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }

    # -- Scan Endpoints (stricter rate limit) -----------------------------
    location /api/v1/scan/ {
        limit_req zone=scan_limit burst=10 nodelay;
        limit_req_status 429;
        proxy_pass http://hades_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }

    # -- Dashboard Static Assets ------------------------------------------
    location /dashboard/ {
        proxy_pass http://hades_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_valid 200 10m;
        expires 10m;
        add_header Cache-Control "public, max-age=600";
    }

    # -- Prometheus Metrics (restrict to internal) ------------------------
    location = /metrics {
        allow 10.0.0.0/8;
        allow 172.16.0.0/12;
        allow 192.168.0.0/16;
        deny all;
        proxy_pass http://hades_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # -- All Other Endpoints ----------------------------------------------
    location / {
        limit_req zone=api_limit burst=20 nodelay;
        limit_req_status 429;
        proxy_pass http://hades_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }

    # -- Error Pages -------------------------------------------------------
    error_page 429 /429.json;
    location = /429.json {
        default_type application/json;
        return 429 '{"error": "Rate limit exceeded. Try again later."}';
    }

    error_page 502 503 504 /50x.json;
    location = /50x.json {
        default_type application/json;
        return 503 '{"error": "Service temporarily unavailable."}';
    }
}
```

### 1.3 Validate and Reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 1.4 Auto-Renewal (Let's Encrypt)

```bash
# Test renewal
sudo certbot renew --dry-run

# Cron entry for automatic renewal
echo "0 3 * * * root certbot renew --quiet --deploy-hook 'systemctl reload nginx'" \
  | sudo tee /etc/cron.d/certbot-renew
```

---

## 2. API Key Management

HADES authenticates API requests via the `X-API-Key` header. The RBAC module in `core/auth/rbac.py` provides user management and API key rotation.

### 2.1 Replace the Default Key

The default development key (`your-api-key-here`) must be replaced before production use.

**Option A: Environment variable (recommended for containers)**

```bash
export HADES_API_KEY=$(python3 -c "import secrets; print(secrets.token_urlsafe(48))")
echo "Generated key: $HADES_API_KEY"

# Pass to Docker
docker run -e HADES_API_KEY="$HADES_API_KEY" hades-scanner:latest
```

**Option B: Config file (multi-key support)**

Edit `config/cli/hades_config.json`:

```json
{
  "api": {
    "api_keys": [
      "primary-production-key-here",
      "secondary-rotation-key-here"
    ]
  }
}
```

Multiple keys are supported simultaneously, enabling zero-downtime rotation.

### 2.2 Create an Admin User (RBAC)

```bash
python core/hades_enhanced_cli.py --create-admin
```

This prompts for username and password. The admin account can then manage other users.

### 2.3 Create Analyst and Viewer Accounts

```bash
python core/hades_enhanced_cli.py --create-user analyst1 --user-role analyst
python core/hades_enhanced_cli.py --create-user viewer1 --user-role viewer
```

Role permissions:
- **admin**: Full access (user management, config, scans, cases, system)
- **analyst**: Scan, create/update cases, export evidence, threat intel, ML detection
- **viewer**: Read-only access to scans, cases, evidence, audit logs

### 2.4 Rotate API Keys

```bash
# Via the API (requires admin role)
curl -X POST https://hades.example.com/api/v1/auth/api-key/rotate \
  -H "X-API-Key: current-admin-key" \
  -H "Content-Type: application/json" \
  -d '{"username": "analyst1"}'
```

### 2.5 Key Storage Best Practices

- Store API keys in a secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) rather than in plaintext config files.
- For Docker deployments, use Docker secrets:
  ```bash
  echo "your-production-key" | docker secret create hades_api_key -
  ```
- For Kubernetes deployments, use the `hades-secrets` Secret referenced in the Helm chart `values.yaml`:
  ```bash
  kubectl create secret generic hades-secrets \
    --from-literal=api-key="$(python3 -c 'import secrets; print(secrets.token_urlsafe(48))')" \
    --from-literal=encryption-key="$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')"
  ```
- Rotate keys on a regular schedule (every 90 days minimum).
- Never commit keys to version control. Ensure `.env` files and `*.key` files are listed in `.gitignore`.
- Use separate keys for each environment (development, staging, production).

---

## 3. Backup Strategy

### 3.1 SQLite (WAL Mode) Backup

HADES uses SQLite in WAL (Write-Ahead Logging) mode by default. WAL allows concurrent reads during backup, but requires a specific approach for consistent snapshots.

**Hot backup using sqlite3 `.backup` command:**

```bash
# Backup the main scan database
sqlite3 /path/to/hades_scan.db ".backup '/backups/hades_scan_$(date +%Y%m%d_%H%M%S).db'"

# Backup the evidence chain database
sqlite3 /path/to/evidence.db ".backup '/backups/evidence_$(date +%Y%m%d_%H%M%S).db'"

# Backup the playbook database
sqlite3 /path/to/playbooks.db ".backup '/backups/playbooks_$(date +%Y%m%d_%H%M%S).db'"
```

The `.backup` command handles WAL checkpointing automatically and produces a consistent snapshot even while the application is running.

**Do NOT simply copy the `.db` file.** Copying a WAL-mode SQLite file requires also copying the `-wal` and `-shm` files atomically, which is error-prone. Always use the `.backup` command.

**Automated backup script (`/opt/hades/backup_sqlite.sh`):**

```bash
#!/usr/bin/env bash
set -euo pipefail
BACKUP_DIR="/backups/hades/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

for db in /opt/hades/data/*.db; do
    dbname=$(basename "$db" .db)
    sqlite3 "$db" ".backup '${BACKUP_DIR}/${dbname}_$(date +%H%M%S).db'"
done

# Retain 30 days of backups
find /backups/hades/ -type d -mtime +30 -exec rm -rf {} + 2>/dev/null || true
echo "[$(date)] SQLite backup complete: $BACKUP_DIR"
```

Schedule via cron:

```bash
echo "0 2 * * * root /opt/hades/backup_sqlite.sh >> /var/log/hades-backup.log 2>&1" \
  | sudo tee /etc/cron.d/hades-backup
```

### 3.2 PostgreSQL Backup

When using the PostgreSQL backend (`docker-compose.full-stack.yml`), use `pg_dump` for logical backups.

**Single database dump:**

```bash
pg_dump -h localhost -U hades -d hades -Fc \
  -f "/backups/hades_pg_$(date +%Y%m%d_%H%M%S).dump"
```

**Restore from dump:**

```bash
pg_restore -h localhost -U hades -d hades --clean --if-exists \
  "/backups/hades_pg_20260301_020000.dump"
```

**Docker Compose backup (containerized PostgreSQL):**

```bash
docker compose -f docker/docker-compose.full-stack.yml exec postgres \
  pg_dump -U hades -Fc hades > "/backups/hades_pg_$(date +%Y%m%d_%H%M%S).dump"
```

**Automated PostgreSQL backup script:**

```bash
#!/usr/bin/env bash
set -euo pipefail
BACKUP_DIR="/backups/hades-pg"
mkdir -p "$BACKUP_DIR"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

docker compose -f /opt/hades/docker/docker-compose.full-stack.yml exec -T postgres \
  pg_dump -U hades -Fc hades > "${BACKUP_DIR}/hades_${TIMESTAMP}.dump"

# Retain 30 days
find "$BACKUP_DIR" -name "*.dump" -mtime +30 -delete
echo "[$(date)] PostgreSQL backup complete: ${BACKUP_DIR}/hades_${TIMESTAMP}.dump"
```

### 3.3 Redis State

Redis stores ephemeral state (rate limiting, sessions, cache, distributed locks). It is not a primary data store in HADES. However, if persistence of Redis data is desired:

- The `docker-compose.full-stack.yml` already enables `appendonly yes` with a data volume (`redis_data`).
- For manual backup: `docker compose exec redis redis-cli BGSAVE`
- The AOF (Append Only File) at `/data/appendonly.aof` inside the container provides crash recovery.

### 3.4 Supplementary Backups

Also back up these artifacts:
- `config/cli/hades_config.json` (application configuration)
- `rules/*.yar` and `rules/*.yara` (custom YARA rules)
- `models/*.joblib` (trained ML models)
- `config/encryption.key` (encryption key -- store separately, encrypted)

---

## 4. Monitoring

HADES exposes Prometheus metrics at `/metrics` and provides pre-built Grafana dashboards.

### 4.1 Deploy the Observability Stack

The project ships a ready-to-use Docker Compose file with Prometheus v2.50, Grafana 10.3, and Alertmanager v0.27:

```bash
# Standalone observability (assumes HADES API is already running)
docker compose -f docker/docker-compose.observability.yml up -d

# Full stack with PostgreSQL + Redis + API + Workers + Observability
docker compose -f docker/docker-compose.full-stack.yml up -d
```

Services and ports:
| Service      | Port | Purpose                              |
|-------------|------|--------------------------------------|
| Prometheus  | 9090 | Metric scraping, alerting rules      |
| Grafana     | 3000 | Dashboards and visualization         |
| Alertmanager| 9093 | Alert routing and notification       |

### 4.2 Enable Metrics on the HADES API

Metrics collection requires the `prometheus_client` package:

```bash
pip install "hades-scanner[observability]"
```

Start the API with metrics enabled:

```bash
python core/hades_enhanced_cli.py --serve --port 8666 --metrics
```

Verify metrics are exposed:

```bash
curl -s http://localhost:8666/metrics | head -20
```

### 4.3 Prometheus Configuration

The scrape configuration at `docker/prometheus/prometheus.yml` targets the HADES API:

```yaml
scrape_configs:
  - job_name: "hades-api"
    static_configs:
      - targets: ["hades-api:8666"]
    metrics_path: /metrics
    scrape_interval: 10s
```

If running multiple API instances behind a load balancer, add each instance as a separate target or use service discovery.

### 4.4 Pre-Built Alert Rules

Seven alerting rules ship in `docker/prometheus/alerts.yml`:

| Alert                  | Condition                        | Severity |
|-----------------------|----------------------------------|----------|
| HADESHighErrorRate    | Scan error rate > 5% for 5 min   | warning  |
| HADESQueueBacklog     | Queue depth > 100 for 10 min     | warning  |
| HADESNoWorkers        | 0 active workers for 2 min       | critical |
| HADESHighLatency      | p99 scan duration > 30s for 5 min| warning  |
| HADESAPIErrors        | HTTP 5xx rate > 10% for 5 min    | critical |
| HADESCacheDegraded    | Cache hit rate < 50% for 15 min  | info     |
| HADESDown             | Service unreachable for 1 min    | critical |

Configure notification targets in `docker/alertmanager/alertmanager.yml` (email, Slack, PagerDuty, etc.).

### 4.5 Grafana Dashboards

Four dashboards are auto-provisioned via `grafana/provisioning/`:

1. **HADES Overview** (`hades-overview.json`) -- Scan throughput, latency percentiles, threat distribution, error rate, cache hit rate
2. **Pipeline** (`hades-pipeline.json`) -- Per-stage duration, ExifTool/YARA/ML timing, bottleneck analysis
3. **Workers** (`hades-workers.json`) -- Active workers, queue depth, task rates, failure rate
4. **Infrastructure** (`hades-infrastructure.json`) -- API request/response metrics, cache, auth attempts

Default Grafana credentials: `admin` / value of `GRAFANA_PASSWORD` env var (default: `admin`). Change immediately on first login.

### 4.6 Health Check Probes

HADES provides a health endpoint and a CLI probe for container orchestrators:

```bash
# HTTP health check
curl http://localhost:8666/api/v1/health

# CLI probe (exits 0 on healthy, 1 on failure -- suitable for Kubernetes liveness/readiness)
python core/hades_enhanced_cli.py --health-check
```

### 4.7 JSON Metrics Endpoint

For custom integrations that do not use Prometheus scraping:

```bash
curl -H "X-API-Key: your-key" http://localhost:8666/api/v1/metrics/json
curl -H "X-API-Key: your-key" http://localhost:8666/api/v1/metrics/health
curl -H "X-API-Key: your-key" http://localhost:8666/api/v1/metrics/alerts
```

---

## 5. Resource Sizing

### 5.1 Deployment Profiles

| Profile      | API Replicas | Workers | CPU (total)   | RAM (total)    | Use Case                                  |
|-------------|-------------|---------|---------------|----------------|---------------------------------------------|
| **Small**   | 1           | 0       | 2 cores       | 2 GB           | Single analyst, < 1000 scans/day            |
| **Medium**  | 2           | 2       | 8 cores       | 8 GB           | Team of 5-10 analysts, < 10,000 scans/day   |
| **Large**   | 4+          | 4-16    | 16+ cores     | 32+ GB         | Enterprise SOC, 10,000+ scans/day            |

### 5.2 Per-Component Resource Limits

These match the limits defined in `docker-compose.full-stack.yml` and the Helm chart `values.yaml`:

| Component         | CPU Request | CPU Limit | Memory Request | Memory Limit |
|------------------|------------|-----------|---------------|-------------|
| API server        | 500m       | 2000m     | 512 Mi        | 2 Gi        |
| Background worker | 1000m      | 2000m     | 1 Gi          | 2 Gi        |
| PostgreSQL        | --         | 2000m     | --            | 1 Gi        |
| Redis             | --         | 1000m     | --            | 768 Mi      |
| Prometheus        | --         | 1000m     | --            | 1 Gi        |
| Grafana           | --         | 1000m     | --            | 512 Mi      |
| Alertmanager      | --         | 500m      | --            | 256 Mi      |

### 5.3 Disk Space Estimates

| Data                        | Estimate                                 |
|----------------------------|------------------------------------------|
| SQLite scan database        | ~1 KB per scan result                     |
| PostgreSQL database         | Start with 20 Gi PVC, monitor growth      |
| Redis                       | 512 MB maxmemory (LRU eviction)          |
| YARA rules                  | < 10 MB                                   |
| ML models (`.joblib`)       | 5-50 MB per model                         |
| Prometheus TSDB (30d)       | ~500 MB for a medium deployment           |
| Grafana data                | < 100 MB                                  |
| Scan artifacts / uploads    | Highly variable; plan for peak daily volume |
| Application logs            | ~100 MB/day at info level; rotate weekly   |

### 5.4 Scaling Notes

- The Helm chart enables HPA (Horizontal Pod Autoscaler) by default: API scales at 70% CPU utilization (2-8 replicas), workers scale on `hades_queue_depth` metric (2-16 replicas).
- Docker Compose scaling: `docker compose -f docker/docker-compose.full-stack.yml up -d --scale hades-worker=4`
- ML detection (ensemble mode with Isolation Forest + Random Forest) adds approximately 500 MB RAM per worker. Add 1 GB headroom when using `--ml-ensemble`.
- File monitor should run on exactly one instance per watched directory to prevent duplicate alerts.

---

## 6. Upgrade Procedure

### 6.1 Pre-Upgrade Checklist

- [ ] Read the CHANGELOG.md and RELEASE_NOTES for the target version.
- [ ] Verify current system health: `curl http://localhost:8666/api/v1/health`
- [ ] Back up all databases (see Section 3).
- [ ] Back up custom YARA rules and configuration files.
- [ ] Back up ML models if trained on site-specific data.
- [ ] Note the current version: `pip show hades-scanner | grep Version`

### 6.2 Upgrade Steps (pip Install)

```bash
# 1. Stop the running service
sudo systemctl stop hades

# 2. Activate the virtual environment
source /opt/hades/venv/bin/activate

# 3. Upgrade the package
pip install --upgrade hades-scanner

# 4. For full dependencies (ML, observability, vulnerability, log analysis)
pip install --upgrade "hades-scanner[full]"

# 5. Run database migrations if the release notes mention schema changes
python core/hades_enhanced_cli.py --migrate-db

# 6. Validate YARA rules (new version may add rules)
python core/hades_enhanced_cli.py --validate-rules-strict

# 7. Verify the CLI starts cleanly
python core/hades_enhanced_cli.py --help

# 8. Start the service
sudo systemctl start hades

# 9. Verify health
curl http://localhost:8666/api/v1/health
```

### 6.3 Upgrade Steps (Docker)

```bash
# 1. Pull or build the new image
docker build -f docker/hades-full.dockerfile -t hades-scanner:NEW_VERSION .

# 2. Back up volumes (PostgreSQL example)
docker compose -f docker/docker-compose.full-stack.yml exec -T postgres \
  pg_dump -U hades -Fc hades > /backups/pre_upgrade_$(date +%Y%m%d).dump

# 3. Update the image tag in docker-compose.full-stack.yml or .env
#    e.g., set HADES_IMAGE_TAG=NEW_VERSION

# 4. Rolling restart (zero downtime with multiple replicas)
docker compose -f docker/docker-compose.full-stack.yml up -d --no-deps hades-api

# 5. Verify the new container is healthy
docker compose -f docker/docker-compose.full-stack.yml ps
curl http://localhost:8666/api/v1/health

# 6. Upgrade workers
docker compose -f docker/docker-compose.full-stack.yml up -d --no-deps hades-worker

# 7. Monitor Grafana dashboards for error spikes after upgrade
```

### 6.4 Upgrade Steps (Kubernetes / Helm)

```bash
# 1. Update values.yaml with the new image tag
#    image.tag: "NEW_VERSION"

# 2. Dry-run the upgrade
helm upgrade hades ./kubernetes/helm/hades/ -f values.yaml --dry-run

# 3. Apply the upgrade (Kubernetes handles rolling deployment)
helm upgrade hades ./kubernetes/helm/hades/ -f values.yaml

# 4. Watch rollout progress
kubectl rollout status deployment/hades-api
kubectl rollout status deployment/hades-worker

# 5. Verify health
kubectl exec deploy/hades-api -- python core/hades_enhanced_cli.py --health-check
```

### 6.5 Rollback

If the upgrade fails:

```bash
# pip: reinstall the previous version
pip install hades-scanner==PREVIOUS_VERSION

# Docker: revert image tag and restart
docker compose -f docker/docker-compose.full-stack.yml up -d

# Helm: rollback to previous release
helm rollback hades

# Restore database from backup if schema migration caused issues
pg_restore -h localhost -U hades -d hades --clean --if-exists /backups/pre_upgrade_YYYYMMDD.dump
```

### 6.6 Post-Upgrade Validation

- [ ] Health endpoint returns `200`: `curl http://localhost:8666/api/v1/health`
- [ ] Prometheus metrics are flowing: check Grafana Overview dashboard
- [ ] Run a test scan: `curl -X POST http://localhost:8666/api/v1/scan -H "X-API-Key: KEY" -F "file=@test.jpg"`
- [ ] Verify RBAC login works: `curl -X POST http://localhost:8666/api/v1/auth/login -H "Content-Type: application/json" -d '{"username":"admin","password":"..."}'`
- [ ] Check playbook engine: `python core/hades_enhanced_cli.py --playbook-status`
- [ ] Verify no alert rules are firing in Alertmanager: `curl http://localhost:9093/api/v2/alerts`
