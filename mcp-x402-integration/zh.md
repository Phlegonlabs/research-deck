# MCP + x402：AI 代理如何自主付费

## 这是什么？

MCP（模型上下文协议）给 AI 代理**使用工具**的能力。x402 给它们**为工具付费**的能力。两者结合创造了第一个实用的基础设施：AI 代理可以自主发现服务、付费、使用 — 无需人工介入。

这不是理论。Coinbase 的 x402 已处理 1 亿+ 笔支付。Cloudflare、Vercel、Coinbase 已发布生产级集成。Claude Desktop、ChatGPT、Cursor 今天就能使用付费 MCP 工具。

## 核心问题

传统 API 访问需要人类：
1. 创建账户（邮箱、密码）
2. 添加支付方式（信用卡、KYC）
3. 订阅或购买额度
4. 生成 API 密钥
5. 配置认证

AI 代理无法自主完成任何一步。x402 通过将支付直接嵌入 HTTP 来消除整个流程。

## x402 协议工作原理

x402 复活了 HTTP 402 "Payment Required" 状态码 — 1997 年定义，直到现在才被标准化。

### 支付握手流程

```
Client                    Server                   Facilitator        Blockchain
  |                         |                          |                  |
  |--- GET /resource ------>|                          |                  |
  |<-- 402 + PAYMENT-       |                          |                  |
  |    REQUIRED header -----|                          |                  |
  |                         |                          |                  |
  | [签名支付 payload]       |                          |                  |
  |                         |                          |                  |
  |--- GET /resource ------>|                          |                  |
  |    + PAYMENT-SIGNATURE  |                          |                  |
  |    header               |--- POST /verify -------->|                  |
  |                         |<-- verified -------------|                  |
  |                         |--- POST /settle -------->|                  |
  |                         |                          |--- tx ---------> |
  |                         |                          |<-- confirmed --- |
  |                         |<-- settled --------------|                  |
  |<-- 200 OK + PAYMENT-    |                          |                  |
  |    RESPONSE header -----|                          |                  |
```

### 三个阶段

| 阶段 | 发生什么 | HTTP |
|------|---------|------|
| 报价 | 客户端请求资源，服务端返回价格 | `402` + `PAYMENT-REQUIRED` header |
| 支付 | 客户端签名支付，携带证明重试 | 请求 + `PAYMENT-SIGNATURE` header |
| 结算 | Facilitator 验证，链上结算，服务端交付 | `200` + `PAYMENT-RESPONSE` header |

### 关键设计决策

**HTTP 原生**：无额外请求、无 WebSocket、无旁路通道。支付搭载在标准 HTTP 头上。

**Facilitator 模式**：服务端不需要区块链基础设施。Facilitator（Coinbase、Cloudflare 或自托管）处理验证和链上结算。这抽象掉了 gas、RPC 端点和网络细节。

**EIP-3009（Transfer With Authorization）**：底层加密原语。Circle 在 2020 年提出，允许预签名代币转账。目前只有 USDC 原生支持 — 这就是 x402 运行在 USDC 上的原因。

**Scheme 可扩展**：`exact`（固定价格/次）已上线。`upto`（基于用量上限）和 `deferred`（批量结算）在 V2 中。

### V2 改进（已发布）

| 特性 | V1 | V2 |
|------|----|----|
| Headers | `X-PAYMENT-*`（已弃用） | `PAYMENT-REQUIRED`, `PAYMENT-SIGNATURE`, `PAYMENT-RESPONSE` |
| 会话 | 无 — 每次调用都付费 | 基于钱包的会话（`@x402/paywall`） |
| 发现 | 手动 | Facilitator 自动索引 |
| Schemes | 仅 `exact` | `exact` + `deferred` + 可扩展 |
| 链 | Base | Base, Solana, Ethereum, L2s |
| SDK | 单体 | 模块化 `@x402/*` 包 |

## MCP + x402 如何集成

### 架构

