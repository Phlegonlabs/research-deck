# Progress

## [2026-02-21] Full Update & Reference Consolidation — All 25 Topics
- **Structural change**: Merged `references.md` into `en.md` and `zh.md` as `## References` section at bottom of each file, then deleted all 25 `references.md` files
- **Content update**: Added `## Latest Updates (2026)` / `## 最新動態 (2026)` section to all 25 topics with fresh web research on 2025-2026 developments
- **Execution**: 5 batches × 5 parallel agents = 25 agents total, all completed successfully
- **Verification**: Zero `references.md` remaining, all `en.md`/`zh.md` have References + Updates sections
- Topics updated: a2a-ap2-agentic-commerce, acp-stripe-openai, ai-coding-models-2026, claude-code-agent-team, claude-code-best-practices, claude-code-memory-system, cloudflare-2026, codex-sandbox-architecture, discord-ai-agent-swarms, erc-8004-agent-identity, geo-generative-engine-optimization, jito-solana, jupiter-solana, mcp-x402-integration, openai-codex-cli, openclaw-ai-agent, parallel-coding-agents, shenzhen-robotics-ecosystem, sherwin-wu-openai, solana-blockchain, threejs-ecosystem, threejs-visual-design, unitree-robotics, voxyz-autonomous-agents, web4-agentic-web
- Files modified: 50 (25× en.md + 25× zh.md), 25 deleted (references.md)
- Next steps: none — consolidation complete

## [2026-02-21] Cloudflare 2025-2026 — Full-Stack Edge Cloud Complete Deep Analysis
- Evolution: CDN/security → vertically-integrated AI infrastructure platform. FY2025 $2.17B revenue (+30%), Q4 accelerated to 34%. 60+ products on 335+ global locations
- Workers: native Node.js APIs (fs/net/tls/http/process), Pages deprecated. Containers (open beta Jun 2025): Docker on DO, scale-to-zero, 10ms billing, up to 400GiB/100vCPU. Sandbox SDK for untrusted code (official Claude Code template)
- AI stack: Workers AI (50+ models, 190+ cities, $0.011/1K neurons), Infire Rust engine (82% CPU reduction vs vLLM), Agents SDK v0.5.0 (DO-based stateful agents, single-threaded = no race conditions), MCP on Workers (Code Mode: 99.9% token reduction), AutoRAG/AI Search (zero-config RAG + NLWeb), AI Gateway (24+ providers, free caching/rate limiting), Vectorize
- x402 protocol (Coinbase co-founded): HTTP 402 micropayments. 75M txns, $24M by Dec 2025. Deferred payment extension. AI Crawl Control → 402 → payment → content (automated monetization)
- Security: 31.4 Tbps DDoS record, post-quantum 43% traffic protected (CRYSTALS-KYBER), Gartner Visionary SASE, CASB for AI (ChatGPT/Claude/Gemini scanning), Firewall for AI
- Acquisitions (3 in 3 months): Replicate (50K+ AI models, Nov 2025), Human Native (AI data marketplace, Jan 2026), Astro (web framework, Jan 2026). Pattern: own AI-era full dev stack
- Storage: R2 zero egress (87-99% cheaper than S3), D1 global read replication, DO free tier + 10GB SQLite GA, Hyperdrive MySQL, Queues 5K msg/sec, Pipelines (Arroyo, 100K records/sec)
- Financials: 332K paying customers (+40%), 269 $1M+ ARR (+55%), $42.5M largest ACV, 114% NRR. FY2026 guidance $2.79B (+28-29%). Stock ~$192, $67.4B cap, 31x P/S
- Competitive: Workers 210% faster than Lambda@Edge, 9x faster cold starts. Only platform with isolates+containers+DB+vectors+AI+email+media+security. vs Vercel (framework vs network optimization)
- Open source: Pingora (Rust proxy, 1T+ req/day, 70% less CPU than NGINX), workerd, 547 repos
- Key patterns: DO as agent primitive, MCP Code Mode (1K tokens for full API), Worker-as-Gateway-to-Container, x402 monetization loop, Innovation Week cadence (5+/year), "anti-lock-in" as lock-in (zero egress)
- Files: `cloudflare-2026/`
- Next steps: prototype Agents SDK agent with MCP + x402, evaluate Sandbox SDK for parallel Claude Code agents, test AutoRAG with R2 data

