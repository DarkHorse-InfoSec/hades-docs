# HADES SIEM Integration Guide

## Overview

HADES provides native SIEM (Security Information and Event Management) integration, allowing scan results and threat alerts to be exported in industry-standard log formats and forwarded in real time to SIEM platforms. This enables security operations centers to incorporate HADES metadata forensics into their existing monitoring workflows.

**Supported output formats:**

| Format   | Standard             | Primary Platforms                         |
|----------|----------------------|-------------------------------------------|
| Syslog   | RFC 5424             | Splunk, rsyslog, syslog-ng, any RFC 5424 receiver |
| CEF      | ArcSight CEF Rev 25  | ArcSight, Microsoft Sentinel, Splunk      |
| STIX     | STIX 2.1 (JSON)      | TAXII servers, OpenCTI, MISP, Sentinel    |
| LEEF     | IBM LEEF 2.0         | IBM QRadar                                |
| ECS      | Elastic Common Schema 8.x | Elasticsearch, Kibana, Elastic SIEM  |

**Delivery methods:**

| Method   | Description                                              |
|----------|----------------------------------------------------------|
| stdout   | Print formatted output to standard output (default)      |
| file     | Append to a file on disk                                 |
| syslog   | Send via UDP to a remote syslog receiver                 |
| http     | POST to an HTTP/HTTPS endpoint                           |

---

## Quick Start

### Scan and output in CEF format

```bash
python core/hades_enhanced_cli.py -r /evidence/ --siem-format cef
```

### Scan and output in Syslog RFC 5424 format

```bash
python core/hades_enhanced_cli.py suspicious.jpg --siem-format syslog
```

### Export as STIX 2.1 bundle

```bash
python core/hades_enhanced_cli.py suspicious.jpg --siem-format stix > threat_bundle.json
```

### Export in LEEF format for QRadar

```bash
python core/hades_enhanced_cli.py -r /evidence/ --siem-format leef
```

### Export in Elastic Common Schema JSON

```bash
python core/hades_enhanced_cli.py -r /evidence/ --siem-format ecs
```

### Forward alerts to a remote syslog server

```bash
python core/hades_enhanced_cli.py --monitor /evidence/ \
    --siem-format syslog --siem-output syslog --siem-target 10.0.0.50:514
```

