# Skill 开发框架——跨平台移植指南

> 如何将本框架移植到其他 AI 编码 agent 工具

---

## 1. 各工具指令格式总览

| 工具 | 指令文件位置 | 格式 | 前置元数据 | 作用域机制 | 全局支持 |
|------|-------------|------|-----------|-----------|---------|
| **Claude Code** | `skills/<name>/SKILL.md` | Markdown + YAML | `name`, `description`, `type`, `language` | description 语义匹配 | ✅ `~/.claude/` |
| **Cursor** | `.cursor/rules/*.mdc` | Markdown + YAML | `description`, `globs`, `alwaysApply` | glob 模式 + AI 判断 | ✅ IDE 设置 |
| **GitHub Copilot** | `.github/copilot-instructions.md`<br>`.github/instructions/*.instructions.md` | Markdown + YAML | `applyTo`, `excludeAgent` | glob 模式 + 目录层级 | ✅ Web 设置 |
| **OpenAI Codex** | `AGENTS.md`（任意目录层级） | 纯 Markdown | 无 | 目录就近原则 | ✅ `~/.codex/` |
| **Gemini CLI** | `GEMINI.md`（任意目录层级） | Markdown + `@import` | 无（文件名可配置） | JIT 自动发现 + 拼接 | ✅ `~/.gemini/` |
| **Windsurf** | `.devin/rules/*.md` | Markdown + YAML | `trigger`, `globs` | `always_on` / `model_decision` / `glob` / `manual` | ✅ 全局规则 |
| **Cline** | `.clinerules/*.md` | Markdown + YAML | `paths` | 文件路径匹配 | ✅ `~/Documents/Cline/` |
| **Aider** | `CONVENTIONS.md`（通过配置加载） | 纯 Markdown | 无 | 手动配置 `read:` 列表 | ✅ `.aider.conf.yml` |
| **Amazon Q** | `.amazonq/rules/*.md` | Markdown + YAML | `description`, `alwaysApply`, `globs` | glob 模式 + 始终加载 | ❌ 仅项目级 |
| **Continue.dev** | `.continue/rules/*.md` | Markdown + YAML | `name`, `globs`, `regex`, `description` | glob + **正则匹配文件内容** | ✅ `~/.continue/` |
| **Trae** | `.trae/rules/*.md` | 纯 Markdown | 通过 IDE UI 管理 | 项目全局 | ✅ 用户规则 |

---

## 2. 移植策略：三层适配

```
┌──────────────────────────────────────────────────────┐
│              第 1 层：通用核心（AGENTS.md）            │
│   纯 Markdown，所有工具都能读取                        │
│   放置项目根目录，包含框架核心约束                      │
├──────────────────────────────────────────────────────┤
│            第 2 层：平台适配（各工具原生格式）          │
│   利用各工具的高级特性（glob、触发模式等）              │
│   内容从核心层派生，不重复编写                          │
├──────────────────────────────────────────────────────┤
│          第 3 层：Claude Code 原生（SKILL.md）         │
│   保留完整 SKILL.md 格式，享受渐进式披露               │
│   作为最丰富的格式存在                                 │
└──────────────────────────────────────────────────────┘
```

---

## 3. 第 1 层：通用核心（AGENTS.md）

**适用工具**：OpenAI Codex、Cursor、GitHub Copilot、Gemini CLI、Windsurf、Cline、Aider 等 20+ 工具

`AGENTS.md` 是由 Agentic AI Foundation（Linux Foundation 旗下）维护的跨工具标准，已被 60,000+ 开源项目采用。

### 操作步骤

在项目根目录创建 `AGENTS.md`：

