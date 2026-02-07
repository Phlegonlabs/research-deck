# 並行編碼代理：架構指南

## What

在雲端同時運行 N 個編碼代理 — 每個有自己的 git 倉庫、文件系統、包管理器和 shell — 來並行化軟件開發。2025→2026 的轉變：從 MacBook 上的本地 worktree 到由編排器管理的雲端沙箱。

```
2025: MacBook 跑 4 個 git worktree，各配一個 agent
2026: MacBook 派發任務，雲端跑 N 個沙箱並行執行
```

## Why — 為什麼是現在

Warp CEO Zach Lloyd 的論述（Sequoia podcast，2026 年 1 月）：

> "編碼問題幾年內就會被解決。瓶頸轉移到人類清晰表達意圖的能力。"

五股力量把 agent 推離筆電：

| 驅動力 | 問題 | 雲端怎麼解 |
|--------|------|-----------|
| **算力天花板** | 2 個 cargo build 就把 MacBook 跑滿 | 雲端水平擴展 |
| **沙箱隔離** | Agent 測試 UI 需要獨佔屏幕 | 每個有自己的容器 |
| **永不停機** | Agent 需要在筆電休眠時繼續工作 | 雲端不會睡覺 |
| **團隊可見性** | 無法追蹤團隊 agent 活動 | 儀表板、審計日誌 |
| **設置成本** | Docker + 機器配置很痛苦 | Agent 現在能自己配環境 |

進展是刻意的 — 不要跳步驟：

| 階段 | 在哪 | 人的角色 | 信任等級 |
|------|------|---------|---------|
| 交互式 | 本地 | 審查每個 diff | 低 |
| 本地並行 | 本地（worktree） | 管理 4-5 個 agent | 中 |
| 雲端環境 | 雲端沙箱 | 審查 PR、批准/拒絕 | 高 |
| 自主運行 | 雲端，事件觸發 | 設策略、處理升級 | 很高 |

## 架構：通用模式

所有平台 — Cloudflare、Warp、E2B、Daytona — 都收斂到相同的分層架構：

```
┌─────────────────────────────────────────────┐
│  觸發層 TRIGGER                              │
│  Slack / Linear / GitHub / Webhook / Cron    │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  編排層 ORCHESTRATE                          │
│  任務隊列 → 調度 → 狀態 → 密鑰注入           │
│  (Cloudflare Agents SDK / Warp / 自建)       │
└──────────────────┬──────────────────────────┘
                   │ 生成 N 個
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Sandbox │ │ Sandbox │ │ Sandbox │ │ Sandbox │
│ Agent 1 │ │ Agent 2 │ │ Agent 3 │ │ Agent 4 │
│ git+npm │ │ git+rsc │ │ git+py  │ │ git+go  │
└─────────┘ └─────────┘ └─────────┘ └─────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  觀測層 OBSERVE                              │
│  會話記錄 → 儀表板 → 審計日誌                 │
└─────────────────────────────────────────────┘
```

**鐵律：永遠不要在編排器裡跑 agent 邏輯。** 編排器負責路由、認證和協調。沙箱負責執行。這個分離不可妥協。

## 平台全景

### 第一梯隊：Agent 原生沙箱

這些平台專為 AI 編碼代理而建。

| 平台 | 冷啟動 | 隔離 | 殺手功能 | 弱點 |
|------|--------|------|---------|------|
| **Daytona** | 27-90ms | Docker（可選 Kata） | Fork/Snapshot | Docker 隔離較弱 |
| **E2B** | ~150ms | Firecracker microVM | 最佳 SDK（Python/TS） | 24h 會話上限 |
| **Fly.io Sprites** | 1-12s | Firecracker microVM | 100GB NVMe 持久化 | 最慢冷啟動 |
| **Blaxel** | 25ms | 輕量級 | 最快冷啟動記錄 | 最新，最少驗證 |

### 第二梯隊：通用計算 + 沙箱支持

| 平台 | 冷啟動 | 殺手功能 | 弱點 |
|------|--------|---------|------|
| **Cloudflare Containers** | 2-3 min | 全球邊緣 + 集成編排 | 資源上限（4 vCPU，20GB 磁盤） |
| **Modal** | 亞秒級 | GPU（H100），5 萬並發 | 僅 Python，gVisor 開銷 |
| **Northflank** | 不定 | BYOC，最便宜（$0.017/hr） | 非 agent 優先 |

### 第三梯隊：完整平台（編排 + 執行）

| 平台 | 方式 | 適合 |
|------|------|------|
| **Warp** | 終端原生 ADE + Namespace 沙箱 | 想要開箱即用 agent 編排的團隊 |
| **GitHub Codespaces** | 開發環境 + Copilot agents | GitHub 原生工作流 |
| **Cloudflare**（全棧） | Workers + Agents SDK + DO + Containers | 自定義編排 + 全球邊緣 |

## 對比矩陣

