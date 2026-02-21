# Cloudflare 2025-2026: The Full-Stack Edge Cloud Deep Analysis

## Executive Summary

Cloudflare evolved from CDN/security into a **vertically-integrated AI infrastructure platform**. FY2025 revenue $2.17B (+30%), Q4 accelerated to 34%. Three acquisitions in 3 months (Replicate, Astro, Human Native). Shipped Containers, Agents SDK, x402 protocol, post-quantum Zero Trust, 31.4 Tbps DDoS mitigation record. 60+ products on 335+ global locations. The bet: become the default infrastructure layer for the agentic internet.

---

## 1. Workers & Containers

### Workers Runtime

V8 isolate-based serverless compute. Major 2025 Node.js compatibility push — native runtime implementations (not polyfills) for `node:fs`, `node:net`, `node:tls`, `node:http`, `node:process`.

**Workers Builds** GA (Birthday Week 2025). New resource-oriented REST API (beta) manages Workers/Versions/Deployments as separate resources.

**Strategic shift**: Pages deprecated in favor of Workers. All future investment in Workers — now supports static assets + SSR. Astro, React Router, Hono, Vue, Nuxt, Svelte all supported.

| Tier | Cost | Included |
|------|------|----------|
| Free | $0 | 100K req/day, 10ms CPU/invocation |
| Paid | $5/mo | 10M req/mo + $0.50/M, 30s CPU, **I/O wait free** |

### Containers (Open Beta, June 2025)

Docker at the edge, built on Durable Objects infrastructure. Scale-to-zero, 10ms billing increments.

**Architecture**: Request → Worker (API Gateway/routing/auth) → Durable Object (programmable sidecar) → Container (Docker image). Workers serve as Gateway, Service Mesh, or Orchestrator.

| Type | RAM | vCPU | CPU Price |
|------|-----|------|-----------|
| Dev | 256 MB | 1/16 | $0.00002/vCPU-sec |
| Basic | 1 GB | 1/4 | $0.00002/vCPU-sec |
| Standard | 4 GB | 1/2 | $0.00002/vCPU-sec |

Scale: up to 400 GiB RAM, 100 vCPUs, 2 TB disk. Auto-sleep when idle. **No GPU support yet**. Linux/amd64 only.

### Sandbox SDK

Workers + Durable Objects + Containers for **secure untrusted code execution**. Shell commands, Python/JS interpreters, file system, Git, background processes, public URL exposure. Official Claude Code template:

```bash
npm create cloudflare@latest -- claude-code-sandbox \
  --template=cloudflare/sandbox-sdk/examples/claude-code
```

Accepts repo URL + task description → provisions sandbox → runs Claude Code → returns diff.

**Workers for Platforms**: Run customer/AI-generated code in isolated Workers with per-customer security boundaries.

---

## 2. AI Stack

### Workers AI

Serverless GPU inference in 190+ cities. 50+ models. $0.011/1K Neurons. OpenAI-compatible API.

| Model | Params | Notes |
|-------|--------|-------|
| Llama 4 Scout | 17B MoE | Multimodal, latest Meta |
| Llama 3.3 70B | 70B | 128K context |
| Mistral Small 3.1 | 24B | Vision + tool calling |
| gpt-oss-120b | 120B | OpenAI, high reasoning |
| gpt-oss-20b | 20B | OpenAI, low latency |

Features: LoRA adapters, function calling, streaming, batch workloads, Responses API for OpenAI models.

### Infire Engine (v0.5.0)

Custom **Rust inference engine** replacing Python-based vLLM:
- Granular CUDA Graphs: JIT-compiles per-batch-size graphs, **82% CPU overhead reduction**
- Paged KV Caching: non-contiguous memory blocks prevent fragmentation
- 40.91 req/s vs vLLM's 38.38 on H100 NVL, only 25% CPU load (vs vLLM 140%)
- Llama-3-8B loads in <4 seconds

