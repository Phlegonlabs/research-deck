# Claude Code Agent Team：從內部解析多智能體協調系統

## 他們構建了什麼

Claude Code Agent Team 是 Anthropic 的原生多智能體協調系統，隨 Opus 4.6 於 2026 年 2 月發布。它將 Claude Code 從單一 AI 編碼助手轉變為**團隊協調者** — 一個主導會話生成多個隊友實例，並行工作、通過直接消息通信、通過共享任務列表自我協調。

這不是一個包裝器或框架。它直接內建在 Claude Code 二進制文件中，名為 `TeammateTool` — 13 個操作，帶有定義好的 schema、目錄結構和環境變量。最初由研究人員通過對二進制文件運行 `strings` 命令發現（功能標誌關閉狀態），後來作為實驗性功能正式發布。

## 為什麼選擇這種架構

### 核心問題：上下文窗口退化

LLM 在上下文擴展時表現會變差。添加無關信息（在調試後端時加入前端代碼）會降低推理質量。傳統的單智能體方法在複雜項目上會碰壁 — 你無法在不影響質量的情況下，將全棧功能的所有上下文塞進一個窗口。

### 為什麼不直接用子智能體？

Claude Code 已經有子智能體（通過 `Task` 工具）— 短期存活的工作者，執行完畢後返回結果。但子智能體有一個關鍵限制：**它們只能向調用者匯報**。沒有對等通信。沒有共享狀態。無法互相質疑發現。

Agent Team 通過讓隊友成為**一等對等體**來解決這個問題：

| 方面 | 子智能體 | Agent Team |
|------|---------|------------|
| 上下文 | 獨立窗口，結果返回調用者 | 獨立窗口，完全獨立 |
| 通信 | 只能向主智能體匯報 | 隊友之間直接消息通信 |
| 協調 | 主智能體管理所有工作 | 共享任務列表，自我協調 |
| 持久性 | 短期存活，任務結束即死亡 | 持久存在，直到關閉 |
| 成本 | 較低（結果被摘要） | 較高（每個都是獨立的 Claude 實例） |

### 為什麼不用外部框架？

與 LangGraph、CrewAI 或 OpenAI Swarm 相比：

- **零設置摩擦**：一個環境變量，不需要 pip install，不需要配置文件
- **原生工具訪問**：隊友繼承所有 Claude Code 能力（Bash、Read、Write、Edit、Glob、Grep、MCP 服務器、技能）
- **項目上下文繼承**：隊友自動加載 CLAUDE.md、MCP 服務器和技能
- **基於文件的協調**：沒有外部數據庫、消息隊列或 API — 只有磁盤上的 JSON 文件

代價：你被鎖定在 Claude 作為 LLM。不能混合模型，不能用開源替代方案。

## 工作原理：完整架構

### 組件映射

```
~/.claude/
├── teams/{team-name}/
│   ├── config.json          # 團隊成員、ID、類型
│   └── inboxes/
│       ├── team-lead.json   # 領導者的消息收件箱
│       ├── worker-1.json    # 隊友收件箱
│       └── worker-2.json
└── tasks/{team-name}/
    ├── 1.json               # 帶狀態、所有者、依賴關係的任務
    ├── 2.json
    └── 3.json
```

### 核心組件

- **Team Lead** — 主 Claude Code 會話。創建團隊、生成隊友、協調工作
- **Teammates** — 獨立的 Claude Code 實例。各有自己的上下文窗口和完整工具訪問權
- **Task List** — 共享工作項，帶狀態跟蹤、所有權和依賴管理
- **Mailbox** — 基於文件的消息系統，用於智能體間通信

### TeammateTool：13 個操作

**團隊生命週期：**
- `spawnTeam` / `discoverTeams` / `cleanup`
- `requestJoin` / `approveJoin` / `rejectJoin`

**通信：**
- `write` — 向一個隊友發送直接消息
- `broadcast` — 向所有隊友發送消息（昂貴：N 個隊友 = N 條消息）

