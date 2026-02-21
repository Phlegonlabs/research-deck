# Claude Code Memory System — Complete Architecture Deep Dive

## What They Built

Claude Code has a **multi-layered, hierarchical memory system** that persists context across sessions. Unlike typical LLM interactions that start fresh every time, Claude Code loads project-specific, user-specific, and organization-specific memories into its system prompt at session start. The system has two fundamental pillars: **CLAUDE.md files** (human-written instructions) and **Auto Memory** (Claude-written learnings).

As of February 2026 (v2.1.33+), a third pillar was added: **Agent Memory** — persistent memory scoped to individual subagents.

## Why This Architecture

LLMs are stateless. Every session starts from zero. The core problem: developers repeat the same instructions ("we use pnpm, not npm", "run tests with vitest", "follow this API pattern") across hundreds of sessions. The memory system exists to eliminate this repetition.

The hierarchical design solves a harder problem: **different scopes of knowledge have different audiences**. Organization security policies apply to everyone. Project conventions apply to team members. Personal preferences apply only to you. A flat memory file can't capture these distinctions.

The 200-line limit on auto memory exists because of context window economics — every token of memory loaded at startup is a token unavailable for actual work.

## Complete Memory Hierarchy

| Memory Type | Location | Purpose | Audience | Loaded At |
|---|---|---|---|---|
| **Managed Policy** | `C:\Program Files\ClaudeCode\CLAUDE.md` (Win) / `/etc/claude-code/CLAUDE.md` (Linux) / `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS) | Org-wide standards | All users | Always |
| **Project Memory** | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team project instructions | Team via git | Always |
| **Project Rules** | `./.claude/rules/*.md` | Modular, topic-specific rules | Team via git | Always (conditional with `paths` frontmatter) |
| **User Memory** | `~/.claude/CLAUDE.md` | Personal cross-project preferences | Just you | Always |
| **User Rules** | `~/.claude/rules/*.md` | Personal modular rules | Just you | Always |
| **Local Memory** | `./CLAUDE.local.md` | Private project-specific prefs | Just you (gitignored) | Always |
| **Auto Memory** | `~/.claude/projects/<project>/memory/` | Claude's learned patterns | Just you (per project) | First 200 lines of MEMORY.md |
| **Agent Memory** (v2.1.33+) | Varies by scope (see below) | Subagent-specific knowledge | Individual subagent | First 200 lines of agent MEMORY.md |

**Priority order**: More specific beats broader. Project rules > user rules. Local > project. Child directory CLAUDE.md files load on-demand when Claude reads files in those directories.

## Mechanism 1: CLAUDE.md Files

### How They Work

CLAUDE.md is a markdown file you write and maintain. It becomes part of Claude's system prompt — loaded in full at session start. Claude treats its contents as instructions.

**Lookup behavior**: Claude Code starts at the current working directory and walks up the directory tree to (but not including) the root `/`, reading every `CLAUDE.md` and `CLAUDE.local.md` it finds. In a monorepo at `foo/bar/`, both `foo/CLAUDE.md` and `foo/bar/CLAUDE.md` are loaded.

CLAUDE.md files in **child** directories are NOT loaded at startup. They load on-demand only when Claude reads files in those subdirectories.

### The `/init` Command

Running `/init` auto-generates a starter CLAUDE.md by analyzing your project structure, package files, existing docs, and code patterns. It's a starting point — manual refinement is essential.

### Import System (`@path` Syntax)

CLAUDE.md files can import other files:

```markdown
See @README for project overview and @package.json for available npm commands.

# Additional Instructions
- git workflow @docs/git-instructions.md
```

Key rules:
- **Relative paths** resolve relative to the containing file, not the working directory
- **Absolute paths** and `@~/...` home-directory paths are supported
- **Recursive imports** up to max-depth of **5 hops**
- Imports inside markdown code spans/blocks are **not** evaluated (prevents collision with npm package names like `@anthropic-ai/claude-code`)
- First-time external imports show an **approval dialog** — one-time decision per project
- For git worktrees, use `@~/.claude/my-project-instructions.md` so all worktrees share the same personal instructions

### CLAUDE.local.md

Automatically added to `.gitignore`. Use for:
- Personal sandbox URLs
- Preferred test data
- Private tooling shortcuts
- Anything you don't want committed

## Mechanism 2: Project Rules (`.claude/rules/`)

### Basic Structure

```
.claude/
  rules/
    code-style.md
    testing.md
    security.md
    frontend/
      react.md
      styles.md
    backend/
      api.md
      database.md
```

All `.md` files are discovered **recursively** through subdirectories. They load with the same priority as `.claude/CLAUDE.md`.

### Path-Specific Rules (Frontmatter)

Rules can be conditionally scoped using YAML frontmatter:

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "lib/**/*.ts"
---