### Export to a file

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format ecs --siem-output file --siem-target /var/log/hades/ecs.json
```

### Re-export a previous scan

```bash
python core/hades_enhanced_cli.py --export-scan abc-123-def --siem-format cef
```

---

## Format Reference

### Syslog (RFC 5424)

HADES generates RFC 5424-compliant syslog messages with structured data elements.

**Message structure:**

```
<PRI>1 TIMESTAMP HOSTNAME HADES PROCID MSGID [hades@57483 key="value" ...] MESSAGE
```

**Facility and severity mapping:**

| HADES Threat Level | Syslog Facility | Syslog Severity | Numeric Priority |
|--------------------|-----------------|-----------------|------------------|
| critical           | local0 (16)     | alert (1)       | 129              |
| high               | local0 (16)     | critical (2)    | 130              |
| medium             | local0 (16)     | warning (4)     | 132              |
| low                | local0 (16)     | notice (5)      | 133              |
| info / safe        | local0 (16)     | informational (6)| 134             |

**Structured data fields (`[hades@57483]`):**

| SD Field         | Description                          | Example                              |
|------------------|--------------------------------------|--------------------------------------|
| `scanId`         | Unique scan identifier               | `a1b2c3d4-e5f6-7890-abcd-ef123456`  |
| `fileName`       | Scanned file name                    | `suspicious.jpg`                     |
| `fileHash`       | SHA-256 hash of the file             | `e3b0c44298fc1c149afb...`            |
| `threatScore`    | Numeric threat score (0-100)         | `85.0`                               |
| `threatLevel`    | Severity category                    | `high`                               |
| `yaraMatches`    | Comma-separated YARA rule names      | `embedded_exe,script_injection`      |
| `iocMatches`     | Number of IOC matches                | `3`                                  |
| `detectionTypes` | Types of detections triggered        | `yara,heuristic,ioc`                 |

**Example output:**

```
<130>1 2026-02-17T14:22:01.000Z forensic-ws HADES 12345 SCAN [hades@57483 scanId="abc-123" fileName="suspicious.jpg" fileHash="e3b0c44..." threatScore="85.0" threatLevel="high" yaraMatches="embedded_exe" iocMatches="2" detectionTypes="yara,heuristic"] Threat detected: suspicious.jpg scored 85.0 (high)
```

---

### CEF (Common Event Format)

CEF messages follow ArcSight CEF revision 25 format with HADES-specific extension keys.

**Header structure:**

```
CEF:0|DarkHorse InfoSec|HADES|1.0.0|<SignatureID>|<Name>|<Severity>|<Extension>
```

**Header fields:**

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Version       | `0`                                        |
| Device Vendor | `DarkHorse InfoSec`                        |
| Device Product| `HADES`                                    |
| Device Version| Engine version (e.g., `1.0.0`)             |
| Signature ID  | Detection type (e.g., `HADES-YARA-001`)    |
| Name          | Threat summary                             |
| Severity      | 0-10 mapped from threat score              |

**CEF severity mapping:**

| HADES Threat Score | CEF Severity | Label    |
|--------------------|-------------|----------|
| 0-19               | 1           | Low      |
| 20-39              | 3           | Low      |
| 40-59              | 5           | Medium   |
| 60-79              | 7           | High     |
| 80-100             | 9           | Very High|

**Extension keys:**

| Key     | Description                   | Example                          |
|---------|-------------------------------|----------------------------------|
| `fname` | File name                     | `suspicious.jpg`                 |
| `fsize` | File size in bytes            | `245760`                         |
| `fhash` | SHA-256 file hash             | `e3b0c44298fc1c149afb...`        |
| `cs1`   | Scan ID                       | `a1b2c3d4-e5f6-7890-abcd`       |
| `cs1Label` | Label for cs1              | `ScanID`                         |
| `cs2`   | YARA rule matches             | `embedded_exe,script_injection`  |
| `cs2Label` | Label for cs2              | `YARAMatches`                    |
| `cs3`   | Threat level                  | `high`                           |
| `cs3Label` | Label for cs3              | `ThreatLevel`                    |
| `cn1`   | Threat score                  | `85`                             |
| `cn1Label` | Label for cn1              | `ThreatScore`                    |
| `cn2`   | IOC match count               | `2`                              |
| `cn2Label` | Label for cn2              | `IOCMatchCount`                  |
| `msg`   | Detection summary             | `Embedded executable detected`   |
| `rt`    | Receipt time (epoch ms)       | `1708182121000`                  |
| `src`   | Source hostname                | `forensic-workstation`           |

**Example output:**

```
CEF:0|DarkHorse InfoSec|HADES|1.0.0|HADES-YARA-001|Embedded executable detected in EXIF comment|9|fname=suspicious.jpg fsize=245760 fhash=e3b0c44298fc1c... cs1=abc-123 cs1Label=ScanID cs2=embedded_exe cs2Label=YARAMatches cs3=high cs3Label=ThreatLevel cn1=85 cn1Label=ThreatScore cn2=2 cn2Label=IOCMatchCount msg=Embedded executable detected in EXIF comment rt=1708182121000
```

---

### STIX 2.1 (Structured Threat Information eXpression)

HADES generates STIX 2.1 JSON bundles that can be shared via TAXII servers, imported into threat intelligence platforms, or ingested by STIX-aware SIEMs.

**Object types generated:**

| STIX Object Type         | When Generated                              |
|--------------------------|---------------------------------------------|
| `observed-data`          | Every scan result                           |
| `file` (SCO)             | Every scanned file (name, hash, size)       |
| `indicator`              | Each YARA match or heuristic finding        |
| `malware`                | When threat score >= 70                     |
| `relationship`           | Links between indicators, observables, malware |
| `identity`               | HADES scanner identity (author)             |
| `report`                 | Wraps all objects from a scan session       |

**Indicator pattern expressions:**

```
[file:hashes.'SHA-256' = 'e3b0c44298fc1c149afb...']
[file:name = 'suspicious.jpg' AND file:hashes.'SHA-256' = 'e3b0c44...']
```

**Relationship graph:**

```
indicator --indicates--> malware
observed-data --consists-of--> file (SCO)
report --object-refs--> [indicator, observed-data, malware]
```

**Example bundle (abbreviated):**

```json
{
  "type": "bundle",
  "id": "bundle--a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "objects": [
    {
      "type": "identity",
      "id": "identity--hades-scanner",
      "name": "HADES Metadata Forensics Engine",
      "identity_class": "system",
      "created_by_ref": "identity--hades-scanner"
    },
    {
      "type": "observed-data",
      "id": "observed-data--...",
      "first_observed": "2026-02-17T14:22:01Z",
      "last_observed": "2026-02-17T14:22:01Z",
      "number_observed": 1,
      "object_refs": ["file--..."]
    },
    {
      "type": "file",
      "id": "file--...",
      "name": "suspicious.jpg",
      "size": 245760,
      "hashes": {
        "SHA-256": "e3b0c44298fc1c149afb..."
      }
    },
    {
      "type": "indicator",
      "id": "indicator--...",
      "name": "YARA: embedded_exe",
      "pattern": "[file:hashes.'SHA-256' = 'e3b0c44...']",
      "pattern_type": "stix",
      "valid_from": "2026-02-17T14:22:01Z",
      "indicator_types": ["malicious-activity"]
    }
  ]
}
```

---

### LEEF (Log Event Extended Format)

LEEF 2.0 format for IBM QRadar integration.

**Message structure:**

```
LEEF:2.0|DarkHorse InfoSec|HADES|1.0.0|<EventID>|<tab-separated key=value pairs>
```

**Field mapping:**

| LEEF Field    | Description                   | Example                          |
|---------------|-------------------------------|----------------------------------|
| `cat`         | Event category                | `Metadata Threat Detection`      |
| `devTime`     | Event timestamp               | `2026-02-17T14:22:01Z`          |
| `sev`         | Severity (1-10)               | `9`                              |
| `src`         | Source hostname                | `forensic-workstation`           |
| `usrName`     | User running scan             | `analyst`                        |
| `fileName`    | Scanned file name             | `suspicious.jpg`                 |
| `fileHash`    | SHA-256 file hash             | `e3b0c44298fc1c149afb...`        |
| `fileSize`    | File size in bytes            | `245760`                         |
| `scanId`      | HADES scan identifier         | `abc-123`                        |
| `threatScore` | Numeric threat score          | `85.0`                           |
| `threatLevel` | Severity category             | `high`                           |
| `yaraMatches` | YARA rules matched            | `embedded_exe,script_injection`  |
| `iocCount`    | IOC match count               | `2`                              |
| `msg`         | Detection summary             | `Embedded executable detected`   |

**Event categories:**

| Event ID          | Category                    | Description                       |
|-------------------|-----------------------------|-----------------------------------|
| `HADES-SCAN`      | Metadata Threat Detection   | Standard scan result              |
| `HADES-YARA`      | YARA Rule Match             | YARA rule triggered               |
| `HADES-IOC`       | IOC Detection               | Indicator of compromise matched   |
| `HADES-HEURISTIC`| Heuristic Detection          | Heuristic analysis finding        |
| `HADES-MONITOR`  | File Monitor Alert           | Real-time monitoring alert        |

**Example output:**

```
LEEF:2.0|DarkHorse InfoSec|HADES|1.0.0|HADES-YARA|cat=YARA Rule Match	devTime=2026-02-17T14:22:01Z	sev=9	fileName=suspicious.jpg	fileHash=e3b0c44...	threatScore=85.0	threatLevel=high	yaraMatches=embedded_exe	msg=Embedded executable detected in EXIF comment
```

---

### ECS (Elastic Common Schema)

ECS-compatible JSON for direct ingestion into Elasticsearch / Elastic SIEM.

**Field mapping:**

| ECS Field                       | Source                       | Example                          |
|---------------------------------|------------------------------|----------------------------------|
| `@timestamp`                    | Scan timestamp               | `2026-02-17T14:22:01.000Z`      |
| `event.kind`                    | Always `alert`               | `alert`                          |
| `event.category`                | Always `[malware]`           | `["malware"]`                    |
| `event.type`                    | Always `[info]`              | `["info"]`                       |
| `event.severity`                | Threat score (0-100)         | `85`                             |
| `event.module`                  | Always `hades`               | `hades`                          |
| `event.dataset`                 | Always `hades.scan`          | `hades.scan`                     |
| `event.id`                      | Scan ID                      | `a1b2c3d4-...`                   |
| `event.outcome`                 | `success` or `failure`       | `success`                        |
| `file.name`                     | File name                    | `suspicious.jpg`                 |
| `file.size`                     | File size in bytes           | `245760`                         |
| `file.hash.sha256`              | SHA-256 hash                 | `e3b0c44298fc1c149afb...`        |
| `file.mime_type`                | Detected MIME type           | `image/jpeg`                     |
| `threat.indicator.type`         | `file`                       | `file`                           |
| `threat.indicator.confidence`   | Threat level                 | `High`                           |
| `threat.indicator.description`  | Detection summary            | `Embedded executable detected`   |
| `threat.technique.name`         | Detection types              | `["YARA Match","IOC Detection"]` |
| `rule.name`                     | YARA rule names              | `["embedded_exe"]`               |
| `rule.category`                 | Detection category           | `Metadata Threat`                |
| `host.hostname`                 | Scanner hostname             | `forensic-workstation`           |
| `observer.product`              | Always `HADES`               | `HADES`                          |
| `observer.vendor`               | Always `DarkHorse InfoSec`   | `DarkHorse InfoSec`              |
| `observer.version`              | Engine version               | `1.0.0`                          |
| `hades.scan_id`                 | Scan ID (custom field)       | `abc-123`                        |
| `hades.threat_score`            | Numeric threat score         | `85.0`                           |
| `hades.threat_level`            | Severity category            | `high`                           |
| `hades.yara_matches`            | YARA rule match names        | `["embedded_exe"]`               |
| `hades.ioc_match_count`         | IOC match count              | `2`                              |
| `hades.heuristic_findings`      | Heuristic findings list      | `["Suspicious EXIF comment"]`    |

**Example JSON document:**

```json
{
  "@timestamp": "2026-02-17T14:22:01.000Z",
  "event": {
    "kind": "alert",
    "category": ["malware"],
    "type": ["info"],
    "severity": 85,
    "module": "hades",
    "dataset": "hades.scan",
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "outcome": "success"
  },
  "file": {
    "name": "suspicious.jpg",
    "size": 245760,
    "hash": {
      "sha256": "e3b0c44298fc1c149afb..."
    },
    "mime_type": "image/jpeg"
  },
  "threat": {
    "indicator": {
      "type": "file",
      "confidence": "High",
      "description": "Embedded executable detected in EXIF comment"
    },
    "technique": {
      "name": ["YARA Match", "Heuristic Detection"]
    }
  },
  "rule": {
    "name": ["embedded_exe"],
    "category": "Metadata Threat"
  },
  "host": {
    "hostname": "forensic-workstation"
  },
  "observer": {
    "product": "HADES",
    "vendor": "DarkHorse InfoSec",
    "version": "1.0.0"
  },
  "hades": {
    "scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "threat_score": 85.0,
    "threat_level": "high",
    "yara_matches": ["embedded_exe"],
    "ioc_match_count": 2,
    "heuristic_findings": ["Suspicious EXIF comment with encoded data"]
  }
}
```

---

## Platform Setup Guides

### Splunk

#### Syslog ingestion

Configure a UDP syslog input in `inputs.conf`:

```ini
[udp://514]
sourcetype = hades:syslog
index = security
connection_host = dns

[monitor:///var/log/hades/syslog.log]
sourcetype = hades:syslog
index = security
```

Forward HADES output to Splunk:

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format syslog --siem-output syslog --siem-target splunk-hec.corp:514
```

#### File-based ingestion (CEF)

```ini
[monitor:///var/log/hades/cef.log]
sourcetype = cef
index = security
```

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format cef --siem-output file --siem-target /var/log/hades/cef.log
```

#### Sample SPL queries

**High-severity HADES alerts:**

```spl
index=security sourcetype="hades:syslog" threatLevel="high" OR threatLevel="critical"
| table _time, fileName, threatScore, threatLevel, yaraMatches
| sort -threatScore
```

**HADES scan activity over time:**

```spl
index=security sourcetype="hades:syslog"
| timechart span=1h count by threatLevel
```

**Top YARA rule matches:**

```spl
index=security sourcetype="hades:syslog" yaraMatches!=""
| makemv delim="," yaraMatches
| mvexpand yaraMatches
| top yaraMatches limit=20
```

**Files with script injection indicators:**

```spl
index=security sourcetype="hades:syslog" (yaraMatches="*script*" OR yaraMatches="*injection*" OR msg="*script*")
| table _time, fileName, threatScore, yaraMatches, msg
```

---

### IBM QRadar

#### LEEF log source configuration

1. Navigate to **Admin > Log Sources > Add**.
2. Set **Log Source Type** to `Universal LEEF`.
3. Set **Protocol Type** to `Syslog` (UDP 514).
4. Set **Log Source Identifier** to `HADES`.
5. In **Extension**, map these custom event properties:

| Custom Property  | LEEF Key       | Property Type |
|------------------|----------------|---------------|
| Threat Score     | `threatScore`  | Numeric       |
| Threat Level     | `threatLevel`  | String        |
| YARA Matches     | `yaraMatches`  | String        |
| Scan ID          | `scanId`       | String        |
| File Hash        | `fileHash`     | String        |

Forward HADES output to QRadar:

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format leef --siem-output syslog --siem-target qradar.corp:514
```

#### Sample AQL queries

**High-severity metadata threats:**

```sql
SELECT "fileName", "threatScore", "threatLevel", "yaraMatches"
FROM events
WHERE LOGSOURCENAME(logsourceid) = 'HADES'
  AND "threatScore" > 70
ORDER BY "threatScore" DESC
LAST 24 HOURS
```

**YARA match breakdown:**

```sql
SELECT "yaraMatches", COUNT(*) AS match_count
FROM events
WHERE LOGSOURCENAME(logsourceid) = 'HADES'
  AND "yaraMatches" IS NOT NULL
GROUP BY "yaraMatches"
ORDER BY match_count DESC
LAST 7 DAYS
```

---

### ArcSight

#### CEF connector configuration

1. Add a **SmartConnector** for Syslog CEF.
2. Set the connector to listen on UDP 514.
3. Configure the source filter to match `Device Vendor = "DarkHorse InfoSec"` and `Device Product = "HADES"`.

Forward HADES output to ArcSight:

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format cef --siem-output syslog --siem-target arcsight.corp:514
```

#### Filter and rule examples

**Active Channel filter for critical threats:**

```
Device Vendor = "DarkHorse InfoSec" AND
Device Product = "HADES" AND
Device Custom Number 1 >= 80
```

**Correlation rule: Multiple high-severity scans from same host:**

```
Condition: COUNT(events) >= 5 within 1 hour
  WHERE Device Vendor = "DarkHorse InfoSec"
    AND Device Product = "HADES"
    AND Device Custom Number 1 >= 70
  GROUP BY Source Address
Action: Send notification to SOC team
```

---

### Elastic / ELK Stack

#### Filebeat configuration

Configure Filebeat to ingest HADES ECS JSON output:

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/hades/ecs.json
    json.keys_under_root: true
    json.add_error_key: true
    json.overwrite_keys: true

output.elasticsearch:
  hosts: ["https://elasticsearch.corp:9200"]
  index: "hades-scans-%{+yyyy.MM.dd}"
  username: "elastic"
  password: "${ES_PASSWORD}"

setup.template.name: "hades-scans"
setup.template.pattern: "hades-scans-*"
```

Forward HADES output to file for Filebeat:

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format ecs --siem-output file --siem-target /var/log/hades/ecs.json
```

#### Index template

```json
{
  "index_patterns": ["hades-scans-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "event.severity": { "type": "integer" },
        "file.name": { "type": "keyword" },
        "file.size": { "type": "long" },
        "file.hash.sha256": { "type": "keyword" },
        "hades.threat_score": { "type": "float" },
        "hades.threat_level": { "type": "keyword" },
        "hades.yara_matches": { "type": "keyword" },
        "hades.ioc_match_count": { "type": "integer" },
        "rule.name": { "type": "keyword" }
      }
    }
  }
}
```

#### Sample Kibana / KQL queries

**High-severity threats:**

```
hades.threat_level: "high" or hades.threat_level: "critical"
```

**Scans with YARA matches:**

```
hades.yara_matches: * and hades.threat_score >= 50
```

**Script injection indicators:**

```
hades.yara_matches: (*script* or *injection*) or threat.indicator.description: *script*
```

**Elasticsearch aggregation for trend analysis:**

```json
{
  "aggs": {
    "scans_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h"
      },
      "aggs": {
        "avg_threat_score": {
          "avg": { "field": "hades.threat_score" }
        },
        "threat_levels": {
          "terms": { "field": "hades.threat_level" }
        }
      }
    }
  }
}
```

---

### Microsoft Sentinel

#### CEF connector via Log Analytics agent

1. In the Azure portal, navigate to **Microsoft Sentinel > Data connectors > Common Event Format (CEF)**.
2. Install the Log Analytics agent on the HADES host or a dedicated syslog forwarder.
3. Configure rsyslog to forward CEF messages to the agent:

```
# /etc/rsyslog.d/95-hades-cef.conf
if $programname == 'HADES' then @@127.0.0.1:25226
```

4. Forward HADES output to the local syslog:

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format cef --siem-output syslog --siem-target 127.0.0.1:514
```

