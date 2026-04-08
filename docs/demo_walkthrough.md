# HADES Demo Walkthrough

Step-by-step instructions for demonstrating HADES capabilities. Each section includes the command to run, what to expect, and talking points for a live or recorded presentation.

**Prerequisites:**
- Python 3.8+ with `pip install -e ".[dev]"` (or `pip install -r requirements.txt`)
- Pillow (`pip install Pillow`) for corpus generation
- curl for API demos
- Optional: `yara-python` for YARA detection, `scikit-learn` for ML features

---

## Setup

Create a workspace for demo artifacts:

```bash
cd /path/to/HADES
mkdir -p demo_workspace/evidence demo_workspace/clean demo_workspace/reports
```

---

## 1. Generate Test Corpus

**What:** Create files with weaponised metadata for scanning. No real malware is involved — these are crafted JPEG/PDF/SVG files with attack payloads injected into metadata fields only.

**Command:**

```bash
python -c "
from tests.corpus.generators.exif_injection import *
from tests.corpus.generators.document_attacks import *
from tests.corpus.generators.polyglot_files import *
from tests.corpus.generators.clean_baseline import *

# Malicious files
generate_php_backdoor_jpeg('demo_workspace/evidence/exif_php_backdoor.jpg')
generate_xss_injection_jpeg('demo_workspace/evidence/exif_xss_injection.jpg')
generate_sql_injection_jpeg('demo_workspace/evidence/exif_sql_injection.jpg')
generate_command_injection_jpeg('demo_workspace/evidence/exif_cmd_injection.jpg')
generate_base64_payload_jpeg('demo_workspace/evidence/exif_base64_payload.jpg')
generate_pdf_javascript('demo_workspace/evidence/doc_pdf_javascript.pdf')
generate_svg_script_injection('demo_workspace/evidence/doc_svg_script.svg')
generate_jpeg_zip_polyglot('demo_workspace/evidence/poly_jpeg_zip.jpg')

# Clean baseline files
generate_camera_jpeg('demo_workspace/clean/clean_camera.jpg')
generate_minimal_jpeg('demo_workspace/clean/clean_minimal.jpg')
generate_clean_png('demo_workspace/clean/clean_image.png')
generate_clean_pdf('demo_workspace/clean/clean_document.pdf')

print('Corpus generated: 8 malicious + 4 clean files.')
"
```

**Expected output:** Confirmation message. The `evidence/` directory will contain 8 files, `clean/` will contain 4.

**Talking points:**
- HADES includes a built-in corpus generator with 28+ attack variants
- Attack categories: EXIF injection, document threats, polyglot files
- Each file is a valid image/document — the attacks hide in metadata
- Clean baseline files verify zero false positives

---

## 2. Single File Scan — PHP Backdoor

**What:** Scan a JPEG that has `<?php system($_GET['cmd']); ?>` injected into its EXIF Artist field.

**Command:**

```bash
python cli/hades_cli.py demo_workspace/evidence/exif_php_backdoor.jpg -v
```

**Expected output:** Scanner identifies the PHP backdoor pattern in the Artist EXIF field. Verbose mode shows each metadata field examined and the specific detection that fired.

**Talking points:**
- HADES examines metadata without executing the file
- The PHP payload is embedded in a standard EXIF field — invisible to normal image viewers
- This technique is used in real attacks: uploaded "images" that execute as PHP on misconfigured servers
- Detection is pattern-based (regex + heuristics), not signature-only

---

## 3. Directory Scan — All Evidence Files

**What:** Recursively scan the entire evidence directory to detect all 8 threat types at once.

**Command:**

```bash
python cli/hades_cli.py -r -v demo_workspace/evidence/
```

**Expected output:** Each file is scanned. Findings include:
| File | Attack Type | Detection |
|------|-------------|-----------|
| `exif_php_backdoor.jpg` | PHP code injection | `<?php` pattern in EXIF |
| `exif_xss_injection.jpg` | XSS/JavaScript | `<script>` tag in metadata |
| `exif_sql_injection.jpg` | SQL injection | SQL keywords in EXIF |
| `exif_cmd_injection.jpg` | OS command injection | Shell command patterns |
| `exif_base64_payload.jpg` | Encoded payload | Base64 blob in metadata |
| `doc_pdf_javascript.pdf` | PDF JavaScript | `/JavaScript` action |
| `doc_svg_script.svg` | SVG script injection | `<script>` in SVG |
| `poly_jpeg_zip.jpg` | Polyglot file | Dual JPEG+ZIP headers |

