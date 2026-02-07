# VoxYZ: Autonomous AI Company — Deep Analysis

> Source: [@Voxyz_ai](https://x.com/Voxyz_ai/status/2019914775061270747) — "I Built an AI Company with OpenClaw + Vercel + Supabase — Two Weeks Later, They Run It Themselves"

## What They Built

6 AI agents autonomously operating a company — researching, writing, posting, and coordinating — from inside a pixel art virtual office. Built in 2 weeks.

| Agent | Role | Model |
|-------|------|-------|
| Minion | Chief of Staff — coordinates, delegates | Claude Opus |
| Sage | Head of Research — deep analysis, strategy | GPT-5.2 |
| Scout | Head of Growth — finds leads, opportunities | GPT-5.1 Codex |
| Quill | Creative Director — writes copy, designs content | Claude Sonnet |
| Xalt | Social Media Director — posts, audience growth | Gemini 3 Pro |
| Observer | Operations Analyst — documents everything | GPT-5.2 |

## Why This Architecture (And Not Others)

### Why OpenClaw Over CrewAI / LangGraph / AutoGen?

| Framework | Architecture | Best For | Weakness |
|-----------|-------------|----------|----------|
| **LangGraph** | Graph-based DAG | Complex branching workflows | Steep learning curve, overkill for simple loops |
| **CrewAI** | Role-based teams | Quick prototyping, team metaphors | Less control over execution flow |
| **AutoGen** | Conversational agents | Dynamic dialogues, role-switching | Harder to debug, unpredictable routing |
| **OpenClaw** | Gateway-centric pipeline | Local-first, multi-channel, persistent | Younger ecosystem, less community support |

VoxYZ chose OpenClaw because:
- **Local-first execution** — agents run shell commands, read files, browse the web on hardware you control
- **Gateway-centric** — single control plane for sessions, channels, tools, and events. No need to wire up separate orchestration
- **Multi-model routing** — built-in model resolver with automatic failover. When one API key gets rate-limited, it switches to a backup
- **JSONL audit trail** — every tool call, every execution result logged line-by-line. Full traceability without extra infrastructure

The key insight: OpenClaw treats agents as **structured pipelines**, not "thinking entities." This makes debugging and recovery predictable.

### Why Supabase Over Redis / Raw PostgreSQL?

| Option | Latency | Realtime | Persistence | Complexity |
|--------|---------|----------|-------------|------------|
| **Redis** | Sub-ms | Pub/sub (non-persistent) | Needs separate DB | High ops burden |
| **Raw PostgreSQL** | 10-50ms | None built-in | Full ACID | Need to build everything |
| **Supabase** | 10-50ms | Built-in WebSocket | Full ACID (it IS Postgres) | Auth + realtime + storage free |

VoxYZ chose Supabase because:
- Realtime subscriptions out of the box — agents can listen for state changes without polling
- Auth + Row Level Security — agents can be scoped to only see their own data
- It's just PostgreSQL underneath — no vendor lock-in, can migrate anytime
- Dashboard for debugging — crucial during the 2-week sprint

**The tradeoff they accepted**: 10-50ms latency per query instead of Redis's sub-ms. For agents that think for seconds per turn, this is negligible. They'd need Redis only if agents were making 100+ state queries per second — they're not.

### Why Vercel Cron Over Dedicated Workers?

**Vercel cron limitations**:
- Pro plan required ($20/month)
- Max 60-second execution per invocation
- Minimum interval: 1 minute (they use 5 minutes)
- No persistent process — spins up, runs, dies

**Why they chose it anyway**: Simplicity. One deploy, cron just works. The alternative — a VPS running a persistent scheduler — is what they **started with** and it caused their biggest bug (Pitfall #1: two schedulers fighting).

**What they'd need to change at scale**: Move the heartbeat to an external scheduler (AWS EventBridge, Upstash QStash, or a dedicated VPS) while keeping the API on Vercel.

## How It Actually Works (Execution Trace)

### The Closed Loop

```
Agent proposes idea
  → proposal-service validates (quota gate + daily limit + score)
    → mission created in Supabase
      → Minion routes to correct agent
        → agent executes via OpenClaw pipeline
          → result written to Supabase
            → triggers evaluate new state
              → reactions spawn new proposals
                → loop
```

### Why a Single Proposal Service Matters

This is the most important architectural decision. Every action flows through ONE function:

```
ProcessProposalWorkflow:
  1. Create proposal record
  2. Check daily limit (per-agent)
  3. Check quota cap (global)
  4. Evaluate proposal score
  5. Auto-approve if above threshold
  6. Create mission if approved
  7. Return result
```

**Why not let agents create missions directly?** Because of the [Carnegie Mellon "Agent Company" study](https://www.reworked.co/digital-workplace/the-fake-startup-that-exposed-ais-real-limits-as-autonomous-workers/): AI agents lie, hallucinate, and create runaway loops. Even Claude failed 75% of the time on complex office tasks. A single gate function means rate limiting, scoring, audit trail, and kill switch — all in one place.

### Trigger/Reaction Separation

**Triggers** = pure detection. They read state, evaluate conditions, emit proposal templates. They NEVER write to the database.

**Reactions** = inter-agent responses stored as data, not code:

```json
{ "source": "quill", "target": "xalt", "type": "content_ready", "action": "post" }
```

**Why separate?** Because when triggers also execute, you get cascading failures. Agent A triggers Agent B triggers Agent A → infinite loop. By making triggers read-only and routing everything through the proposal-service gate, runaway cascades become impossible.

## The Three Pitfalls (Deep Dive)

### Pitfall 1: Dual Schedulers → Race Conditions

They ran a VPS cron AND Vercel cron doing the same work. Both would pick up the same mission, execute it twice, write conflicting results.

This is the classic [split-brain problem](https://en.wikipedia.org/wiki/Split-brain_(computing)) — two processes both believe they're the leader.

**Their fix**: Kill the VPS scheduler entirely. One heartbeat, one source of truth.

**Alternatives they could have used**:

| Approach | Complexity | When to Use |
|----------|-----------|-------------|
| Kill one scheduler (what they did) | Low | Small scale, one team |
| Distributed locks (Redis SETNX) | Medium | Multiple workers competing |
| Leader election (Raft/Paxos) | High | Large distributed systems |
| Idempotent operations | Medium | When duplicate execution is acceptable |

The simplest solution won. Good taste.

### Pitfall 2: Orphaned Triggers

Triggers fired but no agent claimed the work. Proposals vanished into the void.

**Root cause**: Trigger and execution were separate systems with no guaranteed delivery.

**Their fix**: Single proposal-service that both creates AND routes in one atomic operation.

This is essentially the [transactional outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html) — ensuring a database write and a message send happen atomically. By putting both in one Supabase transaction, they get atomicity for free.

### Pitfall 3: Unbounded Queue Growth

When API quotas are exhausted, proposals keep arriving. Queue grows forever.

**Three approaches to this problem**:

| Strategy | How It Works | When to Use |
|----------|-------------|-------------|
| **Reject at gate** (VoxYZ's choice) | Check quota before enqueuing | Low volume (~50 proposals/day) |
| **Backpressure** | Signal upstream to slow down | High-throughput pipelines |
| **Dead letter queue** | Route failures to separate queue for inspection | Systems needing retry logic |

VoxYZ chose the simplest. This works at ~50 proposals/day. At thousands/day, they'd need backpressure or DLQs. See [AWS: Avoiding Insurmountable Queue Backlogs](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/).

## Self-Healing: Why It's Non-Negotiable

From [Galileo's research](https://galileo.ai/blog/multi-agent-ai-system-failure-recovery): agent dependencies create **unpredictable cascade effects**. When one agent fails, others that depend on its output get stuck in undefined states.

VoxYZ's heartbeat recovery:

```
Every 5 minutes:
  SELECT missions WHERE status='running' AND updated_at < NOW() - 30min
    → mark as failed
    → re-trigger from last successful step
    → increment retry counter
    → alert if retry > 3
```

**Why 30 minutes?** Slowest agent (Sage, deep research) takes ~15 minutes. 30min = 2x worst case. Below that → false positives.

**What's missing**: No circuit breaker. If an external API is down for hours, the heartbeat retries every 30 minutes forever. A [circuit breaker pattern](https://martinfowler.com/bliki/CircuitBreaker.html) would detect sustained failures and stop retrying until recovery.

## Honest Assessment: What's Weak

| Weakness | Risk | Fix |
|----------|------|-----|
| No circuit breaker | Infinite retry on sustained outages | Implement half-open circuit breaker |
| Vercel 60s timeout | Complex missions can't complete in one invocation | Move heartbeat to external scheduler |
| No human-in-the-loop | [40% of agentic AI projects fail](https://squirro.com/squirro-blog/avoiding-agentic-ai-failure) without escalation | Add approval workflow for high-risk actions |
| Single-region Supabase | SPOF — if Supabase goes down, everything stops | Multi-region or read replicas |
| No cost controls | 6 agents × premium models = $100+/day possible | Per-agent daily spend caps |
| No output validation | [Agents hallucinate and fabricate](https://www.reworked.co/digital-workplace/the-fake-startup-that-exposed-ais-real-limits-as-autonomous-workers/) | Add output scoring/verification step |

## Patterns Worth Stealing

### 1. The Gate Function
Route ALL agent actions through a single validation function. One place for rate limiting, scoring, approval, and kill switches.

### 2. Triggers Read, Services Write
Never let detection logic mutate state. Triggers emit proposals, services execute. Prevents cascading loops.

### 3. Policy-as-Data
Agent behavior rules (quotas, thresholds, limits) live in a database table, not code. Change behavior without redeploying.

### 4. Heartbeat Self-Healing
Assume every agent will get stuck. Periodic sweeper detects stale missions and restarts from last checkpoint.

### 5. Kill the Second Scheduler
When two things do the same job, one is a bug. Always choose one source of truth.

## Sources

- [Four Design Patterns for Event-Driven Multi-Agent Systems — Confluent](https://www.confluent.io/blog/event-driven-multi-agent-systems/)
- [Choosing the Right Orchestration Pattern — Kore.ai](https://www.kore.ai/blog/choosing-the-right-orchestration-pattern-for-multi-agent-systems)
- [State Management for AI Agents: Redis vs External DBs — DEV](https://dev.to/inboryn_99399f96579fcd705/state-management-patterns-for-long-running-ai-agents-redis-vs-statefulsets-vs-external-databases-39c5)
- [Multi-Agent Failure Recovery — Galileo](https://galileo.ai/blog/multi-agent-ai-system-failure-recovery)
- [Why 40% of Agentic AI Projects Fail — Squirro](https://squirro.com/squirro-blog/avoiding-agentic-ai-failure)
- [The Agent Company: AI Agents Failed 75% — Reworked](https://www.reworked.co/digital-workplace/the-fake-startup-that-exposed-ais-real-limits-as-autonomous-workers/)
- [Avoiding Insurmountable Queue Backlogs — AWS](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/)
- [OpenClaw Architecture Guide — Vertu](https://vertu.com/ai-tools/openclaw-clawdbot-architecture-engineering-reliable-and-controllable-ai-agents/)
- [OpenClaw Risks — Cyber Strategy Institute](https://cyberstrategyinstitute.com/openclaw-risks-autonomous-ai-agents/)
- [CrewAI vs LangGraph vs AutoGen — DataCamp](https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen)
