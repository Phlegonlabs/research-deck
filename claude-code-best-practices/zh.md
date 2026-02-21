# Claude Code 最佳實踐

深入研究 Claude Code CLI 的生產工作流程、配置模式和進階技巧，最大化開發效率。

## 他們構建了什麼

Claude Code 是 Anthropic 官方的 AI 輔助開發 CLI 工具。社群已發展出完整的最佳實踐體系：

- **CLAUDE.md 配置** - 跨會話持久化的項目級 AI 上下文
- **終端整合** - Ghostty + tmux 並行開發工作流
- **權限管理** - Hooks、沙盒和安全護欄
- **上下文優化** - Token 管理和對話新鮮度策略
- **多代理編排** - 並行 Claude Code 實例處理複雜任務

## 為什麼選擇這種方法

### 上下文視窗問題

LLM 性能隨上下文填充而下降。單次調試會話可能消耗數萬 tokens。社群的解決方案：

| 策略 | 為什麼有效 |
|------|-----------|
| 頻繁使用 `/clear` | 防止陳舊上下文降低輸出質量 |
| 重大工作前使用 `/compact` | 壓縮歷史，釋放約 70% token 空間 |
| 每個主題開新對話 | 保持模型峰值性能 |
| 交接文檔 | 跨會話邊界保留進度 |

### CLAUDE.md 優於重複提示

與其每次會話重複指令，`CLAUDE.md` 提供：
- **持久性** - 對話開始時自動載入
- **版本控制** - 團隊共享的項目知識
- **Token 效率** - 無需重複注入上下文

但有限制：前沿 LLM 只能一致地遵循約 150-200 條指令。過度填充 CLAUDE.md 會降低遵從度。

### 終端優先而非 IDE 擴展

| 方案 | 權衡 |
|------|------|
| **VS Code 擴展** | 高內存（8GB+）、更新滯後、並行實例有限 |
| **Ghostty 終端** | 每實例 <500MB、即時更新、無限並行會話 |

特別選擇 Ghostty 是因為其 GPU 加速渲染可防止長時間 AI 會話時的卡頓。

## 運作原理

### 三層配置層級

```
~/.claude/settings.json          # 用戶級默認值
.claude/settings.json            # 項目級（版本控制）
.claude/settings.local.json      # 個人級（gitignore）
```

設置合併時後面的文件覆蓋前面的。

### CLAUDE.md 結構（What/Why/How）

```markdown
# WHAT - 給 Claude 一張地圖
- 技術棧：Next.js 14、TypeScript、Prisma、PostgreSQL
- 結構：/src/app（路由）、/src/lib（工具）、/src/components（UI）

# WHY - 目的和上下文
- 這是一個 SaaS 計費儀表板
- 我們優先考慮類型安全而非開發速度

# HOW - 工作指令
- 使用 `bun` 而非 `npm`
- 提交前運行 `bun test`
- 永遠不要直接修改 /src/generated/* 文件
```

### Hooks 架構

Hooks 攔截工具調用以實現自動化護欄：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "check-dangerous-commands.sh"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "command": "audit-log.sh"
      }
    ]
  }
}
```

**阻塞式 hooks**（Trail of Bits 模式）：
- 阻止 `rm -rf`（建議改用 `trash`）
- 阻止直接推送到 main（要求功能分支）
- 反合理化關卡，強制完成未完成的工作

### tmux 整合

tmux 實現持久化、並行的 Claude Code 會話：

```bash
# 創建命名會話
tmux new-session -s claude-main

# 分割成多個窗格
tmux split-window -h
tmux split-window -v

# 在每個窗格運行 Claude
claude  # 窗格 1：主功能
claude  # 窗格 2：測試
claude  # 窗格 3：文檔
```

**核心優勢：**
- 會話持久化（斷開連接後保留）
- 並行任務執行
- 通知整合（點擊通知 → 跳轉到窗格）
- 後台自主任務

### Ghostty 配置

```ini
# ghostty 配置 (~/.config/ghostty/config)
theme = catppuccin-mocha
font-family = JetBrains Mono
font-size = 14
window-padding = 10

