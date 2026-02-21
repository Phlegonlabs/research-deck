# ACP：代理商务协议 — Stripe 与 OpenAI 如何构建代理结账

## 这是什么？

ACP（Agentic Commerce Protocol，代理商务协议）是由 **Stripe 和 OpenAI** 联合开发的开放标准，让 AI 代理能够代表用户完成购买。它是 **ChatGPT 即时结账（Instant Checkout）** 背后的协议 — 用户无需离开对话即可购买商品。

核心洞察：现有的支付轨道是为人类点击按钮设计的。ACP 将结账重新设计为代理可调用的**程序化 API**。五个 REST 端点处理整个购买生命周期，一个名为 **Shared Payment Token（SPT，共享支付令牌）** 的新支付原语确保代理永远看不到你的信用卡。

**状态**：生产环境运行中。ChatGPT 即时结账于 2026 年 2 月 16 日面向美国用户（Plus、Pro、Free）上线。规范以 Apache 2.0 许可在 GitHub 开源。已发布四个规范版本（2025-09-29 → 2026-01-30）。

## 为什么重要

代理商务问题分三个部分：

| 问题 | ACP 之前 | 有了 ACP |
|------|---------|----------|
| 代理如何购物？ | 屏幕抓取结账表单 | 调用结构化 REST API |
| 谁持有支付凭证？ | 代理或用户重新输入 | SPT — 范围化、一次性令牌 |
| 谁是记录商户？ | 不明确 — 代理？平台？ | 商户保留完全控制权 |

ACP **不是**市场平台。OpenAI 不是卖家。商户保留客户关系、接受/拒绝订单、处理履约和退款。代理只是一个高级结账助手。

## 架构

### 四个角色

```mermaid
flowchart LR
    Buyer["买家 (人类)\n批准购买"] --> Agent["AI 代理 (ChatGPT)\n编排结账流程"] --> Merchant["商户 (卖家)\n履行订单"] --> PSP["PSP (Stripe)\n处理支付"]
```

**关键分离**：代理编排但从不处理资金。PSP（Stripe）处理资金但看不到对话。商户控制商品目录和履约但不需要理解 AI。

### 五个端点

| 端点 | 方法 | 功能 |
|------|------|------|
| 创建结账 | `POST /checkout_sessions` | 代理发送商品 + 买家信息 → 商户返回结账状态 |
| 获取结账 | `GET /checkout_sessions/{id}` | 读取当前结账状态（价格、选项、状态） |
| 更新结账 | `POST /checkout_sessions/{id}` | 修改商品、收货地址、履约选项 |
| 完成结账 | `POST /checkout_sessions/{id}/complete` | 提交 SPT → 商户处理支付 → 创建订单 |
| 取消结账 | `POST /checkout_sessions/{id}/cancel` | 中止交易，可附 `intent_trace` |

**状态机：**
```mermaid
stateDiagram-v2
    not_ready_for_payment --> ready_for_payment
    not_ready_for_payment --> in_progress
    not_ready_for_payment --> authentication_required
    ready_for_payment --> completed
    ready_for_payment --> canceled
    in_progress --> completed
    in_progress --> canceled
    authentication_required --> completed: 3DS flow
    authentication_required --> canceled
```

必须请求头：`API-Version: 2026-01-30`。所有 POST 请求需要 `Idempotency-Key`（UUID v4）。

### 完整购买流程

```mermaid
sequenceDiagram
    participant B as 买家
    participant C as ChatGPT
    participant M as 商户 API
    participant S as Stripe (PSP)
    B->>C: "帮我买一件蓝色夹克"
    C->>M: POST /checkouts {items, buyer}
    M-->>C: 结账状态 (价格、配送选项)
    Note over B,C: 展示选项
    B->>C: "送到家里"
    C->>M: PUT /checkouts/:id {fulfillment_opt}
    M-->>C: 更新后状态
    B->>C: "好的，买吧"
    C->>S: 委托支付 {card, billing, risk signals}
    S-->>C: SPT (spt_xxx)
    C->>M: POST /complete {token: spt_xxx}
    M->>S: PaymentIntent
    S-->>M: 确认
    M-->>C: 订单确认
    C-->>B: "已下单！在这里追踪："
```

## 共享支付令牌（SPT）— 关键创新

