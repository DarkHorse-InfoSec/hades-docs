# Changelog

All notable changes to HADES will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **ZIP Forensic Analyzer** (`core/zip_forensics.py`): Zombie ZIP detection (compression method mismatch), ZIP concatenation (Gootloader-style), CRC32 mismatches, ZIP bomb detection (ratio + nested + total size), malformed central directory validation, timestamp anomalies, and entry name tricks (RLO, null byte, double extension).
- **ZIP Structure YARA Rules** (`rules/zip_structure.yar`): 8 rules for ZIP structural anomalies including Zombie ZIP method mismatch, multiple EOCD records, ZIP bomb indicators, RLO filenames, nested archives, Gootloader campaign patterns, null byte filenames, and encrypted archive detection.
- **Image Forensic Analyzer** (`core/image_forensics.py`): Decompression bomb detection across JPEG/PNG/GIF/BMP via raw header parsing, oversized/malicious ICC color profile detection, EXIF thumbnail vs main image divergence analysis, IPTC injection detection, TIFF IFD chain validation (circular references, invalid offsets, duplicate tags), and GIF frame analysis (comment injection, excessive frames, zero-delay timing).
- **Timestamp Forensic Analyzer** (`core/timestamp_forensics.py`): Anti-forensic timestamp stomping detection comparing filesystem timestamps against embedded metadata, impossible date detection, zeroed/mass-reset timestamp identification, and creation-after-modification paradox detection on Windows.
- **SFX Archive Detector** (`core/sfx_detector.py`): Self-extracting archive detection for PE/ELF/Mach-O executables with embedded ZIP/RAR/7z/CAB archives, 10 known SFX tool families (WinRAR, NSIS, UPX, PyInstaller, 7-Zip, Inno Setup, AutoIt, Electron, InstallShield, WiX), and archive-at-EOF appended payload detection.
- **PDF Shadow Attack Detection** (`core/deep_format_analyzer.py`): Incremental update detection scanning content appended after `%%EOF` for injected JavaScript, Launch actions, form replacement, page content substitution, XFA forms, and RichMedia.
- **PDF Stream Filter Chain Detection** (`core/deep_format_analyzer.py`): Deep filter chain obfuscation detection (3+ layers), unusual filter combinations, suspicious filters (JBIG2Decode CVE-2009-0658, Crypt, RunLengthDecode), Object Stream detection, and heavy ASCII encoding layer analysis.
- **Spreadsheet Formula Injection** (`core/deep_format_analyzer.py`): Detection for dangerous functions (CMD, EXEC, SYSTEM, CALL, HYPERLINK, WEBSERVICE, IMPORTXML, IMPORTDATA, FILTERXML, DDE via formula) and external data connections in XLSX files.
- **DOCX Hidden Content Detection** (`core/deep_format_analyzer.py`): Track changes detection (hidden text via `w:vanish`, tracked deletions via `w:del`, hidden bookmarks), comment injection scanning, custom XML part analysis, and glossary document inspection.
- **OLE Equation Editor Detection** (`core/deep_format_analyzer.py`): Equation Editor ClassID detection (CVE-2017-11882 class), Package Shell Object ClassID, `Equation.3` string references, and executable signatures in OLE embeddings.
- **SVG CSS Injection Detection** (`core/deep_format_analyzer.py`): CSS injection pattern detection for `expression()`, `-moz-binding`, `behavior:url()`, and `javascript:` in CSS `@import`/`url()` within `<style>` elements.
- **Audio Forensic Analyzer** (`core/audio_forensics.py`): MP3 ID3v2 tag analysis (oversized tags, PRIV frame smuggling, executable payloads, WXXX/TXXX injection), ID3v1 injection, WAV chunk injection and bomb detection, OGG Vorbis comment abuse, FLAC metadata block anomalies, and MP4/M4A atom injection.
- **Video Forensic Analyzer** (`core/video_forensics.py`): MKV/WebM subtitle track injection (ASS/SSA script content), MKV attachment abuse (embedded executables), MP4/MOV atom injection in moov/udta metadata, AVI RIFF chunk injection, FLV script data analysis, and standalone subtitle file detection.
- **RAR/7z Forensic Analyzer** (`core/rar7z_forensics.py`): RAR5 solid archive bomb detection, archive-level header encryption, compression ratio bombs, oversized recovery records, RAR4 encrypted/solid flags, RAR4 comment injection, 7z start header CRC mismatches, invalid header offsets, deep filter chains, and bomb indicators.
- **VBA Stomping Detector** (`core/vba_stomping.py`): Source/p-code mismatch detection in Office macro documents, missing VBA source with large p-code, dangerous patterns in p-code but not source, module count mismatches, stomping tool artifacts (EvilClippy), locked projects, suspicious references, and auto-execution entry points.
- **Steganography Analyzer** (`core/stego_analyzer.py`): JPEG DCT-domain analysis (quantization table quality estimation, file size anomaly, Huffman table count, scan data entropy via chi-square test), 10 stego tool artifact signatures, EOF appended data detection for JPEG/PNG/GIF, GIF/PNG palette analysis (luminance-sorted and LSB near-duplicate detection), and LSB distribution uniformity checks.
- **Archive Depth Analyzer** (`core/archive_depth_analyzer.py`): Nested decompression bomb detection with cross-format nesting tracking, total extraction size monitoring, and single-entry compression ratio analysis.
- **NTFS ADS Analyzer** (`core/ntfs_ads_analyzer.py`): NTFS Alternate Data Stream detection using Windows API (FindFirstStreamW/FindNextStreamW) with executable extension and size-based threat assessment.
- **Streaming Playlist Analyzer** (`core/streaming_analyzer.py`): HLS (.m3u8) and DASH (.mpd) analysis for external segment URLs, JavaScript injection, external encryption key URIs, mixed protocols, and XML attribute injection.
- **Unicode Homoglyph Detection** (`core/security_middleware.py`): 39 Cyrillic/Greek confusable characters and 15 bidi/zero-width control characters for filename spoofing detection.
- **MIME/Extension/Magic Consistency Validation** (`core/security_middleware.py`): Cross-referencing file extension, declared MIME type, and magic bytes to detect file type masquerading across 20 common formats.
- **Script Encoding Trick Detection** (`core/security_middleware.py`): UTF-16/UTF-32 BOM marker detection and null-byte interleaving identification in script files that evade ASCII pattern matching.
- **PUA Metadata Signatures** (`rules/pua_signatures.yar`): 7 YARA rules detecting cryptominers, adware installers, keyloggers, screenloggers, remote access tools, browser hijackers, and surveillance tools.
- **Polyglot Archive Hybrids** (`core/deep_format_analyzer.py`): Extended polyglot detection for PDF+ZIP, image+RAR, image+7z, and image+PE executable combinations.
- **ZIP Comment Injection** (`core/zip_forensics.py`): End-of-central-directory comment scanning for script injection, oversized comments, and executable signatures.
- **Zip64 Field Abuse** (`core/zip_forensics.py`): Zip64 marker detection, unnecessary Zip64 on small files, multiple EOCD locators, and size manipulation in extended fields.
- **Forensic Gap Guide** (`docs/GAPGUIDE.md`): Comprehensive detection gap tracking document with build phases and implementation blueprints.
- 297 new tests across 20 test files covering all new detection capabilities.

## [1.0.0] - 2026-03-02

