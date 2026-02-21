# Cloudflare 2025-2026：全栈边缘云深度分析

## 概述

Cloudflare 从 CDN/安全公司演变为**垂直整合的 AI 基础设施平台**。FY2025 营收 $21.7 亿（+30%），Q4 加速至 34%。3 个月内完成 3 笔收购（Replicate、Astro、Human Native）。发布 Containers、Agents SDK、x402 协议、后量子 Zero Trust，刷新 31.4 Tbps DDoS 缓解记录。335+ 全球节点上运行 60+ 产品。核心赌注：成为 Agentic Internet 的默认基础设施层。

---

## 1. Workers 与 Containers

### Workers 运行时

基于 V8 隔离的无服务器计算。2025 年大规模 Node.js 兼容性推进——`node:fs`、`node:net`、`node:tls`、`node:http`、`node:process` 原生运行时实现（非 polyfill）。

**Workers Builds** GA（Birthday Week 2025）。新的面向资源的 REST API（beta）将 Workers/Versions/Deployments 作为独立资源管理。

**战略转向**：Pages 已弃用，所有投资转向 Workers——现在同时支持静态资源服务和 SSR。支持 Astro、React Router、Hono、Vue、Nuxt、Svelte。

| 套餐 | 费用 | 包含 |
|------|------|------|
| 免费 | $0 | 10 万请求/天，10ms CPU/调用 |
| 付费 | $5/月 | 1000 万请求/月 + $0.50/百万，30s CPU，**I/O 等待免费** |

### Containers（公测，2025 年 6 月）

边缘 Docker 容器，基于 Durable Objects 基础设施。缩放到零，10ms 计费粒度。

**架构**：请求 → Worker（API 网关/路由/认证）→ Durable Object（可编程 sidecar）→ Container（Docker 镜像）。Worker 充当网关、服务网格或编排器。

| 类型 | 内存 | vCPU | CPU 价格 |
|------|------|------|----------|
| Dev | 256 MB | 1/16 | $0.00002/vCPU-秒 |
| Basic | 1 GB | 1/4 | $0.00002/vCPU-秒 |
| Standard | 4 GB | 1/2 | $0.00002/vCPU-秒 |

最大：400 GiB 内存、100 vCPU、2 TB 磁盘。空闲自动休眠。**暂不支持 GPU**，仅 Linux/amd64。

### Sandbox SDK

Workers + Durable Objects + Containers 实现**安全的不可信代码执行**。Shell 命令、Python/JS 解释器、文件系统、Git、后台进程、公开 URL。官方 Claude Code 模板：

```bash
npm create cloudflare@latest -- claude-code-sandbox \
  --template=cloudflare/sandbox-sdk/examples/claude-code
```

接受仓库 URL + 任务描述 → 创建沙箱 → 运行 Claude Code → 返回 diff。

**Workers for Platforms**：在隔离 Workers 中运行客户/AI 生成的代码，每客户独立安全边界。

---

## 2. AI 技术栈

### Workers AI

190+ 城市的无服务器 GPU 推理。50+ 模型。$0.011/千 Neurons。OpenAI 兼容 API。

| 模型 | 参数 | 备注 |
|------|------|------|
| Llama 4 Scout | 17B MoE | 多模态，Meta 最新 |
| Llama 3.3 70B | 70B | 128K 上下文 |
| Mistral Small 3.1 | 24B | 视觉 + 工具调用 |
| gpt-oss-120b | 120B | OpenAI，高推理 |
| gpt-oss-20b | 20B | OpenAI，低延迟 |

功能：LoRA 适配器、函数调用、流式输出、批处理、OpenAI 模型 Responses API。

### Infire 引擎（v0.5.0）

自研 **Rust 推理引擎**取代 Python 的 vLLM：
- 细粒度 CUDA 图：按 batch 大小 JIT 编译，**CPU 开销降低 82%**
- 分页 KV 缓存：非连续内存块防止碎片化
- H100 NVL 上 40.91 req/s vs vLLM 38.38，CPU 负载仅 25%（vLLM 140%）
- Llama-3-8B 加载 <4 秒

### Replicate 收购（2025 年 11 月）

50,000+ 生产就绪模型加入 Workers AI。模型市场 + Cloudflare 边缘网络 = 大规模分布式推理。

### Agents SDK（v0.5.0，2026 年 2 月）

基于 Durable Objects 的有状态 AI 代理。每个代理 = 身份 + 状态 + 单线程执行 + WebSocket。

| 层 | 技术 | 角色 |
|---|------|------|
| 运行时 | Durable Objects | 有状态微服务器，单线程 |
| 存储 | 内嵌 SQLite | 1GB/实例（10GB GA），零延迟 |
| 通信 | WebSocket + HTTP | 实时双向，支持休眠 |
| 推理 | Infire (Rust) | 自研边缘 GPU 引擎 |

核心能力：`this.state` 自动同步、`this.schedule()` 定时任务、WebSocket 休眠、AI 模型调用（Workers AI/OpenAI/Anthropic via AI SDK）、MCP 服务器/客户端（`McpAgent`）、邮件收发、任务队列、`this.retry()` 指数退避。

