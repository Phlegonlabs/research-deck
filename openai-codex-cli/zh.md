# OpenAI Codex CLI — 完整使用指南与社区实践

## 是什么

Codex CLI 是 OpenAI 的开源（Rust 构建）终端编程代理。它能读取你的代码库、建议/实现变更、执行命令，并通过操作系统级别的沙箱维护安全性。支持 macOS、Linux，Windows 为实验性支持（推荐使用 WSL）。包含在 ChatGPT Plus/Pro/Business/Edu/Enterprise 订阅中。

可以理解为 OpenAI 对标 Claude Code 的产品 —— 一个本地优先的 AI 编程代理，住在你的终端里。

## 为什么选择这种方案

OpenAI 选择终端优先的原因和 Anthropic 做 Claude Code 一样：**IDE 是瓶颈**。终端代理可以：

- 不受扩展限制地读取整个仓库
- 执行任意 shell 命令（构建、测试、部署）
- 跨任何语言/框架工作，无需 IDE 插件
- 集成到 CI/CD 和脚本管线中
- 无头运行实现自动化（`codex exec`）

与 Claude Code 的关键差异：**Codex Cloud** —— 能将任务卸载到沙箱化的云环境并在本地应用差异。Claude Code 是纯本地的。

## 安装与设置

### 安装

```bash
npm i -g @openai/codex
```

需要 Node.js >= 22。

### 认证

```bash
codex          # 首次运行触发 OAuth 登录
codex login    # 显式登录（ChatGPT 账号或 API key）
```

Token 保存在 `~/.codex/`，无需手动复制密钥。

### 验证

```bash
codex --version
```

## 核心概念

### 审批模式（权限等级）

| 模式 | 功能 | 标志 |
|------|------|------|
| **只读** | 仅浏览文件，修改前询问 | 默认 |
| **自动** | 读取、编辑、在工作目录内执行命令 | `--full-auto` |
| **完全访问** | 不受限的机器访问，包括网络 | `--sandbox danger-full-access` |
| **YOLO** | 跳过所有安全检查（仅限隔离 VM） | `--dangerously-bypass-approvals-and-sandbox` |

实用工作流：先用只读模式探索，然后 `--full-auto` 进行日常开发。

### 沙箱模式

| 沙箱 | 范围 | 网络 |
|------|------|------|
| `read-only` | 仅读取文件 | 阻断 |
| `workspace-write` | 在项目目录内编辑 | 阻断（可配置） |
| `danger-full-access` | 完全系统访问 | 允许 |

在 workspace-write 中启用网络：
```bash
codex -c 'sandbox_workspace_write.network_access=true'
```

### 权限模式对比：Codex vs Claude Code 的 "Dangerous Mode"

Claude Code 的 `--dangerously-skip-permissions` 跳过所有确认弹窗，但底层**没有真正的沙箱** —— 本质是"信任制"。Codex 采用 **OS 级沙箱**（macOS 用 seatbelt，Linux 用 landlock），即使 `--full-auto` 也物理隔离，是"隔离制"。

| Codex CLI | 命令 | 等同 Claude Code |
|-----------|------|-----------------|
| 只读（默认） | `codex` | 默认模式（每步要确认） |
| Auto | `codex --full-auto` | 温和版 skip-permissions — 自动批准编辑和命令，限制在工作目录内，**无网络** |
| Full Access | `codex --full-auto --sandbox danger-full-access` | 最接近 `--dangerously-skip-permissions` — 完整系统访问 + 网络 |
| YOLO | `codex --dangerously-bypass-approvals-and-sandbox` | 比 Claude Code 更极端 — 跳过所有安全检查，**仅限隔离 VM** |

**日常开发实际用法：**

```bash
# 大多数人日常够用 — 自动读写执行，网络阻断是安全设计
codex --full-auto

# 需要网络（npm install、API 调用等）时升级
codex --full-auto --sandbox danger-full-access

# 只开网络不开完全访问的折中方案
codex -c 'sandbox_workspace_write.network_access=true'
```

**架构差异的本质**：Claude Code 的权限是软件层面的确认弹窗，跳过后 agent 拥有和你一样的系统权限。Codex 的沙箱是操作系统内核级的隔离，`--full-auto` 下 agent 物理上无法写工作目录外的文件或访问网络 —— 要突破这个边界必须显式用 `danger-full-access`。这意味着即使 agent 被恶意提示注入，Codex 的沙箱仍能限制损害范围。

### 模型选择

