# Claude Code Agent Team: Multi-Agent Orchestration from the Inside

## What They Built

Claude Code Agent Team is Anthropic's native multi-agent orchestration system, shipped with Opus 4.6 in February 2026. It transforms Claude Code from a single AI coding assistant into a **team coordinator** — one lead session spawns multiple teammate instances that work in parallel, communicate via direct messaging, and self-coordinate through a shared task list.

This isn't a wrapper or framework. It's built directly into the Claude Code binary as `TeammateTool` — 13 operations with defined schemas, directory structures, and environment variables. Initially discovered by researchers running `strings` on the binary (feature-flagged off), it was officially released as an experimental feature.

## Why This Architecture

### The Core Problem: Context Window Degradation

LLMs perform worse as context expands. Adding unrelated information (frontend code while debugging backend) degrades reasoning quality. The traditional single-agent approach hits a wall on complex projects — you can't fit a full-stack feature's context into one window without quality loss.

### Why Not Just Subagents?

Claude Code already had subagents (via the `Task` tool) — short-lived workers that execute and return results. But subagents have a critical limitation: **they can only report back to the caller**. No peer-to-peer communication. No shared state. No ability to challenge each other's findings.

Agent Teams solve this by making teammates **first-class peers**:

| Aspect | Subagents | Agent Teams |
|--------|-----------|-------------|
| Context | Own window, results return to caller | Own window, fully independent |
| Communication | Report back to main agent only | Teammates message each other directly |
| Coordination | Main agent manages all work | Shared task list with self-coordination |
| Persistence | Short-lived, dies after task | Persistent until shutdown |
| Cost | Lower (results summarized) | Higher (each is a separate Claude instance) |

### Why Not External Frameworks?

Compared to LangGraph, CrewAI, or OpenAI Swarm:

- **Zero setup friction**: one environment variable, no pip install, no config files
- **Native tool access**: teammates inherit all Claude Code capabilities (Bash, Read, Write, Edit, Glob, Grep, MCP servers, skills)
- **Project context inheritance**: teammates auto-load CLAUDE.md, MCP servers, and skills
- **File-based coordination**: no external databases, message queues, or APIs — just JSON files on disk

The tradeoff: you're locked into Claude as the LLM. No model mixing, no open-source alternatives.

## How It Works: Full Architecture

### Component Map

```
~/.claude/
├── teams/{team-name}/
│   ├── config.json          # Team members, IDs, types
│   └── inboxes/
│       ├── team-lead.json   # Leader's message inbox
│       ├── worker-1.json    # Teammate inboxes
│       └── worker-2.json
└── tasks/{team-name}/
    ├── 1.json               # Task with status, owner, dependencies
    ├── 2.json
    └── 3.json
```

### Core Components

| Component | Role |
|-----------|------|
| **Team Lead** | Main Claude Code session. Creates team, spawns teammates, coordinates work |
| **Teammates** | Separate Claude Code instances. Each has own context window, full tool access |
| **Task List** | Shared work items with status tracking, ownership, and dependency management |
| **Mailbox** | File-based messaging system for inter-agent communication |

### TeammateTool: 13 Operations

**Team Lifecycle:**
- `spawnTeam` / `discoverTeams` / `cleanup`
- `requestJoin` / `approveJoin` / `rejectJoin`

**Communication:**
- `write` — direct message to one teammate
- `broadcast` — message all teammates (expensive: N messages for N teammates)

**Plan Control:**
- `approvePlan` / `rejectPlan`

**Shutdown:**
- `requestShutdown` / `approveShutdown` / `rejectShutdown`

### Message Types

| Type | Purpose |
|------|---------|
| `message` | Direct text between agents |
| `broadcast` | Same message to all teammates |
| `shutdown_request` / `shutdown_response` | Graceful lifecycle management |
| `idle_notification` | Auto-sent when teammate stops (normal, not an error) |
| `task_completed` | Task completion signal |
| `plan_approval_request` | Teammate sends plan for leader review |
| `join_request` | Agent requests to join team |

### Task Dependency System

```
TaskCreate → TaskUpdate(addBlockedBy) → Auto-unblock on completion
```

Tasks use file-based locking to prevent race conditions when multiple teammates try to claim simultaneously. When a blocking task completes, all dependent tasks auto-unblock.

### Environment Variables (Auto-set for Teammates)

