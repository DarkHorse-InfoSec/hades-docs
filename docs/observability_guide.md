# Observability Guide

> **NOTICE (2026-05-06):** This guide describes observability features that have been in HADES since v0.7.1. The functionality remains accurate; only the install instructions have changed. References below to `pip install "hades-scanner[observability]"` and similar pip-extras commands no longer apply -- HADES now ships as a single Nuitka-compiled binary (v1.4.2) with all observability dependencies (`prometheus-client`, etc.) bundled inline. Customers do NOT need to install anything separately for the metrics exporter to work; it is enabled via `hades-server --metrics` or the `enable_metrics` config option. See [Installation Guide](installation.md) for current install paths (Path A direct portal download, Path B Homebrew tap).

HADES v0.7.1 includes a built-in Prometheus metrics exporter, pre-configured Grafana dashboards, and alerting rules for production monitoring.

## Prerequisites

- Install the observability extras: `pip install "hades-scanner[observability]"` (installs `prometheus-client`)
- Docker and Docker Compose (for the observability stack)

## Quick Start

```bash
# Start the full stack with observability
docker compose -f docker/docker-compose.full-stack.yml up -d

# Or add observability to an existing deployment
docker compose -f docker/docker-compose.observability.yml up -d
```

| Service | URL | Purpose |
|---------|-----|---------|
| Prometheus | `http://localhost:9090` | Metric storage and alerting |
| Grafana | `http://localhost:3000` | Dashboards (default login: admin/admin) |
| Alertmanager | `http://localhost:9093` | Alert routing |
| HADES `/metrics` | `http://localhost:8666/metrics` | Prometheus scrape target |

## Prometheus Metrics

### Enabling Metrics

Metrics are auto-enabled when `prometheus_client` is installed. Control via CLI:

```bash
# Explicitly enable
hades-enhanced --serve --metrics

# Disable
hades-enhanced --serve --no-metrics
```

Or via environment variable:

```bash
export HADES_METRICS_ENABLED=true
```

### Metric Reference

#### Counters

| Metric | Labels | Description |
|--------|--------|-------------|
| `hades_scans_total` | `status` (completed, error) | Total scans processed |
| `hades_findings_total` | `severity` (critical, high, medium, low, info) | Total findings by severity |
| `hades_cache_total` | `result` (hit, miss) | Cache lookup results |
| `hades_worker_tasks_total` | `status` (completed, failed) | Worker task completions |
| `hades_api_requests_total` | `method`, `endpoint`, `status` | API request count |
| `hades_auth_events_total` | `event` (login_success, login_failure, key_rotation) | Auth events |

#### Histograms

| Metric | Labels | Description |
|--------|--------|-------------|
| `hades_scan_duration_seconds` | -- | End-to-end scan duration |
| `hades_stage_duration_seconds` | `stage` | Per-stage timing (metadata, yara, heuristic, ml) |
| `hades_api_request_duration_seconds` | `method`, `endpoint` | API request latency |
| `hades_file_size_bytes` | -- | Scanned file sizes |

#### Gauges

| Metric | Labels | Description |
|--------|--------|-------------|
| `hades_active_scans` | -- | Currently running scans |
| `hades_queue_depth` | -- | Pending scan queue size |
| `hades_active_workers` | -- | Active worker processes |
| `hades_cache_size` | -- | Current cache entry count |
| `hades_uptime_seconds` | -- | Time since server start |

### API Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /metrics` | Prometheus text format (scrape target) |
| `GET /api/v1/metrics/json` | JSON metric summary |
| `GET /api/v1/metrics/health` | Enhanced health check with scan rate and error rate |
| `GET /api/v1/metrics/alerts` | Active alert conditions |

## Prometheus Configuration

The default scrape configuration (`docker/prometheus/prometheus.yml`) scrapes the HADES API at 10-second intervals:

```yaml
scrape_configs:
  - job_name: 'hades-api'
    static_configs:
      - targets: ['hades-api:8666']
    metrics_path: /metrics
    scrape_interval: 10s
```

### PromQL Examples

**Scan throughput (scans per second, 5-minute window):**

```promql
rate(hades_scans_total[5m])
```

**Error rate percentage:**

```promql
rate(hades_scans_total{status="error"}[5m])
/ rate(hades_scans_total[5m]) * 100
```

**p95 scan latency:**

```promql
histogram_quantile(0.95, rate(hades_scan_duration_seconds_bucket[5m]))
```

**Cache hit rate:**

```promql
rate(hades_cache_total{result="hit"}[5m])
/ rate(hades_cache_total[5m]) * 100
```

**Findings by severity (last hour):**

```promql
increase(hades_findings_total[1h])
```

**API request rate by endpoint:**

```promql
sum by (endpoint) (rate(hades_api_requests_total[5m]))
```

## Grafana Dashboards

Four dashboards are provisioned automatically:

### Overview Dashboard

High-level operational view: total scans, scan rate, error rate, threat distribution by severity, active scans gauge, and queue depth.

### Pipeline Dashboard

Per-stage timing breakdown: metadata extraction, YARA matching, heuristic analysis, and ML detection. Identifies bottleneck stages.

### Workers Dashboard

Worker pool status: active workers, task completion rate, failed tasks, dead letter queue, queue depth trends, and per-worker task distribution.

### Infrastructure Dashboard

API request rate and latency, HTTP status code breakdown, cache hit/miss rates, cache size, authentication events, and uptime.

### Custom Dashboards

Create custom dashboards in Grafana using the Prometheus datasource. All HADES metrics use the `hades_` prefix.

## Alerting

### Prometheus Alert Rules

Seven alert rules are defined in `docker/prometheus/alerts.yml`:

| Alert | Condition | Severity | For |
|-------|-----------|----------|-----|
| `HADESHighErrorRate` | Scan error rate > 5% | warning | 5m |
| `HADESQueueBacklog` | Queue depth > 100 | warning | 10m |
| `HADESNoWorkers` | Active workers = 0 | critical | 2m |
| `HADESHighLatency` | p99 scan duration > 30s | warning | 5m |
| `HADESAPIErrors` | HTTP 5xx rate > 10% | critical | 5m |
| `HADESCacheDegraded` | Cache hit rate < 50% | info | 15m |
| `HADESDown` | Target unreachable | critical | 1m |

### Alertmanager

The default Alertmanager configuration routes alerts to a webhook receiver. Configure the webhook URL via the `ALERTMANAGER_WEBHOOK_URL` environment variable, or edit `docker/alertmanager/alertmanager.yml` directly.

Critical alerts suppress matching warning alerts for the same alert name.

## Troubleshooting

### Metrics endpoint returns empty response

Verify `prometheus_client` is installed:

```bash
pip install prometheus-client
```

Check that metrics are enabled:

```bash
curl http://localhost:8666/metrics
```

### Prometheus cannot scrape HADES

Ensure the HADES API is reachable from the Prometheus container. In Docker Compose, services share the same network. Check the target status in Prometheus at `http://localhost:9090/targets`.

### Grafana dashboards show "No data"

1. Verify the Prometheus datasource is configured (provisioned automatically)
2. Check that Prometheus is scraping successfully
3. Generate some scan activity to produce metrics

### High cardinality warnings

The HADES metrics use bounded label sets. If you add custom labels, ensure the cardinality stays manageable (< 10,000 unique series).
