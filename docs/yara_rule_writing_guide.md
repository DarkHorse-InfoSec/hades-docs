# Writing YARA Rules for HADES

This guide covers writing YARA rules that are compatible with the HADES detection engine, including naming conventions, metadata requirements, metadata-specific patterns, testing, and submission.

## YARA Basics for Analysts

If you are new to YARA, here is the minimum you need to know:

```yara
rule RULE_NAME {
    meta:
        key = "value"
    strings:
        $text_string = "pattern" nocase
        $hex_bytes = { DE AD BE EF }
        $regex = /pattern[0-9]+/
    condition:
        any of them
}
```

- **`meta`**: Metadata about the rule (author, description, severity). Not used for matching.
- **`strings`**: Patterns to search for in the file. Can be text, hex bytes, or regex.
- **`condition`**: Boolean expression that determines if the rule matches. References string variables.

---

## HADES Naming Convention

All HADES rules must follow:

```
HADES_<CATEGORY>_<SPECIFIC>
```

| Category | Use For |
|---|---|
| `POLYGLOT` | Polyglot/multi-format file detection |
| `METADATA` | Suspicious metadata content (Base64, URLs, PII, commands) |
| `STEGANOGRAPHY` | Steganography indicators |
| `ENTERPRISE` | Enterprise-level threats (APT, ransomware, supply chain) |
| `ADVANCED` | Advanced attack patterns (web shells, encoded payloads) |
| `CORE` | Fundamental detection (reverse shells, executables) |
| `CUSTOM` | User-contributed rules |

Examples:

- `HADES_POLYGLOT_JPEG_ZIP`
- `HADES_METADATA_SHELL_COMMAND`
- `HADES_ENTERPRISE_APT_TOOLKIT`

---

## Required Metadata Fields

Every HADES rule must include these metadata fields:

```yara
rule HADES_CATEGORY_NAME {
    meta:
        author = "Your Name or Organization"
        description = "Clear description of what this rule detects"
        severity = 7
        date = "2026-02-19"
        category = "category_name"
        reference = "https://attack.mitre.org/techniques/T1059/"
    strings:
        ...
    condition:
        ...
}
```

| Field | Required | Type | Description |
|---|---|---|---|
| `author` | Yes | string | Author name or organization |
| `description` | Yes | string | What the rule detects and why it matters |
| `severity` | Yes | integer | Severity level 1-10 |
| `date` | Yes | string | Rule creation or last update date (YYYY-MM-DD) |
| `category` | Recommended | string | Rule category for grouping and filtering |
| `reference` | Recommended | string | MITRE ATT&CK technique URL or other reference |

### Severity Scale

| Range | Level | When to Use |
|---|---|---|
| 1-3 | Low | Informational findings, potential false positives |
| 4-6 | Medium | Suspicious patterns warranting investigation |
| 7-9 | High | Likely malicious content |
| 10 | Critical | Confirmed malicious indicators |

---

## Metadata-Specific Patterns

HADES rules should focus on metadata fields rather than general file content. Here are patterns commonly found in metadata attacks:

### Base64 Payloads in Metadata

```yara
rule HADES_METADATA_BASE64_PAYLOAD {
    meta:
        author = "DarkHorse Information Security LLC"
        description = "Detects large Base64-encoded content in file metadata"
        severity = 7
        date = "2026-02-19"
        category = "metadata"
    strings:
        $b64_long = /[A-Za-z0-9+\/]{500,}={0,2}/
    condition:
        $b64_long
}
```

### Script Injection in EXIF

```yara
rule HADES_METADATA_SCRIPT_INJECTION {
    meta:
        author = "DarkHorse Information Security LLC"
        description = "Detects script tags in image metadata fields"
        severity = 8
        date = "2026-02-19"
        category = "metadata"
        reference = "https://attack.mitre.org/techniques/T1059/007/"
    strings:
        $script = "<script" nocase
        $onload = "onload=" nocase
        $onerror = "onerror=" nocase
        $iframe = "<iframe" nocase
    condition:
        any of them
}
```

### Shell Commands in Metadata

