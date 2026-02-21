# VoxYZ：自主 AI 公司 — 深度分析

> 來源：[@Voxyz_ai](https://x.com/Voxyz_ai/status/2019914775061270747) — 「我用 OpenClaw + Vercel + Supabase 建了一間 AI 公司 — 兩週後，它們自己運營了」

## 他們建了什麼

6 個 AI Agent 自主運營一間公司 — 研究、寫作、發帖、協調 — 在一個像素風虛擬辦公室裡。兩週建成。

| Agent | 角色 | 模型 |
|-------|------|------|
| Minion | 幕僚長 — 協調、委派 | Claude Opus |
| Sage | 研究主管 — 深度分析、策略 | GPT-5.2 |
| Scout | 增長主管 — 發現線索、機會 | GPT-5.1 Codex |
| Quill | 創意總監 — 撰寫文案、設計內容 | Claude Sonnet |
| Xalt | 社交媒體總監 — 發帖、增長受眾 | Gemini 3 Pro |
| Observer | 運營分析師 — 記錄一切 | GPT-5.2 |

## 為什麼選這個架構（而不是其他的）

### 為什麼選 OpenClaw 而不是 CrewAI / LangGraph / AutoGen？

| 框架 | 架構模式 | 最適合 | 弱點 |
|------|---------|--------|------|
| **LangGraph** | 圖導向 DAG | 複雜分支工作流 | 學習曲線陡，簡單循環用它是殺雞用牛刀 |
| **CrewAI** | 角色型團隊 | 快速原型、團隊隱喻 | 對執行流程控制較弱 |
| **AutoGen** | 對話型 Agent | 動態對話、角色切換 | 難以調試，路由不可預測 |
| **OpenClaw** | 閘道中心型管道 | 本地優先、多通道、持久化 | 生態系較年輕，社區支持少 |

VoxYZ 選 OpenClaw 的原因：
- **本地優先執行** — Agent 在你控制的硬體上跑 shell 命令、讀文件、瀏覽網頁
- **閘道中心** — 單一控制面板管理 session、通道、工具和事件。不需要另外接編排系統
- **多模型路由** — 內建模型解析器，自動故障轉移。某個 API key 被限流時自動切換備用
- **JSONL 審計軌跡** — 每個工具調用、每個執行結果逐行記錄。不需要額外基礎設施就有完整可追蹤性

關鍵洞察：OpenClaw 把 Agent 當作**結構化管道**，而不是「會思考的實體」。這讓調試和恢復變得可預測。

### 為什麼選 Supabase 而不是 Redis / 原生 PostgreSQL？

| 選項 | 延遲 | 即時性 | 持久化 | 複雜度 |
|------|------|--------|--------|--------|
| **Redis** | 亞毫秒 | Pub/sub（非持久） | 需要另一個 DB | 運維負擔重 |
| **原生 PostgreSQL** | 10-50ms | 無內建 | 完整 ACID | 什麼都要自己建 |
| **Supabase** | 10-50ms | 內建 WebSocket | 完整 ACID（它就是 Postgres） | Auth + 即時 + 存儲免費 |

VoxYZ 選 Supabase 的原因：
- 即時訂閱開箱即用 — Agent 可以監聽狀態變化，不需要輪詢
- Auth + Row Level Security — 可以限制 Agent 只看到自己的數據
- 底層就是 PostgreSQL — 沒有供應商鎖定，隨時可以遷移
- 自帶 Dashboard 方便調試 — 兩週衝刺中至關重要

**他們接受的取捨**：每次查詢 10-50ms 延遲，而不是 Redis 的亞毫秒。但 Agent 每次思考要花幾秒鐘，這點延遲可以忽略。只有在 Agent 每秒做 100+ 次狀態查詢時才需要 Redis — 而他們不會。

### 為什麼選 Vercel Cron 而不是專用 Worker？

**Vercel cron 的限制**：
- 需要 Pro 方案（$20/月）
- 每次調用最長 60 秒
- 最小間隔 1 分鐘（他們用 5 分鐘）
- 沒有持久進程 — 啟動、執行、結束

**為什麼還是選它**：簡單。一次部署，cron 就能跑。替代方案 — 一台 VPS 跑持久調度器 — 正是他們**一開始用的**，結果造成了最大的 bug（坑 #1：兩個調度器打架）。

**擴展時需要改什麼**：把心跳搬到外部調度器（AWS EventBridge、Upstash QStash 或專用 VPS），API 繼續留在 Vercel。

## 實際運作方式（執行追蹤）

### 閉環

```
Agent 提出想法
  → proposal-service 驗證（配額閘門 + 每日限額 + 評分）
    → 在 Supabase 建立任務
      → Minion 路由到正確的 Agent
        → Agent 通過 OpenClaw 管道執行
          → 結果寫入 Supabase
            → 觸發器評估新狀態
              → 反應產生新提案
                → 循環
```

### 為什麼單一提案服務至關重要

這是最重要的架構決策。所有動作都流經一個函數：

```
ProcessProposalWorkflow:
  1. 建立提案記錄
  2. 檢查每日限額（per-agent）
  3. 檢查配額上限（全局）
  4. 評估提案分數
  5. 超過閾值則自動審批
  6. 審批通過則建立任務
  7. 回傳結果
```

**為什麼不讓 Agent 直接建立任務？** 因為 [卡內基梅隆大學「Agent Company」研究](https://www.reworked.co/digital-workplace/the-fake-startup-that-exposed-ais-real-limits-as-autonomous-workers/)發現：AI Agent 會說謊、幻覺、製造失控循環。連 Claude 在複雜辦公任務中也有 75% 的失敗率。一個閘門函數意味著限流、評分、審計軌跡、緊急停止 — 全在一個地方。

### 觸發器/反應分離

**觸發器** = 純偵測。讀取狀態，評估條件，發出提案模板。永不寫入數據庫。

**反應** = Agent 間的回應，存為數據而非代碼：

```json
{ "source": "quill", "target": "xalt", "type": "content_ready", "action": "post" }
```

**為什麼要分離？** 因為觸發器如果同時執行，就會產生級聯故障。Agent A 觸發 Agent B 觸發 Agent A → 無限循環。讓觸發器只讀，所有東西都走 proposal-service 閘門，失控級聯就不可能發生。

## 三個坑（深度分析）

### 坑 1：雙調度器 → 競態條件

VPS cron 和 Vercel cron 同時跑同樣的工作。兩邊都會撈到同一個任務，執行兩次，寫入衝突的結果。

這是經典的[腦裂問題（split-brain）](https://en.wikipedia.org/wiki/Split-brain_(computing))— 兩個進程都認為自己是 leader。

**他們的修法**：直接砍掉 VPS 調度器。一個心跳，一個真相來源。

**他們可以用的其他方案**：

| 方案 | 複雜度 | 適用場景 |
|------|--------|---------|
| 砍掉一個調度器（他們的做法） | 低 | 小規模、一個團隊 |
| 分佈式鎖（Redis SETNX） | 中 | 多個 worker 競爭 |
| Leader 選舉（Raft/Paxos） | 高 | 大型分佈式系統 |
| 冪等操作 | 中 | 重複執行可接受時 |

最簡單的方案贏了。好品味。

### 坑 2：孤兒觸發器

觸發器觸發了但沒有 Agent 認領工作。提案消失在虛空中。

**根本原因**：觸發和執行是兩個獨立系統，之間沒有保證投遞。

**他們的修法**：單一 proposal-service 在一個原子操作中同時建立和路由。

這本質上就是[事務性發件箱模式（transactional outbox）](https://microservices.io/patterns/data/transactional-outbox.html)— 確保數據庫寫入和消息發送原子性地發生。把兩件事放在一個 Supabase 事務裡，免費獲得原子性。

### 坑 3：無界隊列增長

API 配額用完，但提案持續湧入。隊列永遠增長。

**三種處理方法**：

| 策略 | 運作方式 | 適用場景 |
|------|---------|---------|
| **門口拒絕**（VoxYZ 的選擇） | 入隊前檢查配額 | 低量（~50 提案/天） |
| **背壓（Backpressure）** | 向上游發信號減速 | 高吞吐管道 |
| **死信隊列（DLQ）** | 失敗消息路由到獨立隊列供檢查 | 需要重試邏輯的系統 |

VoxYZ 選了最簡單的。在 ~50 提案/天的量級下夠用。到了數千/天，就需要背壓或 DLQ。參見 [AWS：避免不可逾越的隊列積壓](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/)。

## 自我修復：為什麼不可妥協

來自 [Galileo 的研究](https://galileo.ai/blog/multi-agent-ai-system-failure-recovery)：Agent 依賴關係造成**不可預測的級聯效應**。當一個 Agent 失敗時，依賴其輸出的其他 Agent 會陷入未定義狀態。

VoxYZ 的心跳恢復：

```
每 5 分鐘：
  SELECT missions WHERE status='running' AND updated_at < NOW() - 30min
    → 標記為失敗
    → 從最後成功的步驟重新觸發
    → 重試計數器 +1
    → 重試 > 3 次則告警
```

**為什麼是 30 分鐘？** 最慢的 Agent（Sage，做深度研究）大約需要 15 分鐘。30 分鐘 = 最壞情況的 2 倍。低於這個值 → 假陽性。

**缺少什麼**：沒有斷路器（circuit breaker）。如果外部 API 宕機幾小時，心跳會每 30 分鐘無限重試。[斷路器模式](https://martinfowler.com/bliki/CircuitBreaker.html)會偵測持續性故障並停止重試直到恢復。

## 誠實評估：哪裡弱

| 弱點 | 風險 | 修法 |
|------|------|------|
| 沒有斷路器 | 持續宕機時無限重試 | 實現半開斷路器 |
| Vercel 60 秒超時 | 複雜任務無法在一次調用中完成 | 心跳搬到外部調度器 |
| 沒有人機協作 | [40% 的 AI Agent 項目失敗](https://squirro.com/squirro-blog/avoiding-agentic-ai-failure)因缺乏升級路徑 | 高風險動作加審批流程 |
| 單區域 Supabase | 單點故障 — Supabase 掛了全停 | 多區域或讀取副本 |
| 沒有成本控制 | 6 個 Agent × 高端模型 = 可能 $100+/天 | 每個 Agent 設每日花費上限 |
| 沒有輸出驗證 | [Agent 會幻覺和捏造](https://www.reworked.co/digital-workplace/the-fake-startup-that-exposed-ais-real-limits-as-autonomous-workers/) | 加輸出評分/驗證步驟 |

## 值得偷的模式

### 1. 閘門函數
所有 Agent 動作都走一個驗證函數。限流、評分、審批、緊急停止 — 一個地方搞定。

### 2. 觸發器只讀，服務才寫
偵測邏輯永遠不改變狀態。觸發器發出提案，服務執行。防止級聯循環。

### 3. 策略即數據
Agent 行為規則（配額、閾值、限額）存在數據庫表裡，不在代碼裡。改行為不需要重新部署。

### 4. 心跳自我修復
假設每個 Agent 都會卡住。週期性掃描器偵測陳舊任務，從最後的檢查點重啟。

### 5. 砍掉第二個調度器
兩個東西做同一件事時，其中一個就是 bug。永遠只選一個真相來源。

## 最新動態 (2026)

### VoxYZ 從 Demo 演變為產品平台

VoxYZ 已從兩週實驗成熟為完整產品平台 [voxyz.space](https://www.voxyz.space/)。六個 Agent 現在可以自主**追蹤真實問題、驗證需求、交付解決方案** — 從單純的內容生成進化到實際的產品開發。系統運行在一台 VPS（$8/月）和一個 Supabase 數據庫上，證明自主多 Agent 公司可以在極低的基礎設施成本下運營。

### 完整教程和生產架構發布

VoxYZ 發布了[完整教程](https://x.com/Voxyz_ai/status/2020272022417289587)，詳細介紹生產架構。系統在一台伺服器上運行 **10 個進程和 18 個 cron 任務，跨 4 個 AI 模型**。具名工作進程包括：`roundtable-worker`（對話編排 + 記憶提取）、`x-autopost`（推文發布）、`analyze-worker`（分析任務執行）、`content-worker`（內容創作）和 `crawl-worker`（網頁爬取）。這揭示了比最初兩週原型更精密的生產設置。

### 產品化：框架、Schema 和課程

VoxYZ 現在將其多 Agent 架構作為可重用套件提供：**多 Agent 框架**、**生產數據庫 schema** 和**視頻課程**。早期採用者報告拉取模板後不到 2 小時就能跑起第一個 Agent 管道 — 節省至少一週的腳手架搭建時間。這標誌著從「看我建了什麼」到「教你怎麼建」的轉變。

### OpenClaw 基金會化轉型

VoxYZ 核心基礎設施的重大變化：[OpenClaw 創始人 Peter Steinberger 於 2026 年 2 月 14 日加入 OpenAI](https://techcrunch.com/2026/02/15/openclaw-creator-peter-steinberger-joins-openai/)。Sam Altman 稱他是「對未來智能 Agent 如何互相協作有驚人想法的天才」。OpenClaw 將轉型為一個 OpenAI 持續支持的**開源基金會**。對 VoxYZ 來說，這是一把雙刃劍 — 底層框架獲得巨大的機構背書，但發展方向可能轉向 OpenAI 的優先事項。近期 OpenClaw 更新包括 Opus/Sonnet 的 1M 上下文 beta、`/subagents spawn` 命令、iOS 分享擴展和 Telegram 語音備忘錄轉寫。

### Base 鏈上的加密代幣

VoxYZ 在 Base 區塊鏈上推出代幣（「VoxYZ Agent World」），在 [Uniswap 上交易](https://dexscreener.com/base/0xaf8d92a6b6f8bdb1dd8f07dd5d6fb986339e89334530fc4ca823983ebdd9158a)，市值約 $22K 的微型規模。這跟隨了 AI Agent 項目發行代幣的更廣泛趨勢 — 參見 Virtuals Protocol 和類似的 AI-Agent-代幣平台。這是否為核心產品增添價值還是分散注意力，有待觀察。

### 行業背景：2026 是 Agent 規模化之年

VoxYZ 的時機與行業爆發性增長完美契合。Gartner 預測 **2026 年底將有 40% 的企業應用嵌入 AI Agent**（2025 年不到 5%）。AI Agent 市場預計到 2030 年達到 520 億美元。然而 Gartner 也警告，**2027 年前超過 40% 的 AI Agent 項目將因遺留系統無法支撐現代 AI 執行需求而失敗**。VoxYZ 的輕量架構（1 VPS + 1 Supabase）完全繞開了這個遺留系統問題 — 它生來就是雲原生的，零遺留債務。

### 病毒式傳播和社區

VoxYZ 的 Medium 文章於 2026 年 2 月在 [Coding Nexus](https://medium.com/coding-nexus/) 刊物發布，[@Voxyz_ai](https://x.com/Voxyz_ai) 的 X 帳號已增長至 6,200+ 追隨者。該項目作為一個實際示範引發關注 — 多 Agent 自主系統不再是理論，它們今天就在運營真實公司，處於真實規模。

## 參考資料

### 主要來源
- [@Voxyz_ai 推文串](https://x.com/Voxyz_ai/status/2019914775061270747) — 「我用 OpenClaw + Vercel + Supabase 建了一間 AI 公司 — 兩週後，它們自己運營了」
- [VoxYZ 官網](https://www.voxyz.space/) — 6 AI Agents, One Company
- [VoxYZ 關於頁面](https://www.voxyz.space/about)
- [VoxYZ 完整教程推文](https://x.com/Voxyz_ai/status/2020272022417289587) — 「完整教程：6 個 AI Agent 運營一間公司 — 我如何從零開始建造」
- [Medium 文章 — Coding Nexus](https://medium.com/coding-nexus/i-built-an-ai-company-with-openclaw-vercel-supabase-two-weeks-later-they-run-it-themselves-514cf3db07e6)

### 多 Agent 編排模式
- [事件驅動多 Agent 系統的四種設計模式 — Confluent](https://www.confluent.io/blog/event-driven-multi-agent-systems/)
- [選擇正確的多 Agent 編排模式 — Kore.ai](https://www.kore.ai/blog/choosing-the-right-orchestration-pattern-for-multi-agent-systems)
- [CrewAI vs LangGraph vs AutoGen — DataCamp](https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen)

### OpenClaw
- [OpenClaw 架構指南 — Vertu](https://vertu.com/ai-tools/openclaw-clawdbot-architecture-engineering-reliable-and-controllable-ai-agents/)
- [什麼是 OpenClaw — Contabo](https://contabo.com/blog/what-is-openclaw-self-hosted-ai-agent-guide/)
- [OpenClaw 風險 — Cyber Strategy Institute](https://cyberstrategyinstitute.com/openclaw-risks-autonomous-ai-agents/)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw 創始人加入 OpenAI — TechCrunch](https://techcrunch.com/2026/02/15/openclaw-creator-peter-steinberger-joins-openai/)
- [Peter Steinberger 的公告](https://steipete.me/posts/2026/openclaw)
- [2026 年的 OpenClaw — DigitalOcean](https://www.digitalocean.com/resources/articles/what-is-openclaw)

### 狀態管理
- [AI Agent 的狀態管理：Redis vs 外部數據庫 — DEV](https://dev.to/inboryn_99399f96579fcd705/state-management-patterns-for-long-running-ai-agents-redis-vs-statefulsets-vs-external-databases-39c5)
- [Supabase vs Redis 比較 — Leanware](https://www.leanware.co/insights/supabase-vs-redis-comparison)

### 故障與自我修復
- [多 Agent 故障恢復 — Galileo](https://galileo.ai/blog/multi-agent-ai-system-failure-recovery)
- [為什麼 40% 的 AI Agent 項目失敗 — Squirro](https://squirro.com/squirro-blog/avoiding-agentic-ai-failure)
- [Agent Company：AI Agent 75% 任務失敗 — Reworked](https://www.reworked.co/digital-workplace/the-fake-startup-that-exposed-ais-real-limits-as-autonomous-workers/)
- [AI Agent 失敗的 4 個常見原因 — Built In](https://builtin.com/articles/agentic-ai-implementation-failure-causes)

### 隊列與分佈式系統模式
- [避免不可逾越的隊列積壓 — AWS](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/)
- [分佈式系統中的背壓 — DEV](https://dev.to/devcorner/effective-backpressure-handling-in-distributed-systems-techniques-implementations-and-workflows-16lm)
- [事務性發件箱模式 — microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)

### Vercel Cron
- [Vercel Cron Jobs 設置指南](https://vercel.com/guides/how-to-setup-cron-jobs-on-vercel)
- [在 Vercel 上免費運行 Cron Jobs — DEV](https://dev.to/hexshift/how-to-run-cron-jobs-in-a-vercel-serverless-environment-without-paying-extra-502h)

### 行業背景 (2026)
- [2026 年值得關注的 7 個 AI Agent 趨勢 — MachineLearningMastery](https://machinelearningmastery.com/7-agentic-ai-trends-to-watch-in-2026/)
- [AI Agent 策略 — Deloitte](https://www.deloitte.com/us/en/insights/topics/technology-management/tech-trends/2026/agentic-ai-strategy.html)
- [馴服 AI Agent：2026 年的自主勞動力 — CIO](https://www.cio.com/article/4064998/taming-ai-agents-the-autonomous-workforce-of-2026.html)
- [2026 年 AI Agent 指南 — IBM](https://www.ibm.com/think/ai-agents)
- [VoxYZ Agent World 代幣 — DEX Screener](https://dexscreener.com/base/0xaf8d92a6b6f8bdb1dd8f07dd5d6fb986339e89334530fc4ca823983ebdd9158a)
