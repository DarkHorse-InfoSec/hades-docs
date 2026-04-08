# HADES Troubleshooting Guide
Version: 1.0.0

Author: DarkHorse Information Security LLC

This guide covers common issues encountered when installing, configuring, and operating HADES. Each entry provides the symptom, likely root cause, and resolution steps.

---

## Table of Contents

- [1. Installation Issues](#1-installation-issues)
  - [1.1 ExifTool Not Found](#11-exiftool-not-found)
  - [1.2 YARA Python Compilation Failure](#12-yara-python-compilation-failure)
  - [1.3 YARA Rule Load Errors](#13-yara-rule-load-errors)
  - [1.4 scikit-learn Installation Failure](#14-scikit-learn-installation-failure)
  - [1.5 pikepdf Build Failure](#15-pikepdf-build-failure)
  - [1.6 olefile Import Error](#16-olefile-import-error)
  - [1.7 bcrypt or cryptography Wheel Build Failure](#17-bcrypt-or-cryptography-wheel-build-failure)
  - [1.8 psycopg2-binary Platform Error](#18-psycopg2-binary-platform-error)
  - [1.9 uvloop Not Available on Windows](#19-uvloop-not-available-on-windows)
  - [1.10 py7zr Missing for Archive Extraction](#110-py7zr-missing-for-archive-extraction)
- [2. API Server Issues](#2-api-server-issues)
  - [2.1 Port Already in Use](#21-port-already-in-use)
  - [2.2 API Key Authentication Failures](#22-api-key-authentication-failures)
  - [2.3 CORS Errors in Browser](#23-cors-errors-in-browser)
  - [2.4 WebSocket Connection Drops](#24-websocket-connection-drops)
  - [2.5 422 Validation Error on Upload](#25-422-validation-error-on-upload)
  - [2.6 Server Fails to Start — Module Import Error](#26-server-fails-to-start--module-import-error)
- [3. Scanning Issues](#3-scanning-issues)
  - [3.1 File Too Large — Memory Exhaustion](#31-file-too-large--memory-exhaustion)
  - [3.2 Unsupported File Type Returns Empty Results](#32-unsupported-file-type-returns-empty-results)
  - [3.3 ExifTool Process Hangs](#33-exiftool-process-hangs)
  - [3.4 ExifTool Process Crashes Mid-Scan](#34-exiftool-process-crashes-mid-scan)
  - [3.5 Scan Cache Returns Stale Results](#35-scan-cache-returns-stale-results)
  - [3.6 Async Pipeline Deadlock](#36-async-pipeline-deadlock)
- [4. Docker Issues](#4-docker-issues)
  - [4.1 Build Fails at yara-python Wheel](#41-build-fails-at-yara-python-wheel)
  - [4.2 Health Check Fails After Container Start](#42-health-check-fails-after-container-start)
  - [4.3 Volume Mount Permission Denied](#43-volume-mount-permission-denied)
  - [4.4 Custom YARA Rules Not Loaded](#44-custom-yara-rules-not-loaded)
  - [4.5 Worker Container Cannot Reach Redis](#45-worker-container-cannot-reach-redis)
  - [4.6 Container OOM Killed](#46-container-oom-killed)
- [5. Enterprise Feature Issues](#5-enterprise-feature-issues)
  - [5.1 PostgreSQL Connection Refused](#51-postgresql-connection-refused)
  - [5.2 Redis Fallback to In-Memory](#52-redis-fallback-to-in-memory)
  - [5.3 Encryption Key Not Found](#53-encryption-key-not-found)
  - [5.4 RBAC — Permission Denied After User Creation](#54-rbac--permission-denied-after-user-creation)
  - [5.5 SSO Callback Fails with Invalid Token](#55-sso-callback-fails-with-invalid-token)
  - [5.6 License Key Rejected](#56-license-key-rejected)
  - [5.7 Multi-Tenant Isolation Leak](#57-multi-tenant-isolation-leak)
- [6. ML Detection Issues](#6-ml-detection-issues)
  - [6.1 Not Enough Training Samples](#61-not-enough-training-samples)
  - [6.2 Model File Not Found](#62-model-file-not-found)
  - [6.3 XGBoost Unavailable — Ensemble Fallback](#63-xgboost-unavailable--ensemble-fallback)
  - [6.4 Feature Extraction Returns NaN](#64-feature-extraction-returns-nan)
  - [6.5 High False Positive Rate After Training](#65-high-false-positive-rate-after-training)
  - [6.6 Model Incompatible After scikit-learn Upgrade](#66-model-incompatible-after-scikit-learn-upgrade)

---

## 1. Installation Issues

### 1.1 ExifTool Not Found

**Symptom:** `ExifTool not found` error at startup, or scan results return no metadata fields.

**Cause:** ExifTool is a Perl-based external binary that must be installed separately from the Python package. The `PyExifTool` wrapper expects the `exiftool` binary on `PATH`.

**Solution:**

```bash
# Linux (Debian/Ubuntu)
sudo apt-get install libimage-exiftool-perl

# macOS
brew install exiftool

# Windows — download from https://exiftool.org
# Rename exiftool(-k).exe to exiftool.exe and place it on PATH
```

Verify installation:

```bash
exiftool -ver
```

If the binary is installed but not on `PATH`, set the full path in your environment:

```bash
export EXIFTOOL_PATH=/usr/local/bin/exiftool
```

The persistent ExifTool process pool in `async_scanner.py` also requires this binary. If unavailable, the pipeline falls back to per-invocation mode with degraded throughput.

---

### 1.2 YARA Python Compilation Failure

**Symptom:** `pip install yara-python` fails with C compiler errors.

**Cause:** `yara-python` compiles native C extensions. Missing build tools or YARA C library headers will cause the build to fail.

**Solution:**

```bash
# Linux (Debian/Ubuntu)
sudo apt-get install build-essential libssl-dev libmagic-dev

# macOS
xcode-select --install
brew install openssl

# Windows — install Visual C++ Build Tools
# https://visualstudio.microsoft.com/visual-cpp-build-tools/
```

If compilation continues to fail, install a pre-built wheel:

```bash
pip install yara-python --only-binary=:all:
```

YARA is a core dependency but HADES degrades gracefully without it. The `YARA_AVAILABLE` flag is checked before any YARA operations. Pattern-based detection rules will be skipped, but heuristic and ML detection still function.

---

### 1.3 YARA Rule Load Errors

**Symptom:** `Error compiling YARA rules` or `syntax error` referencing a `.yar` file.

**Cause:** A YARA rule file contains syntax errors, duplicate rule names, or references undefined modules.

**Solution:**

Validate all rules before deploying:

```bash
python scripts/validate_rules.py
```

For strict CI-mode validation (non-zero exit on failure):

```bash
python scripts/validate_rules_ci.py
```

Common issues:
- Duplicate rule identifiers across files: each rule name must be globally unique.
- Missing `meta:` section: the HADES convention requires `author`, `description`, and `severity` metadata fields.
- Unsupported YARA module references: ensure the YARA installation includes the modules your rules reference (e.g., `pe`, `math`, `hash`).

---

### 1.4 scikit-learn Installation Failure

**Symptom:** `pip install scikit-learn` fails or hangs during build, especially on ARM or minimal containers.

**Cause:** scikit-learn requires NumPy and SciPy, which compile Fortran/C extensions. Minimal environments may lack BLAS/LAPACK libraries.

**Solution:**

```bash
# Linux — install BLAS/LAPACK first
sudo apt-get install libopenblas-dev liblapack-dev gfortran

# Use pre-built wheels (preferred)
pip install scikit-learn --only-binary=:all:
```

If the install environment cannot support scikit-learn (e.g., restricted build hosts), HADES operates without ML detection. The `SKLEARN_AVAILABLE` flag disables all ML paths. Install the base package without ML:

```bash
pip install hades-scanner
# ML features will report as unavailable; all other detection works
```

---

### 1.5 pikepdf Build Failure

**Symptom:** `pip install pikepdf` fails with QPDF-related errors.

**Cause:** pikepdf wraps the QPDF C++ library. Missing QPDF headers or an incompatible C++ compiler will cause the build to fail.

**Solution:**

```bash
# Linux (Debian/Ubuntu)
sudo apt-get install libqpdf-dev

# macOS
brew install qpdf
```

pikepdf is optional. Without it, PDF analysis falls back to regex-based parsing. Deep format analysis for PDF JavaScript, OpenAction, and embedded file detection still works, with slightly lower fidelity for obfuscated content.

Install with integrations extra to include pikepdf:

```bash
pip install "hades-scanner[integrations]"
```

---

### 1.6 olefile Import Error

**Symptom:** `ImportError: No module named 'olefile'` in logs during Office document scanning.

**Cause:** olefile is an optional dependency for OLE2 compound document parsing (legacy .doc, .xls, .ppt formats).

**Solution:**

```bash
pip install olefile
```

Or install with integrations:

```bash
pip install "hades-scanner[integrations]"
```

Without olefile, OLE-specific checks (embedded OLE objects, legacy macro streams) are skipped. OOXML (.docx, .xlsx) analysis via ZIP parsing remains fully functional.

---

### 1.7 bcrypt or cryptography Wheel Build Failure

**Symptom:** Build errors when installing `bcrypt` or `cryptography` packages.

**Cause:** Both packages contain Rust and/or C extensions. Older pip versions or missing Rust toolchain can cause failures.

**Solution:**

```bash
# Ensure pip is current
pip install --upgrade pip setuptools wheel

# Install Rust if needed (cryptography >= 38 requires it)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Prefer binary wheels
pip install bcrypt cryptography --only-binary=:all:
```

These are enterprise-tier dependencies. If not needed, the base installation works without them:

```bash
pip install hades-scanner
# RBAC and encryption features will be unavailable
```

---

### 1.8 psycopg2-binary Platform Error

**Symptom:** `psycopg2-binary` fails to install on Alpine Linux or ARM architectures.

**Cause:** The `-binary` variant ships pre-compiled wheels for common platforms only.

**Solution:**

For Alpine, install the source package with system dependencies:

```bash
apk add postgresql-dev gcc musl-dev
pip install psycopg2
```

For ARM (e.g., Raspberry Pi, Apple Silicon):

```bash
pip install psycopg2-binary --only-binary=:all:
# If no wheel exists:
brew install libpq  # macOS ARM
pip install psycopg2 --no-binary=psycopg2
```

PostgreSQL is optional. Without it, HADES uses SQLite (WAL mode) as the database backend.

---

### 1.9 uvloop Not Available on Windows

**Symptom:** Warning about uvloop at startup on Windows.

**Cause:** uvloop is a Unix-only (Linux/macOS) asyncio event loop replacement. It is excluded from Windows installs via the `sys_platform != 'win32'` marker in `pyproject.toml`.

**Solution:** No action required. HADES uses the default asyncio event loop on Windows. This is not a bug. Performance on Windows is adequate for most workloads; for high-throughput deployments, use Linux.

---

### 1.10 py7zr Missing for Archive Extraction

**Symptom:** 7-Zip archives (`.7z`) are skipped during file ingestion.

**Cause:** The `py7zr` package is an optional dependency for 7z extraction. ZIP and tar archives are handled by the Python standard library and always work.

**Solution:**

```bash
pip install py7zr
```

Or install with enterprise extras:

```bash
pip install "hades-scanner[enterprise]"
```

---

## 2. API Server Issues

### 2.1 Port Already in Use

**Symptom:** `[Errno 98] Address already in use` or `[WinError 10048]` when starting the API server.

**Cause:** Another process (or a previous HADES instance) is bound to port 8666.

**Solution:**

Identify the process:

```bash
# Linux/macOS
lsof -i :8666

# Windows
netstat -ano | findstr :8666
```

Kill the conflicting process or use a different port:

```bash
python core/hades_enhanced_cli.py --serve --port 9000
```

In Docker, remap the host port:

```bash
docker run -p 9000:8666 hades-scanner:1.0.0
```

Or set via environment variable:

```bash
HADES_PORT=9000 docker compose -f docker/docker-compose.yml up -d
```

---

### 2.2 API Key Authentication Failures

**Symptom:** HTTP 401 Unauthorized on all API requests; `X-API-Key` header rejected.

**Cause:** The `HADES_API_KEY` environment variable or configuration value does not match the key sent in the `X-API-Key` request header. If no API key is configured, authentication may be disabled entirely (development mode).

**Solution:**

1. Verify the configured key:

```bash
# Check if the env var is set
echo $HADES_API_KEY
```

2. Send the key in your request:

```bash
curl -H "X-API-Key: your-key-here" http://localhost:8666/api/v1/health
```

3. If using RBAC, ensure the API key is associated with a valid user. Rotate a compromised key:

```bash
python core/hades_enhanced_cli.py --create-admin
# This generates a new admin user with API key
```

4. In Docker, pass the key via environment:

```bash
docker run -e HADES_API_KEY=your-key hades-scanner:1.0.0
```

---

### 2.3 CORS Errors in Browser

**Symptom:** Browser console shows `Access-Control-Allow-Origin` errors when calling the API from a web frontend.

**Cause:** The FastAPI CORS middleware restricts origins. By default, the API may not include your frontend's origin.

**Solution:**

HADES configures `CORSMiddleware` in `hades_api.py`. If you are accessing the API from a custom frontend origin, set the allowed origins via environment or configuration.

For development, the API allows `*` origins. For production, restrict to your domain. If the web dashboard at `/dashboard/` works but external origins do not, add your origin to the CORS configuration.

When using a reverse proxy (nginx), ensure the proxy passes CORS headers through and does not strip them:

```nginx
location /api/ {
    proxy_pass http://hades-api:8666;
    proxy_set_header Host $host;
    proxy_set_header Origin $http_origin;
}
```

---

### 2.4 WebSocket Connection Drops

**Symptom:** WebSocket connection to `ws://localhost:8666/ws` connects then immediately closes, or fails with HTTP 403.

**Cause:**

- Missing `websockets` package.
- Reverse proxy not configured for WebSocket upgrade.
- API key not provided in WebSocket query parameters.

**Solution:**

1. Ensure the `websockets` package is installed:

```bash
pip install websockets
```

2. If behind nginx, add WebSocket upgrade support:

```nginx
location /ws {
    proxy_pass http://hades-api:8666;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;
}
```

3. Pass the API key as a query parameter:

```
ws://localhost:8666/ws?api_key=your-key
```

4. Check that no firewall or load balancer is terminating long-lived connections.

---

### 2.5 422 Validation Error on Upload

**Symptom:** File upload to `/api/v1/scan` returns HTTP 422 with a Pydantic validation error.

**Cause:** FastAPI uses Pydantic for request validation. The 422 status indicates the request body or multipart form data does not match the expected schema.

**Solution:**

Ensure the file is sent as multipart form data with the correct field name:

```bash
curl -X POST http://localhost:8666/api/v1/scan \
  -H "X-API-Key: your-key" \
  -F "file=@suspicious.jpg"
```

Common mistakes:
- Sending JSON body instead of multipart form data.
- Using the wrong field name (must be `file`).
- Missing `Content-Type: multipart/form-data` header (curl sets this automatically with `-F`).

---

### 2.6 Server Fails to Start — Module Import Error

**Symptom:** `ModuleNotFoundError` or `ImportError` when running `--serve`.

**Cause:** The Python path does not include the `core/` and `cli/` directories, or a required dependency is missing.

**Solution:**

1. Run from the project root directory:

```bash
cd /path/to/darkhorse-hades
python core/hades_enhanced_cli.py --serve
```

2. Verify the installation:

```bash
pip install -e ".[dev]"
```

3. Check which optional modules are available:

```bash
python -c "from core.hades_api import create_app; app = create_app()"
```

Import failures for optional modules (integrations, auth, storage) are logged as warnings but do not prevent startup. Only core modules (`enhanced_detection_engine`, `metadata_parser_engine`) are fatal.

---

## 3. Scanning Issues

### 3.1 File Too Large — Memory Exhaustion

**Symptom:** Process killed (OOM) or `MemoryError` when scanning large files (>500 MB).

**Cause:** HADES loads file content into memory for heuristic analysis, YARA matching, and feature extraction. Extremely large files exhaust available RAM.

**Solution:**

1. Set resource limits in Docker:

```yaml
deploy:
  resources:
    limits:
      memory: 4G
```

2. Use the file ingestion engine with size filtering:

```bash
python core/hades_enhanced_cli.py --max-file-size 100000000 -r /evidence/
```

3. For batch processing of large directories, enable the async pipeline with bounded concurrency:

```bash
python core/hades_enhanced_cli.py --async --concurrency 2 -r /large-evidence/
```

4. Enable the scan cache to avoid re-processing files:

```bash
python core/hades_enhanced_cli.py --cache -r /evidence/
```

---

### 3.2 Unsupported File Type Returns Empty Results

**Symptom:** Scan completes but reports zero findings for a file type you expect to be analyzed.

**Cause:** HADES analyzes metadata, not file content execution. Some file types have limited metadata (e.g., plain text files, raw binaries). The detection engine prioritizes images, documents, archives, and multimedia formats.

**Solution:**

1. Check supported types:

```bash
python core/hades_enhanced_cli.py --help
# See --file-types flag for filtering
```

2. For custom file types, write a detection plugin:

```python
# plugins/my_custom_detector.py
from core.plugin_api import HADESPlugin, Finding

class MyDetector(HADESPlugin):
    def supported_file_types(self):
        return ['.custom', '.bin']
    # ...
```

3. Verify ExifTool recognizes the format:

```bash
exiftool -j suspicious_file.xyz
```

If ExifTool returns no metadata, the file format genuinely has no extractable metadata.

---

### 3.3 ExifTool Process Hangs

**Symptom:** Scan appears to freeze indefinitely on a specific file. CPU usage drops to zero.

**Cause:** Certain malformed files can cause ExifTool to hang during metadata extraction (e.g., deeply nested containers, circular references, or multi-GB embedded data).

**Solution:**

1. The async pipeline (`AsyncScanPipeline`) uses timeouts on ExifTool invocations. Enable it:

```bash
python core/hades_enhanced_cli.py --async -r /evidence/
```

2. If using the basic CLI and a scan is stuck, kill the ExifTool process:

```bash
# Linux/macOS
pkill -f exiftool

# Windows
taskkill /F /IM exiftool.exe
```

3. Identify the problematic file and exclude it:

```bash
python core/hades_enhanced_cli.py -r /evidence/ --exclude "*.problematic"
```

4. Report the file to the ExifTool maintainer if it triggers a reproducible hang.

---

### 3.4 ExifTool Process Crashes Mid-Scan

**Symptom:** `BrokenPipeError` or `ExifTool process terminated unexpectedly` during batch scanning.

**Cause:** The persistent ExifTool process pool (`PersistentExifTool` in `async_scanner.py`) reuses long-running ExifTool subprocesses. A segfault or Perl error in ExifTool terminates the subprocess.

**Solution:**

1. The async pipeline automatically restarts crashed ExifTool processes. Verify it is enabled:

```bash
python core/hades_enhanced_cli.py --async -r /evidence/
```

2. If crashes are frequent, update ExifTool:

```bash
# macOS
brew upgrade exiftool

# Linux
sudo apt-get update && sudo apt-get install --only-upgrade libimage-exiftool-perl
```

3. As a fallback, disable persistent process mode (uses per-file invocations):

```bash
# Set environment variable
export HADES_EXIFTOOL_PERSISTENT=false
```

---

### 3.5 Scan Cache Returns Stale Results

**Symptom:** A file was modified but the scan returns old results.

**Cause:** The two-tier scan cache (`ScanCache`) keys entries by content hash. If the file content changes, the hash changes and the cache misses. However, if only file metadata (timestamps, permissions) changed, the content hash remains the same and the cached result is returned.

**Solution:**

1. Force a cache bypass for a specific scan:

```bash
python core/hades_enhanced_cli.py --no-cache /path/to/file.jpg
```

2. Clear the cache entirely:

```bash
# The cache invalidates automatically on file content changes.
# For manual reset, restart the server or clear Redis:
redis-cli FLUSHDB
```

3. Reduce cache TTL in configuration to expire entries faster.

---

### 3.6 Async Pipeline Deadlock

**Symptom:** Async scan starts but never completes. Worker count shows active tasks but no progress.

**Cause:** All async workers are blocked waiting for ExifTool responses on files that will never complete (e.g., hangs from section 3.3). The bounded semaphore prevents new tasks from starting.

**Solution:**

1. Reduce concurrency to isolate the issue:

```bash
python core/hades_enhanced_cli.py --async --concurrency 1 -r /evidence/
```

2. Enable profiling to identify the bottleneck:

```bash
python core/hades_enhanced_cli.py --async --profile -r /evidence/
```

3. Kill and restart. The pipeline supports graceful shutdown:

```bash
# Send SIGINT (Ctrl+C) — the pipeline drains active tasks and exits
```

---

## 4. Docker Issues

### 4.1 Build Fails at yara-python Wheel

**Symptom:** `docker build` fails during `pip install` with yara-python compilation errors.

**Cause:** The builder stage in `hades-full.dockerfile` uses `python:3.12-slim`, which includes `build-essential` but may miss specific header files.

**Solution:**

1. Ensure you are building from the project root (the Dockerfile expects context at `.`):

```bash
docker build -f docker/hades-full.dockerfile -t hades-scanner:1.0.0 .
```

2. If the build fails on a specific dependency, add the missing system package to the builder stage in `hades-full.dockerfile`:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libmagic1 \
    libmagic-dev \
    libssl-dev \    # Add if needed
    curl \
    && rm -rf /var/lib/apt/lists/*
```

3. For CI environments with limited resources, increase Docker memory:

```bash
docker build --memory=4g -f docker/hades-full.dockerfile -t hades-scanner:1.0.0 .
```

---

### 4.2 Health Check Fails After Container Start

**Symptom:** Container starts but is marked `unhealthy`. Logs show the server is running but `curl` health check fails.

**Cause:** The health check (`curl -f http://localhost:8666/api/v1/health`) runs before the uvicorn server finishes binding. The default `start_period` is 15 seconds, which may be insufficient for large rule sets or model loading.

**Solution:**

1. Increase the start period:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8666/api/v1/health"]
  interval: 30s
  timeout: 10s
  retries: 5
  start_period: 30s
```

2. Check container logs for startup errors:

```bash
docker logs hades-api
```

3. Verify the port matches:

```bash
docker exec hades-api curl -f http://localhost:${HADES_PORT:-8666}/api/v1/health
```

4. Ensure `curl` is installed in the runtime image. The `hades-full.dockerfile` includes it, but custom images may not.

---

### 4.3 Volume Mount Permission Denied

**Symptom:** `Permission denied` errors when reading/writing to `/data/scans`, `/data/rules`, or `/data/config`.

**Cause:** The HADES container runs as non-root user `hades` (UID 1000, GID 1000). Host-mounted directories may be owned by root or a different user.

**Solution:**

1. Set correct ownership on the host:

```bash
sudo chown -R 1000:1000 /host/path/to/scans
sudo chown -R 1000:1000 /host/path/to/rules
```

2. Or use named volumes (Docker manages permissions):

```yaml
volumes:
  scan-data:
  custom-rules:
```

3. On SELinux systems, add the `:z` or `:Z` suffix:

```yaml
volumes:
  - /host/scans:/data/scans:z
```

---

### 4.4 Custom YARA Rules Not Loaded

**Symptom:** Custom `.yar` files mounted into `/data/rules` are not detected during scans.

**Cause:** The entrypoint script copies rules from `/data/rules` to `/app/rules/` at startup. Rules added after container start are not picked up. The copy command also filters on `.yar` and `.yara` extensions only.

**Solution:**

1. Ensure rule files have `.yar` or `.yara` extension.

2. Restart the container after adding new rules:

```bash
docker restart hades-api
```

3. Verify rules are present inside the container:

```bash
docker exec hades-api ls -la /app/rules/
```

4. Validate rule syntax before mounting:

```bash
python scripts/validate_rules.py
```

---

### 4.5 Worker Container Cannot Reach Redis

**Symptom:** Worker containers log `Connection refused` to Redis. Tasks are not distributed.

**Cause:** The worker connects to the Redis URL specified by `HADES_REDIS_URL`. If Redis is in a different Docker network or not yet started, the connection fails.

**Solution:**

1. Verify Redis is running and on the same Docker network:

```bash
docker compose -f docker/docker-compose.scale.yml ps
```

2. Check the `HADES_REDIS_URL` value:

```bash
docker exec hades-worker-1 env | grep REDIS
```

3. Use the Docker service name as the hostname:

```yaml
environment:
  - HADES_REDIS_URL=redis://hades-redis:6379/0
```

4. If Redis requires authentication:

```yaml
- HADES_REDIS_URL=redis://:password@hades-redis:6379/0
```

Without Redis, the worker pool falls back to local multiprocessing (single-host mode).

---

### 4.6 Container OOM Killed

**Symptom:** Container exits with code 137. `dmesg` or Docker events show OOM kill.

**Cause:** The default memory limit in `docker-compose.yml` is 2 GB. Large YARA rule sets, ML models, or scanning many files concurrently can exceed this.

**Solution:**

1. Increase the memory limit:

```yaml
deploy:
  resources:
    limits:
      memory: 4G
```

2. Reduce API workers to lower per-process memory:

```yaml
environment:
  - HADES_WORKERS=1
```

3. Monitor memory with the Prometheus metrics stack:

```bash
docker compose -f docker/docker-compose.observability.yml up -d
# Check Grafana infrastructure dashboard for memory trends
```

---

## 5. Enterprise Feature Issues

### 5.1 PostgreSQL Connection Refused

**Symptom:** `psycopg2.OperationalError: connection refused` when using `--db-backend postgresql`.

**Cause:** PostgreSQL is unreachable. Common reasons: wrong DSN, PostgreSQL not started, firewall blocking port 5432, or the database does not exist.

**Solution:**

1. Verify PostgreSQL is running:

```bash
pg_isready -h localhost -p 5432
```

2. Verify the DSN:

```bash
python core/hades_enhanced_cli.py --serve \
  --db-backend postgresql \
  --db-dsn "postgresql://hades_user:password@localhost:5432/hades_db"
```

3. Create the database if it does not exist:

```sql
CREATE DATABASE hades_db;
CREATE USER hades_user WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE hades_db TO hades_user;
```

4. In Docker Compose, ensure the API service depends on PostgreSQL and waits for readiness:

```yaml
depends_on:
  hades-postgres:
    condition: service_healthy
```

5. If PostgreSQL is unavailable, HADES falls back to SQLite automatically. Check logs for `Falling back to SQLite backend`.

---

### 5.2 Redis Fallback to In-Memory

**Symptom:** Logs show `Redis unavailable, using in-memory fallback`. Rate limiting, sessions, and distributed locks operate in-process only.

**Cause:** The `redis` Python package is not installed, or the Redis server is unreachable.

**Solution:**

1. Install the Redis package:

```bash
pip install redis
```

2. Verify Redis connectivity:

```bash
redis-cli -h localhost -p 6379 ping
# Should return: PONG
```

3. Pass the URL to HADES:

```bash
python core/hades_enhanced_cli.py --serve --redis-url redis://localhost:6379/0
```

4. In-memory fallback is functional for single-instance deployments. Multi-instance deployments (Docker scaling, Kubernetes) require Redis for shared state. Without it, each instance maintains independent rate limits and sessions.

---

### 5.3 Encryption Key Not Found

**Symptom:** `FileNotFoundError: encryption.key not found` or `Encryption key not configured` when using `--encrypt`.

**Cause:** AES-256-GCM encryption requires a key file. The key must be generated before enabling encryption.

**Solution:**

1. Generate a new encryption key:

```bash
python core/hades_enhanced_cli.py --generate-encryption-key
```

This creates `config/encryption.key`. The file contains a 256-bit random key, base64-encoded.

2. In Docker, pass the key via environment variable:

```bash
docker run -e HADES_ENCRYPTION_KEY=$(cat config/encryption.key) hades-scanner:1.0.0
```

3. Critical: Never commit `encryption.key` to version control. Verify `.gitignore` includes:

```
*.key
config/encryption.key
```

4. Back up the key securely. Encrypted data is unrecoverable without it.

---

### 5.4 RBAC — Permission Denied After User Creation

**Symptom:** A newly created user receives HTTP 403 on endpoints they should have access to.

**Cause:** The user was created without the correct role, or the role's permission matrix does not include the requested endpoint.

**Solution:**

1. Verify the user's role:

```bash
python core/hades_enhanced_cli.py --list-users
```

2. The default roles and permissions are:
   - **admin**: Full access to all endpoints.
   - **analyst**: Scan, monitor, evidence, and reporting endpoints. No user management.
   - **viewer**: Read-only access to scan results and reports.

3. Reassign a role:

```bash
python core/hades_enhanced_cli.py --create-user analyst1 --user-role analyst
```

4. If using API key authentication, ensure the API key belongs to the correct user. Rotate if compromised:

```bash
# Via API endpoint
curl -X POST http://localhost:8666/api/v1/auth/rotate-key \
  -H "X-API-Key: admin-key"
```

---

### 5.5 SSO Callback Fails with Invalid Token

**Symptom:** OIDC or SAML callback returns `Invalid token` or `Signature verification failed`.

**Cause:** Clock skew between the HADES server and the identity provider, expired tokens, or misconfigured issuer/audience.

**Solution:**

1. Sync system clock:

```bash
# Linux
sudo ntpdate -u pool.ntp.org
# Or with systemd
sudo timedatectl set-ntp true
```

2. For OIDC, verify the configuration:
   - `issuer` must match the IdP's discovery document.
   - `audience` (client ID) must match the registered application.
   - The signing key (JWKS) must be reachable from the HADES server.

3. For SAML, verify:
   - The IdP certificate is current and correctly configured.
   - The assertion consumer service URL matches the HADES callback URL.
   - Response and assertion are both signed (or match your IdP's policy).

4. Enable debug logging to see the full token/assertion:

```bash
HADES_LOG_LEVEL=debug python core/hades_enhanced_cli.py --serve
```

---

### 5.6 License Key Rejected

**Symptom:** `Invalid license key` or `License expired` when using `--license-key`.

**Cause:** The key is malformed, expired, or does not match the expected HMAC-SHA256 signature.

**Solution:**

1. Verify the key format. HADES license keys follow the pattern: `HADES-{TIER}-XXXX-XXXX`

```bash
python core/hades_enhanced_cli.py --license-key HADES-PRO-XXXX-XXXX
```

2. Check expiration:

```bash
python core/hades_enhanced_cli.py --license-info
```

3. License tiers gate specific features:
   - **Free**: Core scanning, basic detection.
   - **Professional**: ML ensemble, behavioral analysis, vulnerability management.
   - **Enterprise**: All features including log analysis, multi-tenant, SSO.

4. Contact DarkHorse Information Security LLC for key reissuance if the key is valid but rejected.

---

### 5.7 Multi-Tenant Isolation Leak

**Symptom:** Scan results or cases from one tenant appear in another tenant's queries.

**Cause:** The `tenant_id` filter was not applied, or API requests are missing the tenant context header.

**Solution:**

1. Verify tenant isolation is enabled:

```bash
python core/hades_enhanced_cli.py --list-tenants
```

2. All API requests in a multi-tenant deployment must include tenant identification. The tenant is resolved from the API key.

3. Check that the API key is mapped to the correct tenant:

```bash
# Admin endpoint to verify tenant-key mapping
curl -H "X-API-Key: admin-key" http://localhost:8666/api/v1/admin/tenants
```

4. If data has leaked, audit the access log:

```bash
python core/hades_enhanced_cli.py --audit-verify
```

---

## 6. ML Detection Issues

### 6.1 Not Enough Training Samples

**Symptom:** `InsufficientDataError` or `Need at least N samples to train` when running `--ml-baseline` or `--ml-train`.

**Cause:** Isolation Forest and Random Forest require a minimum number of samples to build meaningful models. The single-model detector (`ml_detection.py`) needs at least 10 samples. The ensemble (`ml_ensemble.py`) may require more for the supervised Random Forest component.

**Solution:**

1. Collect a representative corpus of known-good files:

```bash
# Scan a directory of clean files to build the baseline
python core/hades_enhanced_cli.py --ml-train /path/to/known-good-files/
```

2. The training directory should contain at least 50 files across multiple formats for robust models. Include JPEG, PNG, PDF, DOCX, and other formats you expect to encounter.

3. For the ensemble detector with labeled data, import a CSV of labeled samples:

```bash
# CSV format: file_path,label (0=benign, 1=malicious)
python core/hades_enhanced_cli.py --ml-ensemble --import-labels labels.csv
```

4. Use the demo sample generator to create initial training data:

```bash
python core/hades_enhanced_cli.py --generate-demo-samples
```

---

### 6.2 Model File Not Found

**Symptom:** `FileNotFoundError` referencing a `.joblib` file in the `models/` directory, or `Model not loaded` warnings during ML scans.

**Cause:** The ML model has not been trained yet, or the model file was deleted/moved.

**Solution:**

1. Check if a model exists:

```bash
ls -la models/
```

2. Train a new baseline model:

```bash
python core/hades_enhanced_cli.py --ml-baseline
```

3. For the ensemble detector:

```bash
python core/hades_enhanced_cli.py --ml-ensemble --ml-train /path/to/training-data/
```

4. Verify model status:

```bash
python core/hades_enhanced_cli.py --ml-status
```

5. Models are not included in the git repository (the `models/` directory contains only `.gitkeep`). Each deployment must train its own models. In Docker, persist the `models/` directory via a volume:

```yaml
volumes:
  - model-data:/app/models
```

---

### 6.3 XGBoost Unavailable — Ensemble Fallback

**Symptom:** Log message: `XGBoost not available, using 2-model ensemble (Isolation Forest + Random Forest)`.

**Cause:** The `xgboost` Python package is not installed. The ML ensemble detector uses three models when XGBoost is available, and falls back to two models without it.

**Solution:**

1. Install XGBoost:

```bash
pip install xgboost
```

2. This is informational, not an error. The 2-model ensemble (Isolation Forest + Random Forest) is fully functional and provides strong detection. XGBoost adds incremental accuracy for edge cases.

3. Verify availability:

```bash
python -c "import xgboost; print(xgboost.__version__)"
```

---

### 6.4 Feature Extraction Returns NaN

**Symptom:** ML anomaly scores are `NaN` or `None`. Log shows `Feature extraction produced NaN values`.

**Cause:** The metadata feature extractor computes 15-25 numeric features from file metadata. Files with no extractable metadata produce all-zero or NaN feature vectors.

**Solution:**

1. Verify ExifTool is extracting metadata:

```bash
exiftool -j /path/to/file.jpg
```

2. If the file genuinely has no metadata, ML detection is not applicable. The engine should return a neutral score rather than NaN. If NaN is returned, this is a bug in feature normalization.

3. Retrain with a diverse corpus that includes metadata-sparse files:

```bash
python core/hades_enhanced_cli.py --ml-train /path/to/diverse-corpus/
```

4. As a workaround, disable ML for the scan:

```bash
python core/hades_enhanced_cli.py -r /evidence/
# Without --ml-detect or --ml-ensemble, ML is not invoked
```

---

### 6.5 High False Positive Rate After Training

**Symptom:** Many benign files score above the anomaly threshold. The corpus validation reports FPR > 10%.

**Cause:** The training data is not representative of the files being scanned. The Isolation Forest learns "normal" from the training set; if the training set is narrow (e.g., only JPEG files), other formats appear anomalous.

**Solution:**

1. Retrain with a broader corpus:

```bash
# Include all file types you expect to scan
python core/hades_enhanced_cli.py --ml-train /path/to/representative-files/
```

2. Adjust the anomaly threshold. The default contamination parameter is set for ~5% anomaly rate. Increase it if your environment has a higher benign diversity.

3. Use the ensemble detector with labeled data for supervised learning:

```bash
python core/hades_enhanced_cli.py --ml-ensemble --import-labels /path/to/labels.csv
```

4. Run the corpus validation to measure FPR:

```bash
python -m pytest tests/test_corpus_full_validation.py -v -s
```

The target is TPR >= 90% and FPR <= 10%.

---

### 6.6 Model Incompatible After scikit-learn Upgrade

**Symptom:** `ModuleNotFoundError` or `AttributeError` when loading a previously trained model after upgrading scikit-learn.

**Cause:** scikit-learn serializes models with `joblib`. Models trained on one version may not deserialize on a different version due to internal class changes.

**Solution:**

1. Retrain the model on the new scikit-learn version:

```bash
# Delete the old model
rm models/*.joblib

# Retrain
python core/hades_enhanced_cli.py --ml-baseline
```

2. Pin scikit-learn in your deployment to avoid unintended upgrades:

```bash
pip install scikit-learn==1.4.0
```

3. In Docker, the model is rebuilt on each fresh deployment. Persist trained models via a volume if you want to avoid retraining:

```yaml
volumes:
  - ml-models:/app/models
```

When upgrading scikit-learn, rebuild the volume contents.

---

## Additional Resources

- [Quick Start Guide](quick_start.md)
- [API Reference](api_reference.md)
- [Deployment Guide](deployment_guide.md)
- [Docker Guide](docker_guide.md)
- [Enterprise Deployment Guide](enterprise_deployment_guide.md)
- [ML Detection Guide](ml_detection_guide.md)
- [SIEM Integration Guide](siem_integration_guide.md)
- [Performance Guide](performance_guide.md)
- [Observability Guide](observability_guide.md)
- [Plugin Development Guide](plugin_development_guide.md)

---

Copyright (c) DarkHorse Information Security LLC. All rights reserved.
