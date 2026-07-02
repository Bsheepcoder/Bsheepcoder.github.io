## llmstxt.site 是什么

[llmstxt.site](https://llmstxt.site/) 是一个第三方目录站，专门收录全球部署了 `/llms.txt` 文件的网站。llms.txt 是 Jeremy Howard（Answer.AI 联合创始人、fast.ai 创始人）于 2024 年 9 月提出的标准：在网站根路径放一个 Markdown 格式的 `/llms.txt`，为 LLM 提供干净的内容入口，让 AI 在推理时零噪音获取网站内容。

目前有两个独立的目录站收录 llms.txt 采用者：

| 目录 | 收录量 | 特点 |
|------|--------|------|
| [directory.llmstxt.cloud](https://directory.llmstxt.cloud/) | 849 站 | 高质量策展，有审核团队，收录 Anthropic、Cursor、Cloudflare 等一线品牌 |
| [llmstxt.site](https://llmstxt.site/) | 1600+ 站 | 开放收录，门槛低，量大但 SEO/营销站占比高 |

llmstxt.site 的特点在于**收录量大、覆盖广**。它的页面是一个长表格，每行一个站点，列出名称、主站 URL、`llms.txt` URL、token 数，以及可选的 `llms-full.txt` URL 和 token 数。token 数由爬虫自动统计，能直观反映每个站点给 AI 提供了多少内容。

不过门槛低也意味着泥沙俱下——大量收录是酒店、本地商家、SEO 营销页这类对 AI 生态毫无价值的站点。真正有意思的站点需要从 1600+ 条记录里淘出来。

## 有趣网站精选

我翻完了 llmstxt.site 的全部记录，过滤掉噪音，选出真正值得关注的站点。以下是按类别整理的结果。

## AI 平台与 LLM 工具

这个类别是 llms.txt 最自然的受众——AI 公司需要让其他 AI 能读懂自己的平台。

| 站点 | llms.txt tokens | 说明 |
|------|:-:|------|
| [Anthropic](https://claude.com/llms.txt) | 8.4K | Claude 母公司，AI 安全实验室（full 481K） |
| [Fireworks AI](https://fireworks.ai/llms.txt) | 4.4K | Serverless LLM 推理平台（full 88K） |
| [Langbase](https://langbase.com/llms.txt) | 367K | 可组合 AI pipes，LLM 开发平台 |
| [AgentDock](https://agentdock.ai/llms.txt) | 293K | AI Agent 构建框架 |
| [ZenML](https://www.zenml.io/llms.txt) | 98K | 开源 ML pipeline 框架（full 575K） |
| [Maxim AI](https://getmaxim.ai/llms.txt) | 46K | LLM 评估与可观测性（full 410K） |
| [deepset](https://deepset.ai/llms.txt) | 1.6K | Haystack/RAG 框架厂商 |
| [Keywords AI](https://www.keywordsai.co/llms.txt) | 316 | LLM 日志/可观测性（极简） |
| [Tenthe AI Dictionary](https://tenthe.com/llms.txt) | 47K | AI 术语词典（full 3.6M tokens） |

几个观察：

- **Anthropic 的 llms.txt 约 8K tokens**，作为 Claude 的母公司，内容适中——既提供导航，也附带一定量的核心文档
- **Langbase 367K tokens** 是 AI 平台中最重的，说明它把大量 API 文档和示例都塞进了 llms.txt
- **Keywords AI 仅 316 tokens**，证明即使是很小的公司也可以快速接入

## 开发者工具与 SaaS

开发者工具是 llms.txt 采纳最积极的传统行业——文档站天然适合 Markdown 化。

| 站点 | llms.txt tokens | 说明 |
|------|:-:|------|
| [Sourcegraph](https://sourcegraph.com/docs) | 1.2M | 代码搜索平台（语料最大之一） |
| [Cloudflare Docs](https://developers.cloudflare.com/llms.txt) | 34K | 边缘平台文档（full 3.8M） |
| [Retool](https://docs.retool.com/llms.txt) | 32K | 内部应用构建器 |
| [Better Auth](https://better-auth.com/llms.txt) | 174K | 开源认证框架 |
| [CircleCI](https://circleci.com/llms.txt) | 1.2K | CI/CD 平台 |
| [Apify](https://apify.com/llms.txt) | 2.5K | 爬虫/自动化平台 |
| [Axiom](https://axiom.co/llms.txt) | 10K | 日志/可观测性 |
| [Activepieces](https://activepieces.com/llms.txt) | 4.6K | 开源 Zapier 替代品 |
| [Terminal Trove](https://terminaltrove.com/llms.txt) | 360 | CLI/TUI 工具目录（极简） |
| [Unkey](https://unkey.com/llms.txt) | 4K | API key 管理服务 |
| [liblab](https://liblab.com/llms.txt) | 9.3K | 从 API 自动生成 SDK |
| [DeployHQ](https://www.deployhq.com/llms.txt) | 3.3K | 自动化代码部署 |

**Sourcegraph 以 1.2M tokens 位居全目录语料量前列**。作为一个代码搜索平台，它把几乎所有文档都开放给了 AI——这本身就是对"AI 时代文档该怎么写"的一种表态。

**Terminal Trove 只有 360 tokens**，但它是一个 CLI 工具目录站，用极简的 llms.txt 就能让 LLM 知道"有哪些好用的命令行工具"。小而美的典范。

## 开源框架与文档

前端框架和开源项目是 llms.txt 最早一批采纳者。

| 站点 | llms.txt tokens | 说明 |
|------|:-:|------|
| [Next.js](https://nextjs.org/docs/llms.txt) | 14K | React 框架文档 |
| [Svelte](https://svelte.dev/llms.txt) | 281 | Svelte 框架（极简典范） |
| [Angular](https://angular.dev/llms.txt) | 1.5K | Angular 框架文档 |
| [Astro](https://astro.build/llms.txt) | 556 | 内容导向 Web 框架 |
| [Strapi](https://strapi.io/llms.txt) | 3.8K | Headless CMS |
| [Hugging Face Transformers](https://huggingface.co/) | 809K | Transformers 库文档（语料最大） |
| [Hugging Face Diffusers](https://huggingface.co/) | 383K | Diffusers 库文档 |
| [Meilisearch](https://www.meilisearch.com/llms.txt) | 331 | 开源搜索引擎 |
| [Apache Camel](https://camel.apache.org/llms.txt) | 1.1K | 集成框架 |
| [Stripe](https://docs.stripe.com/llms.txt) | 17K | 支付 API 文档 |
| [NVIDIA Developer](https://developer.nvidia.com/llms.txt) | 5.5K | NVIDIA 开发平台 |

**Svelte 的 281 tokens 是全目录最极简的前端框架实现**。对比 Next.js 的 14K tokens，Svelte 选择只放最核心的导航链接。两种策略各有道理——Next.js 文档量大需要详细索引，Svelte 文档结构简单不需要。

**Hugging Face 三件套（Transformers 809K + Diffusers 383K + Hub 72K）加起来超过 1.2M tokens**，是目前开源生态中对 llms.txt 投入最重的。作为模型托管平台，让 AI 能直接读取模型文档是刚需。

## 金融与加密

金融科技类站点对 llms.txt 的采纳出乎意料地积极。

| 站点 | llms.txt tokens | 说明 |
|------|:-:|------|
| [Bitcoin.com](https://www.bitcoin.com/llms.txt) | 722K | 加密新闻/钱包/交易所 |
| [Chainspect](https://chainspect.app/llms.txt) | 938K | 区块链分析平台 |
| [Mangopay](https://mangopay.com/llms.txt) | 11K | 嵌入式支付（full 1.7M） |
| [FinFeedAPI](https://finfeedapi.com/llms.txt) | 8K | 金融市场数据 API |
| [KuCoin API](https://www.kucoin.com/llms.txt) | 15K | 加密交易所 API |
| [Method Financial](https://methodfi.com/llms.txt) | 3.6K | 嵌入式金融 API |
| [Paysafe](https://developer.paysafe.com/llms.txt) | 6.2K | 支付网关 API |

**Bitcoin.com 和 Chainspect 的语料量（722K 和 938K）甚至超过很多技术文档站**。加密行业对"让 AI 读懂自己"有异常强的动力——可能是因为加密项目的技术叙事复杂，需要 AI 能准确理解其机制而非依赖媒体二手报道。

## MCP 生态

MCP（Model Context Protocol）相关的目录站已经出现在 llms.txt 收录中，说明两个标准正在交汇。

| 站点 | llms.txt tokens | 说明 |
|------|:-:|------|
| [MCP Server Space](https://mcpserver.space/llms.txt) | 1.2K | MCP 服务器目录 |
| [uminai MCP Directory](https://mcp.umin.ai/llms.txt) | 1K | MCP 服务器目录 |

MCP 是 Anthropic 提出的 AI 应用上下文接口标准，和 llms.txt 解决的是不同层面的问题——llms.txt 解决"AI 怎么读网站内容"，MCP 解决"AI 怎么调用工具和数据源"。两者的交汇点在于：一个 llms.txt 文件可以声明本站提供 MCP 适配器，让 AI 知道这里不仅可读，还可调用。

## 个人与小众站点

个人技术博客在 llms.txt 收录中凤毛麟角——大多数实现要么是大公司的深度文档站，要么是蹭热度的营销页。以下是少数有真正价值的个人/小众站：

| 站点 | llms.txt tokens | 说明 |
|------|:-:|------|
| [Huberman Lab](https://www.hubermanlab.com/llms.txt) | 14K | 神经科学播客（名人站） |
| [Readwise](https://readwise.io/llms.txt) | 96K | 稍后读/笔记应用 |
| [Listen Notes](https://www.listennotes.com/llms.txt) | 1K | 播客搜索引擎 |
| [Light Pollution Map](https://lightpollutionmap.app/llms.txt) | 544 | 光污染地图（新颖用例） |
| [Aurora Map](https://auroramap.app/llms.txt) | 463 | 极光预报地图 |
| [Rasul Kireev](https://www.rasulkireev.com/llms.txt) | 127K | 知名开发者个人站 |
| [MASI Longevity](https://masi.eu/llms.txt) | 1.1K | 长寿科学研究 |

**Light Pollution Map 和 Aurora Map 是两个非常规用例**——它们不是技术文档站，而是数据可视化工具。通过 llms.txt，它们让 AI 知道"这里有一个光污染/极光数据源"，拓宽了 llms.txt 的应用边界。

**Huberman Lab 是唯一的名人个人站**——Andrew Huberman 是斯坦福神经科学家，他的播客有大量科学内容。用 llms.txt 让 AI 能准确引用他的观点，而非依赖二手转述，是对抗信息失真的好方法。

## 三个观察

### 1. Token 数量两极分化

全目录的 token 分布呈双峰：

- **巨无霸**：Sourcegraph 1.2M、HF Transformers 809K、Bitcoin.com 722K——这些是文档站，llms.txt 只是冰山一角
- **极简派**：Svelte 281、Terminal Trove 360、Keywords AI 316——证明 llms.txt 不需要大才有价值

**llms.txt 的价值不在于自身包含多少内容，而在于它是一个精确的导航入口。** Svelte 只用 281 tokens 就让 LLM 知道去哪里找框架文档，这比塞 10 万 tokens 的全文更高效。

### 2. 中间地带缺失

收录站点要么是 Anthropic / Cloudflare / Next.js 这类一线技术品牌深度实现，要么是填表蹭热度的营销站。**独立开发者博客、中小技术站点是缺失的中间层。** 这不是好事——llms.txt 的价值需要更多真实内容生产者参与才能体现。

### 3. MCP 与 llms.txt 正在交汇

MCP 目录站已出现在 llms.txt 收录中。两个标准解决不同层面的问题（llms.txt = AI 读内容，MCP = AI 调工具），但正在产生交集——一个站点可以同时提供 llms.txt（可读）和 MCP 适配器（可调用），形成完整的 AI 可交互内容栈。

本站的 PDC 协议正处在这个交汇点上：基于 llms.txt 标准提供内容通道，同时通过 MCP 适配层让 Claude Desktop、Cursor 等客户端可直接访问。如果你也在做类似的事，欢迎交流。