# 長時間 AI 會話的性能優化
scrollback-limit = 100000
term = xterm-256color
```

**原生優勢：**
- Shift+Enter 無需 `/terminal-setup` 即可使用
- GPU 渲染防止內存膨脹
- 分割窗格會話分叉

## 權衡取捨

### 新鮮上下文 vs 連續性

| 選擇 | 獲得 | 失去 |
|------|------|------|
| 頻繁 `/clear` | 峰值性能、低 token 成本 | 項目上下文、對話歷史 |
| 長會話 | 連續性、積累的上下文 | 輸出質量下降、高 token 使用 |

**緩解方案：** `/clear` 前寫交接文檔：
```
/compact
"寫一份交接文檔，總結當前進度和下一步計劃"
# 保存輸出，然後 /clear
```

### 權限嚴格度 vs 流暢度

| 模式 | 風險 | 摩擦 |
|------|------|------|
| `--dangerously-skip-permissions` | 完全系統訪問 | 零 |
| 白名單特定命令 | 中等（已知命令） | 低 |
| 默認（每次詢問） | 最小 | 高 |

**建議：** 使用 hooks 處理絕不允許的規則，用 `/permissions` 白名單處理常見安全操作。

### Opus vs Sonnet vs Haiku

| 模型 | 最適合 | Token 成本 |
|------|--------|-----------|
| **Opus 4.5** | 複雜多步規劃、架構設計 | 最高 |
| **Sonnet 4.5** | 默認編碼、常規任務 | 中等 |
| **Haiku 4.5** | 簡單查詢、快速查找 | 最低 |

系統默認在 50% 使用率時自動從 Opus 切換到 Sonnet。

## 備選方案比較

### IDE 擴展 vs CLI

| 工具 | 優勢 | 劣勢 |
|------|------|------|
| **Cursor** | 視覺 diff、內聯建議 | 專有、內存重 |
| **Cline (VS Code)** | IDE 整合 | 插件限制、更新較慢 |
| **Claude Code CLI** | 功能優先更新、可組合 | 學習曲線、無視覺 diff |

CLI 適合重視可組合性和最新功能的高級用戶。

### 本地模型 vs API

Trail of Bits 通過 LM Studio 本地運行 Qwen3-Coder-Next（80B MoE）：
- **優勢：** 隱私、無速率限制、離線能力
- **劣勢：** 硬件要求、與 Opus 的質量差距

## 可直接複用的模式

### 1. 狀態行自定義

```bash
# 顯示：模型 | 目錄 | 分支 | 未提交 | token%
# 安裝：claude /install-status-line
```

實時可見上下文消耗，防止 token 意外耗盡。

### 2. Git Worktrees 並行功能開發

```bash
git worktree add ../feature-auth feature/auth
git worktree add ../feature-billing feature/billing

# 運行獨立的 Claude 實例
cd ../feature-auth && claude
cd ../feature-billing && claude
```

無需切換分支、獨立文件系統、真正的並行。

### 3. Gemini CLI 作為被封鎖站點的後備

當 WebFetch 失敗時（Reddit、付費牆站點）：

```markdown
# .claude/commands/fetch-blocked.md
使用 Gemini CLI 從以下地址獲取內容：$URL
返回完整文本內容。
```

### 4. 自主任務的寫-測試循環

```markdown
# CLAUDE.md
實現功能時：
1. 先寫失敗的測試
2. 實現直到測試通過
3. 運行 `bun test` 驗證
4. 只有測試通過才提交
```

測試成為驗證機制，使自主迭代成為可能。

### 5. 三層沙盒（Trail of Bits）

```
第一層：/sandbox 命令（OS 級隔離）
第二層：settings.json 中的權限拒絕規則
第三層：Devcontainers 實現完全隔離
```

```json
{
  "deny": [
    "Read(.env*)",
    "Read(~/.ssh/*)",
    "Read(~/.aws/*)",
    "Bash(rm -rf *)"
  ]
}
```

### 6. 必備鍵盤快捷鍵

| 動作 | 按鍵 |
|------|------|
| 換行（不提交） | Shift+Enter |
| 停止生成 | Escape |
| 訪問上一條消息 | Escape × 2 |
| 切換擴展思考 | Alt+T |
| 粘貼圖片 | Ctrl+V（不是 Cmd+V） |

### 7. 會話管理

```bash
# 快速訪問的別名
alias c='claude'
alias ch='claude --chrome'
alias cr='claude -c'  # 繼續最近的會話