```yara
rule HADES_METADATA_SHELL_COMMAND {
    meta:
        author = "DarkHorse Information Security LLC"
        description = "Detects shell commands embedded in metadata fields"
        severity = 8
        date = "2026-02-19"
        category = "metadata"
        reference = "https://attack.mitre.org/techniques/T1059/"
    strings:
        $wget = "wget " nocase
        $curl = "curl " nocase
        $bash = "bash -i" nocase
        $nc = "nc -e" nocase
        $chmod = "chmod +x" nocase
        $sh = "/bin/sh" nocase
    condition:
        any of them
}
```

### PHP Backdoor in EXIF

```yara
rule HADES_METADATA_PHP_BACKDOOR {
    meta:
        author = "DarkHorse Information Security LLC"
        description = "Detects PHP code in image metadata for web shell attacks"
        severity = 9
        date = "2026-02-19"
        category = "metadata"
        reference = "https://attack.mitre.org/techniques/T1505/003/"
    strings:
        $php_open = "<?php" nocase
        $eval = "eval(" nocase
        $system = "system(" nocase
        $exec = "shell_exec(" nocase
        $passthru = "passthru(" nocase
    condition:
        $php_open and any of ($eval, $system, $exec, $passthru)
}
```

---

## Tips for Metadata-Focused Rules

1. **Target metadata, not file structure**: Focus on strings that appear in EXIF comments, XMP data, PDF info dictionaries, and Office properties.

2. **Use `nocase`**: Metadata field values have inconsistent casing. Always add `nocase` for text patterns.

3. **Be specific with regex**: Overly broad patterns cause false positives. A regex matching 100+ alphanumeric characters will fire on many legitimate files.

4. **Combine conditions**: Use `and` to reduce false positives:

    ```yara
    condition:
        $suspicious_url and $shell_command
    ```

5. **Test against benign files**: Always scan a corpus of clean images, documents, and archives before deploying.

6. **Avoid PCRE-specific features**: YARA uses its own regex engine with limitations:
    - No non-capturing groups `(?:...)` -- use regular groups `(...)`
    - No lookahead/lookbehind
    - No backreferences
    - No named groups

---

## Validation and Testing

### Validate Syntax

```bash
python scripts/validate_rules.py
```

This checks for:

- Syntax errors in all `.yar` and `.yara` files
- Duplicate rule names across files
- Missing required metadata fields
- Summary statistics by category and severity

### Test Against Clean Files

```bash
# Scan known-good files to check for false positives
hades-enhanced -r /path/to/known-good/ --format json --output fp_check.json
```

### Test Against Malicious Samples

```bash
# Run the detection corpus
python -m pytest tests/test_corpus_full_validation.py -v -s
```

---

## Submission Process

### Before Submitting

1. Follow the naming convention: `HADES_<CATEGORY>_<SPECIFIC>`
2. Include all required metadata fields
3. Run `python scripts/validate_rules.py` -- no errors
4. Test against both malicious samples and clean file corpus
5. Document the attack technique the rule detects
6. Provide a MITRE ATT&CK reference where applicable

### File Organization

Place rules in the appropriate file:

| Rule Type | File |
|---|---|
| General metadata patterns | `suspicious_metadata.yar` |
| Polyglot/multi-format | `polyglot_detection.yar` |
| Enterprise/APT patterns | `enterprise_threats.yar` |
| Steganography detection | `steganography.yar` |
| New category | Create a new `.yar` file |

### Code Review Checklist

- [ ] Rule name follows `HADES_<CATEGORY>_<SPECIFIC>` convention
- [ ] All required metadata fields present
- [ ] Severity is an integer 1-10 and appropriate for the threat
- [ ] No YARA syntax errors
- [ ] No duplicate rule names
- [ ] Tested against clean files (no false positives)
- [ ] Description clearly explains the detection logic

---

## Next Steps

- [YARA Rules Guide](yara_rules_guide.md) -- Overview of shipped rules
- [YARA Repository](yara_repository.md) -- Browse the rule repository
- [Contributing](contributing.md) -- General contribution guide
