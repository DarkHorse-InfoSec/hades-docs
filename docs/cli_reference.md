# CLI Reference

HADES provides two CLI entry points: the basic CLI (`hades`) for simple scanning and the enhanced CLI (`hades-enhanced`) for the full feature set. This reference covers the enhanced CLI.

## Usage

```bash
hades-enhanced [OPTIONS] [TARGET...]
```

`TARGET` is one or more file paths or directory paths to scan. When no target is given, certain standalone commands (e.g., `--serve`, `--list-plugins`, `--ml-status`) can be run without a target.

---

## Basic Scanning

| Flag | Description | Default |
|---|---|---|
| `TARGET` | File or directory path(s) to scan | Required (unless using standalone commands) |
| `-r`, `--recursive` | Scan directories recursively | `false` |
| `-v`, `--verbose` | Enable verbose output | `false` |
| `--file-types EXT [EXT ...]` | Filter by file extensions (e.g., `.jpg .png .pdf`) | All supported types |
| `--max-file-size MB` | Skip files larger than MB megabytes | `50` |

**Examples:**

```bash
# Scan a single file
hades-enhanced suspicious.jpg

# Scan a directory recursively with verbose output
hades-enhanced -r -v /evidence/

# Scan only JPEG and PNG files
hades-enhanced -r --file-types .jpg .png /evidence/
```

---

## Output and Reporting

| Flag | Description | Default |
|---|---|---|
| `--format FORMAT` | Output format: `text`, `json`, `html`, `csv` | `text` |
| `--output FILE` | Write output to file instead of stdout | stdout |
| `--siem-format FORMAT` | SIEM export format: `syslog`, `cef`, `stix`, `leef`, `ecs` | None |
| `--siem-output METHOD` | SIEM delivery: `stdout`, `file`, `syslog`, `http` | `stdout` |
| `--siem-target TARGET` | SIEM destination (host:port, file path, or URL) | None |
| `--export-scan SCAN_ID` | Re-export a previous scan result | None |

**Examples:**

```bash
# JSON report to file
hades-enhanced -r /evidence/ --format json --output report.json

# CEF output to syslog server
hades-enhanced -r /evidence/ --siem-format cef --siem-output syslog --siem-target 10.0.0.50:514

# Re-export a previous scan as STIX
hades-enhanced --export-scan abc-123 --siem-format stix
```

---

## API Server

| Flag | Description | Default |
|---|---|---|
| `--serve` | Start the REST API server | `false` |
| `--port PORT` | API server port | `8666` |
| `--host HOST` | API server bind address | `127.0.0.1` |
| `--api-workers N` | Number of uvicorn worker processes | `1` |

**Examples:**

```bash
# Start API server on default port
hades-enhanced --serve

# Start on custom port with 4 workers
hades-enhanced --serve --port 9000 --api-workers 4

# Bind to all interfaces
hades-enhanced --serve --host 0.0.0.0 --port 8666
```

---

## File Monitoring

| Flag | Description | Default |
|---|---|---|
| `--monitor DIR [DIR ...]` | Watch directories for new/modified files | None |
| `--alert-threshold N` | Minimum threat score to trigger an alert | `7` |
| `--webhook URL` | POST alert payloads to this URL | None |

**Examples:**

```bash
# Monitor a single directory
hades-enhanced --monitor /evidence/incoming

# Monitor multiple directories with webhook
hades-enhanced --monitor /dir1 --monitor /dir2 --webhook https://hooks.example.com/alert

# Custom alert threshold
hades-enhanced --monitor /dir --alert-threshold 5
```

---

## Plugins

| Flag | Description | Default |
|---|---|---|
| `--list-plugins` | List loaded plugins and exit | `false` |
| `--plugins-dir DIR` | Override the plugins directory | `plugins` |
| `--disable-plugins` | Skip loading all detection plugins | `false` |

**Examples:**

```bash
# List available plugins
hades-enhanced --list-plugins

# Scan with custom plugin directory
hades-enhanced --plugins-dir /path/to/plugins -r /evidence/

# Scan without plugins
hades-enhanced --disable-plugins -r /evidence/
```

---

## SIEM Integration

| Flag | Description | Default |
|---|---|---|
| `--siem-format FORMAT` | Output format: `syslog`, `cef`, `stix`, `leef`, `ecs` | None |
| `--siem-output METHOD` | Delivery method: `stdout`, `file`, `syslog`, `http` | `stdout` |
| `--siem-target TARGET` | Destination (host:port, file path, or URL) | None |

See the [SIEM Integration Guide](siem_integration_guide.md) for platform-specific setup.

---

## Threat Intelligence

| Flag | Description | Default |
|---|---|---|
| `--threat-intel` | Enable threat intel enrichment during scans | `false` |
| `--ti-providers PROV [PROV ...]` | Specific providers to query | All configured |
| `--ti-offline` | Use only cached results (no API calls) | `false` |
| `--ti-lookup IOC` | Standalone IOC lookup (hash, IP, URL, domain) | None |
| `--ti-cache-stats` | Display threat intel cache statistics | `false` |

**Examples:**

```bash
# Scan with threat intel
hades-enhanced -r --threat-intel /evidence/

# Lookup a specific hash
hades-enhanced --ti-lookup d41d8cd98f00b204e9800998ecf8427e

# Offline mode with specific providers
hades-enhanced --ti-lookup 8.8.8.8 --ti-providers abuseipdb otx --ti-offline
```

---

## ML Detection

