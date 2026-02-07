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

## 來源

- [事件驅動多 Agent 系統的四種設計模式 — Confluent](https://www.confluent.io/blog/event-driven-multi-agent-systems/)
- [選擇正確的多 Agent 編排模式 — Kore.ai](https://www.kore.ai/blog/choosing-the-right-orchestration-pattern-for-multi-agent-systems)
- [AI Agent 的狀態管理：Redis vs 外部數據庫 — DEV](https://dev.to/inboryn_99399f96579fcd705/state-management-patterns-for-long-running-ai-agents-redis-vs-statefulsets-vs-external-databases-39c5)
- [多 Agent 故障恢復 — Galileo](https://galileo.ai/blog/multi-agent-ai-system-failure-recovery)
- [為什麼 40% 的 AI Agent 項目失敗 — Squirro](https://squirro.com/squirro-blog/avoiding-agentic-ai-failure)
- [Agent Company：AI Agent 75% 任務失敗 — Reworked](https://www.reworked.co/digital-workplace/the-fake-startup-that-exposed-ais-real-limits-as-autonomous-workers/)
- [避免不可逾越的隊列積壓 — AWS](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/)
- [OpenClaw 架構指南 — Vertu](https://vertu.com/ai-tools/openclaw-clawdbot-architecture-engineering-reliable-and-controllable-ai-agents/)
- [OpenClaw 風險 — Cyber Strategy Institute](https://cyberstrategyinstitute.com/openclaw-risks-autonomous-ai-agents/)
- [CrewAI vs LangGraph vs AutoGen — DataCamp](https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen)