#### KQL queries

**High-severity HADES alerts:**

```kql
CommonSecurityLog
| where DeviceVendor == "DarkHorse InfoSec" and DeviceProduct == "HADES"
| where toint(DeviceCustomNumber1) >= 70
| project TimeGenerated, FileName, DeviceCustomNumber1, DeviceCustomString3, Message
| sort by toint(DeviceCustomNumber1) desc
```

**Scan volume over time:**

```kql
CommonSecurityLog
| where DeviceVendor == "DarkHorse InfoSec" and DeviceProduct == "HADES"
| summarize ScanCount = count() by bin(TimeGenerated, 1h), DeviceCustomString3
| render timechart
```

**Top YARA rule matches:**

```kql
CommonSecurityLog
| where DeviceVendor == "DarkHorse InfoSec" and DeviceProduct == "HADES"
| where isnotempty(DeviceCustomString2)
| extend YARARule = split(DeviceCustomString2, ",")
| mv-expand YARARule
| summarize MatchCount = count() by tostring(YARARule)
| top 20 by MatchCount
```

**Files with script injection:**

```kql
CommonSecurityLog
| where DeviceVendor == "DarkHorse InfoSec" and DeviceProduct == "HADES"
| where DeviceCustomString2 contains "script" or Message contains "script injection"
| project TimeGenerated, FileName, DeviceCustomNumber1, DeviceCustomString2, Message
```

