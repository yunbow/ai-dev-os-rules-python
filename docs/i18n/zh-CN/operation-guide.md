# 运营与贡献指南

本文档定义了 ai-dev-os 的更新和扩展运营规则。

---

## 1. 各部分的更新策略

### 1.1 更新频率与影响范围

| 部分 | 更新频率 | 影响范围 | 注意事项 |
|------|---------|---------|---------|
| `01_philosophy/` | 极少 | 所有项目 | 核心理念变更需要 MAJOR 版本升级 |
| `02_decision-criteria/` | 少 | 所有项目 | 决策标准变更需确认与下游指南的一致性 |
| `03_guidelines/common/` | 中等 | 所有项目 | 通用标准变更需验证对所有框架的影响 |
| `03_guidelines/frameworks/` | 频繁 | 相关框架的项目 | 跟随框架更新 |
| `04_ai-prompts/` | 中等 | AI 助手用户 | 提示改进可自由进行 |
| `templates/` | 中等 | 仅新项目 | 不影响现有项目 |

### 1.2 更新规则

**所有部分通用：**
- 不包含项目特定的描述（特定服务名称、特定领域术语）
- 需要具体示例时，使用 `{domain}`、`{Payment Service}` 等占位符的通用示例
- 变更后，将 `README.md` 的目录结构更新为最新状态

**语言政策：**
- `01_philosophy/` 和 `02_decision-criteria/` 包含**英语示例内容**——克隆后请**用母语重写**（抽象思维和决策框架用母语表达能保留细微差别）
- 其他所有部分（`03_guidelines/`、`04_ai-prompts/`、`templates/`）必须使用**英语**编写——为了 AI 兼容性和国际可访问性
- 多语言运营指南维护在 `docs/i18n/`（JA、ZH、KO、ES）
- 添加或更新内容时，始终遵循此语言政策

**更新 `01_philosophy/`：**
- 包含示例内容（英语）。克隆后请**用母语重写**
- 添加新原则时，验证与现有原则没有矛盾
- 删除和重大变更视为 MAJOR 版本
- 同时验证与 `02_decision-criteria/` 及以下部分的一致性

**更新 `02_decision-criteria/`：**
- 包含示例内容（英语）。克隆后请**用母语重写**
- 变更决策标准时，检查 `03_guidelines/` 的对应部分是否也需要更新
- 添加新的决策轴时，验证是否存在对应的指南

**更新 `03_guidelines/`：**
- 注意避免 `common/` 和 `frameworks/` 之间的内容重复
- 定期检查应共享的内容是否仅写在框架特定文件中
- 详见"2. 添加框架指南"

**更新 `04_ai-prompts/`：**
- 验证提示中引用的指南路径确实存在
- `skills/` 必须保持 Claude Code SKILL.md 格式（frontmatter + procedures）

**更新 `templates/`：**
- 模板用于新项目。不要自动应用到现有项目
- 配置文件（pyproject.toml、ruff 配置等）必须与 `03_guidelines/` 标准一致

---

## 2. 添加框架指南

### 2.1 添加标准

添加新框架指南的条件：

| 条件 | 必需/推荐 |
|------|----------|
| 在 2 个以上项目中使用 | 必需 |
| 无法在 `common/` 中表达的框架特定模式 | 必需 |
| 框架处于稳定版本（v1.0+） | 推荐 |
| 预期长期维护 | 推荐 |

### 2.2 目录结构

```
03_guidelines/frameworks/{framework-name}/
├── overview.md            # 技术栈定义（必需）
├── project-structure.md   # 目录结构（必需）
└── ...                    # 框架特定指南
```

### 2.3 添加步骤

#### Step 1: 创建 overview.md

所有框架指南的必需入口点。包含：

```markdown
# {Framework} Technology Stack

## Core
- Framework: {Name} v{Version}
- Language: Python（带类型提示，mypy/pyright strict 模式）
- Package Manager: {uv / pip / poetry}

## Recommended Libraries
{按类别列出推荐库}

## Prerequisites
{指定与 common/ 指南的关系}
```

