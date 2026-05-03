# HADES v0.6.0 -- Enterprise Detection & Community Platform

**Release Date:** 2026-02-19

HADES v0.6.0 is the largest release to date, spanning three development phases that add enterprise-grade security infrastructure, dramatically deeper detection capabilities, and a community ecosystem for plugins and documentation. Enterprise deployments gain RBAC, SSO, PostgreSQL, Redis, field-level encryption, and multi-tenant isolation. Detection depth improves with a three-model ML ensemble, behavioral campaign analysis, MITRE ATT&CK mapping, and automated threat feed ingestion. A new documentation site, plugin marketplace, YARA contribution workflow, and GitHub Actions CI/CD round out the release.

All new features are opt-in. Existing v0.5.0 configurations and workflows continue to work without modification.

## What's New

### Phase 1: Enterprise Security & Storage

#### Role-Based Access Control

A new RBAC system (`core/auth/rbac.py`) introduces admin, analyst, and viewer roles with a fine-grained permission matrix governing every API endpoint. Passwords are hashed with bcrypt (SHA-256 fallback when bcrypt is unavailable). API key authentication supports key rotation, and FastAPI dependencies (`get_current_user`, `require_role`) enforce access control at the endpoint level. When RBAC is not configured, the existing flat API key authentication is preserved as a fallback.

#### Single Sign-On (OIDC & SAML 2.0)

SSO support (`core/auth/sso.py`) enables integration with enterprise identity providers. The OIDC provider handles JWT validation, authorization URL generation, and configurable role mapping. The SAML 2.0 provider generates AuthnRequests, validates responses, and extracts user attributes. Both providers support Just-In-Time (JIT) user provisioning -- authenticated SSO users are automatically created with a default role on first login. Auth endpoints are served at `/api/v1/auth/`.

#### License Key System

A three-tier license system (`core/auth/license.py`) supports free, professional, and enterprise feature sets. License keys are signed with HMAC-SHA256 and validated at startup. A `require_license` FastAPI dependency gates premium features based on the active license tier. Administrators can generate, validate, and inspect license keys through both the CLI and API.

#### PostgreSQL Backend

The new database abstraction layer (`core/storage/database.py`) provides a pluggable backend interface with implementations for both SQLite (WAL mode, thread-safe) and PostgreSQL (connection pooling via psycopg2). A factory function selects the backend at startup, with environment variable override support (`HADES_DB_BACKEND`, `HADES_DB_DSN`). Scan results, audit logs, and case data are all backed by the pluggable storage layer through dedicated store modules (`scan_store.py`, `audit_store.py`, `case_store.py`).

#### Redis State Manager

The Redis state manager (`core/storage/redis_state.py`) provides sliding-window rate limiting, session management, pub/sub messaging, distributed locks, and caching for multi-instance deployments. When Redis is unavailable, the system falls back to an in-memory implementation with identical APIs, ensuring single-instance deployments work without Redis.

#### AES-256-GCM Encryption at Rest

Field-level encryption (`core/storage/encryption.py`) protects sensitive scan data at rest using AES-256-GCM. The `EncryptedDatabaseBackend` wrapper transparently encrypts on write and decrypts on read, so encryption is invisible to upstream code. Encryption keys are managed via the CLI (`--generate-encryption-key`) or environment variable (`HADES_ENCRYPTION_KEY`).

#### Multi-Tenant Isolation

Multi-tenant support (`core/storage/tenant.py`) enables organization-level data isolation for managed service deployments. Tenant CRUD operations, request-scoped `TenantContext`, and API key to tenant mapping ensure that scan results, cases, and audit logs are strictly partitioned. Storage admin endpoints at `/api/v1/admin/` provide tenant management, database status, migration, Redis status, and encryption status.

#### Enterprise CLI and Docker

New CLI flags expose every enterprise feature: `--create-admin`, `--create-user`, `--user-role`, `--list-users`, `--license-info`, `--license-key`, `--db-backend`, `--db-dsn`, `--redis-url`, `--encrypt`, `--generate-encryption-key`, `--create-tenant`, `--list-tenants`, and `--migrate-db`. Docker Compose gains PostgreSQL 16 and Redis 7 services under the `enterprise` profile, with new environment variables for database, Redis, encryption, and license configuration.

Install enterprise dependencies:

```bash
pip install "hades-scanner[enterprise]"
```

This pulls in bcrypt, PyJWT, cryptography, psycopg2-binary, and redis.

### Phase 2: Detection Depth

#### ML Ensemble Detector

The new ML ensemble (`core/ml_ensemble.py`) combines Isolation Forest, Random Forest, and optional XGBoost into a weighted voting system. An extended feature extractor computes 25 features (15 base metadata features plus 10 security-focused features) for each file. A labeled data manager with SQLite storage and CSV import/export supports training workflows, and an auto-retrainer tunes contamination parameters based on labeled feedback.

