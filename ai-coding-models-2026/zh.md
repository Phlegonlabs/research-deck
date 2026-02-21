# AI 编程模型：OpenAI vs Anthropic — 2026 完整版图

## 摘要

截至 2026 年 2 月 7 日，Anthropic 和 OpenAI 正处于有史以来最激烈的 AI 编程竞赛中。两天前（2 月 5 日），两家公司在 20 分钟内先后发布了竞争模型：Claude Opus 4.6 和 GPT-5.3-Codex。没有一家公司拥有全面领先优势——各自在不同的基准、工作流和开发者群体中占主导。

**核心结论：** Claude 在 SWE-bench Verified（补丁生成）上领先。GPT-5.3-Codex 在 Terminal-Bench 2.0（代理终端工作流）上领先。竞争已从"哪个模型更聪明"转向"哪个工具/工作流更高效"。

---

## 1. 2026 年 2 月 5 日正面对决

| 基准 | Claude Opus 4.6 | GPT-5.3-Codex | 胜者 |
|------|-----------------|---------------|------|
| SWE-bench Verified | **80.8%** | 未报告 | Claude |
| SWE-bench Pro Public | 未报告 | **56.8%** | GPT-5.3 |
| Terminal-Bench 2.0 | 65.4% | **77.3%** | GPT-5.3 (+12pp) |
| OSWorld-Verified | **72.7%** | 64.7% | Claude (+8pp) |
| GPQA Diamond | **77.3%** | 73.8% | Claude |
| MMLU Pro | **85.1%** | 82.9% | Claude |
| GDPval-AA (Elo) | **1,606** | ~1,462 | Claude (+144 Elo) |
| Cybersecurity CTF | — | **77.6%** | GPT-5.3 |

**关键警告：** Anthropic 报告 SWE-bench Verified，OpenAI 报告 SWE-bench Pro Public。不同变体、不同题集。这是刻意策略——各自在偏好指标上宣称"领先"。

---

## 2. 完整模型时间线

### OpenAI（2025 年 4 月 — 2026 年 2 月）

| 日期 | 模型 | SWE-bench Verified | 核心特性 |
|------|------|--------------------|---------|
| 4/14 | GPT-4.1 / mini / nano | 54.6% | API 编程模型，100 万上下文，$0.10-$2.00/MTok |
| 4/16 | o3 | 69.1% | 推理模型，Codeforces 2727 Elo |
| 4/16 | o4-mini | 68.1% | 高性价比推理，Codeforces 2719 |
| 6/10 | o3-pro | — | 最大算力，$20/$80 每 MTok |
| 6/10 | o3 降价 | — | 降 80%：$10/$40 → $2/$8 |
| 8/7 | GPT-5 | 74.9% | 统一旗舰，比 o3 少用 22% token |
| 11/12 | GPT-5.1 + Codex-Max | 77.9% | 长周期编程，24 小时自主任务 |
| 12/18 | GPT-5.2 + Codex | 75.4% | 上下文压缩，SWE-bench Pro 56.4% |
| 2/5 | **GPT-5.3-Codex** | — | Terminal-Bench 77.3%，快 25% |

### Anthropic（2025 年 5 月 — 2026 年 2 月）

| 日期 | 模型 | SWE-bench Verified | 核心特性 |
|------|------|--------------------|---------|
| 5/22 | Opus 4 | 72.5% | 首个 Claude 4，发布时"最强编程模型" |
| 5/22 | Sonnet 4 | — | 混合推理（即时 + 扩展思考） |
| 8/5 | Opus 4.1 | 74.5% | 多文件重构 |
| 9 月 | Sonnet 4.5 | 77.2% | 代理工作流，并行计算可达 82% |
| 10 月 | Haiku 4.5 | 73.3% | 小模型匹配 Opus 4 |
| 11/24 | Opus 4.5 | **80.9%** | 首个 >80%，降价 67% |
| 2/5 | **Opus 4.6** | 80.8% | 100 万上下文，Agent Teams，发现 500+ 零日漏洞 |

---

## 3. OpenAI 深度分析

### GPT-4.1 家族 — 编程主力

API 专用。三种规格（GPT-4.1 / mini / nano）。100 万 token 上下文。设计决策：将"编程"与"推理"分离——GPT-4.1 做日常编程，o3 做复杂推理。双轨持续到 GPT-5 合并两者。

| 特性 | GPT-4.1 | mini | nano |
|------|---------|------|------|
| 上下文 | 100 万 | 100 万 | 100 万 |
| SWE-bench | 54.6% | — | — |
| 输入/MTok | $2.00 | $0.40 | $0.10 |
| 输出/MTok | $8.00 | $1.60 | $0.40 |

比 GPT-4o 高 60%（Windsurf 基准）。工具调用效率高 30%。发布当天入驻 Copilot。

### o3 / o4-mini — 推理编程

o3 先思考再编码——扩展思维链用于多步重构和复杂 bug 链。Codeforces 2727 Elo（超过 OpenAI 首席科学家 2665）。权衡：慢，每请求数分钟。

o4-mini 在 Codeforces 上击败 o3（2719 vs 2706），成本低约 50%。竞赛编程性价比更高，但真实世界 SWE-bench o3 仍占优（69.1% vs 68.1%）。

### GPT-5 — 代际飞跃（2025 年 8 月）

替代 GPT-4o / o3 / o4-mini / GPT-4.1 成为 ChatGPT 默认。核心洞察：比 o3 结果更好，同时**少用 22% token** 和 **45% 工具调用**。Token 效率直接转化为代理成本节省。$1.25/$10.00 每 MTok。

### GPT-5.3-Codex — 最新（2026 年 2 月 5 日）

