# AI Dev OS Rules — Python

[![Lint & Link Check](https://github.com/yunbow/ai-dev-os-rules-python/actions/workflows/lint.yml/badge.svg)](https://github.com/yunbow/ai-dev-os-rules-python/actions/workflows/lint.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)

> Python 项目（FastAPI、Flask、Typer CLI 等）的 4 层指南。
> 作为 git submodule 添加，供 AI 编码助手引用。

**[AI Dev OS](https://github.com/yunbow/ai-dev-os) 生态系统的一部分。**

## 为什么选择这些规则？

AI Dev OS Rules 为您的 AI 编码助手提供**具体、可验证的标准**，而非模糊的指令：

- **13 条通用规则** — 命名、错误处理、安全、测试、日志、i18n 等
- **框架特定规则** — FastAPI 模式、Python CLI 最佳实践
- **内置冲突解决** — Specificity Cascade 自动解决规则优先级
- **版本化且可审计** — 固定到标签、查看差异、在 PR 中审查

## 快速开始

```bash
npx ai-dev-os init --rules python --plugin claude-code
```

> 自动完成所有设置。详见 [AI Dev OS CLI](https://github.com/yunbow/ai-dev-os-cli)。

<details>
<summary>手动设置</summary>

### 作为 submodule 添加

```bash
cd /path/to/your-project
git submodule add https://github.com/yunbow/ai-dev-os-rules-python.git docs/ai-dev-os
git submodule update --init
```

### 使用模板设置（Python CLI）

```bash
bash docs/ai-dev-os/templates/python-cli/submodule-setup.sh
```

### 编辑 CLAUDE.md

将 `templates/python-cli/CLAUDE.md.template` 复制为 `./CLAUDE.md`，并填写项目名称和特定指南。

### 更新 submodule

```bash
git submodule update --remote docs/ai-dev-os
```

</details>

## 包含内容

| 层级 | 路径 | 内容 |
|------|------|------|
| L1 — 设计哲学 | `01_philosophy/` | 原则、思维模型、反模式 |
| L2 — 决策标准 | `02_decision-criteria/` | 抽象化、技术选型、架构、错误、安全 |
| L3 — 通用指南 | `03_guidelines/common/` | 13 条规则：代码、命名、验证、错误、日志、安全、测试等 |
| L3 — 框架指南 | `03_guidelines/frameworks/` | [FastAPI](../../../03_guidelines/frameworks/fastapi/README.md)、[Python CLI](../../../03_guidelines/frameworks/python-cli/README.md) |
| 模板 | `templates/` | [Python CLI 脚手架](../../../templates/README.md) |

## Specificity Cascade

当规则冲突时，**编号越小优先级越高**。

| 优先级 | 层级 | 示例 |
|--------|------|------|
| 1（最高） | 框架特定指南 | `03_guidelines/frameworks/python-cli/*` |
| 2 | 通用指南 | `03_guidelines/common/*` |
| 3 | 决策标准 | `02_decision-criteria/*` |
| 4 | 设计哲学 | `01_philosophy/*` |

<details>
<summary>目录结构</summary>

```text
ai-dev-os/
├── docs/
│   ├── operation-guide.md        # 运维与贡献指南
│   └── i18n/                     # 多语言指南
│       ├── ja/                   #   日本語
│       ├── zh-CN/                #   简体中文
│       ├── ko/                   #   한국어
│       └── es/                   #   Español
│
├── 01_philosophy/                # 设计哲学 [示例 - 用母语重写]
│   ├── principles.md             #   三大支柱: Correctness, Observability, Pragmatism
│   ├── mental-models.md          #   10个思维框架
│   └── anti-patterns.md          #   应避免的模式（含代码示例）
│
├── 02_decision-criteria/         # 决策标准 [示例 - 用母语重写]
│   ├── abstraction.md            #   抽象化的时机与阈值
│   ├── technology-selection.md   #   技术选型判断框架
│   ├── architecture.md           #   API 设计、状态管理、模块组织
│   ├── error-strategy.md         #   错误分类、重试、Result 模式
│   └── security-vs-ux.md        #   安全措施优先级与平衡
│
├── 03_guidelines/                # 指南 [英语]
│   ├── common/                   #   通用（语言/框架无关）
│   │   ├── code.md               #     编码规范
│   │   ├── naming.md             #     命名规则
│   │   ├── validation.md         #     验证
│   │   ├── error-handling.md     #     错误处理
│   │   ├── logging.md            #     日志
│   │   ├── security.md           #     安全
│   │   ├── rate-limiting.md      #     速率限制
│   │   ├── testing.md            #     测试
│   │   ├── performance.md        #     性能
│   │   ├── cors.md               #     CORS
│   │   ├── env.md                #     环境变量
│   │   ├── cicd.md               #     CI/CD
│   │   └── i18n.md               #     国际化
│   │
│   └── frameworks/               #   框架特定（参见各 README.md）
│       ├── fastapi/              #     → [README.md](../../../03_guidelines/frameworks/fastapi/README.md)
│       └── python-cli/           #     → [README.md](../../../03_guidelines/frameworks/python-cli/README.md)
│
│
└── templates/                    # 项目模板 [英语]
    └── python-cli/               #     → [README.md](../../../templates/README.md)
```

</details>

<details>
<summary>运维与版本管理</summary>

更新策略、框架添加步骤和版本管理详情，请参阅 **[docs/operation-guide.md](../../../docs/operation-guide.md)**。

### 更新频率指南

| 部分 | 频率 | 影响范围 |
|------|------|---------|
| `01_philosophy/` | 极少 | 所有项目（MAJOR 变更） |
| `02_decision-criteria/` | 少 | 所有项目 |
| `03_guidelines/common/` | 中等 | 所有项目 |
| `03_guidelines/frameworks/` | 频繁 | 仅相关框架的项目 |
| `templates/` | 中等 | 仅新项目 |

### 添加框架

添加新框架（如 FastAPI, Flask, Django）时：

1. 在 `03_guidelines/frameworks/{framework}/` 下创建 `overview.md` 和 `project-structure.md`
2. 严格与 `common/` 进行职责分离（通用规则 → common，框架特定模式 → frameworks）
3. 在 `templates/{framework}/` 下准备模板
4. 更新本 README 的目录结构

详细步骤和清单请参阅 [docs/operation-guide.md](../../../docs/operation-guide.md)。

**版本管理** — 使用语义化版本（git 标签）管理。

| 变更类型 | 版本 | 示例 |
|---------|------|------|
| 哲学/决策标准的重大变更 | MAJOR | v2.0.0 |
| 指南添加/改进 | MINOR | v1.1.0 |
| 拼写修正、补充说明 | PATCH | v1.0.1 |

将 submodule 固定到特定标签：

```bash
cd docs/ai-dev-os
git checkout v1.2.0
cd ../..
git add docs/ai-dev-os
git commit -m "chore: pin ai-dev-os to v1.2.0"
```

</details>

## 语言政策

- `01_philosophy/` 和 `02_decision-criteria/` 包含**英语示例内容**。克隆后请**用母语重写**，以保留团队抽象思维和决策框架的细微差别。
- 其他所有部分用**英语**编写，以确保 AI 兼容性和国际可访问性。
- 多语言操作指南位于 `docs/i18n/`。

## 相关

| 仓库 | 说明 |
|---|---|
| [ai-dev-os](https://github.com/yunbow/ai-dev-os) | 框架规范与理论 |
| [rules-typescript](https://github.com/yunbow/ai-dev-os-rules-typescript) | TypeScript / Next.js / Node.js 指南 |
| [plugin-claude-code](https://github.com/yunbow/ai-dev-os-plugin-claude-code) | Claude Code 的 Skills、Hooks、Agents |
| [plugin-kiro](https://github.com/yunbow/ai-dev-os-plugin-kiro) | Kiro 的 Steering Rules 与 Hooks |
| [plugin-cursor](https://github.com/yunbow/ai-dev-os-plugin-cursor) | 指南驱动开发的 Cursor Rules (.mdc) |
| [cli](https://github.com/yunbow/ai-dev-os-cli) | 设置自动化 — `npx ai-dev-os init` |
| [benchmark](https://github.com/yunbow/ai-dev-os-benchmark) | 定量基准测试 — 指南对代码质量的影响 |

## 许可证

[MIT](../../../LICENSE)

---

Languages: [English](../../../README.md) | [日本語](../ja/README.md) | 简体中文 | [한국어](../ko/README.md) | [Español](../es/README.md)
