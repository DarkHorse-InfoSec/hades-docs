# HADES Community Edition

**Hidden Artifact Detection & EXIF Scanner**

A metadata forensics engine for security professionals, malware analysts, and incident responders. HADES analyzes hidden and malformed metadata across common file types (images, documents, video, audio, archives, web files) to identify threats -- without executing target files.

Built and maintained by DarkHorse Information Security LLC.

---

## Quick Start

HADES ships as a single Nuitka-compiled signed binary with everything bundled (Python interpreter, ExifTool, YARA engine, ML model). Two install paths -- both gated by a license key from the customer portal:

```bash
# Path A: direct download from the portal (Linux x86_64 today)
export HADES_LICENSE_KEY="<paste from portal email>"
curl -fL -H "Authorization: Bearer $HADES_LICENSE_KEY" \
  "https://portal.darkhorseinfosec.com/api/v1/download/linux-x86_64/v1.4.2/hades" \
  -o hades && chmod +x hades

# Path B: Homebrew tap (macOS and Linux)
export HOMEBREW_HADES_LICENSE_KEY="<paste from portal email>"
brew tap DarkHorse-InfoSec/tap
brew install DarkHorse-InfoSec/tap/hades-scanner
```

Full install instructions + troubleshooting + platform notes: [docs/installation.md](docs/installation.md).

### Scan a File

```bash
# Single file
hades scan suspicious_file.jpg

# Recursive directory
hades scan -r /path/to/evidence/

# JSON output for downstream tools
hades scan -r --json /path/to/evidence/
```

### Start the API Server (Pro+ tier)

```bash
hades-server --port 8666
curl http://localhost:8666/api/v1/health
```

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

HADES is available in three tiers. The same binary ships to all tiers; your license key determines which features are active. Community (Free) tier users get a license key that unlocks the core scanning features below at no charge.

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

To activate or change your tier, set your license key:

```bash
# Via environment variable (one-shot)
export HADES_LICENSE_KEY=HADES-PRO-XXXX-XXXX

# Persistent across sessions (recommended)
mkdir -p ~/.hades && chmod 700 ~/.hades
echo "HADES_LICENSE_KEY=HADES-PRO-XXXX-XXXX" > ~/.hades/.env
chmod 600 ~/.hades/.env

# Verify the loaded tier
hades --version          # banner shows current tier
```

---

## Installation Paths

Two paths, both gated by the customer portal:

### Path A: direct download from the portal (canonical)

The most direct flow. License key in `Authorization: Bearer`, signed binary out, no source access required.

```bash
export HADES_LICENSE_KEY="<paste from portal email>"
curl -fL -H "Authorization: Bearer $HADES_LICENSE_KEY" \
  "https://portal.darkhorseinfosec.com/api/v1/download/linux-x86_64/v1.4.2/hades" \
  -o hades && chmod +x hades
```

The portal validates your license, generates a 15-minute HMAC-SHA256-signed URL to Cloudflare R2, and `curl` follows the redirect. The binary is RSA-PSS-signed and verifies its own integrity at first run.

### Path B: Homebrew tap (macOS and Linux)

```bash
export HOMEBREW_HADES_LICENSE_KEY="<paste from portal email>"
brew tap DarkHorse-InfoSec/tap
brew install DarkHorse-InfoSec/tap/hades-scanner
```

The formula uses `HadesPortalDownloadStrategy` to inject your license key as a Bearer token, hits the same portal endpoint as Path A, and stages the binary under `$(brew --prefix)/bin/hades`.

### Older install paths (no longer supported)

Pre-v1.4 distribution via PyPI (`pip install hades-scanner`), Docker Hub images at `darkhorse-security/hades-scanner`, the Cloudsmith private registry, public source-clone, and standalone `brew install hades-scanner` (no tap) are all no longer supported. For enterprise security-audit or source-review needs under NDA, contact `support@darkhorseinfosec.com`.

---

## CLI Reference

