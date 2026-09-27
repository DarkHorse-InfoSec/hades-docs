# HADES ML Anomaly Detection Guide

## Overview

The ML anomaly detection module extends HADES with machine-learning-based analysis that identifies files whose metadata patterns deviate from normal. While rule-based engines (YARA, IOC matching, heuristics) excel at catching known threats, anomaly detection catches patterns that rules cannot express -- novel obfuscation techniques, unusual metadata combinations, and subtle signs of tampering that only emerge statistically.

The module uses an Isolation Forest algorithm to model "normal" metadata and flag outliers. It runs alongside the existing detection engines and enriches scan results with anomaly scores, confidence values, and the specific features that contributed to the detection.

## How It Works

1. **Feature extraction** -- `MetadataFeatureExtractor` reads a file and computes a fixed-length numeric feature vector from its raw bytes and metadata fields.
2. **Anomaly scoring** -- `AnomalyDetector` (backed by scikit-learn's `IsolationForest`) scores the feature vector. Lower raw scores indicate greater deviation from the training distribution.
3. **Result assembly** -- `MLDetectionEngine` wraps extraction and scoring, normalises the raw score to a 0--1 anomaly scale, identifies the top contributing features, and returns an `AnomalyResult` dataclass.

## Feature Descriptions

As of HADES v1.7.1, `MetadataFeatureExtractor` (`core/ml_detection.py`) computes **33 features** per file (`FEATURE_COUNT`, feature schema version 9). The separate ensemble path (`--ml-ensemble`, `core/ml_ensemble.py`) uses its own `ExtendedFeatureExtractor` with **25 features**, listed further down. The active production classifier is `SignedXGBClassifier`, a supervised XGBoost model loaded only if its signature verifies; the Isolation Forest `AnomalyDetector` described in this guide is the anomaly-scoring path used by `--ml-train` and `--ml-baseline`.

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

**Ensemble features (`ExtendedFeatureExtractor`, 25):** `metadata_field_count`, `field_value_avg_length`, `field_value_max_length`, `field_value_stddev_length`, `field_count_ratio`, `numeric_field_ratio`, `ascii_ratio`, `entropy_avg`, `entropy_max`, `entropy_stddev`, `has_gps_data`, `has_thumbnail`, `timestamp_count`, `timestamp_consistency`, `url_count`, `email_count`, `executable_pattern_count`, `base64_likelihood`, `special_char_ratio`, `field_name_anomaly_score`, `nested_depth_max`, `binary_content_ratio`, `file_size_to_metadata_ratio`, `duplicate_value_count`, `language_consistency`.

## Getting Started

### Prerequisites

Install the required Python packages:

```bash
pip install scikit-learn numpy
```

These are optional dependencies -- HADES continues to function without them, but the ML features will be unavailable.

### Generate a Baseline Model

The fastest way to start is to generate a baseline model trained on synthetic data:

```bash
python core/hades_enhanced_cli.py --ml-baseline
```

This creates `models/baseline_model.joblib` using synthetically generated feature vectors that represent a range of normal file metadata patterns. The baseline provides reasonable detection out of the box but should be retrained on your real data for best results.

### Scan with ML Detection

Once a model exists, enable ML anomaly detection during scans:

```bash
python core/hades_enhanced_cli.py -r --ml-detect /path/to/evidence/
```

Each file in the scan results will be enriched with an `ml_anomaly` field (if flagged) containing the anomaly score, confidence, and contributing features.

### Check Model Status

View the current state of the ML engine:

```bash
python core/hades_enhanced_cli.py --ml-status
```

Output includes whether a model is loaded, its training sample count, feature count, and contamination parameter.

## Training Your Own Model

For best accuracy, train the model on files representative of your environment:

```bash
python core/hades_enhanced_cli.py --ml-train /path/to/known-good-files/
```

This scans all supported files in the directory, extracts features, trains an Isolation Forest, and saves the model to `models/trained_model.joblib`. The training directory should contain files that represent "normal" for your environment -- the model learns what normal looks like and flags deviations.

Tips for training data:
- Include a variety of file types you commonly analyse
- Use files from trusted sources (corporate assets, known-good samples)
- Aim for at least 100 files for reasonable accuracy; 500+ is better
- Exclude files you know to be malicious (unless you want them in the model)

## Interpreting Anomaly Scores

Each anomaly result contains:

- **anomaly_score** (0.0--1.0): Higher values indicate greater deviation from normal. Scores above 0.5 are flagged as anomalies by default.
- **confidence** (0.0--1.0): How confident the model is in its classification. Higher confidence means the file is more clearly an inlier or outlier.
- **contributing_features**: List of feature names that contributed most to the anomaly score. These tell you *why* the file was flagged.

Example interpretation:
- Score 0.85, contributing features `["file_header_entropy", "embedded_file_indicators"]` -- the file header has unusually high entropy and the file shows signs of embedded content, possibly indicating an encrypted payload inside an otherwise normal file.
- Score 0.62, contributing features `["base64_string_count", "metadata_field_count"]` -- the metadata carries more base64-encoded strings than usual and an unusual number of metadata fields.

## Baseline Model Limitations

The baseline model is trained on synthetic data that approximates common file metadata distributions. Limitations include:

- It has never seen your actual files, so false positive rates may be higher than with a custom-trained model.
- Synthetic data cannot capture organisation-specific patterns (e.g., your camera models, document templates, standard software).
- It provides a starting point -- retrain on real data as soon as practical.

## When to Retrain

Consider retraining when:
- You first deploy HADES in a new environment (train on representative samples)
- False positive rates are too high (add more clean samples and retrain)
- False negative rates are too high (adjust the contamination parameter or add anomalous samples)
- Your file types or workflows change significantly
- After accumulating scan data over time (periodic retraining improves accuracy)

## REST API Endpoints

When the API server is running (`--serve`), the following ML endpoints are available:

### GET /api/v1/ml/status

Returns the current state of the ML detection engine.

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/ml/status
```

### POST /api/v1/ml/retrain

Trigger model retraining using accumulated training samples.

```bash
curl -X POST \
     -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/ml/retrain
```

### GET /api/v1/ml/features?file_path=<path>

Extract and return the feature vector for a specific file without scoring.

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/ml/features?file_path=/path/to/file.jpg"
```

## Plugin Integration

The ML anomaly detection plugin (`plugins/ml_anomaly_plugin.py`) integrates automatically with the HADES plugin system. When plugins are enabled, it:

- Loads on startup via `PluginManager` discovery
- Analyses files during scan operations
- Returns `Finding` dataclass instances with severity mapped from the anomaly score
- Supports all file types (no file type restriction)

Verify it is loaded:

```bash
python core/hades_enhanced_cli.py --list-plugins
```

## Configuration

ML detection settings in `cli/hades_config.json`:

```json
{
  "ml_detection": {
    "enabled": false,
    "model_path": "models/baseline_model.joblib",
    "retrain_threshold": 1000,
    "contamination": 0.1
  }
}
```

| Setting | Default | Description |
|---------|---------|-------------|
| `enabled` | `false` | Enable ML detection by default (can still be activated with `--ml-detect`) |
| `model_path` | `models/baseline_model.joblib` | Path to the trained model file |
| `retrain_threshold` | `1000` | Number of samples to accumulate before suggesting retraining |
| `contamination` | `0.1` | Expected proportion of anomalies in the data (0.0--0.5) |

## Privacy

All ML processing runs locally. No file content, metadata, or feature vectors are sent to external services. The trained model file contains only statistical parameters (split thresholds and tree structures), not any file content from the training data.