| 基准 | GPT-5.3-Codex | vs GPT-5.2 |
|------|---------------|------------|
| Terminal-Bench 2.0 | 77.3% | **+13.3pp** |
| OSWorld-Verified | 64.7% | **+26.5pp** |
| Cybersecurity CTF | 77.6% | +10.2pp |
| SWE-bench Pro | 56.8% | +0.4pp |

推理快 25%。输出 token 比任何前代都少。深度 diff（透明化变更推理）。中途交互引导。首个被标记"网络安全高能力"的模型。

真正故事：SWE-bench Pro 提升微弱（+0.4%），但 Terminal-Bench（+13.3pp）和 OSWorld（+26.5pp）巨幅跃升——终端调试和真实计算机交互大幅进化。

### Codex 平台

不只是模型——完整代理平台：

```
┌─────────────────────────────────────────────┐
│  CODEX APP（Web + 桌面）                     │
│  指挥中心，并行代理                           │
├─────────────────────────────────────────────┤
│  CODEX CLI（终端）                           │
│  开源 Rust，本地运行                          │
├─────────────────────────────────────────────┤
│  CODEX MODELS（API）                         │
│  codex-1 → GPT-5-Codex → 5.2 → 5.3         │
└─────────────────────────────────────────────┘
```

核心设计：codex-1 = o3 在真实编程任务上通过 RL 微调。沙箱化（无网络）。AGENTS.md 做仓库级配置。内置 git worktree 并行。Automations（定时触发 issue 分类、CI 监控）。

Codex 桌面（2 月 2 日）：macOS 应用，代理独立运行 30 分钟，内置 worktree。Sam Altman："我们有史以来最受欢迎的内部产品。"

---

## 4. Anthropic Claude 深度分析

### Opus 4.6 — 最新（2026 年 2 月 5 日）

| 特性 | Opus 4.5 | Opus 4.6 |
|------|----------|----------|
| 上下文 | 200K | **100 万（beta）** |
| 输出 token | 32K | **128K** |
| Terminal-Bench 2.0 | 59.3% | **65.4%** |
| OSWorld | 66.3% | **72.7%** |
| 长上下文检索 | ~18% | **76%**（4 倍提升） |
| Agent Teams | 无 | **有** |
| 努力程度控制 | 无 | **有 (low/med/high/max)** |

**Agent Teams** — 多个 Claude Code 实例自主协调。主代理生成队友，队友之间直接通信。共享任务列表带依赖追踪。启用：`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`。压力测试：16 个代理写了 10 万行 C 编译器，能构建 Linux 6.9。

**发现 500+ 零日漏洞** — 红队在沙箱中给 Opus 4.6 提供 Python、调试器和模糊测试工具。自主发现 500+ 高危未知漏洞（GhostScript 崩溃、OpenSC 缓冲区溢出、CGIF 内存损坏）。

**定价：** $5/$25 每 MTok（与 4.5 相同）。比 Opus 4/4.1 的 $15/$75 降价 67%。

### Sonnet 4.5 — 日常主力

SWE-bench 77.2%（并行计算可达 82%）。最佳速度/质量比。$3/$15 每 MTok。最高 pass@5（55.1%）——唯一能解决其他模型都解不了的实例。

### Haiku 4.5 — 黑马

SWE-bench 73.3%——匹配 Opus 4。$1/$5 每 MTok。比 Sonnet 便宜 5 倍。适合写-测-修循环。

### Claude Code CLI

终端代理，本地执行，直连 API。上线 6 个月内 **10 亿美元年化收入**。2 月 5 日起可在 GitHub Copilot 使用。

核心能力：完整仓库遍历、多文件编辑、MCP 集成（Figma、Jira、GitHub）、会话传送、Chrome 集成、技能热重载、LSP 功能。

开发者转变："我现在更多时间在 reviewer 模式。" "本来 2 年的工作，6 个月高质量完成。"

---

## 5. 代理编码工具对比

### 四大工具

| 特性 | Claude Code | OpenAI Codex | Cursor | GitHub Copilot |
|------|------------|-------------|--------|----------------|
| 界面 | 终端 (CLI) | 云端 + 桌面 (Mac) | VS Code 分支 | IDE 扩展 |
| 多代理 | Agent Teams | 并行 worktree | 单代理 | Agent Mode |
| 执行 | 本地优先 | 云端优先 | 本地 + 云端 | 云端 |
| 自主性 | 高（人在环中） | 非常高（30 分钟） | 中等 | 中等 |
| Token 效率 | 比 Cursor 高 5.5 倍 | 未披露 | 100K-400K/请求 | 未披露 |
| MCP 集成 | 最佳 | 有限 | 模型无关 | GitHub 生态 |
| 模型 | 仅 Claude | 仅 GPT | 多模型 | 多模型 |
| 价格 | $20-200/月 | 有免费层 | $20-200/月 | $10-39/月 |

### 新兴替代

Cline（VS Code，最大灵活性）、Aider（开源终端）、Windsurf（AI IDE）、JetBrains Junie、Gemini CLI、AWS Kiro、Augment（企业）。

---

## 6. 定价全景

### OpenAI API

| 模型 | 输入/MTok | 输出/MTok | 最适场景 |
|------|----------|----------|---------|
| GPT-4.1 nano | $0.10 | $0.40 | 自动补全 |
| GPT-4.1 mini | $0.40 | $1.60 | 日常编程 |
| o4-mini | $1.10 | $4.40 | 竞赛编程 |
| GPT-5 | $1.25 | $10.00 | 通用编程 + 推理 |
| GPT-5.2 | $1.75 | $14.00 | 代理编程 |
| o3（降价后） | $2.00 | $8.00 | 高难推理 |
| o3-pro | $20.00 | $80.00 | 极难问题 |

