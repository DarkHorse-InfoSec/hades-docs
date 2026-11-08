# HADES Troubleshooting Guide
Version: 1.0.0

Author: DarkHorse Information Security LLC

> **Updated 2026-11-08:** HADES ships as two self-contained binaries, `hades` and `hades-server` (see [Installation](installation.md)); there is no pip package and no container image, and installing Python packages on the host has no effect on the binaries. Sections that described pip installs and Docker builds have been rewritten or removed. When a component is reported missing, see [1.2](#12-an-optional-component-is-reported-missing).

This guide covers common issues encountered when installing, configuring, and operating HADES. Each entry provides the symptom, likely root cause, and resolution steps.

---

## Table of Contents

- [1. Installation Issues](#1-installation-issues)
  - [1.1 ExifTool Not Found](#11-exiftool-not-found)
  - [1.2 An Optional Component Is Reported Missing](#12-an-optional-component-is-reported-missing)
  - [1.3 YARA Rule Load Errors](#13-yara-rule-load-errors)
  - [1.4 uvloop Not Available on Windows](#14-uvloop-not-available-on-windows)
- [2. API Server Issues](#2-api-server-issues)
  - [2.1 Port Already in Use](#21-port-already-in-use)
  - [2.2 API Key Authentication Failures](#22-api-key-authentication-failures)
  - [2.3 CORS Errors in Browser](#23-cors-errors-in-browser)
  - [2.4 WebSocket Connection Drops](#24-websocket-connection-drops)
  - [2.5 422 Validation Error on Upload](#25-422-validation-error-on-upload)
  - [2.6 Server Fails to Start: Module Import Error](#26-server-fails-to-start-module-import-error)
- [3. Scanning Issues](#3-scanning-issues)
  - [3.1 File Too Large: Memory Exhaustion](#31-file-too-large-memory-exhaustion)
  - [3.2 Unsupported File Type Returns Empty Results](#32-unsupported-file-type-returns-empty-results)
  - [3.3 ExifTool Process Hangs](#33-exiftool-process-hangs)
  - [3.4 ExifTool Process Crashes Mid-Scan](#34-exiftool-process-crashes-mid-scan)
  - [3.5 Scan Cache Returns Stale Results](#35-scan-cache-returns-stale-results)
  - [3.6 Async Pipeline Deadlock](#36-async-pipeline-deadlock)
- [4. Docker Issues](#4-docker-issues)
- [5. Enterprise Feature Issues](#5-enterprise-feature-issues)
  - [5.1 Redis Fallback to In-Memory](#51-redis-fallback-to-in-memory)
  - [5.2 Encryption Key Not Found](#52-encryption-key-not-found)
  - [5.3 RBAC: Permission Denied After User Creation](#53-rbac-permission-denied-after-user-creation)
  - [5.4 License Key Rejected](#54-license-key-rejected)
  - [5.5 Multi-Tenant Isolation Leak](#55-multi-tenant-isolation-leak)
- [6. ML Detection Issues](#6-ml-detection-issues)
  - [6.1 ML Stage Not Running](#61-ml-stage-not-running)
  - [6.2 "ML training on this host is not supported"](#62-ml-training-on-this-host-is-not-supported)
  - [6.3 A Clean File Is Flagged by ML](#63-a-clean-file-is-flagged-by-ml)

---

## 1. Installation Issues

### 1.1 ExifTool Not Found

**Symptom:** `ExifTool not found` error at startup, or scan results return no metadata fields.

**Cause:** ExifTool is an external program. It is optional for scanning (a native fallback ships in the binary) and recommended for the metadata sanitizer; HADES looks for the `exiftool` binary on `PATH`.

**Solution:**

```bash
# Linux (Debian/Ubuntu)
sudo apt-get install libimage-exiftool-perl

# macOS
brew install exiftool

# Windows - download from https://exiftool.org
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

### 1.2 An Optional Component Is Reported Missing

**Symptom:** A feature is unavailable, or the log says an optional module could not be imported.

**Cause:** The `hades` and `hades-server` binaries carry their Python components inside them. Installing a Python package on the host (for example with pip) does not change what the binaries contain.

**Solution:** Run `hades doctor`. It probes each component and says what is disabled when one is missing. The optional ones and their effect, as `hades doctor` reports them:

| Component | Without it |
|---|---|
| `pikepdf` | Deep PDF analysis unavailable, regex-based fallback |
| `olefile` | OLE/Office deep analysis unavailable |
| `py7zr` | 7-Zip archive extraction unavailable, ZIP/tar only |
| `bcrypt` | RBAC password hashing unavailable, auth module limited |
| `cryptography` | AES-256-GCM encryption at rest unavailable |
| `redis` | Redis state/cache unavailable, in-memory fallback used |
| `prometheus_client` | Prometheus metrics collection disabled |
| `watchdog` | File monitoring disabled, cannot watch directories for changes |
| `xgboost` | Signed ML classifier cannot load (ML stage off) |

If a component you need is reported missing, send the `hades doctor` output to support@darkhorseinfosec.com.

---

### 1.3 YARA Rule Load Errors

**Symptom:** `Error compiling YARA rules` or `syntax error` referencing a `.yar` file.

**Cause:** A YARA rule file contains syntax errors, duplicate rule names, or references undefined modules.

**Solution:**

Validate all rules before deploying:

```bash
hades rules validate
```

> **Known issue in v1.7.4:** `hades rules validate` exits with "CI rule validator not found" in the compiled binary; the validator was not included in the build. The fix is tracked for the next release.

`hades rules validate` is the strict, CI-level validator: it exits non-zero when any rule fails.

Common issues:
- Duplicate rule identifiers across files: each rule name must be globally unique.
- Missing `meta:` section: the HADES convention requires `author`, `description`, and `severity` metadata fields.
- Unsupported YARA module references: ensure the YARA installation includes the modules your rules reference (e.g., `pe`, `math`, `hash`).

---

### 1.4 uvloop Not Available on Windows

**Symptom:** Warning about uvloop at startup on Windows.

**Cause:** uvloop is a Unix-only (Linux/macOS) asyncio event loop replacement, so it is not part of the Windows build.

**Solution:** No action required. HADES uses the default asyncio event loop on Windows. This is not a bug. Performance on Windows is adequate for most workloads; for high-throughput deployments, use Linux.

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
hades serve --port 9000
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
hades admin create-admin admin2
# Prompts for a password, then prints the new admin user's API key
```

> **Known issue in v1.7.4:** in the compiled `hades` binary, `hades admin create-admin`, `create-user`, `list-users`, `create-tenant` and `list-tenants` exit with an internal error (`NameError`) instead of running. The fix is tracked for the next release. Until then these commands cannot be used from the shipped binary.

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

- Reverse proxy not configured for WebSocket upgrade.
- API key not provided in WebSocket query parameters.

**Solution:**

1. If behind nginx, add WebSocket upgrade support:

```nginx
location /ws {
    proxy_pass http://hades-api:8666;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;
}
```

2. Pass the API key as a query parameter:

```
ws://localhost:8666/ws?api_key=your-key
```

3. Check that no firewall or load balancer is terminating long-lived connections.

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

### 2.6 Server Fails to Start: Module Import Error

**Symptom:** `ModuleNotFoundError` or `ImportError` when running `--serve`.

**Cause:** A component the server needs is missing from the build, or the binary was started from a broken or partial install.

**Solution:**

1. Check the version and which components are present:

```bash
hades --version
hades doctor
```

2. If `hades doctor` reports a required component missing, reinstall the binaries (see [Installation](installation.md)); if it is still missing, send the `hades doctor` output to support@darkhorseinfosec.com.

Import failures for optional modules (integrations, auth, storage) are logged as warnings but do not prevent startup. Only core modules (`enhanced_detection_engine`, `metadata_parser_engine`) are fatal.

---

## 3. Scanning Issues

### 3.1 File Too Large: Memory Exhaustion

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
hades scan --max-file-size 100000000 -r /evidence/
```

3. For batch processing of large directories, enable the async pipeline with bounded concurrency:

```bash
hades scan --async --concurrency 2 -r /large-evidence/
```

4. Enable the scan cache to avoid re-processing files:

```bash
hades scan --cache -r /evidence/
```

---

### 3.2 Unsupported File Type Returns Empty Results

**Symptom:** Scan completes but reports zero findings for a file type you expect to be analyzed.

**Cause:** HADES analyzes metadata, not file content execution. Some file types have limited metadata (e.g., plain text files, raw binaries). The detection engine prioritizes images, documents, archives, and multimedia formats.

**Solution:**

1. Check supported types:

```bash
hades --help
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
hades scan --async -r /evidence/
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
hades scan -r /evidence/ --exclude-types .problematic
```

4. Report the file to the ExifTool maintainer if it triggers a reproducible hang.

---

### 3.4 ExifTool Process Crashes Mid-Scan

**Symptom:** `BrokenPipeError` or `ExifTool process terminated unexpectedly` during batch scanning.

**Cause:** The persistent ExifTool process pool (`PersistentExifTool` in `async_scanner.py`) reuses long-running ExifTool subprocesses. A segfault or Perl error in ExifTool terminates the subprocess.

**Solution:**

1. The async pipeline automatically restarts crashed ExifTool processes. Verify it is enabled:

```bash
hades scan --async -r /evidence/
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
hades scan --no-cache /path/to/file.jpg
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
hades scan --async --concurrency 1 -r /evidence/
```

2. Enable profiling to identify the bottleneck:

```bash
hades scan --async --profile -r /evidence/
```

3. Kill and restart. The pipeline supports graceful shutdown:

```bash
# Send SIGINT (Ctrl+C) - the pipeline drains active tasks and exits
```

---

## 4. Docker Issues

There is no HADES container image: HADES ships as the `hades` and `hades-server` binaries (see [Installation](installation.md)). Containerized and distributed deployments (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap; see the [Kubernetes Guide](kubernetes_guide.md). The Docker troubleshooting entries from earlier versions of this guide have been removed.

---

## 5. Enterprise Feature Issues

### 5.1 Redis Fallback to In-Memory

**Symptom:** Logs show `Redis unavailable, using in-memory fallback`. Rate limiting, sessions, and distributed locks operate in-process only.

**Cause:** The `redis` component is missing from your build, or the Redis server is unreachable.

**Solution:**

1. Run `hades doctor` and check that `redis` is present in your build.

2. Verify Redis connectivity:

```bash
redis-cli -h localhost -p 6379 ping
# Should return: PONG
```

3. Pass the URL to HADES:

```bash
hades serve --redis-url redis://localhost:6379/0
```

4. In-memory fallback is functional for single-instance deployments. Multi-instance deployments (Docker scaling, Kubernetes) require Redis for shared state. Without it, each instance maintains independent rate limits and sessions.

---

### 5.2 Encryption Key Not Found

**Symptom:** `FileNotFoundError: encryption.key not found` or `Encryption key not configured` when using `--encrypt`.

**Cause:** AES-256-GCM encryption requires a key file. The key must be generated before enabling encryption.

**Solution:**

1. Generate a new encryption key:

```bash
hades serve --generate-encryption-key
```

> **Known issue in v1.7.4:** in the compiled binary, `hades serve --generate-encryption-key` exits with "Enterprise storage module not available": the storage module is not included in the v1.7.4 build. The fix is tracked for the next release.

This creates `config/encryption.key`. The file contains a 256-bit random key, base64-encoded.

2. Critical: Never commit `encryption.key` to version control. Verify `.gitignore` includes:

```
*.key
config/encryption.key
```

3. Back up the key securely. Encrypted data is unrecoverable without it.

---

### 5.3 RBAC: Permission Denied After User Creation

**Symptom:** A newly created user receives HTTP 403 on endpoints they should have access to.

**Cause:** The user was created without the correct role, or the role's permission matrix does not include the requested endpoint.

**Solution:**

1. Verify the user's role:

```bash
hades admin list-users
```

> **Known issue in v1.7.4:** in the compiled `hades` binary, `hades admin create-admin`, `create-user`, `list-users`, `create-tenant` and `list-tenants` exit with an internal error (`NameError`) instead of running. The fix is tracked for the next release. Until then these commands cannot be used from the shipped binary.

2. The default roles and permissions are:
   - **admin**: Full access to all endpoints.
   - **analyst**: Scan, monitor, evidence, and reporting endpoints. No user management.
   - **viewer**: Read-only access to scan results and reports.

3. Reassign a role:

```bash
hades admin create-user analyst1 --role analyst
```

4. If using API key authentication, ensure the API key belongs to the correct user. Rotate if compromised:

```bash
# Via API endpoint
curl -X POST http://localhost:8666/api/v1/auth/users/<user_id>/rotate-key \
  -H "X-API-Key: admin-key"
```

---

### 5.4 License Key Rejected

**Symptom:** `Invalid license key` or `License expired` when using `--license-key`.

**Cause:** The key is malformed, expired, or does not match the expected HMAC-SHA256 signature.

**Solution:**

1. Verify the key format. HADES license keys follow the pattern: `HADES-{TIER}-XXXX-XXXX`

```bash
hades admin license --key HADES-PRO-XXXX-XXXX
```

2. Check expiration:

```bash
hades admin license
```

3. License tiers gate specific features:
   - **Free**: Core scanning, basic detection.
   - **Professional**: ML classifier, behavioral analysis, vulnerability management.
   - **Enterprise**: All features including log analysis and multi-tenant.

4. Contact DarkHorse Information Security LLC for key reissuance if the key is valid but rejected.

---

### 5.5 Multi-Tenant Isolation Leak

**Symptom:** Scan results or cases from one tenant appear in another tenant's queries.

**Cause:** The `tenant_id` filter was not applied, or API requests are missing the tenant context header.

**Solution:**

1. Verify tenant isolation is enabled:

```bash
hades admin list-tenants
```

2. All API requests in a multi-tenant deployment must include tenant identification. The tenant is resolved from the API key.

3. Check that the API key is mapped to the correct tenant:

```bash
# Admin endpoint to verify tenant-key mapping
curl -H "X-API-Key: admin-key" http://localhost:8666/api/v1/admin/tenants
```

4. If data has leaked, audit the access log:

```bash
hades case verify
```

---

## 6. ML Detection Issues

### 6.1 ML Stage Not Running

**Symptom:** No `ml_anomaly` findings on any file.

**Cause:** The ML stage runs only on Professional, Team and Enterprise licences. On a licensed install, the signed model bundle (`models/baseline_classifier_v1.*`) is missing, failed signature verification, or could not be loaded. HADES refuses an unverified model and scans without the ML stage; the other stages are unaffected.

**Solution:**

1. Check the model state:

```bash
hades intel ml-status
```

2. `verification_status` names the cause: `model file missing`, `signature verification failed`, `metadata load failed`, `model load failed` or `xgboost not installed`.

3. For a missing or refused bundle, reinstall HADES from the release so the bundle files match their signature. Do not edit or replace them: any change fails verification by design.

4. If `plugins.plugin_config.ml_anomaly.model_path` is set in your config, it must point at a signed bundle. A path to a missing bundle disables ML; it is never replaced by the bundled model.

---

### 6.2 "ML training on this host is not supported"

**Symptom:** `hades intel ml-train` or `hades intel ml-baseline` exits with this message.

**Cause:** Local training was retired. The model is trained and signed by DarkHorse; a locally trained model could not be loaded, because only a bundle signed with the DarkHorse model-signing key is accepted. The commands read and write nothing.

**Solution:** Nothing to fix. Check the model in use with `hades intel ml-status`, and report files you believe are misclassified with `hades feedback false-positive` (see 6.3).

---

### 6.3 A Clean File Is Flagged by ML

**Symptom:** An ML anomaly finding on a file you know is benign.

**Solution:**

1. Look at `contributing_features` in the finding to see what drove the score, and at the file's other findings: ML is one input to the verdict.

2. Report it, giving the verdict HADES returned and why you believe it is wrong:

```bash
hades feedback false-positive /path/to/file --verdict HIGH \
  --explanation "Signed installer from our software vendor, distributed internally"
```

The file itself is never uploaded. Only its SHA-256 (computed locally), the verdict you report and your explanation are sent. Confirmed false positives are fixed in a later release, either in the model or on the known-clean list that is checked before the model runs.

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
