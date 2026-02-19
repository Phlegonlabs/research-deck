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

## 结论

**MCP + x402 是第一个实用的 "代理为服务付费" 技术栈。** 真实存在、已经上线、今天在测试网和主网上都能用。

**诚实图景**：它解决了每次交易的摩擦（无账户、无 API 密钥、即时支付）。它**没有**解决引导问题（仍需人类为钱包注资）。"自主代理经济" 对机器对机器后端支付是真的，但面向消费者的代理商务出于监管原因可能会用法币轨道（AP2/ACP）。

**今天应该构建的**：如果你有 API 或 MCP 工具，添加 x402 支付大约 10 行中间件。如果你在构建代理，`@x402/fetch` 或 `@x402/axios` 给你的 HTTP 客户端包装自动支付。基础设施已足够成熟，可在 Base 主网上生产使用。
