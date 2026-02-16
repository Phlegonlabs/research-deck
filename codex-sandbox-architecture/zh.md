# OpenAI Codex CLI — 沙箱与隔离架构深度剖析

## 他们构建了什么

OpenAI Codex CLI 实现了**双层安全系统**，将 OS 级别沙箱与审批策略层相结合。每个模型生成的命令都在平台特定的沙箱内运行（macOS Seatbelt、Linux Landlock+seccomp、Windows Restricted Token），默认禁用网络且写入限制在工作区内。审批策略层决定何时需要人类干预。

整个沙箱实现位于 Rust 代码库（`codex-rs/`）中，作为 Cargo 工作空间，每个平台有专用 crate：

| Crate | 平台 | 机制 |
|-------|------|------|
| `core/src/seatbelt/` | macOS | Apple Seatbelt 通过 `sandbox-exec` |
| `linux-sandbox/` | Linux | Landlock LSM + seccomp-BPF |
| `windows-sandbox-rs/` | Windows | Restricted Token + AppContainer |
| `execpolicy/` | 全平台 | 基于 Starlark 的命令策略引擎 |
| `process-hardening/` | 全平台 | 额外安全加固 |

## 为什么选择这个架构

### 核心问题

AI 编程代理需要 shell 访问才有用，但 shell 访问就是无限制的权力。你不能直接 `os.system(model_output)` 然后祈祷好运。威胁模型：

1. **模型幻觉** — 模型意外生成破坏性命令
2. **提示注入** — 文件/URL 中的恶意内容诱骗模型执行有害操作
3. **数据泄露** — 模型读取密钥后通过网络外泄
4. **文件系统损坏** — 写入项目目录之外

### 为什么是 OS 级别（而非仅应用级别）

应用级沙箱（解析命令、维护黑名单）从根本上可被绕过 — shell 中有无限种方式编码 `rm -rf /`。OS 级沙箱是**强制访问控制**：内核强制执行，无论命令多巧妙。即使模型找到新颖的命令编码方式，内核也会阻止系统调用。

### 为什么需要两层

仅靠沙箱不够，因为许多合法开发任务需要提升权限（安装包、运行访问 localhost 的测试）。审批策略提供了**渐进式提权路径** — 沙箱约束默认行为，审批把关例外情况。

## 工作原理 — 平台深度分析

### 架构：arg0 调度

Codex 使用巧妙的"arg0 调度"模式避免分发独立的沙箱二进制文件。主 `codex` 二进制在启动时检查 `argv[0]`：

```
arg0_dispatch_or_else() {
    if 二进制名匹配 "*-linux-sandbox" → 进入沙箱模式
    if 二进制名匹配 "*-apply-patch"  → 进入补丁模式
    else                              → 正常 CLI 调度
}
```

需要沙箱时，core 用覆盖的 `argv[0]`（如 `codex-linux-sandbox`）重新执行自身，触发沙箱代码路径。这消除了独立的辅助二进制文件，同时保持权限分离。模式匹配基于前缀，所以 `codex-x86_64-unknown-linux-musl-linux-sandbox` 等平台特定二进制也能正确工作。

这些处理程序是**终端操作** — 它们直接调用 `std::process::exit()` 而不返回主 CLI 调度器。

### 沙箱模式

三种模式控制沙箱进程可以做什么：

| 模式 | 文件系统 | 网络 | 用例 |
|------|---------|------|------|
| `read-only` | 读取任意位置，不可写入 | 阻断 | 安全浏览/规划 |
| `workspace-write` | 读取任意位置，写入工作区 + tmp | 阻断 | 默认开发模式 |
| `danger-full-access` | 无限制 | 无限制 | 仅用于预隔离环境 |

**关键设计决策**：即使 `read-only` 模式也授予**完整文件系统读取权限** — 整个磁盘，而非仅项目目录。因为限制只读到 `cwd` 会破坏一切（无法读取 `/bin/cat`、系统库等）。这已被标记为安全隐患（GitHub Issue #4410），因为它允许从文件系统任意位置泄露密钥。OpenAI 的立场：限制读取对大多数用户来说是"致命的默认行为"。