```bash
# Basic scanning
hades scan file.jpg                                # Scan single file
hades scan -r /evidence/                           # Recursive directory scan
hades scan -r --json /evidence/                    # JSON output for downstream tools

# Threat-intel-enriched scan (Pro+ tier; requires API keys in ~/.hades/.env)
hades scan --threat-intel /path/to/sample.exe

# YARA rule and IOC management
hades rules list
hades rules build --template <template-name>

# Evidence chain and case management (Pro+)
hades case create "Investigation Name"
hades case scan --case-id CASE_ID -r /evidence/
hades case audit-verify
hades case export --case-id CASE_ID --output /tmp/case.tar.gz

# File monitoring (Pro+)
hades monitor /path/to/watch --webhook https://hooks.example.com/alert

# SIEM export (Pro+)
hades scan -r /evidence/ --siem-format cef
hades monitor /watched/ --siem-format syslog --siem-target 10.0.0.50:514

# Threat intel feed management (Pro+)
hades intel update
hades intel status

# Diagnostics
hades doctor                                       # ~25 environment probes
hades --version                                    # version + tier banner
```

For the full subcommand list:

```bash
hades --help
hades <subcommand> --help
```

---

## API Endpoints

The REST API runs at `http://localhost:8666` by default. Start with `hades-server --port 8666`.

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
[Metadata Extraction] -- ExifTool (bundled)
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
[ML Anomaly Detection] -- Isolation Forest + Random Forest + XGBoost (Pro+)
    |
    v
[Behavioral Analysis] -- Campaign correlation, IOC graph (Pro+)
    |
    v
[Threat Intel Enrichment] -- VirusTotal, AbuseIPDB, OTX, MalwareBazaar (Pro+)
    |
    v
[MITRE ATT&CK Mapping] -- Technique and tactic classification (Pro+)
    |
    v
[Unified Threat Score] -- 0-100 composite score with severity level
```

---

## System Requirements

A 64-bit OS. The HADES binary is Nuitka-compiled with its Python interpreter, ExifTool, YARA engine, and all dependencies bundled inline, so the host needs no pre-installed runtime.

| OS | Status |
|---|---|
| Linux x86_64 (glibc 2.31+) | Path A + Path B (via Linuxbrew) |
| macOS Intel (10.15+) | Path B today; Path A binary on 2026-Q3 roadmap |
| macOS Apple Silicon (11.0+) | Path B today; Path A binary on 2026-Q3 roadmap |
| Windows 10 / 11 / Server 2019+ | Pending code-signing cert acquisition |

Pro+ tier features (Redis caching, PostgreSQL backend, Prometheus metrics, distributed workers) optionally use external services that you provision in your own infrastructure; the HADES binary connects to them but does not embed them.

---

## Feedback & Support

- **Bug reports:** Submit via the issue tracker on GitHub
- **Security vulnerabilities:** Email `security@darkhorseinfosec.com` (do not file public issues)
- **Feature requests:** Submit via the issue tracker
- **Custom plugins:** See `docs/plugin_development_guide.md` for the plugin development guide

---

## License

Proprietary License. See [LICENSE](LICENSE) for the full license agreement. Source code, YARA rule sets, scoring algorithms, and ML feature definitions are also designated trade secrets under the Defend Trade Secrets Act of 2016 (18 U.S.C. Sections 1836-1839) and corresponding state Uniform Trade Secrets Act statutes; see `TRADE_SECRETS.md` in the source repository for the formal designation. For enterprise security-audit or source-review needs under NDA, contact `support@darkhorseinfosec.com`.

Copyright (c) 2024-2026 DarkHorse Information Security LLC. All rights reserved.

---

## Links

- Repository: https://github.com/DarkHorse-InfoSec/hades-docs
- Documentation: https://github.com/DarkHorse-InfoSec/hades-docs/tree/main/docs
- Issue Tracker: https://github.com/DarkHorse-InfoSec/hades-docs/issues
- Customer portal: https://portal.darkhorseinfosec.com
- Live demo: https://demo.darkhorseinfosec.com
- Marketing site: https://darkhorseinfosec.com/hades
- Changelog: https://github.com/DarkHorse-InfoSec/hades-docs/blob/main/CHANGELOG.md

---

DarkHorse Information Security LLC
