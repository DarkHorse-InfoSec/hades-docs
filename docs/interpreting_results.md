# Interpreting Scan Results

This guide explains how to read HADES scan results, understand the JSON structure, and interpret findings from each detection engine.

## JSON Result Structure

When scanning via the API or with `--format json`, HADES produces a result object for each file:

```json
{
  "scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "file_name": "suspicious.jpg",
  "file_size": 245760,
  "status": "suspicious",
  "threat_score": 72.5,
  "threat_level": "high",
  "scanned_at": "2026-02-19T14:22:01+00:00",
  "basic_scan": { ... },
  "enhanced_detection": { ... },
  "metadata_summary": { ... },
  "ml_anomaly": { ... },
  "threat_intel": { ... },
  "deep_format": { ... },
  "mitre_mapping": [ ... ],
  "behavioral": { ... }
}
```

### Top-Level Fields

| Field | Type | Description |
|---|---|---|
| `scan_id` | string | UUID for this scan result (use to retrieve later) |
| `file_name` | string | Original filename |
| `file_size` | integer | File size in bytes |
| `status` | string | Overall verdict: `clean`, `suspicious`, or `malicious` |
| `threat_score` | float | Combined threat score (0-100) |
| `threat_level` | string | Severity band: `safe`, `low`, `medium`, `high`, `critical` |
| `scanned_at` | string | ISO 8601 timestamp |

---

## Threat Score Bands

| Score | Level | Color | Meaning |
|---|---|---|---|
| 0 - 25 | **Low / Safe** | Green | No significant threats detected |
| 26 - 50 | **Medium** | Yellow | Suspicious patterns warrant review |
| 51 - 75 | **High** | Orange | Multiple threat indicators present |
| 76 - 100 | **Critical** | Red | Strong malicious indicators |

The score is computed by combining findings from all engines. Each engine contributes to the score based on finding severity and confidence.

---

## Basic Scan Results

The `basic_scan` section comes from the `ExifScanner` engine:

```json
{
  "basic_scan": {
    "status": "suspicious",
    "scan_time": 0.34,
    "findings": [
      "Embedded executable detected in EXIF comment",
      "Suspicious URL found in metadata"
    ]
  }
}
```

| Field | Description |
|---|---|
| `status` | `clean` or `suspicious` |
| `scan_time` | Processing time in seconds |
| `findings` | List of human-readable finding descriptions |

---

## Enhanced Detection Results

The `enhanced_detection` section comes from the `EnhancedDetectionEngine`:

```json
{
  "enhanced_detection": {
    "overall_threat_score": 72.5,
    "threat_level": "high",
    "processing_time": 1.23,
    "detection_summary": "Multiple threat indicators found",
    "yara_matches": [
      {
        "rule": "HADES_METADATA_BASE64_PAYLOAD",
        "severity": 7,
        "description": "Large Base64 content in metadata fields"
      }
    ],
    "ioc_matches": [
      {
        "type": "url",
        "value": "http://evil.example.com/payload",
        "source": "metadata_field"
      }
    ],
    "heuristic_findings": [
      {
        "type": "base64_payload",
        "description": "Base64-encoded content in EXIF:Comment (452 chars)",
        "severity": 6
      }
    ]
  }
}
```

### Finding Types

| Type | Source | What It Means |
|---|---|---|
| **YARA Match** | Pattern matching against YARA rules | A known threat pattern was found in the file or its metadata |
| **IOC Match** | Indicator of Compromise database | A known malicious hash, URL, IP, or domain was found |
| **Heuristic Finding** | Rule-based analysis | Suspicious patterns detected by heuristic rules (Base64, script tags, SQL injection, etc.) |

---

## ML Anomaly Results

When ML detection is enabled (`--ml-detect` or `--ml-ensemble`), the `ml_anomaly` section appears for flagged files:

```json
{
  "ml_anomaly": {
    "anomaly_score": 0.85,
    "confidence": 0.72,
    "is_anomaly": true,
    "contributing_features": ["entropy", "null_byte_ratio", "metadata_total_length"],
    "model_type": "isolation_forest"
  }
}
```

