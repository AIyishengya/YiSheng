---
name: skill-development-framework
description: 开发、审查或审计 skill 时使用。定义了 skill 的结构规范、跨 agent 可移植性约束和质量门禁。
type: reference
origin: custom
version: 2.0.0
language: zh-CN
---

# Skill 开发约束框架

> **版本**: 2.0.0
> **更新日期**: 2026-06-11
> **适用对象**: 所有 agent 和人类 skill 开发者
> **目的**: 确保 skill 在所有 agent 类型之间可移植、可组合、可一致使用

---

## 0. 为什么需要这个框架

Skill 是被众多 agent 共同消费的**共享知识层**——从只读审查者（`architect`、`planner`）到全权限构建者（`tdd-guide`、`build-error-resolver`），再到轻量级观察者（`doc-updater` 使用 haiku 模型）。一个假设了特定工具集、内嵌了流程逻辑、或使用了模型特定提示词的 skill，在被不同的 agent 加载时会直接失效。

**核心原则：Skill 提供知识（KNOWLEDGE），Agent 提供流程（PROCESS）。**

当这条边界被违反时，skill 就失去了可移植性。本框架定义了保持 skill 通用可消费的约束条件。

---

## 1. Skill 结构规范

### 1.1 目录布局

```
skills/<skill-name>/
├── SKILL.md              # 必需 — skill 主文件
├── template.md           # 可选 — 输出模板（用于 component 类型）
├── examples/             # 可选 — 示例输出
│   └── sample.md
├── references/           # 可选 — 深度知识文档
│   └── api-spec.md
├── scripts/              # 可选 — 辅助脚本（必须做能力门控）
│   └── helper.sh
├── config.json           # 可选 — 可调参数
└── agents/               # 可选 — 内嵌子 agent（仅限紧耦合场景）
    └── observer.md
```

**规则：**
- 目录名：小写、短横线分隔、≤64 字符
- 主文件：始终为 `SKILL.md`（大小写精确匹配）
- `scripts/` 之外不得包含可执行逻辑——skill 是 Markdown 知识文档
- `agents/` 中的内嵌 agent 必须与 skill 的数据格式紧耦合

### 1.2 Frontmatter 模式

```yaml
---
name: <skill-name>                       # 必需，≤64 字符，小写短横线
description: <激活条件描述>               # 必需，≤200 字符
type: component | interactive | workflow  # 可选但推荐
language: zh-CN                          # 必需，固定为 zh-CN
origin: ECC | community | custom         # 可选，来源追踪
version: 1.0.0                           # 可选，语义化版本
---
```

#### 字段约束

| 字段 | 必需 | 最大长度 | 格式 | 用途 |
|------|------|----------|------|------|
| `name` | ✅ | 64 字符 | `小写-短横线` | 唯一标识符，用于交叉引用 |
| `description` | ✅ | 200 字符 | 中文自然语言，以"当……时使用此 skill"开头 | 路由契约——告诉编排器何时加载 |
| `type` | ⚠️ 推荐 | 枚举 | `component`、`interactive`、`workflow` | 决定 skill 原型和节结构 |
| `language` | ✅ | — | 固定 `zh-CN` | 语言标识，所有 skill 必须为中文 |
| `origin` | ❌ | — | 字符串 | 来源追踪 |
| `version` | ❌ | semver | `x.y.z` | 用于持续演进的 skill |

#### description 编写规则

`description` 是 skill 与编排器之间的**API 契约**。它决定了 skill 何时被加载、被哪个 agent 加载。

```yaml
# ✅ 正确——触发条件清晰、工具中立
description: 当编写新功能、修复缺陷或重构代码时使用此 skill。强制测试驱动开发，确保 80%+ 覆盖率。

# ✅ 正确——领域明确、可组合
description: 当添加身份认证、处理用户输入、管理密钥或创建 API 端点时使用此 skill。

# ❌ 错误——模糊、无触发条件
description: 一组最佳实践。

# ❌ 错误——工具特定，只读 agent 无法执行
description: 运行 npm audit 并修复所有漏洞。

# ❌ 错误——超长，超过 200 字符
description: 当需要实现包括 OAuth2、JWT、会话管理、API 密钥管理、RBAC、ABAC 以及各种其他授权模式在内的身份认证系统时使用此 skill，并为每个框架和语言提供详细的代码示例。
```