**关键设计**：单线程 Durable Objects = 无竞态条件。多个输入排队原子处理。消除了其他代理框架中的并发 bug。

**Agent + Workflow 模式**：Agent 单独用于聊天/快速调用。Agent + Workflow 用于长时运行任务（30s+）、多步管道、人机协同。

### MCP on Workers

一流的远程 MCP 服务器托管。每个会话由 Durable Object 支撑（持久状态）。

**传输**：Streamable HTTP（取代 SSE，2025 年 3 月）。**认证**：Worker 同时是 OAuth 客户端（上游）和 OAuth 服务器（面向 MCP 客户端）——双代理模式将令牌范围限定到特定工具。

**Code Mode**：对于大型 API（Cloudflare 有 2,500+ 端点），传统工具定义 ~117 万 tokens。Code Mode 用 2 个工具（`search()` + `execute()`）= **~1,000 tokens——99.9% 减少**。LLM 在沙箱 V8 隔离中针对类型化 SDK 编写 TypeScript。

**Elicitation**（2025 年 8 月）：MCP 服务器在工具执行期间请求用户输入。状态跨休眠保持。

### AutoRAG（AI Search）

托管 RAG 管道，2025 年 4 月公测。R2 存储桶 → 自动索引 → LLM 查询。零配置。

```
R2 → 文件摄入 → Markdown 转换 → 图像处理
  → 分块 → 嵌入 → Vectorize 存储
  → 查询 → 改写 → 向量搜索 → 回答生成
```

10 实例/账户，10 万文件/实例。**NLWeb 集成**（Microsoft）：`/ask` + `/mcp` 端点实现对话式搜索。任何网站变成 MCP 兼容数据源。

### AI Gateway

24+ AI 提供商的统一代理（OpenAI、Anthropic、Azure、Bedrock、Vertex、Groq、Perplexity、xAI、DeepSeek...）。单一 OpenAI 兼容 `/chat/completions` 端点。

| 功能 | 详情 |
|------|------|
| 缓存 | 相同请求边缘缓存 |
| 速率限制 | 按网关配置 |
| 模型降级 | 自动切换到备选提供商 |
| 分析 | 请求量、延迟、成本仪表板 |
| 统一计费 | 通过 Cloudflare 发票支付（2026） |

核心功能（分析、缓存、速率限制）**所有套餐免费**。

### Vectorize

IVF + 乘积量化索引。余弦/欧氏/点积距离。元数据过滤 `$in`/`$nin`。5 万索引/账户。免费：3000 万查询维度/月，500 万存储维度。不收 CPU、内存或"活跃索引小时"费。

---

## 3. 开发者平台

### Durable Objects

有状态微服务器——Agents SDK、Containers、Queues 的基础。**SQLite 存储 GA**：10GB/对象，零延迟，完整 SQL。PITR：30 天时间点恢复。

**免费套餐**（2025 年 4 月）：10 万请求/天，5GB 存储。WebSocket 休眠：代理休眠不断开客户端连接。WebSocket 消息大小：32 MiB。

### D1（无服务器 SQLite）

**全球读副本**（beta）：每个区域自动副本。REST API 延迟 -50-500ms。Worker API 延迟 -40-60%。

### R2（对象存储）——零出口费

| 指标 | 标准 | 低频访问 |
|------|------|----------|
| 存储 | $0.015/GB/月 | 更低 |
| Class A | $4.50/百万 | $9.00/百万 |
| Class B | $0.36/百万 | $0.90/百万 |
| **出口流量** | **$0** | **$0** |

vs S3：10TB 存储 + 10TB 出口/月 = $1,121（S3）vs $150（R2）= **节省 87%**。免费：10GB 存储。

### Hyperdrive

外部数据库连接池 + 查询缓存。**MySQL 支持**（2025 年 4 月）。PlanetScale 合作（2025 年 9 月）。

### Queues

消息队列，P50 写入延迟 ~60ms（从 ~200ms 下降）。最高 5,000 msg/sec。$0.40/百万操作（超出 100 万免费额度）。

### Pipelines

流式摄入（收购 Arroyo，Rust 流处理器）。Workers/HTTP → DO 缓冲 → Arroyo SQL 转换 → R2 Iceberg/Parquet/JSON。~10 万记录/秒，精确一次交付。**Beta 期间免费。**

### Workflows（GA 2025 年 4 月）

持久执行。3 种步骤：`step.do()`（带重试执行）、`step.sleep()`（暂停秒到天）、`step.waitForEvent()`（等待 webhook）。仅 CPU 时间计费——空闲等待免费。**Python beta**（2025 年 11 月）。

### Browser Rendering（GA）

Puppeteer v22.13.1 + Playwright v1.57.0 + **Stagehand**（Workers AI 驱动的浏览器自动化）。10 个并发浏览器/账户。会话保持最长 10 分钟不活动。2025 年 8 月开始计费。

### Secrets Store（Beta，2025 年 4 月）

跨所有 Workers 的集中加密密钥管理。RBAC，审计日志。通过绑定在运行时访问——无 API 密钥泄露风险。