**默认选择逻辑**：Codex 检测版本控制状态：
- Git 仓库 → `workspace-write` + `on-request` 审批（称为"Auto"模式）
- 无 VCS → `read-only` 模式

### macOS：Apple Seatbelt

**技术**：macOS Seatbelt 是内核级强制访问控制框架。命令通过 `/usr/bin/sandbox-exec -p <profile>` 包装执行。配置使用 SBPL（Sandbox Profile Language），一种基于 Scheme 的 DSL。

**实现细节**：

1. `core/src/seatbelt/` 中的 `create_seatbelt_command()` 动态组装配置
2. 硬编码 `/usr/bin/sandbox-exec`（防止 PATH 注入攻击）
3. 通过检查二进制是否存在来验证沙箱可用性
4. 组合基础策略（`seatbelt_base_policy.sbpl`）与运行时生成的权限

**配置结构**（Seatbelt 默认拒绝）：

```scheme
(version 1)
(deny default)                          ; 默认拒绝一切

;; ReadOnly 模式：
(allow file-read*)                      ; 允许所有文件读取

;; WorkspaceWrite 模式额外添加：
(allow file-write* (subpath "/path/to/workspace"))
(allow file-write* (subpath "/tmp"))

;; 网络（启用时）：
(allow network-outbound)
(allow network-inbound)
(allow system-socket)
;; 禁用时：直接省略（默认拒绝处理）
```

**可写根目录枚举**：
- `.git/` 目录明确标记为只读（防止 git 历史篡改）
- `.codex/` 即使工作区可写也保持只读
- 配置中的工作区根目录添加为 `(allow file-write* (subpath ...))`

**网络策略**：从 `seatbelt_network_policy.sbpl` 加载。二元允许/拒绝 — 要么完全网络要么无网络。**已知 Bug**：config.toml 中的 `network_access = true` 被静默忽略；Seatbelt 配置无条件设置 `CODEX_SANDBOX_NETWORK_DISABLED=1`（Issue #10390）。无法表达细粒度规则如"仅 HTTPS 到 api.openai.com"。

**阻断的能力**：文件写入、网络、Apple Events、无障碍服务。

**弃用担忧**：`sandbox-exec` 在 macOS Sierra（2016）前后被标记弃用，但 Apple 仍保持向后兼容。Seatbelt 本身（内核框架）并未弃用。矛盾在于：Apple 不鼓励直接使用但保持其对 Chrome 等关键应用的兼容。OpenAI 继续使用它因为 macOS 上没有更好的编程沙箱替代方案。

**已知限制**：研究人员发现 Seatbelt 配置在不当配置时"未能正确阻止对用户主目录 dotfiles 和 `~/Library` 的访问"。需要深入的 macOS 内部知识，且这些知识随 OS 版本变化。

### Linux：Landlock + seccomp-BPF

**两种互补的内核机制**：
- **Landlock LSM**（Linux Security Module，内核 >= 5.13）：文件系统访问控制
- **seccomp-BPF**（Berkeley Packet Filter）：系统调用过滤

**实现**：`codex-linux-sandbox` crate 作为独立进程运行（通过 arg0 调度）。父进程将 `SandboxPolicy` 序列化为 JSON 并调用辅助程序。辅助程序：
1. 解析序列化的策略
2. 应用 Landlock 限制
3. 安装 seccomp 过滤器
4. 调用 `execvp` 替换自身为目标命令

**Landlock 文件系统规则**：

```
默认：
  - 读取权限：整个文件系统（与 macOS 相同理由）
  - 写入权限：/dev/null（始终允许）

WorkspaceWrite 额外添加：
  - 写入权限：cwd（工作区根目录）
  - 写入权限：$TMPDIR、/tmp（可配置）
  - 写入权限：配置中的 writable_roots[]
  - 不可写：工作区内的 .git/、.codex/
```