### Replicate Acquisition (Nov 2025)

50,000+ production-ready models joining Workers AI. Model marketplace + Cloudflare's edge network = distributed inference at scale.

### Agents SDK (v0.5.0, Feb 2026)

Stateful AI agents on Durable Objects. Each agent = identity + state + single-threaded execution + WebSocket.

| Layer | Technology | Role |
|-------|-----------|------|
| Runtime | Durable Objects | Stateful micro-servers, single-threaded |
| Storage | Embedded SQLite | 1GB/instance (10GB GA), zero-latency |
| Comms | WebSocket + HTTP | Real-time with hibernation |
| Inference | Infire (Rust) | Custom edge GPU engine |

Core capabilities: `this.state` auto-sync, `this.schedule()` cron, WebSocket hibernation, AI model calls (Workers AI/OpenAI/Anthropic via AI SDK), MCP server/client via `McpAgent`, email receive/reply, task queues, `this.retry()` with backoff.

**Key design**: Single-threaded Durable Objects = no race conditions. Multiple inputs queued and processed atomically. Eliminates concurrency bugs plaguing other agent frameworks.

**Agent + Workflow pattern**: Agents alone for chat/quick calls. Agent + Workflow for long-running tasks (30s+), multi-step pipelines, human-in-the-loop.

### MCP on Workers

First-class remote MCP server hosting. Each session backed by Durable Object (persistent state).

**Transport**: Streamable HTTP (replaced SSE, March 2025). **Auth**: Worker acts as both OAuth client (upstream) and OAuth server (to MCP clients) — double-proxy pattern scopes tokens to specific tools.

**Code Mode**: For large APIs (Cloudflare has 2,500+ endpoints), traditional tool definitions = ~1.17M tokens. Code Mode uses 2 tools (`search()` + `execute()`) = **~1,000 tokens — 99.9% reduction**. LLM writes TypeScript against typed SDK in sandboxed V8 isolate.

**Elicitation** (Aug 2025): MCP servers request user input during tool execution. State preserved across hibernation.

### AutoRAG (AI Search)

Managed RAG pipeline, open beta April 2025. R2 bucket → auto-index → query with LLM. Zero-config.

```
R2 → File Ingestion → Markdown Conversion → Image Processing
  → Chunking → Embedding → Vectorize Storage
  → Query → Rewriting → Vector Search → Response Generation
```

10 instances/account, 100K files/instance. **NLWeb integration** (Microsoft): `/ask` + `/mcp` endpoints for conversational search. Any website becomes MCP-compatible data source.

### AI Gateway

Unified proxy for 24+ AI providers (OpenAI, Anthropic, Azure, Bedrock, Vertex, Groq, Perplexity, xAI, DeepSeek...). Single OpenAI-compatible `/chat/completions` endpoint.

| Feature | Details |
|---------|---------|
| Caching | Identical requests cached at edge |
| Rate Limiting | Per-gateway, configurable |
| Model Fallback | Auto-failover to alternate providers |
| Analytics | Dashboard for volume, latency, costs |
| Unified Billing | Pay through Cloudflare invoice (2026) |

Core features (analytics, caching, rate limiting) **free on all plans**.

### Vectorize

IVF + Product Quantization indexing. Cosine/Euclidean/Dot Product distance. Metadata filtering with `$in`/`$nin`. 50K indexes/account, 50K namespaces/index.

Free: 30M queried dimensions/mo, 5M stored. No charges for CPU, memory, or "active index hours."

---

## 3. Developer Platform

### Durable Objects

Stateful micro-servers — foundation for Agents SDK, Containers, Queues. **SQLite storage GA**: 10GB/object, zero-latency, full SQL. PITR: 30-day point-in-time recovery.

**Free tier** (April 2025): 100K req/day, 5GB storage. WebSocket hibernation: agent sleeps without disconnecting clients. WebSocket message size: 32 MiB.