SPT 是 **Stripe 的新支付原语**，解决了代理-凭证问题。核心思想：不是把信用卡给代理，而是 Stripe 发行一个**范围化、一次性、有时限的令牌**，只能用于一笔特定购买。

### SPT 属性

| 属性 | 详情 |
|------|------|
| 一次性 | 每笔交易一个令牌。无法重放 |
| 有时限 | `expires_at` 字段（RFC 3339）。截止时间后令牌失效 |
| 商户范围 | 绑定到特定 `merchant_id`。在其他地方无效 |
| 金额上限 | 许可对象中的 `max_amount`。不能超额收费 |
| 凭证隔离 | 代理和商户都看不到原始卡号 |

### 令牌格式

```
spt_123abc...
```

前缀 `spt_` 标识为共享支付令牌（类似 Stripe 的 `pi_`、`pm_`、`tok_` 约定）。

### 委托支付流程

这是技术上最有趣的部分。三阶段凭证委托：

**阶段 1：OpenAI → PSP (Stripe)**
```json
{
  "payment_method": {
    "card_number_type": "network_token",
    "cryptogram": "AJkBAfd...",
    "eci_value": "05",
    "checks_performed": ["cvc_check"],
    "display_brand": "visa",
    "display_last4": "4242"
  },
  "billing_address": { "...": "..." },
  "risk_signals": {
    "type": "stripe_risk",
    "score": 12,
    "action": "allow"
  },
  "allowance": {
    "checkout_session_id": "cs_abc123",
    "merchant_id": "merch_xyz",
    "currency": "usd",
    "max_amount": 9999,
    "expires_at": "2026-02-19T12:00:00Z",
    "reason": "one_time"
  },
  "idempotency_key": "idk_unique123"
}
```

**阶段 2：PSP → OpenAI**
```json
{
  "token": "spt_123abc",
  "created_at": "2026-02-19T11:00:00Z",
  "metadata": { "checkout_session_id": "cs_abc123" }
}
```

**阶段 3：OpenAI → 商户（通过完成结账）**
```json
{
  "payment_data": {
    "token": "spt_123abc",
    "provider": "stripe",
    "billing_address": { "...": "..." }
  }
}
```

**为什么用网络令牌而不是 FPAN？** 网络令牌（Visa/Mastercard 的令牌化卡号）减少了商户的 PCI 合规范围。`card_number_type` 字段支持 `"fpan"`（完整 PAN）和 `"network_token"`，但强烈推荐网络令牌。

### 安全保证

> **每个角色看到什么：**
>
> | 角色 | 看到 |
> |------|------|
> | 买家 | 完整卡片详情（他们拥有） |
> | ChatGPT | 仅 display_last4 + display_brand |
> | Stripe | 完整卡片 + 风险信号 |
> | 商户 | 仅 SPT 令牌（spt_xxx） |
>
> 除了用户和 Stripe，没人看到真实卡号。代理看到展示信息。商户只看到不透明令牌。

## 支付处理器框架（v4 — 2026-01-30）

最新规范版本用处理器系统替代了简单的 `payment_methods`：

| 处理器 | 类型 | 描述 |
|--------|------|------|
| `dev.acp.tokenized.card` | 外部 | 标准信用/借记卡，通过 Stripe SPT |
| `dev.acp.seller_backed.saved_card` | 卖家端 | 在商户处保存的卡片 |
| `dev.acp.seller_backed.gift_card` | 卖家端 | 商户发行的礼品卡 |
| `dev.acp.seller_backed.points` | 卖家端 | 积分/奖励点数 |
| `dev.acp.seller_backed.store_credit` | 卖家端 | 商户账户余额 |

卖家端处理器设置 `requires_delegate_payment: true` 但 `requires_pci_compliance: false` — 令牌请求不携带实际凭证，只有一个处理器 ID 供商户内部解析。这使商户可以在代理结账中同时接受积分、礼品卡和信用卡。

## 结账数据模型

### 请求/响应模式

每个结账响应包含以下部分：