```
┌──────────────────────────────────────────────────┐
│                  AI 代理 (Claude/ChatGPT)          │
│                                                   │
│   "查东京天气"                                     │
│          │                                        │
│          ▼                                        │
│   ┌─────────────┐    ┌────────────────────┐      │
│   │  MCP Client  │───>│   x402 拦截器       │      │
│   └─────────────┘    │   (@x402/axios 或   │      │
│                       │    @x402/fetch)     │      │
│                       └────────┬───────────┘      │
│                                │                   │
│                       ┌────────▼───────────┐      │
│                       │   加密钱包           │      │
│                       │   (EVM 或 Solana)   │      │
│                       └────────────────────┘      │
└──────────────────────┬────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│                  MCP Server                       │
│                                                   │
│   tool("free_tool")  ──> 无需付费                  │
│   paidTool("weather") ──> $0.001/次               │
│          │                                        │
│          ▼                                        │
│   ┌─────────────────────┐                        │
│   │  x402 中间件          │                        │
│   │  (paymentMiddleware) │                        │
│   └──────────┬──────────┘                        │
│              │                                    │
│              ▼                                    │
│   ┌─────────────────────┐                        │
│   │  资源服务器            │                        │
│   │  (实际 API 逻辑)      │                        │
│   └─────────────────────┘                        │
└──────────────────────────────────────────────────┘
```

### 四种集成模式

#### 模式 1：Coinbase SDK（官方）

MCP server 使用 `@x402/axios` 包装付费 API。Wrapper 自动处理 402 握手。

**服务端：**
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { x402Client, wrapAxiosWithPayment } from "@x402/axios";
import { registerExactEvmScheme } from "@x402/evm/exact/client";

const client = new x402Client();
registerExactEvmScheme(client, { signer: privateKeyToAccount(key) });
const http = wrapAxiosWithPayment(axios.create({ baseURL }), client);

