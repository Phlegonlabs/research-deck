# OpenClaw — Open-Source Personal AI Agent (v2026.2.6)

## TL;DR

OpenClaw (formerly Clawdbot/Moltbot) is an open-source, self-hosted AI agent framework by Peter Steinberger (PSPDFKit founder). It bridges LLMs (Claude, GPT, Grok, local models) with your OS — giving AI full access to files, shell, browser, and 50+ integrations (WhatsApp, Slack, Discord, Gmail, etc.). 145K+ GitHub stars. Latest version v2026.2.6 (Feb 7, 2026) adds Opus 4.6, GPT-5.3-Codex, safety scanner, and token dashboard.

---

## What It Is

A local-first AI agent gateway that:
- Runs on your machine (Mac/Windows/Linux)
- Connects to any LLM (Claude, GPT, Gemini, Grok, DeepSeek, local Llama)
- Interfaces via messaging apps (WhatsApp, Telegram, Discord, Slack, Signal, iMessage)
- Has 100+ AgentSkills for autonomous task execution
- Maintains persistent memory across sessions

**Not a chatbot** — a proactive agent that acts on your behalf, 24/7.

---

## Architecture

### Three-Component Design

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Brain     │     │   Memory    │     │    Arms     │
│ (LLM Layer) │────▶│  (Storage)  │────▶│  (Tools)    │
│              │     │             │     │             │
│ System prompt│     │ Short-term: │     │ fs_tool     │
│ + chat hist  │     │  chat logs  │     │ bash_tool   │
│ → LLM API   │     │ Long-term:  │     │ browser_tool│
│              │     │  .md files  │     │ (CDP)       │
└─────────────┘     └─────────────┘     └─────────────┘
```

| Component | Role | Implementation |
|-----------|------|----------------|
| **Brain** | Constructs system prompt + sends to LLM | External LLM API (Claude, GPT, etc.) |
| **Memory** | Local-first persistence | Flat-file Markdown in `~/.openclaw/memory` |
| **Arms** | JSON API tools on host OS | fs_tool, bash_tool, browser_tool (Chromium CDP) |

### Agentic Loop (Think → Plan → Act → Observe → Iterate)

1. **Think** — LLM analyzes request, consults memory
2. **Plan** — Breaks request into tool-call sequences
3. **Act** — Executes tools on the machine
4. **Observe** — Interprets stdout/stderr
5. **Iterate** — Refines approach, loops until task complete

### Gateway Architecture

The Gateway is the single control plane managing:
- Sessions (scoped by channel/peer/team)
- Channels (WhatsApp, Telegram, Slack, Discord, etc.)
- Tools (skill dispatching)
- Events (scheduling, heartbeats, webhooks)

Access: `http://127.0.0.1:18789/` for local Web UI dashboard.

### Workspace File Structure

| File | Purpose |
|------|---------|
| `AGENTS.md` | Operating instructions |
| `SOUL.md` | Persona and behavioral boundaries |
| `MEMORY.md` | Long-term curated facts |
| `memory/YYYY-MM-DD.md` | Daily append-only logs |
| `HEARTBEAT.md` | Automated health checklist |

---

## Skills System (AgentSkills)

### How It Works

Skills = directories containing a `SKILL.md` with YAML frontmatter. Declarative — tells the agent what tools to use, not how to code them.

### Loading Hierarchy (precedence order)

1. **Workspace skills** (`<workspace>/skills`) — highest priority
2. **Managed/local skills** (`~/.openclaw/skills`) — shared across agents
3. **Bundled skills** — shipped with installation

### Skill Structure

```yaml
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
homepage: https://example.com
user-invocable: true
metadata:
  openclaw:
    requires:
      bins: [ffmpeg]
      env: [GEMINI_API_KEY]
    os: [darwin, linux]
---
# Instructions for the agent...
```

### Gating System

Skills auto-filter at load time based on:
- Required binaries on PATH
- Required environment variables
- Required config paths
- OS platform restrictions

### ClawHub Registry

Public marketplace: `clawhub.com`

```bash
clawhub install <skill-slug>
clawhub update --all
clawhub sync --all
```

### Token Cost

~24 tokens per skill + field lengths. System prompt overhead: 195 chars base + 97 chars per skill.

### Security Warning

**Treat third-party skills as untrusted code.** v2026.2.6 added a code safety scanner, but it's not foolproof. Read skills before enabling.

---

## Supported AI Models (v2026.2.6)