| Field | Description |
|---|---|
| `anomaly_score` | 0.0 (normal) to 1.0 (highly anomalous) |
| `confidence` | Model confidence in the classification |
| `is_anomaly` | Whether the file exceeds the anomaly threshold |
| `contributing_features` | Features that most influenced the score |
| `model_type` | Which model produced this result |

!!! tip "Reading contributing features"
    The contributing features tell you **why** the ML model flagged a file. For example:

    - `entropy` + `null_byte_ratio` -- encrypted or compressed payload appended to the file
    - `extension_mimetype_mismatch` -- file extension does not match actual content type
    - `metadata_total_length` + `suspicious_field_count` -- large or suspicious metadata values

---

## Threat Intelligence Results

When threat intel is enabled (`--threat-intel`), the `threat_intel` section appears:

```json
{
  "threat_intel": {
    "ioc": "e3b0c44298fc1c149afb...",
    "ioc_type": "hash",
    "consensus": {
      "verdict": "malicious",
      "confidence": 85,
      "provider_count": 3
    },
    "results": [
      {
        "source": "virustotal",
        "verdict": "malicious",
        "confidence": 92,
        "details": "Detected by 45/70 engines"
      }
    ]
  }
}
```

The consensus verdict aggregates responses from all queried providers. See the [Threat Intelligence Guide](threat_intel_guide.md) for details.

---

## Deep Format Results

When deep format analysis detects threats in PDF, Office, SVG, or polyglot files:

```json
{
  "deep_format": {
    "findings": [
      {
        "analyzer": "pdf",
        "type": "javascript",
        "description": "PDF contains JavaScript in /Names dictionary",
        "severity": 8
      },
      {
        "analyzer": "polyglot",
        "type": "dual_format",
        "description": "File is valid as both JPEG and ZIP",
        "severity": 7
      }
    ]
  }
}
```

See the [Deep Format Guide](deep_format_guide.md) for the full list of detections.

---

## MITRE ATT&CK Mapping

When `--mitre-map` is enabled, findings are mapped to MITRE ATT&CK techniques:

```json
{
  "mitre_mapping": [
    {
      "technique_id": "T1027",
      "technique_name": "Obfuscated Files or Information",
      "tactic": "Defense Evasion",
      "finding": "Base64-encoded payload in EXIF:Comment"
    },
    {
      "technique_id": "T1036.008",
      "technique_name": "Masquerading: File Type",
      "tactic": "Defense Evasion",
      "finding": "Polyglot: JPEG + ZIP"
    }
  ]
}
```

See the [MITRE Mapping Guide](mitre_mapping_guide.md) for the complete mapping table.

---

## Behavioral Analysis Results

When `--behavioral` is enabled and multiple files are scanned, campaign detection results appear:

```json
{
  "behavioral": {
    "campaigns_detected": 1,
    "campaigns": [
      {
        "campaign_id": "campaign-001",
        "files": ["file_a.jpg", "file_b.jpg", "file_c.jpg"],
        "shared_indicators": ["evil.example.com", "T1059.007"],
        "confidence": 0.82,
        "temporal_window": "2026-02-18T10:00:00Z to 2026-02-18T14:00:00Z"
      }
    ]
  }
}
```

---

## Handling False Positives

Not every finding is a true threat. Common sources of false positives:

| Finding | Possible Benign Cause |
|---|---|
| GPS coordinates | Vacation photos, geotagged camera images |
| High entropy | Legitimate compressed/encrypted content |
| Base64 in metadata | Embedded thumbnails, XMP data packets |
| Long metadata fields | Extensive camera settings, copyright notices |
| MIME type mismatch | Renamed files, container formats |

!!! note "Reducing false positives"
    - Train the ML model on your organization's files (`--ml-train /path/to/known-good/`)
    - Use allowlists in `config/known_good_patterns.txt`
    - Adjust the alert threshold for monitoring (`--alert-threshold`)
    - Focus on high-severity findings (score > 50) for initial triage

---

## Next Steps

- [CLI Reference](cli_reference.md) -- Full command reference
- [API Reference](api_reference.md) -- REST API endpoint documentation
- [Dashboard Guide](dashboard_guide.md) -- Web dashboard usage