server.tool("get-weather", "Paid weather data", {}, async () => {
  const res = await http.get("/weather"); // 402 自动处理
  return { content: [{ type: "text", text: JSON.stringify(res.data) }] };
});
```

**配置（Claude Desktop `claude_desktop_config.json`）：**
```json
{
  "mcpServers": {
    "weather": {
      "command": "pnpm",
      "args": ["--silent", "-C", "/path/to/mcp-server", "dev"],
      "env": {
        "EVM_PRIVATE_KEY": "0x...",
        "RESOURCE_SERVER_URL": "http://localhost:4021",
        "ENDPOINT_PATH": "/weather"
      }
    }
  }
}
```

#### 模式 2：Vercel x402-mcp（paidTool）

Vercel 的封装直接给 MCP server 加了 `paidTool()` 方法。更干净的 API — 价格和工具一起声明。

**服务端：**
```typescript
server.paidTool(
  "add_numbers",
  { price: 0.001 },
  { a: z.number(), b: z.number() },
  async (args) => ({
    content: [{ type: "text", text: String(args.a + args.b) }]
  })
);
```

**客户端：**
```typescript
const mcpClient = await createMCPClient({
  transport: new StreamableHTTPClientTransport(url),
}).then((client) => withPayment(client, { account }));
```

#### 模式 3：Cloudflare Workers + Agents

Workers 路由上的支付中间件。Agent 使用 `fetchWithPay` 封装。

**服务端（Hono on Workers）：**
```typescript
app.use(
  paymentMiddleware(
    process.env.SERVER_ADDRESS,
    {
      "/weather": {
        price: "$0.10",
        network: "base-sepolia",
        config: { description: "Weather data" },
      },
    },
    { url: "https://x402.org/facilitator" }
  )
);
```

**Agent：**
```typescript
const account = privateKeyToAccount(env.PRIVATE_KEY);
const pay = fetchWithPay(account);
const res = await pay("https://api.example.com/weather");
```

#### 模式 4：MCPay（第三方代理）

MCPay 作为代理坐在任何现有 MCP server 前面。上游零代码改动。

```bash
npx mcpay connect --urls https://existing-mcp-server.com --api-key YOUR_KEY
```

或通过 SDK：
```typescript
server.paidTool("weather", "Paid tool", "$0.001",
  { city: z.string() }, {},
  async ({ city }) => ({ /* ... */ })
);
```

## 钱包层：Coinbase Agentic Wallets

代理需要钱包来付费。Coinbase 构建了第一个专为 AI 代理设计的钱包基础设施。

### 架构

```
┌─────────────────────────────────┐
│          AI 代理 (LLM)           │
│    (永远看不到私钥)               │
│              │                   │
│    ┌─────────▼──────────┐       │
│    │   本地会话密钥        │       │
│    │   + 消费限额         │       │
│    └─────────┬──────────┘       │
└──────────────┼──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│    可信执行环境 (TEE)             │
│                                  │
│    ┌──────────────────────┐     │
│    │   私钥存储              │     │
│    │   (永不导出到            │     │
│    │    代理/LLM 上下文)     │     │
│    └──────────────────────┘     │
│                                  │
│    ┌──────────────────────┐     │
│    │   安全护栏              │     │
│    │   • 会话消费上限        │     │
│    │   • 单笔交易限额        │     │
│    │   • KYT 交易审查        │     │
│    │   • 策略检查            │     │
│    └──────────────────────┘     │
└─────────────────────────────────┘
```

### 安全模型

| 控制 | 作用 |
|------|------|
| TEE 隔离 | 私钥存储在硬件隔离区，永不进入 LLM 上下文 |
| 会话上限 | 每个代理会话的最大总消费 |
| 单笔限额 | 单次交易的最大金额 |
| KYT（了解你的交易） | 阻止高风险交互 |
| 非托管 | 用户保留密钥所有权，可导出 |

关键洞察：**代理永远看不到私钥**。TEE 代表代理签名交易，受人类所有者设置的消费限额约束。如果 LLM 被注入恶意提示，最坏情况被会话限额封顶。

## 协议栈全景

MCP + x402 在更广泛的代理网络中处于什么位置？

| 层 | 协议 | 角色 |
|----|------|------|
| 连接 | **MCP** | 代理 ↔ 工具（发现、调用） |
| 编排 | **A2A**（Google） | 代理 ↔ 代理（委托、协调） |
| 支付（加密） | **x402** | HTTP 原生稳定币微支付 |
| 支付（法币） | **AP2**（Google） | 通过传统轨道的法币支付 |
| 支付（卡） | **ACP** | 为代理适配的信用卡基础设施 |
| 身份/信任 | **ERC-8004** | 链上代理声誉、质押 |

"鲻鱼经济"论点：**前端法币（B2C），后端加密（B2B/M2M）**。面向消费者的代理交易用 AP2/ACP（熟悉、受监管）。机器对机器的后端自动化用 x402（即时、无账户、可编程）。

## 权衡与问题

### 有效的部分

| 优势 | 详情 |
|------|------|
| 零摩擦接入 | 无账户、无 API 密钥、机器对机器无需 KYC |
| 即时结算 | 链上最终性在秒级（Base） |
| 开放标准 | 尽管 Coinbase 深度参与，但不锁定 |
| MCP 兼容 | 与 Claude、ChatGPT、Cursor 及任何 MCP 客户端兼容 |
| 微支付可行 | $0.001/次是实际可行的（Base L2 gas 可忽略） |

### 尚未解决的问题

| 问题 | 严重性 | 详情 |
|------|--------|------|
| 仅 USDC | 高 | EIP-3009 仅 USDC 支持。其他代币无法使用 x402 的信任最小化设计 |
| Facilitator 中心化 | 高 | 大多数实现使用 Coinbase 的 facilitator。"开放标准" 一个看门人 |
| 代理无法法币入金 | 高 | 代理需要 USDC。获取 USDC 仍需人类在链条某处完成 KYC |
| 重试/争议处理 | 中 | V1 无重试逻辑或争议解决。V2 改进中但仍薄弱 |
| 人工确认 UX | 中 | 当前 MCP 客户端会提示人类确认支付。尚非真正自主 |
| 钱包密钥管理 | 中 | 即使有 TEE，谁配置钱包？谁注资？仍是人类引导 |

### 中心化讽刺

x402 是 "开放标准" 但基础设施由 Coinbase 主导：
- Coinbase 构建了 SDK
- Coinbase 运营主要 facilitator
- Coinbase 提供钱包基础设施（AgentKit, Agentic Wallets）
- Coinbase 与 Cloudflare 联合创立 x402 Foundation

这是 Web3 模式重演：去中心化协议，中心化实现。

### 引导问题

代理要用 x402 支付，需要人类：
1. 创建钱包（需要人类）
2. 充入 USDC（需要人类 + KYC 交易所）
3. 配置代理的钱包访问权限（需要人类）
4. 设定消费限额（需要人类）

x402 移除了每次交易的人工介入，但初始设置仍完全依赖人类。"完全自主代理经济" 的叙事在引导阶段就崩溃了。

## 替代方案

| 替代方案 | 方法 | 权衡 |
|----------|------|------|
| Stripe MCP | 通过 MCP 的传统支付轨道 | 需要账户，但熟悉且受监管 |
| Google AP2 | 带加密授权的法币代理支付 | 更企业友好，更少加密原生 |
| 普通 API 密钥 | 预配置访问，无按次付费 | 无自主权，但简单且现在就能用 |
| OAuth + 计费 | 标准 web 认证 + 使用量计费 | 人工管理，但久经考验 |
| ACP（代理商务协议） | 为代理适配的信用卡基础设施 | 熟悉的轨道，但仍需发卡 |

## 可偷的模式

| 模式 | 内容 | 用途 |
|------|------|------|
| HTTP 402 握手 | 支付嵌入标准请求-响应 | 任何 API 变现 — 无需账户系统 |
| `paidTool()` 装饰器 | 价格与工具定义一起声明 | 一行代码让 MCP 工具收费 |
| Facilitator 抽象 | 支付验证与业务逻辑分离 | 服务端不需要区块链知识 |
| TEE 钱包隔离 | 私钥永不进入 LLM 上下文 | 有界风险的安全代理自主权 |
| 基于会话的访问 | 付一次，通过 `@x402/paywall` 获取会话令牌 | 减少高频使用的每次调用开销 |
| 延迟结算 | 聚合微支付，批量结算 | 高量低值调用的成本效率 |
| 代理变现（MCPay） | 用支付层包装现有 MCP server | 不动上游代码即可变现 |

## 实战入门

### 最快路径：Coinbase x402 + MCP

```bash
git clone https://github.com/coinbase/x402.git
cd x402/examples/typescript
pnpm install && pnpm build

