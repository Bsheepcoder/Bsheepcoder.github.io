## 问题：AI 读到的不是文章，是噪音

Hexo 生成的 HTML 页面中，正文内容只占约 30%，其余全是导航栏、侧边栏、页脚、JavaScript 脚本、CSS 样式等 UI 元素。当 AI Agent 尝试读取文章时，必须从一堆 HTML 标签中提取正文，效率低且容易丢失格式。

### 现有数据源的问题

| 数据源 | 问题 |
|--------|------|
| HTML 页面 | 80% 是 UI 噪音，AI 要从 HTML 中捞正文 |
| search.json | 所有文章挤一个文件；内容被截断；混入 UI 噪音；表格/代码格式丢失 |
| atom.xml | XML 开销；仅限 20 篇；仍含 HTML 标签 |
| ai-index.json | 只有元数据，没有正文 |

核心矛盾：**人类需要丰富的 UI，AI 需要干净的纯内容。两者不能在同一个 HTML 里兼得。**

## 解决方案：平行数据通道

为 AI 提供一条独立的数据通道——人类看 HTML，AI 读 JSON/MD。两条通道并行，互不干扰。

```
hexo generate
  ├── 生成 HTML 页面（人类阅读）     ← Butterfly 主题渲染
  └── 生成 AI 数据文件（机器阅读）   ← scripts/ai-api.js
      ├── /llms.txt                  ← AI 入口
      ├── /llms-full.txt             ← 全文合并
      ├── /api/index.json            ← 文章列表
      └── /api/posts/<slug>.json     ← 单篇 JSON
      └── /api/posts/<slug>.md       ← 单篇纯 Markdown
```

## 实现原理

### 1. Hexo Generator 注册

在 `scripts/` 目录下创建 `ai-api.js`，注册 Hexo 的 generator 钩子：

```javascript
'use strict'

hexo.extend.generator.register('ai-api', function (locals) {
  const posts = locals.posts
  // ... 遍历文章，生成 JSON/MD/TXT 文件
})
```

这个钩子在 `hexo generate` 时自动执行，无需额外命令。

### 2. 原始 Markdown 提取

```javascript
let rawMd = ''
if (post.raw) {
  rawMd = post.raw.replace(/^---[\s\S]*?---\s*/, '')
}
```

用正则去掉 front-matter，保留纯正文。代码围栏、表格、列表等 Markdown 格式完好无损。

### 3. 加密文章保护

```javascript
const isEncrypted = !!post.password
const content = isEncrypted ? '' : rawMd
```

有 `password` 字段的文章（hexo-blog-encrypt），`content` 为空，`encrypted: true`。AI 知道文章存在但无法获取正文。

## 生成的文件

### /llms.txt — AI 入口

类似 `robots.txt` 但面向 AI/LLM。放在站点根目录，包含站点概览和文章索引：

```
# Q's blog

> Bsheepcoder 的技术博客，记录 AI、编程、计算机科学的学习笔记

## Articles

- [RSS 源大全](/api/posts/rss-source-collection.md): 经过实际验证的 RSS 源大全
- [GitHub Pages 完全指南](/api/posts/hexo-github-pages-guide.md): GitHub Pages 原理与限制
- [用 Hexo 搭建认知管理系统](/api/posts/hexo-cognitive-management.md): AI 索引友好的认知管理系统
```

### /api/index.json — 结构化文章列表

```json
{
  "site": "Q's blog",
  "description": "Bsheepcoder 的技术博客",
  "url": "https://bsheepcoder.github.io",
  "post_count": 5,
  "posts": [
    {
      "title": "RSS 源大全：可信信息获取的数据源清单",
      "slug": "rss-source-collection",
      "date": "2026-06-17",
      "categories": ["技术", "建站"],
      "tags": ["RSS", "信息获取", "数据源"],
      "description": "经过实际验证的 RSS 源大全...",
      "url": "https://bsheepcoder.github.io/2026/06/17/rss-source-collection/",
      "api_json_url": "https://bsheepcoder.github.io/api/posts/rss-source-collection.json",
      "api_md_url": "https://bsheepcoder.github.io/api/posts/rss-source-collection.md",
      "encrypted": false
    }
  ]
}
```

### /api/posts/<slug>.json — 单篇结构化数据

```json
{
  "title": "RSS 源大全：可信信息获取的数据源清单",
  "slug": "rss-source-collection",
  "date": "2026-06-17",
  "categories": ["技术", "建站"],
  "tags": ["RSS", "信息获取", "数据源"],
  "description": "经过实际验证的 RSS 源大全...",
  "url": "https://bsheepcoder.github.io/2026/06/17/rss-source-collection/",
  "api_json_url": "https://bsheepcoder.github.io/api/posts/rss-source-collection.json",
  "api_md_url": "https://bsheepcoder.github.io/api/posts/rss-source-collection.md",
  "encrypted": false,
  "content": "## 为什么需要 RSS\n\n在算法推荐泛滥的时代..."
}
```

