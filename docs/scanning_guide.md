# HADES Scanning Guide

This guide covers every way to scan files with HADES, how to interpret results, handle false positives, and integrate scanning into your incident response workflow.

---

## Single File Scanning (CLI)

The most basic operation -- scan one file:

```bash
hades scan suspicious_file.jpg
```

Add verbose output to see every metadata field inspected:

```bash
hades scan -v suspicious_file.jpg
```

Output JSON for machine processing:

```bash
hades scan --json suspicious_file.jpg
```

---

## Directory Scanning

### Scan all files in a directory

```bash
hades scan /path/to/evidence/
```

### Recursive scan (include all subdirectories)

```bash
hades scan -r /path/to/evidence/
```

### Filter by file type

Only scan images and PDFs:

```bash
hades scan -r --file-types .jpg .png .gif .pdf /path/to/evidence/
```

### Verbose recursive scan

```bash
hades scan -r -v /path/to/evidence/
```

---

## Batch Scanning via API

When the API server is running (`hades serve --port 8666`), you can submit files over HTTP.

### Single file

```bash
curl -X POST http://localhost:8666/api/v1/scan/file \
  -H "X-API-Key: your-api-key-here" \
  -F "file=@suspicious_file.jpg"
```

### Multiple files in one request

```bash
curl -X POST http://localhost:8666/api/v1/scan/batch \
  -H "X-API-Key: your-api-key-here" \
  -F "files=@image1.jpg" \
  -F "files=@document.pdf" \
  -F "files=@archive.zip"
```

### Retrieve a previous scan result

```bash
curl -H "X-API-Key: your-api-key-here" \
  http://localhost:8666/api/v1/scan/<scan-id>
```

---

## Interpreting Results

### Threat Levels

Every scan produces a threat score from 0 to 100. The score is a weighted composite of all detection methods that fired.

| Score | Severity | Meaning |
|-------|----------|---------|
| 0 -- 24 | LOW | Clean or trivial metadata. No action required. |
| 25 -- 49 | MODERATE | Minor anomalies (unusual fields, non-standard values). Worth logging. |
| 50 -- 74 | HIGH | Suspicious patterns. Investigate further before allowing the file into your environment. |
| 75 -- 89 | HIGH | Strong threat indicators. Escalate to senior analyst or IR lead. |
| 90 -- 100 | CRITICAL | Confirmed malicious content (embedded code, known malware signatures, active exploits). Quarantine immediately. |

### YARA Matches

YARA matches mean the file content matched a known threat signature. Each match includes the rule name and a description of what was detected. YARA matches are high-confidence findings -- they indicate a specific, known pattern.

Example findings from YARA:

- `php_backdoor_in_exif` -- PHP code injected into image EXIF fields
- `js_xss_injection` -- JavaScript/XSS payload in metadata
- `office_macro_detected` -- VBA macro code in Office document metadata
- `polyglot_jpeg_zip` -- File is valid as both JPEG and ZIP (common attack technique)

### IOC Findings

IOC (Indicator of Compromise) findings are matches against known-bad values: malicious URLs, suspicious IP addresses, known malware hashes, or command-and-control domains found in metadata fields.

### Heuristic Scores

Heuristic findings flag statistically unusual patterns that do not match a specific signature but are anomalous enough to warrant attention:

- Unusually long metadata fields (potential buffer overflow payloads)
- Base64-encoded content in fields that normally contain plain text
- Metadata timestamps that predate the file format specification
- GPS coordinates pointing to known sensitive locations (Null Island, military bases)

### ML Anomaly Detection

The ML engine uses an ensemble of models (Isolation Forest + Random Forest, optionally XGBoost) trained on normal metadata distributions. Files that deviate significantly from the baseline are flagged. ML findings complement signature-based detection by catching novel threats that lack known signatures.

---

## Common False Positives

| Finding | Why it fires | How to handle |
|---------|-------------|---------------|
| `base64_payload` on camera photos | Some cameras store thumbnail data as Base64 in EXIF | Check the field name. `ThumbnailImage` Base64 is normal. |
| `suspicious_url` in stock photos | Stock photo agencies embed licensing URLs in EXIF | Verify the domain. Known stock agencies (shutterstock.com, gettyimages.com) are benign. |
| `oversized_metadata_field` on RAW photos | RAW camera files legitimately have large metadata blocks | Check the file source. If it came from a known camera, this is expected. |
| `timestamp_anomaly` on scanned documents | Scanners sometimes set incorrect dates | Cross-reference with file system timestamps. |
| `gps_pii_exposure` on geotagged photos | Any photo with GPS data triggers this | This is informational, not necessarily malicious. Evaluate whether GPS data is expected. |

