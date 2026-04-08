# HADES Deployment Guide

This guide covers deploying HADES (Hidden Artifact Detection & EXIF Scanner) in production environments using Docker, Docker Compose, and direct Python installation.

## Prerequisites

- **Docker** 20.10+ and **Docker Compose** v2+ (for containerized deployment)
- **Python** 3.8+ and **pip** (for direct installation)
- At least 2 GB RAM (4 GB recommended for ML anomaly detection)
- Sufficient disk space for scan data, logs, and ML models

## Docker Deployment

### Building the Image

Build the production Docker image from the project root:

```bash
docker build -f docker/hades-api.dockerfile -t hades-api:latest .
```

### Running Standalone

Run the HADES API server as a standalone container:

```bash
docker run -d \
  --name hades-api \
  -p 8666:8666 \
  -e HADES_API_KEY=your-secure-api-key \
  hades-api:latest
```

Verify the server is running:

```bash
curl http://localhost:8666/api/v1/health
```

### Using Docker Compose

Docker Compose provides a pre-configured deployment with volume mounts, environment variables, and health checks.

Start the stack:

```bash
docker compose -f docker/docker-compose.yml up -d
```

Stop the stack:

```bash
docker compose -f docker/docker-compose.yml down
```

View logs:

```bash
docker compose -f docker/docker-compose.yml logs -f
```

Restart after configuration changes:

```bash
docker compose -f docker/docker-compose.yml restart
```

## Environment Variable Reference

| Variable | Description | Default |
|---|---|---|
| `HADES_API_KEY` | API authentication key for all protected endpoints | `your-api-key-here` |
| `VIRUSTOTAL_API_KEY` | VirusTotal API key for threat intelligence lookups | *(none)* |
| `ABUSEIPDB_API_KEY` | AbuseIPDB API key for IP reputation lookups | *(none)* |
| `OTX_API_KEY` | AlienVault OTX API key for threat intel enrichment | *(none)* |

These environment variables are read by the application at startup. When using Docker Compose, set them in a `.env` file alongside the `docker-compose.yml` or pass them directly in the `environment` section.

**Example `.env` file:**

```bash
HADES_API_KEY=your-production-key-here
VIRUSTOTAL_API_KEY=your-vt-key
ABUSEIPDB_API_KEY=your-abuseipdb-key
OTX_API_KEY=your-otx-key
```

The `HADES_API_KEY` variable overrides the key list in `hades_config.json`. Threat intel API keys flow into the `ThreatIntelManager` and are used by the corresponding providers (`VirusTotalProvider`, `AbuseIPDBProvider`, `OTXAlienVaultProvider`).

## Volume Mounts

The Docker Compose configuration exposes three volume mounts:

| Host Path | Container Path | Purpose |
|---|---|---|
| `./config/` | `/app/config/` | Configuration files (`hades_config.json`, `threat_intelligence.config`) |
| `./logs/` | `/app/logs/` | Application logs, monitoring alert logs (JSON Lines) |
| `./scan-data/` | `/app/scan-data/` | Directory for files to be monitored or scanned via API |

### Config Volume

Place your `hades_config.json` and `threat_intelligence.config` in the `config/` directory. The container reads these at startup. Changes require a container restart to take effect.

### Logs Volume

The `logs/` directory receives:
- Application logs from the Python `logging` module
- File monitor alert logs (JSON Lines format, one alert per line)
- SIEM forwarding logs when file-based SIEM output is configured

### Scan Data Volume

Mount directories containing files you want to scan or monitor. The file monitor watches this path for new/modified files and automatically triggers scans.

## Production Configuration

### API Key Management

The default development key (`your-api-key-here`) must be replaced before production deployment.

**Option 1: Environment variable (recommended)**

```bash
docker run -e HADES_API_KEY=your-secure-key hades-api:latest
```

**Option 2: Config file**

Edit `config/hades_config.json`:

```json
{
  "api": {
    "api_keys": ["your-production-key-1", "your-production-key-2"]
  }
}
```

Multiple API keys are supported for key rotation without downtime.

### SIEM Forwarding

HADES supports forwarding scan results and monitor alerts to SIEM systems in multiple formats: Syslog, CEF, STIX, LEEF, and ECS.

**Configure via API:**

```bash
curl -X PUT http://localhost:8666/api/v1/siem/config \
     -H "X-API-Key: your-api-key" \
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

**Configure via CLI:**

```bash
python core/hades_enhanced_cli.py --monitor /watched/ \
  --siem-format syslog --siem-output syslog --siem-target 10.0.0.50:514
```

See `docs/siem_integration_guide.md` for detailed format examples and configuration options.

### Monitoring Directory Configuration

Start the file monitor via the API to watch the scan-data volume:

```bash
curl -X POST http://localhost:8666/api/v1/monitor/start \
     -H "X-API-Key: your-api-key" \
     -H "Content-Type: application/json" \
     -d '{
       "directories": ["/app/scan-data"],
       "recursive": true,
       "alert_threshold": 5
     }'
```

Or via the CLI:

```bash
python core/hades_enhanced_cli.py --monitor /app/scan-data --alert-threshold 5
```

### ML Model Management

The ML anomaly detection engine uses an Isolation Forest model trained on metadata features. In production:

**Generate a baseline model** from known-good files:

```bash
python core/hades_enhanced_cli.py --ml-train /path/to/known-good-files/
```

**Check model status:**

```bash
curl -H "X-API-Key: your-api-key" http://localhost:8666/api/v1/ml/status
```

**Retrain the model** after accumulating new samples:

```bash
curl -X POST -H "X-API-Key: your-api-key" \
     http://localhost:8666/api/v1/ml/retrain
