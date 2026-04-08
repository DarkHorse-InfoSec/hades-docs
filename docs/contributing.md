# Feedback & Support

HADES is proprietary software developed and maintained by DarkHorse Information Security LLC. This page covers how to report issues, request features, submit custom YARA rules, and develop detection plugins.

## Reporting Bugs

Submit bug reports via the issue tracker at [https://github.com/DarkHorse-Security/HADES/issues](https://github.com/DarkHorse-Security/HADES/issues).

When reporting a bug, include:

1. HADES version (`hades-enhanced --version` or check `pyproject.toml`)
2. Python version (`python --version`)
3. Operating system
4. Steps to reproduce
5. Expected vs actual behavior
6. Error messages and stack traces
7. If the issue involves a specific file, describe the file type and size (do not upload potentially malicious files)

Please check existing issues to avoid duplicates before filing.

## Feature Requests

Submit feature requests via the issue tracker at [https://github.com/DarkHorse-Security/HADES/issues](https://github.com/DarkHorse-Security/HADES/issues).

Include a clear description of the desired capability, the use case it addresses, and any relevant examples or references.

## Security Vulnerabilities

For security vulnerabilities, email **info@darkhorsesecurity.com** directly. Do not file public issues for security-related reports.

---

## YARA Rule Submissions

Custom YARA rules can be submitted via the issue tracker or by emailing **info@darkhorsesecurity.com**. See the [Writing YARA Rules](yara_rule_writing_guide.md) guide for naming conventions, required metadata, and testing procedures.

When submitting a rule, include:

1. The rule source following the `HADES_<CATEGORY>_<SPECIFIC>` naming convention
2. Required metadata: `author`, `description`, `severity`, `date`
3. Description of the attack technique detected
4. MITRE ATT&CK reference (if applicable)
5. Test evidence showing the rule triggers on malicious samples and does not trigger on clean files
6. Validation output from `python scripts/validate_rules.py`

---

## Plugin Development

Plugins extend HADES with custom detection logic. See the [Plugin Authoring Guide](plugin_authoring_guide.md) for the full developer walkthrough.

### Quick Reference

1. Create a `.py` file in `plugins/` subclassing `HADESPlugin` from `core/plugin_api.py`
2. Implement required methods: `name`, `version`, `author`, `initialize`, `analyze`, `supported_file_types`, `cleanup`
3. Return `Finding` instances with severity 0.0-10.0
4. Add unit tests
5. Verify loading: `hades-enhanced --list-plugins`

---

## Internal Development Guidelines

The following guidelines apply to authorized developers working on the HADES codebase.

### Code Standards

- **Python 3.8+** with type hints throughout
- **Logging**: Use the `logging` module, never bare `print` for operational output
- **Dataclasses**: Use `@dataclass` where appropriate
- **Error handling**: Use try/except with specific exceptions, log errors, provide user-friendly messages
- **Optional dependencies**: Use try/except pattern for imports with fallback flags (e.g., `YARA_AVAILABLE`, `SKLEARN_AVAILABLE`)
- **Security**: NEVER execute or eval untrusted file content
- **Formatting**: Run `black core/*.py cli/*.py plugins/*.py` before committing
- **Type checking**: Run `mypy --ignore-missing-imports core/*.py cli/*.py` before committing

### Running Tests

```bash
# Full test suite
python -m pytest core/ cli/ config/ plugins/ tests/ -v --tb=short

# Specific test file
python -m pytest core/test_detection_engine.py -v

# Detection corpus validation
python -m pytest tests/test_corpus_full_validation.py -v -s
```

!!! warning "All tests must pass"
    Changes will not be accepted if any tests fail. The test suite has 1,781+ tests -- run the full suite before finalizing changes.

### Commit Messages

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add polyglot detection for BMP+ZIP files
fix: handle empty metadata fields in Base64 detection
docs: update API reference with new endpoints
test: add corpus validation for SVG XXE attacks
refactor: extract metadata parsing into separate module
```

### Branch Naming

| Prefix | Purpose |
|---|---|
| `feature/` | New features |
| `fix/` | Bug fixes |
| `refactor/` | Code refactoring |
| `hotfix/` | Urgent production fixes |
| `docs/` | Documentation changes |
| `test/` | Test additions or fixes |

### Directory Structure

| Directory | Contains |
|---|---|
| `core/` | Core engine Python modules |
| `cli/` | CLI layer and basic scanner |
| `gui/` | Electron + React GUI |
| `docker/` | Docker images and Compose |
| `rules/` | YARA rule files |
| `config/` | Configuration and allowlists |
| `plugins/` | Detection plugins |
| `tests/` | Integration and corpus tests |
| `docs/` | Documentation (MkDocs) |
| `scripts/` | Build and utility scripts |

---

## Next Steps

- [Architecture](architecture.md) -- System design and module relationships
- [Writing YARA Rules](yara_rule_writing_guide.md) -- Rule writing guide
- [Plugin Authoring Guide](plugin_authoring_guide.md) -- Plugin API guide
