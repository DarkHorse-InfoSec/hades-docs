# HADES IR Triage & Investigation Guide

A practical guide for SOC analysts and incident responders using HADES to investigate suspicious files.

---

## 1. Understanding Threat Scores

HADES uses an **enterprise two-axis scoring model** inspired by CrowdStrike and Palo Alto Networks. Every file is evaluated on two independent axes:

### Confidence Level

How certain HADES is that the file is malicious.

| Level | Meaning |
|-------|---------|
| **none** | No indicators found. File appears clean. |
| **low** | Minor anomalies detected. Could be benign. |
| **medium** | Multiple suspicious indicators. Warrants investigation. |
| **high** | Strong evidence of malicious intent from multiple independent signals. |
| **confirmed** | Definitive match, known malware signature, confirmed threat intel hit, or multiple high-confidence signals converging. |

### Severity Level

How dangerous the threat is if confirmed.

| Level | Meaning |
|-------|---------|
| **informational** | Metadata oddity or structural note. No security impact. |
| **low** | Minor risk, unusual metadata, low-confidence heuristic. |
| **medium** | Moderate risk, suspicious scripts, encoded content, evasion indicators. |
| **high** | Serious threat, exploits, RATs, stealers, C2 beacons, offensive tools. |
| **critical** | Destructive/Impact-stage threat, ransomware, wipers, destructive payloads. Requires MITRE ATT&CK Impact tactic indicators. |

**Key rule: CRITICAL severity is reserved for Impact-stage threats.** YARA matches and heuristic findings alone cap at HIGH severity. Only the presence of MITRE ATT&CK Impact tactic indicators (ransomware behavior, data destruction, disk wiping) boosts severity to CRITICAL.

### How the Two Axes Work Together

HADES uses a **max-severity model**, not an additive model. The highest individual finding drives severity. Additional corroborating signals boost confidence rather than inflating the score.

Example: A file with an embedded executable (HIGH severity) plus obfuscated JavaScript (HIGH severity) plus auto-open action (MEDIUM severity) does not become CRITICAL. Instead, the severity stays HIGH and the confidence increases to high or confirmed.

### Backward-Compatible Threat Score

The `overall_threat_score` (0-100) and `threat_level` (SAFE/LOW/MEDIUM/HIGH/CRITICAL) fields still appear in all output for backward compatibility. They are derived from the confidence x severity matrix:

| Confidence | Severity | Score | Threat Level |
|------------|----------|-------|--------------|
| confirmed | critical | 100 | CRITICAL |
| high | critical | 92 | CRITICAL |
| confirmed | high | 85 | CRITICAL |
| high | high | 78 | HIGH |
| medium | high | 56 | MEDIUM |
| high | medium | 50 | MEDIUM |

### Action Table

| Severity | Confidence | Action |
|----------|------------|--------|
| **critical** | high/confirmed | **P1: Immediate containment.** Ransomware/wiper confirmed. Isolate host, preserve evidence, activate IR playbook. |
| **high** | high/confirmed | **P2: Escalate immediately.** Confirmed malicious (exploit, RAT, stealer). Scope exposure, block IOCs. |
| **high** | medium | **P2: Investigate within 1 hour.** Strong suspicion. Deep analysis needed to confirm. |
| **high** | low | **P3: Review within 4 hours.** Suspicious characteristics but low certainty. |
| **medium** | any | **P3: Investigate when possible.** Suspicious indicators present but not conclusive. Check source context. |
| **low** | any | **P4: Monitor.** Minor anomalies. Only escalate if part of a larger incident pattern. |
| **informational** | any | **No action.** Metadata note for forensic record. |

### SIEM Alert Fields

HADES scan results now include both legacy and two-axis fields:
- `hades.confidence_level`, none / low / medium / high / confirmed
- `hades.severity_level`, informational / low / medium / high / critical
- `hades.threat_score`, 0-100 (backward compatible)
- `hades.threat_level`, SAFE/LOW/MEDIUM/HIGH/CRITICAL (backward compatible)

---

## 2. Triage Workflow

### Step 1: Initial Scan

```bash
# Single file
hades scan /path/to/suspicious_file.pdf

# Directory of files from an incident
hades scan /evidence/collected_files/ --output /cases/INC-2026-001/scan_results.json

# With deep format analysis (recommended for IR)
hades scan /path/to/file --deep-format --include-metadata -v
```

### Step 2: Interpret Results