---

### Splunk HEC (HTTP Event Collector)

HADES includes a native Splunk HEC connector for direct event delivery without syslog or file intermediaries.

**Configuration in `hades_config.json`:**

```json
{
  "siem": {
    "splunk_hec": {
      "url": "https://splunk.corp:8088",
      "token": "your-hec-token",
      "index": "hades-scans",
      "source": "hades",
      "sourcetype": "hades:scan",
      "verify_ssl": true
    }
  }
}
```

| Field        | Type   | Required | Description                                |
|--------------|--------|----------|--------------------------------------------|
| `url`        | string | Yes      | Splunk HEC endpoint URL                    |
| `token`      | string | Yes      | HEC authentication token                   |
| `index`      | string | No       | Target index (default: `main`)             |
| `source`     | string | No       | Event source identifier                    |
| `sourcetype` | string | No       | Splunk sourcetype                          |
| `verify_ssl` | bool   | No       | Verify TLS certificates (default: `true`)  |

**CLI usage:**

```bash
python core/hades_enhanced_cli.py -r /evidence/ --siem-auto-forward
```

**Test the connection:**

```bash
curl -X POST http://localhost:8666/api/v1/siem/test \
    -H "X-API-Key: your-api-key"
```

---

### Elasticsearch Native

Direct Elasticsearch ingestion via the REST API, supporting single-document and bulk indexing.

