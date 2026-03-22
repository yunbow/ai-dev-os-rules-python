# Process Lifecycle Guidelines

This document defines how to manage exit codes, signal handling, stdin/stdout, and process cleanup for CLI tools.

---

## 1. Exit Codes

## 1.1 Standard Exit Codes

| Code | Meaning | When |
|------|---------|------|
| `0` | Success | All steps completed successfully |
| `1` | Partial failure | Some steps failed but process completed |
| `2` | Fatal error | Unrecoverable error (invalid input, unhandled exception) |

## 1.2 Implementation

```python
## In CLI action handler
@app.command()
def analyze(...) -> None:
    try:
        results = orchestrator.run()
        has_failure = any(not r.success for r in results)
        raise typer.Exit(code=1 if has_failure else 0)
    except AppError as e:
        typer.echo(f"[{e.code}] Error: {e}", err=True)
        raise typer.Exit(code=2)
    except Exception as e:
        typer.echo(f"Error: {e}", err=True)
        raise typer.Exit(code=2)
```

## 1.3 Rules

- **Always exit explicitly** — do not let the process hang
- **Use `raise typer.Exit()` or `raise SystemExit()` only in the CLI layer** — core logic raises exceptions, CLI layer decides exit code
- **Never use `sys.exit()` in library code** — it prevents proper cleanup and makes code untestable

---

## 2. Signal Handling

## 2.1 Graceful Shutdown

Handle termination signals to clean up resources:

```python
import signal
import sys

def setup_signal_handlers(orchestrator: PipelineOrchestrator) -> None:
    def handler(signum: int, frame) -> None:
        logger.info("Shutting down...")
        orchestrator.save_state()
        sys.exit(128 + signum)

    signal.signal(signal.SIGINT, handler)
    signal.signal(signal.SIGTERM, handler)
```

## 2.2 Signal Exit Codes

| Signal | Exit Code | Meaning |
|--------|-----------|---------|
| SIGINT | 130 | User interrupted (Ctrl+C) |
| SIGTERM | 143 | Process terminated |

## 2.3 KeyboardInterrupt Pattern

Python converts SIGINT to `KeyboardInterrupt`. Handle it in the CLI layer:

```python
@app.command()
def analyze(...) -> None:
    try:
        results = orchestrator.run()
    except KeyboardInterrupt:
        typer.echo("\nInterrupted. State saved. Resume with: my-tool resume", err=True)
        orchestrator.save_state()
        raise typer.Exit(code=130)
```

---

## 3. Error Hierarchy

## 3.1 Base Error Class

```python
class AppError(Exception):
    def __init__(self, message: str, code: str = "UNKNOWN_ERROR", severity: str = "fatal") -> None:
        super().__init__(message)
        self.code = code
        self.severity = severity
```

## 3.2 Domain-Specific Errors

Create subclasses for each error domain:

```python
class InputValidationError(AppError):
    def __init__(self, message: str) -> None:
        super().__init__(message, "INPUT_VALIDATION", "fatal")

class ConnectionError(AppError):
    def __init__(self, url: str, cause: str | None = None) -> None:
        msg = f"Cannot connect to: {url}" + (f" ({cause})" if cause else "")
        super().__init__(msg, "CONNECTION_ERROR", "warning")
        self.url = url
```

## 3.3 Error Display

Format errors differently based on verbosity:

```python
def format_error(error: Exception, verbose: bool = False) -> str:
    if isinstance(error, AppError):
        msg = f"[{error.code}] Error: {error}"
        if verbose:
            import traceback
            msg += "\n" + traceback.format_exc()
        return msg
    return f"Error: {error}"
```

---

## 4. Standard I/O

## 4.1 Channel Separation

| Stream | Usage |
|--------|-------|
| `stdout` | Program output (results, data, reports) |
| `stderr` | Diagnostics (logs, progress, errors, warnings) |

This enables piping: `my-tool analyze > result.json 2> log.txt`

## 4.2 Logger Design

Use structlog or stdlib logging configured to write to stderr:

```python
import logging
import sys

def setup_logger(name: str, verbose: bool = False) -> logging.Logger:
    logger = logging.getLogger(name)
    handler = logging.StreamHandler(sys.stderr)
    handler.setFormatter(logging.Formatter("[%(levelname)s][%(name)s] %(message)s"))
    logger.addHandler(handler)
    logger.setLevel(logging.DEBUG if verbose else logging.INFO)
    return logger
```

With structlog:

```python
import structlog

structlog.configure(
    processors=[
        structlog.dev.ConsoleRenderer(),
    ],
    logger_factory=structlog.PrintLoggerFactory(file=sys.stderr),
)
```

---

## 5. Entry Point

## 5.1 pyproject.toml Configuration

```toml
[project]
name = "my-tool"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "typer>=0.9",
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
]

[project.scripts]
my-tool = "my_tool.cli.main:app"
```

## 5.2 CLI Entry Point

```python
#!/usr/bin/env python3
## src/my_tool/cli/main.py
import typer

app = typer.Typer()

## Register commands...

if __name__ == "__main__":
    app()
```

## 5.3 Development Runner

```bash
## Run directly during development
uv run my-tool analyze --url https://example.com

## Or via python -m
python -m my_tool.cli.main analyze --url https://example.com
```

---

## 6. Resource Cleanup

## 6.1 Context Manager Pattern

Use context managers for resources that need cleanup:

```python
from contextlib import contextmanager

@contextmanager
def managed_browser():
    browser = launch_browser()
    try:
        yield browser
    finally:
        browser.close()

## Usage
with managed_browser() as browser:
    collect_data(browser, config)
```

## 6.2 Async Cleanup

For async resources:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_session():
    async with aiohttp.ClientSession() as session:
        yield session
```

## 6.3 Cleanup Checklist

| Resource | Cleanup Action |
|----------|---------------|
| Browser instances (Playwright) | `browser.close()` |
| File handles | Context manager or `handle.close()` |
| Temporary directories | `shutil.rmtree(tmp_dir)` |
| Child processes | `process.terminate()` |
| Pipeline state | `orchestrator.save_state()` |
| HTTP sessions | `session.close()` |
| Database connections | `connection.close()` |

---

## 7. Summary

- **Exit codes: 0 (success), 1 (partial failure), 2 (fatal)**
- **Handle SIGINT/SIGTERM** for graceful shutdown
- **`typer.Exit()` only in CLI layer** — never in library code
- **stdout for data, stderr for diagnostics**
- **Error hierarchy** with codes and severity levels
- **Always clean up resources** with context managers
- **Module-scoped loggers** with configurable verbosity
