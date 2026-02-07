# Parallel Coding Agents: Architecture Guide

## What

Running N coding agents simultaneously in the cloud — each with its own git repo, filesystem, package manager, and shell — to parallelize software development. The 2025→2026 shift: from local worktrees on your MacBook to cloud sandboxes managed by an orchestrator.

```
2025: MacBook runs 4 git worktrees, each with an agent
2026: MacBook dispatches tasks, cloud runs N sandboxes in parallel
```

## Why Cloud, Why Now

Warp CEO Zach Lloyd's thesis (Sequoia podcast, Jan 2026):

> "Coding will be solved within years. The bottleneck shifts to human capacity to express intent clearly."

Five forces pushing agents off the laptop:

| Force | Problem | Cloud Solves It |
|-------|---------|-----------------|
| **Compute ceiling** | 2 cargo builds saturate a MacBook | Cloud scales horizontally |
| **Sandbox isolation** | Agents testing UI need exclusive screen access | Each gets its own container |
| **Always-on** | Agents need to run while laptop sleeps | Cloud doesn't sleep |
| **Team visibility** | No way to track agent activity across a team | Dashboards, audit logs |
| **Setup cost** | Docker + provisioning was painful | Agents can now configure their own environments |

The progression is intentional — don't skip steps:

| Phase | Where | Human Role | Trust Level |
|-------|-------|------------|-------------|
| Interactive | Local | Review every diff | Low |
| Parallel local | Local (worktrees) | Manage 4-5 agents | Medium |
| Cloud ambient | Cloud sandbox | Review PRs, approve/reject | High |
| Autonomous | Cloud, event-triggered | Set policies, handle escalations | Very high |

## Architecture: The Universal Pattern

Every platform — Cloudflare, Warp, E2B, Daytona — converges on the same layered architecture:

```
┌─────────────────────────────────────────────┐
│  TRIGGER                                     │
│  Slack / Linear / GitHub / Webhook / Cron    │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  ORCHESTRATE                                 │
│  Task queue → scheduling → state → secrets   │
│  (Cloudflare Agents SDK / Warp / custom)     │
└──────────────────┬──────────────────────────┘
                   │ spawns N
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Sandbox │ │ Sandbox │ │ Sandbox │ │ Sandbox │
│ Agent 1 │ │ Agent 2 │ │ Agent 3 │ │ Agent 4 │
│ git+npm │ │ git+rsc │ │ git+py  │ │ git+go  │
└─────────┘ └─────────┘ └─────────┘ └─────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  OBSERVE                                     │
│  Session transcripts → dashboard → audit log │
└─────────────────────────────────────────────┘
```

**Rule: never run agent logic in the orchestrator.** The orchestrator routes, authenticates, and coordinates. Sandboxes execute. This separation is non-negotiable.

## Platform Landscape

### Tier 1: Agent-Native Sandboxes

These were built specifically for AI coding agents.

| Platform | Cold Start | Isolation | Key Feature | Weakness |
|----------|-----------|-----------|-------------|----------|
| **Daytona** | 27-90ms | Docker (Kata optional) | Fork/snapshot | Docker isolation weaker |
| **E2B** | ~150ms | Firecracker microVM | Best SDK (Python/TS) | 24h session cap |
| **Fly.io Sprites** | 1-12s | Firecracker microVM | 100GB NVMe persistence | Slowest cold start |
| **Blaxel** | 25ms | Lightweight | Fastest measured start | Newest, least proven |

### Tier 2: General Compute with Sandbox Support

| Platform | Cold Start | Key Feature | Weakness |
|----------|-----------|-------------|----------|
| **Cloudflare Containers** | 2-3 min | Global edge + integrated orchestration | Resource ceiling (4 vCPU, 20GB disk) |
| **Modal** | Sub-second | GPU (H100), 50k concurrent | Python only, gVisor overhead |
| **Northflank** | Varies | BYOC, cheapest ($0.017/hr) | Not agent-first |

### Tier 3: Full Platforms (Orchestration + Execution)

| Platform | Approach | Best For |
|----------|----------|----------|
| **Warp** | Terminal-native ADE + Namespace sandboxes | Teams wanting turnkey agent orchestration |
| **GitHub Codespaces** | Dev environment + Copilot agents | GitHub-native workflows |
| **Cloudflare** (full stack) | Workers + Agents SDK + Durable Objects + Containers | Custom orchestration + global edge |

## Comparison Matrix

