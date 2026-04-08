# HADES Plugin Authoring Guide

## Overview

The HADES plugin system provides an extensible framework for adding custom detection, enrichment, and reporting capabilities to the metadata forensics engine. Plugins are discovered, loaded, and executed automatically by the `PluginManager`.

This guide walks you through creating, testing, packaging, and distributing a HADES plugin.

## Architecture

```
plugins/
  __init__.py               # Package marker
  registry.py               # Plugin marketplace registry
  manifest_template.json    # Template for plugin manifests
  example_hash_checker.py   # Built-in: file hash computation
  example_entropy_analyzer.py  # Built-in: entropy analysis
  example_steganography_detector.py  # Built-in: steganography detection
  example_pii_scanner.py    # Built-in: PII detection
  your_plugin.py            # Your custom plugin
```

### Plugin Lifecycle

1. **Discovery** — `PluginManager` scans the plugins directory for `.py` files
2. **Loading** — Module is imported and the first `HADESPlugin` subclass is instantiated
3. **Initialization** — `initialize(config)` is called with the plugin configuration
4. **Execution** — `analyze(file_path, metadata_result)` is called for each scanned file
5. **Cleanup** — `cleanup()` is called when the plugin is unloaded

### Key Classes

| Class | Module | Description |
|-------|--------|-------------|
| `HADESPlugin` | `core/plugin_api.py` | Abstract base class — all plugins subclass this |
| `Finding` | `core/plugin_api.py` | Standard output dataclass for detection results |
| `PluginManager` | `core/plugin_api.py` | Discovery, loading, and execution engine |
| `PluginManifest` | `plugins/registry.py` | Plugin metadata for marketplace |
| `PluginRegistry` | `plugins/registry.py` | Marketplace install/uninstall/search |

## Tutorial: Creating a Plugin

### Step 1: Create the Plugin File

Create a new `.py` file in the `plugins/` directory:

```python
#!/usr/bin/env python3
"""
HADES Plugin — My Custom Detector
File: plugins/my_custom_detector.py
Version: 1.0.0
Description: Detects [describe what your plugin detects].
Author: Your Name
License: Proprietary
"""

import sys
from pathlib import Path
from typing import Dict, List, Any

# Allow imports from core/ when loaded standalone or by PluginManager
_CORE_DIR = str(Path(__file__).resolve().parent.parent / "core")
if _CORE_DIR not in sys.path:
    sys.path.insert(0, _CORE_DIR)

from plugin_api import HADESPlugin, Finding


class MyCustomDetectorPlugin(HADESPlugin):
    """Detect [your threat/artifact type]."""

    @property
    def name(self) -> str:
        return "My Custom Detector"

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def author(self) -> str:
        return "Your Name"

    def initialize(self, config: Dict[str, Any]) -> None:
        # Read plugin-specific config
        self._threshold = config.get("plugin_config", {}).get(
            "my_custom_detector", {}
        ).get("threshold", 7.0)

    def cleanup(self) -> None:
        pass

    def supported_file_types(self) -> List[str]:
        # Return empty list for all file types, or specific extensions
        return [".jpg", ".jpeg", ".png"]

    def analyze(self, file_path: str, metadata_result: Any) -> List[Finding]:
        findings: List[Finding] = []

        # Your detection logic here
        # ...

        if suspicious:
            findings.append(Finding(
                severity=7.5,       # 0.0 to 10.0
                title="Suspicious Pattern Detected",
                description="Detailed description of the finding.",
                category="your_category",
                metadata={"key": "value"},
            ))

        return findings
```

### Step 2: The Finding Dataclass

Every detection result is a `Finding`:

```python
@dataclass
class Finding:
    severity: float       # 0.0 (informational) to 10.0 (critical)
    title: str            # Short, descriptive title
    description: str      # Detailed explanation
    category: str         # Grouping category (e.g., "malware", "pii", "stego")
    metadata: Dict[str, Any]  # Structured data for programmatic consumption
```

**Severity Guidelines:**

| Range | Level | Examples |
|-------|-------|---------|
| 0.0-2.0 | Informational | File hashes, entropy stats |
| 2.1-4.0 | Low | Minor anomalies, unusual but not threatening |
| 4.1-6.0 | Medium | Suspicious patterns, potential PII |
| 6.1-8.0 | High | Known tool signatures, GPS leakage |
| 8.1-10.0 | Critical | Malware indicators, SSN/CC, exploit patterns |

### Step 3: File Type Filtering

Return specific extensions from `supported_file_types()` to limit which files your plugin processes:

```python
def supported_file_types(self) -> List[str]:
    return [".jpg", ".jpeg", ".png", ".gif", ".bmp"]
```

Return an empty list to process all file types:

```python
def supported_file_types(self) -> List[str]:
    return []
```

### Step 4: Configuration

Plugin configuration is passed via the `config` parameter in `initialize()`. The standard location is `cli/hades_config.json`:

```json
{
  "plugins": {
    "enabled": true,
    "directory": "plugins",
    "timeout_seconds": 30,
    "plugin_config": {
      "my_custom_detector": {
        "threshold": 7.0,
        "custom_setting": true
      }
    }
  }
}
```

Access it in your plugin:

```python
def initialize(self, config: Dict[str, Any]) -> None:
    my_cfg = config.get("plugin_config", {}).get("my_custom_detector", {})
    self._threshold = float(my_cfg.get("threshold", 7.0))
```

### Step 5: Working with Metadata Results

The `metadata_result` parameter provides parsed metadata from the core engine. It may be a `MetadataAnalysisResult`, a `MockMetadataAnalysisResult`, or `None`.

```python
def analyze(self, file_path: str, metadata_result: Any) -> List[Finding]:
    # Safe pattern for extracting metadata
    if metadata_result and hasattr(metadata_result, "metadata_fields"):
        for field in metadata_result.metadata_fields:
            name = getattr(field, "field_name", "")
            value = getattr(field, "value", "")
            # Process field...
    elif metadata_result and isinstance(metadata_result, dict):
        for key, value in metadata_result.items():
            # Process field...
```

## Testing Your Plugin

### Unit Test Template

```python
import tempfile
import os
from pathlib import Path
from plugins.my_custom_detector import MyCustomDetectorPlugin

def test_detection():
    plugin = MyCustomDetectorPlugin()
    plugin.initialize({})

    # Create a test file
    with tempfile.NamedTemporaryFile(suffix=".jpg", delete=False) as f:
        f.write(b"\xff\xd8\xff\xe0" + b"\x00" * 100)
        path = f.name

    try:
        findings = plugin.analyze(path, None)
        assert isinstance(findings, list)
        # Assert specific findings...
    finally:
        os.unlink(path)
        plugin.cleanup()

def test_loads_via_manager():
    from core.plugin_api import PluginManager
    mgr = PluginManager(plugins_dir="plugins")
    assert "My Custom Detector" in mgr.loaded_plugins
```

### Running Tests

```bash
python -m pytest plugins/test_my_plugin.py -v
```

## Packaging for Distribution

### 1. Generate a Manifest

```python
from plugins.registry import PluginRegistry

registry = PluginRegistry()
manifest = registry.export_manifest("plugins/my_custom_detector.py")
print(manifest.to_dict())
```

Or use the manifest template:

```bash
cp plugins/manifest_template.json my_plugin_manifest.json
# Edit with your plugin details
```

### 2. Compute SHA-256

```bash
sha256sum plugins/my_custom_detector.py
```

### 3. Register in the Marketplace

```python
from plugins.registry import PluginRegistry, PluginManifest

registry = PluginRegistry()
manifest = PluginManifest(
    name="My Custom Detector",
    version="1.0.0",
    author="Your Name",
    description="Detects custom threat patterns",
    download_url="https://example.com/my_custom_detector.py",
    sha256="abc123...",
    category="detection",
    file_types=[".jpg", ".png"],
)
registry.register_plugin(manifest)
```

## Best Practices

1. **Never execute file content** — HADES is a forensics tool that analyzes without executing. Read bytes, parse structures, match patterns — never `exec()` or `eval()`.

2. **Handle missing dependencies gracefully** — Use try/except for optional imports with availability flags.

3. **Respect timeouts** — The `PluginManager` enforces a per-plugin timeout (default 30s). Keep analysis fast and avoid blocking I/O.

4. **Return structured metadata** — Always include useful data in `Finding.metadata` for downstream consumption by SIEM, reports, and dashboards.

5. **Use logging, not print** — Use `logging.getLogger("HADES.Plugins.YourPlugin")` for operational output.

6. **Document severity rationale** — Explain why each finding has its severity score in comments or the description.

7. **Test with both clean and malicious samples** — Ensure low false-positive rates and high detection rates.

8. **Keep plugins self-contained** — A single `.py` file is preferred. If you need multiple files, use a subdirectory with `__init__.py`.

## API Reference

### HADESPlugin (Abstract Base Class)

| Member | Type | Description |
|--------|------|-------------|
| `name` | property | Unique plugin name (str) |
| `version` | property | SemVer version string |
| `author` | property | Author name |
| `initialize(config)` | method | Called once after loading |
| `analyze(file_path, metadata_result)` | method | Analyze one file, return List[Finding] |
| `supported_file_types()` | method | Return supported extensions (empty = all) |
| `cleanup()` | method | Release resources on unload |

### PluginManager

| Method | Description |
|--------|-------------|
| `discover_plugins()` | Return names of discoverable plugins |
| `load_plugin(name)` | Load a single plugin by name |
| `load_all()` | Discover and load all plugins |
| `unload_plugin(name)` | Unload a plugin by name |
| `reload_all()` | Unload all, re-discover, re-load |
| `run_plugins(file_path, metadata_result)` | Execute all plugins on a file |
| `get_plugin_info()` | Return summary info for loaded plugins |
| `loaded_plugins` | Dict of name -> instance |
| `metrics` | Per-plugin call/error/timing metrics |

### PluginRegistry

| Method | Description |
|--------|-------------|
| `register_plugin(manifest)` | Add plugin to local registry |
| `search_plugins(query, category, file_type)` | Search registered plugins |
| `install_plugin(name, target_dir)` | Download, verify, install |
| `uninstall_plugin(name, target_dir)` | Remove plugin file and registry entry |
| `update_plugin(name)` | Re-download and verify |
| `check_updates()` | Check all plugins for updates |
| `export_manifest(plugin_path)` | Generate manifest from plugin file |
| `load_remote_registry(url)` | Fetch remote registry JSON |
