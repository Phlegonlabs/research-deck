# A2A, AP2, ACP: The Agentic Commerce Protocol Stack

## What Problem Are We Solving?

AI agents can now reason and use tools (via MCP). They can pay for services (via x402). But two critical gaps remain:

1. **Agent-to-agent communication**: How does a travel agent delegate hotel booking to a hotel agent? MCP connects agents to tools, not to each other.
2. **Trusted commerce**: When an agent buys something on your behalf, who proves you authorized it? Who's liable if it goes wrong? x402 handles micropayments but has no authorization framework.

Google's A2A and AP2, plus OpenAI/Stripe's ACP, fill these gaps. Together with MCP and x402, they form a complete protocol stack.

## The Full Stack

```
┌─────────────────────────────────────────────────────┐
│                    IDENTITY / TRUST                  │
│              ERC-8004 (onchain reputation)            │
├─────────────────────────────────────────────────────┤
│                     PAYMENTS                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │   x402    │  │   AP2    │  │      ACP         │  │
│  │ Crypto    │  │ Fiat +   │  │ Chat commerce    │  │
│  │ micro-    │  │ crypto   │  │ (Stripe tokens)  │  │
│  │ payments  │  │ mandates │  │                  │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
├─────────────────────────────────────────────────────┤
│                  COORDINATION                        │
│              A2A (Agent-to-Agent)                     │
│         Task delegation, discovery, streaming        │
├─────────────────────────────────────────────────────┤
│                   TOOL ACCESS                        │
│              MCP (Model Context Protocol)             │
│         Agent ↔ APIs, databases, tools               │
└─────────────────────────────────────────────────────┘
```

| Layer | Protocol | Owner | What It Does |
|-------|----------|-------|-------------|
| Tool Access | **MCP** | Anthropic | Agent connects to tools/APIs (vertical) |
| Coordination | **A2A** | Google (150+ partners) | Agent delegates tasks to other agents (horizontal) |
| Payment (crypto) | **x402** | Coinbase/Cloudflare | HTTP-native stablecoin micropayments |
| Payment (mandates) | **AP2** | Google (60+ partners) | Cryptographic authorization + multi-rail payments |
| Payment (commerce) | **ACP** | OpenAI/Stripe | Chat-based checkout with shared payment tokens |
| Identity | **ERC-8004** | Community | Onchain agent reputation and staking |

**Key insight**: These are layers, not competitors. MCP without A2A = isolated agents. A2A without AP2 = agents that talk but can't pay. x402 without AP2 = payments without accountability.

## A2A: Agent-to-Agent Protocol

### What It Is

A2A is an open protocol for **agent-to-agent communication**. Think HTTP, but for agents talking to agents instead of browsers talking to servers.

Core design principle: **opaque agents**. An agent doesn't need to expose its internals (memory, tools, reasoning chain) to collaborate. It just advertises capabilities and accepts tasks.

### Architecture

```
Client Agent                                Remote Agent
    │                                           │
    │── GET /.well-known/agent-card.json ──────>│
    │<── Agent Card (capabilities, skills) ─────│
    │                                           │
    │── tasks/send (JSON-RPC) ────────────────->│
    │<── Task { status: "working" } ────────────│
    │                                           │
    │── tasks/sendSubscribe (SSE stream) ──────>│
    │<── TaskStatusUpdate ──────────────────────│
    │<── TaskArtifactUpdate ────────────────────│
    │<── Task { status: "completed" } ──────────│
```

### Key Concepts

**Agent Card** (`/.well-known/agent-card.json`): A JSON document — the agent's business card. Contains identity, skills, supported modalities, authentication requirements, and service endpoint.

```json
{
  "name": "Hotel Booking Agent",
  "description": "Books hotels, manages reservations",
  "url": "https://hotel-agent.example.com",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "book-hotel",
      "name": "Book Hotel",
      "description": "Search and book hotel rooms"
    }
  ],
  "securitySchemes": { "oauth2": { "..." } }
}
```

**Task Lifecycle**: Unit of work with state machine progression.

