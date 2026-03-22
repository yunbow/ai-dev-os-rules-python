# Contributing to AI Dev OS Rules — Python

English | [日本語](docs/i18n/ja/CONTRIBUTING.md) | [简体中文](docs/i18n/zh-CN/CONTRIBUTING.md) | [한국어](docs/i18n/ko/CONTRIBUTING.md) | [Español](docs/i18n/es/CONTRIBUTING.md)

Thank you for your interest in contributing!

## How to Contribute

### Reporting Issues

- Use [GitHub Issues](https://github.com/yunbow/ai-dev-os-rules-python/issues)
- Specify which guideline file and rule is affected

### Pull Requests

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Commit with a descriptive message
5. Open a Pull Request

### What to Contribute

| Directory | What We Need | Guidelines |
|-----------|-------------|------------|
| `01_philosophy/` | Improvements to sample content | Rewrite in your native language. Keep abstract — no tool/framework names. |
| `02_decision-criteria/` | New decision axes, improved thresholds | Rewrite in your native language. No framework-specific details. |
| `03_guidelines/common/` | Rule improvements, new Before/After examples, new rules | **Cross-repo sync required** — changes must be copied to `ai-dev-os-rules-typescript`. See below. |
| `03_guidelines/frameworks/` | Framework-specific patterns, new framework support | Follow the responsibility separation: common/ = "what to do", frameworks/ = "how to implement". |
| `templates/` | Template improvements, new framework templates | For new projects only. Do not auto-apply to existing projects. |

### Cross-Repository Sync for `common/`

The `03_guidelines/common/` directory is shared across rules repositories. When updating any file:

1. Make the change in this repository first
2. Copy the updated file(s) to `ai-dev-os-rules-typescript`
3. Commit to each repository with the same message
4. Exception: `code.md` may have language-specific examples — sync the rules, adjust examples

**Quick check for drift:**

```bash
diff -rq 03_guidelines/common/ ../ai-dev-os-rules-typescript/03_guidelines/common/
```

### Adding a New Framework

1. Create `03_guidelines/frameworks/{name}/overview.md` and `project-structure.md`
2. No duplication with `common/` — only framework-specific patterns
3. Create `templates/{name}/` with CLAUDE.md.template
4. Update README.md directory structure
5. Release as MINOR version or higher

### Versioning

| Change Type | Version |
|-------------|---------|
| Philosophy/decision-criteria changes | MAJOR |
| New guidelines, framework additions | MINOR |
| Typo fixes, example improvements | PATCH |

### Translation Guide

- Translations are in `docs/i18n/{lang}/`
- Currently supported: `ja`, `zh-CN`, `ko`, `es`
- Guidelines in `03_guidelines/` must be in English (for AI compatibility)
- `01_philosophy/` and `02_decision-criteria/` can be in any language

## Code of Conduct

Be respectful, constructive, and inclusive.

---

Languages: English | [日本語](docs/i18n/ja/CONTRIBUTING.md) | [简体中文](docs/i18n/zh-CN/CONTRIBUTING.md) | [한국어](docs/i18n/ko/CONTRIBUTING.md) | [Español](docs/i18n/es/CONTRIBUTING.md)