### Added
- **Native SIEM Platform Connectors** (`core/siem_connectors/`): Platform-specific API integration for Splunk HEC, Elasticsearch, and Microsoft Sentinel. Abstract `SIEMConnectorBase` base class with `send_event()`, `send_batch()`, `test_connection()`, `flush()` interface. `SIEMConnectorStore` with DatabaseBackend-backed persistent configuration and delivery statistics tracking.
- **Splunk HEC Connector** (`core/siem_connectors/splunk_hec.py`): Token-based authentication, event buffering with configurable batch size, batch sends via `/services/collector/event`, health checks via `/services/collector/health/1.0`, retry with exponential backoff, SSL verification toggle, source/sourcetype/index configuration.
- **Elasticsearch Connector** (`core/siem_connectors/elasticsearch.py`): Bulk API integration with API key and basic auth support, ECS-compatible field mappings, date-based index patterns (`hades-events-YYYY.MM.DD`), HTTP 429 rate limit handling with retry-after, configurable flush intervals, and document ID generation.
- **Microsoft Sentinel Connector** (`core/siem_connectors/sentinel.py`): Azure Monitor HTTP Data Collector API integration, HMAC-SHA256 request signing with workspace key, 30MB payload chunking for large batches, CommonSecurityLog table mapping, custom log type configuration, and Azure Government cloud support.
- **SIEM Connector API Routes** (`core/siem_connectors/connector_routes.py`): Endpoints at `/api/v1/siem-connectors/` prefix — list configured connectors, add connector, remove connector, test connection, delivery status, flush buffer.
- **SIEM Connector Dashboard** (`core/dashboard/js/siem_connectors.js`): Web dashboard view with connector stats cards (events sent, success rate, last delivery), add connector form with platform-specific fields, connector table with test/flush/remove actions.
- **`siem_forward_native` Playbook Action**: New playbook action type that forwards events to configured native SIEM connectors via the connector manager, complementing the existing format-based `siem_forward` action.
- **SIEM Connector CLI Flags**: `--list-siem-connectors` (list configured connectors), `--add-siem-connector PLATFORM` (add a new connector interactively), `--remove-siem-connector ID` (remove a connector), `--test-siem-connector ID` (test connector connectivity), `--siem-connector-status` (show delivery statistics for all connectors).
- **SDK 1.1.0 Retry Logic**: Exponential backoff with configurable max retries (default: 3), base delay, max delay, and jitter. Retry only on 5xx server errors and request timeouts. Non-retryable errors (4xx) raised immediately.
- **SDK Webhook Subscription Methods**: `subscribe_webhook(url, events, secret)` for creating webhook subscriptions, `list_webhooks()` for listing active subscriptions, `delete_webhook(webhook_id)` for removing subscriptions. All subscriptions include HMAC-SHA256 signed delivery via `X-HADES-Signature` header.
- **SDK SSE Streaming**: `stream_events(event_types)` method returning an async iterator for real-time Server-Sent Event consumption with auto-reconnect on connection loss.
- **Server-Side Webhook Routes** (`core/webhook_routes.py`): Webhook subscription management at `/api/v1/webhooks/` prefix — create subscription, list subscriptions, delete subscription. HMAC-SHA256 signature generation for outbound webhook delivery. Event filtering by type. Delivery retry with exponential backoff.
- **Community Edition License Tier**: New tier between free and professional with 13 included features. Community tier requires no PostgreSQL or Redis — runs on SQLite alone. Simplified deployment for individual practitioners and small teams.
- **Dashboard Feature Gating**: License-aware dashboard initialization that fetches tier info on load. Unavailable features display lock icons with upgrade prompts. Graceful degradation when license endpoint is unreachable (defaults to free tier visibility).
- **Community Dockerfile** (`docker/hades-community.dockerfile`): Slim single-container deployment with SQLite backend, minimal dependencies, non-root user, health checks. Optimized for single-machine deployments.
- **Community Docker Compose** (`docker/docker-compose.community.yml`): SQLite-only deployment with no external service dependencies. Single container with volume mounts for data persistence and custom rules.
- **22 New MITRE ATT&CK Techniques**: Reconnaissance (T1595, T1592, T1589), Credential Access (T1552, T1555, T1556, T1110), Collection (T1560, T1119, T1213), Lateral Movement (T1021, T1570, T1080), Discovery (T1083, T1082, T1057, T1518), Persistence (T1137, T1176), Privilege Escalation (T1548, T1068), Impact (T1486). Total coverage expanded from 29 to 51+ techniques across 11 tactics.
- **15 New Finding-to-Technique Mappings**: Identity forensics findings mapped to credential access and persistence techniques. Vulnerability findings mapped to privilege escalation and impact techniques. Log analysis findings mapped to collection and lateral movement techniques. Firmware findings mapped to additional persistence and defense evasion techniques.
- **Production/Stable Classifier**: Development status changed from `4 - Beta` to `5 - Production/Stable`. Added `Framework :: FastAPI` classifier.
- **OCI Labels on Enterprise Docker Image**: Standard OCI labels (`org.opencontainers.image.*`) for source, documentation, version, vendor, title, and description on the enterprise Docker image.
- **Distribution Validation Test Suite**: Tests verifying package classifiers, Docker labels, Homebrew formula version, and distribution artifact integrity.
- **SIEM Connector Config Section**: `siem_connectors` section in `cli/hades_config.json` with per-platform connection settings, retry parameters, batch sizes, and flush intervals.
- **SIEM Connector Test Suite** (`core/siem_connectors/test_siem_connectors.py`): Tests covering base class interface, Splunk HEC token auth and batch sends, Elasticsearch Bulk API and rate limiting, Sentinel HMAC signing and payload chunking, connector store CRUD, delivery stats tracking, and API endpoints.
- **Webhook Test Suite** (`core/test_webhook_routes.py`): Tests covering subscription CRUD, HMAC-SHA256 signature verification, event filtering, delivery retry, and API endpoints.
- **SDK 1.1.0 Test Suite** (`sdk/tests/`): Extended with retry logic tests, webhook subscription tests, and SSE streaming tests.
- **Community Edition Tests**: License tier ordering validation, feature gate verification, dashboard gating behavior, community Docker configuration tests.
- **MITRE Coverage Tests**: Technique count assertions, tactic coverage validation, finding-to-technique mapping completeness tests.
- **GPS Forensic Analysis Engine** (`core/gps_forensics.py`): DMS/decimal coordinate parsing, Haversine distance, custom DBSCAN clustering (no sklearn dependency), Null Island detection, impossible travel detection. Integrated into `EnhancedDetectionEngine._analyze_gps_clustering()`. API endpoint `POST /api/v1/gps-analysis`. MITRE T1614 (System Location Discovery) mapping for all GPS finding types.
- **Quarantine Manager** (`core/quarantine_manager.py`, `core/quarantine_routes.py`): File quarantine with SQLite-backed metadata, SHA-256 hashing, lifecycle management (quarantine/release/delete), audit trail integration. API routes at `/api/v1/quarantine/` prefix. Wired into playbook engine `_action_quarantine` for automated response.
- **Metadata Sanitize Dashboard** (`core/dashboard/js/sanitize.js`): Drag-and-drop file upload, before/after metadata comparison table, sanitized file download. Dashboard sidebar nav item and hash route.
- **Configurable Detection Thresholds**: `detection_thresholds` config section in `cli/hades_config.json` with entropy, GPS, ML, behavioral, analytics, and firmware thresholds. All engines accept config with backward-compatible `.get()` fallbacks.
- **Startup Health Report**: Structured module availability banner logged at API server startup. `/api/v1/health` endpoint enhanced with `modules` dict showing component-level status.
- **GPS Forensics Test Suite** (`core/test_gps_forensics.py`): 32 tests covering Haversine distance, DMS parsing, DBSCAN clustering, Null Island detection, impossible travel, coordinate validation.
- **Quarantine Test Suite** (`core/test_quarantine.py`): 35 tests covering quarantine/release/delete lifecycle, stats, persistence, and API endpoints.
- **Troubleshooting Guide** (`docs/troubleshooting.md`): 37 entries across installation, API server, scanning, Docker, enterprise features, and ML detection.
- **Production Checklist** (`docs/production_checklist.md`): TLS setup, API key management, backup strategy, monitoring, resource sizing, upgrade procedures.

### Changed
- **License Tier Ordering**: Now includes community tier — free < community < professional < enterprise. `LicenseTier` enum updated with ordinal comparison support.
- **MITRE ATT&CK Coverage**: Expanded from 29 to 51+ techniques across 11 tactics (reconnaissance, credential access, collection, lateral movement, discovery, persistence, privilege escalation, impact, defense evasion, execution, exfiltration). Finding-to-technique mappings expanded from 37 to 52+.
- **SDK Version**: Bumped from 1.0.0 to 1.1.0 with retry logic, webhook subscriptions, and SSE streaming.
- **Enterprise Docker Image**: Now installs all 20 optional dependency groups for complete feature availability. Added OCI image labels.
- **Package Classifiers**: Development status changed from `4 - Beta` to `5 - Production/Stable`. Added `Framework :: FastAPI`.
- **Playbook Engine**: Extended with `siem_forward_native` action type for native SIEM connector integration. Expanded from 15 to 15 built-in playbooks (no new default playbooks in v1.0.0, but existing playbooks can use the new action type).
- **Dashboard Navigation**: Sidebar navigation expanded with Sanitize and Quarantine views.
- **GPS Analysis**: `HeuristicAnalyzer._analyze_gps_clustering` now delegates to `GPSForensicsAnalyzer` with fallback to legacy counter-based analysis when module unavailable.
- **Playbook Quarantine**: `PlaybookExecutor._action_quarantine` delegates to `QuarantineManager` when available, with fallback to simple file move.
- **Detection Engine Config**: `HeuristicAnalyzer`, `HADESMetadataParser`, `CampaignDetector`, `FirmwareFormatAnalyzer` accept config params for threshold customization.
- **Homebrew Formula**: Updated to v1.0.0 with revised SHA-256 checksum and dependency list.

