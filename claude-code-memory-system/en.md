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

## Stealable Patterns

1. **The 200-line index pattern**: Keep your main memory file as a concise index with pointers to detailed topic files. This works for any LLM tool, not just Claude Code.

2. **Path-scoped conditional rules**: Only load frontend rules when working on frontend files. Eliminates noise and saves context tokens.

3. **Three-layer progressive disclosure**: Always-loaded (tiny) -> conditionally-loaded (medium) -> on-demand (large). Applies to any context-limited system.

4. **Explicit remember commands**: Don't rely on auto memory alone. When you discover something important, tell the LLM to save it explicitly.

5. **Context hygiene workflow**: `/clear` between tasks, `/compact` before big work, write handoff docs for long sessions. Treat context freshness as a first-class concern.

6. **Don't use LLMs as linters**: If a deterministic tool can enforce it, don't waste context tokens on it. Put `run lint` in CLAUDE.md instead of 200 lines of style rules.

7. **Symlink shared rules**: Use symlinks in `.claude/rules/` to share common rules across multiple projects without duplication.

8. **Agent memory scoping**: Use `user` scope for portable knowledge, `project` scope for team collaboration, `local` scope for personal annotations.
