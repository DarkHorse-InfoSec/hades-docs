# HADES v0.7.0 -- Performance & Scale

**Release Date:** 2026-02-21

HADES v0.7.0 is the performance release, delivering an async scan pipeline, distributed worker pool, scan result caching, a pipeline profiler, and Docker scaling infrastructure. Deployments can now scale from a single-process CLI to horizontally distributed clusters with Redis coordination, nginx load balancing, and dedicated worker containers.

All new features are opt-in. Existing v0.6.0 configurations and workflows continue to work without modification.

---

## What's New

### Async Scan Pipeline

A new non-blocking scan pipeline (`core/async_scanner.py`) enables concurrent file scanning with persistent ExifTool subprocesses. Instead of spawning a new ExifTool process per file, the pipeline maintains a pool of long-lived ExifTool processes that are reused across scans, eliminating per-file process startup overhead.

```bash
# Enable async scanning
hades-enhanced --async -r /evidence/

# Control concurrency (default: CPU count)
hades-enhanced --async --concurrency 8 -r /evidence/
```

Key capabilities:
- Configurable concurrency level
- Persistent ExifTool process pool for reduced spawn overhead
- Graceful shutdown with in-flight scan completion
- Falls back to synchronous mode when ExifTool is unavailable

### Distributed Worker Pool

A multi-process worker pool (`core/worker_pool.py`) distributes scan tasks across processes, with optional Redis-backed task queues for cross-machine distribution.

```bash
# Local multi-worker scanning
hades-enhanced --workers 4 -r /evidence/

# Distributed mode: API server submits to Redis queue
hades-enhanced --serve --distributed --redis-url redis://localhost:6379

# Dedicated worker container
hades-enhanced --worker --redis-url redis://localhost:6379
```

Key capabilities:
- Multi-process workers with configurable pool size
- Redis task queue for distributed scanning across nodes
- Heartbeat monitoring with automatic stale worker detection
- Dead letter queue for failed tasks with retry tracking
- Graceful shutdown with configurable drain timeout

### High-Performance File Ingestion

A file ingestion engine (`core/file_ingestion.py`) handles recursive directory scanning with archive extraction, deduplication, and progress tracking.

```bash
# Ingest a directory with archive extraction
hades-enhanced --async -r /evidence/collected/
```

Key capabilities:
- Archive extraction: ZIP, tar (.tar, .tar.gz, .tar.bz2), 7-Zip (requires py7zr)
- Content-hash deduplication -- identical files are scanned once regardless of filename
- Resumable ingestion with progress tracking
- File type filtering by extension
- Priority queuing for suspected threat files

### Two-Tier Scan Cache

A content-addressed scan cache (`core/scan_cache.py`) avoids redundant work by returning cached results for previously scanned file content.

```bash
# Enable caching
hades-enhanced --cache -r /evidence/

# Disable caching for a fresh scan
hades-enhanced --no-cache -r /evidence/
```

Key capabilities:
- **Tier 1**: In-memory LRU cache (always available, zero-config)
- **Tier 2**: Redis-backed cache (shared across processes and containers)
- Content-hash keyed -- file renames don't cause cache misses, but content changes do
- Smart invalidation on file modification (mtime + size check)
- Configurable TTL and maximum cache size

### Pipeline Profiler

A profiler (`core/profiler.py`) instruments each stage of the scan pipeline and generates bottleneck analysis reports.

```bash
# Profile a scan
hades-enhanced --profile -r /evidence/
```

Key capabilities:
- Per-stage timing: metadata extraction, YARA matching, heuristic analysis, ML detection, deep format analysis
- Bottleneck identification with tuning recommendations
- Trace export for external analysis
- Aggregated statistics across scan batches

### Enhanced Benchmark Suite

The benchmark suite (`core/benchmarks.py`) adds scale testing with configurable corpus sizes and a corpus generator (`benchmarks/generate_test_corpus.py`).

```bash
# Run scale benchmarks
hades-enhanced --benchmark-scale

# Standard benchmarks
hades-enhanced --benchmark
```

