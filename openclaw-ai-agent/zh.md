# OpenClaw — 开源个人 AI Agent (v2026.2.6)

## 摘要

OpenClaw（前身 Clawdbot/Moltbot）是 Peter Steinberger（PSPDFKit 创始人）开发的开源自托管 AI Agent 框架。它将 LLM（Claude、GPT、Grok、本地模型）与你的操作系统桥接——让 AI 完全访问文件、Shell、浏览器和 50+ 集成（WhatsApp、Slack、Discord、Gmail 等）。GitHub 145K+ star。最新版本 v2026.2.6（2026年2月7日）新增 Opus 4.6、GPT-5.3-Codex、安全扫描器和 Token 仪表板。

---

## 它是什么

一个本地优先的 AI Agent 网关：
- 运行在你的机器上（Mac/Windows/Linux）
- 连接任何 LLM（Claude、GPT、Gemini、Grok、DeepSeek、本地 Llama）
- 通过消息应用交互（WhatsApp、Telegram、Discord、Slack、Signal、iMessage）
- 拥有 100+ AgentSkills 用于自主任务执行
- 跨会话持久化记忆

**不是聊天机器人** — 是一个主动代你行事的 Agent，全天候运行。

---

## 架构

### 三组件设计

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Brain     │     │   Memory    │     │    Arms     │
│  (LLM 层)   │────▶│  (存储层)   │────▶│  (工具层)   │
│              │     │             │     │             │
│ 系统提示词    │     │ 短期记忆:   │     │ fs_tool     │
│ + 聊天历史   │     │  聊天日志    │     │ bash_tool   │
│ → LLM API   │     │ 长期记忆:   │     │ browser_tool│
│              │     │  .md 文件   │     │ (CDP)       │
└─────────────┘     └─────────────┘     └─────────────┘
```

| 组件 | 角色 | 实现方式 |
|------|------|----------|
| **Brain** | 构建系统提示词 + 发送给 LLM | 外部 LLM API（Claude、GPT 等） |
| **Memory** | 本地优先持久化 | 扁平文件 Markdown 存储于 `~/.openclaw/memory` |
| **Arms** | 宿主 OS 上的 JSON API 工具 | fs_tool、bash_tool、browser_tool (Chromium CDP) |

### Agent 循环（Think → Plan → Act → Observe → Iterate）

1. **Think** — LLM 分析请求，查询记忆
2. **Plan** — 将请求分解为工具调用序列
3. **Act** — 在机器上执行工具
4. **Observe** — 解读 stdout/stderr
5. **Iterate** — 根据观察优化方案，循环直到任务完成

### 网关架构

Gateway 是管理以下内容的单一控制平面：
- 会话（按 channel/peer/team 作用域隔离）
- 渠道（WhatsApp、Telegram、Slack、Discord 等）
- 工具（技能调度）
- 事件（调度、心跳、Webhook）

访问地址：`http://127.0.0.1:18789/` 本地 Web UI 仪表板。

### 工作区文件结构

| 文件 | 用途 |
|------|------|
| `AGENTS.md` | 操作指令 |
| `SOUL.md` | 人格和行为边界 |
| `MEMORY.md` | 长期整理的知识 |
| `memory/YYYY-MM-DD.md` | 每日追加日志 |
| `HEARTBEAT.md` | 自动化健康检查清单 |

---

## 技能系统（AgentSkills）

### 工作原理

Skills = 包含 `SKILL.md`（带 YAML 前置数据）的目录。声明式——告诉 Agent 使用什么工具，而非如何编码。

### 加载优先级

1. **工作区技能** (`<workspace>/skills`) — 最高优先
2. **本地管理技能** (`~/.openclaw/skills`) — 跨 Agent 共享
3. **内置技能** — 随安装包附带

### 技能结构

```yaml
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
homepage: https://example.com
user-invocable: true
metadata:
  openclaw:
    requires:
      bins: [ffmpeg]
      env: [GEMINI_API_KEY]
    os: [darwin, linux]
---
# 给 Agent 的指令...
```

### 门控系统

技能在加载时自动过滤：
- PATH 中需要的二进制文件
- 需要的环境变量
- 需要的配置路径
- 操作系统平台限制

