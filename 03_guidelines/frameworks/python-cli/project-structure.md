# Project Structure Guidelines
This document defines the directory layout and architecture guidelines for Python CLI tool projects.

---

# 1. Overall Principles
- Adopt **module-based (domain-based) structure**
  - Group types / services / utils per functional domain
- **Separate CLI layer from core logic**
  - `cli/` handles argument parsing and process control only
  - Core logic is importable as a library without CLI dependency
- **Package exports** via `src/{package}/__init__.py` for library usage

---

# 2. Directory Structure

```
src/
└─ {package_name}/
   ├─ __init__.py               # Library barrel export (no CLI dependency)
   │
   ├─ cli/                      # CLI entry point (argument parsing, process control)
   │  ├─ __init__.py
   │  └─ main.py                # Typer app definition & subcommands
   │
   ├─ pipeline/                 # Pipeline orchestration (multi-step processing)
   │  ├─ __init__.py
   │  └─ orchestrator.py        # Orchestrator, step registration, state management
   │
   ├─ {domain_a}/               # Domain module A (e.g., collector, scraper)
   │  ├─ __init__.py             # Public API for the module
   │  └─ {sub_module}.py         # Internal implementation
   │
   ├─ {domain_b}/               # Domain module B (e.g., preprocessor, transformer)
   │  └─ __init__.py
   │
   ├─ {domain_c}/               # Domain module C (e.g., reporter, output generator)
   │  └─ __init__.py
   │
   ├─ extensions/               # Optional/pluggable modules
   │  ├─ __init__.py
   │  ├─ {extension_a}.py
   │  └─ {extension_b}.py
   │
   ├─ cache/                    # Caching system (TTL, hashing, invalidation)
   │  └─ __init__.py
   │
   ├─ config/                   # Configuration resolution & defaults
   │  ├─ __init__.py
   │  └─ settings.py            # Pydantic Settings model
   │
   └─ utils/                    # Shared utilities
      ├─ __init__.py
      ├─ errors.py              # Error class hierarchy
      ├─ logger.py              # Structured logger setup
      ├─ validation.py          # Runtime data validators
      └─ file_io.py             # File read/write helpers

tests/
├─ conftest.py                  # Shared fixtures
├─ unit/                        # Unit tests (mirror src structure)
│  ├─ {domain_a}/
│  └─ {domain_b}/
├─ integration/                 # Integration tests
└─ e2e/                         # End-to-end CLI tests

docs/
├─ specs/                       # Design specifications & diagrams
└─ usages/                      # Usage documentation

pyproject.toml                  # Project metadata, dependencies, entry points
```

---

# 3. Module Design

## 3.1 Internal Structure of a Module

Select files based on the scale of the module. Not all are required.

```
{module}/
├─ __init__.py                  # Public API (re-exports)
├─ {feature}.py                 # Feature implementation
├─ types.py                     # Module-specific types (optional)
└─ constants.py                 # Module-specific constants (optional)
```

## 3.2 Structure Examples by Scale

**Small module** (config, cache):
```
config/
├─ __init__.py
└─ settings.py                  # Pydantic model + resolve logic in a single file
```

**Medium module** (preprocessor, reporter):
```
reporter/
├─ __init__.py                  # Entry: generate_all_reports()
├─ base.py                      # Shared reporter helper
├─ business.py                  # Business report generator
└─ competitor.py                # Competitor report generator
```

**Large module** (collector with multiple strategies):
```
collector/
├─ __init__.py                  # Entry: collect()
├─ web.py                       # Web scraping (Playwright)
├─ source.py                    # Source code scanning
├─ auth.py                      # Authentication handling
└─ types.py                     # Collector-specific types
```

---

# 4. Dependency Rules

## 4.1 Allowed Dependencies
```
cli/ → pipeline/ → {domain modules} → utils/, config/
                 → cache/
                 → extensions/
```

## 4.2 Prohibited Practices
- **Cross-domain module dependencies are prohibited** — if two modules need shared logic, extract it to `utils/`
- **utils/ → domain module dependencies are prohibited**
- **Domain modules → cli/ dependencies are prohibited** — core logic must work without CLI
- **Circular imports are prohibited** — enforce with import linting (ruff, import-linter)

---

# 5. CLI Layer Separation (Strict)

The `cli/` directory is **strictly limited** to:
- Argument parsing and validation (Typer setup)
- Configuration resolution
- Orchestrator invocation
- Process exit code management
- User-facing error formatting

Business logic, data processing, and I/O operations belong in domain modules.

```python
# cli/main.py — GOOD: thin wrapper
config = resolve_config(options)
orchestrator = PipelineOrchestrator(config)
results = orchestrator.run()
raise SystemExit(1 if any(not r.success for r in results) else 0)
```

---

# 6. Library Export Pattern

`src/{package}/__init__.py` exports the public API for programmatic usage:

```python
# __init__.py
from .pipeline.orchestrator import PipelineOrchestrator, create_default_steps
from .config.settings import Settings, DEFAULT_CONFIG
from .utils.errors import AppError
```

This allows the tool to be used as both a CLI and an importable library.

---

# 7. Entry Point Configuration

Define CLI entry points in `pyproject.toml`:

```toml
[project.scripts]
my-tool = "my_tool.cli.main:app"
```

---

# 8. Test Organization

| Location | Purpose |
|----------|---------|
| `tests/unit/{module}/` | Unit tests mirroring src structure |
| `tests/integration/` | Cross-module integration tests |
| `tests/e2e/` | End-to-end CLI invocation tests |

- Mirror source structure in `tests/unit/` for discoverability
- E2E tests invoke the actual CLI binary and verify stdout/stderr/exit codes
- Use `conftest.py` fixtures for shared test setup
- Use `tests/fixtures/` for sample data

---

# 9. Guidelines Summary

- **Module-based structure is the default**
- **CLI layer is a thin wrapper — no business logic**
- **Core logic is importable as a library**
- **Tests mirror source structure in tests/unit/**
- **Domain modules must not depend on each other directly** — use shared utils or the pipeline orchestrator to coordinate
