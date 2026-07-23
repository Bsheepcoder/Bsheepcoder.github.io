## 问题：一个页面服务两类读者

传统博客只为人类设计。当 AI Agent 尝试读取博客内容时，面临三个核心矛盾：

| 矛盾 | 人类需求 | AI 需求 |
|------|---------|---------|
| 格式 | HTML（富渲染、样式、交互） | Markdown（纯结构、零噪音） |
| 噪音 | 导航栏、侧边栏、评论框是必要 UI | 同样的内容是 80% 噪音 |
| 加密 | 密码解锁后可读正文 | API 不应泄露任何加密内容 |

PDC 的核心洞察是：**不要试图在同一个通道里同时满足两类读者**。为人类和 AI 各建一条独立的数据通道，构建时一次生成，互不干扰。

## PDC 是什么

**PDC（Parallel Data Channel，平行数据通道（PDC））** 是 llms.txt 标准的 Hexo 增强实现。它定义了如何让同一份内容以两种形态并行输出：

```
hexo generate
  ├── 生成 HTML 页面（人类通道）      ← 主题渲染 + after_post_render 注入
  └── 生成 AI 数据文件（机器通道）    ← scripts/ai-api.js 构建
      ├── /llms.txt                  ← AI 入口
      ├── /ai-context.md             ← 站点规范
      ├── /llms-full.txt             ← 全文合并
      ├── /api/index.json            ← 结构化索引
      ├── /api/posts/<slug>.json     ← 单篇 JSON
      ├── /api/posts/<slug>.md       ← 单篇纯 Markdown
      ├── /api/categories/<slug>.json ← 分类列表
      └── /api/categories/<slug>.md  ← 分类全文
```

两条通道在构建时分叉，在部署时合并（同一个 `public/` 目录），在运行时完全独立。

## 四层架构

PDC 由四层组成，每层解决一个独立问题。

### 第一层：数据通道（Data Channel）

8 个静态端点，覆盖从发现到获取的完整链路。

| 端点 | 格式 | 用途 | 渐进式层级 |
|------|------|------|-----------|
| `/llms.txt` | TXT | AI 入口：站点概览 + 资源指针 + 文章索引 | 入口 |
| `/ai-context.md` | MD | 站点写作规范与分类体系 | 入口 |
| `/llms-full.txt` | TXT | 全部文章全文合并（一个文件） | 索引 |
| `/api/index.json` | JSON | 结构化文章列表（含元数据 + API 地址） | 索引 |
| `/api/posts/<slug>.json` | JSON | 单篇：元数据 + 原始 Markdown 正文 | 内容 |
| `/api/posts/<slug>.md` | MD | 单篇：纯 Markdown 正文，零噪音 | 内容 |
| `/api/categories/<slug>.json` | JSON | 按分类聚合的文章列表 | 索引 |
| `/api/categories/<slug>.md` | MD | 按分类聚合的全文合并 | 内容 |

设计遵循**渐进式披露**：AI 先读入口（站点概览），再读索引（文章列表），最后读内容（单篇正文）。每层 token 开销递增，AI 按需获取。

### 第二层：文章格式（Content Schema）

数据通道的质量取决于输入内容的质量。PDC 定义了严格的文章格式规范。

**Front-Matter 必填字段**：

```yaml
---
title: "文章标题"
date: 2026-06-22 10:30:00
categories:
  - [技术, 建站]
tags:
  - Hexo
  - AI
description: "50-150 字摘要，供搜索索引和 AI 检索"
---
```

**文件命名规范**：`{领域前缀}-{子主题}.md`，全小写连字符，前缀与分类对齐。例如 `ai-history.md` → 分类 `[技术, 人工智能]`。

**分类映射**：`category_map` 和 `tag_map` 将中文分类名映射为英文 slug，保证 API 路径全 ASCII。

**提示词文章扩展**：`prompt-` 前缀文章是可复用结构化文本，用 `<prompt>...</prompt>` 包裹整段提示词，内部用 XML 标签组织（`<role>`/`<context>`/`<instructions>`/`<output_format>`/`<stop_rules>`），AI 可通过 API 直接提取使用。

### 第三层：内容注入（Injection）

人类页面需要显示 AI 数据通道链接（方便复制 URL），但 AI 读取的 Markdown 不能包含这些链接（否则自我引用循环）。

PDC 的解决方案是利用 Hexo 的 `after_post_render` 过滤器，**只改 `data.content`（渲染后 HTML），不碰 `data.raw`（源 Markdown）**：

```javascript
hexo.extend.filter.register('after_post_render', function (data) {
  if (data.layout !== 'post') return data
  const slug = data.slug
  if (!slug) return data

  const siteUrl = (hexo.config.url || '').replace(/\/$/, '')
  const aiLinks = `<span class="ai-data-links">...JSON/MD 链接...</span>`
  const meta = `<div class="post-header-meta">${aiLinks}</div>`

  data.content = meta + data.content  // HTML 通道：注入链接
  // data.raw 不变 → AI 通道：零污染
  return data
})
```

结果：人类看到的 HTML 文章开头有「AI 数据通道：JSON · Markdown」链接；AI 读取的 `/api/posts/<slug>.md` 是纯净的原始 Markdown，不含任何注入内容。

### 第四层：兼容性（Compatibility）

静态托管平台（如 GitHub Pages）有特定限制，PDC 定义了兼容性规范：

