# Claude Code 记忆系统 — 完整架构深度解析

## 它们构建了什么

Claude Code 拥有一个**多层次、分层级的记忆系统**，可以在会话之间持久化上下文。与典型的 LLM 交互每次从零开始不同，Claude Code 在会话启动时会将项目特定、用户特定和组织特定的记忆加载到系统提示中。该系统有两个基本支柱：**CLAUDE.md 文件**（人工编写的指令）和**自动记忆**（Claude 自己编写的学习笔记）。

截至 2026 年 2 月（v2.1.33+），新增了第三个支柱：**Agent Memory（代理记忆）** — 作用域限定到单个子代理的持久化记忆。

## 为什么采用这种架构

LLM 是无状态的。每次会话从零开始。核心问题：开发者在数百次会话中重复相同的指令（"我们用 pnpm 不是 npm"、"用 vitest 跑测试"、"遵循这个 API 模式"）。记忆系统的存在就是为了消除这种重复。

分层设计解决了一个更难的问题：**不同范围的知识有不同的受众**。组织安全策略适用于所有人。项目约定适用于团队成员。个人偏好只适用于你自己。一个扁平的记忆文件无法捕捉这些区别。

自动记忆的 200 行限制源于上下文窗口的经济学 — 启动时加载的每一个记忆 token 都是实际工作不可用的 token。

## 完整记忆层级

| 记忆类型 | 位置 | 用途 | 受众 | 加载时机 |
|---|---|---|---|---|
| **托管策略** | `C:\Program Files\ClaudeCode\CLAUDE.md`（Win）/ `/etc/claude-code/CLAUDE.md`（Linux）/ `/Library/Application Support/ClaudeCode/CLAUDE.md`（macOS） | 组织级标准 | 所有用户 | 始终 |
| **项目记忆** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队项目指令 | 通过 git 共享 | 始终 |
| **项目规则** | `./.claude/rules/*.md` | 模块化的主题规则 | 通过 git 共享 | 始终（带 `paths` frontmatter 时条件加载） |
| **用户记忆** | `~/.claude/CLAUDE.md` | 跨项目个人偏好 | 仅自己 | 始终 |
| **用户规则** | `~/.claude/rules/*.md` | 个人模块化规则 | 仅自己 | 始终 |
| **本地记忆** | `./CLAUDE.local.md` | 私有项目特定偏好 | 仅自己（gitignored） | 始终 |
| **自动记忆** | `~/.claude/projects/<project>/memory/` | Claude 学到的模式 | 仅自己（按项目） | MEMORY.md 前 200 行 |
| **代理记忆**（v2.1.33+） | 按范围变化（见下文） | 子代理特定知识 | 单个子代理 | 代理 MEMORY.md 前 200 行 |

**优先级顺序**：越具体的优先级越高。项目规则 > 用户规则。本地 > 项目。子目录的 CLAUDE.md 文件在 Claude 读取该目录文件时按需加载。

## 机制 1：CLAUDE.md 文件

### 工作原理

CLAUDE.md 是你编写和维护的 markdown 文件。它成为 Claude 系统提示的一部分 — 在会话启动时完整加载。Claude 将其内容视为指令。

**查找行为**：Claude Code 从当前工作目录开始，沿目录树向上遍历到根目录 `/`（不包括根目录），读取找到的每个 `CLAUDE.md` 和 `CLAUDE.local.md`。在 monorepo 的 `foo/bar/` 中，`foo/CLAUDE.md` 和 `foo/bar/CLAUDE.md` 都会被加载。

**子目录**的 CLAUDE.md 文件启动时**不会**加载。只有当 Claude 读取该子目录中的文件时才会按需加载。

### `/init` 命令

运行 `/init` 会通过分析项目结构、package 文件、现有文档和代码模式来自动生成一个入门 CLAUDE.md。它是起点 — 手动精炼必不可少。

### 导入系统（`@path` 语法）

CLAUDE.md 文件可以导入其他文件：

```markdown
See @README for project overview and @package.json for available npm commands.

# Additional Instructions
- git workflow @docs/git-instructions.md
```