| 功能 | E2B | Daytona | Modal | Sprites | Cloudflare | Warp |
|------|-----|---------|-------|---------|------------|------|
| 冷啟動 | 150ms | 27-90ms | <1s | 1-12s | 2-3min | N/A |
| 最大會話 | 24h | 無限 | 24h | 無限 | 無限 | N/A |
| GPU | 無 | 無 | H100 | 有限 | 無 | 無 |
| Fork/Snapshot | 無 | 有 | 無 | Checkpoint | 無 | 無 |
| 編排層 | 無 | 無 | 無 | 無 | 有（Agents SDK） | 有 |
| 開源 | 是 | 是 | 否 | 否 | 部分 | 否 |
| 費用（1vCPU/2GB/hr） | $0.08 | $0.08 | $0.12 | $0.07 | 按 10ms 計費 | $20/月+額度 |

## 隔離技術

選擇你的權衡：

| 技術 | 隔離強度 | 開銷 | 冷啟動 | 使用者 |
|------|---------|------|--------|--------|
| **Firecracker**（microVM） | 硬件級 | ~5% | 100-200ms | E2B、Sprites、AWS Lambda |
| **Kata**（容器-VM 混合） | 硬件級 | 5-10% | 200-500ms | Northflank |
| **gVisor**（用戶空間內核） | 內核級 | 2-9x 系統調用 | 亞秒級 | Modal、Google |
| **Docker**（命名空間/cgroup） | 進程級 | 最小 | 27-90ms | Daytona |

**實用法則**：如果 agent 只跑你自己 LLM 生成的代碼（不是不受信任的用戶代碼），Docker 就夠了。比 Firecracker 快 2 倍的速度值得這個隔離上的取捨。

## 五種架構模式

### 模式 1：每任務一個臨時沙箱

```
編排器 → 生成 N 個沙箱 → 各跑一個任務 → 收集結果 → 銷毀
```

**何時用**：獨立、定義明確的任務（修 bug A、加功能 B、寫 C 的測試）。
**最佳平台**：E2B（快速啟動、乾淨 SDK），Modal（大規模擴展）。

### 模式 2：Fork 探索

```
Agent 運行 → 決策點 → fork 出 N 個分支 → 評估 → 保留贏家
```

**何時用**：不確定方案時 — 讓 agent 並行嘗試多種方案。跟遊戲 AI 的 MCTS 一個模式。
**最佳平台**：Daytona（原生 fork/snapshot 原語）。

### 模式 3：持久 Agent 工作站

```
Agent 獲得一台「電腦」→ 安裝工具一次 → 工作數天 → checkpoint/restore
```

**何時用**：長期運行的複雜任務，需要累積狀態。
**最佳平台**：Sprites.dev（NVMe 持久化，300ms checkpoint）。

### 模式 4：編排器即大腦（Cloudflare 棧）

```
Worker（API 網關）→ Agents SDK on Durable Objects（狀態 + WebSocket）→ N 個 Sandbox 容器
```

**何時用**：想要自定義編排邏輯、全球邊緣部署、按使用量計費。
**最佳平台**：Cloudflare（Workers + Agents SDK + Containers + Sandbox SDK）。

Cloudflare 官方有在 Sandbox 跑 Claude Code 的教程：
1. Worker 收到請求（倉庫 URL + 任務）
2. 創建 Sandbox，通過 `gitCheckout()` 克隆倉庫
3. Claude Code 在沙箱內執行
4. 返回執行日誌 + git diff

### 模式 5：事件觸發的環境代理（Warp 棧）

```
事件觸發（Slack/Linear/GitHub/cron）→ Warp 編排器 → Namespace 沙箱 → PR/消息
```

**何時用**：想要 agent 對系統事件自動反應，無需人工啟動。
**最佳平台**：Warp Ambient Agents（beta）。

Warp 的關鍵技術決策：
- **Sidecar Volume**：掛載 `/agent/` 卷（Warp CLI + git + CA 證書）到任意用戶 Docker 鏡像 — 零配置
- **共享緩存**：團隊所有沙箱共享代碼庫 embedding 和上下文索引 — agent 不用從頭構建上下文
- **範圍化憑證**：在沙箱創建時注入短期、用戶範圍的 GitHub token — 永遠不要嵌入鏡像

## 關鍵權衡

### 冷啟動 vs 隔離

根本矛盾。你不可能同時有 sub-100ms 啟動和硬件級隔離。

```
更快 ←──────────────────────────────────→ 更安全
Docker (27ms)  gVisor (<1s)  Firecracker (150ms)  Kata (200-500ms)
Daytona        Modal         E2B/Sprites           Northflank
```

### 臨時 vs 持久

24 小時會話上限（E2B、Modal）強制臨時架構 — agent 每次從零開始。無限會話（Sprites、Daytona）支持持久工作站。你的 agent 架構必須二選一。

### 集成 vs 可組合

| 方式 | 優點 | 缺點 |
|------|------|------|
| **集成**（Warp、Cloudflare 全棧） | 開箱即用，膠水代碼少 | 供應商鎖定 |
| **可組合**（E2B + 自建編排器） | 任意替換任何層 | 更多集成工作 |