# API Development Rules
- All endpoints must include input validation
- Use standard error response format
```

Rules **without** a `paths` field load unconditionally.

**Supported glob patterns:**

| Pattern | Matches |
|---|---|
| `**/*.ts` | All TypeScript files in any directory |
| `src/**/*` | All files under `src/` |
| `*.md` | Markdown files in project root only |
| `src/**/*.{ts,tsx}` | Brace expansion for multiple extensions |
| `{src,lib}/**/*.ts` | Brace expansion for multiple directories |

### Symlinks

`.claude/rules/` supports symlinks for sharing rules across projects:

```bash
ln -s ~/shared-claude-rules .claude/rules/shared
ln -s ~/company-standards/security.md .claude/rules/security.md
```

Circular symlinks are detected and handled gracefully.

## Mechanism 3: Auto Memory

### What It Is

Auto memory is a persistent directory where **Claude writes notes for itself** — not instructions you write for Claude. As Claude works, it discovers patterns and saves them automatically.

### What Gets Remembered

- **Project patterns**: build commands, test conventions, code style
- **Debugging insights**: solutions to tricky problems, common error causes
- **Architecture notes**: key files, module relationships, important abstractions
- **Your preferences**: communication style, workflow habits, tool choices

### Directory Structure

```
~/.claude/projects/<project>/memory/
  MEMORY.md          # Concise index (first 200 lines loaded every session)
  debugging.md       # Detailed debugging patterns
  api-conventions.md # API design decisions
  ...                # Any topic files Claude creates