### 1.3 语言约束（强制）

**所有 skill 的正文内容必须使用中文编写。**

具体要求：
- **正文叙述、标题、注释、清单、说明文字**：必须使用中文
- **代码示例**：编程语言本身保持英文（这是代码的自然语言），但代码中的注释应使用中文
- **技术术语**：可使用英文原词（如 API、token、agent），但需在首次出现时给出中文解释
- **frontmatter 字段名**：保持英文（`name`、`description`、`type` 等是机器字段名）
- **`name` 字段值**：保持小写英文短横线格式（这是机器标识符）
- **`description` 字段值**：必须使用中文

```yaml
# ✅ 正确
name: tdd-workflow
description: 当编写新功能、修复缺陷或重构代码时使用此 skill。强制测试驱动开发。

# ❌ 错误——description 使用了英文
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring code.
```

### 1.4 Skill 类型分类

每个 skill 属于以下四种原型之一：

| 类型 | 用途 | 节模式 | 示例 |
|------|------|--------|------|
| **reference**（无 `type` 字段） | 领域知识、模式、代码示例 | 何时激活 → 核心原则 → 代码示例 → 检查清单 → 常见陷阱 | `tdd-workflow`、`python-patterns` |
| **component** | 产出单一制品/模板 | 目的 → 关键概念 → 应用步骤 → 示例 → 陷阱 → 参考 | `problem-statement`、`user-story` |
| **interactive** | 提出 3-8 个自适应问题 | 目的 → 关键概念 → 应用步骤（问答流） → 示例 → 陷阱 | `epic-breakdown-advisor` |
| **workflow** | 多阶段编排 | 目的 → 关键概念 → 应用步骤（阶段） → 示例 → 陷阱 → 参考 | `prd-development`、`discovery-process` |

**规则**：`type` 决定了使用哪种节结构。混合类型会造成消费 agent 的解析混乱。

---

## 2. 各类型的节结构

### 2.1 参考型 Skill（type: 省略或未设置）

最常见的类型——被 agent 加载以获取领域专业知识的纯知识文档。

```markdown
---
name: <名称>
description: <激活条件>
language: zh-CN
origin: ECC
---

# <标题>

一句话概述。

## 何时激活

- <具体场景 1>
- <具体场景 2>
- <具体场景 3>

## 核心原则

### 1. <原则名称>

<理由和解释。>

```<语言>
// ✅ 正确：<为什么好>
<代码示例>
```

```<语言>
// ❌ 错误：<为什么不好>
<代码示例>
```

### 2. <原则名称>
……

## 代码示例

按用例组织的实用、经过测试的示例。

## 检查清单

- [ ] 可验证的条目
- [ ] 条目须工具中立（参见 §3.2）

## 常见陷阱

### 陷阱 1：<名称>
**症状：** 出了什么问题
**原因：** 为什么会发生
**修复：** 如何预防/解决

## 消费方 Agent

以下 agent 引用了此 skill：
- `<agent-name-1>` —— <用途>

## 参考资料

- 相关 skill：`skill-name-1`、`skill-name-2`
- 外部资源
```

### 2.2 组件型 Skill（type: component）

```markdown
---
name: <名称>
description: <描述>
type: component
language: zh-CN
---

## 目的
此 skill 产出什么制品以及为什么重要。

## 关键概念
### 框架
### 为什么有效
### 反模式（这不是什么）
### 何时使用 / 何时不使用

## 应用步骤
### 步骤 1：收集上下文
### 步骤 2-N：填充各节
（如适用，引用 template.md）

## 示例
具体完成的示例或指向 examples/ 目录的链接。

## 常见陷阱
### 陷阱：<名称>
**症状：** / **后果：** / **修复：**

## 参考资料
### 相关 Skill
### 外部框架
```

### 2.3 交互型 Skill（type: interactive）