### Wrangler v4（2025 年 3 月）

esbuild v0.24，本地优先（所有资源命令默认本地），从绑定生成 TypeScript 类型，Node.js v18+。v3→v4 迁移几乎无需改动。

---

## 4. x402 协议

与 Coinbase 联合创立（2025 年 9 月）。HTTP 402 Payment Required 用于机器对机器微支付。

```
客户端 → GET /resource
服务器 → 402 Payment Required（金额、收款人、调解者）
客户端 → GET /resource + X-PAYMENT 头
调解者验证 → 链上结算
服务器 → 200 OK + 资源
```

**截至 2025 年 12 月：7500 万笔交易，$2400 万处理额。** 目前 USDC on Base。

**延迟支付扩展**：批量结算、订阅，无需每次请求区块链开销。Agents SDK 集成：服务端 `withX402()` + `paidTool()`，客户端 `withX402Client()`。

**AI Crawl Control**：检测爬虫 → 402 响应 + 定价 → x402 支付 → 内容交付。自动化变现。

---

## 5. 安全与网络

### DDoS——破纪录之年

| 季度 | 峰值攻击 | 备注 |
|------|----------|------|
| Q1 2025 | 6.5 Tbps | 2050 万次攻击拦截（+358% YoY）|
| Q3 2025 | 29.7 Tbps | 当时最大记录 |
| Q4 2025 | **31.4 Tbps** | 新历史记录，35 秒 |

平均每小时缓解 5,376 次攻击。HTTP DDoS 超过 2 亿请求/秒。

### 后量子加密——行业领先

CRYSTALS-KYBER（NIST 批准）。**43% 人类网络流量**后量子保护。Zero Trust PQC（2025 年 3 月）。所有 WARP 客户端流量通过 PQC 隧道。自动 PQC 握手（Q4 2025）。

### Zero Trust / SASE

Gartner SASE 远见者，SSE 连续第 3 年入选。Cloudflare One 套件：Access（ZTNA）、Gateway（SWG）、Browser Isolation、CASB（现扫描 ChatGPT/Claude/Gemini）、DLP、Magic WAN、Email Security。

**AI 新增**：AI CASB、AI Prompt Protection、Firewall for AI（内容审核）、MCP Server Portals（公测）。

---

## 6. 媒体与邮件

| 产品 | 状态 | 关键功能 |
|------|------|----------|
| Email Service | 内测（2025.9） | 从 Workers 通过绑定发送，无需 API 密钥，自动 SPF/DKIM/DMARC |
| Media Transformations | GA（2025.9） | 图片+视频统一计费，$0.50/千次转换 |
| Stream | GA | 直播剪辑 API |
| Browser Rendering | GA | Puppeteer + Playwright + Stagehand |
| Calls | 公测 | WebRTC 实时媒体 |
| Realtime + RealtimeKit | 2025 | 音视频 SDK（Kotlin、RN、Swift、JS、Flutter）|

---

## 7. 收购（3 个月 3 笔）

| 公司 | 日期 | 内容 | 原因 |
|------|------|------|------|
| Replicate | 2025.11 | 5 万+ AI 模型平台 | Workers AI 模型目录扩展 |
| Human Native | 2026.1.15 | AI 数据市场（DeepMind/Google 校友）| AI 时代内容变现 |
| Astro | 2026.1.16 | 开源 Web 框架（Unilever、Visa）| 拥有框架到边缘的全栈 |

**模式**：Human Native（内容经济）+ Replicate（AI 模型）+ Astro（开发框架）= 垂直整合的 AI 开发平台。

另外：**VibeSDK**（开源，2025.9）——MIT 许可的 AI"氛围编码"平台。React+Vite + Workers + D1 + R2 + KV。默认 Gemini 模型，在 Containers 中运行生成的应用。

---

## 8. 财务表现

### FY2025

| 指标 | 数值 | 同比 |
|------|------|------|
| 营收 | $21.679 亿 | +29.8% |
| Q4 营收 | $6.145 亿 | +33.6% |
| 付费客户 | 33.2 万 | +40% |
| >$10 万 ARR | 4,298 | +23% |
| >$100 万 ARR | 269 | +55% |
| RPO | $24.96 亿 | +48% |
| 最大合同 | $4250 万 ACV | 历史记录 |
| Q4 自由现金流 | $9940 万（16.2%） | +108% |
| 净留存率 | ~114% | 稳定 |

营收增长连续 3 个季度**加速**至 34%——在这个体量下极为罕见。

### FY2026 指引

营收 $27.9 亿（+28-29%）。营业利润 $3.78-3.82 亿（14% 利润率）。EPS $1.11-1.12。

### 股票（NET）

~$192.64，市值 ~$674 亿。52 周：$89-$260。P/S ~31x。分析师共识"买入"，目标价 $229-234。

---

## 9. 竞争格局

