# ACP: Agentic Commerce Protocol — How Stripe & OpenAI Built Agent Checkout

## What Is It?

ACP (Agentic Commerce Protocol) is an open standard co-developed by **Stripe and OpenAI** that lets AI agents complete purchases on behalf of users. It's the protocol powering **ChatGPT Instant Checkout** — the feature where you can buy products without leaving the chat.

The core insight: existing payment rails were designed for humans clicking buttons. ACP redesigns checkout as a **programmatic API** that agents can call. Five REST endpoints handle the entire purchase lifecycle, and a new payment primitive called **Shared Payment Token (SPT)** ensures the agent never sees your credit card.

**Status**: Live in production. ChatGPT Instant Checkout launched Feb 16, 2026 for US users (Plus, Pro, Free). Spec is Apache 2.0 on GitHub. Four spec versions shipped (2025-09-29 → 2026-01-30).

## Why It Matters

The agent commerce problem has three parts:

| Problem | Pre-ACP | With ACP |
|---------|---------|----------|
| How does an agent buy things? | Screen-scrape checkout forms | Call structured REST API |
| Who holds payment credentials? | Agent or user re-enters them | SPT — scoped, single-use token |
| Who is merchant-of-record? | Unclear — agent? platform? | Merchant retains full control |

ACP is **not** a marketplace. OpenAI is not the seller. Merchants keep their customer relationships, accept/decline orders, handle fulfillment, and process refunds. The agent is just a sophisticated checkout assistant.

## The Architecture

### Four Actors

```
┌─────────┐     ┌──────────┐     ┌──────────┐     ┌─────────┐
│  Buyer   │────>│ AI Agent │────>│ Merchant │────>│   PSP   │
│ (human)  │     │(ChatGPT) │     │ (seller) │     │(Stripe) │
└─────────┘     └──────────┘     └──────────┘     └─────────┘
  Approves        Orchestrates     Fulfills          Processes
  purchase        checkout flow    orders            payments
```

**Key separation**: The agent orchestrates but never handles money. The PSP (Stripe) handles money but never sees the conversation. The merchant controls the catalog and fulfillment but doesn't need to understand AI.

### Five Endpoints

| Endpoint | Method | What It Does |
|----------|--------|-------------|
| Create Checkout | `POST /checkout_sessions` | Agent sends items + buyer info → merchant returns checkout state |
| Retrieve Checkout | `GET /checkout_sessions/{id}` | Read current checkout state (prices, options, status) |
| Update Checkout | `POST /checkout_sessions/{id}` | Modify items, shipping address, fulfillment option |
| Complete Checkout | `POST /checkout_sessions/{id}/complete` | Submit SPT → merchant processes payment → order created |
| Cancel Checkout | `POST /checkout_sessions/{id}/cancel` | Abort the transaction, with optional `intent_trace` |

**Status state machine:**
```
not_ready_for_payment → ready_for_payment → completed
                                          → canceled
                      → in_progress       → completed
                                          → canceled
                      → authentication_required → completed  (3DS flow)
                                                → canceled
```

Required header: `API-Version: 2026-01-30`. All POST requests require `Idempotency-Key` (UUID v4).

### The Full Purchase Flow

```
Buyer           ChatGPT              Merchant API         Stripe (PSP)
  │                │                       │                    │
  │ "buy me a      │                       │                    │
  │  blue jacket"  │                       │                    │
  │───────────────>│                       │                    │
  │                │                       │                    │
  │                │── POST /checkouts ───>│                    │
  │                │   {items, buyer}      │                    │
  │                │<── checkout state ────│                    │
  │                │   (prices, shipping)  │                    │
  │                │                       │                    │
  │ [shows options]│                       │                    │
  │ "ship to home" │                       │                    │
  │───────────────>│                       │                    │
  │                │── PUT /checkouts/:id─>│                    │
  │                │   {fulfillment_opt}   │                    │
  │                │<── updated state ─────│                    │
  │                │                       │                    │
  │ "yes, buy it"  │                       │                    │
  │───────────────>│                       │                    │
  │                │── delegated payment ──────────────────────>│
  │                │   {card, billing,     │                    │
  │                │    risk signals}      │                    │
  │                │<── SPT (spt_xxx) ─────────────────────────│
  │                │                       │                    │
  │                │── POST /complete ────>│                    │
  │                │   {payment_data:      │                    │
  │                │    {token: spt_xxx}}  │── PaymentIntent ──>│
  │                │                       │<── confirmation ───│
  │                │<── order confirmed ───│                    │
  │                │                       │                    │
  │ "Order placed! │                       │                    │
  │  Track here:"  │                       │                    │
  │<───────────────│                       │                    │
```