关键规则：
- **相对路径**相对于包含文件解析，不是工作目录
- 支持**绝对路径**和 `@~/...` 主目录路径
- **递归导入**最大深度 **5 跳**
- markdown 代码块/代码段内的导入**不会**被解析（防止与 npm 包名如 `@anthropic-ai/claude-code` 冲突）
- 首次遇到外部导入时显示**审批对话框** — 每个项目一次性决定
- 对于 git worktrees，使用 `@~/.claude/my-project-instructions.md` 让所有 worktrees 共享相同的个人指令

### CLAUDE.local.md

自动添加到 `.gitignore`。用于：
- 个人沙箱 URL
- 偏好的测试数据
- 私有工具快捷方式
- 任何不想提交到 git 的内容

## 机制 2：项目规则（`.claude/rules/`）

### 基本结构

```
.claude/
  rules/
    code-style.md
    testing.md
    security.md
    frontend/
      react.md
      styles.md
    backend/
      api.md
      database.md
```

所有 `.md` 文件通过子目录**递归**发现。它们的加载优先级与 `.claude/CLAUDE.md` 相同。

### 路径特定规则（Frontmatter）

规则可以使用 YAML frontmatter 进行条件范围限定：

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "lib/**/*.ts"
---

# API 开发规则
- 所有端点必须包含输入验证
- 使用标准错误响应格式
```

**没有** `paths` 字段的规则无条件加载。

**支持的 glob 模式：**

| 模式 | 匹配 |
|---|---|
| `**/*.ts` | 任何目录中的所有 TypeScript 文件 |
| `src/**/*` | `src/` 下的所有文件 |
| `*.md` | 仅项目根目录的 Markdown 文件 |
| `src/**/*.{ts,tsx}` | 花括号展开匹配多种扩展名 |
| `{src,lib}/**/*.ts` | 花括号展开匹配多个目录 |

### 符号链接

`.claude/rules/` 支持符号链接，用于跨项目共享规则：

```bash
ln -s ~/shared-claude-rules .claude/rules/shared
ln -s ~/company-standards/security.md .claude/rules/security.md
```

循环符号链接会被检测并优雅处理。

## 机制 3：自动记忆

### 它是什么

自动记忆是一个持久化目录，**Claude 为自己写笔记** — 不是你为 Claude 写的指令。Claude 工作时发现模式并自动保存。

### 记住了什么

- **项目模式**：构建命令、测试约定、代码风格
- **调试洞察**：棘手问题的解决方案、常见错误原因
- **架构笔记**：关键文件、模块关系、重要抽象
- **你的偏好**：沟通风格、工作流习惯、工具选择

### 目录结构

```
~/.claude/projects/<project>/memory/
  MEMORY.md          # 简洁索引（每次会话加载前 200 行）
  debugging.md       # 详细调试模式
  api-conventions.md # API 设计决策
  ...                # Claude 创建的任何主题文件
