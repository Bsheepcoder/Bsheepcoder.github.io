# PDC — llms.txt Enhanced for Hexo

> 本文件定义 PDC（Parallel Data Channel）的完整规范：基于 [llms.txt](https://llmstxt.org) 标准，为 Hexo 博客扩展结构化索引、分类聚合、加密保护、MCP 适配等端点。面向 AI Agent 和开发者。

## 概述

[llms.txt](https://llmstxt.org) 是由 Jeremy Howard（fast.ai）提出的标准，定义了 `/llms.txt` 文件格式，让网站为 LLM 提供 Markdown 友好的内容概览。PDC（Parallel Data Channel，平行数据通道）是 llms.txt 标准的 **Hexo 增强实现**——在完全兼容 llms.txt 的基础上，扩展了以下能力：

- 结构化文章索引（`/api/index.json`，含 tags/categories/description）
- 单篇纯 Markdown 正文（`/api/posts/<slug>.md`，零噪音）
- 按分类聚合（`/api/categories/<slug>.json`）
- 加密文章保护（AI 知道存在但不泄露正文）
- MCP 适配层（可选，接入 Claude Desktop / Cursor 等）
- 采用者互链网络（`/api/adopters.json`）

核心原则：`data.raw`（源 Markdown）是 AI 通道的数据源，`data.content`（渲染后 HTML）是人类通道的数据源。两者独立，注入只改后者。

## 五层架构

| 层 | 职责 | 实现位置 | 必需 |
|----|------|---------|------|
| 数据通道层 | 基于 llms.txt 的 8+ 个静态端点 | scripts/ai-api.js, scripts/ai-category.js | 是 |
| 文章格式层 | front-matter schema、命名、分类映射 | _config.yml, source/_posts/*.md | 是 |
| 注入层 | after_post_render 只改 data.content | scripts/post-header-inject.js | 是 |
| 兼容性层 | .nojekyll、neat 排除、robots.txt | _config.yml, source/robots.txt | 是 |
| MCP 适配层 | MCP 清单生成 + 适配器（接入 MCP 生态） | scripts/ai-mcp.js, mcp-server/ | 否 |

## 数据通道端点

### llms.txt 标准端点（兼容）

| 端点 | 格式 | 用途 | llms.txt 规范 |
|------|------|------|--------------|
| /llms.txt | TXT | AI 入口：站点概览 + 资源指针 + 文章索引 | ✅ 完全兼容 |
| /api/posts/<slug>.md | MD | 单篇纯 Markdown 正文（零噪音） | ✅ 等价于 llms.txt 的 .md 版本 |

### PDC 扩展端点

| 端点 | 格式 | 用途 | 生成脚本 |
|------|------|------|---------|
| /ai-context.md | MD | 站点写作规范与分类体系 | source/ai-context.md（skip_render） |
| /llms-full.txt | TXT | 全部文章全文合并 | scripts/ai-api.js |
| /api/index.json | JSON | 结构化文章列表（title/url/tags/categories/description） | scripts/ai-api.js |
| /api/posts/<slug>.json | JSON | 单篇：元数据 + 原始 Markdown 正文 | scripts/ai-api.js |
| /api/categories/<slug>.json | JSON | 按分类聚合的文章列表 | scripts/ai-category.js |
| /api/categories/<slug>.md | MD | 按分类聚合的全文合并 | scripts/ai-category.js |
| /api/mcp.json | JSON | MCP 清单：Resources/Tools/Prompts 定义 | scripts/ai-mcp.js |
| /api/adopters.json | JSON | llms.txt 采用者列表 | scripts/ai-adopters.js |

### 渐进式披露层级

- 入口层：/llms.txt, /ai-context.md（站点概览，~500 token）
- 索引层：/api/index.json, /api/categories/*.json（文章列表，~100 token/篇）
- 内容层：/api/posts/*.md, /llms-full.txt（完整正文，按需获取）

## 文章格式规范

### Front-Matter 必填字段

- title：文章标题（双引号包裹）
- date：创建时间（YYYY-MM-DD HH:mm:ss）
- categories：二级分类 [大类, 子类]，与文件名前缀对齐
- tags：2-5 个，第一个与文件名前缀对齐
- description：50-150 字摘要，供搜索索引和 AI 检索

### 可选字段

- updated：更新时间
- mathjax：true（需数学公式时）
- cover：封面图路径
- password：加密文章密码

### 文件命名

`{领域前缀}-{子主题}.md`，全小写连字符，不用纯数字后缀，不用下划线。

### 分类映射

category_map 和 tag_map 将中文分类名映射为英文 slug，保证 API 路径全 ASCII。新增分类时必须配置映射。

### 提示词文章扩展

prompt- 前缀文章用 `<prompt>...</prompt>` 包裹整段提示词，内部用 XML 标签组织（role/context/instructions/output_format/stop_rules 等，按需选择不强制全套）。AI 提取流程：

1. GET /api/index.json → 筛选 categories 含 "提示词" 的文章
2. GET /api/posts/<slug>.md → 提取 <prompt>...</prompt> 之间的内容
3. 根据 "变量" 表替换动态部分 → 直接使用

## 注入规范

### 过滤器

after_post_render，只对 layout === 'post' 生效。

### 规则

- 只改 data.content（渲染后 HTML），不碰 data.raw（源 Markdown）
- 注入内容：AI 数据通道链接（JSON/MD 绝对 URL）
- 注入位置：data.content 开头
- 样式走 source/css/custom.css（已被 _config.butterfly.yml 的 inject.head 引用）

### 效果

- 人类通道（HTML）：文章开头显示 AI 数据通道链接
- 机器通道（MD/JSON）：零污染，不含注入内容

## 加密处理规范

有 password 字段的文章（hexo-blog-encrypt）：

- JSON：encrypted: true，content 为空字符串
- MD：内容为 `<!-- This post is encrypted. Content is not available. -->`
- AI 知道文章存在（有元数据），但无法获取正文
- 人类通道：密码解锁后正常阅读

## 兼容性规范

### .nojekyll

必须在 _config.yml 的 include 中配置 .nojekyll。GitHub Pages 默认用 Jekyll，会忽略 _ 开头的文件和部分 .json 文件。.nojekyll 禁用 Jekyll 处理，确保 /api/*.json 等端点可访问。

### hexo-neat 排除

neat_html.exclude 必须包含：

- **/lib/**
- **/hbe.*
- **/api/**
- ai-context.md
- pdc-protocol.md

API 文件是纯文本，HTML 压缩会破坏 JSON 结构。

### robots.txt

- User-agent: *，Allow: /
- Disallow: /api/
- Sitemap 声明

搜索引擎走 HTML 通道，AI 通道不进搜索索引。

## AI 检索标准流程

1. GET /llms.txt → 站点概览 + 文章索引
2. GET /ai-context.md → 了解写作规范与分类体系
3. 获取内容（三选一）：
   - GET /llms-full.txt（全站一次性获取，< 50 篇推荐）
   - GET /api/index.json → 筛选 → GET /api/posts/<slug>.md
   - GET /api/categories/<slug>.md（按分类批量获取）

## 与 llms.txt 标准的关系

PDC **基于** [llms.txt](https://llmstxt.org) 标准，完全兼容 `/llms.txt` 文件格式。在 llms.txt 基础上，PDC 扩展了结构化索引、单篇内容、分类聚合、加密保护、内容注入等端点。

| 维度 | llms.txt 标准 | PDC 扩展 |
|------|--------------|---------|
| /llms.txt | ✅ 标准格式 | ✅ 完全兼容 |
| 单篇 .md | ✅ URL + .md | ✅ /api/posts/<slug>.md |
| 结构化索引 | ❌ | ✅ /api/index.json |
| 分类聚合 | ❌ | ✅ /api/categories/*.json |
| 加密保护 | ❌ | ✅ encrypted 标记 |
| MCP 适配 | ❌ | ✅ /api/mcp.json |
| 采用者网络 | ❌ | ✅ /api/adopters.json |

**定位**：PDC 不是 llms.txt 的竞争标准，而是 llms.txt 的 Hexo 增强实现。如果你的站点不需要 PDC 扩展端点，仅实现 llms.txt 标准即可。

## 参考实现

本站（https://bsheepcoder.github.io）是 PDC 的参考实现。

| 组件 | 文件 |
|------|------|
| 数据通道生成 | scripts/ai-api.js |
| 分类聚合生成 | scripts/ai-category.js |
| MCP 清单生成 | scripts/ai-mcp.js |
| 内容注入 | scripts/post-header-inject.js |
| 采用者列表生成 | scripts/ai-adopters.js |
| 样式 | source/css/custom.css |
| 站点规范 | source/ai-context.md |
| 规范文档 | source/pdc-protocol.md |
| Hexo 配置 | _config.yml |
| 主题配置 | _config.butterfly.yml |

## llms.txt 采用者网络

PDC 基于 llms.txt 标准，采用者网络也是 llms.txt 生态的一部分。

### 如何加入

1. **实现 llms.txt + PDC 扩展端点**：在你的 Hexo/静态站点上实现 `/llms.txt`、`/api/index.json`、`/api/posts/<slug>.md` 等端点
2. **提交申请**：在 [GitHub Issues](https://github.com/Bsheepcoder/Bsheepcoder.github.io/issues) 创建 Issue，填写站点信息（站点名、URL、头像 URL、一句话描述）
3. **自动验证**：验证脚本（[pdc-protocol-verify](https://github.com/Bsheepcoder/pdc-protocol-verify)）会检查你的 `/llms.txt` 和 `/api/index.json` 是否可访问
4. **互链**：验证通过后，你的站点将出现在[友链页](/flink/)和 `/api/adopters.json` 中

### 采用者权利

- 出现在友链页面，获得反向链接
- 列入 `/api/adopters.json`，其他 AI/Agent 可发现你的站点
- 网络成员之间形成内容互链，提升整体搜索可见性

### 采用者义务

- 维持 llms.txt 端点可访问
- 在站点可见位置标注 llms.txt 采用

### 验证脚本

验证脚本和完整使用说明位于 [pdc-protocol-verify](https://github.com/Bsheepcoder/pdc-protocol-verify) 仓库：

```bash
git clone https://github.com/Bsheepcoder/pdc-protocol-verify.git
cd pdc-protocol-verify
node verify.js
```

## MCP 适配层（可选）

PDC 可通过薄适配器接入 MCP（Model Context Protocol）生态，让 Claude Desktop、Cursor 等 MCP 客户端直接访问站点内容。

### 设计原则

- **无状态适配器**：适配器不存储内容，所有数据来自 PDC 静态端点
- **https URI 直连**：MCP Resources 的 uri 指向 GitHub Pages 静态文件，客户端自行 fetch，适配器不代理内容传输
- **构建时预计算**：MCP 清单（/api/mcp.json）在 hexo generate 时一次性生成，适配器运行时只读取

### MCP 清单端点

GET /api/mcp.json 返回 MCP 三大原语的完整定义：

- resources：每篇文章作为一个 Resource，uri 指向 /api/posts/<slug>.md
- resourceTemplates：URI 模板（/api/posts/{slug}.md 等），客户端可自动补全
- tools：search_posts、get_post、list_categories 三个工具定义
- prompts：prompt- 前缀文章映射为 Prompts，arguments 从 front-matter 的 prompt_args 读取

### PDC → MCP 三原语映射

| PDC 端点 | MCP 原语 | 映射方式 |
|---------|---------|---------|
| /api/posts/<slug>.md | Resources | uri 指向静态文件，客户端直连 fetch |
| /api/categories/<slug>.md | Resources | 分类聚合作为 Resource |
| /llms.txt、/ai-context.md、/pdc-protocol.md | Resources | 站点级文档 |
| /api/index.json + 内存过滤 | Tools | search_posts(query, category, tag) |
| /api/posts/<slug>.md 的 <prompt> 标签 | Prompts | prompts/get 提取并返回 |
| front-matter 的 prompt_args | Prompts arguments | 机器可解析的参数定义 |

### 提示词文章扩展字段

prompt- 前缀文章的 front-matter 可选字段 prompt_args，供 MCP Prompts 读取：

```yaml
prompt_args:
  - name: diff
    description: "待审查的代码差异"
    required: true
    example: "git diff main..feature 的输出"
```

Markdown 的"变量"表格保留给人类读者，prompt_args 供机器消费——平行通道理念的延伸。

### 适配器参考架构

两种部署模式，业务逻辑相同，传输层不同：

```
模式一（本地 stdio）：
  MCP Host（Qoder / Claude Desktop / Cursor）
      │ stdio（JSON-RPC 2.0）
      ▼
  mcp-server/index.js（Node.js，零依赖，无状态）
      │ fetch → GitHub Pages 静态端点
      ▼
  /api/mcp.json, /api/index.json, /api/posts/*.md, ...

模式二（远程 HTTP，推荐）：
  MCP Host
      │ HTTP POST（JSON-RPC 2.0）
      ▼
  mcp-server/worker.js（Cloudflare Worker，无状态）
      │ fetch → GitHub Pages 静态端点
      ▼
  /api/mcp.json, /api/index.json, /api/posts/*.md, ...
```

| 模式 | 文件 | 传输 | 配置 | 适用 |
|------|------|------|------|------|
| 本地 stdio | `mcp-server/index.js` | stdin/stdout | 本地文件路径 | 开发者本地 |
| 远程 HTTP | `mcp-server/worker.js` | Streamable HTTP | URL | 跨设备、零安装 |

远程模式部署到 Cloudflare Workers（免费 10 万请求/天），用户只需填 URL。详见 `mcp-server/README.md`。