### Technical Notes
- SIEM connectors use httpx for HTTP communication with optional SSL certificate verification
- All SIEM connectors implement exponential backoff retry with configurable base delay, max delay, and max retries
- Splunk HEC connector validates token format and checks `/services/collector/health/1.0` for connectivity
- Elasticsearch connector handles HTTP 429 responses with `Retry-After` header parsing
- Sentinel connector computes HMAC-SHA256 signatures per Azure Monitor specification using workspace key
- Webhook delivery uses HMAC-SHA256 with `X-HADES-Signature` header; signature computed over raw JSON payload
- Community tier requires no PostgreSQL or Redis — SQLite backend with in-memory rate limiting
- Dashboard feature gating fetches `/api/v1/auth/license` on initialization with graceful fallback to free-tier visibility
- SDK retry logic uses full jitter: `min(max_delay, base_delay * 2^attempt) * random(0, 1)`
- New optional dependency: httpx (already present for vulnerability module, now also used by SIEM connectors)
- GPS forensics module has zero external dependencies (stdlib math/re only); DBSCAN uses Haversine distance matrix instead of sklearn
- Quarantine manager uses per-call sqlite3.connect() for thread safety, matching the playbook engine pattern
- ~77 new tests across SIEM connectors, webhooks, SDK extensions, community edition, MITRE coverage, and distribution validation. Additional 67 tests for GPS forensics and quarantine. Total test suite: ~2,452 tests.

## [0.9.0] - 2026-03-02

### Removed
- Vulnerability management module (extracted to standalone darkhorse-patchwatch).
- Log analysis module (extracted to standalone darkhorse-proxyhound).
- Shadow IT detection (extracted with log analysis).
- Vulnerability-specific playbooks (critical vulnerability response, Shadow IT alert, data exfiltration response).
- Vulnerability-specific trigger conditions (`vuln_severity_gte`, `risk_score_gte`).
- MITRE ATT&CK mappings for non-metadata techniques (T1048, T1199).
- 21 optional dependency groups collapsed to 4 (`enterprise`, `integrations`, `docs`, `dev`).

### Changed
- CLI restructured into subcommands: `scan`, `monitor`, `case`, `serve`, `benchmark`, `plugin`, `intel`, `rules`, `admin`.
- YARA, scikit-learn, FastAPI, and watchdog are now required dependencies (not optional).
- Internal refocus milestone prior to v1.0.0 production release.
- Product positioning: metadata forensics engine (not SOC platform).
- License tiers simplified to match focused feature set.

### Fixed
- ML ensemble feature extraction rebuilt with 25 statistical features (replaces regex pattern counting).
- Playbook engine hardened: priority ordering, exponential backoff retry, per-action status tracking.
- Playbook failures now logged at WARNING/ERROR level (not silently swallowed).
- Playbook concurrency limiting with configurable `max_concurrent_playbooks`.

### Added
- New CLI entry point: `hades` with subcommand dispatch.
- Shannon entropy-based metadata profiling in ML detector.
- Playbook priority field for execution ordering.
- Playbook execution status: completed/partial/failed with per-action counters.
- Backward-compatible SQLite schema migration for new playbook fields.

## [0.8.0] - 2026-02-28

### Added
- **Vulnerability Management Module** (`core/vulnerability/`): Full-lifecycle vulnerability management with NVD API v2.0 integration, asset inventory, CVE matching, and CVSS v3.1 scoring. `VulnerabilityManager` coordinates asset discovery, NVD lookups with local caching, and risk-ranked vulnerability reports. `AssetInventory` tracks software assets with CPE identifiers, versions, and platform metadata. `NVDClient` queries the NVD API v2.0 with rate limiting, response caching, and offline fallback. `CVEMatcher` correlates asset CPEs against CVE records with version range evaluation. CVSS v3.1 vector parsing and base score calculation. Dashboard view at `/dashboard/#vulnerability` with asset table, CVE list, severity breakdown, and remediation tracking.
- **Standalone Vulnerability Agent** (`scripts/hades_vuln_agent.py`): Lightweight cross-platform agent script for endpoint software discovery. Detects installed software on Windows (registry + PowerShell), macOS (system_profiler + Homebrew), and Linux (dpkg/rpm/pacman + snap/flatpak). Runs as a scheduled service or one-shot scan. Reports discovered software inventory to the HADES API server via the agent protocol.
- **Agent Protocol**: Endpoint registration (`POST /api/v1/vulnerability/agents/register`), heartbeat monitoring (`POST /api/v1/vulnerability/agents/{agent_id}/heartbeat`), and scan result ingestion (`POST /api/v1/vulnerability/agents/{agent_id}/results`). Agents self-register with hostname, platform, and IP metadata. Heartbeat tracking with configurable staleness threshold for agent health monitoring.
- **Log Analysis Module** (`core/log_analysis/`): Proxy log parsing engine supporting Squid access logs, Zscaler web logs, and generic CSV formats. `ProxyLogParser` with auto-format detection and field normalization. `ShadowITDetector` with 72+ cloud service mappings across 7 categories (file sharing, messaging, email, development, social media, cloud infrastructure, AI/ML tools) for unauthorized SaaS discovery. `ExfiltrationDetector` with 5 detection patterns: large upload volume (configurable threshold), repeated uploads to single destination, off-hours data transfer (configurable business hours), uploads to uncategorized domains, and high-frequency small transfers. Dashboard view at `/dashboard/#log-analysis` with log upload, Shadow IT summary, exfiltration alerts, and timeline visualization.
- **Cross-Module Correlation Endpoint** (`GET /api/v1/correlation/asset/{asset_id}`): Combines vulnerability data (open CVEs, CVSS scores) with metadata scan results (threat scores, findings) for a unified risk profile per asset. Returns correlated risk assessment with recommended actions.
- **Vulnerability Dashboard** (`core/dashboard/js/vulnerability.js`): Web dashboard view with asset inventory table (sortable by risk), CVE detail panels, severity distribution chart, agent status indicators, and scan trigger controls.
- **Log Analysis Dashboard** (`core/dashboard/js/log_analysis.js`): Web dashboard view with log file upload, parsing status, Shadow IT category breakdown, exfiltration alert timeline, and export controls.
- **3 New Automated Playbooks**: `builtin-critical-vuln-response` (triggers on CVSS >= 9.0, creates case + SIEM forward + Slack/Teams notify), `builtin-shadow-it-alert` (triggers on Shadow IT detection, creates audit log + Slack notify), `builtin-data-exfiltration-response` (triggers on exfiltration detection, creates case + SIEM forward + evidence export + Slack/Teams notify).
- **2 New Playbook Trigger Conditions**: `vuln_severity_gte` (fires when CVSS base score meets or exceeds threshold) and `risk_score_gte` (fires when generic risk score meets or exceeds threshold).
- **MITRE ATT&CK Technique Additions**: T1048 (Exfiltration Over Alternative Protocol) mapped to data exfiltration findings, T1199 (Trusted Relationship) mapped to Shadow IT trust boundary findings.
- **MITRE Finding-to-Technique Mappings**: Expanded with vulnerability and log analysis finding categories for CVE exploitation indicators and unauthorized data transfer patterns.
- **Vulnerability API Endpoints**: `POST /api/v1/vulnerability/scan` (trigger NVD scan for asset), `GET /api/v1/vulnerability/assets` (list assets), `POST /api/v1/vulnerability/assets` (register asset), `GET /api/v1/vulnerability/assets/{asset_id}/cves` (list CVEs for asset), `GET /api/v1/vulnerability/summary` (severity summary), `POST /api/v1/vulnerability/agents/register`, `POST /api/v1/vulnerability/agents/{agent_id}/heartbeat`, `POST /api/v1/vulnerability/agents/{agent_id}/results`.
- **Log Analysis API Endpoints**: `POST /api/v1/log-analysis/upload` (upload and parse log file), `GET /api/v1/log-analysis/shadow-it` (Shadow IT report), `GET /api/v1/log-analysis/exfiltration` (exfiltration alerts), `GET /api/v1/log-analysis/summary` (analysis summary), `GET /api/v1/log-analysis/export` (CSV export).
- **Vulnerability CLI Flags**: `--vuln-scan` (scan assets for vulnerabilities), `--vuln-add-asset` (register software asset), `--vuln-list-assets` (list asset inventory), `--vuln-summary` (severity summary), `--vuln-agent-status` (show agent health).
- **Log Analysis CLI Flags**: `--log-analyze` (parse and analyze proxy log file), `--log-format` (specify log format: squid, zscaler, csv), `--shadow-it-report` (generate Shadow IT report), `--exfil-detect` (run exfiltration detection), `--log-export` (export analysis results).
- **License Tier Gating**: Vulnerability management gated at Professional tier (`vulnerability_management` feature). Log analysis gated at Enterprise tier (`log_analysis` feature).
- **Vulnerability Config Section**: `vulnerability` section in `cli/hades_config.json` with NVD API key, cache TTL, scan interval, agent heartbeat timeout, and CVSS threshold settings.
- **Log Analysis Config Section**: `log_analysis` section in `cli/hades_config.json` with supported formats, Shadow IT service mappings path, exfiltration thresholds (upload size, frequency, business hours), and retention settings.
- **Vulnerability Test Suite** (`core/test_vulnerability_manager.py`, `core/test_nvd_client.py`, `core/test_asset_inventory.py`, `core/test_vuln_api.py`): 34 tests covering asset CRUD, NVD API mocking, CVE matching, CVSS scoring, agent protocol, and API endpoints.
- **Log Analysis Test Suite** (`core/test_log_parser.py`, `core/test_shadow_it.py`, `core/test_exfiltration.py`, `core/test_log_api.py`): 33 tests covering Squid/Zscaler/CSV parsing, Shadow IT detection, exfiltration pattern matching, and API endpoints.

