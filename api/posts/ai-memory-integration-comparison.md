## 元认知：Hexo 系统的根本矛盾

你的 Hexo 认知系统在"知识归档和 AI 可读性"这条线上已经做到了同类方案的天花板——PDC 协议、llms.txt 标准、结构化 API、加密保护、MCP 适配器，这套组合在静态博客领域几乎没有对手。

但它本质上是一个**出版系统**，不是**记忆系统**。

### 出版系统 vs 记忆系统

| 维度 | 出版系统（Hexo PDC） | 记忆系统（Mem0/EverOS） |
|------|---------------------|------------------------|
| **核心问题** | 知识怎么发布 | 经验怎么积累 |
| **写入方式** | 人工写 Markdown | 自动从对话提取 |
| **检索方式** | 全文搜索（grep/FTS） | 向量语义检索 |
| **遗忘机制** | 无（文章永久存在） | 有（ADD-only + 蒸馏） |
| **可审计性** | ✅ Markdown diff | ❌ 二进制文件（Mem0）/ ✅ Markdown diff（EverOS） |
| **适用场景** | 公开分享、SEO | Agent 记忆、经验积累 |

**关键洞察**：你不需要"选一个"，而是"把三个拼起来"。

### 为什么需要记忆系统

你在 `ai-memory-driven-development.md` 里已经写得很清楚：

> "所有当前开发范式的共同天花板：AI 的能力边界不在模型，在**上下文连续性**。"

Hexo PDC 解决了"AI 怎么读内容"，但没有解决"经验怎么积累"。每次对话从零开始，踩过的坑不会自动沉淀，高频模式不会自动提取为 skill。

这就是记忆系统的价值：**让 AI 从做过的事中变聪明**。

---

## 搭积木：三种集成方案对比

### 方案一：Hexo + Mem0

**架构**：

```
Hexo PDC（出版层）+ Mem0（经验层）
├── 人工写文章 → Hexo 构建 → /api/posts/*.md（公开分享）
├── 自动提取记忆 → Mem0 向量库（Agent 检索）
└── MCP 协议连接 → Agent 同时访问两者
```

**核心数据**：
- 60.2k stars，Y Combinator S24
- 2026年4月新算法：LoCoMo 92.5，LongMemEval 94.4
- token 效率：7k tokens vs 全上下文 25k+ tokens
- 20+ 框架集成（LangChain、CrewAI、AutoGen、OpenAI Agents SDK 等）

**Mem0 的核心能力**：

| 能力 | 说明 |
|------|------|
| **自动记忆提取** | 一次 LLM 调用，提取事实、决策、偏好 |
| **多信号融合检索** | 语义相似度 + BM25 关键词 + 实体链接，三者并行评分 |
| **时间感知推理** | 记忆带时间戳，支持"当前状态""过去事件""未来计划"查询 |
| **MCP 原生支持** | 一行命令接入任何 MCP 客户端 |
| **Agent Skills** | `npx skills add` 直接接入 opencode |

**快速开始（5 分钟）**：

```bash
# 安装
pip install mem0ai

# 设置 API Key
export OPENAI_API_KEY="your-key"

# 添加记忆
python -c "
from mem0 import Memory
m = Memory()
m.add([{'role':'user','content':'我叫张三，喜欢爬山'}], user_id='zhangsan')
print(m.search('我喜欢什么运动', filters={'user_id':'zhangsan'}))
"
```

**接入 opencode**：

```bash
npx skills add https://github.com/mem0ai/mem0 --skill mem0
```

**接入 MCP 客户端**：

```bash
npx mcp-add --name mem0-mcp --type http --url "https://mcp.mem0.ai/mcp" --clients "opencode"
```

**优势**：
- 生态最成熟（60.2k stars，20+ 集成）
- MCP 原生支持，一行命令接入
- 检索精度最高（LoCoMo 92.5）
- Agent Skills 支持
- 三种运行模式：库模式、自托管服务器、云平台

**劣势**：
- 依赖 LLM（默认 OpenAI，可换 Ollama）
- 向量数据库二进制文件无法 diff
- 记忆污染问题（ADD-only，无内置遗忘）
- 云端默认，需要自托管

