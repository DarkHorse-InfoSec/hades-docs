# HADES Production Deployment Checklist
Version: 1.0.0

DarkHorse Information Security LLC

> **Updated 2026-11-08:** HADES ships as two self-contained binaries, `hades` and `hades-server`; there is no pip package and no container image (see [Installation Guide](installation.md)). The install, upgrade and rollback steps below use the binaries. Containerized and distributed deployments (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap and are not shipped (see the [Kubernetes Guide](kubernetes_guide.md)). Items that only apply to those deployments say so. Run `hades doctor` to see which optional components (for example `prometheus_client`, `redis`, `bcrypt`) are present in your build.

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

**Option A: Environment variable**

```bash
export HADES_API_KEY=$(python3 -c "import secrets; print(secrets.token_urlsafe(48))")
echo "Generated key: $HADES_API_KEY"

# Start the API server with the key in its environment
hades-server --port 8666
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
hades admin create-admin admin
```

> **Known issue in v1.7.4:** in the compiled `hades` binary, `hades admin create-admin`, `create-user`, `list-users`, `create-tenant` and `list-tenants` exit with an internal error (`NameError`) instead of running. The fix is tracked for the next release. Until then these commands cannot be used from the shipped binary.

This prompts for the password and prints the new admin's API key. The admin account can then manage other users.

### 2.3 Create Analyst and Viewer Accounts

```bash
hades admin create-user analyst1 --role analyst
hades admin create-user viewer1 --role viewer
```

Role permissions:
- **admin**: Full access (user management, config, scans, cases, system)
- **analyst**: Scan, create/update cases, export evidence, threat intel, ML detection
- **viewer**: Read-only access to scans, cases, evidence, audit logs

### 2.4 Rotate API Keys

```bash
# Via the API (requires admin role)
curl -X POST https://hades.example.com/api/v1/auth/users/<user_id>/rotate-key \
  -H "X-API-Key: current-admin-key"
```

### 2.5 Key Storage Best Practices

- Store API keys in a secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) rather than in plaintext config files.
- Keep the key out of shell history and scripts: put it in `~/.hades/.env`, which HADES loads at startup, readable only by the service account (`chmod 600`).
- Docker secrets and Kubernetes Secrets apply only to the roadmap container deployments (see the [Kubernetes Guide](kubernetes_guide.md)); they are not part of a binary deployment.
- Rotate keys on a regular schedule (every 90 days minimum).
- Never commit keys to version control. Ensure `.env` files and `*.key` files are listed in `.gitignore`.
- Use separate keys for each environment (development, staging, production).

---

## 3. Backup Strategy

### 3.1 SQLite (WAL Mode) Backup

HADES uses SQLite in WAL (Write-Ahead Logging) mode; it is the only database backend (PostgreSQL support was removed in v1.7.3). WAL allows concurrent reads during backup, but requires a specific approach for consistent snapshots.

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

### 3.2 Redis State

Redis stores ephemeral state (rate limiting, sessions, cache, distributed locks). It is not a primary data store in HADES. However, if persistence of Redis data is desired:

- Enable `appendonly yes` in your Redis configuration for crash recovery.
- For a manual backup: `redis-cli BGSAVE`

### 3.3 Supplementary Backups

Also back up these artifacts:
- `config/cli/hades_config.json` (application configuration)
- `rules/*.yar` and `rules/*.yara` (custom YARA rules)
- `models/*.joblib` (trained ML models)
- `config/encryption.key` (encryption key -- store separately, encrypted)

---

## 4. Monitoring

HADES exposes Prometheus metrics at `/metrics` on the API server.

### 4.1 Deploy the Observability Stack

Run your own Prometheus (and, if you use them, Grafana and Alertmanager) and scrape `http://<hades-host>:8666/metrics`. The Prometheus, Grafana and Alertmanager compose files and Grafana dashboards described in earlier versions of this checklist are not distributed with the binaries.

### 4.2 Enable Metrics on the HADES API

Metrics are on by default and switch off automatically when the `prometheus_client` component is not present in your build. Check with `hades doctor`, then:

```bash
hades-server --port 8666
curl http://localhost:8666/metrics
```

A `503` reply reading `# Prometheus metrics disabled` usually means `prometheus_client` is missing from your build.

### 4.3 Prometheus Configuration

An example scrape configuration for your own Prometheus (replace `hades-api` with the HADES host):

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

HADES defines seven alerting rules in its source tree; the rule files are not distributed with the binaries, so recreate the ones you want in your own Prometheus:

| Alert                  | Condition                        | Severity |
|-----------------------|----------------------------------|----------|
| HADESHighErrorRate    | Scan error rate > 5% for 5 min   | warning  |
| HADESQueueBacklog     | Queue depth > 100 for 10 min     | warning  |
| HADESNoWorkers        | 0 active workers for 2 min       | critical |
| HADESHighLatency      | p99 scan duration > 30s for 5 min| warning  |
| HADESAPIErrors        | HTTP 5xx rate > 10% for 5 min    | critical |
| HADESCacheDegraded    | Cache hit rate < 50% for 15 min  | info     |
| HADESDown             | Service unreachable for 1 min    | critical |

Configure notification targets (email, Slack, PagerDuty, etc.) in your own Alertmanager.

### 4.5 Grafana Dashboards

HADES has four Grafana dashboards in its source tree; they are not distributed with the binaries:

