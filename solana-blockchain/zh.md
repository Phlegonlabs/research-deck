# Solana 区块链：完整技术与生态深度分析

> 最后更新：2026-02-21 | SOL 价格：~$82 | 市值：~$480亿 | 验证者：~800 | TVL：~$115亿

---

## 目录

1. [架构深度解析](#1-架构深度解析)
2. [性能与现实检验](#2-性能与现实检验)
3. [开发者技术栈](#3-开发者技术栈)
4. [DeFi 生态系统](#4-defi-生态系统)
5. [NFT 与消费者应用](#5-nft-与消费者应用)
6. [Meme 币与 Pump.fun](#6-meme-币与-pumpfun)
7. [SOL 代币经济学](#7-sol-代币经济学)
8. [治理与组织结构](#8-治理与组织结构)
9. [2025-2026 重大进展](#9-2025-2026-重大进展)
10. [诚实评估：多空论点](#10-诚实评估多空论点)
11. [最新動態 (2026)](#最新動態-2026)
12. [可复用模式](#11-可复用模式)
13. [References](#references)

---

## 1. 架构深度解析

Solana 的核心论点：**将单一分片优化到物理极限**，而非将工作分配到多条链上。八项创新作为一个集成系统协同运作。

### 1.1 历史证明 (PoH) — 时钟

**是什么：** 一条持续运行的 SHA-256 哈希链，在共识*之前*就创建了事件的可验证排序。

**实际工作原理：**

```
hash(n) = SHA-256(hash(n-1))
hash(n+1) = SHA-256(hash(n))
...持续顺序计算
```

| 概念 | 详情 |
|------|------|
| 一个 tick | 12,800 次顺序 SHA-256 哈希 |
| 一个 slot | 64 个 tick = ~400ms |
| 一个 epoch | 432,000 个 slot = ~2-3 天 |
| 生成 | 单个 CPU 核心（本质上是顺序的） |
| 验证 | 可在 GPU 核心间并行化（4000 核心 = 0.25ms） |

**为什么用 SHA-256：** 它是抗原像的哈希函数。你无法在不计算中间每一步的情况下算出 hash(n+100)。这创造了可证明的时间流逝——一种类 VDF（可验证延迟函数）机制。

**它不是什么：** PoH 不是共识机制。它是一个**时钟**，在共识开始前提供同步时间源。交易通过在特定点插入哈希链来获得时间戳。任何验证者都可以通过重放哈希链来验证排序。

**关键洞察：** 传统 BFT 协议浪费大量带宽来确定事件*何时*发生。PoH 通过在投票前给每个验证者提供相同的时钟来消除这个问题。

### 1.2 Tower BFT — 共识

**是什么：** 一种修改版的 PBFT（实用拜占庭容错）算法，使用 PoH 作为时钟源。

**为什么重要：**

| 属性 | 传统 PBFT | Tower BFT |
|------|----------|-----------|
| 消息复杂度 | O(n^2) | O(n) |
| 确认轮数 | 多轮 | 可能单轮 |
| 时钟同步 | 需要（外部） | 通过 PoH 内建 |
| 超时计算 | 需要 P2P 通信 | 仅需本地计算 |

**投票机制：**
- 验证者为特定 slot 堆叠投票
- 每次投票将栈中所有下方投票的锁定期翻倍
- 为 slot X 投票的验证者必须等待 2^(深度) 个 slot 才能为不同分叉投票
- 这使回滚的代价呈指数增长——深度为 32 的投票需要等待 2^32（~40亿）个 slot

**分叉解决：** 当分叉发生时，验证者计算每个分叉的权益加权锁定，并选择最重的一个。因为 PoH 提供了同步时钟，每个验证者可以独立计算其他所有验证者的超时——无需 P2P 通信。

### 1.3 Turbine — 区块传播

**是什么：** 一种受 BitTorrent 启发的区块传播协议，将区块分解为称为 **shred** 的小片段。

```
           Leader
          /  |  \
        L1   L1   L1        (第1层：直接接收 shred)
       / \   |   / \
     L2  L2  L2 L2  L2      (第2层：从 L1 转发)
     ...                     (第N层：指数扇出)
```

**工作原理：**
1. Leader 将区块分解为 shred（原子单位，~1280 字节）
2. Shred 通过验证者的树形层级分发
3. 每个验证者将 shred 转发给一小组对等节点
4. Reed-Solomon 纠删编码允许从部分数据重建
5. 指数扇出意味着 O(log n) 传播时间

**为什么不用 gossip：** 传统 gossip 协议带宽为 O(n * message_size)。Turbine 利用树形结构将其降低到 O(log n)。对于 1000 个验证者，这意味着约 10 跳而非 1000 次广播。

### 1.4 Gulf Stream — 无内存池转发

**是什么：** 一种通过直接将交易转发给即将到来的 leader 来消除传统内存池的协议。

**工作原理：**
1. 每个验证者都知道 leader 调度（从 PoH + 权益权重推导）
2. 客户端和 RPC 节点提前将交易转发给预期的 leader
3. Leader 在其 slot 正式开始前就开始执行交易
4. 当 slot 开始时，leader 已经准备好了部分执行的区块

**为什么没有内存池：** 在传统区块链中，内存池造成问题：
- 跨节点冗余存储
- 从排序可见性中提取 MEV
- 确认时间延迟

Gulf Stream 和 Turbine 是**镜像关系**：Gulf Stream 将交易路由*到* leader；Turbine 将处理后的区块路由*从* leader 传出。

### 1.5 Sealevel — 并行运行时

**是什么：** Solana 的智能合约运行时，可以同时处理数千个不重叠的交易。

**关键设计决策：** 每笔 Solana 交易必须*预先声明*它将读写哪些账户。这使调度器能够：

```
交易 A：写入 [Alice_余额]
交易 B：写入 [Bob_余额]
交易 C：读取 [Alice_余额, Bob_余额]

A 和 B → 并行（不同的写集）
A 和 C → 顺序（重叠账户）
B 和 C → 顺序（重叠账户）
```

**执行模型：**
1. 按程序 ID 排序交易
2. 按账户访问模式分组（只读 vs 读写）
3. 不重叠的写集 → 不同 CPU 核心同时处理
4. 相同程序 + 多个账户 → SIMD 向量化
5. 只读访问 → 无限并行（无冲突可能）

**与 EVM 对比：** 以太坊顺序执行所有操作，因为交易不预先声明状态访问。Sealevel 的显式账户声明在现代硬件上实现了最高 4000 倍的理论并行性。

### 1.6 流水线 — 交易处理单元 (TPU)

**是什么：** 一个受 CPU 流水线设计启发的 4 阶段硬件流水线。

```
阶段 1：数据获取     (内核空间 - NIC)
    ↓
阶段 2：签名验证     (GPU - 并行验证)
    ↓
阶段 3：银行处理     (CPU - 状态变更)
    ↓
阶段 4：写入         (内核空间 - 磁盘)
```

在任何时刻，50,000 笔交易可以在不同流水线阶段中处理。当 GPU 验证批次 N 的签名时，CPU 处理批次 N-1 的状态变更，NIC 将批次 N-2 写入磁盘。

**两条并行流水线：**
- **TPU（交易处理单元）：** 验证者作为 leader 时使用（创建区块）
- **TVU（交易验证单元）：** 验证来自其他 leader 的区块时使用

### 1.7 Cloudbreak — 账户数据库

**是什么：** 一个为并发读写优化的水平扩展状态数据库。

**架构：**
- 账户数据分布在 RAID 0 SSD 配置上
- 内存映射文件实现快速随机访问
- 顺序写入追加日志
- 并发访问模式与 Sealevel 的并行执行对齐

**为什么不用传统数据库：** 区块链状态需要：
- 每秒数百万次随机读取（账户查找）
- 数千次并发写入（并行执行）
- 快速顺序写入（区块生产）
- Cloudbreak 的设计同时满足这三个需求

### 1.8 归档器 — 分布式账本存储

**是什么：** 使用复制证明存储历史账本数据的节点网络。

验证者只需存储近期数据（滚动窗口）。历史数据使用 Filecoin 复制证明的变体分布在归档器节点上，归档器定期证明它们正在存储分配的账本段。

### 1.9 八项创新如何协同工作

```
客户端发送交易
    │
    ▼
Gulf Stream：转发给即将到来的 leader（无内存池）
    │
    ▼
TPU 流水线：获取 → 验证签名(GPU) → 执行(CPU) → 写入(磁盘)
    │
    ▼
Sealevel：基于声明的账户访问进行并行执行
    │
    ▼
Cloudbreak：并发读写账户数据库
    │
    ▼
PoH：将结果加时间戳到哈希链中
    │
    ▼
Turbine：通过树形层级传播区块 shred
    │
    ▼
Tower BFT：验证者使用 PoH 时钟投票（O(n) 消息）
    │
    ▼
归档器：使用复制证明存储历史数据
```

**系统洞察：** 每项创新解决一个瓶颈，它们被设计为协同工作。PoH 使 Tower BFT 高效；Tower BFT 使 Gulf Stream 的无内存池设计成为可能；Sealevel 使 Cloudbreak 的并发访问成为可能；流水线将硬件利用率联系在一起。移除任何一个部分，系统都会显著退化。

---

## 2. 性能与现实检验

### 2.1 声称 vs 实际 TPS

| 指标 | 数值 | 上下文 |
|------|------|--------|
| 理论最大 TPS | 65,000 | 营销数字，单分片 |
| 压力测试峰值 | 107,540 | Noop 程序调用，非真实交易 |
| 真实世界持续 | 1,000-4,000 | 包含投票交易 |
| 仅用户交易 | 400-800 | 排除验证者投票 |
| 投票交易占比 | ~60-70% | "TPS"的大部分是验证者开销 |
| Firedancer 实验室演示 | 1,000,000+ | 商用硬件，受控测试 |

**诚实的画面：** Solana 的"65K TPS"是从未在真实交易中实现的理论最大值。典型日常的真实用户吞吐量为 400-800 TPS，在高活跃期可突发到数千。链上交易的大部分是验证者投票消息，而非用户发起的操作。

**上下文很重要：** 即使 400-800 真实用户 TPS 也远超以太坊的 15-30 TPS。理论与实际之间的差距存在是因为网络条件、交易复杂性和状态访问模式限制了实际吞吐量。

### 2.2 当前网络统计（2026年2月）

| 指标 | 值 |
|------|-----|
| 活跃验证者 | ~800（从2023年峰值2,500下降） |
| 质押 SOL 总量 | ~3.34亿 SOL（~67%供应量） |
| 质押年化收益 | ~7-8% |
| 出块时间 | ~400ms |
| 最终性 | ~12-13秒（当前），100-150ms（Alpenglow 目标） |
| 中本聪系数 | 20 |
| 自2024年2月以来正常运行时间 | ~99.9%+（无完全中断） |

### 2.3 完整中断时间线

| 日期 | 时长 | 根本原因 | 修复 |
|------|------|----------|------|
| 2021年9月 | 17小时 | 机器人发送 400K+ TPS，共识崩溃 | 手动验证者重启 |
| 2022年1月6-12日 | ~6天 | 高计算交易压垮网络 | 补丁 + 费用市场改进 |
| 2022年1月21-22日 | ~30小时 | 过多重复交易 | 去重逻辑修复 |
| 2022年4月30-5月1日 | ~7小时 | 验证者因分叉积累内存溢出 | 内存管理补丁 |
| 2022年6月1日 | ~4小时 | 共识失败和时钟漂移 | Bug 修复 |
| 2022年末 | ~5小时 | 热备验证者产生重复区块 | 分叉选择逻辑修复 |
| 2023年2月25日 | ~19小时 | 故障验证者广播超大区块，Turbine 去重失败 | Shred 去重修复 |
| 2024年2月 | ~5小时 | LoadedPrograms 函数 Bug | 验证者重启 + 补丁 |

**模式分析：**
- 8次中断中5次由**客户端 Bug**（软件质量）引起
- 2次由**垃圾/拥堵**（设计限制）引起
- 1次由**操作错误**（热备配置错误）引起
- 自2024年2月以来无完全中断（2年以上连续运行）
- 每次中断都导致了具体的代码修复和协议改进

### 2.4 Firedancer — 第二个验证者客户端

| 属性 | 详情 |
|------|------|
| 构建者 | Jump Crypto（现 Jump Trading Group） |
| 语言 | C/C++（vs Agave 的 Rust） |
| 开发时间 | 3年以上 |
| 主网启动 | 2025年12月15日 |
| 启动前测试网运行 | 100+天连续运行，50K+区块 |
| 实验室基准 | 商用硬件 1,000,000+ TPS |
| 主网采用率 | ~21%权益（207个验证者），截至2025年10月 |
| Frankendancer（混合版） | 26%+验证者在运行 |

**为什么 Firedancer 重要：**
1. **客户端多样性：** 单客户端风险是致命的。在 Firedancer 之前，Agave 客户端的 Bug 可能导致整个网络停机。现在两个独立实现减少了相关故障风险。
2. **性能上限：** 用 C/C++ 编写并进行极端底层优化，Firedancer 将硬件利用边界推到了 Rust 难以达到的水平。
3. **全新代码库：** 从头重写让 Jump 能够修复在原始客户端中累积的架构技术债务。

### 2.5 吞吐量对比

| 链 | 真实 TPS | 最终性 | 架构 |
|----|----------|--------|------|
| Solana | 400-4,000 | 12-13秒（当前） | 单体式，并行执行 |
| Ethereum L1 | 15-30 | 12-15分钟 | 模块化，顺序 EVM |
| Arbitrum (L2) | 250-500 | ~1分钟（软） | Optimistic rollup |
| Base (L2) | 100-300 | ~1分钟（软） | OP Stack rollup |
| Aptos | ~100 | ~1秒 | Move VM，并行(Block-STM) |
| Sui | ~5,000-10,000 | ~500ms | 对象中心，Move VM |

**关键上下文：** 原始 TPS 对比具有误导性。Sui 的对象模型使简单转账极快但复杂 DeFi 交互较慢。以太坊 L2 继承了 L1 安全性但分散了流动性。Solana 的单体设计提供了 L2 生态系统难以匹配的可组合性。

---

## 3. 开发者技术栈

### 3.1 编程模型：账户、程序、指令

Solana 的模型与 EVM 链有根本不同：

| 概念 | Solana | Ethereum |
|------|--------|----------|
| 智能合约 | 程序（无状态） | 合约（有状态） |
| 状态存储 | 账户（与代码分离） | 合约存储（捆绑） |
| 执行模型 | 程序读写账户 | 合约调用修改自身存储 |
| 并行性 | 内建（账户声明） | 无（顺序执行） |
| 可升级性 | 原生（升级权限） | 代理模式变通 |

**账户模型：**
- Solana 上的每个数据片段都是一个**账户**
- 账户有所有者（程序）、lamport 余额和数据字段
- 程序是标记为可执行的特殊账户
- 程序是**无状态的**——它们不自己存储数据，而是读写独立的账户
- 这种代码和状态的分离使并行执行成为可能

**指令和交易：**
- 一条指令 = 对一个程序的一次调用，指定账户
- 一笔交易 = 一条或多条指令（原子性——全部成功或全部失败）
- 最大交易大小：1,232 字节（通过 SIMD-0296 的 v1 格式增加到 4,096）
- 每笔交易必须列出它将涉及的所有账户

**程序派生地址 (PDA)：**
- 从程序 ID + 种子确定性推导的地址
- 没有私钥——只有推导程序可以为其签名
- 用于程序拥有的数据账户、托管、权限委托

### 3.2 Rust vs Anchor vs Seahorse

| 框架 | 语言 | 成熟度 | 使用场景 |
|------|------|--------|----------|
| 原生 Rust | Rust | 生产级 | 最大控制，复杂程序 |
| Anchor | Rust (宏) | 生产级（主导） | 标准开发，安全保证 |
| Seahorse | Python → Rust | 实验性（维护减少） | 原型设计，黑客松 |

**Anchor** 是事实标准。它提供：
- `declare_id!` — 程序地址声明
- `#[program]` — 指令处理器定义
- `#[account]` — 账户序列化/反序列化
- 自动验证账户所有权、签名和约束
- 客户端库的 IDL 生成
- 防止常见漏洞的内建安全检查

**Seahorse** 将 Python 编译为 Anchor 兼容的 Rust。适合快速原型设计，但 2025 年维护已停滞——**不推荐用于生产环境**。

### 3.3 代币扩展 (Token-2022)

Token-2022 程序是原始 SPL Token 程序的超集，具有模块化扩展：

| 扩展 | 用途 | 知名采用者 |
|------|------|-----------|
| 转账费用 | 每次转账自动收费 | $BERN（6.9%费用） |
| 永久委托 | 追回/冻结能力 | Paxos USDP |
| 机密转账 | 加密余额（ZK 证明） | 开发中 |
| 转账钩子 | 每次转账的自定义逻辑 | 合规工具 |
| 不可转让 | 灵魂绑定代币 | 证书、凭证 |
| 生息 | 显示累积利息 | 收益代币 |
| 元数据 | 链上代币元数据 | 替代 Metaplex 元数据 |
| 默认账户状态 | 创建时自动冻结 | 受监管证券 |

**采用状态（2026年）：** 增长中但尚未主导。主要钱包（Phantom、Backpack）支持代币扩展。大多数新的合规/机构代币正在使用 Token-2022。然而，现有生态系统代币的大多数仍使用原始 SPL Token 程序。

### 3.4 压缩 NFT（状态压缩）

**成本革命：**

| 规模 | 传统 NFT | 压缩 NFT | 节省 |
|------|---------|---------|------|
| 1个 NFT | ~$2.00 | ~$0.00005 | 40,000倍 |
| 100万 NFT | ~$120万 | ~$50 | 24,000倍 |
| 1亿 NFT | ~$1.2亿 | ~$500 | 240,000倍 |

**工作原理：**
1. NFT 数据存储在链下
2. 哈希存储在链上的**并发 Merkle 树**中
3. 只有 Merkle 根在链上（一个账户）
4. 更新修改树并在 Solana 账本中存储证明
5. 任何人都可以从账本数据重建和验证完整的树

**并发 Merkle 树创新：** 标准 Merkle 树在并发更新时会崩溃（过期证明）。Solana Labs 发明了一种缓冲机制，接受最多 64 次更新之前的过期证明，实现快速转发而无需重新计算。

### 3.5 优先费用和计算单元

| 参数 | 默认值 | 最大值 |
|------|--------|--------|
| 每条指令计算单元 | 200,000 CU | 可配置 |
| 每笔交易计算单元 | 1,400,000 CU | 1,400,000 CU |
| 每个签名基础费 | 5,000 lamport | 固定 |
| 优先费 | 可选 | 市场驱动 |
| 最小优先小费 (Jito) | 1,000 lamport | — |
| 竞争性 Jito 小费 | 10K-50K lamport | — |

**局部费用市场：** Solana 的费用市场局限于特定账户。热门 AMM 池上的高竞争推高该池交易的费用，但涉及其他账户的交易仍然便宜。这与以太坊的全局 gas 市场有根本不同。

### 3.6 Solana 移动

| 产品 | 详情 |
|------|------|
| Saga（第1代） | 首款加密手机，2023年，销量一般 |
| Seeker（第2代） | 2025年8月发货，$450，140K+预售，覆盖57个国家 |
| dApp Store | Google Play 的替代方案，为 Web3 精选 |
| Seed Vault | 硬件隔离钱包（与 Solflare 合作） |
| Seeker Genesis Token | 用于独家生态系统奖励的灵魂绑定 NFT |

**仅预售收入：** $6300万+（140K台 x $450）。

### 3.7 Solana Actions & Blinks

**是什么：** 一种将任何 Solana 交易转化为可分享 URL 的规范。

**流程：**
1. 客户端向 Action URL 发送 GET 请求 → 接收元数据（标题、图标、按钮）
2. 用户点击操作 → 客户端发送 POST 请求 → 接收可签名交易
3. 用户钱包签名并提交

**使用场景：** 社交媒体上的捐赠按钮、推文中的 NFT 铸造链接、电子邮件中的支付请求——全部无需离开平台。

2024年6月25日启动。任何"Action-aware"客户端（钱包扩展、机器人、网站）都可以将 Blinks 渲染为交互式 UI 组件。

---

## 4. DeFi 生态系统

### 4.1 主要协议

| 协议 | 类别 | TVL（2025年7月） | 关键特性 |
|------|------|------------------|----------|
| Jito | 流动质押 + MEV | $27.2亿 (17.9%) | MEV 共享 LST |
| Kamino | 借贷 | $24.3亿 (16.0%) | 自动化金库策略 |
| Jupiter | DEX 聚合器 | $23.9亿 (15.8%) | 95%聚合器市场份额 |
| Raydium | AMM/DEX | $17.7亿 | 集中流动性，订单簿混合 |
| Marinade | 流动质押 | $16.9亿 (11.2%) | 首个 SOL LST（mSOL），原生质押 |
| Drift | 永续合约 DEX | $9.4亿 (6.2%) | 去中心化永续期货 |
| Orca | AMM/DEX | $3.6亿 (2.4%) | 集中流动性（Whirlpools） |

### 4.2 TVL 历史

| 时期 | TVL | 背景 |
|------|-----|------|
| 2021年11月（峰值） | ~$120亿 | 牛市顶部 |
| 2023年1月（FTX后） | ~$2亿 | FTX 崩溃重创 Solana 生态 |
| 2025年1月 | ~$80亿 | 恢复 + meme 币繁荣 |
| 2025年7月 | ~$175亿 | 历史新高 |
| 2025年Q3 | ~$115亿 | 回调，仍排以太坊之后第二 |

**FTX 复活故事：** Solana 的 TVL 在 2022年11月 FTX 崩溃后下降了 97%（Sam Bankman-Fried 是 Solana 的主要支持者）。在2.5年内恢复到新高是加密货币史上最引人注目的回归之一。

### 4.3 Solana 上的 MEV — Jito 的方法

**Solana MEV 与以太坊的区别：**

| 属性 | 以太坊 MEV | Solana MEV |
|------|-----------|------------|
| 内存池 | 公开 → MEV 提取 | 无内存池（Gulf Stream） |
| 区块构建者 | 专业化（Flashbots） | Leader 验证者 |
| MEV 分配 | 构建者 → 提议者 → 销毁 | Jito 小费 → 验证者 → 质押者 |
| 主导策略 | 三明治攻击 | 套利 + 尾随 |
| 保护机制 | 私有内存池（Flashbots Protect） | Jito 捆绑包 |

**Jito 的捆绑系统：**
- 用户支付小费将交易包含在私有 Jito 捆绑包中
- 捆绑包通过启用 Jito 的验证者处理
- "不抢跑"规则：包含 `jitodontfront` 账户的捆绑包必须将该交易放在首位
- 为付费用户提供 MEV 保护

**三明治攻击现实（2025年数据）：**
- 识别出 500K+ 次三明治攻击，损失超过 770万 SOL
- **93%的三明治是"宽幅"（跨slot）** — 前跑和后跑跨越不同 leader slot
- 这绕过了 Jito 的每捆绑包保护
- 宽幅三明治在一年内提取了 529,000+ SOL
- Jito 的方法是必要的但不充分——需要新的对策

### 4.4 流动质押格局

| LST 代币 | 提供者 | 质押 SOL | APY | 特殊功能 |
|----------|--------|---------|-----|----------|
| JitoSOL | Jito | 1430万 | ~7.5% + MEV | 最大 LST，200+验证者 |
| mSOL | Marinade | ~800万 | ~7-8% | 首个 Solana LST（2021） |
| bSOL | BlazeStake | ~300万 | ~7% | 200+验证者，社区导向 |
| INF | Sanctum | 增长中 | 不等 | 元 LST 聚合器 |

**市场规模：** 流动质押 TVL $107亿+。所有质押 SOL 的 13.3% 是流动的（5700万 SOL）。

**Sanctum 的创新：** 一个"元 LST"平台，通过单一流动性层统一所有 LST。他们的 INF 代币代表一篮子 LST，解决了流动性碎片化问题。

### 4.5 Jupiter 的主导地位

Jupiter 之于 Solana 就像 Uniswap 之于以太坊，但更大：

- **95% DEX 聚合器市场份额**
- **50%+所有 Solana DEX 交易量**通过 Jupiter 路由
- **$30亿+ TVL**（2025年10月）
- **$12亿+日交易量**

2025-2026 扩张：
- Jupiter Lend（借贷，2025年8月）
- JupUSD（稳定币，2025年Q4）
- Jupiter Terminal（类彭博终端的专业交易界面）
- ParaFi Capital $3500万投资（2026年2月）
- Catstanbul 销毁 30亿 JUP 代币（2025年1月）

**担忧：** Jupiter 的收购扩张引发了生态系统中心化忧虑。当一个协议控制了 95%的交换路由 + 借贷 + 稳定币 + 专业交易时，它成为单点故障。

### 4.6 真实收益 vs 代币排放

Solana DeFi 生态系统已从补贴驱动（2020-2022）成熟为收入驱动（2023+）：

| 收益来源 | APY | 可持续性 |
|---------|-----|----------|
| 基础质押奖励 | 6-7% | 通胀资金（递减） |
| MEV 小费 (Jito) | +1-1.5% | 来自套利的真实收入 |
| DeFi 可组合性（LP、借贷） | +3-8% | 市场驱动 |
| 复合堆栈（LST + MEV + DeFi） | 12-20% | 混合：真实收益 + 激励 |

**关键洞察：** Solana 上接近零的交易费意味着策略收益流向用户而非被 gas 成本消耗——这是相对以太坊 DeFi 收益策略的结构性优势。

---

## 5. NFT 与消费者应用

### 5.1 压缩 NFT 革命

NFT 采用的最具影响力的单项技术创新：

- **之前：** 铸造 100万 NFT 花费 ~$120万 租金
- **之后：** 同一系列花费 ~$50
- **机制：** 并发 Merkle 树 + 链下数据 + 链上证明
- **权衡：** 查询稍微复杂（需要 Helius DAS API 等索引器）

**影响：** 使得以前在经济上不可能的用例成为可能——忠诚计划、游戏物品、活动门票、作为 NFT 的社交互动。DRiP 等项目分发了数百万免费压缩 NFT 用于创作者变现。

### 5.2 Metaplex 协议

Metaplex 是 Solana 上 NFT 的标准：
- 编写了原始 NFT 代币标准
- 支持 Solana 上几乎所有铸造的 NFT
- **Metaplex Core**（2024）：简化标准，每个 NFT 一个链上账户
- 从每次 NFT 铸造中收取小额费用
- MPLX 治理代币

**市场格局（2025年）：**

| 市场 | 日交易量 | 活跃地址 |
|------|---------|----------|
| Magic Eden | 45,000+ SOL | 267K |
| Tensor | 16,000+ SOL | 298K |

Tensor 虽然交易量较低但活跃地址领先，表明有更多小交易者。Magic Eden 的交易量来自鲸鱼活动。

### 5.3 Solana 上的 DePIN 项目

Solana 已成为 DePIN（去中心化物理基础设施网络）的默认链：

| 项目 | 类别 | 2025年收入 | 关键指标 |
|------|------|-----------|----------|
| Helium | 无线网络 | $950万/年 | 移动用户激增，墨西哥/巴西扩展 |
| Hivemapper | 地图 | $55万/年 | 2025年399M公里，434K司机 |
| Render Network | GPU 计算 | 主要代币 | AI 渲染工作负载 |
| io.net | 分布式 GPU | 增长中 | AI/ML 计算市场 |
| GEODNET | 精密 GPS | 增长中 | RTK 定位网络 |

**2025年 DePIN 向节点运营者的总分配：** $8100万+（以 Helium 和 Render 为主）。

**为什么 DePIN 选择 Solana：** 向数千个节点运营者进行高频微支付的低交易成本。以太坊的 gas 费使得每设备支付不切实际。

### 5.4 Solana Pay

| 特性 | 详情 |
|------|------|
| 启动 | 2022年 |
| Shopify 集成 | 活跃（数百万商家有资格） |
| PayPal PYUSD | 在 Solana 上运行 |
| Visa 试点 | USDC 结算测试 |
| 交易量 | 自2023年Q4以来 $1000亿+总转账 |
| 交易成本 | 接近零（不到一分钱的一小部分） |

**现实检查：** 虽然技术可行且存在主要集成，但实际商家采用仍然有限。Shopify 集成实现了加密结账，但很少有商家主动推广。消费者认知仍是瓶颈。

---

## 6. Meme 币与 Pump.fun

### 6.1 Pump.fun — 工作原理

```
用户上传图片 + 选择名称/代码
         │
         ▼ (花费 < $2)
代币在联合曲线上创建
         │
         ▼
立即在 pump.fun 上开始交易
         │
         ▼ (如果市值达到 $90K)
"毕业" → 在 Raydium DEX 上市
         │
         ▼
Pump.fun 赚取：1%交换费 + 1.5 SOL 毕业费
```

### 6.2 Pump.fun 数据

| 指标 | 值 |
|------|-----|
| 总收入（累计） | ~$8亿 |
| 2025年 Solana 应用收入份额 | 领先所有 Solana 应用 |
| 代币募资（2025年7月） | $13亿（$6亿公开 + $7亿私募） |
| 已发布代币 | 600万+（截至2025年1月） |
| Solana DEX 交易份额（2025年12月） | 52.8% |
| 2026年1月收入 | $1400万 |
| 跑路代币比例 | ~98.6% |

### 6.3 Meme 币经济

**对 Solana 网络的收入：**
- 仅 Pump.fun 就是 2025 年 Solana 区块链最大的收入驱动者之一
- Meme 交易产生了巨大的交易量和优先费
- 在高峰期，meme 相关交易主导了 Solana 区块空间

**丑陋的真相：**
- 98.6%的 Pump.fun 代币表现出拉高出货行为
- Pump.fun 上 99%的代币和 Raydium 93%的流动性池呈现跑路特征
- 利用 Solana 的低费用快速部署未经审计的合约
- 内部人员预分配常见——创始人向散户抛售

### 6.4 重大争议

- **$LIBRA（2025年2月）：** 阿根廷总统 Milei 发推支持；随后跑路
- **$TRUMP：** 政治 meme 币，交易量巨大，合规担忧
- **FCA 干预（2024年12月）：** 英国 FCA 因无授权运营封锁了 Pump.fun 网站
- **SEC 声明（2025年2月）：** Meme 币根据联邦法律"不是证券"但"波动且有风险"

**对 Solana 声誉的影响：** 双刃剑。Meme 币推动了采用指标和收入，但将网络与赌场文化和骗局联系在一起。Solana 基金会努力与这种叙事保持距离，同时从中受益。

---

## 7. SOL 代币经济学

### 7.1 供应量与通胀

| 指标 | 值 |
|------|-----|
| 创世供应 | 5亿 SOL |
| 当前总供应 | ~5.98-6.2亿 SOL |
| 流通供应 | ~4.8-5.16亿 SOL (~82-86%) |
| 非流通（锁定/基金会） | ~1.17亿 SOL |
| 最大供应 | **无硬顶**（通胀性） |
| 当前通胀率（2025） | ~4.0% |
| 通胀递减 | 每年 15% |
| 终端通胀率 | 1.5%（预计~2031年） |

**通胀时间表：**

```
第1年（启动）：8.0%
第2年：         6.8%
第3年：         5.8%
第4年：         4.9%
第5年（2025）：  4.0%
...
~2031年：       1.5%（终端）
```

### 7.2 初始分配

| 分配 | 百分比 |
|------|--------|
| 社区储备（基金会） | 38.89% |
| 种子投资者 | 16.23% |
| 创始销售 | 12.92% |
| 团队成员 | 12.79% |
| Solana 基金会 | 10.46% |
| 验证者销售 | 5.18% |
| 战略销售 | 1.88% |
| 拍卖 | 1.64% |

**VC 集中度批评：** ~38%分配给了内部人员（团队 + 基金会 + 种子/战略）。与比特币的公平启动甚至以太坊的 ICO 相比，Solana 的初始分配严重偏向 VC，这是反复出现的批评。

### 7.3 质押与验证者经济学

| 指标 | 值 |
|------|-----|
| 质押 SOL | ~3.34亿 SOL (~67%供应量) |
| 质押年化收益 | ~7-8%（包含 MEV） |
| 验证者运营成本 | $4,000+/月（云端） |
| 投票成本 | ~3 SOL/epoch (~$150/天) |
| 硬件要求 | 12-16核，256GB+ RAM，快速 SSD |
| 佣金率 | 0-10%（典型） |

**运行验证者代价昂贵：** 投票成本（~$150/天 = ~$4,500/月）加基础设施（~$4,000/月），验证者需要大量权益才能保本。这导致验证者整合——小运营商无法竞争。

### 7.4 SOL 价格历史

| 日期 | 价格 | 事件 |
|------|------|------|
| 2020年4月（启动） | <$1 | 主网beta |
| 2021年11月（ATH1） | $260 | 牛市顶部 |
| 2022年11月（底部） | $8-10 | FTX 崩溃 |
| 2025年1月（ATH2） | $294.85 | 新历史高点 |
| 2026年1月 | $146 | 2026年初反弹 |
| 2026年2月（当前） | ~$82 | 回调 |

**市值轨迹：** 从接近零到 $470亿（2026年2月），峰值 ~$1300亿（2025年1月）。FTX 崩溃将 SOL 从 $35 带到 $8——77%跌幅——然后在 2 年内恢复 3,500%至新高。

---

## 8. 治理与组织结构

### 8.1 三个实体

| 实体 | 角色 | 关键人物 |
|------|------|----------|
| **Solana 基金会** | 非营利，拨款，生态系统增长 | — |
| **Solana Labs** | 原始开发公司（范围缩小） | Anatoly Yakovenko, Raj Gokal |
| **Anza** | 核心协议开发（2024年分拆） | Stephen Akridge（联合创始人），Amber Christiansen |

**Anza 分拆（2024年1月）：**
- ~50名 Solana Labs 员工（团队的一半）转移到 Anza
- Anza：员工持有，营利性
- Solana Labs 持有 Anza 13%
- Yakovenko 和 Gokal 在 Anza **没有股份**
- 由 Solana 基金会拨款资助
- 维护 **Agave** 验证者客户端（原始 Solana 客户端的分叉）

**其他关键开发团队：**
- **Jump Crypto/Jump Trading Group：** 构建 Firedancer 验证者客户端
- **Jito Labs：** MEV 基础设施，Jito-Solana 客户端
- **Helius：** RPC 基础设施，DAS API，按权益最大的验证者

### 8.2 关键人物

**Anatoly Yakovenko (Toly):**
- 1981年生于乌克兰；90年代初移民美国
- 在 Qualcomm 工作 10年以上（分布式系统，OS 工程）
- 2017年构思历史证明
- 估计净资产：$5-8亿
- 仍是 Solana Labs CEO，活跃于 X/Twitter
- 公开拒绝"以太坊杀手"叙事

**Raj Gokal:**
- 沃顿商学院（宾夕法尼亚大学），经济学学位
- 风险投资和产品开发背景
- Solana Labs 联合创始人和 COO
- 专注于商业战略、合作伙伴关系和社区建设

### 8.3 去中心化辩论

**支持去中心化的论点：**

| 指标 | Solana | Ethereum |
|------|--------|----------|
| 中本聪系数 | 20 | 6 |
| 地理分布 | 45+国家 | — |
| 客户端多样性（2026） | 3个客户端（Agave、Firedancer、Jito-Solana） | 5+客户端 |
| 客户端权益集中度 | 88% Jito-Solana | 48% Geth |

**反对去中心化的论点：**
- 验证者从 2,500（2023）降至 ~800（2026）——68%下降
- 前3个验证者（Helius、Binance、Galaxy）控制 26%+权益
- 硬件要求（$8K+/月）排除了业余运营者
- 88%权益集中在 Jito-Solana 客户端（比以太坊的 Geth 主导更严重）
- 68%权益在欧洲，20%在美国中西部——地理集中
- 基金会"修剪"600+验证者引发了关于无许可运行的质疑

**ETF 风险：** 如果预计的 ETF 权益集中到顶级验证者，中本聪系数可能从 20 降至 ~12（40%下降）。

### 8.4 VC 影响力

| 投资者 | 角色 |
|--------|------|
| a16z (Andreessen Horowitz) | 早期支持者，$5000万投入 Jito（2025年10月） |
| Multicoin Capital | 主要 SOL 持有者，生态投资者 |
| Jump Crypto | 构建 Firedancer，$10亿金库倡议 |
| Galaxy Digital | 与 Multicoin + Jump 的 $10亿金库倡议 |
| Alameda Research（已倒闭） | 前主要持有者，随 FTX 崩溃 |

Galaxy/Multicoin/Jump 的 $10亿 SOL 金库倡议显示了深度 VC 承诺，但也带来集中风险——少数大持有者对治理和生态方向有过大影响力。

---

## 9. 2025-2026 重大进展

### 9.1 Firedancer 主网（2025年12月）

最重要的技术里程碑：
- 2025年12月15日上线，经过3年以上开发
- Jump Trading Group 的独立 C/C++ 实现
- 实验室条件下展示 100万+ TPS
- 21%+权益运行 Firedancer/Frankendancer
- 将单客户端依赖从致命风险降低到可管理风险

### 9.2 Alpenglow 共识升级

**Solana 历史上最大的协议变更**（SIMD-0326）：
- 以 98.27%支持率通过，52%权益参与投票
- 用新架构替换 PoH 和 TowerBFT
- **Votor：** 80%+权益参与时单轮最终性；60%时自动两轮回退
- **Rotor：** 新的区块传播机制
- 目标最终性：**100-150ms**（从当前12-13秒）
- 测试网：2025年12月 | 主网：2026年Q1
- 消除投票费用（大幅降低验证者成本）

### 9.3 Solana ETF 批准

| 里程碑 | 日期 |
|--------|------|
| 首个现货 Solana ETF（21Shares） | 2025年10月28日 |
| SEC 通用上市标准批准 | 2025年9月 |
| 香港现货 SOL ETF（华夏基金） | 2025年10月 |
| 总 ETF 申请 | 23份单独申请 |
| 主要发行方 | Franklin Templeton、Fidelity、Grayscale、VanEck、Bitwise |
| 审批流程 | 75天（从旧规则的240+天缩短） |

**影响：** ETF 带来机构资本和合法性。摩根大通预计初始流入"温和"，但长期结构性需求转变意义重大。

### 9.4 SIMD 提案与协议升级

| SIMD | 标题 | 状态 | 影响 |
|------|------|------|------|
| SIMD-0326 | Alpenglow 共识 | 已批准 (98%) | 亚200ms最终性 |
| SIMD-0296 | 交易 v1 格式 | 进行中 | 4096字节交易限制（从1232） |
| SIMD-0266 | P-token 标准 | 计划2026年H2 | 98%资源减少 |

### 9.5 单体 vs 模块化定位

Solana 在辩论中的立场：

| 方面 | Solana（单体式） | Ethereum（模块化） |
|------|-----------------|-------------------|
| 扩展策略 | 优化单一分片 | 拆分为 L2 rollup |
| 可组合性 | 原生（一切在一条链上） | 跨 L2 碎片化 |
| 流动性 | 统一 | 分散在 Arbitrum、Base、OP 等 |
| 开发者体验 | 单一部署目标 | 选择 L2，桥接复杂性 |
| 去中心化 | 硬件要求限制运营者 | L1 可访问，L2 通常中心化 |
| 用户体验 | 一个钱包，一条链 | L2 间桥接，多种 gas 代币 |

**可组合性论点：** Solana 上的交换可以在同一笔交易中原子性地与借贷协议组合。在以太坊 L2 上，这需要具有延迟和信任假设的跨链消息。这是 Solana 最强的架构论点。

### 9.6 企业采用

| 合作伙伴 | 集成 |
|---------|------|
| Visa | Solana 上的 USDC 结算试点 |
| Shopify | Solana Pay 结账集成 |
| PayPal | Solana 上的 PYUSD 稳定币 |
| Stripe | 支付处理支持 |
| Franklin Templeton | 链上基金（BENJI） |
| Hamilton Lane | 代币化基金产品 |

---

## 10. 诚实评估：多空论点

### 10.1 Solana 真正做得更好的地方

1. **速度+成本：** 亚秒交易，费用不到一美分。没有其他 L1 在生产环境中匹配。
2. **可组合性：** 单链原子可组合性是 Solana 对比碎片化 L2 生态的杀手级特性。
3. **开发者势头：** 10,000+黑客松参与者（2025年7月），FTX 恢复后开发者数量增长。
4. **DePIN 天然契合：** 低成本、高吞吐量的微支付使 Solana 成为默认 DePIN 链。
5. **压缩 NFT：** 状态压缩是真正的技术创新，将 NFT 成本降低了 40,000倍。
6. **韧性：** 2年以上无完全中断（自2024年2月），证明早期中断时代是成熟阶段。
7. **多验证者客户端：** Firedancer + Agave 减少了困扰早期 Solana 的单客户端风险。

### 10.2 真实弱点

1. **验证者中心化：** 68%验证者数量下降，前3实体控制 26%权益，$8K+/月运营成本排除小运营者。
2. **客户端集中：** 88%权益在 Jito-Solana。Firedancer 采用在增长但仍是少数。
3. **VC 集中：** 38%初始供应给了内部人员。Galaxy/Multicoin/Jump $10亿金库进一步集中持有。
4. **Meme 币声誉：** Pump.fun 98.6%跑路率。网络收入严重依赖投机活动。
5. **理论 vs 实际 TPS：** 营销声称 65K TPS；现实是 400-800 用户 TPS。差距侵蚀可信度。
6. **无硬性供应上限：** 持续通胀（递减至 1.5%）意味着持续稀释，不同于比特币的固定供应。
7. **硬件要求：** 256GB RAM + 高速多核 CPU = 机构级硬件要求，损害去中心化。
8. **历史中断：** 运营前3年8次重大中断创造的信任赤字需要数年克服。

### 10.3 "以太坊杀手"叙事是否合理？

**不，Solana 的创始人自己也拒绝了这个说法。**

Anatoly Yakovenko 称"ETH 杀手"框架"很蹩脚"，倡导共存。数据支持这种微妙观点：
- Solana 在原始吞吐量、UX 速度和 DePIN 方面领先
- 以太坊在 TVL（$500亿+ vs $115亿）、开发者数量和机构信任方面领先
- 它们服务不同细分：Solana 擅长高频低价值交易；以太坊擅长高价值 DeFi 和结算

**真正的竞争：** 不是 Solana vs 以太坊——而是单体式 vs 模块化架构。如果 Solana 的方法在不牺牲太多去中心化的情况下满足需求扩展，它就验证了单体论点。如果以太坊 L2 实现无缝互操作性，模块化方法就赢了。

### 10.4 长期可持续性

**看多论点：**
- Alpenglow + Firedancer 解锁 100-150ms 最终性和 100万+ TPS 潜力
- ETF 流入带来机构资本
- DePIN + 支付提供可持续的非投机需求
- 企业采用（Visa、Shopify、PayPal）创造粘性
- 真实收益 DeFi 取代依赖排放的挖矿

**看空论点：**
- 验证者中心化随成本上升而恶化
- Meme 币收入是周期性的——熊市可能重创网络活动
- SOL 通胀继续稀释持有者
- 竞争链（Sui、Aptos）以更好的架构（对象模型）提供类似性能
- 与 meme 币骗局相关的监管风险
- 如果 Alpenglow 部署出现问题，可能触发新一轮中断时代

---

## 最新動態 (2026)

### Solana ETF 势头加速

现货 Solana ETF 于 2025 年末在美国推出，稳步获得关注。截至 2026 年 2 月中旬，累计流入约 **$8.8 亿**，其中 Bitwise 的 BSOL 领先。2025 年 11 月是最强单月，净流入 $4.2 亿。重大里程碑：**摩根士丹利**于 2026 年 1 月 6 日申请了比特币信托和 **Solana 信托**——这是美国资产排名前十的银行首次申请加密货币 ETF。值得注意的是，摩根士丹利的 Solana 信托包含**质押组件**以产生收益，这是比特币 ETF 所没有的结构创新。尽管 2026 年 2 月整体加密 ETF 出现资金流出，Solana ETF 逆势而行，持续正流入。

### Ondo Global Markets：代币化美股登陆 Solana

2026 年 1 月，**Ondo Finance** 将其 Ondo Global Markets 平台扩展到 Solana，将 **200+ 只代币化美股和 ETF** 带到链上——包括 NVDA、AAPL、META、SPY 和 QQQ。代币由美国注册经纪商持有的证券 1:1 支持，可 24/5 铸造和赎回，并可与 DeFi 协议组合。这使 Ondo 成为 Solana 上**最大的真实世界资产 (RWA) 发行方**（按资产数量计），标志着该网络从 meme 币标签向机构金融的转型。此举使 Solana 有资格竞逐价值 16 万亿美元以上的可代币化证券市场。

### 稳定币生态爆发

Solana 的稳定币市值达到 **$130-160 亿**（2025 年 12 月峰值），占全球稳定币市场的约 4.5%。USDC 以约 $100 亿（55.7% 份额）占主导地位。PayPal 的 **PYUSD 增长 112%** 至 $4.45 亿，现在在 Solana 上的交易量已持续超过以太坊——尽管最初在以太坊上推出。Solana 上非 USDC/USDT 稳定币供应自 2025 年 1 月以来**激增约 10 倍**，其交易量份额从 4.4% 增长到 23.7%。这种多元化表明真正的商业采用超越了纯粹的投机。

### Solana 上的 AI Agent 基础设施

Solana 正在成为自主 AI Agent 的默认结算层。**DFlow** 推出了 MCP（模型上下文协议），这是一个通用交易工具，使基于 Claude、Cursor 和其他 AI 工作站构建的 AI Agent 能够在 Solana 上执行交易。**Phantom** 推出了自己的 AI 交互 MCP 服务器。**Solana Agent Kit**（由 SendAI 开发）现在提供 60+ 个预构建操作，涵盖代币操作、NFT 铸造和 DeFi 交互。2026 年初，AI Agent 开始管理 DAO、为自己的密钥充值，并自主部署代币——从原型走向生产。

### Firedancer 突破 20% 权益门槛

自 2025 年 12 月 15 日在主网上线以来，Firedancer 稳步获得采用。运行 100 天后，它在 Solana 主网上突破了 **20% 权益门槛**，验证者报告与 Agave 相比无性能下降。**Figment**，最大的质押提供商之一，于 2025 年 10 月将其主验证者迁移到 Firedancer，并报告 **+28 个基点**的总 SRR 改善。两个客户端保持共识一致性无分歧，证明多客户端互操作性在生产中可行。但 CryptoSlate 指出 Solana 仍然违反了以太坊视为不可协商的"超多数客户端多样性"安全门槛——这是一个值得关注的问题。

### Alpenglow 时间线日趋明确

Alpenglow 共识升级（SIMD-0326）正按计划推进，**2026 年 Q1 测试网，主网目标 Q2 2026**。该升级取消链上投票费用，预计将显著降低验证者的盈利门槛——目前需要大约 **$2000 万**的委托权益才能盈利运营。通过消除这些成本，网络预计将扩大参与范围并改善去中心化。100 倍的最终性改善（12.8 秒到 100-150ms）将使 Solana 与中心化交易所的确认时间具有竞争力。

### SOL 价格与基本面背离

尽管 DeFi TVL 创新高、稳定币活动激增、ETF 流入强劲，SOL 在 **2026 年初下跌约 31%**——从 1 月的 $146 降至 2 月的约 $82。这种网络基本面与代币价格表现之间的显著背离反映了更广泛的加密市场调整。链上现货交易量在 **2025 年达到 $1.6 万亿**，超过除 Binance 以外的所有链下交易所。OG 建设者们认为下一篇章"比 meme 币更大，比 FTX 更大"，指出机构采用是新的增长载体。

### Solana 的机构化转型

叙事正在转变。摩根士丹利的 ETF 申请、Ondo 的代币化股票、Blockdaemon 的机构 Solana 指南、ParaFi 对 Jupiter 的 $3500 万投资——Solana 正在积极从"meme 币链"重新定位为"去中心化纳斯达克"。亚秒级最终性（Alpenglow 后）+ 可组合 DeFi + 不断增长的 RWA 代币化，共同为机构链上金融构建了一个引人注目的技术栈——问题在于验证者中心化的担忧是否会限制这一雄心。

---

## 11. 可复用模式

### 11.1 技术架构模式

| 模式 | Solana 实现 | 可复用于 |
|------|-----------|---------|
| **共识前时钟** | PoH 在投票前提供排序 | 任何需要排序保证的分布式系统 |
| **显式访问声明** | 交易预先声明账户访问 | 数据库调度器，并行处理系统 |
| **异构硬件流水线** | GPU 验签，CPU 执行逻辑，NIC 做 IO | 任何高吞吐数据管道 |
| **树形传播** | Turbine 的 shred 层级 | 内容分发，P2P 文件共享 |
| **消除内存池** | Gulf Stream 的直接转发 | 消息队列优化，推送架构 |
| **并发 Merkle 树** | 过期证明快速转发 | 任何需要并行 Merkle 更新的系统 |
| **代码与状态分离** | 无状态程序 + 数据账户 | 微服务架构（逻辑 vs 存储分离） |

### 11.2 市场推广模式

| 模式 | Solana 如何使用 | 可复用经验 |
|------|----------------|-----------|
| **黑客松驱动采用** | 每次黑客松 10K+参与者 | 通过竞赛获取开发者 |
| **灾难生存叙事** | FTX 崩溃 → "不可杀死"的声誉 | 危机恢复作为品牌建设 |
| **先让投机者进来** | Meme 币推动了大量采用指标 | 投机使用引导网络效应 |
| **移动优先身份** | Saga/Seeker 手机，dApp Store | 替代应用分发渠道 |
| **空投→留存** | Jupiter 的 200万钱包空投 | 代币分发作为营销 |
| **多客户端策略** | 资助 Jump 构建 Firedancer | 赞助独立实现以获取可信度 |

### 11.3 开发者体验模式

| 模式 | 实现 | 收获 |
|------|------|------|
| **单一部署目标** | 一切在一条链上 | 减少部署复杂性 → 更多构建者 |
| **框架优先** | Anchor 成为默认 | 有观点的工具胜过灵活性 |
| **读取免费** | 账户数据无需费用即可读取 | 减少数据访问的摩擦 |
| **可组合原语** | 原子跨程序调用 | 让在他人工作基础上构建变得容易 |
| **压缩状态** | NFT 的状态压缩 | 当经济需要时用计算换存储 |
| **局部费用市场** | 每账户竞争定价 | 隔离热点使普通用户不受影响 |

---

## 总结表：Solana 一览（2026年2月）

| 类别 | 状态 |
|------|------|
| 架构 | 8项集成创新，Alpenglow 替换 PoH + TowerBFT |
| 性能 | 400-800 真实用户 TPS，Firedancer 目标 100万+ |
| 验证者健康 | ~800验证者，中本聪系数 20，3个客户端实现 |
| DeFi | $115亿 TVL，Jupiter/Jito/Kamino 主导，真实收益成熟中 |
| NFT | 压缩 NFT 革命性降低成本，Magic Eden + Tensor 市场 |
| Meme 币 | Pump.fun = $8亿收入，98.6%跑路率，声誉双刃剑 |
| 代币经济学 | ~4%通胀递减至1.5%，无供应上限，67%质押 |
| SOL 价格 | $82（2026年2月），ATH $294.85（2025年1月） |
| 治理 | 三实体结构（基金会/Labs/Anza），VC 集中担忧 |
| 2026展望 | Alpenglow 主网，ETF 流入，企业合作扩展 |
| 诚实评价 | 加密货币中最好的用户体验，真正的可组合性优势，但中心化权衡是真实的 |

---

## References

### Architecture Deep Dive

**Proof of History**
- [Proof of History: A Clock for Blockchain — Solana](https://solana.com/news/proof-of-history--a-clock-for-blockchain)
- [Proof of History — Solana](https://solana.com/news/proof-of-history)
- [A Deep Dive into Proof of History and Accounts on Solana — Medium](https://medium.com/@amansatyawani/a-deep-dive-into-proof-of-history-and-accounts-on-solana-fb47161f3ef1)
- [Solana: How it works — A Technical Deep Dive — Medium](https://medium.com/@AlexAlekhinEth/solana-how-it-works-a-technical-deep-dive-b180468abc3d)
- [How Solana Builds Blocks: A Technical Deep Dive — ruggero.io](https://www.ruggero.io/blog/how_solana_builds_blocks/)
- [What is proof of history and why does Solana use it? — The Block](https://www.theblock.co/learn/302470/what-is-proof-of-history-and-why-does-solana-use-it)
- [Proof of History — GridPlus](https://docs.gridplus.io/blockchain-basics/solana/proof-of-history)

**Tower BFT**
- [Tower BFT: Solana's High Performance Implementation of PBFT — Solana](https://solana.com/news/tower-bft--solana-s-high-performance-implementation-of-pbft)
- [Tower BFT — Anatoly Yakovenko — Medium](https://medium.com/solana-labs/tower-bft-solanas-high-performance-implementation-of-pbft-464725911e79)
- [Consensus on Solana — Helius](https://www.helius.dev/blog/consensus-on-solana)
- [Tower BFT — Solana Handbook (Ackee)](https://ackee.xyz/solana/book/latest/chapter2/tower-bft/)
- [Tower BFT — Solana Validator Docs](https://docs.solanalabs.com/implemented-proposals/tower-bft)
- [EVM to SVM: Consensus — Solana](https://solana.com/developers/evm-to-svm/consensus)

**Turbine & Gulf Stream**
- [Turbine: Block Propagation on Solana — Helius](https://www.helius.dev/blog/turbine-block-propagation-on-solana)
- [Solana's Gulf Stream — Helius](https://www.helius.dev/blog/solana-gulf-stream)
- [Gulf Stream — Solana](https://solana.com/news/gulf-stream--solana-s-mempool-less-transaction-forwarding-protocol)
- [Gulf Stream — Anatoly Yakovenko — Medium](https://medium.com/solana-labs/gulf-stream-solanas-mempool-less-transaction-forwarding-protocol-d342e72186ad)
- [Gulf Stream — Solana Handbook (Ackee)](https://ackee.xyz/solana/book/latest/chapter2/gulf-stream/)

**Sealevel**
- [Sealevel — Parallel Processing Thousands of Smart Contracts — Solana](https://solana.com/news/sealevel---parallel-processing-thousands-of-smart-contracts)
- [Sealevel — Anatoly Yakovenko — Medium](https://medium.com/solana-labs/sealevel-parallel-processing-thousands-of-smart-contracts-d814b378192)
- [Solana: Architecture & Parallel Transactions — Chainstack](https://chainstack.com/solana-architecture-parallel-transactions/)
- [Compare Solana's Transaction Lifecycle & Sui's Object Runtime — Helius](https://www.helius.dev/blog/solana-vs-sui-transaction-lifecycle)
- [Sealevel — Solana Handbook (Ackee)](https://ackee.xyz/solana/book/latest/chapter2/sealevel/)

**Pipelining & Cloudbreak**
- [Pipelining in Solana: The Transaction Processing Unit — Solana](https://solana.com/news/pipelining-in-solana-the-transaction-processing-unit)
- [8 Innovations That Make Solana the First Web-Scale Blockchain — Solana](https://solana.com/news/8-innovations-that-make-solana-the-first-web-scale-blockchain)
- [How Solana Works: 8 Core Technologies Explained — BuildBear](https://www.buildbear.io/blog/solana-core-concepts)

### Performance & Reality Check

**TPS & Network Stats**
- [Solana TPS, Max TPS, Block Time — Chainspect](https://chainspect.app/chain/solana)
- [Solana Compass: Live Statistics — SolanaCompass](https://solanacompass.com/statistics)
- [Solana Statistics 2025 — SQ Magazine](https://sqmagazine.co.uk/solana-statistics/)
- [Solana Transaction Speed & TPS — OKX](https://www.okx.com/en-us/learn/solana-transaction-speed-tps)
- [Solana smashes 107,000 TPS milestone — CryptoSlate](https://cryptoslate.com/solana-smashes-107000-tps-milestone-sparking-questions-about-real-world-use/)
- [Enterprise Solana Infrastructure: What Matters in 2026 — Chainstack](https://chainstack.com/enterprise-solana-infrastructure-what-matters-in-2026/)

**Outage History**
- [A Complete History of Solana Outages — Helius](https://www.helius.dev/blog/solana-outages-complete-history)
- [Solana Outages: A History — StatusGator](https://statusgator.com/blog/solana-outage-history/)
- [Solana Outage Guide 2025 — OKX](https://www.okx.com/en-us/learn/solana-outage-guide)
- [Solana Status — Official](https://status.solana.com/)

**Firedancer**
- [Jump Crypto's Firedancer Goes Live on Solana Mainnet — Unchained](https://unchainedcrypto.com/jump-cryptos-firedancer-goes-live-on-solana-mainnet/)
- [Firedancer hits Solana mainnet — The Block](https://www.theblock.co/post/382411/jump-cryptos-firedancer-hits-solana-mainnet-as-the-network-aims-to-unlock-1-million-tps)
- [Solana's Firedancer Validator Client Deep Dive — Blockdaemon](https://www.blockdaemon.com/blog/solanas-firedancer-validator-client-deep-dive)
- [Firedancer — GitHub](https://github.com/firedancer-io/firedancer)
- [What Is Firedancer and Why It Matters — Backpack](https://learn.backpack.exchange/articles/what-is-firedancer)
- [Firedancer Hits 20% Stake — Coira](https://coira.io/blog/solana-firedancer-upgrade-performance-breakthrough-2026)
- [Figment's Migration to Firedancer — Figment](https://www.figment.io/insights/figments-migration-to-firedancer-unlocking-next-generation-solana-validator-performance/)
- [Firedancer is live, but Solana violates client diversity safety rule — CryptoSlate](https://cryptoslate.com/firedancer-is-live-but-solana-is-violating-the-one-safety-rule-ethereum-treats-as-non-negotiable/)

**Comparisons**
- [Solana vs Ethereum: Which is Better in 2026? — Backpack](https://learn.backpack.exchange/articles/solana-vs-ethereum)
- [Sui vs Solana 2026 — Bitget](https://www.bitget.site/academy/sui-vs-solana-comparison-2026-comprehensive-guide-to-performance-ecosystem-and-investment)
- [Sui vs. Aptos: Competitive Analysis — VanEck](https://www.vaneck.com/us/en/blogs/digital-assets/sui-vs-aptos-competitive-analysis-and-price-prediction/)

### Developer Technical Stack

**Programming Model**
- [Solana Programs — Official Docs](https://solana.com/docs/core/programs)
- [Solana Programming Model — Medium (Sidarth)](https://medium.com/@sidarths/solana-school-lesson-3-solana-programming-model-i-accounts-anchor-pda-cpi-explained-9bbc34a57b23)
- [Introduction to the Solana Programming Model — Syndica](https://blog.syndica.io/an-introduction-to-the-solana-programming-model-accounts-data-and-transactions/)

**Anchor Framework**
- [Anchor Program Structure — Official](https://www.anchor-lang.com/docs/basics/program-structure)
- [Beginner's Guide to Building Solana Programs with Anchor — Helius](https://www.helius.dev/blog/an-introduction-to-anchor-a-beginners-guide-to-building-solana-programs)
- [Anchor Framework — GitHub](https://github.com/solana-foundation/anchor)

**Token Extensions**
- [Token Extensions — Solana](https://solana.com/solutions/token-extensions)
- [Token-2022 Program — SPL Docs](https://spl.solana.com/token-2022)
- [What are Token Extensions? — Helius](https://www.helius.dev/blog/what-is-token-2022)
- [Solana Token Extensions Builders Are Quietly Adopting — Medium](https://medium.com/@jickpatel611/the-7-solana-token-extensions-builders-are-quietly-adopting-55d6f8fd9999)
- [Token Extensions Guide — Phantom](https://phantom.com/learn/crypto-101/solana-token-extensions)

**Compressed NFTs**
- [How to use compressed NFTs on Solana — Solana](https://solana.com/news/how-to-use-compressed-nfts-on-solana)
- [State Compression — Solana Docs](https://solana.com/docs/advanced/state-compression)
- [All You Need to Know About Compression on Solana — Helius](https://www.helius.dev/blog/all-you-need-to-know-about-compression-on-solana)
- [State compression brings down cost — Solana](https://solana.com/news/state-compression-compressed-nfts-solana)

**Priority Fees & Compute**
- [Transaction Fees — Solana Docs](https://solana.com/docs/core/fees)
- [Priority Fees — Helius](https://www.helius.dev/blog/priority-fees-understanding-solanas-transaction-fee-mechanics)
- [Why Solana Transaction Costs Matter — Anza](https://www.anza.xyz/blog/why-solana-transaction-costs-and-compute-units-matter-for-developers)
- [How to use Priority Fees — Solana](https://solana.com/developers/guides/advanced/how-to-use-priority-fees)

**Solana Actions & Blinks**
- [Blockchain Links and Solana Actions — Solana](https://solana.com/solutions/actions)
- [Actions and Blinks Developer Guide — Solana](https://solana.com/developers/guides/advanced/actions)
- [Everything About Solana Blinks — Tatum](https://tatum.io/blog/about-solana-blinks)

**Solana Mobile**
- [Seeker — Solana Mobile](https://x.com/solanamobile)
- [Solana Mobile begins shipping Seeker — The Block](https://www.theblock.co/post/365600/solana-mobile-seeker-crypto-smartphone)
- [Solana Seeker surpassing 140,000 presales — The Block](https://www.theblock.co/post/317154/solana-seeker-saga-crypto-mobile-phone-successor-140000-presales)

**Seahorse**
- [How to Write Solana Programs in Python Using Seahorse — Alchemy](https://www.alchemy.com/overviews/solana-seahorse)
- [Seahorse — GitHub](https://github.com/solana-developers/seahorse)
- [Seahorse Lang — Official](https://seahorse-lang.org/)

### DeFi Ecosystem

**TVL & Major Protocols**
- [Solana — DefiLlama](https://defillama.com/chain/solana)
- [Top 10 DeFi Apps on Solana in 2026 — Eco](https://eco.com/support/en/articles/13225733-top-10-defi-apps-on-solana-in-2026-complete-guide)
- [Solana's DeFi Surge: $17.5B TVL — BlockchainReporter](https://blockchainreporter.net/solanas-defi-surge-jito-kamino-and-cloud-drive-solana-to-17-5-billion-tvl/)
- [State of Solana Q3 2025 — Messari](https://messari.io/report/state-of-solana-q3-2025)
- [Solana Lending Markets Report 2025 — RedStone](https://blog.redstone.finance/2025/12/11/solana-lending-markets/)

**Jupiter**
- [What Is Jupiter (JUP)? — CoinGecko](https://www.coingecko.com/learn/what-is-jupiter-crypto-solana)
- [Jupiter — DefiLlama](https://defillama.com/protocol/jupiter)
- [Jupiter Ecosystem Dominance in 2026 — AInvest](https://www.ainvest.com/news/jupiter-ecosystem-dominance-solana-defi-flywheel-driven-play-2026-2512/)
- [Jupiter's Acquisition Spree — CoinDesk](https://www.coindesk.com/business/2025/01/27/jupiter-s-acquisition-spree-buyback-plan-spark-solana-ecosystem-dominance-concerns)

**MEV & Jito**
- [Solana MEV Economics: Jito, Bundles, and Liquid Staking — QuickNode](https://blog.quicknode.com/solana-mev-economics-jito-bundles-liquid-staking-guide/)
- [Solana MEV: Deep Dive into Jito — sanj.dev](https://sanj.dev/post/solana-mev-jito-deep-dive)
- [Quantifying Sandwiching MEV on Jito — ACM/IMC 2025](https://dl.acm.org/doi/10.1145/3730567.3764493)
- [Solana MEV Exposed — Solana Compass](https://solanacompass.com/learn/accelerate-25/scale-or-die-at-accelerate-2025-the-state-of-solana-mev)
- [Solana users paying millions to stop bots — DL News](https://www.dlnews.com/articles/defi/solana-users-use-jito-to-stop-sandwich-attacks-and-mev/)

**Liquid Staking**
- [Solana Liquid Staking: Ultimate Guide 2026 — Phantom](https://phantom.com/learn/crypto-101/solana-liquid-staking)
- [Best Solana Liquid Staking Tokens 2025 — Sanctum](https://sanctum.so/blog/best-solana-liquid-staking-tokens-2025)
- [Solana Liquid Staking: Everything You Need to Know 2025 — Nansen](https://www.nansen.ai/post/solana-liquid-staking-everything-you-need-to-know-in-2025)
- [Liquid Staking Yields Ranked 2026 — Sanctum](https://sanctum.so/blog/solana-liquid-staking-yields-ranked-highest-paying-lsts-2026)

**Yield & Revenue**
- [Solana DeFi Returns 2025 — Solana Compass](https://solanacompass.com/learn/breakpoint-25/the-state-of-returns-on-solana-solstice-ben-nadareski)
- [Jito becomes largest protocol on Solana — The Block](https://www.theblock.co/post/292146/jito-network-largest-defi-protocol-solana-tvl)

### NFT & Consumer

**Metaplex & Marketplaces**
- [2025 Guide to Metaplex — RiseIn](https://www.risein.com/blog/the-2025-guide-to-metaplex-your-solana-nft-studio)
- [Tensor Embraces Metaplex Core — NFT Plazas](https://nftplazas.com/tensor-metaplex-core/)
- [Solana NFT Market Landscape 2025 — Gate](https://www.gate.com/post/status/16149210)
- [Best NFT Marketplaces on Solana 2026 — Backpack](https://learn.backpack.exchange/articles/best-nft-marketplaces-on-solana)

**DePIN**
- [Deep Dive: Solana DePIN December 2025 — Syndica](https://blog.syndica.io/deep-dive-solana-depin-december-2025/)
- [Solana DePIN Revenue Milestones — Solana Report](https://solanareport.com/2025/10/01/solana-depin-revenue-milestones-how-helium-hivemapper-and-render-are-powering-record-growth/)
- [DePINscan — Solana DePIN Projects](https://depinscan.io/chains/solana)
- [Solana Emerges as Leader in DePIN — DePINscan](https://depinscan.io/news/2025-05-01/solana-emerges-as-leader-in-depin-projects-a-comprehensive-analysis)

**Solana Pay**
- [Solana Pay — Official](https://solanapay.com/)
- [Solana Pay Announcement — Solana](https://solana.com/news/solana-pay-announcement)
- [Payments Tooling — Solana](https://solana.com/solutions/payments-tooling)

### Meme Coins & Pump.fun

- [Pump.fun — Wikipedia](https://en.wikipedia.org/wiki/Pump.fun)
- [Pump.fun Leads as Solana App Revenue Hits $2.4B — CryptoPotato](https://cryptopotato.com/pump-fun-leads-as-solana-app-revenue-hits-2-4b-in-2025/)
- [Pump.fun 101 — 21Shares](https://www.21shares.com/en-us/research/pump-fun-101-the-meme-coin-platform-powering-solana)
- [Pump.fun Surpasses $800M Revenue — CoinMarketCap](https://coinmarketcap.com/academy/article/pumpfun-surpasses-dollar800m-revenue-in-solana-meme-coin-battle)
- [Solana Rug Pulls — Solidus Labs](https://www.soliduslabs.com/reports/solana-rug-pulls-pump-dumps-crypto-compliance)
- [100% of Solana Memecoins Are Scams — CyberNews](https://cybernews.com/almost-examined-solana-memecoins-are-scam/)
- [Tracing $TRUMP — TRM Labs](https://www.trmlabs.com/resources/blog/tracing-trump)

### SOL Token Economics

- [Solana Tokenomics: Circulating Supply, Inflation Schedule — SolanaCompass](https://solanacompass.com/tokenomics)
- [Solana Tokenomics: Everything to Know — Crypto.com](https://crypto.com/us/crypto/learn/solana-tokenomics)
- [Solana Tokenomics — OKX](https://www.okx.com/en-us/learn/solana-tokenomics-supply-inflation)
- [Solana Circulating Supply — Gemini](https://www.gemini.com/cryptopedia/solana-circulating-supply-how-many-sol-tokens-are-there)
- [SOL Price — CoinGecko](https://www.coingecko.com/en/coins/solana)
- [SOL Price History — CoinMarketCap](https://coinmarketcap.com/currencies/solana/historical-data/)

### Governance & Organization

**Solana Foundation / Labs / Anza**
- [Drama at Solana: Creation of Anza — Fortune](https://fortune.com/crypto/2024/01/30/solana-anza-new-legal-entity-third-of-employees-decentralization-theater/)
- [Solana Labs to Agave Fork — Solana](https://solana.com/news/solana-labs-anza-fork)
- [Anza — Official](https://www.anza.xyz/)
- [Anza's 2025 roadmap — Blockworks](https://blockworks.co/news/anza-optimizing-solana-efficiency)

**Key Figures**
- [Solana Founder: Visionary Behind Solana's Rise — CoinPaper](https://coinpaper.com/11839/solana-founder-the-visionary-behind-solana-s-rise)
- [Who is Anatoly Yakovenko — Finst](https://finst.com/en/learn/articles/who-is-anatoly-yakovenko)
- [Who is Raj Gokal — DailyCoin](https://dailycoin.com/raj-gokal-solanas-right-hand-man/)
- [Who Founded Solana — Bitrue](https://www.bitrue.com/blog/who-founded-solana)

**Decentralization**
- [Measuring Solana's Decentralization — Helius](https://www.helius.dev/blog/solana-decentralization-facts-and-figures)
- [Solana Decentralization Dashboard — SolanaCompass](https://solanacompass.com/statistics/decentralization)
- [Solana's Validator Decline — AInvest](https://www.ainvest.com/news/solana-validator-decline-signal-network-health-centralization-risk-2512/)
- [Solana Staking Insights Annual 2025 — Everstake](https://everstake.one/crypto-reports/solana-staking-insights-analysis-annual-2025)

**VC Investment**
- [a16z invests $50M in Jito — CoinDesk](https://www.coindesk.com/markets/2025/10/16/andreessen-horowitz-s-a16z-invests-usd50m-in-solana-staking-protocol-jito)
- [Galaxy, Multicoin, Jump raise $1B for Solana — CryptoBriefing](https://cryptobriefing.com/solana-treasury-fund-initiative/)

### 2025-2026 Developments

**Alpenglow**
- [SIMD-0326: Alpenglow Consensus Protocol — Solana Forum](https://forum.solana.com/t/simd-0326-proposal-for-the-new-alpenglow-consensus-protocol/4236)
- [98% Votes to Approve Alpenglow — CoinDesk](https://www.coindesk.com/tech/2025/09/02/solana-set-for-major-overhaul-after-98-votes-to-approve-historic-alpenglow-upgrade)
- [Inside Alpenglow — 1inch Blog](https://1inch.medium.com/https-blog-1inch-com-inside-alpenglow-d9bca04af089)
- [Alpenglow Consensus: Biggest Protocol Upgrade — QuickNode](https://blog.quicknode.com/solana-alpenglow-upgrade/)
- [Solana 2026 Protocol Upgrades — KuCoin](https://www.kucoin.com/news/flash/solana-to-launch-2026-protocol-upgrades-for-faster-transactions-and-lower-costs)
- [Alpenglow Secures Approval, Faces Challenges — The Defiant](https://thedefiant.io/news/blockchains/solana-alpenglow-upgrade-secures-approval-but-faces-challenges)
- [What is Alpenglow in Solana — Everstake](https://everstake.one/blog/what-is-alpenglow-in-solana)

**ETF**
- [11 Solana ETFs — NerdWallet](https://www.nerdwallet.com/investing/learn/solana-etfs)
- [Solana ETPs Cleared to Trade — Charles Schwab](https://www.schwab.com/learn/story/crypto-etf-approval)
- [16 U.S. Solana Spot ETFs — Helius](https://www.helius.dev/blog/solana-etfs)
- [JPMorgan Sees Modest Inflows — CoinDesk](https://www.coindesk.com/markets/2025/10/09/jpmorgan-sees-modest-inflows-for-solana-etfs-despite-likely-sec-approval)
- [Bitcoin, ether, xrp ETFs bleed while Solana bucks outflow trend — CoinDesk](https://www.coindesk.com/markets/2026/02/19/bitcoin-ether-xrp-etfs-bleed-while-solana-bucks-outflow-trend)
- [Morgan Stanley files for Bitcoin and Solana ETFs — CoinDesk](https://www.coindesk.com/markets/2026/01/06/morgan-stanley-eyes-the-spot-bitcoin-etf-market)
- [Morgan Stanley Solana Trust — SEC Filing](https://www.sec.gov/Archives/edgar/data/2103547/000110465926000988/tm2534148d1_s1.htm)
- [Solana 2026 Predictions: ETF Flows — Solana Compass](https://solanacompass.com/learn/Lightspeed/whats-next-for-solana-in-2026)

**Monolithic vs Modular**
- [Modular vs Monolithic Blockchains: 2026 — CCN](https://www.ccn.com/education/crypto/modular-vs-monolithic-blockchains-crypto-scalability-war-2026/)
- [Comparing Monolithic vs Modular Blockchains — Fidelity](https://www.fidelitydigitalassets.com/research-and-insights/comparing-monolithic-vsmodular-blockchains)
- [Monolithic vs. modular blockchain — Visa](https://usa.visa.com/solutions/crypto/monolithic-vs-modular-blockchain.html)

**Enterprise Adoption**
- [Solana Rise in Payments — SolanaCompass](https://solanacompass.com/learn/Lightspeed/solanas-emergence-as-a-payments-hub-roundup)
- [Solana H1 2025 Report — QuickNode](https://blog.quicknode.com/solana-ecosystem-report-h1-2025/)
- [2025: Dawn of Solana's Institutional Era — Medium](https://medium.com/@scalingx/2025-the-dawn-of-solanas-institutional-era-cda664cf8482)

### Validator Economics & Staking

- [Is Running a Solana Validator Profitable? — Hivelocity](https://www.hivelocity.net/blog/solana-validator-economics/)
- [Solana Staking and Validator Economics 2025 — Gate](https://www.gate.com/learn/articles/understanding-solanas-staking-and-validator-economics-in-2025/6062)
- [Solana Staking Calculator — Helius](https://www.helius.dev/staking/calculator)
- [How Much Do Solana Validators Make? — SolanaCompass](https://solanacompass.com/staking/how-much-do-solana-validators-make)
- [Leveling the Stakes on Solana — Placeholder VC](https://www.placeholder.vc/blog/2025/9/15/leveling-the-stakes-on-solana)
- [Solana Staking — Official](https://solana.com/staking)

### Assessment & Analysis

- [Solana: Ethereum Killer or Bot Playground? — CCN](https://www.ccn.com/analysis/crypto/solana-ccn-reports/)
- [Solana Vs. Ethereum In 2025 — ChiangRaiTimes](https://www.chiangraitimes.com/finance/solana-vs-ethereum-in-2025/)
- [Solana Founder Says It Is Not An Ethereum Killer — Bitcoinist](https://bitcoinist.com/solana-is-not-an-ethereum-killer/)
- [Solana Blockchain Explained — StealthEX](https://stealthex.io/blog/solana-blockchain/)
- [Solana 2025 Breakthrough — AInvest](https://www.ainvest.com/news/solana-2025-breakthrough-era-blockchain-scalability-ecosystem-growth-2601/)

### Solana Treasury Companies

- [Rise of Solana Digital Asset Treasury Companies — Helius](https://www.helius.dev/blog/solana-digital-asset-treasury-companies)
- [Solana Treasuries: SOL Holdings — CoinGecko](https://www.coingecko.com/en/treasuries/solana)
- [Solana Treasury Tracker — The Block](https://www.theblock.co/treasuries/solana-treasuries)

### Latest Updates (2026) Sources

**ETF & Institutional**
- [Morgan Stanley files for Bitcoin and Solana ETFs — Bloomberg](https://www.bloomberg.com/news/articles/2026-01-06/crypto-latecomer-morgan-stanley-files-for-bitcoin-solana-etfs)
- [Morgan Stanley Submits SEC Filing — 401kSpecialist](https://401kspecialistmag.com/morgan-stanley-submits-sec-filing-for-bitcoin-and-solana-etfs/)
- [Morgan Stanley files crypto ETFs — Euronews](https://www.euronews.com/business/2026/01/06/morgan-stanley-files-to-launch-bitcoin-and-solana-etfs-as-wall-street-embraces-crypto)

**Ondo & RWA**
- [Ondo Finance Brings 200+ Tokenized U.S. Stocks to Solana — CoinDesk](https://www.coindesk.com/business/2026/01/21/ondo-finance-brings-200-tokenized-u-s-stocks-and-etfs-to-solana)
- [Ondo Global Markets: Tokenized Stocks & ETFs on Solana — Solana](https://solana.com/news/ondo-global-markets-tokenized-stocks-etfs-solana)
- [Ondo Finance Launches Tokenized Stocks on Solana — Ondo](https://ondo.finance/blog/global-markets-live-on-solana)

**Stablecoins**
- [Stablecoins on Solana in 2026 — Chainstack](https://chainstack.com/solana-stablecoins-2026/)
- [Non-USDC/USDT stablecoin supply surges 10x — Cryptopolitan](https://www.cryptopolitan.com/non-usdc-usdt-stablecoin-supply-on-solana/)
- [Solana's Stablecoin Landscape — Helius](https://www.helius.dev/blog/solanas-stablecoin-landscape)
- [Why Solana stablecoin action boomed — DL News](https://www.dlnews.com/articles/markets/why-solana-stablecoin-action-boomed-over-2025/)

**AI Agents**
- [How to Build a Solana AI Agent in 2026 — Alchemy](https://www.alchemy.com/blog/how-to-build-solana-ai-agents-in-2026)
- [Phantom debuts MCP server for AI — BitcoinEthereumNews](https://bitcoinethereumnews.com/tech/solana-bolsters-tooling-as-phantom-debuts-mcp-server-for-ai/)
- [Solana Agent Kit — GitHub](https://github.com/sendaifun/solana-agent-kit)
- [Intro to AI on Solana — Solana](https://solana.com/developers/guides/getstarted/intro-to-ai)

**Market & Ecosystem**
- [Solana OG builders say next chapter is bigger than memecoins — CoinDesk](https://www.coindesk.com/business/2026/02/12/solana-s-og-builders-say-the-next-chapter-is-bigger-than-memecoins-and-bigger-than-ftx)
- [2 Game-Changing Updates Coming to Solana in 2026 — Motley Fool](https://www.fool.com/investing/2026/02/11/2-game-changing-updates-coming-to-solana-in-2026/)
- [Solana's 2026 Upgrade Cycle: Path to Decentralized Nasdaq — AInvest](https://www.ainvest.com/news/solana-2026-upgrade-cycle-path-decentralized-nasdaq-2601/)
- [Solana Crashes 31% in 2026 Even as DeFi Hits Record Highs — HokaNews](https://www.hokanews.com/2026/02/solana-crashes-31-in-2026-even-as-defi.html)
- [Ethereum and Solana set the stage for 2026's DeFi reboot — CoinDesk](https://www.coindesk.com/tech/2026/01/03/ethereum-and-solana-set-the-stage-for-2026-s-defi-reboot)