### ClawHub 注册表

公共市场：`clawhub.com`

```bash
clawhub install <skill-slug>
clawhub update --all
clawhub sync --all
```

### Token 成本

每个技能约 24 tokens + 字段长度。系统提示词开销：基础 195 字符 + 每个技能 97 字符。

### 安全警告

**将第三方技能视为不受信任的代码。** v2026.2.6 新增了代码安全扫描器，但并非万无一失。启用前务必阅读技能代码。

---

## 支持的 AI 模型 (v2026.2.6)

| 提供商 | 模型 |
|--------|------|
| **Anthropic** | Claude Opus 4.6、Sonnet 4.5 及更早版本 |
| **OpenAI** | GPT-5.3-Codex、GPT-4o 及更早版本 |
| **xAI** | Grok（v2026.2.6 新增） |
| **Baidu** | 千帆（v2026.2.6 新增） |
| **Google** | Gemini 系列 |
| **本地** | Llama 3，任何 Ollama 兼容模型 |

新模型标识符的前向兼容降级（如 Opus 4.6 → 提供商不可用时优雅降级）。

---

## 集成 (50+)

| 类别 | 平台 |
|------|------|
| **消息** | WhatsApp、Telegram、Discord、Slack、Signal、iMessage (BlueBubbles)、Microsoft Teams、Matrix、Google Chat、Zalo、WebChat |
| **办公** | Gmail、Google Calendar、Notion、GitHub |
| **服务** | Stripe、AWS、Zapier、Spotify、YouTube、Figma、Dropbox、Airtable、Twitter/X |
| **智能家居** | 通过集成支持多种设备 |
| **区块链** | Base (Ethereum L2)、MetaMask、DEX（通过技能） |

---

## v2026.2.6 发布 (2026年2月7日) — 新功能

### 模型
- Anthropic Opus 4.6 + OpenAI GPT-5.3-Codex 前向兼容降级
- xAI Grok 提供商
- Baidu 千帆提供商

### 安全
- **技能/插件代码安全扫描器** — 扫描社区技能中的恶意模式
- `config.get` 网关响应中的凭据脱敏
- 网关 Canvas 宿主 + A2UI 资源现需认证

### 功能
- Web UI 中的 **Token 用量仪表板**
- **Voyage AI** 原生支持记忆嵌入
- 会话历史载荷上限（防止上下文溢出）
- CLI 帮助输出按字母排序

### 修复
- 调度/提醒投递回归修复
- Telegram: DM 主题自动注入 threadId
- Slack: `/new` 和 `/reset` 的提及剥离模式
- Chrome 扩展: 打包路径解析
- 上下文溢出的压缩重试逻辑
- 更好的计费错误提示

---

## 记忆系统

### 默认（基于 Markdown）

- 短期记忆：活跃聊天日志（即时上下文）
- 长期记忆：`~/.openclaw/memory` 中的 Markdown 摘要
- 通过 `memory_search` 工具对记忆块进行语义搜索（约 400 tokens）

### QMD 插件 (v2026.2.2+)

**本地混合智能引擎**，结合三种搜索技术：

| 技术 | 类型 | 优势 |
|------|------|------|
| BM25 | 关键词 | 精确词匹配 |
| Vector | 语义 | 基于含义的搜索 |
| LLM Rerank | 重排序 | 上下文相关性 |

优势：
- 搜索本地文件（Markdown、Notion 导出、Obsidian）
- 仅将 2-3 句相关内容拉入提示词
- 60-97% Token 节省
- 完全离线运行
- 告别上下文溢出

### Voyage AI (v2026.2.6+)

原生向量嵌入支持，增强记忆检索能力。

---

## 多 Agent 支持

- 每个 Agent 拥有独立隔离的工作区
- 独立的认证和会话存储
- 路由：peer → guildId → teamId → accountId → channel → 默认 Agent
- 每个 Agent 的技能在工作区目录中
- `~/.openclaw/skills` 中的共享技能对所有 Agent 可见
- 会话作用域模式：`per-channel-peer` 防止多用户场景中的上下文泄漏

---

## 链上 / Web3 集成

