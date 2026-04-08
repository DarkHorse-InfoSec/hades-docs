# Detection Depth Guide

Advanced detection capabilities in HADES v0.6.0 Phase 2, covering ML ensemble models, behavioral analysis, MITRE ATT&CK mapping, threat intelligence scheduling, and YARA rule building.

## ML Ensemble Architecture

### Overview

The ML ensemble detector (`core/ml_ensemble.py`) combines multiple machine learning models with weighted voting to improve anomaly detection accuracy over the single-model Isolation Forest approach.

### Models

| Model | Type | Purpose | Default Weight |
|-------|------|---------|----------------|
| Isolation Forest | Unsupervised | Baseline anomaly detection | 0.40 |
| Random Forest | Supervised | Classification with labeled data | 0.35 |
| XGBoost | Supervised (optional) | Gradient-boosted classification | 0.25 |

XGBoost requires the `xgboost` package. When unavailable, the ensemble degrades to a two-model configuration with re-normalized weights.

### Extended Features (25 total)

The `ExtendedFeatureExtractor` inherits the base 15 features from `MetadataFeatureExtractor` and adds 10 new security-focused features:

| # | Feature | Description |
|---|---------|-------------|
| 0-14 | Base features | Field count, entropy, printable ratio, Base64/URL/exec/SQL counts, timestamps, GPS, etc. |
| 15 | script_tag_count | Number of `<script>` tags in metadata values |
| 16 | php_pattern_count | PHP code patterns (`<?php`, `shell_exec`, `passthru`) |
| 17 | shell_command_count | Shell commands (`wget`, `curl`, `nc`, `chmod`, `bash -i`) |
| 18 | iframe_count | Number of `<iframe>` tags |
| 19 | encoding_layer_depth | Estimated nesting depth of encoding (Base64, hex, URL, double-encoding) |
| 20 | unicode_escape_count | Unicode/hex escape sequences (`\uXXXX`, `\xXX`, `%uXXXX`) |
| 21 | null_byte_count | Null bytes in metadata values (common in injection attacks) |
| 22 | field_name_length_variance | Statistical variance of field name lengths |
| 23 | value_length_variance | Statistical variance of field value lengths |
| 24 | suspicious_field_ratio | Ratio of fields containing suspicious keywords |

### Weighted Voting

The ensemble combines individual model scores using a weighted average:

```
combined_score = sum(model_score * model_weight) / sum(model_weights)
```

A file is classified as anomalous when `combined_score >= 0.5`. Confidence is computed from the standard deviation of individual model scores (lower deviation = higher confidence).

### Training Modes

- **Unsupervised** (no labels): Only Isolation Forest trains. Useful for baseline anomaly detection.
- **Supervised** (with labels): All models train. Requires at least two classes (clean=0, malicious=1).

### Labeled Data Management

`LabeledDataManager` stores training samples in SQLite with CSV import/export:

```python
from ml_ensemble import LabeledDataManager, EnsembleAnomalyDetector

manager = LabeledDataManager(db_path="labeled_data.db")
manager.add_sample(features=[...], label=0, file_path="/path/to/clean.jpg")
manager.add_sample(features=[...], label=1, file_path="/path/to/malicious.jpg")

# Export for external analysis
manager.export_csv("training_data.csv")
```

### Auto-Retraining

`AutoRetrainer` monitors sample counts and triggers retraining:

```python
from ml_ensemble import AutoRetrainer

retrainer = AutoRetrainer(data_manager=manager)
if retrainer.check_retrain_needed(threshold=500):
    contamination = retrainer.auto_tune_contamination(validation_data)
    retrainer.retrain(ensemble, contamination=contamination)
```

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
python core/hades_enhanced_cli.py --behavioral -r /evidence/

# Enable ensemble ML detection
python core/hades_enhanced_cli.py --ml-ensemble -r /evidence/
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
python core/hades_enhanced_cli.py --mitre-map -r /evidence/
```

### API Endpoints

- `GET /api/v1/mitre/matrix` — Full MITRE ATT&CK mapping table
- `GET /api/v1/mitre/lookup?technique=T1059` — Look up a specific technique

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
python core/hades_enhanced_cli.py --schedule-feeds
```

### API Endpoints

- `GET /api/v1/threat-intel/scheduler` — Scheduler status and feed list
- `POST /api/v1/threat-intel/scheduler` — Trigger manual feed update

---

## YARA Rule Builder

### Overview

`YARABuilder` (`core/yara_rule_builder.py`) provides programmatic YARA rule generation from templates and custom parameters.

### Available Templates

Templates are defined as `RuleTemplate` dataclasses. Built-in templates include patterns for metadata injection, steganography, polyglot files, and encoded payloads.

### CLI Usage

```bash
# List available YARA rule templates
python core/hades_enhanced_cli.py --list-templates

# Build a YARA rule from a template
python core/hades_enhanced_cli.py --build-rule metadata_injection
```

### API Endpoints

- `GET /api/v1/rules/templates` — List available rule templates
- `POST /api/v1/rules/build` — Build a rule from template with custom parameters

### Custom Rules

Rules built via the API or CLI are saved to the configured output directory (default: `rules/custom/`) and automatically loaded by the detection engine on the next scan.

---

## Configuration Reference

All Phase 2 features are configured in `cli/hades_config.json`:

```json
{
  "ml_ensemble": {
    "enabled": true,
    "models": ["isolation_forest", "random_forest", "xgboost"],
    "voting": "weighted",
    "feature_count": 25
  },
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
