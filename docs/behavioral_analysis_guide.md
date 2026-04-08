# Behavioral Analysis Guide

HADES v0.6.0 introduces behavioral analysis that correlates scan results across multiple files to detect threat campaigns, identify file relationships, and surface patterns that single-file analysis cannot reveal.

## What Behavioral Analysis Detects

| Detection Type | Description |
|---|---|
| **Campaign Detection** | Groups of files sharing IOCs (IPs, domains, URLs, hashes), similar attack patterns, or temporal clustering |
| **Metadata Pattern Similarity** | Files with similar injection techniques (script injection + Base64 encoding) grouped by Jaccard similarity |
| **Temporal Clustering** | Files scanned within a configurable time window grouped as potentially related |
| **Author Correlation** | Files sharing the same metadata author, creator software, or camera model |
| **GPS Clustering** | Files with geographically proximate GPS coordinates |

---

## How It Works

The `CampaignDetector` in `core/behavioral_analysis.py` correlates scan results using three strategies:

### 1. Shared IOC Correlation

Files sharing indicators of compromise (IP addresses, domains, URLs, file hashes) are grouped using a union-find data structure. If file A shares a domain with file B, and file B shares an IP with file C, all three are grouped into the same campaign.

### 2. Metadata Pattern Similarity

Each file's attack pattern is represented as a set of tags (e.g., `script_injection`, `base64_encoding`, `shell_command`). Files with a Jaccard similarity >= 0.3 on their tag sets are grouped together.

### 3. Temporal Clustering

Files scanned within a configurable time window (default: 168 hours / 7 days) are grouped. This catches attack campaigns where multiple malicious files are delivered in a short period.

Groups from all three strategies are merged when they overlap, and campaign profiles are generated for groups meeting minimum indicator thresholds.

---

## Running Behavioral Analysis

### CLI

Enable behavioral analysis during a scan:

```bash
hades-enhanced --behavioral -r /evidence/
```

Combine with other detection engines for maximum coverage:

```bash
hades-enhanced --behavioral --ml-ensemble --mitre-map --threat-intel -r /evidence/
```

### API

Behavioral analysis results are included in scan responses when the behavioral module is loaded. Query campaigns via:

```bash
curl -H "X-API-Key: your-api-key-here" \
  http://localhost:8666/api/v1/behavioral/campaigns
```

### Dashboard

The dashboard displays campaign detection results in the Cases and ATT&CK views, linking related files and showing shared indicators.

---

## Campaign Confidence Scoring

Each detected campaign receives a confidence score (0.0 to 1.0) based on:

| Factor | Weight | Description |
|---|---|---|
| Shared IOC count | 0.4 | More shared IOCs = higher confidence |
| Pattern similarity | 0.3 | Stronger pattern overlap = higher confidence |
| File count | 0.2 | More files in the group = higher confidence |
| Temporal proximity | 0.1 | Tighter time window = higher confidence |

A confidence threshold of 0.5 is used by default. Campaigns below this threshold are reported with a lower confidence indicator.

---

## File Relationship Graph

The `FileRelationshipGraph` maintains an adjacency list where edges represent shared indicators between files:

```python
from core.behavioral_analysis import FileRelationshipGraph

graph = FileRelationshipGraph()
graph.add_file("/path/to/file_a.jpg", scan_result_a)
graph.add_file("/path/to/file_b.jpg", scan_result_b)

# Find files related to file_a within 2 hops
related = graph.get_related("/path/to/file_a.jpg", max_depth=2)

# Extract campaign profiles
campaigns = graph.get_campaigns()
```

Multi-hop traversal reveals indirect relationships: file A relates to file B through a shared domain, file B relates to file C through a shared IP, making A and C indirectly related.

---

## Cross-Case Comparison

When using the evidence chain (`--case-id`), behavioral analysis can compare scans across different investigation cases to identify:

- Shared threat actors operating across incidents
- Reuse of C2 infrastructure across campaigns
- Common malware distribution patterns

---

## Example Campaign Detection

Given three files with these findings:

| File | Findings |
|---|---|
| `invoice_001.pdf` | JavaScript injection, URL `http://evil.example.com/payload`, Base64-encoded content |
| `receipt_042.pdf` | JavaScript injection, URL `http://evil.example.com/update`, Base64-encoded content |
| `document_077.pdf` | Template injection, URL `http://evil.example.com/config` |

Behavioral analysis groups these into a single campaign because:

1. All three share the domain `evil.example.com` (shared IOC)
2. `invoice_001.pdf` and `receipt_042.pdf` share attack patterns: `{script_injection, base64_encoding}` (Jaccard = 0.67)
3. All three scanned within 2 hours (temporal clustering)

Campaign output:

```json
{
  "campaign_id": "campaign-001",
  "files": ["invoice_001.pdf", "receipt_042.pdf", "document_077.pdf"],
  "shared_indicators": ["evil.example.com"],
  "attack_patterns": ["script_injection", "base64_encoding", "template_injection"],
  "confidence": 0.85,
  "file_count": 3
}
```

---

## Configuration

Configure behavioral analysis in `cli/hades_config.json`:

```json
{
  "behavioral_analysis": {
    "enabled": true,
    "campaign_window_hours": 168,
    "min_campaign_files": 3
  }
}
```

| Setting | Default | Description |
|---|---|---|
| `enabled` | `true` | Enable behavioral analysis module |
| `campaign_window_hours` | `168` (7 days) | Time window for temporal clustering |
| `min_campaign_files` | `3` | Minimum files to form a campaign |

---

## Next Steps

- [MITRE ATT&CK Mapping](mitre_mapping_guide.md) -- Map findings to MITRE techniques
- [Detection Depth Guide](detection_depth_guide.md) -- Full detection depth architecture
- [ML Detection Guide](ml_detection_guide.md) -- ML anomaly detection