```

The ML model file is stored at the path configured in `hades_config.json` under `ml_detection.model_path`. Mount this path as a volume if you want the model to persist across container restarts.

### Threat Intelligence Provider Setup

Configure cloud threat intel providers in `config/hades_config.json`:

```json
{
  "threat_intel": {
    "providers": {
      "virustotal": {
        "enabled": true,
        "api_key": ""
      },
      "abuseipdb": {
        "enabled": true,
        "api_key": ""
      },
      "otx": {
        "enabled": true,
        "api_key": ""
      },
      "malwarebazaar": {
        "enabled": true
      }
    },
    "cache_ttl_seconds": 3600,
    "max_cache_entries": 10000
  }
}
```

Leave `api_key` fields empty in the config file and pass the actual keys through environment variables (`VIRUSTOTAL_API_KEY`, `ABUSEIPDB_API_KEY`, `OTX_API_KEY`). MalwareBazaar does not require an API key.

## Scaling Considerations

### Horizontal Scaling

HADES can be scaled horizontally behind a load balancer:

```
                  +---> hades-api-1 (container)
Load Balancer --->+---> hades-api-2 (container)
                  +---> hades-api-3 (container)
```

Requirements for horizontal scaling:
- Each instance maintains its own SQLite scan result database. For shared result access, configure a shared database path on a network volume or use an external database
- File monitor should run on only one instance per watched directory to avoid duplicate alerts
- ML models should be trained centrally and distributed to all instances
- SIEM forwarding can run independently on each instance

### Shared Storage

When running multiple instances:
- Mount `scan-data/` from a shared NFS or CIFS volume so all instances can access uploaded files
- Mount `config/` from a shared volume to ensure consistent configuration
- Consider mounting `logs/` to a shared volume or use centralized logging

### Resource Allocation

Recommended container resource limits:

| Component | CPU | Memory |
|---|---|---|
| API server (scanning only) | 1 core | 1 GB |
| API server + ML detection | 2 cores | 2 GB |
| API server + file monitor | 2 cores | 2 GB |
| Full stack (all features) | 4 cores | 4 GB |

## Security Hardening

### Non-Root User

The Dockerfile creates a non-root `hades` user. The application runs as this user by default. Do not override this with `--user root`.

### Read-Only Filesystem

For additional security, run with a read-only root filesystem and explicit writable mounts:

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  -v ./logs:/app/logs \
  -v ./scan-data:/app/scan-data \
  -v ./config:/app/config:ro \
  hades-api:latest
```

### Network Policies

In production, restrict outbound network access:
- Allow outbound HTTPS (443) to threat intelligence APIs: `www.virustotal.com`, `api.abuseipdb.com`, `otx.alienvault.com`, `mb-api.abuse.ch`
- Allow outbound to your SIEM target (syslog port, HTTP endpoint)
- Block all other outbound traffic

### Secret Management

- Never commit API keys to version control
- Use Docker secrets or environment variables for sensitive configuration
- Rotate the `HADES_API_KEY` regularly
- Use separate, scoped threat intel API keys for production

```bash
# Docker secrets example
echo "your-secure-api-key" | docker secret create hades_api_key -
```

### Rate Limiting

HADES includes built-in sliding-window rate limiting via `core/security_middleware.py` (`SlidingWindowRateLimiter`). When the security middleware is available, API endpoints are rate-limited per client. For production deployments handling high traffic, you can add an additional layer of rate limiting at the reverse proxy level (nginx, Traefik, or a cloud load balancer):

```nginx
# nginx example (supplementary rate limiting)
limit_req_zone $binary_remote_addr zone=hades:10m rate=10r/s;

location /api/ {
    limit_req zone=hades burst=20 nodelay;
    proxy_pass http://hades-api:8666;
}
```

## Troubleshooting

### Container Fails to Start

Check logs for startup errors:

```bash
docker compose -f docker/docker-compose.yml logs hades-api
```

Common causes:
- Missing or invalid `hades_config.json` — ensure the config volume is mounted correctly
- Port 8666 already in use — change the port mapping in the compose file
- Permission denied on volume mounts — ensure the `hades` user (UID in Dockerfile) has write access to `logs/` and `scan-data/`

### Health Check Fails

```bash
curl -v http://localhost:8666/api/v1/health
```

If the health endpoint returns `"degraded"` for an engine, check:
- YARA rules directory is present and contains valid `.yar`/`.yara` files
- Python dependencies installed correctly (run `pip list` inside the container)

### YARA Not Available

YARA is an optional dependency. If `yara-python` fails to install (common on some platforms), the scanner operates without YARA rule matching. The health endpoint will report `enhanced_detection: degraded`.

### ML Detection Not Available

If `scikit-learn` or `numpy` is not installed, ML detection endpoints return a 503 response. Install the dependencies:

```bash
pip install scikit-learn>=1.2.0 numpy>=1.24.0
```

## Direct Python Deployment

For environments where Docker is not available:

```bash
# Clone and install
pip install -r requirements.txt

# Start the API server
python core/hades_enhanced_cli.py --serve --port 8666

# Or run directly
python core/hades_api.py --host 0.0.0.0 --port 8666
```

For process management, use systemd, supervisord, or a similar process manager to keep the server running.

**Example systemd unit:**

```ini
[Unit]
Description=HADES Metadata Forensics API
After=network.target

[Service]
Type=simple
User=hades
WorkingDirectory=/opt/hades
ExecStart=/opt/hades/venv/bin/python core/hades_enhanced_cli.py --serve --port 8666
Restart=always
RestartSec=5
Environment=HADES_API_KEY=your-production-key

[Install]
WantedBy=multi-user.target
```
