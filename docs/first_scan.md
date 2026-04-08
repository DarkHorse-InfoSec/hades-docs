# Your First Scan

This tutorial walks through scanning files with HADES using the CLI, REST API, and web dashboard. By the end, you will understand how to read scan results and interpret threat scores.

## Scanning a Single File via CLI

The simplest way to use HADES:

```bash
hades-enhanced suspicious_image.jpg
```

Or, if running from source:

```bash
python core/hades_enhanced_cli.py suspicious_image.jpg
```

### Understanding the Output

```
HADES Metadata Forensics Engine v0.7.1
========================================
Scanning: suspicious_image.jpg

File: suspicious_image.jpg
  Type: image/jpeg
  Size: 245,760 bytes
  SHA-256: e3b0c44298fc1c149afb...

  Metadata Fields: 15
  GPS Coordinates: 40.7128, -74.0060

  Findings:
    [YARA]       HADES_METADATA_BASE64_PAYLOAD (severity: 7)
    [HEURISTIC]  Suspicious Base64 content in EXIF:Comment (452 chars)
    [ML]         Anomaly score: 0.72 (contributing: entropy, metadata_total_length)

  Threat Score: 65/100 (HIGH)
  Threat Level: high

Summary: 1 file scanned, 1 suspicious
```

Each section of the output tells you something specific:

| Field | Meaning |
|---|---|
| **Type** | Detected MIME type from magic bytes |
| **Metadata Fields** | Number of metadata key-value pairs extracted |
| **GPS Coordinates** | Geographic coordinates if present (potential PII leakage) |
| **[YARA]** | Pattern-based detection from YARA rules |
| **[HEURISTIC]** | Rule-based analysis (Base64 content, script tags, SQL patterns) |
| **[ML]** | Machine learning anomaly detection score with contributing features |
| **Threat Score** | Combined score from all detection engines (0-100) |

---

## Scanning a Directory

Scan all supported files in a directory:

```bash
hades-enhanced -r /path/to/evidence/
```

Add verbose output to see per-file details:

```bash
hades-enhanced -r -v /path/to/evidence/
```

Filter by file type:

```bash
hades-enhanced -r --file-types .jpg .png .pdf /path/to/evidence/
```

---

## Scanning via the REST API

Start the API server:

```bash
hades-server --port 8666
```

Upload and scan a file:

```bash
curl -X POST http://localhost:8666/api/v1/scan/file \
  -H "X-API-Key: your-api-key-here" \
  -F "file=@suspicious_image.jpg"
```

The response is a JSON object with the full scan result:

```json
{
  "scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "file_name": "suspicious_image.jpg",
  "file_size": 245760,
  "status": "suspicious",
  "threat_score": 65.0,
  "threat_level": "high",
  "basic_scan": {
    "status": "suspicious",
    "findings": ["Suspicious Base64 content in EXIF:Comment"]
  },
  "enhanced_detection": {
    "overall_threat_score": 65.0,
    "yara_matches": ["HADES_METADATA_BASE64_PAYLOAD"],
    "heuristic_findings": ["Base64-encoded payload detected"]
  }
}
```

You can retrieve the result later by scan ID:

```bash
curl -H "X-API-Key: your-api-key-here" \
  http://localhost:8666/api/v1/scan/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

## Scanning via the Web Dashboard

1. Start the API server: `hades-server --port 8666`
2. Open `http://localhost:8666/dashboard/` in your browser
3. Enter your API key (default: `your-api-key-here`)
4. Navigate to the **Scan** view
5. Drag and drop a file onto the upload area, or click to browse
6. View the results in the dashboard with threat level badges and finding details

---

## Threat Score Interpretation

HADES assigns a combined threat score from 0 to 100 based on all detection engine findings:

| Score Range | Level | Meaning | Recommended Action |
|---|---|---|---|
| 0 - 25 | **Low** | No significant threats detected | File is likely clean |
| 26 - 50 | **Medium** | Suspicious patterns found | Manual review recommended |
| 51 - 75 | **High** | Multiple threat indicators | Investigate thoroughly, consider quarantine |
| 76 - 100 | **Critical** | Strong malicious indicators | Quarantine immediately, escalate to IR team |

!!! warning "Scores are additive"
    A file with both a YARA match (severity 7) and an ML anomaly (score 0.8) will score higher than either finding alone. Multiple low-severity findings can combine to produce a high overall score.

!!! tip "Context matters"
    A GPS coordinate in a vacation photo is normal. A GPS coordinate in a corporate document template may indicate PII leakage. HADES flags the presence of metadata -- you provide the context.

---

## Enabling Additional Detection

By default, HADES runs basic metadata extraction, YARA matching, and heuristic analysis. Enable additional engines for deeper detection:

```bash
# Enable ML anomaly detection
hades-enhanced --ml-detect -r /evidence/

# Enable threat intelligence enrichment
hades-enhanced --threat-intel -r /evidence/

# Enable behavioral analysis and MITRE mapping
hades-enhanced --behavioral --mitre-map -r /evidence/

# Enable everything
hades-enhanced --ml-detect --threat-intel --behavioral --mitre-map -r /evidence/
```

---

## Generating Reports

Export scan results in different formats:

```bash
# JSON report
hades-enhanced -r /evidence/ --format json --output report.json

# HTML report
hades-enhanced -r /evidence/ --format html --output report.html

# CSV report
hades-enhanced -r /evidence/ --format csv --output report.csv
```

---

## Next Steps

- [CLI Reference](cli_reference.md) -- Every flag and option explained
- [Interpreting Results](interpreting_results.md) -- Deep dive into scan result fields
- [API Reference](api_reference.md) -- Full REST API documentation
- [YARA Rules Guide](yara_rules_guide.md) -- Understanding detection rules
- [ML Detection Guide](ml_detection_guide.md) -- ML anomaly detection setup and tuning
