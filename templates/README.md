# Project Templates

## Overview

A collection of templates for quickly bootstrapping new projects that comply with ai-dev-os guidelines.

## Available Templates

| Template | Tech Stack | Use Case |
|----------|-----------|----------|
| `python-cli/` | Python + Typer + Pydantic + structlog | CLI application |

> New framework templates should be added under `templates/{framework}/`.

## Usage

### 1. Create a New Project

```bash
# For a Python CLI project
mkdir my-project
cd my-project
uv init
```

### 2. Add ai-dev-os as a Submodule

```bash
# Using submodule-setup.sh (recommended)
bash /path/to/ai-dev-os/templates/python-cli/submodule-setup.sh

# Or manually
git submodule add https://github.com/yunbow/ai-dev-os.git docs/ai-dev-os
git submodule update --init
```

### 3. Copy Template Files

```bash
# CLAUDE.md
cp docs/ai-dev-os/templates/python-cli/CLAUDE.md.template ./CLAUDE.md

# Configuration files
cp docs/ai-dev-os/templates/python-cli/pyproject.toml ./pyproject.toml
cp docs/ai-dev-os/templates/python-cli/.gitignore ./.gitignore

# Claude Code skills
cp -r docs/ai-dev-os/templates/python-cli/.claude/ ./.claude/
```

### 4. Edit Project-Specific Settings

- Update the project name and project-specific guideline paths in `CLAUDE.md`
- Adjust the project name and dependencies in `pyproject.toml`
- Customize skill definitions in `.claude/skills/` to match your project

## Updating Templates

When templates are updated, existing projects are not automatically affected.
Apply updates to existing projects manually (review the diff and merge).

## Adding New Templates

```
templates/{framework}/
├── README.md               # Template description
├── submodule-setup.sh      # Submodule setup script
├── CLAUDE.md.template      # CLAUDE.md template
├── .claude/                # Claude Code skill definitions
│   └── skills/
├── .gitignore              # gitignore template
└── {config files}          # Framework-specific configuration (pyproject.toml, etc.)
```