### Anthropic API

| 模型 | 输入/MTok | 输出/MTok | 最适场景 |
|------|----------|----------|---------|
| Haiku 4.5 | $1 | $5 | 快速修复、迭代 |
| Sonnet 4.5 | $3 | $15 | 日常编程 |
| Opus 4.6 | $5 | $25 | 复杂架构 |
| Opus 4.6 (>200K) | $10 | $37.50 | 大型代码库分析 |

**成本优化：** Prompt 缓存 = 输入省 90%。批量 API = 全部省 50%。两者乘法叠加。

**核心洞察：** 每任务成本 > 每 token 成本。GPT-5 比 o3 少用 22% token。Claude Code 比 Cursor 少用 5.5 倍 token。每 token 最便宜的不一定是每任务最便宜的。

---

## 7. 基准测试深度分析

### SWE-bench Verified 排名（2026 年 2 月）

| 排名 | 模型 | 分数 |
|------|------|------|
| 1 | Claude Opus 4.6 (Thinking) | 80.8% |
| 2 | Claude Opus 4.5 | 80.9%（pass@1: 74.4%） |
| 3 | Gemini 3 Flash | 76.2% |
| 4 | GPT-5.2 | 75.4% |
| 5 | GLM-4.7（开源） | ~74.2% |
| 6 | Qwen3-Coder-Next（开源） | 70.6% |

### Terminal-Bench 2.0（代理 CLI）

| 排名 | 模型 | 分数 |
|------|------|------|
| 1 | GPT-5.3-Codex | **77.3%** |
| 2 | Claude Opus 4.6 | 65.4% |
| 3 | GPT-5.2-Codex | 64.0% |
| 4 | Claude Opus 4.5 | 59.8% |

### OSWorld-Verified（桌面自动化）

| 模型 | 分数 | 人类基线 |
|------|------|---------|
| Claude Opus 4.6 | **72.7%** | ~72% |
| GPT-5.3-Codex | 64.7% | ~72% |

Claude 匹配人类基线。GPT-5.3 比前代跃升 26pp。

### SWE-bench 变体解释

| 变体 | 关注点 | 现实检验 |
|------|--------|---------|
| Verified | 500 人工验证样本 | 黄金标准，最高 ~80% |
| Pro | 企业级，多语言 | 难得多，最高 ~57% |
| Live | 全新题，防污染 | 测试真正泛化 |
| Terminal-Bench 2.0 | 多步终端工作流 | 最接近代理编程实际 |

SWE-bench Pro 分数（23-57%）揭示了与 Verified（~80%）的真实差距。用哪个基准极大影响叙事。

---

## 8. 开发者生态与情绪

### 采用率（2025-2026）

| 指标 | 数值 |
|------|------|
| 定期使用 AI 工具的开发者 | 85% |
| 每周使用 AI 编程助手 | 65% |
| AI 用户中使用 ChatGPT | 82% |
| Copilot 使用率 | 68% |
| Gemini 使用率 | 47% |
| Claude 使用率 | 41% |
| 使用多个工具 | 26%+ |
| Codex 用户（2025 年 12 月起） | 100 万+ |

### METR 重磅研究

随机对照试验——16 名资深开发者，246 个任务：
- AI 工具让开发者在熟悉代码库上**慢了 19%**
- 开发者自估 AI 节省 20-24%——**与现实完全相反**
- 越熟悉代码库的开发者被拖慢越多
- 背景：2025 年初工具（Cursor Pro、Claude 3.5/3.7 Sonnet）。新模型可能不同。

**悖论：** AI 在你不了解代码库时最有价值——而这恰恰是你最想用它的时候。

### 社区工作流共识

没有单一最优工具。多工具策略主导：
- **Cursor** — 内联编辑、快速迭代（基线）
- **Claude Code** — 复杂推理、调试、架构（"最强编码大脑"）
- **Copilot** — 快速补全、低摩擦
- **Codex** — 并行委托、多代理（"最佳 PR 审查器"）
- **Cline** — 追求最大控制的高级用户

### 编码-写作权衡（Opus 4.6）

用户报告编码提升但写作下降。共识：编码用 Opus 4.6，散文用 Opus 4.5。Anthropic 重度优化推理基准——效果可见。

---

## 9. 非显而易见的洞察

### 1. 基准碎片化是策略
两家报告不同 SWE-bench 变体。聪明的消费者看 Terminal-Bench 2.0 和 OSWorld——两家都在这些上报告。

### 2. 开源快速追赶
Qwen3-Coder-Next（80B 中 3B 活跃）SWE-bench 70.6%。前沿闭源与最佳开源差距从 30+ 分缩小到不到 10 分。

### 3. 多代理是新前沿
单代理编码在商品化。差异化转向编排：Claude Agent Teams（群体自组织）vs Codex 桌面（并行 worktree、30 分钟自主）。

### 4. Token 效率是隐藏战场
Claude Code 比 Cursor 少用 5.5 倍 token。GPT-5 比 o3 少用 22%。规模化时这比每 token 定价更重要。

### 5. 企业谨慎
90% 企业团队使用 AI，但 AI PR 审查等待时间增加 4.6 倍。积极情绪从 70% 降到 60%。66% 开发者引用"几乎正确但不完全正确"问题。

---

## 10. 可偷用的模式