**CRITICAL severity + high/confirmed confidence, Impact-Stage Threat (ransomware, wiper)**
1. Do NOT open the file on any production system
2. Activate your IR playbook immediately; this indicates a destructive threat
3. Record the SHA256 hash: `sha256sum <file>`
4. Check if the hash matches known malware: `hades intel lookup --hash <sha256>`
5. Create a case: `hades case create --name "INC-2026-001" --severity critical`
6. Add the evidence: `hades case add-evidence --case INC-2026-001 --file <path>`
7. Proceed to containment (Step 3)

**HIGH severity + high/confirmed confidence, Confirmed Malicious**
1. Review the specific findings in the scan output, check `confidence_level` and `severity_level` fields
2. Look for: embedded executables, suspicious macros, encoded payloads, auto-open actions
3. Cross-reference with threat intel: `hades intel lookup --hash <sha256>`
4. If MITRE Impact indicators emerge: treat as CRITICAL
5. If confirmed non-destructive malware: scope exposure, block IOCs, create case
6. If uncertain: proceed to deep analysis (Step 4)

**HIGH severity + medium/low confidence, Suspicious, Needs Confirmation**
1. The file has characteristics of a serious threat but confidence is not yet high
2. Perform deep analysis (Step 4) to gather more signals
3. Cross-reference with threat intel
4. If confidence increases: escalate per above
5. If confidence does not increase: document and monitor

**MEDIUM severity, Investigate**
1. Review findings, what triggered the score?
2. Common benign causes at this level:
   - Password-protected archives (detected as encrypted container)
   - Files with high entropy (could be legitimate compression)
   - Documents with external references (could be legitimate links)
3. Check if the file is from a known-good source
4. If from untrusted source: proceed to deep analysis (Step 4)
5. If from trusted source: document and close

**LOW severity: Minor Anomalies**
1. Review briefly, usually benign with minor metadata oddities
2. Common causes: unusual EXIF data, GPS coordinates in images, long metadata strings
3. Only escalate if part of a larger incident pattern

### Step 3: Containment

When a CRITICAL or confirmed HIGH file is found:

1. **Isolate the host** where the file was found
2. **Block the hash** across your EDR/firewall:
   ```bash
   hades scan <file> -v | grep "SHA256"
   ```
3. **Search for the file** across your environment:
   ```bash
   hades intel lookup --hash <sha256> --search-scope enterprise
   ```
4. **Export SIEM alert** for correlation:
   ```bash
   hades scan <file> --siem-export --format splunk
   ```
5. **Preserve the evidence chain**:
   ```bash
   hades case add-evidence --case INC-2026-001 --file <path>
   hades case verify --case INC-2026-001  # Verify chain integrity
   ```

### Step 4: Deep Analysis

For files that need further investigation:

```bash
# Full analysis with all engines enabled
hades scan <file> --deep-format --include-metadata --polyglot -v

# Check for specific attack types
hades scan <file> -v | grep -i "macro\|javascript\|auto-open\|embedded\|polyglot"
```

**Key findings to look for:**

| Finding | Indicates | Severity |
|---------|-----------|----------|
| `embedded_javascript` | Script-based attack | HIGH |
| `auto_open_action` | Automatic execution trigger | HIGH |
| `polyglot_detected` | File masquerading as another format | HIGH |
| `embedded_executable` | PE/ELF binary hidden in document | CRITICAL |
| `vba_macro_detected` | Office macro (potentially malicious) | MEDIUM-HIGH |
| `encrypted_archive` | Password-protected container | MEDIUM |
| `suspicious_command` | Shell/PowerShell commands in metadata | HIGH |
| `obfuscated_content` | Deliberately hidden content | MEDIUM-HIGH |

---

## 3. Case Management

### Creating a Case

```bash
# Create a new investigation case
hades case create \
  --name "INC-2026-001" \
  --severity critical \
  --description "Phishing email with malicious PDF attachment"

# Add files as evidence
hades case add-evidence --case INC-2026-001 --file /evidence/phish.pdf
hades case add-evidence --case INC-2026-001 --file /evidence/payload.exe

# List all evidence with integrity status
hades case list --case INC-2026-001
```

### Evidence Chain

HADES maintains a cryptographic evidence chain for each case:
- Every file added gets a SHA256 hash recorded at time of ingestion
- The chain is signed so tampering is detectable
- Export for legal/compliance: `hades case export --case INC-2026-001 --format json`

### Verifying Chain Integrity

```bash
hades case verify --case INC-2026-001
```

This re-hashes all evidence files and compares against the recorded chain. Any modification since ingestion is flagged.

---

## 4. Common Investigation Scenarios

### Scenario A: Phishing Email with PDF Attachment