**Configuration in `hades_config.json`:**

```json
{
  "siem": {
    "elasticsearch": {
      "url": "https://elasticsearch.corp:9200",
      "index_pattern": "hades-scans-{date}",
      "api_key": "your-es-api-key",
      "verify_ssl": true
    }
  }
}
```

| Field           | Type   | Required | Description                                     |
|-----------------|--------|----------|-------------------------------------------------|
| `url`           | string | Yes      | Elasticsearch cluster URL                       |
| `index_pattern` | string | No       | Index name pattern; `{date}` resolves to `YYYY.MM.DD` |
| `api_key`       | string | No       | API key for authentication                      |
| `verify_ssl`    | bool   | No       | Verify TLS certificates (default: `true`)       |

The connector uses the `/_doc` endpoint for single events and the `/_bulk` API for batch delivery.

**CLI usage:**

```bash
python core/hades_enhanced_cli.py -r /evidence/ --siem-auto-forward
```

---

### Microsoft Sentinel Native (Log Analytics Data Collector API)

Direct delivery to Microsoft Sentinel via the Log Analytics Data Collector API with HMAC-SHA256 authentication.

**Configuration in `hades_config.json`:**

```json
{
  "siem": {
    "sentinel": {
      "workspace_id": "your-workspace-id",
      "shared_key": "your-shared-key-base64",
      "log_type": "HADESScan"
    }
  }
}
```