```
submitted → working → completed
                   → failed
                   → canceled
            working → input-required → working (resumes)
```

**Messages**: Bidirectional communication during task execution. Supports text, files, structured data, forms.

**Artifacts**: Deliverables produced by the remote agent — documents, images, data, any tangible output.

### Transport

- JSON-RPC 2.0 over HTTPS
- Three modes: sync request/response, SSE streaming, async push notifications (webhooks)
- Auth: API keys, OAuth2, OpenID Connect, mTLS
- Works through existing API gateways and enterprise SSO

### What A2A Solves That MCP Doesn't

| Dimension | MCP | A2A |
|-----------|-----|-----|
| Interaction | Agent → Tool (invoke function) | Agent → Agent (delegate task) |
| State | Stateless tool calls | Stateful long-running tasks |
| Opacity | Tool internals exposed via schema | Agent internals hidden |
| Modality | Structured inputs/outputs | Text, files, forms, audio, video |
| Discovery | Manual tool configuration | Agent Card auto-discovery |
| Multi-turn | Single request/response | Ongoing dialogue with state |

**The auto repair analogy**: MCP = mechanic using a diagnostic scanner (tool). A2A = shop manager delegating the job to the mechanic (agent).

## AP2: Agent Payments Protocol

### What It Is

AP2 is Google's open protocol for **agent-initiated payments with cryptographic accountability**. The core innovation: **mandates** — cryptographically signed proof of user authorization that creates a non-repudiable audit trail.

AP2 solves three problems no other protocol addresses:
1. **Authorization**: Proving the user actually approved this purchase
2. **Authenticity**: Proving the agent's request matches the user's real intent
3. **Accountability**: Determining who's liable when things go wrong

### The Mandate System

Three types of cryptographic credentials:

#### Cart Mandate (Human Present)

User is watching. They see the cart, approve it, and sign with their device.

```json
{
  "id": "cart_abc123",
  "total": { "currency": "USD", "value": 238.00 },
  "items": [
    { "sku": "TICKET-001", "qty": 2, "price": 119.00 }
  ],
  "merchant_signature": "sig_merchant_xyz",
  "user_signature": "sig_user_device_key",
  "payment_method_token": "tok_visa_4242",
  "timestamp": "2026-02-19T10:30:00Z"
}
```

Cryptographically signed by both the user (hardware-backed key) and the merchant. Non-repudiable proof: "I approved this exact cart at this exact price."

#### Intent Mandate (Human NOT Present)

User delegates a future purchase: "Buy concert tickets the moment they go on sale, under $120 each."

```json
{
  "natural_language_description": "Buy 2 concert tickets under $120 each",
  "required_refundability": true,
  "intent_expiry": "2026-03-01T00:00:00Z",
  "max_total": { "currency": "USD", "value": 240.00 },
  "user_cart_confirmation_required": false,
  "user_device_signature": "sig_device_key_abc"
}
```

The agent acts within these bounds. If it exceeds them, the mandate signature proves the agent violated authorization.

#### Payment Mandate (For Payment Networks)

Signals to Visa/Mastercard/issuers: "This transaction was initiated by an AI agent" + human-present vs. not-present flag. Enables risk assessment without exposing PCI data.

### Roles Architecture

```
┌──────────┐         ┌──────────────┐         ┌──────────────┐
│   User    │────────>│  Shopping     │────A2A──>│   Merchant   │
│           │ intent  │  Agent (SA)   │         │  Endpoint    │
└──────────┘         └──────┬───────┘         └──────┬───────┘
                            │                         │
                     ┌──────▼───────┐         ┌──────▼───────┐
                     │ Credentials   │         │  Merchant    │
                     │ Provider (CP) │         │  Payment     │
                     │ (wallet, card │         │  Processor   │
                     │  tokenization)│         │  (MPP)       │
                     └──────┬───────┘         └──────┬───────┘
                            │                         │
                            └─────────────────────────┘
                                      │
                               ┌──────▼───────┐
                               │  Network /    │
                               │  Issuer       │
                               │  (Visa, etc.) │
                               └──────────────┘
```

