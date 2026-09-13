# HADES - Hidden Artifact Detection & EXIF Scanner

**Metadata-specific threat detection for security professionals, malware analysts, and incident responders.**

HADES analyzes hidden and malformed metadata across common file types -- images, documents, video, audio, archives, and web files -- to identify threats **without executing target files**. Where traditional scanners focus on file content, HADES goes deeper into the metadata layer where attackers hide C2 URLs, embedded payloads, polyglot constructs, and exfiltration channels.

---

## Capabilities

<div class="grid cards" markdown>

-   **Metadata Threat Detection**

    ---

    YARA rules and heuristic analysis purpose-built for metadata layers: EXIF, XMP, IPTC, PDF info dictionaries, Office document properties, and SVG attributes.

-   **ML Anomaly Scoring**

    ---

    Isolation Forest and ensemble models trained on 25 metadata features flag files that deviate from normal patterns -- catching novel threats that rules miss.

-   **Threat Intelligence**

    ---

    Cloud enrichment via VirusTotal, AbuseIPDB, OTX AlienVault, and MalwareBazaar with consensus scoring, caching, and offline mode.

-   **Real-Time Monitoring**

    ---

    Watchdog-based directory monitoring auto-scans new and modified files with configurable alert thresholds and webhook delivery.

-   **Evidence Chain**

    ---

    SHA-256 hash-chained audit logs, case management with scan linking, and self-verifying evidence export packages for forensic integrity.

-   **Enterprise Ready**

    ---

    RBAC, SSO (OIDC/SAML), PostgreSQL, Redis, encryption at rest, multi-tenancy, and license tiers for production deployments.

</div>

---

## Quick Install

HADES ships as a single Nuitka-compiled binary with everything bundled (Python interpreter, ExifTool, YARA engine, ML model). Two install paths:

```bash
# Path A: direct download from the portal (Linux x86_64 today; macOS + Windows on 2026-Q3 roadmap)
export HADES_LICENSE_KEY="<paste from portal email>"
curl -fL -H "Authorization: Bearer $HADES_LICENSE_KEY" \
  "https://portal.darkhorseinfosec.com/api/v1/download/linux-x86_64/latest/hades" \
  -o hades && chmod +x hades

# Path B: Homebrew tap (macOS and Linux)
export HOMEBREW_HADES_LICENSE_KEY="<paste from portal email>"
brew tap DarkHorse-InfoSec/tap
brew install DarkHorse-InfoSec/tap/hades-scanner
```

Full install instructions + troubleshooting: see [Installation Guide](installation.md).

## Quick Example

Scan a suspicious file:

```bash
hades-enhanced suspicious_image.jpg
```

Sample output:

```
HADES Metadata Forensics Engine v0.7.1
Scanning: suspicious_image.jpg
  [YARA]       embedded_exe matched (severity: 8)
  [HEURISTIC]  Base64-encoded payload in EXIF:Comment (452 chars)
  [ML]         Anomaly score: 0.87 (entropy, null_byte_ratio)
  [DEEP]       Polyglot detected: JPEG + ZIP archive
  [MITRE]      T1027 Obfuscated Files, T1036.008 Masquerading

  Threat Score: 85/100 (CRITICAL)
  Findings: 5 | YARA: 1 | Heuristic: 1 | ML: 1 | Deep Format: 1
```

Start the API server:

```bash
hades-server --port 8666
```

Scan via the REST API:

```bash
curl -X POST http://localhost:8666/api/v1/scan/file \
  -H "X-API-Key: your-api-key-here" \
  -F "file=@suspicious_image.jpg"
```

---

## What Makes HADES Unique

HADES occupies a distinct niche in the security tooling landscape. It is the only purpose-built scanner that performs **metadata-specific threat detection** with ML anomaly scoring, YARA rules, and deep format analysis all focused on the metadata layer.

| Capability | HADES | ExifTool | FOCA | Loki/THOR | Strelka |
|---|:---:|:---:|:---:|:---:|:---:|
| Metadata extraction | Yes | Yes | Yes | Limited | Limited |
| Metadata threat detection | **Yes** | No | No | File-level | Broad strokes |
| Metadata-specific YARA rules | **144 rules** | No | No | General rules | General rules |
| ML anomaly on metadata features | **Yes** | No | No | No | No |
| Polyglot file detection | **Yes** | No | No | Limited | Yes |
| Deep format analysis (PDF/Office/SVG) | **Yes** | No | Limited | Yes | Yes |
| MITRE ATT&CK mapping | **Yes** | No | No | Yes | No |
| Behavioral campaign detection | **Yes** | No | No | No | No |
| Evidence chain / case management | **Yes** | No | No | No | No |
| SIEM integration (CEF/STIX/LEEF/ECS) | **Yes** | No | No | Syslog | JSON |
| Real-time file monitoring | **Yes** | No | No | Yes | Yes |
| REST API + WebSocket | **Yes** | No | No | No | Yes |
| Plugin system | **Yes** | No | No | No | Yes |

---

## By the Numbers

| Metric | Value |
|---|---|
| Test suite | 1,781+ tests passing |
| YARA rules | 144 across 55 rule files |
| Detection corpus TPR | 100% (28 attack samples) |
| Detection corpus FPR | 0% (10 clean baselines) |
| Supported file formats | 15+ (JPEG, PNG, GIF, TIFF, PDF, DOCX, SVG, MP4, MP3, ZIP, RAR, HTML, and more) |
| ML features | 25 metadata-derived features |
| SIEM formats | 5 (Syslog, CEF, STIX, LEEF, ECS) |
| Threat intel providers | 4 (VirusTotal, AbuseIPDB, OTX, MalwareBazaar) |

---

## Next Steps

- [Quick Start Guide](quick_start.md) -- Get scanning in 5 minutes
- [API Reference](api_reference.md) -- REST API endpoint documentation
- [Enterprise Deployment](enterprise_deployment_guide.md) -- RBAC, SSO, PostgreSQL, Redis
- [Plugin Development](plugin_development_guide.md) -- Extend HADES with custom detections
- [Architecture](architecture.md) -- System design and module relationships

---

*Built by DarkHorse Information Security LLC.*