```
Checkout Object（结账对象）
├── id: "cs_abc123"
├── status: "ready_for_payment"
├── currency: "usd"
├── buyer（买家）
│   ├── first_name, last_name（必填）
│   ├── email（必填）
│   └── phone_number（可选）
├── payment_provider（支付提供商）
│   ├── provider: "stripe"
│   └── supported_payment_methods: ["card"]
├── line_items[]（商品行项）
│   ├── id, item（产品详情）
│   ├── base_amount, discount, tax
│   └── subtotal, total
├── fulfillment_address（收货地址）
│   ├── name, line_one, line_two
│   ├── city, state, country（ISO 3166-1 alpha-2）
│   └── postal_code
├── fulfillment_options[]（履约选项）
│   ├── type: "shipping" | "digital"
│   ├── title, subtitle, carrier
│   ├── earliest/latest_delivery_time（ISO 8601）
│   └── subtotal, tax, total
├── totals[]（合计）
│   ├── type: items_base_amount | items_discount | subtotal |
│   │         discount | fulfillment | tax | fee | total
│   ├── display_text
│   └── amount（整数，美分）
├── messages[]（消息）
│   ├── type: "info" | "error"
│   ├── code: missing | invalid | out_of_stock |
│   │         payment_declined | requires_sign_in | requires_3ds
│   └── content（plain | markdown）
├── links[]（链接）
│   ├── type: terms_of_use | privacy_policy | seller_shop_policies
│   └── url
└── order（完成后）
    ├── id, checkout_session_id
    └── permalink_url
```

**所有金额为整数美分**（例如 $99.99 = 9999）。三字母 ISO 货币代码，小写。

### 订单事件（Webhook）

结账完成后，商户通过 webhook 通知代理订单状态变化：

| 事件 | 状态 |
|------|------|
| `order_created` | created |
| `order_updated` | manual_review → confirmed → shipped → fulfilled |
| `order_updated` | canceled（任何时候） |

退款类型：`store_credit`（商店积分）或 `original_payment`（原路退回），带金额。

### 错误处理

四类错误：
- `invalid_request` — 参数错误
- `request_not_idempotent` — 幂等键冲突
- `processing_error` — 服务端故障
- `service_unavailable` — 临时中断

结账响应中的 `messages[]` 数组处理内联错误（缺货、支付被拒、需要 3DS），让代理可以对话式地回应而不是崩溃。

## 产品 Feed — 商户如何列出产品

商户通过压缩产品 feed 提供目录：

| 方面 | 详情 |
|------|------|
| 格式 | 压缩 JSON Lines（`.jsonl.gz`）或 CSV（`.csv.gz`） |
| 刷新 | 每 15 分钟 |
| 必填字段 | GTIN/UPC/MPN、标题、价格、库存状态、卖家名称 |
| 政策 URL | 退货政策、隐私政策、服务条款（均必填） |
| 结账标记 | `enable_checkout=true`（需要 `enable_search=true`） |

**关键约束**：产品必须先可搜索（`enable_search=true`），才能可购买（`enable_checkout=true`）。确保 ChatGPT 只卖它能推荐的东西。

## 商户接入 — 四条路径

| 路径 | 适用对象 | 工作量 | 时间线 |
|------|---------|--------|--------|
| **Stripe 直接接入** | 现有 Stripe 商户（WooCommerce、BigCommerce、Wix、Squarespace） | "最少一行代码" | 几天 |
| **Shopify** | 100 万+ Shopify 商户 | 一次设置，同步到 ChatGPT + Perplexity + Copilot | 几周 |
| **PayPal ACP 服务器** | 数千万小企业 | PayPal 处理 ACP 集成 | 2026 年 |
| **直接申请** | 自定义平台 | 按规范构建 REST API，在 chatgpt.com/merchants 申请 | 几个月 |

**需要认证**：上线前，商户必须通过 OpenAI 的一致性检查（产品、支付、订单处理）。

## 经济模型

### 费用结构

| 费用 | 金额 |
|------|------|
| OpenAI 平台费 | 交易额的 4% |
| Stripe 处理费 | ~2.9% + $0.30 |
| **综合费率** | **~9.2%**（使用 Shopify Payments 时） |

**对比：**
| 平台 | 费率 |
|------|------|
| 亚马逊市场 | 25-30% |
| **ChatGPT (ACP)** | **~9.2%** |
| Google Shopping | 0%（广告支持） |
| Microsoft Copilot | 0%（引流模式） |

Shopify 商户的 OpenAI 平台费享有 **30 天免费试用**。

### 市场数据