| Flag | Description | Default |
|---|---|---|
| `--ml-detect` | Enable ML anomaly detection during scans | `false` |
| `--ml-train DIR` | Train ML model on known-good files in DIR | None |
| `--ml-baseline` | Generate a baseline model from synthetic data | `false` |
| `--ml-status` | Display ML engine status | `false` |
| `--ml-ensemble` | Use ensemble ML detector (Isolation Forest + Random Forest + XGBoost) | `false` |

**Examples:**

```bash
# Scan with ML detection
hades-enhanced -r --ml-detect /evidence/

# Generate baseline model
hades-enhanced --ml-baseline

# Train on known-good files
hades-enhanced --ml-train /path/to/known-good/

# Use ensemble detector
hades-enhanced -r --ml-ensemble /evidence/
```

---

## Detection Depth (v0.6.0)

| Flag | Description | Default |
|---|---|---|
| `--behavioral` | Enable behavioral analysis and campaign detection | `false` |
| `--mitre-map` | Enable MITRE ATT&CK mapping for findings | `false` |
| `--schedule-feeds` | Start threat intelligence feed scheduler | `false` |
| `--build-rule TEMPLATE` | Build a YARA rule from a template | None |
| `--list-templates` | List available YARA rule templates | `false` |
| `--malware-validate` | Run MalwareBazaar validation suite | `false` |

**Examples:**

```bash
# Behavioral analysis with MITRE mapping
hades-enhanced --behavioral --mitre-map -r /evidence/

# List and build YARA rules from templates
hades-enhanced --list-templates
hades-enhanced --build-rule metadata_injection

# Start threat intel feed scheduler
hades-enhanced --schedule-feeds
```

---

## Evidence Chain

| Flag | Description | Default |
|---|---|---|
| `--case-create NAME` | Create a new investigation case | None |
| `--case-id ID` | Link scans to an existing case | None |
| `--audit-verify` | Verify the integrity of the audit chain | `false` |
| `--export-case ID` | Export case evidence as a ZIP package | None |

**Examples:**

```bash
# Create a case and scan into it
hades-enhanced --case-create "Phishing Investigation 2026-02"
hades-enhanced --case-id CASE_UUID -r /evidence/

# Verify audit chain integrity
hades-enhanced --audit-verify

# Export evidence package
hades-enhanced --export-case CASE_UUID
```

---

## Enterprise Auth (v0.6.0)

| Flag | Description | Default |
|---|---|---|
| `--create-admin USER` | Create an admin user | None |
| `--create-user USER` | Create a new user | None |
| `--user-role ROLE` | Set role for new user: `admin`, `analyst`, `viewer` | `analyst` |
| `--list-users` | List all users | `false` |
| `--license-info` | Display license information | `false` |
| `--license-key KEY` | Set license key | None |

---

## Storage (v0.6.0)

| Flag | Description | Default |
|---|---|---|
| `--db-backend BACKEND` | Database backend: `sqlite`, `postgres` | `sqlite` |
| `--db-dsn DSN` | Database connection string | None |
| `--redis-url URL` | Redis connection URL | None |
| `--encrypt` | Enable encryption at rest | `false` |
| `--generate-encryption-key` | Generate a new AES-256 encryption key | `false` |
| `--create-tenant NAME` | Create a new tenant | None |
| `--list-tenants` | List all tenants | `false` |
| `--migrate-db` | Run database migrations | `false` |

---

## Benchmarks

| Flag | Description | Default |
|---|---|---|
| `--benchmark` | Run full performance benchmark suite | `false` |
| `--benchmark-quick` | Run quick benchmark | `false` |

---

## Network and Cloud Integrations

| Flag | Description | Default |
|---|---|---|
| `--zeek-watch DIR` | Watch Zeek extract_files directory | None |
| `--suricata-watch DIR` | Watch Suricata filestore directory | None |
| `--email-gateway` | Start SMTP email gateway | `false` |
| `--smtp-port PORT` | SMTP gateway listen port | `10025` |
| `--s3-scan BUCKET` | Scan an AWS S3 bucket | None |
| `--gcs-scan BUCKET` | Scan a Google Cloud Storage bucket | None |
| `--azure-scan CONTAINER` | Scan an Azure Blob container | None |
| `--cloud-prefix PREFIX` | Filter cloud objects by prefix | None |
| `--cloud-tag-results` | Tag scanned cloud objects with results | `false` |
| `--ci-scan FILE [FILE ...]` | CI/CD pipeline scan of changed files | None |
| `--ci-threshold SCORE` | Pass/fail threshold for CI/CD | `7.0` |
| `--ci-format FORMAT` | CI output format: `table`, `json`, `sarif`, `github-comment`, `gitlab-note` | `table` |
| `--slack-bot` | Start Slack bot | `false` |
| `--teams-bot` | Start Teams bot | `false` |

---

## Common Workflows

### Full-spectrum Scan

```bash
hades-enhanced -r -v \
  --ml-detect --threat-intel --behavioral --mitre-map \
  --format json --output full_report.json \
  /evidence/
```

### CI/CD Pipeline Gate

```bash
hades-enhanced --ci-scan *.jpg *.png *.pdf \
  --ci-threshold 5.0 --ci-format sarif
```

### Continuous Monitoring with SIEM

```bash
hades-enhanced --monitor /evidence/incoming \
  --alert-threshold 5 \
  --siem-format cef --siem-output syslog --siem-target siem.corp:514 \
  --webhook https://hooks.slack.com/services/XXX
```

### Forensic Investigation

```bash
# Create a case
hades-enhanced --case-create "IR-2026-0042"

# Scan evidence into the case
hades-enhanced --case-id IR_UUID -r --ml-detect --threat-intel /evidence/

# Verify chain of custody
hades-enhanced --audit-verify

# Export evidence package
hades-enhanced --export-case IR_UUID
```
