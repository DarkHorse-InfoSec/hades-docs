# HADES ML Anomaly Detection Guide

## Overview

The ML anomaly detection module extends HADES with machine-learning-based analysis that identifies files whose metadata patterns deviate from normal. While rule-based engines (YARA, IOC matching, heuristics) excel at catching known threats, anomaly detection catches patterns that rules cannot express -- novel obfuscation techniques, unusual metadata combinations, and subtle signs of tampering that only emerge statistically.

The module uses an Isolation Forest algorithm to model "normal" metadata and flag outliers. It runs alongside the existing detection engines and enriches scan results with anomaly scores, confidence values, and the specific features that contributed to the detection.

## How It Works

1. **Feature extraction** -- `MetadataFeatureExtractor` reads a file and computes a fixed-length numeric feature vector from its raw bytes and metadata fields.
2. **Anomaly scoring** -- `AnomalyDetector` (backed by scikit-learn's `IsolationForest`) scores the feature vector. Lower raw scores indicate greater deviation from the training distribution.
3. **Result assembly** -- `MLDetectionEngine` wraps extraction and scoring, normalises the raw score to a 0--1 anomaly scale, identifies the top contributing features, and returns an `AnomalyResult` dataclass.

## Feature Descriptions

The extractor computes 15 features from each file:

| # | Feature | Description | Why it matters |
|---|---------|-------------|----------------|
| 1 | `file_size` | Raw file size in bytes | Outlier sizes may indicate padding, appended payloads, or truncation |
| 2 | `file_size_log` | Log-scaled file size | Normalises the wide range of file sizes for the model |
| 3 | `entropy` | Shannon entropy of the full file content (0--8) | High entropy suggests encryption or compression; low entropy may indicate padding |
| 4 | `metadata_field_count` | Number of metadata fields extracted | Unusually many or few fields can signal manipulation |
| 5 | `metadata_total_length` | Sum of all metadata value string lengths | Excessively long values may embed payloads |
| 6 | `has_gps` | Binary flag: GPS coordinates present (1) or absent (0) | GPS in unexpected file types may indicate data leakage |
| 7 | `has_thumbnail` | Binary flag: embedded thumbnail present | Thumbnails can carry hidden data or outdated images |
| 8 | `suspicious_field_count` | Count of fields matching suspicious patterns (e.g., script tags, base64) | Direct indicator of potential payload embedding |
| 9 | `creation_modify_delta` | Seconds between creation and modification timestamps | Large or negative deltas suggest timestamp manipulation |
| 10 | `extension_mimetype_mismatch` | Binary flag: file extension does not match detected MIME type | Classic polyglot/masquerading indicator |
| 11 | `header_magic_valid` | Binary flag: file header magic bytes match expected type | Invalid magic bytes indicate file type spoofing |
| 12 | `embedded_file_count` | Number of embedded files or objects detected | Hidden embedded content is a common attack vector |
| 13 | `null_byte_ratio` | Ratio of null bytes to total file size | Abnormal null byte ratios indicate padding or binary injection |
| 14 | `printable_ratio` | Ratio of printable ASCII characters to total file size | Helps distinguish text-heavy payloads from binary content |
| 15 | `longest_run_length` | Length of the longest repeated byte sequence | Long runs may indicate NOP sleds, padding, or steganography |

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
- Score 0.85, contributing features `["entropy", "null_byte_ratio"]` -- the file has unusually high entropy and an abnormal proportion of null bytes, possibly indicating encrypted content appended to an otherwise normal file.
- Score 0.62, contributing features `["extension_mimetype_mismatch", "metadata_field_count"]` -- the file extension does not match its actual content type and has an unusual number of metadata fields.

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
