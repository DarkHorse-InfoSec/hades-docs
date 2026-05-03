# Installation

HADES supports multiple installation methods. Choose the one that fits your environment.

## System Requirements

| Requirement | Minimum | Recommended |
|---|---|---|
| Python | 3.8+ | 3.12+ |
| RAM | 1 GB | 4 GB (with ML detection) |
| Disk | 100 MB | 500 MB (with models and rules) |
| OS | Windows, Linux, macOS | Any |

[ExifTool](https://exiftool.org/) must be installed and available on your `PATH` for full metadata extraction. HADES operates in degraded mode without it, using Pillow for basic EXIF parsing.

---

## From Cloudsmith (Private Registry)

First, configure the registry (one-time):

```bash
pip config set global.extra-index-url https://dl.cloudsmith.io/basic/darkhorse/hades/python/simple/
```

=== "Basic"

    Install core scanning with no optional dependencies:

    ```bash
    pip install hades-scanner
    ```

=== "Full"

    Install all features (API, YARA, ML, Docker, cloud, email, chat, formats, enterprise):

    ```bash
    pip install "hades-scanner[full]"
    ```

=== "API Server"

    Install with REST API and WebSocket support:

    ```bash
    pip install "hades-scanner[api]"
    ```

=== "Enterprise"

    Install with RBAC, SSO, PostgreSQL, Redis, and encryption:

    ```bash
    pip install "hades-scanner[enterprise]"
    ```

=== "Development"

    Install with test and formatting tools:

    ```bash
    pip install "hades-scanner[dev]"
    ```

### Dependency Groups

| Group | Packages | Purpose |
|---|---|---|
| `api` | FastAPI, Uvicorn, python-multipart, websockets, httpx, Pydantic | REST API server and WebSocket |
| `yara` | yara-python | YARA pattern matching engine |
| `ml` | scikit-learn, numpy, joblib | ML anomaly detection |
| `docker` | docker | Sandboxed file viewer |
| `cloud` | boto3, google-cloud-storage, azure-storage-blob | Cloud storage scanning |
| `email` | aiosmtpd | SMTP email gateway |
| `chat` | slack-bolt, botbuilder-core | Slack and Teams bots |
| `crypto` | cryptography | Report signing and encryption |
| `formats` | pikepdf, olefile | Deep PDF and Office analysis |
| `enterprise` | bcrypt, PyJWT, cryptography, psycopg2-binary, redis | RBAC, SSO, PostgreSQL, Redis, encryption |
| `dev` | pytest, black, mypy, httpx, psutil | Development tools |
| `full` | All of the above | Everything |

You can combine groups:

```bash
pip install "hades-scanner[api,yara,ml]"
```

---

## From Homebrew (macOS)

```bash
brew tap DarkHorse-InfoSec/tap
brew install hades-scanner
```

The Homebrew formula installs Python 3.12, ExifTool, and creates a virtualenv with HADES and its dependencies.

---

## From Docker

Pull the production image:

```bash
docker pull darkhorse-security/hades-scanner:0.7.1
```

Run the API server:

```bash
docker run -d \
  --name hades-api \
  -p 8666:8666 \
  -e HADES_API_KEY=your-secure-api-key \
  darkhorse-security/hades-scanner:0.7.1
```

Or use Docker Compose for the full stack:

```bash
docker compose -f docker/docker-compose.yml up -d
```

See the [Docker Guide](docker_guide.md) for production configuration, volume mounts, and scaling.

---

## From Source

Clone the repository and install in editable mode:

```bash
git clone https://github.com/DarkHorse-InfoSec/hades-docs.git
cd HADES
pip install -e ".[dev]"
```

This installs HADES with development dependencies (pytest, black, mypy) in editable mode so code changes take effect immediately.

---

## Verifying the Installation

After installation, verify that HADES is working:

```bash
# Check CLI entry points
hades --help
hades-enhanced --help

# Check API server
hades-server --help

# Run a quick scan
hades suspicious_file.jpg

# Start the API server
hades-server --port 8666

# Health check
curl http://localhost:8666/api/v1/health
```

### Verify Optional Dependencies

```bash
# Check which optional features are available
python -c "
import importlib
checks = {
    'YARA': 'yara',
    'scikit-learn': 'sklearn',
    'FastAPI': 'fastapi',
    'pikepdf': 'pikepdf',
    'olefile': 'olefile',
    'Docker SDK': 'docker',
    'boto3 (S3)': 'boto3',
    'Redis': 'redis',
}
for name, mod in checks.items():
    try:
        importlib.import_module(mod)
        print(f'  {name}: available')
    except ImportError:
        print(f'  {name}: not installed')
"
```

---

## Platform Notes

### Windows

- ExifTool: Download from [exiftool.org](https://exiftool.org/) and add to `PATH`, or install via `choco install exiftool`.
- yara-python: May require Visual C++ Build Tools. Install via `pip install yara-python` or use a prebuilt wheel.
- Docker Desktop required for sandbox and container features.

### Linux

- ExifTool: `sudo apt install libimage-exiftool-perl` (Debian/Ubuntu) or `sudo yum install perl-Image-ExifTool` (RHEL/CentOS).
- yara-python: `pip install yara-python`. If compilation fails, install `libyara-dev` first.

### macOS

- ExifTool: `brew install exiftool`.
- yara-python: `brew install yara && pip install yara-python`.

---

## Upgrading

```bash
pip install --upgrade hades-scanner
```

For enterprise deployments, see the [migration section](enterprise_deployment_guide.md#migration-from-v050) in the Enterprise Deployment Guide.

---

## Next Steps

- [Your First Scan](first_scan.md) -- Walkthrough of your first HADES scan
- [Quick Start Guide](quick_start.md) -- Get scanning in 5 minutes
- [CLI Reference](cli_reference.md) -- Full command-line reference