```markdown
---
name: <名称>
description: <描述>
type: interactive
language: zh-CN
---

## 目的
此 skill 辅助什么决策或分析。

## 关键概念
### 框架 / 决策模型
### 评分标准（如适用）

## 应用步骤
### 步骤 1：提出自适应问题
（编号选项，支持多选）
### 步骤 2：评分 / 评估
### 步骤 3：交付建议

## 示例
展示预期交互的示例问答流。

## 常见陷阱
```

### 2.4 工作流型 Skill（type: workflow）

```markdown
---
name: <名称>
description: <描述>
type: workflow
language: zh-CN
---

## 目的
此 skill 编排什么端到端流程。

## 关键概念
### 前置条件
### 完成定义

## 应用步骤
### 阶段 1：<名称>
### 阶段 2：<名称>
### 阶段 N：<名称>

## 示例
端到端演练。

## 常见陷阱
## 参考资料
```

---

## 3. 跨 Agent 可移植性约束

以下是确保 skill 在所有 agent 类型之间正常工作的**硬性规则**。

### 3.1 知识与流程分离（关键）

**Skill 提供知识。Agent 提供流程。**

| 属于 Skill | 属于 Agent |
|-------------|-----------|
| 领域模式和原则 | 逐步执行流程 |
| 代码示例（错误/正确） | 工具调用序列 |
| 验证检查清单 | 下一步决策逻辑 |
| 反模式和陷阱 | 错误恢复程序 |
| 参考数据和模板 | 子任务编排 |

```markdown
# ✅ 正确——Skill 提供知识
## 安全检查清单
- [ ] 源代码中无硬编码的 API 密钥、令牌或密码
- [ ] 所有密钥存储在环境变量中
- [ ] `.env.local` 已加入 .gitignore

# ❌ 错误——Skill 内嵌流程
## 安全处理流程
1. 首先，使用 Bash 运行 `npm audit`
2. 解析 JSON 输出
3. 对每个 HIGH 级漏洞创建修复 PR
4. 通过 Slack webhook 通知团队
```

**为什么重要**：只读的 `architect` agent（工具：`Read, Grep, Glob`）无法执行 `npm audit`。如果 skill 内嵌了这个流程，architect 只能报告——产生死指令。`security-reviewer` agent 可能有自己的审计流程与之冲突。

### 3.2 工具中立指令

Skill **禁止**指定具体的工具调用。应描述所需的**能力**。

```markdown
# ✅ 正确——基于能力（工具中立）
### 验证
- 确认源代码文件中不存在硬编码的密钥
- 检查环境变量验证是否在启动时执行
- 确保 `.env.local` 在 `.gitignore` 中

# ❌ 错误——工具特定
### 验证
- 使用 `Grep` 搜索 API 密钥：`grep -rn "sk-" src/`
- 使用 `Read` 检查 `.env.local` 是否在 `.gitignore` 中
- 使用 `Bash` 运行：`npx dotenv-linter`
```

**为什么重要**：工具名称在不同 agent 生态系统中不一致。ECC 使用 `"Bash"` 而某些插件使用 `"BashOutput"`。写了"使用 Grep"的 skill 可能无法映射到使用不同工具标识符的 agent。

### 3.3 条件能力模式

当 skill 确实需要引用可执行动作时，使用能适应 agent 工具集的条件语言：

```markdown
# ✅ 正确——条件能力
### 应用修复
如果你具有代码库的写入权限：
- 将配置更新为使用环境变量
- 为必需的密钥添加启动验证

如果你仅进行审查（只读）：
- 报告硬编码密钥的文件路径和行号
- 推荐环境变量模式作为修复方案

# ❌ 错误——假设具有写入权限
### 应用修复
- 编辑 `config.ts` 替换硬编码的密钥
- 创建 `.env.local` 文件存放密钥
```

### 3.4 模型无关语言

Skill **禁止**使用模型特定的提示词模式。

```markdown
# ✅ 正确——模型无关
## 分析深度
将每个安全问题与 OWASP Top 10 进行逐一对照评估。

# ❌ 错误——模型特定提示词
## 分析深度
更深入地思考每个安全问题。使用 ultrathink 模式。
使用链式思维推理逐步分解此问题。
```

