## 什么是 GitHub Pages

GitHub Pages 是 GitHub 提供的**静态网站托管服务**。它直接从 GitHub 仓库读取 HTML/CSS/JS 文件，通过 CDN 分发到全球，无需自己搭建服务器。

核心定位：**为项目文档、个人博客、开源项目主页提供零成本的静态站点托管**。

## 工作原理

### 请求流程

```
用户访问 https://username.github.io/repo/
          ↓
GitHub Pages CDN 节点（全球分布）
          ↓
从仓库的特定分支读取静态文件
          ↓
返回 HTML/CSS/JS 给浏览器
```

### 构建方式

GitHub Pages 支持两种模式：

**1. 直接部署模式（Deploy from a branch）**

```
仓库分支（如 master/main 或 gh-pages）
  ↓ 直接提供文件
GitHub Pages CDN
```

适用于已经构建好的静态文件（如 Hexo 的 `public/` 目录直接推送到仓库）。

**2. GitHub Actions 构建模式**

```
源码分支（如 main）
  ↓ GitHub Actions 自动构建
构建产物
  ↓
GitHub Pages CDN
```

适用于需要编译的项目（如 Jekyll、Hugo、Next.js 等），GitHub 自动运行构建流程。

### 域名机制

| 类型 | 域名 | 说明 |
|------|------|------|
| 用户/组织站点 | `username.github.io` | 仓库名必须是 `username.github.io` |
| 项目站点 | `username.github.io/repo` | 任意仓库名，路径为仓库名 |
| 自定义域名 | `blog.example.com` | CNAME 指向 `username.github.io` |

用户站点（`username.github.io`）是特殊的：
- 仓库名必须与用户名完全匹配
- 访问根域名 `https://username.github.io` 时直接提供内容
- 每个账户只能有一个用户站点

项目站点的 URL 格式是 `https://username.github.io/repo-name/`，注意末尾的路径前缀。如果使用 Hexo，需要在 `_config.yml` 中配置 `root: /repo-name/`。

## 配置方法

### 1. 通过分支直接部署

最常见的 Hexo 部署方式：

```yaml
# _config.yml
deploy:
  type: git
  repo: https://github.com/username/username.github.io.git
  branch: master
```

执行 `hexo deploy` 后，`public/` 目录的内容会被推送到仓库的 `master` 分支，GitHub Pages 直接从该分支提供文件。

### 2. 通过 GitHub Actions 部署

在仓库中创建 `.github/workflows/pages.yml`：

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npx hexo generate
      - uses: actions/upload-pages-artifact@v3
        with:
          path: public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

### 3. 自定义域名

在仓库的 Settings → Pages → Custom domain 填入域名，然后：

DNS 配置（二选一）：

```
# 方式一：CNAME 记录（推荐）
blog.example.com  CNAME  username.github.io

# 方式二：A 记录（指向 GitHub Pages IP）
blog.example.com  A  185.199.108.153
blog.example.com  A  185.199.109.153
blog.example.com  A  185.199.110.153
blog.example.com  A  185.199.111.153
```

GitHub 会自动为自定义域名签发 HTTPS 证书（通过 Let's Encrypt）。

## 使用限制

### 免费账户 vs 付费账户

| 特性 | 免费 (GitHub Free) | GitHub Pro ($4/月) |
|------|-------------------|-------------------|
| 公开仓库的 Pages | ✅ 支持 | ✅ 支持 |
| 私有仓库的 Pages | ❌ 不支持 | ✅ 支持 |
| 构建频率 | 每小时最多 10 次 | 每小时最多 10 次 |
| 站点大小 | 最大 1GB | 最大 1GB |
| 带宽 | 每月 100GB | 每月 100GB |
| 构建时间 | 每次最长 10 分钟 | 每次最长 10 分钟 |

**关键限制：免费账户的私有仓库不能使用 GitHub Pages。** 这意味着如果想用加密功能保护源码，必须升级到 GitHub Pro。

### 流量限制

