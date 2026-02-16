# OpenAI Codex CLI — Complete Usage Guide & Community Insights

## What

Codex CLI is OpenAI's open-source (Rust-built) terminal-based coding agent. It reads your codebase, suggests/implements changes, executes commands, and maintains security through OS-level sandboxing. Available on macOS, Linux, and experimentally on Windows (WSL recommended). Included with ChatGPT Plus/Pro/Business/Edu/Enterprise plans.

Think of it as OpenAI's answer to Claude Code — a local-first AI coding agent that lives in your terminal.

## Why This Approach

OpenAI chose terminal-first for the same reason Anthropic did with Claude Code: **IDEs are bottlenecks**. Terminal agents can:

- Read entire repos without extension limitations
- Execute arbitrary shell commands (build, test, deploy)
- Work across any language/framework without IDE plugins
- Integrate into CI/CD and scripting pipelines
- Run headlessly for automation (`codex exec`)

The key differentiator vs Claude Code: **Codex Cloud** — the ability to offload tasks to sandboxed cloud environments and apply diffs locally. Claude Code is purely local.

## Installation & Setup

### Install

```bash
npm i -g @openai/codex
```

Requires Node.js >= 22.

### Authenticate

```bash
codex          # first run triggers OAuth login
codex login    # explicit login (ChatGPT account or API key)
```

Token saved to `~/.codex/`. No manual key copy needed.

### Verify

```bash
codex --version
```

## Core Concepts

### Approval Modes (Permission Levels)

| Mode | What It Does | Flag |
|------|-------------|------|
| **Read-only** | Browse files only, asks before any change | default |
| **Auto** | Read, edit, run commands within working dir | `--full-auto` |
| **Full Access** | Unrestricted machine access including network | `--sandbox danger-full-access` |
| **YOLO** | Skip ALL safety (isolated VMs only) | `--dangerously-bypass-approvals-and-sandbox` |

Practical workflow: start with read-only to explore, then `--full-auto` for daily dev.

### Sandbox Modes

| Sandbox | Scope | Network |
|---------|-------|---------|
| `read-only` | Read files only | Blocked |
| `workspace-write` | Edit within project dir | Blocked (configurable) |
| `danger-full-access` | Full system access | Allowed |

Enable network in workspace-write:
```bash
codex -c 'sandbox_workspace_write.network_access=true'
```

### Model Selection

| Model | Use Case | Notes |
|-------|----------|-------|
| `gpt-5.3-codex` | Default, daily coding | Best balance of speed/quality |
| `gpt-5.3-codex-spark` | Pro subscribers only | Research preview, faster |
| `gpt-5.2-codex` medium/high | Routine tasks | Cheaper |
| `gpt-5.2-codex` xhigh | Complex reasoning | Expensive, slow — reserve for hard problems |

Switch mid-session: `/model` slash command or `--model` flag at launch.

## Complete Command Reference

### Core Commands

```bash
codex                          # interactive TUI
codex "explain this project"   # start with a prompt
codex exec "fix the tests"     # non-interactive (alias: codex e)
codex resume                   # continue previous session
codex fork                     # clone session into new thread
codex cloud                    # browse/launch cloud tasks
codex app                      # launch macOS desktop app
codex apply                    # apply cloud task diff locally
```

### Key Flags

```bash
codex --full-auto              # auto-approve edits + commands
codex --model gpt-5.3-codex   # specify model
codex -i screenshot.png        # attach image
codex --search                 # enable live web search
codex --profile fast           # load named config profile
codex -C /path/to/project      # set working directory
codex --add-dir /extra/path    # grant additional dir write access
codex --oss                    # use local model via Ollama
```

### Exec Mode (Scripting/CI)

```bash
codex exec "update changelog" --json          # JSONL output
codex exec "fix lint" --ephemeral             # don't save session
codex exec "generate types" -o output.md      # write result to file
codex exec "validate schema" --output-schema schema.json  # validate output
codex exec resume SESSION_ID                  # resume non-interactive session
```

## Slash Commands (24 Built-in)

### Model & Config
| Command | Purpose |
|---------|---------|
| `/model` | Switch models mid-session |
| `/personality` | Set style: `friendly`, `pragmatic`, `none` |
| `/status` | Show session config + token usage |
| `/debug-config` | Print config layers and diagnostics |

### Workflow
| Command | Purpose |
|---------|---------|
| `/plan` | Enter plan mode (think before acting) |
| `/permissions` | Adjust approval presets |
| `/compact` | Summarize conversation (save context) |
| `/diff` | Review Git changes including untracked |
| `/review` | Analyze working tree (code review) |

### Context
| Command | Purpose |
|---------|---------|
| `/mention` | Attach specific files to conversation |
| `/mcp` | List MCP tools |
| `/apps` | Browse available connectors |

### Session
| Command | Purpose |
|---------|---------|
| `/new` | Fresh conversation, same CLI session |
| `/resume` | Reload previous session |
| `/fork` | Clone conversation into new thread |
| `/init` | Generate AGENTS.md scaffold |