```

`<project>` 路径来自 **git 仓库根目录** — 同一仓库内的所有子目录共享一个自动记忆目录。Git worktrees 获得单独的目录。在 git 仓库外，使用工作目录。

### 200 行规则

- **MEMORY.md 前 200 行**在会话启动时注入系统提示
- 超过 200 行的内容**永远不会**自动加载
- Claude 被指示保持 `MEMORY.md` 简洁，将详细笔记移入主题文件
- **主题文件**（`debugging.md`、`patterns.md` 等）启动时**不加载** — Claude 需要时使用文件工具按需读取

这创建了一个两级系统：MEMORY.md 是始终可用的索引，主题文件是按需深度存储。

### 控制自动记忆

```bash
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1  # 强制关闭
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=0  # 强制开启（渐进推出期间 opt-in）
```

双重否定逻辑：`DISABLE=0` 意思是"不要禁用" = 强制开启。

### 告诉 Claude 记住什么

你可以明确请求记忆保存：
- "记住我们使用 pnpm 而不是 npm"
- "保存到记忆：API 测试需要本地 Redis 实例"

### `/memory` 命令

打开文件选择器，显示你的自动记忆入口点和 CLAUDE.md 文件。让你在系统编辑器中编辑任何记忆文件。

### 已知问题：双重加载（2026 年 2 月）

GitHub issue [#24044](https://github.com/anthropics/claude-code/issues/24044) 报告 `MEMORY.md` 每次会话被加载**两次** — 一次由 auto-memory 加载器，一次由 claudeMd/project-instructions 加载器。这每次 API 调用浪费约 3KB 的重复 token（20 轮对话约 60K token）。该 issue 已作为重复关闭 — 预计会修复。

## 机制 4：代理记忆（v2.1.33+）

在 Claude Code v2.1.33（2026 年 2 月）中引入，为**子代理**提供自己的持久化记忆。

### 三种范围

| 范围 | 位置 | 版本控制 | 共享 | 用例 |
|---|---|---|---|---|
| **user** | `~/.claude/agent-memory/<name>/` | 否 | 否 | 跨项目知识（默认） |
| **project** | `.claude/agent-memory/<name>/` | 是 | 是 | 团队共享项目模式 |
| **local** | `.claude/agent-memory-local/<name>/` | 否 | 否 | 个人项目笔记 |

### 工作原理

1. **启动注入**：代理的 `MEMORY.md` 前 200 行加载到代理系统提示
2. **自动工具访问**：为记忆管理启用 Read、Write、Edit 工具
3. **主动管理**：代理在执行过程中读取和更新记忆
4. **智能整理**：当 `MEMORY.md` 超过 200 行时，代理将溢出内容组织到主题文件中

### 代理记忆 vs 其他记忆

| 系统 | 写入者 | 读取者 | 持久化 |
|---|---|---|---|
| CLAUDE.md | 人类 | 所有代理 + 主 Claude | Git / 文件系统 |
| 自动记忆 | 主 Claude | 仅主 Claude | 按项目文件系统 |
| `/memory` 命令 | 人类 | 仅主 Claude | 按项目文件系统 |
| 代理记忆 | 单个代理 | 仅该代理 | 按代理文件系统 |

## 机制 5：Skills（渐进披露）

Skills（`.claude/skills/`）通过提供**按需知识加载**来补充记忆系统 — 描述始终在上下文中，但完整内容仅在调用时加载。

### 三级加载

| 级别 | 加载内容 | 时机 | 上下文成本 |
|---|---|---|---|
| **L1：Frontmatter** | Skill 名称 + 描述 | 始终（系统提示） | 最小 |
| **L2：SKILL.md 正文** | 完整指令 | Claude 判断相关时 | 中等 |
| **L3：支持文件** | 模板、示例、脚本 | Claude 导航到它们时 | 按需 |

### Skill Frontmatter

```yaml
---
name: api-conventions
description: API design patterns for this codebase
disable-model-invocation: true  # 仅手动触发
user-invocable: false           # 仅 Claude 触发
allowed-tools: Read, Grep       # 限制工具
context: fork                   # 在子代理中运行
agent: Explore                  # 子代理类型
model: sonnet                   # 模型覆盖
---
```

### Skills 的上下文预算

Skill 描述从动态预算中消耗上下文：**上下文窗口的 2%**（回退值：16,000 字符）。如果 skill 太多，某些会被排除。用 `/context` 检查警告。用 `SLASH_COMMAND_TOOL_CHAR_BUDGET` 环境变量覆盖。

## 上下文窗口与记忆的交互

### Token 预算分解

典型会话分配（通过 `/context` 命令查看）：

| 组件 | Token 数 |
|---|---|
| 系统提示 | ~2.7K |
| 系统工具 | ~16.8K |
| 自定义代理 | ~1.3K |
| **记忆文件（CLAUDE.md + 自动记忆）** | **~7.4K** |
| Skills（仅描述） | ~1.0K |
| 消息（对话） | ~9.6K |
| **自动压缩缓冲区（保留）** | **33.0K** |

压缩前可用总量：200K 上下文窗口中约 167K。

### 自动压缩

当上下文使用接近限制时，Claude Code 自动压缩：

1. **触发阈值**：~83.5% 使用率（缓冲区：~33K token）
2. **过程**：总结对话历史，压缩较旧的消息
3. **损失**：早期会话的细粒度细节丢失
4. **边界追踪**：通过追踪"压缩边界"防止递归压缩失败

### 手动压缩（`/compact`）

- 在任何使用率水平触发压缩（不像自动压缩等到 98%）
- 接受保留指令：`/compact keep the API patterns we established`
- 减少每条消息发送的 token，降低成本

### 上下文控制环境变量

| 变量 | 效果 |
|---|---|
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 设置压缩触发阈值（1-100） |
| `autoCompact: false`（settings.json） | 完全禁用自动压缩 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | 控制响应长度（不是压缩缓冲区） |
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` | 覆盖 skill 描述的上下文预算 |

