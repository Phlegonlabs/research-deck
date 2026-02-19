# Progress

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