| Feature | E2B | Daytona | Modal | Sprites | Cloudflare | Warp |
|---------|-----|---------|-------|---------|------------|------|
| Cold Start | 150ms | 27-90ms | <1s | 1-12s | 2-3min | N/A |
| Max Session | 24h | Unlimited | 24h | Unlimited | Unlimited | N/A |
| GPU | No | No | H100 | Limited | No | No |
| Fork/Snapshot | No | Yes | No | Checkpoint | No | No |
| Orchestration | No | No | No | No | Yes (Agents SDK) | Yes |
| Open Source | Yes | Yes | No | No | Partial | No |
| Cost (1vCPU/2GB/hr) | $0.08 | $0.08 | $0.12 | $0.07 | Pay-per-10ms | $20/mo + credits |

## Isolation Technologies

Pick your tradeoff:

| Technology | Isolation | Overhead | Cold Start | Used By |
|------------|----------|----------|------------|---------|
| **Firecracker** (microVM) | Hardware-level | ~5% | 100-200ms | E2B, Sprites, AWS Lambda |
| **Kata** (container-VM hybrid) | Hardware-level | 5-10% | 200-500ms | Northflank |
| **gVisor** (user-space kernel) | Kernel-level | 2-9x syscall | Sub-second | Modal, Google |
| **Docker** (namespace/cgroup) | Process-level | Minimal | 27-90ms | Daytona |

**Practical rule**: if agents only run YOUR LLM-generated code (not untrusted user code), Docker is good enough. The 2x speed gain over Firecracker is worth the isolation tradeoff.

## Five Architecture Patterns

### Pattern 1: Ephemeral Sandbox Per Task

```
Orchestrator → spawn N sandboxes → each runs one task → collect results → destroy
```

**When**: Independent, well-defined tasks (fix bug A, add feature B, write tests for C).
**Best platforms**: E2B (fast start, clean SDK), Modal (massive scale).

### Pattern 2: Fork-and-Explore

```
Agent runs → decision point → fork into N branches → evaluate → keep winner
```

**When**: Uncertain approach — let agent explore multiple solutions in parallel. Same pattern as MCTS in game AI.
**Best platform**: Daytona (native fork/snapshot primitives).

### Pattern 3: Persistent Agent Workstation

```
Agent gets a "computer" → installs tools once → works for days → checkpoint/restore
```

**When**: Long-running, complex tasks that build on previous work. Agent needs accumulated state.
**Best platform**: Sprites.dev (NVMe persistence, 300ms checkpoint).

### Pattern 4: Orchestrator-as-Brain (Cloudflare Stack)

```
Worker (API gateway) → Agents SDK on Durable Objects (state + WebSocket) → N Sandbox containers
```

**When**: You want custom orchestration logic, global edge deployment, and scale-to-zero pricing.
**Best platform**: Cloudflare (Workers + Agents SDK + Containers + Sandbox SDK).

Cloudflare has an official tutorial for running Claude Code in a Sandbox:
1. Worker receives request (repo URL + task)
2. Creates Sandbox, clones repo via `gitCheckout()`
3. Claude Code executes inside sandbox
4. Returns execution logs + git diff

### Pattern 5: Event-Triggered Ambient Agents (Warp Stack)

```
Event fires (Slack/Linear/GitHub/cron) → Warp orchestrator → Namespace sandbox → PR/message
```

**When**: You want agents that react to system events without human initiation.
**Best platform**: Warp Ambient Agents (beta).

Warp's key technical decisions:
- **Sidecar volumes**: mount `/agent/` volume (Warp CLI + git + CA certs) onto any user Docker image — zero config
- **Shared cache**: all team sandboxes share codebase embeddings and context indices — agents don't rebuild context from scratch
- **Scoped credentials**: inject short-lived, user-scoped GitHub tokens at sandbox creation — never embed in images

## Tradeoffs That Matter

### Cold Start vs Isolation

The fundamental tension. You can't have sub-100ms starts AND hardware-level isolation.

```
Faster ←──────────────────────────────────→ Safer
Docker (27ms)  gVisor (<1s)  Firecracker (150ms)  Kata (200-500ms)
Daytona        Modal         E2B/Sprites           Northflank
```

### Ephemeral vs Persistent

24-hour session caps (E2B, Modal) force ephemeral architecture — agents start clean every time. Unlimited sessions (Sprites, Daytona) enable persistent workstations. Your agent architecture must choose one.

### Integrated vs Composable

| Approach | Pros | Cons |
|----------|------|------|
| **Integrated** (Warp, Cloudflare full stack) | Turnkey, less glue code | Vendor lock-in |
| **Composable** (E2B + custom orchestrator) | Swap any layer | More integration work |

The Rivet Sandbox Agent SDK is the convergence signal — one API that runs Claude Code, Codex, or OpenCode inside any sandbox (Daytona, E2B, Docker). The sandbox layer is commoditizing. Value moves up to orchestration.

