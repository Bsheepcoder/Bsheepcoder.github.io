# AI Context — Q's blog

> 本文件供 AI/LLM 了解本站写作规范与分类体系，配合 /llms.txt 使用。

## 站点定位

Bsheepcoder 的技术博客（zh-CN），记录 AI、编程、计算机科学学习笔记。重实践，含完整可运行代码。

## 分类体系（二级分类，与文件名前缀对齐）

| 文件名前缀 | 二级分类 |
|------------|---------|
| `ai-` | [技术, 人工智能] |
| `python-` | [技术, Python] |
| `cpp-` | [技术, C/C++] |
| `java-` | [技术, Java] |
| `linux-` | [技术, Linux] |
| `hexo-` | [技术, 建站] |
| `algo-` | [技术, 算法] |
| `db-` | [技术, 数据库] |
| `frontend-` | [技术, 前端] |
| `cs-` | [技术, 计算机基础] |
| `ros-` | [技术, 机器人] |
| `slam-` | [技术, SLAM] |
| `rl-` | [技术, 人工智能] |
| `tcp-ip-` | [技术, TCP-IP] |
| `info-` | [技术, 信息获取] |
| `skill-` | [技术, Skill] |
| `prompt-` | [技术, 提示词] |
| `reading-` | [读书] |
| `embedded-` | [技术, 嵌入式] |

## 文章命名规范

`{领域}-{子主题}.md`，全小写**英文**连字符，如 `ai-history.md`、`python-numpy.md`、`cs-tee-principles.md`。**文件名必须全英文**，不用纯数字后缀，不用下划线。

## Front-Matter 必填字段

- `title`：文章标题
- `date`：创建时间
- `categories`：二级分类 `[大类, 子类]`
- `tags`：2-5 个，第一个与文件名前缀对齐
- `description`：50-150 字摘要，供搜索索引和 AI 检索

## 加密策略

有 `password` 字段的文章（hexo-blog-encrypt），API 中 `encrypted: true`、`content` 为空字符串，不泄露正文。

## 提示词文章规范

`prompt-` 前缀文章是可复用的结构化文本，用 `<prompt>...</prompt>` 包裹整段提示词，内部用 XML 标签组织（`<role>`/`<context>`/`<instructions>`/`<output_format>`/`<stop_rules>` 等，按需选择不强制全套）。AI 提取流程：

1. `GET /api/index.json` → 筛选 categories 含 "提示词" 的文章
2. `GET /api/posts/<slug>.md` → 提取 `<prompt>...</prompt>` 之间的内容
3. 根据 "变量" 表替换动态部分 → 直接使用

## API 端点

| 端点 | 用途 |
|------|------|
| `/llms.txt` | AI 入口（站点概览 + 文章索引 + 本文件指针） |
| `/llms-full.txt` | 全部文章全文合并（可直接一次读取） |
| `/api/index.json` | 结构化文章列表（含元数据和 API 地址） |
| `/api/posts/<slug>.json` | 单篇 JSON（元数据 + Markdown 正文） |
| `/api/posts/<slug>.md` | 单篇纯 Markdown 正文（零噪音） |
| `/api/categories/<slug>.json` | 按分类聚合的文章列表 |
| `/api/categories/<slug>.md` | 按分类聚合的全文合并 |
| `/ai-context.md` | 本文件（站点写作规范与分类体系） |

## AI 获取文章的标准流程

1. `GET /llms.txt` → 站点概览 + 文章索引
2. `GET /ai-context.md` → 了解写作规范与分类体系
3. `GET /llms-full.txt` → 一次性获取全部文章全文（当前规模推荐）
   或 `GET /api/posts/<slug>.md` → 获取单篇纯 Markdown