### Changed
- **Playbook Engine**: Expanded from 10 to 13 built-in playbooks with `builtin-critical-vuln-response`, `builtin-shadow-it-alert`, and `builtin-data-exfiltration-response`. Added `vuln_severity_gte` and `risk_score_gte` trigger condition evaluators.
- **MITRE ATT&CK Mapping**: Finding-to-technique mapping expanded with vulnerability exploitation and log analysis categories. Added T1048 and T1199 technique definitions.

### Added
- **Firmware Detection Tier** (`core/firmware_analyzer.py`): Comprehensive static analysis engine for firmware and boot-level threats. Detects UEFI capsules (EFI Standard, Intel, Windows UX) with recursive volume traversal, SPI flash images (Intel Flash Descriptor with region parsing and permission checks, AMD PSP/BHD), vendor BIOS updates (AMI, Insyde with nested capsule detection, Phoenix, Dell with HDR version extraction, HP), destructive disk payloads (MBR/GPT overwrite, GPT header destruction, PowerShell disk cmdlets, CMD diskpart patterns, 9 known wiper families including Shamoon, WhisperGate, HermeticWiper, NotPetya, CaddyWiper, AcidRain), firmware flashing tools (flashrom, Intel FPT, CH341A, RWEverything, fwupd, AMI AFU with ELF/PE binary detection), bootloader artifacts (boot sectors, BlackLotus UEFI bootkit indicators with CVE-2022-21894 detection), firmware polyglots (firmware signatures embedded in JPEG/PNG/PDF/ZIP containers with entropy scanning), BadUSB/DuckyScript payloads (DuckyScript v1+v3, Flipper Zero SubGhz/IR file formats, P4wnP1, O.MG with exfiltration detection), and firmware entropy anomaly detection (Shannon entropy per 4096-byte chunks with severity scaling). Integrated into EnhancedDetectionEngine as an automatic detection pass. Optional uefi-firmware-parser and binwalk deps for deep analysis.
- **FirmwareFinding Subclass**: Extended `FormatFinding` with `firmware_type`, `vendor`, and `risk_level` fields. Severity-to-risk mapping (Low/Medium/High/Critical) via `_severity_to_risk()` static method.
- **Known-Good Firmware Allowlist**: `core/allowlist/known_firmware_hashes.txt` with SHA-256 hash-based allowlisting to skip analysis of trusted firmware images. Configurable via `--firmware-allowlist` CLI flag or `firmware.allowlist_path` config.
- **ML Firmware Anomaly Detector**: `FirmwareFeatureExtractor` (12 features: file size, entropy metrics, marker counts, printable ratio) and `FirmwareAnomalyDetector` (IsolationForest-based, 0-10 scoring via sigmoid transform, save/load persistence). Optional, requires scikit-learn. Enabled via `--firmware-ml` CLI flag.
- **Firmware YARA Rules** (`rules/firmware_threats.yar`): 16 YARA rules for firmware threat detection — UEFI capsules, SPI flash images, AMI BIOS updates, firmware flashing tools, MBR overwrite patterns, wiper malware signatures, shadow copy deletion, bootloader binaries, BlackLotus indicators, BadUSB payloads, firmware polyglots, plus 5 new rules: recent wipers (AcidRain MTD/Industroyer2), LogoFAIL indicators (CVE-2023-40238), PKfail supply chain compromise, advanced BadUSB (DuckyScript v3/Flipper SubGhz/IR/O.MG), firmware entropy anomalies.
- **Firmware MITRE ATT&CK Mappings**: 11 techniques in `core/mitre_mapping.py` — T1542, T1542.001, T1542.003, T1495, T1561, T1561.001, T1561.002, T1200, T1553.006 (Code Signing Policy Modification), T1059.003 (Windows Command Shell), T1195 (Supply Chain Compromise). Finding-to-technique mappings for 16 firmware finding types.
- **Firmware CLI Flags**: `--firmware-scan` (verbose output), `--firmware-allowlist PATH` (known-good hashes), `--firmware-ml` (ML anomaly detection).
- **Firmware Config Section**: `firmware` section in `cli/hades_config.json` with enable, allowlist path, entropy threshold/chunk size, ML anomaly subsection.
- **Firmware Test Suite** (`core/test_firmware_analyzer.py`): 55 tests (29 original + 26 new) covering FirmwareFinding subclass, allowlist infrastructure, enhanced wipers (PowerShell/diskpart/GPT destruction), enhanced BadUSB (DuckyScript v3/Flipper SubGhz/O.MG exfil), firmware entropy analysis, enhanced SPI flash region parsing, enhanced BIOS update vendor parsing, extended MITRE mappings, feature extraction, and ML anomaly detection.
- **Scan Analytics Integration**: Fixed and validated `core/scan_analytics.py` (ScanAnalyticsEngine with statistics, trends, anomalies, executive summary, CSV export), `core/analytics_routes.py` (API endpoints at `/api/v1/analytics/`), license-gated at Professional tier, CLI flags `--analytics`, `--analytics-period`, `--analytics-export`.
- **Playbook Engine** (`core/playbook_engine.py`): Event-driven response automation that evaluates trigger conditions against incoming events and executes predefined response action sequences. Supports trigger conditions: `threat_score_gte`, `threat_score_lte`, `has_yara_match`, `has_finding_type`, `has_finding_category`, `event_type_eq`, `sender_domain_in`, `file_extension_in`. Supports action types: `create_case`, `siem_forward`, `slack_notify`, `teams_notify`, `webhook`, `audit_log`, `evidence_export`, `quarantine`. SQLite-backed persistence for definitions and execution history. Background thread execution with graceful degradation when services are unavailable. Optional `on_event` callback for real-time WebSocket broadcast of execution results.
- **Playbook API Endpoints** (`core/playbook_routes.py`): Full CRUD, enable/disable, manual trigger, and execution history at `/api/v1/playbooks`.
- **10 Built-in Playbooks**: Phishing Attachment Triage, BEC Detection & Response, Malware Analysis Workflow, Credential Leak Response, Suspicious File Activity, DLP Exfiltration Response, DLP PII Exposure, Ransomware Response, Insider Threat Detection, Campaign Escalation.
- **Alert Dashboard** (`core/dashboard/js/alerts.js`): Real-time playbook execution visualization with summary stats, status filters, playbook dropdown, execution timeline feed, and WebSocket live updates.
- **Playbook Dashboard** (`core/dashboard/js/playbooks.js`): Web dashboard view with playbook table (enable/disable toggle, manual trigger) and execution history.
- **Playbook CLI Flags**: `--list-playbooks`, `--run-playbook`, `--playbook-status`, `--enable-playbook`, `--disable-playbook`.
- **Scan Result Hook**: `_scan_single_file()` in `hades_api.py` now fires matching playbooks after SIEM auto-forward.
- **Monitor Alert Hook**: `FileMonitor._emit_alert()` now dispatches monitor alerts to the playbook engine.
- **Email Scan Hook**: `EmailGateway.scan_email()` fires `email_scan` playbook events with message metadata.
- **Campaign Detection Hook**: `/api/v1/behavioral/campaigns` endpoint fires `campaign_detected` playbook events for detected campaigns.
- **Enhanced Webhook Action**: Template variable resolution (`{{event_type}}`, `{{file_name}}`, `{{threat_score}}`), custom HTTP headers, custom payloads, separate exception handling for connection errors and timeouts.
- **Config Section**: `playbooks` section in `cli/hades_config.json` with enable/disable, database path, default seeding, concurrent execution limits, and webhook URLs.
- **Test Suite**: `core/test_playbook_engine.py` with 65 tests covering data model, store CRUD, trigger matching, execution dispatch, integration, default playbooks, API routes, webhook actions, new trigger conditions, on_event callback, and new playbook triggering.

