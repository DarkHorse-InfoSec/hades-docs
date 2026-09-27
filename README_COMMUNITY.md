# HADES Community Edition

**Hidden Artifact Detection & EXIF Scanner**

A metadata forensics engine for security professionals, malware analysts, and incident responders. HADES analyzes hidden and malformed metadata across common file types (images, documents, video, audio, archives, web files) to identify threats -- without executing target files.

Built and maintained by DarkHorse Information Security LLC.

---

## Quick Start

HADES ships as a single Nuitka-compiled signed binary with its Python interpreter, YARA engine and ML model bundled; ExifTool is optional (a native fallback ships in the binary). Two install paths, both gated by a license key from the customer portal:

```bash
# Path A: direct download from the portal (Linux x86_64 shown; Windows x86_64 also available)
export HADES_LICENSE_KEY="<paste from portal email>"
curl -fL -H "Authorization: Bearer $HADES_LICENSE_KEY" \
  "https://portal.darkhorseinfosec.com/api/v1/download/linux-x86_64/latest/hades" \
  -o hades && chmod +x hades

# Path B: Homebrew tap (Linux, via Linuxbrew)
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

HADES has four tiers. The same binary ships to all of them; the license key determines which features are active. Without a license key HADES runs as Community (Free). If a license key is set but fails validation (expired, bad signature, tampered), HADES stops with an error rather than silently dropping to Community.

The tables below are taken from the license module of HADES v1.7.1 (`core/auth/license.py`: `TIER_FEATURES`, `ANALYZER_TIERS`, `TIER_QUOTA_DEFAULTS`).

| | Community (Free) | Professional | Team | Enterprise |
|---|:---:|:---:|:---:|:---:|
| **Price** | Free | $99/mo or $799/yr | $299/mo or $2,499/yr | Custom |
| **Detection stages** | IOC and heuristic analysis only | Full engine | Full engine | Full engine |
| YARA rule matching | | x | x | x |
| ML detection | | x | x | x |
| Deep format analyzers (PDF, Office, archives and the rest of the engine) | | x | x | x |
| **Licensed features** | | | | |
| CLI scanning | x | x | x | x |
| Basic REST API and health endpoint | x | x | x | x |
| Evidence chain | x | x | x | x |
| Scan analytics | x | x | x | x |
| Threat intelligence lookups and MITRE ATT&CK mapping | | x | x | x |
| SIEM export | | x | x | x |
| Monitoring and playbooks | | x | x | x |
| Plugins | | x | x | x |
| RBAC and SSO | | | x | x |
| Cloud storage scanning | | | x | x |
| CI/CD integration | | | x | x |
| Chat integrations (Slack, Teams) | | | x | x |
| Encrypted storage | | | x | x |
| Multi-tenant administration | | | | x |
| **Default quotas** | | | | |
| Scans per month (CLI) | 10 | 5,000 | 50,000 | Unlimited |
| Scans per month (API) | 5 | 2,500 | 25,000 | Unlimited |
| Batch scanning | No | Yes | Yes | Yes |
| Maximum file size | 10 MB | 100 MB | 500 MB | Unlimited |

Community (Free) runs the IOC and heuristic stages only; YARA, ML and the deep format analyzers need Professional or higher. A license can carry its own quota values, which override the defaults above.

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
  "https://portal.darkhorseinfosec.com/api/v1/download/linux-x86_64/latest/hades" \
  -o hades && chmod +x hades
```

The portal validates your license, generates a 15-minute HMAC-SHA256-signed URL to Cloudflare R2, and `curl` follows the redirect. Windows binaries are Authenticode-signed (self-signed certificate) and RFC3161-timestamped; the bundled ML model is RSA-PSS-signed and is not loaded if its signature fails to verify.

### Path B: Homebrew tap (Linux, via Linuxbrew)

```bash
export HOMEBREW_HADES_LICENSE_KEY="<paste from portal email>"
brew tap DarkHorse-InfoSec/tap
brew install DarkHorse-InfoSec/tap/hades-scanner
```