```

The `<project>` path derives from the **git repository root** — all subdirectories within the same repo share one auto memory directory. Git worktrees get separate directories. Outside a git repo, the working directory is used.

### The 200-Line Rule

- **First 200 lines of `MEMORY.md`** are injected into the system prompt at session start
- Content beyond 200 lines is **never loaded automatically**
- Claude is instructed to keep `MEMORY.md` concise and move detailed notes into topic files
- **Topic files** (`debugging.md`, `patterns.md`, etc.) are **not loaded at startup** — Claude reads them on-demand using file tools when needed

This creates a two-tier system: MEMORY.md is an always-available index, topic files are on-demand deep storage.

### Controlling Auto Memory

```bash
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1  # Force off
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=0  # Force on (opt-in during gradual rollout)
```

Double-negative logic: `DISABLE=0` means "don't disable" = force on.

### Telling Claude What to Remember

You can explicitly request memory saves:
- "Remember that we use pnpm, not npm"
- "Save to memory that the API tests require a local Redis instance"

### The `/memory` Command

Opens a file selector showing your auto memory entrypoint alongside CLAUDE.md files. Lets you edit any memory file in your system editor.

### Known Issue: Double Loading (Feb 2026)

GitHub issue [#24044](https://github.com/anthropics/claude-code/issues/24044) reports that `MEMORY.md` gets loaded **twice per session** — once by the auto-memory loader and once by the claudeMd/project-instructions loader. This wastes ~3KB of duplicated tokens per API call (~60K tokens over a 20-turn conversation). The issue was closed as duplicate — a fix is expected.

## Mechanism 4: Agent Memory (v2.1.33+)

Introduced in Claude Code v2.1.33 (February 2026), this gives **subagents** their own persistent memory.

### Three Scopes

| Scope | Location | Version Controlled | Shared | Use Case |
|---|---|---|---|---|
| **user** | `~/.claude/agent-memory/<name>/` | No | No | Cross-project knowledge (default) |
| **project** | `.claude/agent-memory/<name>/` | Yes | Yes | Team-shared project patterns |
| **local** | `.claude/agent-memory-local/<name>/` | No | No | Personal project notes |

### How It Works

1. **Startup injection**: First 200 lines of agent's `MEMORY.md` load into agent's system prompt
2. **Automatic tool access**: Read, Write, Edit tools enabled for memory management
3. **Active management**: Agent reads and updates memory during execution
4. **Smart curation**: When `MEMORY.md` exceeds 200 lines, agent organizes overflow into topic files

### Agent Memory vs Other Memory

| System | Writer | Reader | Persistence |
|---|---|---|---|
| CLAUDE.md | Human | All agents + main Claude | Git / filesystem |
| Auto Memory | Main Claude | Main Claude only | Per-project filesystem |
| `/memory` command | Human | Main Claude only | Per-project filesystem |
| Agent Memory | Individual agent | That agent only | Per-agent filesystem |

## Mechanism 5: Skills (Progressive Disclosure)

Skills (`.claude/skills/`) complement the memory system by providing **on-demand knowledge loading** — descriptions are always in context, but full content loads only when invoked.

### Three-Level Loading

| Level | What Loads | When | Context Cost |
|---|---|---|---|
| **L1: Frontmatter** | Skill name + description | Always (system prompt) | Minimal |
| **L2: SKILL.md body** | Full instructions | When Claude decides it's relevant | Medium |
| **L3: Supporting files** | Templates, examples, scripts | When Claude navigates to them | On-demand |

### Skill Frontmatter

```yaml
---
name: api-conventions
description: API design patterns for this codebase
disable-model-invocation: true  # Manual only
user-invocable: false           # Claude only
allowed-tools: Read, Grep       # Restrict tools
context: fork                   # Run in subagent
agent: Explore                  # Which subagent type
model: sonnet                   # Model override
---
```

### Context Budget for Skills

Skill descriptions consume context from a dynamic budget: **2% of context window** (fallback: 16,000 characters). If you have too many skills, some get excluded. Check `/context` for warnings. Override with `SLASH_COMMAND_TOOL_CHAR_BUDGET` environment variable.

## Context Window and Memory Interaction

### Token Budget Breakdown

Typical session allocation (via `/context` command):

| Component | Tokens |
|---|---|
| System prompt | ~2.7K |
| System tools | ~16.8K |
| Custom agents | ~1.3K |
| **Memory files (CLAUDE.md + auto memory)** | **~7.4K** |
| Skills (descriptions only) | ~1.0K |
| Messages (conversation) | ~9.6K |
| **Auto-compact buffer (reserved)** | **33.0K** |

Total available before compaction: ~167K of 200K context window.

### Auto-Compaction

When context usage approaches the limit, Claude Code automatically compacts:

1. **Trigger threshold**: ~83.5% usage (buffer: ~33K tokens)
2. **Process**: Summarizes conversation history, compresses older messages
3. **Loss**: Granular details from early session are lost
4. **Boundary tracking**: Prevents recursive compaction failures by tracking "compact boundaries"

### Manual Compaction (`/compact`)

- Triggers compaction at any usage level (unlike auto at 98%)
- Accepts preservation instructions: `/compact keep the API patterns we established`
- Reduces tokens sent per message, lowering costs

### Environment Variables for Context Control

| Variable | Effect |
|---|---|
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Set compaction trigger threshold (1-100) |
| `autoCompact: false` in settings.json | Disable auto-compaction entirely |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Controls response length (NOT compaction buffer) |
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` | Override skill description context budget |

### Extended Context Models

- Default: 200K context window
- Sonnet 4.5: 500K context window
- Opus 4.6: 1M context window (Beta, requires API header)
- Use `sonnet[1m]` for 1M token window with proportionally larger buffer

## Best Practices

### CLAUDE.md Organization

**Keep it under 150-200 lines.** Research shows frontier LLMs can reliably follow 150-200 instructions. Claude Code's system prompt already contains ~50 instructions, leaving room for ~150 of yours.

**Structure: WHAT / WHY / HOW**
- **WHAT**: Tech stack, codebase map, key directories
- **WHY**: Project purpose, component functions
- **HOW**: Workflows, commands, testing procedures

**Don't use LLMs as linters.** If ESLint/Prettier/TypeScript can enforce it, don't put it in CLAUDE.md. Replace 200 lines of style prose with: `Run pnpm lint:fix && pnpm typecheck after changes.`

**Universal applicability.** Claude **deprioritizes** CLAUDE.md content it deems irrelevant to the current task. Only include guidance that applies to nearly every session.

### Progressive Disclosure Architecture

```
CLAUDE.md                    # Always loaded (~60 lines, pointers only)
  @docs/architecture.md      # Imported on reference
.claude/rules/               # Conditional rules with paths frontmatter
.claude/skills/              # On-demand full loading
agent_docs/                  # Claude reads when relevant
```

**Layer 1 — CLAUDE.md (always loaded):** Project overview, essential commands, stack info, pointers to deeper docs. Target: under 60 lines.

**Layer 2 — Rules and imports (conditionally loaded):** Path-scoped rules, imported reference docs. Loaded based on file context.