| 功能 | Cloudflare Workers | Vercel Edge | Lambda@Edge | Deno Deploy |
|------|-------------------|-------------|-------------|-------------|
| 运行时 | V8 隔离 | V8 隔离 | KVM 微虚拟机 | V8 隔离 |
| 冷启动 | <5ms | 极小 | 100ms-1s+ | <5ms |
| 节点 | 335+ | ~20 区域 | 13 区域 | 35+ |
| 存储 | R2, KV, D1, DO | Blob, KV, Postgres | S3, DynamoDB | KV |
| AI 推理 | Workers AI（内建） | 通过 API | SageMaker | 通过 API |
| 容器 | 有（beta） | 无 | Fargate | 无 |
| 出口费 | $0 | 标准 | 标准 | 标准 |

**Workers：比 Lambda@Edge 快 210%，冷启动快 9x。**

**vs Vercel**：Vercel 优化框架层（Next.js ISR），Cloudflare 优化网络层（延迟、安全、缓存）。收购 Astro = 进入 Vercel 的框架领地。

**核心差异化**：唯一在单一集成栈中提供隔离 + 容器 + 托管数据库 + 向量库 + AI 推理 + 邮件 + 媒体 + 安全 + 零出口费的平台。

---

## 10. 产品版图（60+ 服务）

所有产品运行在同一 335+ 节点网络、同一硬件上。添加产品 = "开个开关"。

**应用服务**：CDN、DNS、WAF、DDoS、Bot Management、AI Crawl Control、SSL/TLS、Page Shield、API Shield、Turnstile、Load Balancing、Argo Smart Routing、Images、Stream、Zaraz、Observatory

**开发者平台**：Workers、Workers AI、AI Gateway、Containers、R2、D1、KV、DO、Queues、Workflows、Hyperdrive、Vectorize、Browser Rendering、Realtime、Secrets Store、Pipelines

**网络服务**：Magic WAN、Magic Transit、Magic Firewall、Spectrum、Network Interconnect、WARP、Tunnel

**Zero Trust（Cloudflare One）**：Access、Gateway、Browser Isolation、CASB、DLP、Email Security、DEM

**开源**：Pingora（Rust 代理，1 万亿+请求/天，比 NGINX 减少 70% CPU）、workerd（JS/Wasm 运行时）、547 个仓库

---

## 11. 诚实评估

### 优势
- **集成栈**：每个产品原生互通，零胶水代码
- **零出口费**：R2 比 S3 节省 87-99%
- **缩放到零**：Containers、Workers、DO 全部休眠——只为活跃计算付费
- **MCP 领先**：最佳远程 MCP 托管。OAuth、状态、elicitation 内建
- **后量子领导力**：43% 流量已保护，多数公司尚未开始
- **开发者速度**：单一 `wrangler deploy` 处理一切

### 弱点
- **内存**：Workers 128MB（vs Lambda 10GB）
- **容器成熟度**：Beta，无 GPU，无 exec，无全球自动扩展
- **V8 限制**：无原生编译语言（Rust/Go = 仅 Wasm）
- **D1 规模**：SQLite——不能替代严肃规模的 Postgres
- **SASE 位置**：远见者而非领导者（Zscaler/Palo Alto 领先）
- **供应商锁定**：Durable Objects、D1、KV 无可移植等效物
- **Infire 成熟度**：目前仅 Llama 3.1 8B，多 GPU 在路线图上
- **毛利率下降**：73.6%（从 76.4%）因 GPU 投资

### 战略风险
- AWS/Google 可以用更深的资本匹配功能
- Vercel 占据 Next.js 心智份额
- 31x P/S 要求多年保持 25%+ 增长
- x402 临近加密——监管不确定性
- 3 个月 3 笔收购 = 整合风险

---

## 12. 可偷的模式

### 架构
1. **Durable Object 作为 Agent 原语**：身份 + 状态 + 单线程 + WebSocket。消除分布式状态 bug
2. **大 API 的 Code Mode**：2 个工具（`search` + `execute`）对类型化 SDK。比完整工具定义减少 99.9% token
3. **MCP OAuth 双代理**：MCP 服务器 = OAuth 客户端（上游）+ OAuth 服务器（面向代理）。范围令牌限制爆炸半径
4. **Worker-as-Gateway-to-Container**：便宜隔离处理路由/认证/缓存，重计算转发到缩放到零的容器
5. **x402 变现闭环**：AI Crawl Control → 402 → 支付 → 内容交付。自动化变现
6. **Infire 细粒度 CUDA 图**：按 batch 大小 JIT 图而非一刀切。CPU 降低 82%
7. **休眠感知状态**：`serializeAttachment`/`deserializeAttachment` 在代理休眠中保活，不断开 WebSocket

### 商业
8. **Innovation Week 节奏**：每年 5+ 主题发布周。制造新闻、开发者兴奋度、发布节奏
9. **以安全着陆，以计算扩展**：DDoS = 即时痛点。流量流过后，计算边际成本接近零
10. **"反锁定"即锁定**：零出口费吸引客户，然后深度构建在平台上

---

## 最新動態 (2026)

### Moltworker：边缘自托管 AI 代理（2026 年 2 月 7 日）

