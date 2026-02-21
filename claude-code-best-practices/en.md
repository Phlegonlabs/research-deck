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

## Latest Updates (2026)

### Opus 4.6 and Model Lineup Refresh

Anthropic released Claude Opus 4.6 on February 5, 2026, alongside Sonnet 4.6. Opus 4.6 features a 1M token context window (beta), 128K output token limit (doubled from 64K), and adaptive thinking that lets the model adjust extended thinking depth based on contextual clues. Agent teams (TeammateTool) launched officially with the Opus 4.6 release, enabled via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Sonnet 4.5 with 1M context has been retired; Sonnet 4.6 replaces it with the same 1M context window. These model upgrades mean the Opus vs Sonnet decision matrix in Claude Code has shifted -- Opus 4.6 handles larger codebases natively with the expanded context, while Sonnet 4.6 provides near-Opus coding quality at Sonnet pricing.

### Skills System Replaces Slash Commands

Custom slash commands (`.claude/commands/`) have been officially merged into the skills system (`.claude/skills/`). Existing command files still work, but skills are now the recommended path. Skills add key capabilities over plain commands: YAML frontmatter for controlling invocation behavior (`disable-model-invocation`, `user-invocable`), supporting file directories for templates and scripts, `allowed-tools` restrictions for sandboxed execution, `context: fork` for subagent isolation, and dynamic context injection via `!`command`` syntax that preprocesses shell output before sending to Claude. Skills also follow the [Agent Skills](https://agentskills.io) open standard for cross-tool portability. Monorepo support works automatically -- skills in nested `.claude/skills/` directories are discovered based on the file being edited.

### Built-in Git Worktree Support (`--worktree` Flag)

Claude Code now has a native `--worktree` (`-w`) flag that creates an isolated git worktree and launches a session inside it. This eliminates the manual `git worktree add` workflow previously required for parallel feature development. Subagents also support `"worktree"` isolation, enabling parallel work inside a single session where each subagent operates in its own worktree without file conflicts. Combined with `--tmux`, you can launch fully isolated background sessions: `claude --worktree my-feature --tmux`. New `WorktreeCreate` and `WorktreeRemove` hook events (v2.1.50) allow custom VCS setup automation when worktrees are created or destroyed.

### Agent Teams for Multi-Agent Orchestration

Agent teams let you coordinate multiple Claude Code instances where one session acts as team lead, delegating tasks and synthesizing results. Unlike subagents (which share the parent's context window), teammates each get their own independent context window and can communicate directly with each other. You can also interact with individual teammates directly, bypassing the lead. Best use cases: research with parallel investigation, new modules with separate ownership, debugging with competing hypotheses, and cross-layer coordination (frontend/backend/tests). The tradeoff is significant token overhead -- agent teams are best when teammates can operate independently on different files.

### IDE Integration Matured

The VS Code extension now provides a native graphical interface with plan preview, auto-accept edits, @-mention files with line ranges, conversation history tabs, and multiple parallel conversations. The JetBrains plugin (IntelliJ, PyCharm, WebStorm) runs the CLI inside the IDE terminal with changes surfaced in the IDE's diff viewer. VS Code integration is slightly more polished (faster context loading), but both platforms work reliably. For power users, the CLI remains the most flexible option with faster feature delivery.

### Hooks System Expansion

The hooks system has grown from the original `PreToolUse`/`PostToolUse` to 15 lifecycle events. New additions include `TeammateIdle` and `TaskCompleted` for multi-agent workflows, `ConfigChange` for security auditing when settings are modified, and `WorktreeCreate`/`WorktreeRemove` for worktree lifecycle management. A `last_assistant_message` field was added to `Stop` and `SubagentStop` hook inputs, providing the final response text without parsing transcripts. Startup performance improved by ~500ms by deferring `SessionStart` hook execution. The `disableAllHooks` setting now properly respects managed settings hierarchy, preventing non-managed configs from overriding organization policy hooks.

### Performance and Platform Improvements (v2.1.37-v2.1.50)

The February 2026 release cadence brought 60+ fixes in v2.1.47 alone. Key improvements: Windows ARM64 native binary support, `claude auth login/status/logout` CLI subcommands, automatic session name generation with `/rename`, memory leak fixes in agent teams and task state management, improved compaction behavior (caches cleared after compaction, file history snapshots capped), and `chat:newline` keybinding action for configurable multi-line input. Automatic memory (MEMORY.md) was introduced in v2.1.32 -- Claude now writes its own notes about project conventions and user preferences, separate from CLAUDE.md.

## Key Insight

The non-obvious insight: **Context freshness beats context accumulation**.

Intuition says "more context = better understanding." Reality: LLM output quality degrades with context length. The best practitioners `/clear` aggressively and write handoff documents, treating each session as a fresh start with curated context rather than a continuous conversation.

This inverts the mental model from "AI assistant remembers everything" to "AI assistant starts fresh with perfect briefing documents."

## References

### Official Documentation
- [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Common Workflows - Claude Code Docs](https://code.claude.com/docs/en/common-workflows)
- [Optimize Your Terminal Setup - Claude Code Docs](https://code.claude.com/docs/en/terminal-config)
- [Orchestrate Teams of Claude Code Sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Using CLAUDE.MD Files: Customizing Claude Code for Your Codebase | Claude](https://claude.com/blog/using-claude-md-files)
- [Extend Claude with Skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Hooks Reference - Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Claude Code Releases - GitHub](https://github.com/anthropics/claude-code/releases)
- [What's New in Claude 4.6 - Claude API Docs](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-6)
- [Introducing Claude Opus 4.6 - Anthropic](https://www.anthropic.com/news/claude-opus-4-6)

### Comprehensive Guides
- [How I Use Claude Code (+ My Best Tips) - Builder.io](https://www.builder.io/blog/claude-code)
- [32 Claude Code Tips: From Basics to Advanced - Agentic Coding Substack](https://agenticcoding.substack.com/p/32-claude-code-tips-from-basics-to)
- [Claude Code CLI Cheatsheet: Config, Commands, Prompts, + Best Practices - Shipyard](https://shipyard.build/blog/claude-code-cheat-sheet/)
- [Claude Code Complete Guide 2026: From Basics to Advanced MCP, Agents & Git Workflows](https://www.jitendrazaa.com/blog/ai/claude-code-complete-guide-2026-from-basics-to-advanced-mcp-2/)
- [Cooking with Claude Code: The Complete Guide | Sid Bharath](https://www.siddharthbharath.com/claude-code-the-complete-guide/)
- [50 Claude Code Tips & Tricks for Daily Coding in 2026 - Geeky Gadgets](https://www.geeky-gadgets.com/claude-code-tips-2/)
- [Claude Code Best Practices: 15 Tips from 6 Projects (2026) | aiorg.dev](https://aiorg.dev/blog/claude-code-best-practices)

### GitHub Resources
- [ykdojo/claude-code-tips: 45 Tips for Getting the Most Out of Claude Code](https://github.com/ykdojo/claude-code-tips)
- [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
- [hesreallyhim/awesome-claude-code: Curated List of Skills, Hooks, Slash-Commands](https://github.com/hesreallyhim/awesome-claude-code)
- [trailofbits/claude-code-config: Opinionated Defaults and Workflows](https://github.com/trailofbits/claude-code-config)
- [disler/claude-code-hooks-mastery: Master Claude Code Hooks](https://github.com/disler/claude-code-hooks-mastery)

### CLAUDE.md Configuration
- [Writing a Good CLAUDE.md | HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [Creating the Perfect CLAUDE.md for Claude Code - Dometrain](https://dometrain.com/blog/creating-the-perfect-claudemd-for-claude-code/)
- [ClaudeLog - Claude Code Docs, Guides, Tutorials & Best Practices](https://claudelog.com/configuration/)
- [How to Write a Good CLAUDE.md File - Builder.io](https://www.builder.io/blog/claude-md-guide)
- [Claude Skills and CLAUDE.md: A Practical 2026 Guide for Teams - Gend](https://www.gend.co/blog/claude-skills-claude-md-guide)

### Terminal Integration
- [Claude Code + tmux: The Ultimate Terminal Workflow for AI Development](https://www.blle.co/blog/claude-code-tmux-beautiful-terminal)
- [How to Use Claude Code CLI and Tmux for Continuous Workflows - Geeky Gadgets](https://www.geeky-gadgets.com/making-claude-code-work-247-using-tmux/)
- [Build a 24/7 Autonomous Coding Assistant with Tmux & Claude Code - Geeky Gadgets](https://www.geeky-gadgets.com/autonomous-coding-assistant-setup/)
- [How tmux Automation Made Claude Code Development Much More Efficient - Qiita](https://qiita.com/vibecoding/items/c04741332b6617781684)
- [Combining tmux and Claude to Build an Automated AI Agent System - Scuti Ai](https://scuti.asia/combining-tmux-and-claude-to-build-an-automated-ai-agent-system-for-mac-linux/)
- [Multi-agent Claude Code Workflow Using tmux for Parallel Sessions - GitHub Gist](https://gist.github.com/andynu/13e362f7a5e69a9f083e7bca9f83f60a)
- [Notification System for Tmux and Claude Code - Alexandre Quemy](https://quemy.info/2025-08-04-notification-system-tmux-claude.html)

### Ghostty Terminal
- [How to Integrate Claude Code with Neovim Using Ghostty Terminal Panes | Daniel Miessler](https://danielmiessler.com/blog/claude-code-neovim-ghostty-integration)
- [The State of Vibe Coding: Agentic Software Development with Ghostty, Git Worktree & Claude Code | Medium](https://medium.com/@takafumi.endo/the-state-of-vibe-coding-agentic-software-development-with-ghostty-git-worktree-claude-code-18f4d56b8e01)
- [Why Ghostty Terminal Is My Fastest Claude Code Workflow | Engr Mejba Ahmed](https://www.mejba.me/blog/ghostty-terminal-claude-code-workflow)
- [Ghostty Terminal Configuration | DeepWiki](https://deepwiki.com/awwsillylife/ghostty-claude-code-setup/4.1-ghostty-terminal-configuration)
- [Claude Code Session Fork for Ghostty - GitHub Gist](https://gist.github.com/yottahmd/8e6d0a4213be6a559dfe3dcdd350ce09)
- [Add Shift-Enter Support for Ghostty via `/terminal-setup` - GitHub Issue](https://github.com/anthropics/claude-code/issues/1282)

### Productivity & Workflows
- [Claude Code Tips: 10 Real Productivity Workflows for 2026 - F22 Labs](https://www.f22labs.com/blogs/10-claude-code-productivity-tips-for-every-developer/)
- [The Claude Code Playbook: 5 Tips Worth $1000s in Productivity | White Prompt Blog](https://blog.whiteprompt.com/the-claude-code-playbook-5-tips-worth-1000s-in-productivity-22489d67dd89)
- [My 7 Essential Claude Code Best Practices for Production-Ready AI in 2025 - Eesel](https://www.eesel.ai/blog/claude-code-best-practices)
- [The Ultimate Guide to Claude Code: Production Prompts, Power Tricks, and Workflow Recipes | Medium](https://medium.com/@tonimaxx/the-ultimate-guide-to-claude-code-production-prompts-power-tricks-and-workflow-recipes-42af90ca3b4a)
- [Two Simple Tricks That Will Dramatically Improve Your Productivity with Claude | Medium](https://julsimon.medium.com/two-simple-tricks-that-will-dramatically-improve-your-productivity-with-claude-db90ce784931)
- [Mastering Claude Code: Essential Tips for Maximum Productivity - Tembo](https://www.tembo.io/blog/mastering-claude-code-tips)
- [How I Use Claude Code to Maximize Productivity | Medium](https://medium.com/@shivang.tripathii/how-i-use-claude-code-to-maximize-productivity-c853104804d6)
- [How I Use Every Claude Code Feature - Shrivu Shankar](https://blog.sshh.io/p/how-i-use-every-claude-code-feature)

### Agent Teams & Multi-Agent
- [How to Set Up and Use Claude Code Agent Teams | Medium](https://darasoba.medium.com/how-to-set-up-and-use-claude-code-agent-teams-and-actually-get-great-results-9a34f8648f6d)
- [How Claude Code Agent Teams Changed Everything About AI Coding | How Do I Use AI](https://www.howdoiuseai.com/blog/2026-02-10-how-claude-code-agent-teams-changed-everything-abo)
- [Claude Code Agent Teams: The Complete Guide 2026 - ClaudeFast](https://claudefa.st/blog/guide/agents/agent-teams)
- [Claude Code's Hidden Multi-Agent System - Paddo.dev](https://paddo.dev/blog/claude-code-hidden-swarm/)
- [Claude Code Swarm Orchestration Skill - GitHub Gist](https://gist.github.com/kieranklaassen/4f2aba89594a4aea4ad64d753984b2ea)
- [Multi-agent Orchestration for Claude Code in 2026 - Shipyard](https://shipyard.build/blog/claude-code-multi-agent/)
- [Claude Code Swarms - Addy Osmani](https://addyosmani.com/blog/claude-code-agent-teams/)

### IDE Integration
- [Use Claude Code in VS Code - Claude Code Docs](https://code.claude.com/docs/en/vs-code)
- [Claude Code Plugin for JetBrains IDEs - JetBrains Marketplace](https://plugins.jetbrains.com/plugin/27310-claude-code-beta-)
- [Claude Code IDE Integrations for JetBrains and VS Code | Medium](https://medium.com/vibecodingpub/claude-code-ide-integrations-for-jetbrains-ides-and-vs-code-71023d27b86d)

### Worktree & Isolation
- [Claude Code Worktree: Complete Guide for Developers - SupaTest](https://supatest.ai/blog/claude-code-worktree-the-complete-developer-guide)
- [Claude Code Multiple Agent Systems: Complete 2026 Guide - Eesel](https://www.eesel.ai/blog/claude-code-multiple-agent-systems-complete-2026-guide)

### Model Updates
- [Anthropic Releases Opus 4.6 with New 'Agent Teams' | TechCrunch](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams/)
- [Claude Opus 4.6: What Actually Changed and Why It Matters | Medium](https://medium.com/data-science-collective/claude-opus-4-6-what-actually-changed-and-why-it-matters-1c81baeea0c9)
- [Claude Sonnet 4.6 Promises Opus-Level Coding at Sonnet Pricing - The New Stack](https://thenewstack.io/claude-sonnet-46-launch/)

### Tutorials
- [Claude Code Tutorial for Beginners - Complete 2026 Guide to AI Coding - codewithmukesh](https://codewithmukesh.com/blog/claude-code-for-beginners/)
- [Claude Code CLI Best Practices Checklist - Engineering Notes](https://notes.muthu.co/2026/02/claude-code-cli-best-practices-checklist/)
- [Mastering the Vibe: Claude Code Best Practices That Actually Work | Medium](https://dinanjana.medium.com/mastering-the-vibe-claude-code-best-practices-that-actually-work-823371daf64c)
- [Claude Code Hooks: Complete Guide with 20+ Ready-to-Use Examples (2026) | aiorg.dev](https://aiorg.dev/blog/claude-code-hooks)