```markdown
# 项目开发约束

## Skill 开发规范

所有 skill/指令文档必须遵循以下约束：

### 语言要求
- 正文内容、注释、说明文字必须使用中文
- 代码本身保持英文，注释使用中文
- 技术术语首次出现时需给出中文解释

### 知识与流程分离
- Skill/指令文档提供**知识**（模式、原则、示例、检查清单）
- Agent 自身提供**流程**（执行步骤、工具调用序列、决策逻辑）
- 禁止在 skill 中内嵌具体的工具调用指令

### 工具中立
- 不指定具体工具名称（不写"使用 Bash 运行"、"使用 Grep 搜索"）
- 描述所需**能力**，让 agent 根据自身工具集选择实现方式
- 对需要写权限的操作，使用条件语句适配只读 agent

### 模型无关
- 禁止使用模型特定提示词（think harder、ultrathink 等）
- 描述所需的分析深度，而非使用特定模型的触发词

### 结构规范
- 每个 skill 须包含：触发条件、核心原则、代码示例、检查清单、常见陷阱
- 代码示例须标注编程语言
- 交叉引用使用显式路径

### 反模式（禁止）
- ❌ 内嵌工具调用流程
- ❌ 指定具体工具名称
- ❌ 使用模型特定提示词
- ❌ 与 agent 内置流程冲突的"强制"清单
- ❌ 超过 5000 token 的单体文档（拆分到子文件）
```

### 子目录覆盖

AGENTS.md 支持目录就近原则——子目录的 AGENTS.md 覆盖父目录：

```
project/
├── AGENTS.md                      # 全局约束（上面的内容）
├── frontend/
│   └── AGENTS.md                  # 前端特定约束（追加/覆盖）
├── backend/
│   └── AGENTS.md                  # 后端特定约束
└── skills/
    └── AGENTS.md                  # Skill 编写特定约束
```

---

## 4. 第 2 层：各平台适配

### 4.1 Cursor（`.cursor/rules/`）

Cursor 使用 `.mdc` 文件（不是 `.md`），支持四种触发模式：

```
.cursor/rules/
├── skill-dev-standards.mdc        # Skill 开发规范（Agent 判断加载）
├── coding-standards.mdc           # 编码规范（始终加载）
├── typescript.mdc                 # TS 规范（glob 自动附加）
└── security.mdc                   # 安全规范（glob 自动附加）
```

**skill-dev-standards.mdc**（Agent Requested 模式——AI 判断何时加载）：

```markdown
---
description: "Skill/指令文档的开发约束框架。当编写、审查或修改 skill 文档时应用此规则。"
alwaysApply: false
---

# Skill 开发约束框架（Cursor 适配版）

## 语言要求
- 所有 skill 正文内容必须使用中文
- 代码注释使用中文
- 技术术语首次出现时给出中文解释

## 核心原则

### 知识与流程分离
Skill 提供**知识**（模式、原则、示例），Agent 提供**流程**（步骤、工具调用）。

### 工具中立
不指定具体工具名称，描述所需能力。

### 模型无关
禁止 think harder、ultrathink 等模型特定提示词。

## 文件结构
每个 skill 须包含：
- 触发条件（何时激活）
- 核心原则（带正反代码示例）
- 检查清单（可验证条目）
- 常见陷阱（症状/原因/修复）

## 禁止项
- ❌ 内嵌工具调用流程
- ❌ 指定具体工具名称
- ❌ 与 agent 内置流程冲突
- ❌ 超过 500 行（Cursor 建议上限）
```

**coding-standards.mdc**（Always 模式——始终加载）：

```markdown
---
alwaysApply: true
---

# 编码规范

- 正文和注释使用中文
- 代码保持英文，注释用中文
- 遵循不可变性原则：创建新对象，不修改现有对象
- 文件不超过 800 行
- 函数不超过 50 行
```

**typescript.mdc**（Auto Attached 模式——匹配文件时自动加载）：

```markdown
---
globs: "**/*.ts,**/*.tsx"
alwaysApply: false
---

# TypeScript 规范

- 使用 2 空格缩进
- 优先使用 `const` 而非 `let`
- 显式声明返回类型
- 接口使用 PascalCase
```