自 v2026.2.2 起，与 Base（Coinbase 的以太坊 L2）深度集成：
- Agent 可自主发起和管理区块链交易
- 社区项目：4claw、lobchanai、starkbotai
- 通过技能连接 MetaMask + DEX (ColorPool)
- Virtual Protocol：任何 OpenClaw Agent 可在链上发现、雇佣和支付其他 Agent
- 使用场景：钱包监控、空投自动化、自主交易

---

## 安装

```bash
# macOS/Linux
curl -fsSL https://openclaw.ai/install.sh | bash

# Windows
iwr -useb https://openclaw.ai/install.ps1 | iex

# npm
npm i -g openclawd

# Homebrew
brew install openclawd
```

**需求**：Node.js 22+

**设置**：`openclaw onboard --install-daemon` 运行交互式向导。

---

## 价格

| 组件 | 费用 |
|------|------|
| 软件 | 免费 (MIT 许可证) |
| 硬件 | $0（现有机器）到 $599（Mac mini M2） |
| API 用量 | 约 $5-50/月，取决于模型和使用量 |
| OpenClawd 托管 | 托管服务（2026年2月10日推出） |

无订阅费 — 仅按 API 请求计费。

---

## 安全分析 — 残酷真相

### 它需要的权限

- 文件系统（读/写/删除）
- Shell（任意命令执行）
- 浏览器会话（通过 CDP 的无头 Chromium）
- API 密钥、凭据、SSH 密钥
- 邮件、日历、消息账户

### 已知漏洞向量

| 向量 | 风险 | 严重性 |
|------|------|--------|
| **权限提升** | 以完整用户权限运行 — 可访问 SSH 密钥、`.env`、浏览器 Cookie | 严重 |
| **间接提示注入** | 读取含隐藏指令的网站/文档 → 执行数据窃取命令 | 严重 |
| **明文存储** | API 密钥和聊天历史以可读 `config.json` 和 `.md` 文件存储 | 高 |
| **供应链（技能）** | ClawHub 上的社区技能可能包含恶意脚本（钱包盗取、后门） | 高 |
| **上下文溢出** | 会话载荷溢出可导致不可预测行为 | 中 |

### 缓解最佳实践

1. **容器化** — Docker/VM 必须，永远不要在有敏感数据的裸机上运行
2. **网络隔离** — 防火墙监控出站流量（macOS 用 Little Snitch）
3. **人在回路** — bash、文件删除、git push 需要审批
4. **限定范围的 API 密钥** — 项目级凭据，预算上限（$5 最大）
5. **临时浏览器** — 与主配置文件分离的干净实例
6. **视为不受信任** — 假设 Agent 已被攻破来设计系统

> IBM 杰出工程师 Chris Hay：OpenClaw 让用户暴露在"太多安全漏洞"中，不适合企业使用。

---

## 替代方案对比

| 工具 | 类型 | 关键区别 |
|------|------|----------|
| **Nanobot** | 轻量级（4K 行 Python vs OpenClaw 的 430K+） | 研究友好、可审计 |
| **Moltworker** | Cloudflare Workers 部署 | 无服务器、沙盒化、持久状态 |
| **Zapier** | 云端自动化 | 无代码，无需系统访问 |
| **Knolli** | 企业 AI 副驾驶 | 结构化权限、托管基础设施 |
| **Claude Code** | Anthropic CLI Agent | 代码专注、内建沙盒 |
| **Cursor/Windsurf** | IDE 集成 AI | 编辑器绑定，自主性较低 |

### 为什么选 OpenClaw

- 完全开源，无供应商锁定
- 模型无关（任何 LLM）
- 消息原生（通过 WhatsApp 和你的 Agent 聊天）
- 100+ 技能生态
- 社区驱动（145K+ star，20K+ fork）

### 为什么不选 OpenClaw

- 没有容器化就是安全噩梦
- 430K+ 行代码 — 巨大攻击面
- 企业不友好（IBM："漏洞太多"）
- 明文存储密钥
- 社区技能 = 供应链风险

---

## 关键洞察

### 1. 松耦合集成有时胜过垂直整合

IBM 的 Kaoutar El Maghraoui："垂直整合在某些领域因安全性而重要。但在其他领域，也许我们不需要。"