| 指标 | 数值 |
|------|------|
| ChatGPT 周活 | 8-9 亿 |
| 每日购物查询 | 5000 万 |
| 使用 GenAI 购物的美国消费者 | 59% |
| GenAI 购物者中偏好 ChatGPT | 65% |
| 来自 ChatGPT 的沃尔玛引流流量 | ~36% |
| 预计 2030 年美国代理电商 | $1900-3850 亿 |

## 规范演进

规范使用基于日期的版本控制，快速迭代：

| 版本 | 日期 | 关键变更 |
|------|------|---------|
| 初始版 | 2025-09-29 | 核心结账端点、SPT、Etsy 上线 |
| v2 | 2025-12-12 | 履约增强 |
| v3 | 2026-01-16 | 能力协商 |
| v4 | 2026-01-30 | 扩展、折扣、处理器 |
| 未发布 | 开发中 | 下一次迭代 |

**仓库结构：**
```
agentic-commerce-protocol/
├── rfcs/                     # 设计理由
├── spec/
│   ├── 2025-09-29/          # 初始版
│   ├── 2025-12-12/          # 履约
│   ├── 2026-01-16/          # 能力
│   ├── 2026-01-30/          # 扩展
│   └── unreleased/          # 开发中
├── examples/                 # 请求/响应示例
├── changelog/
└── docs/                     # 治理、SEP 指南
```

关键文件：`openapi.agentic_checkout.yaml`（结账 API）、`openapi.delegate_payment.yaml`（支付委托）、所有数据模型的 JSON schema。

## ACP vs Google UCP vs AP2

Google 对 ACP 的回应是 **Universal Commerce Protocol（UCP，通用商务协议）** — 一个更广泛的标准，覆盖 Google 表面（Gemini、搜索 AI 模式）上的发现、结账和支付。

| 维度 | ACP（OpenAI/Stripe） | UCP（Google） | AP2（Google） |
|------|---------------------|-------------|-------------|
| 范围 | 仅结账 | 全商务（发现→结账→订单管理） | 仅支付授权 |
| 发现 | 产品 feed → ChatGPT 搜索 | `/.well-known/ucp` 清单 | 不适用 |
| 传输 | REST API 或 MCP | REST、A2A、MCP | 任务令（加密） |
| 支付 | SPT via Stripe | 令牌化工具，多处理器 | 三种任务令类型 |
| 复杂度 | 5 个端点 | 完整服务架构 | 6 个角色，复杂流程 |
| 角色 | 4 个（买家、代理、商户、PSP） | 4+（表面、后端、提供者） | 6 个角色 |
| 合作伙伴 | Stripe、Shopify、Etsy、PayPal | Shopify、Etsy、Wayfair、Target、Walmart、Stripe、Amex | Google Wallet 生态 |
| 状态 | 生产中（ChatGPT） | 生产中（Gemini、搜索） | V0.1（早期） |

**真实动态**：ACP 和 UCP 是趋同的竞争者。两者都开源，都支持 REST/MCP，都使用基于令牌的支付。区别在于组织 — ACP 以 Stripe 为先，UCP 以 Google 为先。

**实用建议**：同时实现 ACP 和 UCP 的商户比单一协议实现多获得 **40% 的代理流量**。它们并不互斥。

**AP2 的角色**：AP2 处理支付*授权*（任务令），而 ACP/UCP 处理*结账流程*。AP2 与 UCP 兼容，理论上也可以与 ACP 一起工作。

## 参考实现

**locus-technologies/agentic-commerce-protocol-demo** — MIT 许可的 demo，由 Scale AI + Coinbase 校友构建：

| 组件 | 端口 | 角色 |
|------|------|------|
| MCP-UI 客户端 | 3000 | 聊天界面（改编自 Scira Chat） |
| 商户 API | 4001 | 结账会话管理 |
| PSP 服务器 | 4000 | 委托支付令牌化 |

**技术栈**：TypeScript、Node.js 20+、PostgreSQL、Docker Compose。

**演示内容：**
1. 通过语义向量搜索发现产品
2. 完整结账生命周期（创建→更新→完成）
3. 委托支付流程（凭证保管→SPT 发行→兑换）
4. MCP 工具集成作为代理-商务桥梁

## 权衡与问题

### 真实问题