**Talking points:**
- Batch scanning handles mixed file types automatically
- Recursive mode (`-r`) processes nested subdirectories
- Each detection includes severity score and field location

---

## 4. Clean File Validation — Zero False Positives

**What:** Scan legitimate files to verify HADES does not produce false alarms.

**Command:**

```bash
python cli/hades_cli.py -r -v demo_workspace/clean/
```

**Expected output:** All files scanned with minimal or no findings. Clean files should score well below the threat threshold.

**Talking points:**
- False positives are a major concern for production security tools
- HADES validates against a clean baseline to maintain FPR <= 10%
- Clean files include realistic metadata (camera EXIF, PDF bookmarks, etc.)

---

## 5. Advanced Detection with YARA Rules

**What:** Enable advanced detection mode which layers YARA rule matching on top of heuristic analysis.

**Command:**

```bash
python cli/hades_cli.py -a demo_workspace/evidence/exif_php_backdoor.jpg -v
```

**Expected output:** Additional YARA-based detections fire alongside the heuristic findings. Rule names and severity levels are displayed.

**Talking points:**
- HADES ships with 7 YARA rule files (see `rules/` directory):
  - `enhanced_detection.yara` — reverse shells, suspicious metadata
  - `advanced_threats.yar` — encoded payloads, web shells
  - `enterprise_threats.yar` — ransomware, APT toolkits
  - `steganography.yar` — hidden data in images
  - `polyglot_detection.yar` — dual-format files
  - `suspicious_metadata.yar` — anomalous field patterns
  - `metadata_threats.yar` — embedded executables
- Custom rules can be added to `rules/` or passed via `--yara-rules`
- Validate rules: `python scripts/validate_rules.py`

---

## 6. Enhanced Detection Engine

**What:** The enterprise-grade scanner with threat scoring, deep format analysis, and sorting.

**Commands:**

```bash
# Scan with threat score sorting
python core/hades_enhanced_cli.py -r demo_workspace/evidence/ --sort-by score

# Filter to high-severity only
python core/hades_enhanced_cli.py -r demo_workspace/evidence/ --min-score 50 --sort-by score
```

**Expected output:** Each file gets a numeric threat score (0-100). Results sorted highest-first. The `--min-score 50` filter shows only confirmed threats.

**Talking points:**
- Threat scores aggregate multiple detection signals into a single number
- Deep format analysis parses PDF internals, Office OLE, SVG DOM
- `--sort-by score` prioritizes the most dangerous files for triage
- `--min-score` / `--max-score` filters reduce noise during investigations
- Multi-threaded with `--workers N` for large evidence sets

---

## 7. Report Generation

**What:** Export scan results in JSON, HTML, and CSV formats for documentation and compliance.

**Commands:**

```bash
# JSON report (machine-readable, for SIEM ingestion)
python cli/hades_cli.py -r demo_workspace/evidence/ \
  -o demo_workspace/reports/report.json --report-format json

# HTML report (human-readable, for stakeholders)
python cli/hades_cli.py -r demo_workspace/evidence/ \
  -o demo_workspace/reports/report.html --report-format html

# CSV report (for spreadsheet analysis)
python cli/hades_cli.py -r demo_workspace/evidence/ \
  -o demo_workspace/reports/report.csv --report-format csv
```

**Expected output:** Report files created in `demo_workspace/reports/`. JSON includes structured findings with severity scores, field names, and recommendations. HTML includes a formatted table. CSV is importable into Excel/Google Sheets.

**Talking points:**
- JSON reports include: report ID, timestamp, file fingerprint (SHA-256), findings with severity 0-10, recommendations
- Three redaction levels: `--redaction low` (GPS only), `medium` (sensitive fields + emails + paths), `high` (remove + hash)
- `--anonymize` flag strips PII before reporting
- Reports are forensically sound with hash-based file identification