**計劃控制：**
- `approvePlan` / `rejectPlan`

**關閉：**
- `requestShutdown` / `approveShutdown` / `rejectShutdown`

### 消息類型

- **`message`** — 智能體間直接文本消息
- **`broadcast`** — 同一消息發送給所有隊友
- **`shutdown_request` / `shutdown_response`** — 優雅的生命週期管理
- **`idle_notification`** — 隊友停止時自動發送（正常行為，非錯誤）
- **`task_completed`** — 任務完成信號
- **`plan_approval_request`** — 隊友發送計劃供領導審查
- **`join_request`** — 智能體請求加入團隊

### 任務依賴系統

```
TaskCreate → TaskUpdate(addBlockedBy) → 完成時自動解鎖
```

任務使用基於文件的鎖定來防止多個隊友同時嘗試認領時的競爭條件。當阻塞任務完成時，所有依賴任務自動解鎖。

### 環境變量（為隊友自動設置）

```
CLAUDE_CODE_TEAM_NAME="my-project"
CLAUDE_CODE_AGENT_ID="worker-1@my-project"
CLAUDE_CODE_AGENT_NAME="worker-1"
CLAUDE_CODE_AGENT_TYPE="Explore"
CLAUDE_CODE_PLAN_MODE_REQUIRED="false"
CLAUDE_CODE_PARENT_SESSION_ID="session-xyz"
```

## 編排模式

### 模式 1：並行專家

多個智能體從不同角度同時分析同一目標。

```
創建 Agent Team 審查 PR #142：
- 安全審查員：token 處理、輸入驗證
- 性能審查員：查詢優化、內存使用
- 測試審查員：覆蓋率缺口、邊界情況
```

**為什麼有效**：單一審查員會錨定在一種問題類型上。專家們應用正交過濾器，互不重疊。

### 模式 2：競爭假設

多個智能體調查不同理論並積極辯論。

```
用戶報告應用在一條消息後退出。生成 5 個調查員：
- 每個測試不同的假設
- 他們互相交流以推翻競爭理論
- 在辯論中存活的理論最可能是正確的
```

**為什麼有效**：順序調查受錨定偏差影響。並行 + 對抗結構消除了這個問題。

### 模式 3：流水線（帶依賴的順序執行）

```
任務 1：研究緩存模式 →
任務 2：設計 API（被 1 阻塞） →
任務 3：實現（被 2 阻塞） →
任務 4：編寫測試（被 3 阻塞）
```

任務在依賴完成時自動推進。適合必須順序執行但受益於清晰交接點的工作。

### 模式 4：自組織蜂群

創建獨立任務池。生成工作者：
1. 檢查 TaskList 中的待處理任務
2. 認領下一個可用任務（文件鎖定防止競爭）
3. 完成工作
4. 重複直到池空

工作者自然負載均衡。不需要中央分配。

### 模式 5：計劃-審批-執行

```
生成架構師隊友，mode: "plan"
→ 架構師在只讀模式下設計
→ 領導收到 plan_approval_request
→ 領導批准/拒絕並附帶反饋
→ 批准後，架構師退出計劃模式並實施
```

**控制旋鈕**："只批准包含測試覆蓋的計劃"或"拒絕修改數據庫 schema 的計劃"。

### 模式 6：跨層協調

```
任務 1：前端智能體 → React 組件
任務 2：後端智能體 → API 端點
任務 3：測試智能體 → 集成測試（被 1 + 2 阻塞）
```

每個智能體擁有不同的文件。測試智能體只在兩層都完成後運行。

## 顯示與交互模式

| 模式 | 工作方式 | 最適合 |
|------|---------|--------|
| **In-process** | 所有隊友在主終端中。Shift+Up/Down 導航 | 默認，任何終端 |
| **Split panes** | 每個隊友獲得自己的 tmux/iTerm2 面板 | 可見性，直接交互 |
| **Delegate** | 領導被限制為只能使用協調工具（Shift+Tab） | 純粹的編排 |

## 質量門禁：Hooks

