# HADES Plugin Development Guide

## Overview

The HADES Plugin API lets you extend the detection engine with custom analysis modules. Plugins run in a managed lifecycle with timeout enforcement, exception isolation, and per-plugin metrics -- so a misbehaving plugin cannot crash the core scanner.

Plugins are Python modules that subclass `HADESPlugin` from `core.plugin_api`. The `PluginManager` discovers them at startup, calls `initialize()` once, then calls `analyze()` for each scanned file. Results are returned as `Finding` dataclass instances with severity scores that integrate into HADES reports.

### Why Write a Plugin?

- Add organization-specific detection logic (proprietary IOC matching, custom heuristics)
- Integrate external threat feeds or APIs
- Implement specialized analysis for niche file formats
- Prototype new detection techniques before upstreaming into the core engine

## Quick Start

Create a file in the `plugins/` directory:

```python
# plugins/hello_plugin.py
from plugin_api import HADESPlugin, Finding
from typing import Dict, List, Any

class HelloPlugin(HADESPlugin):
    @property
    def name(self) -> str:
        return "Hello Plugin"

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def author(self) -> str:
        return "Your Name"

    def initialize(self, config: Dict[str, Any]) -> None:
        pass  # Setup resources here

    def supported_file_types(self) -> List[str]:
        return []  # Empty = all file types

    def analyze(self, file_path: str, metadata_result: Any) -> List[Finding]:
        return [Finding(
            severity=1.0,
            title="Hello from plugin",
            description=f"Scanned {file_path}",
            category="info",
        )]

    def cleanup(self) -> None:
        pass  # Release resources here
```

Verify it loads:

```bash
python core/hades_enhanced_cli.py --list-plugins
```

## API Reference

### Finding Dataclass

Every detection result from a plugin is a `Finding`:

```python
@dataclass
class Finding:
    severity: float          # 0.0 (informational) to 10.0 (critical)
    title: str               # Short human-readable title
    description: str         # Detailed explanation
    category: str            # Category tag (e.g., "malware", "steganography", "policy")
    metadata: Dict[str, Any] # Optional extra data (default: {})
```

**Severity scale:**
| Range | Meaning |
|-------|---------|
| 0.0 - 2.0 | Informational |
| 2.1 - 4.0 | Low risk |
| 4.1 - 6.0 | Medium risk |
| 6.1 - 8.0 | High risk |
| 8.1 - 10.0 | Critical |

### HADESPlugin Abstract Base Class

All plugins must subclass `HADESPlugin` and implement these members:

#### Properties (required)

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Unique human-readable plugin name |
| `version` | `str` | SemVer version string (e.g., `"1.0.0"`) |
| `author` | `str` | Author or organization name |

#### Methods (required)

**`initialize(config: Dict[str, Any]) -> None`**

Called once after the plugin is loaded. Receives the `plugins` section from `hades_config.json` (or an empty dict if no config is set). Use this to load resources, connect to APIs, or set up internal state.

**`supported_file_types() -> List[str]`**

Return a list of file extensions this plugin handles (e.g., `[".jpg", ".png"]`). Return an empty list `[]` to indicate all file types are supported.

**`analyze(file_path: str, metadata_result: Any) -> List[Finding]`**

Called once per scanned file. Parameters:
- `file_path`: Absolute path to the file being scanned.
- `metadata_result`: The `MetadataAnalysisResult` (or compatible wrapper) from the core parser. May be `None` if the parser failed.

Must return a list of `Finding` objects. Return an empty list if no issues are detected.

**`cleanup() -> None`**

Called once when the plugin is unloaded. Release file handles, database connections, or other resources here.

### PluginManager

The `PluginManager` handles discovery, loading, execution, and metrics.

```python
from plugin_api import PluginManager

pm = PluginManager(
    plugins_dir="plugins",      # Directory to scan for plugins
    config={"key": "value"},    # Config passed to plugin.initialize()
    timeout=30,                 # Per-plugin execution timeout (seconds)
)
```

**Key methods:**

