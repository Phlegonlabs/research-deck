# OpenAI Codex CLI — Sandbox & Isolation Architecture Deep Dive

## What They Built

OpenAI Codex CLI implements a **two-layer security system** combining OS-level sandboxing with an approval policy layer. Every model-generated command runs inside a platform-specific sandbox (macOS Seatbelt, Linux Landlock+seccomp, Windows Restricted Token) with network disabled and writes restricted to the workspace by default. The approval policy layer decides when the human must intervene.

The entire sandbox implementation lives in the Rust codebase (`codex-rs/`) as a Cargo workspace, with dedicated crates for each platform:

| Crate | Platform | Mechanism |
|-------|----------|-----------|
| `core/src/seatbelt/` | macOS | Apple Seatbelt via `sandbox-exec` |
| `linux-sandbox/` | Linux | Landlock LSM + seccomp-BPF |
| `windows-sandbox-rs/` | Windows | Restricted Token + AppContainer |
| `execpolicy/` | All | Starlark-based command policy engine |
| `process-hardening/` | All | Additional security hardening |

## Why This Architecture

### The Core Problem

An AI coding agent needs shell access to be useful, but shell access is unbounded power. You can't just run `os.system(model_output)` and hope for the best. The threat model:

1. **Model hallucination** — the model generates destructive commands by accident
2. **Prompt injection** — malicious content in files/URLs tricks the model into harmful actions
3. **Data exfiltration** — the model reads secrets and sends them over the network
4. **Filesystem corruption** — writing outside the project directory

### Why OS-Level (Not Just Application-Level)

Application-level sandboxing (parsing commands, maintaining blocklists) is fundamentally bypassable — there are infinite ways to encode `rm -rf /` in shell. OS-level sandboxing is **mandatory access control**: the kernel enforces it regardless of how clever the command is. Even if the model finds a novel command encoding, the kernel blocks the syscall.

### Why Two Layers

The sandbox alone isn't enough because many legitimate development tasks need elevated access (installing packages, running tests that hit localhost). The approval policy provides a **graduated escalation path** — sandbox constrains the default, approvals gate the exceptions.

## How It Works — Platform Deep Dive

### Architecture: arg0 Dispatch

Codex uses a clever "arg0 dispatch" pattern to avoid shipping separate sandbox binaries. The main `codex` binary checks `argv[0]` at startup:

```
arg0_dispatch_or_else() {
    if binary_name matches "*-linux-sandbox" → enter sandbox mode
    if binary_name matches "*-apply-patch"  → enter patch mode
    else                                    → normal CLI dispatch
}
```

When sandboxing is needed, the core re-executes itself with `argv[0]` overridden (e.g., `codex-linux-sandbox`), triggering the sandbox codepath. This eliminates separate helper binaries while maintaining privilege separation. The pattern matching is prefix-based, so platform-specific binaries like `codex-x86_64-unknown-linux-musl-linux-sandbox` work correctly.

These handlers are **terminal operations** — they call `std::process::exit()` directly without returning to the main CLI dispatcher.

### Sandbox Modes

Three modes control what sandboxed processes can do:

| Mode | Filesystem | Network | Use Case |
|------|-----------|---------|----------|
| `read-only` | Read anywhere, write nowhere | Blocked | Safe browsing/planning |
| `workspace-write` | Read anywhere, write to workspace + tmp | Blocked | Default development |
| `danger-full-access` | Unrestricted | Unrestricted | Pre-isolated environments only |

