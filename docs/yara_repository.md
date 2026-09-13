# YARA Rule Repository

The HADES YARA rule repository contains 144 detection rules across 55 rule files, all purpose-built for identifying threats hidden in file metadata, embedded payloads, polyglot files, and other attack vectors that traditional file scanners miss.

## Repository Structure

The tree below lists the core packs. The full shipped ruleset is 144 detection
rules across 55 rule files; the 41 per-family packs merged from the DEF CON 34
rule set in v1.7.0 are not listed individually here.

```
rules/
    enhanced_detection.yara    # Core detection patterns
    advanced_threats.yar       # Advanced attack patterns
    enterprise_threats.yar     # Enterprise-level threats
    steganography.yar          # Steganography detection
    metadata_threats.yar       # Metadata-specific threats
    polyglot_detection.yar     # Polyglot file detection
    suspicious_metadata.yar    # Suspicious metadata content
```

---

## Browse by Category

### Core Detection (`enhanced_detection.yara`)

Fundamental detection patterns for common metadata-based attacks:

| Rule | Severity | Description |
|---|---|---|
| Reverse shell patterns | 9 | Common reverse shell command patterns in metadata |
| Suspicious metadata content | 6 | Generic suspicious strings in metadata fields |
| Hidden executables | 8 | Executable headers in non-executable file types |
| Network indicators | 5 | URLs, IPs, and domains in metadata |

### Advanced Threats (`advanced_threats.yar`)

Advanced attack patterns targeting sophisticated threat actors:

| Rule | Severity | Description |
|---|---|---|
| Payload injection | 8 | Encoded payloads embedded in metadata |
| Encoded payloads | 7 | Multi-layer encoding (Base64, hex, URL encoding) |
| Web shells | 9 | PHP/ASP/JSP web shell code in metadata |
| Cryptocurrency indicators | 5 | Wallet addresses and mining pool URLs |
| Exfiltration markers | 7 | Data exfiltration indicators in metadata |
| Persistence mechanisms | 8 | Registry keys, scheduled tasks, startup entries |

### Enterprise Threats (`enterprise_threats.yar`)

Enterprise and nation-state level threat detection:

| Rule | Severity | Description |
|---|---|---|
| Ransomware indicators | 10 | Known ransomware signatures and patterns |
| APT toolkit markers | 10 | Advanced persistent threat tool signatures |
| Insider threat indicators | 6 | Data classification and handling markers |
| Spyware patterns | 8 | Known spyware metadata signatures |
| Cloud threat indicators | 7 | Cloud service abuse patterns |
| Supply chain markers | 9 | Supply chain compromise indicators |

!!! note "Module requirements"
    `enterprise_threats.yar` uses YARA modules (`pe`, `elf`, `math`) which require YARA 4.0+ with modules enabled at compile time.

### Steganography (`steganography.yar`)

Detection of data hiding techniques in file metadata and content:

| Rule | Severity | Description |
|---|---|---|
| Tool indicators | 6 | Steganography tool signatures (steghide, OpenStego) |
| Metadata anomalies | 5 | Unusual metadata field sizes or patterns |
| High-entropy regions | 6 | Anomalous entropy in non-compressed sections |

### Metadata Threats (`metadata_threats.yar`)

Threats specific to file metadata manipulation:

| Rule | Severity | Description |
|---|---|---|
| Polyglot detection | 7 | Files valid in multiple formats |
| Embedded executables | 8 | Executables hidden in non-standard positions |

### Polyglot Detection (`polyglot_detection.yar`)

Dedicated rules for detecting polyglot file constructs:

| Rule | Severity | Description |
|---|---|---|
| `HADES_POLYGLOT_JPEG_ZIP` | 7 | JPEG image containing a ZIP archive |
| `HADES_POLYGLOT_JPEG_RAR` | 7 | JPEG image containing a RAR archive |
| `HADES_POLYGLOT_PDF_JS` | 8 | PDF with embedded JavaScript outside normal structure |
| `HADES_POLYGLOT_PNG_HTML` | 7 | PNG image with HTML content in trailing data |
| `HADES_POLYGLOT_GIF_JS` | 7 | GIF image with JavaScript in comment blocks |
| `HADES_POLYGLOT_DUAL_HEADER` | 6 | Generic detection of two format signatures |

### Suspicious Metadata (`suspicious_metadata.yar`)

Anomalous patterns in metadata fields:

| Rule | Severity | Description |
|---|---|---|
| `HADES_METADATA_BASE64_PAYLOAD` | 7 | Base64-encoded content >500 characters |
| `HADES_METADATA_URL_IN_EXIF` | 5 | URLs found in EXIF metadata fields |
| `HADES_METADATA_EMAIL_PII` | 4 | Email addresses in metadata (PII leakage) |
| `HADES_METADATA_ENCODED_EXECUTABLE` | 8 | Base64-encoded executable headers (MZ, ELF) |
| `HADES_METADATA_SHELL_COMMAND` | 8 | Shell commands in metadata fields |
| `HADES_METADATA_OVERSIZED_FIELD` | 5 | Metadata fields exceeding normal size limits |

---

## Browse by Severity

### Critical (10)

- Ransomware indicators
- APT toolkit markers

### High (7-9)

- Reverse shell patterns
- Web shells
- PHP backdoors
- Payload injection
- Polyglot files
- Encoded executables
- Shell commands
- Template injection markers
- Supply chain indicators

### Medium (4-6)

- Network indicators
- Steganography tool markers
- Metadata anomalies
- URLs in EXIF
- Email PII leakage
- Oversized metadata fields
- Cryptocurrency indicators
- Insider threat markers

---

## Submitting Custom Rules

Custom YARA rules can be submitted via the issue tracker at [https://github.com/DarkHorse-InfoSec/hades-docs/issues](https://github.com/DarkHorse-InfoSec/hades-docs/issues) or by emailing **info@darkhorsesecurity.com**. See the [Writing YARA Rules](yara_rule_writing_guide.md) guide for naming conventions, required metadata, and testing procedures.

When submitting a rule, include:

1. The rule source following the `HADES_<CATEGORY>_<SPECIFIC>` naming convention
2. Required metadata: `author`, `description`, `severity`, `date`
3. Validation output from `python scripts/validate_rules.py`
4. Test evidence against both clean and malicious files
5. Description of the attack technique and MITRE ATT&CK reference (if applicable)

---

## Next Steps

- [YARA Rules Guide](yara_rules_guide.md) -- How HADES uses YARA rules
- [Writing YARA Rules](yara_rule_writing_guide.md) -- Detailed rule writing guide
- [Deep Format Guide](deep_format_guide.md) -- Format-specific threat detection