```
CLAUDE_CODE_TEAM_NAME="my-project"
CLAUDE_CODE_AGENT_ID="worker-1@my-project"
CLAUDE_CODE_AGENT_NAME="worker-1"
CLAUDE_CODE_AGENT_TYPE="Explore"
CLAUDE_CODE_PLAN_MODE_REQUIRED="false"
CLAUDE_CODE_PARENT_SESSION_ID="session-xyz"
```

## Orchestration Patterns

### Pattern 1: Parallel Specialists

Multiple agents analyze the same target from different angles simultaneously.

```
Create an agent team to review PR #142:
- Security reviewer: token handling, input validation
- Performance reviewer: query optimization, memory usage
- Test reviewer: coverage gaps, edge cases
```

**Why it works**: A single reviewer anchors on one issue type. Specialists apply orthogonal filters without overlap.

### Pattern 2: Competing Hypotheses

Multiple agents investigate different theories and actively debate.

```
Users report app exits after one message. Spawn 5 investigators:
- Each tests a different hypothesis
- They talk to each other to disprove competing theories
- The theory that survives debate is most likely correct
```

**Why it works**: Sequential investigation suffers from anchoring bias. Parallel + adversarial structure eliminates it.

### Pattern 3: Pipeline (Sequential with Dependencies)

```
Task 1: Research caching patterns →
Task 2: Design API (blocked by 1) →
Task 3: Implement (blocked by 2) →
Task 4: Write tests (blocked by 3)
```

Tasks auto-progress as dependencies complete. Good for work that must be sequential but benefits from clear handoff points.

### Pattern 4: Self-Organizing Swarm

Create a pool of independent tasks. Spawn workers that:
1. Check TaskList for pending tasks
2. Claim next available (file-locked to prevent races)
3. Complete work
4. Repeat until empty

Workers naturally load-balance. No central assignment needed.

### Pattern 5: Plan-Approve-Execute

```
Spawn architect teammate with mode: "plan"
→ Architect designs in read-only mode
→ Leader receives plan_approval_request
→ Leader approves/rejects with feedback
→ On approval, architect exits plan mode and implements
```

**Control knob**: "Only approve plans that include test coverage" or "reject plans that modify the database schema."

### Pattern 6: Cross-Layer Coordination

```
Task 1: Frontend agent → React components
Task 2: Backend agent → API endpoints
Task 3: Test agent → Integration tests (blocked by 1 + 2)
```

Each agent owns different files. Test agent only runs after both layers complete.

## Display & Interaction Modes

| Mode | How It Works | Best For |
|------|-------------|----------|
| **In-process** | All teammates in main terminal. Shift+Up/Down to navigate | Default, any terminal |
| **Split panes** | Each teammate gets own tmux/iTerm2 pane | Visibility, direct interaction |
| **Delegate** | Lead restricted to coordination-only tools (Shift+Tab) | Pure orchestration |

## Quality Gates: Hooks

Two hooks enforce rules on teammates:

- **`TeammateIdle`**: Runs when teammate is about to go idle. Exit code 2 sends feedback and keeps them working.
- **`TaskCompleted`**: Runs when task is being marked complete. Exit code 2 prevents completion with feedback.

## Tradeoffs