| 模型 | 用途 | 备注 |
|------|------|------|
| `gpt-5.3-codex` | 默认，日常编码 | 速度/质量最佳平衡 |
| `gpt-5.3-codex-spark` | 仅 Pro 用户 | 研究预览，更快 |
| `gpt-5.2-codex` medium/high | 常规任务 | 更便宜 |
| `gpt-5.2-codex` xhigh | 复杂推理 | 贵且慢 —— 留给困难问题 |

会话中切换：`/model` 斜杠命令或启动时 `--model` 标志。

## 完整命令参考

### 核心命令

```bash
codex                          # 交互式 TUI
codex "explain this project"   # 带提示启动
codex exec "fix the tests"     # 非交互式（别名：codex e）
codex resume                   # 继续之前的会话
codex fork                     # 克隆会话到新线程
codex cloud                    # 浏览/启动云任务
codex app                      # 启动 macOS 桌面应用
codex apply                    # 本地应用云任务差异
```

### 关键标志

```bash
codex --full-auto              # 自动审批编辑 + 命令
codex --model gpt-5.3-codex   # 指定模型
codex -i screenshot.png        # 附加图片
codex --search                 # 启用实时网络搜索
codex --profile fast           # 加载命名配置档案
codex -C /path/to/project      # 设置工作目录
codex --add-dir /extra/path    # 授予额外目录写权限
codex --oss                    # 通过 Ollama 使用本地模型
```

### Exec 模式（脚本/CI）

```bash
codex exec "update changelog" --json          # JSONL 输出
codex exec "fix lint" --ephemeral             # 不保存会话
codex exec "generate types" -o output.md      # 写结果到文件
codex exec "validate schema" --output-schema schema.json  # 验证输出
codex exec resume SESSION_ID                  # 恢复非交互式会话
```

## 斜杠命令（24 个内置）

### 模型与配置
| 命令 | 用途 |
|------|------|
| `/model` | 会话中切换模型 |
| `/personality` | 设置风格：`friendly`、`pragmatic`、`none` |
| `/status` | 显示会话配置 + token 用量 |
| `/debug-config` | 打印配置层和诊断信息 |

### 工作流
| 命令 | 用途 |
|------|------|
| `/plan` | 进入计划模式（先思考再行动） |
| `/permissions` | 调整审批预设 |
| `/compact` | 总结对话（节省上下文） |
| `/diff` | 查看 Git 变更（含未追踪文件） |
| `/review` | 分析工作树（代码审查） |

### 上下文
| 命令 | 用途 |
|------|------|
| `/mention` | 附加特定文件到对话 |
| `/mcp` | 列出 MCP 工具 |
| `/apps` | 浏览可用连接器 |

### 会话
| 命令 | 用途 |
|------|------|
| `/new` | 新对话，同一 CLI 会话 |
| `/resume` | 重新加载之前的会话 |
| `/fork` | 克隆对话到新线程 |
| `/init` | 生成 AGENTS.md 脚手架 |

### 工具
| 命令 | 用途 |
|------|------|
| `/ps` | 监控后台终端 |
| `/statusline` | 配置底部显示 |
| `/feedback` | 提交诊断信息 |
| `/quit` / `/exit` | 退出（先保存工作！） |

## AGENTS.md —— 控制平面

AGENTS.md 之于 Codex，就像 CLAUDE.md 之于 Claude Code。它提供持久化指令，Codex 在每次任务前都会读取。

### 发现层级（3 层）

```
1. ~/.codex/AGENTS.override.md  （全局覆盖 —— 最高优先级）
2. ~/.codex/AGENTS.md           （全局默认）
3. <git-root>/AGENTS.md         （项目级别）
4. <git-root>/services/payments/AGENTS.override.md  （嵌套覆盖）
```

文件从根目录向下拼接。越近的文件覆盖越早的内容。

### AGENTS.md 示例

```markdown
# 项目指令

## 构建与测试
- PR 前运行 `npm run lint`
- 所有变更运行 `npm test`
- 支付服务专用 `make test-payments`

## 代码风格
- TypeScript 严格模式，禁止 `any`
- 函数式模式优先于类
- 公共工具方法必须在 `docs/` 中文档化

## 安全
- 轮换 API 密钥前必须通知 #security 频道
- 数据库访问必须使用参数化查询
```

### 配置

```toml
# ~/.codex/config.toml
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
project_doc_max_bytes = 65536  # 默认 32 KiB
```

更改立即生效 —— 无需清除缓存。

## 高级配置（config.toml）

### 配置档案