**禁止在 skill 中使用的短语**：`think harder`、`ultrathink`、`think step by step`、`use extended thinking`、`chain-of-thought`、`reason through this`、`深度思考`、`一步步推理`。这些是模型特定的，无法在 haiku/sonnet/opus 之间或非 Claude 模型之间迁移。

### 3.5 禁止重复权威

当 skill 和 agent 都为同一领域提供检查清单或规则时，skill 必须将其内容定位为**补充参考**，而非**竞争权威**。

```markdown
# ✅ 正确——定位为补充
## 详细漏洞模式
> 本节为 agent 的安全检查清单补充详细的代码示例和边界情况。

# ❌ 错误——与 agent 竞争
## 安全审查流程（强制）
你必须按以下顺序严格执行此检查清单：
1. ……
2. ……
```

### 3.6 代码示例的语言标签

所有代码示例**必须**标注编程语言。当 skill 涵盖语言无关的原则时，使用最相关的语言提供示例但需明确标注。

```markdown
# ✅ 正确
### 仓储模式（TypeScript 示例）
```typescript
interface UserRepository {
  findById(id: string): Promise<User | null>;
}
```

> Python 中的相同模式：
```python
class UserRepository(Protocol):
    def find_by_id(self, id: str) -> User | None: ...
```

# ❌ 错误——未标注，假设为 TypeScript
### 仓储模式
```
interface UserRepository {
  findById(id: string): Promise<User | null>;
}
```
```

---

## 4. 可组合性约束

Skill 之间形成依赖图。以下规则确保干净的组合。

### 4.1 交叉引用格式

使用显式、一致的引用语法：

```markdown
# ✅ 正确——显式交叉引用
## 相关 Skill
- [`problem-statement`](../problem-statement/SKILL.md) —— 定义此方案要解决的问题
- [`epic-hypothesis`](../epic-hypothesis/SKILL.md) —— 提供假设框架

# ❌ 错误——模糊引用
## 参见
- problem statement
- epic hypothesis skill
```

### 4.2 禁止循环依赖

Skill 可以引用其他 skill，但**禁止**创建循环依赖链：

```
# ✅ 正确——DAG（有向无环图）
prd-development → problem-statement → jobs-to-be-done
                                   → proto-persona

# ❌ 错误——循环依赖
skill-a → skill-b → skill-c → skill-a
```

### 4.3 依赖声明

当 skill 依赖其他 skill 才能正常运作时，必须显式声明：

```markdown
## 依赖项
此 skill 基于以下概念：
- [`tdd-workflow`](../tdd-workflow/SKILL.md) —— 假定已了解红-绿-重构循环
- [`python-patterns`](../python-patterns/SKILL.md) —— Python 特定的测试惯用法

如果你不熟悉这些基础概念，请先加载这些 skill。
```

### 4.4 互补深度模式

Agent-Skill 组合的经过验证的模式：

| Agent 的角色 | Skill 的角色 |
|-------------|-------------|
| 知道**如何**审查（诊断命令、审批标准、严重级别） | 知道**什么**是好代码（模式、反模式、代码示例） |
| 提供**流程**（步骤序列、决策逻辑） | 提供**参考**（详细示例、边界情况、检查清单） |
| 做出**决策**（通过/拒绝、修复/跳过） | 提供**标准**（什么构成违规） |

---

## 5. 渐进式披露

Skill 使用三层加载模型来高效管理上下文窗口：

### 第 1 层：元数据（约 100 token）
始终在启动时加载。仅包含 frontmatter 中的 `name` 和 `description`。编排器用它来决定是否加载完整 skill。

### 第 2 层：指令（约 2-5K token）
仅在编排器判定 skill 相关时加载。包含完整的 SKILL.md 正文。

### 第 3 层：资源（可变）
在执行过程中按需加载。包含 `examples/`、`references/`、`template.md` 中的文件。

**约束**：Skill 在第 2 层必须是自包含的。SKILL.md 正文必须提供足够的上下文来理解领域，无需第 3 层资源。第 3 层增加深度，而非理解。

