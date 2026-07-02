## 问题：静态博客如何接入 MCP

[上一篇文章](/2026/06/22/ai-mcp-protocol/)解析了 MCP 协议的核心设计。MCP 需要 JSON-RPC 服务端处理 POST 请求，而 [PDC](/pdc-protocol.md)的站点是纯静态文件（GitHub Pages），没有服务端逻辑。

这是所有静态站点的共同困境：内容在 GitHub Pages 上，AI 想通过 MCP 标准协议访问，但静态托管不支持 POST 请求。

PDC 的解法是**薄适配器**——一个无状态的中间层，所有数据来自 PDC 静态端点，只做 JSON-RPC 协议转换。适配器支持两种部署模式，覆盖从个人开发到团队共享的全部场景。

## 架构总览

```
模式一（本地 stdio）：
  MCP Host（Qoder / Claude Desktop / Cursor）
      │ stdin/stdout（JSON-RPC 2.0）
      ▼
  mcp-server/index.js（Node.js，零依赖）
      │ fetch → GitHub Pages
      ▼
  /api/mcp.json, /api/index.json, /api/posts/*.md

模式二（远程 HTTP）：
  MCP Host
      │ HTTP POST（JSON-RPC 2.0）
      ▼
  mcp-server/worker.js（Cloudflare Worker）
      │ fetch → GitHub Pages
      ▼
  /api/mcp.json, /api/index.json, /api/posts/*.md
```

两种模式业务逻辑完全相同，区别仅在传输层。核心设计原则：

- **无状态**——适配器不存储任何内容，所有数据来自 PDC 静态端点
- **manifest 驱动**——构建时生成 `/api/mcp.json`，适配器启动时一次性读取
- **零依赖**——纯 JavaScript，不依赖 MCP SDK 或任何 npm 包

## 前置：MCP 清单端点

适配器的一切行为由 `/api/mcp.json` 驱动。这个文件在 `hexo generate` 时由 `scripts/ai-mcp.js` 自动生成，包含 MCP 三大原语的完整定义：

```json
{
  "protocolVersion": "2025-06-18",
  "serverInfo": { "name": "Q's blog-mcp", "version": "1.0.0" },
  "capabilities": {
    "resources": { "subscribe": false, "listChanged": false },
    "tools": { "listChanged": false },
    "prompts": { "listChanged": false }
  },
  "resources": [
    {
      "uri": "https://bsheepcoder.github.io/api/posts/pdc-protocol.md",
      "name": "PDC 协议：让博客同时服务人类与 AI",
      "mimeType": "text/markdown"
    }
  ],
  "resourceTemplates": [
    {
      "uriTemplate": "https://bsheepcoder.github.io/api/posts/{slug}.md",
      "name": "文章正文"
    }
  ],
  "tools": [
    {
      "name": "search_posts",
      "description": "按关键词/分类/标签搜索文章",
      "inputSchema": { "type": "object", "properties": { "query": { "type": "string" } } }
    }
  ],
  "prompts": [
    {
      "name": "prompt-code-review",
      "description": "PR 提交后自动化代码审查",
      "arguments": [{ "name": "diff", "required": true }]
    }
  ]
}
```

适配器读取这个清单后，就能响应 MCP 客户端的所有请求——`resources/list` 返回 `resources` 数组，`tools/list` 返回 `tools` 数组，`prompts/list` 返回 `prompts` 数组。文章更新后只需 `hexo g` 重新生成清单，重启适配器即可。

## PDC → MCP 三原语映射

适配器的核心工作是把 PDC 静态端点映射为 MCP 三大原语：

| PDC 端点 | MCP 原语 | 适配器处理方式 |
|---------|---------|--------------|
| `/api/posts/<slug>.md` | Resources | `resources/read` 时 fetch 该 URI，返回 text |
| `/api/categories/<slug>.md` | Resources | 同上，分类聚合作为 Resource |
| `/llms.txt`、`/ai-context.md`、`/pdc-protocol.md` | Resources | 站点级文档 |
| `/api/index.json` + 内存过滤 | Tools | `tools/call search_posts` 时 fetch index.json，过滤后返回 |
| `/api/posts/<slug>.md` 的 `<prompt>` 标签 | Prompts | `prompts/get` 时 fetch .md，正则提取 `<prompt>` |
| front-matter 的 `prompt_args` | Prompts arguments | 构建时写入 manifest，适配器直接返回 |

