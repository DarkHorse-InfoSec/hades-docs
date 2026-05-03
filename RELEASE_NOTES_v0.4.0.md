# HADES v0.4.0 -- Enterprise Metadata Forensics Platform

**Release Date:** 2026-02-18

HADES v0.4.0 is a major release that transforms HADES from a CLI scanning tool into a full enterprise metadata forensics platform. This release adds ML-based anomaly detection, cloud threat intelligence enrichment, SIEM integration, production Docker deployment, a plugin system, and distribution packaging for all major platforms.

## Highlights

- **ML Anomaly Detection** -- Isolation Forest model trained on 15 metadata features flags statistical outliers that rule-based detection misses
- **Cloud Threat Intelligence** -- Enrich scan results with verdicts from VirusTotal, AbuseIPDB, OTX AlienVault, and MalwareBazaar with consensus scoring
- **SIEM Integration** -- Export findings in Syslog RFC 5424, CEF, STIX 2.1, LEEF, or ECS JSON with real-time forwarding via TCP/UDP/HTTP
- **REST API Server** -- 11+ authenticated endpoints for scan, sanitize, monitor, threat intel, and ML operations
- **Real-Time File Monitoring** -- Watch directories for new/modified files with auto-scan and webhook alerts
- **Plugin System** -- Extend detection with custom plugins via the `HADESPlugin` base class
- **Docker Deployment** -- Production-ready Docker image with health checks, resource limits, and non-root user
- **Distribution Packages** -- PyInstaller standalone binaries and Electron GUI installers for Windows, macOS, and Linux
- **238+ Passing Tests** -- Comprehensive test coverage across all modules

## What's New

### ML Anomaly Detection
- `core/ml_detection.py` with `MLDetectionEngine`, `MetadataFeatureExtractor`, and `AnomalyDetector`
- 15-feature vector: header entropy, file size ratio, metadata field count, GPS presence, suspicious string counts, EXIF completeness, and more
- Auto-retrain after configurable sample threshold
- Baseline model generation from synthetic data
- REST API: `/api/v1/ml/status`, `/api/v1/ml/retrain`, `/api/v1/ml/features`
- Plugin wrapper: `plugins/ml_anomaly_plugin.py`
- CLI: `--ml-detect`, `--ml-train`, `--ml-status`, `--ml-baseline`

### Cloud Threat Intelligence
- `core/cloud_threat_intel.py` with `ThreatIntelManager` and 4 providers
- SQLite cache with configurable TTL to reduce API calls
- Consensus scoring across multiple providers
- Offline mode for air-gapped environments
- REST API: `/api/v1/threat-intel/lookup`, `/api/v1/threat-intel/cache/stats`, `/api/v1/threat-intel/cache/flush`
- CLI: `--threat-intel`, `--ti-providers`, `--ti-offline`, `--ti-lookup`, `--ti-cache-stats`

### SIEM Integration
- `core/siem_integration.py` with `SIEMExporter` and `SIEMForwarder`
- 5 output formats: Syslog RFC 5424, CEF, STIX 2.1, LEEF, ECS JSON
- Real-time forwarding via TCP, UDP, or HTTP with retry logic and queue buffering
- File output for offline collection
- REST API: `/api/v1/siem/export`, `/api/v1/siem/config`
- CLI: `--siem-format`, `--siem-output`, `--siem-target`, `--export-scan`

### REST API Server
- Flask-based server with API key authentication
- Endpoints: health, rules status, single/batch scan, result retrieval, sanitization, monitor control, threat intel, ML, SIEM
- SQLite-backed result persistence
- CLI: `--serve --port 8666`

### Real-Time File Monitoring
- `core/file_monitor.py` with watchdog-based filesystem event monitoring
- Auto-scans new and modified files through ExifScanner and EnhancedDetectionEngine
- Configurable alert thresholds, JSON Lines alert logs, webhook delivery
- CLI: `--monitor /path`, `--webhook URL`, `--alert-threshold N`

### Plugin System
- `core/plugin_api.py` with `HADESPlugin` abstract base class and `PluginManager`
- Plugin discovery, sandboxed execution with timeout, per-plugin metrics, hot-reload
- Example plugins: hash checker, entropy analyzer, ML anomaly
- CLI: `--list-plugins`, `--plugins-dir`, `--disable-plugins`

### Docker Deployment
- `docker/hades-api.dockerfile` -- Production image based on Python slim
- `docker/docker-compose.yml` -- Full stack with volume mounts and resource limits
- Non-root user, health checks, ML baseline generation on startup
- Environment variables for API key and threat intel provider keys

### Distribution Packaging
- **PyInstaller**: Dual executables (`hades-cli` for scanning, `hades-server` for API server) with all dependencies bundled
- **Electron GUI**: Cross-platform desktop application with React + Vite + Tailwind CSS
  - Windows: NSIS installer
  - macOS: DMG disk image
  - Linux: AppImage

### GUI Dashboards
- `APIDashboard.jsx` -- REST API controls, scan history, batch upload
- `MonitorDashboard.jsx` -- File monitor controls, live status, alert feed
- `PluginDashboard.jsx` -- Plugin management, metrics, reload
- `Navigation.jsx` -- Collapsible sidebar with server status indicator

## Installation

### pip install
```bash
git clone https://github.com/DarkHorse-InfoSec/hades-docs.git
cd hades && pip install -r requirements.txt
python cli/hades_cli.py --help
```

### Docker
```bash
docker compose -f docker/docker-compose.yml up -d
curl http://localhost:8666/api/v1/health
```

### PyInstaller Binary
Download `hades-cli` or `hades-server` from the Releases page. No Python required.

### Electron GUI
Download the installer for your platform from the Releases page.

## Breaking Changes

- **Directory structure**: Files reorganized into `core/`, `cli/`, `gui/`, `docker/`, `rules/`, `config/`, `templates/`, `docs/`, `examples/`, `plugins/`. Import paths updated accordingly.
- **New config sections**: `cli/hades_config.json` now includes `threat_intel` and `ml_detection` configuration blocks.
- **New dependencies**: scikit-learn, numpy, and joblib added to `requirements.txt` for ML detection.

## Bug Fixes

- Fixed YARA import chain in `cli/advanceddetection.py` and `cli/corescanner.py` -- no longer crashes when yara-python is not installed
- Fixed `core/hades_api.py` optional import guards for corescanner, sanitizer, and file_monitor
- Resolved all 5 pre-existing `test_plugin_api` test errors

## Test Summary

| Module | Tests | Status |
|---|---|---|
| Plugin API | 29 | Passed |
| Cloud Threat Intel | 43 | Passed |
| ML Detection | 59 | Passed |
| REST API | 48 | Passed |
| SIEM Integration | 40 | Passed |
| File Monitor | 18 | Skipped (watchdog optional) |
| CLI | 1 | Passed |
| **Total** | **238 passed, 18 skipped** | **0 failures** |

## Full Changelog

See [CHANGELOG.md](CHANGELOG.md) for the complete list of changes.

## Contributors

- DarkHorse Information Security LLC