# 分叉對話
/fork  # 創建當前會話的分支
```

### 8. MCP 服務器必備配置

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@context7/mcp-server"]
    },
    "exa": {
      "command": "npx",
      "args": ["-y", "@anthropic/exa-mcp-server"]
    }
  }
}
```

- **Context7：** 任何庫的按需文檔
- **Exa：** 增強的網頁/代碼搜索

### 9. 語音輸入工作流

使用本地轉錄（SuperWhisper、MacWhisper）：
- 比打字更快地處理複雜指令
- 自然說話，粘貼到 Claude
- 特別適合架構討論

### 10. 定期審計已批准的命令

```bash
# 檢查已自動批准的內容
cat ~/.claude/settings.json | jq '.allowedTools'

# 移除隨時間積累的危險模式
claude /allowed-tools
```

權限會累積；定期審查防止安全漂移。

## 最新動態 (2026)

### Opus 4.6 與模型陣容更新

Anthropic 於 2026 年 2 月 5 日發布了 Claude Opus 4.6 和 Sonnet 4.6。Opus 4.6 具備 1M token 上下文視窗（測試版）、128K 輸出 token 限制（從 64K 翻倍）以及自適應思考功能，讓模型根據上下文線索自動調整擴展思考深度。Agent Teams（TeammateTool）隨 Opus 4.6 正式發布，通過 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 啟用。Sonnet 4.5 的 1M 上下文版本已退役；Sonnet 4.6 以相同的 1M 上下文視窗取代之。這些模型升級意味著 Claude Code 中的 Opus vs Sonnet 決策矩陣已經改變 -- Opus 4.6 憑藉擴展的上下文原生處理更大的代碼庫，而 Sonnet 4.6 以 Sonnet 的價格提供接近 Opus 的編碼質量。

### Skills 系統取代斜杠命令

