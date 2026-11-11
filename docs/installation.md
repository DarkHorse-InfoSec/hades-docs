# Installation

HADES ships as a license-gated, signed binary. There are two supported install paths; both fetch the same Nuitka-compiled binary from the portal-issued signed URL after validating your license.

---

## Path A: direct download from the portal (canonical)

Get your `HADES_LICENSE_KEY` from the license email after purchase, then:

```bash
export HADES_LICENSE_KEY="<paste from portal email>"

# Linux x86_64
curl -fL -H "Authorization: Bearer $HADES_LICENSE_KEY" \
  "https://portal.darkhorseinfosec.com/api/v1/download/linux-x86_64/latest/hades" \
  -o hades && chmod +x hades

# Windows x86_64 binaries are also served by the portal. macOS is not yet available.
```

The portal validates your license, generates a 15-minute HMAC-SHA256-signed URL to Cloudflare R2, and `curl` follows the redirect. Windows binaries are Authenticode-signed (self-signed certificate) and RFC3161-timestamped; the bundled ML model is RSA-PSS-signed and is not loaded if its signature fails to verify.

To upgrade later, re-run the command with the new version number, or check `https://portal.darkhorseinfosec.com/api/v1/download` for the current version.

## Path B: Homebrew tap (Linux convenience, via Linuxbrew)

```bash
export HOMEBREW_HADES_LICENSE_KEY="<paste from portal email>"
brew tap DarkHorse-InfoSec/tap
brew install DarkHorse-InfoSec/tap/hades-scanner
```

The formula uses `HadesPortalDownloadStrategy` to inject your license key as a Bearer token, hits the same portal endpoint as Path A, and stages the binary under `$(brew --prefix)/bin/hades`. Same gated path, just wrapped in `brew install`.

To upgrade:

```bash
brew update && brew upgrade hades-scanner
```

---

## Verifying the installation

After either path:

```bash
hades --version
# expect: HADES Enhanced Detection Engine v<current release>

hades doctor
# runs ~25 dependency probes + a Threat Intel section reporting which
# API keys are present (length only, never the value)
```

Quick scan:

```bash
hades scan path/to/file.exe
hades scan path/to/dir/ --recursive
```

API server (Pro+ tier):

```bash
hades-server --port 8666
curl http://localhost:8666/api/v1/health
```

---

## Platform notes

The Nuitka-compiled binary ships its Python interpreter + all Python dependencies inline. You do NOT need Python, pip, YARA, scikit-learn, or any other library installed on the host. ExifTool is optional for scanning (a native fallback ships in the binary) and recommended for the metadata sanitizer.

| OS | Status |
|---|---|
| Linux x86_64 (glibc 2.34+) | Path A + Path B (via Linuxbrew) |
| Windows x86_64 | Path A; binaries are Authenticode-signed (self-signed certificate) and RFC3161-timestamped |
| macOS | Not yet available; no release date |

**1.7.5 and later need glibc 2.34+ (RHEL 9 family included); v1.7.3 and v1.7.4 need 2.35+.** Measured 2026-11-10 on the 1.7.5 release binaries: the highest glibc symbol across all 275 programs and libraries they unpack is 2.34, and both binaries run on Red Hat UBI 9 (glibc 2.34), where v1.7.3 fails with `GLIBC_2.35 not found`.

**Correction (2026-11-08): the v1.7.3 and v1.7.4 Linux binaries need glibc 2.35, not 2.34.** This page said 2.34 until 2026-11-08. That figure came from the binary's small outer loader; the program it unpacks and runs needs glibc 2.35. Verified by running the shipped v1.7.3 and v1.7.4 binaries: they start on Ubuntu 22.04 (glibc 2.35) and fail on glibc 2.34 with `version 'GLIBC_2.35' not found`. Distributions on glibc 2.34, such as RHEL 9, Rocky Linux 9, AlmaLinux 9 and Amazon Linux 2023, cannot run current Linux releases. Check a target with `ldd --version | head -1`.

---

## Where HADES keeps its data

From 1.7.5, every database and store HADES creates (scan results, the evidence and audit log, cases, users,
tenants, playbooks, quarantine, threat-intelligence caches) lives in one directory:

| Platform | Default location |
|---|---|
| Linux / macOS | `~/.hades/data` |
| Windows | `%USERPROFILE%\.hades\data` |

- Set `HADES_DATA_DIR` to an **absolute** path to move it; stores then live in `$HADES_DATA_DIR/data`. A
  relative path is refused, because it would put data wherever the program happened to be started.