Key capabilities:
- Configurable corpus generation (file count, formats, threat density)
- Throughput, latency, and memory profiling
- Cache hit/miss performance measurement
- Worker pool scaling efficiency analysis

### Docker Compose Scaling

Two new Docker Compose configurations enable horizontal scaling:

**Scale deployment** (`docker/docker-compose.scale.yml`):
```bash
# Start with 4 worker replicas
docker compose -f docker/docker-compose.scale.yml up -d --scale hades-worker=4
```
- PostgreSQL + Redis backing services
- Nginx load balancer with WebSocket support
- Scalable API server and worker containers

**High-availability deployment** (`docker/docker-compose.ha.yml`):
```bash
docker compose -f docker/docker-compose.ha.yml up -d
```
- Nginx reverse proxy with health-check-based failover
- Multiple API server instances
- Dedicated worker containers with Redis coordination

### Performance & Scaling Documentation

Two new documentation guides:
- **[Performance Tuning Guide](docs/performance_guide.md)** -- configuration, profiling, bottleneck analysis, deployment patterns, monitoring
- **[Scaling Guide](docs/scaling_guide.md)** -- horizontal scaling, database scaling, load balancing, Docker Compose deployments, capacity planning

---

## Migration Guide from v0.6.0

### New Configuration Sections

The following sections have been added to `cli/hades_config.json`. All are optional and have sensible defaults:

| Section | Purpose |
|---------|---------|
| `distributed` | Redis task queue URL, heartbeat interval, task timeout, retry count |

### New Optional Dependencies

```bash
# Install performance extras
pip install "hades-scanner[performance]"
```

| Package | Purpose | Required for |
|---------|---------|-------------|
| `py7zr` | 7-Zip archive extraction | Archive ingestion |
| `uvloop` | Fast asyncio event loop (Linux/macOS only) | Async pipeline performance |

### New CLI Flags

| Flag | Description |
|------|-------------|
| `--async` | Enable async scan pipeline |
| `--concurrency N` | Set async pipeline concurrency (default: CPU count) |
| `--workers N` | Start N local scan workers |
| `--distributed` | Enable Redis-backed distributed task queue |
| `--worker` | Run as a dedicated scan worker (no API server) |
| `--cache` | Enable scan result caching |
| `--no-cache` | Disable scan result caching |
| `--profile` | Enable pipeline profiling |
| `--benchmark-scale` | Run scale benchmarks with corpus generation |

### Backward Compatibility

- All new features are **opt-in** via CLI flags or environment variables
- Existing CLI commands, API endpoints, and configuration files work unchanged
- The default single-process synchronous scan mode is preserved
- Docker Compose `docker-compose.yml` continues to work as before
- No database migrations required

---

## Performance Expectations

| Deployment Pattern | CLI Flag | Expected Throughput | Best For |
|-------------------|----------|-------------------|----------|
| Single-process (default) | *(none)* | Baseline | Small-scale, ad-hoc scanning |
| Async pipeline | `--async` | 2-4x baseline for I/O-bound workloads | Medium directories, mixed file types |
| Multi-worker | `--workers N` | Near-linear scaling with CPU cores | CPU-bound analysis, large directories |
| Distributed | `--distributed` | Horizontal scaling across nodes | Enterprise, multi-node clusters |
| Cached | `--cache` | Near-instant for repeat scans | Monitoring, incremental scanning |

Throughput gains vary with file types, analysis depth, and hardware. Use `--profile` to identify bottlenecks and `--benchmark-scale` to measure your specific workload.

---

## Requirements

- Python 3.8+
- Optional: ExifTool (for persistent process pool in async mode)
- Optional: Redis (for distributed mode, cache tier 2, and task queues)
- Optional: py7zr (for 7-Zip archive extraction)
- Optional: Docker + Docker Compose (for scaled deployments)

---

## Test Suite

1,500+ tests passing across all modules. The v0.7.0 performance phase adds 191 new tests covering async scanning, worker pool distribution, file ingestion, caching, profiling, benchmarks, Docker configuration, and integration.

---

## Full Changelog

See [CHANGELOG.md](CHANGELOG.md) for the complete list of changes.

---

DarkHorse Information Security LLC
