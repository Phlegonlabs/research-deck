# ERC-8004: Onchain Agent Identity, Reputation & Validation

## What Is It?

ERC-8004 ("Trustless Agents") is an Ethereum standard that gives AI agents **persistent onchain identity, reputation scoring, and work validation**. It's the missing trust layer for the agentic web.

The core problem: A2A tells agents *how* to talk. MCP tells them *what tools* to use. AP2/x402 tells them *how to pay*. But none of them answer: **who is this agent, and should I trust it?**

ERC-8004 answers this with three onchain registries — Identity, Reputation, and Validation — deployed as singleton contracts on Ethereum mainnet and 20+ L2s.

## Why It Matters

Without ERC-8004, the agent economy has a trust gap:

| Problem | Without ERC-8004 | With ERC-8004 |
|---------|-----------------|---------------|
| Discovery | Manual configuration, curated allowlists | Query onchain registry by capability |
| Trust | "Trust me bro" or organizational boundary | Onchain reputation history + validation proofs |
| Accountability | Who ran this agent? No verifiable answer | NFT-based identity tied to wallet + domain |
| Payment safety | Pay first, hope it works | Escrow that releases only after validated completion |
| Cross-org agents | Requires pre-existing business relationships | Permissionless interaction via shared trust layer |

## The Three Registries

### 1. Identity Registry (Who Are You?)

**ERC-721 NFT = Agent's passport.** Each agent gets a unique `agentId` token that points to an off-chain registration file.

```
┌────────────────────────────┐
│  Identity Registry (ERC-721)│
│                             │
│  agentId: 22                │
│  owner: 0xABC...            │
│  agentURI: ipfs://Qm...     │──── points to ────┐
│  metadata: { key: value }   │                    │
│  agentWallet: 0xDEF...      │                    ▼
└────────────────────────────┘    ┌──────────────────────┐
                                  │  Registration File    │
                                  │  (JSON, off-chain)    │
                                  │                       │
                                  │  name: "Hotel Agent"  │
                                  │  services:            │
                                  │    - A2A endpoint     │
                                  │    - MCP endpoint     │
                                  │  x402Support: true    │
                                  │  supportedTrust:      │
                                  │    - reputation       │
                                  │    - tee-attestation  │
                                  └──────────────────────┘
```

**Registration file structure:**
```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "myAgentName",
  "description": "Natural language description",
  "services": [
    {
      "name": "A2A",
      "endpoint": "https://agent.example/.well-known/agent-card.json",
      "version": "0.3.0"
    },
    {
      "name": "MCP",
      "endpoint": "https://mcp.agent.eth/",
      "version": "2025-06-18"
    }
  ],
  "x402Support": true,
  "active": true,
  "registrations": [
    { "agentId": 22, "agentRegistry": "eip155:1:0x8004A169..." }
  ],
  "supportedTrust": ["reputation", "crypto-economic", "tee-attestation"]
}
```

**Key design decisions:**
- **ERC-721 base**: Portable, composable, works with existing NFT infra (marketplaces, indexers, wallets)
- **URI-based metadata**: Heavy data lives off-chain (IPFS/HTTPS), onchain stores the pointer
- **Domain verification**: Optional `/.well-known/agent-registration.json` proves HTTPS endpoint ownership
- **agentWallet separation**: Agent's operational wallet ≠ owner's wallet. Requires EIP-712/ERC-1271 signature to change. Auto-clears on NFT transfer
- **Cross-chain**: `agentRegistry` format is `{namespace}:{chainId}:{contract}`, enabling multi-chain identity

**Core Solidity interface:**
```solidity
function register(string agentURI, MetadataEntry[] calldata metadata)
    external returns (uint256 agentId);

function setAgentURI(uint256 agentId, string calldata newURI) external;

function setAgentWallet(uint256 agentId, address newWallet,
    uint256 deadline, bytes calldata signature) external;

function getMetadata(uint256 agentId, string memory metadataKey)
    external view returns (bytes memory);
```

### 2. Reputation Registry (Can I Trust You?)

**Structured feedback from real interactions.** Not a single score — raw signals that consumers aggregate however they want.

```
Client Agent                               Reputation Registry
    │                                            │
    │ (completes task with Agent #22)             │
    │                                            │
    │── giveFeedback(                            │
    │     agentId: 22,                           │
    │     value: 87,                              │
    │     valueDecimals: 0,                       │
    │     tag1: "starred",                        │
    │     tag2: "hotel-booking",                  │
    │     endpoint: "https://agent.example/book", │
    │     feedbackURI: "ipfs://Qm...",            │
    │     feedbackHash: 0xABC...                  │
    │   ) ──────────────────────────────────────> │
    │                                            │
    │── getSummary(                               │
    │     agentId: 22,                            │
    │     clientAddresses: [0x111, 0x222],         │
    │     tag1: "starred",                         │
    │     tag2: ""                                 │
    │   ) <────────────────────────────────────── │
    │   returns: count=47, summaryValue=91        │
```

**Feedback value examples:**