### 扩展上下文模型

- 默认：200K 上下文窗口
- Sonnet 4.5：500K 上下文窗口
- Opus 4.6：1M 上下文窗口（Beta，需要 API header）
- 使用 `sonnet[1m]` 获得 1M token 窗口，缓冲区按比例增大

## 最佳实践

### CLAUDE.md 组织

**保持在 150-200 行以下。** 研究表明前沿 LLM 可以可靠遵循 150-200 条指令。Claude Code 的系统提示已包含约 50 条指令，留给你的约 150 条。

**结构：WHAT / WHY / HOW**
- **WHAT**：技术栈、代码库地图、关键目录
- **WHY**：项目目的、组件功能
- **HOW**：工作流、命令、测试流程

**不要把 LLM 当 linter 用。** 如果 ESLint/Prettier/TypeScript 能强制执行的，不要放在 CLAUDE.md 里。用一行 `Run pnpm lint:fix && pnpm typecheck after changes.` 替代 200 行风格说教。

**通用适用性。** Claude 会**降低优先级**处理它认为与当前任务无关的 CLAUDE.md 内容。只包含几乎每次会话都适用的指导。

### 渐进披露架构

```
CLAUDE.md                    # 始终加载（~60 行，仅指针）
  @docs/architecture.md      # 引用时导入
.claude/rules/               # 带 paths frontmatter 的条件规则
.claude/skills/              # 按需完整加载
agent_docs/                  # Claude 相关时读取
```

**第一层 — CLAUDE.md（始终加载）：** 项目概述、必要命令、技术栈信息、深层文档指针。目标：60 行以下。

**第二层 — 规则和导入（条件加载）：** 路径范围规则、导入的参考文档。基于文件上下文加载。

**第三层 — Skills 和代理文档（按需）：** 完整工作流、详细参考、脚本。仅当 Claude 判断相关时加载。

Token 影响：
- 系统提示：~3.1K token
- 最小 CLAUDE.md：~500 token
- 结果：**196K+ token** 可用于实际工作（~130 轮对话）
- 臃肿的 CLAUDE.md：在**每次**会话中消耗 token；按需文档仅在需要时消耗

### 自动记忆管理

- 用 `/memory` **定期审查**并修剪过时条目
- **保持 MEMORY.md 在 200 行以下** — 将细节移入主题文件
- 重要事项时**明确告诉 Claude 记住**
- **过时的记忆会降低性能** — 过时的模式导致 Claude 做出错误假设

### 上下文窗口卫生

- 在不相关任务之间积极使用 `/clear`
- 在重大工作前使用 `/compact` 最大化可用上下文
- 为跨越压缩边界的上下文密集会话编写交接文档
- **上下文新鲜度 > 上下文积累** — LLM 输出质量随对话长度下降

## 权衡取舍

| 优势 | 劣势 |
|---|---|
| 跨会话持久化上下文 | 200 行限制迫使激进整理 |
| 分层范围匹配真实组织结构 | 双重加载 bug 浪费 token |
| 自动记忆无需手动努力即可学习 | 自动记忆可能积累过时/错误模式 |
| 渐进披露节省上下文 | Skills 准确率上限 79% vs CLAUDE.md 内联 100% |
| 路径特定规则减少噪音 | Frontmatter glob 模式增加维护复杂度 |
| 导入系统实现模块化 | 最大深度 5 和审批对话框增加摩擦 |

## 替代方案对比