- The directory is created owner-only (`0700` on Linux and macOS).
- The location does not change between versions, so an upgrade keeps your audit log, cases and users.
- Nothing is ever written to the working directory or to the binary's extraction directory.

**Earlier versions** kept their databases in a per-version folder (for example
`%LOCALAPPDATA%\hades\1.7.4\core` or `~/.cache/hades/1.7.4/core`). 1.7.5 does not read or migrate them.
`hades doctor` lists any it finds, and the API server logs them at startup, so you can archive or delete
them deliberately.

## Features not included in the 1.7.5 binaries

These features need third-party components that the 1.7.5 `hades` and `hades-server` binaries do not
include. Each one refuses or switches itself off with a message naming the missing component; nothing
fails silently. Including them is planned for 1.7.6.

| Feature | Needs | What you see in 1.7.5 |
|---|---|---|
| Email gateway (`hades monitor --email-gateway`) | `aiosmtpd` | exits with "aiosmtpd package required" |
| Google Cloud Storage scanning | `google-cloud-storage` | the cloud scan API answers HTTP 503 "Dependency Missing" for GCS (AWS S3 works) |
| Azure Blob Storage scanning | `azure-storage-blob` | the cloud scan API answers HTTP 503 "Dependency Missing" for Azure (AWS S3 works) |
| Slack bot | `slack-bolt` | starting the bot fails with an error naming `slack_bolt` |
| Microsoft Teams bot | `botbuilder-core` | starting the bot fails with an error naming `botbuilder-core` |
| Prometheus metrics (`/metrics`, `hades serve --metrics`) | `prometheus-client` | the metrics collector is disabled and logs that `prometheus_client` is missing |

## Upgrading

- **Path A:** re-run the curl one-liner with the new version number from your release email or `https://portal.darkhorseinfosec.com/api/v1/download`.
- **Path B:** `brew update && brew upgrade hades-scanner`.

License behaviour, as implemented in v1.7.1 (`core/auth/license.py`, `core/auth/license_enforcer.py`):

- **Expired license: HADES stops; it does not fall back to Community.** The key's signature and expiry date are checked locally each time HADES starts. If `HADES_LICENSE_KEY` is set and the key has expired (or its signature fails), the CLI exits with `License validation failed: License is no longer valid (expired at ...)` and the API server rejects requests. This is deliberate: a silent downgrade would scan with fewer detection stages without telling you. Renew at `https://portal.darkhorseinfosec.com/billing`, or unset the key to run as Community (Free).
- **Revoked license (API server only):** `hades-server` checks the license with the portal at startup. If the portal reports the license revoked, the server runs at Community tier. The CLI does not contact the portal.
- **Offline:** if `hades-server` cannot reach the portal at startup, it uses its cached validation result, and once that cache is more than 24 hours old it logs a warning and keeps the licensed tier. Only an explicit revocation from the portal lowers the tier, so an air-gapped host keeps working until the license's expiry date.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `curl: (22) The requested URL returned error: 401 Unauthorized` | License key wrong, expired, or no active subscription | Verify in portal dashboard; check the email matches the active sub |
| `hades --version` prints "community" tier despite license | Binary started before env var set | Set `HADES_LICENSE_KEY` first, then run; it loads from `~/.hades/.env` if present |
| `brew install` fails with "Cannot find license key" | `HOMEBREW_HADES_LICENSE_KEY` env var not set | Export it before `brew install`; Homebrew strips most env vars but keeps `HOMEBREW_*` |

For everything else, `hades doctor` checks the runtime, external tools, bundled Python dependencies, YARA rules, threat-intel configuration and license, and reports actionable next steps.

---

## Older install paths (no longer supported)

Pre-v1.4 distribution via pip from the Cloudsmith private registry at `dl.cloudsmith.io/basic/darkhorse/hades` (`pip install hades-scanner` against that private index only), Docker Hub images at `darkhorse-security/hades-scanner`, and public source-clone are no longer supported. Path A and Path B above are the only canonical install paths. For enterprise security-audit or source-review needs under NDA, contact `support@darkhorseinfosec.com`.

Older release notes (`RELEASE_NOTES_v0.5.0.md` through `RELEASE_NOTES_v0.7.1.md`) reference the legacy distribution model and are kept for historical reference; the install commands they show no longer work.

---

## Next Steps

- [Quick Start Guide](quick_start.md), get scanning in 5 minutes
- [CLI Reference](cli_reference.md), full command-line reference
- [Threat Intelligence Guide](threat_intel_guide.md), wiring up MalwareBazaar + VirusTotal
