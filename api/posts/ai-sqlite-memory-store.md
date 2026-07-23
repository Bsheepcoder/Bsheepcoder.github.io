## 一、问题：AI 开发的记忆该用什么存

2026 年，几乎所有 AI 编码工具都有某种形式的"记忆"——Aider 有 `.aider.chat.history.md`，Claude Code 有 `CLAUDE.md` + auto memory（`MEMORY.md`），opencode 有 `AGENTS.md` + skills。但它们用的都是 Markdown 文件。

同时，[记忆驱动开发（MDD）](/2026/07/01/ai-memory-driven-development/)的六阶段闭环里，"阶段 4 维护——先检索再修复"需要的是：**从过去 N 次踩坑记录中，快速找到和当前 bug 相关的那一条**。当记忆条目从 10 条增长到 1000 条时，Markdown 文件的 `Ctrl+F` 还够用吗？

这就是本文要回答的核心问题：**SQLite 做开发记忆库，有价值吗？如果有，怎么接入？对比 Markdown 式记忆，优势在哪？**

## 二、现状盘点：谁在用什么存记忆

### Markdown 派：人可读优先

| 工具 | 记忆载体 | 格式 | 检索方式 |
|------|---------|------|---------|
| Aider | `.aider.chat.history.md` + `.aider.input.history` | Markdown / 纯文本 | 人眼读、grep |
| Claude Code | `CLAUDE.md` + auto memory（`MEMORY.md` + 主题 .md） | Markdown | 启动时自动注入前 200 行/25KB |
| Cline / Roo Code | `.clinerules` / `.roomodes` | Markdown | 启动注入 |
| opencode | `AGENTS.md` + `.opencode/skills/*/SKILL.md` | Markdown | AGENTS.md 全量注入，skills 按需懒加载 |

Markdown 派的共同优势：**可 diff、可版本控制、人可直读**。你能在 git log 里看到记忆的演化历史，能在 PR 里 review 一条新规则。这是 Markdown 的杀手锏。

但它的局限同样明显：**没有结构化检索能力**。当 `lessons/` 目录下有 500 条踩坑记录时，找"上次 PostgreSQL 连接池耗尽怎么解的"，只能靠文件名约定 + 全文搜索。没有语义匹配，没有元数据过滤，没有关联查询。

### SQLite 派：机器可检索优先

| 工具/框架 | SQLite 用途 | 证据 |
|-----------|------------|------|
| LangGraph | `SqliteSaver` checkpoint（官方包 `langgraph-checkpoint-sqlite` v3.1.0，2026-05 更新） | PyPI 官方包，存对话状态/中断点 |
| Continue.dev | 本地持久化（`package.json` 依赖 `sqlite` + `sqlite3`） | 代码库已核实 |
| ChromaDB | 默认后端的元数据层（向量索引用 HNSW，元数据存 SQLite） | 28.6k stars，默认本地模式 |

注意一个细节：**ChromaDB 用 SQLite 存元数据，但向量索引不用 SQLite 存**——它用专用的 HNSW 索引。这说明 SQLite 在向量检索场景不是万能的，它的价值在**结构化数据的存储和查询**，不在向量索引本身。

### Mem0：都不用 SQLite

Mem0（最知名的 Agent 记忆框架）默认用 Qdrant 做向量存储，自托管用 pgvector（PostgreSQL）。`mem0/vector_stores/` 目录下有 qdrant/chroma/pgvector/faiss/milvus/pinecone 等 26 个 provider，**没有 `sqlite.py`**。

这能说明 SQLite 不适合做记忆吗？不能。Mem0 的场景是 SaaS 级 Agent 记忆——高并发、多租户、百万级记忆条目。而个人开发者的记忆库通常 <1 万条，SQLite 的零部署优势在这个量级远大于它的并发短板。

## 三、SQLite + 向量检索：技术栈成熟度

### sqlite-vec：可用的轻量方案

`sqlite-vec`（`asg017/sqlite-vec`，7.8k stars）是当前最活跃的 SQLite 向量扩展。核心事实：

