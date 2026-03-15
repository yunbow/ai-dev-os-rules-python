# Operation Guide

English | [日本語](./i18n/ja/operation-guide.md) | [简体中文](./i18n/zh-CN/operation-guide.md) | [한국어](./i18n/ko/operation-guide.md) | [Español](./i18n/es/operation-guide.md)

This document defines the operational rules for updating ai-dev-os.

---

## 1. Update Policy by Section

### 1.1 Update Frequency and Impact Scope

| Section | Update Frequency | Impact Scope | Notes |
|-----------|---------|---------|--------|
| `01_philosophy/` | Extremely rare | All projects | Changes to core philosophy require a MAJOR version bump |
| `02_decision-criteria/` | Rare | All projects | Changes to decision criteria require consistency checks with downstream guidelines |
| `03_guidelines/common/` | Medium | All projects | Changes to common standards require verification of impact on all frameworks |
| `03_guidelines/frameworks/` | High | Projects using the relevant FW | Keep up with framework updates |
| `04_ai-prompts/` | Medium | AI assistant users | Prompt improvements can be made freely |
| `templates/` | Medium | New projects only | No impact on existing projects |

### 1.2 Rules for Updates

**Common to all sections:**
- Do not include project-specific descriptions (specific service names, specific domain terminology)
- When concrete examples are needed, use generic examples with placeholders such as `{domain}`, `{Payment Service}`, etc.
- After changes, update the directory structure in `README.md` to reflect the latest state

**Language Policy:**
- `01_philosophy/` and `02_decision-criteria/` contain **sample content in English**. After cloning, rewrite these in your **native language** — abstract thinking and decision-making frameworks are best expressed in the native language to preserve nuance
- All other sections (`03_guidelines/`, `04_ai-prompts/`, `templates/`) must be written in **English** — for AI compatibility and international accessibility
- Multilingual operation guides are maintained in `docs/i18n/` (JA, ZH, KO, ES)
- When adding or updating content, always follow this language policy

**Updating `01_philosophy/`:**
- The repository contains sample content. After cloning, rewrite in your **native language**
- When adding new principles, verify there are no contradictions with existing principles
- Deletions and major changes are treated as MAJOR versions
- Also verify consistency with `02_decision-criteria/` and below

**Updating `02_decision-criteria/`:**
- The repository contains sample content. After cloning, rewrite in your **native language**
- When changing decision criteria, check whether the corresponding sections in `03_guidelines/` also need updates
- When adding new decision axes, verify whether corresponding guidelines exist

**Updating `03_guidelines/`:**
- Be careful not to duplicate content between `common/` and `frameworks/`
- Periodically review whether content that should be shared is written only in framework-specific files
- See "2. Adding Framework Guidelines" for details

**Cross-Repository Sync for `03_guidelines/common/`:**
See [CONTRIBUTING.md](../../CONTRIBUTING.md) for the full sync procedure, file-by-file sync table, and quick diff command.


**Updating `templates/`:**
- Templates are for new projects. Do not auto-apply to existing projects
- Configuration files (pyproject.toml, ruff config, etc.) must be consistent with `03_guidelines/` standards

---

## 2. Adding Framework Guidelines

### 2.1 Criteria for Addition

Conditions for adding new framework guidelines:

| Condition | Required/Recommended |
|------|----------|
| Used in 2 or more projects | Required |
| Framework-specific patterns that cannot be expressed in `common/` | Required |
| Framework is at stable release (v1.0+) | Recommended |
| Expected to be maintained long-term | Recommended |

### 2.2 Directory Structure

```
03_guidelines/frameworks/{framework-name}/
├── overview.md            # Technology stack definition (required)
├── project-structure.md   # Directory structure (required)
└── ...                    # Framework-specific guidelines
```

### 2.3 Addition Steps

#### Step 1: Create overview.md

The required entry point for all framework guidelines. Include:

```markdown
# {Framework} Technology Stack

## Core
- Framework: {Name} v{Version}
- Language: Python (with type hints, mypy/pyright strict mode)
- Package Manager: {uv / pip / poetry}

## Recommended Libraries
{List recommended libraries by category}

## Prerequisites
{Specify relationship with common/ guidelines}
```

#### Step 2: Create project-structure.md

Directory structure guidelines. Follow these principles:

- Apply concepts already defined in `common/` (vertical slices, dependency rules, etc.) as-is
- Only describe framework-specific directory conventions
- Use placeholders like `{domain}` for concrete examples

#### Step 3: Add Framework-Specific Guidelines

Relationship with corresponding `common/` guidelines:

| Pattern | Approach |
|---------|------|
| Applies common content as-is | Do not create a file on the framework side |
| Content that extends common | Create on framework side, referencing common and describing only the differences |
| Framework-specific concepts | Create only on the framework side |

**File naming rules:**
- Use the same name when corresponding to `common/` (e.g., `common/security.md` → `python-cli/security.md`)
- Use descriptive names for framework-specific concepts (e.g., `dependency-injection.md`, `cli-commands.md`)

#### Step 4: Create Templates

```
templates/{framework-name}/
├── CLAUDE.md.template     # CLAUDE.md template (required)
├── submodule-setup.sh     # Setup script (recommended)
├── .claude/skills/        # Claude Code skills (recommended)
└── {configuration files}  # pyproject.toml, ruff config, etc.
```

#### Step 5: Update README.md

- Add the new framework to the directory structure
- Add framework-specific instructions to the introduction section if needed

### 2.4 Responsibility Separation Principle with common/

```
common/          → "What to do" (language/FW-independent rules)
frameworks/xxx/  → "How to implement" (FW-specific implementation patterns)
```

**Example: Validation**
- `common/validation.md` → "Server-side validation is required" "Pydantic recommended"
- `frameworks/python-cli/validation.md` → "Pydantic model validation with Typer argument parsing"
- `frameworks/fastapi/validation.md` → "Request body validation with Pydantic models and dependency injection"

**Example: Error Handling**
- `common/error-handling.md` → "Error classification" "Principles for user display"
- `frameworks/python-cli/error-handling.md` → "Custom exception hierarchy with Typer exit codes"

### 2.5 Addition Checklist

- [ ] overview.md and project-structure.md have been created
- [ ] No project-specific descriptions are included
- [ ] No content duplication with `common/`
- [ ] Appropriate references to `common/` where applicable
- [ ] `templates/{framework}/CLAUDE.md.template` has been created
- [ ] Directory structure in `README.md` has been updated
- [ ] Released as MINOR version or higher

---

## 3. Versioning and Change Management

See [CONTRIBUTING.md](../../CONTRIBUTING.md) for versioning rules (MAJOR/MINOR/PATCH).

**Breaking change handling:** File deletion or renaming requires a MAJOR version. When moving files, leave a one-line redirect at the old path and remove it in the next MAJOR version.

---

Languages: English | [日本語](i18n/ja/operation-guide.md) | [简体中文](i18n/zh-CN/operation-guide.md) | [한국어](i18n/ko/operation-guide.md) | [Español](i18n/es/operation-guide.md)