**适合谁**：需要生产环境稳定性、不介意依赖 LLM、注重检索精度

---

### 方案二：Hexo + EverOS

**架构**：

```
Hexo PDC（出版层）+ EverOS（记忆层）
├── 人工写文章 → Hexo 构建 → /api/posts/*.md（公开分享）
├── 自动提取记忆 → .md 文件 + SQLite/LanceDB（本地优先）
└── 直接编辑 .md 文件 → 记忆可 diff、可 Git 版本控制
```

**核心数据**：
- 10.4k stars
- Markdown-native，本地优先
- 用户拥有数据，可直接编辑 .md 文件

**EverOS 的核心能力**：

| 能力 | 说明 |
|------|------|
| **Markdown 源文件** | 记忆存储为 .md 文件，可读、可编辑、可 diff |
| **本地三件套** | Markdown + SQLite + LanceDB，无需 MongoDB/Elasticsearch |
| **用户 + Agent 双轨** | 用户 `episodes/profile` 和 Agent `cases/skills` 分离 |
| **正交检索** | 按 `user_id`、`agent_id`、`app_id`、`project_id`、`session_id` 检索 |
| **知识 Wiki** | 可编辑的 Markdown 知识页面，带分类和 CRUD API |
| **反思机制** | 离线记忆演化，合并 episode 聚类，优化 profile 和 skills |

**快速开始（5 分钟）**：

```bash
# 安装
pip install everos

# 初始化配置
everos init

# 启动服务器
everos server start

# 添加记忆
curl -X POST http://127.0.0.1:8000/api/v1/memory/add \
  -H 'Content-Type: application/json' \
  -d '{
    "session_id": "demo-001",
    "app_id": "default",
    "project_id": "default",
    "messages": [
      {"sender_id": "alice", "role": "user", "content": "我叫张三，喜欢爬山"}
    ]
  }'

# 搜索记忆
curl -X POST http://127.0.0.1:8000/api/v1/memory/search \
  -H 'Content-Type: application/json' \
  -d '{
    "user_id": "alice",
    "query": "我喜欢什么运动",
    "top_k": 5
  }'
```

**优势**：
- Markdown-native，可 diff、可版本控制
- 本地优先，用户拥有数据
- 记忆演化和反思机制
- 直接编辑 .md 文件管理记忆
- 不依赖云端服务

**劣势**：
- 项目较新（10.4k stars），生态不如 Mem0 成熟
- 需要 OpenRouter + DeepInfra API Key
- 检索精度未公开 benchmark
- 记忆演化机制的效果待验证

**适合谁**：注重数据所有权、需要可审计性、想直接编辑 .md 文件

---

### 方案三：Hexo + sqlite-vec（自建）

**架构**：

```
Hexo PDC（出版层）+ SQLite + sqlite-vec（经验层）
├── 人工写文章 → Hexo 构建 → /api/posts/*.md（公开分享）
├── 脚本读取 .md → 存入 SQLite + sqlite-vec（本地向量检索）
└── opencode 自定义工具 → Agent 检索记忆
```

**核心数据**：
- sqlite-vec 7.8k stars，pre-v1
- SQLite 向量扩展，支持 L2/L1/cosine 距离
- <1 万条记忆查询几十 ms

**sqlite-vec 的核心能力**：

| 能力 | 说明 |
|------|------|
| **向量检索** | 支持 8192 维，float32/int8/bit 类型 |
| **混合查询** | 一条 SQL 同时做向量检索 + FTS5 全文搜索 + 元数据过滤 |
| **零部署** | 一个 .db 文件，无需外部服务 |
| **跨平台** | Windows/Linux/macOS/WASM/Android/iOS |

**你已经在 `ai-sqlite-memory-store.md` 里写好了架构**：