| 模式 | 描述 |
|------|------|
| **模型路由** | Haiku/nano 迭代，Sonnet/GPT-5 日常，Opus/Codex 架构 |
| **AGENTS.md / CLAUDE.md** | 仓库级代理配置——最高影响力的质量提升 |
| **每代理一个 worktree** | 每个代理独立 git worktree，无冲突并行 |
| **多工具工作流** | Cursor（内联）+ Claude Code（复杂）+ Copilot（补全）+ Codex（并行） |
| **按任务跟踪成本** | 追踪每完成任务的总 token，而非每 token 成本 |
| **沙箱执行** | 代理无网络，预装依赖，限制供应链风险 |
| **上下文压缩** | 长代理会话的摘要检查点 |
| **Prompt 缓存** | 输入省 90%，与批量 API 乘法叠加 |
| **Terminal-Bench 评估** | 比 SWE-bench Verified 更真实的代理评估 |
| **Codex 做 PR 审查** | 即使 Claude 为主的团队也应用 Codex PR 审查 |
| **版本特定选择** | Opus 4.6 编码，Opus 4.5 写作/文档 |
| **努力程度控制** | `/effort low` 简单补全，`/effort max` 难题 |

---

## 最新动态 (2026)

### Anthropic 爆发式增长与 300 亿美元 G 轮融资

Anthropic 于 2026 年 2 月 12 日完成 **300 亿美元 G 轮融资**，**投后估值 3800 亿美元**——有史以来第二大风投交易。年化收入达 **140 亿美元**，连续三年同比增长超 10 倍。财富 10 强中已有 8 家是 Claude 客户。Claude Code 单项年化收入达 **25 亿美元**（2026 年 1 月 1 日以来翻了一倍多），周活跃用户同期也翻倍。Anthropic 还在 2026 年 1 月推出 **Cowork**，通过 11 个开源插件将 Claude Code 的工程能力扩展到非编码知识工作（销售、法务、财务）。