| Method | Description |
|--------|-------------|
| `discover_plugins() -> List[str]` | List discoverable plugin names |
| `load_plugin(name) -> bool` | Load a single plugin by module name |
| `load_all() -> Dict[str, str]` | Discover and load all plugins; returns `{name: "ok"\|"error"}` |
| `unload_plugin(name) -> bool` | Unload a plugin (calls `cleanup()`) |
| `reload_all() -> Dict[str, str]` | Unload all, re-discover, re-load |
| `run_plugins(file_path, metadata_result, content) -> List[Finding]` | Execute all plugins against a file |
| `get_plugin_info() -> List[Dict]` | Get summary info for all loaded plugins |

**Properties:**

| Property | Description |
|----------|-------------|
| `loaded_plugins` | Dict mapping plugin name to plugin instance |
| `metrics` | Per-plugin runtime metrics (calls, findings, errors, avg time) |

## Configuration

Plugin configuration is stored in `cli/hades_config.json` under the `plugins` key:

```json
{
  "plugins": {
    "enabled": true,
    "directory": "plugins",
    "timeout_seconds": 30,
    "plugin_config": {
      "my_plugin_name": {
        "custom_setting": "value"
      }
    }
  }
}
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | bool | `true` | Master switch for the plugin system |
| `directory` | string | `"plugins"` | Plugin directory (relative to project root or absolute) |
| `timeout_seconds` | int | `30` | Max seconds a plugin can run per file |
| `plugin_config` | object | `{}` | Per-plugin configuration, keyed by plugin name |

The entire `plugins` section is passed to each plugin's `initialize(config)` method. Access your plugin-specific settings via `config.get("plugin_config", {}).get("your_plugin_name", {})`.

## CLI Flags

The enhanced CLI (`core/hades_enhanced_cli.py`) provides these plugin-related flags:

| Flag | Description |
|------|-------------|
| `--plugins-dir DIR` | Override the plugins directory (default: `plugins`) |
| `--disable-plugins` | Skip loading all detection plugins |
| `--list-plugins` | List loaded plugins with version/author info and exit |

Examples:

```bash
# List all discovered plugins
python core/hades_enhanced_cli.py --list-plugins

# Scan with plugins from a custom directory
python core/hades_enhanced_cli.py --plugins-dir /path/to/my/plugins -r /evidence/

# Scan without plugins
python core/hades_enhanced_cli.py --disable-plugins -r /evidence/
```

## Walkthrough: Building a Hash Checker Plugin

This example builds a plugin that checks file SHA-256 hashes against a known-bad list.

### Step 1: Create the Plugin File

```python
# plugins/hash_checker.py
"""
HADES Plugin: Known-Bad Hash Checker
Checks file hashes against a configurable list of known-malicious hashes.
"""

import hashlib
import logging
from pathlib import Path
from typing import Dict, List, Any

from plugin_api import HADESPlugin, Finding

logger = logging.getLogger("HADES.Plugin.HashChecker")


class HashCheckerPlugin(HADESPlugin):

    def __init__(self):
        self._known_bad: set = set()

    @property
    def name(self) -> str:
        return "Known-Bad Hash Checker"

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def author(self) -> str:
        return "Your Organization"

    def initialize(self, config: Dict[str, Any]) -> None:
        """Load known-bad hashes from config."""
        plugin_cfg = config.get("plugin_config", {}).get("hash_checker", {})
        hashes = plugin_cfg.get("known_bad_hashes", [])
        self._known_bad = {h.lower() for h in hashes}
        logger.info("HashChecker loaded %d known-bad hashes", len(self._known_bad))

    def supported_file_types(self) -> List[str]:
        return []  # Check all file types

    def analyze(self, file_path: str, metadata_result: Any) -> List[Finding]:
        findings: List[Finding] = []
        try:
            sha256 = hashlib.sha256(Path(file_path).read_bytes()).hexdigest()
            if sha256 in self._known_bad:
                findings.append(Finding(
                    severity=9.5,
                    title="Known-malicious file hash detected",
                    description=f"SHA-256 {sha256} matches known-bad hash list",
                    category="malware",
                    metadata={"sha256": sha256, "source": "local_blocklist"},
                ))
        except OSError as exc:
            logger.warning("HashChecker could not read %s: %s", file_path, exc)
        return findings

    def cleanup(self) -> None:
        self._known_bad.clear()