### Utility
| Command | Purpose |
|---------|---------|
| `/ps` | Monitor background terminals |
| `/statusline` | Configure footer display |
| `/feedback` | Submit diagnostics |
| `/quit` / `/exit` | Exit (save work first!) |

## AGENTS.md — The Control Plane

AGENTS.md is to Codex what CLAUDE.md is to Claude Code. It provides persistent instructions that Codex reads before every task.

### Discovery Hierarchy (3 tiers)

```
1. ~/.codex/AGENTS.override.md  (global override — highest priority)
2. ~/.codex/AGENTS.md           (global defaults)
3. <git-root>/AGENTS.md         (project level)
4. <git-root>/services/payments/AGENTS.override.md  (nested override)
```

Files concatenated root-down. Closer files override earlier ones.

### Example AGENTS.md

```markdown
# Project Instructions

## Build & Test
- Run `npm run lint` before opening PRs
- Run `npm test` for all changes
- Use `make test-payments` for payment service specifically

## Code Style
- TypeScript strict mode, no `any`
- Prefer functional patterns over classes
- Document public utilities in `docs/`

## Safety
- Never rotate API keys without notifying #security channel
- Always use parameterized queries for database access
```

### Configuration

```toml
# ~/.codex/config.toml
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
project_doc_max_bytes = 65536  # default 32 KiB
```

Changes take effect immediately — no cache clearing.

## Advanced Configuration (config.toml)

### Profiles

```toml
[profiles.fast]
model = "gpt-5.3-codex"
model_reasoning_effort = "low"
approval_policy = "on-request"

[profiles.careful]
model = "gpt-5.2-codex"
model_reasoning_effort = "high"
approval_policy = "untrusted"
```

Switch: `codex --profile fast`

### Custom Model Providers

```toml
[model_providers.proxy]
base_url = "http://proxy.example.com"
env_key = "OPENAI_API_KEY"
http_headers = { "X-Custom" = "value" }
```

### Shell Environment Control

```toml
[shell_environment_policy]
inherit = "core"
exclude = ["AWS_*", "AZURE_*"]
include_only = ["PATH", "HOME"]
```

### Observability (OpenTelemetry)

```toml
[otel]
# Track API requests, prompts, tool approvals
exporter = "otlp-http"  # or "otlp-grpc", "none"
```

### Writable Roots (Sandbox Escape Hatch)

```toml
[sandbox_workspace_write]
network_access = false
writable_roots = ["/Users/YOU/.pyenv/shims"]
```

## Prompting Best Practices (Official Guide)

### Core Principles

1. **Treat Codex as a senior engineer** — it should gather context, plan, implement, test, and refine autonomously
2. **Bias to action** — avoid ending turns with clarifying questions unless truly blocked
3. **Batch operations** — decide ALL files/resources needed upfront, read them in parallel
4. **Preserve existing patterns** — follow codebase conventions, helpers, naming, formatting

### What Works

- Specific prompts with file paths and constraints
- Test-driven: define tests upfront so Codex iterates until passing
- Attach screenshots for UI tasks (`-i screenshot.png`)
- Use `/plan` for complex tasks, skip for simple ones (easiest ~25%)

### What Doesn't Work

- Vague prompts ("make it better")
- Asking for status updates in system prompt (causes premature stopping)
- Sequential file reads when parallel is possible
- Broad try-catch blocks / silent error swallowing
- `as any` type assertions

### Frontend-Specific Tips

Official guide explicitly warns against "AI slop":
- Use expressive typography beyond default font stacks
- Define visual direction through CSS variables
- Employ meaningful animations (not decorative)
- Use gradients/patterns instead of flat colors

## Workflow Patterns

### Daily Development Loop

```
1. codex                    # start interactive session
2. "explain this codebase"  # orient yourself
3. /plan                    # plan complex changes
4. --full-auto              # execute with auto-approval
5. /review                  # review before commit
6. /diff                    # check git changes
```

### CI/Automation Pipeline

```bash
# Lint fix in CI
codex exec "fix all lint errors" --ephemeral --json

# Generate changelog
codex exec "update CHANGELOG.md from recent commits" -o CHANGELOG.md

# Validate PR
codex exec "review this PR for security issues" --output-schema review.json
```

### Code Review Workflow

```bash
# Review against main branch
/review base=main

# Review specific focus areas
/review --focus security,edge-cases

# GitHub PR review (via GitHub app)
# Comment @codex review on any PR
```

### Cloud Task Delegation

```bash
codex cloud                       # interactive task picker
codex cloud --env ENV_ID          # launch in specific environment
codex cloud --attempts 3          # best-of-N attempts
codex apply                       # apply cloud diff locally
```

### Session Persistence

```bash
codex resume                      # pick from recent sessions
codex resume --all                # show all sessions across dirs
codex resume SESSION_ID           # target specific session
codex fork                        # branch current conversation
```

## MCP Integration

Connect external tools via Model Context Protocol:

```bash
codex mcp add my-server --stdio "npx @my/mcp-server"
codex mcp add my-http --url "https://mcp.example.com"
codex mcp list
codex mcp login my-http    # OAuth for HTTP servers
```

## Community Best Practices & Tips

### From the OpenAI Developer Community

1. **Test-driven is king** — define tests upfront, let Codex iterate until green. "When AI is involved, traditional programming assumptions break down"
2. **Tiger Style programming** — red-green-refactor with strict methodology for maximum reliability
3. **High-quality local docs > web scraping** — equip agents with robust local documentation
4. **Minimize AGENTS.md context pollution** — don't load docs unless explicitly needed, preserve token efficiency
5. **Incremental patching** — modify imports, function signatures, and returns separately (no fuzzy matching)
6. **Handle line endings** — LF vs CRLF inconsistency causes patch failures

### VS Code Integration

Install "Codex Companion" extension:
- `@codex` in sidebar for direct conversation
- `Alt+G` sends selected code to CLI
- Agent mode: reads files, runs commands, writes changes

### Power User Tips

- **Session export/import**: `/export session.json` → `/load session.json` for context persistence across long refactors
- **Profiles for context switching**: `--profile fast` for quick fixes, `--profile careful` for production changes
- **Image-driven development**: attach mockups with `-i` for UI implementation
- **Parallel reads**: always batch file reads to minimize round-trips

## Codex CLI vs Claude Code — Head-to-Head

| Dimension | Codex CLI | Claude Code |
|-----------|-----------|-------------|
| **Builder** | OpenAI (Rust) | Anthropic (TypeScript) |
| **Model** | GPT-5.3-Codex | Claude Opus 4.6 |
| **Cloud execution** | Yes (Codex Cloud) | No (local only) |
| **Token efficiency** | 2-3x fewer tokens | More thorough but expensive |
| **Codebase understanding** | Strong for incremental | Better for global/architectural |
| **Instructions file** | AGENTS.md | CLAUDE.md |
| **Sandbox** | OS-level (seatbelt/landlock) | Permission system (often bypassed) |
| **GitHub integration** | Native app (auto PR review) | gh CLI based |
| **Permission UX** | Git-aware, permissive default | Friction-heavy, settings don't persist |
| **IDE extension** | Codex Companion (VS Code) | Claude Code extension |
| **Automation** | `codex exec` (first-class) | Limited scripting support |
| **Cost** | Included with ChatGPT plans | Per-token API billing |

### When to Use Which

- **Codex**: speed, cost efficiency, GitHub-native workflows, CI automation, routine coding tasks
- **Claude Code**: deep codebase understanding, complex multi-file refactoring, architectural decisions, investigative work

### Community Consensus

> "Use Claude Code for investigative work and high-level decisions. Use Codex for producing code after you have mapped the path forward."

The optimal workflow is multi-tool: use Claude Code to understand and plan, Codex to execute and iterate.

## Tradeoffs

| Advantage | Limitation |
|-----------|-----------|
| 2-3x token efficiency vs Claude Code | Windows support still experimental (WSL needed) |
| Cloud execution (async background tasks) | Requires ChatGPT subscription (no free tier) |
| Native GitHub app for PR review | Sandbox network blocked by default (friction for API work) |
| Rust-built (fast, low memory) | Newer ecosystem, fewer community patterns |
| First-class scripting (`codex exec`) | Less proven on complex multi-file refactors |
| Profile system for context switching | AGENTS.md 32KB limit (configurable) |

## Steal — Actionable Patterns

### 1. Test-Driven Agent Loop
Define tests first, let the agent iterate until green. This is the single highest-impact pattern for AI-assisted coding. Works identically in both Codex and Claude Code.

### 2. AGENTS.md / CLAUDE.md Layering
Both tools support hierarchical instruction files. Pattern: global defaults → project conventions → subdirectory overrides. Keep instructions concise, use progressive disclosure.

### 3. Profile-Based Workflows
```toml
[profiles.explore]
approval_policy = "untrusted"
model_reasoning_effort = "low"

[profiles.ship]
approval_policy = "on-request"
model_reasoning_effort = "high"
```

Switch contexts without remembering flags.

### 4. exec for CI Integration
```bash
codex exec "review for security vulnerabilities" --json --ephemeral
```
Non-interactive mode makes AI code review a pipeline step, not a manual process.

### 5. Cloud + Local Hybrid
Use `codex cloud` for slow background tasks (large refactors, test generation), `codex apply` to pull results into local workspace. This is uniquely Codex — no Claude Code equivalent.

### 6. Multi-Tool Workflow
```
Claude Code (understand) → Codex (execute) → Claude Code (review)
```
Use each tool where it's strongest. Don't pick one — use both.

### 7. Image-Driven UI Development
```bash
codex -i mockup.png "implement this design using React + Tailwind"
```
Attach screenshots, mockups, or Figma exports directly. Faster than describing layouts in text.

### 8. Incremental Patching Discipline
When writing AGENTS.md instructions for code modification, specify: "patch one change at a time — imports, function signatures, returns separately." Prevents patch failures from context mismatch.