兩個 Hook 對隊友執行規則：

- **`TeammateIdle`**：當隊友即將閒置時運行。退出碼 2 發送反饋並保持其繼續工作。
- **`TaskCompleted`**：當任務即將被標記完成時運行。退出碼 2 阻止完成並附帶反饋。

## 權衡取捨

### 你得到的
- 可並行化任務上 5-10 倍吞吐量
- 通過領域專業化獲得更好的推理
- 每個智能體獨立的質量檢查點
- 優雅降級（一個智能體失敗不會殺死整個團隊）
- 自然的階段邊界

### 你犧牲的
- **Token 成本**：隨隊友數量線性增長。每個隊友都是完整的 Claude 實例
- **協調開銷**：對順序任務或同文件編輯不值得
- **無會話恢復**：in-process 隊友不能在 `/resume` 或 `/rewind` 後存活
- **每會話單團隊**：不能嵌套團隊或讓隊友生成子團隊
- **固定領導**：不能將隊友提升為領導
- **平台限制**：分割面板需要 tmux/iTerm2（不支持 VS Code 終端、Windows Terminal、Ghostty）
- **任務狀態延遲**：隊友有時未能標記任務完成，阻塞依賴任務

### 何時不應使用

- 每一步之間都有依賴的順序工作
- 同文件編輯（會互相覆蓋）
- 協調開銷大於收益的簡單任務
- 成本敏感場景（常規任務用單會話更划算）

## 替代方案對比

| 框架 | 編排方式 | 通信 | 狀態 | 設置 |
|------|---------|------|------|------|
| **Claude Code Agent Team** | 內建，基於文件 | 直接消息 + 郵箱 | 磁盤上的 JSON | 1 個環境變量 |
| **LangGraph** | 基於圖的 DAG | 通過圖邊 | 有檢查點的狀態 | pip install + 代碼 |
| **CrewAI** | 基於角色的團隊 | 通過框架 | 內存中 | pip install + 配置 |
| **OpenAI Swarm** | 基於例程 | 函數文檔字符串 | 無正式狀態 | pip install + 代碼 |
| **claude-flow** | 外部編排器 | MCP 協議 | Redis/SQLite | npm install + 配置 |

**LangGraph** 在控制性、合規性和生產級狀態管理上勝出。**CrewAI** 在原型速度上勝出。**Claude Code Agent Team** 在與現有 Claude Code 工作流的零摩擦集成上勝出。

## 最新動態 (2026)

### 隨 Opus 4.6 正式發布（2026 年 2 月）

Agent Teams 作為實驗性功能隨 Claude Opus 4.6 於 2026 年 2 月 5 日發布。通過在 settings.json 或環境變量中設置 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 即可啟用。該功能在早期 Claude Code 二進制文件中以功能標誌關閉狀態存在（2026 年 1 月社區研究人員通過對二進制文件運行 `strings` 命令發現），後正式上線。Anthropic 注意到社區已經在獨立構建類似模式，使用自定義 Docker 編排腳本和 OpenClaw 等工具。

### C 編譯器壓力測試：16 個智能體、$20K、10 萬行代碼