## [0.7.1] - 2026-02-21

### Added
- **Prometheus Metrics Exporter** (`core/metrics.py`): Comprehensive metrics collection with counters (scans, findings, cache, workers, API requests, auth), histograms (scan duration, stage timing, API latency, file size), and gauges (active scans, queue depth, workers, cache size, uptime). Zero-overhead when disabled. `prometheus_client` optional dependency.
- **Prometheus Middleware** (`core/metrics.py`): FastAPI middleware for automatic API request metrics (count, duration, status code). Adds `X-Request-Duration` response header.
- **Metrics API Endpoints** (`core/metrics_routes.py`): `/metrics` (Prometheus scrape), `/api/v1/metrics/json` (JSON summary), `/api/v1/metrics/health` (enhanced health check), `/api/v1/metrics/alerts` (active alert conditions).
- **Grafana Dashboards** (`grafana/dashboards/`): Four pre-built dashboards -- Overview (throughput, latency, threat distribution), Pipeline (per-stage timing, bottlenecks), Workers (pool status, queue depth, task rates), Infrastructure (API metrics, cache, auth).
- **Grafana Provisioning** (`grafana/provisioning/`): Auto-configured Prometheus datasource and dashboard discovery.
- **Grafana Alerting** (`grafana/alerting/`): Alert rules for error rate, queue backlog, worker availability, API errors, cache degradation, latency spikes.
- **Kubernetes Helm Chart** (`kubernetes/helm/hades/`): Production-grade Helm chart with API and worker deployments, HPA (CPU/memory for API, custom queue-depth metric for workers), Ingress with WebSocket support, ServiceMonitor for Prometheus, Grafana dashboard ConfigMap, PVCs for rules and config, topology spread constraints.
- **Kustomize Overlays** (`kubernetes/kustomize/`): Base configuration with dev overlay (single replica, SQLite) and production overlay (multi-replica, PostgreSQL, Redis, HPA, TLS ingress).
- **Docker Observability Stack** (`docker/docker-compose.observability.yml`): Prometheus, Grafana, and Alertmanager with pre-configured scraping, dashboards, and alerting.
- **Docker Full Stack** (`docker/docker-compose.full-stack.yml`): Complete deployment combining PostgreSQL, Redis, API server, workers, Prometheus, Grafana, and Alertmanager.
- **Prometheus Alerting Rules** (`docker/prometheus/alerts.yml`): Seven alert rules covering error rates, queue backlogs, worker availability, API errors, cache health, latency, and service availability.
- **Observability Documentation** (`docs/observability_guide.md`): Prometheus setup, Grafana dashboards, alerting configuration, key metrics reference, PromQL examples, troubleshooting.
- **Kubernetes Documentation** (`docs/kubernetes_guide.md`): Helm installation, values reference, secrets management, HPA configuration, ServiceMonitor, upgrading, Kustomize alternative, troubleshooting.
- **CLI Flags**: `--metrics` / `--no-metrics`, `--metrics-port`, `--health-check`.
- **Observability Optional Group**: `pip install "hades-scanner[observability]"` installs prometheus-client.
- **Config Section**: `metrics` section in `cli/hades_config.json`.

### Changed
- **Version**: All components updated to 0.7.1.
- `pyproject.toml`: Version 0.7.1. New `observability` optional dependency group (prometheus-client). `full` group updated to include observability.
- `cli/hades_config.json`: Added `metrics` config section.

## [0.7.0] - 2026-02-21

### Added
- **Async Scan Pipeline** (`core/async_scanner.py`): Non-blocking scan submission with Redis-backed task queue. API returns task ID immediately; workers process scans in the background. Enable with `--async` CLI flag or `HADES_ASYNC_PIPELINE=true` environment variable.
- **Worker Pool** (`core/worker_pool.py`): Configurable concurrent scan workers with `--workers N` flag. Distributed worker mode (`--worker --distributed`) for dedicated scan processing containers.
- **File Ingestion Engine** (`core/file_ingestion.py`): Recursive directory ingestion with glob filtering, deduplication, and priority queuing. Integrates with both sync and async pipelines.
- **Scan Result Cache** (`core/scan_cache.py`): SHA-256 content-addressed LRU cache with configurable TTL. Redis-backed when available, in-memory fallback. Enable with `--scan-cache` flag. Configurable via `--cache-ttl` and `--cache-max-size`.
- **Performance Profiler** (`core/profiler.py`): Per-stage timing instrumentation (metadata extraction, YARA matching, heuristic analysis, ML detection). Enable with `--profile` flag. Generates bottleneck reports.
- **Enhanced Benchmarks** (`core/benchmarks.py`): Extended benchmark suite with async pipeline throughput, cache hit/miss latency, worker pool scaling, and memory profiling metrics.
- **Docker Scaling Infrastructure**: `docker-compose.scale.yml` (PostgreSQL + Redis + scalable API + worker replicas), `docker-compose.ha.yml` (nginx reverse proxy with health-check failover + multiple API instances + dedicated workers + Redis coordination), `nginx.conf` (upstream balancing, WebSocket upgrade, rate limiting, static caching).
- **Performance Documentation**: `docs/performance_guide.md` (tuning, deployment patterns, profiling, monitoring, troubleshooting), `docs/scaling_guide.md` (horizontal scaling, database scaling, load balancing, Kubernetes concepts).
- **Performance CLI Flags**: `--async`, `--workers`, `--worker`, `--distributed`, `--scan-cache`, `--cache-ttl`, `--cache-max-size`, `--profile`.
- **Performance Config Section**: `distributed` section in `cli/hades_config.json` (Redis queue, heartbeat, timeouts, retries).
- **Performance Optional Group**: `pip install "hades-scanner[performance]"` installs py7zr and uvloop (Linux/macOS).
- **Docker Worker Mode**: `HADES_MODE=worker` environment variable runs containers as background scan workers instead of API servers.
- **Test Suites**: `tests/test_docker_compose.py` (Docker Compose YAML validation, service dependencies, env var defaults), `tests/test_performance_integration.py` (config loading, CLI flag parsing, backward compatibility, API endpoint registration).

### Changed
- **Version**: All components updated to 0.7.0.
- `docker/hades-full.dockerfile`: Added `HADES_ASYNC_PIPELINE`, `HADES_CACHE_ENABLED`, `HADES_MODE` env vars. Optimized build layers (combined RUN commands, cleaned pip cache). Updated version label to 0.7.0. Added performance extras to pip install.
- `docker/entrypoint.sh`: Added worker mode support (`HADES_MODE=worker`). Passes `--async` and `--scan-cache` flags when environment variables are set. Added startup logging for new configuration.
- `pyproject.toml`: Version 0.7.0. New `performance` optional dependency group (py7zr, uvloop). `full` group updated to include performance.
- `cli/hades_config.json`: Added `distributed` config section.