### What You Gain
- 5-10x throughput on parallelizable tasks
- Better reasoning through domain specialization
- Independent quality checkpoints per agent
- Graceful degradation (one agent failing doesn't kill the team)
- Natural phase boundaries

### What You Sacrifice
- **Token cost**: scales linearly with teammate count. Each teammate is a full Claude instance
- **Coordination overhead**: not worth it for sequential tasks or same-file edits
- **No session resumption**: in-process teammates don't survive `/resume` or `/rewind`
- **Single team per session**: can't nest teams or have teammates spawn sub-teams
- **Fixed leadership**: can't promote a teammate to lead
- **Platform limits**: split panes require tmux/iTerm2 (no VS Code terminal, no Windows Terminal, no Ghostty)
- **Task status lag**: teammates sometimes fail to mark tasks complete, blocking dependents

### When NOT to Use

- Sequential work with dependencies between every step
- Same-file edits (will overwrite each other)
- Simple tasks where coordination overhead > benefit
- Cost-sensitive scenarios (routine tasks better handled by single session)

## Alternatives Compared

| Framework | Orchestration | Communication | State | Setup |
|-----------|--------------|---------------|-------|-------|
| **Claude Code Agent Team** | Built-in, file-based | Direct messaging + mailbox | JSON on disk | 1 env var |
| **LangGraph** | Graph-based DAG | Through graph edges | Checkpointed state | pip install + code |
| **CrewAI** | Role-based crews | Through crew framework | In-memory | pip install + config |
| **OpenAI Swarm** | Routine-based | Function docstrings | No formal state | pip install + code |
| **claude-flow** | External orchestrator | MCP protocol | Redis/SQLite | npm install + config |

**LangGraph** wins on control, compliance, and production state management. **CrewAI** wins on speed-to-prototype. **Claude Code Agent Team** wins on zero-friction integration with an existing Claude Code workflow.

## Latest Updates (2026)

### Official Launch with Opus 4.6 (February 2026)

Agent Teams shipped as an experimental feature alongside Claude Opus 4.6 on February 5, 2026. Enable it with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json or environment variables. The feature was feature-flagged off in earlier Claude Code binaries (discovered by community researchers running `strings` on the binary in January 2026) before being officially launched. Anthropic recognized the community had already been building similar patterns independently using custom Docker orchestration scripts and tools like OpenClaw.

### The C Compiler Stress Test: 16 Agents, $20K, 100K Lines

Anthropic researcher Nicholas Carlini published a landmark case study: [16 parallel Opus 4.6 agents](https://www.anthropic.com/engineering/building-c-compiler) autonomously built a Rust-based C compiler capable of compiling Linux 6.9 on x86, ARM, and RISC-V. Key metrics: nearly 2,000 Claude Code sessions, 2 billion input tokens, 140 million output tokens, $20,000 in API cost, and 100,000 lines of Rust output over ~2 weeks. Each agent ran in its own Docker container, claiming tasks via lock files in `current_tasks/`, pushing to a shared git upstream. The compiler passes 99% of GCC torture tests and can build QEMU, FFmpeg, SQLite, PostgreSQL, and Redis. Notable limitation: scaling bottlenecked when all agents hit identical bugs simultaneously, causing redundant fixes and merge conflicts — the solution was to use GCC as a "known-good oracle" to partition work differently.

### Claude Code SDK Renamed to Claude Agent SDK

The Claude Code SDK was officially renamed to the **Claude Agent SDK** to reflect its broader scope beyond coding. Available in both Python (`pip install claude-agent-sdk`) and TypeScript (`npm install @anthropic-ai/claude-agent-sdk`), it provides the same tools, agent loop, and context management that power Claude Code as a programmable library. Key additions in early 2026 include `ThinkingConfig` for fine-grained control over extended thinking (effort levels: low/medium/high/max), MCP tool annotations via the `@tool` decorator, and fixes for large agent definitions that previously failed silently due to CLI argument size limits. The SDK supports subagent spawning via the `Task` tool, custom hooks (`PreToolUse`, `PostToolUse`, `Stop`, etc.), and session resumption for multi-turn workflows.

### VS Code Becomes Multi-Agent Command Center

With [VS Code v1.109](https://code.visualstudio.com/blogs/2026/02/05/multi-agent-development) (January 2026), Microsoft made VS Code "the home for multi-agent development." Claude agents now run natively inside VS Code via the official Claude Agent harness — same prompts, tools, and architecture as standalone Claude Code. Developers can run Claude, Codex, and GitHub Copilot agents simultaneously from one interface, choosing between local (interactive), background (async), or cloud (remote autonomous) deployment. February updates added subagent rendering — when Claude spawns subagents during streaming, you can see their tool calls and progress inline. Agent Skills (Anthropic's open standard) are now generally available in VS Code.

### Community Tooling Explosion

The agent team pattern spawned an ecosystem of open-source coordination tools:
- **[claude-swarm](https://blog.nikosbaxevanis.com/2026/02/08/agent-teams-and-claude-swarm/)**: Reusable harness for running multiple Claude Code sessions in Docker containers coordinating through git, using bare repos mounted as read-only volumes.
- **[ccswarm](https://github.com/nwiizo/ccswarm)**: Workflow automation framework with task delegation infrastructure, template scaffolding, and git worktree isolation.
- **[worktree-cli](https://github.com/agenttools/worktree)**: CLI for managing git worktrees with GitHub Issues integration and Claude Code hooks.
- **[Clash](https://github.com/clash-sh/clash)**: Merge conflict manager across git worktrees for parallel AI agent workflows.

The unsolved gap: no tool yet cleanly combines worktree code isolation with full environment isolation (Dev Containers). Multiple GitHub issues show that worktrees and devcontainers don't work well together — the `.git` file format breaks container git operations.

### Enterprise Adoption Surge

Business subscriptions to Claude Code have quadrupled since the start of 2026, with enterprise use now representing over half of all Claude Code revenue. The agent teams feature is cited as a primary driver — teams report 5-10x throughput gains on parallelizable refactoring, migration, and review tasks.

### Microsoft Agent Framework Integration

Microsoft's Semantic Kernel team announced [integration between Microsoft Agent Framework and Claude Agent SDK](https://devblogs.microsoft.com/semantic-kernel/build-ai-agents-with-claude-agent-sdk-and-microsoft-agent-framework/), combining the Agent Framework's consistent agent abstraction with Claude's file editing, code execution, function calling, streaming, multi-turn conversations, and MCP server integration. This opens Claude-powered agents to enterprise .NET and Java ecosystems.

## Patterns to Steal

### 1. File-Based Coordination Over Databases

Teams use JSON files on disk for config, tasks, and messages. No external dependencies. Works offline. Git-trackable. This is elegant for local-first agent systems.

### 2. Idle ≠ Dead

Teammates go idle after every turn — this is normal, not an error. Sending a message wakes them. This "event-driven sleep" pattern prevents busy-waiting and token waste.

### 3. Task Dependency Auto-Unblocking

Instead of manual pipeline orchestration, declare dependencies upfront and let the system handle progression. Blocked tasks auto-unblock when dependencies complete.

### 4. Plan-Approve Gate

Force agents to plan in read-only mode before implementing. Leader reviews and approves. This catches bad approaches before they waste tokens on implementation.

### 5. Adversarial Debugging

Spawn multiple agents with competing hypotheses. Make them actively try to disprove each other. The surviving theory is most likely correct. This is the most underrated pattern in the system.

### 6. File Ownership Boundaries

Assign different files to each teammate. No shared-file editing. This eliminates merge conflicts entirely — the same principle behind microservice team boundaries.

### 7. Hook-Based Quality Gates

`TeammateIdle` and `TaskCompleted` hooks let you inject validation without modifying agent prompts. Exit code 2 = "not done yet, keep working." This is a clean separation of concerns.

### 8. Context Inheritance via CLAUDE.md

All teammates auto-load CLAUDE.md. Put team-wide conventions, constraints, and quality standards there. One file controls the behavior of N agents.

## References

### Original Source
- [@YukerX Twitter Thread — Claude Code Agent Team 入門教程](https://x.com/YukerX/status/2019977867061522525)

### Official Documentation
- [Orchestrate teams of Claude Code sessions — Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Introducing Claude Opus 4.6 — Anthropic](https://www.anthropic.com/news/claude-opus-4-6)
- [Common workflows — Claude Code Docs](https://code.claude.com/docs/en/common-workflows)
- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Agent SDK overview — Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Claude Agent SDK Python — GitHub](https://github.com/anthropics/claude-agent-sdk-python)
- [Claude Agent SDK Demos — GitHub](https://github.com/anthropics/claude-agent-sdk-demos)

### Deep Technical Analysis
- [Claude Code's Hidden Multi-Agent System — paddo.dev](https://paddo.dev/blog/claude-code-hidden-swarm/)
- [Claude Code Swarm Orchestration Skill — Complete guide (GitHub Gist)](https://gist.github.com/kieranklaassen/4f2aba89594a4aea4ad64d753984b2ea)
- [Claude Code Swarms — Addy Osmani](https://addyosmani.com/blog/claude-code-agent-teams/)
- [Building a C compiler with a team of parallel Claudes — Anthropic Engineering](https://www.anthropic.com/engineering/building-c-compiler)
- [Claude Code Agent Teams: How They Work Under the Hood — Claude Code Camp](https://www.claudecodecamp.com/p/claude-code-agent-teams-how-they-work-under-the-hood)

### Guides & Tutorials
- [Claude Code Agent Teams: Parallel AI Agents Setup Guide — Marco Patzelt](https://www.marc0.dev/en/blog/claude-code-agent-teams-multiple-ai-agents-working-in-parallel-setup-guide-1770317684454)
- [Claude Code Swarms Guide: How to Build Native Multi-Agent Teams — Tech On Play](https://techonplay.com/claude-code-swarms/)
- [Claude Swarm Mode Complete Guide — Apiyi.com](https://help.apiyi.com/en/claude-code-swarm-mode-multi-agent-guide-en.html)
- [Claude Code Agent Teams: Multi-Claude Orchestration — claudefa.st](https://claudefa.st/blog/guide/agents/agent-teams)
- [What Is the Claude Code Swarm Feature? — Cyrus](https://www.atcyrus.com/stories/what-is-claude-code-swarm-feature)
- [Claude Code Agent Teams: Run Parallel AI Agents on Your Codebase — SitePoint](https://www.sitepoint.com/anthropic-claude-code-agent-teams/)
- [How to Set Up and Use Claude Code Agent Teams — Dára Sobaloju (Medium)](https://darasoba.medium.com/how-to-set-up-and-use-claude-code-agent-teams-and-actually-get-great-results-9a34f8648f6d)

### Discovery & Exploration
- [I Discovered This Claude Code Agent Swarm Mode — Joe Njenga (Medium)](https://medium.com/@joe.njenga/i-discovered-this-claude-code-agent-swarm-mode-you-dont-know-exists-bf36e3898ad1)
- [I Tried (New) Claude Code Agent Teams (And Discovered New Way to Swarm) — Joe Njenga (Medium)](https://medium.com/@joe.njenga/i-tried-new-claude-code-agent-teams-and-discovered-new-way-to-swarm-28a6cd72adb8)
- [Anthropic releases Opus 4.6 with new 'agent teams' — TechCrunch](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams/)
- [Claude Code's new hidden feature: Swarms — Hacker News](https://news.ycombinator.com/item?id=46743908)

### Git Worktree & Parallel Development
- [Parallel Development with Claude Code and Git Worktrees — Yee Fei (Medium)](https://medium.com/@ooi_yee_fei/parallel-ai-development-with-git-worktrees-f2524afc3e33)
- [Mastering Git Worktrees with Claude Code — Dogukan Uraz Tuna (Medium)](https://medium.com/@dtunai/mastering-git-worktrees-with-claude-code-for-parallel-development-workflow-41dc91e645fe)
- [Clash — Manage merge conflicts across git worktrees for parallel AI agents](https://github.com/clash-sh/clash)
- [ccpm — Project management for Claude Code using GitHub Issues and Git worktrees](https://github.com/automazeio/ccpm)
- [ccswarm — Multi-agent orchestration with Git worktree isolation](https://github.com/nwiizo/ccswarm)
- [worktree — CLI tool for managing Git worktrees with Claude Code integration](https://github.com/agenttools/worktree)
- [Agent Teams and claude-swarm — Nikos Baxevanis](https://blog.nikosbaxevanis.com/2026/02/08/agent-teams-and-claude-swarm/)

### Alternative Frameworks (Comparison)
- [LangGraph vs CrewAI vs AutoGen: Top 10 AI Agent Frameworks](https://o-mega.ai/articles/langgraph-vs-crewai-vs-autogen-top-10-agent-frameworks-2026)
- [claude-flow — Agent orchestration platform for Claude](https://github.com/ruvnet/claude-flow)
- [Support for Claude Code Agent Teams (TeammateTool, SendMessage, TaskList) — superpowers Issue #429](https://github.com/obra/superpowers/issues/429)

### IDE & Platform Integration
- [Your Home for Multi-Agent Development — VS Code Blog](https://code.visualstudio.com/blogs/2026/02/05/multi-agent-development)
- [VS Code becomes multi-agent command center for developers — The New Stack](https://thenewstack.io/vs-code-becomes-multi-agent-command-center-for-developers/)
- [Build AI Agents with Claude Agent SDK and Microsoft Agent Framework — Semantic Kernel](https://devblogs.microsoft.com/semantic-kernel/build-ai-agents-with-claude-agent-sdk-and-microsoft-agent-framework/)
- [Agent Teams with Claude Code and Claude Agent SDK — Isaac Kargar (Medium)](https://kargarisaac.medium.com/agent-teams-with-claude-code-and-claude-agent-sdk-e7de4e0cb03e)

### Best Practices
- [A Guide to Claude Code 2.0 and getting better at using coding agents — sankalp](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/)
- [Best practices for Claude Code subagents — PubNub](https://www.pubnub.com/blog/best-practices-for-claude-code-sub-agents/)
- [How Claude Code Agents and MCPs Work Better Together — Yee Fei (Medium)](https://medium.com/@ooi_yee_fei/how-claude-code-agents-and-mcps-work-better-together-5c8d515fcbbd)