| 规范 | 配置 | 原因 |
|------|------|------|
| `.nojekyll` 必须存在 | `_config.yml` 的 `include` 中 | GitHub Pages 默认用 Jekyll，会忽略 `_` 开头和部分 `.json` 文件 |
| hexo-neat 排除 API | `neat_html.exclude` 加 `**/api/**` | API 文件是纯文本，HTML 压缩会破坏 JSON 结构 |
| robots.txt 禁止 `/api/` | `source/robots.txt` 的 `Disallow` | 搜索引擎走 HTML 通道，AI 通道不进搜索索引 |
| 加密文章保护 | `password` 字段 → `encrypted: true` | JSON `content` 为空，MD 为 HTML 注释，不泄露正文 |

## AI 检索流程

PDC 定义了标准的 AI 检索流程：

```
第 1 步：GET /llms.txt
         → 站点概览 + AI Resources 指针 + 文章索引
         → 知道有哪些文章、有哪些 API 端点

第 2 步：GET /ai-context.md
         → 站点写作规范与分类体系
         → 了解命名规则、分类映射、加密策略

第 3 步：获取内容（三选一）
         ├── GET /llms-full.txt          ← 全站一次性获取（< 50 篇推荐）
         ├── GET /api/index.json          → 筛选 → GET /api/posts/<slug>.md
         └── GET /api/categories/<slug>.md ← 按分类批量获取
```

**小站策略**（< 50 篇）：直接 `GET /llms-full.txt` 一次获取全站内容。现代 LLM 上下文窗口可容纳。

**大站策略**（> 50 篇）：`GET /api/index.json` → 按需 `GET /api/posts/<slug>.md`，避免一次性传输过大文件。

## 加密文章处理

PDC 明确定义了加密文章的处理规则，确保 AI 通道不泄露加密内容：

```json
{
  "title": "加密文章标题",
  "slug": "encrypted-post",
  "encrypted": true,
  "content": ""
}
```

- JSON：`encrypted: true`，`content` 为空字符串
- MD：内容为 `<!-- This post is encrypted. Content is not available. -->`
- AI 知道文章存在（有元数据），但无法获取正文
- 人类通道：密码解锁后正常阅读

## 与 llms.txt 标准的关系

[llms.txt](https://llmstxt.org) 是一个提议中的标准，定义了 `/llms.txt` 文件的格式。PDC 基于 llms.txt 标准，同时扩展了更多端点：

| 维度 | llms.txt 标准 | PDC |
|------|--------------|---------|
| 入口文件 | `/llms.txt` | `/llms.txt`（兼容） |
| 结构化索引 | 未定义 | `/api/index.json` |
| 单篇内容 | 未定义 | `/api/posts/<slug>.{json,md}` |
| 分类聚合 | 未定义 | `/api/categories/<slug>.{json,md}` |
| 加密保护 | 未定义 | `encrypted: true` + content 为空 |
| 内容注入 | 未定义 | `after_post_render` 规范 |

PDC 是 llms.txt 的增强实现。

## 设计原则

### 1. 平行通道，非转换通道

不是「把 HTML 转成 Markdown」，而是「构建时从源 Markdown 分叉出两条通道」。转换会损失信息（代码围栏、表格格式），分叉不会。

### 2. 构建时生成，非运行时计算

所有 AI 数据文件在 `hexo generate` 时一次性生成，部署后是纯静态文件。无服务端逻辑，无运行时开销，完美兼容 GitHub Pages。

### 3. 渐进式披露

AI 不需要一次获取所有内容。入口 → 索引 → 内容三层结构，token 开销逐层递增，AI 按需获取。

### 4. 源与渲染分离

`data.raw`（源 Markdown）是 AI 通道的数据源，`data.content`（渲染后 HTML）是人类通道的数据源。两者独立，注入只改后者。

## 容量分析

| 指标 | 数值 |
|------|------|
| 单篇增量开销 | ~102 KB（HTML + MD + JSON + 索引分摊） |
| GitHub Pages 上限 | 1 GB |
| 固定开销（主题等） | ~1.13 MB |
| 理论最大文章数 | ~10,275 篇 |
| 月带宽限制 | 100 GB（约 8.5 万次访问） |

按每天写 1 篇计算，可写 28 年才会触及容量上限。

## 本站实现

本站是 PDC 的参考实现，完整实现清单：

| 组件 | 文件 | 职责 |
|------|------|------|
| 数据通道生成 | `scripts/ai-api.js` | 生成 llms.txt、llms-full.txt、api/index.json、api/posts/*.{json,md} |
| 分类聚合生成 | `scripts/ai-category.js` | 生成 api/categories/*.{json,md} |
| 内容注入 | `scripts/post-header-inject.js` | after_post_render 注入 AI 数据通道链接 |
| 样式 | `source/css/custom.css` | 注入块的样式（暗色模式自适应） |
| 站点规范 | `source/ai-context.md` | 站点写作规范与分类体系 |
| 协议文档 | `source/pdc-protocol.md` | PDC 完整规范（skip_render） |
| Hexo 配置 | `_config.yml` | skip_render、include .nojekyll、neat 排除 |
| 主题配置 | `_config.butterfly.yml` | inject.head 引入 custom.css |

## 协议规范文档

本文是 PDC 的读者向介绍。完整的协议规范文档（面向 AI 和开发者）位于：

```
GET /pdc-protocol.md
```

该文档定义了所有端点的格式、字段、生成规则，可作为实现 llms.txt + PDC 扩展端点的参考。

## 总结

PDC 的核心价值：

- **平行通道** — 人类看 HTML，AI 读 Markdown，互不干扰
- **零噪音** — AI 读到的是原始 Markdown，不是 HTML
- **零运行时开销** — 构建时生成静态文件，无服务端逻辑
- **加密保护** — 加密文章不泄露正文
- **静态托管兼容** — 完美适配 GitHub Pages
- **渐进式披露** — AI 按需获取，token 开销可控

核心原则：**为人类和 AI 提供平行的数据通道，各取所需，互不干扰。**
