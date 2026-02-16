# GEO：生成式引擎优化

如何让你的内容被 AI 搜索引擎（ChatGPT、Perplexity、Claude、Google AI Overviews）发现和引用。

## 核心要点

GEO 是 AI 搜索时代的新 SEO。传统 SEO 让你在 Google 排名；GEO 让 AI 在回答问题时**引用你**。Google 排名和 AI 引用的重叠度已从 70% 降至 20% 以下——AI 有自己的偏好。

**关键洞察**：AI 搜索不再是关键词游戏。它关乎**成为 AI 想引用的权威答案**。

## 什么是 GEO？

| 维度 | SEO | GEO |
|------|-----|-----|
| 目标 | SERP 排名，获取点击 | 被 AI 答案引用 |
| 指标 | 排名位置、CTR | 引用频率、品牌提及 |
| 信号 | 外链、关键词、meta 标签 | 权威性、结构化、直接答案 |
| 内容风格 | 关键词优化 | 对话式、答案优先 |
| 索引速度 | 数天到数周 | 数小时（Perplexity 实时） |

## AI 搜索格局

### 主要平台

| 平台 | 爬虫 | 行为特点 |
|------|------|---------|
| **ChatGPT Search** | OAI-SearchBot, GPTBot | 使用 Bing 索引 + 自有缓存索引。可实时搜索或使用缓存 |
| **Perplexity** | PerplexityBot | 实时爬取，较小的精选索引。每秒处理 1 万+ 索引更新 |
| **Claude** | ClaudeBot | 通过 Brave 等合作伙伴提供网络搜索 |
| **Google AI Overviews** | Googlebot | 从现有 Google 索引提取，严重偏好前 10 名网页 |

### 引用模式差异

每个平台有明显的偏好：
- **ChatGPT**：严重依赖 Wikipedia，偏好老牌权威来源
- **Perplexity**：Reddit 浓度高，偏好新鲜内容和论坛
- **Google AI Overviews**：来源分布较广，52% 来自自然排名前 10

## 研究发现：什么真正有效