| 问题 | 严重性 | 详情 |
|------|--------|------|
| 仅限美国 | 高 | 尚无国际支持。货币、税务、运输法规差异巨大 |
| Stripe 锁定 | 高 | Stripe 是唯一支持 SPT 的 PSP。"开放标准"但单一实现 |
| 4% 平台费 | 中 | 高于 Google（0%），低于亚马逊（25-30%）。可能劝退低利润商户 |
| 产品 feed 负担 | 中 | 15 分钟刷新、需要 GTIN/UPC、压缩格式 — 对小商户负担重 |
| AI 推荐责任 | 中 | 代理推荐错误产品，买家退货→谁付钱？未指定 |
| 亚马逊缺席 | 中 | 亚马逊通过 robots.txt 封锁 ChatGPT 爬虫。最大电商玩家缺席 |
| 行为改变 | 中 | 消费者必须信任 AI 代为购买。采用曲线未知 |
| 欺诈面 | 中 | 代理发起的购买是新的攻击向量。SPT 缓解但不消除 |
| 规范变动 | 低 | 4 个月内四个版本。对实现者来说是移动目标 |

### ACP 故意不做的事

- **没有产品发现**：商户提供 feed；ChatGPT 处理搜索/推荐
- **没有库存管理**：商户的问题
- **没有争议解决**：商户处理退货/退款
- **没有多币种**：最初仅 USD
- **没有循环支付**：SPT 仅 `one_time`。不支持订阅
- **没有代理对代理商务**：ACP 是 买家→代理→商户，不是 代理→代理

### Stripe 依赖

ACP 技术上是"开放标准"，但实际上是 Stripe 产品：
- Stripe 共同开发规范
- Stripe 是唯一支持 SPT 的 PSP
- Stripe 处理所有支付
- Stripe 提供欺诈风险信号
- 即使使用"其他提供商"的商户仍依赖 Stripe 的风险评分

规范*允许*其他 PSP 实现委托支付规范，但没人做。这和 x402-Coinbase 如出一辙 — 开放标准，单一实现。

## 可复用的模式

| 模式 | 内容 | 重要性 |
|------|------|--------|
| 范围化支付令牌 | SPT = 一次性、有时限、商户+金额范围化凭证 | 解决代理-凭证问题，无需给代理支付权限 |
| 委托支付 | 三阶段凭证流（用户→PSP→令牌→商户） | 代理和商户都不接触原始凭证 |
| 网络令牌优于 FPAN | 优先使用令牌化卡号以减少 PCI 范围 | 以最小实现成本获得实际安全改进 |
| 产品 feed 作为发现 | 压缩 JSONL/CSV，15 分钟刷新 | 代理只能卖它能找到的东西 — 搜索门控商务 |
| 状态驱动结账 | 状态机 + messages[] 处理内联错误 | 代理对话式处理"缺货"而不是崩溃 |
| Webhook 订单生命周期 | order_created → confirmed → shipped → fulfilled 事件 | 代理主动更新用户订单状态 |
| 基于日期的规范版本 | YYYY-MM-DD 版本，所有版本保留 | 清晰演进，不破坏现有集成 |
| 双协议策略 | 同时实现 ACP 和 UCP 获得 40% 更多流量 | 协议无关的商户基础设施有回报 |
| 风险信号转发 | SPT 包含 Stripe 风险评分，即使非 Stripe 商户也能用 | 欺诈保护作为共享基础设施 |

## 最新动态 (2026)

**ChatGPT 即时结账正式上线（2026 年 2 月 16 日）。** OpenAI 面向所有美国 ChatGPT 用户（Plus、Pro、Free）推出"Buy it in ChatGPT"。美国 Etsy 卖家从第一天起即可使用。超过 100 万 Shopify 商户（包括 Glossier、SKIMS、Spanx、Vuori）已宣布即将接入。用户点击"Buy"，确认订单/配送/支付详情，无需离开对话即可完成购买。

**PayPal 成为第二个 PSP，打破 Stripe 垄断（2025 年 10 月宣布，2026 年推出）。** PayPal 与 OpenAI 宣布合作为 ChatGPT 即时结账提供支持。PayPal 的 ACP 服务器 — 基于 PayPal 的订单管理系统、结账和商务平台构建 — 将把数千万小企业和主要零售品牌的产品目录带入 ChatGPT。PayPal 将负责支付合规、商户路由、PCI 合规和速率限制。PayPal 用户可以使用 PayPal 余额即时结账。这是 ACP 生态系统中第一个真正替代 Stripe 的 PSP，支持 ACP、A2A 和 AP2 协议。