## [2026-02-19] Sherwin Wu — OpenAI API Engineering Leader & "Sorcerer" Thesis Deep Analysis
- Background: MIT BS & MEng CS (2010-2014), Palantir, Doximity, Quora (News Feed), Opendoor (5+ yrs Pricing ML), then OpenAI (~2022). Currently Head of Engineering, OpenAI API/Developer Platform. Also at Sancus Ventures
- Content catalog: 15 pieces across 4 podcasts (Lenny's, a16z, BG2, Latent Space), 1 conference talk + panel (QCon NY 2023), 5+ notable tweets, 2 LinkedIn posts, multiple media articles
- "Engineers are becoming sorcerers" thesis (Lenny's Podcast, Feb 2026): 95% of OpenAI engineers use Codex daily, 10-20 parallel agents per engineer, 100% PR review by AI, 70% more PRs, code review 10-15min → 2-3min
- Model portfolio over "god model": "Even within OpenAI, the thinking was that there would be one model that rules them all. It's definitely completely changed." Reinforcement fine-tuning (RFT) as the big unlock
- Forward Deployed Engineering (FDE): Palantir-style engineer embedding with enterprise customers (T-Mobile, Amgen, Los Alamos). Customer problems → model improvements feedback loop
- "Developers as distribution layer of AGI": OpenAI can't reach every vertical through ChatGPT alone — developers carry AGI into niches. 6B tokens/minute, five nines uptime target
- Startup thesis: bearish on one-person $1B startups, bullish on thousands of $10M niche businesses. "Golden age of B2B SaaS"
- Skill evolution: prompt engineering (2023) → function calling (2024) → context design (2025) → agent orchestration (2026)
- "Models will eat your scaffolding for breakfast" — don't over-invest in workarounds for current model limitations
- QCon 2023: "A Bicycle for the (AI) Mind: GPT-4 + Tools" — function calling debut, the primitive that evolved into Agents SDK
- Files: `sherwin-wu-openai/`
- Next steps: track Wu's future content as OpenAI platform evolves, monitor agent orchestration patterns from OpenAI engineering practices

## [2026-02-19] Jito — Solana's MEV Infrastructure & Liquid Staking Protocol Deep Analysis
- Jito-Solana: modified Agave validator client with 3 additional pipeline stages (RelayerStage, BlockEngineStage, BundleStage). 95-97% of Solana stake weight runs Jito-Solana by 2026
- Block Engine: off-chain partial-block bundle auction. Relayer holds txns 200ms for auction. Bundles = max 5 txns, atomic, sequential. $674M+ in tips paid to stakers/validators through 2024
- MEV on Solana: latency-driven (not gas-bidding like Ethereum), continuous block building vs discrete slots, known leader schedule creates co-location pressure
- Mempool controversy (March 2024): Jito shut down public mempool after sandwich attacks peaked at 10K SOL/day. Problem went underground to private mempools. $370-500M extracted by sandwich bots over 16 months post-shutdown
- BAM (Block Assembly Marketplace): launched July 2025. TEE-based transaction scheduling with Plugin architecture. Engineering solution to sandwich attacks (privacy by design, not policy)
- JitoSOL: largest Solana LST, ~40-76% market share, 15.1M SOL TVL. Yield = base staking (6-6.6%) + MEV tips (1.2-1.8%) = 7.2-8.4% APY. VanEck filed S-1 for JitoSOL ETF (Aug 2025)
- Restaking: NCN/Vault/Operator model. $227M TVL across 9 VRTs. TipRouter = first live NCN (Jan 30, 2025), decentralizes Merkle root computation for tip distribution
- JTO token: 1B supply, -94.7% from ATH ($5.32 → $0.30). JIP-24 (Aug 2025) redirects 100% of Block Engine + BAM fees to DAO treasury. a16z invested $50M (Oct 2025)
- Team: Lucas Bruder (CEO, CMU, ex-Tesla/Built Robotics) + Zano Sherwani (CTO, GMU, ex-Amazon). Founded Sep 2021. $60M+ raised total
- Key pattern: MEV + LST flywheel — MEV infrastructure → higher staking yield → more stake → more MEV coverage → repeat. Self-reinforcing network effect
- Centralization tension: 95%+ stake on one client = systemic risk + economic lock-in (not running Jito = -13-15% rewards). Genuine progressive decentralization steps (TipRouter, JIP-24, StakeNet, BAM) but economic gravity makes alternatives hard
- Files: `jito-solana/`
- Next steps: monitor BAM adoption vs Harmonic competition, NCN ecosystem growth beyond TipRouter, VanEck JitoSOL ETF approval, Firedancer-BAM integration progress

## [2026-02-19] Jupiter — Solana's DeFi Super-App Deep Analysis
- Product suite: DEX aggregator (95% market share) + Perps (250x leverage, 66% Solana perps share) + DCA + Limit Orders + JLP Pool + Lock + Terminal + Mobile + Ape (meme trading) + LFG Launchpad
- Routing evolution: Metis (Bellman-Ford) → Iris (Golden-section + Brent's method, 100x faster) → Juno (meta-aggregator: Iris + JupiterZ RFQ + 3rd party). JupiterZ RFQ: $100M/day zero-slippage market maker execution
- MEV protection: Ultra V3 achieves 34x better sandwich protection via Iris routing splits + JupiterZ off-AMM execution + Jito validator direct submission + dynamic slippage
- JLP pool: 5-asset ($2.5B TVL), counterparty to perps traders, 75% of fees to LPs, 14-20% APY typical, max -18% drawdown. 80% held by 10 whales (concentration risk)
- Oracle stack: Edge (Chaos Labs, primary) → Chainlink → Pyth, multi-oracle failover with threshold checks
- JUP token: 7B supply (3B burned Jan 2025), $2.04 ATH → $0.17 current (-91.7%). 50% revenue buyback program. Litterbox burn: 130M JUP ($44.5M) burned Nov 2025
- Jupuary airdrops: 3 rounds (2024-2026), final reduced from 700M → 200M to limit dilution. Token unlock pressure overwhelms buybacks
- Governance: paused June 2025 due to "trust breakdown" between team and community, resumed 2026 with revised tools
- LFG Launchpad: 78 projects launched, DLMM fair-launch mechanism, DAO-curated. Notable: WEN, JUP, Zeus, Sanctum, deBridge
- Team: Meow (pseudonymous, OG DeFi) + Ben Chow (IDEO designer, resigned Feb 2025 amid LIBRA meme coin scandal). Bootstrapped 4+ years, first outside funding $35M ParaFi Capital (Feb 2026)
- Revenue: $509M gross (2025), ~$122M net protocol. Perps = 80% of revenue. Aggregator is free user acquisition → perps monetization funnel
- 2026: CatLumpurr conference — Jupiter Lend ($1B TVL in 8 days), Polymarket integration, Jupiter Global (fiat payments), Offerbook (P2P lending), Jupnet omnichain testnet
- Key paradox: every business metric trending up, token down 91.7%. Structural unlock pressure unsolved
- Centralization concern: 95% aggregator monopoly, Moonshot/SonarWatch acquisitions, team governance influence
- Stealable patterns: meta-aggregation (aggregate aggregators), RFQ+AMM hybrid, free core → paid perps, bootstrapped → strategic investment, conference-driven roadmap, named community (Catdets)
- Files: `jupiter-solana/`
- Next steps: monitor Jupnet mainnet timeline, JUP token unlock schedule impact, Jupiter Lend growth trajectory, governance reform results

## [2026-02-19] Solana Blockchain — Complete Technical & Ecosystem Deep Analysis
- 8 core innovations architecture: PoH (SHA-256 hash chain clock), Tower BFT (O(n) message PBFT), Turbine (BitTorrent-like shred propagation), Gulf Stream (mempool-less forwarding), Sealevel (parallel execution via explicit account declarations), TPU pipelining (GPU sig verify + CPU banking + kernel IO), Cloudbreak (RAID 0 accounts DB), Archivers (distributed storage)
- Performance reality: theoretical 65K TPS, actual 400-800 user TPS (60-70% are validator votes). 8 major outages 2021-2024, zero since Feb 2024
- Firedancer (Jump Crypto): C/C++ validator, mainnet Dec 2025, 1M+ TPS lab demo, 21%+ stake adoption. Frankendancer hybrid at 26%+ validators
- Alpenglow (SIMD-0326): biggest protocol change in history, replaces PoH + TowerBFT, 100-150ms finality target, 98.27% approval, Q1 2026 mainnet
- DeFi: $11.5B TVL (Q3 2025), Jito ($2.72B) + Kamino ($2.43B) + Jupiter ($2.39B) dominate. Jupiter controls 95% DEX aggregator share
- MEV: Jito bundles provide protection but 93% of sandwiches are "wide" (multi-slot), extracting 529K+ SOL/year. Problem unsolved
- Pump.fun: $800M lifetime revenue, 6M+ tokens launched, 98.6% rug-pull rate, 52.8% of Solana DEX transactions (Dec 2025)
- Token economics: ~4% inflation declining 15%/yr to 1.5% terminal, no supply cap, 67% staked, $82 SOL price (Feb 2026), ATH $294.85 (Jan 2025)
- Governance: 3-entity split (Foundation/Labs/Anza), validators dropped 68% to ~800, Nakamoto coefficient 20, 88% stake on Jito-Solana client
- ETF: First spot SOL ETF approved Oct 2025, 23 filings total, major issuers (Fidelity, Grayscale, VanEck, Franklin Templeton)
- DePIN: Helium ($9.5M/yr), Hivemapper (399M km), Render, io.net — $81M+ total node operator distributions
- Enterprise: Visa (USDC settlement), Shopify (Solana Pay), PayPal (PYUSD), Stripe, Franklin Templeton (on-chain fund)
- Key patterns: clock-before-consensus, explicit access declarations for parallelism, tree-based propagation, mempool elimination, code/state separation
- Honest assessment: best UX in crypto + composability advantage, but validator centralization (top 3 = 26% stake), VC concentration (38% initial to insiders), meme coin reputation risk
- Files: `solana-blockchain/`
- Next steps: monitor Alpenglow mainnet deployment Q1 2026, track Firedancer adoption growth, watch ETF inflow data

## [2026-02-19] Shenzhen Robotics Ecosystem — The World's Densest Humanoid Robot Cluster
- 74,000+ robotics enterprises, 34 listed companies, 9 unicorns, 18+ humanoid body manufacturers — all concentrated in Nanshan district
- Nanshan "Robot Valley" (Liuxian Avenue, 10km): same-day prototype-to-production, 90% local supply chain, 70% manufacturing within corridor
- UBTECH (9880.HK): 1,000 Walker S2 produced, 500+ delivered, 1.3B yuan orders, market cap HK$71B, 2026 target 5,000 units/year
- UBTECH acquiring Zhejiang Fenglong (43% for $237M) — supply chain vertical integration play
- LimX Dynamics: $200M+ funded (Alibaba, JD.com), Oli ($22.7K) + TRON 2 ($7K modular), 3-year Middle East expansion plan
- EngineAI: $210M funded (JD.com, CATL), 5 products in <2 years ($5.4K-$50K), ex-Xpeng founder, T800 heavy industrial at CES 2026
- Dobot Atom: $27.5K, ±0.05mm precision, third batch mass production (Feb 2026), cobot DNA → humanoid transition
- Leju Robot: $210M pre-IPO (Tencent-backed), KUAVO humanoid, targeting Shanghai STAR Market IPO
- Lumos ($28M angel), Cyborg R01 (62 DOF heavy-duty), DexForce W1 Pro (wheeled, 8hr battery), Daimon (tactile sensing, China Mobile backed)
- Government: 100B yuan fund for AI/robotics, 100B yuan industry target by 2027, 60% model training subsidies, FAIR expo, "Eight King Kongs"
- Supply chain advantage: PCB/batteries 90%+ local, manufacturing 2.2x cheaper than US, rare earths 90% global supply
- Key patterns: ship imperfect at volume, product price laddering, vertical integration as moat, government as patient capital GP
- Shenzhen vs Shanghai vs Hangzhou: Shenzhen = hardware density, Shanghai = international, Hangzhou = software/AI talent
- Files: `shenzhen-robotics-ecosystem/`
- Next steps: monitor UBTECH 2026 production ramp, Leju IPO filing, LimX Middle East deployments, ecosystem consolidation

## [2026-02-19] Unitree Robotics — The $16K Humanoid That Outshipped Everyone
- Founded 2016 by Wang Xingxing (ex-DJI), 4 people, $275K angel. Now $7B IPO-bound on Shanghai Star Market Q2 2026
- 5,500+ humanoids shipped in 2025 (30% of global market) — more than Tesla, Figure AI, Agility combined. Target: 20,000 in 2026
- Product line: R1 ($4,900), G1 ($13,500), H2 ($29,900), H1 ($90K+). Plus Go2/B2 quadrupeds, L1/L2 LiDAR, M107 actuators
- M107 joint motor: 189 N·m/kg torque density (claimed world record), self-developed, key cost advantage via vertical integration
- UnifoLM AI framework: WMA-0 (world model + action), VLA-0 (vision-language-action), X1-0 (industrial). Open-sourced on Hugging Face
- Data engine strategy: G1 humanoids assemble motor parts in Unitree factory → production = training data generation (flywheel)
- Spring Festival Gala 2026: fully autonomous swarm kung fu with G1/H2. World-first aerial flips (3m+), 7.5-rotation airflare, 4 m/s cluster speed
- China ecosystem: $8.2B national AI fund, 150+ humanoid companies, 90% of global humanoid market by volume
- Competition: vs AgiBot (5,100 shipped), Tesla Optimus (not for sale), Figure 03 (not for sale), Boston Dynamics Atlas (research only)
- Revenue: ~¥1B (~$140M), already profitable. 50% overseas quadruped sales. IPO with CITIC Securities
- Key bet: ship imperfect at volume → collect real-world data → close capability gap. Counter-strategy to Tesla/Figure's "perfect then ship"
- Files: `unitree-robotics/`
- Next steps: monitor Q2 2026 IPO filing, H2/R1 delivery quality, factory deployment ROI data

## [2026-02-19] ACP: Agentic Commerce Protocol — Stripe & OpenAI Implementation Deep Dive
- ACP: open standard by OpenAI + Stripe for AI agent commerce. 5 REST endpoints (Create/Retrieve/Update/Complete/Cancel Checkout)
- Shared Payment Token (SPT): Stripe's new payment primitive — single-use, time-limited, merchant+amount-scoped, credential-isolated
- Delegated Payment Flow: 3-phase credential delegation (OpenAI→Stripe→SPT→Merchant). Network tokens preferred over FPANs for PCI reduction
- ChatGPT Instant Checkout: live Feb 16 2026 US users. 800M+ WAU, 50M daily shopping queries
- Fee structure: 4% OpenAI + ~2.9% Stripe = ~9.2% combined (vs Amazon 25-30%, Google 0%)
- 4 onboarding paths: Stripe Direct ("one line of code"), Shopify (1M+ merchants), PayPal ACP Server, Direct Application
- Product feed: compressed JSONL/CSV, 15-min refresh, GTIN/UPC required, enable_search gates enable_checkout
- Google UCP as competing response: /.well-known/ucp discovery, supports REST/A2A/MCP. Dual ACP+UCP = 40% more traffic
- Spec evolution: 4 versions in 4 months (2025-09-29 → 2026-01-30). Date-based versioning, all versions preserved
- Real issues: US only, Stripe lock-in (only SPT PSP), 4% fee, no recurring payments, Amazon excluded via robots.txt
- Reference demo: locus-technologies/agentic-commerce-protocol-demo (TypeScript, Node.js, PostgreSQL, Docker)
- Files: `acp-stripe-openai/`
- Next steps: test ChatGPT Instant Checkout merchant onboarding, watch UCP for competitive convergence

## [2026-02-19] ERC-8004: Onchain Agent Identity, Reputation & Validation
- ERC-8004 "Trustless Agents": three onchain registries — Identity (ERC-721 NFT), Reputation (structured feedback), Validation (independent verification)
- Deployed on Ethereum mainnet (Jan 29 2026) + 20 L2s. Identity: 0x8004A169..., Reputation: 0x8004BAa1...
- Authors: MetaMask (identity), Ethereum Foundation (protocol), Google (A2A), Coinbase (x402)
- Identity Registry: NFT passport with off-chain registration file listing A2A/MCP endpoints, x402 support, trust models
- Reputation: raw signals (not scores), client-filtered getSummary(), proofOfPayment to prove reviewer paid for service
- Validation: 3 tiers — social (reputation only), crypto-economic (staked re-execution), cryptographic (zkML/TEE proofs)
- Killer pattern: escrow-on-validation — smart contract releases payment only after ValidationRegistry confirms correct work
- Hybrid on/off-chain: pointers + hashes onchain, full data offchain (95%+ cost savings)
- Integration: registration file lists A2A/MCP endpoints, x402Support flag, agentId referenced in AP2 mandates
- Problems: Sybil attacks (partial mitigation), identity ≠ capability, validator incentives unspecified, ecosystem adoption chicken-and-egg
- Files: `erc-8004-agent-identity/`
- Next steps: register a test agent in Identity Registry, watch V2 for deeper MCP/x402 integration

## [2026-02-19] A2A, AP2, ACP: The Agentic Commerce Protocol Stack
- Full protocol stack mapped: MCP (tools) → A2A (agent coordination) → AP2/ACP/x402 (payments) → ERC-8004 (identity)
- A2A: Google's agent-to-agent protocol. JSON-RPC 2.0 over HTTPS, Agent Card discovery (/.well-known/agent-card.json), task state machine, SSE streaming, 150+ partners
- A2A vs MCP: MCP = vertical (agent→tool), A2A = horizontal (agent→agent). Complementary, not competing
- AP2: Google's payment authorization protocol. Three mandate types: Cart (human present), Intent (human absent), Payment (network signals)
- AP2 security: 6-role separation — shopping agent NEVER sees raw payment credentials. Cryptographic mandates = non-repudiable authorization proof
- AP2 + x402: AP2 provides trust/authorization layer, x402 provides settlement. AP2 mandates can route to Visa OR x402 stablecoins
- ACP: OpenAI/Stripe's chat commerce protocol. Shared Payment Token = scoped, time-limited credential delegation. Powers ChatGPT Instant Checkout today
- ACP vs AP2: ACP = shipping, simple (4 actors), Stripe-first. AP2 = rigorous, complex (6 roles + mandates), V0.1 only
- "Mullet Economy": fiat front (B2C via ACP/AP2), crypto back (M2M via x402)
- Meta-problem: 6 protocols for one commerce transaction = real integration complexity
- McKinsey projects $3-5T agentic transaction volume by 2030
- Files: `a2a-ap2-agentic-commerce/`
- Next steps: prototype A2A Agent Card for a service, watch AP2 V1.x for autonomous mandates

## [2026-02-19] MCP + x402 Integration — How AI Agents Pay for Services
- x402 revives HTTP 402 status code: 3-phase payment handshake (Quote → Pay → Settle) embedded in HTTP headers
- EIP-3009 (Transfer With Authorization) underneath — currently USDC-only due to Circle's standard
- Facilitator pattern: servers don't need blockchain infra, Coinbase/Cloudflare handle verification + settlement
- 4 integration patterns: Coinbase SDK (@x402/axios), Vercel x402-mcp (paidTool), Cloudflare Workers, MCPay proxy
- V2 shipped: wallet-based sessions, auto-discovery, deferred settlement, modular @x402/* SDK, multi-chain
- Coinbase Agentic Wallets: TEE-isolated private keys, session caps, per-tx limits, KYT screening — LLM never sees keys
- Protocol stack: MCP (tools) + A2A (agent coordination) + x402 (crypto payments) + AP2 (fiat) + ERC-8004 (identity)
- "Mullet Economy": fiat front (B2C via AP2/ACP), crypto back (M2M via x402)
- Centralization irony: "open standard" but Coinbase dominates SDK, facilitator, wallet infra, and foundation
- Bootstrap problem unsolved: wallet creation + USDC funding still requires human KYC
- Files: `mcp-x402-integration/`
- Next steps: prototype x402 paid MCP tool on Base Sepolia testnet

## [2026-02-19] Web 4.0: The Agentic Web — Deep Analysis
- Three competing definitions: crypto-native (Sigil Wen/Dragonfly), enterprise (EU/Gartner), pragmatic (Will Hackett)
- Core thesis: AI agents become independent economic actors — bottleneck is permission, not intelligence
- Key infrastructure: x402 protocol (50M+ txns, Coinbase + Cloudflare), MCP tool access, crypto wallets for agents
- Conway/Automaton: MCP-compatible infra giving agents wallets, servers, deployment — no human logins
- 7-level agent maturity framework: we're at Level 3-4, Web 4.0 assumes Level 5+
- Six-layer academic framework: Environmental → Infrastructure → Data → Agent → Behavioral → Governance
- Honest assessment: underlying needs (payments, identity, tools, coordination) are real; "Web 4.0" branding is crypto-aligned marketing
- Skeptic points: Web3 déjà vu, semantic web's 20yr failure, 90% blockchain failure rate, regulatory vacuum
- Files: `web4-agentic-web/`
- Next steps: monitor x402 protocol adoption, evaluate MCP + wallet patterns for agent autonomy

## [2026-02-16] OpenAI Codex CLI — Sandbox & Isolation Architecture Deep Dive
- Deep analysis of Codex CLI's OS-level sandbox implementation across macOS, Linux, Windows
- macOS Seatbelt: sandbox-exec with SBPL profiles, deny-by-default, dynamic profile generation in Rust
- Linux Landlock + seccomp-BPF: filesystem ACL via Landlock LSM (kernel >= 5.13), syscall filtering blocks SYS_connect/bind/listen/sendto + io_uring, AF_UNIX exempted for IPC
- Windows: experimental AppContainer + restricted tokens via CreateRestrictedToken(), proactive denial of ADS/UNC/device handles, PATH injection defense with stub executables
- arg0 dispatch pattern: single binary re-executes itself as codex-linux-sandbox via argv[0] matching, eliminates separate helper binaries
- 3 sandbox modes (read-only / workspace-write / danger-full-access) + 4 approval policies (untrusted / on-failure / on-request / never)
- writable_roots config: expand write boundary via config.toml or --add-dir CLI flag, .git/ and .codex/ always read-only
- Network isolation: disabled by default, binary all-or-nothing on all platforms, no per-domain filtering possible
- Execpolicy: Starlark-based semantic command control with allow/prompt/forbidden three-tier decisions
- Enterprise enforcement: /etc/codex/requirements.toml non-overridable, MDM support, cloud policy fetch
- CVE-2025-59532 (CVSS 8.6): model-generated cwd treated as writable root, fixed in v0.39.0
- Key weakness: full filesystem read in all modes (security concern #4410), macOS network_access config bug (#10390)
- Comparison with Claude Code: Codex has kernel-level sandbox + default network blocking; Claude Code uses permission deny-rules + devcontainers
- 10 stealable patterns: arg0 dispatch, deny-by-default allowlists, dual kernel primitives, AF_UNIX exemption, env sanitization, serialized policy handoff, .git protection, enterprise ceiling, output capping, graduated escalation
- Files: `codex-sandbox-architecture/`
- Next steps: evaluate Trail of Bits sandboxing for Claude Code vs Codex approach

## [2026-02-16] OpenAI Codex CLI — Complete Usage Guide & Community Insights
- Comprehensive research: full installation flow, all CLI commands/flags, 24 slash commands, AGENTS.md system, advanced config.toml
- Approval modes: read-only (default) → auto (--full-auto) → full-access → YOLO (--yolo)
- Sandbox: OS-level (seatbelt/landlock), network blocked by default, configurable writable_roots
- Models: gpt-5.3-codex (default), spark (Pro only), 5.2-codex medium/high/xhigh (cost vs reasoning tradeoff)
- AGENTS.md = CLAUDE.md equivalent: 3-tier hierarchy (global → project → nested), 32KB limit, immediate effect
- Key differentiator vs Claude Code: Codex Cloud (async cloud tasks + local apply), `codex exec` for CI automation
- Prompting guide: bias to action, batch parallel reads, preserve codebase patterns, no "AI slop" for frontend
- Community consensus: Codex 2-3x more token-efficient, Claude Code better for deep codebase understanding
- Optimal workflow: Claude Code (understand/plan) → Codex (execute/iterate) → Claude Code (review)
- 8 stealable patterns: test-driven agent loop, instruction layering, profile workflows, exec CI, cloud+local hybrid, multi-tool, image-driven UI, incremental patching
- Files: `openai-codex-cli/`
- Next steps: set up Codex CLI alongside Claude Code for multi-tool workflow

## [2026-02-13] GEO: Generative Engine Optimization — How to Get Cited by AI Search
- Comprehensive research: how to optimize content for ChatGPT, Perplexity, Claude, Google AI Overviews
- GEO = new SEO for AI search. Goal: get **cited**, not just ranked. Google/AI overlap dropped from 70% → <20%
- GEO paper results: +40% visibility overall, +115% for lower-ranked sites with citation strategy
- Technical requirements: robots.txt (allow GPTBot, OAI-SearchBot, PerplexityBot, ClaudeBot), SSR, no paywalls
- Content patterns: answer-first ("answer capsule"), structured content (lists 3x citation rate), FAQ schema, statistics with sources
- Platform differences: ChatGPT (Wikipedia-heavy, Bing index), Perplexity (Reddit-heavy, real-time indexing), Google AI (52% from top-10)
- Schema priority: Article + FAQ + Organization + Person (author), JSON-LD format
- Monitoring tools: Otterly.ai, Geoptie, Relixir — track citation frequency, AI share of voice, sentiment
- GPTBot dilemma: training (GPTBot) vs search (OAI-SearchBot) — many publishers allow search but block training
- Key insight: **被引用比被排名更重要** — AI 时代的流量是品牌提及，不是点击
- Files: `geo-generative-engine-optimization/`
- Next steps: audit own sites' robots.txt, add FAQ schema, implement answer-first writing

## [2026-02-13] Claude Code Memory System — Complete Architecture Deep Dive
- Comprehensive research: all memory mechanisms, persistence, context window interaction, best practices
- 6 memory layers: Managed Policy > Project Memory > Project Rules > User Memory > Local Memory > Auto Memory
- Auto Memory: `~/.claude/projects/<project>/memory/MEMORY.md` — first 200 lines loaded per session, topic files on-demand
- Agent Memory (v2.1.33, Feb 2026): subagent-specific persistent memory with user/project/local scopes
- CLAUDE.md imports: `@path` syntax, recursive up to 5 hops, approval dialog for external imports
- Project Rules: `.claude/rules/*.md` with YAML frontmatter `paths` field for conditional loading (glob patterns)
- Skills: three-level progressive disclosure (frontmatter always -> SKILL.md on-invoke -> supporting files on-navigate)
- Context budget: ~167K usable of 200K (33K reserved for auto-compact buffer), memory files consume ~7.4K tokens
- Auto-compaction at ~83.5% usage, manual `/compact` with preservation instructions, `/clear` between tasks
- Known issue: MEMORY.md double-loading bug (#24044) wastes ~3KB/call
- Key insight: **context freshness > context accumulation** — keep CLAUDE.md under 150 lines, use progressive disclosure
- Best practice: WHAT/WHY/HOW structure, don't use LLMs as linters, explicit remember commands
- Files: `claude-code-memory-system/`
- Next steps: optimize own CLAUDE.md with progressive disclosure pattern, set up path-specific rules

## [2026-02-07] AI Coding Models 2026 — OpenAI vs Anthropic Complete Landscape
- Consolidated research: all OpenAI + Anthropic coding models, head-to-head comparison, developer ecosystem
- Feb 5 dual release: Claude Opus 4.6 vs GPT-5.3-Codex (released 20 min apart)
- OpenAI: 9+ models (GPT-4.1 family, o3/o4-mini, GPT-5/5.1/5.2/5.3-Codex) + Codex platform (app + CLI + API)
- Anthropic: 7 models (Opus 4/4.1/4.5/4.6, Sonnet 4/4.5, Haiku 4.5) + Claude Code CLI ($1B revenue)
- Claude leads: SWE-bench Verified (80.8%), OSWorld (72.7% ≈ human), GDPval-AA (+144 Elo)
- GPT-5.3 leads: Terminal-Bench 2.0 (77.3%, +12pp over Claude), speed (25% faster)
- Neither wins all benchmarks — strategic benchmark variant selection by both companies
- METR bombshell: AI tools made experienced devs 19% slower (opposite of their 20-24% self-estimate)
- Open source closing fast: Qwen3-Coder-Next (3B active/80B) at 70.6% SWE-bench
- Community consensus: multi-tool workflow (Cursor + Claude Code + Copilot + Codex)
- Files: `ai-coding-models-2026/`
- Next steps: test Agent Teams, monitor Terminal-Bench 2.0 as unified benchmark

## [2026-02-07] Parallel Coding Agents — Architecture Guide
- Consolidated research: how to run N coding agents in parallel in the cloud
- Covers Cloudflare (Workers + Containers + Sandbox SDK), Warp (Namespace + Ambient Agents), E2B, Daytona, Modal, Sprites, Northflank, GitHub Codespaces
- Universal pattern: Trigger → Orchestrate → Execute (N sandboxes) → Observe
- Key insight: sandbox layer is commoditizing (Rivet SDK), value moves to orchestration and intent specification
- Warp's "Workers" = generic concept (Namespace containers), NOT Cloudflare Workers
- Cloudflare CAN run parallel agents via Containers + Sandbox SDK (official Claude Code tutorial exists)
- 5 architecture patterns: ephemeral-per-task, fork-and-explore, persistent workstation, Cloudflare stack, Warp ambient
- 8 stealable patterns: orchestrator≠executor, sidecar volumes, fork-snapshot, shared cache, scoped credentials, progressive autonomy, friction-point activation, TOEO
- Decision framework with cost analysis (4 agents × 8hr/day × 20 days: $11-$115/mo depending on platform)
- Files: `parallel-coding-agents/`
- Next steps: prototype parallel agent system with Cloudflare Sandbox SDK or Daytona

## [2026-02-07] Three.js Visual Design
- Deep analysis of how to make Three.js scenes look cinematic — from lightsaber glow to photorealistic rendering
- Key insight: 80% of visual quality from 5 things — tone mapping, HDRI environment, bloom, color space, shadows
- Lightsaber effect breakdown: emissive core + glow shader + Polyboard trail + selective bloom via Layers
- Complete "cinematic recipe": 4-line renderer setup → HDRI → materials → shadows → post-processing
- Visual effects cookbook: volumetric light (radial blur), dissolve (noise discard), god rays, fog, color grading/LUT
- 5 layers of visual quality: Default → Foundation (5min) → Lighting (15min) → Post-processing (30min) → Polish (hours) → WebGPU
- Files: `threejs-visual-design/`
- Next steps: build a lightsaber demo to validate the full stack

## [2026-02-07] Three.js Ecosystem
- Deep analysis of Three.js in 2025-2026: from demo toy to production 3D platform
- Key findings: WebGPU production-ready (r171+, all browsers), 10-150x performance gains, TSL node-graph shader system
- Architecture: scene graph + Object3D hierarchy, lazy init, shader caching, needsUpdate flags
- TSL deep dive: JavaScript → Node AST → GLSL/WGSL compilation, automatic cross-platform shaders
- Ecosystem: R3F + drei + rapier + postprocessing + zustand + leva = complete 3D app stack
- Compared with Babylon.js (batteries-included engine) and PlayCanvas (cloud editor)
- Real-world: browser MMORPGs, award-winning portfolios, IKEA-level product configurators, generative art
- Files: `threejs-ecosystem/`
- Next steps: build something with R3F + WebGPU to validate patterns

## [2026-02-07] VoxYZ Autonomous AI Company
- Researched how VoxYZ built 6 autonomous AI agents with OpenClaw + Vercel + Supabase
- Key patterns: closed-loop orchestration, gate-before-queue, trigger/reaction matrix, self-healing heartbeat
- Files: `docs/voxyz-autonomous-agents/`
- Next steps: explore more agentic coding patterns

## [2026-02-07] Claude Code Agent Team
- Researched Claude Code's native multi-agent orchestration system (Agent Teams / Swarm Mode)
- Source: @YukerX Twitter thread + official docs + multiple deep-dive articles
- Key patterns: parallel specialists, competing hypotheses, self-organizing swarm, plan-approve-execute, file ownership boundaries
- Architecture: TeammateTool (13 ops), file-based coordination (JSON on disk), task dependency auto-unblocking, hook-based quality gates
- Compared with LangGraph, CrewAI, OpenAI Swarm, claude-flow
- Files: `claude-code-agent-team/`
- Next steps: try Agent Team on a real multi-file refactoring task

## [2026-02-13] OpenClaw AI Agent — Full Architecture Deep Dive
- Source: OpenClaw official docs + GitHub releases + IBM/DigitalOcean/Sapt analysis + CyberSecurity News
- Latest v2026.2.6 (Feb 7, 2026): Opus 4.6, GPT-5.3-Codex, xAI Grok, Baidu Qianfan, safety scanner, token dashboard
- Architecture: Brain (LLM) → Memory (flat-file Markdown) → Arms (fs_tool, bash_tool, browser_tool CDP)
- Agentic loop: Think → Plan → Act → Observe → Iterate
- Skills system: SKILL.md declarative format, YAML frontmatter + natural language, ClawHub marketplace
- QMD memory plugin: BM25 + Vector + LLM Rerank, 60-97% token savings, fully local
- 145K+ GitHub stars, 50+ integrations, 100+ AgentSkills, Web3/Base onchain integration
- Security: full system access = critical risk without containerization; IBM says enterprise-hostile
- Key insight: Gateway-as-control-plane is the converging pattern for personal AI agents
- Files: `openclaw-ai-agent/`
- Next steps: compare OpenClaw skill system with Claude Code hooks/MCP for extensibility patterns

## [2026-02-13] Claude Code Best Practices — Production Workflows & Advanced Techniques
- Comprehensive research: configuration patterns, terminal integration, context optimization, multi-agent workflows
- CLAUDE.md best practices: What/Why/How structure, ~150-200 instruction limit, start simple and iterate
- Terminal-first approach: Ghostty (GPU rendering, <500MB/instance) beats IDE extensions (8GB+)
- Context management: `/clear` aggressively, `/compact` before major work, write handoff documents
- Key insight: **Context freshness > context accumulation** — LLM output quality degrades with length
- tmux integration: session persistence, parallel Claude instances, notification→pane jumping
- Three-layer sandboxing (Trail of Bits): /sandbox + permission deny-rules + devcontainers
- Hooks architecture: PreToolUse/PostToolUse interceptors, blocking hooks for dangerous commands
- Stealable patterns: git worktrees parallel dev, Gemini CLI fallback, write-test autonomous cycle
- MCP essentials: Context7 (library docs), Exa (web/code search)
- Model strategy: Opus (complex), Sonnet (default), Haiku (simple) — auto-switch at 50% usage
- Files: `claude-code-best-practices/`
- Next steps: implement tmux + Ghostty workflow, set up Trail of Bits sandboxing

## [2026-02-08] Discord AI Agent Swarms — Full Architecture Deep Dive
- Source: @jumperz X article "I Built an AI Agent Swarm in Discord" + Zach Wills' 20-agent swarm case study + broader multi-agent architecture research
- Core thesis: Discord's primitives (channels, threads, reactions, webhooks) map perfectly onto multi-agent coordination — eliminating custom orchestration infra
- Architecture patterns: Channel-as-Queue, Thread-as-Work-Item, Webhook-based, MCP + Discord, Consensus-based swarms
- Zach Wills' 8 rules: plan alignment, restart ruthlessly, checkpoint to files not agent history, sub-agents per phase, trust autonomous loops, self-updating CLAUDE.md, frequent commits
- Key metrics: ~800 commits, 100+ PRs, $6K/week, ~$75/PR, 3-hour human orchestration ceiling
- Broader context: Hybrid architectures dominate production (hierarchy is load-bearing for scale), MoA achieves 65.1% AlpacaEval, optimal team 3-7 agents
- Framework comparison: LangGraph (fastest), CrewAI (easiest), OpenAI Swarm (educational), Claude Code Teams (task-based)
- Discord advantages: zero infra, human-in-the-loop by default, debugging = reading conversations
- Discord limits: 50 req/sec rate limit, 100-500ms latency, not for production scale
- Files: `discord-ai-agent-swarms/`
- Next steps: prototype Discord-based agent swarm, test with code review workflow