**Layer 3 — Skills and agent docs (on-demand):** Full workflows, detailed references, scripts. Loaded only when Claude decides they're relevant.

Token impact:
- System prompt: ~3.1K tokens
- Minimal CLAUDE.md: ~500 tokens
- Result: **196K+ tokens** available for actual work (~130 conversation turns)
- Bloated CLAUDE.md: consumes tokens in **every** session; on-demand docs consume only when needed

### Auto Memory Management

- **Periodically review** with `/memory` and prune stale entries
- **Keep MEMORY.md under 200 lines** — move details to topic files
- **Tell Claude to remember** specific things explicitly when important
- **Stale memory degrades performance** — outdated patterns cause Claude to make wrong assumptions

### Context Window Hygiene

- `/clear` aggressively between unrelated tasks
- `/compact` before major work to maximize available context
- Write handoff documents for context-heavy sessions that span compaction boundaries
- **Context freshness > context accumulation** — LLM output quality degrades with conversation length

## Tradeoffs

| Advantage | Disadvantage |
|---|---|
| Persistent context across sessions | 200-line limit forces aggressive curation |
| Hierarchical scoping matches real org structure | Double-loading bugs waste tokens |
| Auto memory learns without manual effort | Auto memory can accumulate stale/wrong patterns |
| Progressive disclosure saves context | Skills max out at 79% accuracy vs 100% for inline CLAUDE.md |
| Path-specific rules reduce noise | Frontmatter glob patterns add maintenance complexity |
| Import system enables modularity | Max-depth 5 and approval dialogs add friction |

## Alternatives Compared

| System | Approach | Strengths | Weaknesses |
|---|---|---|---|
| **Claude Code Memory** | File-based hierarchy (CLAUDE.md + auto memory + rules) | Native integration, zero setup, team sharing via git | 200-line limit, no semantic search, manual curation |
| **claude-mem** (plugin) | Automatic session capture + AI compression + injection | Captures everything automatically, AI-curated | External dependency, compression may lose nuance |
| **Cursor Rules** | `.cursorrules` file + `@docs` imports | Similar hierarchy, IDE-integrated | Tied to Cursor IDE, less flexible scoping |
| **Cline Memory Bank** | Structured markdown memory bank | Detailed architecture documentation | Manual maintenance heavy, no auto-learning |
| **Custom MCP Memory** | MCP server with vector DB backend | Semantic search, unlimited storage | Complex setup, external infrastructure |
| **Windsurf Rules** | `.windsurfrules` file | Simple single-file approach | No hierarchy, no auto memory, no conditional rules |

## Latest Updates (2026)

### Auto Memory Goes GA (v2.1.32, Feb 5 2026)

Auto memory graduated from experimental opt-in to enabled-by-default in v2.1.32. Claude now automatically records and recalls memories as it works without requiring the `CLAUDE_CODE_DISABLE_AUTO_MEMORY=0` environment variable. Users who want to disable it can still set `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`. This is the single biggest change to the memory system since its introduction — every Claude Code session now builds persistent project knowledge by default.

### Agent Memory Frontmatter (v2.1.32-2.1.33)

The `memory` frontmatter field for custom subagents was formalized in v2.1.32, supporting three scopes (`user`, `project`, `local`). v2.1.33 expanded it with `TeammateIdle` and `TaskCompleted` hook events, enabling multi-agent teams to coordinate memory updates. Each subagent gets its own persistent `MEMORY.md` with the same 200-line loading rule as the main session. Combined with `isolation: worktree` (which gives each agent its own git worktree), agents can now run fully parallel with independent memory and filesystem state.

### "Summarize from here" — Partial Compaction (v2.1.32)

A new alternative to `/compact` was added: using `Esc + Esc` or `/rewind`, users can select a message checkpoint and choose "Summarize from here" to condense only messages from that point forward, keeping earlier context intact. This gives much finer-grained control over context management compared to the all-or-nothing `/compact` command.

### Built-in Git Worktree Support (v2.1.39+)

The `--worktree` CLI flag enables running Claude Code in an isolated git worktree. Subagents can declare `isolation: worktree` in their agent definition. `WorktreeCreate` and `WorktreeRemove` hook events were added for custom VCS setup/teardown. This is critical for parallel agent workflows — multiple Claude sessions can edit the same repo without clobbering each other. Known limitation: worktrees share local databases and Docker state, creating potential race conditions for stateful operations.

### Memory Leak Fixes (v2.1.45-2.1.49)

