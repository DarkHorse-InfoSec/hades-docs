# Observability Guide

> **Updated 2026-11-08:** HADES ships as the `hades` and `hades-server` binaries; there is no pip package and nothing to install separately. The metrics exporter is built into the API server and is on by default when the `prometheus_client` component is present in your build; check with `hades doctor`. The Prometheus, Grafana and Alertmanager configuration files and dashboards described below are in the HADES source tree and are not distributed with the binaries: use them as a reference for your own monitoring stack. See [Installation Guide](installation.md) for the install paths.

HADES includes a built-in Prometheus metrics exporter. This guide lists its metrics and describes the dashboards and alert rules DarkHorse uses, so you can reproduce them in your own Prometheus, Grafana and Alertmanager.

## Prerequisites

- The `hades-server` (or `hades serve`) API server running, with `prometheus_client` present in your build (`hades doctor` reports it)
- Your own Prometheus, and optionally Grafana and Alertmanager

## Quick Start

```bash
# Confirm prometheus_client is present
hades doctor

# Start the API server; metrics are on by default
hades-server --port 8666

# Check the scrape endpoint
curl http://localhost:8666/metrics
```

| Endpoint | URL | Purpose |
|---------|-----|---------|
| HADES `/metrics` | `http://localhost:8666/metrics` | Prometheus scrape target |

## Prometheus Metrics

### Enabling Metrics

Metrics are on by default whenever `prometheus_client` is present in your build, and off when it is not. `hades doctor` shows which applies.

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

An example scrape configuration that scrapes the HADES API at 10-second intervals:

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

DarkHorse uses four dashboards (they are in the HADES source tree and are not distributed with the binaries):

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

DarkHorse defines seven alert rules (the rule file is in the HADES source tree and is not distributed with the binaries):

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

Route the alerts with your own Alertmanager configuration (for example to a webhook, email or PagerDuty receiver).

Critical alerts suppress matching warning alerts for the same alert name.

## Troubleshooting

### Metrics endpoint returns empty response

Run `hades doctor` and check that `prometheus_client` is present. If it is reported missing, your build does not include it and `/metrics` answers `503` with `# Prometheus metrics disabled`; contact support@darkhorseinfosec.com.

Check the endpoint directly:

```bash
curl -i http://localhost:8666/metrics
```

### Prometheus cannot scrape HADES

Ensure the HADES API is reachable from the Prometheus host on the API port (default 8666). Check the target status in Prometheus at `http://localhost:9090/targets`.

### Grafana dashboards show "No data"

1. Verify the Prometheus datasource is configured in your Grafana
2. Check that Prometheus is scraping successfully
3. Generate some scan activity to produce metrics

### High cardinality warnings

The HADES metrics use bounded label sets. If you add custom labels, ensure the cardinality stays manageable (< 10,000 unique series).
