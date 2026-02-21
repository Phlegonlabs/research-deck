# Jupiter: Solana's DeFi Super-App — Deep Technical & Ecosystem Analysis

> Jupiter started as a DEX aggregator. It became Solana's financial operating system. This is how.

## Table of Contents

1. [Product Deep Dive](#1-product-deep-dive)
2. [Architecture & Technical Stack](#2-architecture--technical-stack)
3. [Jupiter Ape & Meme Coin Trading](#3-jupiter-ape--meme-coin-trading)
4. [JUP Token](#4-jup-token)
5. [Jupiter DAO & Governance](#5-jupiter-dao--governance)
6. [LFG Launchpad](#6-lfg-launchpad)
7. [Team & Background](#7-team--background)
8. [Business Model & Revenue](#8-business-model--revenue)
9. [Market Position & Competition](#9-market-position--competition)
10. [2025-2026 Developments](#10-2025-2026-developments)
11. [Honest Assessment](#11-honest-assessment)
12. [Stealable Patterns](#12-stealable-patterns)

---

## 1. Product Deep Dive

Jupiter is no longer just a DEX aggregator. It is a vertically integrated DeFi super-app on Solana with seven distinct product lines, each serving a different user segment but sharing liquidity and user base.

### 1.1 DEX Aggregator — The Core

**What it does**: Finds the best swap price across all Solana DEXs (Raydium, Orca, Meteora, Phoenix, Lifinity, etc.), splitting and routing orders through multi-hop paths.

**How routing works**:

| Generation | Engine | Algorithm | Key Improvement |
|------------|--------|-----------|-----------------|
| V1-V2 | Legacy | Simple graph traversal | Basic multi-DEX routing |
| V3-V5 | Metis | Modified Bellman-Ford | Incremental streaming, split+merge at any stage |
| V6 | Metis 1.5 | Enhanced Bellman-Ford | 5.22% better prices vs V2, parallel quoting |
| Ultra V3 | Iris + Juno | Golden-section search + Brent's method | 100x pathfinding perf, meta-aggregation |

**Metis core mechanics**:
- Treats Solana's entire liquidity as a directed graph (tokens = nodes, pools = edges)
- Streams tokens incrementally through routes, allowing splits and merges at any stage
- Same DEX can appear in multiple legs of a single route
- Merges route generation and price quoting into a single operation (eliminates dead-end exploration)
- Handles Solana's 64-account lock limit (max 4 DEXs per swap instruction)

**Iris** (current, Ultra V3): Replaces Metis with advanced mathematical optimization — Golden-section search and Brent's method for optimal order splitting. 100x performance improvement in pathfinding vs Metis.

**Juno** (meta-aggregator layer): Sits above all routing engines. Aggregates quotes from Iris, JupiterZ (RFQ), and third-party sources (DFlow, Hashflow, OKX). Self-learning mechanism automatically sidelines underperforming or suspicious quote sources.

**JupiterZ** (RFQ system): Connects users directly with professional market makers for top token pairs. ~$100M daily volume with zero slippage. Market makers provide firm quotes; no AMM curve involved.

### 1.2 Limit Orders

Users set a target price; Jupiter's keeper network monitors on-chain prices via Jupiter Price API and executes when the market reaches the specified price.

**Keeper mechanics**:
- Keepers function like liquidators in lending protocols
- Monitor on-chain prices continuously
- Execute orders using Jupiter's aggregation engine (so fills get best routing too)
- Platform fee: 0.1% on fills

### 1.3 DCA (Dollar Cost Averaging)

Automated recurring purchases of any SPL token over a configurable interval.

**How it works**:
1. User deposits full amount into a program-owned vault (e.g., 1000 USDC to DCA into SOL)
2. First order executes immediately
3. Remaining orders execute at user-defined intervals
4. Each fill uses Jupiter's aggregation engine
5. Orders have ±2-30 second randomized timing to prevent exploitation
6. Platform fee: 0.1% per execution

### 1.4 Jupiter Perps (Perpetual Futures)

**Model**: LP-to-trader (not order book). Traders open leveraged positions against the JLP pool.

**Supported markets**: SOL, ETH, BTC — up to 250x leverage.

**Oracle pricing**: Trades execute at oracle price, not AMM price. This means:
- No price impact from large trades (oracle reflects global market price)
- A price impact fee is artificially imposed to simulate slippage and protect LPs
- Three oracles: Edge (Chaos Labs, primary), Chainlink (backup), Pyth (backup)
- Multi-oracle failover: if primary is stale or deviates beyond threshold, falls back to secondary pair

**Fee structure**:
- Opening/Closing fee: 6 bps (0.06%) of position size
- Borrow fee: hourly, based on pool utilization — `(Tokens Locked / Tokens in Pool) × Hourly Rate × Position Size`
- No funding rates (unlike most perps exchanges)

**Liquidation**: Position liquidated when collateral minus fees plus/minus PnL < 0.2% of position size.

### 1.5 JLP (Jupiter Liquidity Provider Pool)

The counterparty pool for perpetual traders. Holding JLP = being the house.

**Pool composition (target weights)**:

| Asset | Target Weight | Role |
|-------|--------------|------|
| SOL | 44% | Long collateral |
| ETH | 10% | Long collateral |
| wBTC | 11% | Long collateral |
| USDC | 26% | Short collateral + stable |
| USDT | 9% | Short collateral + stable |

**Yield sources**:
- 75% of all perps fees (opening, closing, borrowing, price impact)
- Native SOL staking yield (~7% APR, added August 2025)
- Realized APY: 14-20% in stable periods, historically peaked at 120%+

**Risks**:
- JLP holders are the counterparty — if traders collectively profit, JLP loses
- Maximum historical drawdown: -18% (August 5 2025 market crash)
- 80% of JLP supply held by top 10 whales (concentration risk)
- Dynamic swap fees adjust based on pool asset deviation from target weights

**Scale**: $2.5B+ TVL (October 2025).

### 1.6 Jupiter Lock (Token Vesting)

Open-source, audited (Sec3 + OtterSec) program for token vesting schedules. Anyone can create a lock with a custom vesting plan. Used by LFG launchpad projects for team token locks.

### 1.7 Jupiter Terminal (Embeddable Swap Widget)

Drop-in React component or vanilla JS widget that any dApp can embed for swap functionality.

**Integration modes**:
- NPM package: `@jup-ag/terminal`
- Window object injection (simplest)
- Display modes: integrated, widget, or modal

**Key feature**: Terminal can operate without any RPCs — Ultra handles transaction sending, wallet balances, and token info. For dApps without an existing wallet provider, Terminal provides its own wallet adapter.

### 1.8 Jupiter Mobile

Full-featured Solana wallet + DeFi app (iOS + Android).

**Stats**: 1.1M+ users, 380K+ total downloads, 4.9/5 rating.

**Features**:
- All swap/DCA/limit order functionality
- Web3 browser with security scans
- Apple Pay crypto onboarding
- Email/Apple ID login (no seed phrase required)
- Jupiter Global: fiat on/off ramp, Jupiter Card, QR Pay across APAC

---

## 2. Architecture & Technical Stack

### 2.1 Routing Engine Evolution

```
V1-V2 (2021-2022): Simple graph traversal → basic multi-DEX
     ↓
Metis (2023-2024): Modified Bellman-Ford → streaming, split/merge, 5.22% better
     ↓
Metis 1.5 (Apr 2025): Enhanced for Ultra → lessons inform Iris design
     ↓
Iris (Oct 2025): Golden-section + Brent's method → 100x faster pathfinding
     ↓
Juno (Oct 2025): Meta-aggregator → combines Iris + JupiterZ + 3rd party
```

**Why this evolution matters**: The aggregator war is won on latency. Solana blocks every ~400ms. The faster you can compute optimal routes, the more likely the quoted price still exists when the transaction lands.

### 2.2 Smart Contract Architecture

Jupiter's on-chain programs are written in Rust using the Anchor framework on Solana. Key programs:

- **Swap program**: Executes the routed swaps with multi-hop CPI (Cross-Program Invocation) calls across AMMs
- **Perps program**: Manages positions, collateral, liquidations, oracle integration
- **DCA program**: Manages vaults, scheduling, execution
- **Limit order program**: Order storage, keeper authorization, execution
- **Lock program**: Vesting schedules, cliff/linear/custom curves

Solana constraint: 64-account lock limit per instruction means max ~4 DEX hops per swap. Metis/Iris handle this by finding optimal routes within this constraint.

### 2.3 MEV Protection

MEV (sandwich attacks) is a critical problem on Solana due to low transaction costs making attacks cheap.

**Jupiter's multi-layer approach**:

| Layer | Mechanism | Protection Level |
|-------|-----------|-----------------|
| MEV Protect Mode | Sends txns directly to Jito block engine (bypasses public mempool) | Strong |
| Iris routing | Splits large orders across pools, reducing single-pool price impact | Moderate |
| Dynamic slippage | Statistically optimized slippage per-route, per-token | Moderate |
| JupiterZ RFQ | Off-AMM execution via market makers, zero slippage | Strong |
| Ultra V3 | 34x better sandwich protection vs previous versions | Strong |

**Ultra V3's MEV defense**: Combines all layers — preferentially routes through JupiterZ (zero-slippage market maker execution), uses dynamic slippage that is tighter than user-set, and sends via Jito validators. Result: 34x reduction in sandwich attack surface.

### 2.4 Keeper Network

Jupiter's limit orders and DCA use a keeper network (similar to Chainlink Keepers or Gelato):

- Keepers monitor on-chain prices via Jupiter Price API
- When conditions are met, keepers submit execution transactions
- Keepers are incentivized by the spread between fill price and limit price
- Orders have randomized timing (±2-30s) to prevent keeper front-running

### 2.5 Oracle Integration

**Perps oracle stack**:
1. **Edge (Chaos Labs)** — primary oracle, purpose-built for perps
2. **Chainlink** — backup, cross-checked against Edge
3. **Pyth** — backup, used as sanity check and fallback

**Selection logic**: If Edge is fresh and within threshold of both Chainlink and Pyth → use Edge. If Edge is stale or deviated → compare Chainlink vs Pyth → use latest of the two. If 2/3 oracles fail → no price update (positions safe from bad data).

### 2.6 Developer APIs & SDKs

| API | Purpose | Key Feature |
|-----|---------|-------------|
| Ultra Swap API | Simplified swap lifecycle | Quote → Order → Sign → Execute |
| V6 Swap API | Granular route control | Custom instructions, CPI integration |
| Price API V3 | Reference pricing | Real-time prices for all tokens |
| Tokens V2 API | Token discovery | Verified token lists, metadata |
| Terminal SDK | Embeddable swap | React component, zero-config |

**SDK ecosystem**: Official Rust client (`jup-swap-api-client`), community Python SDK, C# wrapper. Self-hosted V6 API available for high-volume integrators.

---

## 3. Jupiter Ape & Meme Coin Trading

### 3.1 How Jupiter Became the Meme Coin Hub

Jupiter's aggregation naturally routes through meme coin liquidity pools on Raydium, Orca, and Meteora. When pump.fun tokens graduate to Raydium, Jupiter is the first aggregator to surface them.

**The flywheel**: Pump.fun launches token → token graduates to Raydium → Jupiter aggregates the pool → traders use Jupiter to buy → volume flows through Jupiter.

Jupiter processes 55%+ of Raydium trades, and Raydium is where most pump.fun tokens land. This makes Jupiter the de facto trading layer for meme coins.

### 3.2 Jupiter Ape Platform

Launched as a dedicated meme coin trading interface with safety features:

**Core features**:
- **Rugcheck integration**: Every token scored for safety, suspicious tokens flagged "Danger"
- **Dedicated trading vault**: Isolated from user's main wallet (security layer)
- **Live token feed**: New launches from Raydium, Orca, Meteora in real-time
- **Quick buy**: Optimized for speed (critical for meme coin sniping)

**Philosophy**: Rather than fight the meme coin trend, Jupiter embraced it with safety rails. "Safer, smoother, and cheaper way to trade meme coins."

### 3.3 Volume Impact

Meme coin trading has been a massive volume driver for Jupiter. During peak meme seasons (late 2024, early 2025), daily volumes exceeded $5B, largely from meme coin activity. The TRUMP token launch alone drove billions in volume through Jupiter's infrastructure.

---

## 4. JUP Token

### 4.1 Tokenomics

| Metric | Value |
|--------|-------|
| Max supply (original) | 10B JUP |
| Burned (Catstanbul Jan 2025) | 3B JUP (~$3.6B) |
| Current total supply | 7B JUP |
| Circulating supply | ~3.24B JUP |
| ATH | $2.04 (Jan 31, 2024) |
| ATL | $0.136 (Feb 13, 2026) |
| Current price (Feb 2026) | ~$0.17 |
| Market cap (Feb 2026) | ~$540M |

**Distribution** (post-burn):
- Community/Airdrops: ~50%
- Team: ~20%
- Strategic reserve: ~15%
- Ecosystem/Grants: ~15%

### 4.2 Jupuary Airdrops

| Airdrop | Date | Amount | Recipients | Key Details |
|---------|------|--------|------------|-------------|
| Jupuary 1 | Jan 31, 2024 | ~1B JUP | ~900K wallets | First airdrop, massive |
| Jupuary 2 | Jan 22, 2025 | 700M JUP ($616M) | 2M wallets | 440M users, 60M stakers, 200M growth |
| Jupuary 3 | Jan 30, 2026 | 200M JUP | Active users | Reduced from 700M to 200M to limit dilution. 175M users, 25M stakers. Final Jupuary. |

**Eligibility criteria**: Swap volume, consistency (8+ months active = bonus), staking duration, governance vote participation.

### 4.3 Staking & Active Staking Rewards (ASR)

**Mechanism**: Stake JUP → get governance voting power (1 JUP = 1 vote) → earn ASR rewards.

**ASR reward pool**:
- 75% of LFG Launchpad fees
- 100M JUP tokens per year (distributed quarterly)

**How ASR works**:
- Daily snapshots of staked JUP amounts
- Quarterly reward distribution
- Rewards proportional to: average staked amount × governance participation rate
- Consistent staking and voting = higher rewards

### 4.4 Buyback Program

Announced at Catstanbul (January 2025): 50% of protocol fees go to JUP buybacks.

**Execution**: Jupiter repurchases JUP on open market (first buyback: 4.88M JUP / $3.33M on Feb 26, 2025). Bought tokens locked for 3 years in Litterbox Trust.

**Litterbox Burn**: DAO voted (86% approval, November 2025) to burn 130M JUP (~$44.5M) from the Litterbox Trust, removing 4% of circulating supply. Shift from accumulation to deflationary burns.

### 4.5 Price Performance — The Paradox

Jupiter is arguably the most successful DeFi product on Solana by every metric except token price. JUP hit $2.04 at launch hype, then declined 91.7% to $0.17 by February 2026 despite:
- $509M gross revenue (2025)
- $3B+ TVL
- 95% aggregator market share
- Active buyback program

**Why the disconnect**: Continuous token unlocks (team, airdrops) create selling pressure that overwhelms buyback demand. The Jupuary airdrops, while great for user acquisition, distribute hundreds of millions of tokens that get sold immediately.

---

## 5. Jupiter DAO & Governance

### 5.1 Structure

- **J.U.P. Promise**: Jupiter United Planet — the community governance framework
- **Catdets**: Community members (play on "cadets" + cats branding)
- **Working Groups**: Task-specific teams funded by DAO budgets for grants, ecosystem development
- **Vote mechanism**: 1 staked JUP = 1 vote, on-chain governance via vote.jup.ag

### 5.2 Key Governance Votes

| Vote | Outcome | Significance |
|------|---------|--------------|
| LFG Launchpad approval | Passed | Community-selected projects for token launches |
| 3B token burn | Passed | Largest supply reduction in Solana DeFi |
| Litterbox burn (130M JUP) | 86% approval | Shifted from accumulation to burns |
| Jupuary 3 reduction (700M → 200M) | Passed | Reduced dilution for final airdrop |

### 5.3 Governance Pause (June 2025)

Jupiter DAO paused all governance votes until 2026, citing:
- **"Breakdown in trust"** between team and community
- Perpetual FUD cycles around token emissions and team voting power
- Concerns about team's concentrated voting power influencing outcomes
- DAO, holders, and team "stuck in a negative feedback loop"

**Meow's statement**: "In 2026, we will return to governance with a fresh approach that unifies, rather than divides."

This is notable: one of DeFi's largest DAOs voluntarily shut down governance to redesign it. Staking rewards continued during the pause.

### 5.4 Governance Resumed (Early 2026)

At CatLumpurr (January-February 2026), Jupiter indicated governance would resume with revised tools and clearer separation of team vs community voting power.

---

## 6. LFG Launchpad

### 6.1 Mechanism

**LFG = "Let's F*cking Go"** — community-driven launchpad on Solana.

**How it works**:
1. Projects apply via Jupiter Research Forum
2. Catdet community reviews: traction, team quality, TGE readiness
3. DAO votes on candidates
4. Selected projects launch via **single-sided DLMM** (Dynamic Liquidity Market Maker) pool
5. Fair launch: everyone buys at the same on-chain price, no private sales or pre-seed allocations
6. Projects airdrop to existing community members

**DLMM mechanism**: Dynamically adjusts token price based on real-time demand and supply. Ensures continuous liquidity and reduces post-launch crashes. The DLMM pool is single-sided (only project tokens initially), with buyers providing the other side.

### 6.2 Notable Launches

| Project | Token | Category | Notable |
|---------|-------|----------|---------|
| WEN | $WEN | Memecoin | First LFG launch, test run |
| Jupiter | $JUP | Governance | Self-launch, massive airdrop |
| Zeus Network | $ZEUS | Cross-chain | BTC-Solana bridge |
| Sanctum | $CLOUD | Liquid staking | LST infrastructure |
| deBridge | $DBR | Cross-chain | Bridge protocol |
| Sharky | $SHARKY | NFT lending | NFT-fi |

**Scale**: 78 projects launched as of 2025, $1.2B in total value locked, 780K user base.

### 6.3 Comparison with Other Solana Launchpads

| Feature | Jupiter LFG | Pump.fun | Raydium AcceleRaytor |
|---------|------------|----------|---------------------|
| Curation | DAO-voted | Permissionless | Team-selected |
| Fair launch | Yes (DLMM) | Yes (bonding curve) | Variable |
| Rug pull risk | Low (vetted) | Extremely high (98.6%) | Medium |
| Volume | Moderate | Massive | Low |
| Community | Catdets | Degens | Traders |

---

## 7. Team & Background

### 7.1 Founders

**Meow (pseudonymous)**:
- Co-founder of Jupiter and Meteora (formerly Mercurial Finance)
- Advisor to Instadapp and Kyber Network
- Involved with BitGo and Ren in the launch of Wrapped Bitcoin (WBTC)
- OG DeFi background, connected to Solana ecosystem since early days
- Remains the public face of Jupiter despite pseudonymity

**Ben Chow**:
- Co-founder of Jupiter and Meteora
- Former product designer at IDEO (leading design firm)
- Founded WishWell, Friended, Minute Inc.
- **Resigned from Meteora in February 2025** amid LIBRA memecoin scandal allegations

### 7.2 The LIBRA Scandal

In February 2025, Meteora (co-founded by Meow and Ben Chow) was implicated in the LIBRA token controversy:

- Meteora's platform facilitated launches for HAWK (Haliey Welch), TRUMP, MELANIA, and LIBRA tokens
- LIBRA — briefly promoted by Argentine President Javier Milei — saw 74% of 44,000 buyers lose money
- Ben Chow accused of insider trading and receiving LIBRA tokens
- Chow resigned; Meow cited "lack of judgment and care"
- Fenwick & West (law firm) retained for independent investigation
- Meow stated: "No one at Jupiter or Meteora committed any insider trading or financial wrongdoing"

**Impact**: This scandal tested Jupiter's credibility. The response — swift resignation, external investigation, public statements — was textbook crisis management but the reputational damage lingered.

### 7.3 Funding

Jupiter was **bootstrapped** — no VC funding until February 2026 when ParaFi Capital invested $35M (the protocol's first outside investment in 4+ years of operation). This is remarkable for a protocol of Jupiter's scale.

**ParaFi deal terms**:
- $35M investment settled in JupUSD (Jupiter's dollar-pegged token)
- Market-priced token purchase, no discounts
- Extended lockup for ParaFi
- Warrants to acquire more tokens at higher prices
- ParaFi ($2B AUM) previously backed Aave, Compound, MakerDAO

---

## 8. Business Model & Revenue

### 8.1 Fee Structure

| Product | Fee | Who Pays | Who Receives |
|---------|-----|----------|-------------|
| Aggregator swaps | 10 bps (non-stable), 2 bps (stable) | Traders | JLP pool (dynamic) |
| Perps open/close | 6 bps | Traders | 75% JLP, 25% protocol |
| Perps borrow | Hourly, utilization-based | Traders | 75% JLP, 25% protocol |
| Limit orders | 10 bps | Traders | Protocol |
| DCA | 10 bps per execution | Traders | Protocol |
| LFG Launchpad | Launch fees | Projects | 75% ASR, 25% protocol |

### 8.2 Revenue Numbers

| Period | Metric | Amount |
|--------|--------|--------|
| 2025 Annual (gross) | Total fees | ~$509M |
| Q3 2025 | Revenue | $46M |
| Q3 2025 | Perps revenue | $24.6M (~53% of Q3) |
| Q3 2025 | Spot volumes | $176.8B |
| Projected annual protocol revenue | Net to protocol (30% of perps fees) | ~$122M |

**Key insight**: Perps generate ~80% of Jupiter's revenue. The aggregator (95% market share) is the user acquisition channel; perps are the monetization engine. This is a classic "free → premium" funnel.

### 8.3 Revenue Distribution

Starting February 17, 2025:
- 50% of protocol revenue → JUP buybacks (Litterbox Trust)
- 25% → Protocol treasury
- 25% → Team operations

### 8.4 How They Monetize "Free" Aggregation

Jupiter doesn't charge aggregator routing fees to integrators or individual swaps below certain thresholds. The aggregator builds:
1. **Habit** — users default to Jupiter for all swaps
2. **Data** — Jupiter sees all Solana swap flow, informing JupiterZ market maker quotes
3. **Upsell** — users who swap also use perps, DCA, limit orders (which all charge fees)
4. **JLP inflow** — swap fees on JLP pool entry/exit generate revenue

---

## 9. Market Position & Competition

### 9.1 Solana DEX Landscape

| Platform | Role | Market Share | Key Metric |
|----------|------|-------------|------------|
| Jupiter | Aggregator + Perps | 95% aggregator, 66% perps | $176.8B Q3 spot |
| Raydium | AMM (underlying liquidity) | ~55% of Jupiter-routed trades | $43B/month peak |
| Orca | Concentrated liquidity AMM | #3 by volume | 140% volume growth |
| Meteora | DLMM AMM | Rising, powers LFG launches | Growing share |
| Phoenix | Order book DEX | Niche | Professional traders |

**Critical nuance**: Jupiter is not "competing" with Raydium/Orca — it is routing through them. 55%+ of trades Jupiter routes execute on Raydium. Jupiter's dominance actually benefits underlying DEXs by driving volume to their pools.

### 9.2 vs Ethereum Aggregators

| Feature | Jupiter | 1inch | CoW Swap | ParaSwap |
|---------|---------|-------|----------|----------|
| Chain | Solana | Multi-chain (13+) | Ethereum | Multi-chain |
| Latency | ~400ms (1 Solana block) | 12s+ (Ethereum block) | Batched auctions | 12s+ |
| Gas cost | <$0.01 | $5-50+ | $5-50+ | $5-50+ |
| MEV protection | Jito validators + Iris | Flashbots, Fusion | Batch auctions (best) | MEV Blocker |
| Cross-chain | No (Jupnet planned) | Yes | No | Yes |
| Perps | Yes (250x) | No | No | No |
| Unique approach | RFQ + AMM hybrid (JupiterZ) | Fusion (intent-based) | Solver competition | MEV-focused |

**Why Jupiter wins on Solana**: Solana's speed + low fees make aggregation far more effective than on Ethereum. On Ethereum, gas costs often exceed the aggregation savings. On Solana, even a 0.5% price improvement is pure profit for the user.

### 9.3 Why Jupiter Won the Aggregator War on Solana

1. **First-mover advantage**: Launched October 2021 when Solana's DEX landscape was fragmented
2. **Execution speed**: Metis/Iris compute routes within Solana's 400ms block time
3. **Product breadth**: Perps, DCA, limit orders — one-stop shop
4. **Community flywheel**: Massive airdrops → millions of users → volume → revenue → more airdrops
5. **Mobile + Terminal**: Distribution channels competitors lack
6. **Network effects**: 95% share means integrators build on Jupiter API first, reinforcing dominance

---

## 10. 2025-2026 Developments

### 10.1 Catstanbul Conference (January 25-26, 2025)

Jupiter's first-ever conference, held in Istanbul. Major announcements:

- **3B JUP token burn**: Symbolic fire ritual destroying a metal cat sculpture. Reduced total supply from 10B to 7B.
- **50% fee buyback program**: Protocol revenue → JUP purchases → Litterbox Trust
- **Moonshot acquisition**: Majority stake in the meme coin launchpad (surged during TRUMP token launch)
- **SonarWatch acquisition**: Portfolio tracker absorbed into Jupiter ecosystem
- **Ultra Mode**: Real-time slippage estimation, dynamic priority fees, optimized transaction handling
- **Jupiter Shield**: Enhanced security tool for user assets
- **$10M AI fund**: Partnership with Eliza Labs for open-source AI development
- **Jupnet announcement**: Omnichain network in early testnet

JUP price surged 40% ($0.90 → $1.27) on these announcements.

### 10.2 Ultra V3 (October 2025)

The most significant technical upgrade:

- **Iris routing engine**: 100x faster pathfinding
- **Juno meta-aggregator**: Self-learning route selection
- **JupiterZ RFQ**: $100M/day zero-slippage market maker execution
- **34x better MEV protection**: Combination of Iris, JupiterZ, and Jito integration
- **Gasless support**: Jupiter can cover gas for users
- **Industry-best slippage control**: Dynamic, per-route optimization

### 10.3 Jupnet (Omnichain Network)

Jupiter's most ambitious project — a cross-chain aggregation layer.

**Three components**:
1. **DOVE Network**: Decentralized Oracle Validator Executors — validators that retrieve, agree on, and execute cross-chain transactions
2. **Omnichain Ledger**: Single distributed ledger hosting state across all connected chains
3. **Aggregated Decentralized Identity (ADI)**: Account-based UX replacing wallet-centric interaction

**Status**: Early testnet as of January 2025, public beta targeted Q4 2025.

**Vision**: All tradable assets (crypto, stocks, commodities) accessible through one interface, powered by crypto rails.

### 10.4 CatLumpurr Conference (January 31 - February 2, 2026)

Jupiter's second conference, Kuala Lumpur. 40+ product launches:

- **Jupiter Lend**: Fastest-growing Solana protocol — $1B TVL in 8 days after public launch
- **Polymarket integration**: First and only Polymarket venue on Solana
- **Jupiter Global**: Fiat payments — Jupiter Card, Global Send (USD SWIFT to 200+ countries), QR Pay (APAC)
- **Jupiter Offerbook**: Permissionless P2P money market, borrow using any on-chain asset including RWAs
- **$35M ParaFi investment**: First outside funding, settled in JupUSD

### 10.5 Key 2025-2026 Stats

| Metric | Value | Period |
|--------|-------|--------|
| Gross revenue | $509M | 2025 |
| Q3 spot volume | $176.8B | Q3 2025 |
| Q2 swaps | 1.4B | Q2 2025 |
| Q2 volume | $80B | Q2 2025 |
| TVL | $3B+ | Oct 2025 |
| JLP TVL | $2.5B | Oct 2025 |
| Jupiter Lend TVL | $1B+ | 8 days after launch |
| Daily volume | $1.2B+ | Late 2025 |
| Daily active wallets | 300K-1M+ | 2025 |
| Q3 active wallets | 8.4M | Q3 2025 |
| Mobile users | 1.1M+ | 2025 |
| Aggregator market share | 95% | Solana |
| Perps market share | 66% | Solana |

---

## 11. Honest Assessment

### 11.1 What Jupiter Genuinely Does Better

1. **Routing quality**: The Iris/Juno/JupiterZ stack is genuinely best-in-class. No other Solana aggregator comes close.
2. **Product breadth**: Spot + perps + DCA + limit orders + lending + mobile + launchpad in one app. This is unmatched.
3. **MEV protection**: Ultra V3's 34x improvement is real and measurable.
4. **Community building**: The catdet culture and massive airdrops created genuine user loyalty.
5. **Developer experience**: Terminal SDK and Ultra API make integration trivially easy.
6. **Bootstrapped discipline**: Building to $509M revenue without VC funding forced product-market fit.

### 11.2 Weaknesses & Risks

1. **Token price collapse**: JUP down 91.7% from ATH despite record metrics. The token doesn't capture value proportional to the protocol's success. Continuous unlocks overwhelm buybacks.
2. **Solana dependency**: Jupiter is 100% Solana. If Solana has outages or loses relevance, Jupiter goes with it. Jupnet is the hedge, but it's early.
3. **LIBRA scandal proximity**: The Meteora/Ben Chow/LIBRA situation damaged trust. Even though Jupiter maintained distance, the shared founder connection is uncomfortable.
4. **Governance shutdown**: Voluntarily pausing governance for 6+ months is concerning. It suggests the team couldn't make decentralized governance work and chose centralized control instead.
5. **JLP concentration**: 80% of JLP in 10 wallets. A whale exit could destabilize the perps platform.
6. **Meme coin dependency**: A significant portion of Jupiter's volume comes from meme coin trading, which is inherently cyclical. When memes die down, Jupiter's volume drops dramatically.
7. **Revenue sustainability**: $509M gross, but only ~$122M net protocol revenue. Of that, 50% goes to buybacks that haven't supported the token price. The unit economics on aggregation (the core product) are thin.

### 11.3 Centralization Concerns

- **95% aggregator market share** — this is a monopoly. If Jupiter decides to charge routing fees or delist tokens, there's no viable alternative.
- **Acquisition strategy** (Moonshot, SonarWatch) criticized as "monopolistic behavior"
- **Team voting power** in governance large enough to influence outcomes → triggered governance pause
- **Over-reliance on one project** for Solana DeFi: if Jupiter fails, Solana DeFi is severely impacted
- **JupiterZ RFQ** concentrates execution with professional market makers, moving away from permissionless AMM ethos

### 11.4 Sustainability of the Model

The core question: **Can Jupiter sustain $500M+ annual revenue?**

- **Bull case**: DeFi volume on Solana keeps growing. Perps are high-margin. Jupnet unlocks cross-chain volume. Jupiter Lend adds another revenue line. Global payments bring fiat users.
- **Bear case**: Meme coin seasons end. Perps revenue is cyclical (high leverage = high volume in volatile markets, low in calm). 95% market share means growth can only come from total market expansion, not share capture. Token unlocks continue pressuring price.

---

## 12. Stealable Patterns

### 12.1 Product Patterns

| Pattern | How Jupiter Uses It | Reusable In |
|---------|-------------------|-------------|
| **Free core → premium features** | Free aggregator → paid perps/DCA/limits | Any marketplace/platform |
| **Aggregator-as-distribution** | Own the routing layer → own the user relationship | API businesses, marketplaces |
| **Vertical integration** | Aggregator + DEX + perps + lending + mobile + launchpad | Fintech super-apps |
| **Embeddable widget** (Terminal) | Other dApps embed Jupiter swap → Jupiter gets volume | Any B2B2C product |
| **Mobile + web parity** | Same features on mobile with Apple Pay onboarding | Consumer fintech |

### 12.2 Community Patterns

| Pattern | How Jupiter Uses It | Reusable In |
|---------|-------------------|-------------|
| **Massive airdrops** | 2M+ wallets, $616M in value → instant community | Token-based projects |
| **Named community** (Catdets) | Identity + belonging → retention | Any community product |
| **Community-curated launchpad** | DAO votes on what launches → community feels ownership | Marketplaces, app stores |
| **Annual conference** (Catstanbul/CatLumpurr) | IRL events build trust + generate media buzz | Any project with community |
| **Governance-as-marketing** | ASR rewards voting → users feel invested | DAOs, co-ops |

### 12.3 Technical Patterns

| Pattern | How Jupiter Uses It | Reusable In |
|---------|-------------------|-------------|
| **Meta-aggregation** (Juno) | Aggregate aggregators — combine AMM routing + RFQ + 3rd party | Any routing/search system |
| **Self-learning routing** | Auto-sideline bad quote sources | ML-powered recommendation |
| **Oracle stack redundancy** | Primary + 2 backup oracles with automatic failover | Any oracle-dependent system |
| **RFQ + AMM hybrid** | Market makers for top pairs, AMM for long tail | Order flow optimization |
| **Keeper network** | Decentralized execution of conditional orders | Automation platforms |
| **Embeddable SDK** | Zero-config widget for third-party integration | B2B distribution |

### 12.4 Business Patterns

| Pattern | How Jupiter Uses It | Reusable In |
|---------|-------------------|-------------|
| **Bootstrapped → strategic investment** | 4 years with no VC, then $35M from strategic partner at market price | Capital-efficient startups |
| **Buyback-and-burn** | 50% of revenue → buy token → lock or burn | Token value accrual |
| **Conference-driven roadmap** | Catstanbul/CatLumpurr as product launch events | Product marketing |
| **Crisis management** | Swift resignation + external investigation during LIBRA | Any company facing scandal |
| **Free → fee introduction** | Started with zero fees, gradually introduced platform fees | Marketplace monetization |

---

## Latest Updates (2026)

### Buyback Strategy Failure and Zero-Emissions Pivot

Jupiter spent over $70 million on JUP buybacks throughout 2025, using roughly half of all protocol fee revenue. Despite this massive expenditure, JUP's price declined approximately 89% from its peak, falling to the $0.20-$0.22 range by early January 2026. The root cause: monthly token unlocks of ~53 million JUP (scheduled through June 2026) overwhelmed buyback demand, with the buybacks covering only about 6% of unlocked tokens. Solana co-founder Anatoly Yakovenko publicly commented on the structural futility of buybacks against this level of supply expansion. In January 2026, co-founder Siong Ong proposed halting buybacks entirely and redirecting the funds toward user growth incentives. By February 2026, the Jupiter DAO voted on a "Net-Zero Emissions" proposal to cancel Jupuary, pause team vesting indefinitely, and accelerate Mercurial stakeholder vesting with buyback-funded absorption. The vote drew 24,500+ wallets, and despite 13,000+ wallets supporting the airdrop option, 73.9% of total voting weight (81.7% among whales) favored the zero-emissions path.

### JupUSD Stablecoin Launch (Q4 2025)

Jupiter launched JupUSD in partnership with Ethena Labs. JupUSD is fully collateralized by Ethena's USDtb stablecoin, which itself is backed by traditional treasury assets including BlackRock's BUIDL fund. The team plans to add USDe as a secondary backing asset for yield generation. JupUSD is natively integrated across Jupiter's perps, lending, and trading interfaces. Jupiter has stated plans to progressively convert approximately $750 million of USDC from its JLP pool into JupUSD, creating significant native demand. The ParaFi Capital $35M investment was notably settled in JupUSD, validating the stablecoin's institutional credibility.

### Robinhood Integration (November 2025)

Robinhood integrated Jupiter's Swap API into its native crypto wallet app, enabling Robinhood wallet users to trade any SPL token across Solana's DeFi ecosystem. This is a significant distribution milestone — Robinhood's millions of retail users now route trades through Jupiter's Ultra V3 engine, further cementing Jupiter's position as Solana's default swap infrastructure layer.

### Mobile V3 (January 2026)

Jupiter launched Mobile V3 on January 1, 2026, positioning it as the first fully native pro trading terminal on mobile. Key improvements include: swap fees up to 10x lower than competing mobile apps, 34x stronger MEV protection, gasless trading for more token pairs (including memecoin-to-memecoin swaps), built-in portfolio P&L analytics, detailed token analysis (net pressure, liquidity changes, holder distribution), and a redesigned discovery flow. The upgrade moves mobile from "browser-based dApp access" to a true native trading terminal.

### Polymarket Prediction Markets Integration (February 2026)

Jupiter became the first and only venue to bring Polymarket to Solana. A dedicated "Prediction" tab was added directly within the Jupiter app, allowing users to trade event-based prediction markets without bridging stablecoins or leaving the platform. Prediction markets recorded approximately $12 billion in trading volume in January 2026 alone, generating over $11 million in on-chain fees — positioning this as a significant new product vertical alongside swaps and perps.

### Jupiter Lend Rapid Growth

Jupiter Lend exited beta as a fully open-source lending protocol and reached $1 billion in supplied assets within just 8 days of public launch — the fastest growth of any protocol on Solana. This adds a third major revenue vertical (alongside aggregation and perps) and deepens the super-app strategy.

### DAO Governance Resumed with Structural Changes

After a 6+ month governance pause, Jupiter DAO reopened voting in early 2026 with the Net-Zero Emissions proposal as one of its first major votes. The governance resumption featured revised tools, clearer separation of team vs community voting power, and an immediate high-stakes decision that demonstrated the DAO's renewed willingness to make deflationary structural changes to the token model.

## Key Takeaways

1. **Jupiter is the rare DeFi protocol that found product-market fit without VC money.** Four years of bootstrapping forced them to build things people actually use, not things that look good in pitch decks.

2. **The aggregator is the moat, perps are the monetization.** 95% aggregator share gives Jupiter the user relationship. Perps ($509M revenue) is where the money is. This "free core + premium features" model is the most proven pattern in DeFi.

3. **The token is the Achilles' heel.** Despite every business metric trending up, JUP is down 91.7%. Continuous unlocks and airdrops dilute holders faster than buybacks can offset. This is the structural problem Jupiter hasn't solved.

4. **Community building at scale works — until it doesn't.** The catdet culture, massive airdrops, and governance participation drove explosive growth. But governance had to be paused due to trust breakdown, and the LIBRA scandal showed the limits of community trust.

5. **Monopoly position is a double-edged sword.** 95% market share gives Jupiter pricing power and network effects. But it also makes Solana DeFi a single point of failure dependent on one team's execution and integrity.

6. **The evolution from aggregator → super-app → omnichain platform** (Jupnet) is the most ambitious roadmap in DeFi. If Jupnet works, Jupiter becomes a cross-chain financial operating system. If it doesn't, they remain a Solana-only player in an increasingly multi-chain world.

---

## References

### Routing & Architecture

- [Jupiter Routing Engines — Developer Docs](https://dev.jup.ag/docs/routing)
- [Jupiter Metis Routing: Optimizing Swaps on Solana — Jupiter Research Forum](https://discuss.jup.ag/t/jupiter-metis-routing-optimizing-swaps-on-solana/22152)
- [Jupiter Ultra V3: Technical and Strategic Analysis — Ayush Kumar Jha (Medium)](https://medium.com/@ayushkmrjha/jupiter-ultra-v3-a-technical-and-strategic-analysis-of-the-end-to-end-trading-engine-0a61e9f244df)
- [Solana DeFi Deep Dives: Jupiter Ultra V3 — Dhru Patel (Medium)](https://medium.com/@Scoper/solana-defi-deep-dives-jupiter-ultra-v3-next-gen-dex-aggregator-late-2025-2cef75c97301)
- [All about Jupiter's Aggregation Engine: Juno — Jupiter Support](https://support.jup.ag/hc/en-us/articles/18735544617628-All-about-Jupiter-s-Aggregation-Engine-Juno)
- [Juno: Next-Gen Aggregation Engine — Jupiter Legion](https://www.jupiterlegion.net.ng/2025/04/juno-next-gen-aggregation-engine.html)
- [Jupiter Launches Ultra V3 — The Block](https://www.theblock.co/press-releases/375289/jupiter-launches-ultra-v3-the-ultimate-trading-engine-for-solana)
- [Jupiter Ultra V3: MEV Protections and Gasless Support — The Block](https://www.theblock.co/post/375184/solana-decentralized-exchange-aggregator-jupiter-unveils-ultra-v3-improved-trade-execution-mev-protections-gasless-support)
- [Solana DEX Aggregator Comparison 2026 — Carbium Blog](https://blog.carbium.io/solana-dex-aggregator-comparison-2026-carbium-vs-jupiter-vs-titan-vs-okx-vs-dflow/)

### Ultra Swap API & Developer Tools

- [Ultra Swap API Overview — Jupiter Developers](https://dev.jup.ag/docs/ultra)
- [V6 Swap API — Jupiter Developer Docs](https://hub.jup.ag/docs/apis/swap-api)
- [Self-hosted V6 Swap API — Jupiter Developers](https://hub.jup.ag/docs/apis/self-hosted)
- [How to Use Jupiter API to Create a Solana Trading Bot — QuickNode](https://www.quicknode.com/guides/solana-development/3rd-party-integrations/jupiter-api-trading-bot)
- [How to Swap Tokens With Jupiter Ultra API — QuickNode](https://www.quicknode.com/guides/solana-development/3rd-party-integrations/jupiter-ultra-swap)
- [jupiter-swap-api-client — GitHub](https://github.com/jup-ag/jupiter-swap-api-client)
- [Jupiter Python SDK — GitHub](https://github.com/0xTaoDev/jupiter-python-sdk)
- [Integrate Swap Terminal — Jupiter Developers](https://dev.jup.ag/docs/tool-kits/terminal)
- [Jupiter Terminal](https://terminal.jup.ag/)
- [Jupiter API: Solana Swaps, Limit Orders, Routing Data — Bitquery](https://docs.bitquery.io/docs/blockchain/Solana/solana-jupiter-api/)

### Perpetuals & JLP

- [Understanding How Jupiter Perps Works — Jupiter Station](https://station.jup.ag/guides/perpetual-exchange/how-it-works)
- [Understanding Jupiter Perps — Jupiter Station](https://station.jup.ag/guides/perpetual-exchange/overview)
- [Perps Quickstart — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/18734952106908-Perps-Quickstart)
- [What is JLP? — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/18453802442268-What-is-JLP)
- [Fees — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/18735045234588-Fees)
- [Price Oracles — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/19355326396700-Price-Oracles)
- [Jupiter Perps: A Smarter Way to Trade Perpetuals on Solana — Kripto Dnevnik](https://kriptodnevnik.com/jupiter-perps-a-smarter-way-to-trade-perpetuals-on-solana/)
- [A Deep Dive into the Jupiter Perpetual Exchange — Decentral Park Capital](https://decentralparkcapital.substack.com/p/a-deep-dive-into-the-jupiter-perpetual-0d2)
- [Mastering Jupiter Perps — Jupiter Legion](https://www.jupiterlegion.net.ng/2025/10/mastering-jupiter-perps-beginners-guide.html)
- [Jupiter Perp DEX Review — Coin360](https://coin360.com/review/jupiter-perp)
- [Three Major Perp DEX Mechanics: Hyperliquid vs Jupiter vs GMX — PANews](https://www.panewslab.com/en/articles/6jeoramg)
- [JLP: Jupiter's Juicy Yield (PDF) — Contentful](https://assets.ctfassets.net/i0qyt2j9snzb/4ueTPh7PwgopwILuLbhoYL/2fc57577db5f223edfaacf20bd557de1/JLP_-_Jupiter-s_Juicy_Yield.pdf)
- [What Is Jupiter Perps LP (JLP) And How Does It Work? — CoinMarketCap](https://coinmarketcap.com/cmc-ai/jupiter-perps-lp/what-is/)
- [JLP APY Prediction Principle — Bitget News](https://www.bitget.com/news/detail/12560604200685)

### MEV Protection

- [Meow on MEV Protection — X/Twitter](https://x.com/weremeow/status/1831754731045159272)
- [Jupiter's MEV Protection Architecture — Bravian Oyaro (Medium)](https://medium.com/@bnyamosie/jupiters-mev-protection-architecture-how-solana-traders-get-safer-onchain-execution-34a5bccb33be)
- [Solana MEV Wars: Understanding the Game — Astralane (Medium)](https://medium.com/@astralaneio/solana-mev-wars-understanding-the-game-protecting-your-alpha-43ba5e4846e9)
- [What is MEV and How to Protect on Solana — QuickNode](https://www.quicknode.com/guides/solana-development/defi/mev-on-solana)
- [Solana Users Paying Millions to Stop Bot Attacks — DL News](https://www.dlnews.com/articles/defi/solana-users-use-jito-to-stop-sandwich-attacks-and-mev/)

### JUP Token & Airdrops

- [Jupiter Airdrop: JUP Token Guide (2026) — Phantom](https://phantom.com/learn/crypto-101/jupiter-jup-airdrop)
- [Jupiter's $616M Solana Airdrop: 2025 JUP Token Guide — KuCoin](https://www.kucoin.com/news/articles/jupiter-s-616m-solana-airdrop-the-2025-jup-token-guide)
- [Everything You Need to Know About Jupuary — CoinGecko](https://www.coingecko.com/learn/everything-you-need-to-know-jupiter-s-upcoming-airdrop-jupuary)
- [Jupuary 2025: Upcoming Airdrop — Bitrue FAQ](https://support.bitrue.com/hc/en-001/articles/41564995596697-Jupiter-Jupuary-2025-Everything-You-Need-to-Know-About-the-Upcoming-Airdrop)
- [Jupiter Announces $575M Airdrop — Unchained](https://unchainedcrypto.com/jupiter-announces-details-of-its-upcoming-airdrop-of-575-million-worth-of-jup-tokens/)
- [Jupiter (JUP) Tokenomics, Supply & Release Schedule — Tokenomist](https://tokenomist.ai/jupiter-exchange-solana)
- [Jupiter Airdrop — Airdrops.io](https://airdrops.io/jupiter/)
- [Jupiter Price (JUP) — CoinMarketCap](https://coinmarketcap.com/currencies/jupiter-ag/)
- [Jupiter Price (JUP) — CoinGecko](https://www.coingecko.com/en/coins/jupiter)

### Buyback & Token Burns

- [Jupiter Spikes 40% After 50% Fee Buyback Announcement — The Block](https://www.theblock.co/post/337071/jupiter-spikes-40-as-founder-says-50-of-fees-will-go-to-token-buybacks)
- [Jupiter to Buyback JUP with 50% of Fees — CryptoSlate](https://cryptoslate.com/jupiter-to-buyback-jup-tokens-with-50-of-fees-starting-next-week/)
- [Discussion: Burning the JUP Tokens in Litterbox Trust — Jupiter Research Forum](https://discuss.jup.ag/t/discussion-burning-the-jup-tokens-in-the-litterbox-trust/39569)
- [Jupiter Community Votes to Burn 130M JUP — BitcoinEthereumNews](https://bitcoinethereumnews.com/finance/jupiter-community-votes-to-burn-130-million-jup-a-fresh-start-for-the-ecosystem/)
- [The Silent Loop: Buybacks, Unlocks, Litterbox — Jupiter Research Forum](https://discuss.jup.ag/t/the-silent-loop-buybacks-unlocks-litter-box-and-the-search-for-purpose-in-jup/37859)
- [Jupiter's Paradox: Profitable Platform, Underperforming Token — Bokiko (Medium)](https://medium.com/@bokiko/jupiters-paradox-a-profitable-platform-an-underperforming-token-30a8d34c7410)
- [Why Jupiter's JUP Buyback Struggled Despite $70M Spent — Crypto.news](https://crypto.news/jupiter-jup-token-buyback-unlocks-solana-2026/)
- [Anatoly Yakovenko Addresses Jupiter's $70 Million Buybacks — BeInCrypto](https://beincrypto.com/jupiter-buyback-solana-token-unlocks/)
- [Jupiter Founder Questions $70 Million Buyback Strategy — Yellow](https://yellow.com/news/jupiter-founder-questions-dollar70-million-buyback-strategy-after-89-price-decline)
- [Jupiter Exchange Halts JUP Buybacks, Pivots to Growth — Bitget](https://www.bitget.com/news/detail/12560605130545)

### DAO & Governance

- [Jupiter DAO Voting](https://vote.jup.ag/)
- [ASR — Vote | Jupiter](https://vote.jup.ag/asr)
- [What is the Jupiter DAO? — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/20007985083420-What-is-the-Jupiter-DAO)
- [Understanding Jupiter DAO Proposals — Beyond The Swap (Medium)](https://beyondtheswap.medium.com/understanding-jupiter-dao-proposals-ef3eacc5572b)
- [Jupiter Halts DAO Voting Until 2026 — AInvest](https://www.ainvest.com/news/jupiter-jup-halts-dao-voting-2026-governance-reform-2506/)
- [Jupiter DAO Suspends Governance Votes — The Block](https://www.theblock.co/post/358988/dao-behind-dex-aggregator-jupiter-suspends-governance-votes-until-early-2026-amid-community-concerns)
- [Jupiter DAO Pauses Voting After Backlash — The Defiant](https://thedefiant.io/news/defi/jupiter-dao-pauses-voting-after-backlash-over-team-s-voting-power)
- [Solana DEX Jupiter Pauses DAO Votes — CoinDesk](https://www.coindesk.com/business/2025/06/20/solana-dex-jupiter-pauses-dao-votes-citing-breakdown-in-trust)
- [Jupiter DAO Suspends Governance: Staking Rewards Continue — Solana Floor](https://solanafloor.com/news/jupiter-dao-suspends-governance-staking-rewards-continue)
- [ASR (Active Staking Rewards) Notes — Jupiter Research Forum](https://discuss.jup.ag/t/asr-active-staking-rewards-notes/12032/157)
- [Jupiter (JUP) Staking Rewards Calculator — StakingRewards](https://www.stakingrewards.com/asset/jupiter-exchange-solana)
- [Proposal: Net-Zero Emissions — Jupiter Research Forum](https://discuss.jup.ag/t/proposal-net-zero-emissions/39948)
- [Over 81% of Whale Voting Weight Backs Zero Emissions — StepData](https://stepdata.substack.com/p/over-81-of-whale-voting-weight-backs)
- [Jupiter DAO Opens Vote on Canceling Jupuary Airdrops — Cryptopolitan](https://www.cryptopolitan.com/jupiter-dao-vote-canceling-jupuary-airdrops/)
- [JUP Moves Toward Net-Zero Emissions — BitcoinEthereumNews](https://bitcoinethereumnews.com/tech/jup-moves-toward-net-zero-emissions-in-dao-proposal/)

### LFG Launchpad

- [LFG Launchpad — Jupiter](https://lfg.jup.ag/)
- [$WEN — LFG Jupiter](https://lfg.jup.ag/wen)
- [$ZEUS — LFG Jupiter](https://lfg.jup.ag/zeus)
- [Jupiter LFG Launchpad (Archived) — Jupiter Research Forum](https://discuss.jup.ag/t/archived-jupiter-lfg-launchpad/21723)
- [DLMM: The Building Block of Jupiter's LFG Launchpad — Florent Koenig (Medium)](https://medium.com/@florentkoenig/dlmm-the-building-block-of-jupiters-lfg-launchpad-48633fa4cb47)
- [Jupiter LFG: A Game-Changer for Solana Projects in 2025 — Gate.io](https://www.gate.com/learn/articles/jupiter-lfg-launchpad-a-game-changer-for-solana-projects/2047)
- [Jupiter LFG — Top Launchpad Projects by ROI — Chain Broker](https://chainbroker.io/platforms/jupiter-lfg/)
- [Jupiter LFG — CryptoRank](https://cryptorank.io/fundraising-platforms/jupiter-lfg)
- [Jupiter LFG — CoinLaunch](https://coinlaunch.space/launchpads/jupiter-lfg/)

### Team & LIBRA Scandal

- [Ben Chow — Jupiter Exchange Co-Founder — Phantom Podcast](https://podcast.phantom.app/episodes/ben-chow-jupiter-exchange-co-founder-ep-2)
- [Lightspeed Podcast: Jupiter — Aggregator Fueling Solana's GDP (Meow) — Blockworks](https://blockworks.co/podcast/lightspeed/dfae1328-9498-11ee-9434-d34510648730)
- [LIBRA, Solana Drama: Meteora Co-Founder Resigns — Cointelegraph](https://cointelegraph.com/news/libra-solana-meltdown-meteora-jupiter)
- [Meteora CEO Ben Chow Resigns Amid LIBRA Allegations — Blockhead](https://www.blockhead.co/2025/02/19/meteora-ceo-ben-chow-resigns-amid-libra-insider-trading-allegations-2/)
- [How Jupiter and Meteora Fueled LIBRA's Collapse — Crypto News](https://crypto.news/libra-solana-jupiter-meteora-scandal/)
- [Jupiter and Meteora Deny Insider Trading — BitcoinEthereumNews](https://bitcoinethereumnews.com/finance/jupiter-and-meteora-deny-any-insider-trading-or-financial-misconduct/)
- [Meteora Faces Insider Trading Allegations — The Defiant](https://thedefiant.io/news/defi/meteora-faces-insider-trading-allegations-as-ceo-ben-chow-resigns)
- [Jupiter Project — RootData](https://www.rootdata.com/Projects/detail/Jupiter?k=MTAwMDA%3D)
- [Jupiter — Messari](https://messari.io/project/jupiter-exchange)

### Jupiter Ape & Meme Coin Trading

- [Jupiter Unveils Ape — Solana Floor](https://solanafloor.com/news/jupiter-ape-safer-smoother-cheaper-way-trade-meme-coins)
- [Pump.fun — Wikipedia](https://en.wikipedia.org/wiki/Pump.fun)
- [98% of Tokens on Pump.fun Have Been Rug Pulls — CoinDesk](https://www.coindesk.com/business/2025/05/07/98-of-tokens-on-pump-fun-have-been-rug-pulls-or-an-act-of-fraud-new-report-says)

### Market Position & Competition

- [Raydium, Jupiter, Orca, Meteora: Who Will Dominate Solana DEX? — PANews](https://www.panewslab.com/en/articles/5zrwdqh9)
- [In-depth Analysis of Four Major Solana DEXs — ChainCatcher](https://www.chaincatcher.com/en/article/2168403)
- [Best DEX for Solana 2025: Orca, Raydium, or Jupiter? — SnapX](https://blog.snapx.co/best-dex-for-solana-in-2025-orca-raydium-or-jupiter)
- [Jupiter vs Rivals: DEX Aggregators Compared — BestDapps](https://bestdapps.com/blogs/news/jupiter-vs-rivals-dex-aggregators-compared)
- [Best DEX Aggregator Platforms 2026 — CoinsClone](https://www.coinsclone.com/best-dex-aggregator/)

### 2025-2026 Developments

- [Jupiter Announces Major Acquisitions at Catstanbul — Blockhead](https://www.blockhead.co/2025/01/27/jupiter-announces-major-acquisitions-token-buyback-program-platform-overhaul-at-catstanbul-conference/)
- [Jupiter Token Price Rises 40% After Token Burn — CoinMarketCap](https://coinmarketcap.com/academy/article/jupiters-token-price-rises-40percent-after-announcing-3-billion-token-burn-and-50percent-fee-buyback-plan)
- [Jupiter's 30% Supply Cut — Unchained](https://unchainedcrypto.com/jupiters-30-supply-cut-of-jup-worth-3-billion-to-be-celebrated-with-a-physical-fire-ritual/)
- [Jupiter Takes Majority Stake in Moonshot — CryptoBriefing](https://cryptobriefing.com/jupiter-acquires-moonshot-sonarwatch/)
- [Jupiter Acquires Majority Stake in Moonshot — Cointelegraph](https://cointelegraph.com/news/solana-dex-jupiter-acquires-majority-stake-in-moonshot)
- [Jupiter Acquires Moonshot After $400M Trading Surge — Bitget](https://www.bitget.com/news/detail/12560604528488)

### Jupnet & Omnichain

- [Jupiter Unveils Omnichain Network Jupnet — Blocmates](https://www.blocmates.com/news-posts/jupiter-unveils-omnichain-network-jupnet)
- [Jupnet — Meow.bio](https://meow.bio/jupnet)
- [Jupnet Is a Proposed Network — Jupiter Legion](https://www.jupiterlegion.net.ng/2025/04/did-you-know-jupnet-is-proposed-network.html)
- [Jupiter Launches Omnichain Network Jupnet — Bitget](https://www.bitget.com/news/detail/12560604526190)
- [Jupiter: Will Launch Full-Chain Network Jupnet — ChainCatcher](https://www.chaincatcher.com/en/article/2164722)

### CatLumpurr 2026 & ParaFi Investment

- [Jupiter Brings Polymarket to Solana, Lands $35M — CoinDesk](https://www.coindesk.com/markets/2026/02/02/jupiter-brings-polymarket-to-solana-and-lands-usd35-million-investment-deal)
- [Jupiter Secures $35M ParaFi Capital Deal — Blockhead](https://www.blockhead.co/2026/02/04/jupiter-secures-first-ever-outside-investment-with-35m-parafi-capital-deal/)
- [Jupiter Secures $35M ParaFi Investment — Crypto Reporter](https://www.crypto-reporter.com/press-releases/jupiter-secures-35m-strategic-investment-from-parafi-capital-to-accelerate-onchain-financial-infrastructure-121740/)
- [ParaFi Invests $35M in Solana DeFi Platform Jupiter — FintechFutures](https://www.fintechfutures.com/venture-capital-funding/parafi-invests-35m-in-solana-defi-platform-jupiter)
- [Polymarket Expands to Solana Through Jupiter — The Block](https://www.theblock.co/post/387945/jupiter-polymarket-solana)
- [Jupiter Integrates Polymarket — Invezz](https://invezz.com/news/2026/02/02/jupiter-integrates-polymarket-bringing-on-chain-prediction-markets-to-solana/)

### JupUSD Stablecoin

- [Solana's Jupiter to Develop JupUSD Stablecoin With Ethena Labs — CoinDesk](https://www.coindesk.com/web3/2025/10/08/solana-s-jupiter-to-develop-jupusd-stablecoin-with-backing-from-ethena-labs)
- [Jupiter Launches JupUSD Stablecoin With Ethena Labs on Solana — Blockworks](https://blockworks.co/news/jupiter-jupusd-stablecoin)
- [Jupiter Launches JupUSD Stablecoin on Solana With Ethena Labs — CoinMarketCap](https://coinmarketcap.com/academy/article/jupiter-launches-jupusd-stablecoin-on-solana-with-ethena-labs)
- [Jupiter Teams Up With Ethena to Launch JupUSD Stablecoin — Unchained](https://unchainedcrypto.com/jupiter-teams-up-with-ethena-to-launch-jupusd-stablecoin/)

### Mobile V3

- [Jupiter Rolls Out Mobile V3 With Native Pro Trading Tools — Crypto.news](https://crypto.news/jupiter-launches-mobile-v3-native-pro-trading-2026/)
- [Jupiter Introduces V3 Mobile Trading Terminal — CryptoBriefing](https://cryptobriefing.com/jupiter-mobile-v3-enhanced-trading/)
- [Jupiter Mobile V3 Launch: First Native Pro-Trading App on Solana — EGW News](https://egw.news/crypto/news/31581/jupiter-announces-mobile-v3-the-first-full-fledged-henGAxm74)

### Robinhood Integration

- [Can Robinhood's Jupiter Integration Reassert the Aggregator's Dominance? — Solana Floor](https://solanafloor.com/news/robinhoods-jupiter-integration-reassert-aggregator-dominance)
- [Jupiter Unveils JupUSD Stablecoin and Major DeFi Upgrades at Solana Breakpoint 2025 — Coinpedia](https://coinpedia.org/news/jupiter-unveils-jupusd-stablecoin-and-major-defi-upgrades-at-solana-breakpoint-2025/)

### Revenue, TVL & Stats

- [Jupiter — DefiLlama](https://defillama.com/protocol/jupiter)
- [Jupiter Aggregator — DefiLlama](https://defillama.com/protocol/jupiter-aggregator)
- [Jupiter Revenues Surge on Solana — MEXC News](https://www.mexc.com/news/139989)
- [Number One DEX on Solana: Data on Volume, TVL, Users — Hive](https://hive.blog/jupiter/@dalz/a-look-at-the-number-one-dex-on-solana-jupiter-or-data-on-trading-volume-tvl-users-top-pairs-and-more-or-apr-2025)
- [Jupiter's Ecosystem Dominance in Solana DeFi — AInvest](https://www.ainvest.com/news/jupiter-ecosystem-dominance-solana-defi-flywheel-driven-play-2026-2512/)

### Centralization & Criticism

- [Jupiter's Moves in Solana: Mixed Bag — OneSafe](https://www.onesafe.io/blog/jupiter-strategic-moves-solana-decentralization)
- [Jupiter's Acquisition Spree Sparks Dominance Concerns — CoinDesk](https://www.coindesk.com/business/2025/01/27/jupiter-s-acquisition-spree-buyback-plan-spark-solana-ecosystem-dominance-concerns)
- [Jupiter Pauses DAO Voting Amid Trust Breakdown — DL News](https://www.dlnews.com/articles/defi/solana-exchange-jupiter-pauses-dao-voting-until-2026/)

### General Overview

- [What Is Jupiter (JUP)? Solana's Largest DeFi Superapp — CoinGecko](https://www.coingecko.com/learn/what-is-jupiter-crypto-solana)
- [What is Jupiter (JUP)? — Bitcoin.com](https://www.bitcoin.com/get-started/what-is-jupiter-solana/)
- [Jupiter's Evolution: Solana DEX Reshaping DeFi — OKX](https://www.okx.com/en-us/learn/jupiter-solana-dex-defi-evolution)
- [Jupiter's Evolution: From Aggregator to Full-Stack DeFi — OKX](https://www.okx.com/en-us/learn/jupiter-solana-dex-defi-platform)
- [What Is Jupiter Exchange? Guide — Nansen](https://www.nansen.ai/post/what-is-jupiter-exchange)
- [Jupiter DEX Review — Coin360](https://coin360.com/review/jupiter)
- [Jupiter (JUP): Solana Liquidity Engine — Laika Labs](https://laikalabs.ai/market-intelligence/jupiter-jup-solana-liquidity-engine-guide)
- [Jupiter on Solana: Project Reviews — Solana Compass](https://solanacompass.com/projects/jupiter)
- [Jupiter — IQ.wiki](https://iq.wiki/wiki/jupiter)

### DCA & Limit Orders

- [How Dollar Cost Averaging Works — Jupiter Station](https://betastation.jup.ag/guides/dca/how-dca-work)
- [How Limit Orders Work — Jupiter Station](https://station.jup.ag/guides/limit-order/how-lo-work)
- [How to Use DCA on Jupiter — Jupiter Station](https://station.jup.ag/guides/dca/how-to-dca)

### Jupiter Lock

- [Lock | Jupiter](https://lock.jup.ag/)
- [jup-lock — GitHub](https://github.com/jup-ag/jup-lock)

### Jupiter Mobile

- [Jupiter Mobile — App Store](https://apps.apple.com/us/app/jupiter-mobile-solana-wallet/id6484069059)
- [Jupiter Mobile — Google Play](https://play.google.com/store/apps/details?id=ag.jup.jupiter.android)
- [Jupiter Mobile — jup.ag](https://jup.ag/mobile)
- [Solana's Top Swap Venue Jupiter Released a Mobile App — Blockworks](https://blockworks.co/news/solana-venue-jupiter-released-mobile-app)

### Community

- [Jupiter's Success: Community-First Approach — VALR Blog](https://blog.valr.com/blog/jupiters-success-a-community-first-approach-to-growth-ft-meow-founder-of-jupiter-on-solana)
- [Catalyzing Sustainable Growth — Jupiter Research Forum](https://discuss.jup.ag/t/catalyzing-sustainable-growth-a-strategic-initiative-for-jupiter-community-adoption-liquidity-depth/39497)
- [Catstanbul 2025 — Luma](https://luma.com/catstanbul)
- [Catstanbul 2025 — jup.ag](https://catstanbul.jup.ag/)