February 2026 saw a wave of critical memory fixes after v2.1.27 shipped a regression that caused OOM crashes within seconds. The fixes included: releasing API stream buffers and agent context after use (v2.1.47), trimming agent task message history after completion to eliminate O(n^2) message accumulation (v2.1.47), capping shell command output to prevent unbounded RSS growth (v2.1.45), periodically resetting the tree-sitter parser to stop WASM memory growth (v2.1.49), and fixing out-of-memory crashes when resuming sessions with heavy subagent usage (v2.1.49). These were not cosmetic fixes — users reported 15-20GB RAM consumption in sessions longer than 20 minutes.

### New Hook Events for Enterprise Control

Two significant hook events were added: **PermissionRequest** (v2.1.45) lets external scripts automatically approve or deny tool permission requests with custom logic, and **ConfigChange** (v2.1.49) fires when configuration files change during a session, enabling enterprise security auditing and optional blocking of unauthorized settings changes. Both hooks support the same user/project/local scoping as the rest of the configuration system.

### CLAUDE.md from Additional Directories (v2.1.20)

The `--add-dir` flag gained the ability to load CLAUDE.md files from additional directories (via `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`). Skills in `.claude/skills/` within `--add-dir` directories also auto-load (v2.1.32). This is particularly useful for monorepos and shared configuration repositories where memory and rules live outside the immediate project root.

### Compaction Buffer Reduction

The autocompact buffer was reduced from ~45K tokens to ~33K tokens (16.5% of the 200K window), freeing roughly 12K additional tokens for actual work. Compaction also now correctly handles conversations containing many PDF documents by stripping document blocks alongside images before sending to the compaction API. Additionally, skills invoked by subagents no longer incorrectly appear in the main session context after compaction.

## Stealable Patterns

1. **The 200-line index pattern**: Keep your main memory file as a concise index with pointers to detailed topic files. This works for any LLM tool, not just Claude Code.

2. **Path-scoped conditional rules**: Only load frontend rules when working on frontend files. Eliminates noise and saves context tokens.

3. **Three-layer progressive disclosure**: Always-loaded (tiny) -> conditionally-loaded (medium) -> on-demand (large). Applies to any context-limited system.

4. **Explicit remember commands**: Don't rely on auto memory alone. When you discover something important, tell the LLM to save it explicitly.

5. **Context hygiene workflow**: `/clear` between tasks, `/compact` before big work, write handoff docs for long sessions. Treat context freshness as a first-class concern.

6. **Don't use LLMs as linters**: If a deterministic tool can enforce it, don't waste context tokens on it. Put `run lint` in CLAUDE.md instead of 200 lines of style rules.

7. **Symlink shared rules**: Use symlinks in `.claude/rules/` to share common rules across multiple projects without duplication.

8. **Agent memory scoping**: Use `user` scope for portable knowledge, `project` scope for team collaboration, `local` scope for personal annotations.

## References

### Official Documentation

