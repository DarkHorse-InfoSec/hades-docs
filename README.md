# HADES: Hidden Artifact Detection & EXIF Scanner

**The metadata forensics engine for security professionals.**

![Version](https://img.shields.io/badge/Version-1.1.0-red)
![Python](https://img.shields.io/badge/Python-3.9%2B-blue)
![License](https://img.shields.io/badge/License-Proprietary-red)
![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey)

Built by [DarkHorse Information Security LLC](https://github.com/DarkHorse-InfoSec).

---

## What is HADES?

HADES scans files for hidden threats buried in metadata, detects malicious patterns using YARA rules and machine learning, and produces court-ready forensic evidence -- all without executing target files. It is built for malware analysts, incident responders, and forensic investigators who need to analyze images, documents, archives, and other file types at the metadata level.

---

## Key Capabilities

- **Deep metadata extraction** -- EXIF, IPTC, XMP, document properties, and embedded fields across images, documents, video, audio, archives, and web files
- **Threat detection** -- YARA rules, ML anomaly detection (Isolation Forest + Random Forest + XGBoost ensemble), IOC matching, and heuristic scoring
- **Deep file format analysis** -- Native parsing for PDF (JavaScript, OpenAction, embedded files), Office (macros, OLE, DDE, template injection), SVG threats, and polyglot file detection
- **PDF advanced analysis** -- Shadow attacks (incremental update abuse), stream filter chain obfuscation, and XDP container detection
- **Office advanced analysis** -- Hidden text, track changes, OLE Equation Editor exploits (CVE-2017-11882), formula injection, SVG CSS injection, custom XML inspection, and VBA stomping (source/p-code mismatch) detection
- **Image forensics** -- Decompression bomb detection (pixel flood), ICC profile injection, EXIF thumbnail mismatch, IPTC injection, TIFF IFD chain validation, and GIF frame/comment analysis
- **Archive forensics** -- ZIP structure validation (Zombie ZIP, concatenation, bombs, CRC32 mismatches), RAR5/7z header analysis, archive depth tracking, self-extracting archive detection (PE/ELF/Mach-O + archive), and Zip64 field abuse detection
- **Audio forensics** -- ID3v2 tag analysis, WAV chunk injection, OGG Vorbis comment abuse, FLAC metadata anomalies, and MP4/M4A atom inspection
- **Video container forensics** -- MKV subtitle/attachment abuse, MP4 atom injection, AVI chunk injection, FLV script data, and HLS/DASH playlist analysis
- **Advanced steganography detection** -- JPEG DCT-domain analysis, palette-based stego detection, EOF appended data, stego tool artifact scanning, and LSB distribution analysis
- **Anti-forensic detection** -- Timestamp stomping, Unicode homoglyph attacks, script encoding tricks, and MIME/extension/magic consistency validation
- **Firmware threat detection** -- UEFI capsules, BadUSB/DuckyScript payloads, boot sector analysis, and wiper malware signatures
- **Campaign detection and behavioral analysis** -- Correlate shared IOCs, metadata patterns, and temporal clusters across file sets to identify coordinated campaigns
- **MITRE ATT&CK technique mapping** -- Automatically map findings to ATT&CK techniques and tactics
- **Court-ready forensic evidence chain** -- Hash-chained append-only audit log, case management with scan linking, and self-verifying evidence export packages
- **Real-time file monitoring** -- Watch directories for new or modified files with automated scanning, configurable thresholds, and webhook alerts
- **SIEM export** -- Output findings in Syslog, CEF, STIX, LEEF, and ECS formats with real-time forwarding
- **REST API + Web Dashboard + CLI** -- FastAPI server with WebSocket support, browser-based dashboard, and full-featured command line interface
- **Plugin system** -- Extensible detection framework with marketplace, hot-reload, and sandboxed execution

HADES covers 32 forensic detection categories across images, documents, archives, audio, video, firmware, and streaming formats with 55 YARA rule files carrying 144 detection rules.

---

## Quick Start

```bash
# Path A: direct download from the customer portal (Linux x86_64 today)
export HADES_LICENSE_KEY="<paste from portal email>"
curl -fL -H "Authorization: Bearer $HADES_LICENSE_KEY" \
  "https://portal.darkhorseinfosec.com/api/v1/download/linux-x86_64/latest/hades" \
  -o hades && chmod +x hades

# Or Path B: Homebrew tap (macOS and Linux)
export HOMEBREW_HADES_LICENSE_KEY="<paste from portal email>"
brew tap DarkHorse-InfoSec/tap && brew install DarkHorse-InfoSec/tap/hades-scanner

# Scan a file
hades scan suspicious_file.jpg

# Recursive directory scan
hades scan -r /path/to/evidence/

# Start the REST API server (Pro+ tier)
hades-server --port 8666
```

For a detailed walkthrough aimed at IR and SOC analysts, see [docs/quick_start.md](docs/quick_start.md).

For scanning workflows, result interpretation, and case management, see [docs/scanning_guide.md](docs/scanning_guide.md).

---

## CLI Subcommands

| Command | Description |
|---------|-------------|
| `hades scan` | Scan files and directories for metadata threats |
| `hades monitor` | Watch directories for new files and auto-scan |
| `hades case` | Evidence chain and case management |
| `hades serve` | Start the REST API server and web dashboard |
| `hades benchmark` | Run performance benchmarks |
| `hades plugin` | Manage detection plugins |
| `hades intel` | Threat intelligence feed management |
| `hades rules` | YARA rule management and builder |
| `hades admin` | User, license, and system administration |

Run `hades <command> --help` for detailed usage on any subcommand.

---

## Installation

### Prerequisites

A 64-bit OS. The HADES binary is Nuitka-compiled with its Python interpreter, ExifTool, YARA engine, and all dependencies bundled inline, so the host needs no pre-installed runtime.

| OS | Status |
|---|---|
| Linux x86_64 (glibc 2.31+) | Path A + Path B |
| macOS Intel (10.15+) | Path B today; Path A binary on 2026-Q3 roadmap |
| macOS Apple Silicon (11.0+) | Path B today; Path A binary on 2026-Q3 roadmap |
| Windows 10 / 11 / Server 2019+ | Pending code-signing cert acquisition |

See [Installation Guide](docs/installation.md) for full details, troubleshooting, and platform notes.

### Older install paths (no longer supported)

Pre-v1.4 distribution via PyPI (`pip install hades-scanner`), Docker Hub images at `darkhorse-security/hades-scanner`, the Cloudsmith private registry at `dl.cloudsmith.io/basic/darkhorse/hades`, and public source-clone are no longer supported. Path A and Path B are the only canonical install paths. For enterprise security-audit or source-review needs under NDA, contact `support@darkhorseinfosec.com`.

---

## Enterprise Features

HADES supports production deployments with:

- **Access control** -- RBAC with admin/analyst/viewer roles, SSO via OIDC and SAML, API key authentication
- **Data security** -- AES-256-GCM field-level encryption at rest, multi-tenant isolation
- **Storage backends** -- PostgreSQL with connection pooling, Redis for state management and caching
- **Observability** -- Prometheus metrics exporter with 20+ metrics, four pre-built Grafana dashboards, Alertmanager integration
- **Deployment** -- Kubernetes Helm chart with HPA, Kustomize overlays, Docker Compose configurations for horizontal scaling and high availability
- **Playbook automation** -- Event-driven response engine with configurable triggers and action chains (case creation, SIEM forwarding, Slack/Teams notification, webhooks, evidence export, quarantine)

---

## Architecture

```
                         +------------------+
                         |   Input Files    |
                         +--------+---------+
                                  |
                    +-------------v--------------+
                    |    Metadata Extraction      |
                    |  (ExifTool + native parsers) |
                    +-------------+--------------+
                                  |
              +-------------------v--------------------+
              |         Detection Pipeline             |
              |  YARA | ML Ensemble | Heuristics | IOC |
              |  Deep Format | Firmware | Behavioral   |
              |  Stego | Anti-Forensic | Archive/Media  |
              +-------------------+--------------------+
                                  |
                    +-------------v--------------+
                    |      Evidence Chain         |
                    | Audit log | Case management |
                    +-------------+--------------+
                                  |
              +--------+----------+----------+---------+
              |        |          |          |         |
           +--v--+  +--v--+  +---v---+  +---v--+  +--v---+
           | CLI |  | API |  | SIEM  |  | Dash |  |Plugin|
           +-----+  +-----+  +-------+  +------+  +------+
```

---

## Documentation

| Document | Location |
|----------|----------|
| Quick Start | [docs/quick_start.md](docs/quick_start.md) |
| Scanning Guide | [docs/scanning_guide.md](docs/scanning_guide.md) |
| API Reference | [docs/api_reference.md](docs/api_reference.md) |
| Plugin Development | [docs/plugin_development_guide.md](docs/plugin_development_guide.md) |
| SIEM Integration | [docs/siem_integration_guide.md](docs/siem_integration_guide.md) |
| Deployment Guide | [docs/deployment_guide.md](docs/deployment_guide.md) |
| Enterprise Deployment | [docs/enterprise_deployment_guide.md](docs/enterprise_deployment_guide.md) |

---

## License

Proprietary and confidential. Copyright (c) 2024-2026 DarkHorse Information Security LLC. All rights reserved. See [LICENSE](LICENSE) for the full license agreement.

---

**DarkHorse Information Security LLC**
