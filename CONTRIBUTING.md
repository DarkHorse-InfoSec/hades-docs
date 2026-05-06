# Contributing to HADES

Thank you for your interest in HADES (Hidden Artifact Detection & EXIF Scanner), a proprietary product of **DarkHorse Information Security LLC**.

## Source code is not public

HADES source code, YARA rule sets, scoring algorithms, and ML feature definitions are proprietary trade secrets of DarkHorse Information Security LLC under the Defend Trade Secrets Act of 2016 (18 U.S.C. Sections 1836-1839) and corresponding state Uniform Trade Secrets Act statutes. Source distribution is via the private Forgejo repository at `forgejo.darkhorseinfosec.com/DarkHorse/darkhorse-hades`, which is access-controlled. Customer source review for security audit purposes is available under enterprise NDA terms.

This means the standard open-source contribution workflow (fork the repo, clone, branch, PR back) is **not** how HADES contributions work. The "fork and pull request" instructions in earlier versions of this file were drawn from a pre-v1.4 era when HADES briefly considered an open-source community model; that was never operationalized.

## What this PUBLIC docs repo IS for

This repository (`https://github.com/DarkHorse-InfoSec/hades-docs`) is the **public documentation mirror**. Issues and pull requests against this repo are welcome for:

- **Documentation fixes:** typos, broken links, version-stale install instructions, unclear explanations, missing edge-case coverage in user-facing guides.
- **Issue reports against HADES the product:** bug reports, feature requests, performance complaints, false positives or negatives reported with sample files.
- **Plugin contributions:** if you've written a HADES plugin under the public plugin SDK, file a PR against `plugins/community/` here. Plugin contributions are released under the LICENSE in this repo and we evaluate them for inclusion in the official plugin marketplace.

## What this repo is NOT for

- **Source code patches:** if you have a bug fix or feature idea that requires modifying core HADES code, file an issue describing the problem and DarkHorse will route it to the internal Forgejo repo. We cannot accept source patches from the public.
- **Reverse-engineering output:** decompiling the Nuitka binary or analyzing the encrypted YARA rules and submitting the results is prohibited under the LICENSE and TRADE_SECRETS.md.

## Issue reports

Open an issue on this repo with:

- A clear title and description.
- HADES version (`hades --version`) and platform.
- Steps to reproduce (if applicable).
- Expected vs. actual behavior.
- Relevant logs, error output, or scan output. **Never include real customer files; redact filenames and use synthetic samples where possible.**

For security vulnerabilities, do NOT open a public issue. See [SECURITY.md](SECURITY.md) for the responsible disclosure process; if SECURITY.md is missing, email `security@darkhorseinfosec.com`.

## Documentation pull requests

If you spot a fix in this docs repo, the workflow is the standard GitHub flow:

1. Fork `https://github.com/DarkHorse-InfoSec/hades-docs` to your account.
2. Branch from `main`.
3. Make your changes (markdown only -- no code in this repo).
4. Open a PR with a clear description.

PRs are reviewed by DarkHorse on a best-effort basis. Documentation contributions are accepted under the LICENSE in this repo; you retain copyright in your contribution but grant DarkHorse a perpetual, royalty-free license to use, modify, and redistribute it as part of the HADES documentation.

## Plugin contributions

The HADES plugin SDK is shipped as part of the customer-facing binary; plugins compile against a stable API surface documented in `docs/plugin_development_guide.md`. To contribute a community plugin:

1. Read `docs/plugin_development_guide.md` for the API.
2. Develop and test your plugin against your own HADES installation.
3. Open a PR against this repo with the plugin source under `plugins/community/<your-plugin-name>/`, including unit tests + a README describing the plugin's purpose, configuration, and usage.
4. Sign the [Contributor License Agreement (CLA)](CLA.md) if one is provided in the repo (DarkHorse may require this before merging plugin contributions).

DarkHorse evaluates community plugins for technical correctness, security implications, and fit with the marketplace; we may decline contributions that overlap heavily with planned first-party features.

## Becoming a paid contractor / employee

If you'd like to work on HADES core (the proprietary source), DarkHorse occasionally hires contractors and employees under standard NDA + IP-assignment agreements. Reach out via `careers@darkhorseinfosec.com`.

## License

By contributing to this PUBLIC docs repo, you agree that your contributions will be governed by the project's proprietary license. See [LICENSE](LICENSE) for details.

---

HADES is built and maintained by DarkHorse Information Security LLC. Copyright (c) 2024-2026, all rights reserved.