普林斯顿/IIT 研究者的 [GEO 论文 (arXiv 2311.09735)](https://arxiv.org/abs/2311.09735) 在真实 AI 系统上测试了优化策略。

### 量化结果

| 策略 | 可见度提升 |
|------|-----------|
| **添加引用来源** | 低排名网站 +115% |
| **添加统计数据** | +22% 位置调整词数 |
| **添加引语** | +37% 主观印象得分 |
| **综合 GEO 方法** | 最高 +40% 可见度 |

**关键发现**：排名较低的网站从 GEO 获益更多。添加引用使 SERP 第 5 位的网站获得 115% 的可见度提升——GEO 是均衡器。

## 技术要求

### 1. 允许 AI 爬虫

你的 `robots.txt` 必须允许 AI 机器人：

```txt
# 允许 AI 搜索爬虫
User-agent: GPTBot
Allow: /

User-agent: OAI-SearchBot
Allow: /

User-agent: ChatGPT-User
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: Claude-Web
Allow: /

User-agent: Google-Extended
Allow: /
```

**警告**：有些网站在 CDN/WAF 层意外阻止了 AI 爬虫。检查你的服务器是否拒绝 AI 机器人请求。

### 2. 服务端渲染

AI 爬虫无法执行 JavaScript。如果你的内容是客户端渲染的：
- 使用 SSR/SSG（Next.js、Nuxt、Astro）
- 预渲染关键内容
- 用 `curl -A "GPTBot"` 测试爬虫实际看到什么

### 3. 无付费墙/登录

需要认证的内容对 AI 不可见。想要被引用，内容必须公开访问。

## 内容优化策略

### "答案胶囊"技术

先给答案。AI 系统提取它找到的第一个完整答案。

**反面教材**：
```
在本文中，我们将探讨 React hooks 的历史、
它们的演变，以及为什么重要。首先，让我们了解
背景。早在 2018 年...[3 段之后] useEffect
在渲染后执行。
```

**正确做法**：
```
useEffect 默认在每次渲染后执行。要只在挂载时执行一次，
传入空依赖数组：useEffect(() => {}, [])。

以下是原理和常见陷阱...
```

### AI 友好的内容结构

AI 系统偏好：
- **列表和要点**：被引用可能性提高 3 倍
- **表格**：易于解析，引用率高
- **FAQ 格式**：问答对与用户查询直接对应
- **操作步骤**：编号序列会被整体提取

### 值得引用的内容模式

什么让 AI 想引用你：

| 模式 | 为什么有效 |
|------|-----------|
| **具体统计数据** | "40% 提升" 优于 "显著提升" |
| **带归属的专家引语** | 具名权威表明可信度 |
| **对比表格** | AI 喜欢结构化对比 |
| **原创研究/数据** | 独特数据 = 独特来源 |
| **时效性标识** | "截至 2026 年 2 月" 表明新鲜度 |

### AI 时代的 E-E-A-T

Google 的 E-E-A-T（经验、专业、权威、可信）对 AI 更加重要：
- 带资质的**作者署名**
- 说明专业背景的**关于页面**
- 引用**权威外部来源**
- **一贯准确**（AI 系统会学会信任/不信任特定来源）

## Schema 结构化标记

结构化数据告诉 AI 你的内容确切含义。

### 优先 Schema 类型

```json
// Article Schema - 用于博文、新闻
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "如何针对 AI 搜索优化",
  "author": {
    "@type": "Person",
    "name": "专家姓名",
    "url": "https://example.com/about"
  },
  "datePublished": "2026-02-10",
  "dateModified": "2026-02-13"
}
```

```json
// FAQ Schema - 对 GEO 非常有效
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [{
    "@type": "Question",
    "name": "什么是 GEO？",
    "acceptedAnswer": {
      "@type": "Answer",
      "text": "GEO（生成式引擎优化）是..."
    }
  }]
}
```

```json
// Organization Schema - 建立权威
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "公司名称",
  "url": "https://example.com",
  "sameAs": ["linkedin", "twitter 链接"]
}
```

### 按内容类型选择 Schema

| 内容类型 | Schema 类型 |
|----------|-------------|
| 博客/文章 | Article, Person (作者), BreadcrumbList |
| 产品页面 | Product, Review, AggregateRating |
| 操作指南 | HowTo, FAQPage |
| 公司页面 | Organization, LocalBusiness |

## 平台特定策略

### ChatGPT Search

- 以 Bing 索引为基础——传统 SEO 仍然重要
- 严重依赖缓存内容
- Wikipedia 风格的权威语调效果好
- 层次清晰的结构化内容

**技术细节**：确保 `OAI-SearchBot` 能访问你的网站。OpenAI 发布了 IP 地址供白名单使用。

### Perplexity

- **实时索引**：新内容可以在数小时内出现，而非数周
- 偏好：综合指南、原创研究、对比文章、专家意见
- Reddit 浓度高：监控并参与相关 subreddit
- 使用较小的精选索引——新鲜度和清晰度决定胜负

**机会**：Perplexity 每次回答引用的来源比 ChatGPT 多，增加了被收录的机会。

### Google AI Overviews

- 52% 的引用来自自然排名前 10
- 传统 SEO 是前提条件
- 移动端优化至关重要
- 专注长尾、问题型查询
- 需要时用 `nosnippet` 控制摘要展示

## GEO 监控与工具

### 关键指标

| 指标 | 描述 |
|------|------|
| **引用频率** | 你的域名在 AI 回答中出现的频率 |
| **AI 声量份额** | 你相对竞争对手的可见度 |
| **情感倾向** | AI 如何描述你的品牌 |
| **来源位置** | 在回答中被引用的位置 |

### 监控工具

| 工具 | 重点 |
|------|------|
| **Otterly.ai** | 跨平台监控（6 个 AI 平台）|
| **Geoptie** | GEO 审计 + 内容优化 |
| **Relixir** | 引用追踪 + 收入归因 |
| **AmICited** | 简单引用检查器 |

### DIY 监控

```bash
# 简单监控：跟踪你的域名在各 AI 查询中的表现
# 每周运行与业务相关的查询
# 记录哪些平台引用你，在什么语境下
```

## GEO 检查清单

### 技术基础
- [ ] `robots.txt` 允许 AI 爬虫（GPTBot、OAI-SearchBot、PerplexityBot、ClaudeBot）
- [ ] 服务端渲染内容（非纯 JS）
- [ ] 想被索引的页面无付费墙/登录
- [ ] 快速加载（<3 秒）
- [ ] 移动端响应式设计

### Schema 标记
- [ ] 内容页面有 Article/BlogPosting schema
- [ ] 适用时有 FAQ schema
- [ ] 关于/首页有 Organization schema
- [ ] 作者 Person schema 带资质

### 内容结构
- [ ] 答案优先格式（第一段就给答案）
- [ ] 清晰的 H2/H3 层级
- [ ] 使用要点和编号列表
- [ ] 对比使用表格
- [ ] 包含 FAQ 部分

### 权威信号
- [ ] 带来源的统计数据
- [ ] 带归属的专家引语
- [ ] 带资质的清晰作者署名
- [ ] 链接到权威外部来源
- [ ] 可见的发布/更新日期

### 持续工作
- [ ] 每周监控 AI 引用情况
- [ ] 更新内容保持新鲜
- [ ] 跟踪竞争对手 GEO 可见度
- [ ] 参与 AI 系统偏好的平台（Reddit、论坛）

## 权衡与局限

### GEO 不能做什么
- **无法修复烂内容**：GEO 优化呈现，不优化实质
- **无法保证引用**：AI 系统是黑盒
- **平台变化**：AI 搜索快速演进
- **训练 vs 搜索爬虫**：有些机器人训练模型，有些用于搜索——要区分

### GPTBot 困境
- `GPTBot`：用于**训练** OpenAI 模型
- `OAI-SearchBot`：用于 **ChatGPT 搜索**
- 阻止 GPTBot 不会阻止搜索，但它们经常被捆绑处理

很多出版商阻止训练机器人但允许搜索机器人。由你决定。

### 时间投入
- GEO 在 SEO 基础上增加了复杂度
- 需要监控多个平台
- 内容更新需要更频繁的刷新周期

## 可直接借鉴的做法

1. **答案优先写作**：每个页面应在前 100 字回答核心问题
2. **结构化内容**：表格、列表、FAQ——AI 系统喜欢可解析的内容
3. **带来源的统计**："40% 提升（哈佛研究，2025）" > "显著提升"
4. **Schema 标记**：仅 FAQ schema 就能大幅提高引用率
5. **robots.txt 审计**：大多数网站意外阻止了 AI 爬虫
6. **新鲜度信号**：可见的发布和更新日期
7. **Perplexity 优势**：它索引快——用于时效性内容

## 未来方向

Gartner 预测到 2026 年，AI 辅助搜索将影响 50%+ 的消费者查询。趋势是：
- 从**排名**到**成为答案**
- 从**流量**到**品牌提及**
- 从**关键词**到**实体识别**

GEO 不是替代 SEO，你两者都需要。但忽视 GEO 意味着对越来越多的搜索者变得隐形。
