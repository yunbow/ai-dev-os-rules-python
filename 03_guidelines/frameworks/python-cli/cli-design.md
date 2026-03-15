# CLI Design Guidelines

> **[Replaceable]** This guide uses **Typer** as the CLI framework. The same architectural principles apply when using alternatives such as **Click**, **argparse**, or **Fire**.

This document defines how to design subcommands, options, arguments, and user interactions for Python CLI tools.

---

# 1. Subcommand Design

## 1.1 One Command = One Responsibility

Each subcommand should have a single, well-defined purpose.

```
my-tool analyze   # Run analysis pipeline
my-tool resume    # Resume interrupted analysis
my-tool compare   # Compare multiple results
```

## 1.2 Command Registration Pattern

Use a Typer app with separate command functions:

```python
import typer

app = typer.Typer(name="my-tool", help="Tool description")

@app.command()
def analyze(
    url: Annotated[str | None, typer.Option("--url", "-u", help="Target URL")] = None,
    output: Annotated[str, typer.Option("--output", "-o", help="Output directory")] = "./output",
) -> None:
    """Run analysis."""
    ...

@app.command()
def compare(
    reports: Annotated[list[str], typer.Argument(help="Report directories (2 or more)")],
) -> None:
    """Compare multiple results."""
    ...

if __name__ == "__main__":
    app()
```

## 1.3 Subcommand Grouping

For complex CLIs, use sub-apps to group related commands:

```python
app = typer.Typer()
data_app = typer.Typer(help="Data management commands")
app.add_typer(data_app, name="data")

@data_app.command()
def fetch(...) -> None:
    """Fetch data from source."""
    ...

@data_app.command()
def clean(...) -> None:
    """Clean cached data."""
    ...
```

## 1.4 Subcommand Guidelines

| Rule | Rationale |
|------|-----------|
| Use **verbs** for command names | `analyze`, `compare`, `export` — not `analysis`, `comparison` |
| Keep to **1-2 words** | Short and memorable |
| Provide docstrings for every command | Shown in `--help` output |
| Use non-optional parameters for mandatory inputs | Typer enforces this via type hints |

---

# 2. Option & Argument Design

## 2.1 Option Conventions

Typer derives CLI options from function parameters and type annotations:

```python
from typing import Annotated

@app.command()
def analyze(
    # Required option (no default)
    url: Annotated[str, typer.Option("--url", "-u", help="Target URL")],
    # Optional with default
    output: Annotated[str, typer.Option("--output", "-o", help="Output directory")] = "./output",
    # Boolean flag
    verbose: Annotated[bool, typer.Option("--verbose", "-v", help="Enable verbose output")] = False,
    # Negatable boolean flag
    cache: Annotated[bool, typer.Option("--cache/--no-cache", help="Enable/disable cache")] = True,
) -> None:
    ...
```

| Convention | Example | Rationale |
|-----------|---------|-----------|
| Short + long form | `"--output", "-o"` | Usability for both beginners and power users |
| Provide defaults where possible | `= "./output"` | Minimize required configuration |
| Use `Annotated` with `typer.Option` | `Annotated[str, typer.Option(...)]` | Explicit metadata, good IDE support |
| Boolean flags use `--flag/--no-flag` | `--cache/--no-cache` | Standard CLI convention |

## 2.2 Common Options (Recommended)

Standardize these across all subcommands using a callback:

```python
@app.callback()
def common_options(
    verbose: Annotated[bool, typer.Option("--verbose", "-v", help="Enable verbose output")] = False,
    config: Annotated[str | None, typer.Option("--config", "-c", help="Configuration file path")] = None,
) -> None:
    """My CLI tool."""
    ...
```

## 2.3 Enum Options

Use `Enum` for constrained choices:

```python
from enum import Enum

class OutputFormat(str, Enum):
    markdown = "md"
    json = "json"
    pdf = "pdf"

@app.command()
def analyze(
    format: Annotated[OutputFormat, typer.Option("--format", "-f", help="Output format")] = OutputFormat.markdown,
) -> None:
    ...
```

## 2.4 Multi-Value Options

For list values:

```python
@app.command()
def analyze(
    modules: Annotated[list[str] | None, typer.Option("--module", "-m", help="Modules to run")] = None,
) -> None:
    # Usage: my-tool analyze --module business --module ux
    ...
```

---

# 3. Input Validation

## 3.1 Fail Fast

Validate inputs at the CLI layer before passing to core logic:

