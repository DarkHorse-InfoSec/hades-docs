# Changelog

> **NOTICE (2026-05-06):** Entries below v1.4 reference the pre-v1.4 distribution model (Cloudsmith pip registry, Docker Hub, optional dependency groups). Those install paths no longer work; HADES has shipped exclusively as a Nuitka-compiled signed binary distributed via the customer portal (Path A) or Homebrew tap (Path B) since v1.4.1. Pre-v1.4 entries remain for historical reference. See [installation.md](installation.md) for current install paths.

All notable changes to HADES are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.7.5] - 2026-11-11

### Security
- Licence validation hardened.

### Fixed
- `hades admin create-admin`, `create-user`, `list-users`, `create-tenant` and `list-tenants` run in the binary.
- `hades rules validate` runs in the binary; it now also validates your own rule files (`PATH`).
- `hades serve --generate-encryption-key`, tenant management and the `hades monitor` Zeek / Suricata
  connectors load in the `hades` binary, and `hades serve` serves the same storage, evidence and
  integration routes as `hades-server`.
- One data directory: databases no longer live in a per-version folder or the working directory, so an
  upgrade keeps your data. See [Where HADES keeps its data](installation.md#where-hades-keeps-its-data).

### Not included
- Email gateway, GCS / Azure scanning, Slack / Teams bots and Prometheus metrics need components the 1.7.5
  binaries do not include; see [Features not included](installation.md#features-not-included-in-the-175-binaries).

## Known issues in v1.7.4 (fixed in 1.7.5)

Found while checking these docs against the shipped binaries on 2026-11-06. Each was reproduced on the signed v1.7.4 Windows binary from an empty working directory:

- `hades admin create-admin`, `create-user`, `list-users`, `create-tenant` and `list-tenants` exit with an internal error (`NameError`). Fixed in source; the fix ships with the next release.
- `hades rules validate` exits with "CI rule validator not found": the validator was not included in the build.
- `hades serve --generate-encryption-key` and `hades admin list-tenants` report "Enterprise storage module not available": the storage module is not included in the v1.7.4 build.

## Documentation now matches the shipped commands

The guides previously used `hades-enhanced` and `python core/hades_enhanced_cli.py`, which are not part of the product: HADES ships `hades` and `hades-server`. Every command is now written in its `hades <command>` form, and the [CLI Reference](cli_reference.md) is generated from the shipped command-line parser.

## Removed in v1.7.3 and v1.7.4

The full release notes for these versions are not in this file yet. These removals affect documented features, so they are recorded here:

- **v1.7.4 (2026-11-06): SSO removed.** The OIDC and SAML endpoints (`/api/v1/auth/sso/...`) and the `auth.sso` configuration section are gone. Use RBAC users with per-user API keys.
- **v1.7.3 (2026-10-01): PostgreSQL removed.** SQLite is the only database backend. `--db-backend postgres` and `--db-dsn` are command-line errors; a leftover `HADES_DB_BACKEND=postgres` or `HADES_DB_DSN` makes `hades serve --health-check` report unhealthy.
- **v1.7.3 (2026-10-01): ML ensemble removed.** `core/ml_ensemble.py`, the `--ml-ensemble` flag and `GET /api/v1/ml/ensemble/status` are gone; the scan path never used them. The ML stage is the signed XGBoost classifier (production model v1.18.0, 33 features); see the [ML Detection Guide](ml_detection_guide.md).
- **Not served by v1.7.4:** the API reference previously listed `POST /api/v1/ml/retrain`, `GET /api/v1/ml/features`, `GET /api/v1/threat-intel/lookup`, `GET /api/v1/threat-intel/cache/stats` and `DELETE /api/v1/threat-intel/cache`. The v1.7.4 API server does not serve them (they were removed from the docs to match). Use `hades intel lookup` and `hades intel cache-stats` on the command line.

---

## [0.6.0] - 2026-02-19

### Added

- **Role-Based Access Control** (`core/auth/rbac.py`): User management with admin, analyst, and viewer roles. Permission matrix for fine-grained API access control. bcrypt password hashing with SHA-256 fallback. API key authentication and rotation. FastAPI dependencies for endpoint protection.
- **Single Sign-On** (`core/auth/sso.py`): OIDC provider with JWT validation. SAML 2.0 provider with AuthnRequest generation and response validation. JIT (Just-In-Time) user provisioning for SSO identities.
- **License Key System** (`core/auth/license.py`): HMAC-SHA256 signed license keys with three tiers (free, professional, enterprise). Feature gating via FastAPI dependency.
- **Auth API Endpoints** (`core/auth/auth_routes.py`): Login, user CRUD, API key rotation, license info, SSO endpoints at `/api/v1/auth/`.
- **Database Abstraction Layer** (`core/storage/database.py`): Pluggable backend interface with SQLite (WAL mode) and PostgreSQL (connection pooling) implementations.
- **Enterprise Scan Store** (`core/storage/scan_store.py`): Multi-tenant scan result storage.
- **Audit Store** and **Case Store**: Persistence backed by pluggable DatabaseBackend.
- **Redis State Manager** (`core/storage/redis_state.py`): Sliding-window rate limiting, session management, pub/sub, distributed locks, and caching. In-memory fallback.
- **Encryption at Rest** (`core/storage/encryption.py`): AES-256-GCM field-level encryption.
- **Multi-Tenant Isolation** (`core/storage/tenant.py`): Tenant CRUD, TenantContext for request-scoped isolation.
- **ML Ensemble Detector** (`core/ml_ensemble.py`): Multi-model anomaly detection (Isolation Forest + Random Forest + optional XGBoost) with weighted voting and 25 features.
- **Behavioral Analysis** (`core/behavioral_analysis.py`): Campaign detection via shared IOC correlation, metadata pattern similarity, and temporal clustering. File relationship graph.
- **MITRE ATT&CK Mapping** (`core/mitre_mapping.py`): Maps detection findings to MITRE ATT&CK techniques and tactics.
- **Threat Intelligence Scheduler** (`core/threat_intel_scheduler.py`): Automated periodic ingestion of threat intelligence feeds.
- **YARA Rule Builder** (`core/yara_rule_builder.py`): Programmatic YARA rule generation from templates.
- **Enterprise CLI Flags**: `--create-admin`, `--create-user`, `--user-role`, `--list-users`, `--license-info`, `--db-backend`, `--redis-url`, `--encrypt`, `--create-tenant`, `--list-tenants`, `--migrate-db`.
- **Detection Depth CLI Flags**: `--ml-ensemble`, `--behavioral`, `--mitre-map`, `--schedule-feeds`, `--build-rule`, `--list-templates`, `--malware-validate`.
- **Phase 2 API Endpoints**: MITRE matrix/lookup, threat intel scheduler, rule templates/build, ML ensemble status, behavioral campaigns.
- **Enterprise Docker Support**: PostgreSQL 16 and Redis 7 in Docker Compose enterprise profile.
- **Test Suites**: 183 new enterprise tests, 88 detection depth tests. 1,231+ total passing.

### Changed

- Version updated to 0.6.0 across all components.
- `hades_api.py`: Enterprise auth and storage modules optionally imported. Phase 2 modules optionally imported with route registration.
- `hades_enhanced_cli.py`: Enterprise and Detection Depth argument groups added.
- Docker images updated with enterprise dependencies and environment variables.
- `pyproject.toml`: New `enterprise` dependency group. `full` group updated.

---

## [0.5.0] - 2026-02-19

### Added

- **FastAPI Migration**: REST API migrated from Flask to FastAPI with async endpoints, Pydantic models, OpenAPI docs, and WebSocket support.
- **Evidence Chain & Case Management**: SHA-256 hash-chained audit log, investigation case CRUD, self-verifying evidence export packages.
- **Web Dashboard SPA**: Vanilla HTML+CSS+JS single-page application with dark theme, drag-and-drop scanning, monitor dashboard, case management, audit trail.
- **Security Middleware**: Rate limiting, magic byte validation, path traversal protection, XSS sanitization, security headers.
- **Deep Format Analyzer**: Native PDF, Office, SVG, and polyglot file analysis.
- **Performance Benchmarks**: Cross-platform benchmark suite.
- **YARA Rule Repository**: `polyglot_detection.yar` (6 rules), `suspicious_metadata.yar` (6 rules), validation script.
- **Network Integrations**: Zeek and Suricata file extraction connectors.
- **Email Gateway**: SMTP gateway for email attachment scanning.
- **Cloud Storage Scanning**: AWS S3, GCS, Azure Blob connectors.
- **CI/CD Integration**: Pipeline scanner with SARIF 2.1.0 output.
- **Chat Bots**: Slack and Teams bot integrations.
- **Package Distribution**: `pip install hades-scanner` with optional dependency groups.
- **Docker Production Image**: Multi-stage build, non-root user, health checks.
- **Homebrew Formula**: macOS installation via Homebrew.
- 760+ passing tests.

### Changed

- Flask to FastAPI migration.
- Consolidated `enterprise_server.py` into `hades_api.py`.
- YARA rules reorganized by category.

### Fixed

- YARA compilation error in `advanced_threats.yar` (non-capturing group).
- Security audit: rate limiting and security headers applied to all API endpoints.

---

## [0.4.0] - 2026-02-17

### Added

- REST API Server with 11 endpoints.
- Real-time file monitoring with watchdog.
- Plugin system with `HADESPlugin` base class.
- Cloud threat intelligence (VirusTotal, AbuseIPDB, OTX, MalwareBazaar).
- ML anomaly detection (Isolation Forest).
- SIEM integration (Syslog, CEF, STIX, LEEF, ECS).
- GUI dashboard components.
- 238+ passing tests.

---

## [0.3.0] - 2025-02-06

### Added

- Enhanced GUI with Electron + React + Vite + Tailwind CSS.
- Sandboxed file viewer using Docker + Express.js.
- Auto-reporting system (HTML, JSON, CSV).
- Configuration system via `hades_config.json`.
- Advanced detection and metadata sanitizer modules.
- Enterprise core engine modules.

---

## [0.2.0] - 2024-12-15

### Added

- Basic CLI scanner with ExifScanner class.
- Report generation (JSON, HTML, CSV).
- Docker sandbox integration.
- Threat intelligence configuration.

---

## [0.1.0] - 2024-10-01

### Added

- Initial release with basic EXIF metadata extraction.
- ExifTool wrapper.
- Core scanner module.
- Basic suspicious pattern detection.