Anthropic 研究員 Nicholas Carlini 發布了一個里程碑式的案例研究：[16 個並行 Opus 4.6 智能體](https://www.anthropic.com/engineering/building-c-compiler)自主構建了一個基於 Rust 的 C 編譯器，能夠在 x86、ARM 和 RISC-V 架構上編譯 Linux 6.9。關鍵指標：近 2,000 個 Claude Code 會話、20 億輸入 token、1.4 億輸出 token、$20,000 API 成本、約 2 週內產出 10 萬行 Rust 代碼。每個智能體在自己的 Docker 容器中運行，通過在 `current_tasks/` 中創建鎖文件認領任務，推送到共享 git 上游。編譯器通過 99% 的 GCC torture 測試，能夠構建 QEMU、FFmpeg、SQLite、PostgreSQL 和 Redis。顯著限制：當所有智能體同時遇到相同 bug 時，擴展出現瓶頸，導致重複修復和合併衝突 — 解決方案是使用 GCC 作為「已知正確的預言機」來重新劃分工作。

### Claude Code SDK 更名為 Claude Agent SDK

Claude Code SDK 正式更名為 **Claude Agent SDK**，以反映其超越編碼的更廣泛範圍。提供 Python（`pip install claude-agent-sdk`）和 TypeScript（`npm install @anthropic-ai/claude-agent-sdk`）版本，提供與 Claude Code 相同的工具、智能體循環和上下文管理作為可編程庫。2026 年初的關鍵新增功能包括：`ThinkingConfig` 用於細粒度控制擴展思考（努力級別：low/medium/high/max），通過 `@tool` 裝飾器的 MCP 工具註解，以及修復了大型智能體定義因 CLI 參數大小限制而靜默失敗的問題。SDK 支持通過 `Task` 工具生成子智能體、自定義 Hook（`PreToolUse`、`PostToolUse`、`Stop` 等）以及多輪工作流的會話恢復。

### VS Code 成為多智能體指揮中心

隨著 [VS Code v1.109](https://code.visualstudio.com/blogs/2026/02/05/multi-agent-development)（2026 年 1 月），Microsoft 將 VS Code 打造為「多智能體開發的家園」。Claude 智能體現在通過官方 Claude Agent harness 原生運行在 VS Code 中 — 與獨立 Claude Code 相同的提示詞、工具和架構。開發者可以從一個界面同時運行 Claude、Codex 和 GitHub Copilot 智能體，可選擇本地（交互式）、後台（異步）或雲端（遠程自主）部署。2 月更新新增了子智能體渲染 — 當 Claude 在流式傳輸期間生成子智能體時，你可以內聯查看其工具調用和進度。Agent Skills（Anthropic 的開放標準）現在在 VS Code 中已正式可用。

### 社區工具生態爆發

Agent Team 模式催生了一系列開源協調工具：
- **[claude-swarm](https://blog.nikosbaxevanis.com/2026/02/08/agent-teams-and-claude-swarm/)**：用於在 Docker 容器中運行多個 Claude Code 會話的可重用框架，通過 git 協調，使用裸倉庫以只讀方式掛載為卷。
- **[ccswarm](https://github.com/nwiizo/ccswarm)**：工作流自動化框架，提供任務委派基礎設施、模板腳手架和 git worktree 隔離。
- **[worktree-cli](https://github.com/agenttools/worktree)**：管理 git worktree 的 CLI 工具，集成 GitHub Issues 和 Claude Code Hook。
- **[Clash](https://github.com/clash-sh/clash)**：跨 git worktree 的合併衝突管理器，用於並行 AI 智能體工作流。

尚未解決的缺口：目前沒有工具能乾淨地將 worktree 代碼隔離與完整環境隔離（Dev Containers）結合起來。多個 GitHub issue 顯示 worktree 和 devcontainer 無法良好配合 — `.git` 文件格式會破壞容器內的 git 操作。

### 企業採用激增

自 2026 年初以來，Claude Code 的企業訂閱增長了四倍，企業使用現在占 Claude Code 收入的一半以上。Agent Teams 功能被認為是主要推動力 — 團隊報告在可並行化的重構、遷移和審查任務上獲得了 5-10 倍的吞吐量提升。

### Microsoft Agent Framework 集成

Microsoft 的 Semantic Kernel 團隊宣布了 [Microsoft Agent Framework 與 Claude Agent SDK 的集成](https://devblogs.microsoft.com/semantic-kernel/build-ai-agents-with-claude-agent-sdk-and-microsoft-agent-framework/)，結合了 Agent Framework 一致的智能體抽象與 Claude 的文件編輯、代碼執行、函數調用、流式傳輸、多輪對話和 MCP 服務器集成。這將 Claude 驅動的智能體開放給企業級 .NET 和 Java 生態系統。

## 可以偷學的模式

### 1. 基於文件的協調優於數據庫

團隊使用磁盤上的 JSON 文件進行配置、任務和消息。無外部依賴。離線可用。可被 Git 跟蹤。這對本地優先的智能體系統來說非常優雅。

### 2. 閒置 ≠ 死亡

隊友在每次輪次後都會閒置 — 這是正常的，不是錯誤。發送消息就能喚醒它們。這種「事件驅動睡眠」模式防止了忙等待和 token 浪費。

### 3. 任務依賴自動解鎖

不需要手動流水線編排，預先聲明依賴關係，讓系統處理推進。被阻塞的任務在依賴完成時自動解鎖。

### 4. 計劃-審批門禁

強制智能體在只讀模式下先做計劃再實施。領導審查並批准。這在浪費 token 實施之前就能發現糟糕的方案。

### 5. 對抗性調試

生成具有競爭假設的多個智能體。讓它們積極嘗試推翻彼此。存活的理論最可能正確。這是系統中最被低估的模式。

### 6. 文件所有權邊界

分配不同的文件給每個隊友。不允許共享文件編輯。這完全消除了合併衝突 — 與微服務團隊邊界相同的原則。

### 7. 基於 Hook 的質量門禁

`TeammateIdle` 和 `TaskCompleted` Hook 讓你在不修改智能體提示詞的情況下注入驗證。退出碼 2 = "還沒完成，繼續工作"。這是一個乾淨的關注點分離。

### 8. 通過 CLAUDE.md 的上下文繼承

所有隊友自動加載 CLAUDE.md。把團隊範圍的約定、約束和質量標準放在那裡。一個文件控制 N 個智能體的行為。

## References

### 原始來源
- [@YukerX Twitter Thread — Claude Code Agent Team 入門教程](https://x.com/YukerX/status/2019977867061522525)

### 官方文檔
- [Orchestrate teams of Claude Code sessions — Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Introducing Claude Opus 4.6 — Anthropic](https://www.anthropic.com/news/claude-opus-4-6)
- [Common workflows — Claude Code Docs](https://code.claude.com/docs/en/common-workflows)
- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Agent SDK overview — Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Claude Agent SDK Python — GitHub](https://github.com/anthropics/claude-agent-sdk-python)
- [Claude Agent SDK Demos — GitHub](https://github.com/anthropics/claude-agent-sdk-demos)

### 深度技術分析
- [Claude Code's Hidden Multi-Agent System — paddo.dev](https://paddo.dev/blog/claude-code-hidden-swarm/)
- [Claude Code Swarm Orchestration Skill — Complete guide (GitHub Gist)](https://gist.github.com/kieranklaassen/4f2aba89594a4aea4ad64d753984b2ea)
- [Claude Code Swarms — Addy Osmani](https://addyosmani.com/blog/claude-code-agent-teams/)
- [Building a C compiler with a team of parallel Claudes — Anthropic Engineering](https://www.anthropic.com/engineering/building-c-compiler)
- [Claude Code Agent Teams: How They Work Under the Hood — Claude Code Camp](https://www.claudecodecamp.com/p/claude-code-agent-teams-how-they-work-under-the-hood)

### 指南與教程
- [Claude Code Agent Teams: Parallel AI Agents Setup Guide — Marco Patzelt](https://www.marc0.dev/en/blog/claude-code-agent-teams-multiple-ai-agents-working-in-parallel-setup-guide-1770317684454)
- [Claude Code Swarms Guide: How to Build Native Multi-Agent Teams — Tech On Play](https://techonplay.com/claude-code-swarms/)
- [Claude Swarm Mode Complete Guide — Apiyi.com](https://help.apiyi.com/en/claude-code-swarm-mode-multi-agent-guide-en.html)
- [Claude Code Agent Teams: Multi-Claude Orchestration — claudefa.st](https://claudefa.st/blog/guide/agents/agent-teams)
- [What Is the Claude Code Swarm Feature? — Cyrus](https://www.atcyrus.com/stories/what-is-claude-code-swarm-feature)
- [Claude Code Agent Teams: Run Parallel AI Agents on Your Codebase — SitePoint](https://www.sitepoint.com/anthropic-claude-code-agent-teams/)
- [How to Set Up and Use Claude Code Agent Teams — Dára Sobaloju (Medium)](https://darasoba.medium.com/how-to-set-up-and-use-claude-code-agent-teams-and-actually-get-great-results-9a34f8648f6d)

### 發現與探索
- [I Discovered This Claude Code Agent Swarm Mode — Joe Njenga (Medium)](https://medium.com/@joe.njenga/i-discovered-this-claude-code-agent-swarm-mode-you-dont-know-exists-bf36e3898ad1)
- [I Tried (New) Claude Code Agent Teams (And Discovered New Way to Swarm) — Joe Njenga (Medium)](https://medium.com/@joe.njenga/i-tried-new-claude-code-agent-teams-and-discovered-new-way-to-swarm-28a6cd72adb8)
- [Anthropic releases Opus 4.6 with new 'agent teams' — TechCrunch](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams/)
- [Claude Code's new hidden feature: Swarms — Hacker News](https://news.ycombinator.com/item?id=46743908)

### Git Worktree 與並行開發
- [Parallel Development with Claude Code and Git Worktrees — Yee Fei (Medium)](https://medium.com/@ooi_yee_fei/parallel-ai-development-with-git-worktrees-f2524afc3e33)
- [Mastering Git Worktrees with Claude Code — Dogukan Uraz Tuna (Medium)](https://medium.com/@dtunai/mastering-git-worktrees-with-claude-code-for-parallel-development-workflow-41dc91e645fe)
- [Clash — Manage merge conflicts across git worktrees for parallel AI agents](https://github.com/clash-sh/clash)
- [ccpm — Project management for Claude Code using GitHub Issues and Git worktrees](https://github.com/automazeio/ccpm)
- [ccswarm — Multi-agent orchestration with Git worktree isolation](https://github.com/nwiizo/ccswarm)
- [worktree — CLI tool for managing Git worktrees with Claude Code integration](https://github.com/agenttools/worktree)
- [Agent Teams and claude-swarm — Nikos Baxevanis](https://blog.nikosbaxevanis.com/2026/02/08/agent-teams-and-claude-swarm/)

### 替代框架（對比）
- [LangGraph vs CrewAI vs AutoGen: Top 10 AI Agent Frameworks](https://o-mega.ai/articles/langgraph-vs-crewai-vs-autogen-top-10-agent-frameworks-2026)
- [claude-flow — Agent orchestration platform for Claude](https://github.com/ruvnet/claude-flow)
- [Support for Claude Code Agent Teams (TeammateTool, SendMessage, TaskList) — superpowers Issue #429](https://github.com/obra/superpowers/issues/429)

### IDE 與平台集成
- [Your Home for Multi-Agent Development — VS Code Blog](https://code.visualstudio.com/blogs/2026/02/05/multi-agent-development)
- [VS Code becomes multi-agent command center for developers — The New Stack](https://thenewstack.io/vs-code-becomes-multi-agent-command-center-for-developers/)
- [Build AI Agents with Claude Agent SDK and Microsoft Agent Framework — Semantic Kernel](https://devblogs.microsoft.com/semantic-kernel/build-ai-agents-with-claude-agent-sdk-and-microsoft-agent-framework/)
- [Agent Teams with Claude Code and Claude Agent SDK — Isaac Kargar (Medium)](https://kargarisaac.medium.com/agent-teams-with-claude-code-and-claude-agent-sdk-e7de4e0cb03e)

### 最佳實踐
- [A Guide to Claude Code 2.0 and getting better at using coding agents — sankalp](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/)
- [Best practices for Claude Code subagents — PubNub](https://www.pubnub.com/blog/best-practices-for-claude-code-sub-agents/)
- [How Claude Code Agents and MCPs Work Better Together — Yee Fei (Medium)](https://medium.com/@ooi_yee_fei/how-claude-code-agents-and-mcps-work-better-together-5c8d515fcbbd)
