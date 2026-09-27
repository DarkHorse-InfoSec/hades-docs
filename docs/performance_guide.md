# HADES Performance Guide

> **ROADMAP, not a shipped deployment option (updated 2026-09-27).** HADES currently ships as a single self-contained binary (the `hades` CLI and `hades-server`). Containerized and distributed deployments (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap. This guide describes that planned architecture. **No throughput, scans-per-day or hardware-sizing figure in it is a measured HADES result**, and none is given; size a deployment by benchmarking on your own hardware and file mix.
>
> References below to `pip install "hades-scanner[full]"` or `pip install "hades-scanner[observability]"` do not apply to the shipped binary, which bundles its dependencies inline. See [Installation Guide](installation.md) for current install paths.

This guide covers performance tuning, deployment patterns, benchmarking, and monitoring for HADES metadata forensics engine.

## Architecture Overview

### Synchronous vs Asynchronous Pipeline

HADES supports two scanning modes:

- **Synchronous (default):** Each scan request processes sequentially through metadata extraction, YARA matching, heuristic analysis, and ML detection. Suitable for single-server deployments handling moderate volumes.
- **Asynchronous pipeline (`--async`):** Scan tasks are submitted to a queue and processed by background workers. The API returns immediately with a task ID. Workers pull tasks from Redis and write results to the database. Suitable for high-throughput environments.

Enable the async pipeline:

```bash
hades-enhanced --serve --async
```

### ExifTool Persistent Process

HADES uses PyExifTool to maintain a persistent ExifTool subprocess, avoiding the overhead of spawning a new process per file. The persistent process handles metadata extraction for all scan requests.

### Worker Pool

The `--workers` flag controls how many concurrent scan workers process files:

```bash
# 4 scan workers + 2 uvicorn workers
hades-enhanced --serve --workers 4 --api-workers 2
```

For distributed deployments, run dedicated worker containers:

```bash
# Worker mode (no API server, processes queue tasks only)
hades-enhanced --worker --distributed
```

## Tuning Parameters

### API Server Workers

The `--api-workers` flag sets the number of uvicorn worker processes:

```bash
# Production: 2-4 workers per CPU core
hades-enhanced --serve --api-workers 4
```

| Deployment Size | API Workers | Scan Workers |
|----------------|------------|-------------|
| Development | 1 | 2 |
| Small team | 2 | 4 |
| Enterprise | 4 | 8-16 |
| Large scale | 4-8 | 16-32 (distributed) |

### Scan Cache

Enable scan caching to skip re-scanning unchanged files:

```bash
hades-enhanced --serve --scan-cache --cache-ttl 3600 --cache-max-size 10000
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--scan-cache` | disabled | Enable SHA-256 based result caching |
| `--cache-ttl` | 3600 | Cache entry lifetime in seconds |
| `--cache-max-size` | 10000 | Maximum cached entries (LRU eviction) |

With Redis enabled, cache is shared across all API instances.

### PostgreSQL Tuning

For enterprise deployments using PostgreSQL:

```ini
# postgresql.conf recommendations for HADES workloads
shared_buffers = 256MB          # 25% of available RAM
effective_cache_size = 768MB    # 75% of available RAM
work_mem = 16MB
maintenance_work_mem = 128MB
max_connections = 200
checkpoint_completion_target = 0.9
wal_buffers = 16MB
```

Connection pooling with PgBouncer (recommended for > 50 concurrent connections):

```bash
# docker-compose.ha.yml includes PgBouncer preconfigured
docker compose -f docker/docker-compose.ha.yml up -d
```

### Redis Configuration

Redis serves as scan cache backend, rate limiter, and task queue:

```
maxmemory 512mb
maxmemory-policy allkeys-lru
appendonly yes
```

For high-availability, use Redis Sentinel (included in `docker-compose.ha.yml`).

## Deployment Patterns

### Single Server

```bash
pip install "hades-scanner[full]"

hades-enhanced --serve \
    --host 0.0.0.0 \
    --port 8666 \
    --api-workers 2 \
    --workers 4 \
    --scan-cache
```

### Small Team

```bash
docker compose -f docker/docker-compose.scale.yml up -d
```

Uses PostgreSQL, Redis, and 2 worker replicas.

### Enterprise

```bash
# Scale workers based on load
docker compose -f docker/docker-compose.scale.yml up -d --scale hades-worker=6

# Environment variables
export HADES_WORKERS=4
export HADES_WORKER_REPLICAS=6
export HADES_ASYNC_PIPELINE=true
export HADES_CACHE_ENABLED=true
```

Uses a dedicated PostgreSQL host.

### Large Enterprise

```bash
docker compose -f docker/docker-compose.ha.yml up -d \
    --scale hades-api=4 \
    --scale hades-worker=12
```

Full HA deployment with nginx load balancer, PgBouncer, Redis Sentinel, on multiple hosts behind a load balancer with dedicated database and Redis clusters.

### Kubernetes (Horizontal Pod Autoscaling)

Planned for Kubernetes-native deployments (roadmap): a Helm chart and Kustomize overlays with HPA, ServiceMonitor, and ingress support:

```bash
# Helm install with autoscaling
helm install hades kubernetes/helm/hades \
  --set api.replicas=3 \
  --set autoscaling.api.enabled=true \
  --set autoscaling.worker.enabled=true \
  --set redis.enabled=true

# Kustomize production overlay
kubectl apply -k kubernetes/kustomize/overlays/production
```

See the **[Kubernetes Guide](kubernetes_guide.md)** for values reference, secrets management, HPA tuning, and resource sizing.

## Benchmarking

### Running Benchmarks

```bash
# Full benchmark suite
hades-benchmark

# Quick benchmark (faster, fewer iterations)
hades-benchmark --quick

# Standalone
python core/benchmarks.py
python core/benchmarks.py --quick
```

### Interpreting Results

The benchmark suite measures:

| Metric | Description |
|--------|-------------|
| Single-file latency | Time to scan one file end-to-end |
| Batch throughput | Sustained scanning rate |
| Memory per scan | Peak RSS during single scan |
| ML feature extraction | Feature vector generation time |
| Cache hit latency | Cached result retrieval time |
| API response (health) | Health endpoint round-trip |

HADES publishes no target or reference values for these metrics; results depend on hardware and file mix, so compare runs on your own systems.

### Saving Benchmark Results

```bash
# Save results to JSON
python core/benchmarks.py --output results.json

# Compare across runs
python core/benchmarks.py --output results_after.json
```

## Profiling

### Enabling the Profiler

```bash
# Profile a scan directory
hades-enhanced -r --profile /path/to/files/

# Profile the API server (writes profile on shutdown)
hades-enhanced --serve --profile
```

### Reading Profiler Output

The profiler generates a report showing:

- **Bottleneck functions:** Top 20 functions by cumulative time
- **Stage breakdown:** Time spent in metadata extraction, YARA matching, heuristic analysis, ML detection
- **I/O wait:** Time waiting on file reads, database writes, network calls

Look for:
- Functions consuming > 10% of total time
- Unexpected I/O waits (may indicate slow disk or network)
- Memory allocation hotspots

## Monitoring

### Health Check Endpoint

```bash
curl http://localhost:8666/api/v1/health
```

Returns:

```json
{
    "status": "healthy",
    "version": "0.7.1",
    "uptime_seconds": 3600,
    "scans_completed": 12500,
    "engine": "ready",
    "yara_rules_loaded": 144,
    "plugins_loaded": 5
}
```

### CLI Health Check Probe

For Kubernetes liveness/readiness probes or external monitoring scripts:

```bash
hades-enhanced --health-check
# Exits with code 0 (healthy) or 1 (unhealthy)
```

### Worker Status

```bash
curl -H "X-API-Key: YOUR_KEY" http://localhost:8666/api/v1/workers/status
```

### Prometheus Metrics and Grafana

HADES includes a Prometheus metrics exporter with metrics covering scans, findings, cache, workers, API requests, and pipeline stage timing. When `prometheus_client` is installed, the `/metrics` endpoint is automatically available.

```bash
# Install observability support
pip install "hades-scanner[observability]"

# Enable metrics on the API server
hades-enhanced --serve --metrics

# Prometheus scrape endpoint
curl http://localhost:8666/metrics

# JSON metrics summary
curl http://localhost:8666/api/v1/metrics/json

# Active alerts
curl http://localhost:8666/api/v1/metrics/alerts
```

Four pre-built Grafana dashboards are included (`grafana/dashboards/`):

- **HADES Overview** -- scan throughput, latency, threat distribution
- **HADES Pipeline** -- per-stage timing, bottleneck identification
- **HADES Workers** -- worker pool status, queue depth, task rates
- **HADES Infrastructure** -- API metrics, cache hit/miss ratio, auth activity

Deploy the full observability stack (Prometheus + Grafana + Alertmanager):

```bash
docker compose -f docker/docker-compose.observability.yml up -d
```

For detailed setup, PromQL examples, and alerting configuration, see the **[Observability Guide](observability_guide.md)**.

### Key Metrics to Monitor

| Metric | Alert Threshold | Description |
|--------|----------------|-------------|
| Health endpoint response | > 5s | Server may be overloaded |
| Scan queue depth | > 1000 | Add more workers |
| Scan latency p95 | > 2s | Check ExifTool process, disk I/O |
| Memory usage | > 80% limit | Reduce workers or increase limits |
| PostgreSQL connections | > 80% pool | Increase pool size or add PgBouncer |
| Redis memory | > 80% maxmemory | Increase limit or reduce cache TTL |
| Error rate | > 1% | Check logs for failing scans |

### Docker Health Checks

All Docker Compose configurations include health checks. Monitor with:

```bash
# Check service health
docker compose -f docker/docker-compose.scale.yml ps

# View logs
docker compose -f docker/docker-compose.scale.yml logs -f hades-api

# Resource usage
docker stats
```

## Troubleshooting

### Slow Scan Performance

1. **Check ExifTool:** Ensure ExifTool is installed and on PATH. A missing ExifTool falls back to slower pure-Python extraction.
2. **Enable caching:** `--scan-cache` avoids re-scanning unchanged files.
3. **Increase workers:** `--workers 8` for CPU-bound workloads.
4. **Check disk I/O:** Scanning from network mounts is significantly slower than local SSD.
5. **Profile:** Run with `--profile` to identify the bottleneck stage.

### High Memory Usage

1. **Reduce workers:** Each worker consumes memory independently.
2. **Check ML models:** ML ensemble loads models into memory. Disable with `--disable-ml` if not needed.
3. **Reduce cache size:** `--cache-max-size 5000` limits in-memory cache.
4. **Use Redis:** External Redis moves cache out of process memory.

### Database Connection Errors

1. **Check PostgreSQL status:** `pg_isready -h localhost -U hades`
2. **Check connection pool:** Monitor active connections vs. `max_connections`.
3. **Use PgBouncer:** Connection pooling reduces connection overhead.
4. **Check DSN:** Verify `HADES_DB_DSN` format: `postgresql://user:pass@host:5432/dbname`

### Redis Connection Errors

1. **Check Redis status:** `redis-cli ping`
2. **Check memory:** `redis-cli info memory` -- if maxmemory is hit, keys are evicted.
3. **Fallback:** HADES falls back to in-memory state when Redis is unavailable. No data loss, but state is not shared across instances.

### Worker Not Processing Tasks

1. **Check Redis queue:** `redis-cli llen hades:tasks`
2. **Check worker logs:** `docker logs hades-worker-1`
3. **Verify HADES_MODE:** Worker containers must have `HADES_MODE=worker` set.
4. **Check connectivity:** Workers need access to both Redis and PostgreSQL.
