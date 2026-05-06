# HADES v0.5.0 -- Enterprise Metadata Forensics Platform

**Release Date:** 2026-02-19

> **HISTORICAL NOTICE (added 2026-05-06):** This release note describes HADES under the pre-v1.4 distribution model (Cloudsmith pip registry, optional dependency groups, Docker Hub images, public source-clone). **Those install paths no longer work.** Cloudsmith Free plan stopped allowing RAW packages on 2026-04-29; HADES now ships exclusively as a Nuitka-compiled signed binary distributed via the customer portal (Path A) or the Homebrew tap (Path B). For current installation, see the [Installation Guide](docs/installation.md). The features described below were accurate as of v0.5.0; many have evolved or been replaced in subsequent releases through v1.4.2.

HADES v0.5.0 is a major release that completes the transition from a CLI scanning tool to a production-grade forensics platform. The REST API has been rebuilt on FastAPI with WebSocket support, a full evidence chain with case management enables courtroom-ready forensic workflows, and a self-contained web dashboard provides browser-based access to every feature. Security hardening, deep file format analysis, and an expanded YARA rule repository round out the release.

## What's New

### FastAPI REST API with WebSocket Support

The API server has been completely rewritten on FastAPI, replacing the previous Flask implementation. The new server delivers automatic OpenAPI documentation at `/docs`, Pydantic request/response validation, native async endpoint support, and a `WebSocketManager` for real-time scan progress and alert streaming. The migration also brings a measurable performance improvement for concurrent scan workloads.

### Evidence Chain and Case Management

New forensic workflow capabilities in `core/evidence_chain.py` provide an append-only, SHA-256 hash-chained audit log that records every scan, export, and analyst action. Investigators can create cases, link scan results, add notes, and export self-verifying evidence packages -- ZIP archives containing the evidence manifest, file hashes, HMAC-SHA256 signature, and an embedded verification script. Every operation is accessible through the REST API at `/api/v1/evidence/`.

### Web Dashboard

A fully self-contained single-page application served at `/dashboard/` provides browser-based access to scan upload (with drag-and-drop), real-time file monitor status, case management, audit trail browsing, and system health monitoring. Built with vanilla HTML, CSS, and JavaScript with no external CDN dependencies, the dashboard works in air-gapped environments.

### Deep File Format Analysis

The `DeepFormatAnalyzer` adds native parsing for PDF (JavaScript, OpenAction, embedded files, SubmitForm), Office documents (macros, OLE objects, DDE, template injection), SVG (inline scripts, XXE, event handlers, foreignObject), and polyglot detection. This analysis runs automatically as part of the `EnhancedDetectionEngine.analyze()` pipeline.

### Security Hardening

A new security middleware layer (`core/security_middleware.py`) applies sliding-window rate limiting, file upload magic byte validation, path traversal protection, HTML output sanitization, and security response headers (CSP, X-Frame-Options, HSTS) to all API endpoints.

### Expanded YARA Rule Repository

Two new rule files target metadata-specific threats:

- **`rules/polyglot_detection.yar`** -- Detects JPEG+ZIP, JPEG+RAR, PDF+JavaScript, PNG+HTML, GIF+JavaScript polyglots and generic dual-header files.
- **`rules/suspicious_metadata.yar`** -- Flags Base64 payloads over 500 characters, URLs in EXIF comments, email PII leakage, encoded executable headers, shell commands, and oversized metadata fields.

A validation script (`scripts/validate_rules.py`) compiles all rules, checks for duplicate names and missing metadata, and reports severity/category statistics.

### Performance Benchmarks

The `core/benchmarks.py` suite measures single-file scan latency, batch throughput, memory consumption, ML feature extraction speed, and API response times. Run benchmarks from the CLI with `--benchmark` or `--benchmark-quick`.

## Installation

### pip (Cloudsmith)

```bash
pip install hades-scanner --extra-index-url https://dl.cloudsmith.io/basic/darkhorse/hades/python/simple/
hades --help
hades-server --port 8666
```

### From Source

```bash
git clone https://github.com/DarkHorse-InfoSec/hades-docs.git
cd hades && pip install -r requirements.txt
python cli/hades_cli.py --help
```

### Docker

```bash
docker compose -f docker/docker-compose.yml up -d
curl http://localhost:8666/api/v1/health
```

Open `http://localhost:8666/dashboard/` for the web interface.

## Migration from v0.4.0

### Breaking Changes

- **Flask to FastAPI**: `create_app()` now returns a FastAPI instance instead of a Flask app. If you have custom middleware or route registrations that depend on Flask APIs, they will need to be updated.
- **`enterprise_server.py` removed**: `WebSocketManager` and Pydantic models have been consolidated into `hades_api.py`. Remove any direct imports from `enterprise_server`.
- **Test client changes**: Tests must use `starlette.testclient.TestClient` instead of Flask's `test_client()`. Response JSON is accessed via `resp.json()` instead of `resp.get_json()`. Validation errors now return HTTP 422 instead of 400.
- **New dependencies**: `fastapi`, `uvicorn`, `python-multipart`, `websockets`, and `httpx` are now required. `pikepdf` and `olefile` are optional but recommended for deep format analysis.

### Migration Steps

1. Update dependencies: `pip install -r requirements.txt`
2. Replace any `from enterprise_server import ...` with imports from `hades_api`
3. Update test code to use `TestClient` from `starlette.testclient`
4. If using custom YARA rules, verify they compile with `python scripts/validate_rules.py`

## Known Issues

- **YARA modules**: `enterprise_threats.yar` imports the `pe`, `elf`, and `math` YARA modules. These require a YARA build compiled with module support. Rules in other files work with any YARA 4.0+ installation.
- **pikepdf/olefile optional**: Deep format analysis for PDF and Office files requires pikepdf and olefile respectively. If not installed, the engine falls back to regex-based parsing with reduced detection coverage.
- **WebSocket requires websockets package**: The `WebSocketManager` in `hades_api.py` requires the `websockets` package. Install via `pip install websockets`.
- **Large JSX files**: `EnhancedGUI.jsx` (133KB) remains monolithic. Modifications to the GUI should target specific sections rather than full rewrites.

## What's Next (v0.6.0 Preview)

- **Scheduled scanning**: Cron-style scan scheduling for automated periodic analysis
- **Multi-tenant support**: Organization-level isolation for managed service deployments
- **Report templates**: Customizable HTML/PDF report templates with branding support
- **Rule auto-update**: Automatic YARA rule updates from a curated threat feed
- **Enhanced ML models**: Additional model architectures beyond Isolation Forest for improved anomaly detection accuracy

## Full Changelog

See [CHANGELOG.md](CHANGELOG.md) for the complete list of changes.

## Contributors

- DarkHorse Information Security LLC