### D1 (Serverless SQLite)

**Global read replication** (beta): automatic replicas in every region. REST API latency -50-500ms. Worker API latency -40-60%.

### R2 (Object Storage) — Zero Egress

| Metric | Standard | Infrequent Access |
|--------|----------|-------------------|
| Storage | $0.015/GB/mo | Lower |
| Class A | $4.50/M | $9.00/M |
| Class B | $0.36/M | $0.90/M |
| **Egress** | **$0** | **$0** |

vs S3: 10TB stored + 10TB egress/month = $1,121 (S3) vs $150 (R2) = **87% savings**. Free: 10GB storage, 1M Class A, 10M Class B.

### Hyperdrive

Connection pooling + query caching for external databases. **MySQL support** (April 2025). PlanetScale partnership (Sep 2025) for one-click Postgres/MySQL.

### Queues

Message queue, P50 write latency ~60ms (down from ~200ms). Max 5,000 msg/sec per queue. $0.40/M operations beyond 1M free.

### Pipelines

Streaming ingestion (Arroyo acquisition, Rust stream processor). Workers/HTTP → DO buffer → Arroyo SQL transforms → R2 Iceberg/Parquet/JSON. ~100K records/sec, exactly-once delivery. **Beta, free during beta.**

### Workflows (GA April 2025)

Durable execution. 3 step types: `step.do()` (execute with retry), `step.sleep()` (pause seconds-to-days), `step.waitForEvent()` (block for webhook). CPU-time only billing — idle wait is free. **Python beta** (Nov 2025).

### Browser Rendering (GA)

Puppeteer v22.13.1 + Playwright v1.57.0 + **Stagehand** (AI-powered automation via Workers AI). 10 concurrent browsers/account. Session persistence up to 10min inactive. Billing started Aug 2025.

### Secrets Store (Beta, April 2025)

Centralized encrypted secrets across all Workers. RBAC, audit logging. Secrets accessed via bindings at runtime — no API key leakage.

### Wrangler v4 (March 2025)

esbuild v0.24, local-first (all resource commands default local), TypeScript type generation from bindings, Node.js v18+. Minimal v3→v4 migration.

---

## 4. x402 Protocol

Co-founded with Coinbase (Sep 2025). HTTP 402 Payment Required for machine-to-machine micropayments.

```
Client → GET /resource
Server → 402 Payment Required (amount, recipient, facilitator)
Client → GET /resource + X-PAYMENT header
Facilitator verifies → settles on-chain
Server → 200 OK + resource
```

**75M transactions, $24M processed by Dec 2025.** Currently USDC on Base.

**Deferred Payment Extension**: Batch settlements, subscriptions without per-request blockchain overhead. Agents SDK integration: `withX402()` + `paidTool()` server-side, `withX402Client()` client-side.

**AI Crawl Control**: Detects crawlers → 402 response with pricing → x402 payment → content served. Automated monetization. Bot management for free to Project Galileo (~750 journalists).

---

## 5. Security & Networking

### DDoS — Record-Breaking Year

| Quarter | Peak Attack | Notable |
|---------|-------------|---------|
| Q1 2025 | 6.5 Tbps | 20.5M attacks blocked (+358% YoY) |
| Q3 2025 | 29.7 Tbps | Largest ever at time |
| Q4 2025 | **31.4 Tbps** | New all-time record, 35 seconds |

5,376 attacks mitigated per hour average. HTTP DDoS 200M+ req/sec. Aisuru botnet (1-4M hosts) primary threat.

### Post-Quantum Cryptography — Industry First

CRYSTALS-KYBER (NIST-approved). **43% of human web traffic** post-quantum protected. Zero Trust PQC (March 2025). All WARP client traffic tunneled over PQC. Automatic PQC handshakes with origins (Q4 2025).

### Zero Trust / SASE