```toml
[profiles.fast]
model = "gpt-5.3-codex"
model_reasoning_effort = "low"
approval_policy = "on-request"

[profiles.careful]
model = "gpt-5.2-codex"
model_reasoning_effort = "high"
approval_policy = "untrusted"
```

切换：`codex --profile fast`

### 自定义模型提供者

```toml
[model_providers.proxy]
base_url = "http://proxy.example.com"
env_key = "OPENAI_API_KEY"
http_headers = { "X-Custom" = "value" }
```

### Shell 环境控制

```toml
[shell_environment_policy]
inherit = "core"
exclude = ["AWS_*", "AZURE_*"]
include_only = ["PATH", "HOME"]
```

### 可观测性（OpenTelemetry）

```toml
[otel]
# 追踪 API 请求、提示、工具审批
exporter = "otlp-http"  # 或 "otlp-grpc"、"none"
```

### 可写根目录（沙箱逃逸口）

```toml
[sandbox_workspace_write]
network_access = false
writable_roots = ["/Users/YOU/.pyenv/shims"]
```

## 提示词最佳实践（官方指南）

### 核心原则

1. **把 Codex 当高级工程师对待** —— 它应该自主收集上下文、规划、实现、测试和改进
2. **偏向行动** —— 除非真的被阻塞，否则不要以澄清问题结束回合
3. **批量操作** —— 预先决定所有需要的文件/资源，并行读取
4. **保持现有模式** —— 遵循代码库的约定、辅助函数、命名、格式

### 有效的做法

- 带文件路径和约束的具体提示
- 测试驱动：预先定义测试，让 Codex 迭代直到通过
- UI 任务附加截图（`-i screenshot.png`）
- 复杂任务用 `/plan`，简单任务跳过（最简单的约 25%）

### 无效的做法

- 模糊提示（"让它更好"）
- 在系统提示中要求状态更新（导致过早停止）
- 可以并行时却顺序读取文件
- 宽泛的 try-catch / 静默吞错
- `as any` 类型断言

### 前端专项技巧

官方指南明确警告"AI 生成风格"（AI slop）：
- 使用超越默认字体族的表达性排版
- 通过 CSS 变量定义视觉方向
- 使用有意义的动画（不是装饰性的）
- 用渐变/纹理代替纯色

## 工作流模式

### 日常开发循环

```
1. codex                    # 启动交互式会话
2. "explain this codebase"  # 了解项目
3. /plan                    # 规划复杂变更
4. --full-auto              # 自动审批执行
5. /review                  # 提交前审查
6. /diff                    # 检查 git 变更
```

### CI/自动化管线

```bash
# CI 中修复 lint
codex exec "fix all lint errors" --ephemeral --json

# 生成 changelog
codex exec "update CHANGELOG.md from recent commits" -o CHANGELOG.md

# 验证 PR
codex exec "review this PR for security issues" --output-schema review.json
```

### 代码审查工作流

```bash
# 对比 main 分支审查
/review base=main

# 审查特定关注点
/review --focus security,edge-cases

# GitHub PR 审查（通过 GitHub 应用）
# 在任何 PR 上评论 @codex review
```

### 云任务委派

```bash
codex cloud                       # 交互式任务选择器
codex cloud --env ENV_ID          # 在特定环境中启动
codex cloud --attempts 3          # best-of-N 尝试
codex apply                       # 本地应用云差异
```

### 会话持久化

```bash
codex resume                      # 从最近会话中选择
codex resume --all                # 显示所有目录的会话
codex resume SESSION_ID           # 指定特定会话
codex fork                        # 分支当前对话
```

## MCP 集成

通过 Model Context Protocol 连接外部工具：

```bash
codex mcp add my-server --stdio "npx @my/mcp-server"
codex mcp add my-http --url "https://mcp.example.com"
codex mcp list
codex mcp login my-http    # HTTP 服务器的 OAuth
```

## 社区最佳实践与技巧

### 来自 OpenAI 开发者社区

1. **测试驱动为王** —— 预先定义测试，让 Codex 迭代到绿灯。"当 AI 参与时，传统编程假设就不成立了"
2. **Tiger Style 编程** —— 严格方法论的红-绿-重构，追求最大可靠性
3. **高质量本地文档 > 网络抓取** —— 为代理配备完善的本地文档
4. **最小化 AGENTS.md 上下文污染** —— 非必要不加载文档，保持 token 效率
5. **增量打补丁** —— 分别修改 import、函数签名、返回值（不支持模糊匹配）
6. **处理换行符** —— LF vs CRLF 不一致会导致补丁失败