**Salesforce 通过 Agentforce Commerce 集成 ACP（2025 年 10 月）。** Salesforce 宣布与 Stripe 和 OpenAI 合作支持 ACP。来自 Agentforce Commerce 的产品目录将出现在 ChatGPT 中，通过 Stripe 令牌化实现嵌入式 ACP 结账。Salesforce 数据显示，2025 年上半年 AI 助手带来的线上流量同比增长 119%，预计 AI 代理将在 Cyber Week 驱动全球 22% 的订单。

**Visa 推出可信代理协议（TAP）（2025 年 10 月）。** Visa 推出了一个开放框架 — 与 Cloudflare 联合开发 — 使商户能够在结账时区分合法 AI 代理和恶意机器人。该协议应对美国零售网站 AI 驱动流量激增 4,700% 的挑战。超过 100 家合作伙伴正与 Visa 合作，30+ 家在 VIC 沙盒中构建，20+ 家代理服务商直接集成。Visa 预测到 2026 年假日季，数百万消费者将使用 AI 代理购物。亚太和欧洲的试点项目正在推进中。

**Mastercard Agent Pay 和代理令牌（2025 年 4 月 → 2026 年 1 月）。** Mastercard 推出 Agent Pay，引入 Mastercard 代理令牌 — 基于移动非接触支付和 Payment Passkeys 的成熟令牌化能力。Mastercard 正与 Microsoft 合作将 Agent Pay 引入 Copilot Checkout，并与 PayPal 合作将其集成到 PayPal 钱包中进行代理发起的交易。2026 年 1 月，Mastercard 推出 Agent Suite（2026 年 Q2 可用），结合可定制 AI 代理和咨询支持。Mastercard 正在跨协议合作：Google UCP、AP2、A2A 和 OpenAI ACP。

**Google UCP 进入生产（2026 年 1 月）。** Google 在 NRF 2026 大会上推出通用商务协议，与 Shopify、Etsy、Wayfair、Target 和 Walmart 联合开发，获得 20+ 家公司背书，包括 Stripe、Mastercard、Visa、American Express、Best Buy、Macy's、Home Depot 和 Zalando。UCP 驱动的结账已在 AI Mode（Google 搜索）和 Gemini 应用中面向美国消费者上线，可从 Etsy 和 Wayfair 购买。Shopify、Target 和 Walmart 的集成计划随后跟进。自 NRF 发布以来，Google 收到数百家顶级科技公司、支付合作伙伴和零售商的兴趣。

**多协议现实。** 代理商务格局已经形成并行生态系统：ACP（OpenAI/Stripe）在 ChatGPT 中，UCP（Google）在搜索 AI 模式和 Gemini 中，Mastercard Agent Pay 在 Copilot 中，Visa TAP 作为跨平台认证层。Etsy 和 Shopify 已在 ACP 和 UCP 上上线或宣布接入。PayPal 通过其 ACP 服务器桥接协议。原始分析中的实用建议 — 同时实现 ACP 和 UCP — 现已被生态系统趋同所验证。

## 总结

**已经落地的**：ACP 在生产环境运行。真实用户通过 ChatGPT 即时结账购买真实产品。SPT 机制很巧妙 — 它真正解决了"代理需要付款但不应该持有凭证"的问题。规范简洁、精简，迭代迅速。

**尚未就绪的**：国际支持、多 PSP 生态、循环支付、代理对代理商务。"开放标准"的说法是愿景性的 — 今天是在 Stripe 轨道上运行的 Stripe/OpenAI 双头垄断。

**诚实评估**：ACP 是大规模代理商务的第一个可工作实现。8 亿+ ChatGPT 用户基础给了它任何竞争协议无法匹敌的分发能力。但 4% 费用 + Stripe 依赖制造了与 x402+Coinbase 相同的中心化张力。真正的问题是 UCP（Google）是否能迫使真正的多供应商竞争，还是我们最终得到两个专有的"开放标准"。

**今天该做什么**：如果你在线销售实体产品，申请 ChatGPT 即时结账。4% 费用的 30 天免费试用意味着测试零风险。如果你在构建代理基础设施，实现 ACP 的结账端点 — 它们是简单的 REST，规范足够稳定可以在上面构建。但也要密切关注 UCP，考虑同时实现两者。

## References

### Official Specification & Source Code