```typescript
// .opencode/tools/memory-query.ts
import { tool } from "@opencode-ai/plugin"
import { Database } from "bun:sqlite"
import sqliteVec from "sqlite-vec"

const db = new Database("memory.db")
db.enableLoadExtension(true)
sqliteVec.load(db)

export default tool({
  description: "检索项目记忆库中的历史经验。",
  args: {
    query: tool.schema.string().describe("自然语言查询"),
    limit: tool.schema.number().optional().describe("返回条数，默认 5"),
  },
  async execute(args) {
    const embedding = new Float32Array(await embed(args.query))
    const results = db.query(`
      SELECT m.id, m.content, m.type, m.created_at, v.distance
      FROM memories m
      JOIN memories_vec v ON m.id = v.id
      WHERE v.embedding MATCH ? AND k = ?
        AND m.project = ?
      ORDER BY v.distance
    `).all(embedding, args.limit ?? 5, process.cwd())
    return JSON.stringify(results, null, 2)
  },
})
```

**表结构**：

```sql
-- 记忆主表
CREATE TABLE memories (
  id INTEGER PRIMARY KEY,
  content TEXT NOT NULL,
  type TEXT NOT NULL,
  project TEXT NOT NULL,
  tags TEXT,
  source TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  embedding BLOB
);

-- 向量索引
CREATE VIRTUAL TABLE memories_vec USING vec0(
  id INTEGER PRIMARY KEY,
  embedding FLOAT[768] distance_metric=cosine
);

-- 全文索引
CREATE VIRTUAL TABLE memories_fts USING fts5(
  content, type, project, tags
);
```

**优势**：
- 零部署成本（一个 .db 文件）
- 向量语义 + FTS5 + 元数据混合查询
- 你已经在文章里写好了架构
- 完全可控，不依赖外部服务

**劣势**：
- 需要写代码（但你已经写好了）
- sqlite-vec 还是 pre-v1，API 可能变
- 没有自动写入机制
- 没有遗忘机制

**适合谁**：想完全掌控代码、轻量级需求、愿意自己实现

---

## 案例即原理：具体集成效果

### 场景一：Agent 检索历史经验

**问题**：Agent 遇到一个 bug，想查"上次 PostgreSQL 连接池耗尽怎么解的"。

| 方案 | 检索方式 | 精度 | 延迟 | 示例 |
|------|---------|------|------|------|
| **Hexo PDC** | 全文搜索（grep） | 低（关键词匹配） | 快（<10ms） | `grep "连接池" source/_posts/*.md` |
| **Hexo + Mem0** | 向量语义 + BM25 + 实体匹配 | 高（LoCoMo 92.5） | 中（~100ms） | `m.search("PostgreSQL 连接池耗尽")` |
| **Hexo + EverOS** | 向量语义 + SQLite 全文搜索 | 中 | 中（~100ms） | `POST /api/v1/memory/search` |
| **Hexo + sqlite-vec** | 向量语义 + FTS5 + 元数据过滤 | 中 | 快（<50ms） | `SELECT * FROM memories WHERE embedding MATCH ?` |

**关键差异**：
- **Hexo PDC**：只能做关键词匹配，找不到"连接池耗尽"和"数据库连接数不够"是同一个问题
- **Mem0**：语义理解"连接池耗尽"和"数据库连接数不够"是同一个问题，还能匹配实体"PostgreSQL"
- **EverOS**：语义检索 + SQLite 全文搜索，但精度未公开 benchmark
- **sqlite-vec**：语义检索 + FTS5，但需要自己实现嵌入计算

---

### 场景二：记忆自动写入

**问题**：Agent 解决了一个 bug，想自动记录解法。

| 方案 | 写入方式 | 摩擦成本 | 示例 |
|------|---------|---------|------|
| **Hexo PDC** | 手动写文章 | 高（写 Markdown + 配置 front-matter + 部署） | `hexo new "bug-fix"` → 写文章 → `hexo deploy` |
| **Hexo + Mem0** | 自动从对话提取 | 低（一次 LLM 调用） | `m.add(messages, user_id="zhangsan")` |
| **Hexo + EverOS** | 自动从对话提取 | 低（一次 LLM 调用） | `POST /api/v1/memory/add` |
| **Hexo + sqlite-vec** | 手动写入 | 中（调用 API） | `INSERT INTO memories (content, embedding) VALUES (?, ?)` |