| Field          | Type   | Required | Description                                |
|----------------|--------|----------|--------------------------------------------|
| `workspace_id` | string | Yes      | Log Analytics workspace ID                 |
| `shared_key`   | string | Yes      | Primary or secondary key (Base64-encoded)  |
| `log_type`     | string | No       | Custom log table name (default: `HADESScan`) |

The connector constructs a `SharedKey` authorization header with an HMAC-SHA256 signature per Microsoft's Data Collector API specification.

**CLI usage:**

```bash
python core/hades_enhanced_cli.py -r /evidence/ --siem-auto-forward
```

---

## Troubleshooting

### Connection timeouts / refused connections

- Verify the target host and port are reachable from the HADES host: `curl -v https://siem.corp:8088/services/collector/health`
- Check firewall rules allow outbound traffic to the SIEM port.
- For TCP/TLS transports, ensure the SIEM receiver is actively listening.

### Authentication failures

- **Splunk HEC:** Verify the token is active in Splunk under **Settings > Data Inputs > HTTP Event Collector**. Tokens can be disabled or expire.
- **Elasticsearch:** Confirm the API key has write permissions to the target index. Regenerate if expired.
- **Microsoft Sentinel:** Ensure the shared key is the Base64-encoded primary or secondary key from **Log Analytics workspace > Agents > Log Analytics agent instructions**.