## Shared Payment Token (SPT) — The Key Innovation

SPT is **Stripe's new payment primitive** that solves the agent-credential problem. The core idea: instead of giving the agent your credit card, Stripe issues a **scoped, single-use, time-limited token** that can only be used for one specific purchase.

### SPT Properties

| Property | Detail |
|----------|--------|
| Single-use | One token per transaction. Can't be replayed |
| Time-limited | `expires_at` field (RFC 3339). Token dies after deadline |
| Merchant-scoped | Tied to specific `merchant_id`. Useless elsewhere |
| Amount-capped | `max_amount` in allowance. Can't charge more than approved |
| Credential-isolated | Neither agent nor merchant sees raw card number |

### Token Format

```
spt_123abc...
```

Prefix `spt_` identifies it as a Shared Payment Token (like Stripe's `pi_`, `pm_`, `tok_` conventions).

### The Delegated Payment Flow

This is the most technically interesting part. Three-phase credential delegation:

**Phase 1: OpenAI → PSP (Stripe)**
```json
{
  "payment_method": {
    "card_number_type": "network_token",
    "cryptogram": "AJkBAfd...",
    "eci_value": "05",
    "checks_performed": ["cvc_check"],
    "display_brand": "visa",
    "display_last4": "4242"
  },
  "billing_address": { "...": "..." },
  "risk_signals": {
    "type": "stripe_risk",
    "score": 12,
    "action": "allow"
  },
  "allowance": {
    "checkout_session_id": "cs_abc123",
    "merchant_id": "merch_xyz",
    "currency": "usd",
    "max_amount": 9999,
    "expires_at": "2026-02-19T12:00:00Z",
    "reason": "one_time"
  },
  "idempotency_key": "idk_unique123"
}
```

**Phase 2: PSP → OpenAI**
```json
{
  "token": "spt_123abc",
  "created_at": "2026-02-19T11:00:00Z",
  "metadata": { "checkout_session_id": "cs_abc123" }
}
```

**Phase 3: OpenAI → Merchant (via Complete Checkout)**
```json
{
  "payment_data": {
    "token": "spt_123abc",
    "provider": "stripe",
    "billing_address": { "...": "..." }
  }
}
```

**Why network tokens over FPANs?** Network tokens (tokenized card numbers from Visa/Mastercard) reduce the merchant's PCI compliance scope. The `card_number_type` field supports both `"fpan"` (full PAN) and `"network_token"`, but network tokens are strongly preferred.

### Security Guarantees

```
┌─────────────────────────────────────────────────┐
│  What each actor sees:                           │
│                                                  │
│  Buyer:     Full card details (they own it)      │
│  ChatGPT:   display_last4 + display_brand only   │
│  Stripe:    Full card + risk signals             │
│  Merchant:  SPT token only (spt_xxx)             │
│                                                  │
│  Nobody except the user and Stripe sees the      │
│  actual card number. Agent sees display info.     │
│  Merchant sees only the opaque token.            │
└─────────────────────────────────────────────────┘
```

## Payment Handler Framework (v4 — 2026-01-30)

The latest spec version replaced simple `payment_methods` with a handler-based system:

| Handler | Type | Description |
|---------|------|-------------|
| `dev.acp.tokenized.card` | External | Standard credit/debit via Stripe SPT |
| `dev.acp.seller_backed.saved_card` | Seller-backed | Card saved with merchant |
| `dev.acp.seller_backed.gift_card` | Seller-backed | Merchant-issued gift card |
| `dev.acp.seller_backed.points` | Seller-backed | Loyalty/rewards points |
| `dev.acp.seller_backed.store_credit` | Seller-backed | Merchant account balance |

Seller-backed handlers set `requires_delegate_payment: true` but `requires_pci_compliance: false` — the token request carries no actual credentials, just a handler ID for the merchant to resolve internally. This enables merchants to accept loyalty points and gift cards alongside credit cards in agent checkout.

## Checkout Data Model

### Request/Response Schema

Every checkout response includes these sections:

```
Checkout Object
├── id: "cs_abc123"
├── status: "ready_for_payment"
├── currency: "usd"
├── buyer
│   ├── first_name, last_name (required)
│   ├── email (required)
│   └── phone_number (optional)
├── payment_provider
│   ├── provider: "stripe"
│   └── supported_payment_methods: ["card"]
├── line_items[]
│   ├── id, item (product details)
│   ├── base_amount, discount, tax
│   └── subtotal, total
├── fulfillment_address
│   ├── name, line_one, line_two
│   ├── city, state, country (ISO 3166-1 alpha-2)
│   └── postal_code
├── fulfillment_options[]
│   ├── type: "shipping" | "digital"
│   ├── title, subtitle, carrier
│   ├── earliest/latest_delivery_time (ISO 8601)
│   └── subtotal, tax, total
├── totals[]
│   ├── type: items_base_amount | items_discount | subtotal |
│   │         discount | fulfillment | tax | fee | total
│   ├── display_text
│   └── amount (integer, cents)
├── messages[]
│   ├── type: "info" | "error"
│   ├── code: missing | invalid | out_of_stock |
│   │         payment_declined | requires_sign_in | requires_3ds
│   └── content (plain | markdown)
├── links[]
│   ├── type: terms_of_use | privacy_policy | seller_shop_policies
│   └── url
└── order (after completion)
    ├── id, checkout_session_id
    └── permalink_url
```

**All amounts in integer cents** (e.g., $99.99 = 9999). Three-letter ISO currency codes, lowercase.

### Order Events (Webhooks)

After checkout completes, merchants notify the agent of order status changes:

| Event | Statuses |
|-------|----------|
| `order_created` | created |
| `order_updated` | manual_review → confirmed → shipped → fulfilled |
| `order_updated` | canceled (at any point) |

Refunds come as `store_credit` or `original_payment` with amount.

### Error Handling

Four error categories:
- `invalid_request` — bad parameters
- `request_not_idempotent` — conflicting idempotency key
- `processing_error` — server-side failure
- `service_unavailable` — temporary outage

The `messages[]` array in checkout responses handles inline errors (out of stock, payment declined, 3DS required) so the agent can respond conversationally.

## Product Feed — How Merchants List Products

Merchants supply their catalog via compressed product feeds:

| Aspect | Detail |
|--------|--------|
| Format | Compressed JSON Lines (`.jsonl.gz`) or CSV (`.csv.gz`) |
| Refresh | Every 15 minutes |
| Required fields | GTIN/UPC/MPN, title, price, availability, seller name |
| Policy URLs | Return policy, privacy policy, terms of service (all required) |
| Checkout flag | `enable_checkout=true` (requires `enable_search=true`) |

**Critical constraint**: Products must be searchable (`enable_search=true`) before they can be purchasable (`enable_checkout=true`). This ensures ChatGPT can only sell what it can recommend.

## Merchant Onboarding — Four Paths

| Path | Who | Effort | Timeline |
|------|-----|--------|----------|
| **Stripe Direct** | Existing Stripe merchants (WooCommerce, BigCommerce, Wix, Squarespace) | "As little as one line of code" | Days |
| **Shopify** | 1M+ Shopify merchants | Single setup, syndication to ChatGPT + Perplexity + Copilot | Weeks |
| **PayPal ACP Server** | Tens of millions of small businesses | PayPal handles ACP integration | Coming 2026 |
| **Direct Application** | Custom platforms | Build REST API per spec, apply at chatgpt.com/merchants | Months |

**Certification required**: Before production, merchants must pass OpenAI conformance checks on products, payments, and order processing.

## Economics

### Fee Structure

| Fee | Amount |
|-----|--------|
| OpenAI platform fee | 4% of transaction |
| Stripe processing | ~2.9% + $0.30 |
| **Combined take rate** | ~9.2% (with Shopify Payments) |

**Comparison:**
| Platform | Take Rate |
|----------|-----------|
| Amazon Marketplace | 25-30% |
| **ChatGPT (ACP)** | **~9.2%** |
| Google Shopping | 0% (ad-supported) |
| Microsoft Copilot | 0% (referral model) |

**30-day free trial** for Shopify merchants on the OpenAI platform fee.

### Market Numbers

| Metric | Value |
|--------|-------|
| ChatGPT WAU | 800-900M |
| Daily shopping queries | 50M |
| US consumers using GenAI for shopping | 59% |
| Prefer ChatGPT for shopping | 65% of GenAI shoppers |
| Walmart referral traffic from ChatGPT | ~36% |
| Projected US agentic e-commerce (2030) | $190-385B |

## Spec Evolution

The spec uses date-based versioning with rapid iteration:

| Version | Date | Key Changes |
|---------|------|------------|
| Initial | 2025-09-29 | Core checkout endpoints, SPT, Etsy launch |
| v2 | 2025-12-12 | Fulfillment enhancements |
| v3 | 2026-01-16 | Capability negotiation |
| v4 | 2026-01-30 | Extensions, discounts, handlers |
| Unreleased | In progress | Next iteration |

**Repo structure:**
```
agentic-commerce-protocol/
├── rfcs/                     # Design rationale
├── spec/
│   ├── 2025-09-29/          # Initial
│   ├── 2025-12-12/          # Fulfillment
│   ├── 2026-01-16/          # Capabilities
│   ├── 2026-01-30/          # Extensions
│   └── unreleased/          # WIP
├── examples/                 # Sample requests/responses
├── changelog/
└── docs/                     # Governance, SEP guidelines
```

Key files: `openapi.agentic_checkout.yaml` (checkout API), `openapi.delegate_payment.yaml` (payment delegation), JSON schemas for all data models.

## ACP vs Google UCP vs AP2

Google's response to ACP is the **Universal Commerce Protocol (UCP)** — a broader standard that covers discovery, checkout, and payment across Google surfaces (Gemini, AI Mode in Search).

| Dimension | ACP (OpenAI/Stripe) | UCP (Google) | AP2 (Google) |
|-----------|---------------------|-------------|-------------|
| Scope | Checkout only | Full commerce (discovery → checkout → order mgmt) | Payment authorization only |
| Discovery | Product feed → ChatGPT search | `/.well-known/ucp` manifest | Not applicable |
| Transport | REST API or MCP | REST, A2A, MCP | Mandates (cryptographic) |
| Payment | SPT via Stripe | Tokenized instruments, multiple handlers | Three mandate types |
| Complexity | 5 endpoints | Full service architecture | 6 roles, complex flows |
| Actors | 4 (buyer, agent, merchant, PSP) | 4+ (surfaces, backends, providers) | 6 roles |
| Partners | Stripe, Shopify, Etsy, PayPal | Shopify, Etsy, Wayfair, Target, Walmart, Stripe, Amex | Google Wallet ecosystem |
| Status | Production (ChatGPT) | Production (Gemini, Search) | V0.1 (early) |

**The real dynamic**: ACP and UCP are converging competitors. Both are open-source, both support REST/MCP, both use token-based payments. The difference is organizational — ACP is Stripe-first, UCP is Google-first.

**Practical advice**: Merchants implementing both ACP and UCP capture **40% more agentic traffic** than single-protocol implementations. They're not mutually exclusive.

**AP2's role**: AP2 handles payment *authorization* (mandates), while ACP/UCP handle the *checkout flow*. AP2 is compatible with UCP and could theoretically work alongside ACP.

## Reference Implementation

**locus-technologies/agentic-commerce-protocol-demo** — MIT-licensed demo by Scale AI + Coinbase alumni:

| Component | Port | Role |
|-----------|------|------|
| MCP-UI Client | 3000 | Chat interface (adapted Scira Chat) |
| Merchant API | 4001 | Checkout session management |
| PSP Server | 4000 | Delegated payment tokenization |

**Tech stack**: TypeScript, Node.js 20+, PostgreSQL, Docker Compose.

**What it demonstrates:**
1. Product discovery via semantic vector search
2. Full checkout lifecycle (create → update → complete)
3. Delegated payment flow (credential vaulting → SPT issuance → redemption)
4. MCP tool integration for agent-commerce bridge

## Tradeoffs and Problems

### Real Issues

| Problem | Severity | Detail |
|---------|----------|--------|
| US only | High | No international support yet. Currency, tax, shipping regulations vary wildly |
| Stripe lock-in | High | Stripe is the only PSP with SPT support. "Open standard" but single implementation |
| 4% platform fee | Medium | Higher than Google (0%), lower than Amazon (25-30%). May deter low-margin merchants |
| Product feed burden | Medium | 15-min refresh, GTIN/UPC required, compressed format — heavy for small merchants |
| AI recommendation liability | Medium | Agent recommends wrong product, buyer returns → who pays? Not specified |
| Amazon excluded | Medium | Amazon blocks ChatGPT crawlers via robots.txt. Largest e-commerce player absent |
| Behavioral change | Medium | Consumers must trust AI to buy on their behalf. Adoption curve unknown |
| Fraud surface | Medium | Agent-initiated purchases are new attack vector. SPT mitigates but doesn't eliminate |
| Spec churn | Low | Four versions in 4 months. Moving target for implementers |

### What ACP Intentionally Doesn't Do

- **No product discovery**: Merchants supply feeds; ChatGPT handles search/recommendation
- **No inventory management**: Merchant's problem
- **No dispute resolution**: Merchant handles returns/chargebacks
- **No multi-currency**: USD only initially
- **No recurring payments**: SPT is `one_time` only. Subscriptions not supported
- **No agent-to-agent commerce**: ACP is buyer→agent→merchant, not agent→agent

### The Stripe Dependency

ACP is technically an "open standard" but practically a Stripe product:
- Stripe co-developed the spec
- Stripe is the only PSP with SPT support
- Stripe processes all payments
- Stripe provides fraud risk signals
- Even merchants using "other providers" still rely on Stripe's risk scores

The spec *allows* alternative PSPs to implement the Delegated Payment Spec, but nobody else has. This is x402-Coinbase all over again — open standard, single implementation.

## Steal These Patterns

| Pattern | What | Why It Matters |
|---------|------|----------------|
| Scoped payment tokens | SPT = single-use, time-limited, merchant+amount-scoped credential | Solves agent-credential problem without giving agents payment access |
| Delegated payment | Three-phase credential flow (user→PSP→token→merchant) | Neither agent nor merchant touches raw credentials |
| Network tokens over FPANs | Prefer tokenized card numbers for reduced PCI scope | Practical security improvement with minimal implementation cost |
| Product feed as discovery | Compressed JSONL/CSV with 15-min refresh | Agent can only sell what it can find — search gates commerce |
| Status-driven checkout | State machine with messages[] for inline errors | Agent handles "out of stock" conversationally instead of crashing |
| Webhook-based order lifecycle | Events for order_created → confirmed → shipped → fulfilled | Agent can proactively update user on order status |
| Date-based spec versioning | YYYY-MM-DD versions, all versions preserved | Clean evolution without breaking existing integrations |
| Dual-protocol strategy | Implement both ACP and UCP for 40% more traffic | Protocol-agnostic merchant infrastructure pays off |
| Risk signal forwarding | SPT includes Stripe risk scores even for non-Stripe merchants | Fraud protection as shared infrastructure |

## Bottom Line

**What's real**: ACP is in production. Real users buy real products through ChatGPT Instant Checkout. The SPT mechanism is clever — it genuinely solves the "agent needs to pay but shouldn't have credentials" problem. The spec is clean, minimal, and iterating fast.

**What's not ready**: International support, multi-PSP ecosystem, recurring payments, agent-to-agent commerce. The "open standard" claim is aspirational — today it's a Stripe/OpenAI duopoly running on Stripe rails.

**The honest assessment**: ACP is the first working implementation of agent commerce at scale. The 800M+ ChatGPT user base gives it distribution that no competing protocol can match. But the 4% fee + Stripe dependence creates the same centralization tension as x402+Coinbase. The real question is whether UCP (Google) forces genuine multi-vendor competition, or whether we end up with two proprietary "open standards."

**What to do today**: If you sell physical products online, apply for ChatGPT Instant Checkout. The 30-day free trial on the 4% fee means zero downside to testing. If you're building agent infrastructure, implement ACP's checkout endpoints — they're simple REST, and the spec is stable enough to build on. But also watch UCP closely, and consider implementing both.