Cloudflare 发布了 **Moltworker**，一个开源概念验证项目，用于在 Cloudflare 开发者平台上运行 Moltbot（前身为 Clawdbot）——一个自托管的个人 AI 代理。无需专用本地硬件（如 Mac mini）。架构：入口 Worker（API 路由 + 管理层）+ 隔离的 Sandbox 容器（Moltbot 运行时）。集成 AI Gateway 实现多提供商模型路由、Browser Rendering 实现无头 Chromium 自动化、Zero Trust Access 实现认证。持久状态存储在 R2。展示了完整 Cloudflare 技术栈作为代理托管平台的协同工作能力。

### 容器与沙箱的 Docker-in-Docker 支持（2026 年 2 月 17 日）

Containers 和 Sandboxes 现在支持 **Docker-in-Docker**——在容器内运行 Docker。关键用例：为代理提供沙箱化开发环境、隔离的容器镜像测试、容器内 CI/CD 镜像构建、运行时部署用户提供的任意镜像。这对需要完整开发环境控制的 AI 编码代理来说是重大能力升级。

### Queues 免费套餐（2026 年 2 月 4 日）

Cloudflare Queues 现已加入 **Workers 免费套餐**。免费额度包含 10,000 次操作/天（读、写、删除），每账户最多 10,000 个队列，保证消息送达至 Workers 或 HTTP pull 消费者，无限事件订阅。唯一限制：最长 24 小时保留期（付费套餐为 14 天）。降低了消息驱动架构的入门门槛。

### Workers AI 上的 GLM-4.7-Flash + TanStack AI（2026 年 2 月 13 日）

新模型加入：**GLM-4.7-Flash**，快速多语言文本生成模型，支持多轮工具调用。同时发布 `@cloudflare/tanstack-ai` 包和 `workers-ai-provider` v3.1.1，完全兼容 TanStack AI 和 Vercel AI SDK。可以完全在边缘用主流 AI SDK 工具构建代理应用。

### AI Search 品牌升级 + 外部模型支持

AutoRAG 正式更名为 **AI Search**，承载更大使命。现支持通过 AI Gateway 提供商密钥使用 OpenAI 和 Anthropic 的外部模型。新规范导出 `createAISearch`（替代 `AutoRAG`）。嵌入和生成模型都可来自外部提供商，打破了仅限 Workers AI 模型的锁定。

### 容器并发扩展

容器并发限制大幅提高：最多 **1,000 个 dev 实例**或 **400 个 basic 实例**同时运行（此前为 50/25）。最大规格不变：400 GiB 内存、100 vCPU、2 TB 磁盘。GPU 支持在底层容器平台内部开发中，但尚未对 Containers 产品开放。

### 学生免费 + 创业中心

美国大学生凭 `.edu` 邮箱可获 Workers 付费套餐免费使用（12 个月）。自 2026 年 1 月起，在 Cloudflare 上构建的获批创业公司可在 **奥斯汀、里斯本、伦敦和旧金山** 的办公空间指定日期使用共享办公。2026 年全球计划招收 1,111 名实习生。已向 4,000+ 初创公司发放超过 $3.7 亿美元信用额度。

---

## 关键日期时间线

| 日期 | 事件 |
|------|------|
| 2025.2 | Agents SDK 初始发布 |
| 2025.3 | 后量子 Zero Trust、Wrangler v4 |
| 2025.4 | Developer Week：AutoRAG、D1 复制、DO 免费套餐、Secrets Store、Workflows GA、MySQL Hyperdrive |
| 2025.6 | Containers 公测 |
| 2025.8 | NLWeb 合作、Sandbox SDK 更新、OpenAI 模型上 Workers AI |
| 2025.9 | Birthday Week：Email Service、VibeSDK、x402 Foundation、Workers Builds GA、Pingora 取代 NGINX |
| 2025.11 | Replicate 收购、Container CPU 定价、Python Workflows |
| 2026.1 | Human Native + Astro 收购 |
| 2026.2 | Agents SDK v0.5.0、FY2025 财报（$21.7 亿）、31.4 Tbps DDoS 记录、Moltworker、Docker-in-Docker、Queues 免费、GLM-4.7-Flash |

---

## References

