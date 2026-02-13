# Claude Code Best Practices

A deep-dive into production workflows, configuration patterns, and advanced techniques for maximizing productivity with Claude Code CLI.

## What They Built

Claude Code is Anthropic's official CLI tool for AI-assisted software development. The community has developed extensive best practices around:

- **CLAUDE.md configuration** - Project-specific AI context that persists across sessions
- **Terminal integration** - Ghostty + tmux workflows for parallel development
- **Permission management** - Hooks, sandboxing, and security guardrails
- **Context optimization** - Token management and conversation freshness strategies
- **Multi-agent orchestration** - Parallel Claude Code instances for complex tasks

## Why This Approach

### The Context Window Problem

LLM performance degrades as context fills. A single debugging session can consume tens of thousands of tokens. The community response:

| Strategy | Why It Works |
|----------|--------------|
| `/clear` frequently | Prevents stale context from degrading output quality |
| `/compact` before major work | Condenses history, frees ~70% token space |
| Fresh conversations per topic | Maintains peak model performance |
| Handoff documents | Preserves progress across session boundaries |

### CLAUDE.md Over Prompt Repetition

Instead of repeating instructions every session, `CLAUDE.md` provides:
- **Persistence** - Loads automatically at conversation start
- **Version control** - Team-shared project knowledge
- **Token efficiency** - No repeated context injection

But there's a limit: frontier LLMs follow ~150-200 instructions consistently. Over-stuffing CLAUDE.md degrades compliance.

### Terminal-First Over IDE Extension

| Approach | Tradeoff |
|----------|----------|
| **VS Code Extension** | Higher memory (8GB+), update lag, limited parallel instances |
| **Ghostty Terminal** | <500MB per instance, instant updates, unlimited parallel sessions |

Ghostty specifically chosen for GPU-accelerated rendering that prevents lag during long AI sessions.

## How It Works

### Three-Tier Configuration Hierarchy

```
~/.claude/settings.json          # User-wide defaults
.claude/settings.json            # Project (version controlled)
.claude/settings.local.json      # Personal (gitignored)
```

Settings merge with later files overriding earlier ones.

### CLAUDE.md Structure (What/Why/How)

```markdown
# WHAT - Give Claude a map
- Tech stack: Next.js 14, TypeScript, Prisma, PostgreSQL
- Structure: /src/app (routes), /src/lib (utilities), /src/components (UI)

# WHY - Purpose and context
- This is a SaaS billing dashboard
- We prioritize type safety over development speed

# HOW - Working instructions
- Use `bun` not `npm`
- Run `bun test` before committing
- Never modify /src/generated/* files directly
```

### Hooks Architecture

Hooks intercept tool calls for automated guardrails:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "check-dangerous-commands.sh"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "command": "audit-log.sh"
      }
    ]
  }
}
```

**Blocking hooks** (Trail of Bits pattern):
- Prevent `rm -rf` (suggest `trash` instead)
- Block direct pushes to main (require feature branches)
- Anti-rationalization gates forcing completion of incomplete work

### tmux Integration

tmux enables persistent, parallel Claude Code sessions:

```bash
# Create named session
tmux new-session -s claude-main

# Split into multiple panes
tmux split-window -h
tmux split-window -v

# Run Claude in each pane
claude  # pane 1: main feature
claude  # pane 2: tests
claude  # pane 3: docs
```

**Key benefits:**
- Session persistence (survives disconnection)
- Parallel task execution
- Notification integration (click notification → jump to pane)
- Background autonomous tasks

### Ghostty Configuration

```ini
# ghostty config (~/.config/ghostty/config)
theme = catppuccin-mocha
font-family = JetBrains Mono
font-size = 14
window-padding = 10

