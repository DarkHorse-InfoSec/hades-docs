# HADES Enhanced Detection Engine v0.7.1

**Enterprise Metadata Forensics & Threat Intelligence Platform**

![License](https://img.shields.io/badge/License-Proprietary-red)
[![Version](https://img.shields.io/badge/Version-0.7.1-blue.svg)]()
[![Enterprise Grade](https://img.shields.io/badge/Enterprise-Grade-red.svg)]()

> **NOTICE (2026-05-06):** This file describes the v0.7.1 era. The current source product has moved on to a different distribution model (Nuitka-compiled signed binary via the customer portal -- see [installation.md](installation.md) and the canonical [/README.md](../README.md)). The Cloudsmith pip-based install commands below no longer work. Most architectural and feature content remains accurate (HADES kept the same async pipeline + worker pool + ML ensemble + evidence chain across the v0.7 -> v1.4 transition); only the install + version-numbering sections are stale. Prefer the canonical README at the repo root over this file.

## 🌟 Overview

HADES (High-Performance Advanced Detection Engine for Security) is a multi-million-dollar enterprise-grade forensic analysis platform designed for comprehensive metadata threat detection, analysis, and intelligence correlation. Built for security professionals, incident responders, and forensic investigators who demand the highest levels of accuracy, performance, and compliance.

### 🎯 Key Features

- **Advanced Metadata Forensics**: Deep analysis of file metadata with sophisticated threat detection
- **Real-Time Threat Intelligence**: Integration with multiple TI feeds and custom IOC management
- **Enterprise Scalability**: High-performance batch processing with horizontal scaling support
- **Compliance Ready**: Built-in support for GDPR, HIPAA, SOX, PCI-DSS, and ISO27001
- **API-First Architecture**: RESTful API with WebSocket real-time updates
- **Multi-Format Reporting**: JSON, HTML, PDF, CSV outputs with chain of custody
- **Advanced Heuristics**: ML-powered anomaly detection and behavioral analysis
- **Zero-Day Detection**: Custom YARA rules and advanced pattern matching

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    HADES Enhanced Detection Engine              │
├─────────────────────────────────────────────────────────────────┤
│  🖥️  Enhanced CLI  │  🌐 REST API  │  📊 WebSocket  │  🔧 Management │
├─────────────────────────────────────────────────────────────────┤
│         🔍 Detection Engine Core (Metadata Analysis)            │
├─────────────────────────────────────────────────────────────────┤
│  📝 YARA Rules  │  🕵️ IOC Engine  │  🧠 Heuristics  │  🔗 Threat Intel │
├─────────────────────────────────────────────────────────────────┤
│           🗄️ Enterprise Database (PostgreSQL/MySQL)             │
├─────────────────────────────────────────────────────────────────┤
│    📈 Monitoring   │   🔒 Security   │   📋 Compliance   │   🚀 DevOps    │
└─────────────────────────────────────────────────────────────────┘
```

## 🚀 Quick Start

### Prerequisites

- **Python**: 3.8+ (3.10+ recommended for production)
- **Memory**: 4GB+ RAM (16GB+ for production)
- **Storage**: 10GB+ free space (200GB+ for production)
- **Database**: PostgreSQL 12+ or MySQL 8.0+
- **OS**: Linux x86_64 (glibc 2.34+) and Windows x86_64. macOS is not available (updated 2026-09-27; see [installation.md](installation.md) for the current platform table).

### Enterprise Installation

```bash
# 1. Obtain the signed binary from the customer portal: see installation.md
#    (the pip-based install this step used to show is retired; see the notice above)

# 2. Run automated installation
sudo python3 enterprise_installer.py \
  --environment production \
  --install-dir /opt/hades \
  --database-url "postgresql://hades:password@localhost/hades_db" \
  --enable-tls \
  --compliance GDPR HIPAA SOX \
  --workers 8

# 3. Start the service
sudo systemctl start hades
sudo systemctl enable hades

# 4. Verify installation
/opt/hades/bin/hades --version
curl -k https://localhost:8080/health
```

### Development Setup

```bash
# 1. Clone the repository
git clone https://github.com/darkhorse-infosec/hades-enhanced.git
cd hades-enhanced

# 2. Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Initialize development database
python core/database/enterprise_database.py \
  --database-url "sqlite:///hades_dev.db" \
  --create-tables

# 5. Run development server
python core/hades_enhanced_cli.py --demo
```

## 📋 Core Components

### 1. Enhanced CLI (`hades_enhanced_cli.py`)

The command-line interface provides comprehensive file analysis capabilities with enterprise-grade features.

**Key Features:**
- Batch processing with multi-threading
- Advanced filtering and sorting
- Threat intelligence integration
- Allowlist management for false positive reduction
- Multiple output formats (JSON, HTML, PDF, CSV)
- Real-time progress tracking
- Compliance mode support

**Example Usage:**
```bash
# Basic single file analysis
hades suspicious_file.jpg

# Enterprise batch analysis with threat intelligence
hades -r /evidence_drive \
  --yara-rules enterprise_rules/ \
  --ti-feeds threat_feeds.json \
  --allowlist corporate_whitelist.txt \
  --format html \
  --output investigation_report.html \
  --audit-mode \
  --chain-custody

# High-performance scanning
hades -r /large_dataset \
  --workers 16 \
  --min-score 50 \
  --sort-by score \
  --limit 100 \
  --performance-metrics
```

### 2. REST API Server (`hades_api.py`)

RESTful API server with authentication, rate limiting, and real-time capabilities.

**Key Endpoints:**
- `POST /api/v1/analyze` - Single file analysis
- `POST /api/v1/analyze/batch` - Batch analysis
- `POST /api/v1/upload` - File upload and analysis
- `POST /api/v1/threat-intelligence/lookup` - IOC lookup
- `POST /api/v1/compliance/report` - Compliance reporting
- `GET /api/v1/analyze/{id}` - Get analysis results
- `WebSocket /ws/{client_id}` - Real-time updates

**Example API Usage:**
```python
import requests

# Authenticate
headers = {"Authorization": "Bearer your_api_key_here"}

# Upload and analyze file
files = {"file": open("suspicious.exe", "rb")}
response = requests.post(
    "https://hades-api.company.com/api/v1/upload",
    files=files,
    headers=headers
)

analysis_id = response.json()["analysis_id"]

# Get results
result = requests.get(
    f"https://hades-api.company.com/api/v1/analyze/{analysis_id}",
    headers=headers
)
```

### 3. Database Integration (`enterprise_database.py`)

Enterprise-grade database layer with full audit trails and compliance support.

**Features:**
- Multi-database support (PostgreSQL, MySQL, SQLite)
- Automatic schema migration
- Connection pooling and optimization
- Comprehensive audit logging
- Data retention policies
- Backup and recovery

### 4. Threat Intelligence (`threat_intelligence.json` + Manager)

Advanced threat intelligence integration with multiple feed support.

**Supported Sources:**
- Custom JSON/CSV feeds
- MISP integration
- AlienVault OTX
- VirusTotal API
- Custom IOC databases

### 5. Configuration Validation (`validation_utility.py`)

Enterprise configuration management with security policy enforcement.

**Features:**
- JSON Schema validation
- Security policy checking
- Compliance requirement validation
- Performance optimization recommendations
- Environment-specific validation

### 6. Performance Benchmarking (`performance_benchmark.py`)

Comprehensive performance testing and optimization framework.

**Benchmark Types:**
- Single file analysis performance
- Batch processing scalability
- Memory usage patterns
- Concurrent access testing
- Threat intelligence lookup speed

## 🔒 Security Features

### Authentication & Authorization

- **API Key Management**: Role-based access control with customizable permissions
- **Rate Limiting**: Configurable per-key and global rate limits
- **Session Management**: Secure session handling with timeout controls
- **Audit Logging**: Comprehensive audit trails for all operations

### Data Protection

- **Encryption at Rest**: AES-256-GCM encryption for sensitive data
- **TLS/SSL**: End-to-end encryption for all communications
- **Input Validation**: Comprehensive input sanitization and validation
- **Secure File Handling**: Sandboxed file processing with resource limits

### Compliance Support

- **GDPR**: Data minimization, consent tracking, right to deletion
- **HIPAA**: PHI protection, access controls, audit trails
- **SOX**: Financial data handling, change management, retention
- **PCI-DSS**: Secure card data handling, network monitoring
- **ISO27001**: Information security management system compliance

## 📊 Reporting & Analytics

### Report Formats

1. **HTML Reports**: Interactive dashboards with visualizations
2. **PDF Reports**: Professional documents with chain of custody
3. **JSON Reports**: Machine-readable for automation and integration
4. **CSV Reports**: Spreadsheet-compatible for analysis
5. **XML Reports**: Structured data for enterprise systems

### Dashboard Features

- Real-time threat level distribution
- Processing performance metrics
- Historical trend analysis
- Geographic threat mapping
- Compliance status tracking

### Sample Report Sections

```
🛡️ HADES Enhanced Detection Report
=====================================
Executive Summary
- Total Files Analyzed: 1,247
- High-Risk Files: 23
- Average Threat Score: 15.7/100
- Processing Time: 3.2 minutes

Threat Distribution
- Critical: 3 files
- High: 20 files  
- Medium: 87 files
- Low: 234 files
- Safe: 903 files

Key Findings
- 5 malware signatures detected
- 12 suspicious metadata patterns
- 3 potential data exfiltration indicators
- 8 policy violations identified
```

## 🔧 Configuration

### Environment Configuration

Create `hades_config.yaml`:

```yaml
detection_engine:
  yara_rules_directory: "/opt/hades/rules"
  threat_intelligence:
    enable_local_feeds: true
    enable_remote_feeds: true
    cache_timeout_seconds: 3600

api_server:
  host: "0.0.0.0"
  port: 8080
  workers: 8
  authentication:
    enabled: true
    api_keys:
      "enterprise_key_001":
        permissions: ["analyze", "batch", "threat_intel", "compliance"]
        organization: "Security Team"
        rate_limit: 1000

database:
  url: "postgresql://hades:secure_password@db.company.com/hades_prod"
  connection_pool:
    pool_size: 20
    max_overflow: 30

security:
  encryption:
    enabled: true
    algorithm: "AES-256-GCM"
  tls:
    enabled: true
    cert_file: "/opt/hades/config/server.crt"
    key_file: "/opt/hades/config/server.key"

compliance:
  frameworks: ["GDPR", "HIPAA", "SOX"]
  data_retention:
    analysis_results_days: 365
    audit_logs_days: 2555
```

### YARA Rules

Custom threat detection rules in `rules/`:

```yara
rule MetadataPayloadInjection {
    meta:
        description = "Detects malicious payload injection in metadata"
        severity = "critical"
        author = "HADES"
    
    strings:
        $payload1 = "powershell" nocase
        $payload2 = "cmd.exe" nocase
        $payload3 = "VirtualAlloc" nocase
    
    condition:
        any of them
}
```

### Allowlist Patterns

Reduce false positives with `allowlist_patterns.txt`:

```
# Known good hashes
hash:d41d8cd98f00b204e9800998ecf8427e

# Corporate domains
domain:company.com
domain:trusted-vendor.net

# Safe paths
path:/usr/bin/*
path:C:\Program Files\*

# Regex patterns
regex:^Copyright \(c\) Microsoft Corporation$
```

## 🚀 Deployment Scenarios

### Single Server Deployment

```bash
# Simple production deployment
python3 enterprise_installer.py \
  --environment production \
  --install-dir /opt/hades \
  --database-url "postgresql://hades:password@localhost/hades" \
  --workers 4 \
  --enable-tls
```

### High-Availability Cluster

```bash
# Load balancer + multiple API servers + shared database
# Node 1
python3 enterprise_installer.py \
  --environment production \
  --install-dir /opt/hades \
  --database-url "postgresql://hades:password@db-cluster.internal/hades" \
  --api-host 10.0.1.10 \
  --workers 8

# Node 2  
python3 enterprise_installer.py \
  --environment production \
  --install-dir /opt/hades \
  --database-url "postgresql://hades:password@db-cluster.internal/hades" \
  --api-host 10.0.1.11 \
  --workers 8
```

### Container Deployment

```dockerfile
# Dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    python3 python3-pip postgresql-client yara

COPY . /opt/hades
WORKDIR /opt/hades

RUN pip3 install -r requirements.txt

EXPOSE 8080
CMD ["python3", "core/hades_enhanced_cli.py", "--serve", "--port", "8080"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  hades-api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://hades:password@postgres:5432/hades
    depends_on:
      - postgres
    
  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: hades
      POSTGRES_USER: hades
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## 📈 Performance Optimization

### Hardware Recommendations

| Environment | CPU | RAM | Storage | Network |
|-------------|-----|-----|---------|---------|
| Development | 2 cores | 4GB | 50GB SSD | 1Gbps |
| Testing | 4 cores | 8GB | 100GB SSD | 1Gbps |
| Staging | 8 cores | 16GB | 200GB SSD | 10Gbps |
| Production | 16+ cores | 32GB+ | 500GB+ NVMe | 10Gbps+ |

### Tuning Parameters

```yaml
performance:
  max_workers: 16          # 1-2x CPU cores
  max_file_size_mb: 100    # Limit large files
  timeout_seconds: 300     # Analysis timeout
  memory_limit_mb: 8192    # Per-worker memory limit

database:
  connection_pool:
    pool_size: 20          # Connection pool size
    max_overflow: 30       # Additional connections
    pool_timeout: 30       # Connection timeout
```

### Benchmarking

```bash
# Run performance benchmarks
python3 core/benchmark/performance_benchmark.py \
  --database-url "postgresql://hades:password@localhost/hades" \
  --workers 8 \
  --output benchmark_results/

# Quick performance test
hades --benchmark --workers 8
```

## 🔍 Troubleshooting

### Common Issues

**1. High Memory Usage**
```bash
# Monitor memory usage
hades --debug --memory-limit 2GB large_files/

# Solution: Reduce workers or file size limits
```

**2. Database Connection Errors**
```bash
# Test database connectivity
python3 core/database/enterprise_database.py \
  --database-url "postgresql://user:pass@host/db" \
  --validate

# Check connection pool settings
```

**3. Performance Issues**
```bash
# Run performance analysis
hades --performance-metrics --verbose slow_processing/

# Check system resources
top -p $(pgrep -f hades)
```

### Log Analysis

```bash
# View application logs
journalctl -u hades -f

# Check API access logs
tail -f /opt/hades/logs/hades.log

# Audit log analysis
grep "CRITICAL\|ERROR" /opt/hades/logs/audit.log
```

### Debug Mode

```bash
# Enable comprehensive debugging
hades --debug --log-file debug.log --verbose problematic_file.exe

# API server debug mode
python3 core/hades_enhanced_cli.py --serve --port 8666
```

## 🤝 Integration Examples

### SIEM Integration (Splunk)

```bash
# Send high-risk alerts to Splunk
hades -r /monitored_directory \
  --min-score 75 \
  --format json \
  --output /dev/stdout | \
  logger -t HADES-HIGH-RISK
```

### SOC Workflow Integration

```python
# Automated incident response
import requests

def check_file_with_hades(file_path):
    response = requests.post(
        "https://hades.company.com/api/v1/analyze",
        json={"file_path": file_path},
        headers={"Authorization": "Bearer API_KEY"}
    )
    
    result = response.json()
    if result["threat_score"] >= 75:
        # Trigger incident response
        create_incident(file_path, result)
        quarantine_file(file_path)
```

### Threat Hunting Platform

```bash
# Hunt for specific IOCs across infrastructure
hades -r /network_shares \
  --ti-feeds hunting_iocs.json \
  --min-score 25 \
  --format csv \
  --output threat_hunt_results.csv
```

## 📚 Advanced Features

### Machine Learning Integration

```python
# Custom ML model integration
from hades.ml import ThreatClassifier

classifier = ThreatClassifier()
classifier.load_model("models/malware_detector.pkl")

# Use in custom analysis pipeline
def enhanced_analysis(file_path):
    base_result = hades_analyze(file_path)
    ml_score = classifier.predict(file_path)
    
    return combine_scores(base_result, ml_score)
```

### Custom Plugin Development

```python
# Custom heuristic plugin
from hades.plugins import HeuristicPlugin

class CustomThreatDetector(HeuristicPlugin):
    def analyze(self, metadata, content):
        # Custom threat detection logic
        if self.detect_custom_pattern(metadata):
            return ThreatFinding(
                severity="high",
                description="Custom threat pattern detected"
            )
```

### Webhook Integration

```yaml
# Configuration for webhooks
api_server:
  webhooks:
    enabled: true
    endpoints:
      - url: "https://siem.company.com/alerts"
        events: ["high_risk_detection"]
        authentication:
          type: "bearer"
          token: "webhook_token"
```

## 📖 API Documentation

The complete API documentation is available at:
- Production: `https://your-hades-server/docs`
- Interactive API: `https://your-hades-server/redoc`

### Authentication

```bash
# Get API key from administrator
curl -X POST "https://hades-api.company.com/api/v1/auth/token" \
  -H "Content-Type: application/json" \
  -d '{"username": "analyst", "password": "secure_password"}'
```

### Rate Limits

| Tier | Requests/Minute | Requests/Hour | Concurrent |
|------|-----------------|---------------|------------|
| Basic | 100 | 1,000 | 2 |
| Professional | 500 | 10,000 | 5 |
| Enterprise | 2,000 | 50,000 | 20 |

## 🆘 Support & Resources

### Documentation
- **Installation Guide**: [docs/installation.md](docs/installation.md)
- **API Reference**: [docs/api-reference.md](docs/api-reference.md)
- **Configuration Guide**: [docs/configuration.md](docs/configuration.md)
- **Troubleshooting**: [docs/troubleshooting.md](docs/troubleshooting.md)

### Community
- **GitHub Issues**: https://github.com/darkhorse-infosec/hades-enhanced/issues
- **Security Advisories**: security@darkhorse-infosec.com
- **Feature Requests**: https://github.com/darkhorse-infosec/hades-enhanced/discussions

### Enterprise Support
- **Email**: enterprise-support@darkhorse-infosec.com
- **Phone**: +1-555-HADES-01 (US/Canada)
- **Emergency**: +1-555-HADES-911 (24/7 for Critical customers)
- **Professional Services**: consulting@darkhorse-infosec.com

### Training & Certification
- **HADES Certified Analyst (HCA)**: 2-day certification program
- **HADES Advanced Administrator (HAA)**: 3-day advanced training
- **Custom Training**: On-site training available for enterprise customers

## 📄 License & Legal

### License
HADES Enhanced Detection Engine is licensed under the Proprietary License. See [LICENSE](LICENSE) for details.

### Enterprise Licensing
Commercial licenses with additional features and support are available:
- **Professional**: Small teams (up to 10 analysts)
- **Enterprise**: Large organizations (unlimited users)
- **Government**: Special pricing for government agencies
- **Academic**: Educational institutions and research

Contact: licensing@darkhorse-infosec.com

### Compliance Framework Mapping
HADES maps detection findings to the following compliance frameworks, helping organizations align forensic analysis with their regulatory requirements:
- **NIST CSF** - Cybersecurity Framework alignment for detection and response
- **ISO 27001** - Information security management system controls
- **GDPR** - Data protection and privacy compliance support

### Third-Party Components
HADES includes third-party open-source components. Third-party licenses are documented in `pyproject.toml` under project dependencies.

---

## 🎯 Conclusion

HADES Enhanced Detection Engine represents the pinnacle of enterprise metadata forensics technology. With its combination of advanced threat detection, comprehensive compliance support, enterprise-grade scalability, and professional services backing, HADES provides organizations with the tools they need to defend against the most sophisticated threats.

Whether you're a security analyst investigating suspicious files, an incident responder triaging a potential breach, or a forensic investigator building a case for prosecution, HADES delivers the accuracy, performance, and reliability demanded by today's cybersecurity professionals.

**Ready to deploy HADES in your environment?**

Contact our enterprise team today:
- 📧 **Sales**: sales@darkhorse-infosec.com
- 🌐 **Website**: https://www.darkhorse-infosec.com/hades
- 📞 **Phone**: +1-555-HADES-SALES

---

*© 2024 DarkHorse Information Security LLC. All rights reserved. HADES is a trademark of DarkHorse Information Security LLC.*