### VS Code 集成

安装 "Codex Companion" 扩展：
- 侧边栏 `@codex` 直接对话
- `Alt+G` 将选中代码发送到 CLI
- Agent 模式：读文件、运行命令、写变更

### 高级技巧

- **会话导出/导入**：`/export session.json` → `/load session.json` 实现跨长期重构的上下文持久化
- **档案切换上下文**：`--profile fast` 快速修复，`--profile careful` 生产变更
- **图片驱动开发**：用 `-i` 附加原型图进行 UI 实现
- **并行读取**：始终批量读取文件以减少往返

## Codex CLI vs Claude Code —— 正面对决

| 维度 | Codex CLI | Claude Code |
|------|-----------|-------------|
| **构建方** | OpenAI（Rust） | Anthropic（TypeScript） |
| **模型** | GPT-5.3-Codex | Claude Opus 4.6 |
| **云执行** | 有（Codex Cloud） | 无（纯本地） |
| **Token 效率** | 2-3 倍更少 | 更深入但更贵 |
| **代码库理解** | 增量变更强 | 全局/架构级更好 |
| **指令文件** | AGENTS.md | CLAUDE.md |
| **沙箱** | OS 级别（seatbelt/landlock） | 权限系统（经常被绕过） |
| **GitHub 集成** | 原生应用（自动 PR 审查） | 基于 gh CLI |
| **权限 UX** | Git 感知，默认宽松 | 摩擦大，设置不持久 |
| **IDE 扩展** | Codex Companion（VS Code） | Claude Code 扩展 |
| **自动化** | `codex exec`（一等公民） | 有限的脚本支持 |
| **成本** | 包含在 ChatGPT 订阅中 | 按 token API 计费 |

### 何时用哪个

- **Codex**：速度、成本效率、GitHub 原生工作流、CI 自动化、常规编码任务
- **Claude Code**：深度代码库理解、复杂多文件重构、架构决策、调查性工作

### 社区共识

> "用 Claude Code 做调查和高层决策。用 Codex 在你确定路线后产出代码。"

最优工作流是多工具协作：用 Claude Code 理解和规划，用 Codex 执行和迭代。

## 权衡

| 优势 | 局限 |
|------|------|
| Token 效率比 Claude Code 高 2-3 倍 | Windows 支持仍为实验性（需要 WSL） |
| 云执行（异步后台任务） | 需要 ChatGPT 订阅（无免费层） |
| 原生 GitHub 应用做 PR 审查 | 沙箱默认阻断网络（API 工作有摩擦） |
| Rust 构建（快速，低内存） | 生态较新，社区模式较少 |
| 一等脚本支持（`codex exec`） | 复杂多文件重构验证不足 |
| 档案系统做上下文切换 | AGENTS.md 32KB 限制（可配置） |

## 可偷 —— 可直接复用的模式

### 1. 测试驱动代理循环
预先定义测试，让代理迭代到绿灯。这是 AI 辅助编码中影响最大的单一模式。在 Codex 和 Claude Code 中都适用。

### 2. AGENTS.md / CLAUDE.md 分层
两个工具都支持层级式指令文件。模式：全局默认 → 项目约定 → 子目录覆盖。保持指令简洁，使用渐进式披露。

### 3. 基于档案的工作流
```toml
[profiles.explore]
approval_policy = "untrusted"
model_reasoning_effort = "low"

[profiles.ship]
approval_policy = "on-request"
model_reasoning_effort = "high"
```

切换上下文无需记忆标志。

### 4. exec 集成 CI
```bash
codex exec "review for security vulnerabilities" --json --ephemeral
```
非交互模式让 AI 代码审查成为管线步骤，而非手动流程。

### 5. 云 + 本地混合
用 `codex cloud` 做慢速后台任务（大型重构、测试生成），`codex apply` 拉取结果到本地工作区。这是 Codex 独有的 —— Claude Code 没有对应方案。

### 6. 多工具工作流
```
Claude Code（理解）→ Codex（执行）→ Claude Code（审查）
```
让每个工具发挥其强项。不要选边 —— 两个都用。

### 7. 图片驱动 UI 开发
```bash
codex -i mockup.png "implement this design using React + Tailwind"
```
直接附加截图、原型图或 Figma 导出。比用文字描述布局快得多。

### 8. 增量打补丁纪律
在 AGENTS.md 中给代码修改写指令时，明确指定："一次打一个补丁 —— import、函数签名、返回值分开。"防止上下文不匹配导致补丁失败。