```

### Step 2: Add Configuration

In `cli/hades_config.json`, add your plugin config under `plugins.plugin_config`:

```json
"plugins": {
    "enabled": true,
    "directory": "plugins",
    "timeout_seconds": 30,
    "plugin_config": {
        "hash_checker": {
            "known_bad_hashes": [
                "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
            ]
        }
    }
}
```

### Step 3: Test It

```bash
# Verify the plugin loads
python core/hades_enhanced_cli.py --list-plugins

# Run a scan
python core/hades_enhanced_cli.py suspicious_file.exe -v
```

## Plugin Discovery

The `PluginManager` discovers plugins in two ways:

1. **Single-file plugins**: Any `.py` file in the plugins directory (excluding files starting with `_`).
2. **Package plugins**: Subdirectories containing an `__init__.py` file.

For each discovered module, the manager imports it and looks for the first class that subclasses `HADESPlugin`. Only one plugin class per module is loaded.

### File Structure Examples

```
plugins/
    __init__.py              # Package marker (ignored by discovery)
    hash_checker.py          # Single-file plugin
    entropy_analyzer.py      # Single-file plugin
    my_complex_plugin/       # Package plugin
        __init__.py          # Must contain or import a HADESPlugin subclass
        helpers.py
        data/
            signatures.json
```

## Testing Your Plugin

### Unit Testing

Test your plugin in isolation without the full HADES stack:

```python
# tests/test_my_plugin.py
import tempfile
import os
from plugins.hash_checker import HashCheckerPlugin


def test_clean_file():
    plugin = HashCheckerPlugin()
    plugin.initialize({"plugin_config": {"hash_checker": {"known_bad_hashes": []}}})

    with tempfile.NamedTemporaryFile(delete=False, suffix=".txt") as f:
        f.write(b"clean content")
        f.flush()
        findings = plugin.analyze(f.name, None)

    assert findings == []
    os.unlink(f.name)
    plugin.cleanup()


def test_malicious_hash():
    import hashlib
    content = b"malicious payload"
    bad_hash = hashlib.sha256(content).hexdigest()

    plugin = HashCheckerPlugin()
    plugin.initialize({
        "plugin_config": {
            "hash_checker": {"known_bad_hashes": [bad_hash]}
        }
    })

    with tempfile.NamedTemporaryFile(delete=False, suffix=".exe") as f:
        f.write(content)
        f.flush()
        findings = plugin.analyze(f.name, None)

    assert len(findings) == 1
    assert findings[0].severity >= 9.0
    assert findings[0].category == "malware"
    os.unlink(f.name)
    plugin.cleanup()
```

Run with: `python -m pytest tests/test_my_plugin.py -v`

### Integration Testing

Test that the PluginManager loads and runs your plugin:

```python
from plugin_api import PluginManager

pm = PluginManager(plugins_dir="plugins", config={"plugin_config": {}})
info = pm.get_plugin_info()
assert any(p["name"] == "Known-Bad Hash Checker" for p in info)
```

## Security Considerations

Plugins run within the HADES process, so security is enforced through conventions and runtime safeguards:

### Timeout Enforcement
Each plugin's `analyze()` call runs in a dedicated thread with a configurable timeout (default: 30 seconds). If a plugin exceeds the timeout, the call is cancelled and an error is logged. The plugin's error count is incremented in its metrics.

### Exception Isolation
All exceptions raised during `analyze()` are caught, logged, and counted -- they never propagate to the caller or affect other plugins. A plugin that repeatedly fails will accumulate errors visible in the metrics.

### File Access
Plugins receive a file path and should only read the target file. Plugins must NOT:
- Execute or eval file content
- Write to the file system (except temporary files cleaned up in `cleanup()`)
- Make network calls without user consent (document any external calls in your plugin README)
- Access files outside the scan scope

### Resource Management
- Release all resources in `cleanup()` (file handles, DB connections, thread pools)
- Avoid loading very large data structures in `initialize()` -- the plugin stays loaded for the entire scan session
- Respect the timeout: break long operations into checkpoints if possible

### Code Review
Before deploying a third-party plugin:
1. Review the source code for malicious behavior
2. Check that it only reads the target file
3. Verify it does not shell out to external commands
4. Test it in an isolated environment first