OpenClaw 证明自主 Agent 不需要单一供应商控制模型、记忆、工具和界面。社区驱动的开源 Agent 如果拥有完整系统访问权限，可以"极其强大"。

### 2. Agent = 网关模式

OpenClaw 的核心洞察：**网关**是 Agentic AI 的正确抽象。一个进程在 LLM 大脑和 OS 手臂之间调解，记忆作为状态层。这是每个个人 AI Agent 最终都会收敛到的模式。

### 3. 技能就是新的插件

SKILL.md 声明式格式（YAML 前置数据 + 自然语言指令）相当于 AI Agent 的浏览器扩展。ClawHub 就是 Chrome Web Store。相同的生态动态——以及相同的供应链风险。

### 4. 记忆才是护城河

QMD 插件（BM25 + Vector + LLM Rerank，全部本地）实现 60-97% Token 节省。这才是 OpenClaw 真正价值复利的地方——使用越久，它越了解你。切换成本变得天文数字级。

### 5. Web3 集成是真实的（不只是炒作）

与大多数"加密 + AI"玩法不同，OpenClaw 的链上集成有实际使用场景：自主交易、钱包监控、Agent 间支付。Base 生态（Coinbase L2）正在构建真实的基础设施。

---

## 最新動態 (2026)

### 创始人加入 OpenAI，项目转入基金会

2026年2月14日，Peter Steinberger 宣布加入 OpenAI，Sam Altman 在 X 上称其将"推动下一代个人 Agent"。OpenClaw 将过渡到一个独立的开源基金会，继续使用 MIT 许可证——由 OpenAI 赞助但不受其控制。这次 acqui-hire 表明 OpenAI 将个人 Agent 网关模式视为其未来战略的核心，而非附属项目。VentureBeat 称之为"ChatGPT 时代终结的开始"——从聊天界面转向代你行事的自主 Agent。

### 爆发式增长：180K+ Star，一周 200 万访客

OpenClaw 成为 GitHub 历史上增长最快的开源 AI 项目。到2026年2月中旬，项目已超过 180,000 GitHub star，一周内吸引超过 200 万访客。这一增长受到 Moltbook 项目的病毒式传播和广泛媒体报道（CNBC、Scientific American、Fortune、Bloomberg）的推动。Raspberry Pi Holdings 股价飙升 43%，因为爱好者纷纷在专用硬件上部署 OpenClaw。

### 严重安全危机（CVE-2026-25253 + 六个额外漏洞）

2026年2月3日披露了一个严重的远程代码执行漏洞（CVE-2026-25253，CVSS 8.8）——这是一个逻辑缺陷，允许攻击者通过受害者的浏览器一次点击即可窃取用户认证令牌并实现 RCE。该漏洞在 v2026.1.29 中被修复。随后，Endor Labs 研究人员又揭示了六个额外漏洞，包括 SSRF、缺失认证和路径遍历缺陷。Bitsight 观测到超过 30,000 个暴露在公网上的 OpenClaw 实例存在认证绕过条件。生态中记录了超过 341 个恶意技能在流通，独立扫描报告超过三分之一的社区技能包含漏洞或风险行为。

### 企业禁令：Meta、Google DeepMind 限制使用

Meta、Google DeepMind 及一系列 AI 公司在2026年1月底/2月初限制或禁止了 OpenClaw 的使用。Meta 高管告知员工，如果在工作笔记本上使用该软件将面临失业风险。Google DeepMind 完全撤回了其贡献的多个模块。这代表了开源 AI 运动中最重大的逆转之一——最初为项目做出贡献的公司因安全顾虑而主动撤退，其中包括 OpenClaw 核心模块可被改用于自主武器瞄准和关键基础设施攻击的演示。

### Astrix OpenClaw Scanner — 企业检测工具

Astrix Security 发布了 OpenClaw Scanner（在 PyPI 上免费提供），这是一个开源工具，用于检测企业环境中自主 AI Agent 的运行位置。它与现有 EDR 遥测数据（CrowdStrike Falcon、Microsoft Defender）配合使用，分析端点上的 OpenClaw 活动行为指标，并生成可移植报告——全部以只读方式访问，无数据外传。CrowdStrike 也通过其 AI Service Usage Monitor 仪表板增加了原生 OpenClaw 可见性。

