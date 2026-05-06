# HADES v0.7.1 -- Observability & Kubernetes

**Release Date:** 2026-02-21

> **HISTORICAL NOTICE (added 2026-05-06):** This release note describes HADES under the pre-v1.4 distribution model (Cloudsmith pip registry, optional dependency groups, Docker Hub images, public source-clone). **Those install paths no longer work.** Cloudsmith Free plan stopped allowing RAW packages on 2026-04-29; HADES now ships exclusively as a Nuitka-compiled signed binary distributed via the customer portal (Path A) or the Homebrew tap (Path B). For current installation, see the [Installation Guide](docs/installation.md). The features described below were accurate as of v0.7.1; many have evolved or been replaced in subsequent releases through v1.4.2.

HADES v0.7.1 adds production-grade observability with Prometheus metrics, pre-configured Grafana dashboards, alerting rules, and Kubernetes deployment support via Helm and Kustomize. Operators can now monitor scan throughput, pipeline stage timing, worker pool health, cache efficiency, and API performance in real time -- and deploy HADES on Kubernetes with horizontal pod autoscaling, ingress, and ServiceMonitor integration out of the box.

All new features are opt-in. Existing v0.7.0 configurations and workflows continue to work without modification.

---

## What's New

### Prometheus Metrics Exporter

A comprehensive metrics module (`core/metrics.py`) instruments every major subsystem with 20+ Prometheus metrics. Collection is zero-overhead when disabled or when `prometheus_client` is not installed.

**Counters:**
- `hades_scans_total` -- completed scans by status
- `hades_findings_total` -- detection findings by severity
- `hades_cache_hits_total` / `hades_cache_misses_total` -- cache performance
- `hades_worker_tasks_total` -- worker task completions by status
- `hades_api_requests_total` -- API requests by method, endpoint, status code
- `hades_auth_attempts_total` -- authentication attempts by method and result

**Histograms:**
- `hades_scan_duration_seconds` -- end-to-end scan latency
- `hades_stage_duration_seconds` -- per-stage timing (metadata, YARA, heuristic, ML, deep format)
- `hades_api_request_duration_seconds` -- API endpoint latency
- `hades_exiftool_duration_seconds` -- ExifTool subprocess latency
- `hades_file_size_bytes` -- scanned file size distribution

**Gauges:**
- `hades_active_scans` -- currently running scans
- `hades_queue_depth` -- pending task queue size
- `hades_active_workers` -- live worker count
- `hades_cache_size` -- current cache entry count
- `hades_uptime_seconds` -- process uptime
- `hades_yara_rules_loaded` -- compiled YARA rule count
- `hades_ioc_database_size` -- IOC database entry count

### FastAPI Prometheus Middleware

A `PrometheusMiddleware` class automatically instruments every API endpoint with request counting, duration tracking, and per-status-code breakdowns. Each response includes an `X-Request-Duration` header for client-side latency visibility.

### Metrics API Endpoints

Four new endpoints expose metrics and operational health:

| Endpoint | Purpose |
|----------|---------|
| `GET /metrics` | Prometheus scrape endpoint (text/plain exposition format) |
| `GET /api/v1/metrics/json` | JSON summary of all metric families |
| `GET /api/v1/metrics/health` | Enhanced health check with scan rate, error rate, cache hit rate |
| `GET /api/v1/metrics/alerts` | Active alert conditions (error spikes, queue backlogs, worker shortages) |

The `/api/v1/metrics/health` endpoint is designed for Kubernetes readiness probes -- it returns structured health data with pass/fail status.

### Grafana Dashboards

Four pre-built Grafana dashboards (`grafana/dashboards/`) are included and auto-provisioned:

- **HADES Overview** -- scan throughput, average latency, threat severity distribution, top findings
- **HADES Pipeline** -- per-stage timing breakdown, bottleneck identification, ExifTool performance
- **HADES Workers** -- worker pool status, queue depth, task completion rates, dead letter queue
- **HADES Infrastructure** -- API request rates, cache hit/miss ratio, authentication activity, uptime

Grafana provisioning (`grafana/provisioning/`) auto-configures the Prometheus datasource and dashboard discovery, so dashboards appear immediately after deployment with no manual import steps.