### 4.2 GitHub Copilot

```
.github/
├── copilot-instructions.md                    # 仓库级指令
└── instructions/
    ├── skill-development.instructions.md       # Skill 开发规范
    ├── typescript.instructions.md              # TS 规范
    └── security.instructions.md                # 安全规范
```

**copilot-instructions.md**（仓库级，所有对话都加载）：

```markdown
# 项目约束

- 所有文档和注释使用中文
- 代码保持英文，注释用中文
- 遵循 AGENTS.md 中定义的完整开发规范
```

**skill-development.instructions.md**（路径级，编辑 skill 文件时加载）：

```markdown
---
applyTo: "skills/**,docs/skills/**,.cursor/rules/**"
---

# Skill 开发约束

## 语言
所有 skill 正文内容必须使用中文。代码注释使用中文。

## 结构
每个 skill 须包含：触发条件、核心原则（正反示例）、检查清单、常见陷阱。

## 核心约束
- 知识与流程分离：skill 提供知识，agent 提供流程
- 工具中立：描述能力，不指定工具名
- 模型无关：禁止 think harder、ultrathink 等
- 交叉引用使用显式路径
```

### 4.3 Gemini CLI

Gemini 支持 `@import` 语法，可以将框架拆分为可引用的模块：

```
GEMINI.md                          # 入口文件
docs/
├── skill-framework-core.md        # 核心约束
├── skill-templates.md             # 模板
└── skill-anti-patterns.md         # 反模式
```

**GEMINI.md**：

```markdown
# 项目开发约束

所有 skill/指令文档必须使用中文编写，遵循知识与流程分离原则。

详细约束参见：
@./docs/skill-framework-core.md
@./docs/skill-templates.md
@./docs/skill-anti-patterns.md
```

Gemini 也可以在 `.gemini/settings.json` 中配置文件名兼容：

```json
{
  "context": {
    "fileName": ["GEMINI.md", "AGENTS.md", "CLAUDE.md"]
  }
}
```

### 4.4 Windsurf

```
.devin/rules/
├── skill-development.md           # Skill 开发规范
├── coding-standards.md            # 编码规范
└── typescript.md                  # TS 规范
```

**skill-development.md**（model_decision 模式——AI 判断加载）：

```markdown
---
trigger: model_decision
description: "Skill/指令文档的开发约束框架。当编写、审查或修改 skill 文档时激活。"
---

# Skill 开发约束框架（Windsurf 适配版）

（内容与 Cursor 版本相同，注意 Windsurf 限制 12,000 字符/规则）
```

**coding-standards.md**（always_on 模式）：

```markdown
---
trigger: always_on
---

# 编码规范

- 正文和注释使用中文
- 遵循不可变性原则
- 文件不超过 800 行
```

### 4.5 Cline

```
.clinerules/
├── skill-development.md           # Skill 开发规范
├── coding-standards.md            # 编码规范
└── typescript.md                  # TS 规范（条件激活）
```

**typescript.md**（路径条件激活）：

```markdown
---
paths:
  - "src/**/*.ts"
  - "src/**/*.tsx"
---

# TypeScript 规范

- 使用 2 空格缩进
- 优先使用 const
- 显式声明返回类型
```

### 4.6 Aider

Aider 不支持自动发现，需在配置中显式加载：

**.aider.conf.yml**：

```yaml
read:
  - AGENTS.md
  - docs/skill-framework-core.md
  - docs/skill-templates.md
```

### 4.7 Amazon Q

```
.amazonq/rules/
├── skill-development.md
└── coding-standards.md
```

**skill-development.md**：

```markdown
---
description: "Skill 开发约束框架"
alwaysApply: false
globs: "skills/**,.cursor/rules/**,.github/instructions/**"
---

（约束内容同其他平台）
```

### 4.8 Continue.dev

```
.continue/rules/
├── 01-skill-development.md        # 数字前缀控制加载顺序
├── 02-coding-standards.md
└── 03-typescript.md
```