规则在 `execvp` 之前应用，确保沙箱化的子进程无法逃逸。

**seccomp-BPF 系统调用过滤**（网络隔离）：

网络禁用时阻断的系统调用：
```
SYS_connect    — 阻止出站连接
SYS_accept     — 阻止入站连接
SYS_bind       — 阻止端口绑定
SYS_listen     — 阻止监听
SYS_sendto     — 阻止发送数据
SYS_sendmsg    — 阻止发送消息
SYS_sendmmsg   — 阻止批量发送
io_uring       — 阻止 io_uring 网络绕过（后期添加）
```

**明确不阻断的**：
```
SYS_recvfrom   — cargo clippy 等工具需要，
                  它们使用 socketpair 进行子进程管理
socket()       — 仅允许 AF_UNIX（本地 IPC）；
                  通过检查第一个参数（domain）
                  拒绝 AF_INET/AF_INET6
```

过滤器使用**默认允许策略加特定拒绝规则**处理网络系统调用，检查 `socket()` 的 domain 参数以允许 `AF_UNIX` 同时拒绝网络套接字。这比完全阻断 `socket()` 更精确。

**架构支持**：仅 x86_64 和 aarch64。其他架构会触发 panic。

**环境变量**：
- `CODEX_SANDBOX_NETWORK_DISABLED=1` — 通知子进程网络已禁用
- `CODEX_UNSAFE_ALLOW_NO_SANDBOX` — 沙箱失败时的紧急回退（如在容器中）
- `prctl(PR_SET_PDEATHSIG)` — 注册父进程死亡信号处理器防止孤儿进程

**替代管道**：Bubblewrap（bwrap）可作为替代方案，通过 `features.use_linux_sandbox_bwrap = true` 启用。Bubblewrap 创建新的 mount namespace 并可通过在敏感文件上挂载空 tmpfs 来隐藏它们。Linux 沙箱构建过程始终编译内置的 bubblewrap。

### Windows：Restricted Token + AppContainer

**状态**：实验性。使用 `codex-windows-sandbox` crate（库，非独立二进制）。

**机制**：
1. 通过 `CreateRestrictedToken()` Windows API 创建受限令牌
2. 从 AppContainer 配置文件派生令牌
3. 通过安全标识符授予选择性文件系统能力
4. 配置 Job Objects 进行进程隔离

**文件系统控制**：
- 除声明的工作区根目录 + `%TEMP%` 外，所有位置阻止写入
- ACL 管理为特定目录授予 AppContainer 权限
- 主动拒绝常见逃逸向量：
  - **备用数据流**（NTFS ADS）
  - **UNC 路径**（`\\server\share`）
  - **设备句柄**（`\\.\PhysicalDrive0`）

**网络锁定**：覆盖代理变量并注入桩可执行文件。

**PATH 注入防御**：CLI 在主机 PATH 前注入桩可执行文件（如包装 `ssh`），在危险工具离开沙箱前拦截它们。

**关键限制**：无法阻止 `Everyone` SID 已有写入权限的目录（全局可写文件夹）中的写入。Codex 会扫描并建议移除此类权限。

**浏览器逃逸**：只读模式仍可通过 `Start-Process 'https://...'` 启动默认浏览器，因为 Explorer 在沙箱外处理 `ShellExecute`。

### Windows + WSL（推荐路径）

原生 Windows 沙箱是实验性的。推荐方式是 WSL2：
- 提供 Linux shell + Unix 语义，匹配模型训练数据
- 使用完整的 Linux Landlock + seccomp 沙箱
- 性能：将仓库放在 Linux 主目录（`~/code/...`），而非 `/mnt/c/`（避免较慢的 I/O 和符号链接/权限问题）

## 网络隔离

**默认**：所有平台禁用网络。这是最强的安全保证 — 即使模型读取了密钥也能防止数据泄露。

**各平台实现**：

