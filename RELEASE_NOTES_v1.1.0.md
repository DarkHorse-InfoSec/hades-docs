# HADES v1.1.0 Release Notes

**Release Date:** 2026-03-21
**Release Type:** Major Feature Release

## Highlights

HADES v1.1.0 adds comprehensive forensic detection capabilities across 26 new analyzer modules, closing 44 detection gaps. Every analyzer uses stdlib only — zero new external dependencies required.

## New Detection Capabilities

### Archive Forensics
- **ZIP Forensic Analyzer** — Zombie ZIP (compression method mismatch), ZIP concatenation (Gootloader campaigns), CRC32 mismatches, ZIP bombs (ratio + nested + total size), malformed central directory, timestamp anomalies, entry name tricks (RLO, null byte, double extension), comment injection, Zip64 field abuse
- **RAR/7z Forensic Analyzer** — RAR5 solid archive bombs, header encryption, compression ratio bombs, RAR4 analysis, 7z CRC mismatches, invalid headers, deep filter chains
- **Archive Depth Analyzer** — Nested decompression bomb detection, cross-format nesting, total extraction size monitoring
- **SFX Detector** — PE/ELF/Mach-O executables with embedded ZIP/RAR/7z/CAB, 10 SFX tool families, archive-at-EOF detection

### Image Forensics
- **Image Forensic Analyzer** — Decompression bombs (JPEG/PNG/GIF/BMP/WebP/HEIF), ICC profile injection, EXIF thumbnail mismatch, IPTC injection, TIFF IFD chain validation, GIF frame analysis, WebP/HEIF deep analysis

### Document Forensics
- **RTF Analyzer** — OLE object/Equation Editor detection (CVE-2017-11882), template injection, embedded executables, obfuscation techniques, Unicode abuse, DDE links
- **LNK Analyzer** — Suspicious targets (24 executables), dangerous arguments (12 patterns), icon harvesting, env var abuse, oversized shortcuts
- **OLE Document Analyzer** — Legacy .doc/.xls/.ppt VBA macros, dangerous VBA patterns, auto-execution, Equation Editor/Package Shell ClassIDs, DDE, encryption
- **OneNote Analyzer** — Embedded file attachments, executable payloads, dangerous filenames, script content, URL analysis
- **VBA Stomping Detector** — Source/p-code mismatch, module count anomalies, stomping tool artifacts (EvilClippy)

### PDF Advanced Analysis
- **Shadow Attack Detection** — Incremental update scanning for injected JavaScript, Launch actions, form replacement, page substitution
- **Filter Chain Detection** — Deep filter chains, suspicious filters (JBIG2Decode), Object Streams, heavy ASCII encoding
- **Spreadsheet Formula Injection** — CMD, EXEC, HYPERLINK, WEBSERVICE, DDE via formula

### Office Advanced Analysis
- **Hidden Content Detection** — Track changes, hidden text (w:vanish), deleted text (w:del), suspicious comments, custom XML parts, glossary inspection
- **OLE Equation Editor** — ClassID detection (CVE-2017-11882, severity 9.0), Package Shell objects
- **SVG CSS Injection** — expression(), -moz-binding, behavior:url(), javascript: in CSS

### Audio/Video Forensics
- **Audio Forensic Analyzer** — MP3 ID3v2 anomalies, WAV chunk injection/bombs, OGG Vorbis comment abuse, FLAC metadata anomalies, MP4/M4A atom injection
- **Video Forensic Analyzer** — MKV subtitle/attachment abuse, MP4 atom injection, AVI chunk injection, FLV script data

### Steganography
- **Stego Analyzer** — JPEG DCT-domain analysis (quality estimation, entropy, Huffman tables), 10 stego tool signatures, EOF appended data, GIF/PNG palette analysis, LSB distribution

### Campaign Delivery Formats
- **Disk Image Analyzer** — ISO 9660/UDF autorun detection, executable content, MotW bypass risk, hidden content in system area
- **Email File Analyzer** — EML/MSG parsing, Reply-To mismatch, sender spoofing, phishing URLs, credential forms, dangerous attachments, MIME mismatches
- **Font Analyzer** — TTF/OTF/WOFF table anomalies, name injection, CFF threats, embedded content
- **MSI Analyzer** — Custom actions, dangerous scripts, embedded executables, PowerShell patterns
- **CHM Analyzer** — Embedded scripts, ActiveX, dangerous commands, file references, chained loading

### Modern Formats
- **JAR/APK Analyzer** — Manifest parsing, suspicious classes, native libraries, JSP web shells, Android permissions, DEX validation
- **Notebook Analyzer** — Dangerous imports, shell execution, reverse shells, credential exposure, code obfuscation

### Security Middleware
- **Unicode Homoglyph Detection** — 39 Cyrillic/Greek confusables, 15 bidi/zero-width control characters
- **MIME/Extension/Magic Consistency** — Cross-referencing across 20 file types
- **Script Encoding Tricks** — UTF-16/UTF-32 BOM markers, null-byte interleaving detection
- **Timestamp Forensics** — Stomping detection, impossible dates, zeroed timestamps, creation paradox

### YARA Rules
- **zip_structure.yar** — 8 rules for ZIP structural anomalies including Gootloader campaign patterns
- **pua_signatures.yar** — 7 rules for cryptominers, adware, keyloggers, screenloggers, remote access tools, browser hijackers, surveillance tools

## Test Coverage
- 520 new tests across 26 test files
- 100% True Positive Rate on all synthetic corpus specimens
- Full test suite: 2,757 passed

## Performance
- Single scan mean: 3.06ms (no regression from 26 new analysis stages)
- Batch throughput: 39.5 files/sec
- All analyzers check magic bytes first and bail immediately for non-matching formats

## Architecture
- All 26 new modules follow the established analyzer pattern
- Zero new external dependencies — stdlib only throughout
- Optional import with availability flags for graceful degradation
- Profiled analysis stages with _pstart/_pend instrumentation
- Comprehensive documentation in GAPGUIDE.md and GAPGUIDE_v2.md

## Upgrade Notes
- No breaking changes — all new capabilities are additive
- New analyzers activate automatically when the detection engine initializes
- Configuration sections in hades_config.json are optional (sensible defaults)
- Existing YARA rules, plugins, and API endpoints are unchanged