| tag1 | Measurement | Example | value | decimals |
|------|-------------|---------|-------|----------|
| `starred` | Quality (0-100) | 87/100 | 87 | 0 |
| `uptime` | Availability (%) | 99.77% | 9977 | 2 |
| `successRate` | Task completion (%) | 89% | 89 | 0 |
| `tradingYield` | Return (%) | -3.2% | -32 | 1 |
| `latency` | Response time (ms) | 150ms | 150 | 0 |

**Anti-gaming measures:**
- **No self-feedback**: Agent owners/operators can't rate their own agents
- **Client address filtering**: `getSummary()` requires explicit client addresses — forces consumers to curate whose feedback they trust, limiting Sybil impact
- **Revocable**: Feedback can be revoked (marked, not deleted)
- **Append-only**: Onchain hashes can't be deleted — permanent audit trail
- **x402 proof integration**: Feedback file can include `proofOfPayment` showing the reviewer actually paid for the service

**Off-chain feedback file:**
```json
{
  "agentId": 22,
  "clientAddress": "eip155:1:0xABC...",
  "value": 100,
  "tag1": "starred",
  "endpoint": "https://agent.example.com/GetPrice",
  "a2a": {
    "skills": ["book-hotel"],
    "taskId": "task-123"
  },
  "proofOfPayment": {
    "txHash": "0x00...",
    "chainId": "1"
  }
}
```

### 3. Validation Registry (Did You Actually Do It?)

**Independent verification of agent work.** This is where trust gets teeth — not just "people say it's good" but "we can prove it was done correctly."

```
Client                Validation Registry           Validator
  │                          │                          │
  │── validationRequest(     │                          │
  │     validator: 0xVAL,    │                          │
  │     agentId: 22,         │                          │
  │     requestURI: "...",   │                          │
  │     requestHash: 0x...   │                          │
  │   ) ────────────────────>│                          │
  │                          │── event ────────────────>│
  │                          │                          │
  │                          │   [validator re-executes │
  │                          │    or checks proof]      │
  │                          │                          │
  │                          │<── validationResponse(   │
  │                          │      requestHash: 0x..., │
  │                          │      response: 95,       │
  │                          │      responseURI: "...", │
  │                          │      tag: "hard-finality" │
  │                          │    ) ────────────────────│
  │                          │                          │
  │<── getValidationStatus() │                          │
  │    returns: response=95  │                          │
```

**Three validation tiers — security proportional to value:**

| Tier | Method | Cost | Use Case | How It Works |
|------|--------|------|----------|-------------|
| 1 — Social | Reputation only | Free | Pizza ordering, low-value tasks | Check reputation scores, no validation needed |
| 2 — Crypto-Economic | Staked re-execution | Medium | Data analysis, medium-value | Validators stake ETH, re-execute work, get slashed if wrong |
| 3 — Cryptographic | zkML / TEE | High | Medical diagnosis, large trades | Mathematical proof of correct execution |

**The escrow pattern — this is the killer app:**
```
IF (ValidationRegistry.read(jobHash) == TRUE)
THEN releasePayment()
```

A smart contract locks funds in escrow. Payment only releases when the Validation Registry confirms the work was done correctly. No human arbitration needed. This is trustless agent commerce.

## Deployed Contracts

Live on Ethereum mainnet (Jan 29, 2026) and 20+ networks:

| Contract | Mainnet Address |
|----------|----------------|
| Identity Registry | `0x8004A169FB4a3325136EB29fA0ceB6D2e539a432` |
| Reputation Registry | `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` |

Also deployed on: Base, Arbitrum, Polygon, Optimism, Avalanche, Celo, Gnosis, Linea, Scroll, Taiko, Monad, BSC + testnets.

## How It Connects to the Protocol Stack

```
┌───────────────────────────────────────────────────────┐
│  ERC-8004 (Trust Layer)                                │
│  "Who is this agent? Should I trust it?"               │
│                                                        │
│  Identity ──── Reputation ──── Validation              │
│  (who)         (track record)   (proof of work)        │
├───────────────────────────────────────────────────────┤
│  Payment Layer                                         │
│  x402 (micropayments) │ AP2 (mandates) │ ACP (checkout)│
├───────────────────────────────────────────────────────┤
│  Coordination Layer                                    │
│  A2A (agent-to-agent tasks)                            │
├───────────────────────────────────────────────────────┤
│  Tool Layer                                            │
│  MCP (agent-to-tool access)                            │
└───────────────────────────────────────────────────────┘
```

**Concrete integrations:**

| Protocol | How ERC-8004 Connects |
|----------|----------------------|
| **A2A** | Registration file lists A2A endpoint. Agent Card discovery enhanced by onchain identity verification |
| **MCP** | Registration file lists MCP endpoint. V2 spec adding deeper MCP support |
| **x402** | `x402Support: true` flag. Reputation feedback includes `proofOfPayment` with tx hash |
| **AP2** | Mandate system can reference `agentId` for accountability. Validation Registry provides proof for dispute resolution |

**The full lifecycle:**
1. Agent registers in Identity Registry → gets `agentId` NFT
2. Publishes registration file with A2A/MCP endpoints
3. Client discovers agent via registry query (filter by capability, trust model)
4. Client delegates task via A2A
5. Agent executes using MCP tools
6. Agent pays for services via x402
7. Client submits reputation feedback to Reputation Registry
8. For high-value work: client requests validation via Validation Registry
9. Escrow releases payment based on validation result