#### Step 2: 创建 project-structure.md

目录结构指南。遵循以下原则：

- 将 `common/` 中已定义的概念（垂直切片、依赖规则等）原样应用
- 仅描述框架特定的目录约定
- 具体示例使用 `{domain}` 等占位符

#### Step 3: 添加框架特定指南

与 `common/` 对应指南的关系：

| 模式 | 方法 |
|------|------|
| 原样应用通用内容 | 不在框架侧创建文件 |
| 扩展通用内容 | 在框架侧创建，引用通用部分并仅描述差异 |
| 框架特定概念 | 仅在框架侧创建 |

**文件命名规则：**
- 与 `common/` 对应时使用相同名称（例如 `common/security.md` → `python-cli/security.md`）
- 框架特定概念使用描述性名称（例如 `dependency-injection.md`、`cli-commands.md`）

#### Step 4: 创建模板

```
templates/{framework-name}/
├── CLAUDE.md.template     # CLAUDE.md 模板（必需）
├── submodule-setup.sh     # 设置脚本（推荐）
├── .claude/skills/        # Claude Code skills（推荐）
└── {configuration files}  # pyproject.toml、ruff 配置等
```

#### Step 5: 更新 README.md

- 将新框架添加到目录结构中
- 如需要，在介绍部分添加框架特定说明

### 2.4 与 common/ 的职责分离原则

```
common/          → "做什么"（与语言/框架无关的规则）
frameworks/xxx/  → "如何实现"（框架特定的实现模式）
```

**示例：验证**
- `common/validation.md` → "必须进行服务端验证" "推荐使用 Pydantic"
- `frameworks/python-cli/validation.md` → "使用 Pydantic 模型与 Typer 参数解析的验证模式"
- `frameworks/fastapi/validation.md` → "使用 Pydantic 模型与依赖注入的请求体验证"

**示例：错误处理**
- `common/error-handling.md` → "错误分类" "用户展示原则"
- `frameworks/python-cli/error-handling.md` → "自定义异常层级与 Typer 退出码"

### 2.5 添加检查清单

- [ ] 已创建 overview.md 和 project-structure.md
- [ ] 不包含项目特定的描述
- [ ] 与 `common/` 无内容重复
- [ ] 在适当的地方引用了 `common/`
- [ ] 已创建 `templates/{framework}/CLAUDE.md.template`
- [ ] 已更新 `README.md` 中的目录结构
- [ ] 以 MINOR 版本或更高版本发布

---

## 3. 版本管理与变更管理

### 3.1 确定变更类型

| 变更 | 版本 | 示例 |
|------|------|------|
| 设计哲学的根本性变更 | MAJOR | 变更三大支柱、反转依赖规则 |
| 决策标准的重大变更 | MAJOR | 抽象阈值的根本性变更 |
| 添加框架指南 | MINOR | 创建 `frameworks/fastapi/` |
| 改进/扩展现有指南 | MINOR | 向安全指南添加新模式 |
| 添加提示/skills | MINOR | 添加新的 skill 定义 |
| 拼写修正/措辞改进 | PATCH | 修正错别字、替换示例 |
| 仅模板变更 | PATCH | 配置文件的微调 |

### 3.2 处理破坏性变更

以下被视为需要 MAJOR 版本的破坏性变更：

- 文件删除或重命名（会破坏 CLAUDE.md 的路径引用）
- 目录结构变更
- 废除或反转现有规则

**移动文件时：**
1. 在旧路径留下一行重定向文件："Moved to: `new-path`"
2. 在下一个 MAJOR 版本中移除重定向文件

---

Languages: [English](../../operation-guide.md) | [日本語](../ja/operation-guide.md) | 简体中文 | [한국어](../ko/operation-guide.md) | [Español](../es/operation-guide.md)
