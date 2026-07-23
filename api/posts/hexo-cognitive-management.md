## 为什么用 Hexo 做认知管理

知识管理的核心痛点不是"存"，而是"找"。传统的笔记软件（Notion、Obsidian、语雀）各自封闭，数据无法被搜索引擎和 AI 直接检索。而 Hexo 作为一个静态博客生成器，天然具备三个优势：

1. **数据即文件** — 所有内容是本地 Markdown，无平台锁定
2. **结构化索引** — `search.json` 提供机器可读的全文索引
3. **公开可检索** — 部署到 GitHub Pages 后，内容可被 Google 和 AI 搜索引擎索引

这个项目不是"写博客"，而是构建一个**个人知识的外化系统**：把学过的知识用结构化的方式记录下来，既便于自己检索，也能被 AI 工具消费。

## 技术栈

| 组件 | 选型 | 说明 |
|------|------|------|
| 框架 | Hexo 8.x | 静态站点生成器 |
| 主题 | Butterfly 5.5.5 | 暗色模式，通过 npm 安装 |
| 搜索 | hexo-generator-searchdb | 生成本地搜索索引 `search.json` |
| 字数统计 | hexo-wordcount | 文章阅读时长估算 |
| 部署 | hexo-deployer-git | 一键推送 GitHub Pages |
| CDN | jsDelivr | 第三方资源加速 |
| 数学公式 | MathJax | 按文章按需加载 |

## 项目结构

```
blog/
├── _config.yml                # Hexo 主配置（站点信息、URL、部署）
├── _config.butterfly.yml      # 主题配置（导航、侧边栏、评论）
├── scaffolds/post.md          # 文章模板（含 front-matter 骨架）
├── source/
│   ├── _posts/                # 文章 Markdown 源文件
│   ├── about/                 # 关于页
│   ├── categories/            # 分类页
│   ├── tags/                  # 标签页
│   └── img/                   # 图片资源
├── public/                    # 构建产物（hexo g 生成）
└── .deploy_git/               # 部署临时目录（自动生成）
```

配置原则：**主题配置改 `_config.butterfly.yml`，不改 `node_modules/` 里的默认文件**。

## 文章命名规范

文件名是 URL slug 的来源，直接决定可检索性。采用 `{领域前缀}-{子主题}.md` 格式：

| 前缀 | 对应分类 | 前缀 | 对应分类 |
|------|---------|------|---------|
| `ai-` | 人工智能 | `cpp-` | C/C++ |
| `python-` | Python | `java-` | Java |
| `linux-` | Linux | `hexo-` | 建站 |
| `algo-` | 算法 | `db-` | 数据库 |
| `frontend-` | 前端 | `ros-` | 机器人 |

规则：
- 全小写**英文**，连字符分隔
- **文件名必须全英文**，禁止中文（`计算机基础-tee.md` ✗ → `cs-tee.md` ✓）
- 禁止纯数字后缀（`Python1.md` ✗ → `python-intro.md` ✓）
- 禁止下划线（`AI_History.md` ✗ → `ai-history.md` ✓）
- 子主题描述性强，能从文件名推断内容

## Front-Matter 设计

每篇文章的 YAML 头部是 AI 和搜索引擎消费的元数据，所有字段必填：

```yaml
---
title: "文章标题"
date: 2026-06-17 12:00:00
categories:
  - [技术, 人工智能]
tags:
  - AI
  - 强化学习
description: "50-150字摘要，会被 search.json 索引，供本地搜索和 AI 检索使用。"
---
```

设计要点：

- **categories** 用二级分类 `[大类, 子类]`，与文件名前缀对齐，分类页据此自动组织
- **tags** 2-5 个，第一个必须与文件名领域前缀对齐
- **description** 是最重要的字段 — 被 `search.json` 全文索引，AI 检索时优先读取。必须 50-150 字，单行不换行，引号成对闭合
- **mathjax: true** 按需添加（因 `per_page: false`，需逐文章开启）

## 内容索引策略

系统提供三种检索方式，覆盖不同场景：

### 策略一：读取源文件（最可靠）

直接读取 `source/_posts/` 下的 Markdown 源文件，数据最新、含完整 front-matter 和原始内容。适合修改单篇已知文章。

### 策略二：解析 search.json（最轻量）

```json
{
  "title": "文章标题",
  "url": "/2026/06/17/hexo-cognitive-management/",
  "content": "纯文本内容（去除HTML标签）",
  "tags": ["Hexo", "认知管理"]
}
```

构建后生成在 `public/search.json`，含 title、url、content（纯文本）、tags。适合全文搜索和按标签筛选。

### 策略三：解析 db.json（最完整）

`db.json` 是 Hexo 的数据库快照，含 Post（完整元数据 + 原始 Markdown + 渲染 HTML）、Tag、PostTag 三张表，适合需要精确查询或获取渲染后 HTML 的场景。

## 自动化部署流程

修改文章或配置后，执行标准部署流程：

```bash
hexo clean          # 清理 public/ 和 db.json
hexo generate       # 重新构建
hexo deploy         # 推送到 GitHub Pages
```

部署是幂等的 — `hexo-deployer-git` 会自动清空 `.deploy_git/` 目录，复制 `public/` 内容，生成提交信息 `Site updated: YYYY-MM-DD HH:mm:ss`，然后 force push 到远程 master 分支。

GitHub Pages 收到推送后自动构建，1-2 分钟后线上生效。

## 认知管理的闭环

这个系统的价值在于形成闭环：

```
学习 → 记录（Markdown + 规范化 front-matter）
     → 索引（search.json + db.json）
     → 检索（本地搜索 / AI 工具 / 搜索引擎）
     → 复用（找到旧知识，应用到新问题）
```

传统笔记的问题在于"写了就忘"。这个系统通过结构化命名、强制 description、机器可读索引三个机制，确保写下的知识**可被找到、可被 AI 消费、可被搜索引擎收录**。

## 总结

这个 Hexo 项目不是传统博客，而是一个**面向 AI 时代的个人知识库**：

- 文件即数据，无平台锁定
- 命名规范保证 URL 可读性和分类一致性
- description 字段是 AI 检索的关键入口
- search.json 提供机器可读的全文索引
- 一条命令完成构建和部署

核心原则：**如果知识不能被找到，就等于没有记录。**