When XGBoost is not installed, the ensemble falls back to a two-model configuration (Isolation Forest + Random Forest) with no loss of functionality.

#### Behavioral Analysis

Behavioral analysis (`core/behavioral_analysis.py`) detects coordinated attack campaigns by correlating shared IOCs across scan results using union-find clustering, computing metadata pattern similarity via Jaccard thresholds, and grouping files by temporal proximity. A file relationship graph supports multi-hop traversal for threat campaign profiling -- connecting files that share C2 domains, similar metadata fingerprints, or overlapping modification timestamps.

#### MITRE ATT&CK Mapping

All HADES detection findings are now mapped to MITRE ATT&CK techniques and tactics (`core/mitre_mapping.py`). Coverage includes script injection (T1059), PHP backdoors (T1505.003), command injection (T1059.004), reverse shells (T1059), Base64 encoding (T1027), steganography (T1027.003), polyglot files (T1036.008), macro execution (T1204.002), and template injection (T1221). Lookup and matrix endpoints are available at `/api/v1/mitre/matrix` and `/api/v1/mitre/lookup`.

#### Threat Intelligence Scheduler

Automated threat feed ingestion (`core/threat_intel_scheduler.py`) supports STIX/TAXII, MISP, CSV, and custom feed formats. Each feed has a configurable polling interval, and feeds can be triggered manually through the CLI (`--schedule-feeds`) or the API at `/api/v1/threat-intel/scheduler`. Ingested indicators are merged into the existing threat intelligence cache for use during scans.

#### YARA Rule Builder

Programmatic YARA rule generation (`core/yara_rule_builder.py`) enables analysts to create rules from built-in templates covering metadata injection, steganography, polyglot files, and encoded payloads. Rules can be generated via the CLI (`--build-rule`, `--list-templates`) or the API at `/api/v1/rules/templates` and `/api/v1/rules/build`.

#### MalwareBazaar Validation Corpus

A standalone validation script (`scripts/run_malware_validation.py`) scans mocked or live MalwareBazaar samples and produces a detection matrix. The test suite (`tests/test_malwarebazaar_validation.py`) asserts TPR >= 90% and FPR <= 10% against the validation corpus.

#### Phase 2 API Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/api/v1/mitre/matrix` | GET | Full MITRE ATT&CK technique coverage matrix |
| `/api/v1/mitre/lookup` | GET | Look up ATT&CK mapping for a detection finding |
| `/api/v1/ml/ensemble/status` | GET | ML ensemble model status and configuration |
| `/api/v1/behavioral/campaigns` | GET | Detected behavioral campaigns |
| `/api/v1/threat-intel/scheduler` | GET/POST | Threat feed scheduler status and manual trigger |
| `/api/v1/rules/templates` | GET | Available YARA rule templates |
| `/api/v1/rules/build` | POST | Build a YARA rule from a template |

### Phase 3: Community & Adoption

#### MkDocs Documentation Site

A complete documentation site built with MkDocs and the Material theme provides 25+ pages covering installation, CLI reference, API reference, detection depth guides, enterprise deployment, architecture, plugin authoring, YARA rule writing, and contributing guidelines. Build and serve locally:

```bash
bash scripts/build_docs.sh
bash scripts/serve_docs.sh
```

#### Plugin Marketplace

The plugin marketplace (`plugins/registry.py`) provides a registry for discovering, installing, updating, and uninstalling plugins. Plugin packages are verified with SHA-256 checksums. Marketplace management is available through the CLI (`--plugin-search`, `--plugin-install`, `--plugin-uninstall`, `--plugin-check-updates`) and API endpoints at `/api/v1/plugins/marketplace/`.

Two new example plugins ship with this release:

- **Steganography Detector** (`plugins/example_steganography_detector.py`) -- Performs LSB (Least Significant Bit) analysis, detects appended data after EOF markers, and identifies tool signatures from common steganography utilities.
- **PII Scanner** (`plugins/example_pii_scanner.py`) -- Detects email addresses, phone numbers, Social Security numbers, and credit card numbers (with Luhn checksum validation) embedded in file metadata.

A plugin authoring guide (`docs/plugin_authoring_guide.md`) provides a step-by-step tutorial for building custom detection plugins.

#### YARA Rule Contribution Workflow

A formalized contribution workflow for community YARA rules defines naming conventions, required metadata fields (author, description, severity, MITRE mapping), and submission guidelines. An enhanced CI validator (`scripts/validate_rules_ci.py`) enforces metadata requirements, severity ranges, MITRE technique references, duplicate detection, and naming conventions.

#### GitHub Actions CI/CD

Two GitHub Actions workflows automate quality assurance:

- **Test Suite** (`.github/workflows/test-suite.yml`) -- Runs the full test suite across a Python 3.9-3.12 matrix on every push and pull request.
- **YARA Rule Validation** (`.github/workflows/validate-rules.yml`) -- Compiles and validates all YARA rules on every change to the `rules/` directory.

