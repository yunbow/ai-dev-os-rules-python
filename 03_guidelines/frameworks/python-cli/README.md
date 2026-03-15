# Python CLI Tool Guidelines

Framework-specific guidelines for building CLI tools with Python and Typer.
See [overview.md](./overview.md) for the full technology stack and replaceable library table.

## File List

| File | Topic | Replaceable Library |
|------|-------|-------------------|
| [overview.md](./overview.md) | Technology Stack & Replaceable Libraries | — |
| [project-structure.md](./project-structure.md) | Directory Structure & Architecture | — |
| [cli-design.md](./cli-design.md) | Command Design, Options & Arguments | Click, argparse, Fire |
| [pipeline.md](./pipeline.md) | Pipeline Orchestration & Step Management | — |
| [config.md](./config.md) | Configuration Resolution & Defaults | python-dotenv, dynaconf |
| [process-lifecycle.md](./process-lifecycle.md) | Exit Codes, Signals & I/O | — |

> Files marked with a replaceable library contain `[Replaceable]` notes at the top. The architectural principles remain the same regardless of the library chosen.