| 系统 | 方法 | 优势 | 劣势 |
|---|---|---|---|
| **Claude Code 记忆** | 文件层级（CLAUDE.md + 自动记忆 + 规则） | 原生集成、零配置、通过 git 团队共享 | 200 行限制、无语义搜索、手动整理 |
| **claude-mem**（插件） | 自动会话捕获 + AI 压缩 + 注入 | 自动捕获所有内容、AI 策展 | 外部依赖、压缩可能丢失细微差别 |
| **Cursor Rules** | `.cursorrules` 文件 + `@docs` 导入 | 类似层级、IDE 集成 | 绑定 Cursor IDE、范围限定灵活性差 |
| **Cline Memory Bank** | 结构化 markdown 记忆库 | 详细架构文档 | 手动维护繁重、无自动学习 |
| **自定义 MCP 记忆** | MCP 服务器 + 向量数据库后端 | 语义搜索、无限存储 | 配置复杂、需要外部基础设施 |
| **Windsurf Rules** | `.windsurfrules` 文件 | 简单的单文件方法 | 无层级、无自动记忆、无条件规则 |

## 最新動態 (2026)

### 自动记忆正式上线 (v2.1.32, 2026年2月5日)

自动记忆从实验性 opt-in 升级为默认启用（v2.1.32）。Claude 现在自动记录和调用记忆，无需设置 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=0` 环境变量。想要禁用的用户仍可设置 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`。这是记忆系统自引入以来最大的变化 — 现在每个 Claude Code 会话默认构建持久化的项目知识。

### 代理记忆 Frontmatter (v2.1.32-2.1.33)

自定义子代理的 `memory` frontmatter 字段在 v2.1.32 中正式化，支持三种范围（`user`、`project`、`local`）。v2.1.33 扩展了 `TeammateIdle` 和 `TaskCompleted` 钩子事件，使多代理团队能协调记忆更新。每个子代理获得自己的持久化 `MEMORY.md`，遵循与主会话相同的 200 行加载规则。结合 `isolation: worktree`（为每个代理提供独立的 git worktree），代理现在可以完全并行运行，拥有独立的记忆和文件系统状态。

### "从此处总结" — 部分压缩 (v2.1.32)

新增了 `/compact` 的替代方案：使用 `Esc + Esc` 或 `/rewind`，用户可以选择消息检查点并选择"从此处总结"，仅压缩该点之后的消息，保持之前的上下文不变。相比 `/compact` 的全部压缩，这提供了更精细的上下文管理控制。

### 内置 Git Worktree 支持 (v2.1.39+)

`--worktree` CLI 标志支持在隔离的 git worktree 中运行 Claude Code。子代理可以在代理定义中声明 `isolation: worktree`。新增 `WorktreeCreate` 和 `WorktreeRemove` 钩子事件用于自定义 VCS 设置/拆卸。这对并行代理工作流至关重要 — 多个 Claude 会话可以编辑同一仓库而不会互相覆盖。已知限制：worktrees 共享本地数据库和 Docker 状态，有状态操作可能产生竞争条件。

### 内存泄漏修复 (v2.1.45-2.1.49)

2026 年 2 月经历了一波关键内存修复，起因是 v2.1.27 发布了一个导致 OOM 崩溃的回归 bug。修复包括：使用后释放 API 流缓冲区和代理上下文（v2.1.47）、任务完成后修剪代理任务消息历史以消除 O(n^2) 消息累积（v2.1.47）、限制 shell 命令输出以防止 RSS 无限增长（v2.1.45）、定期重置 tree-sitter 解析器以停止 WASM 内存增长（v2.1.49）、修复恢复大量子代理使用的会话时内存溢出崩溃（v2.1.49）。这些不是表面修复 — 用户报告超过 20 分钟的会话 RAM 消耗达到 15-20GB。

### 企业控制的新钩子事件

新增两个重要钩子事件：**PermissionRequest**（v2.1.45）允许外部脚本用自定义逻辑自动批准或拒绝工具权限请求；**ConfigChange**（v2.1.49）在会话中配置文件更改时触发，支持企业安全审计和可选阻止未授权设置更改。两个钩子都支持与其余配置系统相同的 user/project/local 范围。

