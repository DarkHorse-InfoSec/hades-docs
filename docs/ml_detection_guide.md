# HADES ML Anomaly Detection Guide

## Overview

HADES includes a machine-learning stage that flags files whose metadata and structure look like malware even when no YARA rule, IOC or heuristic fires. Rule-based engines excel at known threats; the ML stage looks at the whole feature profile of a file and catches combinations that rules cannot express.

As of HADES v1.7.4 the ML stage is a single **supervised XGBoost classifier** (`SignedXGBClassifier` in `core/ml_detection.py`). The production model is **v1.18.0**, uses **33 features** (feature schema version 9), and ships inside the `hades` and `hades-server` binaries. The model bundle is signed by DarkHorse; HADES verifies the signature every time it loads the model and **refuses to use a model whose signature does not verify** (it fails closed rather than scanning with an untrusted model).

The earlier multi-model "ML ensemble" (Isolation Forest + Random Forest + XGBoost, `core/ml_ensemble.py`, the `--ml-ensemble` flag and `GET /api/v1/ml/ensemble/status`) was **removed in v1.7.3**. It had no trained model and was never used by the scan path.

## How It Works

1. **Feature extraction** -- `MetadataFeatureExtractor` computes the 33-feature vector described below from the file's metadata, raw bytes and the findings of the PE and PDF analyzers.
2. **Classification** -- the signed classifier scores the vector.
3. **Result assembly** -- `MLDetectionEngine` turns the score into an `AnomalyResult` (see "Interpreting ML results") and the scan's verdict logic weighs it together with the other engines.

The stage runs automatically during scans as the **ML Anomaly Detector** plugin on Professional, Team and Enterprise licences. Community scans do not run it.

## Feature Descriptions

The production classifier's 33 features, by name as the code defines them:

**Metadata and content features (f0-f14)**

| # | Feature |
|---|---------|
| 0 | `metadata_field_count` |
| 1 | `total_metadata_size_bytes` |
| 2 | `field_name_entropy` |
| 3 | `field_value_entropy` |
| 4 | `printable_ratio` |
| 5 | `base64_string_count` |
| 6 | `url_string_count` |
| 7 | `executable_string_count` |
| 8 | `sql_string_count` |
| 9 | `file_size_metadata_ratio` |
| 10 | `timestamp_consistency_score` |
| 11 | `gps_present` |
| 12 | `embedded_file_indicators` |
| 13 | `exif_completeness_score` |
| 14 | `file_header_entropy` |

**Analyzer-derived features (f15-f24)**, taken from the PE and PDF analyzers' findings and the detected format:

| # | Feature |
|---|---------|
| 15 | `pe_finding_count` |
| 16 | `pe_kernel_driver_flag` |
| 17 | `pe_suspicious_imports_flag` |
| 18 | `pe_packed_or_overlay_flag` |
| 19 | `pe_max_risk_score_norm` |
| 20 | `pdf_js_flag` |
| 21 | `pdf_structural_anomaly_count` |
| 22 | `pdf_max_risk_score_norm` |
| 23 | `detected_format_pe` |
| 24 | `detected_format_pdf` |

**Trust and false-positive-control features (f25-f32)**

| # | Feature | Description |
|---|---------|-------------|
| 25 | `publisher_trust_valid_chain` | Valid publisher signature: Authenticode for PE files, the signature block for PowerShell scripts; 0 for other files |
| 26 | `is_known_clean_format` | First bytes match a recognized image or font format (PNG, JPEG, GIF, SVG, TTF, OTF, WOFF, WOFF2, WebP, ICO, TIFF, BMP, HEIC) |
| 27 | `is_epson_driver_naming` | File name matches Epson printer-driver naming |
| 28 | `is_generic_oem_driver_naming` | File name matches a generic vendor-driver naming pattern |
| 29 | `is_in_oem_driver_path` | Path contains Windows DriverStore markers |
| 30 | `parent_dir_vendor_token_present` | Parent directory name contains a known vendor token |
| 31 | `is_powershell_test_harness` | Pester / Microsoft test-harness shape |
| 32 | `image_icon_size_heuristic` | Tiny known-clean-format file (icon shape) |

## Using ML Detection

There is nothing to install: the classifier and its dependencies are bundled in the binaries.

### Show the ML outcome in scan output

The ML stage runs on every licensed scan. To have the scan report its per-file outcome and a summary, add `--ml-detect`:

```bash
hades scan --ml-detect -r /path/to/evidence/
```

### Check model status

```bash
hades intel ml-status
```

This prints the ML engine's status fields.

### Training and updates

The shipped binaries do not train or retrain models. Production models are trained, evaluated against per-category false-positive gates and signed by DarkHorse, and delivered with HADES releases. A binary only ever runs the signed model it was built with.

## Interpreting ML Results

Each ML result (`AnomalyResult`) contains:

- **anomaly_score** (0.0 to 1.0): higher values mean the file looks more like the malicious class.
- **is_anomaly**: whether the score crossed the model's threshold for the file's category.
- **confidence** (0.0 to 1.0): how clearly the file sits on one side of the decision.
- **contributing_features**: the feature names that contributed most. These tell you *why* the file was flagged.
- **raw_score** and **feature_vector**: the unnormalised model output and the 33 input values, for analysts who want to dig in.

Example: score 0.85 with contributing features `["file_header_entropy", "embedded_file_indicators"]` means the file header has unusually high entropy and the file shows signs of embedded content, possibly an encrypted payload inside an otherwise normal file.

An ML flag is one input to the verdict, not the verdict itself: the final threat level also reflects YARA, IOC, heuristic and format-analyzer findings.

## REST API

When `hades-server` is running, ML status is available at:

### GET /api/v1/ml/status

Returns the current state of the ML detection engine.

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/ml/status
```

The training endpoints `POST /api/v1/ml/retrain` and `GET /api/v1/ml/features` documented for earlier versions do not exist in v1.7.4.

## Privacy

All ML processing runs locally. No file content, metadata or feature vectors are sent to external services.
