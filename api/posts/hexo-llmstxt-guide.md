## 两个收录目录

llms.txt 是 Jeremy Howard（Answer.AI）提出的网站约定，在根路径放一个 `/llms.txt` 文件，为 LLM 提供 Markdown 格式的内容入口。目前有两个第三方目录站收录这些文件，都独立于标准官方运营：

- **directory.llmstxt.cloud**：由 `@ifox` 和 `@joyceverheije` 创建，有审核团队，收录标准偏向有影响力的公司/产品，质量较高，收录了 Anthropic、Cursor、Cloudflare、Next.js 等一线品牌，分类统计 849 个 Websites、447 个 Products、358 个 Developer tools、187 个 AI、167 个 Finance。
- **llmstxt.site**：独立运营，收录门槛低、量大（1600+ 站点），但 SEO/营销型站点占比高。

本文从 llmstxt.site 的 1600+ 站点中筛选出有趣的，按类别介绍。

## AI 平台与 LLM 工具

### Anthropic

- 站点：[anthropic.com](https://anthropic.com/)
- llms.txt：8K tokens / llms-full.txt：481K tokens

Claude 的母公司，AI 安全实验室。作为 llms.txt 标准最早的一批采纳者，Anthropic 的 llms.txt 提供了公司理念、研究方向和 Claude 模型信息的结构化入口。有趣的是，它和 Cursor、Cloudflare 一起构成了"AI 公司自己用 llms.txt 服务 AI"的闭环——AI 公司最懂 AI 需要什么。

### Fireworks AI

- 站点：[fireworks.ai](https://fireworks.ai/)
- llms.txt：4K tokens / llms-full.txt：88K tokens

Serverless LLM 推理平台，主打"以 1/10 成本跑开源模型"。它的 llms.txt 是标准的 API 文档入口风格，帮助 LLM 快速定位模型列表、定价、推理端点。

### Langbase

- 站点：[langbase.com](https://langbase.com)
- llms.txt：367K tokens

可组合 AI pipes / LLM 开发平台。367K tokens 的 llms.txt 在 AI 类里算巨无霸，说明它把大量文档内容直接塞进了入口文件——这对一次性读取全站的 AI Agent 很友好，但也意味着每次请求都要消耗大量 token。

### AgentDock

- 站点：[agentdock.ai](https://agentdock.ai)
- llms.txt：293K tokens

AI Agent 构建框架。293K tokens 的体量同样偏大，推测是把完整的框架文档、API 参考和示例都合并进了 llms.txt。

### ZenML

- 站点：[zenml.io](https://www.zenml.io/)
- llms.txt：98K tokens / llms-full.txt：575K tokens

开源 ML pipeline 框架，定位类似"ML 工程界的 Terraform"。它同时提供了 llms.txt（精简入口）和 llms-full.txt（完整文档合并），是标准规范的最佳实践样本——让 AI 先读精简版定位，再按需读完整版。

### Maxim AI

- 站点：[getmaxim.ai](https://getmaxim.ai)
- llms.txt：46K tokens / llms-full.txt：410K tokens

LLM 评估与可观测性平台。作为"给 AI 做监控"的公司，用 llms.txt 给 AI 提供自己的文档，有"监控 AI 的工具被 AI 监控"的递归意味。

### deepset

- 站点：[deepset.ai](https://deepset.ai/)
- llms.txt：1.6K tokens

Haystack 框架厂商，做 RAG 和搜索。1.6K tokens 的极简 llms.txt 是另一种风格——只提供导航入口，让 AI 按需去读具体页面，不把所有内容塞进一个文件。

## 开发者工具与 SaaS

### Sourcegraph

- 站点：[sourcegraph.com/docs](https://sourcegraph.com/docs)
- llms.txt：1.2M tokens

代码搜索与导航平台。1.2M tokens 是整个目录里语料最大的之一，说明 Sourcegraph 把完整的 API 文档、使用指南全部铺进了 llms.txt。对于被 Cursor / Copilot 这类代码 AI 调用的场景，这是最直接的内容投喂。

### Cloudflare Docs

- 站点：[developers.cloudflare.com](https://developers.cloudflare.com)
- llms.txt：34K tokens / llms-full.txt：3.8M tokens

Cloudflare 边缘平台文档。3.8M tokens 的 llms-full.txt 是目录里最大的文件之一，覆盖了 Workers、Pages、R2、D1、KV 等全部产品文档。Cloudflare 是少数同时提供精简版和超大全文版的厂商——让 AI 根据上下文窗口灵活选择。

### Retool

- 站点：[docs.retool.com](https://docs.retool.com)
- llms.txt：32K tokens / llms-full.txt：399K tokens

内部应用构建器平台。Retool 的 llms.txt 实现比较规范，同时提供精简入口和完整文档。

### Better Auth

- 站点：[better-auth.com](https://better-auth.com/)
- llms.txt：174K tokens

开源认证框架。作为一个独立开源项目，174K tokens 的 llms.txt 体量不小，说明作者认真对待 AI 可读性——认证库的文档被 AI 正确理解，直接影响 AI 生成代码的安全性。

### CircleCI

- 站点：[circleci.com](https://circleci.com/)
- llms.txt：1.2K tokens

CI/CD 平台。CircleCI 选择了极简路线，1.2K tokens 只够放一个导航入口。这适合"让 AI 知道 CircleCI 文档在哪"的场景，但要深入配置细节仍需 AI 去爬具体页面。

### Apify

- 站点：[apify.com](https://apify.com/)
- llms.txt：2.5K tokens

爬虫与自动化平台。Apify 做爬虫工具，自己用 llms.txt 给 AI 提供文档——"爬虫公司被 AI 爬"。

### Axiom

- 站点：[axiom.co](https://axiom.co/)
- llms.txt：10K tokens / llms-full.txt：398K tokens

日志与可观测性平台，和 Datadog / Grafana 同赛道。llms.txt 实现规范，同时提供精简和完整版。

### Activepieces

- 站点：[activepieces.com](https://activepieces.com/)
- llms.txt：4.6K tokens / llms-full.txt：57K tokens

开源 Zapier 替代品。作为自动化工具，它的 llms.txt 帮助 AI 理解如何配置各种集成节点——AI 写自动化流程时直接读取文档。

### Terminal Trove

- 站点：[terminaltrove.com](https://terminaltrove.com/)
- llms.txt：360 tokens / llms-full.txt：1.4K tokens

CLI/TUI 工具精选目录。整个目录里最极简的实现之一，360 tokens 只够放一句介绍和几个分类链接。但 Terminal Trove 本身就是工具导航站，llms.txt 极简反而合理——它的价值是"让 AI 知道有哪些 CLI 工具存在"。

### Bun

- 站点：[bun.sh](https://bun.sh)
- 站点：JS 运行时（Node.js 替代品）

Bun 在 llmstxt.site 目录的文件末尾，是最后一个条目。作为新兴 JS 运行时，Bun 的文档本身就很受 AI 关注——大量开发者在用 AI 写 Bun 代码时需要准确的 API 参考。

## 开源框架与文档

### Next.js

- 站点：[nextjs.org/docs](https://nextjs.org/docs)
- llms.txt：14K tokens / llms-full.txt：676K tokens

React 框架文档。Next.js 是 llms.txt 最常被引用的案例之一——前端开发者大量使用 AI 辅助编码，App Router 的配置复杂度高，准确的 Markdown 文档直接提升 AI 生成代码质量。

### Svelte

- 站点：[svelte.dev](https://svelte.dev/)
- llms.txt：281 tokens / llms-full.txt：226K tokens

Svelte UI 框架。281 tokens 是整个目录里最极简的 llms.txt 之一，但它是官方框架文档入口，足够让 LLM 知道去哪找。这证明 llms.txt 不需要大才有价值——它的核心作用是**精确的导航入口**，而非内容仓库。

### Angular

- 站点：[angular.dev](https://angular.dev/)
- llms.txt：1.5K tokens / llms-full.txt：152K tokens

Angular 框架文档。Google 出品，文档规范度高。

### Astro

- 站点：[astro.build](https://astro.build/)
- llms.txt：556 tokens / llms-full.txt：591K tokens

内容导向 Web 框架，做博客和文档站很流行。Astro 自己用 llms.txt 服务 AI，形成"做博客框架的博客被 AI 读取"的循环。

### Hugging Face Transformers

- 站点：[huggingface.co](https://huggingface.co/)
- llms.txt：809K tokens

Transformers 库文档。809K tokens 是整个目录里语料最大的之一，覆盖了完整的模型 API、训练指南、推理示例。Hugging Face 同时为 Transformers、Diffusers、Accelerate、Hub Python Library 四个项目分别提供 llms.txt，是标准的多项目实践。

### Meilisearch

- 站点：[meilisearch.com](https://www.meilisearch.com/)
- llms.txt：331 tokens / llms-full.txt：298K tokens

开源搜索引擎。331 tokens 的极简 llms.txt 和 Svelte 一样走"只做导航"路线。作为搜索引擎项目，用 llms.txt 给 AI 提供搜索入口，有"搜索引擎被 AI 搜索"的意味。

### Stripe

- 站点：[docs.stripe.com](https://docs.stripe.com)
- llms.txt：17K tokens

Stripe 支付 API 文档。支付 API 的准确性要求极高，AI 生成 Stripe 集成代码时必须读准确文档——Stripe 提供 llms.txt 直接解决了"AI 从 HTML 里提取 API 参数容易出错"的问题。

### NVIDIA Developer

- 站点：[developer.nvidia.com](https://developer.nvidia.com)
- llms.txt：5.5K tokens

NVIDIA 开发平台。作为 GPU 厂商，NVIDIA 的 llms.txt 帮助 AI 定位 CUDA、cuDNN、TensorRT 等开发文档。

## 金融与加密

### Bitcoin.com

- 站点：[bitcoin.com](https://www.bitcoin.com/)
- llms.txt：722K tokens

加密新闻、钱包和交易所。722K tokens 的 llms.txt 体量惊人，说明 Bitcoin.com 把大量新闻内容塞进了入口文件——这更像一个"内容投喂"策略，让 AI 一次读取就能覆盖大量加密资讯。

### Chainspect

- 站点：[chainspect.app](https://chainspect.app/)
- llms.txt：938K tokens

区块链分析平台。938K tokens 在金融类里最大，推测包含完整的链上数据分析方法和 API 参考。

### Mangopay

- 站点：[mangopay.com](https://mangopay.com/)
- llms.txt：11K tokens / llms-full.txt：1.7M tokens

嵌入式支付平台。1.7M tokens 的 llms-full.txt 是目录里最大的文件之一，覆盖完整的支付 API、合规流程、沙箱测试指南。

### FinFeedAPI

- 站点：[finfeedapi.com](https://finfeedapi.com)
- llms.txt：8K tokens / llms-full.txt：1.1M tokens

金融市场数据 API。1.1M tokens 的完整文档，涵盖股票、期货、外汇等市场数据接口。金融数据 API 对准确性要求极高，llms.txt 让 AI 获取准确的接口定义而非猜测。

### KuCoin API

- llms.txt：15K tokens

加密交易所 API。交易所 API 的参数复杂（签名、时间戳、限频），AI 生成交易代码时必须读准确文档。

## MCP 生态

### MCP Server Space

- 站点：[mcpserver.space](https://mcpserver.space/)
- llms.txt：1.2K tokens / llms-full.txt：1.2K tokens

MCP 服务器目录站。本身就是为 AI Agent 服务的目录，用 llms.txt 给 AI 提供目录索引——"给 AI 看的 AI 工具目录"。

### uminai MCP Directory

- 站点：[mcp.umin.ai](https://mcp.umin.ai)
- llms.txt：1K tokens

另一个 MCP 服务器目录。MCP 目录站出现在 llms.txt 收录中，说明 MCP 生态与 llms.txt 标准正在交汇——AI Agent 既可以通过 llms.txt 发现网站内容，也可以通过 MCP 目录找到可调用的工具。

## 个人与小众站点

### Huberman Lab

- 站点：[hubermanlab.com](https://www.hubermanlab.com/)
- llms.txt：14K tokens

斯坦福神经科学家 Andrew Huberman 的播客站。名人站使用 llms.txt 比较少见，14K tokens 覆盖了播客集数和主题索引。这让 AI 在被问"Huberman 说过什么关于睡眠的"时能准确定位到具体集数。

### Readwise

- 站点：[readwise.io](https://readwise.io)
- llms.txt：96K tokens

稍后读与笔记应用。96K tokens 的 llms.txt 体量不小，覆盖完整的产品文档。作为"帮人读东西"的工具，用 llms.txt 帮 AI 读自己的文档。

### Listen Notes

- 站点：[listennotes.com](https://www.listennotes.com/)
- llms.txt：1K tokens

播客搜索引擎。1K tokens 的极简入口，让 AI 知道"有一个播客搜索引擎，API 在这里"。

### Light Pollution Map

- 站点：[lightpollutionmap.app](https://lightpollutionmap.app)
- llms.txt：544 tokens / llms-full.txt：1.9K tokens

光污染地图。一个非技术类的数据可视化工具用 llms.txt 比较少见，说明 llms.txt 正在从开发者文档扩展到更广泛的"让 AI 理解我的数据服务"场景。

### Rasul Kireev

- 站点：[rasulkireev.com](https://www.rasulkireev.com/)
- llms.txt：127K tokens

知名开发者个人站。127K tokens 的个人站 llms.txt 是目录里少见的"独立开发者深度实现"——大部分个人站要么不做 llms.txt，要么随便写几行，但这位把博客、项目、笔记都结构化塞了进去。

## 几个观察

### Token 数量两极分化

整个目录的 token 分布呈现明显的双峰：

- **巨无霸派**：Sourcegraph 1.2M、HF Transformers 809K、Bitcoin.com 722K、Cloudflare full 3.8M——这些是文档站，llms.txt 只是冰山一角
- **极简派**：Svelte 281、Terminal Trove 360、Meilisearch 331、Light Pollution Map 544——只做导航入口

两种策略各有道理。巨无霸派适合"一次性读取全站"的 AI Agent，极简派适合"按需深入"的交互式查询。关键是和自己的内容形态匹配——文档站可以做大，工具站适合做小。

### 中间地带缺失

整个目录里，真正的"中小技术博客"很少。要么是 Anthropic / Cloudflare / Next.js 这类一线品牌的深度实现，要么是"填个表蹭热度"的营销页。独立开发者博客、技术笔记站、个人知识库是缺失的那层——这正是本站 PDC 协议想要补的空缺。

### AI 公司自己用 llms.txt

一个有趣的现象：AI 公司（Anthropic、Cursor、Fireworks AI、Hugging Face）是最早采纳 llms.txt 的群体。因为他们最清楚 AI 读 HTML 有多痛苦。这形成了一个信号：**一个网站是否有 llms.txt，某种程度上反映了它的技术团队是否理解 AI 的工作方式。**

## 本站的实践

本站基于 llms.txt 标准，通过 PDC（Parallel Data Channel）协议做了扩展实现，覆盖了标准未涉及的三个空白：

1. **加密文章保护**：加密文章在 API 中返回 `encrypted: true` + 空内容，不泄露正文
2. **分类聚合端点**：自动生成 `/api/categories/<slug>.json` 和 `.md`，按分类聚合全文
3. **MCP 适配层**：生成 `/api/mcp.json` 清单，可通过薄适配器接入 Claude Desktop、Cursor 等

如果你也在做类似的事情，欢迎加入 PDC 采用者网络。完整规范文档位于 `/pdc-protocol.md`。