## [0.6.0] - 2026-02-19

### Added
- **Role-Based Access Control** (`core/auth/rbac.py`): User management with admin, analyst, and viewer roles. Permission matrix for fine-grained API access control. bcrypt password hashing with SHA-256 fallback. API key authentication and rotation. FastAPI dependencies (`get_current_user`, `require_role`) for endpoint protection.
- **Single Sign-On** (`core/auth/sso.py`): OIDC provider with JWT validation, authorization URL generation, and configurable role mapping. SAML 2.0 provider with AuthnRequest generation, response validation, and attribute extraction. JIT (Just-In-Time) user provisioning for SSO identities.
- **License Key System** (`core/auth/license.py`): HMAC-SHA256 signed license keys with three tiers (free, professional, enterprise). Feature gating via `require_license` FastAPI dependency. License generation, validation, and expiry checking.
- **Auth API Endpoints** (`core/auth/auth_routes.py`): Login (username/password), user CRUD (admin only), API key rotation, license info, SSO authorization and callback endpoints at `/api/v1/auth/`.
- **Database Abstraction Layer** (`core/storage/database.py`): Pluggable backend interface with SQLite (WAL mode, thread-safe) and PostgreSQL (connection pooling via psycopg2) implementations. Factory function with environment variable override support.
- **Enterprise Scan Store** (`core/storage/scan_store.py`): Refactored scan result storage backed by pluggable DatabaseBackend. Multi-tenant isolation with tenant_id column filtering.
- **Audit Store** (`core/storage/audit_store.py`): Audit log persistence backed by DatabaseBackend.
- **Case Store** (`core/storage/case_store.py`): Case management persistence backed by DatabaseBackend.
- **Redis State Manager** (`core/storage/redis_state.py`): Sliding-window rate limiting, session management, pub/sub, distributed locks, and caching backed by Redis. In-memory fallback when Redis is unavailable.
- **Encryption at Rest** (`core/storage/encryption.py`): AES-256-GCM field-level encryption with EncryptionManager. EncryptedDatabaseBackend wrapper for transparent encrypt-on-write, decrypt-on-read.
- **Multi-Tenant Isolation** (`core/storage/tenant.py`): Tenant CRUD operations, TenantContext for request-scoped isolation, API key to tenant mapping.
- **Storage Admin Routes** (`core/storage/storage_routes.py`): Database status, migration, Redis status, encryption status, and tenant CRUD endpoints at `/api/v1/admin/`.
- **Enterprise CLI Flags**: `--create-admin`, `--create-user`, `--user-role`, `--list-users`, `--license-info`, `--license-key`, `--db-backend`, `--db-dsn`, `--redis-url`, `--encrypt`, `--generate-encryption-key`, `--create-tenant`, `--list-tenants`, `--migrate-db`.
- **Enterprise Config Sections**: `database`, `redis`, `encryption`, `auth`, and `tenants` sections added to `cli/hades_config.json`.
- **Enterprise Docker Support**: PostgreSQL 16 and Redis 7 services in `docker-compose.yml` (enterprise profile). New environment variables: `HADES_DB_BACKEND`, `HADES_DB_DSN`, `HADES_REDIS_URL`, `HADES_ENCRYPTION_KEY`, `HADES_LICENSE_KEY`.
- **Enterprise Extras**: `pip install "hades-scanner[enterprise]"` installs bcrypt, PyJWT, cryptography, psycopg2-binary, redis.
- **Enterprise Deployment Guide** (`docs/enterprise_deployment_guide.md`): RBAC setup, SSO configuration (OIDC/SAML), PostgreSQL and Redis setup, encryption, multi-tenant setup, license management, Docker Compose production deployment, migration guide.
- **Test Suites**: `core/test_auth.py` (RBAC, SSO, license tests), `core/test_storage.py` (database, scan store, audit store, case store, Redis, encryption, tenant tests), `core/test_enterprise_integration.py` (end-to-end integration tests) -- 183 new tests, 1050+ total passing.
- **ML Ensemble Detector** (`core/ml_ensemble.py`): Multi-model anomaly detection combining Isolation Forest, Random Forest, and optional XGBoost with weighted voting. Extended feature extractor (25 features: 15 base + 10 security-focused). Labeled data manager with SQLite storage and CSV import/export. Auto-retrainer with contamination tuning.
- **Behavioral Analysis** (`core/behavioral_analysis.py`): Campaign detection via shared IOC correlation (union-find), metadata pattern similarity (Jaccard threshold), and temporal clustering. File relationship graph with multi-hop traversal for threat campaign profiling.
- **MITRE ATT&CK Mapping** (`core/mitre_mapping.py`): Maps HADES detection findings to MITRE ATT&CK techniques and tactics. Covers script injection, PHP backdoors, command injection, reverse shells, Base64 encoding, steganography, polyglot files, macro execution, and template injection.
- **Threat Intelligence Scheduler** (`core/threat_intel_scheduler.py`): Automated periodic ingestion of threat intelligence feeds (STIX/TAXII, MISP, CSV, custom). Configurable per-feed intervals with manual trigger support.
- **YARA Rule Builder** (`core/yara_rule_builder.py`): Programmatic YARA rule generation from templates. Built-in templates for metadata injection, steganography, polyglot files, and encoded payloads.
- **MalwareBazaar Validation** (`scripts/run_malware_validation.py`, `tests/test_malwarebazaar_validation.py`): Standalone validation script scanning mocked or live MalwareBazaar samples with detection matrix. Test suite asserting TPR >= 90% and FPR <= 10%.
- **Detection Depth Guide** (`docs/detection_depth_guide.md`): Documentation covering ML ensemble architecture, behavioral analysis, MITRE ATT&CK integration, threat intel scheduling, and YARA rule building.
- **Phase 2 API Endpoints**: `/api/v1/mitre/matrix`, `/api/v1/mitre/lookup`, `/api/v1/threat-intel/scheduler`, `/api/v1/rules/templates`, `/api/v1/rules/build`, `/api/v1/ml/ensemble/status`, `/api/v1/behavioral/campaigns`.
- **Phase 2 CLI Flags**: `--ml-ensemble`, `--behavioral`, `--mitre-map`, `--schedule-feeds`, `--build-rule`, `--list-templates`, `--malware-validate`.
- **Phase 2 Config Sections**: `ml_ensemble`, `behavioral_analysis`, `threat_intel_scheduler`, `yara_rule_builder`, `mitre_mapping` added to `cli/hades_config.json`.
- **Phase 2 Test Suites**: `core/test_ml_ensemble.py` (48 tests), `core/test_behavioral_analysis.py` (28 tests), `tests/test_malwarebazaar_validation.py` (12 tests).

### Added (Phase 3 -- Community & Adoption)
- **MkDocs Documentation Site**: Material theme, 25+ pages covering installation, CLI reference, API reference, detection guides, enterprise deployment, architecture, contributing.
- **Plugin Marketplace** (`plugins/registry.py`): Registry with install/uninstall/update, SHA-256 verification, remote registry support.
- **Example Plugins**: Steganography detector (LSB analysis, appended data, tool signatures) and PII scanner (email, phone, SSN, credit card with Luhn validation).
- **Plugin Authoring Guide** (`docs/plugin_authoring_guide.md`): Step-by-step tutorial.
- **YARA Rule Contribution Workflow**: Naming conventions, required metadata, CI validation.
- **Enhanced YARA Rule Validator** (`scripts/validate_rules_ci.py`): Meta fields, severity, MITRE mapping, duplicates, naming.
- **GitHub Actions CI/CD**: Test suite (Python 3.9-3.12 matrix), YARA rule validation.
- **GitHub Templates**: PR and issue templates.
- **Interactive Demo Environment**: Rate limiting, sample files, landing page, Docker image and Compose file.
- **CLI Flags**: `--plugin-search`, `--plugin-install`, `--plugin-uninstall`, `--plugin-check-updates`, `--validate-rules-strict`, `--demo-mode`, `--generate-demo-samples`.
- **Plugin Marketplace Endpoints**: API at `/api/v1/plugins/marketplace/`.