### Workers & Containers
- [Containers coming to Workers June 2025](https://blog.cloudflare.com/cloudflare-containers-coming-2025/)
- [Containers public beta](https://blog.cloudflare.com/containers-are-available-in-public-beta-for-simple-global-and-programmable/)
- [Container pricing](https://developers.cloudflare.com/containers/pricing/)
- [New CPU pricing for Containers/Sandboxes](https://developers.cloudflare.com/changelog/2025-11-21-new-cpu-pricing/)
- [Container instance types & limits](https://developers.cloudflare.com/containers/platform-details/limits/)
- [Custom instance types (Jan 2026)](https://developers.cloudflare.com/changelog/2026-01-05-custom-instance-types/)
- [Docker-in-Docker support (Feb 2026)](https://developers.cloudflare.com/changelog/2026-02-17-docker-in-docker/)
- [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Node.js compatibility 2025](https://blog.cloudflare.com/nodejs-workers-2025/)
- [Node.js fs in Workers](https://developers.cloudflare.com/changelog/2025-08-15-nodejs-fs/)
- [New Workers REST API (beta)](https://developers.cloudflare.com/changelog/2025-09-03-new-workers-api/)
- [Workers best practices](https://developers.cloudflare.com/changelog/2026-02-15-workers-best-practices/)

### Sandbox SDK
- [Sandbox SDK docs](https://developers.cloudflare.com/sandbox/)
- [Sandbox SDK GitHub](https://github.com/cloudflare/sandbox-sdk)
- [Run Claude Code on Sandbox](https://developers.cloudflare.com/sandbox/tutorials/claude-code/)
- [Build AI Code Executor](https://developers.cloudflare.com/sandbox/tutorials/ai-code-executor/)
- [Workers for Platforms](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/)

### Agents SDK
- [Agents docs](https://developers.cloudflare.com/agents/)
- [Agents GitHub](https://github.com/cloudflare/agents)
- [Agents SDK v0.5.0 (Feb 2026)](https://developers.cloudflare.com/changelog/2026-02-17-agents-sdk-v050/)
- [Agents SDK v0.3.0 + AI SDK v6 (Dec 2025)](https://developers.cloudflare.com/changelog/2025-12-22-agents-sdk-ai-sdk-v6/)
- [Agents SDK v0.1.0 + AI SDK v5 (Sep 2025)](https://developers.cloudflare.com/changelog/2025-09-03-agents-sdk-beta-v5/)
- [MCP Elicitation, Task Queues, Email (Aug 2025)](https://developers.cloudflare.com/changelog/2025-08-05-agents-mcp-update/)
- [Building agents with OpenAI](https://blog.cloudflare.com/building-agents-with-openai-and-cloudflares-agents-sdk/)
- [Agents SDK v0.5.0 analysis (MarkTechPost)](https://www.marktechpost.com/2026/02/17/cloudflare-releases-agents-sdk-v0-5-0-with-rewritten-cloudflare-ai-chat-and-new-rust-powered-infire-engine-for-optimized-edge-inference-performance/)
- [Agents API reference](https://developers.cloudflare.com/agents/api-reference/agents-api/)
- [Build AI agents on Cloudflare](https://blog.cloudflare.com/build-ai-agents-on-cloudflare/)
- [MCP + auth + DO free tier](https://blog.cloudflare.com/building-ai-agents-with-mcp-authn-authz-and-durable-objects/)

### Moltworker
- [Introducing Moltworker (Feb 2026)](https://blog.cloudflare.com/moltworker-self-hosted-ai-agent/)
- [Moltworker analysis (InfoQ)](https://www.infoq.com/news/2026/02/cloudflare-moltworker/)

### Infire Engine
- [How we built most efficient inference engine](https://blog.cloudflare.com/cloudflares-most-efficient-ai-inference-engine/)

### MCP Servers
- [Code Mode: MCP in 1,000 tokens](https://blog.cloudflare.com/code-mode-mcp/)
- [Build MCP servers on Workers](https://blog.cloudflare.com/model-context-protocol/)
- [Remote MCP servers guide](https://developers.cloudflare.com/agents/guides/remote-mcp-server/)
- [MCP transport docs](https://developers.cloudflare.com/agents/model-context-protocol/transport/)
- [MCP authorization docs](https://developers.cloudflare.com/agents/model-context-protocol/authorization/)
- [Cloudflare MCP servers catalog](https://developers.cloudflare.com/agents/model-context-protocol/mcp-servers-for-cloudflare/)
- [Anthropic + Cloudflare MCP partnership](https://www.cloudflare.com/press/press-releases/2025/cloudflare-helps-anthropic-and-leading-tech-companies-to-unlock-real-ai-through-claude-mcp/)

### x402 Protocol
- [x402 Foundation announcement](https://blog.cloudflare.com/x402/)
- [Coinbase x402 Foundation blog](https://www.coinbase.com/blog/coinbase-and-cloudflare-will-launch-x402-foundation)
- [x402 official site](https://www.x402.org)
- [x402 docs](https://developers.cloudflare.com/agents/x402/)
- [x402 deep dive (DWF Labs)](https://www.dwf-labs.com/research/inside-x402-how-a-forgotten-http-code-becomes-the-future-of-autonomous-payments)
- [x402 deep dive (Fintech Wrap Up)](https://www.fintechwrapup.com/p/deep-dive-is-x402-payments-protocol)

### Workers AI
- [Workers AI docs](https://developers.cloudflare.com/workers-ai/)
- [Workers AI models catalog](https://developers.cloudflare.com/workers-ai/models/)
- [Workers AI pricing](https://developers.cloudflare.com/workers-ai/platform/pricing/)
- [OpenAI open models on Workers AI (Aug 2025)](https://developers.cloudflare.com/changelog/2025-08-05-openai-open-models/)
- [gpt-oss-120b](https://developers.cloudflare.com/workers-ai/models/gpt-oss-120b/)
- [OpenAI gpt-oss on Workers AI](https://blog.cloudflare.com/openai-gpt-oss-on-workers-ai/)
- [OpenAI compatible endpoints](https://developers.cloudflare.com/workers-ai/configuration/open-ai-compatibility/)
- [GLM-4.7-Flash on Workers AI (Feb 2026)](https://developers.cloudflare.com/changelog/2026-02-13-glm-47-flash-workers-ai/)

### AutoRAG / AI Search
- [AI Search docs](https://developers.cloudflare.com/ai-search/)
- [Introducing AutoRAG](https://blog.cloudflare.com/introducing-autorag-on-cloudflare/)
- [NLWeb + AutoRAG conversational search](https://blog.cloudflare.com/conversational-search-with-nlweb-and-autorag/)
- [AutoRAG InfoQ analysis](https://www.infoq.com/news/2025/04/cloudflare-autorag-rag-llm/)
- [AI Search external models (Sep 2025)](https://developers.cloudflare.com/changelog/2025-09-25-ai-search-more-models/)
- [Bring your own generation model](https://developers.cloudflare.com/ai-search/how-to/bring-your-own-generation-model/)

### AI Gateway
- [AI Gateway docs](https://developers.cloudflare.com/ai-gateway/)
- [AI Gateway features](https://developers.cloudflare.com/ai-gateway/features/)
- [AI Gateway supported providers](https://developers.cloudflare.com/ai-gateway/usage/providers/)
- [AI Gateway pricing](https://developers.cloudflare.com/ai-gateway/reference/pricing/)

### Vectorize
- [Vectorize docs](https://developers.cloudflare.com/vectorize/)
- [Vectorize pricing](https://developers.cloudflare.com/vectorize/platform/pricing/)
- [Building Vectorize](https://blog.cloudflare.com/building-vectorize-a-distributed-vector-database-on-cloudflare-developer-platform/)

### Durable Objects
- [Durable Objects docs](https://developers.cloudflare.com/durable-objects/)
- [SQLite storage API](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/)
- [DO free tier (Apr 2025)](https://developers.cloudflare.com/changelog/2025-04-07-durable-objects-free-tier/)
- [SQLite in DO GA 10GB (Apr 2025)](https://developers.cloudflare.com/changelog/2025-04-07-sqlite-in-durable-objects-ga/)
- [SQLite billing (Dec 2025)](https://developers.cloudflare.com/changelog/2025-12-12-durable-objects-sqlite-storage-billing/)
- [Zero-latency SQLite in DO](https://blog.cloudflare.com/sqlite-in-durable-objects/)

### D1, R2, Hyperdrive, KV
- [D1 docs](https://developers.cloudflare.com/d1/)
- [D1 global read replication (InfoQ)](https://www.infoq.com/news/2025/05/cloudflare-d1-global-replication/)
- [R2 pricing](https://developers.cloudflare.com/r2/pricing/)
- [R2 vs S3](https://www.cloudflare.com/pg-cloudflare-r2-vs-aws-s3/)
- [Hyperdrive MySQL support (Apr 2025)](https://developers.cloudflare.com/changelog/2025-04-08-hyperdrive-mysql-support/)
- [PlanetScale partnership](https://blog.cloudflare.com/planetscale-postgres-workers/)

### Queues & Pipelines
- [Queues docs](https://developers.cloudflare.com/queues/)
- [Queues free plan (Feb 2026)](https://developers.cloudflare.com/changelog/2026-02-04-queues-free-plan/)
- [Pipelines docs](https://developers.cloudflare.com/pipelines/)
- [Streaming ingestion with Arroyo](https://blog.cloudflare.com/cloudflare-acquires-arroyo-pipelines-streaming-ingestion-beta/)
- [10x speedup for Queues](https://blog.cloudflare.com/how-we-built-cloudflare-queues/)

### Workflows
- [Workflows docs](https://developers.cloudflare.com/workflows/)
- [Workflows GA](https://blog.cloudflare.com/workflows-ga-production-ready-durable-execution/)
- [Python Workflows (InfoQ)](https://www.infoq.com/news/2025/11/cloudflare-python-ai-workflows/)

### Browser Rendering
- [Browser Rendering docs](https://developers.cloudflare.com/browser-rendering/)
- [Puppeteer on Workers](https://developers.cloudflare.com/browser-rendering/puppeteer/)
- [Playwright support](https://developers.cloudflare.com/browser-rendering/playwright/)
- [Increased limits (Jan 2025)](https://developers.cloudflare.com/changelog/2025-01-30-browser-rendering-more-instances/)

### Security
- [Post-quantum Zero Trust](https://www.cloudflare.com/press/press-releases/2025/cloudflare-advances-industrys-first-cloud-native-quantum-safe-zero-trust/)
- [State of post-quantum 2025](https://blog.cloudflare.com/pq-2025/)
- [Q4 2025 DDoS report (31.4 Tbps)](https://blog.cloudflare.com/ddos-threat-report-2025-q4/)
- [Q1 2025 DDoS report](https://blog.cloudflare.com/ddos-threat-report-for-2025-q1/)
- [Gartner SSE MQ 2025](https://blog.cloudflare.com/cloudflare-sse-gartner-magic-quadrant-2025/)
- [Gartner SASE MQ 2025](https://blog.cloudflare.com/cloudflare-sase-gartner-magic-quadrant-2025/)
- [AI Crawl Control](https://blog.cloudflare.com/introducing-ai-crawl-control/)

### Secrets Store
- [Secrets Store docs](https://developers.cloudflare.com/secrets-store/)
- [Secrets Store announcement](https://blog.cloudflare.com/secrets-store/)

### Media & Email
- [Cloudflare Media updates](https://blog.cloudflare.com/whats-next-for-cloudflare-media/)
- [Media Transformations beta](https://blog.cloudflare.com/media-transformations-for-video-open-beta/)
- [Email Service](https://blog.cloudflare.com/email-service/)

### Acquisitions
- [Replicate acquisition](https://www.cloudflare.com/press/press-releases/2025/cloudflare-to-acquire-replicate-to-build-the-most-seamless-ai-cloud-for-developers/)
- [Why Replicate joining](https://blog.cloudflare.com/why-replicate-joining-cloudflare/)
- [Astro acquisition](https://www.cloudflare.com/press/press-releases/2026/cloudflare-acquires-astro-to-accelerate-the-future-of-high-performance-web-development/)
- [Astro joins Cloudflare](https://astro.build/blog/joining-cloudflare/)
- [Human Native acquisition](https://www.cloudflare.com/press/press-releases/2026/cloudflare-strengthens-content-offering-to-ai-companies-with-acquisition-of-human-native/)
- [Human Native (CNBC)](https://www.cnbc.com/2026/01/15/cloudflare-ai-human-native-acquisition.html)

### VibeSDK
- [VibeSDK GitHub](https://github.com/cloudflare/vibesdk)
- [VibeSDK blog](https://blog.cloudflare.com/deploy-your-own-ai-vibe-coding-platform/)

### Innovation Weeks
- [Birthday Week 2025 wrap-up](https://blog.cloudflare.com/birthday-week-2025-wrap-up/)
- [Developer Week 2025 wrap-up](https://blog.cloudflare.com/developer-week-2025-wrap-up/)
- [AI Week 2025 wrap-up](https://blog.cloudflare.com/ai-week-2025-wrapup/)
- [Connect 2025](https://events.cloudflare.com/connect/2025/)

### Financial
- [FY2025 Q4 earnings](https://www.cloudflare.com/press/press-releases/2026/cloudflare-announces-fourth-quarter-and-fiscal-year-2025-financial-results/)
- [Q4 2025 analysis (Investing.com)](https://www.investing.com/news/company-news/cloudflare-q4-2025-slides-revenue-surges-34-large-customers-drive-growth-93CH-4498481)
- [Revenue data (MacroTrends)](https://www.macrotrends.net/stocks/charts/NET/cloudflare/revenue)
- [NET stock 16% surge on agentic AI strategy](https://markets.financialcontent.com/stocks/article/marketminute-2026-2-18-edge-of-the-future-cloudflare-shares-skyrocket-16-as-agentic-ai-strategy-ignites-massive-enterprise-demand)

### Competitive
- [Deno vs Workers vs Vercel (Medium)](https://techpreneurr.medium.com/deno-deploy-vs-cloudflare-workers-vs-vercel-edge-functions-which-serverless-platform-wins-in-2025-3affd9c7f45e)
- [Lambda@Edge vs Workers vs Vercel](https://prabhatgiri.com/blogs/lambdaedge-vs-cloudflare-workers-vs-vercel-edge-latency-limits-and-cost-in-2025/)
- [Workers alternatives 2026 (Northflank)](https://northflank.com/blog/best-cloudflare-workers-alternatives)
- [Vercel vs Cloudflare deep dive (SparkCo)](https://sparkco.ai/blog/vercel-vs-cloudflare-edge-deployment-deep-dive)
- [Fluid Compute benchmarks (Vercel)](https://vercel.com/blog/fluid-compute-benchmark-results)

### Students & Startups
- [Free Workers for students](https://blog.cloudflare.com/workers-for-students/)
- [Cloudflare for Students](https://www.cloudflare.com/students/)
- [Startup hubs announcement](https://blog.cloudflare.com/new-hubs-for-startups/)
- [1,111 internships in 2026](https://www.cloudflare.com/press/press-releases/2025/cloudflare-aims-to-hire-1111-interns-in-2026-to-help-train-the-next-gen/)

### Open Source
- [Pingora open source](https://blog.cloudflare.com/pingora-open-source/)
- [Pingora GitHub](https://github.com/cloudflare/pingora)
- [workerd open source](https://blog.cloudflare.com/workerd-open-source-workers-runtime/)
- [Cloudflare GitHub org](https://github.com/cloudflare)

### Strategy & General
- [Connectivity Cloud](https://www.cloudflare.com/connectivity-cloud/)
- [Cloudflare vs AWS/Azure/GCP (InfoWorld)](https://www.infoworld.com/article/2336185/how-cloudflare-emerged-to-take-on-aws-azure-and-google-cloud.html)
- [Cloudflare product portfolio](https://www.cloudflare.com/cloudflare-product-portfolio/)
- [Developer platform products](https://www.cloudflare.com/developer-platform/products/)
- [Developer platform updates (Feb 2026)](https://blog.cloudflare.com/cloudflare-developer-platform-keeps-getting-better-faster-and-more-powerful/)
