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

# (macOS and Windows binaries on R2 by 2026-Q3; until then, use Path B for macOS.)
```

The portal validates your license, generates a 15-minute HMAC-SHA256-signed URL to Cloudflare R2, and `curl` follows the redirect. The binary is RSA-PSS-signed and verifies its own integrity at first run.

To upgrade later, re-run the command with the new version number, or check `https://portal.darkhorseinfosec.com/api/v1/download` for the current version.

## Path B: Homebrew tap (macOS and Linux convenience)

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

The Nuitka-compiled binary ships its Python interpreter + all dependencies inline. You do NOT need Python, pip, ExifTool, YARA, scikit-learn, or any other library installed on the host. Just the OS.

| OS | Status |
|---|---|
| Linux x86_64 (glibc 2.31+) | Path A + Path B (via Linuxbrew) |
| macOS Intel (10.15+) | Path B today; Path A x86_64 macOS binary on R2 by 2026-Q3 |
| macOS Apple Silicon (11.0+) | Path B today; Path A arm64 macOS binary on R2 by 2026-Q3 |
| Windows 10 / 11 / Server 2019+ | Windows binary build pending code-signing cert acquisition |

---

## Upgrading

- **Path A:** re-run the curl one-liner with the new version number from your release email or `https://portal.darkhorseinfosec.com/api/v1/download`.
- **Path B:** `brew update && brew upgrade hades-scanner`.

License re-validation runs at every binary startup; expired or revoked licenses fall back to community tier (heuristics + IOC only). Upgrade your subscription at `https://portal.darkhorseinfosec.com/billing` to restore full features.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `curl: (22) The requested URL returned error: 401 Unauthorized` | License key wrong, expired, or no active subscription | Verify in portal dashboard; check the email matches the active sub |
| `hades --version` prints "community" tier despite license | Binary started before env var set | Set `HADES_LICENSE_KEY` first, then run; it loads from `~/.hades/.env` if present |
| `brew install` fails with "Cannot find license key" | `HOMEBREW_HADES_LICENSE_KEY` env var not set | Export it before `brew install`; Homebrew strips most env vars but keeps `HOMEBREW_*` |

For everything else, `hades doctor` covers ~25 environment probes and reports actionable next steps.

---

## Older install paths (no longer supported)

Pre-v1.4 distribution via PyPI (`pip install hades-scanner`), Docker Hub images at `darkhorse-security/hades-scanner`, the Cloudsmith private registry at `dl.cloudsmith.io/basic/darkhorse/hades`, and public source-clone are no longer supported. Path A and Path B above are the only canonical install paths. For enterprise security-audit or source-review needs under NDA, contact `support@darkhorseinfosec.com`.

Older release notes (`RELEASE_NOTES_v0.5.0.md` through `RELEASE_NOTES_v0.7.1.md`) reference the legacy distribution model and are kept for historical reference; the install commands they show no longer work.

---

## Next Steps

- [Quick Start Guide](quick_start.md), get scanning in 5 minutes
- [CLI Reference](cli_reference.md), full command-line reference
- [Threat Intelligence Guide](threat_intel_guide.md), wiring up MalwareBazaar + VirusTotal