Continue.dev 支持**正则匹配文件内容**（独有特性）：

```markdown
---
name: TypeScript 规范
globs: ["**/*.ts", "**/*.tsx"]
regex: ["interface\\s+\\w+", "type\\s+\\w+\\s*="]
alwaysApply: false
---

# TypeScript 规范

- 优先使用 interface 而非 type alias
- 使用 strict null checks
```

### 4.9 Trae

```
.trae/rules/
├── skill-development.md
└── coding-standards.md
```

Trae 的元数据通过 IDE UI 管理而非文件 frontmatter，文件本身为纯 Markdown。

---

## 5. 完整项目结构（全平台覆盖）

```
project/
│
├── AGENTS.md                                  # 第 1 层：通用核心（20+ 工具兼容）
├── CLAUDE.md                                  # Claude Code 项目指令（可软链到 AGENTS.md）
├── GEMINI.md                                  # Gemini CLI 入口（@import 引用子文件）
│
├── skills/                                    # 第 3 层：Claude Code 原生 skill
│   └── skill-development-framework/
│       ├── SKILL.md                           # 完整框架（渐进式披露）
│       └── references/
│           ├── templates.md                   # 模板
│           └── porting-guide.md               # 本文件
│
├── .cursor/rules/                             # 第 2 层：Cursor 适配
│   ├── skill-dev-standards.mdc
│   ├── coding-standards.mdc
│   └── typescript.mdc
│
├── .github/                                   # 第 2 层：GitHub Copilot 适配
│   ├── copilot-instructions.md
│   └── instructions/
│       ├── skill-development.instructions.md
│       └── typescript.instructions.md
│
├── .devin/rules/                              # 第 2 层：Windsurf 适配
│   ├── skill-development.md
│   └── coding-standards.md
│
├── .clinerules/                               # 第 2 层：Cline 适配
│   ├── skill-development.md
│   └── coding-standards.md
│
├── .amazonq/rules/                            # 第 2 层：Amazon Q 适配
│   └── skill-development.md
│
├── .continue/rules/                           # 第 2 层：Continue.dev 适配
│   ├── 01-skill-development.md
│   └── 02-coding-standards.md
│
├── .trae/rules/                               # 第 2 层：Trae 适配
│   ├── skill-development.md
│   └── coding-standards.md
│
├── .aider.conf.yml                            # Aider 配置（引用 AGENTS.md）
│
└── docs/                                      # 共享参考文档
    ├── skill-framework-core.md                # 核心约束（被 GEMINI.md @import）
    ├── skill-templates.md                     # 模板
    └── skill-anti-patterns.md                 # 反模式
```

---

## 6. 各平台特性对照与适配要点

### 6.1 触发机制差异

| 机制 | 支持工具 | 适配方式 |
|------|---------|---------|
| **语义匹配**（AI 读 description 判断） | Claude Code、Cursor、Windsurf、Continue.dev | 写好 description，包含明确触发条件 |
| **Glob 模式**（文件路径匹配） | Cursor、Copilot、Windsurf、Cline、Amazon Q、Continue.dev | 设置 `globs: "skills/**"` |
| **目录就近**（子目录覆盖父目录） | AGENTS.md 生态、Gemini CLI | 在 `skills/` 子目录放置专属 AGENTS.md |
| **始终加载** | 所有工具均支持 | 用于编码规范等通用约束 |
| **手动引用** | Cursor（`@rule`）、Aider（`/read`）、Cline | 用于低频使用的参考文档 |

### 6.2 关键限制

