# Contributing to HADES

Thank you for your interest in contributing to HADES (Hidden Artifact Detection & EXIF Scanner), a project by **DarkHorse Information Security LLC**.

## Getting Started

### Prerequisites

- Python 3.8 or higher
- Git
- A virtual environment tool (venv, virtualenv, or conda)

### Setup

1. Fork and clone the repository.
2. Create a virtual environment and activate it:
   ```bash
   python -m venv .venv
   source .venv/bin/activate  # Linux/macOS
   .venv\Scripts\activate     # Windows
   ```
3. Install development dependencies:
   ```bash
   pip install -r requirements.txt
   pip install -e ".[dev]"
   ```

## Development Workflow

### Code Style

This project uses **Black** for code formatting and **Ruff** for linting:

```bash
black .
ruff check .
ruff check --fix .
```

All submissions must pass both formatters without errors before merging.

### Running Tests

HADES uses **pytest** for its test suite:

```bash
pytest
pytest tests/ -v              # verbose output
pytest tests/ -x              # stop on first failure
pytest tests/ -k "test_name"  # run specific tests
```

Ensure all existing tests pass and add tests for any new functionality.

### Commit Messages

Use clear, descriptive commit messages. Prefix with the area of change when applicable (e.g., `core: add HEIF metadata parser`, `plugins: fix stego-scanner threshold`).

## Plugin Development

HADES features an extensible plugin architecture. To contribute a new plugin:

1. Review existing plugins in the `plugins/` directory for structure and conventions.
2. Implement the required plugin interface for your scanner or analyzer.
3. Include unit tests for your plugin under `tests/`.
4. Document your plugin's purpose, configuration options, and usage.

Refer to the SDK documentation in `sdk/` for programmatic integration details.

## Submitting Changes

1. Create a feature branch from `main`.
2. Make your changes with clear, focused commits.
3. Ensure all tests pass and code style checks are clean.
4. Open a pull request with a description of what your change does and why.

## Reporting Issues

Open an issue on the repository with:

- A clear title and description.
- Steps to reproduce (if applicable).
- Expected vs. actual behavior.
- Relevant logs or error output.

For security vulnerabilities, do **not** open a public issue. See [SECURITY.md](SECURITY.md) for the responsible disclosure process.

## License

By contributing to HADES, you agree that your contributions will be governed by the project's proprietary license. See [LICENSE](LICENSE) for details.

---

HADES is maintained by DarkHorse Information Security LLC.
