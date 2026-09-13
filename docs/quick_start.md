# HADES Quick Start Guide

This guide is written for incident response and SOC analysts who need to get HADES running and scan their first file. No prior HADES experience is assumed.

---

## What HADES Does

HADES extracts and analyzes metadata from files -- images, documents, PDFs, archives, firmware -- to find hidden threats without ever executing the file. It flags code injection in EXIF fields, malicious macros, polyglot files, steganography, GPS/PII leakage, and more. Think of it as a metadata-level triage tool that sits before your sandbox or detonation chamber.

---

## Prerequisites

You need a 64-bit OS and your `HADES_LICENSE_KEY` from the license email. That's it. The HADES binary is Nuitka-compiled with its Python interpreter, ExifTool, YARA engine, and ML model bundled inline, so the host needs no pre-installed runtime.

Linux x86_64 (glibc 2.31+) and macOS (via Homebrew tap) are shipping today. Windows binary is on the 2026-Q3 roadmap pending code-signing certificate acquisition.

---

## Installation

### Path A: direct download from the portal (canonical)

```bash
export HADES_LICENSE_KEY="<paste from portal email>"

# Linux x86_64
curl -fL -H "Authorization: Bearer $HADES_LICENSE_KEY" \
  "https://portal.darkhorseinfosec.com/api/v1/download/linux-x86_64/latest/hades" \
  -o hades && chmod +x hades
```

The portal validates your license, generates a 15-minute HMAC-SHA256-signed URL, and `curl` follows the redirect. The binary is RSA-PSS-signed and verifies its own integrity at first run.

### Path B: Homebrew tap (macOS and Linux convenience)

```bash
export HOMEBREW_HADES_LICENSE_KEY="<paste from portal email>"
brew tap DarkHorse-InfoSec/tap
brew install DarkHorse-InfoSec/tap/hades-scanner
```

For full install details + troubleshooting see `docs/installation.md`.

---

## Your First Scan

Scan a single file from the command line:

```bash
hades scan suspicious_file.jpg
```

### Reading the Output

A scan produces output like this:

```
File: suspicious_file.jpg
Threat Score: 82/100
Severity: HIGH

Findings:
  [YARA]       php_backdoor_in_exif    (severity: CRITICAL)
               PHP code detected in EXIF Comment field
  [HEURISTIC]  base64_payload          (severity: HIGH)
               Base64-encoded executable content in UserComment
  [IOC]        suspicious_url          (severity: MEDIUM)
               URL found in EXIF Artist field: http://evil.example.com/shell.php

Metadata Fields Analyzed: 47
YARA Rules Matched: 1
Detection Methods: YARA, Heuristic, IOC
```

**Field-by-field breakdown:**

| Field | Meaning |
|-------|---------|
| **Threat Score** | Composite score from 0 (clean) to 100 (confirmed malicious). Combines all detection methods. |
| **Severity** | Human-readable label derived from the score (see table below). |
| **Findings** | Individual detections, each tagged with its source method (YARA, HEURISTIC, IOC, ML). |
| **Metadata Fields Analyzed** | How many metadata fields HADES extracted and inspected. |
| **YARA Rules Matched** | Count of YARA signature matches. |
| **Detection Methods** | Which detection engines contributed findings. |

---

## Understanding Threat Scores

| Score Range | Severity | What it means | Recommended action |
|-------------|----------|---------------|--------------------|
| 0 -- 24 | LOW | No significant findings. File metadata appears normal. | No action needed. |
| 25 -- 49 | MODERATE | Minor anomalies detected. Could be benign but worth noting. | Review findings. Log for correlation. |
| 50 -- 74 | HIGH | Suspicious patterns found. Likely warrants investigation. | Escalate to senior analyst. Isolate the file. |
| 75 -- 89 | HIGH | Strong indicators of malicious content. | Escalate immediately. Do not open the file. Create a case. |
| 90 -- 100 | CRITICAL | Confirmed malicious patterns (e.g., embedded code, known malware signatures). | Quarantine. Create a case. Forward to IR team. |

When in doubt, treat anything scored 50 or above as worth investigating.

---

## Batch Scanning

### Scan an entire directory

```bash
hades scan /path/to/evidence/
```

### Scan recursively (include subdirectories)

```bash
hades scan -r /path/to/evidence/
```

### Filter by file type

```bash
hades scan -r --file-types .jpg .png .pdf /path/to/evidence/
```

### Verbose output (shows every metadata field inspected)

```bash
hades scan -r -v /path/to/evidence/
```

---

## Starting the API Server

For team use, start the REST API server (Pro+ tier) so multiple analysts can submit scans through API calls:

```bash
hades-server --port 8666
```

Verify it is running:

```bash
curl http://localhost:8666/api/v1/health
```

### Scanning via the API

```bash
curl -X POST http://localhost:8666/api/v1/scan/file \
  -H "X-API-Key: your-api-key-here" \
  -F "file=@suspicious_file.jpg"
```

### Batch scan via the API

```bash
curl -X POST http://localhost:8666/api/v1/scan/batch \
  -H "X-API-Key: your-api-key-here" \
  -F "files=@image1.jpg" \
  -F "files=@document.pdf" \
  -F "files=@archive.zip"
```

---

## Using the Web Dashboard

Once the API server is running, open your browser to:

```
http://localhost:8666/dashboard/
```

The dashboard provides:

- **Scan view** -- Drag and drop files to scan, view results in the browser.
- **Monitor view** -- Start/stop directory monitoring, see live alerts.
- **Cases view** -- Create investigation cases, link scans, add analyst notes.
- **Audit trail** -- Review the hash-chained audit log for evidence integrity.
- **MITRE ATT&CK view** -- See findings mapped to ATT&CK techniques.
- **Playbooks view** -- Manage automated response playbooks (quarantine, alert, escalate).
- **Quarantine view** -- Review and manage quarantined files.

No additional software is needed -- the dashboard is a self-contained web application served by the API server.

---

## Creating a Case

When you find something worth investigating, create a case to track your work:

```bash
hades case create "Phishing attachment from ticket INC-4821"
```

This returns a case ID. Link subsequent scans to it:

```bash
hades scan --case-id <CASE_ID> suspicious_attachment.docx
```

Export the case as a self-verifying evidence package:

```bash
hades case export <CASE_ID>
```

This creates a ZIP file containing scan results, the audit log, file hashes, and an HMAC signature for chain-of-custody verification.

---

## Next Steps

| Topic | Guide |
|-------|-------|
| Comprehensive scanning workflows | [docs/scanning_guide.md](scanning_guide.md) |
| Automated response playbooks | [docs/playbook documentation](../core/playbook_engine.py) |
| Case management and evidence chain | [docs/first_scan.md](first_scan.md) |
| SIEM integration (Splunk, QRadar, Elastic) | [docs/siem_integration_guide.md](siem_integration_guide.md) |
| Threat intelligence feeds | [docs/threat_intel_guide.md](threat_intel_guide.md) |
| Full REST API reference | [docs/api_reference.md](api_reference.md) |
| Deployment and production hardening | [docs/deployment_guide.md](deployment_guide.md) |
| ML detection tuning | [docs/ml_detection_guide.md](ml_detection_guide.md) |
