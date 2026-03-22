# AI Dev OS Rules — Python

[![Lint & Link Check](https://github.com/yunbow/ai-dev-os-rules-python/actions/workflows/lint.yml/badge.svg)](https://github.com/yunbow/ai-dev-os-rules-python/actions/workflows/lint.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

> 4-layer guidelines for Python projects (FastAPI, Flask, Typer CLI, etc.).
> Add as a git submodule, referenced by AI coding assistants.

**Part of the [AI Dev OS](https://github.com/yunbow/ai-dev-os) ecosystem.**

## Why These Rules?

AI Dev OS Rules give your AI coding assistant **concrete, verifiable standards** instead of vague instructions:

- **13 common rules** — Naming, error handling, security, testing, logging, i18n, and more
- **Framework-specific rules** — FastAPI patterns, Python CLI best practices
- **Conflict resolution built-in** — Specificity Cascade resolves rule priority automatically
- **Versioned & auditable** — Pin to a tag, diff changes, review in PRs

## Quick Start

```bash
npx ai-dev-os init --rules python --plugin claude-code
```

> Sets up everything automatically. See [AI Dev OS CLI](https://github.com/yunbow/ai-dev-os-cli).

<details>
<summary>Manual setup</summary>

### Add as submodule

```bash
cd /path/to/your-project
git submodule add https://github.com/yunbow/ai-dev-os-rules-python.git docs/ai-dev-os
git submodule update --init
```

### Set up using templates (for Python CLI)

```bash
bash docs/ai-dev-os/templates/python-cli/submodule-setup.sh
```

### Edit CLAUDE.md

Copy `templates/python-cli/CLAUDE.md.template` to `./CLAUDE.md` and fill in your project name and project-specific guidelines.

### Update submodule

```bash
git submodule update --remote docs/ai-dev-os
```

</details>

## What's Included

| Layer | Path | Contents |
|-------|------|----------|
| L1 — Philosophy | `01_philosophy/` | Principles, mental models, anti-patterns |
| L2 — Decision Criteria | `02_decision-criteria/` | Abstraction, tech selection, architecture, errors, security |
| L3 — Common Guidelines | `03_guidelines/common/` | 13 rules: code, naming, validation, errors, logging, security, testing, etc. |
| L3 — Framework Guidelines | `03_guidelines/frameworks/` | [FastAPI](03_guidelines/frameworks/fastapi/README.md), [Python CLI](03_guidelines/frameworks/python-cli/README.md) |
| Templates | `templates/` | [Python CLI scaffolding](templates/README.md) |

## Specificity Cascade

When rules conflict, **lower number wins**.

| Priority | Layer | Example |
|----------|-------|---------|
| 1 (Highest) | Framework-specific guidelines | `03_guidelines/frameworks/python-cli/*` |
| 2 | Common guidelines | `03_guidelines/common/*` |
| 3 | Decision criteria | `02_decision-criteria/*` |
| 4 | Design philosophy | `01_philosophy/*` |

<details>
<summary>Directory Structure</summary>

```text
ai-dev-os/
├── docs/
│   ├── operation-guide.md        # Operation & Contribution Guide
│   └── i18n/                     # Multilingual Guides
│       ├── ja/                   #   Japanese
│       ├── zh-CN/                #   Simplified Chinese
│       ├── ko/                   #   Korean
│       └── es/                   #   Spanish
│
├── 01_philosophy/                # Design Philosophy [Sample - rewrite in your native language]
│   ├── principles.md             #   Three Pillars: Correctness, Observability, Pragmatism
│   ├── mental-models.md          #   10 Thinking Frameworks
│   └── anti-patterns.md          #   Patterns to Avoid (with code examples)
│
├── 02_decision-criteria/         # Decision Criteria [Sample - rewrite in your native language]
│   ├── abstraction.md            #   Timing and Thresholds for Abstraction
│   ├── technology-selection.md   #   Technology Selection Framework
│   ├── architecture.md           #   API Design, State Management, Module Organization
│   ├── error-strategy.md         #   Error Classification, Retry, Result Pattern
│   └── security-vs-ux.md        #   Security Measure Priority and Balance
│
├── 03_guidelines/                # Guidelines [EN]
│   ├── common/                   #   Common (Language/FW Independent)
│   │   ├── code.md               #     Coding Standards
│   │   ├── naming.md             #     Naming Conventions
│   │   ├── validation.md         #     Validation
│   │   ├── error-handling.md     #     Error Handling
│   │   ├── logging.md            #     Logging
│   │   ├── security.md           #     Security
│   │   ├── rate-limiting.md      #     Rate Limiting
│   │   ├── testing.md            #     Testing
│   │   ├── performance.md        #     Performance
│   │   ├── cors.md               #     CORS
│   │   ├── env.md                #     Environment Variables
│   │   ├── cicd.md               #     CI/CD
│   │   └── i18n.md               #     Internationalization
│   │
│   └── frameworks/               #   Framework-Specific (see each README.md)
│       ├── fastapi/              #     → [README.md](03_guidelines/frameworks/fastapi/README.md)
│       └── python-cli/           #     → [README.md](03_guidelines/frameworks/python-cli/README.md)
│
│
└── templates/                    # Project Templates [EN]
    └── python-cli/               #     → [README.md](templates/README.md)
```

</details>

<details>
<summary>Operations & Versioning</summary>

For update policies, framework addition steps, and versioning details, see **[docs/operation-guide.md](./docs/operation-guide.md)**.

### Update Frequency Guide

| Section | Frequency | Impact Scope |
|---------|-----------|--------------|
| `01_philosophy/` | Extremely rare | All projects (MAJOR change) |
| `02_decision-criteria/` | Rare | All projects |
| `03_guidelines/common/` | Medium | All projects |
| `03_guidelines/frameworks/` | High | Projects using the relevant FW only |
| `templates/` | Medium | New projects only |

### Adding Frameworks

To add a new framework (e.g., FastAPI, Flask, Django):

1. Create `overview.md` and `project-structure.md` under `03_guidelines/frameworks/{framework}/`
2. Enforce responsibility separation with `common/` (common rules -> common, FW-specific patterns -> frameworks)
3. Prepare templates under `templates/{framework}/`
4. Update the directory structure in this README

See [docs/operation-guide.md](./docs/operation-guide.md) for detailed steps and checklist.

**Versioning** — Managed with semantic versioning (git tags).

| Change Type | Version | Example |
|-------------|---------|---------|
| Major changes to philosophy / decision-criteria | MAJOR | v2.0.0 |
| Guideline additions/improvements | MINOR | v1.1.0 |
| Typo fixes, supplementary additions | PATCH | v1.0.1 |

Pin the submodule to a specific tag:

```bash
cd docs/ai-dev-os
git checkout v1.2.0
cd ../..
git add docs/ai-dev-os
git commit -m "chore: pin ai-dev-os to v1.2.0"
```

</details>

## Language Policy

- `01_philosophy/` and `02_decision-criteria/` contain **sample content in English**. After cloning, rewrite these in your **native language** to preserve nuance in your team's abstract thinking and decision-making frameworks.
- All other sections are written in **English** for AI compatibility and international accessibility.
- Multilingual operation guides are available in `docs/i18n/`.

## Related

| Repository | Description |
|---|---|
| [ai-dev-os](https://github.com/yunbow/ai-dev-os) | Framework specification and theory |
| [rules-typescript](https://github.com/yunbow/ai-dev-os-rules-typescript) | TypeScript / Next.js / Node.js guidelines |
| [plugin-claude-code](https://github.com/yunbow/ai-dev-os-plugin-claude-code) | Skills, Hooks, and Agents for Claude Code |
| [plugin-kiro](https://github.com/yunbow/ai-dev-os-plugin-kiro) | Steering Rules and Hooks for Kiro |
| [plugin-cursor](https://github.com/yunbow/ai-dev-os-plugin-cursor) | Cursor Rules (.mdc) for guideline-driven development |
| [cli](https://github.com/yunbow/ai-dev-os-cli) | Setup automation — `npx ai-dev-os init` |
| [benchmark](https://github.com/yunbow/ai-dev-os-benchmark) | Quantitative benchmark — guideline impact on AI code quality |

## License

[MIT](./LICENSE)

---

Languages: English | [日本語](docs/i18n/ja/README.md) | [简体中文](docs/i18n/zh-CN/README.md) | [한국어](docs/i18n/ko/README.md) | [Español](docs/i18n/es/README.md)