The formula uses `HadesPortalDownloadStrategy` to inject your license key as a Bearer token, hits the same portal endpoint as Path A, and stages the binary under `$(brew --prefix)/bin/hades`.

### Older install paths (no longer supported)

Pre-v1.4 distribution via pip from the Cloudsmith private registry (`pip install hades-scanner`; HADES is not on public PyPI), Docker Hub images at `darkhorse-security/hades-scanner`, public source-clone, and standalone `brew install hades-scanner` (no tap) are all no longer supported. For enterprise security-audit or source-review needs under NDA, contact `support@darkhorseinfosec.com`.

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

Tier is the lowest tier whose license unlocks the route (v1.7.1 `core/hades_api.py` route gates). Pro, Team and Enterprise each include every lower tier.

| Endpoint | Method | Tier | Description |
|----------|--------|------|-------------|
| `/api/v1/health` | GET | Free | Health check |
| `/api/v1/scan` | POST | Free | Scan file (multipart upload) |
| `/api/v1/scan/batch` | POST | Pro | Batch scan multiple files |
| `/api/v1/scan/{id}` | GET | Free | Retrieve scan results |
| `/api/v1/sanitize` | POST | Free | Sanitize file metadata |
| `/api/v1/rules/templates` | GET | Free | YARA rule templates |
| `/api/v1/rules/build` | POST | Free | Build YARA rule |
| `/dashboard/` | GET | Free | Web dashboard |
| `/docs` | GET | Free | Swagger/OpenAPI docs |
| `/ws` | WS | Free | WebSocket for live updates |
| `/api/v1/evidence/*` | GET/POST | Free | Evidence chain and case management |
| `/api/v1/mitre/*` | GET | Pro | MITRE ATT&CK mappings |
| `/api/v1/ml/ensemble/*` | GET | Pro | ML ensemble status |
| `/api/v1/behavioral/*` | GET | Pro | Campaign detection |
| `/api/v1/monitor/*` | GET/POST | Pro | File monitoring |
| `/api/v1/plugins/*` | GET/POST | Pro | Plugin management |
| `/api/v1/siem/*` | GET/POST/PUT | Pro | SIEM configuration and export |
| `/api/v1/export` | POST | Pro | SIEM export |
| `/api/v1/threat-intel/*` | GET/POST | Pro | Threat intel scheduler |
| `/api/v1/playbooks/*` | GET/POST | Pro | Playbook engine |
| `/metrics` | GET | Free | Prometheus metrics |
| `/api/v1/metrics/*` | GET | Free | JSON metrics and health |
| `/api/v1/auth/*` | GET/POST | Team | RBAC user management |
| `/api/v1/integrations/cloud/*` | GET/POST | Team | Cloud storage scanning |
| `/api/v1/integrations/cicd/*` | POST | Team | CI/CD scanning |
| `/api/v1/integrations/slack/*` | POST | Team | Slack bot |
| `/api/v1/integrations/teams/*` | POST | Team | Teams bot |
| `/api/v1/admin/tenants/*` | GET/POST | Enterprise | Multi-tenant management |
| `/api/v1/admin/encryption/*` | GET | Team | Encryption status |

---

## Detection Pipeline

```
File Input
    |
    v
[Metadata Extraction] -- ExifTool (bundled)
    |
    v
[YARA Pattern Matching] -- 55 rule files, 144 rules
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

A 64-bit OS. The HADES binary is Nuitka-compiled with its Python interpreter, YARA engine, and all Python dependencies bundled inline, so the host needs no pre-installed runtime. ExifTool is optional for scanning (a native fallback ships in the binary) and recommended for the metadata sanitizer.

| OS | Status |
|---|---|
| Linux x86_64 (glibc 2.34+) | Path A + Path B (via Linuxbrew) |
| Windows x86_64 | Path A; binaries are Authenticode-signed (self-signed certificate) and RFC3161-timestamped |
| macOS | Not yet available; no release date |

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
