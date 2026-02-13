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

## 關鍵洞察

非顯而易見的洞察：**上下文新鮮度優於上下文積累**。

直覺認為「更多上下文 = 更好的理解」。現實是：LLM 輸出質量隨上下文長度增加而下降。最佳實踐者積極使用 `/clear` 並撰寫交接文檔，將每次會話視為帶有精心策劃上下文的全新開始，而非連續對話。

這顛覆了心智模型：從「AI 助手記住一切」轉變為「AI 助手以完美的簡報文檔重新開始」。
