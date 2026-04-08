# Plugin Marketplace

The HADES plugin ecosystem extends the detection engine with custom analysis modules. Plugins run in a managed lifecycle with timeout enforcement, exception isolation, and per-plugin metrics.

## Available Plugins

### ML Anomaly Plugin

**File:** `plugins/ml_anomaly_plugin.py`

Wraps the `MLDetectionEngine` as a HADES plugin, running anomaly detection on every scanned file when the ML model is available.

| Property | Value |
|---|---|
| Author | DarkHorse Information Security LLC |
| Version | 1.0.0 |
| File Types | All |
| Dependencies | scikit-learn, numpy |

### Hash Checker

**File:** `plugins/example_hash_checker.py`

Computes MD5, SHA-1, and SHA-256 hashes of scanned files and checks them against a configurable list of known-bad hashes.

| Property | Value |
|---|---|
| Author | DarkHorse Information Security LLC |
| Version | 1.0.0 |
| File Types | All |
| Dependencies | None |

**Configuration** (`cli/hades_config.json`):

```json
{
  "plugins": {
    "plugin_config": {
      "hash_checker": {
        "known_bad_hashes": [
          "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
        ]
      }
    }
  }
}
```

### Entropy Analyzer

**File:** `plugins/example_entropy_analyzer.py`

Computes per-chunk Shannon entropy to detect anomalous high-entropy regions within files, which may indicate encrypted payloads, compressed data, or steganographic content.

| Property | Value |
|---|---|
| Author | DarkHorse Information Security LLC |
| Version | 1.0.0 |
| File Types | All |
| Dependencies | None |

---

## Installing Plugins

### From the Repository

Plugins in the `plugins/` directory are automatically discovered and loaded at startup. To add a plugin:

1. Place the `.py` file in the `plugins/` directory
2. Restart the scanner or API server
3. Verify: `hades-enhanced --list-plugins`

### From a Custom Directory

Point HADES to a custom plugins directory:

```bash
hades-enhanced --plugins-dir /path/to/my/plugins -r /evidence/
```

### Disabling Plugins

To run without any plugins:

```bash
hades-enhanced --disable-plugins -r /evidence/
```

---

## Creating Your Own Plugin

Every HADES plugin is a Python class that subclasses `HADESPlugin` from `core/plugin_api.py`. The minimum implementation requires:

```python
from plugin_api import HADESPlugin, Finding
from typing import Dict, List, Any

class MyPlugin(HADESPlugin):
    @property
    def name(self) -> str:
        return "My Plugin"

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def author(self) -> str:
        return "Your Name"

    def initialize(self, config: Dict[str, Any]) -> None:
        pass

    def supported_file_types(self) -> List[str]:
        return []  # Empty = all file types

    def analyze(self, file_path: str, metadata_result: Any) -> List[Finding]:
        return []

    def cleanup(self) -> None:
        pass
```

For a full walkthrough with examples, tests, and security guidelines, see the [Plugin Development Guide](plugin_development_guide.md).

---

## Plugin API

### Finding Dataclass

```python
@dataclass
class Finding:
    severity: float          # 0.0 to 10.0
    title: str               # Short description
    description: str         # Detailed explanation
    category: str            # Category tag
    metadata: Dict[str, Any] # Optional extra data
```

### Severity Scale

| Range | Level |
|---|---|
| 0.0 - 2.0 | Informational |
| 2.1 - 4.0 | Low |
| 4.1 - 6.0 | Medium |
| 6.1 - 8.0 | High |
| 8.1 - 10.0 | Critical |

### Plugin Lifecycle

1. **Discovery**: `PluginManager` scans the plugins directory for `.py` files
2. **Loading**: Each module is imported; the first `HADESPlugin` subclass is instantiated
3. **Initialization**: `initialize(config)` is called with the plugins config section
4. **Analysis**: `analyze(file_path, metadata_result)` is called for each scanned file
5. **Cleanup**: `cleanup()` is called when the plugin is unloaded

### Safety Guarantees

- **Timeout**: Each `analyze()` call has a configurable timeout (default: 30 seconds)
- **Isolation**: Exceptions in plugins never propagate to the caller
- **Metrics**: Per-plugin call counts, error counts, and average execution times

---

## REST API

### List Plugins

```bash
curl -H "X-API-Key: your-api-key-here" \
  http://localhost:8666/api/v1/plugins
```

### Reload Plugins

```bash
curl -X POST -H "X-API-Key: your-api-key-here" \
  http://localhost:8666/api/v1/plugins/reload
```

---

## Next Steps

- [Plugin Development Guide](plugin_development_guide.md) -- Full plugin API reference and walkthrough
- [Contributing](contributing.md) -- Contribution guide
- [Architecture](architecture.md) -- System architecture