- [Manage Claude's memory - Claude Code Docs](https://code.claude.com/docs/en/memory)
- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Using CLAUDE.MD files - Anthropic Blog](https://claude.com/blog/using-claude-md-files)
- [Context windows - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Compaction - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [Memory tool - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Agent Skills - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Skill authoring best practices - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Hooks reference - Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Common workflows - Claude Code Docs](https://code.claude.com/docs/en/common-workflows)

### GitHub Issues and Discussions

- [MEMORY.md loaded twice: auto-memory and claudeMd loaders both inject same file - Issue #24044](https://github.com/anthropics/claude-code/issues/24044)
- [Option to disable auto-memory - Issue #23750](https://github.com/anthropics/claude-code/issues/23750)
- [Slavka Memory Pattern: Unlimited scalable memory with a fixed context window - Issue #24718](https://github.com/anthropics/claude-code/issues/24718)
- [Configurable Context Window Compaction Threshold - Issue #15719](https://github.com/anthropics/claude-code/issues/15719)
- [Compaction fails with 'Conversation too long' at 48% (Opus 4.6) - Issue #23751](https://github.com/anthropics/claude-code/issues/23751)
- [Memory gauge forces chat termination at 0% while 60%+ token budget remains - Issue #10996](https://github.com/anthropics/claude-code/issues/10996)
- [Context compaction fails with 'Conversation too long' when context limit is reached - Issue #26317](https://github.com/anthropics/claude-code/issues/26317)
- [Critical memory regression in 2.1.27 - OOM crash on simple input - Issue #22042](https://github.com/anthropics/claude-code/issues/22042)
- [Subagent processes not terminating on macOS, causing memory leak - Issue #22554](https://github.com/anthropics/claude-code/issues/22554)
- [Memory leak in long-running idle Claude Code sessions - Issue #18859](https://github.com/anthropics/claude-code/issues/18859)
- [Add PreCompact and PostCompact hooks for custom context management - Issue #17237](https://github.com/anthropics/claude-code/issues/17237)

### Community Guides and Blog Posts

- [Writing a good CLAUDE.md - HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [Stop Bloating Your CLAUDE.md: Progressive Disclosure - alexop.dev](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/)
- [How I use Claude Code (+ my best tips) - Builder.io](https://www.builder.io/blog/claude-code)
- [Claude Code Memory System - Developer Toolkit](https://developertoolkit.ai/en/claude-code/advanced-techniques/memory-system/)
- [Claude Code Tips & Tricks: Maximising Memory - Cloud Artisan](https://cloudartisan.com/posts/2025-04-16-claude-code-tips-memory/)
- [Stop Repeating Yourself: Give Claude Code a Memory - ProductTalk](https://www.producttalk.org/give-claude-code-a-memory/)
- [Claude Code's Memory: Working with AI in Large Codebases - Thomas Landgraf](https://thomaslandgraf.substack.com/p/claude-codes-memory-working-with)
- [Claude Code Context Buffer: The 33K-45K Token Problem - ClaudeFast](https://claudefa.st/blog/guide/mechanics/context-buffer-management)
- [Persistent Memory for Claude Code: Setup Guide - Agent Native (Medium)](https://agentnativedev.medium.com/persistent-memory-for-claude-code-never-lose-context-setup-guide-2cb6c7f92c58)
- [Claude Code Best Practices: Memory Management - Cuong Tham (Medium)](https://medium.com/@codecentrevibe/claude-code-best-practices-memory-management-7bc291a87215)
- [Claude Code Compaction - Steve Kinney](https://stevekinney.com/courses/ai-development/claude-code-compaction)
- [Claude Code's Memory Evolution: Auto Memory & PreCompact Hooks - Yuanchang's Blog](https://yuanchang.org/en/posts/claude-code-auto-memory-and-hooks/)
- [Six Things That Changed in Claude Code This Month - Brent W. Peterson (Medium)](https://medium.com/@brentwpeterson/six-things-that-changed-in-claude-code-this-month-8012f49fcb90)
- [Claude Code Context Backups: Beat Auto-Compaction - ClaudeFast](https://claudefa.st/blog/tools/hooks/context-recovery-hook)
- [Claude Code Hooks: Complete Guide with 20+ Examples - aiorg.dev](https://aiorg.dev/blog/claude-code-hooks)
- [Claude Skills and CLAUDE.md: a practical 2026 guide for teams - Gend](https://www.gend.co/blog/claude-skills-claude-md-guide)

### Research and Analysis

- [Claude Code Agent Memory Report - shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice/blob/main/reports/claude-agent-memory.md)
- [Context Window & Compaction - DeepWiki](https://deepwiki.com/anthropics/claude-code/3.3-session-and-conversation-management)
- [Token Budget Management - DeepWiki](https://deepwiki.com/shanraisshan/claude-code-best-practice/4.3-token-budget-management)
- [Claude Code by Anthropic - Release Notes February 2026 - Releasebot](https://releasebot.io/updates/anthropic/claude-code)
- [Claude Agent Skills: A First Principles Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
- [Claude Code Changelog - ClaudeLog](https://claudelog.com/claude-code-changelog/)
- [Claude Code Changelog: Complete Version History - ClaudeFast](https://claudefa.st/blog/guide/changelog)

### Templates and Starter Configs

- [claude-code-best-practice - shanraisshan](https://github.com/shanraisshan/claude-code-best-practice)
- [my-claude-code-setup - centminmod](https://github.com/centminmod/my-claude-code-setup)
- [claude-mem - thedotmack](https://github.com/thedotmack/claude-mem)
- [awesome-claude-skills - travisvn](https://github.com/travisvn/awesome-claude-skills)

### Tools

- [CLAUDE.md for .NET Developers - codewithmukesh](https://codewithmukesh.com/blog/claude-md-mastery-dotnet/)
- [Claude Code Tutorial for Beginners - Complete 2026 Guide - codewithmukesh](https://codewithmukesh.com/blog/claude-code-for-beginners/)
- [Claude Skills and CLAUDE.md: a practical 2026 guide for teams - Gend](https://www.gend.co/blog/claude-skills-claude-md-guide)