**关键差异**：
- **Hexo PDC**：摩擦成本最高，需要人工写文章、配置 front-matter、部署
- **Mem0/EverOS**：摩擦成本最低，自动从对话中提取事实
- **sqlite-vec**：需要手动调用 API，但比写文章简单

---

### 场景三：记忆可审计性

**问题**：团队需要 review 新增的记忆。

| 方案 | 可 diff | 可版本控制 | 可 PR review |
|------|---------|-----------|-------------|
| **Hexo PDC** | ✅ | ✅ | ✅ |
| **Hexo + Mem0** | ❌ | ❌ | ❌ |
| **Hexo + EverOS** | ✅ | ✅ | ✅ |
| **Hexo + sqlite-vec** | ❌ | ❌ | ❌ |

**关键差异**：
- **Hexo PDC/EverOS**：Markdown 文件可以 diff、可以 PR review
- **Mem0/sqlite-vec**：二进制 .db 文件无法有意义地 diff

**这就是 EverOS 的杀手锏**：如果你需要团队协作和可审计性，EverOS 是唯一选择。

---

### 场景四：记忆遗忘

**问题**：过时的记忆需要清理。

| 方案 | 遗忘机制 | 实现方式 |
|------|---------|---------|
| **Hexo PDC** | 无 | 文章永久存在 |
| **Hexo + Mem0** | 无内置 | ADD-only，需要手动 `delete_memory` |
| **Hexo + EverOS** | 有 | 反思机制，离线记忆演化 |
| **Hexo + sqlite-vec** | 无 | 需要手动实现 |

**关键差异**：
- **Mem0**：ADD-only 策略，不做 UPDATE/DELETE，长期积累垃圾
- **EverOS**：有反思机制，可以合并 episode 聚类、优化 profile 和 skills
- **sqlite-vec**：需要自己实现遗忘逻辑

---

## 缺陷与批判：每种方案的边界

### Hexo + Mem0 的边界

**边界一：依赖 LLM**
- 默认需要 OpenAI API Key
- 可换 Ollama（本地模型），但质量较低
- 嵌入模型影响检索质量

**边界二：记忆污染**
- ADD-only 策略的代价：长期积累垃圾
- 没有内置的遗忘机制
- 需要定期人工 review

**边界三：可审计性**
- 向量数据库二进制文件无法 diff
- 团队协作时无法 PR review 记忆

**边界四：成本**
- 每次记忆写入需要 LLM 调用
- 每次检索需要嵌入计算
- 云平台按量计费

---

### Hexo + EverOS 的边界

**边界一：生态成熟度**
- 项目较新（10.4k stars vs Mem0 60.2k stars）
- 集成不如 Mem0 丰富
- 社区和文档不如 Mem0 完善

**边界二：API Key 依赖**
- 需要 OpenRouter + DeepInfra API Key
- 不像 Mem0 可以直接用 OpenAI

**边界三：检索精度**
- 未公开 LoCoMo/LongMemEval benchmark
- 精度是否能达到 Mem0 的 92.5 未知

**边界四：记忆演化效果**
- 反思机制的效果待验证
- 什么情况下该合并、什么情况下该保留，规则不明确

---

### Hexo + sqlite-vec 的边界

**边界一：需要写代码**
- 需要实现嵌入计算、记忆写入、遗忘逻辑
- 你已经在文章里写好了架构，但仍需实现

**边界二：sqlite-vec 成熟度**
- 还是 pre-v1，API 可能变
- 默认暴力线性扫描，>1 万条变慢

**边界三：没有自动写入**
- 需要手动调用 API
- 摩擦成本比 Mem0/EverOS 高

**边界四：没有遗忘机制**
- 需要自己实现
- 什么该忘、什么时候忘、忘了之后如何恢复，这些问题需要自己解决

---

## 更多可能性：混合架构

### 方案四：Hexo + Mem0 + sqlite-vec（分层架构）