### Changed
- **Version**: All components updated to 0.6.0.
- `core/hades_api.py`: Enterprise auth and storage modules optionally imported and initialized in `create_app()`. Auth routes registered at `/api/v1/auth/`, storage admin routes at `/api/v1/admin/`. Legacy flat API key auth preserved as fallback when RBAC is disabled. Phase 2 modules (MITRE mapping, threat intel scheduler, YARA builder, ML ensemble, behavioral analysis) optionally imported with route registration for Phase 2 endpoints.
- `core/hades_enhanced_cli.py`: Enterprise argument group added with user management, licensing, encryption, tenant, and migration commands. Detection Depth argument group added with ML ensemble, behavioral analysis, MITRE mapping, threat intel scheduling, YARA rule building, and MalwareBazaar validation flags.
- `docker/hades-full.dockerfile`: Enterprise dependencies added to pip install. New environment variables for database, Redis, encryption, and license.
- `docker/docker-compose.yml`: PostgreSQL and Redis services added under enterprise profile. New environment variable passthrough.
- `pyproject.toml`: Version 0.6.0. New `enterprise` optional dependency group. `full` group updated to include enterprise.

## [0.5.0] - 2026-02-19

### Added
- **FastAPI Migration**: REST API server (`core/hades_api.py`) fully migrated from Flask to FastAPI with async endpoints, Pydantic request/response models, automatic OpenAPI documentation at `/docs`, and WebSocket support via `WebSocketManager`.
- **Evidence Chain & Case Management** (`core/evidence_chain.py`): SHA-256 hash-chained append-only audit log, investigation case CRUD with scan linking and analyst notes, self-verifying evidence export packages (ZIP with manifest, hashes, HMAC-SHA256 signature, verify script).
- **Evidence API Endpoints** (`core/evidence_api_endpoints.py`): Async endpoint functions wired into FastAPI via APIRouter at `/api/v1/evidence/` prefix for audit log query, chain verification, case CRUD, scan linking, and evidence export.
- **Web Dashboard SPA** (`core/dashboard/`): Fully self-contained vanilla HTML+CSS+JS single-page application with hash-based routing, dark theme (#1a1a2e), drag-and-drop scan upload, monitor dashboard, case management, audit trail viewer, and system status. Served by FastAPI at `/dashboard/` with no external CDN dependencies.
- **Security Middleware** (`core/security_middleware.py`): Sliding window rate limiting, file upload magic byte validation, path traversal protection, HTML output sanitization, security response headers. Dual Flask/FastAPI support with `Depends()` callables for the main API.
- **Deep Format Analyzer** (`core/deep_format_analyzer.py`): Native parsing for PDF (JavaScript/OpenAction/embedded files/SubmitForm), Office (macros/OLE/DDE/template injection), polyglot detection (multi-format files), SVG (script/XXE/event handlers). Integrated into `EnhancedDetectionEngine.analyze()` as additional detection pass.
- **Performance Benchmarks** (`core/benchmarks.py`): Cross-platform benchmark suite measuring single-file latency, batch throughput, memory usage, ML extraction, cache performance, and API response times. Runnable standalone or via CLI `--benchmark`/`--benchmark-quick`.
- **YARA Rule Repository**: New `rules/polyglot_detection.yar` (6 rules for JPEG+ZIP, JPEG+RAR, PDF+JS, PNG+HTML, GIF+JS, generic dual-header), `rules/suspicious_metadata.yar` (6 rules for Base64 payloads, URLs in EXIF, email PII, encoded executables, shell commands, oversized fields). Rule validation script at `scripts/validate_rules.py`.
- **YARA Rules Documentation** (`rules/README.md`): Rule categories, naming convention (`HADES_<CATEGORY>_<SPECIFIC>`), severity mapping, writing custom rules guide, contributing guidelines.
- **Network Integrations** (`core/integrations/network_connector.py`): Zeek and Suricata file extraction connectors with watchdog-based monitoring, network context enrichment (src/dst IP, protocol, connection UID), auto-scanning of extracted files.
- **Email Gateway** (`core/integrations/email_gateway.py`): SMTP gateway for email attachment scanning with RFC 5322 parsing, quarantine management (list/release/delete), configurable threat thresholds.
- **Cloud Storage Scanning** (`core/integrations/cloud_storage.py`): AWS S3, Google Cloud Storage, and Azure Blob connectors with concurrent bucket scanning, file type/size filtering, object tagging, and scheduled scans.
- **CI/CD Integration** (`core/integrations/cicd_scanner.py`): Pipeline file scanner with pass/fail thresholds, output formats (table, JSON, SARIF 2.1.0, GitHub PR comment, GitLab MR note). GitHub Actions workflow at `.github/workflows/hades-scan.yml`.
- **Chat Bots** (`core/integrations/chat_bot.py`): Slack bot with slash commands and Block Kit UI, Microsoft Teams bot with Adaptive Cards, file upload scanning, and alert channel routing.
- **Integration API Routes**: `network_email_routes.py`, `cloud_routes.py`, `cicd_chat_routes.py` wired into FastAPI at `/api/v1/integrations/` prefix.
- **Package Distribution**: `pyproject.toml` and `MANIFEST.in` for `pip install hades-scanner` with optional dependency groups (`api`, `yara`, `ml`, `docker`, `cloud`, `email`, `chat`, `crypto`, `formats`, `full`, `dev`). Console entry points: `hades`, `hades-enhanced`, `hades-server`, `hades-benchmark`.
- **Docker Production Image** (`docker/hades-full.dockerfile`): Multi-stage build with python:3.12-slim, non-root user (UID 1000), YARA rules and dashboard bundled, health checks, environment variable configuration (`HADES_API_KEY`, `HADES_PORT`, `HADES_WORKERS`, `HADES_LOG_LEVEL`), OCI labels.
- **Docker Compose** (`docker/docker-compose.yml`): Full stack deployment with named volumes, resource limits, health checks, and environment variable passthrough.
- **Homebrew Formula** (`Formula/hades-scanner.rb`): macOS installation via Homebrew with python@3.12 and exiftool dependencies.
- **Build & Release Scripts**: `scripts/build_release.py` (automated release verification), `scripts/docker_build.sh` (Docker image build/tag), `scripts/test_docker.sh` (Docker integration tests), `scripts/validate_rules.py` (YARA rule validation).
- CLI flags: `--case-create`, `--case-id`, `--audit-verify`, `--export-case`, `--benchmark`, `--benchmark-quick`, `--zeek-watch`, `--suricata-watch`, `--email-gateway`, `--smtp-port`, `--s3-scan`, `--gcs-scan`, `--azure-scan`, `--cloud-prefix`, `--cloud-tag-results`, `--ci-scan`, `--ci-threshold`, `--ci-format`, `--slack-bot`, `--teams-bot`.
- **Test Suites**: `core/test_evidence_chain.py`, `core/test_dashboard.py`, `core/test_security.py`, `core/test_deep_format.py`, `core/test_integrations_network.py`, `core/test_integrations_email.py`, `core/test_integrations_cloud.py`, `core/test_integrations_cicd.py`, `core/test_integrations_chat.py`, `tests/test_full_pipeline.py` -- total 760+ passing tests.

### Changed
- **Flask to FastAPI**: `core/hades_api.py` now uses FastAPI with `create_app()` factory returning a FastAPI instance. Tests use `starlette.testclient.TestClient`. FastAPI returns 422 for validation errors. The `--serve` flag uses `uvicorn.run()`.
- **Consolidated `enterprise_server.py`**: `WebSocketManager` and Pydantic models moved into `hades_api.py`. `core/enterprise_server.py` removed.
- **YARA Rule Reorganization**: Rules organized into dedicated files by category with standardized naming convention and metadata requirements.
- **Version Standardization**: All modules aligned to 0.5.0 versioning scheme.
- `requirements.txt` updated with fastapi, uvicorn, python-multipart, websockets, httpx, pikepdf, olefile.

### Fixed
- Fixed YARA compilation error in `rules/advanced_threats.yar` line 244: replaced unsupported non-capturing group `(?:...)` with standard group `(...)` in credit card regex.
- Security audit: rate limiting, path traversal protection, magic byte validation, XSS output sanitization, security headers applied to all API endpoints.
- Test flakiness in file monitor and WebSocket tests resolved with proper async handling and timeout adjustments.

## [0.4.0] - 2026-02-17

### Added
- **REST API Server** (`core/hades_api.py`) with 11 endpoints: health check, YARA rules status, single-file scan, batch scan, scan result retrieval, metadata sanitization, monitor status/start/stop. Authenticated via `X-API-Key` header with SQLite-backed result persistence.
- **Real-Time File Monitoring** (`core/file_monitor.py`) using watchdog for filesystem event monitoring. Auto-scans new and modified files through both ExifScanner and EnhancedDetectionEngine. Supports configurable alert thresholds, JSON Lines alert logs, and webhook delivery.
- **Plugin System** (`core/plugin_api.py`) with `HADESPlugin` abstract base class, `Finding` dataclass, and `PluginManager`. Supports plugin discovery, sandboxed execution with timeout enforcement, per-plugin metrics, and hot-reload.
- **Example Plugins**: `plugins/example_hash_checker.py` (known-bad hash matching) and `plugins/example_entropy_analyzer.py` (file entropy analysis).
- **Cloud Threat Intelligence** (`core/cloud_threat_intel.py`) with 4 providers (VirusTotal, AbuseIPDB, OTX AlienVault, MalwareBazaar), SQLite cache with TTL, consensus scoring, offline mode, and REST API endpoints (`/api/v1/threat-intel/lookup`, `/api/v1/threat-intel/cache/stats`, `/api/v1/threat-intel/cache/flush`).
- **ML Anomaly Detection** (`core/ml_detection.py`) using Isolation Forest on 15 metadata features. Includes MetadataFeatureExtractor, AnomalyDetector, MLDetectionEngine with auto-retrain, baseline model generation, and REST API endpoints (`/api/v1/ml/status`, `/api/v1/ml/retrain`, `/api/v1/ml/features`). Plugin integration via `plugins/ml_anomaly_plugin.py`.
- **SIEM Integration** (`core/siem_integration.py`) with 5 output formats (Syslog RFC 5424, CEF, STIX 2.1, LEEF, ECS JSON), real-time forwarding via TCP/UDP/HTTP, file output, and REST API endpoints for export and config.
- **Docker Deployment** (`docker/hades-api.dockerfile`, `docker/docker-compose.yml`) with production Docker image, non-root user, ML baseline generation, health checks, and resource limits.
- **GUI Dashboard Components**: `APIDashboard.jsx` (REST API controls, scan history, batch upload), `MonitorDashboard.jsx` (file monitor controls, live status, alert feed), `PluginDashboard.jsx` (plugin management, metrics, reload), `Navigation.jsx` (collapsible sidebar with server status indicator).
- **Updated AppSwitcher** with sidebar navigation and React.lazy code splitting for dashboard views.
- **Electron API Server Management**: IPC handlers in `main.js` for starting, stopping, and checking API server status from the GUI.
- **YARA Rules**: Added `steganography.yar` and `metadata_threats.yar` rule files.
- **Integration Demo Bridge** (`core/integration_demo.py`) re-exports real classes from `metadata_parser_engine.py` as mock wrappers for backward compatibility.
- **Deployment Documentation**: `docs/deployment_guide.md` (Docker deployment, environment variables, security hardening), `docs/quick_start.md` (5-minute getting started guide).
- **API Reference Documentation** (`docs/api_reference.md`) covering all REST endpoints with examples.
- **Plugin Development Guide** (`docs/plugin_development_guide.md`) with walkthrough, API reference, and security considerations.
- **Threat Intelligence Guide** (`docs/threat_intel_guide.md`) with provider setup and configuration.
- **ML Detection Guide** (`docs/ml_detection_guide.md`) with anomaly detection workflow and tuning.
- **Usage Examples** (`examples/usage_examples.sh`) with comprehensive CLI usage demonstrations.
- CLI flags: `--threat-intel`, `--ti-providers`, `--ti-offline`, `--ti-lookup`, `--ti-cache-stats`, `--ml-detect`, `--ml-train`, `--ml-status`, `--ml-baseline`, `--siem-format`, `--siem-output`, `--siem-target`, `--export-scan`.
- **Test Suites**: `core/test_api.py` (REST API endpoint tests), `core/test_file_monitor.py` (file monitoring engine tests), `core/test_plugin_api.py` (plugin API lifecycle tests), `core/test_cloud_threat_intel.py` (43 cloud threat intel tests), `core/test_ml_detection.py` (59 ML detection tests), `core/test_siem_integration.py` (40 SIEM integration tests), `cli/test_integration.py` and `cli/test_enhanced_scan.py` (CLI integration tests), `config/test_config.py` (configuration tests) -- total 238+ passing tests.
- **Distribution Packaging**: Dual PyInstaller executables (`hades-cli`, `hades-server`), Electron GUI with cross-platform targets (Windows NSIS, macOS DMG, Linux AppImage), build scripts.
- `.gitignore` for project root covering Python, Node.js, and project-specific artifacts.

### Changed
- **Directory Reorganization**: Moved files from flat structure into organized directories -- `core/`, `cli/`, `gui/`, `docker/`, `rules/`, `config/`, `templates/`, `docs/`, `examples/`, `plugins/`.
- Updated all import paths and `sys.path` references across the codebase to reflect the new directory structure.
- Enhanced CLI (`core/hades_enhanced_cli.py`) now supports `--serve`, `--monitor`, `--list-plugins`, `--plugins-dir`, `--disable-plugins`, `--threat-intel`, `--ml-detect`, `--siem-format`, and additional flags.
- Enhanced `cli/advanceddetection.py` with `YARA_AVAILABLE` and `EXIFTOOL_AVAILABLE` flags for graceful degradation.
- Enhanced `cli/corescanner.py` with separate `PIL_AVAILABLE`, `EXIFTOOL_AVAILABLE`, `YARA_AVAILABLE` flags.
- Electron preload script (`gui/preload.js`) now exposes `apiServer` context bridge for API server lifecycle management.
- Updated `hades.spec` PyInstaller spec for dual executables (`hades-cli` and `hades-server`).
- `cli/hades_config.json` updated with `threat_intel` and `ml_detection` configuration sections.
- Flask added to `requirements.txt` for the REST API server.
- Watchdog added to `requirements.txt` for file monitoring.
- scikit-learn, numpy, joblib added to `requirements.txt` for ML detection.

### Fixed
- Created `core/integration_demo.py` bridge module to resolve missing import errors in `hades_enhanced_cli.py` and `enterprise_server.py`.
- Fixed YARA import chain: `cli/advanceddetection.py` and `cli/corescanner.py` no longer crash when yara-python is not installed (graceful degradation with `YARA_AVAILABLE` flag).
- Fixed `core/hades_api.py` to wrap optional imports (corescanner, sanitizer, file_monitor) in try/except guards.
- All 5 pre-existing `test_plugin_api` errors resolved (238 tests pass, 0 errors).

## [0.3.0] - 2025-02-06

### Added
- Enhanced GUI interface with Electron + React + Vite + Tailwind CSS.
- EnhancedGUI component with full scanner, viewer, and reporting capabilities.
- Sandboxed file viewer using Docker container with Express.js server.
- Auto-reporting system with HTML, JSON, and CSV output formats.
- Configuration system via `hades_config.json`.
- Advanced detection module (`advanceddetection.py`) with heuristic analysis.
- Metadata sanitizer (`sanitizer.py`) with backup creation.
- YARA rule files: `enhanced_detection.yara`, `advanced_threats.yar`, `enterprise_threats.yar`.
- Enterprise-grade core engine: `metadata_parser_engine.py`, `enhanced_detection_engine.py`, `advanced_reporting.py`.
- Enterprise database, server, installer, and benchmark modules.
- Windows NSIS installer via electron-builder.
- Test suites: `test_cli.py`, `test_reverse_shell.py`.

### Changed
- Version bumped from 0.2.0 to 0.3.0 across CLI and GUI.
- Updated documentation with sandbox viewer guide and reporting guides.

## [0.2.0] - 2024-12-15

### Added
- Basic CLI scanner (`hades_cli.py`) with ExifScanner class.
- Report generation in JSON, HTML, and CSV formats.
- Docker sandbox integration for safe file viewing.
- Threat intelligence configuration files.
- Known-good patterns allowlist.

## [0.1.0] - 2024-10-01

### Added
- Initial release with basic EXIF metadata extraction.
- ExifTool wrapper (`exif_tool.py`).
- Core scanner module (`corescanner.py`).
- Basic suspicious pattern detection.
