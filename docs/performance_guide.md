# HADES Performance Guide

> **ROADMAP, not a shipped deployment option (updated 2026-09-27).** HADES currently ships as a single self-contained binary (the `hades` CLI and `hades-server`). Containerized and distributed deployments (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap. This guide describes that planned architecture. **No throughput, scans-per-day or hardware-sizing figure in it is a measured HADES result**, and none is given; size a deployment by benchmarking on your own hardware and file mix.
>
> References below to `pip install "hades-scanner[full]"` or `pip install "hades-scanner[observability]"` do not apply to the shipped binary, which bundles its dependencies inline. See [Installation Guide](installation.md) for current install paths.

This guide covers performance tuning, deployment patterns, benchmarking, and monitoring for HADES metadata forensics engine.

## Architecture Overview

### Synchronous vs Asynchronous Pipeline

HADES supports two scanning modes:

- **Synchronous (default):** Each scan request processes sequentially through metadata extraction, YARA matching, heuristic analysis, and ML detection. Suitable for single-server deployments handling moderate volumes.
- **Asynchronous pipeline:** Scan tasks are submitted to a queue and processed by background workers. The API returns immediately with a task ID. Workers pull tasks from Redis and write results to the database. Suitable for high-throughput environments.

Enable the async pipeline:

```bash
# Command-line scans
hades scan -r --async --concurrency 8 /evidence/

# Docker deployment (API server and workers)
HADES_ASYNC_PIPELINE=true
```

The `hades serve` command has no async option; in the Docker deployment the container entrypoint turns the pipeline on from `HADES_ASYNC_PIPELINE`.

### ExifTool Persistent Process

HADES uses PyExifTool to maintain a persistent ExifTool subprocess, avoiding the overhead of spawning a new process per file. The persistent process handles metadata extraction for all scan requests.

### Worker Pool

`hades serve --api-workers` sets the number of API (Uvicorn) worker processes:

```bash
hades serve --api-workers 2
```

`hades scan --workers` is accepted but reserved in v1.7.4: command-line scanning is single-threaded, and the value does not yet parallelise anything. For concurrent command-line scans use `--async --concurrency N`.

For distributed deployments, run dedicated worker containers:

```bash
# API server side: enable the distributed worker pool (requires Redis)
hades serve --distributed --redis-url redis://redis:6379/0

# Worker containers (Docker deployment): the entrypoint starts worker mode
HADES_MODE=worker
```

There is no worker-only command in the `hades` binary; worker processes run in the Docker deployment.

## Tuning Parameters

### API Server Workers

The `--api-workers` flag sets the number of uvicorn worker processes:

```bash
# Production: 2-4 workers per CPU core
hades serve --api-workers 4
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
# Command-line scans
hades scan -r --cache /evidence/

# Docker deployment (API server and workers)
HADES_CACHE_ENABLED=true
```

Caching is keyed on the file's SHA-256. The cache lifetime and size are not configurable from the command line in v1.7.4.

With Redis enabled, cache is shared across all API instances.

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

hades serve \
    --host 0.0.0.0 \
    --port 8666 \
    --api-workers 2
```

### Small Team

```bash
docker compose -f docker/docker-compose.scale.yml up -d
```

Uses Redis and 2 worker replicas.

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

Each instance keeps its own SQLite database.

### Large Enterprise

```bash
docker compose -f docker/docker-compose.ha.yml up -d \
    --scale hades-api=4 \
    --scale hades-worker=12
```

Full HA deployment with nginx load balancer and Redis Sentinel, on multiple hosts behind a load balancer with a dedicated Redis cluster.

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
hades benchmark --full

# Quick benchmark (faster, fewer iterations)
hades benchmark --quick
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
hades benchmark --quick --output results.json

# Compare across runs
hades benchmark --quick --output results_after.json
```

## Profiling

### Enabling the Profiler

```bash
# Profile a scan directory
hades scan -r --profile /path/to/files/

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
hades serve --health-check
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
hades serve --metrics

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
2. **ML model:** the signed classifier is loaded once per process; it is small next to per-file parsing and is not a tuning lever.
3. **Reduce cache size:** `--cache-max-size 5000` limits in-memory cache.
4. **Use Redis:** External Redis moves cache out of process memory.

### Database Connection Errors

SQLite is the only database backend (PostgreSQL was removed in v1.7.3).

1. **Check the health check:** `hades serve --health-check` reports any leftover setting from the removed PostgreSQL backend (`HADES_DB_BACKEND`, `HADES_DB_DSN`); remove it.
2. **Check the database path:** the process needs write access to the SQLite file and its directory (WAL mode creates `-wal` and `-shm` files beside it).
3. **One writer per file:** do not point several hosts at one SQLite file on network storage.

### Redis Connection Errors

1. **Check Redis status:** `redis-cli ping`
2. **Check memory:** `redis-cli info memory` -- if maxmemory is hit, keys are evicted.
3. **Fallback:** HADES falls back to in-memory state when Redis is unavailable. No data loss, but state is not shared across instances.

### Worker Not Processing Tasks

1. **Check Redis queue:** `redis-cli llen hades:tasks`
2. **Check worker logs:** `docker logs hades-worker-1`
3. **Verify HADES_MODE:** Worker containers must have `HADES_MODE=worker` set.
4. **Check connectivity:** Workers need access to Redis.