| 平台 | 机制 | 粒度 |
|------|------|------|
| macOS | Seatbelt 配置规则 | 二元：全有或全无 |
| Linux | seccomp-BPF 系统调用过滤 | 系统调用级别，AF_UNIX 豁免 |
| Windows | 代理覆盖 + 桩可执行文件 | 应用级别 |

**根本限制**：所有平台都无法表达**应用层**网络策略，如"仅 HTTPS 到 api.openai.com"或"仅允许 localhost:3000 但不允许互联网"。网络层面只能全有或全无。

**Web 搜索行为**：网络禁用时，Web 搜索使用"缓存"模式 — OpenAI 维护的索引结果而非实时抓取，减少提示注入暴露。完全网络访问切换为实时结果。

**配置**：
```toml
[sandbox_workspace_write]
network_access = true  # 默认：false
```

## writable_roots 配置

`writable_roots` 设置将写入边界扩展到工作区之外：

```toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
writable_roots = ["/Users/YOU/.pyenv/shims", "/home/you/.cargo/bin"]
network_access = false
exclude_tmpdir_env_var = false  # 默认：false（TMPDIR 可写）
exclude_slash_tmp = false       # 默认：false（/tmp 可写）
```

**CLI 等效方式**：`--add-dir` 标志（可重复）：
```bash
codex --cd apps/frontend --add-dir ../backend --add-dir ../shared
```

**workspace-write 中始终可写的**：
- 当前工作目录（cwd）
- `$TMPDIR`（除非 `exclude_tmpdir_env_var = true`）
- `/tmp`（除非 `exclude_slash_tmp = true`）
- `writable_roots[]` 中的路径

**workspace-write 中始终只读的**：
- `.git/` 目录（防止 git 历史篡改）
- `.codex/` 目录（防止配置自修改）

## 审批策略系统

### 四个审批级别

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `untrusted` | 仅已知安全的只读命令自动运行；其他全部提示 | 新的/不信任的项目 |
| `on-failure` | 在沙箱中自动运行；仅在命令失败时提示 | 迭代开发 |
| `on-request` | 模型决定何时请求（默认） | 正常开发 |
| `never` | 从不提示；在沙箱约束内执行 | 完全自动化 |

### 与沙箱的交互

两层是**独立但互补的**：

```
sandbox_mode = "workspace-write"  →  OS 限制"能"发生什么
approval_policy = "on-request"    →  策略限制"会"发生什么
```

强力组合：`danger-full-access` + `untrusted` = "Codex 可以做任何事，但每个操作都需要明确的人类批准"（模仿 Claude Code 的默认方式）。

### 提权流程

当沙箱执行因权限失败时：
1. 系统检测到失败
2. 提示用户进行非沙箱重试
3. 如果批准，以 `SandboxType::None` 重新执行
4. 会话级别的批准跟踪防止对相同命令模式的重复提示

### Execpolicy：语义命令控制

在 OS 级沙箱之上，`execpolicy/` crate 使用 Starlark（类 Python 配置语言）添加**应用级语义控制**：

```starlark
# 示例规则
rule(
    pattern = ["rm", "-rf", "*"],
    decision = "forbidden",
    justification = "递归强制删除太危险"
)
```

三级决策：`allow` > `prompt` > `forbidden`。当多个规则匹配时，**最严格的**获胜。这能在沙箱边界内捕获语义上危险的操作。

规则文件位于 `~/.codex/rules/` 或 `.codex/rules/`（项目级别）。

### 企业强制执行

**`/etc/codex/requirements.toml`** — 管理员强制，不可覆盖：
```toml
allowed_sandbox_modes = ["read-only", "workspace-write"]
allowed_approval_policies = ["untrusted", "on-failure", "on-request"]
```

这阻止用户选择 `danger-full-access` 或 `never` 审批。

**`/etc/codex/managed_config.toml`** — 起始默认值，用户可在会话中覆盖。