```markdown
# ✅ 正确——第 2 层自包含
## 核心原则
### 1. 不可变性
始终创建新对象，永远不修改现有对象。
[完整的解释 + 内联代码示例]

## 深入探讨
更多示例和边界情况，参见 `references/immutability-patterns.md`

# ❌ 错误——需要第 3 层才能理解
## 核心原则
参见 `references/principles.md` 获取所有原则。
```

---

## 6. 质量门禁

每个 skill 在被视为完成之前**必须**通过这些检查。

### 6.1 结构验证

- [ ] Frontmatter 包含有效的 `name`（≤64 字符，小写短横线）和 `description`（≤200 字符）
- [ ] `language` 字段设置为 `zh-CN`
- [ ] `type` 字段已设置且与节结构匹配（如适用）
- [ ] 节顺序遵循类型特定模板（§2）
- [ ] SKILL.md 存在且是主文件
- [ ] 目录名与 `name` 字段匹配

### 6.2 可移植性验证

- [ ] 无工具特定指令（无"使用 `Bash`"、"运行 `Grep`"等）
- [ ] 无模型特定提示词（`think harder`、`ultrathink` 等）
- [ ] 代码示例已标注编程语言
- [ ] 可执行动作使用了条件能力模式（§3.3）
- [ ] 无属于 agent 的流程逻辑（§3.1）
- [ ] 检查清单是工具中立的，任何 agent 类型均可执行

### 6.3 可组合性验证

- [ ] 交叉引用使用显式路径格式（§4.1）
- [ ] 无循环依赖（§4.2）
- [ ] 依赖已声明（§4.3）
- [ ] 与消费方 agent 无重复权威（§3.5）
- [ ] Skill 补充而非替代 agent 流程

### 6.4 内容验证

- [ ] "何时激活"节包含具体触发场景（非抽象描述）
- [ ] 至少存在一个具体示例
- [ ] 至少记录了一个显式反模式
- [ ] 无模糊或套话语言
- [ ] 检查清单条目可验证（agent 能逐项检查）

### 6.5 语言验证

- [ ] 正文内容全部使用中文
- [ ] 代码注释使用中文
- [ ] `description` 字段使用中文
- [ ] 技术术语首次出现时有中文解释

### 6.6 Token 预算验证

- [ ] SKILL.md 正文 ≤5,000 token（第 2 层预算）
- [ ] 深度内容已移至 `references/`（第 3 层）
- [ ] 代码示例简洁——展示模式而非整个文件
- [ ] 无与其他 skill 重复的内容（使用引用代替）

---

## 7. 反模式目录

以下是破坏 skill 可移植性的最常见错误：

### 7.1 内嵌流程反模式

**症状**：Skill 包含带工具调用的逐步执行指令。
**影响**：只读 agent 无法执行；与 agent 自身流程冲突。
**修复**：将流程移至 agent。Skill 仅提供知识。

### 7.2 工具处方反模式

**症状**：Skill 写了"使用 Bash 运行 X"或"使用 Grep 查找 Y"。
**影响**：工具名称在不同生态系统中不一致；只读 agent 无法遵从。
**修复**：描述所需能力，而非具体工具调用。

### 7.3 重复权威反模式

**症状**：Skill 提供了与 agent 检查清单竞争的"强制"检查清单。
**影响**：Agent 不知道该遵循哪个清单。
**修复**：将 skill 检查清单定位为补充参考。

### 7.4 上下文溢出反模式

**症状**：SKILL.md 超过 10,000 token，包含大量内联文档。
**影响**：消耗过多上下文窗口；其他 skill 和代码无法放入。
**修复**：将深度内容移至 `references/`。保持 SKILL.md ≤5,000 token。

### 7.5 孤儿 Skill 反模式

**症状**：Skill 无交叉引用、无依赖声明、无"相关 Skill"节。
**影响**：Agent 不知道何时将它与其他 skill 一起加载。
**修复**：添加显式交叉引用和依赖声明。

### 7.6 单语言反模式

**症状**：语言无关的 skill 仅使用 TypeScript 示例且未标注。
**影响**：Python/Go/Swift agent 认为该 skill 仅适用于 TypeScript。
**修复**：为所有代码示例标注语言。为通用模式提供多语言示例。

### 7.7 模型耳语者反模式