### 提示词文章的 front-matter 扩展

PDC 为提示词文章新增了 `prompt_args` 字段，让机器可解析提示词参数：

```yaml
---
title: "提示词：代码审查助手"
categories:
  - [技术, 提示词]
prompt_args:
  - name: diff
    description: "待审查的代码差异（git diff 输出）"
    required: true
---
```

Markdown 的"变量"表格保留给人类读者，`prompt_args` 供 MCP 客户端消费——平行通道理念的延伸。`scripts/ai-mcp.js` 构建时读取此字段，生成 MCP Prompts 的 `arguments` 定义。

## 模式一：本地 stdio

### 适用场景

- 个人开发者本地使用
- 开发调试（配合 `hexo s` 本地预览）
- 不想部署云服务

### 实现

`mcp-server/index.js`，零依赖纯 Node.js，~200 行。核心是一个 stdin 读取循环：

```javascript
process.stdin.on('data', function (chunk) {
  buffer += chunk
  let idx
  while ((idx = buffer.indexOf('\n')) !== -1) {
    const line = buffer.slice(0, idx).trim()
    buffer = buffer.slice(idx + 1)
    if (!line) continue
    handleMessage(line)  // JSON.parse → 分发到 handler → stdout 写响应
  }
})
```

JSON-RPC 2.0 over stdio 的规则很简单：每行一个 JSON 消息，stdin 读请求，stdout 写响应。MCP 规范要求 stdio 传输时 stdout 不能有非 JSON-RPC 内容，所以日志走 stderr。

### 配置

Qoder：Settings → MCP → My Servers → Add：

```json
{
  "mcpServers": {
    "bsheepcoder-blog": {
      "command": "node",
      "args": ["D:/Code/Hexo/blog/mcp-server/index.js"]
    }
  }
}
```

Claude Desktop：编辑 `claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "bsheepcoder-blog": {
      "command": "node",
      "args": ["D:/Code/Hexo/blog/mcp-server/index.js"]
    }
  }
}
```

Cursor：Settings → MCP → Add Server，格式同上。

### 本地开发

配合 `hexo s` 本地预览时，添加环境变量指向 localhost：

```json
{
  "mcpServers": {
    "bsheepcoder-blog": {
      "command": "node",
      "args": ["D:/Code/Hexo/blog/mcp-server/index.js"],
      "env": { "SITE_URL": "http://localhost:4000" }
    }
  }
}
```

### 限制

- 需要本地安装 Node.js 18+
- 需要克隆博客仓库（至少 `mcp-server/` 目录）
- 每台设备都要单独配置
- 文章更新后需重启适配器刷新 manifest 缓存

## 模式二：远程 HTTP（Cloudflare Worker）

### 适用场景

- 跨设备使用
- 团队共享
- 不想安装 Node.js
- 在移动端 MCP 客户端使用

### 为什么选 Cloudflare Workers

| 平台 | 免费额度 | 冷启动 | 部署难度 | 适合 |
|------|---------|--------|---------|------|
| **Cloudflare Workers** | 10 万请求/天 | ~5ms | 粘贴代码即可 | ✅ |
| Vercel | 100 次/天 | ~500ms | 需 npm 项目 | ❌ 额度太低 |
| 自建 VPS | 按月付费 | 无 | 需运维 | ❌ 成本高 |

Workers 的 V8 运行时原生支持 `fetch`，零依赖代码可直接运行，5 秒内完成部署。

### 实现

`mcp-server/worker.js`，与 `index.js` 业务逻辑完全相同，传输层从 stdio 改为 HTTP：