Gartner Visionary for SASE, SSE named vendor 3rd year. Cloudflare One suite: Access (ZTNA), Gateway (SWG), Browser Isolation, CASB (now scans ChatGPT/Claude/Gemini), DLP, Magic WAN, Email Security.

**AI additions**: CASB for AI (misconfig scanning), AI Prompt Protection, Firewall for AI (content moderation), MCP Server Portals (open beta).

---

## 6. Media & Email

| Product | Status | Key Feature |
|---------|--------|-------------|
| Email Service | Private beta (Sep 2025) | Send from Workers via bindings, no API keys. Auto SPF/DKIM/DMARC |
| Media Transformations | GA (Sep 2025) | Unified image+video billing, $0.50/1K transforms |
| Stream | GA | Live Clipping API |
| Browser Rendering | GA | Puppeteer + Playwright + Stagehand |
| Calls | Open beta | WebRTC real-time media |
| Realtime + RealtimeKit | 2025 | Audio/video SDKs (Kotlin, React Native, Swift, JS, Flutter) |

---

## 7. Acquisitions (3 in 3 Months)

| Company | Date | What | Why |
|---------|------|------|-----|
| Replicate | Nov 2025 | 50K+ AI models, deployment platform | Workers AI model catalog expansion |
| Human Native | Jan 15, 2026 | AI data marketplace (DeepMind/Google alumni) | Content monetization for AI era |
| Astro | Jan 16, 2026 | Open-source web framework (Unilever, Visa) | Own the framework-to-edge stack |

**Pattern**: Human Native (content economics) + Replicate (AI models) + Astro (developer framework) = vertically integrated AI development platform.

Also: **VibeSDK** (open source, Sep 2025) — MIT-licensed AI "vibe coding" platform. React+Vite + Workers + D1 + R2 + KV. Uses Gemini models, runs generated apps in Containers.

---

## 8. Financial Performance

### FY2025

| Metric | Value | YoY |
|--------|-------|-----|
| Revenue | $2,167.9M | +29.8% |
| Q4 Revenue | $614.5M | +33.6% |
| Paying Customers | 332,000 | +40% |
| >$100K ARR | 4,298 | +23% |
| >$1M ARR | 269 | +55% |
| RPO | $2,496M | +48% |
| Largest Deal | $42.5M ACV | Record |
| Q4 FCF | $99.4M (16.2%) | +108% |
| Net Retention | ~114% | Stable |

Revenue growth **re-accelerated** 3 consecutive quarters to 34% — rare at this scale.

### FY2026 Guidance

Revenue $2.79B (+28-29%). Operating income $378-382M (14% margin). EPS $1.11-1.12.

### Stock (NET)

~$192.64, market cap ~$67.4B. 52-week: $89-$260. P/S ~31x. Analyst consensus Buy, avg target $229-234.

---

## 9. Competitive Position

| Feature | Cloudflare Workers | Vercel Edge | Lambda@Edge | Deno Deploy |
|---------|-------------------|-------------|-------------|-------------|
| Runtime | V8 Isolates | V8 Isolates | KVM microVMs | V8 Isolates |
| Cold Start | <5ms | Minimal | 100ms-1s+ | <5ms |
| Locations | 335+ | ~20 regions | 13 regions | 35+ |
| Storage | R2, KV, D1, DO | Blob, KV, Postgres | S3, DynamoDB | KV |
| AI Inference | Workers AI (built-in) | Via API | SageMaker | Via API |
| Containers | Yes (beta) | No | Fargate | No |
| Egress | $0 | Standard | Standard | Standard |

**Workers: 210% faster than Lambda@Edge, 9x faster cold starts.**

**vs Vercel**: Vercel optimizes framework-level (Next.js ISR), Cloudflare optimizes network-level (latency, security, cache). Astro acquisition = reaching into Vercel's framework territory.

**Key differentiator**: Only platform offering isolates + containers + managed DB + vectors + AI inference + email + media + security in one integrated stack with zero egress.

---