# 启动演示付费 API 服务
cd servers/express && pnpm dev

# 在另一个终端，配置 Claude Desktop 的 MCP server
# 将模式 1 的配置添加到 claude_desktop_config.json
```

要求：
- Node.js v20+
- 在 Base Sepolia（测试网）或 Base Mainnet 上有 USDC 的钱包
- 启用 MCP 的 Claude Desktop

### 最快路径：给现有 MCP Server 收费

```bash
npx mcpay connect --urls https://your-mcp-server.com --wallet-key 0x...
```

MCPay 作为代理包装你的 server — 零代码改动。

## 最新動態 (2026)

### MCP 捐赠给 Agentic AI Foundation（2025 年 12 月）

Anthropic 将 MCP 捐赠给 Linux Foundation 下新成立的 **Agentic AI Foundation (AAIF)**。OpenAI 和 Block 作为联合创始人加入，AWS、Google、Microsoft、Cloudflare、Bloomberg 作为白金会员。这是重大治理转变：MCP 不再是 Anthropic 的项目 — 它现在是由中立治理的行业级开放标准。OpenAI 贡献了 AGENTS.md（已被 60,000+ 开源项目采用），Block 贡献了 goose（开源本地优先 AI 代理框架）。

### MCP 规范 2025-11-25：异步任务和 URL Elicitation

2025 年 11 月的规范发布引入了两个关键原语：

- **Tasks（任务）**：异步、长时间运行操作的新抽象。任何请求现在都可以变成"立即调用，稍后获取" — 服务端立即返回任务元数据，客户端轮询结果。任务在状态间流转（`working`、`input_required`、`completed`、`failed`、`cancelled`），使得仅靠同步工具调用无法实现的多步工作流成为可能。
- **URL Mode Elicitation（SEP-1036）**：服务端可以发送 URL 将用户重定向到浏览器进行敏感操作（OAuth、支付、API 密钥配置），而非在 MCP 客户端内处理凭证。这与 x402 直接相关 — 支付确认现在可以在合适的浏览器上下文中进行，而不是硬塞进代理 UX。

### x402 Facilitator 去中心化：Dexter 超越 Coinbase

截至 2026 年 1 月，**Dexter 已超越 Coinbase 成为最大的每日 x402 Facilitator**，处理约 50% 的每日交易量。四个 Facilitator 各处理超过 1000 万笔交易：Coinbase、Dexter、PayAI、DayDreams。自 2025 年 10 月以来，Base 链以 6800 万+ 笔 x402 总交易量领先所有链。这直接回应了前文提到的"中心化讽刺" — Facilitator 层正在真正多元化，尽管 Base（Coinbase 的 L2）仍主导结算链。

### Google AP2 + x402 扩展：法币与加密汇聚

Google 发布了 **A2A x402 Extension**，作为其 Agent-to-Agent (A2A) 协议与 x402 加密支付之间的生产级桥梁。AP2 是支付方式无关的（卡、银行转账、稳定币），x402 扩展让代理在 AP2 的 Intent Mandate 框架下用稳定币结算 API 调用。Google 召集了 60+ 启动合作伙伴，包括 Mastercard、Adyen、PayPal 和 Coinbase。"鲻鱼经济"论点现在是实际基础设施：AP2 处理法币前端，x402 处理加密后端。

### Browserbase x402 端点：代理付费浏览网页

Browserbase 推出 x402 端点，让 AI 代理**自主购买浏览器会话**。代理可以打开 Chrome、导航网站、点击按钮、填写表单、读取页面 — 在 Base 上用 USDC 按会话付费。无需 API 密钥、无需账户、无需计费门户。这是 x402 最早的非 API 用例之一：代理不仅在调用数据端点，它们在按需购买计算资源（无头浏览器时间）。

### XRP Ledger x402 Facilitator（2026 年 2 月）

t54.ai 在 **XRP Ledger** 上启动了 x402 facilitator，使 AI 代理能够使用 XRP 和 RLUSD（Ripple 的美元稳定币）支付。这打破了 x402 对 EVM 兼容链和 USDC 的依赖。该 facilitator 处理付款人签名的预签名 Payment 交易 blob，支持 XRP 和 IOU 代币。已在 BlockRunAI 上投入生产 — 这是一个统一网关，为代理提供 30+ LLM 模型（GPT、Claude、Grok）的访问权限，通过 x402 按请求付费在 XRPL 上结算。

### x402 交易量激增

自 2025 年 9 月协议发布以来，x402 总交易量已飙升至数千万笔。已处理超过 1 亿笔支付，覆盖 API、应用和 AI 代理。每周交易量已达 500,000+。生态系统现在跨越多条链（Base、Solana、Ethereum、XRPL、BNB Chain）和多个 facilitator，验证了协议在测试网实验之外的生产就绪性。

## 结论

**MCP + x402 是第一个实用的 "代理为服务付费" 技术栈。** 真实存在、已经上线、今天在测试网和主网上都能用。

**诚实图景**：它解决了每次交易的摩擦（无账户、无 API 密钥、即时支付）。它**没有**解决引导问题（仍需人类为钱包注资）。"自主代理经济" 对机器对机器后端支付是真的，但面向消费者的代理商务出于监管原因可能会用法币轨道（AP2/ACP）。

**今天应该构建的**：如果你有 API 或 MCP 工具，添加 x402 支付大约 10 行中间件。如果你在构建代理，`@x402/fetch` 或 `@x402/axios` 给你的 HTTP 客户端包装自动支付。基础设施已足够成熟，可在 Base 主网上生产使用。

## References

### x402 协议核心

- [x402.org — 官方网站](https://www.x402.org/) — 协议规范、生态系统、基金会信息
- [x402 V2 发布公告](https://www.x402.org/writing/x402-v2-launch) — V2 变更：会话、发现、延迟结算、模块化 SDK
- [x402 白皮书 (PDF)](https://www.x402.org/x402-whitepaper.pdf) — 完整协议规范
- [coinbase/x402 — GitHub](https://github.com/coinbase/x402) — 参考实现（TypeScript, Python, Go, Java）
- [x402 协议详解 — QuickNode](https://blog.quicknode.com/x402-protocol-explained-inside-the-https-native-payment-layer/) — 技术深潜：EIP-3009、facilitator 角色、结算
- [什么是 x402？ — Solana](https://solana.com/x402/what-is-x402) — Solana 端 x402 说明
- [什么是 x402？ — Thirdweb](https://blog.thirdweb.com/what-is-x402-protocol-the-http-based-payment-standard-for-onchain-commerce/) — 总体概述

### MCP + x402 集成

- [MCP Server with x402 — Coinbase 文档](https://docs.cdp.coinbase.com/x402/mcp-server) — 官方指南：MCP server 配置 x402 支付
- [x402-mcp: MCP 工具的开放协议支付 — Vercel](https://vercel.com/blog/introducing-x402-mcp-open-protocol-payments-for-mcp-tools) — Vercel 的 `paidTool()` 封装
- [x402 · Cloudflare Agents 文档](https://developers.cloudflare.com/agents/x402/) — Workers + Agents x402 集成
- [用 x402 实现自主 API 和 MCP Server 支付 — Zuplo](https://zuplo.com/blog/mcp-api-payments-with-x402) — 实现示例、服务端/客户端代码
- [MCP Server with x402 — x402 Gitbook](https://x402.gitbook.io/x402/guides/mcp-server-with-x402) — 分步指南
- [x402 MCP 示例 — GitHub](https://github.com/coinbase/x402/tree/main/examples/typescript/mcp) — 官方示例代码

### MCPay（第三方）

- [MCPay — GitHub](https://github.com/microchipgnu/MCPay) — 为任何 MCP server 添加 x402 支付的开源代理
- [mcp-go-x402 — GitHub](https://github.com/mark3labs/mcp-go-x402) — MCP 的 x402 传输 Go 实现

### 钱包基础设施

- [Agentic Wallets — Coinbase](https://www.coinbase.com/developer-platform/discover/launches/agentic-wallets) — TEE 架构、消费限额、安全模型
- [Agentic Wallet 文档 — Coinbase](https://docs.cdp.coinbase.com/agentic-wallet/welcome) — 技术文档
- [AgentKit — GitHub](https://github.com/coinbase/agentkit) — "每个 AI 代理都值得拥有一个钱包" — 框架无关的代理钱包工具包
- [AgentKit MCP 扩展 — Coinbase 文档](https://docs.cdp.coinbase.com/agent-kit/core-concepts/model-context-protocol) — AgentKit 的 MCP 集成
- [Coinbase 推出 AI 代理加密钱包基础设施 — PYMNTS](https://www.pymnts.com/cryptocurrency/2026/coinbase-debuts-crypto-wallet-infrastructure-for-ai-agents/)
- [Coinbase 推出"给任何代理一个钱包"AI 工具 — The Block](https://www.theblock.co/post/389524/coinbase-rolls-out-ai-tool-to-give-any-agent-a-wallet)

### x402 基金会与合作

- [与 Coinbase 联合启动 x402 基金会 — Cloudflare 博客](https://blog.cloudflare.com/x402/) — 基金会使命、延迟支付方案、Workers 集成
- [Cloudflare 与 Coinbase 启动 x402 基金会 — The Block](https://www.theblock.co/post/372064/cloudflare-coinbase-launch-x402-foundation)
- [x402 生态系统](https://www.x402.org/ecosystem) — 完整生态系统图

### 协议全景

- [MCP, A2A, AP2, ACP, x402 & ERC-8004 详解 — PayRam](https://payram.com/blog/mcp-a2a-ap2-acp-x402-erc-8004) — 完整协议栈对比
- [从 DeFi Summer 到 x402 Summer — Medium](https://medium.com/@a6b8/from-defi-summer-to-x402-summer-mcp-x402-and-the-fragmented-web-94faa1c5ffb7) — MCP + x402 融合分析
- [x402 + AnChain.AI: 解锁 Agentic AI 支付信任](https://www.anchain.ai/blog/x402) — 安全视角
- [x402 V2 升级 — InfoQ](https://www.infoq.com/news/2026/01/x402-agentic-http-payments/) — V2 发布行业报道

### 社区实现

- [P-Link-MCP — GitHub](https://github.com/paracetamol951/P-Link-MCP) — x402 客户端 + HTTP 402 服务端 + P-Link.io 的 MCP
- [x402-playground — Morpheus](https://github.com/MorpheusAIs/x402-playground) — 实验性 AI 聊天 + x402 API + MCP server
- [rome-x402-mcp — GitHub](https://github.com/xarmian/rome-x402-mcp) — 通过 x402 + MCP 的跨链代币桥接
- [tip-md-x402-mcp-server — GitHub](https://github.com/xR0am/tip-md-x402-mcp-server) — x402 加密打赏 MCP server
- [@civic/x402-mcp — npm](https://www.npmjs.com/package/@civic/x402-mcp) — Civic 的 x402 MCP 实现

### 最新动态来源 (2026)

- [Linux Foundation 宣布成立 AAIF](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation) — MCP 捐赠给 Agentic AI Foundation
- [Anthropic：捐赠 MCP 并成立 AAIF](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation) — Anthropic 的公告
- [MCP 规范 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) — 2025 年 11 月规范：Tasks、URL elicitation
- [MCP 2025-11-25 规范更新 — WorkOS](https://workos.com/blog/mcp-2025-11-25-spec-update) — 异步 Tasks、OAuth 改进
- [Dexter 超越 Coinbase 成为顶级 x402 Facilitator — The Coin Republic](https://www.thecoinrepublic.com/2026/01/02/dexter-overtakes-coinbase-as-top-x402-facilitator/) — Facilitator 去中心化
- [Dexter 超越 Coinbase 成为顶级 x402 交易 Facilitator — ainvest](https://www.ainvest.com/news/dexter-overtakes-coinbase-top-x402-transaction-facilitator-2601/) — 市场份额数据
- [Google AP2 + x402：代理现在可以互相支付 — Coinbase](https://www.coinbase.com/developer-platform/discover/launches/google_x402) — AP2/x402 集成
- [A2A x402 Extension — GitHub](https://github.com/google-agentic-commerce/a2a-x402) — Google 的 A2A x402 开源扩展
- [Browserbase 通过 x402 启用代理浏览器会话 — Coinbase](https://www.coinbase.com/en-fr/developer-platform/discover/launches/browserbase-x402-usdc) — 通过 x402 的浏览器会话
- [Browserbase + Coinbase x402 — Browserbase 博客](https://www.browserbase.com/blog/browserbase-and-coinbase-x402) — 技术细节
- [XRP Ledger 获得 x402 Facilitator — CryptBull](https://www.cryptbull.net/2026/02/21/xrp-ledger-gets-x402-facilitator-for-ai-agent-payments-why-this-is-bullish/) — XRPL x402 支持
- [AI 代理现可通过 x402 使用 XRP 和 RLUSD 支付 — Coinpedia](https://coinpedia.org/news/xrp-ledger-news-today-ai-agents-can-now-pay-with-xrp-and-rlusd-via-x402/) — XRP/RLUSD 支付
- [Coinbase 扩展稳定币 AI 代理支付工具覆盖范围 — CoinDesk](https://www.coindesk.com/tech/2025/12/11/coinbase-expands-the-reach-of-its-stablecoin-based-ai-agent-payments-tool) — V2 扩展报道
- [x402 孵化的 V2 推出 — The Block](https://www.theblock.co/post/382284/coinbase-incubated-x402-payments-protocol-built-for-ais-rolls-out-v2) — V2 发布报道
