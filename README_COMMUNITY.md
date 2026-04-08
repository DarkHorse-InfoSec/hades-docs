# HADES Community Edition

**Hidden Artifact Detection & EXIF Scanner**

A metadata forensics engine for security professionals, malware analysts, and incident responders. HADES analyzes hidden and malformed metadata across common file types (images, documents, video, audio, archives, web files) to identify threats -- without executing target files.

Built and maintained by DarkHorse Information Security LLC.

---

## Quick Start

### Install from PyPI

```bash
# Basic installation
pip install hades-scanner

# Full installation (all features)
pip install "hades-scanner[full]"
```

### Scan a File

```bash
# Basic scan
hades suspicious_file.jpg

# Recursive directory scan with verbose output
hades -r /path/to/evidence/ -v

# Enhanced scan with ML detection
hades-enhanced -r --ml-detect /path/to/evidence/
```

### Start the API Server

```bash
hades-enhanced --serve --port 8666
```

Then visit `http://localhost:8666/dashboard/` for the web interface.

---

## What HADES Does

HADES inspects file metadata for signs of compromise, including:

- **Script injection** in EXIF fields (PHP backdoors, JavaScript, SQL injection)
- **Steganography** indicators (LSB analysis, appended data, tool signatures)
- **Polyglot files** (files valid in multiple formats simultaneously)
- **Macro threats** in Office documents (VBA macros, DDE, template injection)
- **PDF threats** (JavaScript, OpenAction, embedded files, form submission)
- **SVG threats** (script injection, XXE, encoded payloads, event handlers)
- **Timestamp anomalies** (future dates, impossible EXIF timestamps)
- **Geolocation and PII leakage** (GPS coordinates, email addresses, phone numbers in metadata)
- **Base64 and encoded payloads** hidden in metadata fields
- **Command injection** and reverse shell patterns

All analysis is non-execution: HADES reads file structure and metadata without running potentially malicious code.

---

## Features by Tier

HADES is available in three tiers. All tiers include the full installation — your license key determines which features are active.

| Feature | Free | Professional | Enterprise |
|---------|:----:|:------------:|:----------:|
| **Core Scanning** | | | |
| CLI scanner (single file, directory, recursive) | x | x | x |
| YARA pattern matching with 7 rule sets | x | x | x |
| Deep format analysis (PDF, Office, SVG, polyglot) | x | x | x |
| REST API server with WebSocket support | x | x | x |
| Health check endpoint | x | x | x |
| Metadata sanitization | x | x | x |
| Async scan pipeline | x | x | x |
| YARA rule builder with templates | x | x | x |
| **Detection & Intelligence** | | | |
| ML anomaly detection (Isolation Forest) | | x | x |
| ML ensemble (Isolation Forest + Random Forest + XGBoost) | | x | x |
| Cloud threat intelligence (VirusTotal, AbuseIPDB, OTX, MalwareBazaar) | | x | x |
| Behavioral analysis and campaign detection | | x | x |
| MITRE ATT&CK technique mapping | | x | x |
| Threat intelligence feed scheduler | | x | x |
| **Operations & Integration** | | | |
| Evidence chain and case management | | x | x |
| Forensic audit trail with hash-chain verification | | x | x |
| SIEM integration (Syslog, CEF, STIX, LEEF, ECS, Kafka) | | x | x |
| File monitoring (watchdog) | | x | x |
| Plugin system with marketplace | | x | x |
| Web dashboard (scan, monitor, cases, audit, MITRE, rules, playbooks) | | x | x |
| Playbook engine (10 pre-built response automations) | | x | x |
| Distributed worker pool | | x | x |
| Scan result caching (LRU + Redis) | | x | x |
| Prometheus metrics exporter | | x | x |
| Grafana dashboards (4 pre-built) | | x | x |
| Scan analytics with trend/anomaly detection | | x | x |
| **Enterprise Security & Scale** | | | |
| RBAC (admin/analyst/viewer roles) | | | x |
| SSO (OIDC + SAML) | | | x |
| Cloud storage scanning (S3, GCS, Azure Blob) | | | x |
| CI/CD pipeline scanner (SARIF, GitHub, GitLab) | | | x |
| Chat bot integration (Slack, Teams) | | | x |
| Multi-tenant isolation | | | x |
| AES-256-GCM encryption at rest | | | x |
| PostgreSQL backend with connection pooling | | | x |
| Kubernetes Helm chart with HPA | | | x |
| Docker Compose HA deployments | | | x |