```python
@app.command()
def analyze(
    url: Annotated[str | None, typer.Option()] = None,
    source: Annotated[str | None, typer.Option()] = None,
) -> None:
    if not url and not source:
        typer.echo("Error: specify at least --url or --source.", err=True)
        raise typer.Exit(code=1)

    config = resolve_config(url=url, source=source)
    validation = validate_inputs(config)
    if not validation.valid:
        typer.echo(f"Error: {'; '.join(validation.warnings)}", err=True)
        raise typer.Exit(code=1)
```

## 3.2 Typer Built-in Validation

Use Typer's callback validators for common patterns:

```python
def validate_path(value: str) -> str:
    path = Path(value)
    if not path.exists():
        raise typer.BadParameter(f"Path does not exist: {value}")
    return value

@app.command()
def analyze(
    source: Annotated[str, typer.Option(callback=validate_path)],
) -> None:
    ...
```

## 3.3 Validation Layers

| Layer | Responsibility | Example |
|-------|---------------|---------|
| Typer built-in | Required params, types, enums | Type annotations |
| Callback validators | Path existence, format checks | `callback=validate_path` |
| CLI action body | Mutual exclusivity, basic sanity | `if not url and not source` |
| Pydantic model | Deep structural validation | Config file parsing |

---

# 4. Help & Documentation

## 4.1 Auto-Generated Help

Typer generates `--help` automatically from docstrings and `help=` parameters. Ensure:
- Every command has a docstring
- Every option/argument has `help=`
- Default values are specified (shown in help)

## 4.2 Rich Help Output

Typer supports Rich for formatted help:

```python
app = typer.Typer(rich_markup_mode="markdown")

@app.command()
def analyze(
    url: Annotated[str, typer.Option(help="Target **URL** to analyze")],
) -> None:
    """Run analysis pipeline.

    Collects data, processes it, and generates reports.
    """
    ...
```

## 4.3 Epilog Examples

Add usage examples at the end of help output:

```python
app = typer.Typer(
    epilog="Examples:\n  my-tool analyze --url https://example.com\n  my-tool analyze --source ./src --module business",
)
```

---

# 5. Output Design

## 5.1 Output Channels

| Channel | Purpose | Example |
|---------|---------|---------|
| `stdout` | Primary output (results, data) | Report content, JSON output |
| `stderr` | Diagnostics (logs, progress, errors) | Progress bars, warnings, errors |

This separation allows piping: `my-tool analyze --url ... > result.json`

```python
# Data to stdout
typer.echo(json.dumps(result))

# Diagnostics to stderr
typer.echo("Processing...", err=True)
```

## 5.2 Verbosity Levels

| Flag | Behavior |
|------|----------|
| (default) | INFO-level messages only |
| `--verbose` / `-v` | DEBUG-level messages included |
| `--quiet` / `-q` (optional) | Suppress all non-error output |

## 5.3 Progress Reporting

For long-running operations, use Rich progress or step-based output:

```python
import typer

with typer.progressbar(items, label="Processing") as progress:
    for item in progress:
        process(item)
```

Or step-based:

```
[1/4] Collecting data...
[2/4] Preprocessing...
[3/4] Summarizing... (15.2s)
[4/4] Generating reports...
```

---

# 6. Interactive vs Non-Interactive

## 6.1 Detection

```python
import sys

is_interactive = sys.stdin.isatty() and sys.stdout.isatty()
```

## 6.2 Behavior Differences

| Feature | Interactive | Non-Interactive (CI) |
|---------|------------|---------------------|
| Confirmation prompts | Show and wait | Skip (use `--yes` flag) |
| Progress display | Rich progress bars | Static line-by-line |
| Color output | Enabled | Disabled (respect `NO_COLOR`) |
| stdin prompts | Prompt user | Error or use defaults |

## 6.3 Confirmation Pattern

```python
if config.pause and sys.stdin.isatty():
    typer.confirm("Continue?", abort=True)
else:
    logger.debug("Non-interactive mode: skipping confirmation")
```

---

# 7. Summary

- **One subcommand = one responsibility**
- **Fail fast** with clear error messages at the CLI layer
- **Separate stdout (data) from stderr (diagnostics)**
- **Provide defaults** for all optional parameters
- **Support both interactive and CI environments**
- **Validate inputs in layers** — Typer type hints → callbacks → CLI body → Pydantic
- **Add help text and docstrings** for all commands and options
