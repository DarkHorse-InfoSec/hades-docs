# Detection Depth Guide

Advanced detection capabilities in HADES v1.7.4: ML detection, behavioral analysis, MITRE ATT&CK mapping, threat intelligence scheduling, and YARA rule building.

## ML Detection

The ML stage in v1.7.4 is a single signed, supervised XGBoost classifier (production model v1.18.0, 33 features, feature schema 9). It runs automatically on Professional, Team and Enterprise scans as the ML Anomaly Detector plugin and fails closed if the model's signature does not verify. Features, result fields and the status endpoint are documented in the [ML Detection Guide](ml_detection_guide.md).

The multi-model ML ensemble (`core/ml_ensemble.py`, `--ml-ensemble`, `LabeledDataManager`, `AutoRetrainer`) that earlier versions of this guide described was removed in v1.7.3; it had no trained model and the scan path never used it.

---

## Behavioral Analysis

### Campaign Detection

`CampaignDetector` (`core/behavioral_analysis.py`) identifies threat campaigns by correlating scan results across multiple files using three strategies:

1. **Shared IOCs**: Files sharing IP addresses, domains, URLs, or file hashes are grouped using union-find.
2. **Metadata Pattern Similarity**: Files with similar attack patterns (script injection, Base64 encoding, shell commands) are grouped using Jaccard similarity (threshold >= 0.3).
3. **Temporal Clustering**: Files scanned within a configurable time window (default: 24 hours) are grouped together.

Groups from all three strategies are merged when they overlap, and campaign profiles are generated for groups meeting minimum indicator thresholds.

### File Relationship Graph

`FileRelationshipGraph` maintains an adjacency list where edges represent shared indicators between files:

```python
from behavioral_analysis import FileRelationshipGraph

graph = FileRelationshipGraph()
graph.add_file("/path/to/file_a.jpg", scan_result_a)
graph.add_file("/path/to/file_b.jpg", scan_result_b)

# Find files related to file_a within 2 hops
related = graph.get_related("/path/to/file_a.jpg", max_depth=2)

# Extract campaign profiles from the graph
campaigns = graph.get_campaigns()
```

### CLI Usage

```bash
# Enable behavioral analysis during a scan
hades scan --behavioral -r /evidence/

# Report the ML stage outcome per file
hades scan --ml-detect -r /evidence/
```

---

## MITRE ATT&CK Integration

### Overview

`MITREMapper` (`core/mitre_mapping.py`) maps HADES detection findings to MITRE ATT&CK techniques, providing standardized threat classification.

### Mapping Table

| HADES Finding Category | MITRE Technique | Tactic |
|----------------------|-----------------|--------|
| Script injection | T1059.007 (JavaScript) | Execution |
| PHP backdoor | T1505.003 (Web Shell) | Persistence |
| Command injection | T1059 (Command and Scripting Interpreter) | Execution |
| Reverse shell | T1059.004 (Unix Shell) | Execution |
| Base64 encoding | T1027 (Obfuscated Files or Information) | Defense Evasion |
| Steganography | T1027.003 (Steganography) | Defense Evasion |
| Polyglot file | T1036.008 (Masquerading: File Type) | Defense Evasion |
| Macro execution | T1204.002 (Malicious File) | Execution |
| Template injection | T1221 (Template Injection) | Execution |

### CLI Usage

```bash
# Show MITRE ATT&CK mapping for scan results
hades scan --mitre-map -r /evidence/
```

### API Endpoints

- `GET /api/v1/mitre/matrix` - Full MITRE ATT&CK mapping table
- `GET /api/v1/mitre/lookup?technique=T1059` - Look up a specific technique

---

## Threat Intelligence Scheduling

### Overview

`ThreatIntelScheduler` (`core/threat_intel_scheduler.py`) automates periodic ingestion of threat intelligence feeds (STIX/TAXII, MISP, CSV, custom).

### Feed Configuration

Feeds are configured via `cli/hades_config.json`:

```json
{
  "threat_intel_scheduler": {
    "feeds": [
      {
        "name": "AlienVault OTX",
        "type": "otx",
        "interval_hours": 6,
        "enabled": true
      }
    ],
    "default_interval_hours": 6
  }
}
```

### CLI Usage

```bash
# Start the threat intel feed scheduler
hades intel schedule-feeds
```

### API Endpoints

- `GET /api/v1/threat-intel/scheduler` - Scheduler status and feed list
- `POST /api/v1/threat-intel/scheduler` - Trigger manual feed update

---

## YARA Rule Builder

### Overview

`YARABuilder` (`core/yara_rule_builder.py`) provides programmatic YARA rule generation from templates and custom parameters.

### Available Templates

Templates are defined as `RuleTemplate` dataclasses. Built-in templates include patterns for metadata injection, steganography, polyglot files, and encoded payloads.

### CLI Usage

```bash
# List available YARA rule templates
hades rules list-templates

# Build a YARA rule from a template
hades rules build metadata_injection
```

### API Endpoints

- `GET /api/v1/rules/templates` - List available rule templates
- `POST /api/v1/rules/build` - Build a rule from template with custom parameters

### Custom Rules

Rules built via the API or CLI are saved to the configured output directory (default: `rules/custom/`) and automatically loaded by the detection engine on the next scan.

---

## Configuration Reference

These features are configured in `cli/hades_config.json`:

```json
{
  "behavioral_analysis": {
    "enabled": true,
    "campaign_window_hours": 168,
    "min_campaign_files": 3
  },
  "threat_intel_scheduler": {
    "feeds": [],
    "default_interval_hours": 6
  },
  "yara_rule_builder": {
    "templates_dir": "rules/templates",
    "output_dir": "rules/custom"
  },
  "mitre_mapping": {
    "enabled": true
  }
}
```