### Format parsing errors

- If the SIEM platform does not parse fields correctly, verify the expected format matches the configured `--siem-format`.
- For CEF, ensure the ArcSight SmartConnector or Sentinel CEF parser is configured to recognize `Device Vendor = "DarkHorse InfoSec"`.
- For LEEF, ensure QRadar has a Universal LEEF log source configured.

### High queue backlog

- Check the forwarding queue status: `GET /api/v1/siem/status`
- A growing `queue_depth` indicates the SIEM target is not consuming events fast enough.
- Consider adding additional targets for load distribution, or increasing the SIEM receiver's ingestion capacity.

### TLS/SSL certificate errors

- Set `verify_ssl: false` in the connector configuration to bypass certificate validation (for testing only).
- For production, install the SIEM server's CA certificate in the system trust store.
- Use `--siem-target` with the correct hostname that matches the server certificate's CN or SAN.

---

## REST API

The HADES REST API includes endpoints for SIEM export and configuration.

### POST /api/v1/export

Export a stored scan result in a SIEM format.

**Request body (JSON):**

| Field       | Type   | Required | Description                                     |
|-------------|--------|----------|-------------------------------------------------|
| `scan_id`   | string | Yes      | UUID of a previously stored scan result         |
| `format`    | string | Yes      | Output format: `cef`, `syslog`, `stix`, `leef`, `ecs` |

**Example request:**

```bash
curl -X POST http://localhost:8666/api/v1/export \
    -H "X-API-Key: your-api-key-here" \
    -H "Content-Type: application/json" \
    -d '{"scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890", "format": "stix"}'
```

**Example response (200):**

```json
{
  "scan_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "format": "stix",
  "exported_at": "2026-02-17T14:30:00+00:00",
  "data": {
    "type": "bundle",
    "id": "bundle--...",
    "objects": [ ]
  }
}
```

### GET /api/v1/siem/config

Retrieve the current SIEM forwarding configuration.

**Example request:**

```bash
curl -H "X-API-Key: your-api-key-here" \
    http://localhost:8666/api/v1/siem/config
```

**Example response (200):**

```json
{
  "enabled": true,
  "default_format": "cef",
  "targets": [
    {
      "type": "syslog",
      "host": "10.0.0.50",
      "port": 514,
      "protocol": "udp",
      "format": "cef"
    }
  ],
  "auto_forward_scans": true,
  "auto_forward_monitor_alerts": true,
  "min_severity_to_forward": "medium"
}
```

### PUT /api/v1/siem/config

Update the SIEM forwarding configuration.

**Example request:**

```bash
curl -X PUT http://localhost:8666/api/v1/siem/config \
    -H "X-API-Key: your-api-key-here" \
    -H "Content-Type: application/json" \
    -d '{
      "enabled": true,
      "default_format": "cef",
      "targets": [
        {
          "type": "syslog",
          "host": "10.0.0.50",
          "port": 514,
          "protocol": "udp",
          "format": "cef"
        }
      ],
      "auto_forward_scans": true,
      "auto_forward_monitor_alerts": true,
      "min_severity_to_forward": "medium"
    }'
```

**Example response (200):**

