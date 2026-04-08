# MITRE ATT&CK Mapping Guide

HADES maps detection findings to MITRE ATT&CK techniques, providing standardized threat classification that integrates with SOC workflows, threat intelligence platforms, and compliance frameworks.

## Overview

The `MITREMapper` module (`core/mitre_mapping.py`) takes HADES detection findings (YARA matches, heuristic findings, ML anomalies, deep format detections) and maps each to the most relevant MITRE ATT&CK technique and tactic.

This mapping enables:

- Standardized threat classification across all detections
- Integration with ATT&CK Navigator for visual coverage assessment
- Correlation with threat intelligence using ATT&CK technique IDs
- Compliance reporting that references industry-standard frameworks

---

## Coverage Table

| HADES Finding Category | MITRE Technique ID | Technique Name | Tactic |
|---|---|---|---|
| Script injection (JavaScript) | T1059.007 | JavaScript | Execution |
| PHP backdoor / web shell | T1505.003 | Web Shell | Persistence |
| Command injection | T1059 | Command and Scripting Interpreter | Execution |
| Reverse shell | T1059.004 | Unix Shell | Execution |
| Base64 encoding / obfuscation | T1027 | Obfuscated Files or Information | Defense Evasion |
| Steganography | T1027.003 | Steganography | Defense Evasion |
| Polyglot file | T1036.008 | Masquerading: File Type | Defense Evasion |
| Macro execution | T1204.002 | Malicious File | Execution |
| Template injection (Office) | T1221 | Template Injection | Execution |
| SQL injection in metadata | T1190 | Exploit Public-Facing Application | Initial Access |
| Embedded executable | T1027.009 | Embedded Payloads | Defense Evasion |
| PII / GPS data leakage | T1005 | Data from Local System | Collection |
| URL / C2 in metadata | T1071.001 | Web Protocols | Command and Control |
| Encoded executable | T1027.002 | Software Packing | Defense Evasion |
| DDE attack | T1559.002 | Dynamic Data Exchange | Execution |

---

## Enrichment Pipeline

When `--mitre-map` is enabled, the mapping pipeline runs after all other detection engines:

```
File → Metadata Extraction → YARA → Heuristic → ML → Deep Format
                                                         ↓
                                                   MITRE Mapper
                                                         ↓
                                                  Enriched Result
```

Each finding is annotated with:

- `technique_id`: MITRE ATT&CK technique ID (e.g., `T1059.007`)
- `technique_name`: Human-readable technique name
- `tactic`: MITRE tactic category (e.g., `Execution`, `Defense Evasion`)

---

## CLI Usage

```bash
# Enable MITRE mapping for scan results
hades-enhanced --mitre-map -r /evidence/

# Combine with behavioral analysis
hades-enhanced --mitre-map --behavioral -r /evidence/
```

The CLI output includes MITRE technique badges next to each finding:

```
  [YARA]  HADES_METADATA_BASE64_PAYLOAD (severity: 7)
          ATT&CK: T1027 Obfuscated Files or Information (Defense Evasion)
```

---

## API Endpoints

### GET /api/v1/mitre/matrix

Returns the full MITRE ATT&CK mapping table used by HADES:

```bash
curl -H "X-API-Key: your-api-key-here" \
  http://localhost:8666/api/v1/mitre/matrix
```

### GET /api/v1/mitre/lookup

Look up a specific technique:

```bash
curl -H "X-API-Key: your-api-key-here" \
  "http://localhost:8666/api/v1/mitre/lookup?technique=T1059"
```

---

## ATT&CK Dashboard

The web dashboard includes an ATT&CK view that shows:

- **Technique Matrix**: Visual grid of detected techniques across all scans, colored by frequency
- **Technique Details**: Click a technique to see which files and findings triggered it
- **Tactic Breakdown**: Findings grouped by MITRE tactic
- **Coverage Gaps**: Techniques in the HADES mapping that have not been triggered (useful for assessing detection coverage)

---

## Navigator Layer Export

HADES can export its detection coverage as an ATT&CK Navigator layer JSON file for import into the [MITRE ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/):

```bash
curl -H "X-API-Key: your-api-key-here" \
  "http://localhost:8666/api/v1/mitre/matrix?format=navigator"
```

This produces a JSON file compatible with ATT&CK Navigator v4.x, with technique scores based on detection frequency and severity.

---

## Filtering by Technique

Filter scan results by MITRE technique in the API:

```bash
# Find all scans that triggered T1059 (Command and Scripting Interpreter)
curl -H "X-API-Key: your-api-key-here" \
  "http://localhost:8666/api/v1/mitre/lookup?technique=T1059"
```

Filter by tactic:

```bash
# Find all Defense Evasion detections
curl -H "X-API-Key: your-api-key-here" \
  "http://localhost:8666/api/v1/mitre/lookup?tactic=defense-evasion"
```

---

## Configuration

```json
{
  "mitre_mapping": {
    "enabled": true
  }
}
```

The MITRE mapping module has no external dependencies and is available in all installations.

---

## Next Steps

- [Behavioral Analysis](behavioral_analysis_guide.md) -- Campaign detection across files
- [SIEM Integration](siem_integration_guide.md) -- Export MITRE-enriched results to SIEM
- [Detection Depth Guide](detection_depth_guide.md) -- Full detection architecture