**macOS MDM**：在偏好域 `com.openai.codex` 中使用 Base64 编码的 TOML 载荷，键为 `config_toml_base64` 和 `requirements_toml_base64`。

**云端强制**：使用 ChatGPT Business/Enterprise 登录时，Codex 从 Codex 服务获取管理员强制的要求。

**不信任项目**：将项目标记为不信任会跳过所有项目级 `.codex/` 层（包括 config.toml），仅回退到用户/系统/内置默认值。

## YOLO 模式：--dangerously-bypass-approvals-and-sandbox

`--yolo` 是 `--dangerously-bypass-approvals-and-sandbox` 的别名。它：

1. 设置 `sandbox_mode = "danger-full-access"`（无 OS 沙箱）
2. 设置 `approval_policy = "never"`（无人类提示）
3. 授予完整文件系统 + 网络访问
4. 立即运行每个模型生成的命令

**预期用途**：仅在预隔离环境中（Docker 容器、VM、CI/CD 管道）。

**变化**：Web 搜索从缓存切换到实时结果。所有文件系统读写不受限制。所有网络连接允许。

**名称就是警告**：标志名称本身就是一种安全机制 — 它故意吓人以防止随意使用。

## 安全分析

### 纵深防御层

```
第 1 层：Execpolicy    — 语义命令过滤（Starlark 规则）
第 2 层：审批门控      — 危险操作的人在回路
第 3 层：OS 沙箱       — 内核强制的强制访问控制
第 4 层：Git VCS       — 检测 + .git 只读保护
第 5 层：企业          — requirements.toml 强制执行
```

### 已知漏洞

**CVE-2025-59532（CVSS 8.6）** — 通过路径配置 Bug 绕过沙箱
- 影响：CLI 0.2.0–0.38.0，IDE Extension 到 0.4.11
- Bug：模型生成的 `cwd` 被当作沙箱可写根目录，允许在用户会话目录之外任意写入文件
- 修复（v0.39.0）：规范化并验证沙箱边界基于会话启动位置，拒绝模型生成的路径
- 漏洞利用期间网络限制保持完好

### 结构性弱点

| 弱点 | 影响 | 缓解措施 |
|------|------|---------|
| read-only 模式下完整文件系统读取 | 密钥泄露（SSH 密钥、.env、凭据） | 网络阻断防止外泄；敏感环境推荐 Docker |
| macOS Seatbelt 已"弃用"（sandbox-exec） | 未来 Apple 可能破坏兼容性 | Apple 保持向后兼容；无更好替代方案 |
| Seatbelt 无法正确保护 dotfiles/~/Library | 配置/凭据暴露 | 需要仔细的配置工程 |
| 二元网络控制（全有或全无） | 无按域过滤 | 无法限制"仅 localhost"或"仅 api.openai.com" |
| Windows 无法阻断全局可写目录 | 通过 Everyone SID 文件夹写入逃逸 | Codex 扫描并警告；建议修复权限 |
| Windows 浏览器逃逸 | 可通过 ShellExecute 启动 URL | 固有 OS 限制 |
| seccomp 仅支持 x86_64/aarch64 | 不支持 ARM32/RISC-V | 不支持的架构会 panic |
| Landlock 需要内核 >= 5.13 | 旧发行版/WSL 可能不支持 | 回退到 CODEX_UNSAFE_ALLOW_NO_SANDBOX |
| io_uring 最初未被阻断 | 通过 io_uring 系统调用绕过网络 | 在后续版本中修复 |
| macOS network_access=true 在配置中被静默忽略 | 尽管配置了但网络仍被阻断 | 必须使用 CLI 标志作为变通 |
| 环境清理缺口 | 继承的环境变量可能泄露信息 | `spawn_child_async` 清除环境并重建 |

### 设计优秀之处