**症状**：Skill 使用 `think harder`、`ultrathink` 或其他模型特定提示词。
**影响**：无法在 haiku/sonnet/opus 之间或非 Claude 模型之间迁移。
**修复**：移除模型特定提示词。改为描述所需的分析深度。

### 7.8 错误类型反模式

**症状**：组件任务被建模为 `workflow`，或参考内容被建模为 `interactive`。
**影响**：节结构与内容类型不匹配；agent 解析错误。
**修复**：根据 skill 的主要用途选择类型：
- 产出单一制品？ → `component`
- 通过提问引导决策？ → `interactive`
- 编排多个阶段？ → `workflow`
- 提供领域知识？ → reference（无 type 字段）

---

## 8. Agent 兼容性矩阵

开发 skill 时使用此矩阵确保它适用于所有 agent 类别：

| Agent 类别 | 可用工具 | 模型 | Skill 必须…… |
|-----------|---------|------|-------------|
| **只读分析师**（`architect`、`planner`、`*-reviewer`） | Read, Grep, Glob | opus/sonnet | 不要求写入/编辑/bash；提供可评估的检查清单 |
| **修复者/构建者**（`tdd-guide`、`build-error-resolver`、`e2e-runner`） | Read, Write, Edit, Bash, Grep, Glob | sonnet | 不与 agent 自身流程冲突；用示例补充 |
| **轻量级操作者**（`doc-updater`、`observer`） | 可变 | haiku | 无需深度推理即可理解；使用清晰结构 |
| **元 Agent**（`skill-reviewer`、`plugin-validator`） | Read, Grep, Glob | inherit | 具有有效 frontmatter 和结构；通过自动化验证 |
| **内嵌 Agent**（在 skill 内部发布） | 继承 | 可变 | 仅与父 skill 的数据格式紧耦合 |

### 兼容性检查清单

发布 skill 前，验证它适用于**受限程度最高**的目标 agent：

1. **只读 agent 能使用此 skill 吗？**（不需要写入/编辑/bash）
2. **haiku 模型 agent 能理解此 skill 吗？**（结构清晰，解析不需要复杂推理）
3. **使用不同工具的 agent 能使用此 skill 吗？**（无工具特定指令）
4. **skill 是否补充（而非竞争）agent 的内置流程？**

---

## 9. Agent-Skill 绑定约定

Agent 使用以下标准格式引用 skill：

```markdown
关于详细的 <领域> 模式，参见 skill: `<skill-name>`。
```

为确保你的 skill 被正确的 agent 加载：

1. **编写清晰的 `description`**，匹配 agent 会搜索的内容
2. **包含"消费方 Agent"节**（可选但推荐）：

```markdown
## 消费方 Agent
以下 agent 引用了此 skill：
- `security-reviewer` —— 用于漏洞模式和代码示例
- `code-reviewer` —— 用于安全相关的代码审查标准
- `build-error-resolver` —— 用于安全相关的构建失败
```

---

## 10. 版本管理与演进

### 10.1 Skill 语义化版本

- **MAJOR**（x.0.0）：frontmatter 模式、节结构或交叉引用格式的破坏性变更
- **MINOR**（0.x.0）：新增节、内容扩展、新示例
- **PATCH**（0.0.x）：错别字修复、措辞澄清、代码示例更正

### 10.2 向后兼容

演进 skill 时：
- 永远不删除 agent 引用的节
- 永远不更改 `name` 字段（会破坏交叉引用）
- 先弃用再删除——删除内容前添加说明
- 重构时递增 MAJOR 版本

---

## 11. 快速参考卡