`content` 字段是完整原始 Markdown，保留代码围栏、表格、列表等格式。

### /api/posts/<slug>.md — 纯 Markdown

零包装、零噪音的纯正文：

```markdown
## 为什么需要 RSS

在算法推荐泛滥的时代，RSS 是唯一让你**主动选择信息源**的方式...

## 可信度分级

| 级别 | 说明 | 特征 |
|------|------|------|
| ⭐⭐⭐ | 源头直供 | 官方机构/学术/大厂自家博客 |
```

### /llms-full.txt — 全文合并

所有文章的完整内容合并到一个文件中，方便 AI 一次性获取全站内容。

## AI 获取文章的标准流程

```
第 1 步：fetch /llms.txt
         → 站点概览 + 所有文章标题和 MD 链接

第 2 步：fetch /api/index.json
         → 结构化文章列表（含元数据和 API 地址）

第 3 步：fetch /api/posts/<slug>.md
         → 纯 Markdown 正文，零噪音

   或：fetch /api/posts/<slug>.json
         → 结构化 JSON（元数据 + Markdown 正文）
```

## 加密文章处理

有 `password` 字段的文章：

```json
{
  "title": "Hexo 博客文章加密实现",
  "encrypted": true,
  "content": ""
}
```

- JSON 中 `encrypted: true`，`content` 为空字符串
- MD 文件内容为 HTML 注释：`<!-- This post is encrypted. Content is not available. -->`
- AI 知道文章存在但无法获取正文
- 不泄露任何加密内容

## 与其他插件的兼容性

| 插件 | 关系 | 处理方式 |
|------|------|---------|
| hexo-blog-encrypt | 加密文章不泄露 | `encrypted: true`，content 为空 |
| hexo-neat | 不压缩 API 文件 | neat 配置中排除 `**/api/**`、`llms.txt`、`llms-full.txt` |
| hexo-generator-searchdb | search.json 仍生成 | 人类搜索用 search.json，AI 用 /api/ 通道 |
| hexo-deployer-git | API 文件随部署推送 | 在 public/ 目录中，自动包含 |
| Butterfly 主题 | 完全不变 | 人类页面不受任何影响 |

### hexo-neat 排除配置

```yaml
# _config.yml
neat_html:
  enable: true
  exclude:
    - '**/lib/**'
    - '**/hbe.*'
    - '**/api/**'
    - 'llms.txt'
    - 'llms-full.txt'
```

## GitHub Pages 兼容性

所有文件都是静态文件（.json/.md/.txt），完美兼容 GitHub Pages：

| 检查项 | 状态 |
|--------|------|
| 静态文件托管 | ✅ 无服务端代码 |
| CORS / 跨域 | ✅ 同域，无跨域问题 |
| 运行时开销 | ✅ 零开销，构建时生成 |
| 部署 | ✅ 随 hexo deploy 一起推送 |
| HTTPS | ✅ GitHub Pages 自动提供 |

## 容量分析

| 指标 | 数值 |
|------|------|
| 单篇增量开销 | ~102 KB（HTML + MD + JSON + 索引分摊） |
| GitHub Pages 上限 | 1 GB |
| 固定开销（主题等） | ~1.13 MB |
| 理论最大文章数 | ~10,275 篇 |
| 月带宽限制 | 100 GB（约 8.5 万次访问） |

按每天写 1 篇计算，可写 28 年才会触及容量上限。

## 完整实现代码