---

## 8. Metadata Sanitization

**What:** Strip dangerous metadata from files while preserving a backup.

**Command:**

```bash
python cli/hades_cli.py -R demo_workspace/evidence/exif_xss_injection.jpg
```

**Expected output:** Displays sanitization recommendations for each dangerous metadata field found.

**Talking points:**
- Sanitization removes attack payloads from metadata fields
- Original file is backed up before modification
- Use case: cleaning uploaded files before serving them to users
- The API also exposes `POST /api/v1/sanitize` for automated pipelines

---

## 9. REST API Server

**What:** Start the FastAPI server and interact with it via curl.

**Commands:**

```bash
# Start the server (runs on port 8666 by default)
python core/hades_enhanced_cli.py --serve --port 8666 &

# Wait for startup
sleep 3

# Health check
curl -s http://localhost:8666/api/v1/health | python -m json.tool

# Scan a file via upload
curl -s -X POST \
  -H "X-API-Key: your-api-key-here" \
  -F "file=@demo_workspace/evidence/exif_php_backdoor.jpg" \
  http://localhost:8666/api/v1/scan/file | python -m json.tool

# Check YARA rules status
curl -s -H "X-API-Key: your-api-key-here" \
  http://localhost:8666/api/v1/rules/status | python -m json.tool

# List plugins
curl -s -H "X-API-Key: your-api-key-here" \
  http://localhost:8666/api/v1/plugins/list | python -m json.tool

# Stop the server
kill %1
```

**Expected output:**
- Health endpoint returns: `{"status": "healthy", "version": "0.7.1", ...}`
- Scan endpoint returns: `{"scan_id": "...", "threat_score": N, "threat_level": "...", ...}`
- Rules endpoint returns: list of loaded YARA rule files
- Plugins endpoint returns: loaded detection plugins with metrics

**Talking points:**
- FastAPI with async support, OpenAPI docs at `/docs`
- Authentication via `X-API-Key` header
- WebSocket at `/ws/{client_id}` for real-time updates
- Web dashboard at `http://localhost:8666/dashboard/`
- Rate limiting: 60 req/min for scans, 200 req/min for health
- Batch scanning: `POST /api/v1/scan/batch` with multiple files
- Evidence chain endpoints at `/api/v1/evidence/`
- SIEM export: `POST /api/v1/export` with format selection

---

## 10. Web Dashboard

**What:** Open the built-in web dashboard for visual scanning.

**Steps:**
1. Start the API server: `python core/hades_enhanced_cli.py --serve --port 8666`
2. Open browser to `http://localhost:8666/dashboard/`
3. Enter API key: `your-api-key-here`
4. Navigate sections: Scan, Monitor, Cases, Audit Trail, System