| 维度 | 数据 |
|------|------|
| 版本 | v0.1.9 稳定（2026-03），v0.1.10-alpha 开发中 |
| 维度上限 | 8192 维（覆盖 OpenAI text-embedding-3-large 的 3072 维） |
| 距离度量 | L2（默认）、L1、cosine |
| 向量类型 | float32 / int8 / bit（支持量化） |
| 元数据过滤 | KNN 查询中支持 `=/!=/>/</<=` |
| 安装 | `pip install sqlite-vec` / `npm install sqlite-vec` |
| 平台 | Windows/Linux/macOS/WASM/Android/iOS |

关键限制：**默认暴力线性扫描，不是 ANN（近似最近邻）**。没有 HNSW 或 IVF 索引，ANN 算法在仓库里还是实验性 `.c` 文件，未进稳定版。这意味着：

- <1 万条：查询几十 ms，够用
- 1-10 万条：可接受但开始变慢
- >10 万条：明显吃力

前代 `sqlite-vss` 基于 Faiss（C++），安装麻烦，已于 2024 年停止维护，**不要用**。

### DuckDB vss：要 ANN 性能的替代

如果数据量超过 10 万条或需要毫秒级延迟，DuckDB 的 `vss` 扩展（`duckdb/duckdb-vss`，260 stars）提供真正的 HNSW 索引。但 DuckDB 是 OLAP 引擎，比 SQLite 重得多，不适合 CLI 工具的零部署场景。

### SQLite 独有的杀手锏：混合查询

这是 SQLite 记忆库最被低估的优势。一条 SQL 同时做向量检索 + 全文搜索 + 元数据过滤：

```sql
-- 找"和当前 bug 语义相似 + 包含'连接池'关键词 + 属于数据库分类"的历史经验
SELECT id, content, distance
FROM memories
WHERE embedding MATCH :current_bug_embedding
  AND k = 10
  AND content MATCH '连接池 池耗尽 timeout'
  AND category = 'database'
  AND project = 'abr'
ORDER BY distance;
```

这段查询用到了 `sqlite-vec`（向量 MATCH）+ SQLite FTS5（全文 MATCH）+ 原生 SQL（元数据过滤）。三者同一查询内 JOIN，不需要跨系统编排。Markdown 做不到，Mem0 + Qdrant 要跨两个系统才能实现。

## 四、opencode 如何接入 SQLite 记忆库

opencode 当前没有内置记忆数据库——它的"记忆"完全靠文件驱动（`AGENTS.md` 全量注入 + skills 按需懒加载）。但它提供了三个接入点，足以实现 SQLite 记忆库。

### 接入点一：自定义工具（Agent 主动查询）

`.opencode/tools/` 下定义工具，Agent 按需调用。文件名即工具名：