来源：[TechCrunch](https://techcrunch.com/2026/02/12/anthropic-raises-another-30-billion-in-series-g-with-a-new-value-of-380-billion/)、[Anthropic](https://www.anthropic.com/news/anthropic-raises-30-billion-series-g-funding-380-billion-post-money-valuation)、[Constellation Research](https://www.constellationr.com/insights/news/anthropics-claude-code-revenue-doubled-jan-1)

### Cursor 2.0 与 Composer 模型

Cursor 于 2025 年 10 月发布 **2.0 版本**，推出首个自有模型：**Composer**，一个混合专家模型，通过在真实代码库中使用实际开发工具（语义搜索、文件编辑器、终端命令）进行强化学习训练。Composer 大部分交互在 30 秒内完成——号称比同等能力系统**快 4 倍**。Cursor 2.0 支持最多 **8 个并行代理**，每个在隔离的工作区克隆中运行。到 2026 年 2 月，**Composer 1.5** 发布，RL 训练扩大 20 倍，在 **Terminal-Bench 2.0 上得分 47.9%**，配备自适应思考和自摘要。Cursor 年化收入突破 **5 亿美元**，估值 **100 亿美元**，超过 50% 的财富 500 强采用（包括 Nvidia、Uber、Adobe）。2026 年 2 月新推出**插件系统**，代理可通过 Marketplace 连接外部工具（Amplitude、AWS、Figma、Stripe 预置）。

来源：[VentureBeat](https://venturebeat.com/ai/vibe-coding-platform-cursor-releases-first-in-house-llm-composer-promising)、[The New Stack](https://thenewstack.io/cursor-2-0-ide-is-now-supercharged-with-ai-and-im-impressed/)、[DigitalApplied](https://www.digitalapplied.com/blog/cursor-composer-1-5-ai-coding-model-guide)

### Devin 2.0 — 从 $500/月 降到 $20/月

Cognition 推出 **Devin 2.0**，价格大幅下调：从 **$500/月** 降至 **$20/月 Core 计划**。新版本引入交互式云端 IDE，用户可并行启动多个 Devin。据 Cognition 称，Devin 2.0 每 Agent 计算单元完成的初级任务量比前代**多 83%**。价格层级：Core（$20/月）、Team（$500/月含 250 ACU）、Enterprise（定制）。额外 ACU 每个 $2。

来源：[VentureBeat](https://venturebeat.com/programming-development/devin-2-0-is-here-cognition-slashes-price-of-ai-software-engineer-to-20-per-month-from-500)、[TechCrunch](https://techcrunch.com/2025/04/03/devin-the-viral-coding-ai-agent-gets-a-new-pay-as-you-go-plan/)

### GitHub Copilot Agent Mode 正式发布

GitHub Copilot 的 agent mode 正式全面可用，新增重要功能：**find_symbol 工具**实现语言感知的符号导航（C++、C#、TypeScript 及任何支持 LSP 的语言），**Agent Skills** 可为特定项目工作流定制 Copilot，以及 **Copilot Coding Agent**——异步自主后台代理，在 GitHub Actions 环境中工作并提交 PR 等待审查。**Explore agent** 现已集成 GitHub MCP 工具。

来源：[GitHub Newsroom](https://github.com/newsroom/press-releases/coding-agent-for-github-copilot)、[GitHub Features](https://github.com/features/copilot/whats-new)

### Google Gemini Code Assist Agent Mode

Google 的 **Gemini Code Assist** agent mode 现已对所有用户开放（不再限于 insiders）。功能包括：IDE 重启间保持代理/对话状态、实时 shell 命令输出、**Next Edit Predictions**（VS Code 预览版）、聊天中内联 diff 视图和直接 diff 编辑。Google AI Pro/Ultra 订阅者的更高模型请求限额现在在 **Gemini CLI、agent mode 和 Gemini Code Assist** 之间共享。底层模型为 **Gemini 2.5**。

来源：[Google Developers Blog](https://developers.googleblog.com/new-in-gemini-code-assist/)、[Google Developers](https://developers.google.com/gemini-code-assist/resources/release-notes)

### 开源模型进一步缩小差距

**DeepSeek V3.2**（6850 亿参数，128K 上下文）在 SWE-bench Verified 上得分 **70.2%**，提供标准版和 V3.2-Speciale（专攻数学/编码）两个变体。**MiniMax M2.5** 达到 **80.2%**——首个突破 80% 的非 Anthropic 模型。**GLM-5**（智谱 AI）达 **77.8%**，**Kimi K2.5** 达 **76.8%**。SWE-bench Verified 排行榜现有 **71 个评估模型**，部分对比中开源与闭源的前五名差距已缩小到 1 个百分点以内。

来源：[llm-stats.com](https://llm-stats.com/benchmarks/swe-bench-verified)、[VentureBeat](https://venturebeat.com/technology/qwen3-coder-next-offers-vibe-coders-a-powerful-open-source-ultra-sparse)、[Epoch AI](https://epoch.ai/benchmarks/swe-bench-verified)

### Windsurf 与 AWS Kiro 加入 IDE 大战

**Windsurf**（Wave 13）推出一流的**多代理并行会话**，支持 Git worktree、并列 Cascade 窗格，以及 **Arena Mode**（IDE 内盲测并列模型对比）。**AWS Kiro** 作为基于 VS Code 的新型规范驱动代理 IDE 发布，底层使用 Claude Sonnet 4.0/3.7，采用结构化开发方法论：Kiro 在写代码前自动生成 `requirements.md`，包含用户故事和 EARS 验收标准。Aurora DSQL MCP 服务器集成支持一键数据库感知开发。

来源：[Windsurf Changelog](https://windsurf.com/changelog)、[VentureBeat](https://venturebeat.com/programming-development/amazon-launches-kiro-its-own-claude-powered-challenger-to-windsurf-and-codex)

### 开发者 AI 采用率达 93%

据 **JetBrains AI Pulse**（2026 年 1 月），**93% 的开发者**现在定期使用 AI 编程工具——高于 2025 年中的 85%。MIT 科技评论将**生成式编程**列为 [2026 年 10 大突破技术](https://www.technologyreview.com/2026/01/12/1130027/generative-coding-ai-software-2026-breakthrough-technology/) 之一。讨论已不再是"要不要用 AI？"，而是"哪种 AI 工具组合能最大化产出？"

来源：[JetBrains AI Blog](https://blog.jetbrains.com/ai/2026/02/the-best-ai-models-for-coding-accuracy-integration-and-developer-fit/)、[MIT Technology Review](https://www.technologyreview.com/2026/01/12/1130027/generative-coding-ai-software-2026-breakthrough-technology/)

---

## 最终判决：谁赢了？

**没有单一赢家。**

| 类别 | 领先者 | 差距 |
|------|--------|------|
| SWE-bench Verified | Claude Opus 4.6 | 比 GPT-5.2 领先约 5pp |
| Terminal-Bench 2.0 | GPT-5.3-Codex | 比 Claude 领先约 12pp |
| OSWorld（桌面） | Claude Opus 4.6 | 比 GPT-5.3 领先约 8pp |
| 推理（GPQA、MMLU） | Claude Opus 4.6 | 约 2-4pp |
| 速度 | GPT-5.3-Codex | 推理快 25% |
| 上下文窗口 | Claude Opus 4.6 | 100 万 vs 40 万 |
| 多代理编排 | 平手 | 不同方法 |
| IDE 集成 | Cursor | 使用两家模型 |
| 企业采用 | GitHub Copilot | 市场主导 |
| 开源替代 | Qwen3-Coder-Next | 70.6%，成本零头 |
| 开发者心智（编码） | Claude | "最强编码大脑" |
| 开发者心智（速度） | Codex | "快速迭代" |
| Token 效率 | Claude Code | Cursor 的 5.5 倍 |

---

## References

### Official -- OpenAI

- [Introducing OpenAI o3 and o4-mini](https://openai.com/index/introducing-o3-and-o4-mini/)
- [Introducing GPT-4.1 in the API](https://openai.com/index/gpt-4-1/)
- [Introducing GPT-5](https://openai.com/index/introducing-gpt-5/)
- [Introducing GPT-5 for developers](https://openai.com/index/introducing-gpt-5-for-developers/)
- [Introducing GPT-5.2](https://openai.com/index/introducing-gpt-5-2/)
- [Introducing GPT-5.2-Codex](https://openai.com/index/introducing-gpt-5-2-codex/)
- [Introducing GPT-5.3-Codex](https://openai.com/index/introducing-gpt-5-3-codex/)
- [GPT-5.3-Codex System Card (PDF)](https://cdn.openai.com/pdf/23eca107-a9b1-4d2c-b156-7deb4fbc697c/GPT-5-3-Codex-System-Card-02.pdf)
- [Introducing Codex](https://openai.com/index/introducing-codex/)
- [Introducing the Codex App](https://openai.com/index/introducing-the-codex-app/)
- [Introducing upgrades to Codex](https://openai.com/index/introducing-upgrades-to-codex/)
- [Unlocking the Codex harness: App Server architecture](https://openai.com/index/unlocking-the-codex-harness/)
- [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [Codex product page](https://openai.com/codex/)
- [O3 80% price cut + o3-pro](https://community.openai.com/t/o3-is-80-cheaper-and-introducing-o3-pro/1284925)
- [OpenAI for Developers 2025](https://developers.openai.com/blog/openai-for-developers-2025/)
- [OpenAI API Pricing](https://openai.com/api/pricing/)
- [OpenAI Models comparison](https://platform.openai.com/docs/models/compare)

### Official -- Anthropic

- [Introducing Claude 4 (Opus 4 + Sonnet 4)](https://www.anthropic.com/news/claude-4)
- [Claude Opus 4.1](https://www.anthropic.com/news/claude-opus-4-1)
- [Introducing Claude Sonnet 4.5](https://www.anthropic.com/news/claude-sonnet-4-5)
- [Introducing Claude Opus 4.5](https://www.anthropic.com/news/claude-opus-4-5)
- [Introducing Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)
- [500+ Zero-Days Found (Red Team)](https://red.anthropic.com/2026/zero-days/)
- [Claude's Extended Thinking](https://www.anthropic.com/news/visible-extended-thinking)
- [The "think" Tool](https://www.anthropic.com/engineering/claude-think-tool)
- [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use)
- [Building C Compiler with Agent Teams](https://www.anthropic.com/engineering/building-c-compiler)
- [Claude SWE-Bench Performance](https://www.anthropic.com/research/swe-bench-sonnet)
- [Claude API Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Models Overview](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Claude Code Overview](https://code.claude.com/docs/en/overview)
- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents)
- [Anthropic raises $30B Series G at $380B valuation](https://www.anthropic.com/news/anthropic-raises-30-billion-series-g-funding-380-billion-post-money-valuation)

### Developer Documentation

- [Codex CLI on GitHub](https://github.com/openai/codex)
- [Codex CLI docs](https://developers.openai.com/codex/cli/)
- [Codex Models](https://developers.openai.com/codex/models/)
- [Codex Cloud](https://developers.openai.com/codex/cloud)
- [Gemini Code Assist overview](https://developers.google.com/gemini-code-assist/docs/overview)
- [Gemini Code Assist release notes](https://developers.google.com/gemini-code-assist/resources/release-notes)
- [GitHub Copilot features](https://docs.github.com/en/copilot/get-started/features)

### Benchmark Leaderboards

- [SWE-bench Verified -- llm-stats.com](https://llm-stats.com/benchmarks/swe-bench-verified)
- [SWE-bench Official](https://www.swebench.com/)
- [SWE-bench Pro Public -- Scale](https://scale.com/leaderboard/swe_bench_pro_public)
- [SWE-bench Live](https://swe-bench-live.github.io/)
- [Terminal-Bench 2.0 -- llm-stats.com](https://llm-stats.com/benchmarks/terminal-bench-2)
- [Terminal-Bench Official](https://www.tbench.ai/)
- [Live-SWE-agent Leaderboard](https://live-swe-agent.github.io/)
- [Best AI for Coding -- llm-stats.com](https://llm-stats.com/leaderboards/best-ai-for-coding)
- [SWE-bench Verified -- Epoch AI](https://epoch.ai/benchmarks/swe-bench-verified)
- [LLM Leaderboard -- Vellum](https://www.vellum.ai/llm-leaderboard)
- [Best LLM for Coding -- Vellum](https://www.vellum.ai/best-llm-for-coding)
- [EvalPlus Leaderboard (HumanEval)](https://evalplus.github.io/leaderboard.html)
- [LiveBench](https://livebench.ai/)
- [Artificial Analysis](https://artificialanalysis.ai/models)

### Head-to-Head Comparisons

- [Claude Opus 4.6 vs GPT-5.3 Codex -- DigitalApplied](https://www.digitalapplied.com/blog/claude-opus-4-6-vs-gpt-5-3-codex-comparison)
- [Claude vs Codex: AI Coding Agent Battle 2026 -- WaveSpeedAI](https://wavespeed.ai/blog/posts/claude-vs-codex-comparison-2026/)
- [Codex App vs Claude Code Showdown -- Serenities AI](https://serenitiesai.com/articles/openai-codex-app-vs-claude-code-2026)
- [Claude Code vs OpenAI Codex -- Northflank](https://northflank.com/blog/claude-code-vs-openai-codex)
- [Codex vs Claude Code -- Builder.io](https://www.builder.io/blog/codex-vs-claude-code)
- [GPT-5.3 Codex vs Claude Opus 4.6 -- NxCode](https://www.nxcode.io/resources/news/gpt-5-3-codex-vs-claude-opus-4-6-ai-coding-comparison-2026)
- [Codex CLI vs Claude Code Benchmark -- SmartScope](https://smartscope.blog/en/generative-ai/chatgpt/codex-vs-claude-code-2026-benchmark/)
- [Devin vs Cursor -- Builder.io](https://www.builder.io/blog/devin-vs-cursor)

### Analysis & Reviews

- [Best AI for Coding 2026: SWE-Bench Breakdown -- marc0.dev](https://www.marc0.dev/en/blog/best-ai-for-coding-2026-swe-bench-breakdown-opus-4-6-qwen3-coder-next-gpt-5-3-and-what-actually-matters-1770387434111)
- [GPT-5 Benchmarks -- Vellum](https://www.vellum.ai/blog/gpt-5-benchmarks)
- [Claude Opus 4.6 Benchmarks -- Vellum](https://www.vellum.ai/blog/claude-opus-4-6-benchmarks)
- [GPT-5.3-Codex Guide -- DigitalApplied](https://www.digitalapplied.com/blog/gpt-5-3-codex-release-features-benchmarks-guide)
- [Claude Opus 4.6 Guide -- DigitalApplied](https://www.digitalapplied.com/blog/claude-opus-4-6-release-features-benchmarks-guide)
- [Claude Opus 4.6 -- DataCamp](https://www.datacamp.com/blog/claude-opus-4-6)
- [O4-Mini -- DataCamp](https://www.datacamp.com/blog/o4-mini)
- [Best AI Coding Agents 2026 -- Faros AI](https://www.faros.ai/blog/best-ai-coding-agents-2026)
- [Best AI Coding Assistants 2026 -- PlayCode](https://playcode.io/blog/best-ai-coding-assistants-2026)
- [Best LLMs for Coding 2026 -- Builder.io](https://www.builder.io/blog/best-llms-for-coding)
- [AI Dev Tool Power Rankings -- LogRocket](https://blog.logrocket.com/ai-dev-tool-power-rankings/)
- [Best AI Coding Models 2026 -- CodeConductor](https://codeconductor.ai/blog/ai-coding-models/)
- [Best AI Models for Coding -- JetBrains AI Blog](https://blog.jetbrains.com/ai/2026/02/the-best-ai-models-for-coding-accuracy-integration-and-developer-fit/)
- [Cursor 2.0 -- The New Stack](https://thenewstack.io/cursor-2-0-ide-is-now-supercharged-with-ai-and-im-impressed/)
- [Cursor Composer 1.5 Guide -- DigitalApplied](https://www.digitalapplied.com/blog/cursor-composer-1-5-ai-coding-model-guide)
- [Devin 2.0 Review -- ai-coding-flow](https://ai-coding-flow.com/blog/devin-review-2026/)

### News Coverage

- [AI coding wars heat up -- VentureBeat](https://venturebeat.com/technology/openais-gpt-5-3-codex-drops-as-anthropic-upgrades-claude-ai-coding-wars-heat)
- [Claude Opus 4.6 1M context + Agent Teams -- VentureBeat](https://venturebeat.com/technology/anthropics-claude-opus-4-6-brings-1m-token-context-and-agent-teams-to-take)
- [GPT-5.3-Codex 25% faster -- Neowin](https://www.neowin.net/news/openai-debuts-gpt-53-codex-25-faster-and-setting-new-coding-benchmark-records/)
- [Opus 4.6 Agent Teams -- TechCrunch](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams/)
- [Claude Opus 4.6 -- CNBC](https://www.cnbc.com/2026/02/05/anthropic-claude-opus-4-6-vibe-working.html)
- [500 Zero-Days -- Axios](https://www.axios.com/2026/02/05/anthropic-claude-opus-46-software-hunting)
- [500 Zero-Days -- CSO Online](https://www.csoonline.com/article/4128889/claude-ai-finds-500-high-severity-software-vulnerabilities.html)
- [500 Zero-Days -- Fortune](https://fortune.com/2026/02/06/anthropic-claude-ai-model-cybersecurity-security-vulnerabilities-risks/)
- [GPT-5.3-Codex cybersecurity risks -- Fortune](https://fortune.com/2026/02/05/openai-gpt-5-3-codex-warns-unprecedented-cybersecurity-risks/)
- [Opus 4.6 coding-writing tradeoff -- Winbuzzer](https://winbuzzer.com/2026/02/05/claude-opus-4-6-coding-writing-tradeoff-xcxwbn/)
- [GPT-5.1-Codex-Max 24hr task -- VentureBeat](https://venturebeat.com/ai/openai-debuts-gpt-5-1-codex-max-coding-model-and-it-already-completed-a-24)
- [AI coding everywhere -- MIT Technology Review](https://www.technologyreview.com/2025/12/15/1128352/rise-of-ai-coding-developers-2026/)
- [Generative coding: 10 Breakthrough Technologies 2026 -- MIT Technology Review](https://www.technologyreview.com/2026/01/12/1130027/generative-coding-ai-software-2026-breakthrough-technology/)
- [Opus 4.6 GitHub Copilot -- GitHub Changelog](https://github.blog/changelog/2026-02-05-claude-opus-4-6-is-now-generally-available-for-github-copilot/)
- [Opus 4.6 on Azure -- Azure Blog](https://azure.microsoft.com/en-us/blog/claude-opus-4-6-anthropics-powerful-model-for-coding-agents-and-enterprise-workflows-is-now-available-in-microsoft-foundry-on-azure/)
- [Anthropic $30B Series G -- TechCrunch](https://techcrunch.com/2026/02/12/anthropic-raises-another-30-billion-in-series-g-with-a-new-value-of-380-billion/)
- [Anthropic $30B Series G -- Bloomberg](https://www.bloomberg.com/news/articles/2026-02-12/anthropic-finalizes-30-billion-funding-at-380-billion-value)
- [Claude Code revenue doubled -- Constellation Research](https://www.constellationr.com/insights/news/anthropics-claude-code-revenue-doubled-jan-1)
- [Devin 2.0 price cut -- VentureBeat](https://venturebeat.com/programming-development/devin-2-0-is-here-cognition-slashes-price-of-ai-software-engineer-to-20-per-month-from-500)
- [Devin 2.0 -- Cognition](https://cognition.ai/blog/devin-2)
- [Cursor Composer model -- VentureBeat](https://venturebeat.com/ai/vibe-coding-platform-cursor-releases-first-in-house-llm-composer-promising)
- [AWS Kiro launch -- VentureBeat](https://venturebeat.com/programming-development/amazon-launches-kiro-its-own-claude-powered-challenger-to-windsurf-and-codex)
- [Copilot Coding Agent -- GitHub Newsroom](https://github.com/newsroom/press-releases/coding-agent-for-github-copilot)

### Developer Productivity Studies

- [METR: AI Impact on Experienced OS Devs](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)
- [METR Paper -- arXiv](https://arxiv.org/abs/2507.09089)
- [Stack Overflow 2025 AI Survey](https://survey.stackoverflow.co/2025/ai)
- [JetBrains State of Developer Ecosystem 2025](https://blog.jetbrains.com/research/2025/10/state-of-developer-ecosystem-2025/)
- [AI Coding Impact 2026 -- Opsera](https://opsera.ai/resources/report/ai-coding-impact-2025-benchmark-report/)
- [AI Pair Programming Statistics -- Index.dev](https://www.index.dev/blog/ai-pair-programming-statistics)

### Multi-Agent & Agentic Coding

- [Claude Agent Teams Guide -- ClaudeFast](https://claudefa.st/blog/guide/agents/agent-teams)
- [Agent Teams Setup -- marc0.dev](https://www.marc0.dev/en/blog/claude-code-agent-teams-multiple-ai-agents-working-in-parallel-setup-guide-1770317684454)
- [Claude Code Hidden Swarm -- paddo.dev](https://paddo.dev/blog/claude-code-hidden-swarm/)
- [Multi-Agent Orchestration -- Shipyard](https://shipyard.build/blog/claude-code-multi-agent/)
- [Codex Desktop Parallel Agents -- VentureBeat](https://venturebeat.com/orchestration/openai-launches-a-codex-desktop-app-for-macos-to-run-multiple-ai-coding)
- [Codex Desktop -- DevOps.com](https://devops.com/openai-shifts-toward-autonomous-team-model-with-codex-desktop-launch/)
- [Terminal-Bench Paper -- arXiv](https://arxiv.org/html/2601.11868v1)
- [Terminal-Bench -- GitHub](https://github.com/laude-institute/terminal-bench)

### Open Source Models

- [Qwen3-Coder -- Qwen Blog](https://qwenlm.github.io/blog/qwen3-coder/)
- [Qwen3-Coder-Next -- Dev Genius](https://blog.devgenius.io/qwen3-coder-next-just-launched-open-source-is-winning-0724b76f13cc)
- [Qwen3-Coder-Next Sparse MoE -- Winbuzzer](https://winbuzzer.com/2026/02/04/alibaba-qwen3-coder-next-open-source-sparse-moe-coding-model-xcxwbn/)
- [Qwen3-Coder-Next -- VentureBeat](https://venturebeat.com/technology/qwen3-coder-next-offers-vibe-coders-a-powerful-open-source-ultra-sparse)

### Claude Code Coverage

- [Claude Code Product Page](https://claude.com/product/claude-code)
- [Creator of Claude Code Workflow -- VentureBeat](https://venturebeat.com/technology/the-creator-of-claude-code-just-revealed-his-workflow-and-developers-are)
- [Karpathy's Claude Code Field Notes -- DEV](https://dev.to/jasonguo/karpathys-claude-code-field-notes-real-experience-and-deep-reflections-on-the-ai-programming-era-4e2f)
- [Claude Code Has Engineers Buzzing -- GeekWire](https://www.geekwire.com/2026/a-new-era-of-software-development-claude-code-has-seattle-engineers-buzzing-as-ai-coding-hits-new-phase/)
- [Claude Code CLI Cheatsheet -- Shipyard](https://shipyard.build/blog/claude-code-cheat-sheet/)
- [Claude Code's "ChatGPT" moment -- Uncover Alpha](https://www.uncoveralpha.com/p/anthropics-claude-code-is-having)
- [Claude Code creator predicts -- AOL](https://www.aol.com/articles/anthropics-claude-code-creator-predicts-100101646.html)

### Pricing Guides

- [OpenAI Pricing 2026 -- Finout](https://www.finout.io/blog/openai-pricing-in-2026)
- [Anthropic API Pricing 2026 -- nOps](https://www.nops.io/blog/anthropic-api-pricing/)
- [LLM API Pricing Comparison -- CloudIDR](https://www.cloudidr.com/llm-pricing)
- [Claude Code Pricing -- o-mega](https://o-mega.ai/articles/claude-code-pricing-2026-costs-plans-and-alternatives)

### IDE & Tool Coverage

- [Windsurf Changelog](https://windsurf.com/changelog)
- [Windsurf Review 2026 -- Second Talent](https://www.secondtalent.com/resources/windsurf-review/)
- [AWS Kiro -- The New Stack](https://thenewstack.io/kiro-is-awss-specs-centric-answer-to-windsurf-and-cursor/)
- [Gemini Code Assist Updates -- Google Blog](https://developers.googleblog.com/new-in-gemini-code-assist/)
- [Cursor Changelog 2026 -- PromptLayer](https://blog.promptlayer.com/cursor-changelog-whats-coming-next-in-2026/)
- [GitHub Copilot What's New](https://github.com/features/copilot/whats-new)
- [GitHub Copilot Agents](https://github.com/features/copilot/agents)

### Community & Discussion

- [Claude Code vs Codex Sentiment -- Hacker News](https://news.ycombinator.com/item?id=45610266)
- [Claude vs Cursor vs Copilot -- Cursor Forum](https://forum.cursor.com/t/comparison-claude-vs-cursor-vs-copilot-review-from-a-regular-coder/130701)
- [METR Study Discussion -- Sean Goedecke](https://www.seangoedecke.com/impact-of-ai-study/)
- [OpenAI Codex App Analysis -- Latent Space](https://www.latent.space/p/ainews-openai-codex-app-death-of)

### Wikipedia

- [OpenAI o3](https://en.wikipedia.org/wiki/OpenAI_o3)
- [GPT-4.1](https://en.wikipedia.org/wiki/GPT-4.1)
- [Claude (Language Model)](https://en.wikipedia.org/wiki/Claude_(language_model))