```javascript
// scripts/ai-api.js
'use strict'

hexo.extend.generator.register('ai-api', function (locals) {
  const posts = locals.posts
  const config = hexo.config
  const siteUrl = (config.url || '').replace(/\/$/, '')
  const resultList = []
  const jsonList = []
  const mdList = []
  let llmsLines = []
  let llmsFullLines = []

  const siteTitle = config.title || 'Blog'
  const siteDesc = config.description || ''
  llmsLines.push(`# ${siteTitle}`, '')
  llmsLines.push(`> ${siteDesc}`, '')
  llmsLines.push('', '## Articles', '')
  llmsFullLines.push(`# ${siteTitle}`, '')
  llmsFullLines.push(`> ${siteDesc}`, '')

  posts.sort('-date').forEach(function (post) {
    const slug = post.slug
    const title = post.title || slug
    const dateStr = post.date ? post.date.format('YYYY-MM-DD') : ''
    const updatedStr = post.updated ? post.updated.format('YYYY-MM-DD') : dateStr
    const categories = (post.categories && post.categories.data)
      ? post.categories.data.map(function (c) { return c.name }) : []
    const tags = (post.tags && post.tags.data)
      ? post.tags.data.map(function (t) { return t.name }) : []
    const desc = post.description || ''
    const isEncrypted = !!post.password
    const postUrl = siteUrl + '/' + post.path
    const apiJsonUrl = '/api/posts/' + slug + '.json'
    const apiMdUrl = '/api/posts/' + slug + '.md'

    // 提取原始 Markdown（去掉 front-matter）
    let rawMd = ''
    if (post.raw) {
      rawMd = post.raw.replace(/^---[\s\S]*?---\s*/, '')
    } else if (post.content) {
      rawMd = post.content
    }

    // JSON 对象
    const jsonObj = {
      title: title,
      slug: slug,
      date: dateStr,
      updated: updatedStr,
      categories: categories,
      tags: tags,
      description: desc,
      url: postUrl,
      api_json_url: siteUrl + apiJsonUrl,
      api_md_url: siteUrl + apiMdUrl,
      encrypted: isEncrypted,
      content: isEncrypted ? '' : rawMd
    }

    // MD 内容
    const mdContent = isEncrypted
      ? '<!-- This post is encrypted. Content is not available. -->'
      : rawMd

    // 列表条目
    const listEntry = {
      title: title,
      slug: slug,
      date: dateStr,
      categories: categories,
      tags: tags,
      description: desc,
      url: postUrl,
      api_json_url: siteUrl + apiJsonUrl,
      api_md_url: siteUrl + apiMdUrl,
      encrypted: isEncrypted
    }
    resultList.push(listEntry)

    // llms.txt 条目
    llmsLines.push(`- [${title}](${apiMdUrl}): ${desc || title}`)

    // llms-full.txt 条目
    if (!isEncrypted) {
      llmsFullLines.push('', '---', '')
      llmsFullLines.push(`# ${title}`, '')
      llmsFullLines.push(`Date: ${dateStr} | Tags: ${tags.join(', ')} | URL: ${postUrl}`, '')
      llmsFullLines.push(mdContent)
    }

    // 生成单篇 JSON
    jsonList.push({
      path: 'api/posts/' + slug + '.json',
      data: JSON.stringify(jsonObj, null, 2)
    })

    // 生成单篇 MD
    mdList.push({
      path: 'api/posts/' + slug + '.md',
      data: mdContent
    })
  })

  // api/index.json
  const apiIndex = {
    site: siteTitle,
    description: siteDesc,
    url: siteUrl,
    post_count: resultList.length,
    posts: resultList
  }

  const result = [
    { path: 'api/index.json', data: JSON.stringify(apiIndex, null, 2) },
    { path: 'llms.txt', data: llmsLines.join('\n') + '\n' },
    { path: 'llms-full.txt', data: llmsFullLines.join('\n') + '\n' }
  ]

  jsonList.forEach(function (f) { result.push(f) })
  mdList.forEach(function (f) { result.push(f) })

  return result
})
```

## 验证方法

### 本地验证

```powershell
# 构建后检查文件存在
Test-Path public\llms.txt, public\api\index.json, public\api\posts

# 检查加密文章 content 为空
$json = Get-Content public\api\posts\encrypted-post.json -Raw | ConvertFrom-Json
$json.encrypted  # 应为 true
$json.content    # 应为空字符串
```

### 线上验证

| 端点 | URL |
|------|-----|
| llms.txt | `https://bsheepcoder.github.io/llms.txt` |
| api/index.json | `https://bsheepcoder.github.io/api/index.json` |
| 单篇 MD | `https://bsheepcoder.github.io/api/posts/<slug>.md` |
| 单篇 JSON | `https://bsheepcoder.github.io/api/posts/<slug>.json` |
| 全文合并 | `https://bsheepcoder.github.io/llms-full.txt` |

## 总结

这个方案的核心价值：

- **零噪音** — AI 读到的是原始 Markdown，不是 HTML
- **零运行时开销** — 构建时生成静态文件，无服务端逻辑
- **完美兼容 GitHub Pages** — 纯静态文件，随部署推送
- **加密保护** — 加密文章不泄露正文
- **不改主题** — 人类看到的页面完全不变
- **容量无忧** — 单篇仅 ~102 KB，1GB 限制可存 1 万+ 篇

核心原则：**为人类和 AI 提供平行的数据通道，各取所需，互不干扰。**