## Authorship

| Author | Organization | Role |
|--------|-------------|------|
| Marco De Rossi | MetaMask | Identity/wallet |
| Davide Crapis | Ethereum Foundation | Protocol design |
| Jordan Ellis | Google | A2A integration |
| Erik Reppel | Coinbase | x402/payment integration |

Ethereum Foundation's dAI (Decentralized AI) team designated ERC-8004 as strategic infrastructure.

## Tradeoffs and Problems

### Real Issues

| Problem | Severity | Detail |
|---------|----------|--------|
| Sybil attacks | High | Malicious actors create multiple identities to inflate reputation. Partially mitigated by client filtering, but not solved |
| Identity ≠ capability | High | Having an NFT proves you *exist*, not that you're *good*. Registration file can claim anything |
| Validator incentive design | High | ERC-8004 intentionally punts on incentive/slashing design. "Left to specific validation protocols" |
| Gas costs at scale | Medium | Even on L2, millions of feedback entries and validation requests add up |
| Storage exhaustion | Medium | Unbounded validation requests store pending tuples indefinitely. DoS vector |
| Off-chain dependency | Medium | Registration files, feedback details, validation evidence all live off-chain. If IPFS pins disappear, metadata is lost |
| Collusion networks | Medium | Agents can amplify each other's reputations. Feedback loops distort trust signals |
| V2 still in development | Medium | Enhanced MCP support and improved x402 integration not shipped yet |

### What ERC-8004 Intentionally Doesn't Do

The spec is deliberately minimal. It does NOT handle:
- **Payments**: No native payment mechanism (x402/AP2 handle this)
- **Pricing**: No marketplace or pricing logic
- **Business models**: No revenue share or commission structure
- **Reputation algorithms**: Only raw signals — aggregation is off-chain
- **Validator economics**: No staking/slashing built in

This is a design choice, not a limitation: "The smartest thing a new foundational protocol can do is remain unopinionated about the layers above it."

### The Chicken-and-Egg Problem

For ERC-8004 to be useful, agents need to register AND clients need to check the registry. But:
- Why would agents register if no one checks?
- Why would clients check if few agents are registered?

This is the classic network effect bootstrap problem. Ethereum Foundation's dAI team is actively pushing adoption through builder programs and conferences, but it's still early.

## Alternatives

| Alternative | What | Trade-off |
|-------------|------|-----------|
| A2A Agent Cards alone | Self-declared capabilities | No independent verification, no reputation history |
| AP2 allowlists | Manual curation of trusted agents | Doesn't scale, centralized gatekeeping |
| ENS + attestations | Domain-based identity | No reputation or validation framework |
| Worldcoin/Proof of Humanity | Human identity verification | Agents aren't humans — wrong primitive |
| Custom reputation systems | Platform-specific trust | Siloed, non-portable, vendor-locked |
| ERC-7007 (Verifiable AI) | ZK proofs for AI inference | Focused on model verification, not agent identity |

## Steal These Patterns

| Pattern | What | Why It Matters |
|---------|------|----------------|
| NFT-as-identity | ERC-721 token = portable, composable agent passport | Works with existing NFT infra — wallets, indexers, marketplaces |
| Hybrid on/off-chain | Onchain: pointers + hashes. Off-chain: full data | 95%+ cost reduction vs full onchain storage |
| Client-filtered reputation | Consumers choose whose feedback to trust | Sybil resistance without permissioned access |
| Tiered validation | Social → Crypto-economic → Cryptographic | Security proportional to value at risk |
| Escrow-on-validation | Smart contract releases payment only after verified work | Trustless agent commerce — no human arbitration |
| proofOfPayment in feedback | Include tx hash proving reviewer paid for service | Separates real users from fake reviewers |
| Domain verification | `/.well-known/agent-registration.json` | Proves agent controls its claimed HTTPS endpoint |
| Progressive validation | Multiple responses per requestHash with tags | "Soft finality" → "hard finality" for staged verification |

## Bottom Line

**What's real**: ERC-8004 is deployed on mainnet and 20+ L2s. The contracts work. The Identity Registry is straightforward ERC-721. The specification is thoughtful and minimal.

**What's not ready**: The ecosystem. Very few agents have registered. Reputation aggregation services don't exist yet. Validator networks with proper incentive design are theoretical. V2 with deeper MCP/x402 integration is still in development.

**The honest assessment**: ERC-8004 is the right *shape* for agent identity — NFT-based, composable, with layered trust. But it's infrastructure waiting for adoption. The escrow-on-validation pattern is genuinely powerful and could become the foundation for trustless agent commerce. Whether that happens on ERC-8004 specifically, or a successor, depends on ecosystem momentum in 2026-2027.

**What to do today**: If you're building agents, register in the Identity Registry. It costs one transaction and makes your agent discoverable. Don't wait for the reputation ecosystem to mature — early registrants benefit from first-mover positioning when the trust layer gets traction.
