# HADES Enterprise SIEM Integration Guide

## Overview

This guide covers platform-specific configuration for connecting HADES to enterprise SIEM systems. HADES exports scan results and threat alerts in five industry-standard formats and delivers them via syslog, file, HTTP, or stdout. Each section below walks through the receiver-side setup for a specific platform, paired with the corresponding HADES CLI flags and API configuration.

For full format specifications and field-level reference, see `docs/siem_integration_guide.md`.

---

## Splunk (CEF via HTTP Event Collector)

HADES produces ArcSight CEF revision 25 messages that Splunk can ingest natively through its HTTP Event Collector (HEC).

### 1. Enable HEC in Splunk

In Splunk Web, navigate to **Settings > Data Inputs > HTTP Event Collector** and create a new token. Record the token value; you will pass it as the `Authorization` header.

| Setting        | Recommended Value            |
|----------------|------------------------------|
| Index          | `threat_intel` or `hades`    |
| Source type    | `cef`                        |
| Default source | `hades-scanner`              |

### 2. Configure props.conf

Add the following stanza to `$SPLUNK_HOME/etc/apps/search/local/props.conf` so Splunk parses CEF headers and extension keys correctly:

```ini
[cef]
SHOULD_LINEMERGE = false
TIME_FORMAT = %s%3N
MAX_TIMESTAMP_LOOKAHEAD = 13
TRANSFORMS-cef = cef_header, cef_extension
```

### 3. Configure transforms.conf

Add corresponding extraction rules in `transforms.conf`:

```ini
[cef_header]
REGEX = CEF:(?<cef_version>\d+)\|(?<cef_vendor>[^|]*)\|(?<cef_product>[^|]*)\|(?<cef_device_version>[^|]*)\|(?<cef_sig_id>[^|]*)\|(?<cef_name>[^|]*)\|(?<cef_severity>[^|]*)
FORMAT = cef_version::$1 cef_vendor::$2 cef_product::$3 cef_device_version::$4 cef_sig_id::$5 cef_name::$6 cef_severity::$7

[cef_extension]
REGEX = \|(\S+=\S+.*)$
FORMAT = cef_extension::$1
```

### 4. HADES CLI: Forward to Splunk HEC

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format cef \
    --siem-output http \
    --siem-target https://splunk.internal:8088/services/collector/event \
    --siem-header "Authorization: Splunk <HEC_TOKEN>"
```

### 5. HADES API Configuration

In `cli/hades_config.json`, add a SIEM target entry:

```json
{
  "siem": {
    "enabled": true,
    "default_format": "cef",
    "auto_forward_scans": true,
    "targets": [
      {
        "type": "http",
        "url": "https://splunk.internal:8088/services/collector/event",
        "format": "cef",
        "headers": {
          "Authorization": "Splunk <HEC_TOKEN>"
        }
      }
    ]
  }
}
```

---

## Elastic / ELK Stack (ECS via Filebeat)

HADES exports scan results in Elastic Common Schema (ECS) 8.x JSON format. The recommended ingestion path is writing ECS JSON to a log file and shipping it with Filebeat.

### 1. HADES CLI: Write ECS to File

```bash
python core/hades_enhanced_cli.py --monitor /evidence/ \
    --siem-format ecs \
    --siem-output file \
    --siem-target /var/log/hades/ecs.json
```

### 2. Filebeat Configuration

Add a log input to `filebeat.yml` that watches the HADES output file:

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/hades/ecs.json
    json.keys_under_root: true
    json.add_error_key: true
    json.message_key: message
    fields:
      event.module: hades
      event.dataset: hades.scan
    fields_under_root: true

output.elasticsearch:
  hosts: ["https://elasticsearch.internal:9200"]
  index: "hades-scans-%{+yyyy.MM.dd}"
  username: "${ES_USERNAME}"
  password: "${ES_PASSWORD}"
```

### 3. Index Template

