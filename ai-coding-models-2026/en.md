# AI Coding Models: OpenAI vs Anthropic — The Complete 2026 Landscape

## Executive Summary

As of February 7, 2026, Anthropic and OpenAI are locked in the closest AI coding race in history. Two days ago (Feb 5), both companies released competing models within 20 minutes of each other: Claude Opus 4.6 and GPT-5.3-Codex. Neither company has a clear overall lead — each dominates different benchmarks, workflows, and developer personas.

**The bottom line:** Claude leads on SWE-bench Verified (patch generation). GPT-5.3-Codex leads on Terminal-Bench 2.0 (agentic terminal workflows). The real competition has shifted from "which model is smarter" to "which tool/workflow is more productive."

---

## 1. The February 5, 2026 Head-to-Head

| Benchmark | Claude Opus 4.6 | GPT-5.3-Codex | Winner |
|-----------|-----------------|---------------|--------|
| SWE-bench Verified | **80.8%** | Not reported | Claude |
| SWE-bench Pro Public | Not reported | **56.8%** | GPT-5.3 |
| Terminal-Bench 2.0 | 65.4% | **77.3%** | GPT-5.3 (+12pp) |
| OSWorld-Verified | **72.7%** | 64.7% | Claude (+8pp) |
| GPQA Diamond | **77.3%** | 73.8% | Claude |
| MMLU Pro | **85.1%** | 82.9% | Claude |
| GDPval-AA (Elo) | **1,606** | ~1,462 | Claude (+144 Elo) |
| Cybersecurity CTF | — | **77.6%** | GPT-5.3 |

**Critical caveat:** Anthropic reports SWE-bench Verified; OpenAI reports SWE-bench Pro Public. Different benchmark variants, different problem sets. This is a deliberate strategy — each claims "leadership" on their preferred metric.

---

## 2. Complete Model Timelines

### OpenAI (Apr 2025 — Feb 2026)

| Date | Model | SWE-bench Verified | Key Feature |
|------|-------|--------------------|-------------|
| Apr 14 | GPT-4.1 / mini / nano | 54.6% | API coding model, 1M context, $0.10-$2.00/MTok |
| Apr 16 | o3 | 69.1% | Reasoning model, Codeforces 2727 Elo |
| Apr 16 | o4-mini | 68.1% | Cost-efficient reasoning, Codeforces 2719 |
| Jun 10 | o3-pro | — | Max-compute, $20/$80 per MTok |
| Jun 10 | o3 price drop | — | 80% cheaper: $10/$40 → $2/$8 |
| Aug 7 | GPT-5 | 74.9% | Unified flagship, 22% fewer tokens than o3 |
| Nov 12 | GPT-5.1 + Codex-Max | 77.9% | Long-horizon coding, 24hr autonomous task |
| Dec 18 | GPT-5.2 + Codex | 75.4% | Context compaction, 56.4% SWE-bench Pro |
| Feb 5 | **GPT-5.3-Codex** | — | 77.3% Terminal-Bench, 25% faster |

### Anthropic (May 2025 — Feb 2026)

| Date | Model | SWE-bench Verified | Key Feature |
|------|-------|--------------------|-------------|
| May 22 | Opus 4 | 72.5% | First Claude 4, "best coding model" at launch |
| May 22 | Sonnet 4 | — | Hybrid reasoning (instant + extended thinking) |
| Aug 5 | Opus 4.1 | 74.5% | Multi-file refactoring |
| Sep | Sonnet 4.5 | 77.2% | Agentic workflow focus, 82% with parallel compute |
| Oct | Haiku 4.5 | 73.3% | Small model matching Opus 4 |
| Nov 24 | Opus 4.5 | **80.9%** | First model >80%, 67% price cut |
| Feb 5 | **Opus 4.6** | 80.8% | 1M context, Agent Teams, 500+ zero-days found |

---

## 3. OpenAI Deep Dive

### GPT-4.1 Family — The Coding Workhorse

API-only. Three sizes (GPT-4.1 / mini / nano). 1M token context. Design decision: separated "coding" from "reasoning" — GPT-4.1 for everyday coding, o3 for hard reasoning. This dual-track persisted until GPT-5 merged them.

| Feature | GPT-4.1 | mini | nano |
|---------|---------|------|------|
| Context | 1M | 1M | 1M |
| SWE-bench | 54.6% | — | — |
| Input/MTok | $2.00 | $0.40 | $0.10 |
| Output/MTok | $8.00 | $1.60 | $0.40 |

Why it matters: 60% higher than GPT-4o on Windsurf coding benchmark. 30% more efficient tool calling. Available in Copilot day one.

### o3 / o4-mini — Reasoning for Code

