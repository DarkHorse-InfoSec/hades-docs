# HADES Next Steps

**Version:** 1.0.0 GA
**Author:** DarkHorse Information Security LLC

> **HISTORICAL NOTICE (added 2026-05-06):** This is a planning checklist from the v1.0.0 release era, kept for historical reference. The release-packaging items (Cloudsmith package publish, Docker image push, Homebrew formula bump, etc.) were applicable under the pre-v1.4 distribution model that has since been retired. Current release-packaging work happens in the private source repo at `tasks/TODO_manual.md`. Strategic / product / customer-success items below may still be relevant; verify against the current TODO before acting.

HADES v1.0.0 is the first General Availability release. All core technical features were implemented at that milestone. The synthetic-corpus validation committed with the 2026-03-01 initial import (`tests/corpus/validation_results.json` in the source repo) detected 88 of 90 malicious synthetic files (TPR 97.78%) and flagged 0 of 18 clean ones. That is a synthetic corpus, not real-world malware; for the current real-world measurement see the HADES product page. This document outlines the next steps across engineering, validation, and business execution.

---

## Table of Contents

1. [v1.0.0 Release Status](#v100-release-status)
2. [Immediate Next Steps](#immediate-next-steps)
3. [v1.1.0 Feature Roadmap](#v110-feature-roadmap)
4. [Testing & Validation Expansion](#testing--validation-expansion)
5. [Infrastructure & Operations](#infrastructure--operations)
6. [Business Execution](#business-execution)
7. [Long-Term Vision](#long-term-vision)

---

## 1. v1.0.0 Release Status

### Completed in v1.0.0

| Area | Deliverables |
|------|-------------|
| **SIEM Connectors** | Splunk HEC, Elasticsearch, Microsoft Sentinel with platform APIs |
| **SDK 1.1.0** | Retry logic, webhook subscriptions, SSE streaming |
| **Community Edition** | 13-feature free tier for individual practitioners |
| **MITRE ATT&CK** | 51 techniques across 14 tactics |
| **GPS Forensics** | DMS/decimal parsing, Haversine clustering, impossible travel detection |
| **Quarantine Manager** | SQLite-backed lifecycle management with API and dashboard |
| **Sanitize Dashboard** | Drag-and-drop metadata removal with before/after comparison |
| **Configurable Thresholds** | Externalized detection parameters across all engines |
| **Startup Health Report** | Structured module availability at API server boot |

### Key Metrics

| Metric | Value |
|--------|-------|
| Synthetic-corpus TPR (2026-03-01) | 97.78% (88 of 90 malicious synthetic files) |
| Synthetic-corpus FPR (2026-03-01) | 0% (0 of 18 clean synthetic files) |
| YARA rules | 57 across 8 rule files at the 2026-03-01 import (144 across 55 at v1.7.1) |

---

## 2. Immediate Next Steps

These items should be completed before or shortly after the v1.0.0 public release.

### 2.1 Release Packaging

- [ ] Tag `v1.0.0` in git and create GitHub release
- [ ] Build and publish Cloudsmith package (`pip install hades-scanner==1.0.0`)
- [ ] Build and push Docker image with OCI labels (`hades-scanner:1.0.0`)
- [ ] Update Homebrew formula to v1.0.0
- [ ] Verify all install paths: pip, Docker, Homebrew, from-source

### 2.2 Documentation Finalization

- [ ] Build and deploy MkDocs site to GitHub Pages
- [ ] Review and publish `RELEASE_NOTES_v1.0.0.md`
- [ ] Verify all 42+ documentation pages render correctly
- [ ] Ensure API reference matches actual endpoint behavior
- [ ] Cross-reference CLI help output with `cli_reference.md`

### 2.3 Known Issues to Address

These are documented in `RELEASE_NOTES_v1.0.0.md`:

| Issue | Priority | Target |
|-------|----------|--------|
| Splunk HEC token hot-swap | Medium | v1.1.0 |
| Sentinel managed identity auth | Medium | v1.1.0 |
| Community Docker image size (450MB) | Low | v1.1.0 |
| Dashboard license cache requires refresh | Low | v1.1.0 |
| Elasticsearch ILM policy management | Low | v1.1.0 |

---

## 3. v1.1.0 Feature Roadmap

Planned features previewed in the v1.0.0 release notes.

### 3.1 Custom SIEM Connector SDK

Public API for building connectors to additional SIEM platforms (Chronicle, LogRhythm, Exabeam, QRadar, etc.).

**Scope:**
- Abstract connector base class with standard lifecycle (connect, send, health check, disconnect)
- Connector packaging format for distribution via plugin marketplace
- Documentation and example connector implementation
- Connector testing framework with mock SIEM server

### 3.2 SOAR Playbook Marketplace

Shareable playbook definitions with import/export across HADES instances.

**Scope:**
- Playbook export format (JSON schema with version metadata)
- Import validation and conflict resolution
- Community playbook repository
- Playbook testing/dry-run mode

### 3.3 Threat Intelligence Sharing

Collaborative IOC sharing between HADES deployments via STIX/TAXII federation.

**Scope:**
- TAXII 2.1 server endpoint for publishing IOCs
- TAXII client for subscribing to external collections
- IOC deduplication and confidence scoring
- Privacy controls (anonymization, allowlists)

### 3.4 SDK Local Engine Mode

Embedded scan engine within the Python SDK for offline analysis without an API server.

**Scope:**
- `HadesLocal` class wrapping `EnhancedDetectionEngine` directly
- No network dependency, no API key required
- Subset of features (scan, detect, sanitize)
- Portable deployment for air-gapped environments

---

## 4. Testing & Validation Expansion

The current test suite uses synthetic files. Expanding to real-world samples is the highest-priority quality initiative.

### 4.1 Real-World Malware Validation

**Status:** Framework exists (`tests/test_malwarebazaar_validation.py`, `scripts/run_malware_validation.py`). Needs execution with real samples.

**Action items:**

- [ ] Download 50+ MalwareBazaar samples across diverse malware families
- [ ] Run detection pipeline against each sample, record TPR
- [ ] Identify false negatives and create new YARA rules or heuristics
- [ ] Target: TPR >= 85% on real-world samples (lower than synthetic due to noise)

See [Real-World Validation Guide](real_world_validation_guide.md) for step-by-step procedures.

### 4.2 Live SIEM Connector Testing

**Status:** All 37 SIEM connector tests use mocked HTTP. No live platform validation.

**Action items:**

- [ ] Set up Splunk Free (Docker) and test HEC connector end-to-end
- [ ] Set up Elasticsearch + Kibana (Docker) and test bulk indexing
- [ ] Set up Azure Log Analytics workspace and test Sentinel connector
- [ ] Validate event format compatibility with native platform dashboards

### 4.3 Firmware Detection with Real Samples

**Status:** 29 tests use synthetic headers only. No real firmware images tested.

**Action items:**

- [ ] Obtain sample UEFI capsules from vendor update packages
- [ ] Test SPI flash image detection with real dumps
- [ ] Validate BadUSB/DuckyScript detection with Flipper Zero payloads
- [ ] Ensure zero false positives on normal executables

### 4.4 Performance Benchmarking at Scale

**Status:** Benchmark framework exists (`core/benchmarks.py`). Needs large-scale corpus testing.

**Action items:**

- [ ] Generate 10,000+ file test corpus using `benchmarks/generate_test_corpus.py`
- [ ] Measure throughput targets: single-node 10K files/hour, distributed 100K files/hour
- [ ] Profile memory usage under sustained load
- [ ] Identify and document bottlenecks with `core/profiler.py`

### 4.5 Security Auditing

- [ ] Run `pip-audit` or `safety check` against all dependencies
- [ ] Review all API endpoints for input validation gaps
- [ ] Fuzz file upload endpoints with malformed inputs
- [ ] Validate rate limiting under load (sliding window correctness)
- [ ] Test path traversal defenses with comprehensive payloads

---

## 5. Infrastructure & Operations

### 5.1 Cloud Deployment

- [ ] Create Terraform modules for AWS deployment (ECS/EKS, RDS, ElastiCache)
- [ ] Create Terraform modules for Azure deployment (AKS, PostgreSQL, Redis Cache)
- [ ] Create Terraform modules for GCP deployment (GKE, Cloud SQL, Memorystore)
- [ ] Document managed database and Redis configuration

### 5.2 Observability Hardening

- [ ] Add distributed tracing (OpenTelemetry integration)
- [ ] Create runbook for each Alertmanager alert
- [ ] Set up log aggregation pipeline (Loki or Elasticsearch)
- [ ] Define SLOs: API latency p99 < 500ms, scan latency p95 < 2s, availability 99.9%

### 5.3 CI/CD Pipeline Expansion

- [ ] Add automated Cloudsmith publishing on release tags
- [ ] Add Docker image building and pushing to container registry
- [ ] Add SBOM generation (CycloneDX or SPDX)
- [ ] Add dependency vulnerability scanning in CI
- [ ] Add performance regression testing (benchmark comparison across commits)

---

## 6. Business Execution

The technical product is feature-complete. The following business execution items are the path to commercialization.

### 6.1 Licensing & Pricing

| Tier | Target Audience | Key Features |
|------|----------------|--------------|
| **Community** (free) | Individual practitioners | CLI scan, basic API, evidence chain, scan analytics; IOC and heuristic stages only |
| **Professional** | Small security teams | + YARA, ML, full engine, SIEM export, threat intel, MITRE mapping, monitoring, playbooks, plugins |
| **Team** | Security teams, MSPs | + RBAC, SSO, cloud scanning, CI/CD, chat integrations, encrypted storage |
| **Enterprise** | Large organizations | + multi-tenant administration |

**Action items:**
- [ ] Finalize pricing strategy
- [ ] Draft EULA and Terms of Service
- [ ] Set up license key distribution workflow
- [ ] Build self-service license portal

### 6.2 Go-to-Market

- [ ] Launch documentation site publicly
- [ ] Submit to AWS, Azure, and GCP marketplace listings
- [ ] Create conference demo environment
- [ ] Develop content marketing strategy (blog posts, case studies, comparison guides)
- [ ] Identify MSSP partnership opportunities

### 6.3 HADES-as-a-Service (SaaS)

- [ ] Deploy cloud infrastructure (Terraform)
- [ ] Set up managed PostgreSQL and Redis
- [ ] Implement API gateway with usage-based metering
- [ ] Create billing integration
- [ ] Begin SOC 2 Type I audit preparation

---

## 7. Long-Term Vision

### v1.2.0 and Beyond

| Feature | Description |
|---------|-------------|
| **Video/Audio Forensics** | Metadata analysis for MP4, MKV, WAV, FLAC containers |
| **Memory Forensics Integration** | Analyze metadata artifacts from memory dumps (Volatility integration) |
| **Threat Hunting Notebooks** | Jupyter notebook integration for interactive investigation workflows |
| **Federated Deployment** | Multi-site HADES deployments with centralized management plane |
| **Custom ML Model Training** | Web UI for training organization-specific anomaly models on labeled data |
| **Compliance Reporting** | Automated SOX, PCI DSS, HIPAA, GDPR compliance report generation |
| **Mobile Device Forensics** | iOS and Android backup file metadata extraction |

### Research Directions

- **Adversarial ML robustness:** Test detection models against evasion techniques (metadata crafting to avoid ML detection)
- **Zero-day metadata exploits:** Monitor CVE feeds for new metadata-based attack vectors
- **Steganography advancement:** Deep learning approaches for detecting steganographic content in images
- **Supply chain metadata:** Track software provenance through build system metadata artifacts

---

## Priority Matrix

| Priority | Item | Timeline |
|----------|------|----------|
| **P0 : Now** | Release packaging (Cloudsmith, Docker, tag) | Week 1 |
| **P0 : Now** | Documentation site deployment | Week 1 |
| **P1 : Soon** | Real-world malware validation (50+ samples) | Weeks 2-4 |
| **P1 : Soon** | Live SIEM connector testing | Weeks 2-4 |
| **P1 : Soon** | Security dependency audit | Week 2 |
| **P2 : Next** | v1.1.0 feature development | Months 2-3 |
| **P2 : Next** | Cloud deployment (Terraform) | Months 2-3 |
| **P2 : Next** | Licensing and pricing | Month 2 |
| **P3 : Later** | HADES-as-a-Service | Months 4-6 |
| **P3 : Later** | Marketplace listings | Months 4-6 |
| **P4 : Future** | v1.2.0 features (video, memory, notebooks) | Months 6+ |