Create an index template so Elasticsearch maps HADES-specific fields correctly:

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
        "event.module":    { "type": "keyword" },
        "event.category":  { "type": "keyword" },
        "event.kind":      { "type": "keyword" },
        "event.severity":  { "type": "integer" },
        "event.outcome":   { "type": "keyword" },
        "file.name":       { "type": "keyword" },
        "file.hash.sha256":{ "type": "keyword" },
        "file.size":       { "type": "long" },
        "threat.indicator.type": { "type": "keyword" },
        "threat.technique.id":   { "type": "keyword" },
        "threat.technique.name": { "type": "keyword" },
        "hades.scan_id":         { "type": "keyword" },
        "hades.threat_score":    { "type": "float" },
        "hades.threat_level":    { "type": "keyword" },
        "hades.yara_matches":    { "type": "keyword" },
        "hades.ioc_count":       { "type": "integer" }
      }
    }
  }
}
```

Apply the template via the Elasticsearch API:

```bash
curl -X PUT "https://elasticsearch.internal:9200/_index_template/hades-scans" \
    -H "Content-Type: application/json" \
    -d @hades_index_template.json
```

---

## Microsoft Sentinel (STIX via Log Analytics)

HADES generates STIX 2.1 JSON bundles that can be ingested into Microsoft Sentinel through the Log Analytics Data Collector API.

### 1. HADES CLI: Export STIX

```bash
python core/hades_enhanced_cli.py -r /evidence/ \
    --siem-format stix \
    --siem-output http \
    --siem-target "https://<WORKSPACE_ID>.ods.opinsights.azure.com/api/logs?api-version=2016-04-01"
```

### 2. Log Analytics Workspace Setup

In the Azure portal, navigate to **Log Analytics workspace > Agents management** and record:

| Value          | Where to Find                                     |
|----------------|----------------------------------------------------|
| Workspace ID   | Log Analytics workspace > Properties                |
| Primary Key    | Log Analytics workspace > Agents management > Keys  |

### 3. Custom Log Type

Configure a custom log type named `HADES_STIX_CL` in Sentinel. STIX bundle fields map as follows:

| STIX Field              | Sentinel Column           |
|-------------------------|---------------------------|
| `objects[].type`        | `stix_object_type_s`      |
| `objects[].name`        | `stix_name_s`             |
| `objects[].pattern`     | `stix_pattern_s`          |
| `objects[].confidence`  | `stix_confidence_d`       |
| `objects[].created`     | `TimeGenerated`           |

### 4. HADES Config for Sentinel

```json
{
  "siem": {
    "enabled": true,
    "default_format": "stix",
    "auto_forward_scans": true,
    "targets": [
      {
        "type": "http",
        "url": "https://<WORKSPACE_ID>.ods.opinsights.azure.com/api/logs?api-version=2016-04-01",
        "format": "stix",
        "headers": {
          "Log-Type": "HADES_STIX",
          "Authorization": "SharedKey <WORKSPACE_ID>:<BASE64_SIGNATURE>"
        }
      }
    ]
  }
}
```

---

## IBM QRadar (LEEF via Syslog)

HADES produces IBM LEEF 2.0 messages that QRadar ingests through its syslog receiver. QRadar auto-discovers LEEF-formatted events and maps them to its internal schema.

### 1. HADES CLI: Forward LEEF to QRadar

```bash
python core/hades_enhanced_cli.py --monitor /evidence/ \
    --siem-format leef \
    --siem-output syslog \
    --siem-target 10.0.0.100:514