### 从附加目录加载 CLAUDE.md (v2.1.20)

`--add-dir` 标志获得了从附加目录加载 CLAUDE.md 文件的能力（通过 `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`）。`--add-dir` 目录中的 `.claude/skills/` 也自动加载（v2.1.32）。这对 monorepo 和共享配置仓库特别有用，其中记忆和规则位于直接项目根目录之外。

### 压缩缓冲区缩减

自动压缩缓冲区从约 45K token 减少到约 33K token（200K 窗口的 16.5%），为实际工作释放了约 12K 额外 token。压缩现在还能正确处理包含大量 PDF 文档的对话，在发送到压缩 API 之前剥离文档块和图像。此外，子代理调用的 skills 在压缩后不再错误地出现在主会话上下文中。

## 可复用模式

1. **200 行索引模式**：将主记忆文件保持为简洁索引，指向详细主题文件。适用于任何 LLM 工具，不仅限于 Claude Code。

2. **路径范围条件规则**：仅在处理前端文件时加载前端规则。消除噪音并节省上下文 token。

3. **三层渐进披露**：始终加载（微小）-> 条件加载（中等）-> 按需（大型）。适用于任何上下文受限的系统。

4. **明确记忆命令**：不要仅依赖自动记忆。当你发现重要内容时，明确告诉 LLM 保存它。

5. **上下文卫生工作流**：任务间 `/clear`，大工作前 `/compact`，长会话写交接文档。将上下文新鲜度视为一等公民。

6. **不要把 LLM 当 linter**：如果确定性工具能强制执行的，不要浪费上下文 token。在 CLAUDE.md 中放 `run lint` 而不是 200 行风格规则。

7. **符号链接共享规则**：在 `.claude/rules/` 中使用符号链接跨多个项目共享通用规则，无需重复。

8. **代理记忆范围选择**：`user` 范围用于可移植知识，`project` 范围用于团队协作，`local` 范围用于个人注释。

## References

### Official Documentation