## 10. Product Map (60+ Services)

All products run on same 335+ location network, same hardware. Adding a product = "flip a switch."

**Application Services**: CDN, DNS, WAF, DDoS, Bot Management, AI Crawl Control, SSL/TLS, Page Shield, API Shield, Turnstile, Load Balancing, Argo Smart Routing, Images, Stream, Zaraz, Observatory

**Developer Platform**: Workers, Workers AI, AI Gateway, Containers, R2, D1, KV, DO, Queues, Workflows, Hyperdrive, Vectorize, Browser Rendering, Realtime, Secrets Store, Pipelines

**Network Services**: Magic WAN, Magic Transit, Magic Firewall, Spectrum, Network Interconnect, WARP, Tunnel

**Zero Trust (Cloudflare One)**: Access, Gateway, Browser Isolation, CASB, DLP, Email Security, DEM

**Open Source**: Pingora (Rust proxy, 1T+ req/day, 70% less CPU than NGINX), workerd (JS/Wasm runtime), 547 repos

---

## 11. Honest Assessment

### Strengths
- **Integrated stack**: Every product talks to every other natively. Zero glue code
- **Zero egress**: R2 saves 87-99% vs S3
- **Scale-to-zero**: Containers, Workers, DO all hibernate — pay only for active compute
- **MCP-first**: Best remote MCP hosting story. OAuth, state, elicitation built-in
- **Post-quantum leadership**: 43% traffic protected while most companies haven't started
- **Developer velocity**: Single `wrangler deploy` handles everything

### Weaknesses
- **Memory**: 128MB for Workers (vs Lambda 10GB)
- **Container maturity**: Beta, no GPU, no exec, no global autoscaling
- **V8 constraints**: No native compiled languages (Rust/Go = Wasm only)
- **D1 scale**: SQLite — not a Postgres replacement at serious scale
- **SASE position**: Visionary, not Leader (Zscaler/Palo Alto ahead)
- **Vendor lock-in**: Durable Objects, D1, KV have no portable equivalents
- **Infire maturity**: Only Llama 3.1 8B currently, multi-GPU on roadmap
- **Gross margin declining**: 73.6% (from 76.4%) due to GPU investment

### Strategic Risks
- AWS/Google can match features with deeper CapEx
- Vercel owns Next.js mindshare
- 31x P/S requires sustained 25%+ growth for years
- x402 is crypto-adjacent — regulatory uncertainty
- 3 acquisitions in 3 months = integration risk

---

## 12. Stealable Patterns

### Architecture
1. **Durable Object as Agent Primitive**: identity + state + single-threaded + WebSocket. Eliminates distributed state bugs
2. **Code Mode for Large APIs**: 2 tools (`search` + `execute`) against typed SDK. 99.9% token reduction vs full tool definitions
3. **MCP OAuth Double Proxy**: MCP server = OAuth client (upstream) + OAuth server (to agents). Scoped tokens limit blast radius
4. **Worker-as-Gateway-to-Container**: Cheap isolate handles routing/auth/cache, forwards heavy compute to scale-to-zero container
5. **x402 Monetization Loop**: AI Crawl Control → 402 → payment → content served. Automated monetization
6. **Infire Granular CUDA Graphs**: JIT per-batch-size graphs instead of one-size-fits-all. 82% CPU reduction
7. **Hibernation-Aware State**: `serializeAttachment`/`deserializeAttachment` survives agent sleep without WebSocket disconnect

### Business
8. **Innovation Week cadence**: 5+ themed launch weeks/year. Creates press, developer excitement, shipping rhythm
9. **Land with security, expand to compute**: DDoS = immediate pain. Once traffic flows through you, compute has near-zero marginal cost
10. **"Anti-lock-in" as lock-in**: Zero egress attracts customers who then build deeply on the platform

---

## Latest Updates (2026)

### Moltworker: Self-Hosted AI Agents at the Edge (Feb 7, 2026)

