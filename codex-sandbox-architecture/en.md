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