```typescript
// .opencode/tools/memory-query.ts
import { tool } from "@opencode-ai/plugin"
import { Database } from "bun:sqlite"
import sqliteVec from "sqlite-vec"

const db = new Database("memory.db")
db.enableLoadExtension(true)
sqliteVec.load(db)

export default tool({
  description: "检索项目记忆库中的历史经验。当遇到 bug、做技术决策、或需要参考历史方案时使用。",
  args: {
    query: tool.schema.string().describe("自然语言查询，如'PostgreSQL 连接池耗尽'"),
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

Agent 在开发过程中遇到 bug 时，会自动调用 `memory-query` 工具检索历史经验——不需要人手动翻 `lessons/` 目录。

### 接入点二：Plugin hooks（自动写入记忆）

`.opencode/plugins/memory-sink.ts` 在会话空闲时自动提取记忆：

```typescript
// .opencode/plugins/memory-sink.ts
export default async ({ client }) => ({
  "session.idle": async (input, output) => {
    // 会话空闲时，提取本次对话中的经验
    const summary = await client.session.summarize(input.sessionID)
    const embedding = await embed(summary)
    
    db.query(`INSERT INTO memories (content, embedding, type, project) VALUES (?, ?, ?, ?)`)
      .run(summary, embedding, "experience", process.cwd())
  },
  "tool.execute.after": async (input, output) => {
    // 工具执行后，如果是修 bug，记录解法
    if (input.tool === "bash" && output.result.includes("fix")) {
      const embedding = await embed(output.result)
      db.query(`INSERT INTO memories (content, embedding, type) VALUES (?, ?, ?)`)
        .run(output.result, embedding, "bugfix")
    }
  },
})
```

`session.idle` 是最接近"对话后自动写入记忆"的 hook；`tool.execute.after` 可以在工具执行后做后处理。两者配合实现记忆的自动积累。

### 接入点三：MCP server（跨工具共享记忆）

如果同时用 opencode、Claude Code、Cursor，记忆不该锁在单一工具里。把 SQLite 记忆库包装成 MCP server，所有工具通过统一协议访问：

```jsonc
// opencode.json
{
  "mcp": {
    "memory": {
      "type": "local",
      "command": ["npx", "-y", "sqlite-memory-server"],
      "environment": { "DB_PATH": ".memory/memory.db" }
    }
  }
}
```

MCP 的价值在[MCP 协议详解](/2026/06/22/ai-mcp-protocol/)中讲过：M+N 线性解法。记忆库作为 MCP server 后，换 AI 工具不丢记忆——这正是[开源 Agent 架构拆解](/2026/06/26/ai-agent-2026-landscape/)说的"框架可换，记忆不可换"。

### 三种接入的取舍

| 接入方式 | 触发 | 适合场景 | 复杂度 |
|---------|------|---------|--------|
| 自定义工具 | Agent 主动调用 | 按需检索历史经验 | 低，单文件 |
| Plugin hooks | 事件驱动自动触发 | 自动积累记忆（无需手动记） | 中，需处理摘要/嵌入 |
| MCP server | 多工具共享 | 跨 opencode/Claude Code/Cursor | 高，需独立服务 |

个人开发者建议从自定义工具起步——一个 `.opencode/tools/memory-query.ts` 就能让 Agent 查历史经验。积累一段时间后，再加 Plugin hooks 实现自动写入。多工具协作时才上 MCP server。

## 五、对比：SQLite 记忆 vs Markdown 记忆

### 逐维度对比

| 维度 | Markdown 文件 | SQLite + sqlite-vec |
|------|-------------|---------------------|
| **可读性** | 人直读，git diff 友好 | 需 SQL 查询，不可直读 |
| **版本控制** | 原生 git diff，PR review | 二进制 .db 文件，diff 无意义 |
| **检索精度** | 全文搜索（grep/FTS） | 向量语义 + FTS5 + 元数据过滤混合 |
| **检索速度** | O(n) 扫描，<100 条够用 | <1 万条几十 ms，>10 万条吃力 |
| **结构化查询** | 不可能 | SQL 原生支持（JOIN/过滤/聚合） |
| **部署成本** | 零，纯文本 | 低，一个 .db 文件 + 一个扩展 |
| **可移植性** | 任何编辑器 | 需要 sqlite-vec 扩展 |
| **记忆量级** | <100 条舒适 | <1 万条舒适 |
| **协作友好** | PR review 天然支持 | 需要额外机制（导出/同步） |

### 各自的不可替代优势

**Markdown 的不可替代优势**：**可 diff**。一条记忆从"Proposed"改成"Accepted"，一个 git commit 就说清了。团队协作时，新规则要 PR review——Markdown 天然支持，SQLite 做不到（二进制文件无法有意义地 diff）。

**SQLite 的不可替代优势**：**结构化检索**。找"所有数据库分类的 bugfix，按时间倒序，关联到 PostgreSQL 连接池"——Markdown 要遍历所有文件正则匹配，SQLite 一条 SQL 三十毫秒返回。

### 结论：不是二选一

两者的优势不可互相替代。Markdown 的可 diff 性 SQLite 永远做不到；SQLite 的结构化检索 Markdown 永远做不到。正确的做法是**分层**。

## 六、最佳实践：分层记忆架构

经过以上分析，一个能接受工程检验的最佳实践是**三层记忆架构**——不是"用 SQLite 替代 Markdown"，而是各层各管一件事：

```
┌─────────────────────────────────────────────────┐
│  第一层：约束层（Markdown）                       │
│  AGENTS.md / CLAUDE.md / .cursor/rules          │
│  存什么：项目规则、技术栈、命名约定                  │
│  检索方式：每次会话自动注入                         │
│  版本控制：git diff，PR review                    │
│  量级：<10 个文件                                 │
├─────────────────────────────────────────────────┤
│  第二层：决策层（ADR + Markdown）                  │
│  docs/decisions/NN-标题.md                       │
│  存什么：架构决策、技术选型、为什么选 A 不选 B        │
│  检索方式：按主题 grep / 按序号                    │
│  版本控制：git diff，PR review                    │
│  量级：<100 条                                   │
├─────────────────────────────────────────────────┤
│  第三层：经验层（SQLite）                          │
│  .memory/memory.db (sqlite-vec)                  │
│  存什么：踩坑日志、bug 解法、代码片段、会话摘要       │
│  检索方式：向量语义 + FTS5 + 元数据过滤混合查询      │
│  版本控制：.gitignore（或定期导出为 Markdown 归档）  │
│  量级：<1 万条                                   │
└─────────────────────────────────────────────────┘
```

### 为什么是三层

- **约束层用 Markdown**：因为它是"每次都要加载的规则"，量小、变化少、必须可 review。SQLite 在这个场景没有任何优势。
- **决策层用 ADR（Markdown）**：因为决策的推理过程需要人写、人审、git 追踪。ADR 一旦 Accepted 不可删除，Markdown 的可 diff 性是必需的。
- **经验层用 SQLite**：因为经验是"量大、低频读、高频写、需要语义检索"的。500 条踩坑记录里找相关的 3 条，SQL + 向量检索比 grep 快三个数量级。这个场景 Markdown 没有优势。

### 程序性记忆（skills）的位置

[上一篇文章](/2026/07/01/ai-memory-driven-development/)定义的第四类记忆——程序性记忆（`.opencode/skills/*/SKILL.md`）——用 Markdown。因为 skill 是"可复用的流程"，量小、需人工审核、需版本控制。从经验层的 SQLite 记忆中提取高频模式，固化为 skill 时，是从第三层向第一层/程序层的"蒸馏"。

### SQLite 经验层的表结构

一个最小可用的记忆库 schema：

```sql
-- 记忆主表（结构化数据 + 向量）
CREATE TABLE memories (
  id INTEGER PRIMARY KEY,
  content TEXT NOT NULL,           -- 记忆正文
  type TEXT NOT NULL,              -- bugfix / decision / learning / snippet
  project TEXT NOT NULL,           -- 项目路径，隔离多项目
  tags TEXT,                       -- 逗号分隔标签
  source TEXT,                     -- 来源：session-id / tool / manual
  created_at TEXT DEFAULT (datetime('now')),
  embedding BLOB                   -- 序列化的嵌入向量（768 维 float32）
);

-- 向量索引（sqlite-vec 虚拟表）
CREATE VIRTUAL TABLE memories_vec USING vec0(
  id INTEGER PRIMARY KEY,
  embedding FLOAT[768] distance_metric=cosine
);

-- 全文索引（SQLite FTS5）
CREATE VIRTUAL TABLE memories_fts USING fts5(
  content, type, project, tags
);

-- 插入时同步三张表（触发器或应用层保证）
-- 查询示例：语义 + 全文 + 元数据混合
SELECT m.id, m.content, m.type, m.created_at, v.distance
FROM memories m
JOIN memories_vec v ON m.id = v.id
JOIN memories_fts f ON m.id = f.rowid
WHERE v.embedding MATCH :query_embedding
  AND v.k = 10
  AND m.project = :project
  AND memories_fts MATCH '连接池 池耗尽'  -- FTS5 全文
ORDER BY v.distance;
```

注意三张表的设计：`memories`（结构化数据）+ `memories_vec`（向量索引）+ `memories_fts`（全文索引）。三者通过 `id` 关联，一条 SQL 三表 JOIN 混合查询。这是 SQLite 独有的优势——向量库和关系库天然不在同一个引擎里时，这种混合查询需要跨系统编排。

### 记忆生命周期

```
产生（session.idle hook 自动写入）
  → 积累（ADD-only，不更新不删除，类似 Mem0 v3 策略）
    → 检索（memory-query 工具按需查询）
      → 蒸馏（高频模式提取为 skill / ADR）
        → 遗忘（过时记忆标记 deprecated，不物理删除）
```

"不物理删除"借鉴 ADR 的不可变原则——过时的记忆本身也是历史，能解释"为什么曾经这么做"。遗忘不是 DELETE，是标记。

## 七、缺陷与边界：诚实地说

### sqlite-vec 的 pre-v1 风险

`sqlite-vec` 明确标注 "pre-v1, expect breaking changes"。v0.1.9 可用，但 v0.2 可能改 API。生产环境使用需要锁定版本号，做好升级适配的准备。这不是一个"配好用一辈子"的稳定依赖。

### 暴力扫描的性能天花板

默认线性扫描意味着记忆条目超过 1 万条后查询明显变慢。对于个人开发者，1 万条记忆大约对应 2-3 年的持续积累——大多数人到不了这个量级。但如果你在做高频记忆写入（每次工具调用都记），可能半年就到瓶颈。解法是迁移到 DuckDB vss（HNSW）或专用向量库。

### SQLite 的并发限制

SQLite 是单写者模型。如果多个 Agent 实例同时写记忆（多窗口、多项目并行），会遇到 `database is locked`。个人开发者单窗口使用无此问题，但团队协作场景需要考虑用 WAL 模式或迁移到 PostgreSQL。

### 嵌入模型的依赖

SQLite 存向量，但生成向量需要嵌入模型。这意味着记忆库运行时需要调用 embedding API（本地 bge-small 或远程 OpenAI）。离线场景下记忆检索不可用——除非用本地模型（如 `sqlite-lembed` 加载 gguf embedding，但质量低于云端模型）。

### Markdown 的协作优势 SQLite 补不了

这是最重要的一条边界。如果团队需要 PR review 每条新记忆，SQLite 的二进制 .db 文件无法 diff，review 流程断裂。这种场景下，即使经验层用 SQLite 做检索，仍需定期将关键经验导出为 Markdown 归档——因为**团队协作的信任机制建立在可 diff 的文本上**。

## 八、总结：SQLite 记忆库的本质

回到最初的问题：SQLite 做记忆库有价值吗？

**有价值，但不是万能的。** SQLite 的不可替代价值在一个具体场景：**当记忆条目超过 100 条、需要语义检索和结构化过滤时，SQL + 向量 + FTS5 的混合查询比 Markdown grep 快三个数量级。**

但它不能替代 Markdown 的可 diff 性，不能替代 ADR 的决策留痕，不能替代 skills 的程序性复用。它只补齐了"经验检索"这一层——而这一层，恰恰是当前所有 Markdown-first 工具的共同短板。

最佳实践不是"选 SQLite 还是选 Markdown"，而是**分层**：

| 层 | 载体 | 存什么 | 为什么选它 |
|----|------|--------|-----------|
| 约束 | Markdown | 项目规则 | 可注入、可 review |
| 决策 | ADR (Markdown) | 架构权衡 | 可 diff、不可删除 |
| 经验 | SQLite + sqlite-vec | 踩坑日志 | 可语义检索、可结构化过滤 |
| 程序 | Skills (Markdown) | 可复用流程 | 可发现、可加载 |

四层各管一件事，互不替代。约束告诉你边界，决策告诉你为什么，经验告诉你上次怎么做的，skills 告诉你怎么自动做。一个成熟的 AI 开发记忆系统，四层缺一不可。

opencode 的接入路径清晰：自定义工具做检索、Plugin hooks 做写入、MCP server 做跨工具共享。从一个 `memory-query.ts` 文件起步，不需要一步到位。

---

## 参考资料

- 本站 [AI 时代开发范式演化：从 Waterfall 到 Memory-Driven 的终局](/2026/07/01/ai-memory-driven-development/) — MDD 六阶段闭环，记忆资产分层
- 本站 [开源 Agent 架构深度拆解](/2026/06/26/ai-agent-2026-landscape/) — 框架可换记忆不可换，陈述性 vs 程序性记忆
- 本站 [深入浅出 RAG](/2026/06/24/ai-rag-engineering/) — 非参数化记忆，RAG 的本质是"可更新的外部记忆"
- 本站 [MCP 协议详解](/2026/06/22/ai-mcp-protocol/) — 记忆的标准化访问接口，M+N 线性解法
- [sqlite-vec](https://github.com/asg017/sqlite-vec) — SQLite 向量扩展，7.8k stars，pre-v1
- [LangGraph SqliteSaver](https://pypi.org/project/langgraph-checkpoint-sqlite/) — 官方 SQLite checkpoint 持久化
- [opencode 文档](https://opencode.ai/docs/) — 自定义工具、Plugin hooks、MCP 配置
- [ADR 模板](https://github.com/architecture-decision-record/architecture-decision-record) — Michael Nygard 格式，16.3k stars