**Talking points:**
- Self-contained vanilla HTML/CSS/JS — no external CDN dependencies
- Dark theme (#1a1a2e) with threat-level color coding
- Drag-and-drop file upload for scanning
- Live WebSocket updates for monitoring alerts
- Case management with scan linking

---

## 11. ML Anomaly Detection

**What:** Use Isolation Forest to detect statistical outliers in metadata feature vectors.

**Commands:**

```bash
# Check model status
python core/hades_enhanced_cli.py --ml-status

# Train on known-good files
python core/hades_enhanced_cli.py --ml-train demo_workspace/clean/

# Scan with ML enabled
python core/hades_enhanced_cli.py -r --ml-detect demo_workspace/evidence/
```

**Expected output:** ML status shows model availability. Training builds an Isolation Forest from 15 metadata features. ML-enabled scans report anomaly scores alongside traditional detections.

**Talking points:**
- 15 features extracted per file: field count, value lengths, entropy, special char ratios, etc.
- Isolation Forest identifies files that are statistically unusual vs. the baseline
- Works alongside (not replacing) YARA and heuristic detection
- Requires scikit-learn: `pip install scikit-learn`
- Graceful degradation: ML features silently disabled if sklearn is unavailable

---

## 12. SIEM Integration

**What:** Export scan results in SIEM-compatible formats.

**Commands:**

```bash
# CEF format (ArcSight)
python core/hades_enhanced_cli.py -r demo_workspace/evidence/ --siem-format cef

# Syslog format (RFC 5424)
python core/hades_enhanced_cli.py -r demo_workspace/evidence/ --siem-format syslog

# STIX 2.1 bundle
python core/hades_enhanced_cli.py -r demo_workspace/evidence/ --siem-format stix

# ECS for Elastic
python core/hades_enhanced_cli.py -r demo_workspace/evidence/ --siem-format ecs
```

**Expected output:** Scan results formatted for the selected SIEM platform.

**Talking points:**
- Supported formats: Syslog (RFC 5424), CEF (ArcSight), STIX 2.1, LEEF (QRadar), ECS (Elastic)
- Real-time forwarding with `--siem-output syslog --siem-target host:port`
- HTTP forwarding for cloud SIEMs: `--siem-output http --siem-target https://siem.example.com/api`

---

## 13. Corpus Validation (Detection Rates)

**What:** Run the automated test suite that calculates True Positive Rate, False Positive Rate, and False Negative Rate.

**Command:**

```bash
python -m pytest tests/test_corpus_full_validation.py -v -s
```

**Expected output:**
```
True Positive Rate:  100% (28/28 malicious files detected)
False Positive Rate:   0% (0/10 clean files flagged)
False Negative Rate:   0% (0/28 malicious files missed)
```

**Talking points:**
- Validation thresholds: TPR >= 90%, FPR <= 10%
- 28 malicious test files across 4 attack categories
- 10 clean baseline files with realistic metadata
- Run this after any detection logic changes to verify no regression

---

## 14. Performance Benchmark

**What:** Measure scan throughput, latency, and memory usage.

**Commands:**

```bash
# Quick benchmark (~30 seconds)
python core/hades_enhanced_cli.py --benchmark-quick

# Full benchmark (~2-5 minutes)
python core/hades_enhanced_cli.py --benchmark
```

**Expected output:** Table showing:
- Single-file scan latency (ms)
- Batch throughput (files/second)
- Memory usage per scan (MB)
- ML feature extraction time
- Cache performance (hit rate %)

**Talking points:**
- Cross-platform benchmarks (uses psutil, not Unix-specific `resource`)
- Results saved to JSON for trend tracking
- Standalone mode: `python core/benchmarks.py --quick`

---

## 15. Full Test Suite

**What:** Run the complete test suite (1,781+ tests) to verify all components.

**Command:**

```bash
python -m pytest core/ cli/ config/ plugins/ tests/ -v --tb=short
```

**Expected output:** All tests pass (1,781+ passed, 0 errors, 0 failures).

**Talking points:**
- Tests cover: metadata parsing, detection engine, YARA rules, REST API, evidence chain, file monitoring, plugins, SIEM, threat intel, ML, security middleware, deep format analysis, all integrations
- Corpus tests validate detection rates against real-world attack patterns
- API tests use FastAPI TestClient for full endpoint coverage
- All test files generate their own fixtures — no external dependencies

---

## Cleanup

Remove the demo workspace when finished:

```bash
rm -rf demo_workspace/
```

---

## Recording Tips

### Screen Recording (OBS, Camtasia, etc.)
1. Use `scripts/demo_recording.sh` — it pauses between steps for you to narrate
2. Set terminal font to 16-18pt for readability
3. Use a dark terminal theme for contrast
4. Record at 1080p or higher

### Asciinema
1. Install: `pip install asciinema`
2. Record: `asciinema rec --command="bash scripts/demo_asciinema.sh" hades_demo.cast`
3. Play locally: `asciinema play hades_demo.cast`
4. Upload: `asciinema upload hades_demo.cast`
5. Embed in docs: use the asciinema player `<script>` tag

### Timing Adjustments
Both scripts have configurable timing variables at the top:
- `demo_recording.sh`: Interactive — you control the pace with ENTER
- `demo_asciinema.sh`: Automated — adjust `TYPE_DELAY`, `LINE_PAUSE`, `SECTION_PAUSE`