Six roles with strict separation of concerns:

| Role | Sees | Never Sees |
|------|------|-----------|
| Shopping Agent | Products, prices, cart | Payment credentials, PCI data |
| Credentials Provider | Tokenized payment methods | Shopping context, merchant internals |
| Merchant | Cart mandate, signed authorization | Raw card numbers |
| Payment Processor | Payment mandate, transaction data | User's full shopping history |

**Critical**: The shopping agent NEVER touches raw payment credentials. It gets tokenized payment methods exclusively. This is the key architectural constraint.

### Payment Flow (Human Present)

1. User tells agent: "Buy those shoes"
2. Agent negotiates cart with merchant via A2A
3. Merchant cryptographically signs cart (confirms price + availability)
4. Credentials Provider provides tokenized payment options
5. User reviews cart in **trusted UI** (not the agent's UI)
6. User approves via device authentication → signed Cart Mandate
7. Agent submits Payment Mandate + attestation to Credentials Provider
8. Payment processed through traditional rails (Visa/Mastercard) OR x402
9. Issuer receives AI-presence signals for risk assessment
10. 3D Secure challenge routes to user if needed
11. Success → fulfillment

### Payment Flow (Human NOT Present)

1. User signs Intent Mandate: "Buy X when condition Y, max $Z"
2. Agent monitors for condition
3. When triggered: agent executes within mandate bounds
4. Merchant may force user confirmation if uncertain
5. Cart Mandate created → payment flows same as human-present

### How AP2 Connects to x402

AP2 is **payment-rail agnostic**. V0.1 supports "pull" payments (cards). Future versions add "push" payments — including x402 stablecoins.

The integration pattern:

```
AP2 Mandate (authorization layer)
    │
    ├── Traditional rails: Visa/Mastercard/PayPal
    ├── Real-time bank transfers
    └── x402 stablecoin settlement ← here
```

AP2 provides the **trust and accountability** (who authorized what). x402 provides the **settlement** (instant on-chain payment). Together:
- AP2 Intent Mandate sets spending limits and rules
- x402 handles the actual micropayment per API call
- AP2 Payment Mandate creates audit trail for each transaction
- x402 settles on Base/Solana

There's even an official demo repo: `google-agentic-commerce/a2a-x402`.

## ACP: Agentic Commerce Protocol

### What It Is

OpenAI and Stripe's protocol for **chat-based commerce**. Powers "Instant Checkout in ChatGPT" — buy from Shopify/Etsy directly in chat.

### Architecture

Simpler than AP2 — four actors, not six:

| Actor | Role |
|-------|------|
| Buyer | Discovers products via AI, authorizes payment |
| AI Agent | Shows products, collects payment intent, manages checkout |
| Business | Receives checkout requests, fulfills orders |
| Payment Provider | Issues Shared Payment Token, processes charges |

### Core Flow

```
Buyer → "I want blue sneakers" → Agent
Agent → Create Checkout (SKU) → Merchant ACP endpoint
Merchant → Cart + pricing → Agent → Buyer reviews
Buyer → "Buy it" → Agent
Agent → Shared Payment Token → Merchant
Merchant → Complete Checkout (via Stripe) → Confirmation
```

**Shared Payment Token**: Stripe's key innovation. A scoped, time-limited, usage-capped token that lets the merchant charge without seeing raw card data. Programmatically controlled, permissioned, logged.

### ACP Endpoints

| Endpoint | Purpose |
|----------|---------|
| Create Checkout | Agent sends SKU, gets cart/pricing |
| Complete Checkout | Agent submits payment token, merchant charges |
| Cancel Checkout | Release reserved inventory |

### ACP vs AP2

| Dimension | ACP | AP2 |
|-----------|-----|-----|
| Focus | Consumer checkout UX | Enterprise accountability |
| Complexity | Simple (4 actors) | Complex (6 roles + mandates) |
| Auth model | Shared Payment Token | Cryptographic mandates |
| Payment rails | Stripe-first (delegated PSP) | Rail-agnostic (cards + crypto + bank) |
| Dispute model | Standard Stripe disputes | Mandate-based crypto evidence |
| Live today? | Yes (ChatGPT Instant Checkout) | V0.1 (human-present cards only) |
| Partners | OpenAI, Stripe, Shopify, Etsy | Google, Mastercard, PayPal, 60+ |
| Philosophy | "Make checkout invisible" | "Make transactions provable" |

**Honest take**: ACP is **shipping in production** and works today. AP2 is **architecturally more rigorous** but mostly still V0.1. Choose by time horizon: ACP for now, AP2 for enterprise-grade future.

## How Everything Fits Together

### The Agentic Commerce Stack

```
User: "Find me flights to Tokyo under $800, book the best one"
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Personal Agent (orchestrator)                       │
│                                                      │
│  1. Uses MCP to query calendar, check budget         │
│  2. Uses A2A to delegate to Flight Search Agent      │
│  3. Flight Agent uses MCP to call airline APIs       │
│  4. Flight Agent returns options via A2A artifacts   │
│  5. Personal Agent selects best option               │
│  6. Uses AP2 mandate to authorize payment            │
│  7. Credentials Provider tokenizes payment method    │
│  8. x402 settles the transaction on-chain            │
│     OR AP2 routes through Visa/Mastercard            │
│  9. Confirmation flows back via A2A                  │
│                                                      │
│  Every step has: cryptographic proof, audit trail,   │
│  spending limits, human can be asked at any point    │
└─────────────────────────────────────────────────────┘
```

### Protocol Interaction Matrix

| Scenario | MCP | A2A | AP2 | ACP | x402 |
|----------|-----|-----|-----|-----|------|
| Agent uses a database | **yes** | - | - | - | - |
| Agent delegates task to another agent | - | **yes** | - | - | - |
| Agent buys something (consumer) | - | - | - | **yes** | - |
| Agent buys something (enterprise) | - | **yes** | **yes** | - | - |
| Agent pays for API call | - | - | - | - | **yes** |
| Agent-to-agent paid service | - | **yes** | optional | - | **yes** |
| Dispute resolution needed | - | - | **yes** | Stripe | - |
| Multi-agent workflow | **yes** | **yes** | **yes** | - | **yes** |

### The "Mullet Economy"

**Front (B2C)**: Consumers interact with agents that use ACP (ChatGPT checkout) or AP2 (Google Shopping). Familiar payment rails: Visa, Mastercard, PayPal. Regulated, insured, chargeback-protected.

**Back (B2B/M2M)**: Agents pay each other for API access, data, compute via x402. Instant stablecoin settlement. No accounts needed. Machine-speed, machine-volume.

AP2 bridges both worlds: it can authorize fiat payments (front) AND x402 stablecoin payments (back) with the same mandate system.

## Tradeoffs and Problems

### A2A Issues

| Problem | Severity | Detail |
|---------|----------|--------|
| Discovery fragmentation | High | Agent Cards on well-known URIs work for public agents; enterprise discovery needs registries that don't exist yet |
| Auth complexity | High | OAuth2/mTLS between agents from different orgs is non-trivial to configure |
| No economic layer | Medium | A2A has no concept of payment — needs AP2/x402 on top |
| Google dominance | Medium | 150+ partners, but Google controls the spec. Same Web3 centralization pattern? |
| Streaming overhead | Low | SSE for real-time but adds complexity vs. simple request/response |

### AP2 Issues

| Problem | Severity | Detail |
|---------|----------|--------|
| V0.1 only | Critical | Only human-present card payments work today. Autonomous + crypto = "planned" |
| Mandate UX | High | Users must sign cryptographic mandates. Current UX for this is terrible |
| 6-role complexity | High | Cart Mandate → Credentials Provider → Payment Mandate → Processor → Network. Many integration points |
| Allowlist bootstrapping | High | Short-term trust via manual curation. Doesn't scale |
| No micropayment story | Medium | AP2 mandates are heavyweight for $0.001 API calls. x402 is better here |

### ACP Issues

| Problem | Severity | Detail |
|---------|----------|--------|
| Stripe lock-in | High | "Open standard" but Shared Payment Token = Stripe-first. Others can implement delegated PSP spec, but Stripe is the reference |
| Consumer-only | High | No enterprise audit trail, no cryptographic mandates |
| ChatGPT-centric | Medium | Powers Instant Checkout in ChatGPT. Other agents need to build their own UX |
| Limited payment rails | Medium | Credit/debit cards via Stripe. No bank transfers, no crypto (yet) |

### The Meta-Problem: Too Many Standards

```
MCP + A2A + AP2 + ACP + x402 + ERC-8004 = 6 protocols
```

For a single commerce transaction, an agent might need: MCP (tools) → A2A (delegation) → AP2 (authorization) → x402 (settlement). That's 4 protocol layers. Integration complexity is real.

The XKCD "standards" problem applies: each protocol claims to be "the one open standard," but together they create a combinatorial integration burden.

## Alternatives

| Approach | What | Trade-off |
|----------|------|-----------|
| Just use Stripe | Payment Links + existing checkout | No agent autonomy, but works everywhere today |
| Just use MCP + x402 | Skip A2A/AP2 entirely | Simpler, but no agent delegation or accountability |
| Platform lock-in | Apple Pay, Google Pay | Battle-tested, but agents can't use them autonomously |
| Custom agent protocols | Build your own | Full control, zero interoperability |
| Wait and see | Let standards shake out | Reasonable. Most of this is V0.1 or beta |

## Steal These Patterns

| Pattern | What | Why It Matters |
|---------|------|----------------|
| Agent Card discovery | `/.well-known/agent-card.json` | Standard agent capability advertisement — use this even without A2A |
| Task state machine | submitted → working → input-required → completed | Clean lifecycle model for any long-running agent interaction |
| Mandate system | Cryptographic proof of user authorization | Accountability for any autonomous agent action, not just payments |
| Role separation | Agent ≠ payment handler ≠ credential store | Never let the LLM see raw credentials. Architectural constraint |
| Shared Payment Token | Scoped, time-limited, usage-capped | Safe delegation of payment authority — applicable beyond ACP |
| Payment rail abstraction | AP2 mandate → multiple settlement methods | Decouple authorization from settlement — AP2 mandate can route to x402 OR Visa |
| Push notifications | Webhooks for async task updates | Better than polling for multi-agent workflows |

## Timeline and Maturity

| Protocol | Status | Production Ready? |
|----------|--------|-------------------|
| MCP | Stable, widely adopted | Yes |
| A2A | V0.2+, 150+ partners | Yes (basic), evolving |
| x402 | V2, 100M+ payments | Yes |
| ACP | Beta, versioned spec | Yes (ChatGPT checkout) |
| AP2 | V0.1, 60+ partners | Partial (human-present cards only) |
| ERC-8004 | Draft | No |

## Bottom Line

**What's real today**: MCP (tools) + x402 (micropayments) + ACP (consumer checkout) work in production. A2A is functional for basic agent delegation.

**What's coming**: AP2 autonomous mandates, AP2 + x402 integration, ACP expanding beyond ChatGPT.

**The honest assessment**: We have too many protocols and not enough production implementations. The stack *makes sense architecturally* — each layer solves a real problem. But the integration complexity is high, and most of it is V0.1.

**What to build**: If you're building agents today, use MCP + x402 for tool access and micropayments. Add A2A if you need multi-agent delegation. Watch AP2 but don't build on it until V1.x. Use ACP if you're doing consumer commerce on ChatGPT.

**The $5T question**: McKinsey projects $3-5T in agentic transaction volume by 2030. Whether these specific protocols win, or get consolidated/replaced, the *patterns* they establish (mandate-based authorization, payment-rail abstraction, agent discovery, role separation) will persist. Learn the patterns, not just the APIs.
