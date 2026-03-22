# Configuration Guidelines

> **[Replaceable]** This guide uses **Pydantic Settings** for configuration management. The same principles apply when using **dynaconf**, **python-dotenv**, or **tomli** for configuration loading.

This document defines how to design configuration resolution, defaults, and validation for CLI tools.

---

## 1. Configuration Resolution Order

Configuration is resolved in priority order (higher number wins):

```text
1. Default values (model field defaults)
2. Configuration file (e.g., tool.config.toml)
3. Environment variables
4. CLI arguments
```

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    output_dir: str = "./output"
    format: str = "md"
    verbose: bool = False

    model_config = SettingsConfigDict(
        env_prefix="MY_TOOL_",
        toml_file="tool.config.toml",
    )

def resolve_config(cli_options: dict) -> Settings:
    # Pydantic Settings handles layers 1-3 automatically
    # Layer 4: CLI overrides
    overrides = {k: v for k, v in cli_options.items() if v is not None}
    return Settings(**overrides)
```

---

## 2. Default Configuration

Define all defaults in the Pydantic model as field defaults:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    # Input
    target_url: str | None = None
    source_dir: str | None = None

    # Output
    output_dir: str = "./output"
    format: str = "md"
    lang: str = "en"

    # Behavior
    verbose: bool = False
    pause: bool = True
    modules: list[str] | None = None

    # Cache
    cache_enabled: bool = True
    cache_ttl: str = "24h"
    work_dir: str = ".work"

    model_config = SettingsConfigDict(
        env_prefix="MY_TOOL_",
    )
```

### Rules

- Every config field **must** have a default value (or be explicitly `None`)
- Defaults should produce a **working configuration** with minimal user input
- Export the `Settings` class for use in tests and documentation

---

## 3. Configuration File

## 3.1 File Format

Use TOML as the primary config format (Python ecosystem standard):

```toml
## tool.config.toml
output_dir = "./reports"
format = "pdf"
modules = ["business", "ux", "competitor"]

[cache]
ttl = "48h"
```

## 3.2 File Discovery

Search for configuration files in standard locations:

```python
CONFIG_FILE_NAMES = [
    "tool.config.toml",
    "pyproject.toml",  # [tool.my-tool] section
    ".toolrc.toml",
]
```

Or accept an explicit path via `--config <path>`.

## 3.3 Validation

Pydantic validates the configuration automatically:

```python
from pydantic import field_validator

class Settings(BaseSettings):
    format: str = "md"
    cache_ttl: str = "24h"

    @field_validator("format")
    @classmethod
    def validate_format(cls, v: str) -> str:
        allowed = {"md", "json", "pdf"}
        if v not in allowed:
            raise ValueError(f"format must be one of {allowed}, got '{v}'")
        return v

    @field_validator("cache_ttl")
    @classmethod
    def validate_ttl(cls, v: str) -> str:
        import re
        if not re.match(r"^\d+(ms|s|m|h|d)$", v):
            raise ValueError(f"Invalid TTL format: '{v}' (expected e.g. '24h', '30m')")
        return v
```

---

## 4. Environment Variables

## 4.1 Naming Convention

```text
{TOOL_NAME}_{SETTING} in SCREAMING_SNAKE_CASE
```

Example:

```python
MY_TOOL_OUTPUT_DIR=./reports
MY_TOOL_VERBOSE=true
MY_TOOL_CACHE_TTL=48h
```

## 4.2 Implementation

Pydantic Settings handles environment variables automatically via `env_prefix`:

```python
class Settings(BaseSettings):
    output_dir: str = "./output"
    verbose: bool = False

    model_config = SettingsConfigDict(
        env_prefix="MY_TOOL_",
        # MY_TOOL_OUTPUT_DIR → output_dir
        # MY_TOOL_VERBOSE → verbose
    )
```

---

## 5. Derived Configuration

Some values are computed from resolved configuration:

```python
from pydantic import computed_field

class Settings(BaseSettings):
    target_url: str | None = None
    source_dir: str | None = None

    @computed_field
    @property
    def input_mode(self) -> str:
        has_url = bool(self.target_url)
        has_source = bool(self.source_dir)
        if has_url and has_source:
            return "hybrid"
        return "url" if has_url else "source"
```

---

## 6. pyproject.toml Integration

Support reading tool-specific configuration from `pyproject.toml`:

```toml
[tool.my-tool]
output_dir = "./reports"
format = "pdf"
modules = ["business", "ux"]
```

```python
import tomllib
from pathlib import Path

def load_pyproject_config() -> dict:
    pyproject = Path("pyproject.toml")
    if not pyproject.exists():
        return {}
    with open(pyproject, "rb") as f:
        data = tomllib.load(f)
    return data.get("tool", {}).get("my-tool", {})
```

---

## 7. Summary

- **Resolution order: defaults → file → env → CLI** (CLI wins)
- **Every field has a default** — minimal input produces a working config
- **Validate with Pydantic validators** at load time
- **Derive computed values** with `@computed_field`
- **Single Settings model** used throughout the application
- **Environment variables** follow `TOOL_NAME_SETTING` convention
- **TOML is the standard config format** for Python projects
