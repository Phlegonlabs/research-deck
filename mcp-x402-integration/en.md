# MCP + x402: How AI Agents Pay for Services

## What Is This?

MCP (Model Context Protocol) gives AI agents the ability to **use tools**. x402 gives them the ability to **pay for tools**. Together they create the first practical infrastructure where an AI agent can autonomously discover a service, pay for it, and use it — no human in the loop.

This isn't theoretical. Coinbase's x402 has processed 100M+ payments. Cloudflare, Vercel, and Coinbase have shipped production integrations. Claude Desktop, ChatGPT, and Cursor can all use paid MCP tools today.

## The Core Problem

Traditional API access requires humans to:
1. Create accounts (email, password)
2. Add payment methods (credit card, KYC)
3. Subscribe or buy credits
4. Generate API keys
5. Configure authentication

An AI agent can't do any of this autonomously. x402 eliminates the entire flow by embedding payment directly into HTTP.

## How x402 Works (The Protocol)

x402 revives the HTTP 402 "Payment Required" status code — defined in 1997, never standardized until now.

### The Payment Handshake

```
Client                    Server                   Facilitator        Blockchain
  |                         |                          |                  |
  |--- GET /resource ------>|                          |                  |
  |<-- 402 + PAYMENT-       |                          |                  |
  |    REQUIRED header -----|                          |                  |
  |                         |                          |                  |
  | [sign payment payload]  |                          |                  |
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

### Three Phases

| Phase | What Happens | HTTP |
|-------|-------------|------|
| Quote | Client requests resource, server responds with price | `402` + `PAYMENT-REQUIRED` header |
| Pay | Client signs payment, retries with proof | Request + `PAYMENT-SIGNATURE` header |
| Settle | Facilitator verifies, settles on-chain, server delivers | `200` + `PAYMENT-RESPONSE` header |

### Key Design Decisions

**HTTP-native**: No extra requests, no WebSocket, no side channels. Payment rides on standard HTTP headers.

**Facilitator pattern**: Servers don't need blockchain infrastructure. A facilitator (Coinbase, Cloudflare, self-hosted) handles verification and on-chain settlement. This abstracts away gas, RPC endpoints, and network specifics.

**EIP-3009 (Transfer With Authorization)**: The cryptographic primitive underneath. Proposed by Circle in 2020, it allows pre-signed token transfers. Currently only USDC natively supports this — which is why x402 runs on USDC.

**Scheme-extensible**: `exact` (fixed price per call) ships today. `upto` (usage-based ceiling) and `deferred` (batch settlement) are in V2.

### V2 Improvements (Shipped)

| Feature | V1 | V2 |
|---------|----|----|
| Headers | `X-PAYMENT-*` (deprecated) | `PAYMENT-REQUIRED`, `PAYMENT-SIGNATURE`, `PAYMENT-RESPONSE` |
| Sessions | None — pay every call | Wallet-based sessions via `@x402/paywall` |
| Discovery | Manual | Auto-indexing by facilitators |
| Schemes | `exact` only | `exact` + `deferred` + extensible |
| Chains | Base | Base, Solana, Ethereum, L2s |
| SDK | Monolithic | Modular `@x402/*` packages |

## How MCP + x402 Integrate

### Architecture

```
┌──────────────────────────────────────────────────┐
│                  AI Agent (Claude/ChatGPT)        │
│                                                   │
│   "Get weather for Tokyo"                         │
│          │                                        │
│          ▼                                        │
│   ┌─────────────┐    ┌────────────────────┐      │
│   │  MCP Client  │───>│   x402 Interceptor │      │
│   └─────────────┘    │   (@x402/axios or   │      │
│                       │    @x402/fetch)     │      │
│                       └────────┬───────────┘      │
│                                │                   │
│                       ┌────────▼───────────┐      │
│                       │   Crypto Wallet     │      │
│                       │   (EVM or Solana)   │      │
│                       └────────────────────┘      │
└──────────────────────┬────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│                  MCP Server                       │
│                                                   │
│   tool("free_tool")  ──> no payment needed        │
│   paidTool("weather") ──> $0.001 per call         │
│          │                                        │
│          ▼                                        │
│   ┌─────────────────────┐                        │
│   │  x402 Middleware     │                        │
│   │  (paymentMiddleware) │                        │
│   └──────────┬──────────┘                        │
│              │                                    │
│              ▼                                    │
│   ┌─────────────────────┐                        │
│   │  Resource Server     │                        │
│   │  (actual API logic)  │                        │
│   └─────────────────────┘                        │
└──────────────────────────────────────────────────┘
```

### Three Integration Patterns

#### Pattern 1: Coinbase SDK (Official)

MCP server wraps a paid API using `@x402/axios`. The wrapper automatically handles the 402 handshake.

**Server side:**
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { x402Client, wrapAxiosWithPayment } from "@x402/axios";
import { registerExactEvmScheme } from "@x402/evm/exact/client";

const client = new x402Client();
registerExactEvmScheme(client, { signer: privateKeyToAccount(key) });
const http = wrapAxiosWithPayment(axios.create({ baseURL }), client);

server.tool("get-weather", "Paid weather data", {}, async () => {
  const res = await http.get("/weather"); // 402 handled automatically
  return { content: [{ type: "text", text: JSON.stringify(res.data) }] };
});
```

**Config (Claude Desktop `claude_desktop_config.json`):**
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

#### Pattern 2: Vercel x402-mcp (paidTool)

Vercel's wrapper adds a `paidTool()` method directly to MCP server definition. Cleaner API — price declared alongside the tool.

**Server side:**
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

**Client side:**
```typescript
const mcpClient = await createMCPClient({
  transport: new StreamableHTTPClientTransport(url),
}).then((client) => withPayment(client, { account }));
```

#### Pattern 3: Cloudflare Workers + Agents

Payment middleware on Workers routes. Agents use `fetchWithPay` wrapper.

**Server (Hono on Workers):**
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

**Agent:**
```typescript
const account = privateKeyToAccount(env.PRIVATE_KEY);
const pay = fetchWithPay(account);
const res = await pay("https://api.example.com/weather");
```

#### Pattern 4: MCPay (Third-party Proxy)

MCPay sits as a proxy in front of any existing MCP server. Zero code changes to upstream — just wrap it.

```bash
npx mcpay connect --urls https://existing-mcp-server.com --api-key YOUR_KEY
```

Or via SDK:
```typescript
server.paidTool("weather", "Paid tool", "$0.001",
  { city: z.string() }, {},
  async ({ city }) => ({ /* ... */ })
);
```

## The Wallet Layer: Coinbase Agentic Wallets

Agents need wallets to pay. Coinbase built the first wallet infrastructure specifically for AI agents.

### Architecture

```
┌─────────────────────────────────┐
│          AI Agent (LLM)          │
│    (never sees private key)      │
│              │                   │
│    ┌─────────▼──────────┐       │
│    │   Local Session Key │       │
│    │   + Spending Limits │       │
│    └─────────┬──────────┘       │
└──────────────┼──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│    Trusted Execution Environment │
│            (TEE)                 │
│                                  │
│    ┌──────────────────────┐     │
│    │   Private Key Storage │     │
│    │   (never exported to  │     │
│    │    agent/LLM context) │     │
│    └──────────────────────┘     │
│                                  │
│    ┌──────────────────────┐     │
│    │   Guardrails          │     │
│    │   • Session caps      │     │
│    │   • Per-tx limits     │     │
│    │   • KYT screening     │     │
│    │   • Policy checks     │     │
│    └──────────────────────┘     │
└─────────────────────────────────┘
```

### Security Model

| Control | What It Does |
|---------|-------------|
| TEE isolation | Private keys stored in hardware-isolated enclave, never in LLM context |
| Session caps | Maximum total spend per agent session |
| Per-tx limits | Maximum per single transaction |
| KYT (Know Your Transaction) | Blocks high-risk interactions |
| Non-custodial | User retains key ownership, can export |

The critical insight: **the agent never sees the private key**. The TEE signs transactions on the agent's behalf, bounded by spending limits set by the human owner. If the LLM is prompt-injected, the worst case is capped by session limits.

## The Protocol Stack

Where does MCP + x402 fit in the broader agentic web?

| Layer | Protocol | Role |
|-------|----------|------|
| Connectivity | **MCP** | Agent ↔ Tools (discover, invoke) |
| Orchestration | **A2A** (Google) | Agent ↔ Agent (delegate, coordinate) |
| Payment (crypto) | **x402** | HTTP-native stablecoin micropayments |
| Payment (fiat) | **AP2** (Google) | Fiat payments via traditional rails |
| Payment (cards) | **ACP** | Credit card infra adapted for agents |
| Identity/Trust | **ERC-8004** | Onchain agent reputation, staking |

The "Mullet Economy" thesis: **fiat in the front (B2C), crypto in the back (B2B/M2M)**. Consumer-facing agent transactions use AP2/ACP (familiar, regulated). Machine-to-machine backend automation uses x402 (instant, no accounts, programmable).

## Tradeoffs and Problems

### What Works

| Strength | Detail |
|----------|--------|
| Zero-friction onboarding | No accounts, no API keys, no KYC for machine-to-machine |
| Instant settlement | On-chain finality in seconds (Base) |
| Open standard | Not locked to Coinbase despite their heavy involvement |
| MCP compatibility | Works with Claude, ChatGPT, Cursor, any MCP client |
| Micropayment viable | $0.001 per call is practical (Base L2 gas is negligible) |

### What Doesn't Work (Yet)

| Problem | Severity | Detail |
|---------|----------|--------|
| USDC-only | High | EIP-3009 only supported by USDC. Other tokens can't use x402's trust-minimizing design |
| Facilitator centralization | High | Most implementations use Coinbase's facilitator. "Open standard" with one gatekeeper |
| No fiat on-ramp for agents | High | Agents need USDC. Getting USDC still requires human KYC somewhere in the chain |
| Retry/dispute handling | Medium | V1 had no retry logic or dispute resolution. V2 improving but still thin |
| Human confirmation UX | Medium | Current MCP clients prompt humans for payment approval. Not truly autonomous yet |
| Wallet key management | Medium | Even with TEE, who provisions the wallet? Who funds it? Still human bootstrapping |

### The Centralization Irony

x402 is an "open standard" but the infrastructure is dominated by Coinbase:
- Coinbase built the SDK
- Coinbase runs the primary facilitator
- Coinbase provides the wallet infrastructure (AgentKit, Agentic Wallets)
- Coinbase co-founded the x402 Foundation with Cloudflare

This is the Web3 pattern repeating: decentralized protocol, centralized implementation.

### The Bootstrap Problem

For an agent to pay with x402, someone must:
1. Create a wallet (requires human)
2. Fund the wallet with USDC (requires human + KYC exchange)
3. Configure the agent with wallet access (requires human)
4. Set spending limits (requires human)

x402 removes the per-transaction human involvement, but the initial setup is still fully human-dependent. The "fully autonomous agent economy" narrative breaks down at bootstrapping.

## Alternatives

| Alternative | Approach | Trade-off |
|-------------|----------|-----------|
| Stripe MCP | Traditional payment rails via MCP | Requires accounts, but familiar and regulated |
| Google AP2 | Fiat agent payments with cryptographic mandates | More enterprise-friendly, less crypto-native |
| Plain API keys | Pre-provisioned access, no per-call payment | No autonomy, but simple and works today |
| OAuth + billing | Standard web auth with usage billing | Human-managed, but battle-tested |
| ACP (Agentic Commerce) | Credit card infra adapted for agents | Familiar rails, but still requires card issuance |

## Steal These Patterns

| Pattern | What | Use Case |
|---------|------|----------|
| HTTP 402 handshake | Payment embedded in standard request-response | Any API monetization — no account system needed |
| `paidTool()` decorator | Price declared alongside tool definition | Monetize MCP tools with one line |
| Facilitator abstraction | Separate payment verification from business logic | Server doesn't need blockchain knowledge |
| TEE wallet isolation | Private keys never in LLM context | Secure agent autonomy with bounded risk |
| Session-based access | Pay once, get session token via `@x402/paywall` | Reduce per-call overhead for high-frequency use |
| Deferred settlement | Aggregate micropayments, settle in batches | Cost-effective for high-volume, low-value calls |
| Proxy monetization (MCPay) | Wrap existing MCP servers with payment | Monetize without touching upstream code |

## Practical Getting Started

### Fastest Path: Use Coinbase x402 + MCP

```bash
git clone https://github.com/coinbase/x402.git
cd x402/examples/typescript
pnpm install && pnpm build

# Start the demo paid API server
cd servers/express && pnpm dev

# In another terminal, configure Claude Desktop with the MCP server
# Add to claude_desktop_config.json (see Pattern 1 above)
```

Requirements:
- Node.js v20+
- A wallet with USDC on Base Sepolia (testnet) or Base Mainnet
- Claude Desktop with MCP enabled

### Fastest Path: Monetize an Existing MCP Server

```bash
npx mcpay connect --urls https://your-mcp-server.com --wallet-key 0x...
```

MCPay wraps your server as a proxy — zero code changes.

## Bottom Line

**MCP + x402 is the first practical "agent pays for service" stack.** It's real, it's shipping, and it works today on testnet and mainnet.

**The honest picture**: It solves the per-transaction friction (no accounts, no API keys, instant payment). It does NOT solve the bootstrapping problem (someone still funds the wallet). The "autonomous agent economy" is real for machine-to-machine backend payments, but consumer-facing agent commerce will likely use fiat rails (AP2/ACP) for regulatory reasons.

**What to build today**: If you have an API or MCP tool, adding x402 payment is ~10 lines of middleware. If you're building agents, `@x402/fetch` or `@x402/axios` wraps your HTTP client with automatic payment. The infrastructure is mature enough for production use on Base mainnet.