| Provider | Models |
|----------|--------|
| **Anthropic** | Claude Opus 4.6, Sonnet 4.5, earlier versions |
| **OpenAI** | GPT-5.3-Codex, GPT-4o, earlier versions |
| **xAI** | Grok (new in v2026.2.6) |
| **Baidu** | Qianfan (new in v2026.2.6) |
| **Google** | Gemini models |
| **Local** | Llama 3, any Ollama-compatible model |

Forward-compatibility fallbacks for new model identifiers (e.g., Opus 4.6 → graceful downgrade if provider unavailable).

---

## Integrations (50+)

| Category | Platforms |
|----------|-----------|
| **Messaging** | WhatsApp, Telegram, Discord, Slack, Signal, iMessage (BlueBubbles), Microsoft Teams, Matrix, Google Chat, Zalo, WebChat |
| **Productivity** | Gmail, Google Calendar, Notion, GitHub |
| **Services** | Stripe, AWS, Zapier, Spotify, YouTube, Figma, Dropbox, Airtable, Twitter/X |
| **Smart Home** | Various devices via integrations |
| **Blockchain** | Base (Ethereum L2), MetaMask, DEXs via skills |

---

## v2026.2.6 Release (Feb 7, 2026) — What's New

### Models
- Anthropic Opus 4.6 + OpenAI GPT-5.3-Codex with forward-compat fallbacks
- xAI Grok provider
- Baidu Qianfan provider

### Security
- **Skill/plugin code safety scanner** — scans community skills for malicious patterns
- Credential redaction from `config.get` gateway responses
- Gateway canvas host + A2UI assets now require authentication

### Features
- **Token usage dashboard** in Web UI
- **Voyage AI** native support for memory embeddings
- Session history payload caps (prevents context overflow)
- CLI help output alphabetically ordered

### Fixes
- Scheduling/reminder delivery regression fixed
- Telegram: auto-injects DM topic threadId
- Slack: mention stripPatterns for `/new` and `/reset`
- Chrome extension: bundled path resolution
- Compaction retry logic for context overflow
- Better billing error messaging

---

## Memory System

### Default (Markdown-based)

- Short-term: active chat logs (immediate context)
- Long-term: Markdown summaries in `~/.openclaw/memory`
- Semantic search over memory chunks (~400 tokens) via `memory_search` tool

### QMD Plugin (v2026.2.2+)

**Local Hybrid Intelligence engine** combining three search technologies:

| Technology | Type | Strength |
|------------|------|----------|
| BM25 | Keyword | Exact term matching |
| Vector | Semantic | Meaning-based search |
| LLM Rerank | Re-ranking | Contextual relevance |

Benefits:
- Searches local files (Markdown, Notion exports, Obsidian)
- Pulls only 2-3 relevant sentences into prompt
- 60-97% token savings
- Runs entirely offline
- No more context overflow

### Voyage AI (v2026.2.6+)

Native vector embedding support for enhanced memory retrieval.

---

## Multi-Agent Support

- Each agent gets its own isolated workspace
- Separate authentication and session stores
- Routing: peer → guildId → teamId → accountId → channel → default agent
- Per-agent skills in workspace directories
- Shared skills in `~/.openclaw/skills` visible to all agents
- Session scope modes: `per-channel-peer` prevents context leakage in multi-user scenarios

---

## Onchain / Web3 Integration