### ClawBody — 物理机器人桥接

社区构建的 ClawBody 项目（Tom Rikert 开发）将 OpenClaw 桥接到物理机器人硬件，集成 MuJoCo 仿真支持，可在高保真 3D 物理环境中训练 Agent，然后再部署到真实电机。初始集成使用 Reachy Mini 人形平台进行多关节控制。专门的 OpenClaw Robotics Community 已经成立，ClawBox（预配置的 Jetson Orin Nano，针对 OpenClaw 生产使用优化）可在2026年 Q2 预订。

### Raspberry Pi 作为专用 AI Agent 硬件

Raspberry Pi 官方认可在 Pi 5（或 8GB RAM 的 Pi 4）上运行 OpenClaw，作为注重安全的部署方案——提供隔离性、全天候运行和低功耗。这引发了将消费级单板计算机作为专用 AI Agent 基础设施的更广泛趋势，Raspberry Pi 官方博客发布了配置指南。

---

## 可偷的模式

| 模式 | 可以偷什么 |
|------|-----------|
| **网关即控制平面** | 单进程调解 LLM ↔ OS，含会话/渠道/工具管理 |
| **SKILL.md 声明式格式** | YAML 前置数据 + 自然语言 = 可移植、模型无关的工具定义 |
| **门控系统** | 按 OS、二进制、环境变量自动过滤能力——用户零配置 |
| **记忆层级** | 短期（聊天）→ 长期（整理的 .md）→ 语义搜索（QMD/Voyage） |
| **工作区隔离** | 每个 Agent 独立工作区，含独立认证 + 会话存储，用于多 Agent |
| **会话作用域模式** | `per-channel-peer` 防止上下文泄漏——多用户场景必备 |
| **前向兼容降级** | 新模型标识符优雅降级——永不因未知模型而崩溃 |
| **心跳模式** | `HEARTBEAT.md` 自动化健康检查——自愈 Agent 基础设施 |

---

## References

### 官方
- OpenClaw GitHub: https://github.com/openclaw/openclaw
- OpenClaw 官网: https://openclaw.ai/
- OpenClaw 文档 — 入门: https://docs.openclaw.ai/start/getting-started
- OpenClaw 文档 — 技能: https://docs.openclaw.ai/tools/skills
- OpenClawd AI (托管平台): https://openclawd.ai/
- ClawHub (技能注册表): https://clawhub.com

### 版本发布
- v2026.2.6 发布: https://github.com/openclaw/openclaw/releases/tag/v2026.2.6
- v2026.2.6 公告 (X): https://x.com/openclaw/status/2020059808444084506
- v2026.2.6 报道 (CyberSecurity News): https://cybersecuritynews.com/openclaw-v2026-2-6-released/
- v2026.2.2 链上发布: https://evolutionaihub.com/openclaw-2026-2-2-ai-agent-framework-onchain/

### 分析与深度研究
- IBM Think — OpenClaw、Moltbook 与 AI Agent 的未来: https://www.ibm.com/think/news/clawdbot-ai-agent-testing-limits-vertical-integration
- Sapt — 架构、安全与最佳实践: https://sapt.ai/insights/openclaw-architecture-security-agentic-ai-best-practices
- DigitalOcean — 什么是 OpenClaw?: https://www.digitalocean.com/resources/articles/what-is-openclaw
- Sterlites — 架构、演进与安全风险: https://sterlites.com/blog/moltbot-local-first-ai-agents-guide-2026
- Wikipedia — OpenClaw: https://en.wikipedia.org/wiki/OpenClaw

### 指南与教程
- Molt Founders — OpenClaw 超级速查表: https://moltfounders.com/openclaw-mega-cheatsheet
- Codecademy — OpenClaw 教程: https://www.codecademy.com/article/open-claw-tutorial-installation-to-first-chat-setup
- DigitalOcean — 如何运行 OpenClaw: https://www.digitalocean.com/community/tutorials/how-to-run-openclaw

