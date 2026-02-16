# References

## Official Documentation

- [Codex Security](https://developers.openai.com/codex/security/) — Official security model overview: sandbox modes, approval policies, network controls, enterprise enforcement
- [Codex CLI Reference](https://developers.openai.com/codex/cli/reference/) — Command-line options including --sandbox, --yolo, --add-dir
- [Codex Advanced Configuration](https://developers.openai.com/codex/config-advanced/) — writable_roots, network_access, tmpdir exclusion
- [Codex Configuration Reference](https://developers.openai.com/codex/config-reference/) — All config keys: sandbox_mode, approval_policy, sandbox_workspace_write table
- [Codex Sample Configuration](https://developers.openai.com/codex/config-sample/) — Complete config.toml example with all sandbox settings
- [Codex Config Basics](https://developers.openai.com/codex/config-basic/) — Default sandbox selection logic based on VCS detection
- [Codex Windows](https://developers.openai.com/codex/windows/) — WSL recommendation, native Windows sandbox, AppContainer details
- [Codex CLI Features](https://developers.openai.com/codex/cli/features/) — Feature overview including sandbox subcommands
- [Codex Changelog](https://developers.openai.com/codex/changelog/) — Version history with sandbox fixes and improvements

## Source Code

- [codex-rs/ README](https://github.com/openai/codex/blob/main/codex-rs/README.md) — Cargo workspace structure: core/, linux-sandbox/, windows-sandbox-rs/, execpolicy/, process-hardening/
- [codex-rs/ directory](https://github.com/openai/codex/tree/main/codex-rs) — Root of Rust implementation
- [sandbox.md](https://github.com/openai/codex/blob/main/docs/sandbox.md) — Points to official security docs
- [windows_sandbox_security.md](https://github.com/openai/codex/blob/main/docs/windows_sandbox_security.md) — Windows sandbox technical details: restricted tokens, escape vectors, PATH injection defense

## Deep Analysis Articles

- [A deep dive on agent sandboxes — Pierce Freeman](https://pierce.dev/notes/a-deep-dive-on-agent-sandboxes) — Detailed analysis of Codex sandbox: Seatbelt profile generation, Landlock rules, seccomp syscall lists, command safety assessment, security limitations
- [Breaking Out of the Codex Sandbox — Vincent Schmalbach](https://www.vincentschmalbach.com/breaking-out-of-the-codex-sandbox-while-keeping-approval-controls/) — danger-full-access + untrusted combination for Claude Code-style workflow
- [How Codex CLI Flags Actually Work — Vincent Schmalbach](https://www.vincentschmalbach.com/how-codex-cli-flags-actually-work-full-auto-sandbox-and-bypass/) — Full-auto, sandbox, and bypass flag interactions
- [How to Effectively Bypass the Codex Sandbox — APIdog](https://apidog.com/blog/bypass-codex-sandbox/) — Various methods to bypass sandbox for development

## DeepWiki Analysis

- [Sandboxing and Security Policies](https://deepwiki.com/openai/codex/6.3-configuration-management) — Configuration management and sandbox integration
- [Sandboxing and Security Policies (6.4)](https://deepwiki.com/openai/codex/6.4-sandboxing-and-security-policies) — Security policy evaluation flow
- [Sandbox Implementations](https://deepwiki.com/yulin0629/codex/4.4-sandbox-implementations) — arg0 dispatch, platform implementations, Starlark integration
- [Sandbox Implementation (zkbkb fork)](https://deepwiki.com/zkbkb/codex/4.2-sandbox-implementation) — Seatbelt command construction, Landlock rules, seccomp filter details, output capping
- [CLI Entry Points and Dispatch](https://deepwiki.com/openai/codex/4.3-cli-entry-points-and-dispatch) — MultitoolCli structure, arg0 dispatch, process flow

## Zread Analysis

- [Apple Seatbelt Implementation](https://zread.ai/openai/codex/13-apple-seatbelt-implementation) — macOS sandbox capabilities, policy modes, configuration
- [Linux Landlock and seccomp](https://zread.ai/openai/codex/14-linux-landlock-and-seccomp) — Linux sandbox architecture overview
- [Execpolicy System](https://zread.ai/openai/codex/16-execpolicy-system) — Starlark-based command policy engine, pattern matching, three-tier decisions

## Security Advisories

- [GHSA-w5fx-fh39-j5rw / CVE-2025-59532](https://github.com/openai/codex/security/advisories/GHSA-w5fx-fh39-j5rw) — Sandbox bypass via path configuration bug, CVSS 8.6, model-generated cwd treated as writable root
- [GitLab Advisory for CVE-2025-59532](https://advisories.gitlab.com/pkg/npm/@openai/codex/CVE-2025-59532/) — Same vulnerability in GitLab advisory database

## GitHub Issues

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

## GitHub PRs & Discussions

- [PR #4905 — Windows Sandbox Alpha](https://github.com/openai/codex/pull/4905) — Restricted token implementation, AppContainer, ACL management, network lockdown
- [PR #3987 — Seatbelt policy for Java on macOS](https://github.com/openai/codex/pull/3987) — Seatbelt profile adjustments for JVM
- [PR #5536 — Model summary and risk assessment for commands](https://github.com/openai/codex/pull/5536) — Experimental sandbox_command_assessment feature
- [Discussion #1174 — Codex CLI Going Native](https://github.com/openai/codex/discussions/1174) — Transition from TypeScript to Rust, native sandbox benefits
- [Discussion #1260 — Auto-approved commands via execpolicy](https://github.com/openai/codex/issues/1260) — Configurable trusted command list

## System Cards

- [GPT-5-Codex System Card (Sep 2025)](https://cdn.openai.com/pdf/97cc5669-7a25-4e63-b15f-5fd5bdc4d149/gpt-5-codex-system-card.pdf) — Initial security evaluation
- [GPT-5.1-Codex-Max System Card (Nov 2025)](https://cdn.openai.com/pdf/2a7d98b1-57e5-4147-8d0e-683894d782ae/5p1_codex_max_card_03.pdf) — Extended capability assessment
- [GPT-5.2-Codex System Card (Dec 2025)](https://cdn.openai.com/pdf/ac7c37ae-7f4c-4442-b741-2eabdeaf77e0/oai_5_2_Codex.pdf) — 79% network attack success rate
- [GPT-5.3-Codex System Card (Feb 2026)](https://cdn.openai.com/pdf/23eca107-a9b1-4d2c-b156-7deb4fbc697c/GPT-5-3-Codex-System-Card-02.pdf) — First High cybersecurity capability rating

## Community & Misc

- [Hacker News — sandbox-exec deprecation](https://news.ycombinator.com/item?id=44283454) — Discussion on Seatbelt deprecation status and implications
- [SmartScope — Codex CLI Guide](https://smartscope.blog/en/generative-ai/chatgpt/openai-codex-cli-comprehensive-guide/) — Comprehensive setup guide including sandbox configuration
- [SmartScope — Codex Approval Modes](https://smartscope.blog/en/generative-ai/chatgpt/codex-cli-approval-modes-no-approval/) — Approval mode configuration walkthrough
- [SmartScope — Fix Network Restrictions](https://smartscope.blog/en/generative-ai/chatgpt/codex-network-restrictions-solution/) — Network access troubleshooting
- [Enterprise Governance — Developer Toolkit](https://developertoolkit.ai/en/codex/advanced-techniques/enterprise-governance/) — Enterprise admin enforcement patterns
- [Bubblewrap (bwrap) — GitHub](https://github.com/containers/bubblewrap) — Low-level unprivileged sandboxing tool, vendored in Codex Linux build