1. **HADES Overview** (`hades-overview.json`) -- Scan throughput, latency percentiles, threat distribution, error rate, cache hit rate
2. **Pipeline** (`hades-pipeline.json`) -- Per-stage duration, ExifTool/YARA/ML timing, bottleneck analysis
3. **Workers** (`hades-workers.json`) -- Active workers, queue depth, task rates, failure rate
4. **Infrastructure** (`hades-infrastructure.json`) -- API request/response metrics, cache, auth attempts

### 4.6 Health Check Probes

HADES provides a health endpoint and a CLI probe for container orchestrators:

```bash
# HTTP health check
curl http://localhost:8666/api/v1/health

# CLI probe (exits 0 on healthy, 1 on failure -- suitable for Kubernetes liveness/readiness)
hades serve --health-check
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

HADES publishes no measured capacity per profile. Multi-replica and worker deployments are on the Enterprise roadmap; a current deployment is one `hades-server` binary. Size CPU and RAM by benchmarking on your own file mix.

### 5.2 Per-Component Resource Limits

These are the limits planned for the roadmap container deployments (Docker Compose, Helm); they are not measured HADES requirements:

| Component         | CPU Request | CPU Limit | Memory Request | Memory Limit |
|------------------|------------|-----------|---------------|-------------|
| API server        | 500m       | 2000m     | 512 Mi        | 2 Gi        |
| Background worker | 1000m      | 2000m     | 1 Gi          | 2 Gi        |
| Redis             | --         | 1000m     | --            | 768 Mi      |
| Prometheus        | --         | 1000m     | --            | 1 Gi        |
| Grafana           | --         | 1000m     | --            | 512 Mi      |
| Alertmanager      | --         | 500m      | --            | 256 Mi      |

### 5.3 Disk Space Estimates

| Data                        | Estimate                                 |
|----------------------------|------------------------------------------|
| SQLite scan database        | ~1 KB per scan result                     |
| Redis                       | 512 MB maxmemory (LRU eviction)          |
| YARA rules                  | < 10 MB                                   |
| ML models (`.joblib`)       | 5-50 MB per model                         |
| Prometheus TSDB (30d)       | ~500 MB for a medium deployment           |
| Grafana data                | < 100 MB                                  |
| Scan artifacts / uploads    | Highly variable; plan for peak daily volume |
| Application logs            | Depends on scan volume; rotate weekly       |

### 5.4 Scaling Notes

- Horizontal scaling (Helm HPA, Docker Compose worker replicas) belongs to the roadmap container deployments. They (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap and are not shipped.
- File monitor should run on exactly one instance per watched directory to prevent duplicate alerts.

---

## 6. Upgrade Procedure

### 6.1 Pre-Upgrade Checklist

- [ ] Read the CHANGELOG.md and RELEASE_NOTES for the target version.
- [ ] Verify current system health: `curl http://localhost:8666/api/v1/health`
- [ ] Back up all databases (see Section 3).
- [ ] Back up custom YARA rules and configuration files.
- [ ] Back up ML models if trained on site-specific data.
- [ ] Note the current version: `hades --version`

### 6.2 Upgrade Steps (Binary)

```bash
# 1. Stop the running service
sudo systemctl stop hades

# 2. Install the new release with the same path you installed with
#    (Path A or Path B, see Installation > Upgrading)

# 3. Run database migrations if the release notes mention schema changes
hades admin migrate

# 4. Validate YARA rules (new version may add rules)
hades rules validate

# 5. Verify the new version starts cleanly
hades --version

# 6. Start the service
sudo systemctl start hades

# 7. Verify health
curl http://localhost:8666/api/v1/health
```

See [Upgrading](installation.md#upgrading) for the download commands.

> **Known issue in v1.7.4:** `hades rules validate` exits with "CI rule validator not found" in the compiled binary; the validator was not included in the build. The fix is tracked for the next release.

### 6.3 Upgrade Steps (Docker)

Not applicable: there is no HADES container image. Containerized and distributed deployments (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap and are not shipped (see the [Kubernetes Guide](kubernetes_guide.md)). A current deployment is the `hades-server` binary.

### 6.4 Upgrade Steps (Kubernetes / Helm)

Not applicable for the same reason; see the [Kubernetes Guide](kubernetes_guide.md).

### 6.5 Rollback

If the upgrade fails:

```bash
# 1. Stop the service
sudo systemctl stop hades

# 2. Reinstall the previous release: Path A with the previous version number
#    (see Installation > Upgrading)

# 3. If a schema migration caused issues, restore the SQLite databases from the
#    backup taken before the upgrade (see Section 3)

# 4. Start the service and check health
sudo systemctl start hades
curl http://localhost:8666/api/v1/health
```

### 6.6 Post-Upgrade Validation

- [ ] Health endpoint returns `200`: `curl http://localhost:8666/api/v1/health`
- [ ] Prometheus metrics are flowing: `curl http://localhost:8666/metrics`
- [ ] Run a test scan: `curl -X POST http://localhost:8666/api/v1/scan -H "X-API-Key: KEY" -F "file=@test.jpg"`
- [ ] Verify RBAC login works: `curl -X POST http://localhost:8666/api/v1/auth/login -H "Content-Type: application/json" -d '{"username":"admin","password":"..."}'`
- [ ] Check playbook engine: `hades admin playbook status`
- [ ] If you run Alertmanager, verify no alert rules are firing: `curl http://localhost:9093/api/v2/alerts`
