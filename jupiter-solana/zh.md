# Jupiter：Solana 的 DeFi 超级应用 — 深度技术与生态分析

> Jupiter 从 DEX 聚合器起步，最终成为 Solana 的金融操作系统。这就是它的故事。

## 目录

1. [产品深度解析](#1-产品深度解析)
2. [架构与技术栈](#2-架构与技术栈)
3. [Jupiter Ape 与 Meme 币交易](#3-jupiter-ape-与-meme-币交易)
4. [JUP 代币](#4-jup-代币)
5. [Jupiter DAO 与治理](#5-jupiter-dao-与治理)
6. [LFG Launchpad](#6-lfg-launchpad)
7. [团队与背景](#7-团队与背景)
8. [商业模式与收入](#8-商业模式与收入)
9. [市场地位与竞争](#9-市场地位与竞争)
10. [2025-2026 年发展动态](#10-2025-2026-年发展动态)
11. [诚实评估](#11-诚实评估)
12. [可复用模式](#12-可复用模式)

---

## 1. 产品深度解析

Jupiter 不再只是一个 DEX 聚合器。它是 Solana 上一个垂直整合的 DeFi 超级应用，拥有七条独立的产品线，每条服务不同的用户群体，但共享流动性和用户基础。

### 1.1 DEX 聚合器 — 核心产品

**功能**：在所有 Solana DEX（Raydium、Orca、Meteora、Phoenix、Lifinity 等）中寻找最优交易价格，通过多跳路径进行订单拆分和路由。

**路由工作原理**：

| 世代 | 引擎 | 算法 | 关键改进 |
|------|------|------|----------|
| V1-V2 | 传统 | 简单图遍历 | 基本多 DEX 路由 |
| V3-V5 | Metis | 改良 Bellman-Ford | 增量流式处理，任意阶段拆分+合并 |
| V6 | Metis 1.5 | 增强 Bellman-Ford | 价格比 V2 好 5.22%，并行报价 |
| Ultra V3 | Iris + Juno | 黄金分割搜索 + Brent 方法 | 路径查找性能提升 100 倍，元聚合 |

**Metis 核心机制**：
- 将 Solana 的整个流动性视为有向图（代币 = 节点，池子 = 边）
- 通过路由增量流式传输代币，允许在任意阶段拆分和合并
- 同一个 DEX 可以出现在单次路由的多个环节中
- 将路由生成和价格报价合并为单一操作（消除死胡同探索）
- 处理 Solana 的 64 账户锁定限制（每笔交易指令最多 4 个 DEX 跳）

**Iris**（当前版本，Ultra V3）：用高级数学优化取代 Metis — 黄金分割搜索和 Brent 方法进行最优订单拆分。路径查找性能比 Metis 提升 100 倍。

**Juno**（元聚合器层）：位于所有路由引擎之上。聚合来自 Iris、JupiterZ（RFQ）和第三方来源（DFlow、Hashflow、OKX）的报价。自学习机制自动排除表现不佳或可疑的报价来源。

**JupiterZ**（RFQ 系统）：将用户直接连接到专业做市商，服务顶级交易对。日交易量约 1 亿美元，零滑点。做市商提供确定报价，不涉及 AMM 曲线。

### 1.2 限价单

用户设定目标价格；Jupiter 的 keeper 网络通过 Jupiter Price API 监控链上价格，当市场达到指定价格时自动执行。

**Keeper 机制**：
- Keeper 的功能类似借贷协议中的清算者
- 持续监控链上价格
- 使用 Jupiter 聚合引擎执行订单（因此成交也能获得最优路由）
- 平台手续费：0.1%

### 1.3 DCA（定投）

对任何 SPL 代币进行自动化定期购买，时间间隔可配置。

**工作原理**：
1. 用户将全部金额存入程序拥有的金库（如 1000 USDC 定投 SOL）
2. 第一笔订单立即执行
3. 剩余订单按用户定义的间隔执行
4. 每次成交使用 Jupiter 的聚合引擎
5. 订单有 ±2-30 秒的随机化时间偏移，防止被利用
6. 平台手续费：每次执行 0.1%

### 1.4 Jupiter Perps（永续合约）

**模型**：LP 对交易者（非订单簿）。交易者对 JLP 池开仓杠杆头寸。

**支持市场**：SOL、ETH、BTC — 最高 250 倍杠杆。

**预言机定价**：交易按预言机价格执行，而非 AMM 价格。这意味着：
- 大额交易无价格影响（预言机反映全球市场价格）
- 人为施加价格影响费来模拟滑点并保护 LP
- 三个预言机：Edge（Chaos Labs，主要）、Chainlink（备用）、Pyth（备用）
- 多预言机故障转移：如果主要预言机过期或偏离超过阈值，回退到备用对

**费用结构**：
- 开/平仓费：6 基点（0.06%）的头寸规模
- 借贷费：每小时，基于池子利用率 — `(锁定代币 / 池中代币) × 每小时费率 × 头寸规模`
- 无资金费率（与大多数永续合约交易所不同）

**清算**：当交易者的抵押品减去费用加减未实现盈亏 < 头寸规模的 0.2% 时，头寸被清算。

### 1.5 JLP（Jupiter 流动性提供者池）

永续合约交易者的对手方池。持有 JLP = 充当庄家。

**池子组成（目标权重）**：

| 资产 | 目标权重 | 角色 |
|------|---------|------|
| SOL | 44% | 多头抵押品 |
| ETH | 10% | 多头抵押品 |
| wBTC | 11% | 多头抵押品 |
| USDC | 26% | 空头抵押品 + 稳定币 |
| USDT | 9% | 空头抵押品 + 稳定币 |

**收益来源**：
- 所有永续合约费用的 75%（开仓、平仓、借贷、价格影响）
- 原生 SOL 质押收益（约 7% APR，2025 年 8 月添加）
- 实际 APY：稳定期 14-20%，历史峰值超过 120%

**风险**：
- JLP 持有者是对手方 — 如果交易者整体盈利，JLP 就亏损
- 最大历史回撤：-18%（2025 年 8 月 5 日市场崩盘）
- 前 10 个鲸鱼持有 80% 的 JLP 供应（集中度风险）
- 动态交换费根据池子资产偏离目标权重进行调整

**规模**：25 亿美元以上 TVL（2025 年 10 月）。

### 1.6 Jupiter Lock（代币锁仓）

开源、已审计（Sec3 + OtterSec）的代币锁仓程序。任何人都可以创建自定义解锁计划的锁仓。LFG launchpad 项目使用它进行团队代币锁仓。

### 1.7 Jupiter Terminal（可嵌入交换组件）

任何 dApp 都可以嵌入的即插即用 React 组件或原生 JS 组件，提供交换功能。

**集成模式**：
- NPM 包：`@jup-ag/terminal`
- Window 对象注入（最简单）
- 显示模式：集成式、悬浮窗、模态框

**关键特性**：Terminal 可以在没有任何 RPC 的情况下运行 — Ultra 处理交易发送、钱包余额和代币信息。对于没有现有钱包提供者的 dApp，Terminal 提供自己的钱包适配器。

### 1.8 Jupiter Mobile

功能完整的 Solana 钱包 + DeFi 应用（iOS + Android）。

**数据**：110 万以上用户，38 万以上总下载量，4.9/5 评分。

**功能**：
- 所有交换/DCA/限价单功能
- 带安全扫描的 Web3 浏览器
- Apple Pay 加密货币入金
- 电子邮件/Apple ID 登录（无需助记词）
- Jupiter Global：法币出入金、Jupiter Card、覆盖亚太地区的 QR Pay

---

## 2. 架构与技术栈

### 2.1 路由引擎演进

```
V1-V2 (2021-2022): 简单图遍历 → 基本多 DEX
     ↓
Metis (2023-2024): 改良 Bellman-Ford → 流式处理、拆分/合并、价格好 5.22%
     ↓
Metis 1.5 (2025年4月): 为 Ultra 增强 → 经验为 Iris 设计提供信息
     ↓
Iris (2025年10月): 黄金分割 + Brent 方法 → 路径查找快 100 倍
     ↓
Juno (2025年10月): 元聚合器 → 合并 Iris + JupiterZ + 第三方
```

**为什么这个演进重要**：聚合器之战靠延迟取胜。Solana 每 ~400 毫秒出一个块。你越快计算出最优路由，报价到交易落地时价格还存在的可能性就越大。

### 2.2 智能合约架构

Jupiter 的链上程序使用 Rust 在 Solana 的 Anchor 框架上编写。核心程序：

- **交换程序**：执行通过多跳 CPI（跨程序调用）在 AMM 间路由的交换
- **永续合约程序**：管理头寸、抵押品、清算、预言机集成
- **DCA 程序**：管理金库、调度、执行
- **限价单程序**：订单存储、keeper 授权、执行
- **锁仓程序**：解锁计划、线性/悬崖/自定义曲线

Solana 约束：每条指令 64 账户锁定限制意味着每笔交换最多约 4 个 DEX 跳。Metis/Iris 在这一约束内寻找最优路由。

### 2.3 MEV 防护

MEV（三明治攻击）是 Solana 上的关键问题，因为低交易成本使攻击变得廉价。

**Jupiter 的多层防护方案**：

| 层级 | 机制 | 防护级别 |
|------|------|----------|
| MEV 保护模式 | 将交易直接发送到 Jito 区块引擎（绕过公共内存池） | 强 |
| Iris 路由 | 将大订单拆分到多个池子，减少单池价格影响 | 中等 |
| 动态滑点 | 每路由、每代币的统计优化滑点 | 中等 |
| JupiterZ RFQ | 通过做市商的 AMM 外执行，零滑点 | 强 |
| Ultra V3 | 三明治攻击防护比前版本好 34 倍 | 强 |

**Ultra V3 的 MEV 防御**：组合所有层 — 优先通过 JupiterZ 路由（零滑点做市商执行），使用比用户设定更严格的动态滑点，通过 Jito 验证者发送。结果：三明治攻击面减少 34 倍。

### 2.4 Keeper 网络

Jupiter 的限价单和 DCA 使用 keeper 网络（类似 Chainlink Keepers 或 Gelato）：

- Keeper 通过 Jupiter Price API 监控链上价格
- 当条件满足时，keeper 提交执行交易
- Keeper 通过成交价和限价之间的价差获得激励
- 订单有随机化时间（±2-30 秒）防止 keeper 抢跑

### 2.5 预言机集成

**永续合约预言机栈**：
1. **Edge（Chaos Labs）** — 主要预言机，专为永续合约构建
2. **Chainlink** — 备用，与 Edge 交叉验证
3. **Pyth** — 备用，用作完整性检查和回退

**选择逻辑**：如果 Edge 新鲜且在 Chainlink 和 Pyth 阈值内 → 使用 Edge。如果 Edge 过期或偏离 → 比较 Chainlink 和 Pyth → 使用两者中最新的。如果 3 个预言机中 2 个失败 → 不更新价格（头寸不受坏数据影响）。

### 2.6 开发者 API 与 SDK

| API | 用途 | 关键特性 |
|-----|------|----------|
| Ultra Swap API | 简化交换生命周期 | 报价 → 订单 → 签名 → 执行 |
| V6 Swap API | 精细路由控制 | 自定义指令，CPI 集成 |
| Price API V3 | 参考定价 | 所有代币的实时价格 |
| Tokens V2 API | 代币发现 | 验证代币列表，元数据 |
| Terminal SDK | 可嵌入交换 | React 组件，零配置 |

**SDK 生态**：官方 Rust 客户端（`jup-swap-api-client`）、社区 Python SDK、C# 包装器。高交易量集成方可使用自托管 V6 API。

---

## 3. Jupiter Ape 与 Meme 币交易

### 3.1 Jupiter 如何成为 Meme 币中心

Jupiter 的聚合自然路由通过 Raydium、Orca 和 Meteora 上的 meme 币流动性池。当 pump.fun 代币毕业到 Raydium 时，Jupiter 是第一个展示它们的聚合器。

**飞轮**：Pump.fun 发射代币 → 代币毕业到 Raydium → Jupiter 聚合该池 → 交易者使用 Jupiter 购买 → 交易量流经 Jupiter。

Jupiter 处理 Raydium 55% 以上的交易，而 Raydium 是大多数 pump.fun 代币的目的地。这使 Jupiter 成为 meme 币的事实标准交易层。

### 3.2 Jupiter Ape 平台

作为专用 meme 币交易界面推出，带有安全功能：

**核心功能**：
- **Rugcheck 集成**：每个代币都有安全评分，可疑代币标记为「危险」
- **专用交易金库**：与用户主钱包隔离（安全层）
- **实时代币推送**：Raydium、Orca、Meteora 上的新发布实时展示
- **快速购买**：针对速度优化（对 meme 币狙击至关重要）

**理念**：与其对抗 meme 币趋势，不如带着安全护栏拥抱它。「更安全、更顺畅、更便宜的 meme 币交易方式。」

### 3.3 交易量影响

Meme 币交易是 Jupiter 巨大的交易量驱动力。在 meme 高峰期（2024 年末、2025 年初），日交易量超过 50 亿美元，大部分来自 meme 币活动。仅 TRUMP 代币发行就通过 Jupiter 基础设施产生了数十亿美元的交易量。

---

## 4. JUP 代币

### 4.1 代币经济学

| 指标 | 值 |
|------|-----|
| 最大供应量（原始） | 100 亿 JUP |
| 已销毁（2025年1月 Catstanbul） | 30 亿 JUP（约 36 亿美元） |
| 当前总供应量 | 70 亿 JUP |
| 流通供应量 | 约 32.4 亿 JUP |
| 历史最高价 | $2.04（2024年1月31日） |
| 历史最低价 | $0.136（2026年2月13日） |
| 当前价格（2026年2月） | 约 $0.17 |
| 市值（2026年2月） | 约 5.4 亿美元 |

**分配**（销毁后）：
- 社区/空投：约 50%
- 团队：约 20%
- 战略储备：约 15%
- 生态/资助：约 15%

### 4.2 Jupuary 空投

| 空投 | 日期 | 数量 | 接收者 | 关键详情 |
|------|------|------|--------|----------|
| Jupuary 1 | 2024年1月31日 | 约 10 亿 JUP | 约 90 万钱包 | 首次空投，规模巨大 |
| Jupuary 2 | 2025年1月22日 | 7 亿 JUP（6.16 亿美元） | 200 万钱包 | 4.4 亿给用户，6000 万给质押者，2 亿给增长活动 |
| Jupuary 3 | 2026年1月30日 | 2 亿 JUP | 活跃用户 | 从 7 亿减至 2 亿以限制稀释。1.75 亿给用户，2500 万给质押者。最后一次 Jupuary |

**资格标准**：交易量、一致性（活跃 8 个月以上 = 奖励加成）、质押时长、治理投票参与度。

### 4.3 质押与主动质押奖励（ASR）

**机制**：质押 JUP → 获得治理投票权（1 JUP = 1 票）→ 赚取 ASR 奖励。

**ASR 奖励池**：
- LFG Launchpad 费用的 75%
- 每年 1 亿 JUP 代币（按季度分配）

**ASR 工作原理**：
- 每日快照质押 JUP 数量
- 季度奖励分配
- 奖励与以下成正比：平均质押量 × 治理参与率
- 持续质押和投票 = 更高奖励

### 4.4 回购计划

在 Catstanbul 大会（2025年1月）宣布：协议费用的 50% 用于 JUP 回购。

**执行**：Jupiter 在公开市场回购 JUP（首次回购：2025年2月26日回购 488 万 JUP / 333 万美元）。购买的代币锁定 3 年存入 Litterbox Trust。

**Litterbox 销毁**：DAO 投票（86% 赞成，2025年11月）销毁 Litterbox Trust 中的 1.3 亿 JUP（约 4450 万美元），移除 4% 的流通供应。从累积转向通缩性销毁。

### 4.5 价格表现 — 悖论

Jupiter 可以说是 Solana 上每个指标都最成功的 DeFi 协议，除了代币价格。JUP 在发行炒作时达到 $2.04，然后到 2026 年 2 月下跌 91.7% 至 $0.17，尽管：
- 5.09 亿美元总收入（2025 年）
- 30 亿美元以上 TVL
- 95% 聚合器市场份额
- 活跃的回购计划

**为什么会断裂**：持续的代币解锁（团队、空投）产生的卖压压倒了回购需求。Jupuary 空投虽然有利于用户获取，但分发了数亿代币随即被抛售。

---

## 5. Jupiter DAO 与治理

### 5.1 结构

- **J.U.P. 承诺**：Jupiter United Planet — 社区治理框架
- **Catdets**：社区成员（「cadets」（学员）+ 猫品牌的组合）
- **工作组**：由 DAO 预算资助的特定任务团队，负责资助、生态发展
- **投票机制**：1 个质押 JUP = 1 票，链上治理通过 vote.jup.ag

### 5.2 关键治理投票

| 投票 | 结果 | 意义 |
|------|------|------|
| LFG Launchpad 批准 | 通过 | 社区选择代币发行项目 |
| 30 亿代币销毁 | 通过 | Solana DeFi 最大供应减少 |
| Litterbox 销毁（1.3 亿 JUP） | 86% 赞成 | 从累积转向销毁 |
| Jupuary 3 减少（7 亿 → 2 亿） | 通过 | 减少最后一次空投的稀释 |

### 5.3 治理暂停（2025年6月）

Jupiter DAO 暂停所有治理投票至 2026 年，原因：
- 团队与社区之间 **「信任崩溃」**
- 围绕代币发行和团队投票权的持续 FUD 循环
- 对团队集中投票权影响结果的担忧
- DAO、持有者和团队 「陷入负反馈循环」

**Meow 的声明**：「2026 年，我们将以统一而非分裂的新方式回归治理。」

值得注意的是：DeFi 最大的 DAO 之一自愿关闭治理来重新设计。暂停期间质押奖励照常发放。

### 5.4 治理恢复（2026年初）

在 CatLumpurr 大会（2026年1月-2月），Jupiter 表示将以修订后的工具和更清晰的团队与社区投票权分离来恢复治理。

---

## 6. LFG Launchpad

### 6.1 机制

**LFG = "Let's F*cking Go"** — Solana 上的社区驱动 launchpad。

**工作方式**：
1. 项目通过 Jupiter Research Forum 申请
2. Catdet 社区审查：产品力、团队质量、TGE 准备度
3. DAO 对候选者投票
4. 选中的项目通过 **单边 DLMM**（动态流动性做市商）池发行
5. 公平发行：所有人在链上以相同价格购买，无私募或种子轮分配
6. 项目向现有社区成员空投

**DLMM 机制**：根据实时需求和供应动态调整代币价格。确保持续流动性并减少上市后的崩盘。DLMM 池是单边的（最初只有项目代币），买家提供另一边。

### 6.2 知名发行项目

| 项目 | 代币 | 类别 | 备注 |
|------|------|------|------|
| WEN | $WEN | Meme 币 | 第一个 LFG 发行，测试运行 |
| Jupiter | $JUP | 治理 | 自我发行，大规模空投 |
| Zeus Network | $ZEUS | 跨链 | BTC-Solana 桥接 |
| Sanctum | $CLOUD | 流动性质押 | LST 基础设施 |
| deBridge | $DBR | 跨链 | 桥接协议 |
| Sharky | $SHARKY | NFT 借贷 | NFT-fi |

**规模**：截至 2025 年已发行 78 个项目，12 亿美元 TVL，78 万用户。

### 6.3 与其他 Solana Launchpad 对比

| 特性 | Jupiter LFG | Pump.fun | Raydium AcceleRaytor |
|------|------------|----------|---------------------|
| 筛选 | DAO 投票 | 无门槛 | 团队选择 |
| 公平发行 | 是（DLMM） | 是（联合曲线） | 不确定 |
| Rug pull 风险 | 低（已审核） | 极高（98.6%） | 中等 |
| 交易量 | 中等 | 巨大 | 低 |
| 社区 | Catdets | Degens | 交易者 |

---

## 7. 团队与背景

### 7.1 创始人

**Meow（匿名）**：
- Jupiter 和 Meteora（前身 Mercurial Finance）联合创始人
- Instadapp 和 Kyber Network 顾问
- 参与了 BitGo 和 Ren 的 Wrapped Bitcoin (WBTC) 发行
- OG DeFi 背景，从早期就连接 Solana 生态
- 尽管匿名，仍是 Jupiter 的公众代言人

**Ben Chow**：
- Jupiter 和 Meteora 联合创始人
- 前 IDEO（顶级设计公司）产品设计师
- 创办了 WishWell、Friended、Minute Inc.
- **2025年2月因 LIBRA meme 币丑闻从 Meteora 辞职**

### 7.2 LIBRA 丑闻

2025 年 2 月，Meteora（由 Meow 和 Ben Chow 联合创办）卷入 LIBRA 代币争议：

- Meteora 平台促成了 HAWK（Haliey Welch）、TRUMP、MELANIA 和 LIBRA 代币的发行
- LIBRA — 被阿根廷总统 Javier Milei 短暂推广 — 44,000 名买家中 74% 亏损
- Ben Chow 被指控内幕交易和接收 LIBRA 代币
- Chow 辞职；Meow 指出其「缺乏判断力和审慎」
- 聘请 Fenwick & West（律师事务所）进行独立调查
- Meow 声明：「Jupiter 或 Meteora 没有任何人参与内幕交易或财务不当行为」

**影响**：这一丑闻考验了 Jupiter 的信誉。迅速辞职、外部调查、公开声明的回应是教科书式的危机管理，但声誉损害持续存在。

### 7.3 融资

Jupiter 是 **自力更生** 的 — 直到 2026 年 2 月 ParaFi Capital 投资 3500 万美元（该协议运营 4 年多来的首次外部投资）才有外部资金。这对于 Jupiter 规模的协议来说非常罕见。

**ParaFi 交易条款**：
- 3500 万美元投资以 JupUSD（Jupiter 的稳定币）结算
- 市场价格购买代币，无折扣
- ParaFi 长期锁定期
- 认购权证以更高价格购买更多代币
- ParaFi（管理 20 亿美元）此前投资了 Aave、Compound、MakerDAO

---

## 8. 商业模式与收入

### 8.1 费用结构

| 产品 | 费用 | 谁支付 | 谁收取 |
|------|------|--------|--------|
| 聚合器交换 | 10 基点（非稳定币），2 基点（稳定币） | 交易者 | JLP 池（动态） |
| 永续合约开/平仓 | 6 基点 | 交易者 | 75% JLP，25% 协议 |
| 永续合约借贷 | 每小时，基于利用率 | 交易者 | 75% JLP，25% 协议 |
| 限价单 | 10 基点 | 交易者 | 协议 |
| DCA | 每次执行 10 基点 | 交易者 | 协议 |
| LFG Launchpad | 发行费 | 项目方 | 75% ASR，25% 协议 |

### 8.2 收入数据

| 期间 | 指标 | 金额 |
|------|------|------|
| 2025 年全年（毛） | 总费用 | 约 5.09 亿美元 |
| 2025 年 Q3 | 收入 | 4600 万美元 |
| 2025 年 Q3 | 永续合约收入 | 2460 万美元（约占 Q3 的 53%） |
| 2025 年 Q3 | 现货交易量 | 1768 亿美元 |
| 预计年化协议收入 | 协议净收入（永续合约费用的 30%） | 约 1.22 亿美元 |

**关键洞察**：永续合约产生约 80% 的 Jupiter 收入。聚合器（95% 市场份额）是用户获取渠道；永续合约是变现引擎。这是经典的「免费 → 付费」漏斗。

### 8.3 收入分配

自 2025 年 2 月 17 日起：
- 协议收入的 50% → JUP 回购（Litterbox Trust）
- 25% → 协议金库
- 25% → 团队运营

### 8.4 如何从「免费」聚合中获利

Jupiter 不向集成方或低于一定阈值的单笔交换收取聚合路由费。聚合器构建的是：
1. **习惯** — 用户默认使用 Jupiter 进行所有交换
2. **数据** — Jupiter 看到所有 Solana 交换流，为 JupiterZ 做市商报价提供信息
3. **向上销售** — 交换用户也使用永续合约、DCA、限价单（全部收费）
4. **JLP 流入** — JLP 池进出的交换费产生收入

---

## 9. 市场地位与竞争

### 9.1 Solana DEX 格局

| 平台 | 角色 | 市场份额 | 关键指标 |
|------|------|---------|----------|
| Jupiter | 聚合器 + 永续合约 | 95% 聚合器，66% 永续合约 | Q3 现货 1768 亿美元 |
| Raydium | AMM（底层流动性） | Jupiter 路由交易的约 55% | 月峰值 430 亿美元 |
| Orca | 集中流动性 AMM | 交易量第三 | 交易量增长 140% |
| Meteora | DLMM AMM | 上升中，支撑 LFG 发行 | 份额增长 |
| Phoenix | 订单簿 DEX | 细分市场 | 专业交易者 |

**关键细微差别**：Jupiter 并非在与 Raydium/Orca 「竞争」— 它是在通过它们路由。Jupiter 路由的 55% 以上交易在 Raydium 上执行。Jupiter 的主导地位实际上通过为它们的池子带来交易量而使底层 DEX 受益。

### 9.2 对比以太坊聚合器

| 特性 | Jupiter | 1inch | CoW Swap | ParaSwap |
|------|---------|-------|----------|----------|
| 链 | Solana | 多链（13+） | 以太坊 | 多链 |
| 延迟 | 约 400ms（1 个 Solana 块） | 12 秒以上（以太坊块） | 批量拍卖 | 12 秒以上 |
| Gas 成本 | <$0.01 | $5-50+ | $5-50+ | $5-50+ |
| MEV 防护 | Jito 验证者 + Iris | Flashbots、Fusion | 批量拍卖（最佳） | MEV Blocker |
| 跨链 | 否（Jupnet 计划中） | 是 | 否 | 是 |
| 永续合约 | 是（250x） | 否 | 否 | 否 |
| 独特方法 | RFQ + AMM 混合（JupiterZ） | Fusion（意图驱动） | 求解器竞争 | MEV 专注 |

**为什么 Jupiter 在 Solana 上取胜**：Solana 的速度 + 低费用使聚合远比以太坊上更有效。在以太坊上，gas 成本经常超过聚合节省的金额。在 Solana 上，即使 0.5% 的价格改善对用户来说也是纯利润。

### 9.3 为什么 Jupiter 赢得了 Solana 上的聚合器之战

1. **先发优势**：2021 年 10 月在 Solana DEX 格局碎片化时推出
2. **执行速度**：Metis/Iris 在 Solana 的 400ms 出块时间内计算路由
3. **产品广度**：永续合约、DCA、限价单 — 一站式服务
4. **社区飞轮**：大规模空投 → 数百万用户 → 交易量 → 收入 → 更多空投
5. **移动端 + Terminal**：竞争对手缺乏的分发渠道
6. **网络效应**：95% 份额意味着集成方优先在 Jupiter API 上构建，强化主导地位

---

## 10. 2025-2026 年发展动态

### 10.1 Catstanbul 大会（2025年1月25-26日）

Jupiter 首届大会，在伊斯坦布尔举行。主要公告：

- **30 亿 JUP 代币销毁**：销毁金属猫雕塑的象征性火焰仪式。总供应从 100 亿减至 70 亿
- **50% 费用回购计划**：协议收入 → JUP 购买 → Litterbox Trust
- **收购 Moonshot**：获得 meme 币 launchpad 的多数股权（在 TRUMP 代币发行期间激增）
- **收购 SonarWatch**：组合追踪器并入 Jupiter 生态
- **Ultra 模式**：实时滑点估算、动态优先费、优化交易处理
- **Jupiter Shield**：增强的用户资产安全工具
- **1000 万美元 AI 基金**：与 Eliza Labs 合作开发开源 AI
- **Jupnet 公告**：全链网络进入早期测试网

JUP 价格在这些公告后飙升 40%（$0.90 → $1.27）。

### 10.2 Ultra V3（2025年10月）

最重要的技术升级：

- **Iris 路由引擎**：路径查找快 100 倍
- **Juno 元聚合器**：自学习路由选择
- **JupiterZ RFQ**：每日 1 亿美元零滑点做市商执行
- **MEV 防护提升 34 倍**：Iris、JupiterZ 和 Jito 集成的组合
- **无 gas 支持**：Jupiter 可以为用户承担 gas
- **行业最佳滑点控制**：动态、每路由优化

### 10.3 Jupnet（全链网络）

Jupiter 最雄心勃勃的项目 — 跨链聚合层。

**三个组件**：
1. **DOVE 网络**：去中心化预言机验证执行器 — 验证者检索、达成共识并执行跨链交易
2. **全链账本**：托管所有连接链状态的单一分布式账本
3. **聚合去中心化身份（ADI）**：基于账户的用户体验替代以钱包为中心的交互

**状态**：2025 年 1 月进入早期测试网，公测目标 2025 年 Q4。

**愿景**：所有可交易资产（加密货币、股票、商品）通过一个界面访问，由加密货币基础设施驱动。

### 10.4 CatLumpurr 大会（2026年1月31日 - 2月2日）

Jupiter 第二届大会，吉隆坡。40 多个产品发布：

- **Jupiter Lend**：Solana 增长最快的协议 — 公开发布 8 天内达到 10 亿美元 TVL
- **Polymarket 集成**：Solana 上第一个也是唯一的 Polymarket 平台
- **Jupiter Global**：法币支付 — Jupiter Card、Global Send（USD SWIFT 转账到 200 多个国家）、QR Pay（亚太地区）
- **Jupiter Offerbook**：无许可 P2P 货币市场，使用任何链上资产（包括 RWA）借贷
- **ParaFi 3500 万美元投资**：首次外部融资，以 JupUSD 结算

### 10.5 2025-2026 年关键数据

| 指标 | 值 | 时期 |
|------|-----|------|
| 总收入 | 5.09 亿美元 | 2025 年 |
| Q3 现货交易量 | 1768 亿美元 | 2025 年 Q3 |
| Q2 交换次数 | 14 亿次 | 2025 年 Q2 |
| Q2 交易量 | 800 亿美元 | 2025 年 Q2 |
| TVL | 30 亿美元以上 | 2025 年 10 月 |
| JLP TVL | 25 亿美元 | 2025 年 10 月 |
| Jupiter Lend TVL | 10 亿美元以上 | 发布后 8 天 |
| 日交易量 | 12 亿美元以上 | 2025 年末 |
| 日活钱包 | 30 万 - 100 万以上 | 2025 年 |
| Q3 活跃钱包 | 840 万 | 2025 年 Q3 |
| 移动端用户 | 110 万以上 | 2025 年 |
| 聚合器市场份额 | 95% | Solana |
| 永续合约市场份额 | 66% | Solana |

---

## 11. 诚实评估

### 11.1 Jupiter 真正做得更好的地方

1. **路由质量**：Iris/Juno/JupiterZ 栈是真正的最佳水平。没有其他 Solana 聚合器能接近。
2. **产品广度**：现货 + 永续合约 + DCA + 限价单 + 借贷 + 移动端 + Launchpad 在一个应用中。无与伦比。
3. **MEV 防护**：Ultra V3 的 34 倍改进是真实且可衡量的。
4. **社区建设**：catdet 文化和大规模空投创造了真正的用户忠诚度。
5. **开发者体验**：Terminal SDK 和 Ultra API 使集成变得非常简单。
6. **自力更生的纪律**：在没有 VC 融资的情况下建立 5.09 亿美元收入，迫使产品与市场契合。

### 11.2 弱点与风险

1. **代币价格崩溃**：JUP 从 ATH 下跌 91.7%，尽管创纪录的指标。代币没有捕获与协议成功成比例的价值。持续解锁压倒回购。
2. **Solana 依赖**：Jupiter 100% 在 Solana 上。如果 Solana 出现宕机或失去相关性，Jupiter 也会受影响。Jupnet 是对冲，但还很早期。
3. **LIBRA 丑闻影响**：Meteora/Ben Chow/LIBRA 事件损害了信任。尽管 Jupiter 保持了距离，但共同创始人的联系令人不安。
4. **治理关闭**：自愿暂停治理 6 个月以上令人担忧。这表明团队无法让去中心化治理运作，转而选择了中心化控制。
5. **JLP 集中度**：10 个钱包持有 80% 的 JLP。鲸鱼退出可能动摇永续合约平台。
6. **Meme 币依赖**：Jupiter 很大一部分交易量来自 meme 币交易，这本质上是周期性的。当 meme 热潮消退，Jupiter 的交易量会急剧下降。
7. **收入可持续性**：5.09 亿美元毛收入，但只有约 1.22 亿美元净协议收入。其中 50% 用于未能支撑代币价格的回购。聚合（核心产品）的单位经济性很薄。

### 11.3 中心化担忧

- **95% 聚合器市场份额** — 这是垄断。如果 Jupiter 决定收取路由费或下架代币，没有可行的替代方案。
- **收购策略**（Moonshot、SonarWatch）被批评为「垄断行为」
- **团队在治理中的投票权** 大到足以影响结果 → 触发治理暂停
- **对一个项目的过度依赖** 对 Solana DeFi 来说：如果 Jupiter 失败，Solana DeFi 严重受影响
- **JupiterZ RFQ** 将执行集中到专业做市商，远离无许可 AMM 精神

### 11.4 模型可持续性

核心问题：**Jupiter 能维持 5 亿美元以上的年收入吗？**

- **看涨情况**：Solana 上的 DeFi 交易量持续增长。永续合约高利润。Jupnet 解锁跨链交易量。Jupiter Lend 增加另一收入线。全球支付带来法币用户。
- **看跌情况**：Meme 币季节结束。永续合约收入是周期性的（高杠杆 = 波动市场高交易量，平静市场低交易量）。95% 市场份额意味着增长只能来自总市场扩张，而非份额获取。代币解锁继续压制价格。

---

## 12. 可复用模式

### 12.1 产品模式

| 模式 | Jupiter 如何使用 | 可复用于 |
|------|-----------------|----------|
| **免费核心 → 付费功能** | 免费聚合器 → 付费永续合约/DCA/限价单 | 任何市场/平台 |
| **聚合器即分发** | 拥有路由层 → 拥有用户关系 | API 业务、市场 |
| **垂直整合** | 聚合器 + DEX + 永续合约 + 借贷 + 移动端 + Launchpad | 金融科技超级应用 |
| **可嵌入组件**（Terminal） | 其他 dApp 嵌入 Jupiter 交换 → Jupiter 获得交易量 | 任何 B2B2C 产品 |
| **移动端 + Web 对等** | 移动端相同功能，带 Apple Pay 入金 | 消费者金融科技 |

### 12.2 社区模式

| 模式 | Jupiter 如何使用 | 可复用于 |
|------|-----------------|----------|
| **大规模空投** | 200 万以上钱包，6.16 亿美元价值 → 即时社区 | 基于代币的项目 |
| **命名社区**（Catdets） | 身份 + 归属感 → 留存 | 任何社区产品 |
| **社区策划的 Launchpad** | DAO 投票决定什么发行 → 社区有主人翁感 | 市场、应用商店 |
| **年度大会**（Catstanbul/CatLumpurr） | 线下活动建立信任 + 产生媒体热度 | 任何有社区的项目 |
| **治理即营销** | ASR 奖励投票 → 用户感觉被投资 | DAO、合作社 |

### 12.3 技术模式

| 模式 | Jupiter 如何使用 | 可复用于 |
|------|-----------------|----------|
| **元聚合**（Juno） | 聚合聚合器 — 合并 AMM 路由 + RFQ + 第三方 | 任何路由/搜索系统 |
| **自学习路由** | 自动排除不良报价来源 | ML 驱动的推荐 |
| **预言机栈冗余** | 主要 + 2 个备用预言机，自动故障转移 | 任何依赖预言机的系统 |
| **RFQ + AMM 混合** | 顶级交易对用做市商，长尾用 AMM | 订单流优化 |
| **Keeper 网络** | 条件订单的去中心化执行 | 自动化平台 |
| **可嵌入 SDK** | 第三方集成的零配置组件 | B2B 分发 |

### 12.4 商业模式

| 模式 | Jupiter 如何使用 | 可复用于 |
|------|-----------------|----------|
| **自力更生 → 战略投资** | 4 年无 VC，然后从战略合作伙伴以市场价格获得 3500 万美元 | 资本高效的创业公司 |
| **回购并销毁** | 50% 收入 → 购买代币 → 锁定或销毁 | 代币价值积累 |
| **大会驱动路线图** | Catstanbul/CatLumpurr 作为产品发布活动 | 产品营销 |
| **危机管理** | LIBRA 期间迅速辞职 + 外部调查 | 任何面对丑闻的公司 |
| **免费 → 引入收费** | 从零费用开始，逐步引入平台费 | 市场变现 |

---

## 最新動態 (2026)

### 回购策略失败与零排放转型

Jupiter 在 2025 年全年花费超过 7000 万美元进行 JUP 回购，使用了大约一半的协议费用收入。尽管投入巨大，JUP 价格从峰值下跌约 89%，到 2026 年 1 月初跌至 $0.20-$0.22 区间。根本原因：每月约 5300 万 JUP 的代币解锁（计划持续到 2026 年 6 月）压倒了回购需求，回购仅覆盖了已解锁代币的约 6%。Solana 联合创始人 Anatoly Yakovenko 公开评论了在如此大规模的供应扩张面前回购的结构性无效。2026 年 1 月，联合创始人 Siong Ong 提议完全停止回购，将资金重新导向用户增长激励。到 2026 年 2 月，Jupiter DAO 对「净零排放」提案进行投票，内容包括取消 Jupuary、无限期暂停团队代币解锁、以及通过回购资金加速 Mercurial 利益相关者的解锁。投票吸引了 24,500 多个钱包，尽管 13,000 多个钱包支持空投选项，但 73.9% 的总投票权重（鲸鱼中为 81.7%）支持零排放路径。

### JupUSD 稳定币发布（2025 年 Q4）

Jupiter 与 Ethena Labs 合作推出 JupUSD。JupUSD 完全由 Ethena 的 USDtb 稳定币作为抵押，而 USDtb 本身由传统国债资产支持，包括 BlackRock 的 BUIDL 基金。团队计划添加 USDe 作为辅助支持资产以产生收益。JupUSD 原生集成于 Jupiter 的永续合约、借贷和交易界面。Jupiter 已宣布计划逐步将 JLP 池中约 7.5 亿美元的 USDC 转换为 JupUSD，创造显著的原生需求。ParaFi Capital 3500 万美元的投资以 JupUSD 结算，验证了该稳定币的机构信誉。

### Robinhood 集成（2025 年 11 月）

Robinhood 将 Jupiter 的 Swap API 集成到其原生加密钱包应用中，使 Robinhood 钱包用户能够在 Solana DeFi 生态中交易任何 SPL 代币。这是一个重要的分发里程碑 — Robinhood 的数百万零售用户现在通过 Jupiter 的 Ultra V3 引擎路由交易，进一步巩固了 Jupiter 作为 Solana 默认交换基础设施层的地位。

### Mobile V3（2026 年 1 月）

Jupiter 于 2026 年 1 月 1 日推出 Mobile V3，将其定位为移动端首个完全原生的专业交易终端。主要改进包括：交易费用比竞争对手移动应用低 10 倍、MEV 防护强 34 倍、更多代币对支持无 gas 交易（包括 meme 币对 meme 币的交换）、内置投资组合盈亏分析、详细代币分析（净压力、流动性变化、持有者分布）以及重新设计的发现流程。此次升级将移动端从「基于浏览器的 dApp 访问」转变为真正的原生交易终端。

### Polymarket 预测市场集成（2026 年 2 月）

Jupiter 成为首个也是唯一将 Polymarket 带到 Solana 的平台。Jupiter 应用内直接添加了专门的「预测」标签页，允许用户交易基于事件的预测市场，无需桥接稳定币或离开平台。预测市场仅在 2026 年 1 月就录得约 120 亿美元的交易量，产生超过 1100 万美元的链上费用 — 将其定位为与交换和永续合约并列的重要新产品垂直领域。

### Jupiter Lend 快速增长

Jupiter Lend 作为完全开源的借贷协议退出 Beta 测试，在公开发布仅 8 天内就达到 10 亿美元的供应资产 — 这是 Solana 上任何协议的最快增长速度。这增加了第三个主要收入垂直领域（与聚合和永续合约并列），深化了超级应用战略。

### DAO 治理恢复并进行结构性变革

经过 6 个月以上的治理暂停后，Jupiter DAO 于 2026 年初重新开放投票，净零排放提案是其首批重大投票之一。治理恢复采用了修订后的工具、更清晰的团队与社区投票权分离，并立即面对一个高风险决策，展示了 DAO 对代币模型进行通缩性结构改革的意愿。

## 核心要点

1. **Jupiter 是极少数在没有 VC 资金的情况下找到产品市场契合度的 DeFi 协议。** 四年的自力更生迫使他们构建人们真正使用的东西，而不是在融资演示中好看的东西。

2. **聚合器是护城河，永续合约是变现方式。** 95% 的聚合器份额给了 Jupiter 用户关系。永续合约（5.09 亿美元收入）是赚钱的地方。这种「免费核心 + 付费功能」模式是 DeFi 中最经过验证的模式。

3. **代币是阿喀琉斯之踵。** 尽管每项业务指标都在上升，JUP 下跌了 91.7%。持续的代币解锁和空投对持有者的稀释速度超过了回购所能抵消的速度。这是 Jupiter 尚未解决的结构性问题。

4. **大规模社区建设有效 — 直到失效。** catdet 文化、大规模空投和治理参与推动了爆发式增长。但治理不得不因信任崩溃而暂停，LIBRA 丑闻也展示了社区信任的局限性。

5. **垄断地位是双刃剑。** 95% 的市场份额给了 Jupiter 定价权和网络效应。但这也使 Solana DeFi 成为依赖一个团队执行力和诚信的单点故障。

6. **从聚合器 → 超级应用 → 全链平台**（Jupnet）的演进是 DeFi 中最雄心勃勃的路线图。如果 Jupnet 成功，Jupiter 将成为跨链金融操作系统。如果失败，他们将在日益多链的世界中仍然是一个 Solana 专属玩家。

---

## References

### Routing & Architecture

- [Jupiter Routing Engines — Developer Docs](https://dev.jup.ag/docs/routing)
- [Jupiter Metis Routing: Optimizing Swaps on Solana — Jupiter Research Forum](https://discuss.jup.ag/t/jupiter-metis-routing-optimizing-swaps-on-solana/22152)
- [Jupiter Ultra V3: Technical and Strategic Analysis — Ayush Kumar Jha (Medium)](https://medium.com/@ayushkmrjha/jupiter-ultra-v3-a-technical-and-strategic-analysis-of-the-end-to-end-trading-engine-0a61e9f244df)
- [Solana DeFi Deep Dives: Jupiter Ultra V3 — Dhru Patel (Medium)](https://medium.com/@Scoper/solana-defi-deep-dives-jupiter-ultra-v3-next-gen-dex-aggregator-late-2025-2cef75c97301)
- [All about Jupiter's Aggregation Engine: Juno — Jupiter Support](https://support.jup.ag/hc/en-us/articles/18735544617628-All-about-Jupiter-s-Aggregation-Engine-Juno)
- [Juno: Next-Gen Aggregation Engine — Jupiter Legion](https://www.jupiterlegion.net.ng/2025/04/juno-next-gen-aggregation-engine.html)
- [Jupiter Launches Ultra V3 — The Block](https://www.theblock.co/press-releases/375289/jupiter-launches-ultra-v3-the-ultimate-trading-engine-for-solana)
- [Jupiter Ultra V3: MEV Protections and Gasless Support — The Block](https://www.theblock.co/post/375184/solana-decentralized-exchange-aggregator-jupiter-unveils-ultra-v3-improved-trade-execution-mev-protections-gasless-support)
- [Solana DEX Aggregator Comparison 2026 — Carbium Blog](https://blog.carbium.io/solana-dex-aggregator-comparison-2026-carbium-vs-jupiter-vs-titan-vs-okx-vs-dflow/)

### Ultra Swap API & Developer Tools

- [Ultra Swap API Overview — Jupiter Developers](https://dev.jup.ag/docs/ultra)
- [V6 Swap API — Jupiter Developer Docs](https://hub.jup.ag/docs/apis/swap-api)
- [Self-hosted V6 Swap API — Jupiter Developers](https://hub.jup.ag/docs/apis/self-hosted)
- [How to Use Jupiter API to Create a Solana Trading Bot — QuickNode](https://www.quicknode.com/guides/solana-development/3rd-party-integrations/jupiter-api-trading-bot)
- [How to Swap Tokens With Jupiter Ultra API — QuickNode](https://www.quicknode.com/guides/solana-development/3rd-party-integrations/jupiter-ultra-swap)
- [jupiter-swap-api-client — GitHub](https://github.com/jup-ag/jupiter-swap-api-client)
- [Jupiter Python SDK — GitHub](https://github.com/0xTaoDev/jupiter-python-sdk)
- [Integrate Swap Terminal — Jupiter Developers](https://dev.jup.ag/docs/tool-kits/terminal)
- [Jupiter Terminal](https://terminal.jup.ag/)
- [Jupiter API: Solana Swaps, Limit Orders, Routing Data — Bitquery](https://docs.bitquery.io/docs/blockchain/Solana/solana-jupiter-api/)

### Perpetuals & JLP

- [Understanding How Jupiter Perps Works — Jupiter Station](https://station.jup.ag/guides/perpetual-exchange/how-it-works)
- [Understanding Jupiter Perps — Jupiter Station](https://station.jup.ag/guides/perpetual-exchange/overview)
- [Perps Quickstart — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/18734952106908-Perps-Quickstart)
- [What is JLP? — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/18453802442268-What-is-JLP)
- [Fees — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/18735045234588-Fees)
- [Price Oracles — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/19355326396700-Price-Oracles)
- [Jupiter Perps: A Smarter Way to Trade Perpetuals on Solana — Kripto Dnevnik](https://kriptodnevnik.com/jupiter-perps-a-smarter-way-to-trade-perpetuals-on-solana/)
- [A Deep Dive into the Jupiter Perpetual Exchange — Decentral Park Capital](https://decentralparkcapital.substack.com/p/a-deep-dive-into-the-jupiter-perpetual-0d2)
- [Mastering Jupiter Perps — Jupiter Legion](https://www.jupiterlegion.net.ng/2025/10/mastering-jupiter-perps-beginners-guide.html)
- [Jupiter Perp DEX Review — Coin360](https://coin360.com/review/jupiter-perp)
- [Three Major Perp DEX Mechanics: Hyperliquid vs Jupiter vs GMX — PANews](https://www.panewslab.com/en/articles/6jeoramg)
- [JLP: Jupiter's Juicy Yield (PDF) — Contentful](https://assets.ctfassets.net/i0qyt2j9snzb/4ueTPh7PwgopwILuLbhoYL/2fc57577db5f223edfaacf20bd557de1/JLP_-_Jupiter-s_Juicy_Yield.pdf)
- [What Is Jupiter Perps LP (JLP) And How Does It Work? — CoinMarketCap](https://coinmarketcap.com/cmc-ai/jupiter-perps-lp/what-is/)
- [JLP APY Prediction Principle — Bitget News](https://www.bitget.com/news/detail/12560604200685)

### MEV Protection

- [Meow on MEV Protection — X/Twitter](https://x.com/weremeow/status/1831754731045159272)
- [Jupiter's MEV Protection Architecture — Bravian Oyaro (Medium)](https://medium.com/@bnyamosie/jupiters-mev-protection-architecture-how-solana-traders-get-safer-onchain-execution-34a5bccb33be)
- [Solana MEV Wars: Understanding the Game — Astralane (Medium)](https://medium.com/@astralaneio/solana-mev-wars-understanding-the-game-protecting-your-alpha-43ba5e4846e9)
- [What is MEV and How to Protect on Solana — QuickNode](https://www.quicknode.com/guides/solana-development/defi/mev-on-solana)
- [Solana Users Paying Millions to Stop Bot Attacks — DL News](https://www.dlnews.com/articles/defi/solana-users-use-jito-to-stop-sandwich-attacks-and-mev/)

### JUP Token & Airdrops

- [Jupiter Airdrop: JUP Token Guide (2026) — Phantom](https://phantom.com/learn/crypto-101/jupiter-jup-airdrop)
- [Jupiter's $616M Solana Airdrop: 2025 JUP Token Guide — KuCoin](https://www.kucoin.com/news/articles/jupiter-s-616m-solana-airdrop-the-2025-jup-token-guide)
- [Everything You Need to Know About Jupuary — CoinGecko](https://www.coingecko.com/learn/everything-you-need-to-know-jupiter-s-upcoming-airdrop-jupuary)
- [Jupuary 2025: Upcoming Airdrop — Bitrue FAQ](https://support.bitrue.com/hc/en-001/articles/41564995596697-Jupiter-Jupuary-2025-Everything-You-Need-to-Know-About-the-Upcoming-Airdrop)
- [Jupiter Announces $575M Airdrop — Unchained](https://unchainedcrypto.com/jupiter-announces-details-of-its-upcoming-airdrop-of-575-million-worth-of-jup-tokens/)
- [Jupiter (JUP) Tokenomics, Supply & Release Schedule — Tokenomist](https://tokenomist.ai/jupiter-exchange-solana)
- [Jupiter Airdrop — Airdrops.io](https://airdrops.io/jupiter/)
- [Jupiter Price (JUP) — CoinMarketCap](https://coinmarketcap.com/currencies/jupiter-ag/)
- [Jupiter Price (JUP) — CoinGecko](https://www.coingecko.com/en/coins/jupiter)

### Buyback & Token Burns

- [Jupiter Spikes 40% After 50% Fee Buyback Announcement — The Block](https://www.theblock.co/post/337071/jupiter-spikes-40-as-founder-says-50-of-fees-will-go-to-token-buybacks)
- [Jupiter to Buyback JUP with 50% of Fees — CryptoSlate](https://cryptoslate.com/jupiter-to-buyback-jup-tokens-with-50-of-fees-starting-next-week/)
- [Discussion: Burning the JUP Tokens in Litterbox Trust — Jupiter Research Forum](https://discuss.jup.ag/t/discussion-burning-the-jup-tokens-in-the-litterbox-trust/39569)
- [Jupiter Community Votes to Burn 130M JUP — BitcoinEthereumNews](https://bitcoinethereumnews.com/finance/jupiter-community-votes-to-burn-130-million-jup-a-fresh-start-for-the-ecosystem/)
- [The Silent Loop: Buybacks, Unlocks, Litterbox — Jupiter Research Forum](https://discuss.jup.ag/t/the-silent-loop-buybacks-unlocks-litter-box-and-the-search-for-purpose-in-jup/37859)
- [Jupiter's Paradox: Profitable Platform, Underperforming Token — Bokiko (Medium)](https://medium.com/@bokiko/jupiters-paradox-a-profitable-platform-an-underperforming-token-30a8d34c7410)
- [Why Jupiter's JUP Buyback Struggled Despite $70M Spent — Crypto.news](https://crypto.news/jupiter-jup-token-buyback-unlocks-solana-2026/)
- [Anatoly Yakovenko Addresses Jupiter's $70 Million Buybacks — BeInCrypto](https://beincrypto.com/jupiter-buyback-solana-token-unlocks/)
- [Jupiter Founder Questions $70 Million Buyback Strategy — Yellow](https://yellow.com/news/jupiter-founder-questions-dollar70-million-buyback-strategy-after-89-price-decline)
- [Jupiter Exchange Halts JUP Buybacks, Pivots to Growth — Bitget](https://www.bitget.com/news/detail/12560605130545)

### DAO & Governance

- [Jupiter DAO Voting](https://vote.jup.ag/)
- [ASR — Vote | Jupiter](https://vote.jup.ag/asr)
- [What is the Jupiter DAO? — Jupiter Exchange Support](https://support.jup.ag/hc/en-us/articles/20007985083420-What-is-the-Jupiter-DAO)
- [Understanding Jupiter DAO Proposals — Beyond The Swap (Medium)](https://beyondtheswap.medium.com/understanding-jupiter-dao-proposals-ef3eacc5572b)
- [Jupiter Halts DAO Voting Until 2026 — AInvest](https://www.ainvest.com/news/jupiter-jup-halts-dao-voting-2026-governance-reform-2506/)
- [Jupiter DAO Suspends Governance Votes — The Block](https://www.theblock.co/post/358988/dao-behind-dex-aggregator-jupiter-suspends-governance-votes-until-early-2026-amid-community-concerns)
- [Jupiter DAO Pauses Voting After Backlash — The Defiant](https://thedefiant.io/news/defi/jupiter-dao-pauses-voting-after-backlash-over-team-s-voting-power)
- [Solana DEX Jupiter Pauses DAO Votes — CoinDesk](https://www.coindesk.com/business/2025/06/20/solana-dex-jupiter-pauses-dao-votes-citing-breakdown-in-trust)
- [Jupiter DAO Suspends Governance: Staking Rewards Continue — Solana Floor](https://solanafloor.com/news/jupiter-dao-suspends-governance-staking-rewards-continue)
- [ASR (Active Staking Rewards) Notes — Jupiter Research Forum](https://discuss.jup.ag/t/asr-active-staking-rewards-notes/12032/157)
- [Jupiter (JUP) Staking Rewards Calculator — StakingRewards](https://www.stakingrewards.com/asset/jupiter-exchange-solana)
- [Proposal: Net-Zero Emissions — Jupiter Research Forum](https://discuss.jup.ag/t/proposal-net-zero-emissions/39948)
- [Over 81% of Whale Voting Weight Backs Zero Emissions — StepData](https://stepdata.substack.com/p/over-81-of-whale-voting-weight-backs)
- [Jupiter DAO Opens Vote on Canceling Jupuary Airdrops — Cryptopolitan](https://www.cryptopolitan.com/jupiter-dao-vote-canceling-jupuary-airdrops/)
- [JUP Moves Toward Net-Zero Emissions — BitcoinEthereumNews](https://bitcoinethereumnews.com/tech/jup-moves-toward-net-zero-emissions-in-dao-proposal/)

### LFG Launchpad

- [LFG Launchpad — Jupiter](https://lfg.jup.ag/)
- [$WEN — LFG Jupiter](https://lfg.jup.ag/wen)
- [$ZEUS — LFG Jupiter](https://lfg.jup.ag/zeus)
- [Jupiter LFG Launchpad (Archived) — Jupiter Research Forum](https://discuss.jup.ag/t/archived-jupiter-lfg-launchpad/21723)
- [DLMM: The Building Block of Jupiter's LFG Launchpad — Florent Koenig (Medium)](https://medium.com/@florentkoenig/dlmm-the-building-block-of-jupiters-lfg-launchpad-48633fa4cb47)
- [Jupiter LFG: A Game-Changer for Solana Projects in 2025 — Gate.io](https://www.gate.com/learn/articles/jupiter-lfg-launchpad-a-game-changer-for-solana-projects/2047)
- [Jupiter LFG — Top Launchpad Projects by ROI — Chain Broker](https://chainbroker.io/platforms/jupiter-lfg/)
- [Jupiter LFG — CryptoRank](https://cryptorank.io/fundraising-platforms/jupiter-lfg)
- [Jupiter LFG — CoinLaunch](https://coinlaunch.space/launchpads/jupiter-lfg/)

### Team & LIBRA Scandal

- [Ben Chow — Jupiter Exchange Co-Founder — Phantom Podcast](https://podcast.phantom.app/episodes/ben-chow-jupiter-exchange-co-founder-ep-2)
- [Lightspeed Podcast: Jupiter — Aggregator Fueling Solana's GDP (Meow) — Blockworks](https://blockworks.co/podcast/lightspeed/dfae1328-9498-11ee-9434-d34510648730)
- [LIBRA, Solana Drama: Meteora Co-Founder Resigns — Cointelegraph](https://cointelegraph.com/news/libra-solana-meltdown-meteora-jupiter)
- [Meteora CEO Ben Chow Resigns Amid LIBRA Allegations — Blockhead](https://www.blockhead.co/2025/02/19/meteora-ceo-ben-chow-resigns-amid-libra-insider-trading-allegations-2/)
- [How Jupiter and Meteora Fueled LIBRA's Collapse — Crypto News](https://crypto.news/libra-solana-jupiter-meteora-scandal/)
- [Jupiter and Meteora Deny Insider Trading — BitcoinEthereumNews](https://bitcoinethereumnews.com/finance/jupiter-and-meteora-deny-any-insider-trading-or-financial-misconduct/)
- [Meteora Faces Insider Trading Allegations — The Defiant](https://thedefiant.io/news/defi/meteora-faces-insider-trading-allegations-as-ceo-ben-chow-resigns)
- [Jupiter Project — RootData](https://www.rootdata.com/Projects/detail/Jupiter?k=MTAwMDA%3D)
- [Jupiter — Messari](https://messari.io/project/jupiter-exchange)

### Jupiter Ape & Meme Coin Trading

- [Jupiter Unveils Ape — Solana Floor](https://solanafloor.com/news/jupiter-ape-safer-smoother-cheaper-way-trade-meme-coins)
- [Pump.fun — Wikipedia](https://en.wikipedia.org/wiki/Pump.fun)
- [98% of Tokens on Pump.fun Have Been Rug Pulls — CoinDesk](https://www.coindesk.com/business/2025/05/07/98-of-tokens-on-pump-fun-have-been-rug-pulls-or-an-act-of-fraud-new-report-says)

### Market Position & Competition

- [Raydium, Jupiter, Orca, Meteora: Who Will Dominate Solana DEX? — PANews](https://www.panewslab.com/en/articles/5zrwdqh9)
- [In-depth Analysis of Four Major Solana DEXs — ChainCatcher](https://www.chaincatcher.com/en/article/2168403)
- [Best DEX for Solana 2025: Orca, Raydium, or Jupiter? — SnapX](https://blog.snapx.co/best-dex-for-solana-in-2025-orca-raydium-or-jupiter)
- [Jupiter vs Rivals: DEX Aggregators Compared — BestDapps](https://bestdapps.com/blogs/news/jupiter-vs-rivals-dex-aggregators-compared)
- [Best DEX Aggregator Platforms 2026 — CoinsClone](https://www.coinsclone.com/best-dex-aggregator/)

### 2025-2026 Developments

- [Jupiter Announces Major Acquisitions at Catstanbul — Blockhead](https://www.blockhead.co/2025/01/27/jupiter-announces-major-acquisitions-token-buyback-program-platform-overhaul-at-catstanbul-conference/)
- [Jupiter Token Price Rises 40% After Token Burn — CoinMarketCap](https://coinmarketcap.com/academy/article/jupiters-token-price-rises-40percent-after-announcing-3-billion-token-burn-and-50percent-fee-buyback-plan)
- [Jupiter's 30% Supply Cut — Unchained](https://unchainedcrypto.com/jupiters-30-supply-cut-of-jup-worth-3-billion-to-be-celebrated-with-a-physical-fire-ritual/)
- [Jupiter Takes Majority Stake in Moonshot — CryptoBriefing](https://cryptobriefing.com/jupiter-acquires-moonshot-sonarwatch/)
- [Jupiter Acquires Majority Stake in Moonshot — Cointelegraph](https://cointelegraph.com/news/solana-dex-jupiter-acquires-majority-stake-in-moonshot)
- [Jupiter Acquires Moonshot After $400M Trading Surge — Bitget](https://www.bitget.com/news/detail/12560604528488)

### Jupnet & Omnichain

- [Jupiter Unveils Omnichain Network Jupnet — Blocmates](https://www.blocmates.com/news-posts/jupiter-unveils-omnichain-network-jupnet)
- [Jupnet — Meow.bio](https://meow.bio/jupnet)
- [Jupnet Is a Proposed Network — Jupiter Legion](https://www.jupiterlegion.net.ng/2025/04/did-you-know-jupnet-is-proposed-network.html)
- [Jupiter Launches Omnichain Network Jupnet — Bitget](https://www.bitget.com/news/detail/12560604526190)
- [Jupiter: Will Launch Full-Chain Network Jupnet — ChainCatcher](https://www.chaincatcher.com/en/article/2164722)

### CatLumpurr 2026 & ParaFi Investment

- [Jupiter Brings Polymarket to Solana, Lands $35M — CoinDesk](https://www.coindesk.com/markets/2026/02/02/jupiter-brings-polymarket-to-solana-and-lands-usd35-million-investment-deal)
- [Jupiter Secures $35M ParaFi Capital Deal — Blockhead](https://www.blockhead.co/2026/02/04/jupiter-secures-first-ever-outside-investment-with-35m-parafi-capital-deal/)
- [Jupiter Secures $35M ParaFi Investment — Crypto Reporter](https://www.crypto-reporter.com/press-releases/jupiter-secures-35m-strategic-investment-from-parafi-capital-to-accelerate-onchain-financial-infrastructure-121740/)
- [ParaFi Invests $35M in Solana DeFi Platform Jupiter — FintechFutures](https://www.fintechfutures.com/venture-capital-funding/parafi-invests-35m-in-solana-defi-platform-jupiter)
- [Polymarket Expands to Solana Through Jupiter — The Block](https://www.theblock.co/post/387945/jupiter-polymarket-solana)
- [Jupiter Integrates Polymarket — Invezz](https://invezz.com/news/2026/02/02/jupiter-integrates-polymarket-bringing-on-chain-prediction-markets-to-solana/)

### JupUSD Stablecoin

- [Solana's Jupiter to Develop JupUSD Stablecoin With Ethena Labs — CoinDesk](https://www.coindesk.com/web3/2025/10/08/solana-s-jupiter-to-develop-jupusd-stablecoin-with-backing-from-ethena-labs)
- [Jupiter Launches JupUSD Stablecoin With Ethena Labs on Solana — Blockworks](https://blockworks.co/news/jupiter-jupusd-stablecoin)
- [Jupiter Launches JupUSD Stablecoin on Solana With Ethena Labs — CoinMarketCap](https://coinmarketcap.com/academy/article/jupiter-launches-jupusd-stablecoin-on-solana-with-ethena-labs)
- [Jupiter Teams Up With Ethena to Launch JupUSD Stablecoin — Unchained](https://unchainedcrypto.com/jupiter-teams-up-with-ethena-to-launch-jupusd-stablecoin/)

### Mobile V3

- [Jupiter Rolls Out Mobile V3 With Native Pro Trading Tools — Crypto.news](https://crypto.news/jupiter-launches-mobile-v3-native-pro-trading-2026/)
- [Jupiter Introduces V3 Mobile Trading Terminal — CryptoBriefing](https://cryptobriefing.com/jupiter-mobile-v3-enhanced-trading/)
- [Jupiter Mobile V3 Launch: First Native Pro-Trading App on Solana — EGW News](https://egw.news/crypto/news/31581/jupiter-announces-mobile-v3-the-first-full-fledged-henGAxm74)

### Robinhood Integration

- [Can Robinhood's Jupiter Integration Reassert the Aggregator's Dominance? — Solana Floor](https://solanafloor.com/news/robinhoods-jupiter-integration-reassert-aggregator-dominance)
- [Jupiter Unveils JupUSD Stablecoin and Major DeFi Upgrades at Solana Breakpoint 2025 — Coinpedia](https://coinpedia.org/news/jupiter-unveils-jupusd-stablecoin-and-major-defi-upgrades-at-solana-breakpoint-2025/)

### Revenue, TVL & Stats

- [Jupiter — DefiLlama](https://defillama.com/protocol/jupiter)
- [Jupiter Aggregator — DefiLlama](https://defillama.com/protocol/jupiter-aggregator)
- [Jupiter Revenues Surge on Solana — MEXC News](https://www.mexc.com/news/139989)
- [Number One DEX on Solana: Data on Volume, TVL, Users — Hive](https://hive.blog/jupiter/@dalz/a-look-at-the-number-one-dex-on-solana-jupiter-or-data-on-trading-volume-tvl-users-top-pairs-and-more-or-apr-2025)
- [Jupiter's Ecosystem Dominance in Solana DeFi — AInvest](https://www.ainvest.com/news/jupiter-ecosystem-dominance-solana-defi-flywheel-driven-play-2026-2512/)

### Centralization & Criticism

- [Jupiter's Moves in Solana: Mixed Bag — OneSafe](https://www.onesafe.io/blog/jupiter-strategic-moves-solana-decentralization)
- [Jupiter's Acquisition Spree Sparks Dominance Concerns — CoinDesk](https://www.coindesk.com/business/2025/01/27/jupiter-s-acquisition-spree-buyback-plan-spark-solana-ecosystem-dominance-concerns)
- [Jupiter Pauses DAO Voting Amid Trust Breakdown — DL News](https://www.dlnews.com/articles/defi/solana-exchange-jupiter-pauses-dao-voting-until-2026/)

### General Overview

- [What Is Jupiter (JUP)? Solana's Largest DeFi Superapp — CoinGecko](https://www.coingecko.com/learn/what-is-jupiter-crypto-solana)
- [What is Jupiter (JUP)? — Bitcoin.com](https://www.bitcoin.com/get-started/what-is-jupiter-solana/)
- [Jupiter's Evolution: Solana DEX Reshaping DeFi — OKX](https://www.okx.com/en-us/learn/jupiter-solana-dex-defi-evolution)
- [Jupiter's Evolution: From Aggregator to Full-Stack DeFi — OKX](https://www.okx.com/en-us/learn/jupiter-solana-dex-defi-platform)
- [What Is Jupiter Exchange? Guide — Nansen](https://www.nansen.ai/post/what-is-jupiter-exchange)
- [Jupiter DEX Review — Coin360](https://coin360.com/review/jupiter)
- [Jupiter (JUP): Solana Liquidity Engine — Laika Labs](https://laikalabs.ai/market-intelligence/jupiter-jup-solana-liquidity-engine-guide)
- [Jupiter on Solana: Project Reviews — Solana Compass](https://solanacompass.com/projects/jupiter)
- [Jupiter — IQ.wiki](https://iq.wiki/wiki/jupiter)

### DCA & Limit Orders

- [How Dollar Cost Averaging Works — Jupiter Station](https://betastation.jup.ag/guides/dca/how-dca-work)
- [How Limit Orders Work — Jupiter Station](https://station.jup.ag/guides/limit-order/how-lo-work)
- [How to Use DCA on Jupiter — Jupiter Station](https://station.jup.ag/guides/dca/how-to-dca)

### Jupiter Lock

- [Lock | Jupiter](https://lock.jup.ag/)
- [jup-lock — GitHub](https://github.com/jup-ag/jup-lock)

### Jupiter Mobile

- [Jupiter Mobile — App Store](https://apps.apple.com/us/app/jupiter-mobile-solana-wallet/id6484069059)
- [Jupiter Mobile — Google Play](https://play.google.com/store/apps/details?id=ag.jup.jupiter.android)
- [Jupiter Mobile — jup.ag](https://jup.ag/mobile)
- [Solana's Top Swap Venue Jupiter Released a Mobile App — Blockworks](https://blockworks.co/news/solana-venue-jupiter-released-mobile-app)

### Community

- [Jupiter's Success: Community-First Approach — VALR Blog](https://blog.valr.com/blog/jupiters-success-a-community-first-approach-to-growth-ft-meow-founder-of-jupiter-on-solana)
- [Catalyzing Sustainable Growth — Jupiter Research Forum](https://discuss.jup.ag/t/catalyzing-sustainable-growth-a-strategic-initiative-for-jupiter-community-adoption-liquidity-depth/39497)
- [Catstanbul 2025 — Luma](https://luma.com/catstanbul)
- [Catstanbul 2025 — jup.ag](https://catstanbul.jup.ag/)