- **每月 100GB 带宽**：超过后站点会被暂时禁用，下月自动恢复
- **每小时 10 次构建**：频繁推送会触发限制
- **站点大小上限 1GB**：包括所有 HTML/CSS/JS/图片文件

对于个人博客来说，100GB/月的带宽绑绑有余。正常访问量（每天几百到几千 PV）不会触及限制。

### 内容限制

GitHub Pages 明确禁止以下用途：

- **商业交易**：不能用作在线商店、支付页面
- **内容分发网络（CDN）**：不能纯粹作为文件下载服务
- **非法内容**：违反 GitHub 服务条款的内容
- **自动化生成的低质量内容**：SEO 垃圾站

GitHub Pages 的定位是**项目文档和个人博客**，不是通用的 Web 托管服务。

### 技术限制

| 限制项 | 说明 |
|--------|------|
| 仅支持静态内容 | 不能运行服务端代码（PHP、Node.js、Python 等） |
| 无数据库 | 只能用客户端存储（localStorage）或外部 API |
| 无服务端渲染 | 所有内容必须是预构建的 HTML |
| 文件大小 | 单文件建议不超过 50MB |
| 构建超时 | 单次构建最长 10 分钟 |
| Jekyll 限制 | 默认使用 Jekyll 构建，非 Jekyll 项目需要添加 `.nojekyll` 文件 |

### Jekyll 与非 Jekyll

GitHub Pages 默认使用 Jekyll 构建仓库中的文件。如果使用 Hexo、Hugo 等其他静态生成器，需要在仓库根目录添加一个空的 `.nojekyll` 文件，告诉 GitHub 不要用 Jekyll 处理。

Hexo 的 `hexo-deployer-git` 会自动处理这个问题。

## 私有仓库的 Pages 问题

这是用户最常遇到的坑：

```
公开仓库 + GitHub Pages → ✅ 正常工作
私有仓库 + GitHub Pages + 免费账户 → ❌ 404
私有仓库 + GitHub Pages + GitHub Pro → ✅ 正常工作
```

如果你把仓库改成私有后发现站点 404，原因就是免费账户不支持私有仓库的 Pages。

### 解决方案

**方案一：改回公开**

Settings → Danger Zone → Change visibility → Make public

**方案二：升级 GitHub Pro**

$4/月，支持私有仓库的 Pages，同时获得更多功能（更多 Actions 分钟数、更大存储等）。

**方案三：用其他托管服务**

- **Vercel**：免费支持私有仓库部署，自动 HTTPS，全球 CDN
- **Netlify**：类似 Vercel，免费套餐也很慷慨
- **Cloudflare Pages**：免费，带宽无限制

这些服务对私有仓库的静态站点部署都是免费的。

## 与 Hexo 的配合

Hexo + GitHub Pages 是最常见的静态博客组合：

```
hexo clean && hexo generate && hexo deploy
  ↓
public/ 目录推送到 GitHub 仓库
  ↓
GitHub Pages 自动部署
```

### 常见问题

**1. 部署后页面 404**

检查 `_config.yml` 中的 `root` 配置：
- 用户站点（`username.github.io`）：`root: /`
- 项目站点（`username.github.io/blog`）：`root: /blog/`

**2. 样式丢失**

通常也是 `root` 配置错误，导致 CSS 路径不对。

**3. 自定义域名后 404**

仓库中需要有 `CNAME` 文件，内容为你的域名。Hexo 的 `hexo-deployer-git` 会自动创建。

**4. HTTPS 证书未生效**

GitHub 会自动签发证书，但 DNS 传播需要时间，通常等 1-24 小时。

## 总结

GitHub Pages 的核心价值：

- **零成本**：公开仓库免费托管静态站点
- **零运维**：不需要自己管理服务器、CDN、SSL 证书
- **与 GitHub 生态集成**：代码和站点在同一个地方

核心限制：

- **仅支持静态内容**：不能运行后端代码
- **私有仓库需要付费**：免费账户只能用公开仓库的 Pages
- **流量上限 100GB/月**：对个人博客绑绑有余

适用场景：个人博客、项目文档、开源项目主页。不适用：商业网站、需要服务端逻辑的应用、高流量站点。