```json
{
  "message": "SIEM configuration updated",
  "config": {
    "enabled": true,
    "default_format": "cef",
    "targets": [
      {
        "type": "syslog",
        "host": "10.0.0.50",
        "port": 514,
        "protocol": "udp",
        "format": "cef"
      }
    ],
    "auto_forward_scans": true,
    "auto_forward_monitor_alerts": true,
    "min_severity_to_forward": "medium"
  }
}
```

---

## Real-Time Forwarding

The `SIEMForwarder` class supports continuous forwarding of scan results and monitor alerts to configured SIEM targets.

### Configuration in hades_config.json

Add a `siem` section to `cli/hades_config.json`:

```json
{
  "siem": {
    "enabled": true,
    "default_format": "cef",
    "targets": [
      {
        "type": "syslog",
        "host": "10.0.0.50",
        "port": 514,
        "protocol": "udp",
        "format": "cef"
      },
      {
        "type": "file",
        "path": "/var/log/hades/siem.log",
        "format": "ecs"
      },
      {
        "type": "http",
        "url": "https://siem-collector.corp/api/events",
        "format": "ecs",
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    ],
    "auto_forward_scans": true,
    "auto_forward_monitor_alerts": true,
    "min_severity_to_forward": "medium"
  }
}
```

### CLI usage for real-time forwarding

```bash
# Forward scan results to syslog as they complete
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format syslog --siem-output syslog --siem-target 10.0.0.50:514

# Forward monitor alerts via HTTP
python core/hades_enhanced_cli.py --monitor /evidence/incoming \
    --siem-format ecs --siem-output http --siem-target https://siem-collector.corp/api/events

# Write to file for log shipper pickup
python core/hades_enhanced_cli.py --monitor /evidence/incoming \
    --siem-format ecs --siem-output file --siem-target /var/log/hades/ecs.json
```

---

## Example Dashboards

Use these queries as building blocks for SIEM dashboards that visualize HADES scan activity and threat detection.

### High-Severity Metadata Threats

Identify the most dangerous files detected by HADES.

| Platform  | Query / Filter                                                                     |
|-----------|------------------------------------------------------------------------------------|
| Splunk    | `index=security sourcetype="hades:syslog" threatScore>70 \| sort -threatScore`      |
| QRadar    | `SELECT * FROM events WHERE LOGSOURCENAME(logsourceid)='HADES' AND "threatScore">70` |
| Elastic   | `hades.threat_score >= 70` (KQL), sort by `hades.threat_score` desc               |
| Sentinel  | `CommonSecurityLog \| where DeviceVendor=="DarkHorse InfoSec" and toint(DeviceCustomNumber1)>=70` |

### Scan Trend Over Time

Visualize scan activity and threat levels as a time series.

| Platform  | Query / Filter                                                                     |
|-----------|------------------------------------------------------------------------------------|
| Splunk    | `index=security sourcetype="hades:syslog" \| timechart span=1h count by threatLevel` |
| QRadar    | Use a time series chart on HADES events grouped by `threatLevel`                   |
| Elastic   | Date histogram on `@timestamp`, sub-aggregation terms on `hades.threat_level`      |
| Sentinel  | `CommonSecurityLog \| summarize count() by bin(TimeGenerated,1h), DeviceCustomString3 \| render timechart` |

### Top YARA Rule Matches

Discover which YARA rules fire most frequently.

| Platform  | Query / Filter                                                                     |
|-----------|------------------------------------------------------------------------------------|
| Splunk    | `index=security sourcetype="hades:syslog" yaraMatches!="" \| mvexpand yaraMatches \| top yaraMatches` |
| QRadar    | `SELECT "yaraMatches", COUNT(*) FROM events WHERE ... GROUP BY "yaraMatches"`      |
| Elastic   | Terms aggregation on `hades.yara_matches` field                                   |
| Sentinel  | `... \| extend rule=split(DeviceCustomString2,",") \| mv-expand rule \| summarize count() by tostring(rule)` |

### Files with Script Injection

Detect metadata-based script injection attacks.

| Platform  | Query / Filter                                                                     |
|-----------|------------------------------------------------------------------------------------|
| Splunk    | `index=security sourcetype="hades:syslog" (yaraMatches="*script*" OR msg="*injection*")` |
| QRadar    | `SELECT * FROM events WHERE "yaraMatches" LIKE '%script%' OR "msg" LIKE '%injection%'` |
| Elastic   | `hades.yara_matches: *script* or threat.indicator.description: *injection*`        |
| Sentinel  | `... \| where DeviceCustomString2 contains "script" or Message contains "injection"` |