```
┌──────────────────────────────────────────────────────────────┐
│                    SKILL 开发检查清单                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  结构                                                        │
│  □ 目录：skills/<name>/SKILL.md                             │
│  □ Frontmatter：name (≤64) + description (≤200 中文)        │
│  □ language: zh-CN                                          │
│  □ Type：reference | component | interactive | workflow      │
│  □ 各节遵循类型特定模板                                      │
│  □ SKILL.md ≤ 5,000 token                                   │
│                                                              │
│  可移植性                                                    │
│  □ 无工具特定指令                                            │
│  □ 无模型特定提示词                                          │
│  □ 代码示例已标注语言                                        │
│  □ 可执行动作使用条件能力模式                                │
│  □ 仅知识——无流程逻辑                                       │
│  □ 检查清单工具中立                                          │
│                                                              │
│  可组合性                                                    │
│  □ 交叉引用使用显式路径                                      │
│  □ 无循环依赖                                                │
│  □ 依赖已声明                                                │
│  □ 与 agent 无重复权威                                       │
│  □ 已添加"消费方 Agent"节                                   │
│                                                              │
│  内容                                                        │
│  □ "何时激活"有具体触发场景                                  │
│  □ 至少一个具体示例                                          │
│  □ 至少一个显式反模式                                        │
│  □ 无模糊语言                                                │
│  □ 检查清单条目可验证                                        │
│                                                              │
│  语言                                                        │
│  □ 正文全部使用中文                                          │
│  □ 代码注释使用中文                                          │
│  □ description 使用中文                                      │
│  □ 技术术语首次出现有中文解释                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 附录 A：按类型的真实示例

### 参考型 Skill 示例：`security-review`
- **name**：`security-review`
- **消费方**：`security-reviewer`、`code-reviewer`、`build-error-resolver`
- **为什么有效**：提供错误/正确的代码模式和检查清单——无工具调用、无流程逻辑

### 组件型 Skill 示例：`problem-statement`
- **name**：`problem-statement`
- **消费方**：任何生成问题定义的 agent 或 command
- **为什么有效**：提供模板和标准——agent 填充内容

### 交互型 Skill 示例：`epic-breakdown-advisor`
- **name**：`epic-breakdown-advisor`
- **消费方**：PM 工作流 command
- **为什么有效**：定义问题流和评分——agent 执行对话

### 工作流型 Skill 示例：`skill-authoring-workflow`
- **name**：`skill-authoring-workflow`
- **消费方**：skill 创建 command 和元 agent
- **为什么有效**：定义阶段和验证标准——agent 执行每个阶段

## 附录 B：Frontmatter 对比——Skill vs Agent vs Command

| 字段 | Skill | Agent | Command |
|------|-------|-------|---------|
| `name` | ✅ 必需 | ✅ 必需 | ❌（使用文件名） |
| `description` | ✅ 必需（≤200，中文） | ✅ 必需（触发逻辑） | ✅ 必需 |
| `type` | ⚠️ 推荐 | ❌ | ❌ |
| `language` | ✅ 必需（zh-CN） | ❌ | ❌ |
| `origin` | ❌ 可选 | ❌ | ❌ |
| `version` | ❌ 可选 | ❌ | ❌ |
| `tools` | ❌ | ✅ 必需 | ❌ 可选（`allowed_tools`） |
| `model` | ❌ | ✅ 必需 | ❌ 可选 |
| `color` | ❌ | ❌ 可选 | ❌ |
| `argument-hint` | ❌ | ❌ | ❌ 可选 |

## 附录 C：标准化演进时间线

| 时期        | 范式        | 焦点              | Skill 格式              |
| --------- | --------- | --------------- | --------------------- |
| 2022–2024 | 提示工程      | 优化单次输入输出        | 临时提示文本文件              |
| 2024–2025 | 上下文工程     | 设计信息环境          | 结构化 CLAUDE.md + rules |
| 2025–2026 | Skill 标准化 | 可移植的知识包         | SKILL.md + 渐进式披露      |
| 2026+     | 运行时工程     | 完整的 agent 运行时设计 | Agent Skills 开放标准     |

本框架面向 **2025–2026 Skill 标准化**范式，并向前兼容新兴的 Agent Skills 开放标准。

## 附录 D：跨平台移植

本框架可移植到 Cursor、GitHub Copilot、OpenAI Codex、Gemini CLI、Windsurf、Cline、Aider 等 20+ 工具。

移植采用三层策略：
1. **通用核心**（AGENTS.md）——纯 Markdown，所有工具兼容
2. **平台适配**（各工具原生格式）——利用 glob、触发模式等高级特性
3. **Claude Code 原生**（SKILL.md）——完整格式，渐进式披露

详见 `references/porting-guide.md` 获取各工具的完整适配方案和代码示例。