```javascript
export default {
  async fetch(request) {
    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders })
    }

    if (request.method !== 'POST') {
      // GET 请求返回服务信息（方便浏览器直接访问验证）
      return new Response(JSON.stringify({
        server: 'PDC MCP Server',
        site: SITE_URL,
        manifest: MANIFEST_URL
      }), { headers: { 'Content-Type': 'application/json', ...corsHeaders } })
    }

    // POST 请求：解析 JSON-RPC 消息，分发处理
    const msg = await request.json()
    const response = await handleMessage(msg)

    if (response === null) {
      // notification（无 id）返回 202 Accepted
      return new Response(null, { status: 202, headers: corsHeaders })
    }

    return new Response(JSON.stringify(response), {
      headers: { 'Content-Type': 'application/json', ...corsHeaders }
    })
  }
}
```

关键设计点：

1. **CORS 支持**——`Access-Control-Allow-Origin: *`，允许任意 MCP 客户端跨域访问
2. **OPTIONS 预检**——响应 CORS 预检请求
3. **GET 健康检查**——浏览器直接访问 Worker URL 可看到服务信息
4. **Notification 处理**——JSON-RPC notification（无 `id`）返回 202 Accepted 无响应体

### 部署步骤

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com) → Workers & Pages → Create

2. 创建 Worker，名称如 `bsheepcoder-blog-mcp`

3. 将 `mcp-server/worker.js` 的完整内容粘贴到编辑器中

4. 保存部署，记下 URL：

   ```
   https://bsheepcoder-blog-mcp.<你的子域>.workers.dev
   ```

5. 验证：浏览器访问该 URL，应返回 JSON 服务信息

### 配置

Qoder（`type: "sse"`，Qoder 会自动检测 Streamable HTTP）：

```json
{
  "mcpServers": {
    "bsheepcoder-blog": {
      "type": "sse",
      "url": "https://bsheepcoder-blog-mcp.your-subdomain.workers.dev"
    }
  }
}
```

Claude Desktop / Cursor：

```json
{
  "mcpServers": {
    "bsheepcoder-blog": {
      "url": "https://bsheepcoder-blog-mcp.your-subdomain.workers.dev"
    }
  }
}
```

### 优势

- **零安装**——用户只需填 URL，无需 Node.js 或本地文件
- **跨设备**——任意设备、任意位置可用
- **免费**——10 万请求/天足够个人使用
- **全球 CDN**——Cloudflare 边缘节点低延迟
- **自动 HTTPS**——无需配置证书

### 限制

- Worker 运行时无状态，manifest 缓存在请求级别（每次冷启动重新 fetch）
- 需要注册 Cloudflare 账号（免费）
- Worker 代码更新需手动同步（博客更新 `worker.js` 后需重新粘贴到 Cloudflare）

## 暴露的 MCP 能力

### Resources（17 个）

每篇文章映射为一个 MCP Resource，`uri` 指向 `/api/posts/<slug>.md`。客户端通过 `resources/read` 获取纯 Markdown 正文。

```
resources/list  → 返回 manifest.resources 数组
resources/read  → fetch uri 对应的静态文件 → 返回 { uri, mimeType, text }
```

### Tools（3 个）

| 工具 | 输入 | 执行逻辑 |
|------|------|---------|
| `search_posts` | query?, category?, tag? | fetch `/api/index.json` → 内存过滤 title/description/tags |
| `get_post` | slug | fetch `/api/posts/<slug>.md` → 返回正文 |
| `list_categories` | 无 | fetch `/api/index.json` → 聚合分类计数 |

```
tools/call search_posts
  → fetch /api/index.json
  → 按 query/category/tag 过滤
  → 返回 { content: [{ type: "text", text: "找到 3 篇文章：..." }] }
```

### Prompts（2 个）

| Prompt | 参数 | 来源 |
|--------|------|------|
| `prompt-code-review` | diff（required） | 提示词：代码审查助手 |
| `prompt-weekly-report` | records（required） | 提示词：工作周报助手 |

```
prompts/get prompt-code-review { diff: "diff --git a/test.py ..." }
  → fetch /api/posts/prompt-code-review.md
  → 正则提取 <prompt>...</prompt>
  → 替换 {{DIFF}} 为实际参数
  → 返回 { messages: [{ role: "user", content: { type: "text", text: "..." } }] }
```

