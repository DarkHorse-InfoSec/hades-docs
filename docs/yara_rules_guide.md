# YARA Rules in HADES

HADES ships with 41+ YARA rules across 7 rule files, all purpose-built for detecting threats hidden in file metadata. This guide covers how HADES loads and uses YARA rules, the shipped rule categories, severity mapping, and how to add custom rules.

## Overview

YARA is a pattern matching engine used in malware research. HADES uses YARA to scan file content and metadata for known threat patterns -- embedded executables, script injection, encoded payloads, polyglot constructs, and more.

!!! note "YARA is optional"
    HADES functions without `yara-python` installed. When YARA is not available, the scanner skips YARA matching and relies on heuristic and ML detection. Install YARA for best detection coverage: `pip install yara-python`.

---

## Shipped Rule Files

| File | Category | Rules | Description |
|---|---|---|---|
| `enhanced_detection.yara` | Core Detection | ~8 | Reverse shell patterns, suspicious metadata content, hidden executables, network indicators |
| `advanced_threats.yar` | Advanced Threats | ~8 | Payload injection, encoded payloads, web shells, cryptocurrency indicators, exfiltration, persistence |
| `enterprise_threats.yar` | Enterprise Threats | ~7 | Ransomware, APT toolkits, insider threats, spyware, cloud threats, supply chain, critical infrastructure |
| `steganography.yar` | Steganography | ~4 | Steganography tool indicators, metadata anomalies, high-entropy regions |
| `metadata_threats.yar` | Metadata Threats | ~4 | Polyglot file detection, embedded executables in non-standard positions |
| `polyglot_detection.yar` | Polyglot Detection | 6 | JPEG+ZIP, JPEG+RAR, PDF+JavaScript, PNG+HTML, GIF+JavaScript, generic dual-header |
| `suspicious_metadata.yar` | Suspicious Metadata | 6 | Base64 payloads >500 chars, URLs in EXIF, email PII, encoded executables, shell commands, oversized fields |

All rule files are located in the `rules/` directory at the project root.

---

## How HADES Loads Rules

1. On startup, the `EnhancedDetectionEngine` scans the configured rules directory (default: `rules/`) for `.yar` and `.yara` files.
2. Each file is compiled individually. If a file has syntax errors, it is skipped with a warning -- other files still load.
3. During scanning, compiled rules are matched against the target file's content.
4. Matches are included in the scan result with rule name, severity, and description from the rule's metadata.

### Rules Directory Configuration

The rules directory is configured in `cli/hades_config.json`:

```json
{
  "scanning": {
    "custom_yara_directory": "rules/"
  }
}
```

You can also place rules in a custom directory and point HADES to it:

```bash
hades-enhanced -r /evidence/ --plugins-dir /path/to/custom/rules/
```

---

## Naming Convention

All HADES rules follow the pattern:

```
HADES_<CATEGORY>_<SPECIFIC>
```

Examples:

- `HADES_POLYGLOT_JPEG_ZIP` -- JPEG+ZIP polyglot detection
- `HADES_METADATA_BASE64_PAYLOAD` -- Large Base64 content in metadata
- `HADES_METADATA_EMAIL_PII` -- Email address PII leakage

Legacy rules from earlier versions may use older naming (e.g., `MetadataPayloadInjection`, `ReverseShell_Common_Patterns`). These remain functional but will be renamed in a future release.

---

## Severity Mapping

Rules use integer severity levels 1-10 in their `meta` section:

| Range | Level | Description |
|---|---|---|
| 1-3 | Low | Informational findings, potential false positives, data classification markers |
| 4-6 | Medium | Suspicious patterns warranting investigation: network indicators, PII, anomalous metadata |
| 7-9 | High | Likely malicious: encoded payloads, polyglot files, web shells, persistence mechanisms |
| 10 | Critical | Confirmed malicious indicators: active exploits, ransomware, APT toolkits |

The severity from YARA matches feeds into the overall threat score calculation.

---

## Adding Custom Rules

### 1. Create a Rule File

Create a `.yar` file in the `rules/` directory:

```yara
rule HADES_CUSTOM_SUSPICIOUS_HEADER {
    meta:
        author = "Your Organization"
        description = "Detects suspicious custom header patterns"
        severity = 6
        date = "2026-02-19"
        category = "custom"
    strings:
        $suspicious = "X-Malware-Config:" nocase
    condition:
        $suspicious
}
```

### 2. Validate

Run the rule validation script:

```bash
python scripts/validate_rules.py
```

This checks for syntax errors, duplicate rule names, and missing metadata fields.

### 3. Test

Scan a known-good file corpus to check for false positives:

```bash
hades-enhanced -r /path/to/known-good-files/ --format json
```

Then scan a test file that should trigger the rule:

```bash
hades-enhanced test_file.jpg -v
```

---

## Visual Rule Builder (v0.6.0)

The web dashboard includes a YARA rule builder that lets you create rules from templates without writing YARA syntax directly:

1. Open the dashboard at `http://localhost:8666/dashboard/`
2. Navigate to the **Rule Builder** view
3. Select a template (metadata injection, steganography, polyglot, encoded payloads)
4. Customize strings and conditions
5. Test against uploaded samples
6. Save to the rules directory

Or use the CLI:

```bash
# List available templates
hades-enhanced --list-templates

# Build a rule from a template
hades-enhanced --build-rule metadata_injection
```

---

## Performance Notes

- YARA rules are compiled once at startup and reused for all scans.
- Complex regex patterns in rules can slow down scanning. Prefer literal string matching where possible.
- The `enterprise_threats.yar` file uses YARA modules (`pe`, `elf`, `math`) which must be enabled at compile time.
- Rules requiring only basic string matching are compatible with YARA 3.8+; module-based rules require YARA 4.0+.

!!! tip "Rule validation"
    Always run `python scripts/validate_rules.py` after modifying rules to catch syntax errors before they affect production scanning.

---

## Next Steps

- [Writing YARA Rules](yara_rule_writing_guide.md) -- Detailed guide for writing HADES-compatible rules
- [YARA Repository](yara_repository.md) -- Browse the rule repository
- [Deep Format Guide](deep_format_guide.md) -- Format-specific threat detection