### Resource Ceiling

| If Your Agent Needs... | Avoid | Use |
|------------------------|-------|-----|
| >20GB disk (big monorepo) | Cloudflare | Codespaces (64GB), Sprites (100GB) |
| GPU inference | E2B, Daytona, Cloudflare | Modal (H100) |
| >4 vCPU per agent | Cloudflare | Codespaces (32 vCPU) |
| Multi-day persistence | E2B, Modal (24h cap) | Sprites, Daytona |

## Decision Framework

```
Do you need custom orchestration?
├── Yes → Do you want global edge?
│         ├── Yes → Cloudflare (Workers + Agents SDK + Containers)
│         └── No  → Build on E2B/Daytona + your own orchestrator
└── No  → Do you want turnkey?
          ├── Yes → Warp Ambient Agents (beta) or GitHub Codespaces
          └── No  → What's your priority?
                    ├── Fastest cold start    → Daytona (27ms)
                    ├── Best SDK experience   → E2B (one-liner API)
                    ├── GPU compute           → Modal (H100)
                    ├── Persistent workspace  → Sprites (100GB NVMe)
                    ├── Cheapest at scale     → Northflank ($0.017/hr)
                    └── Enterprise self-host  → Coder / Northflank BYOC
```

## Steal: Patterns to Reuse

### 1. Orchestrator ≠ Executor (from everyone)
Workers route. Containers execute. Never mix these concerns. The orchestrator should be lightweight (V8 isolate, Lambda, or a simple queue consumer). The executor needs full Linux.

### 2. Sidecar Volume (from Warp)
Mount agent tooling as a sidecar volume (`/agent/` with CLI, git, CA certs) onto any user Docker image. Users supply their standard image, you inject your runtime. Decouples agent from environment, enables zero-config onboarding and instant updates.

### 3. Fork-and-Snapshot (from Daytona)
For exploratory tasks, fork the sandbox at a decision point. Run N approaches in parallel. Snapshot the winner, discard the rest. This is MCTS for code — the most powerful pattern for uncertain problems.

### 4. Shared Cache Across Runs (from Warp)
Persist codebase embeddings, context indices, and dependency caches across agent sandbox lifecycles. Agents that cold-start context every run waste minutes. A shared cache volume eliminates this.

### 5. Scoped Credential Injection (from Warp)
Never embed credentials in container images. Inject short-lived, user-scoped tokens at sandbox creation. GitHub tokens should be temporary and limited to the triggering user's repo access.

### 6. Progressive Autonomy (from Warp)
Start local and interactive. Graduate to local-parallel. Then cloud-background. Then event-triggered autonomous. Each step builds trust before increasing autonomy. Don't skip steps — the trust model matters as much as the technology.

### 7. Friction-Point Activation (from Warp)
Instead of always-on agent suggestions, detect friction moments (git conflicts, test failures, build breaks) and offer one-click "let agent fix this." Higher adoption than proactive suggestions.

### 8. TOEO Architecture (from Warp)
Trigger → Orchestrate → Execute → Observe. Four layers, cleanly separated. Swap execution environment (cloud/local/self-hosted) without changing triggers or orchestration logic. Swap triggers (Slack/webhook/cron) without touching execution.

## Cost Analysis

Running 4 parallel agents, 8 hours/day, 20 days/month (each: 1 vCPU, 2GB RAM):

| Platform | Monthly Cost | Notes |
|----------|-------------|-------|
| Northflank | ~$11 | Cheapest, BYOC option |
| Sprites | ~$45 | + storage costs |
| E2B | ~$51 | Or $150/mo Pro flat |
| Daytona | ~$51 | Similar to E2B |
| Cloudflare | ~$60 | Pay-per-10ms active |
| Modal | ~$77 | + GPU if needed |
| Codespaces | ~$115 | Most expensive |
| Warp | ~$20/mo + credits | Hard to estimate, credit-based |

## What's Next

The market is moving fast:
- **Daytona** just raised $24M Series A (Feb 2026) — fork/snapshot is their bet
- **Warp** Ambient Agents in beta — event-triggered cloud agents
- **Cloudflare** Containers in public beta — Sandbox SDK with Claude Code tutorial
- **GitHub** Agents HQ (Feb 2026) — multi-agent execution within GitHub
- **Rivet** Sandbox Agent SDK — universal agent-sandbox abstraction layer

The sandbox layer is commoditizing. The value is moving to **orchestration** (how you coordinate N agents) and **intent specification** (how you tell agents what to build). The terminal becomes a cockpit for managing agent fleets, not a place where you type commands.