### Prometheus Alerting Rules

Seven alerting rules (`docker/prometheus/alerts.yml`) with Alertmanager routing cover critical operational conditions:

| Alert | Condition |
|-------|-----------|
| High scan error rate | Error rate exceeds threshold over 5-minute window |
| Queue backlog | Pending tasks exceed queue depth threshold |
| Worker availability | Active workers drop below minimum |
| API error rate | 5xx response rate exceeds threshold |
| Cache degradation | Cache hit rate drops below expected baseline |
| Latency spike | P95 scan duration exceeds threshold |
| Service unavailable | Health endpoint unreachable |

### Kubernetes Helm Chart

A production-grade Helm chart (`kubernetes/helm/hades/`) deploys HADES on Kubernetes with:

- **API Deployment** -- configurable replicas, resource limits, startup/readiness/liveness probes
- **Worker Deployment** -- dedicated scan workers with separate scaling
- **Horizontal Pod Autoscaler** -- CPU/memory scaling for API pods, custom queue-depth metric scaling for workers
- **Ingress** -- configurable ingress with WebSocket upgrade support for `/ws` path
- **ServiceMonitor** -- Prometheus Operator ServiceMonitor for automatic scrape target discovery
- **Grafana ConfigMap** -- dashboards bundled as ConfigMap for Grafana sidecar injection
- **PVCs** -- persistent volumes for YARA rules and configuration
- **Topology spread** -- pod topology spread constraints for availability zone distribution

```bash
# Install with Helm
helm install hades kubernetes/helm/hades \
  --set api.replicas=3 \
  --set workers.replicas=4 \
  --set redis.url=redis://redis:6379
```

### Kustomize Overlays

Kustomize base and overlay configurations (`kubernetes/kustomize/`) provide an alternative to Helm:

- **Dev overlay** -- single replica, SQLite backend, no external dependencies
- **Production overlay** -- multi-replica API and workers, PostgreSQL, Redis, TLS ingress, HPA

```bash
# Dev deployment
kubectl apply -k kubernetes/kustomize/overlays/dev

# Production deployment
kubectl apply -k kubernetes/kustomize/overlays/production
```

### Docker Observability Stack

Two new Docker Compose files simplify deployment with full observability:

**Observability stack** (`docker/docker-compose.observability.yml`):
```bash
docker compose -f docker/docker-compose.observability.yml up -d
```
- Prometheus with pre-configured scrape targets and alerting rules
- Grafana with auto-provisioned datasource and dashboards
- Alertmanager with routing configuration

**Full stack** (`docker/docker-compose.full-stack.yml`):
```bash
docker compose -f docker/docker-compose.full-stack.yml up -d
```
- PostgreSQL + Redis + HADES API + workers + Prometheus + Grafana + Alertmanager
- Single command to bring up the complete production environment
- `.env.example` documents all available environment variables

### Documentation

Two new documentation guides:
- **[Observability Guide](docs/observability_guide.md)** -- Prometheus setup, metric descriptions, PromQL query examples, Grafana dashboard walkthrough, alerting configuration, troubleshooting
- **[Kubernetes Guide](docs/kubernetes_guide.md)** -- Helm installation, values reference, secrets management, HPA tuning, ServiceMonitor configuration, Kustomize alternative, troubleshooting

---

## Migration Guide from v0.7.0

### New Configuration Section

The following section has been added to `cli/hades_config.json`. It is optional and defaults to enabled:

```json
{
  "metrics": {
    "enabled": true,
    "endpoint": "/metrics"
  }
}
```

### New Optional Dependencies

```bash
# Install observability extras
pip install "hades-scanner[observability]"
```

| Package | Purpose | Required for |
|---------|---------|-------------|
| `prometheus_client` | Prometheus metrics collection and exposition | Metrics endpoints, middleware |

The `full` optional group now includes observability:
```bash
pip install "hades-scanner[full]"
```

### New CLI Flags