# Performance for long AI sessions
scrollback-limit = 100000
term = xterm-256color
```

**Native advantages:**
- Shift+Enter works without `/terminal-setup`
- GPU rendering prevents memory bloat
- Split-pane session forking

## Tradeoffs

### Fresh Context vs Continuity

| Choice | Gain | Lose |
|--------|------|------|
| `/clear` often | Peak performance, lower token cost | Project context, conversation history |
| Long sessions | Continuity, accumulated context | Degraded output quality, high token usage |

**Mitigation:** Write handoff documents before `/clear`:
```
/compact
"Write a handoff document summarizing current progress and next steps"
# Save output, then /clear
```

### Permission Strictness vs Flow

| Mode | Risk | Friction |
|------|------|----------|
| `--dangerously-skip-permissions` | Full system access | Zero |
| Allowlist specific commands | Medium (known commands) | Low |
| Default (ask every time) | Minimal | High |

**Recommended:** Use hooks for must-never-happen rules, `/permissions` allowlist for common safe operations.

### Opus vs Sonnet vs Haiku

| Model | Best For | Token Cost |
|-------|----------|------------|
| **Opus 4.5** | Complex multi-step planning, architecture | Highest |
| **Sonnet 4.5** | Default coding, routine tasks | Medium |
| **Haiku 4.5** | Simple queries, quick lookups | Lowest |

System auto-switches from Opus to Sonnet at 50% usage by default.

## Alternatives Considered

### IDE Extensions vs CLI

| Tool | Strength | Weakness |
|------|----------|----------|
| **Cursor** | Visual diff, inline suggestions | Proprietary, memory-heavy |
| **Cline (VS Code)** | IDE integration | Plugin limitations, slower updates |
| **Claude Code CLI** | Feature-first updates, composable | Learning curve, no visual diff |

CLI wins for power users who value composability and latest features.

### Local Models vs API

Trail of Bits runs Qwen3-Coder-Next (80B MoE) locally via LM Studio:
- **Advantage:** Privacy, no rate limits, offline capability
- **Disadvantage:** Hardware requirements, quality gap vs Opus

## Steal These Patterns

### 1. Status Line Customization

```bash
# Show: model | directory | branch | uncommitted | token%
# Install via: claude /install-status-line
```

Real-time visibility into context consumption prevents surprise token exhaustion.

### 2. Git Worktrees for Parallel Features

```bash
git worktree add ../feature-auth feature/auth
git worktree add ../feature-billing feature/billing

# Run separate Claude instances
cd ../feature-auth && claude
cd ../feature-billing && claude
```

No branch switching, independent file systems, true parallelism.

### 3. Gemini CLI Fallback for Blocked Sites

When WebFetch fails (Reddit, paywalled sites):

```markdown
# .claude/commands/fetch-blocked.md
Use Gemini CLI to fetch content from: $URL
Return the full text content.
```

### 4. Write-Test Cycle for Autonomous Tasks

```markdown
# CLAUDE.md
When implementing features:
1. Write failing test first
2. Implement until test passes
3. Run `bun test` to verify
4. Only commit when tests pass
```

Tests become the verification mechanism enabling autonomous iteration.

### 5. Three-Layer Sandboxing (Trail of Bits)

```
Layer 1: /sandbox command (OS-level isolation)
Layer 2: Permission deny-rules in settings.json
Layer 3: Devcontainers for complete isolation
```

```json
{
  "deny": [
    "Read(.env*)",
    "Read(~/.ssh/*)",
    "Read(~/.aws/*)",
    "Bash(rm -rf *)"
  ]
}
```

### 6. Essential Keyboard Shortcuts

| Action | Keys |
|--------|------|
| New line (not submit) | Shift+Enter |
| Stop generation | Escape |
| Access previous messages | Escape × 2 |
| Toggle extended thinking | Alt+T |
| Paste image | Ctrl+V (not Cmd+V) |

### 7. Session Management

```bash
# Aliases for quick access
alias c='claude'
alias ch='claude --chrome'
alias cr='claude -c'  # continue recent

# Fork conversation
/fork  # Creates branch of current session
```

### 8. MCP Server Essentials

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@context7/mcp-server"]
    },
    "exa": {
      "command": "npx",
      "args": ["-y", "@anthropic/exa-mcp-server"]
    }
  }
}
```

- **Context7:** On-demand documentation for any library
- **Exa:** Enhanced web/code search

### 9. Voice Input Workflow

Use local transcription (SuperWhisper, MacWhisper):
- Faster than typing for complex instructions
- Speak naturally, paste into Claude
- Especially powerful for architectural discussions

### 10. Audit Approved Commands Regularly

```bash
# Check what's been auto-approved
cat ~/.claude/settings.json | jq '.allowedTools'

# Remove dangerous patterns accumulated over time
claude /allowed-tools
```

Permissions accumulate; periodic review prevents security drift.

## Key Insight

The non-obvious insight: **Context freshness beats context accumulation**.

Intuition says "more context = better understanding." Reality: LLM output quality degrades with context length. The best practitioners `/clear` aggressively and write handoff documents, treating each session as a fresh start with curated context rather than a continuous conversation.

This inverts the mental model from "AI assistant remembers everything" to "AI assistant starts fresh with perfect briefing documents."