To upgrade, set your license key:

```bash
# Via environment variable
export HADES_LICENSE_KEY=HADES-PRO-XXXX-XXXX

# Or via CLI flag
hades-enhanced --license-key HADES-PRO-XXXX-XXXX --serve

# Check current license
hades-enhanced --license-info
```

---

## Installation Options

### PyPI (Recommended)

```bash
# Core scanner
pip install hades-scanner

# With specific extras
pip install "hades-scanner[api]"           # REST API
pip install "hades-scanner[yara]"          # YARA support
pip install "hades-scanner[ml]"            # ML detection
pip install "hades-scanner[enterprise]"    # RBAC, SSO, PostgreSQL, Redis
pip install "hades-scanner[observability]" # Prometheus metrics
pip install "hades-scanner[full]"          # Everything
```

### Docker

```bash
# Build the image
docker build -f docker/hades-full.dockerfile -t hades-scanner:latest .

# Run standalone
docker run -p 8666:8666 hades-scanner:latest

# Full stack with PostgreSQL, Redis, Prometheus, Grafana
docker compose -f docker/docker-compose.full-stack.yml up -d
```

### Kubernetes (Helm)

```bash
helm install hades kubernetes/helm/hades \
  --set api.replicas=3 \
  --set workers.replicas=4 \
  --set redis.url=redis://redis:6379
```

### Homebrew (macOS)

```bash
brew install hades-scanner
```

---

## CLI Reference

```bash
# Basic scanning
hades file.jpg                                    # Scan single file
hades -r /evidence/                               # Recursive directory scan
hades -r -v --json /evidence/                     # Verbose JSON output

# Enhanced scanner
hades-enhanced -r --file-types .jpg .png /path/   # Filter by file type
hades-enhanced -r --ml-detect /evidence/          # ML anomaly detection
hades-enhanced -r --ml-ensemble /evidence/        # ML ensemble (multi-model)
hades-enhanced -r --behavioral /evidence/         # Campaign detection
hades-enhanced -r --mitre-map /evidence/          # MITRE ATT&CK mapping
hades-enhanced --async --concurrency 8 -r /dir/   # Async pipeline

# API server
hades-enhanced --serve --port 8666                # Start REST API
hades-enhanced --serve --metrics                  # With Prometheus metrics
hades-enhanced --serve --distributed              # With worker pool

# Evidence management
hades-enhanced --case-create "Investigation Name"
hades-enhanced --case-id CASE_ID -r /evidence/
hades-enhanced --audit-verify
hades-enhanced --export-case CASE_ID

# File monitoring
hades-enhanced --monitor /path/to/watch --webhook https://hooks.example.com/alert

# SIEM export
hades-enhanced -r /evidence/ --siem-format cef
hades-enhanced --monitor /watched/ --siem-format syslog --siem-target 10.0.0.50:514

# Benchmarks
hades-enhanced --benchmark
hades-enhanced --benchmark-quick
```

---

## API Endpoints

The REST API runs at `http://localhost:8666` by default.