```
Hexo PDC（出版层）
├── 人工写文章 → /api/posts/*.md（公开分享）
└── 每篇文章自动导入 Mem0（经验层）

Mem0（经验层）
├── 自动从对话提取记忆
├── 向量语义检索
└── 高频模式蒸馏为 skill

sqlite-vec（索引层）
├── 索引 Hexo 文章（57 篇）
├── 提供本地向量检索
└── 离线场景备用
```

**优势**：
- 出版层（Hexo）和经验层（Mem0）分离
- 本地索引（sqlite-vec）和云端记忆（Mem0）互补
- 离线场景可用 sqlite-vec，在线场景用 Mem0

**劣势**：
- 复杂度高
- 需要维护三个系统

---

### 方案五：Hexo + EverOS + MCP（可审计架构）

```
Hexo PDC（出版层）
├── 人工写文章 → /api/posts/*.md（公开分享）
└── 每篇文章自动导入 EverOS（记忆层）

EverOS（记忆层）
├── 自动从对话提取记忆
├── Markdown 文件可 diff、可 PR review
├── 反思机制自动演化
└── MCP server 暴露给 Agent

Agent
├── 通过 MCP 访问 EverOS 记忆
├── 通过 PDC API 访问 Hexo 文章
└── 两者互补
```

**优势**：
- 可审计性最好（Markdown diff）
- 记忆演化（反思机制）
- MCP 原生支持

**劣势**：
- EverOS 生态不如 Mem0 成熟
- 需要 OpenRouter + DeepInfra API Key

---

## 总结：选择框架

### 决策树

```
你需要自动记忆写入吗？
├── 否 → 继续用 Hexo PDC（出版系统就够了）
└── 是 → 你需要可 diff、可版本控制吗？
    ├── 是 → Hexo + EverOS
    └── 否 → 你需要生产环境稳定性吗？
        ├── 是 → Hexo + Mem0
        └── 否 → Hexo + sqlite-vec（自建）
```

### 我的建议

**短期（本周）**：Hexo + Mem0
- MCP 原生支持，一行命令接入
- Agent Skills 支持，`npx skills add` 直接接入 opencode
- 生态最成熟，20+ 框架集成

**中期（本月）**：评估 EverOS
- 如果需要可审计性（团队协作、PR review）
- 如果需要记忆演化（反思机制）
- 如果需要本地优先（数据所有权）

**长期（下季度）**：根据实际效果选择
- 如果 Mem0 的记忆污染问题严重，考虑 EverOS
- 如果 EverOS 的检索精度不够，继续用 Mem0
- 如果需要离线场景，加 sqlite-vec 做本地索引

### 本质问题

**Mem0 解决的是"经验怎么积累"**——自动从对话中提取、语义检索、遗忘过时。
**EverOS 解决的是"记忆怎么存储"**——Markdown 可读、可 diff、可 Git 版本控制。
**你的 Hexo PDC 解决的是"知识怎么发布"**——结构化、SEO 友好、AI 可读。

三者不是竞争关系，是互补关系。你需要的不是"选一个"，而是"把三个拼起来"。

> 延伸阅读：[AI 时代开发范式演化：从 Waterfall 到 Memory-Driven 的终局](/2026/07/01/ai-memory-driven-development/)（MDD 六阶段闭环，记忆资产分层）、[SQLite 做 AI 开发记忆库：工程检验下的最佳实践](/2026/07/01/ai-sqlite-memory-store/)（分层记忆架构）、[MCP 是伪需求吗？从 AI Native 本质砍掉 80% 无效场景](/2026/07/06/mcp-pseudo-demand/)（MCP 的真与伪）

---

*参考资料：*
- *Mem0, "The Memory Layer for Personalized AI", https://github.com/mem0ai/mem0, 60.2k stars*
- *EverMind AI, "EverOS: One portable memory layer for every AI agent", https://github.com/EverMind-AI/EverOS, 10.4k stars*
- *sqlite-vec, "SQLite vector search extension", https://github.com/asg017/sqlite-vec, 7.8k stars*
- *Mem0 Research, "Token-Efficient Memory Algorithm", https://mem0.ai/research, LoCoMo 92.5*
- *MCP Servers, "Model Context Protocol Servers", https://github.com/modelcontextprotocol/servers, 88.1k stars*