Cloudflare released **Moltworker**, an open-source proof-of-concept for running Moltbot (formerly Clawdbot) -- a self-hosted personal AI agent -- on the Cloudflare Developer Platform. Eliminates the need for dedicated local hardware (e.g., Mac minis). Architecture: entrypoint Worker (API router + admin layer) + isolated Sandbox containers (Moltbot runtime). Integrates AI Gateway for multi-provider model routing, Browser Rendering for headless Chromium automation, and Zero Trust Access for authentication. Persistent state stored in R2. Demonstrates the full Cloudflare stack working together as an agent hosting platform.

### Docker-in-Docker for Containers & Sandboxes (Feb 17, 2026)

Containers and Sandboxes now support **Docker-in-Docker** -- running Docker inside a container. Key use cases: sandboxed development environments for agents, isolated container image testing, CI/CD image builds within containers, and deploying arbitrary user-supplied images at runtime. This is a significant capability upgrade for AI coding agents that need full development environment control.

### Queues on Free Plan (Feb 4, 2026)

Cloudflare Queues is now available on the **Workers Free plan**. Free tier includes 10,000 operations/day (reads, writes, deletes), up to 10,000 queues per account, guaranteed message delivery to Workers or HTTP pull consumers, unlimited event subscriptions. Only constraint: 24-hour max retention (vs 14 days on paid). Lowers the barrier to entry for message-driven architectures.

### GLM-4.7-Flash on Workers AI + TanStack AI (Feb 13, 2026)

New model addition: **GLM-4.7-Flash**, a fast multilingual text generation model with multi-turn tool calling support. Shipped alongside `@cloudflare/tanstack-ai` package and `workers-ai-provider` v3.1.1, bringing full compatibility with TanStack AI and Vercel AI SDK. Enables building agentic applications entirely at the edge with mainstream AI SDK tooling.

### AI Search Rebrand + External Model Support

AutoRAG officially rebranded to **AI Search** with an expanded mission. Now supports external models from OpenAI and Anthropic via AI Gateway provider keys. New canonical export `createAISearch` (replacing `AutoRAG`). Both embedding and generation models can come from external providers, breaking the lock-in to Workers AI models only.

### Container Scaling Expansion

Container concurrency limits significantly raised: up to **1,000 dev instances** or **400 basic instances** concurrently (previously 50/25). Maximum specs remain 400 GiB RAM, 100 vCPUs, 2 TB disk. GPU support still in internal development on the underlying container platform but not yet exposed to Containers product.

### Cloudflare for Students + Startup Hubs

Free Workers Paid plan for US university students with `.edu` email (12 months). Starting January 2026, approved startups building on Cloudflare can use office coworking spaces in **Austin, Lisbon, London, and San Francisco** on select days. 1,111 internship positions planned globally for 2026. $370M+ in startup credits distributed to 4,000+ startups.

---

## Key Dates Timeline

| Date | Event |
|------|-------|
| Feb 2025 | Agents SDK initial launch |
| Mar 2025 | Post-quantum Zero Trust, Wrangler v4 |
| Apr 2025 | Developer Week: AutoRAG, D1 replication, DO free tier, Secrets Store, Workflows GA, MySQL Hyperdrive |
| Jun 2025 | Containers open beta |
| Aug 2025 | NLWeb partnership, Sandbox SDK update, OpenAI models on Workers AI |
| Sep 2025 | Birthday Week: Email Service, VibeSDK, x402 Foundation, Workers Builds GA, Pingora replaces NGINX |
| Nov 2025 | Replicate acquisition, Container CPU pricing, Python Workflows |
| Jan 2026 | Human Native + Astro acquisitions |
| Feb 2026 | Agents SDK v0.5.0, FY2025 earnings ($2.17B), 31.4 Tbps DDoS record, Moltworker, Docker-in-Docker, Queues Free, GLM-4.7-Flash |

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