1. Extract the attachment from the email
2. Scan: `hades scan attachment.pdf --deep-format -v`
3. Look for: `/JS`, `/JavaScript`, `/OpenAction`, `/AA`, embedded URLs
4. If CRITICAL/HIGH: block sender, search mailboxes for same attachment hash
5. Export IOCs for your email gateway block list

### Scenario B: Suspicious Office Document

1. Scan: `hades scan document.docx --deep-format -v`
2. Look for: VBA macros, DDE fields, external template references, embedded OLE objects
3. Check: Is `vba_macro_detected` present? Is `auto_open_action` firing?
4. If macros found: extract and analyze macro code (HADES flags the presence, manual review needed for logic)

### Scenario C: ISO/IMG Disk Image Delivery

1. Scan: `hades scan delivery.iso --deep-format -v`
2. Look for: embedded executables, LNK shortcuts, batch scripts inside the image
3. This is a common MOTW bypass technique, the ISO bypasses Windows security warnings
4. If malicious: search for similar ISO hashes across your environment

### Scenario D: Ransomware Sample Analysis

1. **Do NOT execute.** Scan only: `hades scan sample.exe -v`
2. Look for: high entropy (packed/encrypted), suspicious imports, known ransomware patterns
3. Record: SHA256 hash, YARA rule matches, any C2 indicators
4. Cross-reference: `hades intel lookup --hash <sha256>`
5. If novel: submit hash to MalwareBazaar/VirusTotal for community analysis

### Scenario E: Supply Chain Artifact Investigation

1. Scan all artifacts from the suspected supply chain: `hades scan /path/to/artifacts/ -v`
2. Compare scores against a known-good baseline of the same software
3. Look for: unexpected embedded executables, modified metadata timestamps, injected code in libraries
4. Flag anything scoring above your baseline as potentially tampered

---

## 5. SIEM Integration

### Forwarding Alerts

```bash
# Forward to Splunk
hades scan <file> --siem-export --format splunk --siem-url https://splunk.internal:8088/services/collector

# Forward to Elastic
hades scan <file> --siem-export --format elastic --siem-url https://elastic.internal:9200

# Forward to Microsoft Sentinel
hades scan <file> --siem-export --format sentinel
```

### Alert Fields

HADES SIEM alerts include:
- `hades.confidence_level`, none/low/medium/high/confirmed (two-axis model)
- `hades.severity_level`, informational/low/medium/high/critical (two-axis model)
- `hades.threat_score`, 0-100 score (backward compatible, derived from confidence x severity)
- `hades.threat_level`, SAFE/LOW/MEDIUM/HIGH/CRITICAL (backward compatible)
- `hades.yara_matches[]`, list of triggered YARA rules
- `hades.heuristic_findings[]`, list of heuristic detections
- `hades.file.sha256`, file hash for correlation
- `hades.file.type`, detected file type
- `hades.mitre_techniques[]`, mapped MITRE ATT&CK techniques

### Recommended SIEM Alerts

| Alert Name | Condition | Priority |
|------------|-----------|----------|
| HADES Critical Detection | `hades.severity_level = "critical"` | P1, Immediate (Impact-stage threat) |
| HADES High Confirmed | `hades.severity_level = "high" AND hades.confidence_level IN ("high","confirmed")` | P2, 1 hour |
| HADES Encrypted Container | `hades.heuristic_findings contains "encrypted"` | P3, Review |
| HADES Polyglot File | `hades.heuristic_findings contains "polyglot"` | P2, 1 hour |

---

## 6. Quick Reference

### CLI Commands for IR

| Command | Purpose |
|---------|---------|
| `hades scan <path>` | Scan file or directory |
| `hades scan <path> --deep-format -v` | Full analysis with verbose output |
| `hades case create --name X` | Create investigation case |
| `hades case add-evidence --case X --file Y` | Add evidence to case |
| `hades case verify --case X` | Verify evidence chain integrity |
| `hades case export --case X` | Export case for reporting |
| `hades intel lookup --hash X` | Check threat intel for a hash |
| `hades monitor /path/` | Real-time monitoring of a directory |
| `hades update check` | Check for HADES updates |

### File Types HADES Detects

HADES analyzes 45+ file types, including:

**Documents:** PDF, DOC/DOCX/DOCM, XLS/XLSX/XLSM, PPT/PPTX, RTF, OneNote
**Executables:** EXE, DLL, SYS, OCX, ELF, Mach-O, APK
**Scripts:** JS, VBS/VBE, BAT/CMD, PS1, HTA, AppleScript
**Containers:** ZIP, 7Z, RAR, TAR, ISO, IMG, VHD/VHDX, DMG, MSI, CHM
**Web:** HTML, HTM, SVG
**Shortcuts:** LNK