| 限制 | 工具 | 影响 | 应对方案 |
|------|------|------|---------|
| **12,000 字符/规则** | Windsurf | 完整框架放不下 | 拆分为多条规则 |
| **~500 行/文件** | Cursor | 同上 | 拆分为多条 `.mdc` |
| **4,000 字符（Code Review）** | GitHub Copilot | Code Review 场景截断 | 核心约束放在前 4000 字符内 |
| **纯 Markdown，无元数据** | AGENTS.md、Aider、Gemini | 无法做触发条件控制 | 核心约束精简到 200 行内 |
| **`.mdc` 扩展名** | Cursor | 普通 `.md` 会被忽略 | 确保使用 `.mdc` |
| **无 glob 支持** | Aider、Trae、AGENTS.md | 无法按文件类型自动加载 | 全量加载或手动引用 |

### 6.3 独有特性利用

| 特性 | 工具 | 利用方式 |
|------|------|---------|
| **`@import` 引用** | Gemini CLI | 将框架拆分为可引用模块 |
| **正则匹配文件内容** | Continue.dev | 根据代码内容自动加载对应规范 |
| **`excludeAgent` 排除** | GitHub Copilot | 排除不需要此规则的 agent |
| **Hook 系统** | Cline | `PreToolUse`/`PostToolUse` 自动验证 |
| **远程规则导入** | Cursor | 从 GitHub 仓库导入团队规则 |

---

## 7. 快速移植检查清单

```
跨平台移植检查清单
==================

准备阶段
[  ] 已提取框架核心约束（精简到 200 行内的纯 Markdown）
[  ] 已准备各类型的模板（reference / component / interactive / workflow）
[  ] 已准备反模式目录

AGENTS.md（通用层）
[  ] 在项目根目录创建 AGENTS.md
[  ] 内容使用中文
[  ] 不依赖任何 frontmatter 或工具特定语法
[  ] 在 Codex CLI 中验证可读取

Cursor 适配
[  ] 创建 .cursor/rules/*.mdc 文件
[  ] 文件扩展名为 .mdc（非 .md）
[  ] 设置了合适的触发模式（Always / Auto Attached / Agent Requested）
[  ] 单文件不超过 500 行

GitHub Copilot 适配
[  ] 创建 .github/copilot-instructions.md
[  ] 路径级规则放在 .github/instructions/*.instructions.md
[  ] applyTo glob 模式正确
[  ] Code Review 场景的核心约束在前 4000 字符内

Gemini CLI 适配
[  ] 创建 GEMINI.md 入口文件
[  ] 使用 @import 引用子文件
[  ] 在 .gemini/settings.json 中配置兼容 AGENTS.md

Windsurf 适配
[  ] 创建 .devin/rules/*.md 文件
[  ] 每条规则不超过 12,000 字符
[  ] 设置了 trigger 字段

Cline 适配
[  ] 创建 .clinerules/*.md 文件
[  ] 条件激活规则设置了 paths 字段

Aider 适配
[  ] 在 .aider.conf.yml 的 read 列表中添加了相关文件

验证
[  ] 每个平台至少用一个测试 skill 验证规则被正确加载
[  ] 确认中文内容在各平台正常显示
[  ] 确认核心约束在各平台一致
```

---

## 8. 维护策略

### 单一信息源（Single Source of Truth）

为避免多平台内容不同步，采用以下维护策略：

```
SKILL.md（Claude Code 原生）
    │
    ├── 导出核心约束 ──→ AGENTS.md（通用层）
    │                         │
    │                         ├── 适配 ──→ .cursor/rules/*.mdc
    │                         ├── 适配 ──→ .github/instructions/*.instructions.md
    │                         ├── 适配 ──→ .devin/rules/*.md
    │                         ├── 适配 ──→ .clinerules/*.md
    │                         └── 引用 ──→ .aider.conf.yml
    │
    └── 导出参考文档 ──→ docs/*.md（被 GEMINI.md @import）
```

**原则**：
- SKILL.md 是权威版本，所有其他文件从它派生
- 更新框架时，先改 SKILL.md，再同步到各平台
- 各平台适配文件只包含该平台需要的子集 + 平台特定语法
- 可以编写脚本自动从 SKILL.md 生成各平台的适配文件