### 生态与技能
- Awesome OpenClaw Skills: https://github.com/VoltAgent/awesome-openclaw-skills
- QMD 记忆插件: https://github.com/sac34333/openclawmemory
- OpenClaw 扩展生态指南: https://help.apiyi.com/en/openclaw-extensions-ecosystem-guide-en.html

### 替代方案对比
- CodeConductor — 顶级 OpenClaw 替代方案: https://codeconductor.ai/blog/openclaw-alternatives/
- SuperPrompt — 2026年最佳 OpenClaw 替代方案: https://superprompt.com/blog/best-openclaw-alternatives-2026
- Metana — OpenClaw vs Moltbook: https://metana.io/blog/openclaw-vs-moltbook-what-are-the-key-differences/

### 新闻
- CNBC — 从 Clawdbot 到 OpenClaw: https://www.cnbc.com/2026/02/02/openclaw-open-source-ai-agent-rise-controversy-clawdbot-moltbot-moltbook.html
- Yahoo Finance — OpenClawd 托管平台发布: https://finance.yahoo.com/news/openclawd-ai-launches-hosted-platform-143600648.html
- CoinMarketCap — OpenClaw 与加密: https://coinmarketcap.com/academy/article/what-is-openclaw-moltbot-clawdbot-ai-agent-crypto-twitter
- The Defiant — OpenClaw x 加密生态: https://thedefiant.io/newsletter/defi-daily/the-openclaw-x-crypto-ecosystem

### 安全
- Security.com — OpenClaw 的崛起: https://www.security.com/expert-perspectives/rise-openclaw
- Xpert Digital — AI Agent 失控?: https://xpert.digital/en/local-ai-assistant/

### 最新 (2026 更新)
- TechCrunch — OpenClaw 创始人加入 OpenAI: https://techcrunch.com/2026/02/15/openclaw-creator-peter-steinberger-joins-openai/
- Peter Steinberger — OpenClaw、OpenAI 与未来: https://steipete.me/posts/2026/openclaw
- Fortune — Peter Steinberger 是谁?: https://fortune.com/2026/02/19/openclaw-who-is-peter-steinberger-openai-sam-altman-anthropic-moltbook/
- CNBC — Steinberger 加入 OpenAI: https://www.cnbc.com/2026/02/15/openclaw-creator-peter-steinberger-joining-openai-altman-says.html
- VentureBeat — OpenAI 收购标志 ChatGPT 时代终结: https://venturebeat.com/technology/openais-acquisition-of-openclaw-signals-the-beginning-of-the-end-of-the
- CrowdStrike — 安全团队须知: https://www.crowdstrike.com/en-us/blog/what-security-teams-need-to-know-about-openclaw-ai-super-agent/
- Help Net Security — OpenClaw Scanner: https://www.helpnetsecurity.com/2026/02/12/openclaw-scanner-open-source-tool-detects-autonomous-ai-agents/
- Infosecurity Magazine — 六个新漏洞: https://www.infosecurity-magazine.com/news/researchers-six-new-openclaw/
- SOCRadar — CVE-2026-25253 分析: https://socradar.io/blog/cve-2026-25253-rce-openclaw-auth-token/
- Meta 禁止 OpenClaw (TechBuzz): https://www.techbuzz.ai/articles/meta-bans-viral-ai-tool-openclaw-over-security-risks
- Fortune — OpenClaw 安全风险: https://fortune.com/2026/02/12/openclaw-ai-agents-security-risks-beware/
- Raspberry Pi — 将 Pi 变成 AI Agent: https://www.raspberrypi.com/news/turn-your-raspberry-pi-into-an-ai-agent-with-openclaw/
- Bloomberg — OpenClaw 推动 Raspberry Pi 股价飙升: https://www.bloomberg.com/news/articles/2026-02-17/ai-agent-openclaw-puts-raspberry-pi-shares-on-investor-radars
- Scientific American — OpenClaw 控制你的电脑: https://www.scientificamerican.com/article/moltbot-is-an-open-source-ai-agent-that-runs-your-computer/
- ClawBody GitHub (Reachy Mini 机器人): https://github.com/tomrikert/clawbody
- Astrix Security — OpenClaw Scanner: https://astrix.security/learn/blog/introducing-astrix-openclaw-moltbot-footprint-scanner/