自定義斜杠命令（`.claude/commands/`）已正式合併到 Skills 系統（`.claude/skills/`）。現有的命令文件仍然可用，但 Skills 現在是推薦的方式。Skills 相比普通命令增加了關鍵能力：用於控制調用行為的 YAML 前置資料（`disable-model-invocation`、`user-invocable`）、支持模板和腳本的附屬文件目錄、用於沙盒執行的 `allowed-tools` 限制、用於子代理隔離的 `context: fork`，以及通過 `!`command`` 語法在發送給 Claude 之前預處理 shell 輸出的動態上下文注入。Skills 還遵循 [Agent Skills](https://agentskills.io) 開放標準，實現跨工具可移植性。Monorepo 支持自動運作 -- 嵌套的 `.claude/skills/` 目錄中的 Skills 會根據正在編輯的文件自動被發現。

### 內建 Git Worktree 支持（`--worktree` 標誌）

Claude Code 現在有原生的 `--worktree`（`-w`）標誌，可以創建隔離的 git worktree 並在其中啟動會話。這消除了之前並行功能開發所需的手動 `git worktree add` 工作流程。子代理也支持 `"worktree"` 隔離，使單個會話內可以並行工作，每個子代理在自己的 worktree 中操作而不產生文件衝突。結合 `--tmux`，你可以啟動完全隔離的後台會話：`claude --worktree my-feature --tmux`。新的 `WorktreeCreate` 和 `WorktreeRemove` hook 事件（v2.1.50）允許在 worktree 創建或銷毀時進行自定義 VCS 設置自動化。

### Agent Teams 多代理編排

Agent Teams 讓你協調多個 Claude Code 實例，其中一個會話作為團隊領導，分配任務並綜合結果。與子代理（共享父代理的上下文視窗）不同，隊友各自擁有獨立的上下文視窗，可以直接相互通信。你也可以繞過領導直接與單個隊友互動。最佳使用場景：並行調查的研究、分別所有權的新模塊、競爭假設的調試，以及跨層協調（前端/後端/測試）。權衡是顯著的 token 開銷 -- Agent Teams 最適合隊友可以在不同文件上獨立操作的場景。

### IDE 整合成熟

VS Code 擴展現在提供原生圖形界面，包含計劃預覽、自動接受編輯、帶行範圍的 @-mention 文件、對話歷史標籤頁和多個並行對話。JetBrains 插件（IntelliJ、PyCharm、WebStorm）在 IDE 終端內運行 CLI，更改通過 IDE 的 diff 查看器呈現。VS Code 整合稍微更精緻（更快的上下文載入），但兩個平台都能可靠工作。對於高級用戶，CLI 仍然是最靈活的選擇，功能交付更快。

### Hooks 系統擴展

Hooks 系統已從最初的 `PreToolUse`/`PostToolUse` 增長到 15 個生命週期事件。新增包括用於多代理工作流的 `TeammateIdle` 和 `TaskCompleted`、用於設置修改時安全審計的 `ConfigChange`，以及用於 worktree 生命週期管理的 `WorktreeCreate`/`WorktreeRemove`。`Stop` 和 `SubagentStop` hook 輸入新增了 `last_assistant_message` 字段，提供最終回應文本，無需解析轉錄文件。通過延遲 `SessionStart` hook 執行，啟動性能提升了約 500ms。`disableAllHooks` 設置現在正確遵守託管設置層級，防止非託管配置覆蓋組織策略 hooks。

### 性能和平台改進 (v2.1.37-v2.1.50)

2026 年 2 月的發布節奏中，僅 v2.1.47 就帶來了 60+ 修復。關鍵改進：Windows ARM64 原生二進制支持、`claude auth login/status/logout` CLI 子命令、使用 `/rename` 自動生成會話名稱、Agent Teams 和任務狀態管理的內存洩漏修復、改進的壓縮行為（壓縮後清除緩存、文件歷史快照限制），以及用於可配置多行輸入的 `chat:newline` 按鍵綁定操作。自動記憶（MEMORY.md）在 v2.1.32 中引入 -- Claude 現在會自行記錄項目慣例和用戶偏好的筆記，與 CLAUDE.md 分開。

## 關鍵洞察

非顯而易見的洞察：**上下文新鮮度優於上下文積累**。

直覺認為「更多上下文 = 更好的理解」。現實是：LLM 輸出質量隨上下文長度增加而下降。最佳實踐者積極使用 `/clear` 並撰寫交接文檔，將每次會話視為帶有精心策劃上下文的全新開始，而非連續對話。

這顛覆了心智模型：從「AI 助手記住一切」轉變為「AI 助手以完美的簡報文檔重新開始」。

## References

### 官方文檔
- [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Common Workflows - Claude Code Docs](https://code.claude.com/docs/en/common-workflows)
- [Optimize Your Terminal Setup - Claude Code Docs](https://code.claude.com/docs/en/terminal-config)
- [Orchestrate Teams of Claude Code Sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Using CLAUDE.MD Files: Customizing Claude Code for Your Codebase | Claude](https://claude.com/blog/using-claude-md-files)
- [Extend Claude with Skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Hooks Reference - Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Claude Code Releases - GitHub](https://github.com/anthropics/claude-code/releases)
- [What's New in Claude 4.6 - Claude API Docs](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-6)
- [Introducing Claude Opus 4.6 - Anthropic](https://www.anthropic.com/news/claude-opus-4-6)

### 綜合指南
- [How I Use Claude Code (+ My Best Tips) - Builder.io](https://www.builder.io/blog/claude-code)
- [32 Claude Code Tips: From Basics to Advanced - Agentic Coding Substack](https://agenticcoding.substack.com/p/32-claude-code-tips-from-basics-to)
- [Claude Code CLI Cheatsheet: Config, Commands, Prompts, + Best Practices - Shipyard](https://shipyard.build/blog/claude-code-cheat-sheet/)
- [Claude Code Complete Guide 2026: From Basics to Advanced MCP, Agents & Git Workflows](https://www.jitendrazaa.com/blog/ai/claude-code-complete-guide-2026-from-basics-to-advanced-mcp-2/)
- [Cooking with Claude Code: The Complete Guide | Sid Bharath](https://www.siddharthbharath.com/claude-code-the-complete-guide/)
- [50 Claude Code Tips & Tricks for Daily Coding in 2026 - Geeky Gadgets](https://www.geeky-gadgets.com/claude-code-tips-2/)
- [Claude Code Best Practices: 15 Tips from 6 Projects (2026) | aiorg.dev](https://aiorg.dev/blog/claude-code-best-practices)

### GitHub 資源
- [ykdojo/claude-code-tips: 45 Tips for Getting the Most Out of Claude Code](https://github.com/ykdojo/claude-code-tips)
- [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
- [hesreallyhim/awesome-claude-code: Curated List of Skills, Hooks, Slash-Commands](https://github.com/hesreallyhim/awesome-claude-code)
- [trailofbits/claude-code-config: Opinionated Defaults and Workflows](https://github.com/trailofbits/claude-code-config)
- [disler/claude-code-hooks-mastery: Master Claude Code Hooks](https://github.com/disler/claude-code-hooks-mastery)

### CLAUDE.md 配置
- [Writing a Good CLAUDE.md | HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [Creating the Perfect CLAUDE.md for Claude Code - Dometrain](https://dometrain.com/blog/creating-the-perfect-claudemd-for-claude-code/)
- [ClaudeLog - Claude Code Docs, Guides, Tutorials & Best Practices](https://claudelog.com/configuration/)
- [How to Write a Good CLAUDE.md File - Builder.io](https://www.builder.io/blog/claude-md-guide)
- [Claude Skills and CLAUDE.md: A Practical 2026 Guide for Teams - Gend](https://www.gend.co/blog/claude-skills-claude-md-guide)

### 終端整合
- [Claude Code + tmux: The Ultimate Terminal Workflow for AI Development](https://www.blle.co/blog/claude-code-tmux-beautiful-terminal)
- [How to Use Claude Code CLI and Tmux for Continuous Workflows - Geeky Gadgets](https://www.geeky-gadgets.com/making-claude-code-work-247-using-tmux/)
- [Build a 24/7 Autonomous Coding Assistant with Tmux & Claude Code - Geeky Gadgets](https://www.geeky-gadgets.com/autonomous-coding-assistant-setup/)
- [How tmux Automation Made Claude Code Development Much More Efficient - Qiita](https://qiita.com/vibecoding/items/c04741332b6617781684)
- [Combining tmux and Claude to Build an Automated AI Agent System - Scuti Ai](https://scuti.asia/combining-tmux-and-claude-to-build-an-automated-ai-agent-system-for-mac-linux/)
- [Multi-agent Claude Code Workflow Using tmux for Parallel Sessions - GitHub Gist](https://gist.github.com/andynu/13e362f7a5e69a9f083e7bca9f83f60a)
- [Notification System for Tmux and Claude Code - Alexandre Quemy](https://quemy.info/2025-08-04-notification-system-tmux-claude.html)

### Ghostty 終端
- [How to Integrate Claude Code with Neovim Using Ghostty Terminal Panes | Daniel Miessler](https://danielmiessler.com/blog/claude-code-neovim-ghostty-integration)
- [The State of Vibe Coding: Agentic Software Development with Ghostty, Git Worktree & Claude Code | Medium](https://medium.com/@takafumi.endo/the-state-of-vibe-coding-agentic-software-development-with-ghostty-git-worktree-claude-code-18f4d56b8e01)
- [Why Ghostty Terminal Is My Fastest Claude Code Workflow | Engr Mejba Ahmed](https://www.mejba.me/blog/ghostty-terminal-claude-code-workflow)
- [Ghostty Terminal Configuration | DeepWiki](https://deepwiki.com/awwsillylife/ghostty-claude-code-setup/4.1-ghostty-terminal-configuration)
- [Claude Code Session Fork for Ghostty - GitHub Gist](https://gist.github.com/yottahmd/8e6d0a4213be6a559dfe3dcdd350ce09)
- [Add Shift-Enter Support for Ghostty via `/terminal-setup` - GitHub Issue](https://github.com/anthropics/claude-code/issues/1282)

### 生產力與工作流
- [Claude Code Tips: 10 Real Productivity Workflows for 2026 - F22 Labs](https://www.f22labs.com/blogs/10-claude-code-productivity-tips-for-every-developer/)
- [The Claude Code Playbook: 5 Tips Worth $1000s in Productivity | White Prompt Blog](https://blog.whiteprompt.com/the-claude-code-playbook-5-tips-worth-1000s-in-productivity-22489d67dd89)
- [My 7 Essential Claude Code Best Practices for Production-Ready AI in 2025 - Eesel](https://www.eesel.ai/blog/claude-code-best-practices)
- [The Ultimate Guide to Claude Code: Production Prompts, Power Tricks, and Workflow Recipes | Medium](https://medium.com/@tonimaxx/the-ultimate-guide-to-claude-code-production-prompts-power-tricks-and-workflow-recipes-42af90ca3b4a)
- [Two Simple Tricks That Will Dramatically Improve Your Productivity with Claude | Medium](https://julsimon.medium.com/two-simple-tricks-that-will-dramatically-improve-your-productivity-with-claude-db90ce784931)
- [Mastering Claude Code: Essential Tips for Maximum Productivity - Tembo](https://www.tembo.io/blog/mastering-claude-code-tips)
- [How I Use Claude Code to Maximize Productivity | Medium](https://medium.com/@shivang.tripathii/how-i-use-claude-code-to-maximize-productivity-c853104804d6)
- [How I Use Every Claude Code Feature - Shrivu Shankar](https://blog.sshh.io/p/how-i-use-every-claude-code-feature)

### Agent Teams 與多代理
- [How to Set Up and Use Claude Code Agent Teams | Medium](https://darasoba.medium.com/how-to-set-up-and-use-claude-code-agent-teams-and-actually-get-great-results-9a34f8648f6d)
- [How Claude Code Agent Teams Changed Everything About AI Coding | How Do I Use AI](https://www.howdoiuseai.com/blog/2026-02-10-how-claude-code-agent-teams-changed-everything-abo)
- [Claude Code Agent Teams: The Complete Guide 2026 - ClaudeFast](https://claudefa.st/blog/guide/agents/agent-teams)
- [Claude Code's Hidden Multi-Agent System - Paddo.dev](https://paddo.dev/blog/claude-code-hidden-swarm/)
- [Claude Code Swarm Orchestration Skill - GitHub Gist](https://gist.github.com/kieranklaassen/4f2aba89594a4aea4ad64d753984b2ea)
- [Multi-agent Orchestration for Claude Code in 2026 - Shipyard](https://shipyard.build/blog/claude-code-multi-agent/)
- [Claude Code Swarms - Addy Osmani](https://addyosmani.com/blog/claude-code-agent-teams/)

### IDE 整合
- [Use Claude Code in VS Code - Claude Code Docs](https://code.claude.com/docs/en/vs-code)
- [Claude Code Plugin for JetBrains IDEs - JetBrains Marketplace](https://plugins.jetbrains.com/plugin/27310-claude-code-beta-)
- [Claude Code IDE Integrations for JetBrains and VS Code | Medium](https://medium.com/vibecodingpub/claude-code-ide-integrations-for-jetbrains-ides-and-vs-code-71023d27b86d)

### Worktree 與隔離
- [Claude Code Worktree: Complete Guide for Developers - SupaTest](https://supatest.ai/blog/claude-code-worktree-the-complete-developer-guide)
- [Claude Code Multiple Agent Systems: Complete 2026 Guide - Eesel](https://www.eesel.ai/blog/claude-code-multiple-agent-systems-complete-2026-guide)

### 模型更新
- [Anthropic Releases Opus 4.6 with New 'Agent Teams' | TechCrunch](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams/)
- [Claude Opus 4.6: What Actually Changed and Why It Matters | Medium](https://medium.com/data-science-collective/claude-opus-4-6-what-actually-changed-and-why-it-matters-1c81baeea0c9)
- [Claude Sonnet 4.6 Promises Opus-Level Coding at Sonnet Pricing - The New Stack](https://thenewstack.io/claude-sonnet-46-launch/)

### 教程
- [Claude Code Tutorial for Beginners - Complete 2026 Guide to AI Coding - codewithmukesh](https://codewithmukesh.com/blog/claude-code-for-beginners/)
- [Claude Code CLI Best Practices Checklist - Engineering Notes](https://notes.muthu.co/2026/02/claude-code-cli-best-practices-checklist/)
- [Mastering the Vibe: Claude Code Best Practices That Actually Work | Medium](https://dinanjana.medium.com/mastering-the-vibe-claude-code-best-practices-that-actually-work-823371daf64c)
- [Claude Code Hooks: Complete Guide with 20+ Ready-to-Use Examples (2026) | aiorg.dev](https://aiorg.dev/blog/claude-code-hooks)
