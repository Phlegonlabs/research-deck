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