- [ACP Specification — Stripe Docs](https://docs.stripe.com/agentic-commerce/protocol/specification) — Full OpenAPI spec: 5 endpoints, data schemas, error handling
- [ACP Getting Started — OpenAI](https://developers.openai.com/commerce/guides/get-started/) — Merchant onboarding guide: product feeds, checkout endpoints, certification
- [Delegated Payment Spec — OpenAI](https://developers.openai.com/commerce/specs/payment/) — SPT lifecycle, credential delegation flow, PSP integration
- [ACP GitHub Repository](https://github.com/agentic-commerce-protocol/agentic-commerce-protocol) — Open-source spec (Apache 2.0), RFCs, examples, all spec versions
- [ACP Community Site — agenticcommerce.dev](https://www.agenticcommerce.dev/) — Protocol overview, docs, contribution info
- [Integrate ACP — Stripe Docs](https://docs.stripe.com/agentic-commerce/protocol) — Stripe integration guide: SPT handling, webhook setup

### Announcements & Press

- [Stripe Powers Instant Checkout in ChatGPT — Stripe Newsroom](https://stripe.com/newsroom/news/stripe-openai-instant-checkout) — Launch announcement, merchant list, SPT introduction
- [OpenAI Expands Agentic Commerce Push — Digital Commerce 360](https://www.digitalcommerce360.com/2026/02/16/openai-expands-agentic-commerce-push/) — Feb 2026 expansion coverage
- [Developing ACP — Stripe Blog](https://stripe.com/blog/developing-an-open-standard-for-agentic-commerce) — Stripe's design rationale
- [Buy it in ChatGPT — OpenAI](https://openai.com/index/buy-it-in-chatgpt/) — Official launch announcement

### Technical Deep Dives

- [ChatGPT Instant Checkout Retailer Guide — Ekamoira](https://www.ekamoira.com/blog/chatgpt-instant-checkout-agentic-commerce-protocol-2026) — Fee structure (4% + 2.9%), onboarding paths, market data, early adopters
- [ACP Deep Dive — Fintech Wrap Up](https://www.fintechwrapup.com/p/deep-dive-inside-stripe-and-openais) — Business model analysis, merchant economics, competitive positioning
- [ACP for Product Teams — Department of Product](https://departmentofproduct.substack.com/p/what-is-acp-agentic-commerce-protocol) — Product manager perspective
- [ACP Strategic Blueprint — WeArePresta](https://wearepresta.com/agentic-commerce-protocol-acp-the-definitive-2026-guide-for-ai-driven-retail/) — Strategic analysis for retailers
- [Step-by-Step ACP Implementation — Medium](https://medium.com/@maheshlambe/step-by-step-guide-for-implementing-the-agentic-commerce-protocol-acp-aed4f9b1a457) — Developer walkthrough

### Reference Implementation

- [ACP Demo — locus-technologies/agentic-commerce-protocol-demo](https://github.com/locus-technologies/agentic-commerce-protocol-demo) — TypeScript/Node.js demo: MCP-UI client + Merchant API + PSP server, Docker Compose

### Google UCP (Competing Protocol)

- [Universal Commerce Protocol (UCP) — Google Developers Blog](https://developers.googleblog.com/under-the-hood-universal-commerce-protocol-ucp/) — Architecture, /.well-known/ucp discovery, capability manifest, payment handlers
- [How to Implement UCP — WeArePresta](https://wearepresta.com/how-to-implement-universal-commerce-protocol-ucp-in-2026-complete-well-known-ucp-setup-guide/) — UCP setup guide
- [Google UCP Developer Guide](https://developers.google.com/merchant/ucp) — Official UCP documentation
- [Building the Universal Commerce Protocol — Shopify Engineering](https://shopify.engineering/ucp) — Shopify's UCP implementation perspective
- [UCP Launch at NRF 2026 — Google Blog](https://blog.google/products/ads-commerce/agentic-commerce-ai-tools-protocol-retailers-platforms/) — Google's retail tools and UCP announcement

### Additional Technical Sources

- [Shared Payment Tokens — Stripe Docs](https://docs.stripe.com/agentic-commerce/concepts/shared-payment-tokens) — SPT lifecycle, webhook events, credential isolation
- [Agentic Checkout Spec — OpenAI](https://developers.openai.com/commerce/specs/checkout/) — Full checkout endpoint spec
- [Agentic Commerce Suite — Stripe Blog](https://stripe.com/blog/agentic-commerce-suite) — Stripe's hosted ACP solution for merchants
- [Agentic Commerce Solutions — Stripe Blog](https://stripe.com/blog/introducing-our-agentic-commerce-solutions) — Enterprise integration patterns
- [ChatGPT Product Feed Guide — Lengow](https://www.lengow.com/get-to-know-more/chatgpt-product-feed/) — Product feed format deep dive
- [ACP vs UCP — Checkout.com](https://www.checkout.com/blog/openai-acp-google-ucp-difference) — Head-to-head protocol comparison

### Protocol Comparisons

- [ACP, AP2, x402 Comparison — Orium](https://orium.com/blog/agentic-payments-acp-ap2-x402) — Payment protocol comparison
- [AP2 vs ACP — Grid Dynamics](https://www.griddynamics.com/blog/agentic-payments) — Enterprise analysis
- [MCP, A2A, AP2, ACP, x402 & ERC-8004 — PayRam](https://payram.com/blog/mcp-a2a-ap2-acp-x402-erc-8004) — Full 6-protocol stack comparison
- [Top 6 Agent-Native Rails — MarkTechPost](https://www.marktechpost.com/2025/11/14/comparing-the-top-6-agent-native-rails-for-the-agentic-internet-mcp-a2a-ap2-acp-x402-and-kite/) — Protocol landscape

### PSP & Card Network Initiatives

- [PayPal + OpenAI Partnership — PayPal Newsroom](https://newsroom.paypal-corp.com/2025-10-28-OpenAI-and-PayPal-Team-Up-to-Power-Instant-Checkout-and-Agentic-Commerce-in-ChatGPT) — PayPal ACP Server announcement
- [PayPal Agentic Commerce — paypal.ai](https://www.paypal.ai/agenticcommerce) — PayPal's ACP Server platform
- [Making Sense of the AI Shopping Protocol Moment — PayPal](https://newsroom.paypal-corp.com/2026-01-22-Making-Sense-of-the-AI-Shopping-Protocol-Moment) — PayPal's protocol landscape analysis
- [Visa Trusted Agent Protocol — Visa](https://usa.visa.com/about-visa/newsroom/press-releases.releaseId.21716.html) — TAP framework for AI commerce
- [Visa Completes Secure AI Transactions — Visa](https://usa.visa.com/about-visa/newsroom/press-releases.releaseId.21961.html) — Visa partner progress and 2026 predictions
- [Mastercard Agent Pay — Mastercard](https://www.mastercard.com/us/en/business/artificial-intelligence/mastercard-agent-pay.html) — Agentic Tokens and Agent Pay platform
- [Mastercard Agent Suite Launch — Mastercard](https://www.mastercard.com/global/en/news-and-trends/press/2026/january/mastercard-launches-agent-suite-to-ready-enterprises-for-a-new-e.html) — Q2 2026 enterprise agent tools
- [Mastercard + PayPal Agentic Commerce — PayPal](https://newsroom.paypal-corp.com/2025-10-27-Mastercard-and-PayPal-Join-Forces-To-Accelerate-Secure-Global-Agentic-Commerce) — Joint agentic commerce acceleration
- [Salesforce ACP Support — Salesforce](https://www.salesforce.com/news/press-releases/2025/10/14/stripe-openai-agentic-commerce-protocol-announcement/?bc=OTH) — Agentforce Commerce + ACP integration

### Market Analysis

- [Agentic Commerce Complete Guide 2026 — Ekamoira](https://www.ekamoira.com/blog/what-is-agentic-commerce-the-complete-2026-guide-to-ai-shopping-agents) — 45% consumer AI shopping adoption
- [AI Shopping Assistant Guide 2026 — Opascope](https://opascope.com/insights/ai-shopping-assistant-guide-2026-agentic-commerce-protocols/) — Protocol comparison and market landscape
- [2025: The Year AI Agents Entered Payments — PYMNTS](https://www.pymnts.com/news/artificial-intelligence/2025/2025-the-year-ai-agents-entered-payments-and-changed-whos-in-control) — Industry transformation analysis
- [AI Agent Payments Landscape 2026 — Proxy](https://www.useproxy.ai/blog/ai-agent-payments-landscape-2026) — Protocols, players, and primitives overview