1. **arg0 调度**消除独立二进制 — 单一可执行文件，更少攻击面
2. **.git 只读**即使在 workspace-write 模式也防止历史篡改
3. **子进程生成时环境清理** — 仅传递预期的变量
4. **父进程死亡信号**（`PR_SET_PDEATHSIG`）防止孤儿沙箱进程
5. **Seatbelt 默认拒绝** — 省略权限自动阻断
6. **seccomp 中 AF_UNIX 豁免** — 允许本地 IPC 而不破坏工具链
7. **超时强制**（默认 10s） — 防止失控的沙箱进程
8. **企业要求作为不可覆盖的上限** — 用户配置在管理员要求范围内灵活配置但不能超越

## 进程执行流程

从模型输出到沙箱执行的完整路径：

```
1. 模型生成函数调用（shell 命令）
2. ToolRouter 评估操作类型
3. ExecPolicyManager 检查 Starlark 规则 → allow/prompt/forbid
4. ToolOrchestrator 检查 approval_policy → 需要时提示
5. 用户批准（或根据策略自动批准）
6. 根据配置选择沙箱模式
7. spawn_child_async：
   a. 清除环境，仅用预期变量重建
   b. 如适用设置 CODEX_SANDBOX_NETWORK_DISABLED
   c. 平台调度：
      - macOS: /usr/bin/sandbox-exec -p <generated_profile> <command>
      - Linux: 重新执行为 codex-linux-sandbox → 应用 landlock → 应用 seccomp → execvp
      - Windows: CreateRestrictedToken → job object → 执行
8. 输出上限 10 KiB / 256 行（read_capped）
9. 检查退出码：命令失败 vs 沙箱违规
10. 如果沙箱违规 → 提供带审批的非沙箱重试
11. 结果作为 OutputDelta + ExecCommandEnd 事件流式返回
```

## 与 Claude Code 的对比

| 方面 | Codex CLI | Claude Code |
|------|-----------|-------------|
| 沙箱技术 | Seatbelt / Landlock+seccomp / Restricted Token | 权限拒绝规则 + devcontainers |
| 默认网络 | 阻断 | 允许（带审批提示） |
| 文件系统读取 | 完整磁盘（所有模式） | 完整磁盘 |
| 审批模型 | 4 级策略系统 | 逐命令审批提示 |
| 企业控制 | requirements.toml + MDM | 托管设置 |
| 执行策略 | 基于 Starlark 的规则引擎 | 基于 Hook 的拦截器 |
| YOLO 模式 | `--yolo` 标志 | `--dangerously-skip-permissions` |
| Windows | 推荐 WSL + 实验性原生 | 原生（无沙箱） |

## 可偷取的模式

1. **多工具二进制的 arg0 调度** — 单一二进制，通过 argv[0] 实现多重人格。完全消除了"缺少哪个辅助二进制"这类 Bug。

2. **默认拒绝配合显式白名单** — Seatbelt 通过省略权限来拒绝的方式比黑名单更可维护。你忘记允许的任何东西都自动被阻断。

3. **双内核原语组合**（Landlock + seccomp） — 文件系统隔离和系统调用过滤解决正交问题。合在一起填补了单独使用时的空白。

4. **AF_UNIX 豁免模式** — 阻断网络时始终豁免本地域套接字。工具即使在断网时也需要 IPC。

5. **生成时环境清理** — 永远不要整体继承父环境。清除它并仅用需要的内容重建。防止意外的密钥泄漏。

6. **序列化策略交接** — 父进程将 SandboxPolicy 序列化为 JSON，子进程解析。"强制什么"与"如何强制"的清晰分离。

7. **.git 作为保护区** — 即使允许写入，VCS 目录保持只读。模型永远不应能改写历史。

8. **企业要求作为不可覆盖的上限** — 用户配置在管理员要求范围内可以宽松，但永远不能超越。分层配置配合严格优先级。

9. **输出上限**（10 KiB / 256 行） — 沙箱进程不应能淹没上下文窗口。积极截断。

10. **渐进式提权** — 沙箱失败触发审批提示进行非沙箱重试，而非自动提权。人类在每个权限边界保持在回路中。