- [Manage Claude's memory - Claude Code Docs](https://code.claude.com/docs/en/memory)
- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Using CLAUDE.MD files - Anthropic Blog](https://claude.com/blog/using-claude-md-files)
- [Context windows - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Compaction - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [Memory tool - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Agent Skills - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Skill authoring best practices - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Hooks reference - Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Common workflows - Claude Code Docs](https://code.claude.com/docs/en/common-workflows)

### GitHub Issues and Discussions

- [MEMORY.md loaded twice: auto-memory and claudeMd loaders both inject same file - Issue #24044](https://github.com/anthropics/claude-code/issues/24044)
- [Option to disable auto-memory - Issue #23750](https://github.com/anthropics/claude-code/issues/23750)
- [Slavka Memory Pattern: Unlimited scalable memory with a fixed context window - Issue #24718](https://github.com/anthropics/claude-code/issues/24718)
- [Configurable Context Window Compaction Threshold - Issue #15719](https://github.com/anthropics/claude-code/issues/15719)
- [Compaction fails with 'Conversation too long' at 48% (Opus 4.6) - Issue #23751](https://github.com/anthropics/claude-code/issues/23751)
- [Memory gauge forces chat termination at 0% while 60%+ token budget remains - Issue #10996](https://github.com/anthropics/claude-code/issues/10996)
- [Context compaction fails with 'Conversation too long' when context limit is reached - Issue #26317](https://github.com/anthropics/claude-code/issues/26317)
- [Critical memory regression in 2.1.27 - OOM crash on simple input - Issue #22042](https://github.com/anthropics/claude-code/issues/22042)
- [Subagent processes not terminating on macOS, causing memory leak - Issue #22554](https://github.com/anthropics/claude-code/issues/22554)
- [Memory leak in long-running idle Claude Code sessions - Issue #18859](https://github.com/anthropics/claude-code/issues/18859)
- [Add PreCompact and PostCompact hooks for custom context management - Issue #17237](https://github.com/anthropics/claude-code/issues/17237)

### Community Guides and Blog Posts

- [Writing a good CLAUDE.md - HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [Stop Bloating Your CLAUDE.md: Progressive Disclosure - alexop.dev](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/)
- [How I use Claude Code (+ my best tips) - Builder.io](https://www.builder.io/blog/claude-code)
- [Claude Code Memory System - Developer Toolkit](https://developertoolkit.ai/en/claude-code/advanced-techniques/memory-system/)
- [Claude Code Tips & Tricks: Maximising Memory - Cloud Artisan](https://cloudartisan.com/posts/2025-04-16-claude-code-tips-memory/)
- [Stop Repeating Yourself: Give Claude Code a Memory - ProductTalk](https://www.producttalk.org/give-claude-code-a-memory/)
- [Claude Code's Memory: Working with AI in Large Codebases - Thomas Landgraf](https://thomaslandgraf.substack.com/p/claude-codes-memory-working-with)
- [Claude Code Context Buffer: The 33K-45K Token Problem - ClaudeFast](https://claudefa.st/blog/guide/mechanics/context-buffer-management)
- [Persistent Memory for Claude Code: Setup Guide - Agent Native (Medium)](https://agentnativedev.medium.com/persistent-memory-for-claude-code-never-lose-context-setup-guide-2cb6c7f92c58)
- [Claude Code Best Practices: Memory Management - Cuong Tham (Medium)](https://medium.com/@codecentrevibe/claude-code-best-practices-memory-management-7bc291a87215)
- [Claude Code Compaction - Steve Kinney](https://stevekinney.com/courses/ai-development/claude-code-compaction)
- [Claude Code's Memory Evolution: Auto Memory & PreCompact Hooks - Yuanchang's Blog](https://yuanchang.org/en/posts/claude-code-auto-memory-and-hooks/)
- [Six Things That Changed in Claude Code This Month - Brent W. Peterson (Medium)](https://medium.com/@brentwpeterson/six-things-that-changed-in-claude-code-this-month-8012f49fcb90)
- [Claude Code Context Backups: Beat Auto-Compaction - ClaudeFast](https://claudefa.st/blog/tools/hooks/context-recovery-hook)
- [Claude Code Hooks: Complete Guide with 20+ Examples - aiorg.dev](https://aiorg.dev/blog/claude-code-hooks)
- [Claude Skills and CLAUDE.md: a practical 2026 guide for teams - Gend](https://www.gend.co/blog/claude-skills-claude-md-guide)

### Research and Analysis

- [Claude Code Agent Memory Report - shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice/blob/main/reports/claude-agent-memory.md)
- [Context Window & Compaction - DeepWiki](https://deepwiki.com/anthropics/claude-code/3.3-session-and-conversation-management)
- [Token Budget Management - DeepWiki](https://deepwiki.com/shanraisshan/claude-code-best-practice/4.3-token-budget-management)
- [Claude Code by Anthropic - Release Notes February 2026 - Releasebot](https://releasebot.io/updates/anthropic/claude-code)
- [Claude Agent Skills: A First Principles Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
- [Claude Code Changelog - ClaudeLog](https://claudelog.com/claude-code-changelog/)
- [Claude Code Changelog: Complete Version History - ClaudeFast](https://claudefa.st/blog/guide/changelog)

### Templates and Starter Configs

- [claude-code-best-practice - shanraisshan](https://github.com/shanraisshan/claude-code-best-practice)
- [my-claude-code-setup - centminmod](https://github.com/centminmod/my-claude-code-setup)
- [claude-mem - thedotmack](https://github.com/thedotmack/claude-mem)
- [awesome-claude-skills - travisvn](https://github.com/travisvn/awesome-claude-skills)

### Tools

- [CLAUDE.md for .NET Developers - codewithmukesh](https://codewithmukesh.com/blog/claude-md-mastery-dotnet/)
- [Claude Code Tutorial for Beginners - Complete 2026 Guide - codewithmukesh](https://codewithmukesh.com/blog/claude-code-for-beginners/)
- [Claude Skills and CLAUDE.md: a practical 2026 guide for teams - Gend](https://www.gend.co/blog/claude-skills-claude-md-guide)