## 手动测试

### 本地 stdio

```bash
# 启动并发送 initialize
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}' | node mcp-server/index.js

# 测试搜索
echo '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"search_posts","arguments":{"query":"PDC"}}}' | node mcp-server/index.js
```

### 远程 HTTP

```bash
# 测试 initialize
curl -X POST https://your-worker.workers.dev \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}'

# 测试搜索
curl -X POST https://your-worker.workers.dev \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"search_posts","arguments":{"query":"PDC"}}}'
```

### 预期输出

`initialize` 返回：

```json
{
  "jsonrpc": "2.0", "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": { "resources": {}, "tools": {}, "prompts": {} },
    "serverInfo": { "name": "Q's blog-mcp", "version": "1.0.0" }
  }
}
```

`search_posts` 返回：

```json
{
  "jsonrpc": "2.0", "id": 2,
  "result": {
    "content": [{
      "type": "text",
      "text": "找到 1 篇文章：\n\n- **PDC 协议：让博客同时服务人类与 AI** (`pdc-protocol`)\n  ..."
    }]
  }
}
```

## 两种模式对比

| 维度 | 本地 stdio | 远程 HTTP |
|------|-----------|-----------|
| 配置方式 | 本地文件路径 | URL |
| 前置条件 | Node.js 18+ | 无 |
| 部署成本 | 零 | Cloudflare 免费账号 |
| 跨设备 | ❌ 每台设备单独配 | ✅ 填 URL 即用 |
| 团队共享 | ❌ | ✅ |
| 本地开发 | ✅ 配合 hexo s | ❌ 需部署后测试 |
| 延迟 | 本地零延迟 | ~50ms（CDN 边缘） |
| 离线使用 | ✅ | ❌ |
| 文件 | `mcp-server/index.js` | `mcp-server/worker.js` |

**推荐策略**：

- 开发者自己用 → 本地 stdio（配合 `hexo s` 调试）
- 分享给团队或社区 → 远程 HTTP（填 URL 即用）
- 两者同时配置 → 完美兼容所有场景

## 与 PDC 的关系

本文的适配器是 PDC **第五层（MCP 适配层，可选）**的实现。PDC 五层架构：

| 层 | 职责 | 必需 |
|----|------|------|
| 数据通道层 | 8 个静态端点 | 是 |
| 文章格式层 | front-matter schema | 是 |
| 注入层 | after_post_render | 是 |
| 兼容性层 | .nojekyll、neat、robots | 是 |
| **MCP 适配层** | **MCP 清单 + 适配器** | **否** |

MCP 适配层是可选的——不接入 MCP 生态的站点可以忽略此层，PDC 的前四层已完整覆盖 AI 数据通道需求。接入 MCP 的好处是获得标准化协议入口，Qoder、Claude Desktop、Cursor 等客户端开箱即用，无需手动 fetch API。

## 总结

PDC 接入 MCP 生态的核心思路是**薄适配器**：

- **不重写数据层**——所有内容来自 PDC 静态端点
- **不依赖 SDK**——零依赖纯 JavaScript，~200 行
- **两种部署模式**——本地 stdio 和远程 HTTP，覆盖全部场景
- **manifest 驱动**——构建时预计算 MCP 三原语定义，适配器只读取

核心原则：**PDC 负责内容生成，MCP 适配器负责协议转换，GitHub Pages 负责静态托管。三者各司其职，互不耦合。**

## 参考资料

- [PDC 规范](/pdc-protocol.md) — 本站 PDC 完整规范
- [MCP 协议详解：AI 应用的 USB-C 接口](/2026/06/22/ai-mcp-protocol/) — MCP 协议深入解析
- [MCP 官方文档](https://modelcontextprotocol.io/) — 协议规范、SDK、教程
- [Qoder MCP 配置指南](https://docs.qoder.com/user-guide/chat/model-context-protocol) — Qoder 接入 MCP
- [Cloudflare Workers](https://workers.cloudflare.com/) — Serverless 部署平台
- [mcp-server/README.md](https://github.com/Bsheepcoder/Bsheepcoder.github.io) — 完整部署指南
