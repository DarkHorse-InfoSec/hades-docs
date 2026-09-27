# Testing HADES

**Version:** 1.0.0
**Author:** DarkHorse Information Security LLC

This guide covers the HADES test suite architecture, how to run tests at every level, environment setup, and quality standards.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Test Suite Architecture](#test-suite-architecture)
3. [Running Tests](#running-tests)
4. [Test Categories](#test-categories)
5. [Detection Corpus Validation](#detection-corpus-validation)
6. [CI/CD Integration](#cicd-integration)
7. [Writing New Tests](#writing-new-tests)
8. [Quality Standards](#quality-standards)
9. [Troubleshooting Test Failures](#troubleshooting-test-failures)

---

## 1. Prerequisites

### Required Software

| Software | Version | Purpose |
|----------|---------|---------|
| Python | 3.8+ | Runtime and test runner |
| ExifTool | Latest | Metadata extraction (used by scanner) |
| pip | Latest | Package management |

### Install Development Dependencies

```bash
# Full dev install (editable mode with test dependencies)
pip install -e ".[dev]"

# Or install from requirements.txt
pip install -r requirements.txt
pip install pytest pytest-asyncio
```

### Verify Installation

```bash
python -c "import pytest; print(pytest.__version__)"
python cli/main.py --help
python core/hades_enhanced_cli.py --help
```

!!! note "Optional Dependencies"
    Many HADES modules use optional imports (YARA, scikit-learn, PostgreSQL drivers, Redis, etc.). Tests for optional modules will **skip** automatically if the dependency is not installed. This is expected behavior, not a failure.

---

## 2. Test Suite Architecture

The HADES test suite is organized across 7 directories. Every module has a corresponding test file that validates its public API.

### Directory Layout

```
darkhorse-hades/
|-- cli/                          # CLI layer tests
|   |-- test_cli.py               # Basic CLI smoke test
|   |-- test_reverse_shell.py     # Reverse shell detection
|   |-- test_integration.py       # CLI end-to-end integration
|   `-- test_enhanced_scan.py     # Enhanced CLI pipeline
|-- core/                         # Core engine tests (~50 test files)
|   |-- test_metadata_parser.py   # Foundation parser
|   |-- test_detection_engine.py  # Threat detection
|   |-- test_api.py               # REST API endpoints
|   |-- test_dashboard.py         # Web dashboard
|   |-- test_gps_forensics.py     # GPS forensic analysis
|   |-- test_quarantine.py        # Quarantine lifecycle
|   |-- test_playbook_engine.py   # Playbook automation
|   `-- ...                       # ~40 more test files
|-- config/
|   `-- test_config.py            # Configuration validation
|-- plugins/
|   |-- test_registry.py          # Plugin marketplace
|   `-- test_example_plugins.py   # Built-in plugins
|-- tests/                        # Integration & corpus tests
|   |-- test_full_pipeline.py     # End-to-end pipeline
|   |-- test_corpus_*.py          # Detection corpus validation
|   |-- test_malwarebazaar_validation.py
|   `-- ...
|-- demo/
|   |-- test_demo_server.py       # Demo server endpoints
|   `-- test_demo_samples.py      # Sample file generation
|-- grafana/
|   `-- test_dashboards.py        # Grafana dashboard JSON
`-- kubernetes/
    `-- test_helm.py              # Helm chart validation
```

### Test Naming Convention

All test files follow the pattern `test_<module_name>.py` and reside alongside (or near) the code they test. Test classes use descriptive names:

- `TestClassName`: groups related tests
- `test_<behavior>`: individual test methods describing expected behavior

---

## 3. Running Tests

### Full Test Suite

```bash
# Run everything (recommended before any release)
python -m pytest core/ cli/ config/ plugins/test_registry.py \
    plugins/test_example_plugins.py tests/ demo/ grafana/ kubernetes/ \
    --ignore=plugins/deepfake-detector -v --tb=short
```

Some tests will be **skipped** when optional dependencies (PostgreSQL, Redis, etc.) are not installed.

### Quick Smoke Test

```bash
# Fastest check that nothing is broken (~30 seconds)
python -m pytest cli/test_cli.py core/test_metadata_parser.py \
    core/test_detection_engine.py core/test_api.py -v --tb=short
```

### By Category

```bash
# CLI tests
python -m pytest cli/ -v

# Core engine tests
python -m pytest core/ -v --tb=short

# Enterprise tests (auth, storage, multi-tenant)
python -m pytest core/test_auth_*.py core/test_storage_*.py \
    core/test_enterprise_integration.py -v

# Detection & ML tests
python -m pytest core/test_ml_ensemble.py core/test_behavioral_analysis.py \
    core/test_mitre_mapping.py -v

# Performance tests
python -m pytest core/test_async_scanner.py core/test_worker_pool.py \
    core/test_file_ingestion.py core/test_profiler.py \
    core/test_scan_cache.py core/test_benchmarks_scale.py -v

# Observability tests
python -m pytest core/test_metrics.py grafana/test_dashboards.py \
    kubernetes/test_helm.py tests/test_observability_integration.py -v

# Playbook & automation tests
python -m pytest core/test_playbook_engine.py -v

# GPS forensics tests
python -m pytest core/test_gps_forensics.py -v

# Quarantine tests
python -m pytest core/test_quarantine.py -v

# Dashboard tests
python -m pytest core/test_dashboard.py -v

# Integration connector tests
python -m pytest core/test_integrations_*.py -v

# Detection corpus validation
python -m pytest tests/test_corpus_*.py -v --timeout=60

# Full validation with report output
python -m pytest tests/test_corpus_full_validation.py -v -s
```

### Single Test File or Method

```bash
# Single file
python -m pytest core/test_gps_forensics.py -v

# Single test class
python -m pytest core/test_quarantine.py::TestReleaseFile -v

# Single test method
python -m pytest core/test_api.py::TestScanEndpoint::test_scan_file -v

# With keyword matching
python -m pytest core/ -k "gps" -v
```

### Useful Pytest Flags

| Flag | Purpose |
|------|---------|
| `-v` | Verbose output (show each test name) |
| `--tb=short` | Short traceback on failures |
| `--tb=long` | Full traceback on failures |
| `-x` | Stop on first failure |
| `-s` | Show print/stdout output |
| `--timeout=60` | Per-test timeout (requires pytest-timeout) |
| `-k "keyword"` | Run only tests matching keyword |
| `--ignore=path` | Skip a directory or file |
| `-n auto` | Parallel execution (requires pytest-xdist) |

---

## 4. Test Categories

### Unit Tests

Test individual classes and functions in isolation. Mock external dependencies (HTTP, databases, file I/O).

**Examples:**
- `core/test_metadata_parser.py`: Tests `HADESMetadataParser` with synthetic file content
- `core/test_gps_forensics.py`: Tests `GPSForensicsAnalyzer` coordinate parsing, Haversine distance, clustering
- `core/test_ml_ensemble.py`: Tests feature extraction and model training with synthetic data

**Pattern:** Direct class instantiation, assertions on return values. No subprocess calls, no network access.

### Integration Tests

Test module interactions and the REST API surface.

**Examples:**
- `core/test_api.py`: Tests FastAPI endpoints via `TestClient` (scan, sanitize, health, WebSocket)
- `core/test_quarantine.py::TestQuarantineRoutes`: Tests quarantine API routes via `TestClient`
- `tests/test_full_pipeline.py`: End-to-end: API server start, upload, detect, monitor, sanitize, shutdown

**Pattern:** Uses `starlette.testclient.TestClient` for API tests. Real database creation (temp SQLite). No external network calls.

### Detection Corpus Tests

Validate detection accuracy against a 28-file synthetic malware corpus.

**Examples:**
- `tests/test_corpus_exif_injection.py`: 8 EXIF injection attack types
- `tests/test_corpus_documents.py`: PDF/DOCX/SVG attack types
- `tests/test_corpus_polyglots.py`: Multi-format polyglot files
- `tests/test_corpus_clean.py`: 10 clean files (false positive validation)
- `tests/test_corpus_full_validation.py`: Master TPR/FPR/FNR validation

**Acceptance Criteria:**

| Metric | Threshold |
|--------|-----------|
| True Positive Rate (TPR) | >= 90% |
| False Positive Rate (FPR) | <= 10% |

### Configuration Tests

Validate configuration files parse correctly and contain expected structure.

- `config/test_config.py`: JSON parsing, YARA compilation, threat intel config, HTML templates

### Infrastructure Tests

Validate deployment manifests, Docker configs, and dashboards.

- `tests/test_docker_compose.py`: Docker Compose YAML structure
- `grafana/test_dashboards.py`: Grafana dashboard JSON validity
- `kubernetes/test_helm.py`: Helm chart templates, values, Kustomize overlays
- `tests/test_docs.py`: MkDocs config, file existence, cross-references

---

## 5. Detection Corpus Validation

The detection corpus is a set of programmatically generated files that simulate real-world attack techniques. No actual malware is included: all files are inert.

### Corpus Structure

```
tests/corpus/
|-- generators/
|   |-- exif_injection.py      # 8 EXIF injection attacks
|   |-- document_attacks.py    # 6 document attacks
|   |-- polyglot_files.py      # 4 polyglot files
|   `-- clean_baseline.py      # 10 clean files
|-- validation_results.json    # Generated results
`-- README.md                  # Corpus documentation
```

### Running Validation

```bash
# Full validation with detailed output
python -m pytest tests/test_corpus_full_validation.py -v -s

# Individual categories
python -m pytest tests/test_corpus_exif_injection.py -v
python -m pytest tests/test_corpus_documents.py -v
python -m pytest tests/test_corpus_polyglots.py -v
python -m pytest tests/test_corpus_clean.py -v  # false positive check
```

### Understanding Results

The full validation test outputs a detection matrix:

- **TP (True Positive):** Malicious file correctly detected (score >= 25)
- **FP (False Positive):** Clean file incorrectly flagged (score >= 75)
- **TN (True Negative):** Clean file correctly passed (score < 75)
- **FN (False Negative):** Malicious file missed (score < 25)

### MalwareBazaar Validation

For testing against real-world malware samples (requires network access):

```bash
# Run MalwareBazaar validation tests
python -m pytest tests/test_malwarebazaar_validation.py -v

# Run the standalone validation script
python scripts/run_malware_validation.py
```

See the [Real-World Validation Guide](real_world_validation_guide.md) for detailed procedures on expanding coverage with real samples.

---

## 6. CI/CD Integration

### GitHub Actions

The project includes two workflows in `.github/workflows/`:

**Test Suite** (`test-suite.yml`):
- Triggers on push/PR to main
- Python version matrix testing
- Runs full pytest suite
- Reports coverage

**YARA Validation** (`validate-rules.yml`):
- Triggers on changes to `rules/` directory
- Compiles all YARA rules
- Checks for duplicates
- Validates required metadata fields

### Running CI Checks Locally

```bash
# Replicate CI test run
python -m pytest core/ cli/ config/ plugins/ tests/ demo/ grafana/ kubernetes/ \
    --ignore=plugins/deepfake-detector -v --tb=short

# YARA rule validation (strict mode)
python scripts/validate_rules_ci.py

# Type checking
mypy --ignore-missing-imports core/*.py cli/*.py

# Code formatting check
black --check core/*.py cli/*.py plugins/*.py
```

---

## 7. Writing New Tests

### Test File Template

```python
#!/usr/bin/env python3
"""
HADES <Module Name> Test Suite
File: core/test_<module>.py
Version: 1.0.0
Author: DarkHorse Information Security LLC
License: Proprietary - DarkHorse Information Security LLC
"""

import os
import sys
import unittest
import tempfile

sys.path.insert(0, os.path.join(os.path.dirname(__file__)))

from module_name import ClassName


class TestClassName(unittest.TestCase):
    """Test <description>."""

    def setUp(self):
        """Create test fixtures."""
        self.tmpdir = tempfile.mkdtemp(prefix="hades_test_")

    def tearDown(self):
        """Clean up test fixtures."""
        import shutil
        shutil.rmtree(self.tmpdir, ignore_errors=True)

    def test_expected_behavior(self):
        """Verify <specific behavior>."""
        result = ClassName().method()
        self.assertIsNotNone(result)


if __name__ == "__main__":
    unittest.main()
```

### Guidelines

1. **Every detection method needs malicious AND clean test cases.** Detection tests must verify both true positives and true negatives.

2. **Mock external dependencies.** Never depend on network access, external APIs, or files not in the repository. Use `unittest.mock` for HTTP calls, database connections, etc.

3. **Use temporary directories.** Create test files in `tempfile.mkdtemp()` and clean up in `tearDown()`.

4. **Test the public API.** Focus on exported classes and functions, not internal helpers.

5. **Follow the skip pattern for optional dependencies:**

```python
try:
    from fastapi import FastAPI
    from fastapi.testclient import TestClient
    HAS_FASTAPI = True
except ImportError:
    HAS_FASTAPI = False

class TestAPIRoutes(unittest.TestCase):
    def setUp(self):
        if not HAS_FASTAPI:
            self.skipTest("FastAPI not available")
```

6. **Use descriptive test method names** that explain the expected behavior:

```python
# Good
def test_quarantine_moves_file_to_quarantine_directory(self):
def test_release_nonexistent_entry_raises_value_error(self):

# Bad
def test_1(self):
def test_it_works(self):
```

---

## 8. Quality Standards

### Pass/Fail Criteria

Before any release, ALL of the following must pass:

| Check | Command | Target |
|-------|---------|--------|
| Full test suite | `python -m pytest core/ cli/ config/ plugins/ tests/ demo/ grafana/ kubernetes/ -v --tb=short` | 0 failures |
| CLI smoke test | `python cli/main.py --help` | Exit 0 |
| Enhanced CLI | `python core/hades_enhanced_cli.py --help` | Exit 0 |
| Detection corpus | `python -m pytest tests/test_corpus_full_validation.py -v -s` | TPR >= 90%, FPR <= 10% |
| Enterprise tests | `python -m pytest core/test_auth_*.py core/test_storage_*.py -v` | 0 failures |
| Type check | `mypy --ignore-missing-imports core/*.py cli/*.py` | No errors |
| YARA rules | `python scripts/validate_rules_ci.py` | Exit 0 |

### Coverage Goals

- All public API methods have at least one test
- All detection heuristics have true-positive and true-negative tests
- All API endpoints have request/response tests
- All error paths have tests (invalid input, missing files, etc.)

---

## 9. Troubleshooting Test Failures

### Common Issues

**ImportError on optional modules:**
Tests that require optional dependencies (PostgreSQL, Redis, YARA, scikit-learn) will **skip**, not fail. This is expected if you haven't installed the full dependency set.

```bash
# Install everything
pip install -e ".[full]"
```

**ExifTool not found:**
Some scanner tests require ExifTool on PATH.

```bash
# Verify ExifTool is installed
exiftool -ver
```

**Port conflicts on API tests:**
API tests use random ports via TestClient. If you see bind errors, ensure no other HADES instance is running on port 8666.

**Temp directory cleanup failures (Windows):**
On Windows, file locks can prevent cleanup. These manifest as warnings, not failures. They are harmless.

**Slow test runs:**
The full suite takes ~4 minutes. To speed up development:

```bash
# Run only the module you changed
python -m pytest core/test_<module>.py -v

# Stop on first failure
python -m pytest core/ -x -v

# Run tests in parallel (requires pytest-xdist)
pip install pytest-xdist
python -m pytest core/ -n auto -v
```

**YARA compilation failures:**
If YARA rule tests fail, verify your `yara-python` installation matches the rules:

```bash
python -c "import yara; print(yara.YARA_VERSION)"
python scripts/validate_rules.py
```
