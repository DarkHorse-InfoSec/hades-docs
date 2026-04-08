# Security Policy

## Overview

HADES (Hidden Artifact Detection & EXIF Scanner) is a metadata forensics engine that routinely handles sensitive forensic data, evidence files, and threat intelligence. Security of this tool and its ecosystem is critical. DarkHorse Information Security LLC takes all security vulnerabilities seriously.

## Reporting a Vulnerability

If you discover a security vulnerability in HADES, please report it responsibly. **Do not open a public issue.**

Send your report to:

**security@darkhorseinfosec.com**

Include the following in your report:

- A description of the vulnerability and its potential impact.
- Steps to reproduce the issue or a proof of concept.
- The version(s) of HADES affected.
- Any relevant logs, screenshots, or configuration details.

## Response Timeline

- **48 hours** -- Acknowledgment of your report.
- **7 days** -- Initial assessment and severity classification.
- **Coordinated disclosure** -- We will work with you to determine an appropriate disclosure timeline once a fix is available. We aim to resolve critical issues as quickly as possible.

## Responsible Disclosure

We ask that you:

- Allow us reasonable time to investigate and address the vulnerability before any public disclosure.
- Avoid accessing, modifying, or deleting data belonging to others during your research.
- Act in good faith to avoid privacy violations, service disruption, and destruction of data.

We commit to:

- Acknowledging your contribution in release notes (unless you prefer to remain anonymous).
- Not pursuing legal action against researchers who follow this responsible disclosure policy.
- Keeping you informed of our progress toward a fix.

## Scope

This policy applies to:

- The HADES core scanning engine and analyzers.
- The REST API (FastAPI) and CLI.
- The Electron GUI application.
- The Python SDK.
- Official plugins distributed with HADES.
- Docker and Kubernetes deployment configurations.
- The plugin marketplace infrastructure.

Third-party plugins obtained outside the official marketplace are not covered by this policy.

## Security Best Practices

When deploying HADES in production:

- Keep HADES updated to the latest release.
- Restrict API access with authentication and network controls.
- Follow the hardening guidelines in the deployment documentation.
- Monitor SIEM integration logs for anomalous activity.
- Store evidence files and forensic data in encrypted storage.

---

DarkHorse Information Security LLC