**Critical design choice**: Even `read-only` mode grants **full filesystem read access** — the entire disk, not just the project. This is because restricting reads to just `cwd` would break everything (can't read `/bin/cat`, system libraries, etc.). This has been flagged as a security concern (GitHub Issue #4410) because it enables data exfiltration of secrets anywhere on the filesystem. OpenAI's position: restricting reads would be "crippling default behavior for most users."

**Default selection logic**: Codex detects version control status:
- Git repository → `workspace-write` + `on-request` approvals (called "Auto" mode)
- No VCS → `read-only` mode

### macOS: Apple Seatbelt

**Technology**: macOS Seatbelt is a kernel-level mandatory access control framework. Commands are wrapped with `/usr/bin/sandbox-exec -p <profile>`. The profile uses SBPL (Sandbox Profile Language), a Scheme-based DSL.

**Implementation details**:

1. `create_seatbelt_command()` in `core/src/seatbelt/` assembles the profile dynamically
2. Hardcodes `/usr/bin/sandbox-exec` (prevents PATH injection attacks)
3. Validates sandbox availability by checking if the binary exists
4. Combines a base policy (`seatbelt_base_policy.sbpl`) with runtime-generated permissions

**Profile structure** (Seatbelt is deny-by-default):

```scheme
(version 1)
(deny default)                          ; deny everything by default

;; ReadOnly mode:
(allow file-read*)                      ; allow all file reads

;; WorkspaceWrite mode adds:
(allow file-write* (subpath "/path/to/workspace"))
(allow file-write* (subpath "/tmp"))

;; Network (when enabled):
(allow network-outbound)
(allow network-inbound)
(allow system-socket)
;; When disabled: simply omitted (deny-by-default handles it)
```

**Writable root enumeration**:
- `.git/` directories explicitly carved out as read-only (prevents git history tampering)
- `.codex/` kept read-only even when workspace is writable
- Workspace roots from configuration added as `(allow file-write* (subpath ...))`

**Network policy**: Loaded from `seatbelt_network_policy.sbpl`. Binary allow/deny — either full network or no network. **Known bug**: `network_access = true` in config.toml is silently ignored; the Seatbelt profile unconditionally sets `CODEX_SANDBOX_NETWORK_DISABLED=1` (Issue #10390). Cannot express granular rules like "only HTTPS to api.openai.com."

**Capabilities blocked**: File writes, network, Apple Events, accessibility services.

**Deprecation concern**: `sandbox-exec` was marked deprecated around macOS Sierra (2016), but Apple still maintains backward compatibility. Seatbelt itself (the kernel framework) is NOT deprecated. The tension: Apple discourages direct usage while keeping it working for critical apps like Chrome. OpenAI continues using it because no better alternative exists for programmatic sandboxing on macOS.

**Known limitation**: Researchers found Seatbelt profiles "failed to properly block off access to the user's home directory dotfiles and `~/Library`" when improperly configured. Requires deep macOS internals expertise that shifts between OS releases.

### Linux: Landlock + seccomp-BPF

**Two complementary kernel mechanisms**:
- **Landlock LSM** (Linux Security Module, kernel >= 5.13): Filesystem access control
- **seccomp-BPF** (Berkeley Packet Filter): Syscall filtering

**Implementation**: The `codex-linux-sandbox` crate runs as a separate process (via arg0 dispatch). The parent serializes `SandboxPolicy` to JSON and invokes the helper. The helper:
1. Parses the serialized policy
2. Applies Landlock restrictions
3. Installs seccomp filters
4. Calls `execvp` to replace itself with the target command

**Landlock filesystem rules**:

```
Default:
  - Read access: entire filesystem (same rationale as macOS)
  - Write access: /dev/null (always allowed)

WorkspaceWrite adds:
  - Write access: cwd (workspace root)
  - Write access: $TMPDIR, /tmp (configurable)
  - Write access: writable_roots[] from config
  - NOT writable: .git/, .codex/ within workspace
```

Rules applied before `execvp`, ensuring sandboxed child processes cannot escape.

**seccomp-BPF syscall filtering** (network isolation):

Blocked syscalls when network disabled:
```
SYS_connect    — prevent outbound connections
SYS_accept     — prevent inbound connections
SYS_bind       — prevent port binding
SYS_listen     — prevent listening
SYS_sendto     — prevent sending data
SYS_sendmsg    — prevent sending messages
SYS_sendmmsg   — prevent batch sending
io_uring       — prevent io_uring network bypass (added later)
```

**Explicitly NOT blocked**:
```
SYS_recvfrom   — needed by cargo clippy and other tools
                  that use socketpair for subprocess management
socket()       — only AF_UNIX allowed (local IPC);
                  AF_INET/AF_INET6 denied by checking
                  the first argument (domain)
```

The filter uses a **default-allow strategy with specific deny rules** for network syscalls, checking the domain argument for `socket()` to permit `AF_UNIX` while denying network sockets. This is more precise than blocking `socket()` entirely.

**Architecture support**: Only x86_64 and aarch64. Other architectures cause a panic.

**Environment variables**:
- `CODEX_SANDBOX_NETWORK_DISABLED=1` — signals to child processes that network is disabled
- `CODEX_UNSAFE_ALLOW_NO_SANDBOX` — emergency fallback when sandboxing fails (e.g., in containers)
- `prctl(PR_SET_PDEATHSIG)` — registers parent-death signal handler preventing orphaned processes

**Alternative pipeline**: Bubblewrap (bwrap) is available as an alternative via `features.use_linux_sandbox_bwrap = true`. Bubblewrap creates a new mount namespace and can mask sensitive files by mounting empty tmpfs over them. The Linux sandbox build process always compiles vendored bubblewrap.

### Windows: Restricted Token + AppContainer

**Status**: Experimental. Uses the `codex-windows-sandbox` crate (library, not separate binary).

**Mechanism**:
1. Creates a restricted token via `CreateRestrictedToken()` Windows API
2. Derives token from an AppContainer profile
3. Grants selective filesystem capabilities via security identifiers
4. Configures job objects for process isolation

**Filesystem control**:
- Writes blocked everywhere except declared workspace roots + `%TEMP%`
- ACL management grants AppContainer permissions to specific directories
- Common escape vectors proactively denied:
  - **Alternate data streams** (NTFS ADS)
  - **UNC paths** (`\\server\share`)
  - **Device handles** (`\\.\PhysicalDrive0`)

**Network lockdown**: Overrides proxy variables and injects stub executables.

**PATH injection defense**: The CLI injects stub executables (e.g., wrapping `ssh`) ahead of the host PATH to intercept dangerous tools before they leave the sandbox.

**Critical limitation**: Cannot prevent writes in directories where the `Everyone` SID already has write permissions (world-writable folders). Codex scans for these and recommends removing such access.

**Browser escape**: Read-only runs can still launch the default browser via `Start-Process 'https://...'` because Explorer handles `ShellExecute` outside the sandbox.

### Windows + WSL (Recommended Path)

Native Windows sandbox is experimental. The recommended approach is WSL2:
- Provides Linux shell + Unix semantics matching model training data
- Uses the full Linux Landlock + seccomp sandbox
- Performance: keep repos in Linux home (`~/code/...`), NOT `/mnt/c/` (avoids slower I/O and symlink/permission issues)

## Network Isolation

**Default**: Network disabled across all platforms. This is the strongest security guarantee — prevents data exfiltration even if the model reads secrets.

**Implementation by platform**:

| Platform | Mechanism | Granularity |
|----------|-----------|-------------|
| macOS | Seatbelt profile rules | Binary: all or nothing |
| Linux | seccomp-BPF syscall filter | Syscall-level, AF_UNIX exempted |
| Windows | Proxy override + stub executables | Application-level |

**Fundamental limitation**: None of the platforms can express **application-layer** network policies like "only HTTPS to api.openai.com" or "allow localhost:3000 but not the internet." It's all-or-nothing at the network level.

**Web search behavior**: When network is disabled, web search uses "cached" mode — OpenAI-maintained indexed results rather than live fetches, reducing prompt injection exposure. Full network access switches to live results.

**Configuration**:
```toml
[sandbox_workspace_write]
network_access = true  # default: false
```

## writable_roots Configuration

The `writable_roots` setting expands the write boundary beyond the workspace:

```toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
writable_roots = ["/Users/YOU/.pyenv/shims", "/home/you/.cargo/bin"]
network_access = false
exclude_tmpdir_env_var = false  # default: false (TMPDIR writable)
exclude_slash_tmp = false       # default: false (/tmp writable)
```

**CLI equivalent**: `--add-dir` flag (repeatable):
```bash
codex --cd apps/frontend --add-dir ../backend --add-dir ../shared
```

**What's always writable in workspace-write**:
- Current working directory (cwd)
- `$TMPDIR` (unless `exclude_tmpdir_env_var = true`)
- `/tmp` (unless `exclude_slash_tmp = true`)
- Paths in `writable_roots[]`

**What's always read-only even in workspace-write**:
- `.git/` directory (prevents git history tampering)
- `.codex/` directory (prevents config self-modification)

## The Approval Policy System

### Four Approval Levels

| Policy | Behavior | Best For |
|--------|----------|----------|
| `untrusted` | Only known-safe read-only commands auto-run; all others prompt | New/untrusted projects |
| `on-failure` | Auto-run in sandbox; prompt only when command fails | Iterative development |
| `on-request` | Model decides when to ask (default) | Normal development |
| `never` | Never prompt; execute within sandbox constraints | Full automation |

### Interaction with Sandbox

The two layers are **independent but complementary**:

```
sandbox_mode = "workspace-write"  →  OS restricts what CAN happen
approval_policy = "on-request"    →  Policy restricts what DOES happen
```

Powerful combination: `danger-full-access` + `untrusted` = "Codex can do anything, but only with explicit human approval for every action" (mimics Claude Code's default approach).

### Escalation Flow

When sandboxed execution fails due to permissions:
1. System detects the failure
2. Prompts user for unsandboxed retry
3. If approved, re-executes with `SandboxType::None`
4. Session-scoped approval tracking prevents repeated prompts for same command pattern

### Execpolicy: Semantic Command Control

Beyond OS-level sandboxing, the `execpolicy/` crate adds **application-level semantic control** using Starlark (Python-like configuration language):

```starlark
# Example rule
rule(
    pattern = ["rm", "-rf", "*"],
    decision = "forbidden",
    justification = "Recursive forced deletion too dangerous"
)
```

Three-tier decisions: `allow` > `prompt` > `forbidden`. When multiple rules match, the **most restrictive** wins. This catches semantically dangerous operations even within sandboxed boundaries.

Rules files live under `~/.codex/rules/` or `.codex/rules/` (project-level).

### Enterprise Enforcement

**`/etc/codex/requirements.toml`** — admin-enforced, non-overridable:
```toml
allowed_sandbox_modes = ["read-only", "workspace-write"]
allowed_approval_policies = ["untrusted", "on-failure", "on-request"]
```

This blocks users from selecting `danger-full-access` or `never` approval.

**`/etc/codex/managed_config.toml`** — starting defaults, user-overridable per session.

**macOS MDM**: Base64-encoded TOML payloads at preference domain `com.openai.codex` with keys `config_toml_base64` and `requirements_toml_base64`.

**Cloud enforcement**: When signed in with ChatGPT Business/Enterprise, Codex fetches admin-enforced requirements from the Codex service.

**Untrusted projects**: Marking a project untrusted skips all project-scoped `.codex/` layers (including config.toml), falling back to user/system/built-in defaults only.

## YOLO Mode: --dangerously-bypass-approvals-and-sandbox

`--yolo` is the alias for `--dangerously-bypass-approvals-and-sandbox`. It:

1. Sets `sandbox_mode = "danger-full-access"` (no OS sandbox)
2. Sets `approval_policy = "never"` (no human prompts)
3. Grants full filesystem + network access
4. Runs every model-generated command immediately

**Intended use**: Only inside pre-isolated environments (Docker containers, VMs, CI/CD pipelines).

**What changes**: Web search switches from cached to live results. All filesystem reads/writes unrestricted. All network connections permitted.

**The name is the warning**: The flag name itself is a security mechanism — it's deliberately scary to prevent casual use.

## Security Analysis

### Defense-in-Depth Layers

```
Layer 1: Execpolicy    — semantic command filtering (Starlark rules)
Layer 2: Approval Gate — human-in-the-loop for dangerous ops
Layer 3: OS Sandbox    — kernel-enforced mandatory access control
Layer 4: Git VCS       — detection + .git read-only protection
Layer 5: Enterprise    — requirements.toml enforcement
```

### Known Vulnerabilities

**CVE-2025-59532 (CVSS 8.6)** — Sandbox bypass via path configuration bug
- Affected: CLI 0.2.0–0.38.0, IDE Extension through 0.4.11
- Bug: Model-generated `cwd` was treated as sandbox writable root, enabling arbitrary file writes outside the user's session directory
- Fix (v0.39.0): Canonicalizes and validates sandbox boundaries based on session start location, rejecting model-generated paths
- Network restrictions remained intact during exploit

### Structural Weaknesses

| Weakness | Impact | Mitigation |
|----------|--------|------------|
| Full filesystem read in read-only mode | Secret exfiltration (SSH keys, .env, credentials) | Network blocking prevents exfil; Docker recommended for sensitive envs |
| macOS Seatbelt is "deprecated" (sandbox-exec) | Future Apple breakage possible | Apple maintains backward compat; no better alternative exists |
| Seatbelt can't protect dotfiles/~/Library properly | Config/credential exposure | Requires careful profile engineering |
| Binary network control (all or nothing) | No per-domain filtering | Cannot restrict to "only localhost" or "only api.openai.com" |
| Windows can't block world-writable dirs | Write escape via Everyone SID folders | Codex scans and warns; recommends fixing permissions |
| Windows browser escape | Can launch URLs via ShellExecute | Inherent OS limitation |
| seccomp only x86_64/aarch64 | No ARM32/RISC-V support | Panic on unsupported arch |
| Landlock requires kernel >= 5.13 | Old distros/WSL may not support | Fallback to CODEX_UNSAFE_ALLOW_NO_SANDBOX |
| io_uring was initially not blocked | Network bypass via io_uring syscalls | Fixed in later release |
| macOS network_access=true in config silently ignored | Network remains blocked despite config | Must use CLI flag as workaround |
| Environment sanitization gap | Inherited env vars could leak info | `spawn_child_async` clears env and rebuilds |

### What's Well-Designed

1. **arg0 dispatch** eliminates separate binaries — single executable, less attack surface
2. **.git read-only** prevents history tampering even in workspace-write mode
3. **Environment clearing** on child spawn — only intended variables passed through
4. **Parent death signal** (`PR_SET_PDEATHSIG`) prevents orphaned sandboxed processes
5. **Deny-by-default** on Seatbelt — omitting a permission blocks it automatically
6. **AF_UNIX exemption** in seccomp — allows local IPC without breaking tool chains
7. **Timeout enforcement** (10s default) — prevents runaway sandboxed processes
8. **Enterprise enforcement** via `/etc/codex/requirements.toml` — users can't override

## Process Execution Flow

Complete path from model output to sandboxed execution:

```
1. Model generates function call (shell command)
2. ToolRouter evaluates operation type
3. ExecPolicyManager checks Starlark rules → allow/prompt/forbid
4. ToolOrchestrator checks approval_policy → prompt if needed
5. User approves (or auto-approved based on policy)
6. Sandbox mode selected based on config
7. spawn_child_async:
   a. Clear environment, rebuild with only intended vars
   b. Set CODEX_SANDBOX_NETWORK_DISABLED if applicable
   c. Platform dispatch:
      - macOS: /usr/bin/sandbox-exec -p <generated_profile> <command>
      - Linux: re-exec as codex-linux-sandbox → apply landlock → apply seccomp → execvp
      - Windows: CreateRestrictedToken → job object → execute
8. Output capped at 10 KiB / 256 lines (read_capped)
9. Exit code examined: command failure vs sandbox violation
10. If sandbox violation → offer unsandboxed retry with approval
11. Results streamed back as OutputDelta + ExecCommandEnd events
```

## Comparison with Claude Code

| Aspect | Codex CLI | Claude Code |
|--------|-----------|-------------|
| Sandbox tech | Seatbelt / Landlock+seccomp / Restricted Token | Permission deny-rules + devcontainers |
| Default network | Blocked | Allowed (with approval prompts) |
| Filesystem reads | Full disk (all modes) | Full disk |
| Approval model | 4-tier policy system | Per-command approval prompts |
| Enterprise control | requirements.toml + MDM | Managed settings |
| Execution policy | Starlark-based rules engine | Hook-based interceptors |
| YOLO mode | `--yolo` flag | `--dangerously-skip-permissions` |
| Windows | WSL recommended + experimental native | Native (no sandbox) |

## Latest Updates (2026)

### Codex App: Multi-Agent Desktop with Worktree Isolation (Feb 2, 2026)

OpenAI launched the Codex app for macOS (Apple Silicon, macOS 14+), a dedicated desktop application functioning as a "command center for agents." The app supports three thread modes — **Local** (work in current project directory), **Worktree** (Git worktree isolation), and **Cloud** (remote execution in OpenAI-managed containers). The Worktree mode is architecturally significant: each agent gets an isolated Git worktree so multiple agents can work on the same repo in parallel without conflicts. Cloud mode runs in fully isolated containers with network disabled by default, matching the CLI's security posture. Agents can run autonomously for up to 30 minutes before returning completed code.

### GPT-5.3-Codex: First "High" Cybersecurity Rating (Feb 5, 2026)

GPT-5.3-Codex became the first OpenAI model classified as **High capability in the Cybersecurity domain** under their Preparedness Framework. This means the model can potentially "automate end-to-end cyber operations against reasonably hardened targets" or "automate the discovery and exploitation of operationally relevant vulnerabilities." To mitigate this, OpenAI deployed multiple safeguards: requests detected as having elevated cyber risk are automatically routed from GPT-5.3-Codex to GPT-5.2, API access is restricted (no unrestricted API for high-risk applications), and a new "Trusted Access for Cyber" gated program requires security professional vetting. OpenAI also allocated $10 million in API credits for cybersecurity defense applications. This rating directly validates the sandbox architecture's importance — the stronger the model's capabilities, the more critical OS-level sandboxing becomes.

### ReadOnlyAccess Policy and Configurable Read Permissions (CLI 0.100.0, Feb 12, 2026)

A new **ReadOnlyAccess** policy shape was introduced, making sandbox read access explicitly configurable rather than the previous implicit "read everything" default. The `sandboxPolicy` now supports explicit read-access controls through `readOnly` and `workspaceWrite` configurations with optional `readOnlyAccess` options. This partially addresses the longstanding security concern (GitHub Issue #4410) about full filesystem read access enabling secret exfiltration.

### Unified Permissions Flow and Structured Network Approvals (CLI 0.102.0, Feb 17, 2026)

The approval system received a significant overhaul with a **unified permissions flow** that includes clearer permissions history in the TUI, a slash command to grant sandbox read access when directories are blocked, and **structured network approval handling** with richer host/protocol context shown directly in approval prompts. This moves toward solving the "binary network control" limitation — approval prompts now show which specific host and protocol the agent wants to access, giving users more informed decisions even though OS-level enforcement remains all-or-nothing.

### SOCKS5 Proxy with Policy Enforcement (CLI 0.93.0, Jan 29, 2026)

An optional **SOCKS5 proxy listener with policy enforcement and config gating** was added. Combined with WS_PROXY/WSS_PROXY environment variable support for websocket proxying, this enables granular network control at the application layer. Enterprise and admin users can now define network constraints via `requirements.toml`, and Plus/Pro/Business users can enable internet access for specific environments with control over which domains and HTTP methods Codex can access. This is a significant architectural shift from the previous all-or-nothing network model.

### Linux Bubblewrap Promoted to Experimental (CLI 0.100.0, Feb 12, 2026)

The Bubblewrap (bwrap) sandbox pipeline on Linux was promoted from off-by-default to **Experimental status**. The build now always compiles vendored bubblewrap on Linux, removing the `CODEX_BWRAP_ENABLE_FFI` flag. Bubblewrap creates a new mount namespace and can mask sensitive files by mounting empty tmpfs over them — addressing the full-filesystem-read weakness that Landlock alone cannot solve. Windows sandbox capabilities were also promoted in the same release.

### Protected Paths Expanded: .agents/ Directory (2025-2026)

The sandbox now protects the `.agents/` directory as read-only alongside `.git/` and `.codex/`. This prevents the model from modifying its own instruction files (`AGENTS.md` and related configs), closing a potential prompt injection vector where the agent could rewrite its own behavioral guidelines.

### Enterprise Controls Extended to Web Search and Network (CLI 0.99.0, Feb 11, 2026)

Enterprise administrators gained new enforcement capabilities: `requirements.toml` can now restrict web search modes (e.g., `allowed_web_search_modes = ["cached"]`) and define network constraints. Git hardening was also strengthened — destructive or write-capable Git invocations can no longer bypass approval mechanisms (CLI 0.95.0). Session-scoped "allow and remember" auto-approval for repeated tool calls (CLI 0.97.0) balances security with developer productivity.

## Stealable Patterns

1. **arg0 dispatch for multitool binaries** — single binary, multiple personalities via argv[0]. Eliminates the "which helper binary is missing" class of bugs entirely.

2. **Deny-by-default with explicit allowlists** — Seatbelt's approach of omitting permissions to deny them is more maintainable than blocklists. Anything you forget to allow is automatically blocked.

3. **Two-kernel-primitive combo** (Landlock + seccomp) — filesystem isolation and syscall filtering solve orthogonal problems. Together they close gaps neither can close alone.

4. **AF_UNIX exemption pattern** — when blocking network, always exempt local domain sockets. Tools need IPC even when the internet is off.

5. **Environment sanitization on spawn** — never inherit the parent environment wholesale. Clear it and rebuild with only what's needed. Prevents accidental secret leakage.

6. **Serialized policy handoff** — parent serializes SandboxPolicy to JSON, child parses it. Clean separation of "what to enforce" from "how to enforce it."

7. **.git as protected zone** — even when writes are allowed, the VCS directory stays read-only. The model should never be able to rewrite history.

8. **Enterprise requirements as non-overridable ceiling** — user config can be permissive within the bounds of admin requirements, but never exceed them. Layered config with strict precedence.

9. **Output capping** (10 KiB / 256 lines) — sandboxed processes shouldn't be able to flood the context window. Truncate aggressively.

10. **Graduated escalation** — sandbox failure triggers approval prompt for unsandboxed retry, not automatic escalation. Human stays in the loop at every privilege boundary.

## References

### Official Documentation

- [Codex Security](https://developers.openai.com/codex/security/) — Official security model overview: sandbox modes, approval policies, network controls, enterprise enforcement
- [Codex CLI Reference](https://developers.openai.com/codex/cli/reference/) — Command-line options including --sandbox, --yolo, --add-dir
- [Codex Advanced Configuration](https://developers.openai.com/codex/config-advanced/) — writable_roots, network_access, tmpdir exclusion
- [Codex Configuration Reference](https://developers.openai.com/codex/config-reference/) — All config keys: sandbox_mode, approval_policy, sandbox_workspace_write table
- [Codex Sample Configuration](https://developers.openai.com/codex/config-sample/) — Complete config.toml example with all sandbox settings
- [Codex Config Basics](https://developers.openai.com/codex/config-basic/) — Default sandbox selection logic based on VCS detection
- [Codex Windows](https://developers.openai.com/codex/windows/) — WSL recommendation, native Windows sandbox, AppContainer details
- [Codex CLI Features](https://developers.openai.com/codex/cli/features/) — Feature overview including sandbox subcommands
- [Codex Changelog](https://developers.openai.com/codex/changelog/) — Version history with sandbox fixes and improvements

### Source Code

- [codex-rs/ README](https://github.com/openai/codex/blob/main/codex-rs/README.md) — Cargo workspace structure: core/, linux-sandbox/, windows-sandbox-rs/, execpolicy/, process-hardening/
- [codex-rs/ directory](https://github.com/openai/codex/tree/main/codex-rs) — Root of Rust implementation
- [sandbox.md](https://github.com/openai/codex/blob/main/docs/sandbox.md) — Points to official security docs
- [windows_sandbox_security.md](https://github.com/openai/codex/blob/main/docs/windows_sandbox_security.md) — Windows sandbox technical details: restricted tokens, escape vectors, PATH injection defense

### Deep Analysis Articles

- [A deep dive on agent sandboxes — Pierce Freeman](https://pierce.dev/notes/a-deep-dive-on-agent-sandboxes) — Detailed analysis of Codex sandbox: Seatbelt profile generation, Landlock rules, seccomp syscall lists, command safety assessment, security limitations
- [Breaking Out of the Codex Sandbox — Vincent Schmalbach](https://www.vincentschmalbach.com/breaking-out-of-the-codex-sandbox-while-keeping-approval-controls/) — danger-full-access + untrusted combination for Claude Code-style workflow
- [How Codex CLI Flags Actually Work — Vincent Schmalbach](https://www.vincentschmalbach.com/how-codex-cli-flags-actually-work-full-auto-sandbox-and-bypass/) — Full-auto, sandbox, and bypass flag interactions
- [How to Effectively Bypass the Codex Sandbox — APIdog](https://apidog.com/blog/bypass-codex-sandbox/) — Various methods to bypass sandbox for development

### DeepWiki Analysis

- [Sandboxing and Security Policies](https://deepwiki.com/openai/codex/6.3-configuration-management) — Configuration management and sandbox integration
- [Sandboxing and Security Policies (6.4)](https://deepwiki.com/openai/codex/6.4-sandboxing-and-security-policies) — Security policy evaluation flow
- [Sandbox Implementations](https://deepwiki.com/yulin0629/codex/4.4-sandbox-implementations) — arg0 dispatch, platform implementations, Starlark integration
- [Sandbox Implementation (zkbkb fork)](https://deepwiki.com/zkbkb/codex/4.2-sandbox-implementation) — Seatbelt command construction, Landlock rules, seccomp filter details, output capping
- [CLI Entry Points and Dispatch](https://deepwiki.com/openai/codex/4.3-cli-entry-points-and-dispatch) — MultitoolCli structure, arg0 dispatch, process flow

### Zread Analysis

- [Apple Seatbelt Implementation](https://zread.ai/openai/codex/13-apple-seatbelt-implementation) — macOS sandbox capabilities, policy modes, configuration
- [Linux Landlock and seccomp](https://zread.ai/openai/codex/14-linux-landlock-and-seccomp) — Linux sandbox architecture overview
- [Execpolicy System](https://zread.ai/openai/codex/16-execpolicy-system) — Starlark-based command policy engine, pattern matching, three-tier decisions

### Security Advisories

- [GHSA-w5fx-fh39-j5rw / CVE-2025-59532](https://github.com/openai/codex/security/advisories/GHSA-w5fx-fh39-j5rw) — Sandbox bypass via path configuration bug, CVSS 8.6, model-generated cwd treated as writable root
- [GitLab Advisory for CVE-2025-59532](https://advisories.gitlab.com/pkg/npm/@openai/codex/CVE-2025-59532/) — Same vulnerability in GitLab advisory database

### GitHub Issues

- [#4410 — Do not allow Codex to read whole filesystem by default](https://github.com/openai/codex/issues/4410) — Security concern: read-only mode grants full filesystem read, data exfiltration risk
- [#10390 — network_access = true silently ignored by seatbelt](https://github.com/openai/codex/issues/10390) — macOS bug: Seatbelt unconditionally sets CODEX_SANDBOX_NETWORK_DISABLED=1
- [#7071 — Cannot commit because .git is read-only](https://github.com/openai/codex/issues/7071) — .git directory protection prevents git commit in sandbox
- [#1039 — WSL seccomp/landlock not supported](https://github.com/openai/codex/issues/1039) — Sandboxing fails in some WSL environments
- [#5041 — VS Code blocks network even with danger-full-access](https://github.com/openai/codex/issues/5041) — Network restrictions persist despite full-access config
- [#7837 — Persistent sandbox bypassing](https://github.com/openai/codex/issues/7837) — Reported sandbox bypass attempts
- [#6807 — Expand Seatbelt to restrict to localhost](https://github.com/openai/codex/issues/6807) — Request for localhost-only network policy
- [#11210 — Seatbelt blocks os.cpus()](https://github.com/openai/codex/issues/11210) — Missing mach-host permission breaks yarn/npm
- [#1124 — CODEX_UNSAFE_ALLOW_NO_SANDBOX should move up](https://github.com/openai/codex/issues/1124) — Environment variable usability
- [#4725 — Codex fails on AWS Lambda](https://github.com/openai/codex/issues/4725) — Sandbox not supported in Lambda environment
- [#6828 — Document Landlock requirement](https://github.com/openai/codex/issues/6828) — Kernel >= 5.13 requirement for Linux sandbox

### GitHub PRs & Discussions

- [PR #4905 — Windows Sandbox Alpha](https://github.com/openai/codex/pull/4905) — Restricted token implementation, AppContainer, ACL management, network lockdown
- [PR #3987 — Seatbelt policy for Java on macOS](https://github.com/openai/codex/pull/3987) — Seatbelt profile adjustments for JVM
- [PR #5536 — Model summary and risk assessment for commands](https://github.com/openai/codex/pull/5536) — Experimental sandbox_command_assessment feature
- [Discussion #1174 — Codex CLI Going Native](https://github.com/openai/codex/discussions/1174) — Transition from TypeScript to Rust, native sandbox benefits
- [Discussion #1260 — Auto-approved commands via execpolicy](https://github.com/openai/codex/issues/1260) — Configurable trusted command list

### System Cards

- [GPT-5-Codex System Card (Sep 2025)](https://cdn.openai.com/pdf/97cc5669-7a25-4e63-b15f-5fd5bdc4d149/gpt-5-codex-system-card.pdf) — Initial security evaluation
- [GPT-5.1-Codex-Max System Card (Nov 2025)](https://cdn.openai.com/pdf/2a7d98b1-57e5-4147-8d0e-683894d782ae/5p1_codex_max_card_03.pdf) — Extended capability assessment
- [GPT-5.2-Codex System Card (Dec 2025)](https://cdn.openai.com/pdf/ac7c37ae-7f4c-4442-b741-2eabdeaf77e0/oai_5_2_Codex.pdf) — 79% network attack success rate
- [GPT-5.3-Codex System Card (Feb 2026)](https://cdn.openai.com/pdf/23eca107-a9b1-4d2c-b156-7deb4fbc697c/GPT-5-3-Codex-System-Card-02.pdf) — First High cybersecurity capability rating

### Community & Misc

- [Hacker News — sandbox-exec deprecation](https://news.ycombinator.com/item?id=44283454) — Discussion on Seatbelt deprecation status and implications
- [SmartScope — Codex CLI Guide](https://smartscope.blog/en/generative-ai/chatgpt/openai-codex-cli-comprehensive-guide/) — Comprehensive setup guide including sandbox configuration
- [SmartScope — Codex Approval Modes](https://smartscope.blog/en/generative-ai/chatgpt/codex-cli-approval-modes-no-approval/) — Approval mode configuration walkthrough
- [SmartScope — Fix Network Restrictions](https://smartscope.blog/en/generative-ai/chatgpt/codex-network-restrictions-solution/) — Network access troubleshooting
- [Enterprise Governance — Developer Toolkit](https://developertoolkit.ai/en/codex/advanced-techniques/enterprise-governance/) — Enterprise admin enforcement patterns
- [Bubblewrap (bwrap) — GitHub](https://github.com/containers/bubblewrap) — Low-level unprivileged sandboxing tool, vendored in Codex Linux build

### 2026 Update Sources

- [Introducing the Codex app — OpenAI](https://openai.com/index/introducing-the-codex-app/) — Codex macOS app launch, multi-agent worktree isolation, cloud containers
- [OpenAI launches new macOS app for agentic coding — TechCrunch](https://techcrunch.com/2026/02/02/openai-launches-new-macos-app-for-agentic-coding/) — Codex app coverage with desktop architecture details
- [GPT-5.3-Codex System Card — OpenAI](https://openai.com/index/gpt-5-3-codex-system-card/) — First High cybersecurity capability rating, safeguards
- [Introducing GPT-5.3-Codex — OpenAI](https://openai.com/index/introducing-gpt-5-3-codex/) — Model launch announcement
- [OpenAI's new model leaps ahead but raises cybersecurity risks — Fortune](https://fortune.com/2026/02/05/openai-gpt-5-3-codex-warns-unprecedented-cybersecurity-risks/) — High cybersecurity rating analysis, $10M defense credits
- [Introducing Trusted Access for Cyber — OpenAI](https://openai.com/index/trusted-access-for-cyber/) — Gated access program for security researchers
- [Codex Changelog — OpenAI Developers](https://developers.openai.com/codex/changelog/) — CLI 0.93–0.104 release notes with sandbox/permissions updates
- [Codex Release Notes — Releasebot](https://releasebot.io/updates/openai/codex) — Aggregated release notes for February 2026
- [Codex App Features — OpenAI Developers](https://developers.openai.com/codex/app/features/) — Thread modes (Local, Worktree, Cloud), agent isolation
- [AGENTS.md Guide — OpenAI Developers](https://developers.openai.com/codex/guides/agents-md/) — Custom instructions with protected .agents/ directory
- [SOCKS5 proxy commit — openai/codex](https://github.com/openai/codex/actions/runs/21333938828) — SOCKS5 proxy with policy enforcement implementation
