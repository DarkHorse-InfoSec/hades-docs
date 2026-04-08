# Web Dashboard Guide

The HADES web dashboard is a self-contained single-page application served by the FastAPI server. It provides a browser-based interface for scanning files, monitoring directories, managing cases, viewing audit trails, and checking system status.

## Accessing the Dashboard

1. Start the API server:

    ```bash
    hades-server --port 8666
    ```

2. Open your browser to:

    ```
    http://localhost:8666/dashboard/
    ```

3. Enter your API key when prompted (default: `your-api-key-here`).

The API key is stored in `sessionStorage` and used for all subsequent requests. It is cleared when you close the browser tab.

!!! warning "Replace the default API key"
    The default key `your-api-key-here` is for development only. Set a strong, unique key via the `HADES_API_KEY` environment variable or in `cli/hades_config.json` before any production use.

---

## Scan View

The Scan view is the primary interface for uploading and scanning files.

### Drag-and-Drop Upload

1. Navigate to **Scan** in the sidebar
2. Drag one or more files onto the upload area, or click to browse
3. Files are uploaded and scanned immediately
4. Results appear below the upload area with:
    - Filename and file size
    - Threat score badge (color-coded by severity)
    - Detection findings grouped by engine (YARA, heuristic, ML, deep format)
    - Scan ID for later retrieval

### Batch Scanning

Upload multiple files at once. The dashboard sends them as a batch scan request and displays individual results for each file.

---

## Monitor View

The Monitor view controls the real-time file monitoring engine.

### Starting a Monitor

1. Navigate to **Monitor** in the sidebar
2. Enter one or more directory paths to watch
3. Configure the alert threshold (minimum threat score for alerts)
4. Optionally enter a webhook URL for external notifications
5. Click **Start Monitor**

### Live Status

While the monitor is running, the dashboard shows:

- **Status**: Running/Stopped indicator
- **Files Scanned**: Total count of files processed
- **Alerts Generated**: Count of files exceeding the alert threshold
- **Last Alert**: Details of the most recent alert including filename, threat score, and findings

### Stopping the Monitor

Click **Stop Monitor** to halt file watching. The monitor status and alert counts are preserved until a new monitor session is started.

---

## Cases View

The Cases view provides investigation case management with scan linking.

### Creating a Case

1. Navigate to **Cases** in the sidebar
2. Click **New Case**
3. Enter a case name (e.g., "IR-2026-0042 Phishing Campaign")
4. The case ID is generated and displayed

### Case Detail

Click a case to view its detail page:

- **Linked Scans**: Scans associated with this case, with threat scores and dates
- **Analyst Notes**: Free-text notes added by analysts
- **Timeline**: Chronological view of case activity

### Linking Scans

When scanning via the API or CLI with `--case-id`, scans are automatically linked to the case. From the dashboard, use the case detail page to view all linked scans.

---

## Audit View

The Audit view displays the SHA-256 hash-chained audit log.

### Filtering

Filter audit entries by:

- **Action**: scan, sanitize, case_create, case_update, evidence_export, etc.
- **Actor**: The API key or user that performed the action
- **Date Range**: Start and end date for the time period

### Chain Verification

The audit log uses hash chaining where each entry's hash includes the previous entry's hash. The dashboard shows chain verification status, confirming whether the log has been tampered with.

---

## System View

The System view shows server health and configuration.

### Health Status

- **Server Status**: Overall health (`healthy` or `degraded`)
- **Uptime**: How long the server has been running
- **Engine Status**: Availability of ExifScanner, EnhancedDetectionEngine, ML engine, and plugin system

### Configuration

Displays current server configuration including:

- API version
- Loaded YARA rules count
- Plugin count
- SIEM forwarding status
- Database backend

---

## ATT&CK View (v0.6.0)

When MITRE ATT&CK mapping is enabled, the ATT&CK view shows:

- **Technique Matrix**: Visual grid of MITRE ATT&CK techniques detected across all scans
- **Technique Details**: Click a technique to see which files triggered it
- **Tactic Breakdown**: Findings grouped by MITRE tactic (Execution, Persistence, Defense Evasion, etc.)

---

## Rule Builder View (v0.6.0)

The Rule Builder provides a browser-based interface for creating YARA rules:

1. Select a template (metadata injection, steganography, polyglot, encoded payloads)
2. Customize rule name, strings, and conditions
3. Test the rule against uploaded samples
4. Save to the rules directory for automatic loading

---

## Analytics View (v0.6.0)

The Analytics view displays scan statistics and trends:

- **Scan Volume**: Total scans over time with hourly/daily/weekly aggregation
- **Threat Distribution**: Breakdown of threat levels across all scans
- **Top File Types**: Most commonly scanned file types
- **Detection Engine Stats**: Hit rates for YARA, heuristic, ML, and deep format engines

---

## Playbook View (v0.7.1)

The Playbook view manages event-driven response automation playbooks.

### Playbook Table

The main table lists all configured playbooks with:

- **Name**: Playbook display name
- **Trigger**: Event type that activates the playbook (scan_result, monitor_alert, email_scan, campaign_detected)
- **Status**: Enabled/disabled toggle switch
- **Actions**: Manual trigger button

### Manual Trigger

Click the **Trigger** button on any playbook to execute it manually with a test event. Useful for verifying action chains work correctly.

### Execution History

Below the playbook table, a timeline shows recent executions with:

- Playbook name and execution ID
- Status badge (completed, failed, partial_failure)
- Trigger event type
- Timestamp
- Action results summary

---

## Alert Dashboard (v0.7.1)

The Alert Dashboard provides real-time visibility into playbook execution.

### Summary Stats

Four stat cards at the top show:

- **Total Executions**: Lifetime count of playbook runs
- **Success Rate**: Percentage of fully successful executions
- **Most Fired**: The playbook that has been triggered most often
- **Avg Actions**: Average number of actions per execution

### Filters

Filter the execution feed by:

- **Status**: All, Completed, Failed, Partial Failure
- **Playbook**: Dropdown to filter by specific playbook

### Live Feed

The execution timeline shows playbook runs in reverse chronological order. Each entry displays:

- Playbook name
- Status badge (color-coded: green=completed, red=failed, amber=partial)
- Trigger event type
- Timestamp
- Expandable detail showing individual action results with success/failure status

### Real-Time Updates

When connected via WebSocket, new playbook executions appear at the top of the feed automatically with a toast notification. No page refresh required.

---

## Technical Notes

The dashboard is built with vanilla HTML, CSS, and JavaScript -- no external CDN dependencies or build step required. It uses:

- **Hash-based routing**: Navigation via URL hash fragments (`#scan`, `#monitor`, `#cases`, `#audit`, `#system`, `#mitre`, `#rules`, `#analytics`, `#playbooks`, `#alerts`)
- **Dark theme**: Color scheme based on `#1a1a2e` with CSS variables for threat level colors
- **WebSocket**: Live updates from the server for monitor alerts and scan status
- **sessionStorage**: API key storage (not persisted across sessions)

The dashboard files are located in `core/dashboard/` and served by FastAPI at the `/dashboard/` prefix.

---

## Next Steps

- [API Reference](api_reference.md) -- REST API endpoint documentation
- [CLI Reference](cli_reference.md) -- Command-line interface reference
- [Interpreting Results](interpreting_results.md) -- Understanding scan output