GitHub PR and issue templates (`.github/PULL_REQUEST_TEMPLATE.md`, `.github/ISSUE_TEMPLATE/`) standardize contributions.

#### Interactive Demo Environment

A self-contained demo environment provides a rate-limited, pre-configured instance with sample files and a landing page for evaluating HADES. Launch it via the CLI (`--demo-mode`, `--generate-demo-samples`) or the demo Docker image.

## Installation

### pip (Cloudsmith)

```bash
# Basic installation
pip install hades-scanner --extra-index-url https://dl.cloudsmith.io/basic/darkhorse/hades/python/simple/
hades --help

# Enterprise features
pip install "hades-scanner[enterprise]"

# Everything
pip install "hades-scanner[full]"
```

### From Source

```bash
git clone https://github.com/DarkHorse-InfoSec/hades-docs.git
cd hades && pip install -r requirements.txt
python cli/hades_cli.py --help
python core/hades_enhanced_cli.py --help
```

### Docker

```bash
# Standard deployment
docker compose -f docker/docker-compose.yml up -d

# Enterprise deployment (with PostgreSQL + Redis)
docker compose -f docker/docker-compose.yml --profile enterprise up -d

curl http://localhost:8666/api/v1/health
```

Open `http://localhost:8666/dashboard/` for the web interface.

## Migration from v0.5.0

### No Breaking Changes

There are no breaking API changes in v0.6.0. All new endpoints are additive, and the existing v0.5.0 API surface is fully preserved. Existing configurations work without modification.

### New Optional Dependencies

The `enterprise` dependency group is new in v0.6.0:

```bash
pip install "hades-scanner[enterprise]"
```

This installs bcrypt, PyJWT, cryptography, psycopg2-binary, and redis. None of these are required for existing functionality -- they enable the Phase 1 enterprise features.

XGBoost is optional for the ML ensemble. When not installed, the ensemble uses two models (Isolation Forest + Random Forest) instead of three.

### New Configuration Sections

The following sections have been added to `cli/hades_config.json`. All are optional and have sensible defaults:

- `database` -- Backend selection (sqlite/postgresql) and connection settings
- `redis` -- Redis URL and connection parameters
- `encryption` -- Encryption key and field-level encryption settings
- `auth` -- RBAC, SSO (OIDC/SAML), and license configuration
- `tenants` -- Multi-tenant isolation settings
- `ml_ensemble` -- Ensemble model selection and weights
- `behavioral_analysis` -- Campaign detection thresholds
- `threat_intel_scheduler` -- Feed URLs, polling intervals
- `yara_rule_builder` -- Template paths and output directory
- `mitre_mapping` -- ATT&CK technique coverage configuration
- `plugin_marketplace` -- Registry URL and verification settings
- `demo` -- Demo mode configuration

### Database Migration

For existing deployments with SQLite scan data, use the migration flag to update the schema:

```bash
python core/hades_enhanced_cli.py --migrate-db
```

This is a forward-only migration. Back up your database before migrating.

## Known Issues

- **SAML validation-only**: The SAML 2.0 provider validates IdP-initiated responses but does not yet support SP-initiated authentication flows. Full SP-initiated SAML is planned for a future release.
- **XGBoost optional**: The ML ensemble falls back to a two-model configuration (Isolation Forest + Random Forest) when XGBoost is not installed. Detection accuracy is marginally reduced without XGBoost.
- **Plugin marketplace local-only**: The plugin registry currently operates with local package files. A remote registry for community plugin discovery is planned.
- **PostgreSQL migrations forward-only**: Database schema migrations from SQLite to PostgreSQL and between PostgreSQL versions are forward-only. Always back up before migrating.
- **YARA modules**: `enterprise_threats.yar` imports the `pe`, `elf`, and `math` YARA modules. These require a YARA build compiled with module support.
- **pikepdf/olefile optional**: Deep format analysis for PDF and Office files requires pikepdf and olefile respectively. If not installed, the engine falls back to regex-based parsing.

## Test Coverage

- **1,409+ tests passing**, 0 failures (183 new tests in this release)
- **TPR >= 90%**, FPR <= 10% across 28 validated attack samples in the malware validation corpus
- Test matrix: Python 3.9, 3.10, 3.11, 3.12
- Enterprise, detection depth, and community features each have dedicated test suites

## What's Next (v0.7.0 Preview)

- **Remote plugin registry**: Centralized community plugin discovery and distribution
- **SP-initiated SAML**: Full SAML 2.0 SP-initiated authentication flow
- **Scheduled scanning**: Cron-style scan scheduling for automated periodic analysis
- **Report templates**: Customizable HTML/PDF report templates with organization branding
- **Rule auto-update**: Automatic YARA rule updates from a curated threat feed
- **PostgreSQL reverse migrations**: Bidirectional database migration support

## Full Changelog

See [CHANGELOG.md](CHANGELOG.md) for the complete list of changes.

## Contributors

- DarkHorse Information Security LLC
