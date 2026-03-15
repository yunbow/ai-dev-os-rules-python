# Overview

> **Note:** The technology stack below is a sample configuration. Replace libraries based on your project's requirements. Files marked with `[Replaceable]` in this directory contain library-specific patterns — update the corresponding files when switching libraries.

## Purpose
This is a guideline for designing robust, maintainable CLI tools with Python, balancing usability, extensibility, testability, and developer productivity.

## Technology Stack
- Python 3.12+
  - CLI Argument Parsing (type-hint driven)
  - Subcommand Routing
  - Process Lifecycle Management
  - File I/O & Streaming
- CLI Framework: Typer (built on Click)
- Validation: Pydantic
- Testing: pytest
- Build / Packaging: uv (or pip + pyproject.toml)
- Task Runner: uv run / make
- Type Checking: mypy or pyright

### Replaceable Libraries

| Category | Current | Alternatives | Related Files |
|----------|---------|-------------|---------------|
| CLI Framework | Typer | Click, argparse, Fire | `cli-design.md` |
| Validation | Pydantic | attrs, dataclasses + manual | `config.md` |
| Testing | pytest | unittest, ward | — |
| Configuration | Pydantic Settings | dynaconf, python-dotenv, tomli | `config.md` |
| Logging | structlog | logging (stdlib), loguru | — |

## Basic Principles
- Single-responsibility subcommands with clear input/output contracts
- Fail fast with meaningful error messages and exit codes
- Support both interactive and non-interactive (CI) environments
- Pipeline-based architecture for multi-step processing
- Deterministic output via caching and resumable execution