| Flag | Description |
|------|-------------|
| `--metrics` | Enable Prometheus metrics endpoint (default when prometheus_client is available) |
| `--no-metrics` | Disable Prometheus metrics endpoint |
| `--metrics-port` | Configure metrics port (default: served on same port as API) |
| `--health-check` | Run health check and exit with code 0 (healthy) or 1 (unhealthy) |

### Backward Compatibility

- All new features are **opt-in** -- metrics are enabled by default but have zero overhead when `prometheus_client` is not installed
- Existing CLI commands, API endpoints, and configuration files work unchanged
- No database migrations required
- Docker Compose files from v0.7.0 continue to work without modification

---

## Deployment Patterns

| Pattern | Command | What You Get |
|---------|---------|-------------|
| Standalone API | `hades-enhanced --serve` | API server with `/metrics` endpoint |
| Observability stack | `docker compose -f docker/docker-compose.observability.yml up -d` | API + Prometheus + Grafana + Alertmanager |
| Full stack | `docker compose -f docker/docker-compose.full-stack.yml up -d` | PostgreSQL + Redis + API + workers + observability |
| Kubernetes (Helm) | `helm install hades kubernetes/helm/hades` | API + workers + HPA + ServiceMonitor |
| Kubernetes (Kustomize) | `kubectl apply -k kubernetes/kustomize/overlays/production` | API + workers + HPA + ingress |

---

## Requirements

- Python 3.8+
- Optional: prometheus_client (for metrics collection)
- Optional: Redis (for distributed mode, cache tier 2, worker queues)
- Optional: Docker + Docker Compose (for observability and full stack deployments)
- Optional: Kubernetes 1.24+ with Helm 3 (for Kubernetes deployments)
- Optional: Prometheus Operator (for ServiceMonitor CRD)

---

## Known Issues

- **DOCX macro deep format detection**: 5 pre-existing test failures in `tests/test_corpus_documents.py` for DOCX macro-based attacks. Deep format analysis detects these via the `EnhancedDetectionEngine` but the corpus test assertions require updated thresholds. Detection still works through YARA and heuristic stages.
- **SAML SP-initiated pending**: The SAML 2.0 provider validates IdP-initiated responses but does not yet support SP-initiated authentication flows. Full SP-initiated SAML is planned for v0.8.0.
- **Plugin marketplace local-only**: The plugin registry currently operates with local package files. A remote registry for community plugin discovery is planned for v0.8.0.
- **pikepdf/olefile optional**: Deep format analysis for PDF and Office files requires pikepdf and olefile respectively. If not installed, the engine falls back to regex-based parsing with reduced detection coverage.
- **Windows uvloop unavailable**: The `uvloop` async event loop optimization is not available on Windows. The async pipeline uses the default asyncio event loop on Windows.
- **Flaky TCP syslog test**: `core/test_siem_integration.py::TestSIEMForwarder::test_forwarder_tcp_syslog` may intermittently fail on Windows due to timing-dependent TCP socket behavior.

---

## What's Next (v0.8.0 Preview)

- **Remote plugin registry**: Centralized community plugin discovery and distribution
- **SP-initiated SAML**: Full SAML 2.0 SP-initiated authentication flow
- **Scheduled scanning**: Cron-style scan scheduling for automated periodic analysis
- **Report templates**: Customizable HTML/PDF report templates with organization branding
- **Rule auto-update**: Automatic YARA rule updates from a curated threat feed
- **SOAR integration**: Pre-built connectors for Splunk SOAR, Palo Alto XSOAR, and IBM QRadar SOAR
- **GPU-accelerated ML**: CUDA/ROCm support for faster ML ensemble inference
- **Real-time streaming analysis**: Kafka/NATS consumer for continuous file stream processing

See [HADES_Competitive_Analysis_and_Roadmap.md](HADES_Competitive_Analysis_and_Roadmap.md) for the full roadmap.

---

## Test Suite

1,651+ tests passing across all modules. The v0.7.1 observability phase adds 143 new tests covering Prometheus metrics collection, Grafana dashboard validation, Kubernetes manifest structure, Helm chart templating, Kustomize overlays, Docker Compose configurations, and metrics API endpoints.

---

## Full Changelog

See [CHANGELOG.md](CHANGELOG.md) for the complete list of changes.

---

DarkHorse Information Security LLC