Rivet Sandbox Agent SDK 是收斂信號 — 一個 API 可以在任何沙箱（Daytona、E2B、Docker）裡跑 Claude Code、Codex 或 OpenCode。沙箱層正在商品化。價值向上移到編排層。

### 資源天花板

| 如果你的 Agent 需要... | 避開 | 用這個 |
|------------------------|------|--------|
| >20GB 磁盤（大 monorepo） | Cloudflare | Codespaces (64GB)、Sprites (100GB) |
| GPU 推理 | E2B、Daytona、Cloudflare | Modal (H100) |
| >4 vCPU 每 agent | Cloudflare | Codespaces (32 vCPU) |
| 多天持久化 | E2B、Modal（24h 上限） | Sprites、Daytona |

## 決策框架

```
你需要自定義編排嗎？
├── 要 → 需要全球邊緣嗎？
│        ├── 要 → Cloudflare（Workers + Agents SDK + Containers）
│        └── 不 → E2B/Daytona + 自建編排器
└── 不 → 想要開箱即用嗎？
         ├── 要 → Warp Ambient Agents（beta）或 GitHub Codespaces
         └── 不 → 你的優先級是什麼？
                   ├── 最快冷啟動      → Daytona (27ms)
                   ├── 最佳 SDK 體驗   → E2B（一行代碼 API）
                   ├── GPU 算力        → Modal (H100)
                   ├── 持久工作空間    → Sprites (100GB NVMe)
                   ├── 大規模最便宜    → Northflank ($0.017/hr)
                   └── 企業自託管      → Coder / Northflank BYOC
```

## Steal：可直接複用的模式

### 1. 編排器 ≠ 執行器（全行業共識）
Workers 負責路由。Containers 負責執行。永遠不要混合這兩個關注點。編排器應該是輕量的（V8 isolate、Lambda、或簡單的隊列消費者）。執行器需要完整 Linux。

### 2. Sidecar Volume（來自 Warp）
把 agent 工具掛載為 sidecar 卷（`/agent/` 含 CLI、git、CA 證書）到任意用戶 Docker 鏡像。用戶提供標準鏡像，你注入你的運行時。解耦 agent 和環境，零配置接入，即時更新。

### 3. Fork-and-Snapshot（來自 Daytona）
探索性任務時，在決策點 fork 沙箱。並行跑 N 種方案。快照贏家，丟棄其餘。這是代碼版的 MCTS — 對不確定問題最強大的模式。

### 4. 跨運行共享緩存（來自 Warp）
在 agent 沙箱生命週期間持久化代碼庫 embedding、上下文索引和依賴緩存。每次冷啟動上下文的 agent 浪費幾分鐘。共享緩存卷消除這個問題。

### 5. 範圍化憑證注入（來自 Warp）
永遠不要在容器鏡像中嵌入憑證。在沙箱創建時注入短期、用戶範圍的 token。GitHub token 應該是臨時的，僅限觸發用戶的倉庫訪問。

### 6. 漸進式自主（來自 Warp）
從本地交互開始。畢業到本地並行。再到雲端後台。再到事件觸發自主。每一步在增加自主度前先建立信任。不要跳步驟 — 信任模型和技術一樣重要。

### 7. 摩擦點激活（來自 Warp）
不要用永遠在線的 agent 建議，而是檢測摩擦時刻（git 衝突、測試失敗、構建崩潰）然後提供一鍵「讓 agent 修」。比主動建議的採用率更高。

### 8. TOEO 架構（來自 Warp）
觸發 → 編排 → 執行 → 觀測。四層，乾淨分離。替換執行環境（雲端/本地/自託管）不用改觸發器或編排邏輯。替換觸發器（Slack/webhook/cron）不用動執行層。

## 成本分析

跑 4 個並行 agent，每天 8 小時，每月 20 天（每個：1 vCPU，2GB RAM）：

| 平台 | 月費 | 備註 |
|------|------|------|
| Northflank | ~$11 | 最便宜，有 BYOC |
| Sprites | ~$45 | + 存儲費用 |
| E2B | ~$51 | 或 $150/月 Pro 包月 |
| Daytona | ~$51 | 類似 E2B |
| Cloudflare | ~$60 | 按 10ms 活躍時間計費 |
| Modal | ~$77 | + GPU 另計 |
| Codespaces | ~$115 | 最貴 |
| Warp | ~$20/月 + 額度 | 難估算，基於額度 |

## 接下來

市場在快速移動：
- **Daytona** 剛完成 $24M A 輪（2026 年 2 月）— fork/snapshot 是他們的賭注
- **Warp** Ambient Agents 在 beta — 事件觸發的雲端 agent
- **Cloudflare** Containers 公開測試 — Sandbox SDK 有 Claude Code 教程
- **GitHub** Agents HQ（2026 年 2 月）— GitHub 內多 agent 執行
- **Rivet** Sandbox Agent SDK — 通用 agent-沙箱抽象層

沙箱層正在商品化。價值向上移到**編排**（如何協調 N 個 agent）和**意圖規範**（如何告訴 agent 要構建什麼）。終端成為管理 agent 艦隊的駕駛艙，而不只是你打命令的地方。