Since v2026.2.2, deeper integration with Base (Coinbase's Ethereum L2):
- Agents can initiate and manage blockchain transactions autonomously
- Community projects: 4claw, lobchanai, starkbotai
- MetaMask + DEX (ColorPool) connectivity via skills
- Virtual Protocol: any OpenClaw agent can discover, hire, and pay other agents on-chain
- Use cases: wallet monitoring, airdrop automation, autonomous trading

---

## Installation

```bash
# macOS/Linux
curl -fsSL https://openclaw.ai/install.sh | bash

# Windows
iwr -useb https://openclaw.ai/install.ps1 | iex

# npm
npm i -g openclawd

# Homebrew
brew install openclawd
```

**Requirements**: Node.js 22+

**Setup**: `openclaw onboard --install-daemon` runs interactive wizard.

---

## Pricing

| Component | Cost |
|-----------|------|
| Software | Free (MIT License) |
| Hardware | $0 (existing machine) to $599 (Mac mini M2) |
| API usage | ~$5-50/month depending on model + usage |
| OpenClawd hosted | Managed service (launched Feb 10, 2026) |

No subscription — pay-per-API-request only.

---

## Security Analysis — The Hard Truths

### What It Needs Access To

- File system (read/write/delete)
- Shell (arbitrary command execution)
- Browser sessions (headless Chromium via CDP)
- API keys, credentials, SSH keys
- Email, calendar, messaging accounts

### Known Vulnerability Vectors

| Vector | Risk | Severity |
|--------|------|----------|
| **Privilege Escalation** | Runs with full user permissions — access to SSH keys, `.env`, browser cookies | Critical |
| **Indirect Prompt Injection** | Reads websites/docs with hidden instructions → executes exfiltration commands | Critical |
| **Plaintext Storage** | API keys and chat history in readable `config.json` and `.md` files | High |
| **Supply Chain (Skills)** | Community skills on ClawHub may contain malicious scripts (wallet draining, backdoors) | High |
| **Context Overflow** | Session payloads can overflow, causing unpredictable behavior | Medium |

### Mitigation Best Practices

1. **Containerize** — Docker/VM mandatory, never run on bare metal with sensitive data
2. **Network isolation** — Firewall monitoring outbound traffic (Little Snitch on macOS)
3. **Human-in-the-loop** — Require approval for bash, file deletion, git push
4. **Scoped API keys** — Project-specific credentials, budget cap ($5 max)
5. **Ephemeral browser** — Clean, wiped instances separate from primary profile
6. **Treat as untrusted** — Design systems assuming the agent is compromised

> IBM Distinguished Engineer Chris Hay: OpenClaw exposes users to "too many security vulnerabilities" for enterprise use.

---

## Alternatives Comparison

| Tool | Type | Key Difference |
|------|------|----------------|
| **Nanobot** | Lightweight (4K lines Python vs OpenClaw's 430K+) | Research-friendly, auditable |
| **Moltworker** | Cloudflare Workers deployment | Serverless, sandboxed, persistent state |
| **Zapier** | Cloud automation | No-code, no system access needed |
| **Knolli** | Enterprise AI copilot | Structured permissions, managed infra |
| **Claude Code** | Anthropic CLI agent | Code-focused, built-in sandboxing |
| **Cursor/Windsurf** | IDE-integrated AI | Editor-bound, less autonomous |

### Why OpenClaw Wins

- Fully open-source, no vendor lock-in
- Model-agnostic (any LLM)
- Messaging-native (chat with your agent via WhatsApp)
- 100+ skills ecosystem
- Community-driven (145K+ stars, 20K+ forks)

### Why OpenClaw Loses

- Security nightmare without proper containerization
- 430K+ lines of code — massive attack surface
- Enterprise-hostile (IBM: "too many vulnerabilities")
- Plaintext secrets storage
- Community skills = supply chain risk

---

## Key Insights

### 1. Loose Integration Beats Vertical Integration (Sometimes)

IBM's Kaoutar El Maghraoui: "Vertical integration is important in certain domains because of the security aspect. But in other domains, maybe we don't need that."

OpenClaw proves autonomous agents don't require a single vendor controlling models, memory, tools, and interfaces. Community-driven, open-source agents can be "incredibly powerful if they have full system access."

### 2. The Agent = Gateway Pattern

OpenClaw's core insight: the **gateway** is the right abstraction for agentic AI. One process that mediates between LLM brain and OS arms, with memory as the state layer. This is the pattern every personal AI agent will converge on.

### 3. Skills Are The New Plugins

The SKILL.md declarative format (YAML frontmatter + natural language instructions) is the equivalent of browser extensions for AI agents. ClawHub is the Chrome Web Store. Same ecosystem dynamics — and same supply chain risks.

### 4. Memory Is The Moat

The QMD plugin (BM25 + Vector + LLM Rerank, all local) achieves 60-97% token savings. This is where OpenClaw's real value compounds — the longer you use it, the more it knows about you. Switching costs become astronomical.

### 5. Web3 Integration Is Real (Not Just Hype)

Unlike most "crypto + AI" plays, OpenClaw's onchain integration has practical use cases: autonomous trading, wallet monitoring, agent-to-agent payments. The Base ecosystem (Coinbase L2) is building real infrastructure here.

---

## Stealable Patterns

| Pattern | What To Steal |
|---------|---------------|
| **Gateway-as-control-plane** | Single process mediating LLM ↔ OS, with session/channel/tool management |
| **SKILL.md declarative format** | YAML frontmatter + natural language = portable, model-agnostic tool definitions |
| **Gating system** | Auto-filter capabilities by OS, bins, env vars — zero config for users |
| **Memory hierarchy** | Short-term (chat) → Long-term (curated .md) → Semantic search (QMD/Voyage) |
| **Workspace isolation** | Per-agent workspace with separate auth + session stores for multi-agent |
| **Session scope modes** | `per-channel-peer` prevents context leakage — critical for multi-user |
| **Forward-compat fallbacks** | New model identifiers gracefully downgrade — never crash on unknown model |
| **Heartbeat pattern** | `HEARTBEAT.md` automated health checks — self-healing agent infrastructure |