| Endpoint | Method | Tier | Description |
|----------|--------|------|-------------|
| `/api/v1/health` | GET | Free | Health check |
| `/api/v1/scan` | POST | Free | Scan file (multipart upload) |
| `/api/v1/scan/batch` | POST | Free | Batch scan multiple files |
| `/api/v1/scan/{id}` | GET | Free | Retrieve scan results |
| `/api/v1/sanitize` | POST | Free | Sanitize file metadata |
| `/api/v1/rules/templates` | GET | Free | YARA rule templates |
| `/api/v1/rules/build` | POST | Free | Build YARA rule |
| `/dashboard/` | GET | Free | Web dashboard |
| `/docs` | GET | Free | Swagger/OpenAPI docs |
| `/ws` | WS | Free | WebSocket for live updates |
| `/api/v1/evidence/*` | GET/POST | Pro | Evidence chain and case management |
| `/api/v1/mitre/*` | GET | Pro | MITRE ATT&CK mappings |
| `/api/v1/ml/ensemble/*` | GET | Pro | ML ensemble status |
| `/api/v1/behavioral/*` | GET | Pro | Campaign detection |
| `/api/v1/monitor/*` | GET/POST | Pro | File monitoring |
| `/api/v1/plugins/*` | GET/POST | Pro | Plugin management |
| `/api/v1/siem/*` | GET/POST/PUT | Pro | SIEM configuration and export |
| `/api/v1/export` | POST | Pro | SIEM export |
| `/api/v1/threat-intel/*` | GET/POST | Pro | Threat intel scheduler |
| `/api/v1/playbooks/*` | GET/POST | Pro | Playbook engine |
| `/metrics` | GET | Pro | Prometheus metrics |
| `/api/v1/metrics/*` | GET | Pro | JSON metrics and health |
| `/api/v1/auth/*` | GET/POST | Enterprise | RBAC user management |
| `/api/v1/integrations/cloud/*` | GET/POST | Enterprise | Cloud storage scanning |
| `/api/v1/integrations/cicd/*` | POST | Enterprise | CI/CD scanning |
| `/api/v1/integrations/slack/*` | POST | Enterprise | Slack bot |
| `/api/v1/integrations/teams/*` | POST | Enterprise | Teams bot |
| `/api/v1/admin/tenants/*` | GET/POST | Enterprise | Multi-tenant management |
| `/api/v1/admin/encryption/*` | GET | Enterprise | Encryption status |

---

## Detection Pipeline

```
File Input
    |
    v
[Metadata Extraction] -- ExifTool persistent process pool
    |
    v
[YARA Pattern Matching] -- 7 rule sets, 50+ rules
    |
    v
[Heuristic Analysis] -- Script injection, encoding, anomalies
    |
    v
[Deep Format Analysis] -- PDF, Office, SVG, polyglot
    |
    v
[ML Anomaly Detection] -- Isolation Forest + Random Forest + XGBoost
    |
    v
[Behavioral Analysis] -- Campaign correlation, IOC graph
    |
    v
[Threat Intel Enrichment] -- VirusTotal, AbuseIPDB, OTX, MalwareBazaar
    |
    v
[MITRE ATT&CK Mapping] -- Technique and tactic classification
    |
    v
[Unified Threat Score] -- 0-100 composite score with severity level
```

---

## Plugin Development

Create a `.py` file in `plugins/` that subclasses `HADESPlugin`:

```python
from core.plugin_api import HADESPlugin, Finding

class MyDetector(HADESPlugin):
    @property
    def name(self): return "my-detector"

    @property
    def version(self): return "1.0.0"

    @property
    def author(self): return "Your Name"

    def initialize(self): pass

    def supported_file_types(self):
        return [".jpg", ".png"]

    def analyze(self, file_path, metadata):
        findings = []
        # Your detection logic here
        if suspicious_condition:
            findings.append(Finding(
                title="Suspicious Pattern",
                description="Details...",
                severity=7.5,
                category="custom",
            ))
        return findings

    def cleanup(self): pass
```

See `docs/plugin_development_guide.md` for the full walkthrough.

---

## Feedback & Support

- **Bug reports**: Submit via the issue tracker
- **Security vulnerabilities**: Email info@darkhorsesecurity.com (do not file public issues)
- **Feature requests**: Submit via the issue tracker
- **Custom plugins**: See `docs/plugin_authoring_guide.md` for the plugin development guide

---

## Requirements

- Python 3.8+
- ExifTool (for metadata extraction)
- Optional: YARA (`yara-python` for pattern matching)
- Optional: Docker (for sandbox and deployment)
- Optional: Redis (for caching and distributed mode)
- Optional: PostgreSQL (for enterprise data storage)
- Optional: `prometheus_client` (for metrics)

---

## License

Proprietary License. See [LICENSE](LICENSE) for the full license agreement.

Copyright (c) 2024-2026 DarkHorse Information Security LLC. All rights reserved.

---

## Links

- Repository: https://github.com/DarkHorse-Security/HADES
- Documentation: https://github.com/DarkHorse-Security/HADES/tree/main/docs
- Issue Tracker: https://github.com/DarkHorse-Security/HADES/issues
- PyPI: https://pypi.org/project/hades-scanner/
- Changelog: https://github.com/DarkHorse-Security/HADES/blob/main/CHANGELOG.md

---

DarkHorse Information Security LLC
