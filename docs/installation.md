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

The glibc floor of 2.34 was measured with `readelf` on the shipped v1.7.1 Linux binary; it is a property of the build host (Ubuntu 22.04). Check a target with `ldd --version`.

---

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