To suppress known false positives, add patterns to `config/known_good_patterns.txt`. One pattern per line. HADES will reduce the score contribution of findings matching these patterns.

---

## Case Management Integration

Cases let you group related scans into an investigation with an auditable evidence chain.

### Create a case

```bash
hades case create "Suspicious email attachment - Ticket INC-4821"
```

This returns a case ID (UUID format).

### Scan files into a case

```bash
hades scan --case-id <CASE_ID> /path/to/attachment.docx
hades scan --case-id <CASE_ID> /path/to/second_file.pdf
```

All scans linked to the case are tracked in the hash-chained audit log.

### Add analyst notes

Via the web dashboard (Cases view), select the case and add notes documenting your analysis, findings, and conclusions.

### Export evidence

```bash
hades case export <CASE_ID>
```

This creates a ZIP package containing:

- All scan results (JSON)
- The audit log entries for this case
- SHA-256 hashes of every file scanned
- An HMAC-SHA256 signature for tamper detection
- A verification script that anyone can run to validate the package

This package is suitable for handoff to legal, law enforcement, or compliance teams.

---

## Example Workflows

### Workflow 1: "I received a suspicious email attachment"

1. Save the attachment to disk without opening it.

2. Scan it:
   ```bash
   hades scan suspicious_attachment.docx
   ```

3. Review the threat score:
   - **Below 25:** Likely clean. Verify with your sandbox if policy requires it.
   - **25 to 74:** Review the specific findings. Check for macro detection, template injection, or embedded URLs.
   - **75 and above:** Do not open. Create a case and escalate.

4. If escalating, create a case:
   ```bash
   hades case create "Phishing attachment from sender@suspicious-domain.com"
   hades scan --case-id <CASE_ID> suspicious_attachment.docx
   ```

5. Check MITRE ATT&CK mapping for the findings:
   ```bash
   curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/mitre/lookup?finding=office_macro_detected
   ```

6. Forward to your SIEM:
   ```bash
   curl -X POST http://localhost:8666/api/v1/export \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"scan_id": "<SCAN_ID>", "format": "cef"}'
   ```

### Workflow 2: "I need to scan a USB drive"

1. Mount the USB drive (read-only if possible).

2. Run a recursive scan:
   ```bash
   hades scan -r /media/usb-drive/
   ```

3. Review the summary. Focus on files scoring 50 or above.

4. For a formal investigation, create a case first:
   ```bash
   hades case create "USB drive from employee John Doe - HR request"
   hades scan -r --case-id <CASE_ID> /media/usb-drive/
   ```

5. Export the evidence package:
   ```bash
   hades case export <CASE_ID>
   ```

### Workflow 3: "Monitor a shared drive for new threats"

1. Start continuous monitoring:
   ```bash
   hades monitor /path/to/shared/drive
   ```

2. Or via the API for team visibility:
   ```bash
   curl -X POST http://localhost:8666/api/v1/monitor/start \
     -H "X-API-Key: your-api-key-here" \
     -H "Content-Type: application/json" \
     -d '{"directories": ["/path/to/shared/drive"], "alert_threshold": 50}'
   ```

3. HADES will automatically scan new and modified files. Alerts for files exceeding the threshold appear in the dashboard and can be forwarded to Slack, Teams, or your SIEM via playbooks.

---

## Sanitizing Metadata

To strip metadata from a file (for safe redistribution or before uploading to external systems):

```bash
curl -X POST http://localhost:8666/api/v1/sanitize \
  -H "X-API-Key: your-api-key-here" \
  -F "file=@photo_with_gps.jpg" \
  -F "remove_all=true" \
  -o sanitized_photo.jpg
```

The original file is not modified. HADES creates a clean copy with metadata removed.

---

## Further Reading

- [Quick Start Guide](quick_start.md) -- Installation and first scan
- [API Reference](api_reference.md) -- Full REST API documentation
- [SIEM Integration Guide](siem_integration_guide.md) -- Splunk, QRadar, ArcSight, Elastic, Sentinel setup
- [Threat Intelligence Guide](threat_intel_guide.md) -- Configure threat intel feeds
- [Deployment Guide](deployment_guide.md) -- Production deployment and hardening
- [YARA Rules Guide](yara_rules_guide.md) -- Writing and managing detection rules