o3 thinks before it codes — extended chain-of-thought for multi-step refactors and complex bug chains. Codeforces 2727 Elo (surpassed OpenAI's Chief Scientist at 2665). Tradeoff: slow, minutes per request.

o4-mini beats o3 on Codeforces (2719 vs 2706) at ~50% less cost. Better per-dollar for competitive programming, but o3 wins on real-world SWE-bench (69.1% vs 68.1%).

### GPT-5 — The Generational Leap (Aug 2025)

Replaced GPT-4o, o3, o4-mini, GPT-4.1 as ChatGPT default. Key insight: achieved better results than o3 while using **22% fewer tokens** and **45% fewer tool calls**. Token efficiency directly translates to agent cost savings. $1.25/$10.00 per MTok.

### GPT-5.3-Codex — The Latest (Feb 5, 2026)

| Benchmark | GPT-5.3-Codex | vs GPT-5.2 |
|-----------|---------------|------------|
| Terminal-Bench 2.0 | 77.3% | **+13.3pp** |
| OSWorld-Verified | 64.7% | **+26.5pp** |
| Cybersecurity CTF | 77.6% | +10.2pp |
| SWE-bench Pro | 56.8% | +0.4pp |

25% faster inference. Fewer output tokens than any prior model. Deep diffs (transparent reasoning about changes). Interactive steering mid-task. First model designated "High capability for cybersecurity."

The real story: SWE-bench Pro improvement is marginal (+0.4%), but Terminal-Bench (+13.3pp) and OSWorld (+26.5pp) are massive — dramatically better at terminal debugging and real computer interaction.

### The Codex Platform

Not just a model — a full agent platform:

```
┌─────────────────────────────────────────────┐
│  CODEX APP (Web + Desktop)                   │
│  Command center, parallel agents             │
├─────────────────────────────────────────────┤
│  CODEX CLI (Terminal)                        │
│  Open-source Rust, runs locally              │
├─────────────────────────────────────────────┤
│  CODEX MODELS (API)                          │
│  codex-1 → GPT-5-Codex → 5.2 → 5.3         │
└─────────────────────────────────────────────┘
```

Key design: codex-1 = o3 fine-tuned via RL on real coding tasks. Sandboxed (no internet). AGENTS.md for per-repo config. Built-in git worktrees for parallel agents. Automations (scheduled triggers for issue triage, CI monitoring).

Codex Desktop (Feb 2, 2026): macOS app, agents run 30 min independently, built-in worktrees. Sam Altman: "most loved internal product we've ever had."

---

## 4. Anthropic Claude Deep Dive

### Opus 4.6 — The Latest (Feb 5, 2026)

| Feature | Opus 4.5 | Opus 4.6 |
|---------|----------|----------|
| Context | 200K | **1M (beta)** |
| Output tokens | 32K | **128K** |
| Terminal-Bench 2.0 | 59.3% | **65.4%** |
| OSWorld | 66.3% | **72.7%** |
| Long-context retrieval | ~18% | **76%** (4x improvement) |
| Agent Teams | No | **Yes** |
| Effort controls | No | **Yes (low/med/high/max)** |

**Agent Teams** — Multiple Claude Code instances coordinate autonomously. Lead spawns teammates, teammates message each other directly. Shared task list with dependency tracking. Enable: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Stress test: 16 agents wrote 100K-line C compiler that builds Linux 6.9.

**500+ Zero-Days Found** — Anthropic red team gave Opus 4.6 Python, debuggers, and fuzzers in sandbox. Found 500+ previously unknown high-severity bugs autonomously (GhostScript crashes, OpenSC buffer overflows, CGIF memory corruption).

**Pricing:** $5/$25 per MTok (same as 4.5). 67% cheaper than Opus 4/4.1's $15/$75.

### Sonnet 4.5 — The Daily Driver

77.2% SWE-bench (82% with parallel compute). Best speed/quality ratio. $3/$15 per MTok. Highest pass@5 (55.1%) — uniquely solves instances no other model can.

### Haiku 4.5 — The Sleeper

73.3% SWE-bench — matches Opus 4. $1/$5 per MTok. 5x cheaper than Sonnet. For write-test-fix loops where speed matters more than peak quality.

### Claude Code CLI

Terminal-based agent, local execution, direct API connection. $1B annualized revenue within 6 months. Available on GitHub Copilot since Feb 5, 2026.

Key capabilities: full repo traversal, multi-file editing, MCP integration (Figma, Jira, GitHub), session teleportation, Chrome integration, skill hot-reloading, LSP features.

Developer shift: "I am in reviewer mode more often than coding mode." "What would have taken 2 years was completed in 6 months."

---

## 5. Agent Coding Tool Comparison

### The Big Four

| Feature | Claude Code | OpenAI Codex | Cursor | GitHub Copilot |
|---------|------------|-------------|--------|----------------|
| Interface | Terminal (CLI) | Cloud + Desktop (Mac) | VS Code fork | IDE extension |
| Multi-agent | Agent Teams | Parallel worktrees | Single | Agent Mode |
| Execution | Local-first | Cloud-first | Local + cloud | Cloud |
| Autonomy | High (human-in-loop) | Very high (30min) | Medium | Medium |
| Token efficiency | 5.5x better than Cursor | Not disclosed | 100K-400K/request | Not disclosed |
| MCP integration | Best | Limited | Model-agnostic | GitHub ecosystem |
| Models | Claude only | GPT only | Multi-model | Multi-model |
| Price | $20-200/mo | Free tier available | $20-200/mo | $10-39/mo |

### Emerging Alternatives

Cline (VS Code, max flexibility), Aider (open-source, terminal), Windsurf (AI IDE), JetBrains Junie, Gemini CLI, AWS Kiro, Augment (enterprise).

---

## 6. Pricing Landscape

### OpenAI API

| Model | Input/MTok | Output/MTok | Best For |
|-------|-----------|------------|----------|
| GPT-4.1 nano | $0.10 | $0.40 | Autocomplete |
| GPT-4.1 mini | $0.40 | $1.60 | Everyday coding |
| o4-mini | $1.10 | $4.40 | Competitive programming |
| GPT-5 | $1.25 | $10.00 | General coding + reasoning |
| GPT-5.2 | $1.75 | $14.00 | Agentic coding |
| o3 (post-cut) | $2.00 | $8.00 | Hard reasoning |
| o3-pro | $20.00 | $80.00 | Maximum-difficulty |

### Anthropic API

| Model | Input/MTok | Output/MTok | Best For |
|-------|-----------|------------|----------|
| Haiku 4.5 | $1 | $5 | Quick fixes, iteration |
| Sonnet 4.5 | $3 | $15 | Daily coding |
| Opus 4.6 | $5 | $25 | Complex architecture |
| Opus 4.6 (>200K) | $10 | $37.50 | Large codebase analysis |

**Cost optimization:** Prompt caching = 90% off input. Batch API = 50% off. Both stack multiplicatively.

**Key insight:** Cost-per-task > cost-per-token. GPT-5 uses 22% fewer tokens than o3. Claude Code uses 5.5x fewer tokens than Cursor. The cheapest per-token is not always cheapest per-task.

---

## 7. Benchmark Deep Dive

### SWE-bench Verified (Top Performers, Feb 2026)

| Rank | Model | Score |
|------|-------|-------|
| 1 | Claude Opus 4.6 (Thinking) | 80.8% |
| 2 | Claude Opus 4.5 | 80.9% (pass@1: 74.4%) |
| 3 | Gemini 3 Flash | 76.2% |
| 4 | GPT-5.2 | 75.4% |
| 5 | GLM-4.7 (open-source) | ~74.2% |
| 6 | Qwen3-Coder-Next (open-source) | 70.6% |

### Terminal-Bench 2.0 (Agentic CLI)

| Rank | Model | Score |
|------|-------|-------|
| 1 | GPT-5.3-Codex | **77.3%** |
| 2 | Claude Opus 4.6 | 65.4% |
| 3 | GPT-5.2-Codex | 64.0% |
| 4 | Claude Opus 4.5 | 59.8% |

### OSWorld-Verified (Desktop Automation)

| Model | Score | Human Baseline |
|-------|-------|---------------|
| Claude Opus 4.6 | **72.7%** | ~72% |
| GPT-5.3-Codex | 64.7% | ~72% |

Claude matches human baseline. GPT-5.3 made a 26pp jump from predecessor.

### SWE-bench Variants Explained

| Variant | Focus | Reality Check |
|---------|-------|---------------|
| Verified | 500 human-validated samples | Gold standard, ~80% top score |
| Pro | Enterprise, multi-language | Much harder, top ~57% |
| Live | Fresh problems, anti-contamination | Tests true generalization |
| Terminal-Bench 2.0 | Multi-step terminal workflows | Most realistic for agent coding |

SWE-bench Pro scores (23-57%) reveal the real gap vs Verified (~80%). The benchmark you use dramatically affects the story.

---

## 8. Developer Ecosystem & Sentiment

### Adoption (2025-2026)

| Metric | Value |
|--------|-------|
| Devs using AI tools regularly | 85% |
| Weekly AI coding assistant usage | 65% |
| ChatGPT usage (among AI users) | 82% |
| Copilot usage | 68% |
| Gemini usage | 47% |
| Claude usage | 41% |
| Using multiple tools | 26%+ |
| Codex users since Dec 2025 | 1M+ |

### The METR Bombshell

Randomized controlled trial — 16 experienced devs, 246 tasks:
- AI tools made devs **19% slower** on familiar codebases
- Devs estimated AI saved 20-24% — the **opposite** of reality
- High-familiarity devs were slowed down MORE
- Context: early 2025 tools (Cursor Pro, Claude 3.5/3.7 Sonnet). Newer models may differ.

**The paradox:** AI most valuable when you DON'T know the codebase — exactly when most devs reach for it.

### Community Workflow Consensus

No single tool wins. Multi-tool strategies dominate:
- **Cursor** — inline edits, fast iteration (the baseline)
- **Claude Code** — complex reasoning, debugging, architecture ("strongest coding brain")
- **Copilot** — quick completions, low friction
- **Codex** — parallel delegation, multi-agent ("best PR reviewer")
- **Cline** — power users wanting maximum control

### Coding-Writing Tradeoff (Opus 4.6)

Users report improved coding but degraded writing quality. Consensus: use Opus 4.6 for code, Opus 4.5 for prose. Anthropic optimized heavily for reasoning benchmarks — it shows.

---

## 9. Non-Obvious Insights

### 1. Benchmark Fragmentation Is Strategic
Both companies report on different SWE-bench variants. Smart consumers look at Terminal-Bench 2.0 and OSWorld — both report on these.

### 2. Open Source Is Closing Fast
Qwen3-Coder-Next (3B active out of 80B params) hits 70.6% SWE-bench. The gap between frontier closed-source and best open-source collapsed from 30+ points to under 10.

### 3. Multi-Agent Is the New Frontier
Single-agent coding is becoming commodity. Differentiation moved to orchestration: Claude Agent Teams (swarm, self-organizing) vs Codex Desktop (parallel worktrees, 30min autonomous).

### 4. Token Efficiency Is the Hidden Battleground
Claude Code uses 5.5x fewer tokens than Cursor. GPT-5 uses 22% fewer than o3. At scale, this matters more than per-token pricing.

### 5. Enterprise Is Cautious
90% of enterprise teams use AI, but AI-generated PRs wait 4.6x longer in review. Positive sentiment dropped from 70% to 60%. The "almost right but not quite" problem (66% cite this).

---

## 10. Stealable Patterns

| Pattern | Description |
|---------|-------------|
| **Model routing** | Haiku/nano for iteration, Sonnet/GPT-5 for daily work, Opus/Codex for architecture |
| **AGENTS.md / CLAUDE.md** | Per-repo config for agent behavior — single highest-impact quality improvement |
| **Worktree-per-agent** | Each agent gets own git worktree for conflict-free parallelism |
| **Multi-tool workflow** | Cursor (inline) + Claude Code (complex) + Copilot (completion) + Codex (parallel) |
| **Cost-per-task tracking** | Track total tokens per completed task, not per-token cost |
| **Sandboxed execution** | No internet for agents, pre-installed deps, limits supply chain risk |
| **Context compaction** | Summarization checkpoints for long agent sessions |
| **Prompt caching** | 90% savings on input, stack with batch API for multiplicative discount |
| **Terminal-Bench for eval** | More realistic than SWE-bench Verified for agent evaluation |
| **Codex for PR review** | Even Claude-primary teams should use Codex PR reviewer |
| **Version-specific selection** | Opus 4.6 for code, Opus 4.5 for writing/documentation |
| **Effort controls** | `/effort low` for simple completions, `/effort max` for hard problems |

---

## Latest Updates (2026)

### Anthropic's Explosive Growth and $30B Series G

Anthropic closed a **$30 billion Series G** on February 12, 2026 at a **$380 billion post-money valuation** — the second-largest venture funding deal of all time. Annual run-rate revenue reached **$14 billion**, growing over 10x annually for three consecutive years. Eight of the Fortune 10 are now Claude customers. Claude Code alone hit **$2.5 billion** in annualized run-rate revenue (more than doubled since January 1, 2026), with weekly active users also doubling in the same period. Anthropic also launched **Cowork** in January 2026, extending Claude Code's engineering capabilities to non-coding knowledge work (sales, legal, finance) via eleven open-source plugins.

Sources: [TechCrunch](https://techcrunch.com/2026/02/12/anthropic-raises-another-30-billion-in-series-g-with-a-new-value-of-380-billion/), [Anthropic](https://www.anthropic.com/news/anthropic-raises-30-billion-series-g-funding-380-billion-post-money-valuation), [Constellation Research](https://www.constellationr.com/insights/news/anthropics-claude-code-revenue-doubled-jan-1)

### Cursor 2.0 and the Composer Model

Cursor released **version 2.0** (October 2025) with its first proprietary model: **Composer**, a mixture-of-experts model trained via reinforcement learning inside real codebases using actual dev tools (semantic search, file editors, terminal commands). Composer completes most interactions in under 30 seconds — claimed **4x faster** than similarly capable systems. Cursor 2.0 supports up to **8 parallel agents**, each in an isolated workspace clone. By February 2026, **Composer 1.5** launched with 20x scaled RL training, scoring **47.9% on Terminal-Bench 2.0** with adaptive thinking and self-summarization. Cursor crossed **$500M ARR** and a **$10B valuation**, with over 50% of Fortune 500 companies adopting it (including Nvidia, Uber, Adobe). A new **plugin system** (February 2026) lets agents connect to external tools via a Marketplace (Amplitude, AWS, Figma, Stripe prebuilt).

Sources: [VentureBeat](https://venturebeat.com/ai/vibe-coding-platform-cursor-releases-first-in-house-llm-composer-promising), [The New Stack](https://thenewstack.io/cursor-2-0-ide-is-now-supercharged-with-ai-and-im-impressed/), [DigitalApplied](https://www.digitalapplied.com/blog/cursor-composer-1-5-ai-coding-model-guide)

### Devin 2.0 — From $500/month to $20/month

Cognition launched **Devin 2.0** with a radical price drop: from **$500/month** to a **$20/month Core plan**. The new version introduces an interactive cloud-based IDE allowing users to spin up multiple Devins in parallel. According to Cognition, Devin 2.0 completes **83% more junior-level tasks per Agent Compute Unit** versus its predecessor. Pricing tiers: Core ($20/mo), Team ($500/mo with 250 ACUs), Enterprise (custom). Additional ACUs cost $2 each.

Sources: [VentureBeat](https://venturebeat.com/programming-development/devin-2-0-is-here-cognition-slashes-price-of-ai-software-engineer-to-20-per-month-from-500), [TechCrunch](https://techcrunch.com/2025/04/03/devin-the-viral-coding-ai-agent-gets-a-new-pay-as-you-go-plan/)

### GitHub Copilot Agent Mode Goes GA

GitHub Copilot's agent mode reached general availability with significant new capabilities: a **find_symbol tool** for language-aware symbol navigation (C++, C#, TypeScript, and any language with LSP support), **Agent Skills** for tailoring Copilot to specific project workflows, and **Copilot Coding Agent** — an asynchronous autonomous background agent that works in its own GitHub Actions environment, then opens a PR for review. The **Explore agent** now integrates with GitHub MCP tools for broader tool connectivity.

Sources: [GitHub Newsroom](https://github.com/newsroom/press-releases/coding-agent-for-github-copilot), [GitHub Features](https://github.com/features/copilot/whats-new)

### Google Gemini Code Assist Agent Mode

Google's **Gemini Code Assist** agent mode is now available to all users (no longer insiders-only). Features include: persistent agent/chat state between IDE restarts, real-time shell command output, **Next Edit Predictions** (Preview in VS Code), inline diff views in chat, and direct diff editing. Higher model request limits are now shared across **Gemini CLI, agent mode, and Gemini Code Assist** for Google AI Pro/Ultra subscribers. The underlying model is **Gemini 2.5**.

Sources: [Google Developers Blog](https://developers.googleblog.com/new-in-gemini-code-assist/), [Google Developers](https://developers.google.com/gemini-code-assist/resources/release-notes)

### Open-Source Models Close the Gap Further

**DeepSeek V3.2** (685B parameters, 128K context) scores **70.2% on SWE-bench Verified** in two variants (standard and V3.2-Speciale for intensive math/coding). **MiniMax M2.5** reached **80.2%** on SWE-bench Verified — a non-Anthropic model cracking the 80% barrier for the first time. **GLM-5** (Zhipu AI) hit **77.8%** and **Kimi K2.5** reached **76.8%**. The SWE-bench Verified leaderboard now has **71 evaluated models**, with the top-5 gap between open and closed source narrowing to under 1 percentage point in some comparisons.

Sources: [llm-stats.com](https://llm-stats.com/benchmarks/swe-bench-verified), [VentureBeat](https://venturebeat.com/technology/qwen3-coder-next-offers-vibe-coders-a-powerful-open-source-ultra-sparse), [Epoch AI](https://epoch.ai/benchmarks/swe-bench-verified)

### Windsurf & AWS Kiro Join the IDE Wars

**Windsurf** (Wave 13) shipped first-class **multi-agent parallel sessions** with Git worktrees, side-by-side Cascade panes, and **Arena Mode** (blind side-by-side model comparison inside the IDE). **AWS Kiro** launched as a new spec-driven agentic IDE built on VS Code, powered by Claude Sonnet 4.0/3.7, using a structured development methodology: Kiro auto-generates `requirements.md` with user stories and EARS acceptance criteria before writing code. Aurora DSQL MCP server integration enables one-click database-aware development.

Sources: [Windsurf Changelog](https://windsurf.com/changelog), [VentureBeat](https://venturebeat.com/programming-development/amazon-launches-kiro-its-own-claude-powered-challenger-to-windsurf-and-codex)

### Developer Adoption Hits 93%

According to the **JetBrains AI Pulse** (January 2026), **93% of developers** now regularly use AI tools for coding — up from 85% in mid-2025. MIT Technology Review named **generative coding** one of its [10 Breakthrough Technologies of 2026](https://www.technologyreview.com/2026/01/12/1130027/generative-coding-ai-software-2026-breakthrough-technology/). The shift is no longer "should I use AI?" but "which combination of AI tools maximizes my throughput?"

Sources: [JetBrains AI Blog](https://blog.jetbrains.com/ai/2026/02/the-best-ai-models-for-coding-accuracy-integration-and-developer-fit/), [MIT Technology Review](https://www.technologyreview.com/2026/01/12/1130027/generative-coding-ai-software-2026-breakthrough-technology/)

---

## Verdict: Who Wins?

**No single winner.**

| Category | Leader | Margin |
|----------|--------|--------|
| SWE-bench Verified | Claude Opus 4.6 | ~5pp over GPT-5.2 |
| Terminal-Bench 2.0 | GPT-5.3-Codex | ~12pp over Claude |
| OSWorld (desktop) | Claude Opus 4.6 | ~8pp over GPT-5.3 |
| Reasoning (GPQA, MMLU) | Claude Opus 4.6 | ~2-4pp |
| Speed | GPT-5.3-Codex | 25% faster |
| Context window | Claude Opus 4.6 | 1M vs 400K |
| Multi-agent | Tie | Different approaches |
| IDE integration | Cursor | Uses both models |
| Enterprise adoption | GitHub Copilot | Market dominance |
| Open-source | Qwen3-Coder-Next | 70.6% at fraction of cost |
| Developer mindshare (code) | Claude | "Strongest coding brain" |
| Developer mindshare (speed) | Codex | "Move fast and iterate" |
| Token efficiency | Claude Code | 5.5x vs Cursor |

---

## References

### Official -- OpenAI

- [Introducing OpenAI o3 and o4-mini](https://openai.com/index/introducing-o3-and-o4-mini/)
- [Introducing GPT-4.1 in the API](https://openai.com/index/gpt-4-1/)
- [Introducing GPT-5](https://openai.com/index/introducing-gpt-5/)
- [Introducing GPT-5 for developers](https://openai.com/index/introducing-gpt-5-for-developers/)
- [Introducing GPT-5.2](https://openai.com/index/introducing-gpt-5-2/)
- [Introducing GPT-5.2-Codex](https://openai.com/index/introducing-gpt-5-2-codex/)
- [Introducing GPT-5.3-Codex](https://openai.com/index/introducing-gpt-5-3-codex/)
- [GPT-5.3-Codex System Card (PDF)](https://cdn.openai.com/pdf/23eca107-a9b1-4d2c-b156-7deb4fbc697c/GPT-5-3-Codex-System-Card-02.pdf)
- [Introducing Codex](https://openai.com/index/introducing-codex/)
- [Introducing the Codex App](https://openai.com/index/introducing-the-codex-app/)
- [Introducing upgrades to Codex](https://openai.com/index/introducing-upgrades-to-codex/)
- [Unlocking the Codex harness: App Server architecture](https://openai.com/index/unlocking-the-codex-harness/)
- [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [Codex product page](https://openai.com/codex/)
- [O3 80% price cut + o3-pro](https://community.openai.com/t/o3-is-80-cheaper-and-introducing-o3-pro/1284925)
- [OpenAI for Developers 2025](https://developers.openai.com/blog/openai-for-developers-2025/)
- [OpenAI API Pricing](https://openai.com/api/pricing/)
- [OpenAI Models comparison](https://platform.openai.com/docs/models/compare)

### Official -- Anthropic

- [Introducing Claude 4 (Opus 4 + Sonnet 4)](https://www.anthropic.com/news/claude-4)
- [Claude Opus 4.1](https://www.anthropic.com/news/claude-opus-4-1)
- [Introducing Claude Sonnet 4.5](https://www.anthropic.com/news/claude-sonnet-4-5)
- [Introducing Claude Opus 4.5](https://www.anthropic.com/news/claude-opus-4-5)
- [Introducing Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)
- [500+ Zero-Days Found (Red Team)](https://red.anthropic.com/2026/zero-days/)
- [Claude's Extended Thinking](https://www.anthropic.com/news/visible-extended-thinking)
- [The "think" Tool](https://www.anthropic.com/engineering/claude-think-tool)
- [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use)
- [Building C Compiler with Agent Teams](https://www.anthropic.com/engineering/building-c-compiler)
- [Claude SWE-Bench Performance](https://www.anthropic.com/research/swe-bench-sonnet)
- [Claude API Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Models Overview](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Claude Code Overview](https://code.claude.com/docs/en/overview)
- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents)
- [Anthropic raises $30B Series G at $380B valuation](https://www.anthropic.com/news/anthropic-raises-30-billion-series-g-funding-380-billion-post-money-valuation)

### Developer Documentation

- [Codex CLI on GitHub](https://github.com/openai/codex)
- [Codex CLI docs](https://developers.openai.com/codex/cli/)
- [Codex Models](https://developers.openai.com/codex/models/)
- [Codex Cloud](https://developers.openai.com/codex/cloud)
- [Gemini Code Assist overview](https://developers.google.com/gemini-code-assist/docs/overview)
- [Gemini Code Assist release notes](https://developers.google.com/gemini-code-assist/resources/release-notes)
- [GitHub Copilot features](https://docs.github.com/en/copilot/get-started/features)

### Benchmark Leaderboards

- [SWE-bench Verified -- llm-stats.com](https://llm-stats.com/benchmarks/swe-bench-verified)
- [SWE-bench Official](https://www.swebench.com/)
- [SWE-bench Pro Public -- Scale](https://scale.com/leaderboard/swe_bench_pro_public)
- [SWE-bench Live](https://swe-bench-live.github.io/)
- [Terminal-Bench 2.0 -- llm-stats.com](https://llm-stats.com/benchmarks/terminal-bench-2)
- [Terminal-Bench Official](https://www.tbench.ai/)
- [Live-SWE-agent Leaderboard](https://live-swe-agent.github.io/)
- [Best AI for Coding -- llm-stats.com](https://llm-stats.com/leaderboards/best-ai-for-coding)
- [SWE-bench Verified -- Epoch AI](https://epoch.ai/benchmarks/swe-bench-verified)
- [LLM Leaderboard -- Vellum](https://www.vellum.ai/llm-leaderboard)
- [Best LLM for Coding -- Vellum](https://www.vellum.ai/best-llm-for-coding)
- [EvalPlus Leaderboard (HumanEval)](https://evalplus.github.io/leaderboard.html)
- [LiveBench](https://livebench.ai/)
- [Artificial Analysis](https://artificialanalysis.ai/models)

### Head-to-Head Comparisons

- [Claude Opus 4.6 vs GPT-5.3 Codex -- DigitalApplied](https://www.digitalapplied.com/blog/claude-opus-4-6-vs-gpt-5-3-codex-comparison)
- [Claude vs Codex: AI Coding Agent Battle 2026 -- WaveSpeedAI](https://wavespeed.ai/blog/posts/claude-vs-codex-comparison-2026/)
- [Codex App vs Claude Code Showdown -- Serenities AI](https://serenitiesai.com/articles/openai-codex-app-vs-claude-code-2026)
- [Claude Code vs OpenAI Codex -- Northflank](https://northflank.com/blog/claude-code-vs-openai-codex)
- [Codex vs Claude Code -- Builder.io](https://www.builder.io/blog/codex-vs-claude-code)
- [GPT-5.3 Codex vs Claude Opus 4.6 -- NxCode](https://www.nxcode.io/resources/news/gpt-5-3-codex-vs-claude-opus-4-6-ai-coding-comparison-2026)
- [Codex CLI vs Claude Code Benchmark -- SmartScope](https://smartscope.blog/en/generative-ai/chatgpt/codex-vs-claude-code-2026-benchmark/)
- [Devin vs Cursor -- Builder.io](https://www.builder.io/blog/devin-vs-cursor)

### Analysis & Reviews

- [Best AI for Coding 2026: SWE-Bench Breakdown -- marc0.dev](https://www.marc0.dev/en/blog/best-ai-for-coding-2026-swe-bench-breakdown-opus-4-6-qwen3-coder-next-gpt-5-3-and-what-actually-matters-1770387434111)
- [GPT-5 Benchmarks -- Vellum](https://www.vellum.ai/blog/gpt-5-benchmarks)
- [Claude Opus 4.6 Benchmarks -- Vellum](https://www.vellum.ai/blog/claude-opus-4-6-benchmarks)
- [GPT-5.3-Codex Guide -- DigitalApplied](https://www.digitalapplied.com/blog/gpt-5-3-codex-release-features-benchmarks-guide)
- [Claude Opus 4.6 Guide -- DigitalApplied](https://www.digitalapplied.com/blog/claude-opus-4-6-release-features-benchmarks-guide)
- [Claude Opus 4.6 -- DataCamp](https://www.datacamp.com/blog/claude-opus-4-6)
- [O4-Mini -- DataCamp](https://www.datacamp.com/blog/o4-mini)
- [Best AI Coding Agents 2026 -- Faros AI](https://www.faros.ai/blog/best-ai-coding-agents-2026)
- [Best AI Coding Assistants 2026 -- PlayCode](https://playcode.io/blog/best-ai-coding-assistants-2026)
- [Best LLMs for Coding 2026 -- Builder.io](https://www.builder.io/blog/best-llms-for-coding)
- [AI Dev Tool Power Rankings -- LogRocket](https://blog.logrocket.com/ai-dev-tool-power-rankings/)
- [Best AI Coding Models 2026 -- CodeConductor](https://codeconductor.ai/blog/ai-coding-models/)
- [Best AI Models for Coding -- JetBrains AI Blog](https://blog.jetbrains.com/ai/2026/02/the-best-ai-models-for-coding-accuracy-integration-and-developer-fit/)
- [Cursor 2.0 -- The New Stack](https://thenewstack.io/cursor-2-0-ide-is-now-supercharged-with-ai-and-im-impressed/)
- [Cursor Composer 1.5 Guide -- DigitalApplied](https://www.digitalapplied.com/blog/cursor-composer-1-5-ai-coding-model-guide)
- [Devin 2.0 Review -- ai-coding-flow](https://ai-coding-flow.com/blog/devin-review-2026/)

### News Coverage

- [AI coding wars heat up -- VentureBeat](https://venturebeat.com/technology/openais-gpt-5-3-codex-drops-as-anthropic-upgrades-claude-ai-coding-wars-heat)
- [Claude Opus 4.6 1M context + Agent Teams -- VentureBeat](https://venturebeat.com/technology/anthropics-claude-opus-4-6-brings-1m-token-context-and-agent-teams-to-take)
- [GPT-5.3-Codex 25% faster -- Neowin](https://www.neowin.net/news/openai-debuts-gpt-53-codex-25-faster-and-setting-new-coding-benchmark-records/)
- [Opus 4.6 Agent Teams -- TechCrunch](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams/)
- [Claude Opus 4.6 -- CNBC](https://www.cnbc.com/2026/02/05/anthropic-claude-opus-4-6-vibe-working.html)
- [500 Zero-Days -- Axios](https://www.axios.com/2026/02/05/anthropic-claude-opus-46-software-hunting)
- [500 Zero-Days -- CSO Online](https://www.csoonline.com/article/4128889/claude-ai-finds-500-high-severity-software-vulnerabilities.html)
- [500 Zero-Days -- Fortune](https://fortune.com/2026/02/06/anthropic-claude-ai-model-cybersecurity-security-vulnerabilities-risks/)
- [GPT-5.3-Codex cybersecurity risks -- Fortune](https://fortune.com/2026/02/05/openai-gpt-5-3-codex-warns-unprecedented-cybersecurity-risks/)
- [Opus 4.6 coding-writing tradeoff -- Winbuzzer](https://winbuzzer.com/2026/02/05/claude-opus-4-6-coding-writing-tradeoff-xcxwbn/)
- [GPT-5.1-Codex-Max 24hr task -- VentureBeat](https://venturebeat.com/ai/openai-debuts-gpt-5-1-codex-max-coding-model-and-it-already-completed-a-24)
- [AI coding everywhere -- MIT Technology Review](https://www.technologyreview.com/2025/12/15/1128352/rise-of-ai-coding-developers-2026/)
- [Generative coding: 10 Breakthrough Technologies 2026 -- MIT Technology Review](https://www.technologyreview.com/2026/01/12/1130027/generative-coding-ai-software-2026-breakthrough-technology/)
- [Opus 4.6 GitHub Copilot -- GitHub Changelog](https://github.blog/changelog/2026-02-05-claude-opus-4-6-is-now-generally-available-for-github-copilot/)
- [Opus 4.6 on Azure -- Azure Blog](https://azure.microsoft.com/en-us/blog/claude-opus-4-6-anthropics-powerful-model-for-coding-agents-and-enterprise-workflows-is-now-available-in-microsoft-foundry-on-azure/)
- [Anthropic $30B Series G -- TechCrunch](https://techcrunch.com/2026/02/12/anthropic-raises-another-30-billion-in-series-g-with-a-new-value-of-380-billion/)
- [Anthropic $30B Series G -- Bloomberg](https://www.bloomberg.com/news/articles/2026-02-12/anthropic-finalizes-30-billion-funding-at-380-billion-value)
- [Claude Code revenue doubled -- Constellation Research](https://www.constellationr.com/insights/news/anthropics-claude-code-revenue-doubled-jan-1)
- [Devin 2.0 price cut -- VentureBeat](https://venturebeat.com/programming-development/devin-2-0-is-here-cognition-slashes-price-of-ai-software-engineer-to-20-per-month-from-500)
- [Devin 2.0 -- Cognition](https://cognition.ai/blog/devin-2)
- [Cursor Composer model -- VentureBeat](https://venturebeat.com/ai/vibe-coding-platform-cursor-releases-first-in-house-llm-composer-promising)
- [AWS Kiro launch -- VentureBeat](https://venturebeat.com/programming-development/amazon-launches-kiro-its-own-claude-powered-challenger-to-windsurf-and-codex)
- [Copilot Coding Agent -- GitHub Newsroom](https://github.com/newsroom/press-releases/coding-agent-for-github-copilot)

### Developer Productivity Studies

- [METR: AI Impact on Experienced OS Devs](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)
- [METR Paper -- arXiv](https://arxiv.org/abs/2507.09089)
- [Stack Overflow 2025 AI Survey](https://survey.stackoverflow.co/2025/ai)
- [JetBrains State of Developer Ecosystem 2025](https://blog.jetbrains.com/research/2025/10/state-of-developer-ecosystem-2025/)
- [AI Coding Impact 2026 -- Opsera](https://opsera.ai/resources/report/ai-coding-impact-2025-benchmark-report/)
- [AI Pair Programming Statistics -- Index.dev](https://www.index.dev/blog/ai-pair-programming-statistics)

### Multi-Agent & Agentic Coding

- [Claude Agent Teams Guide -- ClaudeFast](https://claudefa.st/blog/guide/agents/agent-teams)
- [Agent Teams Setup -- marc0.dev](https://www.marc0.dev/en/blog/claude-code-agent-teams-multiple-ai-agents-working-in-parallel-setup-guide-1770317684454)
- [Claude Code Hidden Swarm -- paddo.dev](https://paddo.dev/blog/claude-code-hidden-swarm/)
- [Multi-Agent Orchestration -- Shipyard](https://shipyard.build/blog/claude-code-multi-agent/)
- [Codex Desktop Parallel Agents -- VentureBeat](https://venturebeat.com/orchestration/openai-launches-a-codex-desktop-app-for-macos-to-run-multiple-ai-coding)
- [Codex Desktop -- DevOps.com](https://devops.com/openai-shifts-toward-autonomous-team-model-with-codex-desktop-launch/)
- [Terminal-Bench Paper -- arXiv](https://arxiv.org/html/2601.11868v1)
- [Terminal-Bench -- GitHub](https://github.com/laude-institute/terminal-bench)

### Open Source Models

- [Qwen3-Coder -- Qwen Blog](https://qwenlm.github.io/blog/qwen3-coder/)
- [Qwen3-Coder-Next -- Dev Genius](https://blog.devgenius.io/qwen3-coder-next-just-launched-open-source-is-winning-0724b76f13cc)
- [Qwen3-Coder-Next Sparse MoE -- Winbuzzer](https://winbuzzer.com/2026/02/04/alibaba-qwen3-coder-next-open-source-sparse-moe-coding-model-xcxwbn/)
- [Qwen3-Coder-Next -- VentureBeat](https://venturebeat.com/technology/qwen3-coder-next-offers-vibe-coders-a-powerful-open-source-ultra-sparse)

### Claude Code Coverage

- [Claude Code Product Page](https://claude.com/product/claude-code)
- [Creator of Claude Code Workflow -- VentureBeat](https://venturebeat.com/technology/the-creator-of-claude-code-just-revealed-his-workflow-and-developers-are)
- [Karpathy's Claude Code Field Notes -- DEV](https://dev.to/jasonguo/karpathys-claude-code-field-notes-real-experience-and-deep-reflections-on-the-ai-programming-era-4e2f)
- [Claude Code Has Engineers Buzzing -- GeekWire](https://www.geekwire.com/2026/a-new-era-of-software-development-claude-code-has-seattle-engineers-buzzing-as-ai-coding-hits-new-phase/)
- [Claude Code CLI Cheatsheet -- Shipyard](https://shipyard.build/blog/claude-code-cheat-sheet/)
- [Claude Code's "ChatGPT" moment -- Uncover Alpha](https://www.uncoveralpha.com/p/anthropics-claude-code-is-having)
- [Claude Code creator predicts -- AOL](https://www.aol.com/articles/anthropics-claude-code-creator-predicts-100101646.html)

### Pricing Guides

- [OpenAI Pricing 2026 -- Finout](https://www.finout.io/blog/openai-pricing-in-2026)
- [Anthropic API Pricing 2026 -- nOps](https://www.nops.io/blog/anthropic-api-pricing/)
- [LLM API Pricing Comparison -- CloudIDR](https://www.cloudidr.com/llm-pricing)
- [Claude Code Pricing -- o-mega](https://o-mega.ai/articles/claude-code-pricing-2026-costs-plans-and-alternatives)

### IDE & Tool Coverage

- [Windsurf Changelog](https://windsurf.com/changelog)
- [Windsurf Review 2026 -- Second Talent](https://www.secondtalent.com/resources/windsurf-review/)
- [AWS Kiro -- The New Stack](https://thenewstack.io/kiro-is-awss-specs-centric-answer-to-windsurf-and-cursor/)
- [Gemini Code Assist Updates -- Google Blog](https://developers.googleblog.com/new-in-gemini-code-assist/)
- [Cursor Changelog 2026 -- PromptLayer](https://blog.promptlayer.com/cursor-changelog-whats-coming-next-in-2026/)
- [GitHub Copilot What's New](https://github.com/features/copilot/whats-new)
- [GitHub Copilot Agents](https://github.com/features/copilot/agents)

### Community & Discussion

- [Claude Code vs Codex Sentiment -- Hacker News](https://news.ycombinator.com/item?id=45610266)
- [Claude vs Cursor vs Copilot -- Cursor Forum](https://forum.cursor.com/t/comparison-claude-vs-cursor-vs-copilot-review-from-a-regular-coder/130701)
- [METR Study Discussion -- Sean Goedecke](https://www.seangoedecke.com/impact-of-ai-study/)
- [OpenAI Codex App Analysis -- Latent Space](https://www.latent.space/p/ainews-openai-codex-app-death-of)

### Wikipedia

- [OpenAI o3](https://en.wikipedia.org/wiki/OpenAI_o3)
- [GPT-4.1](https://en.wikipedia.org/wiki/GPT-4.1)
- [Claude (Language Model)](https://en.wikipedia.org/wiki/Claude_(language_model))