```

### 2. QRadar Log Source Configuration

In QRadar Console, navigate to **Admin > Log Sources** and create a new log source:

| Setting            | Value                           |
|--------------------|---------------------------------|
| Log Source Type     | Universal LEEF                 |
| Protocol            | Syslog                        |
| Log Source Identifier | IP of the HADES host         |
| Coalescing          | Disabled                      |

### 3. DSM Configuration Hints

QRadar's Universal LEEF DSM auto-parses the LEEF header. HADES populates these LEEF keys:

| LEEF Key   | QRadar Normalized Field | Description               |
|------------|-------------------------|---------------------------|
| `devTime`  | Start Time              | Scan timestamp            |
| `cat`      | Event Category          | Detection type            |
| `sev`      | Severity                | Threat severity (1-10)    |
| `src`      | Source IP/Host          | Scanner hostname          |
| `usrName`  | Username                | Scanning user             |
| `fileName` | File Name               | Scanned file              |
| `fileHash` | File Hash               | SHA-256 hash              |

If custom fields (e.g., `threatScore`, `yaraMatches`) do not auto-map, create custom event properties in **Admin > Custom Event Properties** using the LEEF key names.

### 4. HADES Config for QRadar

```json
{
  "siem": {
    "enabled": true,
    "default_format": "leef",
    "auto_forward_alerts": true,
    "min_severity_to_forward": 4,
    "targets": [
      {
        "type": "syslog",
        "host": "10.0.0.100",
        "port": 514,
        "format": "leef"
      }
    ]
  }
}
```

---

## Generic Syslog (RFC 5424)

For any syslog-compatible receiver (rsyslog, syslog-ng, Graylog, Fluentd), HADES generates fully compliant RFC 5424 messages.

### 1. HADES CLI: UDP Syslog Forwarding

```bash
python core/hades_enhanced_cli.py --monitor /evidence/ \
    --siem-format syslog \
    --siem-output syslog \
    --siem-target 10.0.0.50:514
```

### 2. Facility and Severity Mapping

HADES uses facility `local0` (numeric 16) for all messages. Severity is mapped from the threat score:

| Threat Score | Syslog Severity    | Numeric |
|--------------|--------------------|---------|
| 9-10         | Alert              | 1       |
| 7-8          | Critical           | 2       |
| 4-6          | Warning            | 4       |
| 1-3          | Notice             | 5       |
| 0            | Informational      | 6       |

### 3. rsyslog Receiver Example

Add the following to `/etc/rsyslog.d/50-hades.conf` on the receiving host:

```
# Accept UDP syslog on port 514
module(load="imudp")
input(type="imudp" port="514")

# Route HADES events to a dedicated log file
if $app-name == 'HADES' then /var/log/hades/siem.log
& stop
```

### 4. syslog-ng Receiver Example

```
source s_hades {
    udp(port(514));
};

filter f_hades {
    program("HADES");
};

destination d_hades {
    file("/var/log/hades/siem.log");
};

log {
    source(s_hades);
    filter(f_hades);
    destination(d_hades);
};
```

---

## Real-Time Monitoring Integration

All SIEM configurations above work with the HADES file monitor for continuous, real-time alert forwarding. Start a monitored session with SIEM output enabled:

```bash
python core/hades_enhanced_cli.py --monitor /watched/directory \
    --siem-format cef \
    --siem-output syslog \
    --siem-target 10.0.0.50:514 \
    --alert-threshold 5
```

The `--alert-threshold` flag sets the minimum threat score (0-10) that triggers a SIEM event. Files scoring below this threshold are logged locally but not forwarded.

---

## API-Driven SIEM Configuration

When running HADES as an API server, SIEM forwarding is configured in `cli/hades_config.json` and activated at startup:

```bash
python core/hades_enhanced_cli.py --serve --port 8666
```

The API server reads the `siem` section of the config file and initializes the `SIEMForwarder` background thread, which queues and delivers events asynchronously. Multiple targets with different formats can be configured simultaneously.

---

## Troubleshooting

| Symptom                            | Likely Cause                         | Resolution                                  |
|------------------------------------|--------------------------------------|----------------------------------------------|
| No events arriving in SIEM         | `siem.enabled` is `false` in config  | Set `"enabled": true` in `hades_config.json` |
| Events arrive but fields unmapped  | Missing index template or DSM config | Apply the platform-specific template above   |
| Partial events or truncation       | UDP packet size limit (65535 bytes)  | Switch to HTTP or TCP delivery               |
| High-volume event loss             | Forwarder queue overflow             | Increase `forwarder.queue_size` in config    |

---

*DarkHorse Information Security LLC*
